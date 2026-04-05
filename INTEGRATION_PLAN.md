# NanoClaw × duck-cli Integration Plan

**Author:** duck-cli sub-agent  
**Date:** 2026-04-05  
**Context:** Study of [qwibitai/nanoclaw](https://github.com/qwibitai/nanoclaw) for potential duck-cli integration  
**Docker Status on this Mac:** ❌ Docker not installed

---

## Executive Summary

NanoClaw is a security-focused, single-process AI assistant built on Claude Agent SDK. Its core innovation is **filesystem isolation via containers** + a **channel polling → SQLite → container → IPC streaming → response** architecture. duck-cli could adopt several NanoClaw patterns for security hardening, multi-channel support, and skills architecture.

---

## 1. NanoClaw Architecture Deep Dive

### 1.1 Core Data Flow

```
Channel (Telegram/Discord/WhatsApp/Slack)
    ↓ poll (POLL_INTERVAL=2000ms)
SQLite (messages.db)
    ↓ getNewMessages() / getMessagesSince()
Message Loop (src/index.ts)
    ↓ GroupQueue (concurrency control, max 5 concurrent)
Container Agent (Docker spawn)
    ↓ stdin JSON (ContainerInput: prompt, sessionId, groupFolder)
    ↓ IPC file polling (/workspace/ipc/input/*.json + _close sentinel)
Agent Runner (container/agent-runner/src/index.ts)
    ↓ Claude Agent SDK (query API, AsyncIterable MessageStream)
    ↓ OUTPUT_START_MARKER / OUTPUT_END_MARKER stdout pairs
    ↓ Streaming parse → channel.sendMessage()
```

### 1.2 Key Components

| File | Purpose |
|------|---------|
| `src/index.ts` | Main entry, channel orchestration, message loop |
| `src/container-runner.ts` | Docker spawn, volume mounts, output streaming |
| `src/agent-runner/src/index.ts` | Runs inside container, SDK query loop, IPC polling |
| `src/db.ts` | SQLite schema (messages, tasks, sessions, groups) |
| `src/group-queue.ts` | Concurrency limiter, retry backoff, IPC message piping |
| `src/channels/registry.ts` | Plugin registry for channels |
| `src/mount-security.ts` | Mount allowlist validation (stored OUTSIDE project root) |
| `src/sender-allowlist.ts` | Per-group sender permission model |
| `src/container-runtime.ts` | Docker lifecycle (start, stop, cleanup orphans) |
| `container/skills/` | Skills synced into each container's `.claude/skills/` |

### 1.3 Credential Security Model (OneCLI Agent Vault)

```
Container (no credentials)
    ↓ HTTPS request (no API key in env)
OneCLI Gateway (host-level, intercepts)
    ↓ injects API key / OAuth token at request time
    ↓ per-agent rate limits + policies
External API (Anthropic, OpenAI, etc.)
```

- `.env` is **shadowed with /dev/null** in container mounts — agent CANNOT read secrets
- OneCLI runs on host, containers route through it
- Allowlists stored at `~/.config/nanoclaw/` (NEVER mounted into containers)

### 1.4 Container Mount Model

**Main group (elevated):**
- `/workspace/project` — READ-ONLY (prevents agent modifying app code)
- `/workspace/project/store` — READ-WRITE (SQLite DB)
- `/workspace/group` — READ-WRITE (group folder)
- `/workspace/global` — READ-WRITE (shared memory)
- `/workspace/ipc` — READ-WRITE (IPC directory)
- `/home/node/.claude` — READ-WRITE (sessions + skills)
- `/workspace/ipc/agent-runner-src` — READ-WRITE (per-group agent-runner copy)

**Non-main groups:**
- `/workspace/group` — READ-WRITE (own folder only)
- `/workspace/global` — READ-ONLY
- `/home/node/.claude` — READ-WRITE (own sessions + skills)
- `/workspace/ipc` — READ-WRITE (own IPC only — prevents cross-group IPC escalation)

### 1.5 IPC Streaming Protocol

The agent-runner uses a **push-based async iterable** (MessageStream) to keep `isSingleUserTurn=false`:
1. stdin carries the initial `ContainerInput` JSON
2. IPC files in `/workspace/ipc/input/` carry follow-up messages
3. `_close` sentinel signals session end
4. Output wrapped in `---NANOCLAW_OUTPUT_START---` / `---NANOCLAW_OUTPUT_END---` pairs
5. Multiple results may be emitted (agent teams produce multiple results)

### 1.6 Skills Architecture

Skills in `container/skills/<skill-name>/SKILL.md` are **copied** into each group's `.claude/skills/` on container startup. This means:
- Each group has isolated skill configuration
- Groups can customize skills without affecting others
- Skills survive container restarts (mounted volumes)

Current NanoClaw skills:
- `agent-browser` — Browser automation
- `capabilities` — Capability reporting
- `slack-formatting` — Slack message formatting
- `status` — Status checking

---

## 2. Integration Opportunities for duck-cli

### 2.1 ✅ ADOPT: Container Isolation for duck-cli Agents

**Could duck-cli agents run in Docker containers?**

**Yes — with modifications.** duck-cli's hybrid orchestrator + tool system could be containerized:

```
duck-cli agent spawn
    ↓ spawn Docker container
    ↓ mount: /workspace/extra/<tool-name>/ → tool implementations
    ↓ mount: /workspace/sessions/<group>/ → session state
Container Agent
    ↓ Claude SDK query loop
    ↓ tool calls via IPC / workspace files
    ↓ OUTPUT markers to stdout
duck-cli host
    ↓ parse streaming output
    ↓ deliver response to Telegram/whatever
```

**Prerequisites:**
- Docker or OrbStack installation on Mac
- Container image with Node.js + duck-cli dependencies
- Per-group session mounting (similar to NanoClaw's `.claude/`)

**Benefits:**
- Malicious tool execution contained
- No accidental filesystem access outside mounted paths
- Agents can't read `.env` or modify host code

**Risks / Trade-offs:**
- **Latency overhead**: Container spawn ~1-2s, vs. in-process ~10ms
- **Complexity**: IPC layer adds failure modes
- **Tool integration**: duck-cli tools (ADB, Grow monitoring, etc.) need container-compatible wrappers
- **macOS**: Docker Desktop adds RAM/CPU overhead on Mac mini

### 2.2 ✅ ADOPT: Channel Polling Pattern

NanoClaw's channel registry is clean and extensible:

```typescript
// Registry pattern
export function registerChannel(name: string, factory: ChannelFactory): void;
export function getChannelFactory(name: string): ChannelFactory | undefined;
export function getRegisteredChannelNames(): string[];
```

Each channel:
1. Implements `Channel` interface (`connect`, `sendMessage`, `isConnected`, `ownsJid`, `disconnect`)
2. Self-registers via barrel import side effect
3. Polls platform API on interval, calls `onMessage` callback

**duck-cli integration:** duck-cli already has channel support via OpenClaw. Adopting this pattern would:
- Enable duck-cli skills to add channels (e.g., a `/add-signal` skill)
- Keep channel code isolated and swappable
- Allow per-channel credential injection via OneCLI-style gateway

### 2.3 ✅ ADOPT: Skills-as-NanoClaw-Style Skills

duck-cli's tools could become NanoClaw-style skills:
- `container/skills/<name>/SKILL.md` + implementation files
- Copied into each container's `.claude/skills/` at startup
- Group-isolated (each group gets own skills copy)

**Current duck-cli tools that would translate well:**
| Tool | → NanoClaw Skill | Notes |
|------|-----------------|-------|
| `grow-*` scripts | `grow-monitor` | Plant monitoring |
| `adb` Android control | `android-control` | ADB wrapper |
| `weather.sh` | `weather` | Open-Meteo integration |
| `ai-council-client.py` | `ai-council` | AI Council integration |
| `subagent-router.sh` | `subagent-orchestrator` | Multi-model routing |

### 2.4 ✅ ADOPT: IPC-Based Streaming (not HTTP polling)

NanoClaw's file-based IPC with `MessageStream` async iterable is elegant:
- No HTTP server needed inside container
- Stateless from host perspective (just writes files)
- Resumable (session ID persisted in SQLite, IPC files on disk)

**duck-cli could use this for:** Agent-to-agent communication, sub-agent coordination.

### 2.5 ⚠️ CONSIDER: OneCLI Agent Vault (Credential Proxy)

OneCLI's credential injection at request time is the gold standard for agent security:
- Agent never holds raw API keys
- Host-level policy enforcement
- Per-agent rate limits

**duck-cli consideration:** duck-cli already uses `.env` files and LM Studio connections. A credential proxy layer would require:
- OneCLI installation / equivalent
- OpenClaw gateway modification
- Rethinking how API keys are passed to sub-agents

**Lower priority** — duck-cli is personal use, lower risk than multi-user scenario.

### 2.6 ⚠️ SKIP: SQLite Message Store (duck-cli uses OpenClaw DB)

NanoClaw's SQLite schema is well-designed but duck-cli already has OpenClaw's message store. No need to duplicate.

### 2.7 ⚠️ SKIP: Group Queue Concurrency Model

NanoClaw's `GroupQueue` with max concurrent containers and backoff retry is excellent for multi-group scenarios. duck-cli's primary use case is single-user → lower priority.

---

## 3. Concrete Integration Plan

### Phase 1: Skills Port (Low Risk, High Value)

**Goal:** Make duck-cli tools available as NanoClaw skills

1. Fork [qwibitai/nanoclaw](https://github.com/qwibitai/nanoclaw) → `Franzferdinan51/nanoclaw`
2. Create skill wrappers in `container/skills/`:
   - `duck-tools/SKILL.md` — documents available duck-cli tools
   - `duck-tools/src/` — wrapper scripts that call duck-cli tools
3. Test skill sync into container's `.claude/skills/`

**Files to create:**
```
container/skills/duck-tools/SKILL.md
container/skills/duck-tools/src/grow-monitor.ts
container/skills/duck-tools/src/android-control.ts
container/skills/duck-tools/src/weather.ts
```

### Phase 2: OrbStack / Docker Integration (Medium Risk)

**Goal:** Enable container isolation for duck-cli

1. **Install OrbStack** (lighter than Docker Desktop for macOS):
   ```bash
   brew install --cask orbstack
   ```
2. **Create duck-cli container image:**
   ```dockerfile
   FROM node:22-alpine
   WORKDIR /app
   COPY package*.json ./
   RUN npm install
   COPY . .
   CMD ["node", "dist/index.js"]
   ```
3. **Modify duck-cli spawn logic:**
   - Check if Docker available
   - If yes: spawn container with tool mounts
   - If no: fall back to in-process execution
4. **IPC bridge:**
   - Write messages to `/workspace/ipc/input/`
   - Parse `OUTPUT_START/END` markers from stdout

### Phase 3: Channel Registry (If Needed)

**Goal:** Enable skill-based channel addition

1. Create `src/channels/registry.ts` (already exists in NanoClaw)
2. Migrate existing OpenClaw channel code to the registry pattern
3. Create skill-based channel installers:

```
# Would work like NanoClaw's /add-telegram
duck-cli /add-telegram  # Runs a skill that configures Telegram channel
```

**Note:** OpenClaw already has robust channel support. This is only valuable if you want NanoClaw's skill-based installation model.

---

## 4. Docker Test Results

```
$ docker --version
zsh: command not found: docker
```

**Docker is NOT installed** on this Mac. Options:
1. **OrbStack** — `brew install --cask orbstack` (recommended, ~10x faster than Docker Desktop)
2. **Docker Desktop** — `brew install --cask docker` (heavier but more compatible)

NanoClaw on macOS uses **Apple Container** (not Docker) for native macOS isolation. This is an alternative worth exploring.

---

## 5. Key Files Reference

| File | Lines | Purpose |
|------|-------|---------|
| `src/index.ts` | ~450 | Main orchestrator, message loop, group registration |
| `src/container-runner.ts` | ~400 | Docker spawn, mount building, output streaming |
| `container/agent-runner/src/index.ts` | ~450 | Runs inside container, SDK query + IPC polling |
| `src/group-queue.ts` | ~300 | Concurrency, retry backoff, IPC message routing |
| `src/db.ts` | ~500 | SQLite schema, migrations, all state access |
| `src/channels/registry.ts` | ~50 | Channel plugin registry |
| `src/mount-security.ts` | ~150 | Mount allowlist validation |
| `src/config.ts` | ~100 | All configuration, paths, timeouts |

**Total core source:** ~2,300 lines TypeScript across ~20 files  
(OpenClaw is ~500,000 lines — NanoClaw is ~0.5% of that size)

---

## 6. Immediate Next Steps

1. **[P0] Install OrbStack** — enables NanoClaw to run on this Mac:
   ```bash
   brew install --cask orbstack
   ```
2. **[P0] Test NanoClaw container locally** — verify Apple Container / Docker works
3. **[P1] Fork NanoClaw** → `Franzferdinan51/nanoclaw`
4. **[P1] Create duck-tools skill** in `container/skills/duck-tools/`
5. **[P2] Build duck-cli container image** (`Dockerfile` + `entrypoint.sh`)
6. **[P2] Write IPC bridge** between duck-cli and NanoClaw agent-runner
7. **[P3] Integrate OneCLI** for credential security (if multi-user scenario develops)

---

## 7. What Makes NanoClaw Special (Key Innovations)

1. **Shadow `.env` with `/dev/null`** — agent literally cannot read secrets
2. **Per-group IPC namespace** — prevents cross-group privilege escalation
3. **MessageStream async iterable** — keeps `isSingleUserTurn=false` for multi-turn agent teams
4. **Mount allowlist outside project root** (`~/.config/nanoclaw/`) — tamper-proof from containers
5. **Skills copied per-group** — groups can customize without affecting main
6. **Startup message recovery** — handles crash between advancing cursor and processing
7. **`_close` sentinel** — graceful container shutdown without SIGKILL

---

*End of integration plan. NanoClaw is an excellent reference architecture — highly recommended to study its source before making changes to either project.*
