# gate-contracts

**GATE JSON Schema contracts - v1.1.1**

Normative JSON Schema (Draft 2020-12) definitions for all GATE control plane
contracts. This is the canonical dependency for GATE implementations -
the tool gateway, evidence pipeline, replay harness, and conformance tooling
all reference these schemas.

Framework: https://deterministicagents.ai  
Organisation: https://github.com/deterministic-agents  
Documentation: CC BY 4.0 - Andrew Stevens · Code: MIT

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
| `control_catalog.yaml` | Machine-readable C01–C19 control catalog with NIST AI RMF and ISO/IEC 42001 mappings |

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

## v1.1.1 (2026-06-16)

First gate-contracts Release Object since v1.0.0. The v1.1.0 tag
carried the v1.3-compatible schema additions onto main but no Release
Object was ever cut, so the content was not downloadable as a release
artifact. v1.1.1 ships that content together with a README cleanup
that removed two lines of staging instruction text accidentally
committed in the v1.1.0 merge. Schema files at v1.1.1 are byte-
identical to v1.1.0; any downstream that pinned schema SHA256s
continues to work without changes.

Adds the six v1.3 control plane contracts and five new schemas for resources that previously lived implicitly inside other envelopes. Existing schemas are unchanged in shape; v1.3 fields are additive only.

### New event schemas

| File | Event type | Control |
|---|---|---|
| `agent_discovered.schema.json` | `gate.discovery.agent_discovered` | C17 |
| `agent_remediation_outcome.schema.json` | `gate.discovery.agent_remediation_outcome` | C17 |
| `quality_decision.schema.json` | `gate.memory.quality_decision` | C18 |
| `drift_decision.schema.json` | `gate.assurance.drift_decision` | C19 |
| `response_action.schema.json` | `gate.assurance.response_action` | C19 |

### New resource schemas

| File | Used by |
|---|---|
| `agent_state.schema.json` | C04 lifecycle (Discovered entry state added in v1.3) |
| `memory_item.schema.json` | C18 quality gate input (content_class, provenance, confidence_score) |
| `memory_request.schema.json` | Memory Gateway request envelope (optional `quality_override`) |
| `memory_response.schema.json` | Memory Gateway response envelope (`quality_flags`, `quality_decision_id`) |
| `abom.schema.json` | C19 baseline binding (`current_baseline_hash`) and C18 bundle binding (`memory_quality_bundle_hash`) |
| `behavioural_baseline.schema.json` | C19 signed baseline artifact (not an event) |

### Boundary rule

`gate.assurance.drift_decision` (C19) and `gate.assurance.adversarial_outcome` (C16) are distinct event types and MUST NOT be merged. C19 is continuous and statistical; C16 is event-driven and adversarial. The two ledger streams are read by separate runbooks.

### Migration notes

Implementations on v1.0.0 do not need to re-shape any existing event. v1.3 fields are additive:

- `memory_item` now requires `content_class`, `provenance_uri`, `provenance_hash`, `confidence_score`, and `created_at`. Implementations with stored items lacking these fields should either backfill or treat un-backfilled items as `unknown` content class (with the default action matrix applied by C18).
- `agent_state.state` enum gains `Discovered` and `Terminated`. Existing inventory entries remain valid.
- `abom` gains optional `current_baseline_hash` and `memory_quality_bundle_hash`. Required at bounded and high-privilege tiers per Check17 and Check18.
- `memory_response` gains `quality_flags` and `quality_decision_id`. Required for retrievals at bounded and high-privilege tiers per Check17.
- `memory_request` gains optional `quality_override` for the C18 break-glass path.

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
