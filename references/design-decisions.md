# Design Decisions — Go Weekly Radar

This document explains the "why" behind each architectural choice.

## 1. AI Curation Over Deterministic Collection

**Decision:** The agent collects raw data via tools, then curates it with reasoning — not a template-fill.

**Why:** Go radar's value is "30 commits → 5 interesting ones with context." A scraper lists all 30. An MCP server template-fills descriptions. Only AI curation produces "why this matters" explanations.

| Approach | Collection | Curation | Output Quality |
|----------|-----------|----------|---------------|
| Scraper script | ✅ Deterministic | ❌ Lists everything | Raw data dump |
| MCP server | ✅ Deterministic | ❌ Template-fill | Structured but shallow |
| **AI agent skill** | ✅ Tool-assisted | ✅ "Why this matters" | Curated, contextual |

**Tradeoff:** AI curation costs API calls (budget). But 12-14 calls within a 16-call budget is fine.

## 2. Hybrid Fetch Model (Not curl-only, Not browser-only)

**Decision:** Use `gh api` for GitHub, browser for JS-rendered pages, curl for JSON APIs.

**Why:**
- `gh api` → structured JSON, 5000 req/h, `--jq` filtering. Best for GitHub.
- Browser → JS rendering, Cloudflare bypass. Required for go.dev/blog and golangweekly.
- `curl` → direct HTTP for JSON APIs (HN Algolia). No rendering needed.

**Tradeoff:** Three different tools to manage. But each is optimal for its source type.

## 3. Browser Request Model for Web Sources

**Decision:** Use browser navigation (not curl) for go.dev/blog and golangweekly.com.

**Why:**
- **go.dev** is on some security TLD blocklists for curl. Browser bypasses this.
- **golangweekly.com** uses Cloudflare Turnstile (JS challenge). Only a real browser passes.
- Both sites render content via JavaScript. `curl` gets empty HTML shell.

**Tradeoff:** Browser is slower than curl. But it's the only option for these sources.

## 4. `gh api` for GitHub (Not Browser)

**Decision:** Use `gh api` CLI instead of browser for all GitHub data.

**Why:**
- **Structured JSON** — no HTML parsing needed
- **Rate limit** — 5000 req/h authenticated vs 60 req/h unauthenticated
- **`--jq` filtering** — server-side field selection, reduces output size
- **Reliability** — no JS rendering, no Cloudflare, no bot detection

**Tradeoff:** Requires `gh` CLI installed and authenticated. But it's a standard developer tool.

## 5. Native `<details>/<summary>` Accordions (Not JS)

**Decision:** Use native HTML `<details>` elements for collapsible sections. No custom JavaScript.

**Why:** Telegram in-app browser (and many other embedded browsers) don't execute custom JS toggle logic reliably. Native HTML works everywhere.

**Tradeoff:** Less visual customization. But reliability > aesthetics.

## 6. `target="_blank" rel="noopener noreferrer"` on All Links

**Decision:** Every external link opens in a new tab with `rel="noopener noreferrer"`.

**Why:**
- **Telegram in-app browser** loses scroll position on same-tab navigation
- **Security:** `noopener` prevents reverse tabnabbing (new tab can't access `window.opener`)
- **Privacy:** `noreferrer` prevents referrer leak to external sites

**Tradeoff:** Users prefer same-tab links in desktop browsers. But the primary delivery channel is Telegram.

## 7. No `href="#"` Placeholders

**Decision:** Every link must have a real, working URL. Never use `href="#"`.

**Why:** Dead links are worse than no links. If a real URL can't be found, fall back to the canonical source URL (issue page, repo root, blog index) with an explicit label.

**Tradeoff:** Some links may point to index pages instead of specific articles. But at least they go somewhere.

## 8. Dark/Light Mode via CSS Variables (Not JS Toggle)

**Decision:** Use `prefers-color-scheme` media query with CSS variables. Zero JavaScript.

**Why:**
- Works in all browsers (including embedded/limited browsers)
- Respects user's system preference automatically
- No JS dependency, no flash of wrong theme
- CSS variables enable easy theming

**Tradeoff:** No manual toggle button. But system preference is usually correct.

## 9. Self-Contained HTML (No External Dependencies)

**Decision:** All CSS inline, no external fonts, no JS libraries, no CDN links.

**Why:**
- **Delivery as file attachment** — works offline, no network needed
- **No external dependency risk** — CDN down = broken page
- **Reproducible** — same HTML renders identically forever

**Tradeoff:** Larger file size (~18KB vs ~5KB with external CSS). But 18KB is trivial.

## 10. Weekly Cadence (Not Daily, Not Monthly)

**Decision:** Weekly report, covering the past 7 days.

**Why:**
- **Daily** = too much noise. Go doesn't move that fast.
- **Monthly** = misses releases, proposal decisions, security fixes.
- **Weekly** = sweet spot. Enough happens to be interesting, not so much that it's noise.

**Tradeoff:** Breaking news (critical CVE) may be up to 7 days old. But for a weekly radar, that's acceptable.

## 11. HTML Output (Not JSON, Not Markdown)

**Decision:** Final output is a self-contained HTML file.

**Why:**
- **Viewable in any browser** — no tool needed
- **Delivered as file attachment** — Telegram, email, Slack all handle HTML files
- **Rich formatting** — dark/light mode, accordions, colored tags, links
- **Self-contained** — no external deps, works offline

**Tradeoff:** Not machine-readable. But the audience is humans, not pipelines.

## 12. SQLite Not Required (Unlike CTI-MCP)

**Decision:** No database. Data is collected fresh each run.

**Why:**
- Go radar is a **weekly snapshot** — not continuous monitoring like CTI-MCP
- ~70 items per week — fits easily in memory
- No cross-referencing needed (CTI-MCP cross-refs CVEs across 5 sources)
- No background refresh needed (CTI-MCP refreshes every 6h)

**Tradeoff:** Each run re-fetches everything. But 7 sources in ~9 API calls is fast enough.