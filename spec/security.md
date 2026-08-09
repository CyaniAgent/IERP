# IERP Security Specification

> **Version:** 0.1-draft
> **License:** CC BY-SA 4.0
> **Status:** Draft

---

## 1. Overview

This document defines the security architecture for the Instance Elastic Relay Protocol (IERP). IERP security is built on two foundational principles:

1. **Zero Trust (ZT)** — Never trust, always verify. No peer is inherently trusted based on network position or prior relationship alone.
2. **Principle of Least Privilege (PoLP)** — Every peer, task, and message exchange must operate with the minimum permissions required to complete its function, for the shortest time possible.

These principles are woven into every layer of the protocol: identity, trust establishment, task coordination, data relay, and audit.

---

## 2. Zero Trust Architecture

### 2.1 Core Tenets

| Tenet | Description | IERP Implementation |
|-------|-------------|---------------------|
| **Explicit Verification** | Every access request is authenticated and authorized using all available signals | Ed25519 signature on every control message; JWT/PASETO tokens scoped per task |
| **Least Privilege Access** | Users and workloads receive only the access needed, for the shortest time needed | Task-scoped Quota (bandwidth, connections), TTL, and profile constraints |
| **Assume Breach** | Security controls expect attackers may operate within the environment | Receipt auditing, lease expiration, anomaly detection, session revocation |
| **Continuous Evaluation** | Trust is not static; it is continuously reassessed during a session | Heartbeat carries runtime trust signals; quota may be dynamically adjusted |
| **Micro-segmentation** | Resources are isolated into granular security zones | Each Task is a logical security boundary; relay endpoints are task-scoped and token-protected |

### 2.2 Trust Is Computed, Not Assigned

IERP treats trust as a dynamic, computed property — not a static label granted at peer acceptance. The trust level of a peer may change based on:

- **Behavioral signals** — Heartbeat latency, response consistency, error rates
- **Load state** — Current utilization vs. declared capability
- **History** — Receipt records, past task performance, incident history
- **Context** — Time of day, geographic region, task profile sensitivity

> **Note:** Implementations SHOULD maintain a trust score per peer that influences TaskOffer decisions and ongoing quota allocation.

### 2.3 Control Plane / Data Plane Separation

Following NIST SP 800-207 guidance, IERP enforces logical separation between:

| Plane | Responsibility | Examples |
|-------|---------------|----------|
| **Control Plane** | Identity verification, policy decisions, task coordination, lease management | PeerInvite, TaskOffer, TaskGrant, TaskHeartbeat, TaskEnd |
| **Data Plane** | Actual relay traffic delivery | Audio streams, file chunks, media packets |

**Normative Requirement:** Control plane messages MUST NOT share transport channels with data plane traffic. This prevents an attacker who compromises the data channel from intercepting or forging trust decisions.

---

## 3. Principle of Least Privilege (PoLP)

### 3.1 Task-Scoped Permissions

Every relay assistance is bounded by a **Task** declaration. The Task object encodes the minimum authority granted to the assisting peer:

```json
{
  "task_id": "task_01jxyzexample",
  "profile": "voice-relay",
  "quota": {
    "bandwidth_out_mbps": 50,
    "connections": 200
  },
  "ttl_seconds": 600,
  "constraints": {
    "direction": "outbound",
    "content_type": "audio",
    "region_preference": ["asia-east"]
  }
}
```

**Required least-privilege constraints:**

| Dimension | Requirement |
|-----------|-------------|
| **Quota** | Concrete resource limits (bandwidth_out_mbps, connections) — no unlimited grants |
| **TTL** | Explicit expiration; tasks MUST NOT persist beyond TTL without renewal |
| **Direction** | Data flow direction: `inbound`, `outbound`, or `bidirectional` |
| **Content Type** | Allowed payload types per profile (audio, video, file, etc.) |
| **Profile** | Task type determines transport behavior; peers MUST NOT exceed profile scope |

### 3.2 Token Scope and Lifetime

Access tokens issued during TaskGrant MUST be:

