# n8n Personal Assistant — Git Repo Bootstrap Plan

> **For agentic workers:** REQUIRED: Use superpowers:subagent-driven-development (if subagents available) or superpowers:executing-plans to implement this plan. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Turn the n8n-personal-assistant folder into a proper git repo with workflow backups, infrastructure-as-code, and a working restore runbook so the entire setup can be rebuilt from scratch on a new machine.

**Architecture:** Three layers — (1) git repo for all config/docs/workflow JSON, (2) an exported n8n workflow JSON that can be re-imported, (3) a `.env.example` template so secrets are documented but never committed.

**Tech Stack:** git, n8n REST API (workflow export), Docker, ngrok, bash

---

## Chunk 1: Repo scaffolding and git init

### Task 1: Initialize the git repo

**Files:**
- Create: `.gitignore`
- Create: `.env.example`
- Modify: (existing folder becomes a git repo)

- [ ] **Step 1: Confirm you are in the right directory and pre-flight checks pass**

```bash
pwd
# Expected: /path/to/n8n-personal-assistant

# Verify the three required base files exist
for file in README.md DECISIONS.md SESSIONS.md; do
  [ -f "$file" ] || (echo "ERROR: Missing $file — are you in the right directory?" && exit 1)
done
echo "Pre-flight OK: all base files present"
```

- [ ] **Step 2: Initialize git**

```bash
git init
git branch -M main
```

Expected output: `Initialized empty Git repository in .../n8n-personal-assistant/.git/`

- [ ] **Step 3: Create `.gitignore`**

```bash
cat > .gitignore << 'EOF'
# Secrets — never commit these
.env
*.env
*.key
*.pem
credentials.json

# n8n local dev artifacts
.n8n/

# macOS
.DS_Store

# Editor
.idea/
.vscode/
EOF
```

- [ ] **Step 4: Create `.env.example`** — documents what env vars are needed, without actual values

```bash
cat > .env.example << 'EOF'
# n8n Docker environment variables
# Copy to .env and fill in values (never commit .env)

# The public HTTPS URL ngrok exposes (used as Telegram webhook base)
WEBHOOK_URL=https://your-ngrok-subdomain.ngrok-free.app

# n8n release channel
NODE_ENV=production
N8N_RELEASE_TYPE=stable

# n8n API key for workflow export script (create at localhost:5678 → Settings → API)
N8N_API_KEY=your-n8n-api-key-here

# Workflow ID for export (find in n8n UI URL: /workflow/123 → WORKFLOW_ID=123)
WORKFLOW_ID=1
EOF
```

- [ ] **Step 5: Stage and commit scaffolding (including plan docs)**

```bash
git add .gitignore .env.example README.md DECISIONS.md SESSIONS.md docs/
git status
# Verify: no .env or credential files in staging area
# Verify: docs/superpowers/plans/2026-03-16-git-repo-bootstrap.md IS listed
git commit -m "chore: init repo with infrastructure docs, env template, and plan"
```

> **If `git commit` fails:** Run `git status` to diagnose. Common causes: nothing staged (`git add` failed silently), or identity not configured (`git config user.email "you@example.com" && git config user.name "Your Name"`). Do not proceed to the next task until this commit succeeds.

---

## Chunk 2: Workflow export

### Task 2: Export the n8n workflow to JSON

**Files:**
- Create: `workflows/personal-life-manager.json`
- Create: `workflows/README.md`

**Context:** n8n workflows live inside the Docker volume `n8n_data`. They can be exported via the n8n UI or via the n8n REST API. The API approach is scriptable and repeatable. n8n's local API is at `http://localhost:5678/api/v1/`.

- [ ] **Step 1: Verify n8n is running and a workflow exists**

```bash
curl -s http://localhost:5678/healthz
```

Expected output: `{"status":"ok"}`

If not: start the Docker container using the run command in README.md, wait ~10 seconds, and retry.

