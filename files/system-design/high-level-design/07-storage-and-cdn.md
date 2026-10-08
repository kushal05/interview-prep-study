# Storage Systems & CDN

> **Learning goal:** Understand storage types, object storage, file systems, and content delivery at scale.

---

## Before You Begin — Plain-English Setup

- **Persistent storage** = data that survives when the computer is turned off (vs. RAM, which is wiped on power-off).
- **HDD (Hard Disk Drive)** = spinning magnetic disk. Cheap, large, slow.
- **SSD (Solid State Drive)** = flash chips. More expensive, much faster than HDD.
- **File** = a named blob of bytes living in a folder hierarchy (`/users/me/photo.jpg`).
- **A blob** = "Binary Large OBject" — a chunk of bytes (image, video, zip, etc.).
- **Object storage** = a service that stores blobs by a unique key. No folders. Access via HTTP (e.g., AWS S3, Google Cloud Storage).
- **A bucket** = a top-level container in object storage. Think of it as a namespace.
- **CDN (Content Delivery Network)** = a network of servers spread worldwide that cache your static files near the user. Reduces distance the bytes have to travel.
- **PoP (Point of Presence)** = one CDN datacenter at a specific physical location (Tokyo, London, etc.).
- **Edge / origin** = "Edge" = the CDN PoP near the user. "Origin" = your main server far away. CDN serves from edge; falls back to origin on cache miss.
- **Static content** = files that don't change per user: JS, CSS, images, video segments.
- **Dynamic content** = generated per request, often personalized (your news feed, your cart).

---

## 1. Storage Types Overview

```
┌──────────────────────────────────────────────────────────────────┐
│                       Storage Hierarchy                          │
├──────────────┬───────────┬───────────┬──────────────────────────┤
│ Type         │ Latency   │ Cost/GB   │ Use Case                 │
├──────────────┼───────────┼───────────┼──────────────────────────┤
│ RAM/Cache    │ ~100 ns   │ $$$$$     │ Hot data, caching        │
│ SSD          │ ~16 μs    │ $$$       │ Databases, active data   │
│ HDD          │ ~2 ms     │ $$        │ Archives, logs           │
│ Object Store │ ~50 ms    │ $         │ Images, videos, backups  │
│ Tape/Glacier │ hours     │ ¢         │ Long-term archival       │
└──────────────┴───────────┴───────────┴──────────────────────────┘
```

---

## 2. Block Storage vs File Storage vs Object Storage

### Block Storage
```
How it works: Raw disk blocks, no file system overhead
Access: Mounted as a volume to ONE server

[Block 1][Block 2][Block 3][Block 4]...[Block N]
  ↑ OS treats this like a raw disk, manages file system on top

Examples: AWS EBS, Azure Disk, GCP Persistent Disk
Best for: Databases (need low-latency random I/O), OS boot volumes
```

### File Storage
```
How it works: Hierarchical file system, shared access via NFS/SMB
Access: Multiple servers can mount and access simultaneously

/home/
  ├── user1/
  │   ├── file1.txt
  │   └── file2.doc
  └── user2/
      └── file3.pdf

Examples: AWS EFS, Azure Files, NFS, GlusterFS
Best for: Shared file access, legacy apps, home directories
```

### Object Storage
```
How it works: Flat namespace, each object = data + metadata + unique key
Access: HTTP API (PUT/GET/DELETE), not mounted as disk

Bucket: "my-photos"
├── Key: "2024/vacation/beach.jpg"  → Object (blob + metadata)
├── Key: "2024/vacation/sunset.jpg" → Object (blob + metadata)
└── Key: "profile/avatar.png"       → Object (blob + metadata)

No folders! Keys are just strings (slashes are cosmetic)

Examples: AWS S3, Azure Blob Storage, Google Cloud Storage, MinIO
Best for: Images, videos, backups, static assets, data lake
```

