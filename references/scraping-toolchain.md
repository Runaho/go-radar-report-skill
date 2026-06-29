# Scraping Toolchain — How the Agent Collects Data

This document explains which tools the agent uses for data collection, why each tool was chosen, and how to parse the results.

## Tool Inventory

| Tool | What It Does | When To Use |
|------|-------------|-------------|
| `gh api` (GitHub CLI) | GitHub REST API queries with `--jq` filtering | All GitHub data: commits, proposals, releases, trending repos, security issues |
| Browser tool (navigate + snapshot) | Fetch web pages with JS rendering | go.dev/blog, golangweekly.com, any JS-rendered or Cloudflare-protected page |
| Browser tool (console/eval) | Execute JavaScript in page context | Extract content from truncated pages, parse iframe content |
| `curl` via terminal | Direct HTTP requests | JSON APIs (HN Algolia), RSS feeds, raw HTML pages |
| Subagent / delegate | Isolated terminal session | When terminal is blocked (cron mode), or for parallel data collection |
| `web_search` | Search engine queries | Finding blog post URLs, HN discussions, article discovery |
| `web_extract` | URL → markdown extraction | Quick article content extraction from discovered URLs |
| MCP browser tools (e.g. Lightpanda) | Alternative browser automation | When primary browser tool fails or is unavailable |

## Source-Fetch Decision Tree

When a source is unreachable, fall through this list before giving up:

```
1. gh api (fastest for github.com URLs, pre-approved in most environments)
   ↓ fail or non-GitHub URL
2. Browser tool (navigate + snapshot/console)
   ↓ fail (Cloudflare, bot detection, TLD block, timeout)
3. curl via terminal (direct HTTP, JSON APIs)
   ↓ fail (terminal blocked in cron, bot detection)
4. Subagent with terminal access (isolated session)
   ↓ fail
5. Skip source, write "Quiet week" in that section
```

**Rule:** Do NOT retry the same failed approach more than twice. After 2 failures, skip and move on.

## Why Hybrid Fetch Model

### Why `gh api` for GitHub (not browser)
- **Structured JSON** — no HTML parsing needed
- **Rate limit** — 5000 req/h authenticated vs 60 req/h unauthenticated
- **`--jq` filtering** — server-side field selection, reduces output size
- **Reliability** — no JS rendering, no Cloudflare, no bot detection

### Why browser for go.dev/blog and golangweekly (not curl)
- **go.dev** — on some security TLD blocklists for curl. Browser tool bypasses this.
- **golangweekly.com** — Cloudflare Turnstile JS challenge. Only a real browser passes.
- **JS rendering** — both sites render content via JavaScript. `curl` gets empty HTML.

### Why curl for HN Algolia (not browser)
- **JSON API** — `hn.algolia.com` returns raw JSON, not HTML. Browser renders nothing.
- **No auth needed** — public API, no rate limit in normal use.
- **Fast** — single HTTP request, no rendering overhead.

## Browser Request Model

### Navigation + Content Extraction
1. **Navigate** to the target URL
2. **Extract** content via snapshot (short pages) or JavaScript eval (long pages)

### Snapshot Truncation Workaround
Pages longer than ~3090 lines truncate in the accessibility tree snapshot. For long pages (blog posts, newsletter issues):

```javascript
// Extract content in chunks via browser console
document.body.innerText.substring(0, 5000)    // first 5000 chars
document.body.innerText.substring(5000, 10000)  // next 5000 chars
// Continue until full text is extracted
```

### iframe Content Extraction (golangweekly)
golangweekly puts article content inside an iframe. The snapshot only shows truncated previews. Full content:
```javascript
document.body.innerText.substring(0, 5000)
```

## MCP Browser Tool Output Parsing

Some MCP browser tools (e.g. Lightpanda) wrap output in a JSON envelope:

```json
{"result": "\n```\n<actual content>\n```"}
```

