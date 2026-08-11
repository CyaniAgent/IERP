# IERP — Instance Elastic Relay Protocol

> An open, invite-only protocol for temporary, consent-based network relay assistance between independent instances.

**Status:** Draft v0.1  
**Specification license:** CC BY-SA 4.0  
**Schemas, examples, and test vectors:** CC0-1.0  
**Reference implementation license:** AGPL-3.0-only

---

## What is IERP?

**IERP**, the **Instance Elastic Relay Protocol**, is a standalone protocol that allows independently operated servers to temporarily lend small amounts of network-layer capacity to one another.

IERP is designed for situations where a single instance is under temporary pressure from network-intensive workloads, such as:

- large live audio rooms;
- real-time media fan-out;
- large file synchronization;
- chunked file transfer assistance;
- temporary edge relay or fan-out support;
- temporary outbound bandwidth and connection lending during burst events (live streaming rooms, large media downloads, streaming connection surges);
- inbound media / federation traffic buffering for large instances or relays under high inbound pressure.

Instead of requiring permanent clusters, centralized CDNs, or global resource pools, IERP lets trusted instances negotiate short-lived assistance tasks.

An instance may ask its trusted peers for help. Those peers may accept or reject the request. If accepted, the assisting instance handles only the scoped relay task, for a limited time, under explicit quotas. When the task ends, the assistance stops and temporary resources are released.

---

## What IERP is not

IERP is intentionally narrow in scope.

IERP is **not**:

- a social networking protocol;
- a replacement for ActivityPub;
- a user identity system;
- a content storage protocol;
- a moderation or reporting system;
- a distributed database protocol;
- a blockchain or token protocol;
- a general-purpose compute orchestration system;
- a permanent peer-to-peer storage network;
- a global CDN standard.

IERP does not define posts, profiles, follows, likes, timelines, rooms, files, or user accounts as first-class social objects. Those concerns belong to the applications built on top of IERP.

IERP only defines how trusted instances can coordinate temporary relay assistance.

---

## Core principles

### 1. Instance sovereignty

Each instance remains fully independent.

IERP does not merge instances into one system. It does not create shared user registries, shared moderation policies, shared databases, or shared administrative control.

### 2. Invite-only trust

IERP peers are established through explicit invitation and administrator consent.

Untrusted instances should not be able to discover, join, or participate in an IERP network automatically.

### 3. Task-scoped assistance

Assistance is always limited to a specific task, profile, quota, and lifetime.

There is no permanent resource contribution unless explicitly renewed through a new task.

### 4. Explicit acceptance

An assisting instance may accept or reject any task invitation.

Implementations may support manual approval, policy-based automatic approval, or a combination of both.

### 5. No content authority

Assisting instances do not gain authority over user data, content, moderation state, account policy, or application logic.

They may temporarily relay or cache data as permitted by the task, but they do not become authoritative instances.

### 6. Language-agnostic

IERP is designed to be implementable in any programming language.

The protocol itself does not depend on a particular runtime, framework, database, or transport stack beyond its normative transport requirements.

---

## Protocol layers

IERP is divided into three conceptual layers.

### Control plane

The control plane handles:

- peer invitation;
- peer acceptance;
- peer identity;
- capability declaration;
- task invitation;
- task claiming;
- task assignment;
- heartbeat and lease renewal;
- task termination;
- usage receipts.

The control plane is intentionally small and simple.

### Profile layer

Profiles define specific relay task types.

The initial draft profiles are:

- `voice-relay`
- `file-relay`

Future profiles may define other forms of temporary network assistance.

### Data plane

The data plane carries the actual relay traffic, such as:

- audio streams;
- media packets;
- file chunks;
- HTTP range requests;
- QUIC streams;
- WebRTC media;
- other application-defined transport payloads.

IERP does not require a single data-plane transport. Each profile defines its own expected or allowed transport behavior.

---

## Core concepts

### Peer

A **Peer** is another IERP-capable instance that has been invited and accepted into a trust relationship.

Peers are identified by stable instance identifiers and authenticated using cryptographic keys.

### Capability