### Comparison
| Aspect | Block | File | Object |
|--------|-------|------|--------|
| Access | Single server mount | Multi-server mount | HTTP API |
| Performance | Highest | Medium | Lower (HTTP overhead) |
| Scalability | Limited | Medium | Virtually unlimited |
| Cost | $$$ | $$ | $ |
| Metadata | Minimal | File system attributes | Rich custom metadata |
| Use case | Databases | Shared files | Media, backups, static |

---

## 3. Object Storage Deep Dive (S3 Model)

### Architecture
```
┌────────────────────────────────────────────┐
│                  S3 Service                 │
│                                            │
│  Bucket: "my-app-media"                    │
│  ├── images/photo1.jpg    (Standard)       │
│  ├── images/photo2.jpg    (Standard)       │
│  ├── videos/clip1.mp4     (Standard-IA)    │
│  └── archive/2023/        (Glacier)        │
│                                            │
│  Features:                                 │
│  • 99.999999999% durability (11 nines)     │
│  • Automatic replication across 3+ AZs     │
│  • Versioning (keep all versions of object)│
│  • Lifecycle policies (auto-move to cheaper tier) │
│  • Server-side encryption                  │
│  • Pre-signed URLs (temporary access)      │
└────────────────────────────────────────────┘
```

### Storage Tiers
```
Hot:        S3 Standard         — Frequent access, instant retrieval
Warm:       S3 Standard-IA      — Infrequent access, instant retrieval, lower cost
Cold:       S3 Glacier Instant  — Rare access, instant retrieval, much lower cost
Archive:    S3 Glacier Deep     — Archival, 12-hour retrieval, cheapest

Lifecycle policy example:
Day 0-30:   Standard ($0.023/GB)
Day 31-90:  Standard-IA ($0.0125/GB)
Day 91-365: Glacier ($0.004/GB)
Day 366+:   Glacier Deep ($0.00099/GB)
```

### Pre-Signed URLs Pattern
```
Problem: You want users to upload files directly to S3 without
         exposing your AWS credentials or routing through your server.

Solution:
1. Client → Your API: "I want to upload photo.jpg"
2. Your API → S3: Generate pre-signed URL (valid 15 min)
3. Your API → Client: Here's the pre-signed URL
4. Client → S3: PUT photo.jpg directly (no middleman)

Benefits:
- Server doesn't handle file bytes (saves bandwidth + CPU)
- Temporary, scoped access (can't abuse after expiry)
```

---

## 4. Content Delivery Network (CDN)

### How It Works
```
First request (cache miss):
User (Mumbai) → CDN Edge (Mumbai) → MISS → Origin (Virginia)
                                            │
                                     CDN Edge caches response
                                            │
User ← CDN Edge (Mumbai) ←─────────────────┘

Subsequent requests (cache hit):
User (Mumbai) → CDN Edge (Mumbai) → HIT → Response (20ms!)
(No trip to origin)
```

### CDN Architecture
```
                          ┌──────────────┐
                          │ Origin Server│
                          └──────┬───────┘
                                 │
                    ┌────────────┼────────────┐
                    ▼            ▼            ▼
              [Edge PoP      [Edge PoP    [Edge PoP
               Mumbai]        London]      Tokyo]
              /    \          /    \       /    \
           Users  Users   Users  Users Users  Users

PoP = Point of Presence (physical datacenter at edge location)
Major CDNs have 200+ PoPs worldwide
```

### What to Cache on CDN
```
Static Content (always cache):
✓ Images (JPEG, PNG, WebP, SVG)
✓ CSS, JavaScript bundles
✓ Fonts
✓ Videos (streaming segments)
✓ Static HTML pages

Dynamic Content (sometimes cache):
✓ API responses that don't change often (e.g., product catalog)
✓ Personalized pages with shared components
✗ User-specific data (profile, cart) — usually not cached

Cache-Control header:
Cache-Control: public, max-age=86400         (cache for 1 day)
Cache-Control: private, no-cache             (don't cache)
Cache-Control: public, s-maxage=3600, stale-while-revalidate=86400
```

