# Profile: preview-relay

> **Profile ID:** `preview-relay`
> **IERP Core Version:** ≥ 0.1
> **Status:** Draft
> **License:** CC BY-SA 4.0

---

## 1. Purpose

The `preview-relay` profile solves the link preview request amplification problem in federated social platforms. When a post containing a link is broadcast across the fediverse, each receiving instance independently fetches the link to generate a preview card (OGP). This creates a "thundering herd" effect — a single post can trigger 1,000+ GET requests to the target website, with traffic amplification ratios exceeding 36,000:1.

This profile allows instances to share pre-fetched, cryptographically verified link previews through trusted IERP peers, eliminating redundant requests to the target website.

---

## 2. Problem Statement

### 2.1 The Mastodon Stampede

Current ActivityPub implementations (Mastodon, Misskey, Pleroma) generate link previews independently per instance:

```
Post with link → Instance A fetches preview
               → Instance B fetches preview
               → Instance C fetches preview
               → ... (hundreds or thousands of instances)
```

**Measured impact (real-world test):**
- 1 post with link → 1,147 GET requests to target website
- Request amplification: 1,147:1
- Traffic amplification: 36,704:1 (114.7 MB of outbound traffic from a 3KB POST)
- Peak request rate: 35 requests/second to a single target

### 2.2 Why This Matters

- Small websites become unresponsive under the request burst
- CDN costs spike for content creators
- Instances near the target website are disproportionately affected
- No protocol-level solution exists in ActivityPub (Mastodon 4.4 still in planning)

---

## 3. Use Cases

### UC-1: Preview Sharing

- Post with link is broadcast to 100+ instances — only origin fetches preview
- Viral content triggers preview requests from thousands of instances
- Link preview sharing reduces redundant HTTP requests by 99%+
- Target website receives O(1) requests instead of O(N) where N = number of instances

### UC-2: Delegated Preview Fetch

When a single post triggers大规模 link preview requests across the fediverse, the origin instance may delegate preview fetching to a trusted peer instead of fetching locally:

- Origin issues `preview-relay` TaskOffer with target URL
- Assisting peer fetches OGP data from the target website (1 request)
- Assisting peer caches and serves preview to other instances
- Origin and other peers retrieve preview from the assisting peer
- Target website receives O(1) requests instead of O(N)

**Constraints (mandatory):**
- Only public links may be delegated for preview fetching
- Strict quota enforcement (bandwidth, connections, request count)
- Short TTL (recommended: 600s default for delegated fetches)
- Assisting instance MUST NOT store sensitive content
- Non-public links MUST NOT be relayed or fetched by peers

---

## 4. Task Semantics

### 4.1 Preview Sharing Flow

```
Origin instance fetches OGP preview (HTML + image)
    ↓
Origin encrypts preview data with per-task key
    ↓
Origin → TaskOffer(profile: preview-relay, subject: link_url)
    ↓
Peer → TaskClaim
    ↓
Origin → TaskGrant(preview_endpoint, access_token)
    ↓
Peer fetches encrypted preview from origin's relay endpoint
    ↓
Peer decrypts and serves preview locally (no fetch to target site)
    ↓
Peer → TaskHeartbeat
    ↓
Origin → TaskEnd (preview expires or post reaches TTL)
    ↓
Both → Receipt
    ↓
Auto-destroy: cached preview, encryption key
```

### 4.2 Preview Data Structure

```json
{
  "url": "https://example.com/article/123",
  "fetched_at": "2026-01-15T08:00:00Z",
  "ogp": {
    "title": "Article Title",
    "description": "Article description...",
    "image": "https://example.com/image.jpg",
    "image_hash": "sha256:abcdef...",
    "type": "article",
    "site_name": "Example Site"
  },
  "favicon": "https://example.com/favicon.ico",
  "fetch_latency_ms": 230,
  "status_code": 200
}
```

---

## 5. Configurable Parameters

### 5.1 Profile Defaults

| Parameter | Default | Type | Description |
|-----------|---------|------|-------------|
| `preview_ttl_seconds` | 3600 | integer | Preview cache lifetime |
| `preview_max_size_bytes` | 1048576 | integer | Max preview data size (1MB) |
| `preview_refresh_threshold` | 0.8 | number | Refresh when TTL reaches 80% |
| `heartbeat_interval_seconds` | 60 | integer | Heartbeat frequency |
| `max_preview_size_image` | 5242880 | integer | Max preview image size (5MB) |
| `admin_approval_mode` | `"single"` | enum | Approval mode: `single`, `multi` |