### Standard Parser
```python
import json, re

def unwrap_mcp_browser(raw_text):
    """Strip the {"result": "..."} wrapper and inner ``` fences."""
    outer = json.loads(raw_text)
    inner = outer["result"]
    inner = re.sub(r'^\s*```\s*\n?', '', inner)
    inner = re.sub(r'\n?\s*```\s*$', '', inner)
    return inner
```

### State Pollution Fix
MCP browser tools cache the last page. If you call `markdown()` without a preceding `goto(url)`, it returns cached content or a "Duplicate tool output" warning.

**Fix:** Always pair `goto(url)` + `markdown()`:
```
1. goto(url="https://example.com")  → fresh navigation
2. markdown()                        → fresh content
3. (next page)
4. goto(url="https://other.com")    → fresh navigation
5. markdown()                        → fresh content
```

## `gh api` Recipes

### Commits
```bash
gh api repos/golang/go/commits?per_page=30 \
  --jq '.[] | "\(.sha[0:7]) \(.commit.author.name): \(.commit.message | split("\n")[0]) (\(.commit.author.date))"' \
  2>/dev/null | head -20
```

### Proposals (closed)
```bash
gh api "search/issues?q=repo:golang/go+label:proposal+is:closed&sort=updated&per_page=15" \
  --jq '.items[] | "\(.number) \(.title) (\(.state)) \(.updated_at)"' \
  2>/dev/null | head -15
```

### Proposals (open)
```bash
gh api "search/issues?q=org:golang+is:issue+label:Proposal+is:open&sort=updated&per_page=10" \
  --jq '.items[] | "\(.number) \(.title) ★\(.reactions.total_count)"' \
  2>/dev/null | head -10
```

### Proposal Details
```bash
gh api "repos/golang/go/issues/NNNN" \
  --jq '{title, state, labels: [.labels[].name], body: (.body[:500])}' \
  2>/dev/null
```

### Releases
```bash
# golang/go — use git/refs/tags (releases endpoint returns empty)
gh api repos/golang/go/git/refs/tags --jq '.[] | .ref' 2>/dev/null | grep -iE 'go1\.2[0-9]' | sort -V | tail -25

# x/ libraries
gh api repos/golang/net/releases?per_page=5 --jq '.[] | "\(.tag_name) \(.published_at)"' 2>/dev/null
gh api repos/golang/tools/releases?per_page=5 --jq '.[] | "\(.tag_name) \(.published_at)"' 2>/dev/null
```

### Trending Repos
```bash
# Replace date with 7 days ago
gh api "search/repositories?q=language:go+created:>=2026-06-22&sort=stars&per_page=15" \
  --jq '.items[] | "\(.full_name) ★\(.stargazers_count) \(.description // "no desc") \(.html_url)"' \
  2>/dev/null | head -15
```

### Trending Repo Details
```bash
gh api repos/<owner>/<repo> \
  --jq '{name: .full_name, stars: .stargazers_count, description, topics: .topics, license: .license.spdx_id}' \
  2>/dev/null
```

### Security Issues
```bash
gh api "search/issues?q=label:security+org:golang&sort=updated&per_page=10" \
  --jq '.items[] | "\(.number) \(.title) (\(.state))"' \
  2>/dev/null | head -10
```

### Commit Detail Analysis
```bash
gh api repos/golang/go/commits/<sha> \
  --jq '{sha: .sha[0:7], message: (.commit.message[:300]), files: [.files[].filename]}' \
  2>/dev/null
```

## HN Algolia Recipe

```bash
# BSD/macOS date math
EPOCH=$(date -v-7d +%s)
# GNU/Linux date math
# EPOCH=$(date -d '7 days ago' +%s)

curl -s -A "Mozilla/5.0" \
  "https://hn.algolia.com/api/v1/search?query=golang&tags=story&numericFilters=created_at_i%3E${EPOCH}&hitsPerPage=25" \
  -o /tmp/hn_go.json
```

Parse:
```python
import json
with open('/tmp/hn_go.json') as f:
    hits = json.load(f)['hits']
for h in hits:
    print(f"[{h['points']}p/{h['num_comments']}c] {h['title'][:100]}")
    print(f"  {h.get('url','') or f'https://news.ycombinator.com/item?id={h[\"objectID\"]}'}")
```

## Common Failure Modes

| Symptom | Cause | Fix |
|---------|-------|-----|
| `gh api` returns empty | Non-GitHub URL or no data | Check URL — `gh api` only works for github.com |
| Browser snapshot truncated | Page >3090 lines | Use `browser_console` with `document.body.innerText.substring()` |
| `curl` to go.dev blocked | TLD on security blocklist | Use browser tool instead |
| `curl` to Reddit empty | Bot detection | Use browser tool or skip |
| MCP browser "Duplicate output" | No fresh `goto()` before `markdown()` | Re-issue `goto(url)` then `markdown()` |
| golangweekly homepage empty | JS-rendered landing | Navigate to `/issues/N` directly |
| HN Algolia returns old stories | Default sort = relevance | Add `numericFilters=created_at_i>...` |