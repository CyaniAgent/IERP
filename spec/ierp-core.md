# IERP Core Specification — Temporary Tenant Context (TTC)

> **Version:** 0.1-draft
> **License:** CC BY-SA 4.0
> **Status:** Draft

---

## 1. Overview

This document defines the **Temporary Tenant Context (TTC)** layer in the Instance Elastic Relay Protocol (IERP). TTC provides a logical grouping mechanism on top of existing Task/Lease semantics, inspired by Azure's ephemeral resource lease patterns.

TTC is an **optional aggregation layer** — it does not replace Task, Lease, or Receipt. It adds a time-bounded context identifier that enables:

- correlating multiple related Tasks across a single relay session;
- fast local lookup via a compact internal identifier;
- classifying lease types for resource management;
- natural time-ordered sorting via UUID v7.

---

## 2. Design Rationale

### 2.1 Why TTC?

In complex relay scenarios (edge-broadcast, media-relay, file-relay), a single logical operation may involve multiple Tasks:

- edge-broadcast to N target instances = N parallel Tasks;
- media-relay with transcoding = distribution Task + transcoding Tasks;
- file-relay with multi-source chunks = one Task per assisting peer.

Without a unifying context, these Tasks are logically independent. TTC groups them under a single temporary identifier for:

- **Correlation** — operators can see all Tasks belonging to one logical session;
- **Observability** — logs and dashboards can filter by `tenant_uuid`;
- **Resource classification** — `lease_type` declares what kind of resource is being borrowed;
- **Lifecycle management** — TTC is destroyed after task completion, enforcing impermanence.

### 2.2 What TTC Is NOT

| Is | Is NOT |
|----|--------|
| A logical grouping container | A new sovereignty entity |
| Bound to Task + TTL | An independent lifecycle |
| Destroyed after task completion | A persistent state |
| Limited relay permissions | Content authority |
| An optional aggregation layer | A mandatory mechanism |

**TTC does NOT introduce:**

- shared user directories;
- shared moderation databases;
- long-term resource contributions;
- cross-instance state synchronization;
- new trust establishment mechanisms.

---

## 3. TTC Structure

### 3.1 Core Fields

```json
{
  "tenant_uuid": "0192e3c4-a5b6-7890-abcd-ef1234567890",
  "internal_id": "aB3dEfGhIjKlMn",
  "lease_type": "relay",
  "created_at": "2026-01-15T08:00:00Z",
  "ttl_seconds": 600,
  "task_ids": [
    "task_01jxyzexample1",
    "task_01jxyzexample2"
  ]
}
```

### 3.2 Field Definitions

#### `tenant_uuid`

| Property | Value |
|----------|-------|
| Type | `string` (UUID v7, RFC 9562) |
| Format | `xxxxxxxx-xxxx-7xxx-yxxx-xxxxxxxxxxxx` |
| Required | Yes |
| Scope | Globally unique per temporary lease context |
| Purpose | Cross-instance correlation, time-ordered sorting |

**UUID v7 selection rationale:**

- **Time-ordered** — 48-bit Unix millisecond timestamp enables natural sorting by creation time;
- **No coordination** — random bits (74 bits) ensure no cross-instance collision without coordination;
- **B-tree friendly** — time-ordered prefix optimizes database index locality;
- **Information leakage** — leaks creation time at millisecond precision, which is acceptable for temporary contexts with no privacy implications.

**Normative requirement:** Implementations MUST generate UUID v7 for `tenant_uuid`. UUID v4 is NOT permitted for TTC because it loses time-ordering and index locality.

#### `internal_id`

| Property | Value |
|----------|-------|
| Type | `string` |
| Pattern | `[a-zA-Z0-9]{14}` |
| Required | Yes |
| Scope | Instance-local unique |
| Purpose | Fast local lookup, human-readable reference |

**Design rationale:**

- 14 characters × 62 possibilities = ~1.2 × 10²⁵ space — collision probability is negligible within a single instance lifetime;
- Shorter than UUID (36 chars) by 61%, shorter than ULID (26 chars) by 46%;
- Suitable for CLI output, log messages, and operator dashboards;
- Case-sensitive alphanumeric avoids ambiguous characters (no `0/O`, `1/l/I`).

**Normative requirement:** Implementations MUST generate a random `internal_id` that is unique within the local instance. The generation algorithm MUST use a CSPRNG.

#### `lease_type`

| Property | Value |
|----------|-------|
| Type | `string` (enum) |
| Required | Yes |
| Purpose | Classify the type of resource being temporarily borrowed |

**Enum values:**

| Value | Description | Typical Profile |
|-------|-------------|-----------------|
| `relay` | Live data relay (no persistent storage) | voice-relay |
| `cache` | Temporary caching of content | file-relay, media-relay |
| `bandwidth` | Outbound bandwidth/connection lending | Cross-profile (burst events) |
| `hybrid` | Combined relay + cache | edge-broadcast |
| `fanout` | High-fan-out distribution | edge-broadcast (viral content) |

