# SOVEREIGN CONTRACT

**Version:** 1.0
**Authority:** KilimanjaroCode organization
**Status:** Active
**Last updated:** 2026-03-16

---

## 1. Purpose and Authority

This document is the **canonical interface contract** for the KilimanjaroCode sovereign stack. It defines the schemas, invariants, sentinels, and change rules that govern communication between the three layers of the stack:

| Layer | Repo | Role |
|---|---|---|
| Constitutional substrate | `KilimanjaroCode/sifiso-os` | Sovereign Gate, PKL, GateLog, MANCE, identity |
| Cognition engine | `KilimanjaroCode/uigen` | Component generation, lineage, semantic deltas, governance |
| Presence layer | `backend/ikhaya/` (inside sifiso-os) | Narrative, interpretation, multi-agent deliberation |

**Rule:** Both repos treat this document as the single source of truth. Any schema change — new field, renamed field, type change, new endpoint — **must update this contract first** via a PR to this repository. The implementing PR in the affected repo(s) must reference the contract PR.

---

## 2. Versioning

Contract versions follow `MAJOR.MINOR`:

- **MAJOR** — breaking change (field removed, type changed, endpoint removed, sentinel changed)
- **MINOR** — additive change (new optional field, new endpoint, new sentinel, clarification)

Both `sifiso-os` and `uigen` pin the contract version they comply with in their respective `CLAUDE.md` or `README.md`. A version mismatch between repos is a governance event.

---

## 3. Agent Identity

All actors in the sovereign stack are identified by a **Decentralized Identifier (DID)**.

### 3.1 DID Format

```
did:key:z{hex_public_key}
```

