# Profile: voice-relay

> **Profile ID:** `voice-relay`
> **IERP Core Version:** ≥ 0.1
> **Status:** Draft
> **License:** CC BY-SA 4.0

---

## 1. Purpose

The `voice-relay` profile enables temporary assistance with live audio distribution. When an origin instance hosts a large audio room that exceeds its network capacity, it may invite trusted peers to relay audio streams to listeners.

---

## 2. Use Cases

- Large audio room exceeds origin instance network capacity
- Late-joining listeners are scheduled to assisting instances
- Real-time audio fan-out across geographic regions
- Load balancing for voice channels during peak usage

---

## 3. Task Semantics

### 3.1 Task Lifecycle

```
Origin → TaskOffer(profile: voice-relay, subject: room_id)
    ↓
Peer → TaskClaim
    ↓
Origin → TaskGrant(relay_endpoint, access_token)
    ↓
Peer relays audio streams (low-latency)
    ↓
Peer → TaskHeartbeat
    ↓
Origin → TaskEnd (room ends or load decreases)
    ↓
Both → Receipt
```

### 3.2 Audio Flow

- Origin streams audio to assisting peers
- Assisting peers relay audio to assigned listeners
- Peers MAY forward uplink audio back to origin
- No persistent storage — relay only

---

## 4. Configurable Parameters

| Parameter | Default | Type | Description |
|-----------|---------|------|-------------|
| `heartbeat_interval_seconds` | 5 | integer | Expected heartbeat frequency (low-latency) |
| `heartbeat_miss_threshold` | 3 | integer | Consecutive misses before abort |
| `token_lifetime_seconds` | 120 | integer | Short-lived token for room session |
| `cache_ttl_seconds` | 0 | integer | No persistent cache (relay only) |
| `max_latency_ms` | 150 | integer | Maximum acceptable audio latency |
| `admin_approval_required` | `false` | boolean | Auto-approve based on policy or manual |

---

## 5. Quota Structure

```json
{
  "bandwidth_out_mbps": 50,
  "connections": 200,
  "max_listeners": 500
}
```

---

## 6. Transport Requirements

- **Protocol:** WebRTC, QUIC, or RTP over UDP
- **Encryption:** DTLS-SRTP for audio streams
- **Latency Target:** < 150ms end-to-end
- **No persistent storage** — relay only

---

## 7. Privacy

- Audio content is ephemeral — no recording or caching by assisting peers
- Room metadata (name, participants) is NOT shared with assisting peers
- Only stream routing information is provided

---

## 8. Security

- Short-lived tokens (default: 120s)
- Rapid teardown on room end
- No persistent state on assisting peers
- Trust score affects relay assignment priority

---

## 9. Receipt

```json
{
  "type": "Receipt",
  "task_id": "task_01jxyzexample",
  "profile": "voice-relay",
  "started_at": "2026-01-15T08:00:00Z",
  "ended_at": "2026-01-15T09:00:00Z",
  "end_reason": "room_ended",
  "quota_used": {
    "bandwidth_out_mbps_avg": 35,
    "connections_peak": 150,
    "max_listeners_peak": 300
  }
}
```
