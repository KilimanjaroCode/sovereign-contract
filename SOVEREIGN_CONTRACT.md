# SOVEREIGN CONTRACT

**Version:** 1.6
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
  uigen_action:            "artifact_publish" | "variant_approve" | "variant_publish" | "blueprint_submit";
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
  artifact_type?:          string;            // blueprint_submit — type of artifact (e.g. "ui-component")
  blueprint_description?:  string;            // blueprint_submit — human description
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

**Side effects**

After every call, one `mance_event` observation is written to `ikhaya_memory` (see §7.3.4). When `context.mergedSuggestions` is present, one `critique_event` observation is also written (see §7.3.3).

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

#### 7.3.3 `critique_event` observation (category: `"critique"`)

Written by `/api/ikhaya/mance-deliberate` when `context.mergedSuggestions` is present. Captures the aggregate result of a coordination round — one entry per critique session, not one per suggestion.

```typescript
{
  artifact_id:      string;
  suggestion_count: number;   // total merged suggestions in the round
  top_agent:        string;   // agent name of the reputation leader
  a11y_suggestion:  boolean;  // true if any suggestion addresses accessibility
  timestamp:        number;
}
```

#### 7.3.4 `mance_event` observation (category: `"mance"`)

Written by `/api/ikhaya/mance-deliberate` after every call, regardless of whether a critique context is present.

```typescript
{
  artifact_id:       string | null;   // from context.artifactId, if present
  agent_count:       number;
  prompt_prefix:     string;          // first 100 chars of the prompt
  consensus_reached: boolean;         // true unless recommended_action === "Pause for council reflection."
  recommended_action: string;
  timestamp:         number;
}
```

#### 7.3.5 Custom observations

External callers may use `write_observation(key, value, category)` to store arbitrary encrypted observations. No schema is enforced beyond the ikhaya_memory table structure. Use the `"preference"` category for Steward preference data.

#### 7.3.6 `forge_event` observation (category: `"forge"`)

Written by `ForgeEngine` via `presence.write_observation()` at construction-layer boundary events.

```typescript
{
  event_type:           "blueprint_submitted" | "gate_approved" | "gate_rejected"
                      | "build_started" | "build_complete" | "build_failed";
  blueprint_id:         string;
  requested_by:         string;            // DID of UIGen system node
  // conditional per event_type:
  artifact_type?:       string;            // blueprint_submitted
  grant_id?:            string;            // gate_approved, build_*
  ubuntu_score?:        number;            // gate_approved
  gate_log_entry_id?:   string;            // gate_approved, gate_rejected
  rationale?:           string;            // gate_rejected
  build_id?:            string;            // build_*
  state?:               string;            // build_*
  message?:             string;            // build_failed
  duration_ms?:         number;            // build_complete, build_failed
  merkle_root?:         string;            // build_complete
  timestamp:            number;            // Unix epoch
}
```

Key format written to ikhaya_memory: `f"forge:{event_type}:{blueprint_id}"`

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
| `FORGE_URL` | uigen | No | Forge API base URL; absent → fail-open stubs in `src/lib/forge.ts` |
| `FORGE_DID` | sifiso-os | No | Forge system DID for BuildGrant stubs; defaults to `did:key:zforge-system-node-dev` |

---

## 12. Blueprint & Build Protocol

The Forge is the **construction layer** of the sovereign stack — it accepts governance-approved blueprints from UIGen and assembles deterministic build artifacts. UIGen (cognition) and The Forge (construction) are constitutionally separated: UIGen never assembles artifacts; The Forge never generates UI.

### 12.1 Core Principles

1. **Separation of cognition and construction** — UIGen generates and evaluates; The Forge assembles. Neither crosses the boundary.
2. **Governance-first** — No `BuildRequest` may be submitted without a `BuildGrant` carrying a valid `grant_id` (the SHA-256 `log_entry_id` from the Sovereign Gate log). Gate bypass is a constitutional violation.
3. **Deterministic reproducibility** — Given the same `Blueprint` and `BuildGrant`, The Forge must produce the same `merkle_root`. Non-deterministic build inputs are rejected.
4. **Narrative transparency** — Every build boundary event (blueprint submitted, gate approved/rejected, build complete/failed) is narrated via iKhaya and written to `ikhaya_memory` as a `forge_event` observation (§7.3.6).
5. **Mesh-verifiable** — Build artifacts carry a `merkle_root` and `grant_id` so any Mesh node can independently verify provenance. (Mesh verification is Phase 32+.)