```bash
# Also confirm Docker sees an n8n container
docker ps --filter "ancestor=docker.n8n.io/n8nio/n8n:latest" --format "{{.Status}}"
# Expected: Up X minutes
```

- [ ] **Step 2: Get your n8n API key**

Open http://localhost:5678 → Settings → API → Create an API Key.
Copy the key — you'll use it in the next step. Store it temporarily in your shell session only:

```bash
export N8N_API_KEY="your-key-here"
```

- [ ] **Step 3: List workflows to find the ID**

```bash
curl -s -H "X-N8N-API-KEY: $N8N_API_KEY" \
  http://localhost:5678/api/v1/workflows | python3 -m json.tool
```

Expected: JSON array of workflows. Find the "Personal life manager" workflow and note its `id` field (e.g. `"id": "1"`).

- [ ] **Step 4: Export the workflow**

```bash
mkdir -p workflows

export WORKFLOW_ID="1"   # replace with actual ID from step above

curl -s -H "X-N8N-API-KEY: $N8N_API_KEY" \
  "http://localhost:5678/api/v1/workflows/$WORKFLOW_ID" \
  | python3 -m json.tool > workflows/personal-life-manager.json

wc -l workflows/personal-life-manager.json
# Expected: non-zero line count
head -5 workflows/personal-life-manager.json
# Expected: JSON with "name", "nodes" fields
```

- [ ] **Step 5: Verify export looks sane**

```bash
python3 -c "
import json
with open('workflows/personal-life-manager.json') as f:
    w = json.load(f)
print('Workflow name:', w.get('name'))
print('Node count:', len(w.get('nodes', [])))
print('Node names:', [n['name'] for n in w.get('nodes', [])])
"
```

Expected: name = "Personal life manager with Telegram...", node count ≥ 8, node names include "Telegram Trigger", "AI Agent", etc.

- [ ] **Step 6: Create `workflows/README.md`**

```bash
cat > workflows/README.md << 'EOF'
# Workflow Exports

## personal-life-manager.json

Export of the active n8n workflow: "Personal life manager with Telegram, Google services & voice-enabled AI"

### To re-import after a fresh n8n install:

1. Open http://localhost:5678
2. Go to Workflows → Import from File
3. Select `personal-life-manager.json`
4. Reconnect all credentials (Telegram, OpenRouter, Gmail, Google Calendar, Google Tasks)
5. Fix the Window Buffer Memory node expression if needed (see README.md Troubleshooting)
6. Activate the workflow

### To update this export (after making changes in the n8n UI):

```bash
export N8N_API_KEY="your-key"
export WORKFLOW_ID="1"
curl -s -H "X-N8N-API-KEY: $N8N_API_KEY" \
  "http://localhost:5678/api/v1/workflows/$WORKFLOW_ID" \
  | python3 -m json.tool > workflows/personal-life-manager.json
git add workflows/personal-life-manager.json
git commit -m "chore: update workflow export"
```
EOF
```

- [ ] **Step 7: Stage and commit workflow export**

```bash
git add workflows/
git status
# Verify: only workflows/ files being added, no credentials
git commit -m "feat: add exported n8n workflow JSON for backup and restore"
```

---

## Chunk 3: Infrastructure scripts

### Task 3: Add Docker run script

**Files:**
- Create: `scripts/start-n8n.sh`

The docker run command is documented in README.md but having it as an executable script prevents typos and makes it one-command to restart.

- [ ] **Step 1: Create scripts directory and start script**

```bash
mkdir -p scripts

cat > scripts/start-n8n.sh << 'EOF'
#!/usr/bin/env bash
# Start n8n Docker container.
# Requires: WEBHOOK_URL set in environment or .env file

set -euo pipefail

# Load .env if present
if [ -f "$(dirname "$0")/../.env" ]; then
  # shellcheck disable=SC1091
  source "$(dirname "$0")/../.env"
fi

if [ -z "${WEBHOOK_URL:-}" ]; then
  echo "ERROR: WEBHOOK_URL is not set. Set it in .env or export it."
  exit 1
fi

echo "Starting n8n with WEBHOOK_URL=$WEBHOOK_URL"

docker run \
  --user=node \
  --env=NODE_ENV=production \
  --env=N8N_RELEASE_TYPE=stable \
  --env="WEBHOOK_URL=$WEBHOOK_URL" \
  --volume=n8n_data:/home/node/.n8n \
  -p 5678:5678 \
  -d docker.n8n.io/n8nio/n8n:latest

echo "n8n started. Open http://localhost:5678"
EOF

chmod +x scripts/start-n8n.sh
```

