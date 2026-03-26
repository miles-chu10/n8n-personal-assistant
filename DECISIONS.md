# Decision Log — n8n Personal Life Manager

---

## 2026-03-16 — ngrok as persistent system daemon

**Context:** Telegram webhooks require a public HTTPS URL. n8n runs locally on localhost:5678 (HTTP only). A tunnel is needed to bridge the two.

**Decision:** Install ngrok as a macOS LaunchDaemon via `sudo ngrok service install`, auto-starting on boot.

**Reasoning:** The n8n built-in tunnel (`n8n start --tunnel`) is a CLI flag only — it can't be set via Docker env var cleanly. ngrok is battle-tested for this use case, has a free static subdomain, and the system daemon approach means zero manual action required after setup. The tunnel URL `discommodious-planular-keenan.ngrok-free.dev` is stable across restarts.

**Consequences:** If ngrok's free tier ever retires static subdomains, the Docker `WEBHOOK_URL` and Telegram webhook would need to be updated. The `ngrok.yml` config must define tunnels before `sudo ngrok service install` is run.

---

## 2026-03-16 — OpenRouter as AI provider

**Context:** The n8n template is pre-wired for OpenRouter. Direct API providers (OpenAI, Anthropic) were alternatives.

**Decision:** Use OpenRouter with a $5/month credit cap, monthly reset.

**Reasoning:** The workflow template already has the OpenRouter node configured. OpenRouter provides model-switching flexibility and has a built-in credit cap feature that prevents runaway costs for a personal tool. At estimated ~10–50 messages/day, actual spend is well under $5/month.

**Consequences:** Model quality depends on which model is selected in the OpenRouter credential/node. For heavy use or complex tasks, may want to switch to a more capable model (adds cost).

---

## 2026-03-16 — Self-hosted n8n over n8n Cloud

**Context:** n8n offers a managed cloud version at ~$20/month.

**Decision:** Self-hosted via Docker on local Mac.

**Reasoning:** Free. Data stays local. Sufficient for personal single-user use. n8n Cloud would add cost with no meaningful benefit for this use case.

**Consequences:** Requires the Mac to be on and Docker running for the bot to work. Not suitable if 24/7 uptime is ever needed — would need to migrate to a VPS or cloud host.