### 12.2 Data Models

```typescript
// Blueprint — the UI construction intent from UIGen
interface Blueprint {
  blueprint_id:      string;    // UIGen artifact or project identifier
  artifact_type:     string;    // e.g. "ui-component", "page", "dashboard"
  name:              string;
  description?:      string;
  components:        string[];  // component names from the UIGen registry
  files_hash?:       string;    // SHA-256 of all virtual file contents
  semantic_summary?: string;    // from artifact introspection
  requested_by:      string;    // DID of the UIGen system node
}

// BuildGrant — proof that the Sovereign Gate approved this blueprint submission
interface BuildGrant {
  grant_id:     string;   // = gate log_entry_id (SHA-256 hex, 64 chars)
  approved_by:  string;   // DID of the UIGen system node (UIGEN_SYSTEM_DID)
  signature:    string;   // Phase 31 stub: "{FORGE_DID}:stub"; Phase 32: Ed25519 hex
  ubuntu_score: number;   // ubuntu_score from the gate result (0.0–1.0)
}

// BuildRequest — what UIGen submits to The Forge
interface BuildRequest {
  blueprint: Blueprint;
  grant:     BuildGrant;
}

// BuildStatus — returned after submission and on status polling
interface BuildStatus {
  build_id:     string;   // UUID4 assigned by Forge
  blueprint_id: string;
  grant_id:     string;
  state:        "ACCEPTED" | "COMPLETE" | "FAILED";
  message:      string;
  submitted_at: string;   // ISO-8601
}

// BuildArtifact — the assembled output of a completed build
interface BuildArtifact {
  build_id:     string;
  blueprint_id: string;
  state:        string;
  merkle_root:  string;   // SHA-256; Phase 31: sha256(blueprint_id); Phase 32: real Merkle tree
  manifest:     ArtifactManifest;
}

interface ArtifactManifest {
  blueprint_id:  string;
  artifact_type: string;
  name:          string;
  files:         FileEntry[];
  merkle_root:   string;
  built_at:      string;  // ISO-8601
}

interface FileEntry {
  path:         string;
  content_hash: string;
  size_bytes:   number;
}
```

### 12.3 Forge Endpoints

All endpoints are mounted at `/forge` on The Forge service.

#### `POST /forge/blueprint`

Validate a blueprint against Forge acceptance criteria. Does not start a build.

**Request:** `Blueprint`

**Response:**
```typescript
{ valid: boolean; blueprint_id: string; message: string; }
```

**Validation rules:**
- `blueprint_id` must be non-empty
- `artifact_type` must be non-empty
- `components` must contain at least one entry
- `requested_by` must match `/^did:key:z/`

#### `POST /forge/build`

Submit a build request. Requires a `BuildGrant` with a non-empty `grant_id`.

**Request:** `BuildRequest`

**Response:** `BuildStatus` (initial state `"ACCEPTED"`, immediately transitions to `"COMPLETE"` in Phase 31 stubs)

**Invariant:** `grant_id` must be non-empty; absent or empty `grant_id` returns HTTP 400.

#### `GET /forge/build/{build_id}/status`

Poll the current state of a build. Returns HTTP 404 when `build_id` is unknown.

**Response:** `BuildStatus`

#### `GET /forge/build/{build_id}/artifact`

Retrieve the assembled artifact for a completed build. Returns HTTP 404 when unknown.

**Response:** `BuildArtifact`

### 12.4 Governance Rules

1. No `BuildRequest` may be submitted without a `BuildGrant` carrying a non-empty `grant_id`.
2. The `grant_id` must be a valid Sovereign Gate log entry ID (64-char SHA-256 hex) in live deployments. Phase 31 stubs accept any non-empty string.
3. Every `POST /forge/blueprint` validation success and every `POST /forge/build` completion must write a `forge_event` observation to iKhaya memory (§7.3.6).
4. UIGen must log a `FORGE_BUILD_SUBMITTED` governance event (with `build_id`, `grant_id`, `ubuntu_score`, `gate_log_entry_id`) before returning to the caller.
5. iKhaya narration (`/api/ikhaya/narrate`) must be called once per blueprint submission.

