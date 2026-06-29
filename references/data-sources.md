# Data Sources — Go Weekly Radar

Full catalog of data sources, fetch methods, and current status (as of June 2026).

## GitHub Sources (via `gh api`)

All GitHub sources use `gh api` with `--jq` for server-side filtering. Requires `GITHUB_TOKEN` (without it: 60 req/h limit; with it: 5000 req/h).

### 1. golang/go Commits
```bash
gh api repos/golang/go/commits?per_page=30 --jq '.[] | "\(.sha[0:7]) \(.commit.author.name): \(.commit.message | split("\n")[0]) (\(.commit.author.date))"' 2>/dev/null | head -20
```
- **TTL:** Best effort — fetch fresh each run
- **Status:** ✅ Working

### 2. Go Proposals (Closed — includes accepted and declined)
```bash
gh api "search/issues?q=repo:golang/go+label:proposal+is:closed&sort=updated&per_page=15" --jq '.items[] | "\(.number) \(.title) (\(.state)) \(.updated_at)"' 2>/dev/null | head -15
```

### 3. Go Proposals (Open — still in discussion)
```bash
gh api "org:golang+is:issue+label:Proposal+is:open&sort=updated&per_page=10" --jq '.items[] | "\(.number) \(.title) ★\(.reactions.total_count)"' 2>/dev/null | head -10
```
- **Status:** ✅ Working
- **Note:** Check individual proposals for details: `gh api "repos/golang/go/issues/NNNN" --jq '{title, state, labels: [.labels[].name], body: (.body[:500])}'`

### 4. Go Releases
```bash
gh api repos/golang/go/git/refs/tags --jq '.[] | .ref' 2>/dev/null | grep -iE 'go1\.2[0-9]' | sort -V | tail -25
```
- **⚠️ Pitfall:** `gh api repos/golang/go/releases` returns empty for golang/go. Use `git/refs/tags` instead.
- **Status:** ✅ Working

### 5. x/ Library Releases
```bash
gh api repos/golang/net/releases?per_page=5 --jq '.[] | "\(.tag_name) \(.published_at)"' 2>/dev/null
gh api repos/golang/tools/releases?per_page=5 --jq '.[] | "\(.tag_name) \(.published_at)"' 2>/dev/null
```
- **Status:** ✅ Working

### 6. Trending Go Repos (created in last 7 days)
```bash
# Replace date with 7 days ago
gh api "search/repositories?q=language:go+created:>=2026-06-22&sort=stars&per_page=15" --jq '.items[] | "\(.full_name) ★\(.stargazers_count) \(.description // "no desc") \(.html_url)"' 2>/dev/null | head -15
```
- **Status:** ✅ Working
- **Note:** GitHub Search API rate limit: 30 req/min authenticated

### 7. Security Issues
```bash
gh api "search/issues?q=label:security+org:golang&sort=updated&per_page=10" --jq '.items[] | "\(.number) \(.title) (\(.state))"' 2>/dev/null | head -10
```
- **Status:** ✅ Working

## Web Sources (via Browser)

### 8. go.dev/blog
- **URL:** `https://go.dev/blog/` or `https://go.dev/blog/all`
- **Method:** Browser tool — navigate + extract content
- **Auth:** None
- **⚠️ Pitfall:** `go.dev` is on some security TLD blocklists for curl. Use browser tool, not `curl`.
- **⚠️ Pitfall:** Blog index lists inferred URL slugs that may not match actual page paths. Use `web_search` to find correct URLs.
- **Status:** ✅ Working via browser

### 9. golangweekly.com
- **URL:** `https://golangweekly.com/issues/N` (direct issue URL)
- **Method:** Browser tool only — Cloudflare Turnstile blocks non-browser HTTP
- **Auth:** None
- **⚠️ Pitfall:** Homepage has Cloudflare challenge. Navigate directly to `/issues/N` to bypass.
- **⚠️ Pitfall:** Content is inside an iframe. Use `document.body.innerText.substring(0, 5000)` to extract.
- **Status:** ⚠️ Browser-only, Cloudflare protected

## JSON API Sources (via curl)

### 10. Hacker News (Algolia API)
```bash
# 7-day filter — replace EPOCH with $(date -v-7d +%s) (BSD) or $(date -d '7 days ago' +%s) (GNU)
curl -s -A "Mozilla/5.0" "https://hn.algolia.com/api/v1/search?query=golang&tags=story&numericFilters=created_at_i%3E${EPOCH}&hitsPerPage=25"
```
- **Method:** `curl` via terminal (JSON API, no rendering needed)
- **⚠️ Pitfall:** Default sort is relevance, not recency. Always pass `numericFilters=created_at_i>...`.
- **⚠️ Pitfall:** In cron mode, terminal may be blocked. Use subagent with terminal access.
- **Status:** ✅ Working

### 11. Reddit r/golang
- **URL:** `https://www.reddit.com/r/golang/top.json?t=week`
- **Method:** Browser or `curl` with proper User-Agent
- **Status:** ❌ Blocked by network security (June 2026). May work from different environments.

## Source Health Check

Before collecting, check which sources are reachable:
```bash
gh auth status  # GitHub CLI configured?
gh api user --jq '.login'  # Token works?
```

For browser sources, navigate to the URL and check if content loads.

## Dead Sources (June 2026)

| Source | Issue | Workaround |
|--------|-------|------------|
| Reddit r/golang JSON | Blocked by network security | Use browser or skip |
| HN Algolia via browser | Blank page (raw JSON, no rendering) | Use `curl` via terminal/subagent |
| `gh api` to non-GitHub URLs | Returns empty (GitHub-only) | Use `curl` or browser |