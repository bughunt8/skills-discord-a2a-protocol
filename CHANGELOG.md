# Changelog

All notable changes to DA2A-HO are documented here.
Versioning follows [Semantic Versioning](https://semver.org/).

---

## [1.3.1] — 2026-05-07

### Changed
- **`schema/protocol.idl.jsonc`** — Removed `timestamp`, `channel_id`, `from/to`, and `delivery_mode` from `L2TalkHeader`. Discord provides all of these on every message object; duplicating them wastes tokens and introduces a potential consistency mismatch.
- **`schema/protocol.idl.jsonc`** — Added prominent `DISCORD-NATIVE FIELDS` comment block to `L2TalkHeader` documenting the boundary clearly for harness implementers.
- **`ThreeLayerPacket`** — `discord_message_id` is now explicitly `string|null` and marked as audit/replay use only; it is not part of normal packet flow.
- **`AGENTS.md`** — Added `Quick Reference` table distinguishing Discord-provided fields from DA2A-carried fields. Updated lifecycle checklist to instruct harnesses to read `message.timestamp` instead of expecting a `timestamp` field in the DA2A payload.
- **`examples/01` through `12`** — Replaced all inline `timestamp` fields with `"≪ from Discord message.timestamp ≫"` annotation.

### Added
- **`schema/admission.idl.jsonc`** — Full Bell–LaPadula admission protocol: `AdmissionRequest`, `AdmissionDecision`, `ViolationNotice`, `SessionPolicy`, `AccessMatrix`, admission state machine, and ten normative rules.
- **`examples/10_admission_request.jsonc`** — Agent joining and submitting BLP agreement.
- **`examples/11_admission_grant.jsonc`** — Orchestrator granting admission and broadcasting `SessionPolicy`.
- **`examples/12_admission_violation_halt.jsonc`** — BLP star-property violation → immediate halt → baton to human.
- **`docs/discord-permissions.jsonc`** — Minimum Discord bot scopes per DA2A role (orchestrator, specialist, observer).

---

## [1.3.0] — 2026-05-06

### Added
- Remote Meld (Subspace) extension: `RemoteMeldSession`, `RemoteMeldPacket`, baton primitives, memory proxy, and artifact announcement.
- Storage escalation ladder documentation.
- Three-lane transport model (control / working / artifact) formalized.

### Changed
- Protocol version string bumped to 1.3.0.
- `AGENTS.md` introduced (was inline in README).

---

## [1.2.0] — 2026-05-05

### Added
- KV-cache reference type (`KvCacheReference`).
- `ChannelQuality` struct and `DEGRADED_SIGNAL` state.
- Compact JSON theme negotiation.
- `BackoffConfig` for collision recovery.

---

## [1.1.0] — 2026-05-04

### Added
- Discord-native DM, reply, and thread delivery modes.
- Addressing precedence order.
- `WhiteboardRecord` for cross-session artifact sharing.

### Changed
- First proper three-layer (L1/L2/L3) packet structure.

---

## [1.0.0] — 2026-05-01

Initial DA2A-HO skeleton release.