- **Short-lived** — Default maximum lifetime of 300 seconds; renewable via heartbeat
- **Task-scoped** — Token validates against a specific `task_id` and `peer` identity
- **Single-purpose** — Token grants access to exactly one relay endpoint; no cross-task reuse
- **Revocable** — Origin instance MUST support immediate token revocation via TaskEnd

### 3.3 Information Minimization

Peers involved in relay assistance MUST NOT receive information beyond what is strictly necessary:

| Entity | What It Knows | What It Does NOT Know |
|--------|--------------|----------------------|
| **Origin Instance** | Full task context, peer capabilities, usage receipts | N/A (it is the data owner) |
| **Assisting Peer** | Relay endpoint, access token, quota limits, profile, TTL | Subject's full semantic content, user identities, other tasks |
| **End User** | Whether relay assistance is active (transparency requirement) | Internal peer selection, quota allocation, trust scores |

---

## 4. Identity and Authentication

### 4.1 Peer Identity

Each IERP peer is identified by:

- **Peer ID** — A stable, unique identifier (e.g., `ierp:example.org`)
- **Ed25519 Public Key** — Used for message signing and identity verification
- **Endpoint** — The control plane URL for peer-to-peer communication

### 4.2 Authentication Mechanisms

| Mechanism | Scope | Requirement |
|-----------|-------|-------------|
| **Ed25519 Digital Signatures** | All control plane messages | **Mandatory** — Every message MUST be signed by the sender's private key |
| **JWT / PASETO** | Task-scoped access tokens | **Mandatory** — Tokens MUST include `task_id`, `peer_id`, `exp`, and `scope` claims |
| **mTLS** | Control plane transport | **Recommended** — Mutual TLS for all peer-to-peer control traffic |
| **TLS 1.3** | Data plane transport | **Mandatory** — All relay traffic MUST traverse encrypted channels |

### 4.3 Replay Protection

All control plane messages MUST include:

- **Timestamp** — ISO 8601 or Unix epoch; receiver rejects messages older than a configurable window (default: 300 seconds)
- **Nonce** — Unique random value per message; receiver maintains a short-lived nonce cache to reject duplicates

---

## 5. Trust Lifecycle

### 5.1 Peer Establishment (Invite-Only)

```
Instance A → PeerInvite → Instance B
Instance B → PeerAccept → Instance A
```

- Trust is established through **explicit administrator invitation and consent**
- No peer can discover or join an IERP network automatically
- Public keys are exchanged and stored persistently

### 5.2 Capability Reporting

Peers periodically report their available capacity:

```json
{
  "type": "CapabilityUpdate",
  "peer": "ierp:edge.example.net",
  "supported_profiles": ["voice-relay", "file-relay"],
  "bandwidth_out_mbps": 200,
  "bandwidth_in_mbps": 100,
  "max_connections": 1000,
  "region": "asia-east",
  "updated_at": "2026-01-15T08:00:00Z"
}
```

- Capabilities are **advisory**, not binding
- Origin instances SHOULD use capability data when selecting assisting peers

### 5.3 Task Coordination

```
Origin → TaskOffer → Peer
Peer   → TaskClaim → Origin
Origin → TaskGrant → Peer
Peer   → TaskHeartbeat (periodic) → Origin
Origin → TaskEnd → Peer
Both   → Receipt
```

**Security checkpoints at each stage:**

| Stage | Security Check |
|-------|---------------|
| **TaskOffer** | Origin selects peers based on capability + trust score |
| **TaskClaim** | Peer confirms capacity; origin re-evaluates trust |
| **TaskGrant** | Origin issues scoped token; sets quota and TTL |
| **TaskHeartbeat** | Peer证明 liveness; origin evaluates continued trust; may adjust quota |
| **TaskEnd** | Origin revokes token; both sides record receipt |

### 5.4 Dynamic Trust Re-Evaluation

During an active task, the origin instance SHOULD continuously evaluate:

