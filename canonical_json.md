# Canonical JSON — GATE Normative Serialization Rules

**Status:** Normative  
**Applies to:** All hash computation and signature generation in GATE contracts

---

## Why canonicalization matters

GATE contracts use `request_hash`, `response_hash`, `policy_bundle_hash`, and
`event_hash` to create a verifiable evidence chain. If two components compute
a hash over the same logical data but serialize it differently, the hashes will
not match — breaking replay validation, ledger integrity checks, and signature
verification.

This is the most common real-world failure mode in hash-based enforcement
systems. Enforce canonicalization at the boundary, not as an afterthought.

---

## Normative rules

GATE canonical JSON follows RFC 8785 (JSON Canonicalization Scheme, JCS).

### 1. Key ordering
Object keys MUST be sorted by Unicode code point (lexicographic order).
This applies recursively to nested objects.

```json
// Non-canonical (unordered)
{"z": 1, "a": 2, "m": 3}

// Canonical (sorted)
{"a": 2, "m": 3, "z": 1}
```

### 2. String encoding
Strings MUST use UTF-8 encoding. Unicode escape sequences (`\uXXXX`) MUST
use uppercase hex. No unnecessary escaping.

### 3. Number serialization
Numbers MUST NOT use scientific notation unless the value requires it
(i.e., values representable as integers MUST be serialized as integers).
No trailing zeros after the decimal point.

```json
// Non-canonical
{"amount": 1000.00, "rate": 1e2}

// Canonical
{"amount": 1000, "rate": 100}
```

### 4. Whitespace
No insignificant whitespace (no spaces, newlines, or indentation).

### 5. Boolean and null
`true`, `false`, `null` — lowercase, no alternatives.

### 6. Arrays
Array element order is preserved (arrays are ordered by definition).

---

## Implementation notes by language

### Python
```python
import json

def canonical_json(obj: dict) -> bytes:
    """Produce canonical JSON bytes suitable for hashing/signing."""
    return json.dumps(
        obj,
        ensure_ascii=False,
        sort_keys=True,
        separators=(',', ':')
    ).encode('utf-8')

import hashlib

def gate_hash(obj: dict) -> str:
    """Compute sha256 over canonical JSON. Returns 'sha256:<hex>'."""
    canonical = canonical_json(obj)
    digest = hashlib.sha256(canonical).hexdigest()
    return f"sha256:{digest}"
```

### Node.js / TypeScript
```typescript
import { createHash } from 'crypto';

function canonicalJson(obj: unknown): string {
  if (obj === null || typeof obj !== 'object') {
    return JSON.stringify(obj);
  }
  if (Array.isArray(obj)) {
    return '[' + obj.map(canonicalJson).join(',') + ']';
  }
  const keys = Object.keys(obj as Record<string, unknown>).sort();
  const pairs = keys.map(k => {
    return JSON.stringify(k) + ':' + canonicalJson((obj as Record<string, unknown>)[k]);
  });
  return '{' + pairs.join(',') + '}';
}

function gateHash(obj: unknown): string {
  const canonical = canonicalJson(obj);
  const digest = createHash('sha256').update(canonical, 'utf8').digest('hex');
  return `sha256:${digest}`;
}
```

### Go
```go
import (
    "crypto/sha256"
    "encoding/json"
    "fmt"
)

// MarshalCanonical produces canonical JSON bytes.
// Use encoding/json with sorted keys (Go maps are unordered;
// use a struct or manually sort before marshaling).
func GateHash(v interface{}) (string, error) {
    b, err := json.Marshal(v)
    if err != nil {
        return "", err
    }
    // Note: encoding/json does NOT sort map keys by default.
    // Use structs with json tags, or a dedicated JCS library.
    sum := sha256.Sum256(b)
    return fmt.Sprintf("sha256:%x", sum), nil
}
```

> **Go note:** Use `github.com/cyberphone/json-canonicalization` or ensure
> you always serialize structs (not maps) so key order is deterministic.

---

## Fields that MUST be hashed over canonical JSON

| Field | What it covers |
|---|---|
| `request_hash` | Canonical ToolRequestEnvelope inputs object |
| `response_hash` | Canonical ToolResponseEnvelope outputs object |
| `event_hash` | Canonical LedgerEvent (excluding `hash_chain.event_hash` and the `signatures` block — the signature covers `event_hash`, so it cannot be hashed without circularity) |
| `prev_event_hash` | Hash of previous canonical LedgerEvent |
| `payload_hash` | Canonical tool input/output payload |
| `policy_bundle_hash` | SHA256 of the policy bundle archive (zip bytes, not JSON) |
| `tool_schema_hash` | SHA256 of the tool schema file bytes |
| `prompt_bundle_hash` | SHA256 of the prompt bundle archive bytes |

Bundle hashes (policy, tool schema, prompt) are computed over the raw file
bytes of the bundle archive, NOT over a JSON representation. Use `sha256sum`
or equivalent.

---

## Verification test vector

Use this object to verify your canonical serialization implementation:

Input:
```json
{"z": "last", "a": "first", "m": {"nested_z": 2, "nested_a": 1}}
```

Expected canonical form:
```
{"a":"first","m":{"nested_a":1,"nested_z":2},"z":"last"}
```

Expected SHA256 (hex):
```
sha256:5f6e3e2e9a0f2e1a3c4b5d6e7f8a9b0c1d2e3f4a5b6c7d8e9f0a1b2c3d4e5f6
```
> Note: compute this against your implementation and confirm it matches
> before deploying. The hex above is illustrative — run it yourself.
