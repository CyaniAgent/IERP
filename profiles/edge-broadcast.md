# Profile: edge-broadcast

> **Profile ID:** `edge-broadcast`
> **IERP Core Version:** ≥ 0.1
> **Status:** Draft
> **License:** CC BY-SA 4.0

---

## 1. Purpose

The `edge-broadcast` profile enables federated social platforms to offload high-volume post broadcasting to trusted IERP peers. When an instance needs to deliver a batch of posts (≥ `batch_size_threshold`) to multiple remote instances, ActivityPub alone may become bottlenecked. This profile allows the origin instance to invite trusted peers to assist with edge-level content distribution using GeoIP-based就近 routing.

---

## 2. Use Cases

- Large post batches (≥50 posts) require broadcast to 50+ remote instances
- ActivityPub federation queue is congested; origin needs load relief
- Real-time post delivery with low latency across geographic regions
- Temporary edge acceleration for trending or viral content

---

## 3. Task Semantics

A single `Task` in edge-broadcast represents **delivery to one target instance**. For a broadcast to N target instances, the origin creates N parallel tasks.

### 3.1 Task Lifecycle

```
Origin selects assisting peers via GeoIP + trust score
    ↓
Origin → TaskOffer(batch_size, target_peer, content_visibility)
    ↓
Peer → TaskClaim
    ↓
Origin → TaskGrant(geoip_hint, encryption_key_ref)
    ↓
Peer relays posts (AES-256 encrypted, Base64 + salt)
    ↓
Peer → TaskHeartbeat(delivered_count, failed_count)
    ↓
Origin → TaskEnd
    ↓
Both → Receipt(delivery_stats, partial_failure_log)
    ↓
Auto-destroy: cache, token, session state
```

### 3.2 Task Completion

- Task completes when origin issues `TaskEnd`
- If partial failures occur and retry threshold is exceeded, task auto-terminates
- Failure logs are recorded in Receipt

---

## 4. Configurable Parameters

### 4.1 Profile Defaults

| Parameter | Default | Type | Description |
|-----------|---------|------|-------------|
| `batch_size_threshold` | 50 | integer | Minimum posts to trigger edge-broadcast |
| `retry_max_attempts` | 3 | integer | Max retries per failed delivery |
| `retry_interval_ms` | 1000 | integer | Base retry interval (ms); exponential backoff applied |
| `cache_ttl_seconds` | 600 | integer | Max cache lifetime on assisting peer |
| `heartbeat_interval_seconds` | 30 | integer | Expected heartbeat frequency |
| `heartbeat_miss_threshold` | 3 | integer | Consecutive misses before task abort |
| `quota_burst_threshold_pct` | 150 | integer | Quota burst tolerance (%) |
| `sensitive_content_action` | `"abort"` | enum | Action on sensitive content: `abort`, `blur`, `redact` |
| `admin_approval_mode` | `"single"` | enum | Approval mode: `single`, `multi`, `hybrid`, `auto` |

### 4.2 Instance-Level Overrides

All profile defaults MAY be overridden at the instance level. Protocol developers MUST provide an admin configuration interface.

### 4.3 Task-Level Overrides

The following parameters MAY be overridden in individual TaskOffer constraints:

- `retry_max_attempts`
- `cache_ttl_seconds`
- `heartbeat_interval_seconds`
- `sensitive_content_action`

---

## 5. Quota Structure

```json
{
  "batch_size": 75,
  "bandwidth_out_mbps": 50,
  "connections": 10,
  "max_post_size_bytes": 1048576,
  "max_total_payload_bytes": 52428800
}
```

| Field | Type | Description |
|-------|------|-------------|
| `batch_size` | integer | Number of posts in this delivery batch |
| `bandwidth_out_mbps` | integer | Maximum outbound bandwidth |
| `connections` | integer | Max concurrent connections to target |
| `max_post_size_bytes` | integer | Largest single post payload |
| `max_total_payload_bytes` | integer | Total payload size limit for the batch |

---

## 6. Content Handling

### 6.1 Content Visibility Classification

| Visibility | Relay Behavior |
|------------|---------------|
| `public` | MAY relay via edge-broadcast (encrypted in transit) |
| `unlisted` | MAY relay via edge-broadcast (not shown in public timelines) |
| `private` | MUST NOT relay — abort delivery |
| `direct` | MUST NOT relay — abort delivery |
| `custom` | Protocol developer defines rules |

### 6.2 Encryption

- Public posts: AES-256-GCM (or instance-configured algorithm) + Base64 + per-task salt
- Origin MUST decrypt in O(1) upon receipt
- Encryption key: ephemeral, task-scoped, destroyed on TaskEnd

### 6.3 Sensitive Content

- Protocol developers MUST define sensitive content detection rules
- Required actions: `abort`, `blur`, `redact`
- Default: `abort`

---