- **Heartbeat consistency** — Missed or delayed heartbeats degrade trust score
- **Quota utilization** — Sudden spikes or zero utilization may indicate anomalies
- **Error rates** — Relay failures or timeouts reduce trust score
- **Behavioral drift** — Unusual patterns trigger review or immediate TaskEnd

**Trust score thresholds (recommended):**

| Score Range | Action |
|-------------|--------|
| **80–100** | Full quota, eligible for priority task assignment |
| **50–79** | Reduced quota, increased heartbeat frequency |
| **20–49** | Minimal quota, flagged for manual review |
| **0–19** | TaskEnd issued; peer suspended from future tasks pending review |

---

## 6. Data Plane Security

### 6.1 Transport Encryption

All relay data MUST be transmitted over TLS 1.3 (or equivalent encrypted transport). Implementations SHOULD prefer:

- **QUIC** for low-latency media relay
- **HTTPS with HTTP/2** for file transfer relay
- **WebRTC** for end-to-end encrypted audio/video streams

### 6.2 Content Confidentiality

- Assisting peers MUST NOT persist relayed content beyond the task TTL
- Temporary caches MUST be encrypted at rest
- End-to-end encryption is RECOMMENDED for sensitive content profiles

### 6.3 Integrity Verification

- File-relay profile: Chunk manifests MUST include SHA-256 hashes; assisting peers MUST verify before serving
- Voice-relay profile: Application-layer integrity (e.g., SRTP) is the responsibility of the data plane implementation

---

## 7. Anomaly Detection and Response

### 7.1 Anomaly Signals

Implementations SHOULD monitor for:

| Signal | Threshold (Recommended) | Response |
|--------|------------------------|----------|
| Heartbeat delay | > 2x expected interval | Degrade trust score by 10 points |
| Heartbeat miss | > 3 consecutive misses | Initiate TaskEnd |
| Quota burst | > 150% of granted quota | Immediate TaskEnd + peer suspension |
| Token reuse | Different task_id with same token | Immediate peer revocation |
| Capability drift | Reported vs. actual capacity deviation > 50% | Reduce trust score; flag for review |

### 7.2 Revocation

The origin instance MAY revoke a task at any time by issuing:

```json
{
  "type": "TaskEnd",
  "task_id": "task_01jxyzexample",
  "reason": "trust_violation",
  "timestamp": "2026-01-15T09:30:00Z"
}
```

Upon receiving TaskEnd, the assisting peer MUST:

1. Immediately cease relay activity
2. Release all temporary resources (connections, cached data)
3. Invalidate the access token
4. Record a Receipt for audit purposes

### 7.3 Peer Suspension and Removal

Administrators MAY suspend or permanently remove peers based on:

- Repeated trust violations
- Manual security review findings
- Policy compliance requirements

Suspended peers MUST be rejected at the TaskOffer stage. Removed peers MUST have all outstanding tasks terminated and keys purged.

---

## 8. Audit and Receipts

### 8.1 Receipt Content

Every completed (or terminated) task MUST produce a Receipt:

```json
{
  "type": "Receipt",
  "task_id": "task_01jxyzexample",
  "peer": "ierp:edge.example.net",
  "profile": "voice-relay",
  "started_at": "2026-01-15T08:00:00Z",
  "ended_at": "2026-01-15T09:00:00Z",
  "end_reason": "normal",
  "quota_used": {
    "bandwidth_out_mbps_avg": 35,
    "connections_peak": 150
  },
  "trust_score_start": 85,
  "trust_score_end": 82,
  "anomalies_detected": 0
}
```

### 8.2 Receipt Usage

Receipts MAY be used for:

- Trust score computation and history
- Operator dashboards and monitoring
- Capacity planning and load balancing
- Audit compliance and incident investigation

**Receipts are NOT payment instruments and do NOT imply financial settlement.**

---

## 9. Configurable Parameters

IERP is designed for openness and high configurability. Protocol developers MUST be able to override the following parameters without modifying the core specification. Parameters are organized in a four-tier precedence model:

```
Task-level override  >  Instance-level config  >  Profile-level default  >  Spec default
```

