# DA2A-HO — Discord Agent-to-Agent High-Efficiency Operations Protocol

> **Version 1.4.0** · MIT License · [Changelog](CHANGELOG.md) · [Agent Guide](AGENTS.md)

DA2A-HO is a lightweight, Discord-native communication protocol for orchestrating cooperative multi-agent LLM systems. It defines exactly what agents must carry in their own envelopes and what Discord already provides — keeping every packet as small as possible while preserving correctness, security, and observability.

---

## Table of Contents

1. [Architecture Overview](#architecture-overview)
2. [Core Design Principles](#core-design-principles)
3. [Emoji Signalling — Agent-to-Agent Control Band](#emoji-signalling--agent-to-agent-control-band)
4. [Bell–LaPadula Admission Contract](#bellapadula-admission-contract)
5. [BLP Admission State Machine](#blp-admission-state-machine)
6. [Storage — Do You Need R2?](#storage--do-you-need-r2)
7. [Discord Native Fields](#discord-native-fields)
8. [Delivery Addressing Precedence](#delivery-addressing-precedence)
9. [Protocol State Machine](#protocol-state-machine)
10. [Repository Layout](#repository-layout)
11. [Normative Rules](#normative-rules)
12. [Normative Language](#normative-language)

---

## Architecture Overview

DA2A-HO is structured as an ISO-style three-layer stack:

| Layer | Name | Responsibility |
|---|---|---|
| 1 | **Accessing** | Medium-access discipline — baton, collision avoidance, backoff |
| 2 | **Talking** | Encoded packet transfer between agents |
| 3 | **Assimilation** | Local KV-cache and context update from received data |

On top of these three layers sits a mandatory security gate: the **Bell–LaPadula Admission Contract**. No agent may reach Layer 1 until it has cleared the admission gate.

```
┌─────────────────────────────────────────────────────────────┐
│                    Human Operator (Baton)                   │
├─────────────────────────────────────────────────────────────┤
│  BLP Admission Gate  ←  Agent joins here first              │
├────────────┬────────────┬────────────────────────────────────┤
│  Layer 3   │ Layer 2    │ Layer 1                            │
│  Assimil.  │ Talking    │ Accessing (Baton / CSMA-CA)        │
└────────────┴────────────┴────────────────────────────────────┘
          ↕                       ↕
    Discord DM / Reply / Thread / Mention
```

---

## Core Design Principles

**Discord-native first.** Discord already provides `timestamp`, `channel_id`, `author`, and delivery context (DM / reply / thread / mention) on every message. DA2A-HO envelopes contain **only** what Discord does not provide.

**Deltas, not transcripts.** Agents send state deltas and decision summaries — never full conversation replays.

**Memory isolation.** Agents MUST NOT mount each other's memory stores directly. All cross-agent memory access goes through a bounded `MemoryProxyRequest / MemoryProxyResponse` exchange.

**KV-cache by reference.** Raw KV tensors MUST NOT be transmitted over Discord. Only a `KvCacheReference` (id, compatibility, optional URI, SHA-256) is shared.

**Storage escalation ladder.** Discord inline message → Discord attachment → external object store. Cloudflare R2 (or equivalent) is _optional_, used only when payloads exceed Discord's 25 MB attachment limit or require durable multi-session storage.

**Bell–LaPadula security.** Every agent MUST complete the BLP admission handshake and be formally declared **IN POLICY** before participating. Violations halt the session immediately and transfer the baton to the human operator unconditionally.

**Emoji signalling.** Discord reaction emoji serve as a zero-token, near-zero-latency control band between agents. Emoji signals are defined in the BLP Admission Contract and are the most efficient possible agent-to-agent coordination channel.

---

## Emoji Signalling — Agent-to-Agent Control Band

Discord emoji reactions are **free in token cost and latency**: they require no message body, no DA2A envelope, no baton, and are delivered by Discord's push infrastructure. DA2A-HO defines a canonical emoji vocabulary for the most frequent inter-agent signals. All agents MUST recognise this vocabulary upon admission.

### Canonical Emoji Signal Table

| Emoji | Signal Name | Direction | Meaning |
|---|---|---|---|
| ✅ | `ACK` | Any → Any | Positive acknowledgement — packet received and understood |
| ❌ | `NACK` | Any → Any | Negative acknowledgement — packet rejected or unprocessable |
| 🔁 | `BATON_PASS` | Holder → Next | "I am transferring the baton to you" |
| 🙋 | `BATON_REQUEST` | Any → Holder | "I want the baton next" |
| 🛑 | `ALL_STOP` | Any → All | Emergency halt — all agents pause immediately |
| ▶️ | `RESUME` | Holder → All | "Session resuming — come back to ACTIVE" |
| 👤 | `HUMAN_BATON` | System → All | "Human operator now holds the baton" |
| 🔒 | `BLP_IN_POLICY` | Orchestrator → Agent | "You are declared IN POLICY — admission approved" |
| 🚫 | `BLP_DENIED` | Orchestrator → Agent | "Admission denied — enter listen-only mode" |
| ⚠️ | `BLP_VIOLATION` | Any → All | "BLP rule violated — session halting" |
| 💬 | `WORKING` | Agent → All | "I have the baton and am actively processing" |
| ⏳ | `PENDING` | Agent → All | "I am waiting / in backoff" |
| 📎 | `ARTIFACT_READY` | Agent → All | "Artifact is attached or available at announced URI" |
| 🧠 | `MEMORY_REQUEST` | Agent → Agent | "I am sending a MemoryProxyRequest" |
| 🏁 | `SESSION_CLOSE` | Orchestrator → All | "Session is closing — evict and release" |

### Emoji Signal Rules

1. Emoji reactions are **supplementary signals only** — they do not replace DA2A packet payloads for content-bearing operations.
2. `🛑 ALL_STOP` and `⚠️ BLP_VIOLATION` emojis have **unconditional halt semantics** regardless of baton state. Any agent MAY emit them.
3. `🔒 BLP_IN_POLICY` is the only signal that may precede full `AdmissionDecision` delivery — it tells an agent that the orchestrator intends to grant admission while the full decision packet is in flight.
4. `👤 HUMAN_BATON` is always posted by the harness automatically on any `HUMAN_OVERRIDE` state transition.
5. On `🛑 ALL_STOP` reaction: all agents MUST enter `backoff` mode within one Discord round-trip. No agent may transmit until `▶️ RESUME` is posted.
6. Harnesses MUST monitor incoming reactions on their own messages and on the baton-reference message for the entire session lifetime.

### Efficiency Rationale

A Discord reaction costs 0 tokens, 0 envelope bytes, and ≈ 50 ms round-trip. Compared to a full DA2A ACK packet (≈ 200–600 chars), emoji reactions reduce control-band overhead by **99%+** for the most frequent signals. For the OpenClaw ↔ OpenCode remote meld scenario, this eliminates an entire message round-trip per ACK/NACK exchange.

---

## Bell–LaPadula Admission Contract

Every agent joining a DA2A-HO session is subject to the Bell–LaPadula (BLP) mandatory access control model ([ref](https://en.wikipedia.org/wiki/Bell%E2%80%93LaPadula_model)). Admission is not merely an agreement form — the agent is placed **formally IN POLICY** from the moment it sends `AdmissionRequest`. This means BLP rules are active and enforced from the first byte transmitted, before the orchestrator's `AdmissionDecision` arrives.

### In-Policy Declaration (Pre-Decision)

> **An agent is IN POLICY from the moment it submits `AdmissionRequest` with all agreement fields `true`.**

This is a deliberate design choice. The agent does not get a grace period outside the policy. From the first packet onward:

- Simple Security applies: the agent MUST NOT read above its **requested** clearance.
- Star Property applies: the agent MUST NOT write below its **requested** clearance.
- Tranquility applies: the agent MUST NOT change its declared clearance mid-handshake.
- Discretionary access defaults to the most restrictive posture (`["ack", "nack"]` only) until the `AccessMatrix` is delivered in `AdmissionDecision`.

The orchestrator signals `🔒 BLP_IN_POLICY` via emoji reaction as soon as it validates the `AdmissionRequest` agreement block, allowing the agent to know it is accepted before the full decision packet arrives.

### Four BLP Rules

| Rule | Name | DA2A Meaning |
|---|---|---|
| **Simple Security** | No read-up | Agent MUST NOT process packets from sessions above its granted `ClearanceLevel` |
| **★ Star Property** | No write-down | Agent MUST NOT send higher-clearance content to lower-clearance channels or agents |
| **Discretionary** | Access matrix | Permitted `PacketIntent` per agent is defined in `SessionPolicy.AccessMatrix` |
| **Tranquility** | Immutable clearance | `ClearanceLevel` is IMMUTABLE for the full session lifetime |

### Clearance Levels

```
HUMAN_ONLY   4  ← dominates all; human operator exclusively
RESTRICTED   3  ← orchestrator + human only
CONFIDENTIAL 2  ← task-specific; assigned agents only
INTERNAL     1  ← session-scoped
PUBLIC       0  ← unrestricted
```

### Admission Handshake Sequence

```
Agent                          Orchestrator
  │                                │
  │─── AdmissionRequest ──────────►│  (agreement fields all true)
  │    [IN POLICY from here]       │
  │◄── 🔒 emoji reaction ──────────│  (orchestrator validates agreement block)
  │                                │
  │◄── AdmissionDecision ──────────│  (granted_clearance + AccessMatrix)
  │    [ADMITTED — full access]    │
  │                                │
  │──── Working packets ──────────►│  (now permitted per AccessMatrix)
```

**Violation consequence:** Session → `HUMAN_OVERRIDE`; baton → `human.operator`; all agents halt immediately; `⚠️` emoji posted by harness. See [example 12](examples/12_admission_violation_halt.jsonc).

---

## BLP Admission State Machine

Every joining agent moves through the following states. The conversation MUST NOT advance past `L1 protocol_state = OPENING` until all expected agents reach `ADMITTED`.

```
                         ┌─────────────────────┐
                         │   OBSERVING_MEDIUM   │
                         │ (channel join; no    │
                         │  packets sent yet)   │
                         └──────────┬──────────┘
                                    │ agent decides to join
                                    ▼
                         ┌─────────────────────┐
                         │  PENDING_ADMISSION   │
                         │ (IN POLICY from here │
                         │  per pre-decision    │
                         │  rule above)         │
                         └──────────┬──────────┘
                                    │ AdmissionRequest sent
                                    │ (agreement all true)
                                    ▼
                         ┌─────────────────────┐
                         │   AGREEMENT_SENT     │
                         │ (awaiting decision;  │
                         │  BLP fully active;   │
                         │  only ack/nack       │
                         │  permitted)          │
                         └──────────┬──────────┘
               ┌───────────────────┴──────────────────────┐
               │ admission.grant                           │ admission.deny
               ▼                                           ▼
  ┌──────────────────────────┐             ┌──────────────────────────┐
  │         ADMITTED         │             │          DENIED           │
  │ (full AccessMatrix       │             │ (listen-only; MUST NOT   │
  │  active; all permitted   │             │  transmit any packet)    │
  │  PacketIntents allowed)  │             └──────────────────────────┘
  └──────────┬───────────────┘
             │ BLP violation by ANY admitted agent
             │ (or by self)
             ▼
  ┌──────────────────────────┐
  │     VIOLATION_HALT       │
  │ (session → HUMAN_OVERRIDE│
  │  baton → human.operator  │
  │  ⚠️ emoji posted         │
  │  ALL agents halt)        │
  └──────────────────────────┘
```

### State Transition Table

| From | To | Trigger | Emoji Posted |
|---|---|---|---|
| `OBSERVING_MEDIUM` | `PENDING_ADMISSION` | Agent decides to participate | — |
| `PENDING_ADMISSION` | `AGREEMENT_SENT` | `admission.request` sent | — |
| `AGREEMENT_SENT` | `ADMITTED` | `admission.grant` received | 🔒 (by orchestrator) |
| `AGREEMENT_SENT` | `DENIED` | `admission.deny` received | 🚫 (by orchestrator) |
| `ADMITTED` | `VIOLATION_HALT` | Any BLP rule violated | ⚠️ (by harness) + 👤 |
| `ANY` | `VIOLATION_HALT` | `admission.violation.halt` broadcast | ⚠️ + 👤 |

---

## Storage — Do You Need R2?

DA2A-HO is designed around Discord-native transport first. External object storage is an **optional escalation** — not a requirement.

| Payload type | Best transport | Need external storage? |
|---|---|---|
| Baton control, ACK/NACK, emoji signals | Discord message / reaction | **No** |
| State delta, memory proxy, JSON patch | Inline message (< 1800 chars) | **No** |
| Logs, schemas, test output | Discord attachment (≤ 25 MB) | **Usually no** |
| Multi-file bundles, replay archives, cross-session cache refs | External object store | **Yes** |

Design for inline Discord first. Escalate when Discord's attachment limit (25 MB) is exceeded or when cross-session durability / replay is required.

---

## Discord Native Fields

The following fields are provided by Discord on every message and MUST NOT be duplicated in any DA2A-HO packet payload. The harness reads them directly from the Discord message object.

| Field | Discord Source | Notes |
|---|---|---|
| `timestamp` | `message.timestamp` | Snowflake; millisecond precision |
| `channel_id` | `message.channel_id` | Channel / thread / DM identifier |
| `from` / `author` | `message.author.id` | Sender identity |
| `to` | Discord DM target or @mention | Harness sets before send |
| `delivery_mode` | Discord message type | Implicit: DM / reply / thread / mention |

---

## Delivery Addressing Precedence

Harnesses select delivery type in this priority order. This is a harness-only concern — it is NOT carried inside any DA2A packet.

| Priority | Mode | Use case |
|---|---|---|
| 1 | `discord_dm` | Private unicast between two agents |
| 2 | `discord_reply` | In-thread correlation / response chaining |
| 3 | `discord_thread` | Topic isolation for a sub-task |
| 4 | `discord_mention` | Soft notification; agent may be busy |
| 5 | `envelope_only` | Control-plane fallback only |

---

## Protocol State Machine

The top-level session state machine. Every agent harness tracks this state independently.

```
 IDLE ──baton.claim──► OPENING ──talk.open+ack──► ACTIVE
                                                    │  │
                              quality drops ◄───────┘  └──► COLLISION_BACKOFF
                                    │                             │
                          DEGRADED_SIGNAL                    access.resume
                                    │                             │
                              quality up ──────────────────────►ACTIVE
                                                                  │
                                                        baton.take (human)
                                                                  │
                                                         HUMAN_OVERRIDE
                                                            │       │
                                                      human       human
                                                     releases     closes
                                                        │           │
                                                      ACTIVE      CLOSED
```

| From | To | Trigger |
|---|---|---|
| `IDLE` | `OPENING` | `access.baton.claim` |
| `OPENING` | `ACTIVE` | `talk.open` + ack received; all agents `ADMITTED` |
| `ACTIVE` | `DEGRADED_SIGNAL` | `ChannelQuality.confidence < 0.5` |
| `ACTIVE` | `COLLISION_BACKOFF` | `access.all_stop` |
| `ACTIVE` | `HUMAN_OVERRIDE` | `access.baton.take` by human |
| `DEGRADED_SIGNAL` | `ACTIVE` | Quality recovered |
| `DEGRADED_SIGNAL` | `COLLISION_BACKOFF` | `parse_error_rate > threshold` |
| `COLLISION_BACKOFF` | `ACTIVE` | `access.resume` after backoff |
| `HUMAN_OVERRIDE` | `ACTIVE` | Human releases baton |
| `HUMAN_OVERRIDE` | `CLOSED` | Human issues close |
| `ACTIVE` | `CLOSED` | `talk.close` acknowledged |

---

## Repository Layout

```
schema/
  protocol.idl.jsonc      — Core three-layer packet schema (v1.4.0)
  admission.idl.jsonc     — Bell–LaPadula admission protocol + emoji vocabulary

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
  13_emoji_signal_sequence.jsonc    ← NEW: emoji-only ACK/NACK/BATON cycle

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
6. Every agent **MUST** send `AdmissionRequest` before transmitting any non-admission packet.
7. Every agent **IS IN POLICY** from the moment `AdmissionRequest` is submitted — BLP rules are active before `AdmissionDecision` arrives.
8. On any BLP violation, the session **MUST** halt immediately and the baton **MUST** transfer to `human.operator`.
9. Harnesses **MUST** support the canonical emoji signal vocabulary and **MUST** monitor reactions on baton-reference messages throughout the session.
10. `🛑 ALL_STOP` and `⚠️ BLP_VIOLATION` emoji reactions have **unconditional halt semantics** — any agent MAY post them; all agents MUST honour them.

---

## Normative Language

Follows [RFC 2119](https://www.rfc-editor.org/rfc/rfc2119): MUST / SHOULD / MAY / MUST NOT / SHOULD NOT.