A **Capability** describes what a peer can currently do.

Capabilities may include:

- supported profiles;
- available outbound bandwidth;
- available inbound bandwidth;
- maximum concurrent connections;
- temporary storage availability;
- regional hints;
- policy restrictions.

Capabilities are advisory and may change over time.

### Task

A **Task** is a single temporary relay assignment.

A task includes:

- a task ID;
- a profile;
- an origin instance;
- a subject reference;
- privacy level;
- requested quota;
- split policy;
- expiration or TTL;
- constraints.

The `subject` is application-defined. IERP does not interpret it as a global object.

### Quota

A **Quota** expresses the amount of relay capacity being requested or granted.

Preferred quota units include concrete resource measures, such as:

```json
{
  "bandwidth_out_mbps": 50,
  "bandwidth_in_mbps": 30,
  "connections": 200
}
```

Percentage-based load targets may be used by an application internally, but IERP messages should prefer concrete, interoperable units.

**Temporary bandwidth and connection lending** is a core IERP capability. During burst events (live streaming rooms, large media downloads, streaming connection surges), an instance may request peers to provide limited outbound bandwidth + connection quotas for a specific subject (room ID, media ID, delivery batch). All resources are released immediately upon task completion.

### Lease

A **Lease** keeps a task alive.

Tasks should expire automatically if heartbeats stop. This prevents abandoned tasks from consuming resources indefinitely.

### Temporary Tenant Context (TTC)

> **Full specification:** [spec/ierp-core.md](spec/ierp-core.md)

A **Temporary Tenant Context (TTC)** is an optional aggregation layer on top of Task/Lease. It groups multiple related Tasks under a single temporary identifier for correlation, observability, and resource classification.

TTC is inspired by Azure's ephemeral resource lease patterns, adapted for IERP's temporary relay model.

**Core fields:**

```json
{
  "tenant_uuid": "0192e3c4-a5b6-7890-abcd-ef1234567890",
  "internal_id": "aB3dEfGhIjKlMn",
  "lease_type": "fanout"
}
```

| Field | Description |
|-------|-------------|
| `tenant_uuid` | UUID v7 (RFC 9562) — globally unique, time-ordered, enables natural sorting |
| `internal_id` | 14-char alphanumeric — instance-local unique, fast lookup, human-readable |
| `lease_type` | Enum: `relay`, `cache`, `bandwidth`, `hybrid`, `fanout` |

**TTC is NOT:**

- a new sovereignty entity — it's a logical grouping container;
- a persistent state — it's destroyed after task completion;
- a content authority — relay peers don't gain data/moderation authority;
- a mandatory mechanism — it's optional; implementations may ignore `tenant_context` fields.

**When to use TTC:**

- edge-broadcast to N target instances = 1 TTC with N parallel Tasks;
- media-relay with distribution + transcoding = 1 TTC with mixed `lease_type` Tasks;
- file-relay with multi-source chunks = 1 TTC with per-peer Tasks.

**When TTC is unnecessary:**

- single voice-relay Task (no correlation needed);
- simple file-relay with one assisting peer;
- any scenario where Tasks are logically independent.

### Receipt

A **Receipt** records the final or interim usage of a task.

Receipts may be used for:

- auditing;
- statistics;
- trust scoring;
- contribution accounting;
- operator dashboards.

Receipts are not payment instruments and do not imply financial settlement.

---

## Minimal message flow

A minimal IERP interaction looks like this:

```text
PeerInvite
PeerAccept
CapabilityUpdate
TaskOffer
TaskClaim
TaskGrant
TaskHeartbeat
TaskEnd
```

### Peer establishment

```text
Instance A invites Instance B.
Instance B administrator approves.
Both instances store each other's public key and endpoint.
```

### Capability exchange

```text
Instance B periodically reports available capacity.
Instance A uses this information when selecting assistance candidates.
```

### Task coordination

```text
Instance A issues a TaskOffer.
Instance B responds with a TaskClaim.
Instance A decides whether to assign capacity.
Instance A sends TaskGrant.
Instance B begins relaying within the granted quota.
Instance B sends periodic TaskHeartbeat messages.
When the task is complete, Instance A sends TaskEnd.
Both sides record usage receipts.
```

