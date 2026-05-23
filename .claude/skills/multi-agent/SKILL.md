---
name: multi-agent
description: Run multiple NanoClaw agents from the same codebase. Each agent has its own identity, Discord bot token, database, and sessions, but shares the codebase and groups/ directory for cross-agent collaboration.
---

# Multi-Agent NanoClaw

This skill enables running multiple independent agent instances (e.g. Avery, Jamie, Sage) from a single NanoClaw installation. Agents can participate in shared project groups and @mention each other.

## Phase 1: Apply Code Changes

### Ensure remote

```bash
git remote -v
```

If `multi-agent` remote is missing, add it (replace with your fork URL):

```bash
git remote add multi-agent <your-fork-url>
```

### Merge the skill branch

```bash
git fetch multi-agent
git merge multi-agent/skill/multi-agent --no-edit
npm run build
```

## Phase 2: Create a New Agent

### 1. Create directories and .env

```bash
mkdir -p ~/nanoclaw/agents/{name}/{data,store}
```

Create `~/nanoclaw/agents/{name}/.env`:

```
TZ=Asia/Shanghai
ONECLI_URL=http://127.0.0.1:10254
DISCORD_BOT_TOKEN=<bot_token>
ASSISTANT_NAME=<Name>
STORE_DIR=/home/nanoclaw/nanoclaw/agents/{name}/store
DATA_DIR=/home/nanoclaw/nanoclaw/agents/{name}/data
GROUPS_DIR=/home/nanoclaw/nanoclaw/groups
```

### 2. Create the agent's group folder

```bash
mkdir -p ~/nanoclaw/groups/{name}/logs
```

### 3. Configure the model (optional)

Edit `~/nanoclaw/agents/{name}/data/sessions/{name}/.claude/settings.json` with API endpoint and model. Other groups this agent hosts will inherit these settings automatically when first created.

### 4. Start in tmux

```bash
tmux new-window -t nanoclaw -n {name} -c ~/nanoclaw/agents/{name}
tmux send-keys -t nanoclaw:{name} 'node ~/nanoclaw/dist/index.js 2>&1 | tee ~/nanoclaw/logs/{name}.log' Enter
```

### 5. Register the main channel

In the agent's Discord channel, send: `@Name /register`

## Phase 3: Add a Shared Project Group

Agents can collaborate in shared channels where they @mention each other.

### 1. Create the group folder

```bash
mkdir -p ~/nanoclaw/groups/{project-name}/logs
```

Optionally add a `CLAUDE.md` with project context.

### 2. Register each participating agent

In the project Discord channel, send `@AgentName /register` for each agent that should participate.

## How it Works

- Each agent is a separate Node.js process with its own SQLite DB and sessions
- `STORE_DIR`, `DATA_DIR`, `GROUPS_DIR` are configured per-agent via `.env`
- The process reads `.env` from `process.cwd()`, which is the agent's directory when launched from tmux
- Bot messages from other agents pass through (`is_from_me = 0` filter); only self-replies are suppressed
- OneCLI credential lookup uses `ASSISTANT_NAME`, not the group folder name
- Container store is mounted at `/workspace/store` (not nested under the read-only `/workspace/project`)
- New group sessions inherit `settings.json` env from the agent's main group session

See the **Multi-Agent Architecture** section in `CLAUDE.md` for the full directory layout and design decisions.

## Logs and Management

```bash
tail -f ~/nanoclaw/logs/{name}.log   # follow agent log
tmux attach -t nanoclaw              # attach to all agent windows
```
