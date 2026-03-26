# Session Log — n8n Personal Life Manager

---

## 2026-03-16 — Session 2: Git Repo Planning & Backup Strategy

### Goal
Design the git repo structure for `n8n-personal-assistant` on GitHub — what to include, why, and how to execute it. Produce a detailed implementation plan reviewable by future Miles or a coding agent.

### What Was Built / Changed
- Created `docs/superpowers/plans/2026-03-16-git-repo-bootstrap.md` — full 8-chunk implementation plan for bootstrapping the git repo, exporting the workflow, writing infrastructure scripts, and creating a disaster recovery runbook
- Plan was reviewed by a parallel agent (completeness/structure review) and revised with 7 fixes: pre-flight checks, `docs/` commit, `gh` CLI validation, `.env.example` extended with API key fields, `export-workflow.sh` updated to source `.env`, git commit failure guidance added, and final validation task added

### Key Decisions
| Decision | Why | Alternatives Rejected |
|----------|-----|----------------------|
| Plan-first before executing | Session was exploratory — Miles asked "what should I include?" before doing anything | Jumping straight to `git init` without a strategy |
| Workflow JSON export via n8n REST API | Scriptable, repeatable, doesn't require UI interaction | Manual UI export — not automatable |
| `RESTORE.md` as top-level file | Disaster recovery is the primary value of the repo; it should be impossible to miss | Burying restore steps inside README.md |

### Problems Hit
- No actual code was executed — session was planning-only
- Infrastructure review agent type (`engineering:code-reviewer`) was unavailable; only completeness review was completed via the Plan agent

### State at End of Session
- ✅ Implementation plan written and reviewed: `docs/superpowers/plans/2026-03-16-git-repo-bootstrap.md`
- ⏳ Plan not yet executed — git repo not initialized, no workflow exported, no scripts created
- ⏳ Next step: run the plan (either manually or via a Claude Code session using `superpowers:executing-plans`)

---

## 2026-03-16 — Session 1: Initial Setup & Go-Live

### Goal
Get the n8n personal life manager bot fully working — Telegram receiving messages, AI responding, and Google services connected.

### What Was Built / Changed
- Imported n8n workflow template: "Personal life manager with Telegram, Google services & voice-enabled AI"
- Connected all credentials: Telegram, OpenRouter, Gmail, Google Calendar, Google Tasks
- Set Google Calendar node to `milesdchu@gmail.com`
- Configured ngrok as HTTPS tunnel for Telegram webhook requirement
- Installed ngrok as macOS system daemon (auto-starts on boot)
- Set Docker `WEBHOOK_URL` env var to ngrok tunnel URL
- Activated workflow — Telegram webhook auto-registered by n8n
- Set OpenRouter credit limit: $5/month, monthly reset
- Disabled Business Mode in Telegram BotFather

### Key Decisions
| Decision | Why | Alternatives Rejected |
|----------|-----|----------------------|
| ngrok over n8n built-in tunnel | n8n built-in tunnel requires CLI flag (`--tunnel`) not an env var; ngrok is more reliable and inspectable | `N8N_TUNNEL=true` env var — doesn't work; `n8n start --tunnel` Docker command — untested |
| ngrok as system daemon | Survives reboots; no manual action required | Running ngrok manually in terminal — dies on terminal close |
| OpenRouter over direct OpenAI/Claude API | Workflow template pre-wired for OpenRouter; easy model switching; cost cap feature | Direct API — would require workflow modifications |
| $5/month OpenRouter cap | Personal use is low-volume; ~$0.50–$3/month expected | Higher cap — unnecessary; lower cap — risk of cutting out mid-month |

### Problems Hit
- `N8N_TUNNEL=true` env var does nothing — n8n tunnel is CLI-flag only (`n8n start --tunnel`)
- Telegram webhook requires HTTPS — localhost:5678 rejected until ngrok was set up
- `ngrok service install` failed without `sudo` (writes to `/Library/LaunchDaemons/`)
- `ngrok service status` doesn't exist — valid subcommands: install/start/stop/restart/run/uninstall
- Window Buffer Memory node had broken node reference (`$('Listen for incoming events')`) — node name mismatch
- Telegram BotFather had Business Mode ON — caused webhook handling issues; must be OFF

### State at End of Session
- ✅ Bot fully live and responding on Telegram
- ✅ ngrok running as system daemon (auto-start on boot)
- ✅ All Google credentials connected
- ✅ OpenRouter $5/mo cap set
- ✅ Workflow active (737 connections, 2,469 HTTP requests processed in first session)