### CDN Invalidation
```
Problem: You updated a file but CDN still serves the old version.

Solutions:
1. Cache busting (versioned URLs):
   /styles.css → /styles.v2.css or /styles.css?v=abc123
   New URL = new cache entry (instant "invalidation")
   ✓ Best practice — most reliable

2. Purge/Invalidate API:
   CDN API: "DELETE /cache/styles.css"
   CDN removes from all edge nodes
   ✗ Slow (propagation across all PoPs), expensive

3. Short TTL:
   Cache-Control: max-age=60
   CDN re-fetches every 60 seconds
   ✗ Reduces CDN effectiveness
```

### CDN Providers
| Provider | Strength |
|----------|----------|
| **CloudFront** | Deep AWS integration |
| **Cloudflare** | Free tier, DDoS protection, edge compute |
| **Akamai** | Largest network, enterprise |
| **Fastly** | Real-time purging, edge compute (Wasm) |

---

## 5. Blob Storage Design Patterns

### Image/Media Upload Pipeline
```
Client → Pre-signed URL → Upload to S3
                              │
                              ▼
                    S3 Event Notification
                              │
                              ▼
                    [Processing Queue (SQS)]
                              │
                    ┌─────────┼──────────┐
                    ▼         ▼          ▼
              [Thumbnail]  [Compress]  [Metadata]
              [Generator]  [/Transcode] [Extractor]
                    │         │          │
                    ▼         ▼          ▼
              S3 (thumbs)  S3 (optimized)  Database
                    │         │
                    ▼         ▼
                    CDN (serves to users)
```

### Video Streaming Architecture
```
Upload → Transcoding Pipeline:
  Original (4K, 50GB) → [Transcoder]
                          ├── 1080p (2GB)
                          ├── 720p (1GB)
                          ├── 480p (500MB)
                          └── 360p (250MB)
                          Each split into segments (2-10 sec chunks)

Streaming (Adaptive Bitrate):
Client → CDN → Manifest file (.m3u8 / .mpd)
                Lists all quality levels + segment URLs
Client monitors bandwidth → requests appropriate quality
Bandwidth drops → seamlessly switches to lower quality
```

---

## 6. Distributed File Systems

### HDFS (Hadoop Distributed File System)
```
Architecture:
┌─────────────┐
│  NameNode    │  ← Metadata: which blocks are where
│  (Master)    │
└──────┬──────┘
       │
┌──────┼──────┬──────────────┐
▼      ▼      ▼              ▼
[DN1] [DN2] [DN3]  ...    [DNn]   ← DataNodes: store actual blocks

File "data.csv" (300MB):
├── Block 1 (128MB) → DN1, DN3, DN5 (3 replicas)
├── Block 2 (128MB) → DN2, DN4, DN6 (3 replicas)
└── Block 3 (44MB)  → DN1, DN4, DN5 (3 replicas)
```
- Block size: 128MB (vs 4KB for local file system) — optimized for large files
- Replication factor: 3 (default)
- Best for: Big data batch processing (MapReduce, Spark)

---

## Active Recall Questions

1. Explain the difference between block, file, and object storage. When would you use each?
2. What are S3 storage tiers? Design a lifecycle policy for a photo-sharing app.
3. What is a pre-signed URL? Why is it better than routing uploads through your server?
4. How does a CDN reduce latency? Walk through the first request vs subsequent requests.
5. What are 3 ways to invalidate CDN cache? Which is best and why?
6. Design a media upload pipeline for Instagram (upload → process → serve).
7. Explain adaptive bitrate streaming. How does Netflix handle different network speeds?
8. What is HDFS? Why is the block size 128MB instead of 4KB?
9. How would you store and serve 1 billion images? Estimate the storage needed.
10. When would you use a CDN for API responses (not just static files)?
