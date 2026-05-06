# DA2A-HO — Discord Agent-to-Agent High-Efficiency Operations Protocol

> **Version 1.3.1** · MIT License

DA2A-HO is a lightweight, Discord-native communication protocol for orchestrating cooperative multi-agent LLM systems. It defines exactly what agents must carry in their own envelopes and what Discord already provides — keeping every packet as small as possible while preserving correctness, security, and observability.

---

## Architecture Overview

DA2A-HO is structured as an ISO-style three-layer stack:

| Layer | Name | Responsibility |
|---|---|---|
| 1 | **Accessing** | Medium-access discipline — baton, collision avoidance, backoff |
| 2 | **Talking** | Encoded packet transfer between agents |
| 3 | **Assimilation** | Local KV-cache and context update from received data |

---

## Core Design Principles

**Discord-native first.** Discord already provides `timestamp`, `channel_id`, `author`, and delivery context (DM / reply / thread / mention) on every message. DA2A-HO envelopes contain **only** what Discord does not provide.

**Deltas, not transcripts.** Agents send state deltas and decision summaries — never full conversation replays.

**Memory isolation.** Agents MUST NOT mount each other’s memory stores directly. All cross-agent memory access goes through a bounded `MemoryProxyRequest / MemoryProxyResponse` exchange.

**KV-cache by reference.** Raw KV tensors MUST NOT be transmitted over Discord. Only a `KvCacheReference` (id, compatibility, optional URI, SHA-256) is shared.

**Storage escalation ladder.** Discord inline message → Discord attachment → external object store. Cloudflare R2 (or equivalent) is _optional_, used only when payloads exceed Discord’s 25 MB attachment limit or require durable multi-session storage.

**Bell–LaPadula security.** Every agent MUST complete the BLP admission handshake before participating. Violations halt the session immediately and transfer the baton to the human operator.

---

## Bell–LaPadula Admission

Every agent joining a DA2A-HO conversation MUST:

1. Send an `AdmissionRequest` (example 10) with all four BLP agreement fields set to `true`.
2. Await an `AdmissionDecision` (example 11) from the orchestrator.
3. Abide by the `SessionPolicy` and `AccessMatrix` for the full session lifetime.

The conversation does not advance past `protocol_state: OPENING` until all expected agents are `ADMITTED`.

| BLP Rule | DA2A Meaning |
|---|---|
| Simple Security | Agent MUST NOT process packets from sessions above its clearance |
| Star (★) Property | Agent MUST NOT send higher-clearance content to lower-clearance channels |
| Discretionary | Permitted `PacketIntent` per agent defined in `AccessMatrix` |
| Tranquility | `ClearanceLevel` is immutable for the session lifetime |

**Violation consequence:** Session → `HUMAN_OVERRIDE`; baton → `human.operator`; all agents halt immediately. See example 12.

---

## Storage — Do You Need R2?

| Payload type | Transport | Need external storage? |
|---|---|---|
| Baton control, ACK/NACK, status | Inline Discord message | No |
| State delta, memory proxy, JSON patch | Inline message or attachment | No |
| Logs, schemas, test output | Discord attachment (≤25 MB) | Usually no |
| Multi-file bundles, replay archives, cross-session cache | External object store | Yes |

External storage is optional. Design for inline Discord first; escalate only when necessary.

---

## Discord Native Fields

The following fields are provided by Discord and MUST NOT be duplicated in any DA2A-HO packet payload:

- `timestamp` → `Discord message.timestamp` (Snowflake; millisecond precision)
- `channel_id` → `Discord message.channel_id`
- `from` / `author` → `Discord message.author.id`
- `to` → Discord DM target or @mention (set by harness before send)
- `delivery_mode` → implicit from Discord message type (DM / reply / thread / mention)

---

## Delivery Addressing Precedence

Harnesses select delivery type in this order (not carried inside DA2A packets):

1. `discord_dm` — private unicast
2. `discord_reply` — in-thread correlation
3. `discord_thread` — topic isolation
4. `discord_mention` — soft notification
5. `envelope_only` — control-plane fallback

---

## Repository Layout

```
schema/
  protocol.idl.jsonc      — Core three-layer packet schema (v1.3.1)
  admission.idl.jsonc     — Bell–LaPadula admission protocol

examples/
  01_session_open.jsonc
  02_session_ack.jsonc
  03_baton_transfer_and_handoff.jsonc
  04_memory_proxy.jsonc
  05_kv_cache_ref.jsonc
  06_all_stop_and_resume.jsonc
  07_degraded_signal_adaptation.jsonc
  08_artifact_whiteboard_extension.jsonc
  09_session_close.jsonc
  10_admission_request.jsonc
  11_admission_grant.jsonc
  12_admission_violation_halt.jsonc

docs/
  discord-permissions.jsonc — Minimum Discord bot scopes per DA2A role

AGENTS.md    — Agent implementation guide
CHANGELOG.md — Version history
```

---

## Normative Rules

1. Implementations **SHOULD** use Discord-native DM, reply, and thread as the primary delivery substrate.
2. Implementations **SHOULD** prefer compact state deltas, hashes, and references over transcript replay.
3. Implementations **MAY** use external object storage for artifacts exceeding Discord limits or requiring durable multi-session retrieval.
4. Implementations **MUST NOT** assume direct remote memory mounting between agent runtimes over Discord transport.
5. Implementations **MUST NOT** duplicate `timestamp`, `channel_id`, `from/author`, `to`, or `delivery_mode` in DA2A-HO packet payloads.
6. Every agent **MUST** complete the BLP admission handshake before transmitting any non-admission packet.
7. On any BLP violation, the session **MUST** halt immediately and the baton **MUST** transfer to `human.operator`.

---

## Normative Language

Follows [RFC 2119](https://www.rfc-editor.org/rfc/rfc2119): MUST / SHOULD / MAY / MUST NOT / SHOULD NOT.
