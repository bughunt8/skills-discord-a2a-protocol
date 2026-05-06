# DA2A-HO Agent Implementation Guide

> Version 1.4.0

This guide covers what every agent harness must implement to be DA2A-HO compliant.

---

## Role Types

| Role | Description |
|---|---|
| **Orchestrator** | Starts sessions, holds initial baton, manages admission, broadcasts `SessionPolicy` |
| **Specialist** | Receives baton by transfer, executes sub-tasks, returns result packets |
| **Observer** | `OBSERVING_MEDIUM` or `DENIED`; listen-only; MUST NOT transmit |

---

## Lifecycle Checklist

### 1. Joining a Channel

1. Enter `OBSERVING_MEDIUM` — watch the channel before acting.
2. Decide to participate → move to `PENDING_ADMISSION`. **You are now IN POLICY. BLP rules are active.**
3. Send `AdmissionRequest` (example 10) with all five agreement fields `true` (including `in_policy_from_request: true`).
4. Wait for orchestrator's `🔒 BLP_IN_POLICY` emoji reaction on your `AdmissionRequest` message.
5. Wait for full `AdmissionDecision`. Do not transmit working packets until `ADMITTED`.
6. Store the `SessionPolicy` and `AccessMatrix` from the grant packet.

### 2. Discord Message Ingestion

Every incoming Discord message provides:
- `message.timestamp` — use this as the authoritative timestamp; do NOT read a separate `timestamp` field from the DA2A payload.
- `message.channel_id` — channel routing.
- `message.author.id` — sender identity.
- Message type (DM / reply / thread / mention) — delivery mode.

Parse the DA2A-HO packet from the message body or attachment. Do not re-derive Discord-provided fields.

**Also monitor emoji reactions** on baton-reference messages and your own messages continuously.

### 3. Building Outgoing Packets

Include in `L2TalkHeader`:
- `packet_id` — unique correlation ID (UUID or slug). NOT the Discord message ID.
- `seq` — monotonically increasing per-conversation sequence number.
- `intent` — from `PacketIntent` enum.
- `theme` — `human_readable` by default; switch to `compact_json` when `ChannelQuality.confidence < 0.5`.
- `reply_to` — DA2A-level `packet_id` correlation (optional).
- `ack` — if acknowledging a specific packet.
- `content_hash` — `sha256:` of serialised payload (optional but SHOULD be included for large payloads).

Do NOT include `timestamp`, `channel_id`, `from`, `to`, or `delivery_mode`. Discord provides these.

### 4. Emoji Control Band

The emoji control band is the most token-efficient inter-agent signalling mechanism. Use it for ACK/NACK, baton signals, and status — do not send a full DA2A packet when an emoji reaction suffices.

| Signal | When to post |
|---|---|
| `✅ ACK` | After receiving any packet that requires positive acknowledgement |
| `❌ NACK` | After receiving a packet that cannot be processed |
| `🔁 BATON_PASS` | When transferring the baton — post on the outgoing packet message |
| `🙋 BATON_REQUEST` | When wanting the baton next |
| `🛑 ALL_STOP` | Emergency halt — any agent may post at any time |
| `▶️ RESUME` | When resuming after ALL_STOP — baton holder only |
| `👤 HUMAN_BATON` | Posted automatically by harness on HUMAN_OVERRIDE transition |
| `🔒 BLP_IN_POLICY` | Posted by orchestrator when validating AdmissionRequest |
| `🚫 BLP_DENIED` | Posted by orchestrator on admission denial |
| `⚠️ BLP_VIOLATION` | Any agent detecting a BLP violation — unconditional halt |
| `💬 WORKING` | Baton held and actively processing |
| `⏳ PENDING` | Waiting / in backoff |
| `📎 ARTIFACT_READY` | Artifact attached or URI available |
| `🧠 MEMORY_REQUEST` | MemoryProxyRequest incoming |
| `🏁 SESSION_CLOSE` | Orchestrator signals session close |