### 12.5 iKhaya forge_event Writes

`ForgeEngine` receives an `IKhayaPresence` instance at `init()` time and calls:

```python
presence.write_observation(
    key=f"forge:{event_type}:{blueprint_id}",
    value={ ...forge_event payload... },
    category="forge"
)
```

Events written:
- `blueprint_submitted` — on successful `POST /forge/blueprint` validation
- `build_complete` — on `POST /forge/build` transitioning to `COMPLETE`
- `build_failed` — on `POST /forge/build` transitioning to `FAILED`

### 12.6 Phase Boundary

| Phase | Forge capability |
|---|---|
| Phase 31 | Scaffold only — in-memory state, stub `merkle_root = sha256(blueprint_id)`, `signature = "{FORGE_DID}:stub"`, state loss on restart acceptable |
| Phase 32 | Real Ed25519 signing, real Merkle tree construction, persistent storage, Mesh-verifiable artifact registry |

---

---

## §13 — Mesh Node Identity & Handshake Protocol

**Version added:** 1.3

This section governs the protocol by which sovereign nodes introduce themselves, verify each other cryptographically, and exchange live state summaries — the constitutional birth of the Mesh pillar.

### 13.1 MeshNode Schema

```
MeshNode {
  node_did:          string    // did:key: Ed25519 — canonical node identity
  node_url:          string    // public HTTPS endpoint of this node
  node_name:         string    // human-readable label, e.g. "Durban-Home-01"
  supported_layers:  string[]  // capabilities: ["uigen", "forge", "ikhaya", "mesh"]
  public_key:        string    // Ed25519 public key as lowercase hex
  mesh_version:      string    // protocol version, e.g. "1.0"
  gate_log: {
    entry_count: int           // total GateLog entries — proxy for node maturity
    merkle_root: string        // SHA-256 Merkle root of current GateLog chain
  }
  pkl_window:        int       // number of exportable PKL memory entries
  timestamp:         int       // Unix epoch at response time
}
```

### 13.2 Handshake Exchange

```
POST /mesh/handshake
Request:  { "signature": string }    // Ed25519 signature of the MeshNode payload
Response: {
  "accepted":     boolean,
  "reason"?:      string,           // set when accepted=false
  "my_node_info": MeshNode | null   // set when accepted=true
}
```

### 13.3 Constitutional Rules

1. **Signature Required** — the responder must verify the Ed25519 signature of the requesting node's payload before accepting. Stub (presence check) allowed in Phase 40; real verification in Phase 41.
2. **Gate Governance** — every handshake is routed through the Sovereign Gate under `action_type = "mesh"`. A gate failure results in `accepted: false`.
3. **State Summary** — the responder must return its DID, capabilities, GateLog `merkle_root`, `pkl_window`, and `mesh_version` in `my_node_info`.
4. **Governance Events** — every handshake outcome is logged to the GateLog: `MESH_HANDSHAKE_ACCEPTED` on success, `MESH_HANDSHAKE_REJECTED` on failure. The GateLog entry IS the governance event.
5. **Re-Handshake** — nodes must re-handshake on reconnect. A changed `merkle_root` signals state divergence and should trigger reconciliation.

### 13.4 UIGen ExternalRepo Integration

When a UIGen instance registers an external repo via `registerExternalRepo`, it attempts `POST /mesh/handshake` on the remote URL. If the handshake succeeds, the remote node's `node_did` is stored as `ExternalRepo.nodeId`. Failure is non-fatal — the repo is registered with `nodeId = null`.

### 13.5 Phase Boundary

| Phase | Mesh capability |
|---|---|
| Phase 40 | Node identity (ephemeral, per-process DID), handshake endpoint, Gate-governed, stub signature verification, UIGen stores nodeId |
| Phase 41 | Persistent node DID (stored in DB), real Ed25519 signature verification, peer registry, bidirectional artifact exchange |

---

---

## §14 — Artifact Exchange Protocol

**Version added:** 1.4

Governs the cross-node exchange of build artifacts, including their content, lineage hops, integrity hash, and stub signature. This is the first real federated capability of the Mesh pillar.