### 9.1 Configuration Tiers

| Tier | Scope | Priority | Use Case |
|------|-------|----------|----------|
| **Spec Default** | All IERP implementations | Lowest | Baseline recommended values |
| **Profile Default** | Per profile (voice-relay, file-relay, edge-broadcast) | Medium | Profile-specific tuning |
| **Instance Config** | Per instance admin settings | High | Deployment-specific needs |
| **Task Override** | Per individual task (TaskOffer/constraints) | Highest | Runtime adjustment |

### 9.2 Configurable Parameters Table

| Parameter | Spec Default | Type | Valid Range | Override Tiers |
|-----------|-------------|------|-------------|----------------|
| `batch_size_threshold` | 50 | integer | 1–10,000 | Profile, Instance, Task |
| `retry_max_attempts` | 3 | integer | 0–10 | Profile, Instance, Task |
| `retry_interval_ms` | 1000 | integer | 100–60,000 | Profile, Instance, Task |
| `cache_ttl_seconds` | 600 | integer | 60–86,400 | Profile, Instance, Task |
| `token_lifetime_seconds` | 300 | integer | 30–3,600 | Instance, Task |
| `replay_window_seconds` | 300 | integer | 30–3,600 | Instance |
| `heartbeat_interval_seconds` | 30 | integer | 5–600 | Profile, Instance, Task |
| `heartbeat_miss_threshold` | 3 | integer | 1–10 | Profile, Instance |
| `trust_score_full` | 80 | integer | 50–100 | Instance |
| `trust_score_reduced` | 50 | integer | 20–80 | Instance |
| `trust_score_review` | 20 | integer | 5–50 | Instance |
| `trust_score_suspend` | 0 | integer | 0–20 | Instance |
| `trust_degrade_per_miss` | 10 | integer | 1–50 | Instance |
| `quota_burst_threshold_pct` | 150 | integer | 100–300 | Profile, Instance |
| `geoip_precision` | `"region"` | enum | `exact`, `region`, `country`, `hidden` | Instance, Task |
| `encryption_algorithm` | `"aes-256-gcm"` | string | Profile-defined | Profile, Instance |
| `sensitive_content_action` | `"abort"` | enum | `abort`, `blur`, `redact`, `passthrough` | Profile, Instance |
| `admin_approval_mode` | `"single"` | enum | `single`, `multi`, `hybrid`, `auto` | Profile, Instance |
| `trust_topology` | `"star"` | enum | `star`, `mesh`, `hierarchical` | Profile, Instance |

### 9.3 Protocol Invariants (Non-Configurable)

The following parameters are protocol invariants and MUST NOT be overridden by protocol developers:

| Invariant | Rationale |
|-----------|-----------|
| Ed25519 message signing | Security baseline — signing is non-negotiable |
| TLS 1.3 for data plane | Encryption baseline — plaintext relay is forbidden |
| Task TTL requirement | Prevents indefinite resource consumption |
| Receipt generation | Audit trail is mandatory for trust computation |
| Invite-only trust establishment | Core design principle — no open registration |
| Control/data plane separation | Prevents data channel compromise from affecting trust decisions |

### 9.4 Extension Points

Protocol developers SHOULD use the following extension mechanisms rather than forking the spec:

| Extension Mechanism | Description | Convention |
|--------------------|-------------|------------|
| **Capability extensions** | Custom fields in CapabilityUpdate | `x-` prefix namespace (e.g., `x-myprotocol.geoip精度`) |
| **Constraint extensions** | Custom fields in TaskOffer constraints | `x-` prefix namespace |
| **Receipt extensions** | Custom fields in Receipt for protocol-specific audit | `x-` prefix namespace |
| **Profile registration** | New profile types declared in CapabilityUpdate `supported_profiles` | Reverse-domain naming (e.g., `com.myprotocol.custom-relay`) |
| **Translators** | Protocol translation layers (e.g., IERP ↔ ActivityPub) | Independent service, manual implementation by protocol developers |

---

## 11. edge-broadcast Security Considerations

