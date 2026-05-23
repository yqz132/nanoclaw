# NanoClaw

Personal Claude assistant. See [README.md](README.md) for philosophy and setup. See [docs/REQUIREMENTS.md](docs/REQUIREMENTS.md) for architecture decisions.

## Quick Context

One Node.js process per agent instance, with skill-based channel system. Channels (WhatsApp, Telegram, Slack, Discord, Gmail) are skills that self-register at startup. Messages route to Claude Agent SDK running in containers (Linux VMs). Each group has isolated filesystem and memory. Multiple agents can share the same codebase and `groups/` directory for cross-agent collaboration.

## Key Files

| File | Purpose |
|------|---------|
| `src/index.ts` | Orchestrator: state, message loop, agent invocation |
| `src/channels/registry.ts` | Channel registry (self-registration at startup) |
| `src/ipc.ts` | IPC watcher and task processing |
| `src/router.ts` | Message formatting and outbound routing |
| `src/config.ts` | Trigger pattern, paths, intervals |
| `src/container-runner.ts` | Spawns agent containers with mounts |
| `src/task-scheduler.ts` | Runs scheduled tasks |
| `src/db.ts` | SQLite operations |
| `agents/{name}/data/sessions/{folder}/.claude/` | Per-group Claude session dir (CLAUDE.md, settings.json, skills) |
| `container/skills/` | Skills loaded inside agent containers (browser, status, formatting) |

## Secrets / Credentials / Proxy (OneCLI)

API keys, secret keys, OAuth tokens, and auth credentials are managed by the OneCLI gateway — which handles secret injection into containers at request time, so no keys or tokens are ever passed to containers directly. Run `onecli --help`.

## Skills

Four types of skills exist in NanoClaw. See [CONTRIBUTING.md](CONTRIBUTING.md) for the full taxonomy and guidelines.

- **Feature skills** — merge a `skill/*` branch to add capabilities (e.g. `/add-telegram`, `/add-slack`)
- **Utility skills** — ship code files alongside SKILL.md (e.g. `/claw`)
- **Operational skills** — instruction-only workflows, always on `main` (e.g. `/setup`, `/debug`)
- **Container skills** — loaded inside agent containers at runtime (`container/skills/`)

| Skill | When to Use |
|-------|-------------|
| `/setup` | First-time installation, authentication, service configuration |
| `/customize` | Adding channels, integrations, changing behavior |
| `/debug` | Container issues, logs, troubleshooting |
| `/update-nanoclaw` | Bring upstream NanoClaw updates into a customized install |
| `/init-onecli` | Install OneCLI Agent Vault and migrate `.env` credentials to it |
| `/qodo-pr-resolver` | Fetch and fix Qodo PR review issues interactively or in batch |
| `/get-qodo-rules` | Load org- and repo-level coding rules from Qodo before code tasks |

## Contributing

Before creating a PR, adding a skill, or preparing any contribution, you MUST read [CONTRIBUTING.md](CONTRIBUTING.md). It covers accepted change types, the four skill types and their guidelines, SKILL.md format rules, PR requirements, and the pre-submission checklist (searching for existing PRs/issues, testing, description format).

## Multi-Agent Architecture

NanoClaw supports multiple agent instances running as separate processes, each with its own identity, database, and Discord bot token. All agents share the same codebase and groups directory.

### Directory Layout

```
nanoclaw/
  agents/              # Per-agent runtime data (gitignored)
    avery/
      .env             # ASSISTANT_NAME, DISCORD_BOT_TOKEN, path overrides
      data/            # sessions, ipc
      store/           # messages.db
    jamie/
      .env, data/, store/
    sage/
      .env, data/, store/
  logs/                # Agent process logs
    avery.log
    jamie.log
    sage.log
  groups/              # Shared group folders (gitignored except CLAUDE.md)
    avery/             # Agent-specific group
    jamie/             # Agent-specific group
    sage/              # Agent-specific group
    discord_project-quant/  # Shared project group
    global/            # Shared context across agents
```

### Key Paths

Each agent's `.env` overrides these defaults:
- `STORE_DIR` → `agents/{name}/store` (SQLite database)
- `DATA_DIR` → `agents/{name}/data` (sessions, ipc)
- `GROUPS_DIR` → `groups/` (shared across all agents)

