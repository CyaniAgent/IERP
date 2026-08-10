# Profile: media-relay

> **Profile ID:** `media-relay`
> **IERP Core Version:** ≥ 0.1
> **Status:** Draft
> **License:** CC BY-SA 4.0

---

## 1. Purpose

The `media-relay` profile enables temporary assistance with media distribution and transcoding in federated social platforms. When popular media (images, videos, GIFs) are simultaneously fetched by many remote instances, or when the origin instance's CPU is saturated by concurrent ffmpeg transcoding, this profile allows the origin to offload media processing and distribution to trusted IERP peers.

---

## 2. Problem Statement

### 2.1 Media CPU Saturation

Federated platforms like Mastodon process media attachments using ffmpeg/imagemagick on the origin instance. When multiple videos are uploaded or federated simultaneously:

- 5-6 ffmpeg processes run in parallel on a 2-core server
- CPU utilization hits 100%, causing Web requests to time out
- Memory exhaustion leads to OOM kills
- Small instances (2 core / 4GB) become completely unresponsive

### 2.2 Media Fetch Amplification

When a post with a video goes viral:
- Hundreds of instances simultaneously fetch the media file
- Origin server's outbound bandwidth is saturated
- Small instances cannot serve media fast enough
- Users experience "media unavailable" errors

---

## 3. Use Cases

### UC-1: Media Distribution Offloading

- Popular media is fetched by 100+ remote instances simultaneously
- Origin instance bandwidth is saturated
- Origin requests trusted peers to temporarily cache and serve media
- Remote instances fetch from nearest peer instead of origin
- Traffic is distributed across multiple geographically diverse peers

### UC-2: Transcoding Offloading

- Multiple video uploads or federated videos trigger concurrent ffmpeg processes
- Origin instance CPU is at 100%
- Origin offloads transcoding tasks to peers with available CPU capacity
- Transcoded results are returned to origin or served directly
- Origin CPU load returns to normal

### UC-3: Multi-Source Chunk Distribution

- Large media files (>100MB) are distributed via chunked transfer
- Origin generates chunk manifest with SHA-256 hashes
- Chunks are distributed to multiple assisting peers
- Downloaders fetch chunks from multiple peers in parallel
- Download speed improves via multi-source retrieval

### UC-4: Inbound Media / Federation Traffic Buffering

When a large instance or relay receives high volumes of inbound media (e.g., viral content influx, disaster event, trending topic), it may temporarily borrow peer storage and bandwidth for buffering and verification:

- Origin instance issues `media-relay` TaskOffer with `direction: inbound`
- Assisting peers provide temporary storage + bandwidth for inbound media chunks
- Inbound media is encrypted, hash-verified, and buffered at the peer
- Origin pulls verified chunks from peer at its own pace
- Reduces冲击 on origin's local disk and object storage
- Task ends when inbound pressure subsides; all buffered data destroyed

**Constraints (mandatory):**
- Buffer TTL MUST NOT exceed `cache_ttl_seconds`
- All buffered data MUST be encrypted at rest
- Origin MUST verify hash before accepting buffered chunks
- Task completion triggers immediate cache destruction
- Strict quota enforcement on storage and bandwidth

---

## 4. Task Semantics

### 4.1 Media Distribution Flow

```
Origin identifies popular media (fetch count > threshold)
    ↓
Origin → TaskOffer(profile: media-relay, subject: media_id)
    ↓
Peer → TaskClaim
    ↓
Origin → TaskGrant(media_endpoint, access_token)
    ↓
Peer fetches encrypted media from origin's relay endpoint
    ↓
Peer caches media (encrypted, hash-verified)
    ↓
Remote instances fetch from peer (HTTP Range or direct)
    ↓
Peer → TaskHeartbeat(fetch_count, bandwidth_used)
    ↓
Origin → TaskEnd (TTL expired or demand decreases)
    ↓
Both → Receipt
    ↓
Auto-destroy: cached media, encryption key
```

### 4.2 Transcoding Offload Flow

```
Origin has pending transcode tasks (CPU overloaded)
    ↓
Origin → TaskOffer(profile: media-relay, subject: transcode_batch)
    ↓
Peer → TaskClaim (peers with available CPU)
    ↓
Origin → TaskGrant(transcode_endpoint, access_token)
    ↓
Origin sends raw media to peer for transcoding
    ↓
Peer runs ffmpeg, returns transcoded result
    ↓
Origin receives transcoded media
    ↓
Origin → TaskEnd
    ↓
Both → Receipt(cpu_time_used, transcoding_duration)
```

---

## 5. Configurable Parameters

### 5.1 Profile Defaults

| Parameter | Default | Type | Description |
|-----------|---------|------|-------------|
| `cache_ttl_seconds` | 3600 | integer | Media cache lifetime |
| `heartbeat_interval_seconds` | 30 | integer | Heartbeat frequency |
| `max_media_size_bytes` | 1073741824 | integer | Max single media file (1GB) |
| `max_cache_size_bytes` | 10737418240 | integer | Max total cache per peer (10GB) |
| `transcode_max_concurrent` | 2 | integer | Max concurrent transcode tasks |
| `transcode_timeout_seconds` | 600 | integer | Max transcode duration |
| `fetch_count_threshold` | 10 | integer | Fetches before offload considered |
| `admin_approval_mode` | `"single"` | enum | Approval mode: `single`, `multi` |

### 5.2 Instance-Level Overrides

All profile defaults MAY be overridden at the instance level. Protocol developers MUST provide an admin configuration interface.

---

## 6. Quota Structure

