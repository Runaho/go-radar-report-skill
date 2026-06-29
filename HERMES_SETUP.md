# Hermes Agent Setup Guide

This guide explains how to use the Go Weekly Radar skill with **Hermes Agent** specifically. If you use a different AI agent, see `SKILL.md` directly — this file is Hermes-specific.

## 1. Install the Skill

Clone this repo into Hermes' skills directory:

```bash
cd ~/.hermes/skills/research/
git clone https://github.com/Runaho/go-radar-report-skill.git go-weekly-radar
```

This places the skill at `~/.hermes/skills/research/go-weekly-radar/SKILL.md`, which Hermes auto-discovers.

**If the directory already exists** (e.g. an older version), remove it first:
```bash
rm -rf ~/.hermes/skills/research/go-weekly-radar
git clone https://github.com/Runaho/go-radar-report-skill.git ~/.hermes/skills/research/go-weekly-radar
```

## 2. Required Environment

- **`gh` CLI** — authenticated (`gh auth login`). Used for all GitHub data sources.
- **`GITHUB_TOKEN`** — environment variable or `gh` auth. Without it: 60 req/h instead of 5000/h.
- **Browser tool** — Hermes has this built-in (`browser_navigate`, `browser_snapshot`, `browser_console`).
- **Terminal tool** — Hermes has this built-in.
- **File write tool** — Hermes has this built-in (`write_file`).

## 3. Manual Run

Just ask Hermes:

> "Run the Go weekly radar"

Or trigger it with:
> "Think deeper about Go ecosystem changes this week"

The skill auto-triggers on architectural decisions and Go-related analysis tasks.

## 4. Cron Job Setup

### Create the cron job

```bash
hermes cron create \
  --name "go-weekly-radar" \
  --schedule "0 7 * * 1" \
  --model "minimax-m3:cloud" \
  --provider "ollama-launch" \
  --skills "go-weekly-radar" \
  --toolsets "terminal,file,web" \
  --deliver origin
```

### What each flag does

| Flag | Value | Why |
|------|-------|-----|
| `--schedule` | `0 7 * * 1` | Every Monday at 07:00 |
| `--model` | `minimax-m3:cloud` | Best depth for Go Radar (923 lines, 41KB output) |
| `--provider` | `ollama-launch` | Cloud provider |
| `--skills` | `go-weekly-radar` | Loads the skill from `~/.hermes/skills/research/go-weekly-radar/` |
| `--toolsets` | `terminal,file,web` | Reduces MCP tool overload (48→~15 tools, saves ~10K context) |
| `--deliver` | `origin` | Sends result back to the chat that created the job |

### Why `--toolsets terminal,file,web`?

Cron jobs load 48+ MCP tools by default, wasting ~10K context tokens. Filtering to `terminal,file,web` loads only what the skill needs:
- **terminal** — `gh api` commands
- **file** — `write_file` for HTML output
- **web** — `web_search` for article discovery

Browser tools are loaded via the built-in browser toolset, not MCP — no need to include MCP browser tools explicitly.

### Model alternatives

| Model | Latency | Output | Use when |
|-------|---------|--------|----------|
| `minimax-m3:cloud` | 3-15s | 923 lines, 41KB | Default — best depth |
| `deepseek-v4-flash:cloud` | 2-10s | 564 lines, 23KB | Need speed over depth |
| `gemma4:31b-cloud` | 4-10s | 182 lines | Minimum acceptable, shallow |

**Never use:** `nemotron-3-super:cloud` (40-440s, never finishes), `nemotron-3-nano:30b-cloud` (99 lines, too shallow).

## 5. Cron Job Management

```bash
# List jobs
hermes cron list

# Manual trigger
hermes cron run <job_id>

# Update model
hermes cron update <job_id> --model "deepseek-v4-flash:cloud"

# Pause/resume
hermes cron pause <job_id>
hermes cron resume <job_id>

# Delete
hermes cron remove <job_id>
```

## 6. File Output Path

Cron jobs save HTML to:
```
~/.hermes/reports/go-radar/go-radar-YYYY-MM-DD_HHMM.html
```

The final response includes `MEDIA:/absolute/path/to/file.html` which Hermes delivers as a file attachment to Telegram (or whatever channel the job is configured for).

## 7. Updating the Skill

When the repo is updated:

```bash
cd ~/.hermes/skills/research/go-weekly-radar
git pull
```

No restart needed — Hermes loads skill content fresh each run.

## 8. Critical Cron Pitfalls

These are documented in `references/cron-deployment.md` and `references/pitfalls.md`. The most important ones:

### Self-patching death spiral
**Do NOT use `skill_view` or `skill_manage` during cron execution.** The skill is already loaded in the system prompt. Reading or modifying skills mid-run burns 2-4 API calls from the 16-call budget. This is the #1 cause of failed cron deliveries.

If the skill needs updating: finish the task FIRST, deliver the HTML, then note what to update and let the next session handle it.

### Terminal blocked in cron
`terminal` with `curl`/`wget` returns `pending_approval` (no user to approve). Use `gh api` (pre-approved) or `delegate_task` with `toolsets=['terminal','file']` for curl-based sources.

### `execute_code` blocked
Returns BLOCKED error in cron mode. Workaround: `write_file` to `/tmp/script.py`, then `terminal(command="python3 /tmp/script.py")`.

### golangweekly Cloudflare
Navigate directly to `https://golangweekly.com/issues/N` — the homepage has a Cloudflare Turnstile challenge that blocks the click path.

## 9. Sample Cron Prompt

If you need to set a custom prompt for the cron job:

```
Run the weekly Go ecosystem radar for the past 7 days.

CRITICAL RULES:
1. NO terminal commands that need approval (curl, wget, firecrawl) — use gh api only
2. NO skill_view or skill_manage — skill is already loaded
3. Browser snapshot truncates at ~3090 lines — use browser_console for long pages
4. All HTML links MUST use target="_blank" rel="noopener noreferrer"
5. NEVER use href="#" placeholders — use real URLs
6. Save HTML to ~/.hermes/reports/go-radar/go-radar-YYYY-MM-DD_HHMM.html
7. Final response: brief summary + MEDIA:/absolute/path/to/file.html

Follow the SKILL.md execution flow. Start with gh api for GitHub data,
then browser for go.dev/blog and golangweekly. Curate findings — don't
just list everything, explain WHY each item matters.
```