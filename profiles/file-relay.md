# Profile: file-relay

> **Profile ID:** `file-relay`
> **IERP Core Version:** ≥ 0.1
> **Status:** Draft
> **License:** CC BY-SA 4.0

---

## 1. Purpose

The `file-relay` profile enables temporary assistance with large file transfer or synchronization. When an origin instance needs to distribute a large file to multiple peers, assisting instances temporarily cache encrypted chunks for parallel download.

---

## 2. Use Cases

- Large file synchronization between trusted instances
- Multi-source chunk distribution for faster downloads
- Temporary edge caching for frequently requested files
- Software distribution across federated instances

---

## 3. Task Semantics

### 3.1 Task Lifecycle

```
Origin → TaskOffer(profile: file-relay, subject: file_manifest)
    ↓
Peer → TaskClaim
    ↓
Origin → TaskGrant(chunk_manifest, relay_endpoint)
    ↓
Peer caches chunks (encrypted, hash-verified)
    ↓
Downloaders retrieve chunks from multiple peers
    ↓
Peer → TaskHeartbeat
    ↓
Origin → TaskEnd (transfer complete or TTL expired)
    ↓
Both → Receipt
    ↓
Auto-destroy: cached chunks
```

### 3.2 Chunk Flow

- Origin generates chunk manifest with SHA-256 hashes
- Origin distributes chunks to assisting peers
- Downloaders fetch chunks from multiple peers in parallel
- Each chunk is verified by hash before acceptance

---

## 4. Configurable Parameters

| Parameter | Default | Type | Description |
|-----------|---------|------|-------------|
| `cache_ttl_seconds` | 3600 | integer | Chunk cache lifetime |
| `heartbeat_interval_seconds` | 60 | integer | Heartbeat frequency |
| `heartbeat_miss_threshold` | 3 | integer | Consecutive misses before abort |
| `max_cache_size_bytes` | 1073741824 | integer | Max cache per peer (1GB) |
| `chunk_size_bytes` | 10485760 | integer | Default chunk size (10MB) |
| `admin_approval_required` | `false` | boolean | Auto-approve based on policy |

---

## 5. Quota Structure

```json
{
  "bandwidth_out_mbps": 100,
  "connections": 50,
  "max_file_size_bytes": 10737418240,
  "max_cache_size_bytes": 1073741824
}
```

---

## 6. Chunk Manifest

```json
{
  "file_id": "file_01jxyzexample",
  "file_hash": "sha256:abcdef1234567890...",
  "total_size_bytes": 5368709120,
  "chunk_size_bytes": 10485760,
  "chunks": [
    {
      "index": 0,
      "hash": "sha256:chunk0hash...",
      "size_bytes": 10485760
    },
    {
      "index": 1,
      "hash": "sha256:chunk1hash...",
      "size_bytes": 10485760
    }
  ]
}
```

---

## 7. Transport Requirements

- **Protocol:** HTTPS with HTTP/2 or QUIC
- **Range Requests:** HTTP Range or equivalent
- **Hash Verification:** SHA-256 per chunk
- **Encryption at Rest:** AES-256 for cached chunks

---

## 8. Privacy

- File content is encrypted at rest on assisting peers
- File metadata (name, type) is NOT shared with assisting peers
- Only chunk hashes and sizes are provided
- Cache expires automatically per `cache_ttl_seconds`

---

## 9. Security

- Chunk integrity via SHA-256 hash verification
- Encrypted cache storage
- Task-scoped access tokens
- Automatic cache destruction on task end

---

## 10. Receipt

```json
{
  "type": "Receipt",
  "task_id": "task_01jxyzexample",
  "profile": "file-relay",
  "started_at": "2026-01-15T08:00:00Z",
  "ended_at": "2026-01-15T10:00:00Z",
  "end_reason": "transfer_complete",
  "delivery_stats": {
    "total_chunks": 512,
    "served_chunks": 512,
    "failed_chunks": 0,
    "total_bytes_served": 5368709120
  },
  "quota_used": {
    "bandwidth_out_mbps_avg": 80,
    "connections_peak": 45
  }
}
```