```json
{
  "bandwidth_out_mbps": 100,
  "connections": 100,
  "max_media_size_bytes": 1073741824,
  "max_cache_size_bytes": 10737418240,
  "transcode_slots": 2
}
```

---

## 7. Media Handling

### 7.1 Supported Media Types

| Type | Extensions | Transcoding Required |
|------|-----------|---------------------|
| Image | jpg, png, gif, webp | Resize/thumbnail only |
| Video | mp4, webm, mov | Yes (H.264/H.265) |
| Audio | mp3, ogg, wav | Transcode to opus/aac |
| GIF (animated) | gif | Convert to video for large files |

### 7.2 Encryption

- All media cached on assisting peers MUST be encrypted (AES-256-GCM)
- Encryption keys are ephemeral and task-scoped
- Keys are destroyed on TaskEnd or TTL expiration
- Origin MUST be able to decrypt in O(1) time

### 7.3 Integrity Verification

- Media files are verified by SHA-256 hash before caching
- Chunk manifests include per-chunk hashes
- Any integrity failure → cache rejection and error report

---

## 8. Cache and Eviction

### 8.1 Cache Policy

- TTL-based expiration (default: 3600s)
- LRU eviction when cache limit is reached
- Explicit purge via TaskEnd
- All cached data MUST be encrypted at rest

### 8.2 Cache Destruction

Upon task completion or failure:
- All cached media files MUST be securely wiped
- Encryption keys MUST be destroyed
- Token MUST be invalidated
- Protocol developers MUST implement verifiable cache destruction

---

## 9. Transcoding

### 9.1 Offload Protocol

- Origin sends raw media + transcoding parameters to peer
- Peer executes ffmpeg with configured parameters
- Peer returns transcoded result + metadata
- Origin verifies hash of transcoded output

### 9.2 Transcoding Parameters

```json
{
  "input_format": "mp4",
  "output_format": "mp4",
  "video_codec": "h264",
  "audio_codec": "aac",
  "resolution": "720p",
  "bitrate": "2M",
  "max_duration_seconds": 300
}
```

### 9.3 Timeout Handling

- Transcoding tasks MUST NOT exceed `transcode_timeout_seconds`
- If timeout is reached, peer MUST kill ffmpeg process and report failure
- Origin MAY retry on a different peer or fall back to local transcoding

---

## 10. Multi-Source Retrieval

### 10.1 Chunk Manifest

```json
{
  "media_id": "media_01jxyzexample",
  "total_size_bytes": 536870912,
  "chunk_size_bytes": 10485760,
  "chunks": [
    {
      "index": 0,
      "hash": "sha256:chunk0hash...",
      "size_bytes": 10485760,
      "peers": ["ierp:edge1.example.net", "ierp:edge2.example.net"]
    }
  ]
}
```

### 10.2 Parallel Download

- Downloader requests chunk manifest from origin or any peer
- Downloader fetches each chunk from the fastest available peer
- Each chunk is verified by hash before acceptance
- Download speed scales with number of available peers

---

## 11. Receipt

```json
{
  "type": "Receipt",
  "task_id": "task_01jxyzexample",
  "profile": "media-relay",
  "started_at": "2026-01-15T08:00:00Z",
  "ended_at": "2026-01-15T12:00:00Z",
  "end_reason": "ttl_expired",
  "delivery_stats": {
    "media_served": 50,
    "total_bytes_served": 5368709120,
    "fetch_count": 500,
    "transcode_tasks": 10,
    "transcode_cpu_seconds": 1200
  }
}
```

---

## 12. Security Requirements

- Media MUST be encrypted at rest on assisting peers (AES-256-GCM)
- Encryption keys MUST be ephemeral and task-scoped
- Cache destruction MUST be verifiable
- Non-public media MUST NOT be relayed
- Admin approval mechanism REQUIRED (configurable: single/multi)
- Transcoded output MUST be hash-verified by origin

---

## 14. Temporary Tenant Context (TTC)

> **Full specification:** [spec/ierp-core.md](../spec/ierp-core.md)

TTC applicability for `media-relay` is **high**. Media distribution + transcoding naturally splits into multiple Tasks.

**Why TTC is useful:**

A single media file may require:

- multiple cache Tasks (peer-a serves original, peer-b serves original);
- one or more transcode Tasks (peer-c transcodes to 720p + serves).

All these Tasks are logically one session. TTC groups them for correlation.

**TTC mapping:**

```
TTC (lease_type: cache)
├── Task 1 (lease_type: cache) → peer-a (serve original media)
├── Task 2 (lease_type: cache) → peer-b (serve original media)
└── Task 3 (lease_type: relay) → peer-c (transcode + serve)
```

**Recommended `lease_type`:**

| Scenario | `lease_type` |
|----------|-------------|
| Pure media serving (no transcode) | `cache` |
| Transcoding offload | `relay` |
| Mixed distribution + transcode | `cache` (parent TTC) |

**Operator benefits:**

- track total bandwidth served across all peers for one media file;
- correlate transcoding Tasks with distribution Tasks;
- compute per-TTC Receipts for capacity planning.

---

## 15. Implementation Checklist

Protocol developers implementing media-relay MUST:

- [ ] Implement media encryption (AES-256-GCM or equivalent)
- [ ] Implement chunk manifest generation and hash verification
- [ ] Implement HTTP Range request support
- [ ] Implement cache with TTL expiration and LRU eviction
- [ ] Implement secure cache destruction
- [ ] Implement ffmpeg transcoding offload (optional)
- [ ] Implement transcode timeout handling
- [ ] Implement multi-source parallel download
- [ ] Implement admin approval mechanism
- [ ] Implement fetch count tracking and offload triggering
- [ ] Generate Receipts with media serving stats