The `edge-broadcast` profile introduces unique security requirements due to its nature as a social content relay mechanism. This section supplements the general security model with profile-specific considerations.

### 11.1 Content Encryption

For public posts relayed through edge-broadcast:

- Content MUST be encrypted using AES-256 (or an equivalent algorithm configured by the profile/instance) during transit
- Encryption MUST include Base64 encoding with per-task salt
- Origin instance MUST be able to decrypt in O(1) time upon receipt
- Encryption keys MUST be ephemeral and task-scoped; they MUST NOT persist beyond task TTL

### 11.2 Non-Public Content Policy

- Non-public posts (private, restricted, or encrypted visibility) MUST NOT be relayed through edge-broadcast
- If a task contains non-public content, the assisting peer MUST abort delivery and report the violation
- Protocol developers MUST define explicit rules for content visibility classification

### 11.3 GeoIP Handling

- GeoIP data is provided by the origin instance and is classified as **sensitive metadata**
- GeoIP data MUST be encrypted during transmission
- Assisting peers MAY declare a GeoIP precision level:
  - `exact` — Full coordinates available
  - `region` — Only region/area-level approximation (recommended default)
  - `country` — Only country-level
  - `hidden` — Peer position is not disclosed
- Origin instance SHOULD prefer peers with similar region for lower latency

### 11.4 Cache and Temporary Storage

- Cached content TTL MUST NOT exceed `cache_ttl_seconds` (default: 600s)
- Cached data MUST be encrypted at rest using the same standard as transit encryption
- Upon task completion or TTL expiration, all cached data MUST be securely destroyed
- Protocol developers MUST implement verifiable cache destruction

### 11.5 Retry and Failure Handling

- Retry attempts MUST be bounded by `retry_max_attempts` (default: 3)
- Upon exceeding retry threshold, the task MUST be automatically terminated
- Failure logs MUST be recorded in the Receipt with `partial_failure_log`
- Origin instance SHOULD implement exponential backoff for retries

### 11.6 Administrator Approval

- Protocol implementations MUST include an administrator approval mechanism for edge-broadcast tasks
- The approval mode is **configurable** — protocol developers MUST support multiple modes, not hardcode a single approach:

| Mode | Description | Use Case |
|------|-------------|----------|
| `single` | One administrator approves | Small deployments, low-risk content |
| `multi` | Multiple administrators must approve | High-security, high-volume deployments |
| `hybrid` | Configurable threshold (e.g., 2-of-3 admins) | Balanced security and availability |
| `auto` | Policy-based automatic approval | Trusted peers, routine operations |

- Protocol developers MUST NOT enforce a specific approval mode as mandatory — the instance administrator chooses the mode
- The approval mechanism (UI, API, CLI) is implementation-defined; IERP only requires that some form of approval exists

### 11.7 Sensitive Content Handling

- Protocol developers MUST define explicit rules for sensitive content within edge-broadcast
- Required options: `abort` (reject delivery), `blur` (degrade content), `redact` (remove sensitive portions)
- Default action: `abort` — when in doubt, do not relay

---

## 12. Relationship to External Standards

| Standard | Relationship |
|----------|-------------|
| **NIST SP 800-207** | Zero Trust Architecture — IERP aligns with ZTA principles (PEP/PDP separation, trust algorithm, micro-segmentation) |
| **NIST SP 800-201** | Audit logging guidance — IERP Receipts align with audit event recommendations |
| **W3C DID** | Peer identity could optionally use Decentralized Identifiers for interoperability |
| **UCAN** | Capability-based delegation — IERP's task-scoped tokens share similar authorization philosophy |

---

## 13. Future Considerations

- **ML-based trust scoring** — Integrating behavioral anomaly detection models for automated trust evaluation
- **Federated revocation** — Cross-peer revocation signal propagation for rapid threat response
- **End-to-end encrypted relay** — v2 capability for content-confidential relay tunnels
- **Privacy-preserving audit** — Zero-knowledge proofs forReceipt verification without content exposure
