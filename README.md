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
- temporary edge relay or fan-out support.

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
  "connections": 200
}
```

Percentage-based load targets may be used by an application internally, but IERP messages should prefer concrete, interoperable units.

### Lease

A **Lease** keeps a task alive.

Tasks should expire automatically if heartbeats stop. This prevents abandoned tasks from consuming resources indefinitely.

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

---

## Security model

IERP assumes that peers are explicitly trusted, not globally open.

Implementations should enforce:

- HTTPS for all control-plane traffic;
- strong TLS for data-plane traffic;
- authenticated peer identities;
- cryptographic message signatures;
- short-lived access tokens;
- replay protection using timestamps and nonces;
- quota enforcement;
- rate limiting;
- peer suspension and removal;
- audit logging.

A recommended baseline is:

```text
Peer identity: Ed25519 key pair
Control transport: HTTPS
Message signing: Ed25519
Access tokens: JWT or PASETO
Replay protection: timestamp + nonce
Lease renewal: periodic heartbeat
```

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

schemas/
  peer.schema.json
  capability.schema.json
  task-offer.schema.json
  task-claim.schema.json
  task-grant.schema.json
  task-heartbeat.schema.json
  task-end.schema.json
  receipt.schema.json

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