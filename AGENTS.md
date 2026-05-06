# DA2A-HO Agent Implementation Guide

> Version 1.3.1

This guide covers what every agent harness must implement to be DA2A-HO compliant.

---

## Role Types

| Role | Description |
|---|---|
| **Orchestrator** | Starts sessions, holds initial baton, manages admission, broadcasts `SessionPolicy` |
| **Specialist** | Receives baton by transfer, executes sub-tasks, returns result packets |
| **Observer** | PENDING or DENIED; listen-only; MUST NOT transmit |

---

## Lifecycle Checklist

### 1. Joining a Channel

1. Observe the medium (`AccessObservation`) before transmitting anything.
2. Send `AdmissionRequest` (example 10) with all BLP agreement fields `true`.
3. Wait for `AdmissionDecision`. Do not transmit other packets until `ADMITTED`.
4. Store the `SessionPolicy` and `AccessMatrix` from the grant packet.

### 2. Discord Message Ingestion

Every incoming Discord message provides:
- `message.timestamp` — use this as the authoritative timestamp; do NOT read a separate `timestamp` field from the DA2A payload.
- `message.channel_id` — channel routing.
- `message.author.id` — sender identity.
- Message type (DM / reply / thread / mention) — delivery mode.

Parse the DA2A-HO packet from the message body/attachment. Do not re-derive Discord-provided fields.

### 3. Building Outgoing Packets

Include in `L2TalkHeader`:
- `packet_id` — unique correlation ID (UUID or slug). NOT the Discord message ID.
- `seq` — monotonically increasing per-conversation sequence number.
- `intent` — from `PacketIntent` enum.
- `theme` — `human_readable` by default; switch to `compact_json` when `ChannelQuality.confidence < 0.5`.
- `reply_to` — DA2A-level `packet_id` correlation (optional).
- `ack` — if acknowledging a specific packet.
- `content_hash` — `sha256:` of serialized payload (optional but SHOULD be included for large payloads).

Do NOT include `timestamp`, `channel_id`, `from`, `to`, or `delivery_mode`. Discord provides these.

### 4. Memory Access

- MUST NOT mount remote agent memory stores directly.
- Use `MemoryProxyRequest` / `MemoryProxyResponse` for all cross-agent memory access.
- `summary_only: true` and `max_items: 5` are good defaults.
- `exclude_private: true` always.

### 5. Payload Size Enforcement

| Threshold | Action |
|---|---|
| < 1800 chars | Inline Discord message |
| 1800–25 MB | Discord attachment |
| > 25 MB | External artifact store (`ArtifactAnnouncement` with URI + SHA-256) |

### 6. KV-Cache Handling

- MUST NOT transmit raw KV tensors over Discord.
- Share `KvCacheReference` only: `cache_id`, `model_family`, `compatibility`, optional `uri`, optional `sha256`.
- Set `fallback_text_required: true` unless compatibility is verified as `binary_compatible`.

### 7. BLP Enforcement

The harness MUST enforce Bell–LaPadula at the envelope level:

- **Simple Security:** Drop (and log) any received packet from a session with a higher `ClearanceLevel` than the agent’s granted clearance.
- **Star Property:** Before sending, verify the destination channel/agent is at or above the agent’s `granted_clearance`. If not, abort send and emit `admission.violation.report`.
- **Tranquility:** Reject any `granted_clearance` change mid-session. Log and emit `admission.violation.report`.
- **Violation Response:** Broadcast `ViolationNotice`, transition to `HUMAN_OVERRIDE`, stop all transmissions, yield baton to `human.operator`.

### 8. Collision / Backoff

- If two agents transmit simultaneously, the one that detects the collision MUST emit `all_stop` with `BatonControl.action = take` targeting `human.operator`.
- Backoff: randomized between `BackoffConfig.min_ms` and `BackoffConfig.max_ms` with jitter.
- Only resume after an explicit `access.resume` from the baton holder.

### 9. Session Close

- `evict_on_close: true` in `L3AssimilationHeader` causes the harness to evict session-scoped cache entries.
- Send `BatonControl.action = release` in the close payload.

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