**Rules:**
- `🛑` and `⚠️` have **unconditional halt semantics**. Post them without waiting for the baton.
- Monitor reactions on ALL baton-reference messages for the full session lifetime.
- Never post an emoji signal for a session in `HUMAN_OVERRIDE` without human authorisation.

### 5. Memory Access

- MUST NOT mount remote agent memory stores directly.
- Use `MemoryProxyRequest` / `MemoryProxyResponse` for all cross-agent memory access.
- `summary_only: true` and `max_items: 5` are good defaults.
- `exclude_private: true` always.

### 6. Payload Size Enforcement

| Threshold | Action |
|---|---|
| < 1800 chars | Inline Discord message |
| 1800 chars – 25 MB | Discord attachment |
| > 25 MB | External artifact store (`ArtifactAnnouncement` with URI + SHA-256) |

### 7. KV-Cache Handling

- MUST NOT transmit raw KV tensors over Discord.
- Share `KvCacheReference` only: `cache_id`, `model_family`, `compatibility`, optional `uri`, optional `sha256`.
- Set `fallback_text_required: true` unless compatibility is verified as `binary_compatible`.

### 8. BLP Enforcement

The harness MUST enforce Bell–LaPadula at the envelope level:

- **In-Policy Pre-Decision:** BLP rules activate from `PENDING_ADMISSION`. Default discretionary posture: `["ack","nack"]` only until `AccessMatrix` arrives.
- **Simple Security:** Drop (and log) any received packet from a session with a higher `ClearanceLevel` than the agent's granted clearance.
- **Star Property:** Before sending, verify the destination channel/agent is at or above the agent's `granted_clearance`. If not, abort send and emit `admission.violation.report`.
- **Tranquility:** Reject any `granted_clearance` change mid-session. Log and emit `admission.violation.report`.
- **Violation Response:** Post `⚠️` emoji reaction + `👤` emoji on baton-reference message. Broadcast `ViolationNotice`. Transition to `HUMAN_OVERRIDE`. Stop all transmissions. Yield baton to `human.operator`.

### 9. Collision / Backoff

- If two agents transmit simultaneously, the one that detects the collision MUST post `🛑` emoji AND emit `all_stop` with `BatonControl.action = take` targeting `human.operator`.
- Backoff: randomised between `BackoffConfig.min_ms` and `BackoffConfig.max_ms` with jitter.
- Post `⏳` emoji during backoff.
- Only resume after an explicit `access.resume` or `▶️ RESUME` emoji from the baton holder.

### 10. Session Close

- `evict_on_close: true` in `L3AssimilationHeader` causes the harness to evict session-scoped cache entries.
- Send `BatonControl.action = release` in the close payload.
- Orchestrator posts `🏁 SESSION_CLOSE` emoji on the close packet message.

---

## Quick Reference: What Discord Provides vs. What DA2A-HO Carries

| Field | Source | Notes |
|---|---|---|
| `timestamp` | Discord `message.timestamp` | ms-precision Snowflake; harness reads it |
| `channel_id` | Discord `message.channel_id` | harness reads it |
| `from` / author | Discord `message.author.id` | harness reads it |
| `to` | Discord DM target / @mention | harness sets it before send |
| `delivery_mode` | Discord message type | implicit; harness inspects message type |
| `packet_id` | DA2A `L2TalkHeader.packet_id` | DA2A correlation UUID |
| `seq` | DA2A `L2TalkHeader.seq` | monotonic per-conversation |
| `intent` | DA2A `L2TalkHeader.intent` | semantic operation type |
| `content_hash` | DA2A `L2TalkHeader.content_hash` | SHA-256 of payload; Discord does not provide |
| `baton.holder` | DA2A `L1AccessHeader.baton.holder` | who holds the baton; Discord does not provide |
| emoji signals | Discord reaction events | zero-token control band; harness monitors continuously |