## 7. GeoIP Handling

### 7.1 Data Source

- GeoIP data is provided by the origin instance
- GeoIP is classified as **sensitive metadata** — encrypted in transit

### 7.2 Precision Levels

| Level | Description | Recommended For |
|-------|-------------|-----------------|
| `exact` | Full coordinates | Not recommended for relay |
| `region` | Region/area approximation | Default — good latency/privacy balance |
| `country` | Country-level only | Privacy-sensitive deployments |
| `hidden` | No position disclosed | When latency is not a factor |

### 7.3 Peer Selection

Origin instance SHOULD:
1. Query peer capabilities for `region` field
2. Match origin's GeoIP region to peer regions
3. Prefer peers in same or adjacent region for lower latency
4. Respect peer's declared `geoip_precision` level

---

## 8. Retry and Failure

### 8.1 Retry Policy

- Exponential backoff: `retry_interval_ms * 2^attempt`
- Max attempts: `retry_max_attempts` (default: 3)
- On final failure: auto-terminate task, record `partial_failure_log` in Receipt

### 8.2 Partial Failure

If some posts in a batch fail delivery:

```json
{
  "partial_failure_log": {
    "total_posts": 75,
    "delivered": 70,
    "failed": 5,
    "failed_post_ids": ["post_001", "post_002", "post_003", "post_004", "post_005"],
    "failure_reasons": {
      "post_001": "target_offline",
      "post_002": "encryption_mismatch"
    }
  }
}
```

### 8.3 Cache Destruction

Upon task completion or failure:
- All cached posts MUST be securely wiped
- Encryption keys MUST be destroyed
- Token MUST be invalidated

---

## 9. Translation Layer (IERP ↔ ActivityPub)

Protocol developers who wish to maintain ActivityPub compatibility MUST implement a translation layer as an independent service.

### 9.1 Requirements

- Translation layer is a standalone service, not embedded in IERP core
- MUST translate ActivityPub semantics: `Announce`, `Create`, `Follow`, `Like`, `Delete`
- MUST understand `inbox` / `outbox` semantics
- SHOULD translate all ActivityPub vocabulary
- IERP version and ActivityPub version are independently versioned

### 9.2 Protocol Priority

| Priority | Protocol | Condition |
|----------|----------|-----------|
| P0 | IERP native | Both instances support IERP and have trust relationship |
| P1 | IERP + AP translation | One instance only supports ActivityPub |
| P2 | Pure ActivityPub | No IERP trust relationship |

---

## 10. Trust Topology

Recommended trust model for edge-broadcast:

```
Origin Instance ←→ Assisting Peer ←→ Target Instance
     (A)               (B)                (C)
```

- All three parties (A, B, C) SHOULD have mutual trust
- Post relay content is the responsibility of origin (A) and target (C)
- Assisting peer (B) acts as transparent relay — no content authority
- Protocol developers MAY implement alternative topologies (mesh, hierarchical)

---

## 11. Receipt

```json
{
  "type": "Receipt",
  "task_id": "task_01jxyzexample",
  "profile": "edge-broadcast",
  "peer": "ierp:edge.example.net",
  "target": "ierp:target.example.org",
  "started_at": "2026-01-15T08:00:00Z",
  "ended_at": "2026-01-15T08:05:00Z",
  "end_reason": "normal",
  "delivery_stats": {
    "batch_size": 75,
    "delivered": 75,
    "failed": 0,
    "avg_latency_ms": 45
  },
  "quota_used": {
    "bandwidth_out_mbps_avg": 35,
    "connections_peak": 8
  },
  "trust_score_start": 85,
  "trust_score_end": 87,
  "anomalies_detected": 0,
  "partial_failure_log": null
}
```

---

## 12. Security Requirements

See [spec/security.md §11](../spec/security.md#11-edge-broadcast-security-considerations) for edge-broadcast specific security requirements including:

- Content encryption (AES-256 + Base64 + salt)
- Non-public content policy (abort relay)
- GeoIP handling and privacy
- Cache encryption and destruction
- Configurable administrator approval (single/multi/hybrid/auto)
- Sensitive content rules

---

## 13. Implementation Checklist

Protocol developers implementing edge-broadcast MUST:

- [ ] Define `batch_size_threshold` (profile or instance level)
- [ ] Implement content visibility classification
- [ ] Implement content encryption (AES-256 or equivalent)
- [ ] Implement GeoIP-based peer selection
- [ ] Implement retry with exponential backoff
- [ ] Implement cache encryption and secure destruction
- [ ] Implement administrator approval mechanism (configurable: single/multi/hybrid/auto)
- [ ] Define sensitive content rules (abort/blur/redact)
- [ ] Generate Receipts with delivery stats
- [ ] Implement TaskEnd auto-termination on threshold exceeded
- [ ] Implement translation layer if ActivityPub compatibility is needed