**Normative requirement:** Implementations SHOULD use `lease_type` to classify resource usage. The enum is extensible — implementations MAY define additional types using the `x-` prefix convention (e.g., `x-transcode`).

#### `created_at`

| Property | Value |
|----------|-------|
| Type | `string` (ISO 8601) |
| Required | Yes |
| Purpose | TTC creation timestamp |

#### `ttl_seconds`

| Property | Value |
|----------|-------|
| Type | `integer` |
| Minimum | 1 |
| Maximum | 86400 |
| Required | Yes |
| Purpose | Maximum lifetime of the TTC context |

**Normative requirement:** TTC MUST be destroyed when `ttl_seconds` expires, regardless of task state. Implementations MUST NOT allow a TTC to persist beyond its TTL.

#### `task_ids`

| Property | Value |
|----------|-------|
| Type | `array` of `string` |
| Required | Yes |
| Purpose | List of correlated Task IDs |

**Normative requirement:** At least one Task ID MUST be present. Implementations MUST maintain the `task_ids` array as Tasks are added to the TTC context.

---

## 4. TTC Lifecycle

### 4.1 State Diagram

```
[Created] → [Active] → [Destroyed]
    ↑           ↑
    │           └── TaskHeartbeat (TTL extended per-task)
    │
    └── TaskOffer (TTC created)
```

### 4.2 Lifecycle Events

| Event | Trigger | Action |
|-------|---------|--------|
| **Create** | TaskOffer with TTC context | Generate `tenant_uuid` + `internal_id`, record `lease_type` |
| **Activate** | TaskGrant accepted | TTC transitions to Active state |
| **Extend** | TaskHeartbeat | TTC remains alive as long as at least one Task heartbeat is active |
| **Destroy (normal)** | All Tasks end | Destroy TTC, release all resources, record Receipt |
| **Destroy (expired)** | TTL expires | Force-destroy TTC, terminate all active Tasks, record Receipt |
| **Destroy (violation)** | Trust violation | Force-destroy TTC, terminate all Tasks, record Receipt with anomaly |

### 4.3 Destruction Semantics

Upon TTC destruction, implementations MUST:

1. Terminate all active Tasks in the TTC (if any remain);
2. Revoke all access tokens issued under the TTC;
3. Destroy all temporary caches and relay state;
4. Flush Receipts for all completed/terminated Tasks;
5. Discard `tenant_uuid` and `internal_id` — they MUST NOT be reused.

**Normative requirement:** TTC destruction MUST be verifiable. Implementations SHOULD log destruction events with timestamp and reason.

---

## 5. TTC in Message Flow

### 5.1 TaskOffer with TTC Context

When the origin instance creates a TaskOffer that belongs to a TTC, the TTC context is included:

```json
{
  "type": "TaskOffer",
  "task_id": "task_01jxyzexample",
  "profile": "edge-broadcast",
  "origin": "ierp:example.org",
  "subject": "post:batch-20260115",
  "privacy": "public",
  "quota": {
    "bandwidth_out_mbps": 50,
    "connections": 200
  },
  "ttl_seconds": 600,
  "tenant_context": {
    "tenant_uuid": "0192e3c4-a5b6-7890-abcd-ef1234567890",
    "internal_id": "aB3dEfGhIjKlMn",
    "lease_type": "fanout"
  }
}
```

### 5.2 TaskGrant with TTC Context

The origin confirms the TTC context in TaskGrant:

```json
{
  "type": "TaskGrant",
  "task_id": "task_01jxyzexample",
  "peer": "ierp:edge.example.net",
  "granted_quota": {
    "bandwidth_out_mbps": 25,
    "connections": 100
  },
  "relay_endpoint": "https://edge.example.net/ierp/relay/task_01jxyzexample",
  "access_token": "short-lived-token",
  "lease_seconds": 30,
  "tenant_context": {
    "tenant_uuid": "0192e3c4-a5b6-7890-abcd-ef1234567890",
    "internal_id": "aB3dEfGhIjKlMn"
  }
}
```

### 5.3 TaskHeartbeat with TTC Context

Heartbeats include the `tenant_uuid` for correlation:

```json
{
  "type": "TaskHeartbeat",
  "task_id": "task_01jxyzexample",
  "peer": "ierp:edge.example.net",
  "timestamp": "2026-01-15T08:05:00Z",
  "tenant_context": {
    "tenant_uuid": "0192e3c4-a5b6-7890-abcd-ef1234567890"
  }
}
```

### 5.4 Receipt with TTC Context

Receipts include the full TTC context for audit:

```json
{
  "type": "Receipt",
  "task_id": "task_01jxyzexample",
  "peer": "ierp:edge.example.net",
  "profile": "edge-broadcast",
  "started_at": "2026-01-15T08:00:00Z",
  "ended_at": "2026-01-15T09:00:00Z",
  "end_reason": "normal",
  "tenant_context": {
    "tenant_uuid": "0192e3c4-a5b6-7890-abcd-ef1234567890",
    "internal_id": "aB3dEfGhIjKlMn",
    "lease_type": "fanout"
  }
}
```

---

## 6. Multi-Task Correlation

### 6.1 Edge-Broadcast Example

A broadcast to 5 target instances creates 5 parallel Tasks, all under one TTC:

```
TTC: tenant_uuid = 0192e3c4-...
├── Task 1 → peer-a (target: instance-a.example.org)
├── Task 2 → peer-b (target: instance-b.example.org)
├── Task 3 → peer-c (target: instance-c.example.org)
├── Task 4 → peer-d (target: instance-d.example.org)
└── Task 5 → peer-e (target: instance-e.example.org)
```

All 5 Tasks share the same `tenant_uuid`. Operators can:

- filter logs by `tenant_uuid` to see the full broadcast;
- compute aggregate Receipts per TTC;
- detect partial failures (some Tasks succeed, others fail).

### 6.2 Media-Relay Example

A media distribution + transcoding session:

```
TTC: tenant_uuid = 0192e3c4-...
├── Task 1 (lease_type: cache) → peer-a (serve original media)
├── Task 2 (lease_type: cache) → peer-b (serve original media)
└── Task 3 (lease_type: relay) → peer-c (transcode + serve)
```

Each Task has its own `lease_type`, but they share the same TTC for correlation.

---

## 7. Security Considerations

### 7.1 TTC Is NOT a Trust Boundary

TTC is a logical grouping, not a security boundary. All existing security mechanisms (Ed25519 signing, TLS 1.3, token scoping, trust re-evaluation) apply at the Task level. TTC does not introduce new trust relationships.

### 7.2 UUID v7 Information Leakage

UUID v7 leaks the creation timestamp at millisecond precision. This is acceptable because:

- TTC is a temporary context (seconds to hours), not a long-lived identifier;
- creation time is not sensitive metadata for relay tasks;
- the same information is already available in message timestamps.

### 7.3 Internal ID Uniqueness

The 14-character `internal_id` has ~83 bits of entropy. Collision probability within a single instance is negligible for any practical instance lifetime. Implementations MUST NOT rely on `internal_id` for cross-instance uniqueness — use `tenant_uuid` for that purpose.

### 7.4 TTL Enforcement

TTC TTL MUST be enforced at the instance level. If a TTC expires:

1. All associated Tasks MUST be terminated;
2. All tokens MUST be revoked;
3. All caches MUST be destroyed;
4. Receipts MUST be flushed before destruction.

Implementations MUST NOT allow a TTC to persist beyond its TTL, even if Tasks are still active.

---

## 8. Relationship to Existing Concepts

```
TTC (optional aggregation)
├── Task 1
│   ├── Quota
│   ├── Lease (heartbeat)
│   └── Receipt
├── Task 2
│   ├── Quota
│   ├── Lease (heartbeat)
│   └── Receipt
└── Task N
    ├── Quota
    ├── Lease (heartbeat)
    └── Receipt
```

| Concept | Level | Lifecycle |
|---------|-------|-----------|
| **Peer** | Permanent | Persists across sessions |
| **TTC** | Optional | Destroyed after task completion |
| **Task** | Required | Bounded by TTL |
| **Quota** | Required | Bounded by Task |
| **Lease** | Required | Bounded by Task TTL |
| **Receipt** | Required | Generated on TaskEnd |

---

## 9. Configurable Parameters

| Parameter | Spec Default | Type | Override Tiers |
|-----------|-------------|------|----------------|
| `tenant_ttl_seconds` | 3600 | integer (1–86400) | Instance, Task |
| `tenant_max_tasks` | 100 | integer (1–1000) | Instance |
| `internal_id_charset` | `[a-zA-Z0-9]` | string | Instance (immutable) |
| `internal_id_length` | 14 | integer (8–20) | Instance (immutable) |

**Note:** `internal_id_charset` and `internal_id_length` are immutable after instance creation. Changing them would invalidate existing `internal_id` values.

---

## 10. Conformance

Implementations that support TTC MUST:

1. Generate UUID v7 for `tenant_uuid`;
2. Generate CSPRNG-based `internal_id` with instance-local uniqueness;
3. Enforce TTL expiration for all TTC contexts;
4. Destroy all resources upon TTC destruction;
5. Include `tenant_context` in Receipts for audit;
6. NOT reuse `tenant_uuid` or `internal_id` after destruction.

Implementations that do NOT support TTC MUST ignore `tenant_context` fields in messages and operate normally at the Task level.