### 5.2 Instance-Level Overrides

All profile defaults MAY be overridden at the instance level.

---

## 6. Quota Structure

```json
{
  "previews_per_task": 100,
  "bandwidth_out_mbps": 10,
  "connections": 50,
  "max_cache_size_bytes": 104857600
}
```

---

## 7. Preview Verification

### 7.1 Origin Authority

- Origin instance fetches and signs the preview data
- Preview includes `fetched_at` timestamp for freshness verification
- Receiving instances MAY verify freshness against `preview_ttl_seconds`

### 7.2 Trust Model

- Preview data is signed by origin instance's Ed25519 key
- Assisting peers verify signature before caching
- Receiving instances verify signature before displaying
- If preview data is tampered with, signature verification fails

### 7.3 Staleness Handling

- Preview expires after `preview_ttl_seconds`
- If a receiving instance detects stale preview, it MAY:
  - Display "preview unavailable" with link
  - Request fresh preview from origin (not re-fetch target website)
  - Use cached version with staleness indicator

---

## 8. Content Handling

### 8.1 Preview Scope

| Content Type | Relay Behavior |
|-------------|---------------|
| Public posts with links | Preview MAY be relayed |
| Unlisted posts with links | Preview MAY be relayed |
| Private posts with links | Preview MUST NOT be relayed |
| Direct posts with links | Preview MUST NOT be relayed |

### 8.2 Image Handling

- Preview images are cached with SHA-256 hash verification
- Images are encrypted at rest using AES-256
- Image cache TTL matches preview TTL
- Large images (> `max_preview_size_image`) are rejected

---

## 9. Cache and Eviction

### 9.1 Cache Key

Preview cache is keyed by normalized URL:
- URL normalization: lowercase, strip trailing slash, remove fragment
- Same URL → same preview cache entry

### 9.2 Eviction Policy

- TTL-based expiration (default: 3600s)
- LRU eviction when cache limit is reached
- Explicit purge via TaskEnd
- Origin instance MAY push preview invalidation

---

## 10. Integration with ActivityPub

### 10.1 Protocol Interaction

```
Origin: Create activity with link
    ↓
Origin: Fetches OGP preview (1 request to target)
    ↓
Origin: Broadcasts activity + preview data via IERP preview-relay
    ↓
Peer: Receives activity + encrypted preview
    ↓
Peer: Decrypts preview, serves locally to its users
    ↓
Peer: NO additional request to target website
```

### 10.2 Fallback Behavior

If preview-relay is unavailable:
- Instance falls back to standard OGP fetching (current behavior)
- No degradation in functionality
- Increased load on target website (expected behavior)

---

## 11. Receipt

```json
{
  "type": "Receipt",
  "task_id": "task_01jxyzexample",
  "profile": "preview-relay",
  "started_at": "2026-01-15T08:00:00Z",
  "ended_at": "2026-01-15T12:00:00Z",
  "end_reason": "ttl_expired",
  "delivery_stats": {
    "previews_served": 150,
    "total_bandwidth_bytes": 52428800,
    "cache_hit_rate": 0.85
  }
}
```

---

## 12. Security Requirements

- Preview data MUST be signed by origin instance
- Preview data MUST be encrypted at rest on assisting peers
- Preview cache MUST be destroyed on TaskEnd or TTL expiration
- Non-public previews MUST NOT be relayed
- Admin approval mechanism REQUIRED (configurable: single/multi)

---

## 13. Implementation Checklist

Protocol developers implementing preview-relay MUST:

- [ ] Implement OGP fetching and parsing
- [ ] Implement preview data signing (Ed25519)
- [ ] Implement preview encryption (AES-256 or equivalent)
- [ ] Implement preview cache with TTL expiration
- [ ] Implement URL normalization for cache keying
- [ ] Implement image size limits and hash verification
- [ ] Implement admin approval mechanism
- [ ] Implement signature verification on receiving end
- [ ] Implement fallback to standard OGP when preview-relay unavailable
- [ ] Generate Receipts with preview serving stats