---

## Example: TaskOffer

```json
{
  "type": "TaskOffer",
  "task_id": "task_01jxyzexample",
  "profile": "voice-relay",
  "origin": "ierp:example.org",
  "subject": "room:public-123",
  "privacy": "public",
  "quota": {
    "bandwidth_out_mbps": 50,
    "connections": 200
  },
  "split_policy": "equal",
  "ttl_seconds": 600,
  "constraints": {
    "max_latency_ms": 150,
    "region_preference": ["asia-east", "asia-northeast"]
  }
}
```

---

## Example: TaskGrant

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
  "lease_seconds": 30
}
```

---

## Profiles

### `voice-relay`

> **TTC applicability:** Low — single Task per audio room, no multi-Task correlation needed.

The `voice-relay` profile is intended for temporary assistance with live audio distribution.

Typical use cases:

- a large audio room exceeds the origin instance’s network capacity;
- late-joining listeners are scheduled to assisting instances;
- assisting instances relay audio from the origin instance to listeners;
- assisting instances may also forward uplink audio back to the origin instance;
- when the room ends or load decreases, assistance stops.

The `voice-relay` profile should support:

- short-lived tokens;
- low-latency transport;
- heartbeat and rapid teardown;
- no persistent storage by default;
- privacy-level restrictions.

### `file-relay`

> **TTC applicability:** Medium — multi-source chunk distribution may use 1 TTC with N per-peer Tasks.

The `file-relay` profile is intended for temporary assistance with large file transfer or synchronization.

Typical use cases:

- a large file is being synchronized between trusted instances;
- assisting instances temporarily cache encrypted or hashed chunks;
- downloading instances retrieve chunks from multiple sources;
- chunks are validated by hash;
- cached chunks expire automatically.

The `file-relay` profile should support:

- chunk manifests;
- hash verification;
- HTTP Range or equivalent range-based retrieval;
- TTL-based temporary caching;
- maximum temporary storage limits;
- explicit no-persist or temporary-cache policies.

### `edge-broadcast`

> **TTC applicability:** High — broadcast to N target instances = 1 TTC with N parallel Tasks. `lease_type: fanout` or `hybrid`.

The `edge-broadcast` profile is intended for high-volume post broadcasting across federated social instances.

Typical use cases:

- an instance needs to broadcast ≥50 posts to 50+ remote instances;
- ActivityPub federation queue is congested and needs load relief;
- real-time post delivery with GeoIP-based就近 routing;
- temporary edge acceleration for trending or viral content.

The `edge-broadcast` profile should support:

- batch post delivery (single Task = delivery to one target instance);
- AES-256 encrypted transit with Base64 + salt;
- GeoIP-based peer selection with configurable precision;
- retry with exponential backoff and partial failure reporting;
- mandatory administrator approval;
- automatic cache destruction on task completion;
- non-public content abort policy;
- ActivityPub translation layer (independent service).

### `preview-relay`

> **TTC applicability:** Low — single preview fetch + cache per task, no multi-Task correlation needed.

The `preview-relay` profile solves the link preview request amplification problem ("Mastodon stampede").

Typical use cases:

- a post with a link is broadcast to 100+ instances, each independently fetching the preview;
- a viral post triggers thousands of GET requests to the target website;
- request amplification ratio reaches 1,147:1 with traffic amplification of 36,704:1;
- the target website becomes unresponsive under the request burst.

The `preview-relay` profile should support:

- origin instance fetches OGP preview once and signs it;
- encrypted preview data is relayed to assisting peers;
- peers serve preview locally (no fetch to target website);
- preview cache with configurable TTL;
- signature verification on receiving end;
- fallback to standard OGP when preview-relay unavailable.

### `media-relay`

> **TTC applicability:** High — distribution + transcoding = 1 TTC with mixed `lease_type` Tasks (`cache` + `relay`).

The `media-relay` profile enables temporary assistance with media distribution and transcoding.

Typical use cases:

- popular media (images, videos, GIFs) is fetched by 100+ remote instances simultaneously;
- origin instance bandwidth is saturated;
- multiple video uploads trigger concurrent ffmpeg processes, CPU at 100%;
- large media files (>100MB) benefit from multi-source chunk distribution.

The `media-relay` profile should support:

- media encryption (AES-256-GCM) at rest on assisting peers;
- chunk manifest with SHA-256 hash verification;
- HTTP Range request support for parallel download;
- transcoding offload to peers with available CPU;
- configurable transcode timeout;
- strict TTL + max cache size limits;
- secure cache destruction on task completion.

---

## Security model

> **Full specification:** [spec/security.md](spec/security.md)

IERP security is built on **Zero Trust** and **Principle of Least Privilege** foundations:

- **Never trust, always verify** — Every peer interaction requires cryptographic authentication; trust is continuously re-evaluated, not statically assigned.
- **Minimum permission** — Tasks are bounded by scoped Quota (bandwidth, connections), TTL, data direction, and content type. Peers receive only what they need, for the shortest time needed.
- **Assume breach** — Lease auto-expiration, anomaly detection, and immediate session revocation limit blast radius.
- **Control/data plane separation** — Trust decisions and relay traffic are logically isolated.

### Security baseline

| Layer | Mechanism |
|-------|-----------|
| Peer identity | Ed25519 key pair |
| Message signing | Ed25519 on all control messages |
| Access tokens | JWT or PASETO, task-scoped, short-lived |
| Transport | HTTPS (control) / TLS 1.3 (data) |
| Replay protection | Timestamp + nonce |
| Lease management | Periodic heartbeat with trust re-evaluation |
| Audit | Receipts with trust scores and anomaly records |

IERP does not require a global certificate authority. Trust is established through direct peer invitation and administrator approval.

---

## Privacy model

IERP should be deployed with privacy-preserving defaults.

Recommended defaults:

- assisting instances should not persist content;
- temporary caches must expire;
- private tasks should be excluded from relay assistance by default;
- sensitive or end-to-end encrypted tasks should require explicit policy approval;
- user-facing applications should disclose when relay assistance is active;
- metadata should be minimized;
- assisting instances should not index, scan, analyze, or reuse relayed content.

IERP does not itself inspect content. Content policy belongs to the application and to the operators of each instance.

---

## Relationship to other protocols

IERP is independent of other federated or social protocols.

It may be used alongside protocols such as ActivityPub, Matrix, Nostr, or custom application protocols, but it does not depend on them.

IERP is not an extension of ActivityPub.

An instance may support ActivityPub and IERP at the same time, but the two protocols serve different purposes:

```text
ActivityPub: social object federation
IERP: temporary relay assistance between trusted instances
```

---

## Discovery

A minimal IERP discovery endpoint may be exposed at:

```text
/.well-known/ierp
```

Example:

```json
{
  "protocol": "ierp",
  "version": "0.1",
  "peer_id": "ierp:example.org",
  "endpoint": "https://example.org/ierp",
  "public_key": "ed25519:examplekey",
  "profiles": [
    "voice-relay",
    "file-relay"
  ]
}
```

Discovery should not expose sensitive configuration, internal load, or private task information.

---

## Implementation notes

IERP is intentionally language-neutral.

A conforming implementation may be written in any language that supports:

- HTTPS;
- JSON or equivalent structured messages;
- cryptographic signatures;
- timers and lease management;
- secure token handling.

For lightweight edge relays, systems languages or compiled networking languages are often a good fit. For application servers, higher-level languages may be more convenient.

The protocol does not mandate a database. Implementations may use:

- SQLite;
- PostgreSQL;
- in-memory state;
- embedded key-value stores;
- external coordination services.

The only normative requirement is that the implementation follows the IERP lifecycle, security, and profile rules.

---

## Repository layout

This repository may contain:

```text
spec/
  ierp-core.md
  security.md
  privacy.md