- [ ] **Step 2: Test the script dry-run (don't actually start a second container)**

```bash
# Just verify the script is executable and has no syntax errors
bash -n scripts/start-n8n.sh && echo "Syntax OK"
```

Expected: `Syntax OK`

- [ ] **Step 3: Add ngrok verification helper**

```bash
cat > scripts/check-status.sh << 'EOF'
#!/usr/bin/env bash
# Check that all n8n infrastructure is running.
set -euo pipefail

echo "=== ngrok tunnel ==="
curl -s http://127.0.0.1:4040/api/tunnels | python3 -m json.tool 2>/dev/null \
  || echo "FAIL: ngrok not running (port 4040 not responding)"

echo ""
echo "=== n8n health ==="
curl -s http://localhost:5678/healthz \
  || echo "FAIL: n8n not running (port 5678 not responding)"

echo ""
echo "=== Docker containers ==="
docker ps --filter "ancestor=docker.n8n.io/n8nio/n8n:latest" \
  --format "table {{.ID}}\t{{.Status}}\t{{.Ports}}"
EOF

chmod +x scripts/check-status.sh
bash -n scripts/check-status.sh && echo "Syntax OK"
```

- [ ] **Step 4: Stage and commit scripts**

```bash
git add scripts/
git commit -m "feat: add start-n8n.sh and check-status.sh helper scripts"
```

---

## Chunk 4: ngrok config backup

### Task 4: Capture ngrok config (redacted)

**Files:**
- Create: `ngrok/ngrok.yml.example`
- Create: `ngrok/README.md`

The actual `ngrok.yml` lives at `~/Library/Application Support/ngrok/ngrok.yml` and contains an auth token (secret). We capture the structure without the secret.

- [ ] **Step 1: Inspect current ngrok config structure**

```bash
cat ~/Library/Application\ Support/ngrok/ngrok.yml
```

Note the structure — specifically the `authtoken` field and `tunnels` section.

- [ ] **Step 2: Create redacted template**

```bash
mkdir -p ngrok

cat > ngrok/ngrok.yml.example << 'EOF'
# ngrok config template
# Real file location: ~/Library/Application Support/ngrok/ngrok.yml
# Copy this to that location and fill in your authtoken.
# Then run: sudo ngrok service install

version: "2"
authtoken: YOUR_NGROK_AUTHTOKEN_HERE

tunnels:
  n8n:
    proto: http
    addr: 5678
    # subdomain only works on paid plans; free users get a random URL
    # unless they have a static domain configured in the ngrok dashboard
EOF
```

- [ ] **Step 3: Create `ngrok/README.md`**

```bash
cat > ngrok/README.md << 'EOF'
# ngrok Setup

ngrok provides the HTTPS tunnel Telegram requires for webhooks.

## Install as macOS system daemon (survives reboots)

1. Install ngrok: https://ngrok.com/download
2. Authenticate: `ngrok config add-authtoken YOUR_TOKEN`
3. Copy `ngrok.yml.example` to `~/Library/Application Support/ngrok/ngrok.yml` and fill in your authtoken
4. Install daemon: `sudo ngrok service install`
5. Start daemon: `sudo ngrok service start`

## Verify tunnel is live

```bash
curl -s http://127.0.0.1:4040/api/tunnels | python3 -m json.tool
```

## Important notes

- Valid service subcommands: install / start / stop / restart / uninstall
- There is NO `ngrok service status` subcommand — use the curl above
- Tunnels must be defined in ngrok.yml BEFORE running `sudo ngrok service install`
- The free static subdomain (e.g. `discommodious-planular-keenan.ngrok-free.dev`) is tied to your ngrok account
EOF
```

- [ ] **Step 4: Stage and commit**

```bash
git add ngrok/
git commit -m "docs: add ngrok config template and setup guide"
```

---

## Chunk 5: GitHub remote setup

### Task 5: Create and push to GitHub

**Files:** none new — just remote setup

- [ ] **Step 1: Verify gh CLI is installed and authenticated**

```bash
gh --version
# Expected: gh version X.Y.Z (...)

gh auth status
# Expected: Logged in to github.com as <your-username>
```

If `gh` is not installed: `brew install gh` then `gh auth login`.

- [ ] **Step 2: Create the GitHub repo (via gh CLI)**

```bash
gh repo create n8n-personal-assistant \
  --private \
  --description "Self-hosted n8n personal life manager bot — Telegram → AI → Gmail/Calendar/Tasks" \
  --source=. \
  --remote=origin \
  --push
```

Expected: repo created, all commits pushed, output includes repo URL.

If `gh` is not available: create the repo manually at https://github.com/new (set Private), then:

```bash
git remote add origin git@github.com:YOUR_USERNAME/n8n-personal-assistant.git
git push -u origin main
```

- [ ] **Step 3: Verify push succeeded**

```bash
git log --oneline
# Expected: 5+ commits visible

gh repo view --web
# Opens the repo in browser — verify all directories and files are present
```

- [ ] **Step 4: Add GitHub repo URL to README.md**

Open `README.md` and add a one-liner at the top linking to the GitHub repo.

```bash
# Prepend the repo badge line
REPO_URL=$(gh repo view --json url -q .url)
echo "Repo: $REPO_URL"
# Add this line manually to the top of README.md under the h1
```

```bash
git add README.md
git commit -m "docs: add GitHub repo link to README"
git push
```

---

## Chunk 6: Export automation script

### Task 6: Add a workflow backup script

**Files:**
- Create: `scripts/export-workflow.sh`

This makes it one command to refresh the workflow JSON backup. Run it whenever you make changes in the n8n UI.

- [ ] **Step 1: Create export script**

```bash
cat > scripts/export-workflow.sh << 'EOF'
#!/usr/bin/env bash
# Export the active n8n workflow to workflows/personal-life-manager.json
# Usage: ./scripts/export-workflow.sh
#   Reads N8N_API_KEY and WORKFLOW_ID from .env (or from environment)

set -euo pipefail

SCRIPT_DIR="$(cd "$(dirname "$0")" && pwd)"
REPO_ROOT="$SCRIPT_DIR/.."

# Load .env if present
if [ -f "$REPO_ROOT/.env" ]; then
  # shellcheck disable=SC1091
  source "$REPO_ROOT/.env"
fi

N8N_API_KEY="${N8N_API_KEY:?Set N8N_API_KEY in .env or environment (create at localhost:5678 → Settings → API)}"
WORKFLOW_ID="${WORKFLOW_ID:?Set WORKFLOW_ID in .env or environment (find it in the n8n UI URL: /workflow/123)}"
OUTPUT="$(dirname "$0")/../workflows/personal-life-manager.json"

echo "Exporting workflow $WORKFLOW_ID from n8n..."

curl -sf -H "X-N8N-API-KEY: $N8N_API_KEY" \
  "http://localhost:5678/api/v1/workflows/$WORKFLOW_ID" \
  | python3 -m json.tool > "$OUTPUT"

echo "Saved to $OUTPUT"
echo "Node count: $(python3 -c "import json; d=json.load(open('$OUTPUT')); print(len(d['nodes']))")"

echo ""
echo "To commit this update:"
echo "  git add workflows/personal-life-manager.json"
echo "  git commit -m 'chore: update workflow export'"
echo "  git push"
EOF

chmod +x scripts/export-workflow.sh
bash -n scripts/export-workflow.sh && echo "Syntax OK"
```

- [ ] **Step 2: Stage and commit**

```bash
git add scripts/export-workflow.sh
git commit -m "feat: add export-workflow.sh for repeatable workflow backups"
git push
```

---

## Chunk 7: Disaster recovery runbook

### Task 7: Write RESTORE.md

**Files:**
- Create: `RESTORE.md`

This is the single most important file for recovery. A developer with only this repo + their ngrok account + their n8n credential values should be able to fully rebuild the system.

- [ ] **Step 1: Create RESTORE.md**

```bash
cat > RESTORE.md << 'EOF'
# Full Restore Runbook — n8n Personal Life Manager

Use this when rebuilding from scratch (new Mac, corrupted Docker volume, etc.).
Estimated time: ~15 minutes if you have all credentials ready.

---

## Prerequisites

Before starting, have these ready:
- [ ] ngrok account + authtoken (https://dashboard.ngrok.com/authtokens)
- [ ] Telegram bot token (from BotFather — `/mybots`)
- [ ] OpenRouter API key (https://openrouter.ai/keys)
- [ ] Google account (milesdchu@gmail.com) for OAuth (Gmail, Calendar, Tasks)
- [ ] Docker installed and running
- [ ] This repo cloned: `git clone git@github.com:YOUR_USERNAME/n8n-personal-assistant.git`

---

## Step 1: Set up .env

```bash
cd n8n-personal-assistant
cp .env.example .env
# Edit .env and fill in WEBHOOK_URL with your ngrok static domain
```

---

## Step 2: Set up ngrok

```bash
# Install ngrok if needed
brew install ngrok/ngrok/ngrok

# Authenticate
ngrok config add-authtoken YOUR_AUTHTOKEN

# Copy ngrok config template and fill in authtoken
cp ngrok/ngrok.yml.example ~/Library/Application\ Support/ngrok/ngrok.yml
# Edit the file: replace YOUR_NGROK_AUTHTOKEN_HERE

# Install as system daemon (auto-starts on boot)
sudo ngrok service install
sudo ngrok service start

# Verify tunnel is live
curl -s http://127.0.0.1:4040/api/tunnels | python3 -m json.tool
# Expected: tunnel URL matches your WEBHOOK_URL in .env
```

---

## Step 3: Start n8n

```bash
./scripts/start-n8n.sh
# Wait ~10 seconds, then verify:
curl -s http://localhost:5678/healthz
# Expected: {"status":"ok"}
```

Open http://localhost:5678 and complete the first-time setup (create an owner account).

---

## Step 4: Import the workflow

1. In n8n UI: Workflows → **Import from File**
2. Select `workflows/personal-life-manager.json`
3. The workflow appears but credentials will be disconnected (red nodes) — this is expected

---

## Step 5: Reconnect credentials

In n8n: Settings → Credentials → New, for each:

| Credential | Type | Notes |
|-----------|------|-------|
| Telegram | Telegram API | Paste bot token from BotFather |
| OpenRouter | HTTP Header Auth | Header: `Authorization`, Value: `Bearer YOUR_KEY` |
| Gmail | Google OAuth2 | Sign in with milesdchu@gmail.com |
| Google Calendar | Google OAuth2 | Reuse Gmail credential or create new |
| Google Tasks | Google OAuth2 | Reuse Gmail credential or create new |

In each red node, select the matching credential from the dropdown.

---

## Step 6: Fix Window Buffer Memory node (if broken)

If the Window Buffer Memory node shows a reference error:
1. Click the node → find the Session Key expression
2. Update to reference the actual Telegram trigger node name exactly
3. Common fix: change `$('Listen for incoming events')` to match the trigger node's display name

---

## Step 7: Configure Google Calendar node

1. Open the Google Calendar tool node
2. Set Calendar: `milesdchu@gmail.com`

---

## Step 8: Set OpenRouter credit cap

1. Go to https://openrouter.ai → Settings → Limits
2. Set monthly credit limit: $5

---

## Step 9: Telegram BotFather settings

1. `/mybots` → select your bot → Bot Settings
2. **Business Mode: OFF** (must be off for standard webhook bots)

---

## Step 10: Activate and test

1. In n8n UI: toggle workflow to **Active** (top-right)
2. Send a message to your Telegram bot
3. Expected: AI response within a few seconds

---

## Verification checklist

```bash
./scripts/check-status.sh
```

- [ ] ngrok tunnel live at expected URL
- [ ] n8n health: `{"status":"ok"}`
- [ ] Docker container running
- [ ] Workflow shows Active in n8n UI
- [ ] Telegram bot replies to a test message

---

## Troubleshooting

| Symptom | Cause | Fix |
|---------|-------|-----|
| Bot not responding | Workflow not Active | Toggle Active in n8n UI |
| Webhook error on activation | ngrok not running | `sudo ngrok service start`, re-activate workflow |
| Credential auth errors | OAuth expired | Re-authenticate in Settings → Credentials |
| Window Buffer Memory error | Node name mismatch | See Step 6 above |
| n8n won't start | Port 5678 in use | `docker ps` to find conflicting container, stop it |
EOF
```

- [ ] **Step 2: Stage and commit**

```bash
git add RESTORE.md
git commit -m "docs: add full disaster recovery runbook"
git push
```

---

## Chunk 8: Final validation

### Task 8: Verify the complete repo state

**Files:** none new — verification only

- [ ] **Step 1: Verify repo structure matches expected Final State**

```bash
git status
# Expected: nothing to commit, working tree clean

git log --oneline
# Expected: 7+ commits

git remote -v
# Expected: origin pointing to github.com/YOUR_USERNAME/n8n-personal-assistant
```

- [ ] **Step 2: Verify all key files exist**

```bash
for f in .gitignore .env.example README.md DECISIONS.md SESSIONS.md RESTORE.md \
          workflows/personal-life-manager.json workflows/README.md \
          scripts/start-n8n.sh scripts/check-status.sh scripts/export-workflow.sh \
          ngrok/ngrok.yml.example ngrok/README.md \
          docs/superpowers/plans/2026-03-16-git-repo-bootstrap.md; do
  [ -f "$f" ] && echo "✅ $f" || echo "❌ MISSING: $f"
done
```

Expected: all lines show ✅

- [ ] **Step 3: Verify .env is protected by .gitignore**

```bash
# Create a test .env and confirm git ignores it
echo "TEST=1" > .env
git status
# Expected: .env does NOT appear in "Untracked files"
rm .env
```

- [ ] **Step 4: Verify workflow JSON is valid**

```bash
python3 -c "
import json
with open('workflows/personal-life-manager.json') as f:
    w = json.load(f)
assert 'nodes' in w, 'Missing nodes key'
assert len(w['nodes']) >= 8, f'Only {len(w[\"nodes\"])} nodes — expected >= 8'
print('✅ Workflow JSON valid')
print('   Name:', w.get('name'))
print('   Nodes:', len(w['nodes']))
"
```

- [ ] **Step 5: Confirm GitHub repo is accessible**

```bash
gh repo view
# Expected: repo name, description, visibility: private
```

---

## Final State

After completing all chunks, the repo should contain:

```
n8n-personal-assistant/
├── .gitignore
├── .env.example
├── README.md
├── DECISIONS.md
├── SESSIONS.md
├── docs/
│   └── superpowers/
│       └── plans/
│           └── 2026-03-16-git-repo-bootstrap.md
├── ngrok/
│   ├── README.md
│   └── ngrok.yml.example
├── scripts/
│   ├── check-status.sh
│   ├── export-workflow.sh
│   └── start-n8n.sh
└── workflows/
    ├── README.md
    └── personal-life-manager.json
```

**Recovery guarantee:** From this repo alone + ngrok account + n8n credential values, a complete rebuild takes ~15 minutes.