### 14.1 ArtifactSummary Schema

```
ArtifactSummary {
  artifact_id:      string    // canonical ID in the origin node (e.g. mesh_id)
  origin_node_did:  string    // DID of the node where this artifact was created
  lineage_hops:     string[]  // ordered list of node DIDs that have touched this artifact
  content_hash:     string    // SHA-256 hash of the serialized content field
  content_mime:     string    // e.g. "application/json"
  gate_log_merkle:  string    // GateLog Merkle root at time of response
  timestamp:        int       // Unix epoch
}
```

### 14.2 ArtifactResponse Schema

```
ArtifactResponse {
  summary:   ArtifactSummary
  content:   string          // JSON-serialized artifact payload (UTF-8)
  signature: string          // SHA-256 stub over artifact_id:content_hash (real Ed25519 in Phase 42)
}
```

### 14.3 Endpoint

```
GET /mesh/artifact/{artifact_id}

Response:
  200 OK  + ArtifactResponse on success
  404     if artifact_id is unknown
  403     if Sovereign Gate denies access (gate check failed)
```

### 14.4 Constitutional Rules

1. **Gate Governance** — all artifact requests are governed by the Sovereign Gate under `action_type = "mesh"`.
2. **DID Binding** — `origin_node_did` must match the node's DID from §13.
3. **Lineage Awareness** — `lineage_hops` must include `origin_node_did` as the first element. Multi-hop propagation is Phase 42.
4. **Integrity** — `content_hash` must be SHA-256 of the `content` field (UTF-8).
5. **Signature** — stub: SHA-256 of `{artifact_id}:{content_hash}`; real Ed25519 in Phase 42.
6. **Non-Leaky** — PKL contents are never exposed; only artifact payload and lineage summary are returned.

### 14.5 UIGen Mesh Client Integration

UIGen instances fetch Mesh artifacts via `fetchMeshArtifact(baseUrl, artifactId)` in `src/lib/mesh-client.ts`. The `fetch-mesh-artifact` server action:
1. Looks up the `ExternalRepo` (already has `nodeId` from §13.4)
2. Calls `GET {repo.url}/mesh/artifact/{artifactId}`
3. Validates signature presence
4. Returns `MeshArtifactResult` to the caller

### 14.6 Phase Boundary

| Phase | Artifact Exchange capability |
|---|---|
| Phase 41 | ForgeEngine artifacts served via `/mesh/artifact/{id}`; stub signature (SHA-256); lineage_hops = [origin]; UIGen mesh client; signature presence validation |
| Phase 42 | Real Ed25519 signing; multi-hop lineage propagation; persistent artifact store |

---

---

## §15 — Cross-Node Lineage Sync

**Version added:** 1.5

Defines how artifacts carry a verifiable trail of nodes that have touched them, and how remote nodes synchronize lineage over the Mesh.

### 15.1 LineageHop Schema

```
LineageHop {
  node_did:   string  // DID of the node that touched the artifact (did:key:…)
  mesh_id:    string  // artifact ID in that node's Mesh
  timestamp:  int     // Unix epoch when this node processed the artifact
}
```

### 15.2 Extended ArtifactSummary

§14's `ArtifactSummary.lineage_hops` is promoted from `string[]` to `LineageHop[]` (origin-first):

```
ArtifactSummary {
  artifact_id:       string
  origin_node_did:   string
  lineage_hops:      LineageHop[]   // ordered, origin first
  content_hash:      string
  content_mime:      string
  gate_log_merkle:   string
  timestamp:         int
}
```

### 15.3 Lineage Sync Endpoint

```
GET /mesh/lineage/{artifact_id}

Response:
  200 OK  + LineageHop[] (ordered from origin to latest known)
  404     if artifact_id is unknown
  403     if Sovereign Gate denies access
```

### 15.4 Constitutional Rules

1. **Origin Inclusion** — `lineage_hops[0].node_did` MUST equal `origin_node_did`.
2. **Append-Only** — nodes may append new `LineageHop` entries but MUST NOT mutate existing ones.
3. **Gate Governance** — all lineage requests are governed by the Sovereign Gate under `action_type = "mesh"`.
4. **DID Consistency** — each `LineageHop.node_did` MUST start with `did:key:`.
5. **Integrity** — lineage served at `GET /mesh/lineage/{id}` MUST be consistent with `lineage_hops` in `GET /mesh/artifact/{id}`.
6. **Non-Leaky** — lineage exposes only node DIDs, mesh_ids, and timestamps — no PKL contents.