### Agent Management

All agents run in a single tmux session `nanoclaw` with named windows:
```bash
tmux new-session -d -s nanoclaw                                    # Create session
tmux new-window -t nanoclaw -n avery -c ~/nanoclaw/agents/avery    # Add agent window
tmux send-keys -t nanoclaw:avery 'node ~/nanoclaw/dist/index.js 2>&1 | tee ~/nanoclaw/logs/avery.log' Enter
```

View logs: `tail -20 ~/nanoclaw/logs/avery.log`
Attach: `tmux attach -t nanoclaw`

### Creating a New Agent

1. Create directory and .env:
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

2. Create agent-specific group folder:
   ```bash
   mkdir -p ~/nanoclaw/groups/{name}/logs
   ```

3. Start the agent in tmux:
   ```bash
   tmux new-window -t nanoclaw -n {name} -c ~/nanoclaw/agents/{name}
   tmux send-keys -t nanoclaw:{name} 'node ~/nanoclaw/dist/index.js 2>&1 | tee ~/nanoclaw/logs/{name}.log' Enter
   ```

4. Register the agent's main Discord channel by sending `@Name /register` in that channel.

### Adding a Shared Project Group

1. Create the group folder:
   ```bash
   mkdir -p ~/nanoclaw/groups/{project-name}/logs
   ```
   Optionally add a `CLAUDE.md` with project context.

2. Each agent that should participate: register the Discord channel by sending `@AgentName /register` in that channel. The group folder is auto-mounted as a sibling in other agents' containers.

### Cross-Agent @Mention

Bots can @mention each other in shared groups. Each agent's own messages are filtered out via `is_from_me` in the DB query, so self-trigger loops are prevented. Other bots' messages have `is_from_me=false` and pass through, allowing inter-bot collaboration.

### Multi-Agent Implementation Decisions

These are intentional divergences from the single-agent defaults — don't revert them:

| Decision | What | Why |
|----------|------|-----|
| `is_from_me = 0` filter in DB queries | Replaces `is_bot_message = 0` | Other bots' messages must be visible for cross-agent @mention to work; only self-replies should be suppressed |
| `ASSISTANT_NAME.toLowerCase()` as OneCLI agent identifier | Replaces `group.folder` | One agent can host sessions for multiple groups; credential config is per-agent, not per-group |
| `settings.json` inherits from main group's env | New groups copy API endpoint, model, etc. from `agents/{name}/data/sessions/{name}/.claude/settings.json` | Avoids having to set model/endpoint per-group; delete a group's settings.json to force re-inherit |
| Store mounted at `/workspace/store` | Not `/workspace/project/store` | Docker can't create a mountpoint inside a read-only bind-mount; nested mount requires the subdir to exist in the parent source |
| `projectRoot` via `import.meta.url` in container-runner | Not `process.cwd()` | `cwd` is the agent's data dir when running multi-instance; `import.meta.url` always points to the compiled JS location |

## Development

Run commands directly—don't tell the user to run them.

```bash
npm run dev          # Run with hot reload
npm run build        # Compile TypeScript
./container/build.sh # Rebuild agent container
```

Service management:
```bash
# macOS (launchd)
launchctl load ~/Library/LaunchAgents/com.nanoclaw.plist
launchctl unload ~/Library/LaunchAgents/com.nanoclaw.plist
launchctl kickstart -k gui/$(id -u)/com.nanoclaw  # restart

# Linux (systemd)
systemctl --user start nanoclaw
systemctl --user stop nanoclaw
systemctl --user restart nanoclaw
```

## Troubleshooting

**WhatsApp not connecting after upgrade:** WhatsApp is now a separate skill, not bundled in core. Run `/add-whatsapp` (or `npx tsx scripts/apply-skill.ts .claude/skills/add-whatsapp && npm run build`) to install it. Existing auth credentials and groups are preserved.

## Container Build Cache

The container buildkit caches the build context aggressively. `--no-cache` alone does NOT invalidate COPY steps — the builder's volume retains stale files. To force a truly clean rebuild, prune the builder then re-run `./container/build.sh`.
