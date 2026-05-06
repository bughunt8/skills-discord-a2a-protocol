# AGENTS.md — DA2A Protocol Repository

> Instructions for AI agents (Claude Code, OpenClaw, GitHub Copilot, Codex, Cursor, etc.)
> working in the `skills-discord-a2a-protocol` repository.

---

## 📋 Project Overview

This repository defines the **DA2A (Discord Agent-to-Agent) Communication Protocol** — a typed,
IDL + JSON-RPC 2.0 message protocol for multi-agent systems communicating over Discord channels.

The repo contains:
- `schema/protocol.idl.jsonc` — canonical type system (structs, interfaces, enums)
- `examples/` — annotated wire-format example messages (`.jsonc`)
- `docs/discord-permissions.jsonc` — Discord bot intent and permission specifications
- `README.md` — human-facing documentation
- `AGENTS.md` — this file

---

## 🧱 Architecture Invariants

Before making any changes, understand these constraints:

1. **Every message on the wire is a `MessageEnvelope`** — the outermost object.
2. **`payload` inside `MessageEnvelope` is always a `RpcFrame`** — strict JSON-RPC 2.0.
3. **`RpcFrame.id` presence/absence determines message type:**
   - `id` present + `method` present = Request
   - `id` present + `result`/`error` present = Response
   - `id` absent + `method` present = Notification (fire-and-forget)
4. **All types are defined in `schema/protocol.idl.jsonc`** — do NOT invent new fields without updating the IDL first.
5. **JSONC format** — `.jsonc` files may contain `//` and `/* */` comments. Never put comments inside string values.
6. **Discord-native addressing (v1.1.0)** — `MessageEnvelope.to` and `.reply_to_msg` are SEMANTIC METADATA only. Discord DM > reply() > @mention > envelope fallback. See SECTION 0 in the IDL.

---

## ✅ Before You Make Changes

- [ ] Read `schema/protocol.idl.jsonc` fully — understand the type system before touching examples
- [ ] Confirm the type or field you need doesn't already exist in the IDL
- [ ] Verify any new field fits within Discord transport constraints
- [ ] Ensure `required: true` fields are present in every example using that struct
- [ ] For new delivery-related fields: follow the `addressing_precedence` order in `transport_constraints`

---

## 📝 Commit Conventions

Use **conventional commits**:

```
feat: add <thing>
fix: correct <thing>
docs: update <section>
schema: add/modify IDL type <TypeName>
example: add/update example <NN_name.jsonc>
chore: <housekeeping>
```

**Attribution on AI-assisted commits:**
```
Assisted-by: <TOOLNAME> (<MODELNAME>)
```
Do NOT add `Signed-off-by` on auto-generated commits — human review required first.

---

## 🗂️ File Conventions

### `schema/protocol.idl.jsonc`
- New **types** → `"types"` object (primitive aliases only)
- New **structs** → `"structs"` with sequential `"@id"` integers (currently 1–13)
- New **methods** → appropriate interface, following `da2a.<interface>.<verb>` naming
- Every field MUST have `"required": true` or `"required": false`
- Every struct/method MUST have `"@doc": "..."` annotation
- Notifications MUST have `"@notification": true`; no `"returns"` field
- Idempotent methods MUST have `"@idempotent": true`

### `examples/` files
- Naming: `NN_snake_case.jsonc` (zero-padded, currently up to 10)
- Required top comment block:
  ```
  // Example NN: <Title>
  // Interface: da2a.<interface>.<method>
  // Direction: <SenderBot> --> <ReceiverBot or "broadcast">
  ```
- Notifications: add comment `// No "id" field — Notification pattern`
- Use fake Snowflakes (18-digit numbers) and UUID pattern `550e8400-e29b-41d4-a716-44665544XXXX`
- Always populate `delivery.mode` in v1.1.0+ examples

---

## 🔍 Validation Checklist

When adding/modifying a **struct**:
- [ ] `@id` is unique and sequential
- [ ] `@doc` is meaningful
- [ ] All fields have `required` set
- [ ] Optional fields document defaults in `@doc`
- [ ] If used in `RpcFrame.params`, the interface method references it

When adding/modifying an **interface method**:
- [ ] `@id` unique within interface
- [ ] Method name follows `da2a.<interface>.<verb>`
- [ ] `params` references existing struct types only
- [ ] Notifications have no `returns`
- [ ] Corresponding example file exists

---

## 🚫 Do NOT

- Add new top-level keys to `MessageEnvelope` without IDL update
- Use trailing commas in `.jsonc` files
- Put JSONC comments inside string values
- Invent error codes outside `-32000` to `-32099`
- Change existing `@id` integers (they are stable references)
- Commit secrets, real tokens, or real Discord Snowflakes
- Treat `envelope.to` as a routing instruction — it is metadata only
- Skip populating `delivery.mode` in v1.1.0+ examples

---

## 🧪 Validation

```bash
# Strip JSONC comments and validate all files parse as JSON
python3 -c "
import re, json, sys, pathlib
for f in pathlib.Path('.').rglob('*.jsonc'):
    src = re.sub(r'//[^\n]*', '', f.read_text())
    src = re.sub(r'/\*.*?\*/', '', src, flags=re.DOTALL)
    try: json.loads(src); print(f'OK: {f}')
    except Exception as e: print(f'FAIL: {f}: {e}'); sys.exit(1)
"
```

---

## 🔗 Key References

- [JSON-RPC 2.0](https://www.jsonrpc.org/specification)
- [Discord Gateway Intents](https://discord.com/developers/docs/topics/gateway)
- [Discord Permissions](https://discord.com/developers/docs/topics/permissions)
- [A2A Protocol](https://a2a-protocol.org)
- [RFC 7396 JSON Merge Patch](https://datatracker.ietf.org/doc/html/rfc7396)
- [AGENTS.md standard](https://agents.md)

---

*DA2A Protocol v1.1.0 — AGENTS.md maintained by Profit Rise Consulting*