### 15.5 UIGen Integration

UIGen extends its Mesh client (`src/lib/mesh-client.ts`) with `fetchMeshLineage(baseUrl, artifactId)`. The `fetchMeshArtifactAction` in `src/actions/fetch-mesh-artifact.ts` fetches both the artifact and its lineage, returning a combined result including a `LineageGraph` built by `src/lib/lineage-graph.ts`.

### 15.6 Phase Boundary

| Phase | Lineage capability |
|---|---|
| Phase 42 | Single-hop lineage (`lineage_hops` = [origin]); `LineageStore` (in-memory); `GET /mesh/lineage/{id}`; UIGen lineage graph builder |
| Phase 43 | Multi-hop propagation (each relay node appends its hop); persistent lineage DB; cross-repo lineage graph rendering |

---

*This contract is the constitution of the interfaces. It is not documentation as an afterthought — it is the written law of the sovereign stack.*

---

## §16 — Trust & Reputation Export

### 16.1 NodeTrustSummary Schema

```
NodeTrustSummary {
  node_did:          string   // DID of the node issuing the summary
  ubuntu_score:      float    // 0.0–1.0 normalized trust score derived from GateLog
  gate_events_total: int      // total GateLog events considered
  last_updated:      int      // Unix epoch when summary was computed
  mesh_version:      string   // Mesh protocol version ("1.0")
}
```

### 16.2 ArtifactTrustHint Schema

```
ArtifactTrustHint {
  artifact_id:  string   // the artifact this hint describes
  node_did:     string   // node issuing the hint (self-signed)
  trust_label:  string   // "high" | "medium" | "low"
  confidence:   float    // 0.0–1.0 ratio of positive gate events for this artifact
  last_updated: int      // Unix epoch
}
```

### 16.3 Trust Endpoints

**GET /mesh/trust/node**
```
Request:  (no body)
Response:
  200 OK  + NodeTrustSummary
  403     if Sovereign Gate denies access
  503     if router not initialised
```

**GET /mesh/trust/artifact/{artifact_id}**
```
Request:  path parameter artifact_id (string)
Response:
  200 OK  + ArtifactTrustHint
  404     if artifact_id is unknown to this node
  403     if Sovereign Gate denies access
  503     if router not initialised
```

### 16.4 Constitutional Rules

1. **PKL-Safe** — No raw PKL content, prompts, or user data may be exposed. Only aggregated scores and labels are returned.
2. **Gate Governance** — All trust requests are governed by the Sovereign Gate under `action_type = "mesh"`, `mesh_action = "trust_request"`.
3. **Local Only** — `NodeTrustSummary` and `ArtifactTrustHint` reflect only this node's view of its own GateLog. No claims about other nodes' internal state.
4. **Deterministic Inputs** — Trust scores MUST be derived from GateLog `decision` fields and evaluation metrics, not opaque heuristics.
5. **Non-Binding** — Trust hints are advisory. Consuming nodes MUST NOT treat them as absolute truth.
6. **Versioned** — `mesh_version` MUST be included to allow future evolution of the trust computation algorithm.

### 16.5 UIGen Integration

UIGen extends `src/lib/mesh-client.ts` with `fetchMeshNodeTrust(baseUrl)` and `fetchMeshArtifactTrust(baseUrl, artifactId)`. `src/lib/trust-signals.ts` provides `combineTrustSignals(node, artifactHint?)` which returns an `EffectiveTrust` label. `fetchMeshArtifactAction` includes a best-effort trust fetch — trust absence never fails the action.

### 16.6 Phase Boundary

| Phase | Trust capability |
|---|---|
| Phase 43 | Node-level ubuntu_score from GateLog decision ratio; artifact-level trust from artifact-scoped gate events; UIGen trust signal combination; best-effort fetch (non-fatal) |
| Phase 44 | Cross-node trust propagation; reputation decay over time; trust-gated mesh relay |
