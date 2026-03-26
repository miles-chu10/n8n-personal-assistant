# n8n Personal Life Manager

Personal AI assistant bot. Message it on Telegram → it handles Gmail, Google Calendar, Google Tasks, and voice notes via OpenRouter AI.

---

## Architecture

```
Telegram (you)
     ↓  message
Telegram API
     ↓  webhook POST
ngrok tunnel (HTTPS)
https://discommodious-planular-keenan.ngrok-free.dev
     ↓
localhost:5678 (n8n in Docker)
     ↓
AI Agent node (OpenRouter → model)
     ├── Gmail tool
     ├── Google Calendar tool
     ├── Google Tasks tool
     └── Voice transcription (Whisper)
     ↓
Telegram reply
```

n8n is the orchestration layer. It receives the Telegram message, passes it to an AI agent with tool access, and sends the response back.

---

## Infrastructure

### Docker — n8n

```bash
docker run \
  --user=node \
  --env=NODE_ENV=production \
  --env=N8N_RELEASE_TYPE=stable \
  --env=WEBHOOK_URL=https://discommodious-planular-keenan.ngrok-free.dev \
  --volume=n8n_data:/home/node/.n8n \
  -p 5678:5678 \
  -d docker.n8n.io/n8nio/n8n:latest
```

- n8n version: 2.9.2
- Data persisted in Docker volume: `n8n_data`
- Editor UI: http://localhost:5678

### ngrok — HTTPS tunnel

ngrok exposes localhost:5678 to the public internet with HTTPS, which Telegram requires for webhooks.

- Tunnel URL: `https://discommodious-planular-keenan.ngrok-free.dev`
- Config: `~/Library/Application Support/ngrok/ngrok.yml`
- Installed as macOS system daemon — auto-starts on boot

```bash
# Manage the daemon
sudo ngrok service start
sudo ngrok service stop
sudo ngrok service restart
# (no 'status' subcommand — use curl below to verify)

# Verify tunnel is live
curl -s http://127.0.0.1:4040/api/tunnels | python3 -m json.tool
```

---

## Credentials

| Service | Notes |
|---------|-------|
| Telegram Bot | Created via BotFather. Token stored in n8n credentials. |
| OpenRouter | $5/month limit, monthly reset. Key stored in n8n credentials. |
| Gmail | OAuth. milesdchu@gmail.com |
| Google Calendar | milesdchu@gmail.com. Set in Calendar node. |
| Google Tasks | OAuth. milesdchu@gmail.com |

All credentials are stored inside n8n (not in any file). Manage at http://localhost:5678/credentials.

---

## Workflow

**Template:** "Personal life manager with Telegram, Google services & voice-enabled AI"

Loaded from n8n template library. Key nodes:

| Node | Purpose |
|------|---------|
| Telegram Trigger ("Listen for incoming events") | Receives messages from your Telegram bot |
| Voice or Text | Routes audio messages vs text messages |
| Transcribe a recording | Whisper transcription for voice notes |
| AI Agent | OpenRouter model with tool access |
| Gmail tool | Read/send emails |
| Google Calendar tool | Read/create events |
| Google Tasks tool | Read/create tasks |
| Window Buffer Memory | Stores last 5 message turns per user (keyed by Telegram chat ID) |
| Telegram Send | Returns the AI response to your chat |

### Activation

The workflow must be **Active** (toggle in top-right of editor). Do not use the "Execute" button — that's for testing before activation. Once active, just message the bot and it responds automatically.

---

## Telegram Bot Setup (BotFather)

Required settings in BotFather (`/mybots` → your bot → Bot Settings):

| Setting | Value |
|---------|-------|
| Business Mode | **OFF** — must be off for standard webhook bots |
| Threaded Mode | OFF |
| Allow Groups | ON (optional) |
| Group Privacy | ON |

The webhook URL is registered automatically by n8n when you activate the workflow. You never paste the ngrok URL into Telegram or BotFather manually.

---

## Operations

### Starting everything from scratch

1. Start ngrok daemon (auto-starts on boot — usually nothing to do):
   ```bash
   sudo ngrok service start
   ```

2. Start n8n Docker container:
   ```bash
   docker run \
     --user=node \
     --env=NODE_ENV=production \
     --env=N8N_RELEASE_TYPE=stable \
     --env=WEBHOOK_URL=https://discommodious-planular-keenan.ngrok-free.dev \
     --volume=n8n_data:/home/node/.n8n \
     -p 5678:5678 \
     -d docker.n8n.io/n8nio/n8n:latest
   ```

3. Open http://localhost:5678, activate the workflow.

4. Message your Telegram bot to test.

### Checking if it's working

```bash
# Is ngrok tunneling?
curl -s http://127.0.0.1:4040/api/tunnels | python3 -m json.tool

# Is n8n running?
curl -s http://localhost:5678/healthz

# Is the Telegram webhook registered?
curl https://api.telegram.org/bot<TOKEN>/getWebhookInfo
```

### Restarting n8n

```bash
docker ps                      # get container ID
docker stop <container_id>
docker run ...                 # same run command as above
```

### If the bot stops responding

1. Check ngrok is running: `curl -s http://127.0.0.1:4040/api/tunnels`
2. Check Docker is running: `docker ps`
3. Check workflow is Active in n8n UI
4. Re-activate workflow if needed (deactivate → activate) — this re-registers the Telegram webhook

---

## Troubleshooting

| Error | Cause | Fix |
|-------|-------|-----|
| `An HTTPS URL must be provided for webhook` | Telegram requires HTTPS; ngrok not running or WEBHOOK_URL not set | Start ngrok, restart n8n with WEBHOOK_URL env var |
| `Can't listen for test executions at the same time as production ones` | Workflow is active; Execute button blocked | Don't use Execute — message the bot directly |
| `Referenced node doesn't exist` in Window Buffer Memory | Node name mismatch in expression | Update Key expression to match actual Telegram trigger node name |
| Bot not responding | Webhook deregistered (ngrok URL changed or Docker restarted) | Re-activate workflow in n8n UI |

---

## Cost

- **n8n**: Free (self-hosted)
- **ngrok**: Free tier (static subdomain)
- **OpenRouter**: $5/month cap, monthly reset. ~$0.50–$3/month typical usage
- **Google APIs**: Free (personal quota)
- **Telegram**: Free
