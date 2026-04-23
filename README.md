# gate-contracts

**GATE JSON Schema contracts - v1.0.0**

Normative JSON Schema (Draft 2020-12) definitions for all GATE control plane
contracts. This is the canonical dependency for GATE implementations -
the tool gateway, evidence pipeline, replay harness, and conformance tooling
all reference these schemas.

Framework: https://deterministicagents.ai  
Organisation: https://github.com/deterministic-agents  
Documentation: CC BY 4.0 — Andrew Stevens · Code: MIT

---

## Contents

| File | What it defines |
|---|---|
| `canonical_json.md` | **Read this first.** Normative canonicalisation rules for all hash computation (RFC 8785). Includes Python, Node.js, and Go implementations. |
| `tool_envelope.schema.json` | `ToolRequestEnvelope` and `ToolResponseEnvelope` - the primary wire format for tool calls traversing the Tool Gateway |
| `policy_decision_record.schema.json` | `PolicyDecisionRecord` - the must-have evidence artifact emitted for every tool attempt |
| `audit_ledger_event.schema.json` | `LedgerEvent` - hash-chained, tamper-evident event committed to WORM storage |
| `replay_trace.schema.json` | `ReplayTrace` - captures all non-determinism at the tool/memory boundary for deterministic replay |
| `hitl_decision_record.schema.json` | `HITLDecisionRecord` - signed human approval record for HITL gates |
| `multi_agent_envelope.schema.json` | `AgentMessageEnvelope` - signed, nonce-protected agent-to-agent message |
| `control_catalog.yaml` | Machine-readable C01–C16 control catalog with NIST AI RMF and ISO/IEC 42001 mappings |

---

## The core constraint

Every GATE contract event carries:
- **Correlation IDs** - `run_id`, `trace_id`, `agent_instance_id`, `tenant_id`
- **Bundle hashes** - `policy_bundle_hash`, `tool_schema_hash`, `prompt_bundle_hash`
- **Payload hashes** - `request_hash`, `response_hash` computed over canonical JSON

These fields create a verifiable evidence chain:

```
ToolRequest.request_hash
  → PolicyDecision.inputs.request_hash
  → LedgerEvent.references.tool_request_hash
  → ReplayTrace step.request_hash
```

Any break in this chain is detectable during conformance verification.

---

## Canonical JSON

All hashes MUST be computed over canonical JSON (RFC 8785): keys sorted by
Unicode code point, no whitespace, UTF-8 encoded.

See `canonical_json.md` for the normative rules and language implementations.
Do not use `json.dumps()`, `JSON.stringify()`, or `json.Marshal()` without
enforcing stable key ordering - this is the most common source of hash drift
across implementations.

---

## Validation

Using `ajv` (Node.js):
```bash
npm install -g ajv-cli
ajv validate -s tool_envelope.schema.json -d your_event.json
```

Using `jsonschema` (Python):
```python
import json, jsonschema
schema = json.load(open("tool_envelope.schema.json"))
event  = json.load(open("your_event.json"))
jsonschema.validate(event, schema)
```

---

## Versioning

Schema version is in the `$id` field of each schema and the `schema_version`
const in each contract event. Breaking changes increment the major version.
Additive changes (new optional fields) increment the minor version.

Downstream repos that bundle these schemas (e.g. `gate-python`) pin to a
specific release. See their `MANIFEST.yaml` for the pinned version and SHA256s.

---

## Related repos

| Repo | What it is |
|---|---|
| [gate-python](https://github.com/deterministic-agents/gate-python) | Python reference library (hashing, envelopes, ledger, replay, signing) |
| [gate-policies](https://github.com/deterministic-agents/gate-policies) | OPA/Rego baseline policy and invariant bundles |
| [gate-conformance](https://github.com/deterministic-agents/gate-conformance) | Conformance checks, self-assessment, runbooks |
| [gate](https://github.com/deterministic-agents/gate) | Framework paper, spec site source |