profiles/
  voice-relay.md
  file-relay.md
  edge-broadcast.md
  preview-relay.md
  media-relay.md

schemas/
  peer.schema.json
  capability.schema.json
  task-offer.schema.json
  task-claim.schema.json
  task-grant.schema.json
  task-heartbeat.schema.json
  task-end.schema.json
  receipt.schema.json
  tenant-context.schema.json

examples/
  peer-invite.json
  task-offer-voice.json
  task-offer-file.json
  task-grant.json
  task-receipt.json

test-vectors/
  signatures/
  expired-lease/
  invalid-token/
  task-lifecycle/

reference/
  ierp-daemon/
  ierp-cli/
  ierp-edge/
```

### `schemas/` — Canonical JSON Templates

Each file in `schemas/` is a **typical regular JSON example** (canonical JSON template) that defines the normative structure of an IERP message type. These templates:

- define the required fields, types, and constraints for each message;
- serve as the single source of truth for message structure;
- are referenced by profiles and specs when describing message formats;
- use `additionalProperties: true` where extensions are allowed.

**Schemas are NOT code.** They are declarative JSON templates that implementers read and translate into their chosen language's data structures.

### `examples/` — Language-Specific Implementation Examples

Each file in `examples/` is a **reference implementation** in a specific programming language, based on the canonical templates in `schemas/`. These examples:

- demonstrate how to construct and parse IERP messages in a real language;
- show idiomatic patterns (Go structs, TypeScript interfaces, Python dataclasses, etc.);
- include comments explaining protocol-specific logic;
- are derived from the canonical `schemas/` templates but are NOT normative.

**Examples are NOT schemas.** They are implementation guides. The canonical message structure is defined in `schemas/`, not in `examples/`.

Not all directories are required for every implementation.

---

## Conformance

A minimal IERP implementation should support:

1. peer invitation and acceptance;
2. peer public key verification;
3. capability reporting;
4. `TaskOffer` handling;
5. `TaskClaim` or equivalent acceptance;
6. `TaskGrant` handling;
7. heartbeat and lease expiry;
8. task termination;
9. usage receipts;
10. at least one profile.

Implementations that only participate as assisting instances may omit task-originating behavior.

---

## Versioning

IERP uses semantic versioning for the specification.

```text
MAJOR.MINOR.PATCH
```

- Backward-incompatible changes require a major version.
- Backward-compatible additions require a minor version.
- Clarifications and errata require a patch version.

Profiles may version independently, but they must declare the IERP core versions they support.

---

## Design goals

IERP aims to be:

- simple to audit;
- simple to self-host;
- simple to implement;
- safe by default;
- privacy-aware;
- operator-controlled;
- extensible through profiles;
- independent of any single application domain.

---

## Non-goals

IERP deliberately avoids:

- global open relays;
- anonymous participation;
- automatic resource pooling;
- permanent storage networks;
- content moderation automation;
- user account federation;
- financial settlement;
- blockchain consensus;
- general compute scheduling.

---

## Contributing

Contributions are welcome.

Before submitting a pull request, please read:

- `CONTRIBUTING.md`
- `SPEC.md`
- `SECURITY.md`
- `PRIVACY.md`

Protocol changes should include:

- a clear problem statement;
- proposed normative text;
- schema updates where applicable;
- example messages;
- test vectors;
- backward-compatibility notes.

---

## License

This project is licensed in layers.

### Specification

The IERP specification documents are licensed under:

```text
Creative Commons Attribution-ShareAlike 4.0 International
CC BY-SA 4.0
```

### Schemas, examples, and test vectors

JSON schemas, example messages, and test vectors are licensed under:

```text
CC0-1.0
```

### Reference implementation

Any reference implementation included in this repository is licensed under:

```text
GNU Affero General Public License v3.0 only
AGPL-3.0-only
```

### Trademarks

The names “IERP” and “Instance Elastic Relay Protocol”, along with related logos or marks, are not licensed under the above licenses unless explicitly stated otherwise.

Use of the IERP name for official conformance claims, endorsements, or branding may be subject to separate trademark rules.

---

## Disclaimer

IERP is provided as-is, without warranty of any kind.

Operators deploying IERP should review their own security, privacy, legal, and operational requirements before enabling relay assistance.