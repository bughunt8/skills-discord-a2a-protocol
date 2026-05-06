<div align="center">

# 🤖 DA2A
### Discord Agent-to-Agent Communication Protocol

**Structured. Typed. Real-time. Built for AI agent fleets on Discord.**

[![Version](https://img.shields.io/badge/version-1.1.0-blue?style=flat-square)](https://github.com/bughunt8/skills-discord-a2a-protocol/releases)
[![Protocol](https://img.shields.io/badge/protocol-JSON--RPC%202.0-green?style=flat-square)](https://www.jsonrpc.org/specification)
[![IDL](https://img.shields.io/badge/schema-IDL%20%2B%20JSONC-orange?style=flat-square)](schema/protocol.idl.jsonc)
[![License](https://img.shields.io/badge/license-MIT-purple?style=flat-square)](LICENSE)
[![Discord](https://img.shields.io/badge/transport-Discord%20Gateway-5865F2?style=flat-square&logo=discord&logoColor=white)](https://discord.com/developers/docs/intro)

</div>

---

> **DA2A** is the missing communication layer for multi-agent Discord bots — a rigorous wire protocol that lets AI agents **discover** each other, **delegate tasks**, **stream progress**, and **exchange rich structured data**, all through Discord's native delivery mechanisms.

---

## 🧠 Why DA2A?

Today's AI bots post raw messages. DA2A agents speak a **contract**.

| Without DA2A | With DA2A |
|---|---|
| Bots post free-text, parsing is guesswork | Every message is a typed `MessageEnvelope` |
| No way to know what other bots can do | `AgentCard` + `SkillDef[]` published on startup |
| Requests and replies are uncorrelated | JSON-RPC 2.0 `id` + Discord `message.reply()` threading |
| Long tasks block or go silent | `ProgressEvent` notifications with `pct` streaming |
| Rich data (tables, code, JSON) arrives mangled | `RichContent` multipart with 7 typed `ContentType`s |
| Unicast requires custom routing logic | Discord DM preferred; envelope `to` is audit metadata |
| Bot permissions are guesswork | Bitfield-precise roles in `docs/discord-permissions.jsonc` |

---

## 📐 Architecture

```
┌──────────────────────────────── Discord Guild ────────────────────────────────┐
│                                                                                │
│   ┌──────────────────┐    Discord DM (preferred unicast)                       │
│   │  OrchestratorBot │ ◄─────────────────────────────────────────────────────┐ │
│   │ role:orchestrator│                                                        │ │
│   └────────┬─────────┘    da2a.task.submit                                    │ │
│            │  (via Discord DM if dm_enabled, else channel)                    │ │
│            ▼                                                                   │ │
│   ┌──────────────────┐  discord.reply() → da2a.task.progress ────────────────┘ │
│   │  DataPipelineBot │  discord.reply() → da2a.task.complete                   │
│   │  role: worker    │                                                          │
│   └──────────────────┘                                                          │
│                                                                                │
│   #agent-bus channel  (broadcast notifications + fallback unicast)             │
│   ┌────────────┐   ┌────────────┐   ┌────────────┐   ┌────────────┐           │
│   │ CodeReview │   │ Validator  │   │  Observer  │   │  Notifier  │           │
│   │  worker    │   │ validator  │   │  observer  │   │  notifier  │           │
│   └────────────┘   └────────────┘   └────────────┘   └────────────┘           │
└────────────────────────────────────────────────────────────────────────────────┘
```

**Discord owns WHERE and HOW. DA2A owns WHAT and WHY.**

---

## ✨ Key Features

### 🔌 IDL-First Type System
All types, structs, and interfaces declared in `schema/protocol.idl.jsonc` — 13 structs, 4 interfaces, 20+ primitive aliases. No ad-hoc fields.

### 📨 JSON-RPC 2.0 Framing
Every message is a `MessageEnvelope` wrapping a strict JSON-RPC 2.0 `RpcFrame`. Request / Response / Notification semantics enforced by presence or absence of `id`.

### 🎯 Discord-Native Addressing (v1.1.0)
Agent-to-agent delivery uses Discord's own mechanisms in priority order: **DM → reply() → @mention → channel fallback**. The envelope `to` and `reply_to_msg` fields are semantic metadata for audit logs, not routing instructions.

### 🃏 AgentCard Discovery
Agents publish a machine-readable `AgentCard` on startup with `SkillDef[]`, JSON Schema contracts, and `dm_enabled` flag. Orchestrators use these for dynamic routing and delivery mode selection.

### 🖼️ RichContent Multipart
A single message carries Markdown, JSON, syntax-highlighted code, tables, and Discord embeds — all typed and ordered.

### ⚡ Async Task Lifecycle
`submitted → queued → running → completed | failed | cancelled | timed_out` with streaming `ProgressEvent` notifications.

### 🛡️ Typed Error Codes
13 DA2A-specific JSON-RPC error codes (`-32000` to `-32012`) including `DM_UNAVAILABLE` for graceful delivery fallback.

---

## 📁 Repository Layout

```
skills-discord-a2a-protocol/
│
├── AGENTS.md                              ← AI agent instructions for this repo
│
├── schema/
│   └── protocol.idl.jsonc                ← Canonical IDL (types, structs, interfaces)
│
├── examples/
│   ├── 01_register_request.jsonc
│   ├── 02_register_response.jsonc
│   ├── 03_task_submit.jsonc
│   ├── 04_task_progress_notification.jsonc
│   ├── 05_task_complete_notification.jsonc
│   ├── 06_error_response.jsonc
│   ├── 07_heartbeat_notification.jsonc
│   ├── 08_rich_message_broadcast.jsonc
│   ├── 09_direct_message.jsonc           ← NEW v1.1.0: Discord DM unicast
│   └── 10_discord_reply.jsonc            ← NEW v1.1.0: discord.reply() threading
│
├── docs/
│   └── discord-permissions.jsonc
│
└── README.md
```

---

## 📬 Addressing Model (v1.1.0)

DA2A is **Discord-native**. Delivery uses Discord's own mechanisms before falling back to envelope fields.

| Priority | Mode | When | How |
|:---:|---|---|---|
| **1** | `discord_dm` | Unicast, `dm_enabled: true` | `create_dm(bot_id)` → `send(envelope)` |
| **2** | `discord_reply` | In-channel response | `original_msg.reply(envelope)` |
| **3** | `discord_mention` | Soft notify | `channel.send(f'<@bot_id> {envelope}')` |
| **4** | `envelope_to_field` | Fallback / audit / replay | `channel.send(envelope)`, filter on `envelope.to` |

> **Key insight:** `MessageEnvelope.to` and `.reply_to_msg` are **semantic metadata** — they record intent and lineage for audit logs and replay systems. Discord handles actual message routing.

---

## 🔑 Core Types

### MessageEnvelope

```jsonc
{
  "da2a":         "1.1.0",
  "msg_id":       "<uuid-v4>",
  "timestamp":    "2026-05-06T22:00:00Z",
  "from":         "<snowflake>",
  "to":           "<snowflake>",    // semantic metadata — not a routing instruction
  "channel_id":   "<snowflake>",
  "priority":     "normal",
  "ttl_seconds":  60,
  "delivery": {
    "mode": "discord_dm",           // actual Discord mechanism used
    "discord_msg_id": "<snowflake>",
    "attempted_modes": ["discord_dm"]
  },
  "payload": { /* RpcFrame */ }
}
```

### RpcFrame (JSON-RPC 2.0)

| Frame Type | `id` | `method` | `result`/`error` |
|---|:---:|:---:|:---:|
| **Request** | ✅ | ✅ | ❌ |
| **Response (ok)** | ✅ | ❌ | ✅ `result` |
| **Response (err)** | ✅ | ❌ | ✅ `error` |
| **Notification** | ❌ | ✅ | ❌ |

---

## 🔌 Interface Reference

### `da2a.registry`
| Method | Type | Description |
|---|---|---|
| `da2a.registry.register` | Request | Publish `AgentCard` with `dm_enabled`; receive peers |
| `da2a.registry.deregister` | Request | Graceful shutdown |
| `da2a.registry.query` | Request | Find agents; response includes `dm_enabled` for delivery routing |
| `da2a.registry.heartbeat` | Notification | 30s liveness signal |

### `da2a.task`
| Method | Type | Description |
|---|---|---|
| `da2a.task.submit` | Request (idempotent) | Dispatch via DM if `dm_enabled`, else channel |
| `da2a.task.get_result` | Request | Poll |
| `da2a.task.cancel` | Request | Best-effort cancel |
| `da2a.task.progress` | Notification | Stream via `discord_reply` on original submit |
| `da2a.task.complete` | Notification | Final result via `discord_reply` |

### `da2a.message` (v1.1.0 expanded)
| Method | Type | Description |
|---|---|---|
| `da2a.message.send` | Request | Channel send with `delivery_mode` param |
| `da2a.message.react` | Notification | Emoji ACK |
| `da2a.message.dm` | Request | **NEW** — Preferred unicast via Discord DM |
| `da2a.message.reply` | Request | **NEW** — Preferred in-thread via `discord.reply()` |

**Emoji ACK convention:** `✅` accept · `🔄` processing · `❌` reject · `⏸` paused · `📦` artifact ready

### `da2a.session`
| Method | Type | Description |
|---|---|---|
| `da2a.session.create` | Request | Start multi-agent workflow session |
| `da2a.session.update_context` | Request | RFC 7396 Merge Patch |
| `da2a.session.close` | Request | Terminate |

---

## 🤖 Discord Permissions

### ⚠️ Enable MESSAGE_CONTENT Intent
**Developer Portal → Your App → Bot → Privileged Gateway Intents → Message Content → ON**

| Intent | Value | Type | Required |
|---|---|---|:---:|
| `GUILDS` | 1 | Standard | ✅ |
| `GUILD_MESSAGES` | 512 | Standard | ✅ |
| `MESSAGE_CONTENT` | 32768 | **PRIVILEGED** | ✅ |
| `GUILD_MEMBERS` | 2 | **PRIVILEGED** | Orchestrator only |
| `GUILD_MESSAGE_REACTIONS` | 1024 | Standard | Recommended |

| Role | Integer | DMs needed |
|---|---|:---:|
| Orchestrator | `2147616784` | ✅ |
| Worker | `274978476` | ✅ |
| Observer | `66560` | ❌ |
| Validator | `74816` | Optional |
| Notifier | `51200` | Optional |

---

## ❌ Error Codes

| Code | Name | Trigger |
|---|---|---|
| -32700 | Parse Error | Invalid JSON |
| -32600 | Invalid Request | Malformed RpcFrame |
| -32601 | Method Not Found | Unknown method |
| -32602 | Invalid Params | IDL validation failed |
| **-32000** | AGENT_NOT_FOUND | Target not registered |
| **-32001** | SKILL_NOT_FOUND | skill_id unknown |
| **-32004** | RATE_LIMITED | Throttled — backoff |
| **-32007** | CAPACITY_EXCEEDED | Agent at max_concurrency |
| **-32011** | CONTENT_TOO_LARGE | Exceeds Discord limits |
| **-32012** | DM_UNAVAILABLE | **NEW** — DMs disabled; fall back to reply/mention |

---

## 🚀 Quick Start

```python
# 1. Publish AgentCard on startup
envelope = {
  "da2a": "1.1.0",
  "msg_id": str(uuid4()),
  "from": MY_BOT_ID,
  "channel_id": DA2A_CHANNEL_ID,
  "timestamp": utcnow(),
  "payload": {
    "jsonrpc": "2.0", "id": "reg-001",
    "method": "da2a.registry.register",
    "params": { "card": { ...MY_AGENT_CARD, "dm_enabled": True } }
  }
}

# 2. Receive a task — handle it, stream progress via discord.reply()
async def on_task(original_msg, spec):
    for pct in [25, 50, 75, 100]:
        progress_env = build_progress_notification(spec["task_id"], pct)
        await original_msg.reply(json.dumps(progress_env))  # Discord threading

# 3. Send unicast via DM (preferred)
async def dm_agent(target_id, content):
    try:
        user = await bot.fetch_user(int(target_id))
        dm = await user.create_dm()
        await dm.send(json.dumps(build_envelope(content, mode="discord_dm")))
    except discord.Forbidden:
        raise DA2AError(-32012, "DM_UNAVAILABLE")  # caller falls back
```

---

## 🗺️ Roadmap

| Version | What | Status |
|---|---|---|
| **v1.0.0** | Core IDL, 4 interfaces, 8 wire examples | ✅ Shipped |
| **v1.1.0** | Discord-native addressing, DM + reply methods, DiscordDelivery struct | ✅ Shipped |
| **v1.2.0** | Python reference implementation (`da2a-py`) | 🔜 Planned |
| **v1.3.0** | JSON Schema validator CLI (`da2a validate msg.json`) | 🔜 Planned |
| **v2.0.0** | WebSocket transport variant | 💡 Exploring |

---

## 🔒 Security

1. Store bot tokens in `.env` / secrets manager — never commit
2. Enable `MESSAGE_CONTENT` only for bots that need it
3. Validate `da2a` version field and required IDL fields before processing
4. Use `idempotency_key` to handle Discord's at-least-once delivery
5. Drop messages where `timestamp + ttl_seconds < now()`
6. Exponential backoff on `-32004 RATE_LIMITED`
7. Handle `-32012 DM_UNAVAILABLE` gracefully — fall back to `discord_reply`

---

## 🤝 Standards

- [JSON-RPC 2.0](https://www.jsonrpc.org/specification) — framing and error model
- [A2A Protocol](https://a2a-protocol.org) — AgentCard and SkillDef patterns
- [RFC 7396](https://datatracker.ietf.org/doc/html/rfc7396) — JSON Merge Patch
- [CommonMark](https://commonmark.org) — Markdown standard
- [AGENTS.md](https://agents.md) — AI agent instruction standard
- [Discord Developer Docs](https://discord.com/developers/docs) — Gateway, REST, Intents

---

## 📄 License

MIT — free to use, modify, and distribute.

---

<div align="center">

**DA2A v1.1.0** — *Discord handles WHERE. DA2A handles WHAT.*

*Built by [Profit Rise Consulting](https://profitriseco.com) · Hong Kong*

</div>
