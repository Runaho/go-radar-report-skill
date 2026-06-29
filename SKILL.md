---
name: go-weekly-radar
description: "Weekly Go ecosystem radar — compiler, stdlib, ecosystem, community. Produces a self-contained HTML report with dark/light mode, collapsible sections, and working links."
version: 3.0.0
author: Runaho (Güneş Korkmaz)
license: MIT
metadata:
  tags: [go, weekly, radar, research, report]
  sources: [github-api, go-dev-blog, golangweekly, hn-algolia]
  tools-required: [browser, terminal, file-write]
---

# Go Weekly Radar

Produce a weekly Go ecosystem radar report as a self-contained HTML file.

## Tool Requirements

The agent needs these tool categories:
- **Browser tool** — navigate to URLs, extract page content, execute JavaScript in page context
- **Terminal tool** — run `gh api` (GitHub CLI) commands
- **File write tool** — save HTML output to disk
- **Search tool** (optional) — find article URLs via web search

If any of these are unavailable, see `references/scraping-toolchain.md` for fallback strategies.

## Coverage Pillars

1. **Official Go**: language, compiler, runtime, stdlib, go command, x/ packages, proposals, releases
2. **Ecosystem**: frameworks, libraries, infra, observability, testing, CLI, databases, tooling
3. **Community**: substantive discussions, benchmark-driven articles, scaling reports, technical writeups

## Must-Look Signals

- One compiler/runtime/spec/toolchain item
- One accepted or likely-accept proposal
- One stdlib change
- One release/RC
- One security-relevant item in a widely used package or tool
- One repo with unusual star velocity
- One major ecosystem release, especially v2+

## Time Window

Last 7 days through today.

## Data Sources

See `references/data-sources.md` for the full catalog with URLs, methods, and status.

**Primary sources:**
1. GitHub API (`gh api`) — commits, proposals, releases, trending repos, security issues
2. go.dev/blog — browser navigation
3. golangweekly — browser navigation (Cloudflare protected)
4. HN Algolia — curl (JSON API)

**Fetch order:** `gh api` first (fastest, structured), then browser sources, then curl.

## Execution Flow

### Phase 1: Source Research

Check sources in this order. Stop each source after collecting findings — do not deep-dive.

1. **GitHub commits** — `gh api repos/golang/go/commits?per_page=30` → extract recent commits touching compiler/runtime/stdlib
2. **Proposals (closed)** — `gh api "search/issues?q=repo:golang/go+label:proposal+is:closed&sort=updated&per_page=15"` → look for accepted labels
3. **Proposals (open)** — `gh api "search/issues?q=org:golang+is:issue+label:Proposal+is:open&sort=updated&per_page=10"`
4. **Releases** — `gh api repos/golang/go/git/refs/tags` → filter `go1.2*` tags
5. **x/ releases** — `gh api repos/golang/{net,tools}/releases?per_page=5`
6. **Trending repos** — `gh api "search/repositories?q=language:go+created:>=YYYY-MM-DD&sort=stars&per_page=15"`
7. **Security** — `gh api "search/issues?q=label:security+org:golang&sort=updated&per_page=10"`
8. **Go blog** — browser navigate to `go.dev/blog/` → recent posts
9. **golangweekly** — browser navigate to `golangweekly.com/issues/N` (latest issue)
10. **HN** — `curl` or subagent: `hn.algolia.com/api/v1/search?query=golang&tags=story&numericFilters=created_at_i>...`

### Phase 2: Synthesis

Compile findings into structured categories:
- 🔧 **Official Go** — compiler, runtime, proposals, releases
- 📦 **Ecosystem Highlights** — libraries, tools, trending repos
- 🔥 **Security** — CVEs, vulnerabilities, supply chain
- 📰 **News & Articles** — blog posts, community articles, HN discussions

If a category has nothing, write "Quiet week" — never leave it blank.

### Phase 3: HTML Generation

Build self-contained HTML with:
- Inline CSS (no external dependencies)
- Dark/light mode via `prefers-color-scheme` media query + CSS variables
- Native `<details>/<summary>` accordions (NO custom JS)
- Responsive, mobile-friendly (viewport meta + media queries)
- All links with `target="_blank" rel="noopener noreferrer"`
- No `href="#"` placeholders — all real URLs

See `references/design-decisions.md` for the rationale behind each HTML requirement.

### Phase 4: Delivery

1. Verify file exists and is non-empty
2. Include file path in response (e.g. `MEDIA:/absolute/path/to/file.html`)
3. Brief highlights summary

## HTML Requirements (Self-Contained, No External Deps)

- **Dark/light mode**: CSS `prefers-color-scheme` media query, CSS variables. No JS required.
- **Mobile-friendly**: viewport meta + media queries
- **⛔ CRITICAL: Per-section accordion MUST use native `<details>/<summary>` elements.** NO custom JS toggle, NO onclick handlers, NO classList.toggle. Just `<details><summary>Section header</summary>content</details>`.
- **⛔ CRITICAL: Every external link MUST have a real, working URL — NEVER use `href="#"` placeholders.** If a real URL cannot be found, use the canonical source URL (issue page, repo root, blog index) and label clearly.
- **⛔ CRITICAL: Every external link MUST open in a new tab via `target="_blank"` AND include `rel="noopener noreferrer"`** (security: prevents reverse tabnabbing). Format: `<a href="https://..." target="_blank" rel="noopener noreferrer">Title</a>`.
- All findings inline — no frames, no embeds, no empty placeholders
- CSS variables for theming, minimal inline JS for toggle only
- Clean technical aesthetic — no filler

## File Output

- Path: `~/.hermes/reports/go-radar/go-radar-YYYY-MM-DD_HHMM.html` (or equivalent in your environment)
- `YYYY-MM-DD` = current date, `HHMM` = current time in 24h format
- No `-tz` suffix, no extra dashes

## Exclusions

- Bot PRs, dependency bumps, typo/formatting-only changes
- Doc-only churn without technical consequence
- Shallow tutorials, marketing posts

## References

- `references/data-sources.md` — Full source catalog with URLs, methods, auth, and status
- `references/scraping-toolchain.md` — Tool inventory, fetch decision tree, `gh api` recipes, parsing patterns
- `references/design-decisions.md` — Rationale for AI curation, hybrid fetch model, HTML requirements
- `references/cron-deployment.md` — Hermes Agent cron-specific setup (optional, skip if not using Hermes)
- `references/pitfalls.md` — Known edge cases and failure modes