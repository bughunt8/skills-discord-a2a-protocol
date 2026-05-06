# Changelog

All notable changes to DA2A-HO are documented here.
Versioning follows [Semantic Versioning](https://semver.org/).

---

## [1.4.0] — 2026-05-07

### Added
- **Emoji Signal Vocabulary (canonical, normative)** — `schema/admission.idl.jsonc` Section 2 defines 15 emoji signals forming a zero-token, ~50 ms control band. All admitted agents MUST recognise and honour them. `🛑 ALL_STOP` and `⚠️ BLP_VIOLATION` have unconditional halt semantics.
- **In-Policy Pre-Decision Rule** — Agents are formally IN POLICY from the moment `AdmissionRequest` is submitted (before `AdmissionDecision` arrives). BLP rules are active immediately; default discretionary posture is `["ack","nack"]` until `AccessMatrix` is delivered. `agreement.in_policy_from_request: true` field added to `AdmissionRequest`.
- **`OBSERVING_MEDIUM` state** — New initial BLP admission state added before `PENDING_ADMISSION`. BLP rules are not yet active in this state. Agents may observe the channel silently before committing to join.
- **`examples/13_emoji_signal_sequence.jsonc`** — Full OpenClaw ↔ OpenCode emoji-only control band walkthrough: ACK, WORKING, BATON_PASS, BATON_REQUEST, RESUME. Demonstrates ~99% overhead reduction vs. full DA2A ACK packets for control signals.
- **`EmojiSignal` type** — Added to `protocol.idl.jsonc` Section 1 types, referencing the vocabulary in `admission.idl.jsonc`.
- **`emoji_reaction_latency_ms` and `emoji_reaction_token_cost`** — Added to `transport_constraints` in `protocol.idl.jsonc`.

### Changed
- **`README.md`** — Full rewrite preserving all original sections. Added: _Emoji Signalling — Agent-to-Agent Control Band_ section with canonical table; _BLP Admission State Machine_ section with ASCII state diagram and transition table; _In-Policy Pre-Declaration_ sub-section under Bell–LaPadula Admission Contract. Normative rules expanded to 10.
- **`schema/admission.idl.jsonc`** — `blp_rules` extended with `in_policy_pre_decision` rule; `admission_state_machine` extended with `OBSERVING_MEDIUM` state and `in_policy_boundary` marker; `ViolationNotice.effect` notes emoji posted on halt; `AccessMatrix` example shows `_default_pre_admission` posture; `SessionPolicy` adds `in_policy_pre_decision_active` flag.
- **`schema/protocol.idl.jsonc`** — Version bumped to 1.4.0. `EmojiSignal` type added. State machine transitions updated to reference emoji triggers (e.g. `🛑 ALL_STOP`, `▶️ RESUME`, `🏁 SESSION_CLOSE`). Transport constraints updated with emoji metadata.
- **`AGENTS.md`** — Section 4 (Emoji Control Band) added with per-signal usage table and rules. Section 8 (BLP Enforcement) updated with in-policy pre-decision note. Section 9 (Collision/Backoff) updated with `🛑` + `⏳` emoji instructions. Quick Reference table updated with emoji signals row.

---

## [1.3.1] — 2026-05-07

### Changed
- **`schema/protocol.idl.jsonc`** — Removed `timestamp`, `channel_id`, `from/to`, and `delivery_mode` from `L2TalkHeader`. Discord provides all of these on every message object.
- **`schema/protocol.idl.jsonc`** — Added `DISCORD-NATIVE FIELDS` comment block to `L2TalkHeader`.
- **`ThreeLayerPacket`** — `discord_message_id` marked as audit/replay use only.
- **`AGENTS.md`** — Added `Quick Reference` table; updated lifecycle checklist.
- **`examples/01` through `12`** — Replaced all inline `timestamp` fields with Discord source annotation.

### Added
- **`schema/admission.idl.jsonc`** — Full Bell–LaPadula admission protocol.
- **`examples/10_admission_request.jsonc`** — Agent joining and submitting BLP agreement.
- **`examples/11_admission_grant.jsonc`** — Orchestrator granting admission and broadcasting `SessionPolicy`.
- **`examples/12_admission_violation_halt.jsonc`** — BLP violation → immediate halt → baton to human.
- **`docs/discord-permissions.jsonc`** — Minimum Discord bot scopes per DA2A role.

---

## [1.3.0] — 2026-05-06

### Added
- Remote Meld (Subspace) extension: `RemoteMeldSession`, `RemoteMeldPacket`, baton primitives, memory proxy, and artifact announcement.
- Storage escalation ladder documentation.
- Three-lane transport model (control / working / artifact) formalised.

### Changed
- Protocol version string bumped to 1.3.0.
- `AGENTS.md` introduced.

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
