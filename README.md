# Go Weekly Radar — AI Agent Skill

An AI agent skill that produces a weekly Go ecosystem radar report as a self-contained HTML file. Covers Go compiler/runtime, proposals, releases, ecosystem libraries, security issues, and community articles.

## What It Does

Every week, the agent:

1. **Collects** data from 7+ sources (GitHub API, go.dev/blog, golangweekly, HN)
2. **Curates** findings — not a raw dump, but "what actually matters this week"
3. **Generates** a self-contained HTML report with dark/light mode, collapsible sections, and working links

## Why AI Curation (Not a Scraper or Binary)

A scraper lists all 30 recent commits. An MCP server templates-fill descriptions. **AI curation** selects the 5 interesting commits and explains *why* they matter. That's the value.

| Approach | Collection | Curation | Output |
|----------|-----------|----------|--------|
| Scraper script | ✅ Deterministic | ❌ Lists everything | Raw data |
| MCP server | ✅ Deterministic | ❌ Template-fill | Structured JSON |
| **AI agent skill** | ✅ Tool-assisted | ✅ "Why this matters" | Curated HTML |

## Data Sources

| Source | URL | Method | Auth |
|--------|-----|--------|------|
| golang/go commits | `github.com/golang/go/commits/master` | `gh api` | GITHUB_TOKEN |
| Go proposals (open + closed) | `github.com/golang/go` issues with `Proposal` label | `gh api search/issues` | GITHUB_TOKEN |
| Go releases | `github.com/golang/go/git/refs/tags` | `gh api` | GITHUB_TOKEN |
| x/ library releases | `github.com/golang/{net,tools,sync,exp}/releases` | `gh api` | GITHUB_TOKEN |
| Trending Go repos | GitHub search: `language:go created:>=YYYY-MM-DD` | `gh api search/repositories` | GITHUB_TOKEN |
| Security issues | GitHub search: `label:security org:golang` | `gh api search/issues` | GITHUB_TOKEN |
| go.dev/blog | `go.dev/blog/all` | Browser (HTTP fetch) | None |
| golangweekly | `golangweekly.com/issues/N` | Browser (Cloudflare) | None |
| Hacker News | `hn.algolia.com/api/v1/search` | `curl` (JSON API) | None |

See [`references/data-sources.md`](references/data-sources.md) for the full catalog with status and notes.

## Scraping Toolchain

The agent uses a **hybrid fetch model** — different tools for different sources:

| Tool | Used For | Why |
|------|---------|-----|
| `gh api` (GitHub CLI) | All GitHub data | Structured JSON, 5000 req/h, `--jq` server-side filtering |
| Browser tool | go.dev/blog, golangweekly | JS rendering, Cloudflare Turnstile bypass |
| `curl` via terminal | HN Algolia JSON API | Direct HTTP, no rendering needed |
| Subagent / delegate | Fallback when terminal is blocked | Isolated session with terminal access |
| `web_search` | Finding article URLs | Search engine discovery |

**Fallback hierarchy:** `gh api` → browser → curl → subagent → skip

See [`references/scraping-toolchain.md`](references/scraping-toolchain.md) for recipes, decision trees, and parsing patterns.

## Usage

### With Hermes Agent
Load the skill and ask: "Run the Go weekly radar" or trigger automatically on architectural decisions.

### With Claude Code / OpenCode / Codex CLI
Copy `SKILL.md` and `references/` into your agent's skill/instructions directory. The agent needs:
- **Browser tool** (navigate to URLs, extract content)
- **Terminal tool** (run `gh api` commands)
- **File write tool** (save HTML output)

### Scheduled (Cron)
See [`references/cron-deployment.md`](references/cron-deployment.md) for Hermes cron-specific setup.

## Sample Output

See [`samples/go-radar-2026-06-29.html`](samples/go-radar-2026-06-29.html) for a real report.

## HTML Output Requirements

- **Self-contained** — no external deps, inline CSS
- **Dark/light mode** — CSS `prefers-color-scheme`, no JS required
- **Native `<details>/<summary>` accordions** — no custom JS toggle
- **All links `target="_blank" rel="noopener noreferrer"`** — open in new tab
- **No `href="#"` placeholders** — all real URLs

See [`references/design-decisions.md`](references/design-decisions.md) for the rationale behind each choice.

## Contributing

PRs welcome for:
- New data sources
- Pitfall reports and fixes
- HTML template improvements
- Tool-specific adaptation notes (Claude Code, OpenCode, etc.)

## License

MIT — see [LICENSE](LICENSE)