- Prefix: `did:key:z` (required; validated by Sovereign Gate predicate #1)
- Suffix: Ed25519 public key encoded as lowercase hex
- Example: `did:key:z4a9b3c8d...`

### 3.2 System DID

UIGen uses a system-level DID for actions not attributed to a human agent:

```
UIGEN_SYSTEM_DID = "did:key:z" + "uigen"
```

Configurable via `UIGEN_SYSTEM_DID` environment variable.

### 3.3 Sentinels

| Sentinel | Value | Meaning |
|---|---|---|
| `BYPASSED_LOG_ID` | `"0" * 64` | Gate was not contacted; score is estimated, not validated |

---

## 4. Ubuntu Alignment Score

The Ubuntu alignment score (U) is computed by `AxiomEngine` in `sifiso-os`.

### 4.1 Formula

```
U = w_H·H + w_S·S + w_C·C + w_E·E + β·min(E, 1 − d_eff) − γ·harm
```

### 4.2 Dimension Schema

| Field | Type | Range | Meaning |
|---|---|---|---|
| `H` | float | 0.0–1.0 | Harmony — accessibility, inclusivity, human alignment |
| `S` | float | 0.0–1.0 | Stewardship — quality, evaluation, responsible change |
| `C` | float | 0.0–1.0 | Context — agent trust, semantic richness, cultural fit |
| `E` | float | 0.0–1.0 | Efficiency-over-Speed — human approval, measured change |
| `harm` | float | 0.0–0.5 | Penalty for accessibility regression |
| `efficiency_deviation` | float | 0.0–0.4 | Penalty for excessive change volume |

### 4.3 Default Manifest Parameters

```json
{
  "weights": { "H": 0.35, "S": 0.30, "C": 0.20, "E": 0.15 },
  "thresholds": { "theta": 0.65, "beta": 0.12, "gamma": 0.18 }
}
```

`theta` (0.65) is the minimum alignment score required for a gate decision of `passed: true`.

---

## 5. Sovereign Gate Contract

### 5.1 Endpoint: `POST /api/gate/check`

Direct gate evaluation for any action type.

**Request**

```typescript
{
  action_type:   string;          // non-empty; one of ActionType enum values
  payload:       Record<string, unknown>;
  initiator_did: string;          // must match /^did:key:z/
  timestamp:     number;          // Unix epoch (int or float)
  nonce?:        string;
  signature?:    string;          // required for GOVERNANCE actions (Ed25519 hex)
}
```

**Response**

```typescript
{
  passed:            boolean;
  ubuntu_score:      number;        // 0.0–1.0
  failed_predicates: string[];      // subset of the 7 predicate names below
  rationale:         string;
  log_entry_id:      string;        // 64-char SHA-256 hex, or BYPASSED_LOG_ID
}
```

**Predicates (evaluated in order)**

| # | Name | Trigger |
|---|---|---|
| 1 | `Schema Validity` | Missing fields, wrong types, invalid DID prefix |
| 2 | `Human Signature` | GOVERNANCE action missing or invalid Ed25519 signature |
| 3 | `Zero-Knowledge Boundary` | MESH/GOVERNANCE payload contains PII/secrets |
| 4 | `Ubuntu Alignment` | `ubuntu_score < theta` |
| 5 | `Determinism` | Non-deterministic payload keys present; GOVERNANCE missing `deterministic: true` |
| 6 | `Invariant Preservation` | `disable_gate`, `bypass_gate`, `skip_gate`, `disable_sovereignty`; or PII in GOVERNANCE/MESH |
| 7 | `Log Append Success` | Merkle-rooted audit log write failed |

**Non-deterministic payload keys (predicate #5)**

```
random_seed, random, random_choice, time_condition,
current_time_check, non_deterministic, external_call,
external_url, live_data
```

**Gate bypass keys (predicate #6 — gate_mandatory invariant)**

```
disable_gate, bypass_gate, skip_gate, disable_sovereignty
```

**Sensitive terms (predicate #6 — zero_knowledge_boundary invariant)**

```
password, secret, private_key, email, phone,
ssn, biometric, location, personal_data
```

---

### 5.2 Endpoint: `POST /api/uigen/gate-check`

UIGen-specific constitutional pre-flight. Translates UIGen action signals into Ubuntu dimensions via `UIGenAxiomScorer`, then evaluates through the full Sovereign Gate.

**Request**

```typescript
{
  uigen_action:            "artifact_publish" | "variant_approve" | "variant_publish";
  initiator_did:           string;            // defaults to UIGEN_SYSTEM_DID
  artifact_name?:          string;
  artifact_description?:   string;
  semantic_summary?:       string;
  tags?:                   string[];
  agent_name?:             string;
  agent_reputation_score?: number;            // 0.0–1.0
  human_approved?:         boolean;
  evaluation_passed?:      boolean;
  a11y_delta?:             number;            // positive = improvement, negative = regression
  lines_changed?:          number;
  added_components?:       string[];
  removed_components?:     string[];
}
```

**Response**

All fields from `/api/gate/check` response, plus:

```typescript
{
  // ...gate check fields...
  dimensions: {
    H:                    number;   // 0.0–1.0
    S:                    number;
    C:                    number;
    E:                    number;
    harm:                 number;   // 0.0–0.5
    efficiency_deviation: number;   // 0.0–0.4
  };
  bypassed?: boolean;               // true when SIFISO_OS_URL not configured
}
```

**Fail-open behaviour**

When `SIFISO_OS_URL` is not configured, UIGen's `checkWithSovereignGate()` returns:

```typescript
{
  passed:            true,
  ubuntu_score:      0.82,
  failed_predicates: [],
  rationale:         "Gate bypassed — SIFISO_OS_URL not configured (development mode).",
  log_entry_id:      "0".repeat(64),   // BYPASSED_LOG_ID
  bypassed:          true
}
```

---

## 6. iKhaya AI API Contract

### 6.1 Tone Enum

All narrate responses include a `tone` field:

| Value | Condition |
|---|---|
| `"affirming"` | `ubuntu_score >= 0.75` |
| `"cautionary"` | `0.65 <= ubuntu_score < 0.75` |
| `"warning"` | `ubuntu_score < 0.65` |
| `"bypassed"` | `gate_log_entry_id == BYPASSED_LOG_ID` |

### 6.2 Endpoint: `POST /api/ikhaya/narrate`

Produces a Ubuntu-grounded narrative of a gate decision. Reads PKL trust scores and ikhaya_memory context to adapt tone and framing.

**Request**

```typescript
{
  artifact_id:       string;
  variant_id?:       string;
  ubuntu_score:      number;        // 0.0–1.0
  gate_log_entry_id: string;        // 64-char hex, or BYPASSED_LOG_ID
  semantic_delta?: {
    a11yDelta?:         number;     // positive = improvement
    linesChanged?:      number;
    addedComponents?:   string[];
    removedComponents?: string[];
  };
  agent_did?:        string;        // DID format; used to read PKL trust score
}
```

**Response**

```typescript
{
  narrative:   string;
  confidence:  number;              // 0.0–1.0
  tone:        "affirming" | "cautionary" | "warning" | "bypassed";
  references:  string[];            // e.g. ["gate:abcdef123456789…"]
}
```

**Side effects**

After every call, one `narrate_event` observation is written to `ikhaya_memory` in PKL (see §8.2).

---

### 6.3 Endpoint: `POST /api/ikhaya/interpret-lineage`

Explains what changed across an artifact's generational lineage and why it matters.

**Request**

```typescript
{
  artifact_id:          string;
  from_version?:        string;
  to_version?:          string;
  lineage_graph?:       object[];   // ancestor nodes from UIGen lineage walker
  semantic_transforms?: object[];   // SemanticTransform records from UIGen
}
```

**Response**

```typescript
{
  summary:                string;
  change_list:            string[];
  reasoning_narrative:    string;
  ubuntu_alignment_notes: string;
}
```

**Side effects**

After every call, one `lineage_event` observation is written to `ikhaya_memory` (see §8.3).

---

### 6.4 Endpoint: `POST /api/ikhaya/mance-deliberate`

Facilitates a Multi-Agent Network for Collective Evaluation (MANCE) deliberation. Reads PKL trust scores for all agent DIDs. Returns a structured account of the council's reasoning.

**Request**

```typescript
{
  agents:   { did: string; name?: string }[];
  prompt:   string;
  context?: object;
}
```

**Response**

```typescript
{
  deliberation_transcript: { agent: string; statement: string }[];
  consensus_summary:       string;
  dissenting_views:        string[];
  recommended_action:      string;
}
```

---

## 7. PKL Memory Schemas

All values stored in PKL are **Fernet-encrypted** (AES-128-CBC with HMAC-SHA-256). Raw SQLite values are never plain JSON.

### 7.1 HPM — Human Preference Model (`hpm` table)

```typescript
{
  key:       string;    // PRIMARY KEY
  value:     any;       // encrypted JSON
  category:  string;    // e.g. "ui", "language", "preference"
  timestamp: number;    // Unix epoch
  source:    string;    // "explicit" | "inferred"
}
```

### 7.2 Trust Scores (`trust_scores` table)

```typescript
{
  did:          string;   // PRIMARY KEY — DID format
  score:        number;   // 0.0–1.0; default 0.5 for unknown DID
  last_updated: number;   // Unix epoch
}
```

### 7.3 iKhaya Memory (`ikhaya_memory` table)

Append-only. Pruned to `IKHAYA_MEMORY_WINDOW = 20` most recent entries after every write.

```typescript
{
  id:        number;    // AUTO — used for ordering and pruning
  key:       string;    // observation type label
  value:     any;       // encrypted JSON — see observation schemas below
  category:  string;    // "narrate" | "lineage" | "critique" | "mance" | "preference" | "general"
  timestamp: number;    // Unix epoch
}
```

#### 7.3.1 `narrate_event` observation (category: `"narrate"`)

Written automatically after every `/api/ikhaya/narrate` call.

```typescript
{
  artifact_id: string;
  ubuntu_score: number;
  tone:        "affirming" | "cautionary" | "warning" | "bypassed";
  bypassed:    boolean;
  has_delta:   boolean;
  agent_did:   string | null;
  timestamp:   number;
}
```

#### 7.3.2 `lineage_event` observation (category: `"lineage"`)

Written automatically after every `/api/ikhaya/interpret-lineage` call.

```typescript
{
  artifact_id:   string;
  generations:   number;
  a11y_trend:    number;   // net a11y delta across all transforms
  added_count:   number;   // unique components added
  removed_count: number;   // unique components removed
  timestamp:     number;
}
```

#### 7.3.3 `critique_event` observation (category: `"critique"`) — *reserved for Phase 29*

```typescript
{
  artifact_id:   string;
  agent_did:     string;
  suggestion_id: string;
  accepted:      boolean;
  score:         number;   // relevance/safety/impact composite
  timestamp:     number;
}
```

#### 7.3.4 Custom observations

External callers may use `write_observation(key, value, category)` to store arbitrary encrypted observations. No schema is enforced beyond the ikhaya_memory table structure. Use the `"preference"` category for Steward preference data.

---

## 8. Governance Event Details Schemas

`GovernanceEvent.details` in UIGen's SQLite database is stored as JSON. The schemas below define what each event type carries.

### Gate-aware events (carry `ubuntuScore` and `gateLogEntryId`)

```typescript
// ARTIFACT_PUBLISHED
{
  ubuntuScore?:     number;   // from gate result
  gateLogEntryId?:  string;   // 64-char hex, or BYPASSED_LOG_ID
  artifactId?:      string;
}

// ARTIFACT_VARIANT_APPROVED
{
  ubuntuScore?:     number;
  gateLogEntryId?:  string;
  approved:         boolean;
  agentName?:       string;
}

// ARTIFACT_VARIANT_PUBLISHED
{
  ubuntuScore?:     number;
  gateLogEntryId?:  string;
  parentArtifactId?: string;
}

// ARTIFACT_VARIANT_REJECTED
{
  ubuntuScore?:     number;
  gateLogEntryId?:  string;
  reason?:          string;
}
```

### Policy events

```typescript
// BRANCH_POLICY_SET | BRANCH_POLICY_CHANGED
{
  branchName:  string;
  policyType:  "OPEN" | "AI_ONLY" | "HUMAN_ONLY" | "LOCKED";
}
```

### Semantic transform events

```typescript
// SEMANTIC_TRANSFORM_RECORDED
{
  fromArtifactId:    string;
  toArtifactId:      string;
  addedComponents:   string[];
  removedComponents: string[];
  modifiedProps:     string[];
  a11yDelta:         number;
  linesChanged:      number;
}
```

---

## 9. SemanticDelta Schema

Produced by `computeSemanticDelta()` in UIGen and passed to both the gate check and iKhaya narrate.

```typescript
{
  fromArtifactId:    string;
  toArtifactId:      string;
  addedComponents:   string[];
  removedComponents: string[];
  modifiedProps:     string[];
  a11yDelta:         number;   // positive = improvement, negative = regression
  linesChanged:      number;
}
```

---

## 10. Change Protocol

1. **Contract-first:** Any schema change must be proposed as a PR to `KilimanjaroCode/sovereign-contract` before implementation begins in either repo.

2. **Reference:** The implementing PR in `sifiso-os` or `uigen` must reference the contract PR (e.g. `KilimanjaroCode/sovereign-contract#N`).

3. **Simultaneous update:** Breaking changes must update both affected repos in coordinated PRs merged in the same session.

4. **Version bump:** Any PR to this document must increment the version in the header — MINOR for additive, MAJOR for breaking.

5. **Reserved fields:** Fields marked *reserved for Phase N* define the intended schema for upcoming phases. Implementing repos must not use these field names for other purposes.

6. **Fail-open invariant:** All cross-repo clients must implement fail-open behaviour. A network failure calling Sifiso OS must never block UIGen. A network failure calling UIGen must never block iKhaya AI. Fail-open stubs must conform to the response schemas above.

7. **Encryption invariant:** All values written to PKL — HPM, trust scores, ikhaya_memory — must be Fernet-encrypted. Plain-text PKL writes are a constitutional violation.

---

## 11. Environment Variables

| Variable | System | Required | Purpose |
|---|---|---|---|
| `PKL_ENCRYPTION_KEY` | sifiso-os | Yes | Fernet key for PKL encryption (44-char URL-safe base64) |
| `ANTHROPIC_API_KEY` | sifiso-os | No | Enables Claude-powered narrative in iKhaya (offline templates used otherwise) |
| `SIFISO_OS_URL` | uigen | No | Base URL of Sifiso OS backend; omit for development (fail-open) |
| `UIGEN_SYSTEM_DID` | uigen | No | DID for UIGen system actions; defaults to `did:key:zuigen` |
| `JWT_SECRET` | uigen | No | Signs session JWTs; defaults to `development-secret-key` |

---

*This contract is the constitution of the interfaces. It is not documentation as an afterthought — it is the written law of the sovereign stack.*
