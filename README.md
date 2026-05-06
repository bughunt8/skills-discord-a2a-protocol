# DA2A — Discord Agent-to-Agent Communication Protocol

> **Version 1.0.0** | IDL + JSON-RPC 2.0 | Transport: Discord Gateway

A rigorous, efficient agent-to-agent (A2A) communication protocol for multi-agent systems operating via Discord channels. DA2A combines **Interface Description Language (IDL)** type safety with **JSON-RPC 2.0** message framing and **JSONC** human-readable annotations — purpose-built for orchestrated AI agent fleets running on Discord.

---

## Table of Contents

1. [Overview](#overview)
2. [Architecture](#architecture)
3. [Protocol Design Principles](#protocol-design-principles)
4. [Repository Structure](#repository-structure)
5. [Core Concepts](#core-concepts)
6. [Interface Reference](#interface-reference)
7. [Discord Bot Permissions](#discord-bot-permissions)
8. [Message Lifecycle](#message-lifecycle)
9. [Error Handling](#error-handling)
10. [Transport Constraints](#transport-constraints)
11. [Wire Format Rules](#wire-format-rules)
12. [Quick Start](#quick-start)
13. [Implementation Guide](#implementation-guide)
14. [Security Considerations](#security-considerations)
15. [Versioning & Compatibility](#versioning--compatibility)
16. [Glossary](#glossary)

---

## Overview

DA2A is a **structured communications protocol** for multiple autonomous AI agents communicating within a shared Discord channel. It solves the core problem of agent coordination: agents need to **discover** each other, **delegate tasks**, **stream progress**, and **exchange rich structured data** — all within Discord’s messaging platform.

### Why DA2A?

| Problem | DA2A Solution |
|---------|______________|
| Unstructured agent messages are ambiguous | IDL-defined types with strict field contracts |
| Agents can’t discover each other’s capabilities | `AgentCard` with `SkillDef[]` published on connect |
| No correlation between requests and responses | JSON-RPC 2.0 `id` + `msg_id`/`reply_to_msg` |
| Async tasks have no lifecycle visibility | `ProgressEvent` notifications + `TaskState` FSM |
| Rich content (tables, code, JSON) is unformatted | `RichContent` multipart with `ContentType` discriminators |
| No shared conversation state | `Session` interface with RFC 7396 context patching |
| Bots have inconsistent permissions | `docs/discord-permissions.jsonc` with bitfield-precise role specs |

### Standards Alignment

| Standard | How DA2A Uses It |
|----------|_________________|
| **JSON-RPC 2.0** ([jsonrpc.org](https://www.jsonrpc.org/specification)) | Message framing, error codes, request/response correlation |
| **IDL principles** (JIDL, CORBA, OpenAPI) | Typed interface definitions with `@doc`, `@id`, `@notification`, `@idempotent` |
| **A2A Protocol** ([a2a-protocol.org](https://a2a-protocol.org)) | AgentCard discovery model, SkillDef structure |
| **JSONC** ([jsonc.org](https://jsonc.org)) | C-style `//` and `/* */` comments in `.jsonc` files |
| **RFC 7396** | JSON Merge Patch for session context updates |
| **CommonMark** | Markdown standard for `text/markdown` content parts (Discord-native) |

---

## Architecture

```
┌──────────────────────────────────────────────────────────────────┐
│                       Discord Guild (Server)                      │
│                                                                   │
│  ┌─────────────────┐    DA2A Channel: #agent-bus                  │
│  │ OrchestratorBot │ ◄─────────────────────────────────────┐  │
│  │ role:orchestrator│                                           │  │
│  └────────╌────────┘    ┌──────────────────────────────────── ┴─┐  │
│           │             │  MessageEnvelope {                       │  │
│    Submit │             │    da2a: "1.0.0",                        │  │
│    Task   │             │    payload: { jsonrpc: "2.0",            │  │
│           ▼             │      method: "da2a.task.submit",         │  │
│  ┌─────────────────┐    │      params: { spec: TaskSpec }          │  │
│  │ DataPipelineBot │ ───┘    }                                     │  │
│  │  role: worker   │                                              │  │
│  └────────╌────────┘                                              │  │
│           │  Broadcasts ProgressEvent notifications              │  │
│           │  Broadcasts TaskResult on completion                 │  │
│           ▼                                                       │  │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐           │  │
│  │ CodeReview   │  │  Validator   │  │   Observer   │           │  │
│  │ role: worker │  │role:validator│  │role: observer│           │  │
│  └──────────────┘  └──────────────┘  └──────────────┘           │  │
└──────────────────────────────────────────────────────────────────┘

Every arrow = a serialized MessageEnvelope containing one RpcFrame
```

### Agent Roles

| Role | Responsibility | Typical Bot |
|------|_______________|_____________|
| `orchestrator` | Workflow planning, task routing, session management | Chief-of-Staff / Planner |
| `worker` | Execute skills, stream progress, return results | DataPipelineBot, CodeReviewBot |
| `observer` | Read-only monitoring, logging, alerting | MetricsBot, AuditBot |
| `validator` | Quality gates, approval/rejection of task outputs | QABot, ComplianceBot |
| `notifier` | Publish announcements, summaries, alerts | ReportBot, AlertBot |

---

## Protocol Design Principles

1. **Every message is a `MessageEnvelope`** — All channel messages MUST be valid JSONC-encoded `MessageEnvelope` objects. No raw text.
2. **Strict JSON-RPC 2.0 payload framing** — Request/Response/Notification enforced by presence or absence of `id`.
3. **IDL-first type system** — All types declared in `schema/protocol.idl.jsonc` before use. No ad-hoc field invention.
4. **Deduplication by design** — `msg_id` (UUID v4) on every message; `idempotency_key` on tasks.
5. **Async-native** — Long tasks use Notification-pattern progress events; `da2a.task.get_result` is the polling fallback.
6. **Rich content as first-class** — `RichContent` with `ContentPart[]` supports Markdown, code, tables, JSON, and Discord embeds in one message.
7. **Capability-driven routing** — `AgentCard.skills[].description` (LLM-parseable markdown) + `CapabilityTag[]` for dynamic agent selection.
8. **Fail loudly** — All errors use DA2A-extended JSON-RPC error codes. Agents MUST NOT swallow errors silently.

---

## Repository Structure

```
discord-a2a-protocol/
│
├── schema/
│   └── protocol.idl.jsonc            # Canonical IDL: all types, structs, interfaces
│
├── examples/
│   ├── 01_register_request.jsonc     # da2a.registry.register → Request
│   ├── 02_register_response.jsonc    # da2a.registry.register → Response
│   ├── 03_task_submit.jsonc          # da2a.task.submit with multipart RichContent
│   ├── 04_task_progress_notification.jsonc   # Async progress notification
│   ├── 05_task_complete_notification.jsonc   # Task completion with artifacts
│   ├── 06_error_response.jsonc       # JSON-RPC 2.0 error (SKILL_NOT_FOUND)
│   ├── 07_heartbeat_notification.jsonc       # Liveness heartbeat
│   └── 08_rich_message_broadcast.jsonc       # Multipart broadcast (MD + table + code)
│
├── docs/
│   └── discord-permissions.jsonc     # Bot intents, permissions, OAuth2 URLs
│
└── README.md                         # This file
```

---

## Core Concepts

### MessageEnvelope

The outermost wrapper for every DA2A channel message. Provides routing, correlation, flow control, observability, and embeds one `RpcFrame`.

```jsonc
{
  "da2a":         "1.0.0",                   // Protocol version (REQUIRED)
  "msg_id":       "<uuid-v4>",               // Deduplication key (REQUIRED)
  "session_id":   "<uuid-v4>",               // Conversation grouping (optional)
  "timestamp":    "2026-05-06T12:00:00Z",    // ISO 8601 UTC (REQUIRED)
  "from":         "<discord-snowflake>",     // Sender bot ID (REQUIRED)
  "to":           "<discord-snowflake>",     // Target bot ID (omit = broadcast)
  "reply_to_msg": "<uuid-v4>",               // Async reply correlation
  "channel_id":   "<discord-snowflake>",     // Discord channel (REQUIRED)
  "priority":     "normal",                  // critical|high|normal|low|background
  "ttl_seconds":  60,                        // Drop if age > TTL
  "trace_id":     "<uuid-v4>",               // OpenTelemetry trace ID
  "payload":      { /* RpcFrame */ }         // JSON-RPC 2.0 frame (REQUIRED)
}
```

**Rules:**
- `msg_id` MUST be a fresh UUID v4 for every new outbound message.
- Broadcasts omit `to`. All channel-joined agents MUST process them.
- Messages older than `ttl_seconds` from `timestamp` MUST be silently dropped.

---

### RpcFrame (JSON-RPC 2.0)

| Frame Type | `id` present? | `method` present? | `result`/`error` present? |
|___________|:---:|:---:|:---:|
| **Request** | ✅ | ✅ | ❌ |
| **Response (success)** | ✅ | ❌ | ✅ `result` |
| **Response (error)** | ✅ | ❌ | ✅ `error` |
| **Notification** | ❌ | ✅ | ❌ |

```jsonc
// Request
{ "jsonrpc": "2.0", "id": "req-001", "method": "da2a.task.submit", "params": {...} }

// Success Response
{ "jsonrpc": "2.0", "id": "req-001", "result": {...} }

// Error Response
{ "jsonrpc": "2.0", "id": "req-001", "error": { "code": -32001, "message": "..." } }

// Notification — fire-and-forget, no response
{ "jsonrpc": "2.0", "method": "da2a.task.progress", "params": {...} }
```

---

### AgentCard

Machine-readable identity document published via `da2a.registry.register` on startup. Key fields:

| Field | Type | Purpose |
|_______|______|_________|
| `agent_id` | `AgentId` | Discord Snowflake Bot User ID |
| `description` | `markdown` | CommonMark — parsed by LLM orchestrators for routing |
| `role` | `AgentRole` | `orchestrator` / `worker` / `observer` / `validator` / `notifier` |
| `capabilities` | `CapabilityTag[]` | Searchable tags (e.g. `["etl", "data-quality"]`) |
| `skills` | `SkillDef[]` | Callable methods with JSON Schema contracts |
| `max_concurrency` | `uint32` | Max parallel tasks; `0` = unlimited |

---

### RichContent & ContentPart

`RichContent` is a **multipart payload**. Each `ContentPart` uses a `ContentType` discriminator:

| `ContentType` | Use case | Discord rendering |
|______________|__________|__________________|
| `text/plain` | Unformatted text | Plain |
| `text/markdown` | Narrative, summaries | CommonMark rendered |
| `application/json` | Structured data | Inline JSON string |
| `text/code` | Source code | Fenced block (syntax highlighted) |
| `text/table` | Tabular data | Markdown table |
| `application/embed` | Discord embeds | Rich embed card |
| `application/error` | Error detail | Error-formatted text |

> ⚠️ Discord caps messages at **2,000 characters** and embeds at **6,000 characters**. Use `content_url` to reference externally hosted large payloads. Violation returns `-32011` `CONTENT_TOO_LARGE`.

---

### TaskSpec & TaskResult

**TaskSpec** is the input contract for delegating work:

```jsonc
{
  "task_id":         "<uuid>",
  "skill_id":        "ingest_csv",
  "input":           { /* RichContent */ },
  "context":         { ... },
  "deadline_utc":    "2026-05-06T13:00:00Z",
  "idempotency_key": "run-q1-001"
}
```

**Task State Machine:**
```
submitted ──► queued ──► running ──► completed
               │             │
               └─────────────┴──► failed | cancelled | timed_out | awaiting_input
```

**TaskResult** carries `state`, `output` (RichContent), `artifacts[]`, `duration_ms`, and `tokens_used`.

---

### Sessions

Sessions group messages into a logical workflow context via a shared `context` object. Patched using RFC 7396 JSON Merge Patch — `null` values delete keys. Default TTL is 3600 seconds.

---

## Interface Reference

### da2a.registry

| Method | Type | Description |
|________|______|_____________|
| `da2a.registry.register` | Request | Publish AgentCard; receive peer cards |
| `da2a.registry.deregister` | Request | Graceful shutdown announcement |
| `da2a.registry.query` | Request | Discover agents by capability/role/skill |
| `da2a.registry.heartbeat` | **Notification** | 30s liveness signal with load metrics |

**Heartbeat SLA:** Agents silent for more than `2 × ttl_seconds` are removed from the registry.

---

### da2a.task

| Method | Type | Description |
|________|______|_____________|
| `da2a.task.submit` | Request (idempotent) | Dispatch TaskSpec to worker |
| `da2a.task.get_result` | Request | Poll for current TaskResult |
| `da2a.task.cancel` | Request | Best-effort cancellation |
| `da2a.task.progress` | **Notification** | Streaming progress with `pct` and metrics |
| `da2a.task.complete` | **Notification** | Final TaskResult broadcast |

---

### da2a.message

| Method | Type | Description |
|________|______|_____________|
| `da2a.message.send` | Request | Rich multipart message (directed or broadcast) |
| `da2a.message.react` | **Notification** | Emoji ACK on a Discord message |

**Emoji ACK convention:** `✅` accept · `🔄` processing · `❌` reject · `⏸` paused · `📦` artifact ready

---

### da2a.session

| Method | Type | Description |
|________|______|_____________|
| `da2a.session.create` | Request | Create session with participant list |
| `da2a.session.update_context` | Request | RFC 7396 patch shared context |
| `da2a.session.close` | Request | Terminate session |

---

## Discord Bot Permissions

### Gateway Intents

> ⚠️ **All DA2A bots MUST enable `MESSAGE_CONTENT`** in Developer Portal → Bot → Privileged Gateway Intents. Without it, all message bodies are empty.

| Intent | Value | Type | Required | Purpose |
|________|_______|______|:---:|_________|
| `GUILDS` | 1 | Standard | ✅ | Server/channel context |
| `GUILD_MESSAGES` | 512 | Standard | ✅ | Receive message events |
| `MESSAGE_CONTENT` | 32768 | **PRIVILEGED** | ✅ | Read message text |
| `GUILD_MEMBERS` | 2 | **PRIVILEGED** | Orchestrator | Member management |
| `GUILD_MESSAGE_REACTIONS` | 1024 | Standard | Recommended | Emoji ACK signals |

**Minimum bitfield (required only):** `33281` | **Full recommended:** `34307`

---

### Permissions by Agent Role

| Permission | Orchestrator | Worker | Observer | Validator | Notifier |
|___________|:---:|:---:|:---:|:---:|:---:|
| VIEW_CHANNEL | ✅ | ✅ | ✅ | ✅ | ✅ |
| SEND_MESSAGES | ✅ | ✅ | ❌ | ✅ | ✅ |
| READ_MESSAGE_HISTORY | ✅ | ✅ | ✅ | ✅ | ❌ |
| MANAGE_CHANNELS | ✅ | ❌ | ❌ | ❌ | ❌ |
| MANAGE_MESSAGES | ✅ | ❌ | ❌ | ✅ | ❌ |
| ADD_REACTIONS | ✅ | ✅ | ❌ | ✅ | ❌ |
| EMBED_LINKS | ✅ | ✅ | ❌ | ❌ | ✅ |
| ATTACH_FILES | ✅ | ✅ | ❌ | ❌ | ✅ |
| USE_APPLICATION_COMMANDS | ✅ | ❌ | ❌ | ❌ | ❌ |
| **Permission integer** | **2147616784** | **274978476** | **66560** | **74816** | **51200** |

> Full bitfield calculations and per-permission `@doc` annotations in `docs/discord-permissions.jsonc`.

---

### OAuth2 Invite URLs

Replace `CLIENT_ID` with your application ID from the [Discord Developer Portal](https://discord.com/developers/applications):

```bash
# Orchestrator
https://discord.com/oauth2/authorize?client_id=CLIENT_ID&scope=bot+applications.commands&permissions=2147616784

# Worker
https://discord.com/oauth2/authorize?client_id=CLIENT_ID&scope=bot&permissions=274978476

# Observer (read-only)
https://discord.com/oauth2/authorize?client_id=CLIENT_ID&scope=bot&permissions=66560

# Validator
https://discord.com/oauth2/authorize?client_id=CLIENT_ID&scope=bot&permissions=74816

# Notifier
https://discord.com/oauth2/authorize?client_id=CLIENT_ID&scope=bot&permissions=51200
```

---

### Developer Portal Checklist

1. **Create Application** — One per agent bot in [Developer Portal](https://discord.com/developers/applications).
2. **Enable Bot User** — Bot tab → Add Bot → copy token to `.env`. **Never commit tokens.**
3. **Enable `MESSAGE_CONTENT` Intent** — Bot → Privileged Gateway Intents → Message Content → **ON**.
4. **Enable `GUILD_MEMBERS` Intent** *(orchestrator only)* — Bot → Server Members Intent → **ON**.
5. **Generate OAuth2 URL** — OAuth2 → URL Generator → select scope and permissions per role table.
6. **Invite bot** — Open URL, select guild, authorize.
7. **Configure channel overrides** — Restrict each bot to only its DA2A channels via Discord role overrides.

---

## Message Lifecycle

### Startup Sequence
```
OrchestratorBot                  DataPipelineBot
      │                                │
      │ ◄── da2a.registry.register ────│  REQUEST (directed)
      │ ──► { accepted, peers, session_id } ────►│  RESPONSE
      │                                │
      │ ◄── da2a.registry.heartbeat ───│  NOTIFICATION (every 30s)
```

### Task Execution Flow
```
Orchestrator           Worker              All Channel Agents
     │                   │                        │
     │ da2a.task.submit ─►│                        │
     │◄─ { task_id, state: queued }               │
     │                   │                        │
     │                   │── da2a.task.progress ──►│ NOTIFICATION
     │                   │── da2a.task.progress ──►│ NOTIFICATION (pct: 75)
     │                   │                        │
     │                   │── da2a.task.complete ──►│ NOTIFICATION
```

---

## Error Handling

| Code | Name | Description |
|______|______|_____________|
| -32700 | Parse Error | Invalid JSON received |
| -32600 | Invalid Request | RpcFrame malformed |
| -32601 | Method Not Found | Unknown `method` name |
| -32602 | Invalid Params | `params` failed IDL validation |
| -32603 | Internal Error | Unhandled exception |
| **-32000** | AGENT_NOT_FOUND | Target `agent_id` not registered |
| **-32001** | SKILL_NOT_FOUND | `skill_id` not in agent’s SkillDefs |
| **-32002** | TASK_NOT_FOUND | `task_id` does not exist |
| **-32003** | SESSION_EXPIRED | `session_id` expired or evicted |
| **-32004** | RATE_LIMITED | Agent throttling; retry with backoff |
| **-32005** | PERMISSION_DENIED | Caller not authorized |
| **-32006** | TASK_TIMEOUT | Task exceeded `deadline_utc` |
| **-32007** | CAPACITY_EXCEEDED | Agent at `max_concurrency` |
| **-32008** | INVALID_AGENT_CARD | AgentCard schema validation failed |
| **-32009** | DUPLICATE_TASK | `idempotency_key` collision |
| **-32010** | BROADCAST_UNDELIVERED | At least one recipient failed |
| **-32011** | CONTENT_TOO_LARGE | RichContent exceeds Discord limits |

---

## Transport Constraints

| Constraint | Limit |
|___________|_______|
| Max message characters | **2,000** |
| Max embed description | **6,000 chars** |
| Max embed fields per message | **25** |
| Max embeds per message | **10** |
| Max file attachment | **25 MB** |
| Rate limit per channel/sec | **5 msgs** |
| Global rate limit/sec | **50 msgs** |
| Encoding | **UTF-8** |

---

## Wire Format Rules

1. All DA2A schema files use the `.jsonc` extension.
2. Comments use `//` (line) and `/* */` (block). Never inside string values.
3. Trailing commas are **prohibited**. Use a JSONC parser on receipt.
4. All messages MUST be UTF-8 encoded. No BOM.
5. Strip JSONC comments before posting to Discord; parse as JSONC on receipt.
6. Messages exceeding 2,000 chars MUST be chunked with `reply_to_msg` and `"chunk": N, "total_chunks": M` in part metadata.

---

## Quick Start

### 1. Bot setup
Follow the [Developer Portal Checklist](#developer-portal-checklist). Enable `MESSAGE_CONTENT` intent.

### 2. Define your AgentCard
```jsonc
{
  "agent_id": "YOUR_BOT_SNOWFLAKE",
  "name": "MyWorkerBot",
  "description": "**Processes data pipelines.**\nSupports CSV ingestion and schema validation.",
  "version": "1.0.0",
  "role": "worker",
  "capabilities": ["etl", "csv"],
  "skills": [{
    "id": "ingest_csv",
    "name": "Ingest CSV",
    "description": "Parses and validates a CSV file.",
    "method": "da2a.task.submit",
    "tags": ["csv", "ingest"]
  }],
  "max_concurrency": 3,
  "ttl_seconds": 300
}
```

### 3. Register on startup
Wrap a `da2a.registry.register` call in a `MessageEnvelope` and post to the DA2A channel. See `examples/01_register_request.jsonc`.

### 4. Handle tasks
Filter messages where `payload.method === "da2a.task.submit"` and `payload.params.spec.skill_id` matches your `SkillDef.id`.

### 5. Stream progress and complete
Broadcast `da2a.task.progress` Notifications (no `id`). When done, broadcast `da2a.task.complete` with full `TaskResult`.

---

## Implementation Guide

```python
import json, uuid, datetime

def parse_envelope(raw: str) -> dict:
    envelope = json.loads(raw)
    assert envelope["da2a"] == "1.0.0"
    return envelope

def is_for_me(env: dict, my_id: str) -> bool:
    return env.get("to") is None or env["to"] == my_id

def route_rpc(rpc: dict):
    method  = rpc.get("method")
    rpc_id  = rpc.get("id")  # None = Notification
    params  = rpc.get("params", {})
    if method == "da2a.task.submit":
        handle_task_submit(rpc_id, params["spec"])
    elif method == "da2a.task.progress":
        handle_progress(params)      # No response
    elif method == "da2a.registry.heartbeat":
        update_registry(params)      # No response

def make_progress(task_id: str, pct: int, msg: str) -> dict:
    return {
        "da2a": "1.0.0",
        "msg_id": str(uuid.uuid4()),
        "timestamp": datetime.datetime.utcnow().isoformat() + "Z",
        "from": MY_AGENT_ID,
        "channel_id": CHANNEL_ID,
        "payload": {
            "jsonrpc": "2.0",
            # No "id" — Notification
            "method": "da2a.task.progress",
            "params": {"task_id": task_id, "state": "running", "pct": pct, "message": msg}
        }
    }
```

---

## Security Considerations

1. **Token security** — Store bot tokens in `.env` / secrets manager. Never commit to Git.
2. **Channel isolation** — Use Discord channel-level permission overrides to confine each bot.
3. **Message validation** — Validate `da2a` version and all required IDL fields before processing.
4. **Idempotency** — Always set `idempotency_key` on task submissions to prevent replay.
5. **TTL enforcement** — Drop messages where `timestamp + ttl_seconds < now()`.
6. **Rate limiting** — Exponential backoff on `-32004`. Respect Discord’s 5 msg/s limit.
7. **Minimal privilege** — Only request `GUILD_MEMBERS` / `MESSAGE_CONTENT` for bots that need them.
8. **Secret scanning** — Run secret scanning CI on all commits. Replace placeholder IDs before production.

---

## Versioning & Compatibility

- DA2A follows **Semantic Versioning** (`MAJOR.MINOR.PATCH`).
- The `da2a` field in `MessageEnvelope` carries the protocol version. Agents MUST reject incompatible major versions.
- `AgentCard.version` is the agent’s **software version**, independent of protocol version.
- **Minor versions** add optional fields only. Agents MUST ignore unknown fields (forward compatibility).
- **Major versions** require explicit negotiation and a dual-version deployment window.

---

## Glossary

| Term | Definition |
|______|___________|
| **AgentCard** | Machine-readable identity + capability document published on connect |
| **CapabilityTag** | Kebab-case string for a broad capability domain (e.g. `"data-pipeline"`) |
| **ContentPart** | A single typed unit within a `RichContent` multipart payload |
| **DA2A** | Discord Agent-to-Agent — this protocol |
| **IDL** | Interface Description Language — type system for message contracts |
| **MessageEnvelope** | Outermost wrapper for every DA2A channel message |
| **Notification** | JSON-RPC 2.0 message with no `id` — fire-and-forget, no response |
| **RichContent** | Ordered `ContentPart[]` forming a multipart message body |
| **RpcFrame** | JSON-RPC 2.0 object embedded in `MessageEnvelope.payload` |
| **SessionId** | UUID correlating messages in one logical multi-agent workflow |
| **SkillDef** | IDL method signature declaring a callable capability on an agent |
| **Snowflake** | Discord’s 64-bit unique ID for users, channels, guilds, messages |
| **TaskId** | UUID uniquely identifying one delegated work unit |
| **TaskState** | FSM status: `submitted → queued → running → completed/failed/...` |
| **TTL** | Time-to-live; messages and registrations expire after this many seconds |

---

*DA2A Protocol v1.0.0 — Licensed under MIT.*
