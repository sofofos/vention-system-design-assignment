# Video Upload System Design

**Context:** 5 developers, 6-month timeline, 20-50k DAU (Canada + Europe), mobile and desktop uploads.

---

## 1. System Architecture Diagram

```mermaid
sequenceDiagram
    participant Client as Browser/Mobile
    participant API as Rails API
    participant Spaces as DO Spaces
    participant Sidekiq as Sidekiq Worker
    participant Coconut as Coconut.co

    Client->>API: POST /api/v1/videos {title, filename, content_type}
    API->>API: Create Video record (status: pending)
    API->>Spaces: Generate presigned upload URL
    API-->>Client: {video_id, upload_url, storage_key}

    Client->>Spaces: PUT file to presigned URL
    Spaces-->>Client: 200 OK

    Client->>API: POST /api/v1/videos/:id/upload_complete
    API->>Sidekiq: Queue HandleUploadCompleteJob
    API-->>Client: {status: processing_queued}

    Sidekiq->>Spaces: Verify file exists
    Sidekiq->>Spaces: Generate presigned download URL
    Sidekiq->>Coconut: Create transcoding job (source URL, output config)
    Coconut-->>Sidekiq: {job_id}
    Sidekiq->>API: Update Video (status: processing)

    Coconut->>Spaces: Fetch source video
    Coconut->>Coconut: Transcode to HLS (360p, 480p, 720p, 1080p)
    Coconut->>Spaces: Store processed files + thumbnails

    Coconut->>API: POST /webhooks/coconut {job.completed}
    API->>API: Update Video (status: ready, hls_url, thumbnail_url)

    Client->>API: GET /api/v1/videos/:id
    API-->>Client: {playback_url, thumbnail_url, status: ready}
```

---
## System Components  
  
```mermaid
flowchart LR
	subgraph Client
	    UI[Browser UI]
	  end  
	
	subgraph "DigitalOcean App Platform"
		API[Rails API]
		Sidekiq[Sidekiq Workers]
	end  
    
    subgraph "DigitalOcean Managed Services"
    
	    PG[(PostgreSQL)]        
	    Redis[(Redis)]   
	 end  
     
    subgraph "DigitalOcean Spaces"
		 Raw[raw uploads]
			Processed[processed videos]
			CDN[CDN Edge]  
    end
      
    subgraph "External Services"
			Coconut[Coconut.co]    
    end
      
    UI -->|API requests| API
		UI -->|Direct upload| Raw
		UI -->|Stream video| CDN  
		
    API --> PG    
    API --> Redis    
    Sidekiq --> Redis    
    Sidekiq --> PG    
    Sidekiq -->|Submit job| Coconut  
    
    Coconut -->|Fetch source| Raw    
    Coconut -->|Store output| Processed    
    Coconut -->|Webhook| API  
    
    CDN --> Processed
```

---

**Video State Machine:**

```mermaid
stateDiagram-v2
    [*] --> pending: Video created
    pending --> uploading: Presigned URL generated
    uploading --> uploaded: Client confirms upload
    uploaded --> processing: Coconut job submitted
    processing --> ready: Webhook success
    processing --> errored: Webhook failed
    uploading --> errored: Upload verification failed
```

---


## 2. Technology Stack

| Layer | Choice | Rationale |
|-------|--------|-----------|
| API | Ruby on Rails | fast development, good ecosystem |
| Database | PostgreSQL | Video metadata is relational, mature, reliable |
| Object Storage | DigitalOcean Spaces | S3-compatible, $5/250GB, EU regions available |
| Background Jobs | Sidekiq + Redis | Simple, proven, handles async workflows well |
| Transcoding | Coconut.co | Managed transcoding API, S3-compatible output, cheaper than Mux, eliminates FFmpeg ops burden |
| CDN | Digital Ocean | Free tier generous, global network, easy setup |
| Output Format | HLS | Adaptive bitrate streaming, broad device support |

**Why not:**
- Self-hosted FFmpeg: Ops burden too high for 5 devs
- Kafka/RabbitMQ: Overkill for this scale, Sidekiq sufficient
- Kubernetes: ECS/Fargate simpler; K8s expertise expensive

---

## 3. Design Trade-offs & Assumptions

**Assumptions:**
- Max file size: 5GB
- Presigned URL expiry: 15 minutes
- Rate limit: 10 uploads/hour per user
- Output resolutions: 360p, 480p, 720p, 1080p

**Trade-offs:**

| Decision | Trade-off |
|----------|-----------|
| Managed transcoding vs self-hosted | Higher per-minute cost (~$0.015/min) but eliminates ops burden. Worth it for 5 devs. |
| Async transcoding | User sees "processing" for minutes after upload. Better UX than blocking, simpler architecture. |
| Multi-region | For Canada + Europe coverage, would add second region (ca-central-1) behind Cloudflare load balancer with geo-routings|
| Transcode all resolutions upfront | Wastes compute on unwatched videos. Simpler than on-demand, predictable costs. |
| Eventual consistency for status | User may briefly see stale status. Acceptable, avoids distributed transactions. |
| Keep original uploads | Higher storage cost but enables re-transcoding for future formats. |
| Presigned URLs vs proxy | API doesn't handle video bytes (better scalability). URL sharing mitigated by short expiry. |

---

## 4. API Specification

**Create Video**
```
POST /api/v1/videos
Content-Type: application/json

Request:  { "title": "My Video", "filename": "video.mp4", "content_type": "video/mp4" }
Response: { "video_id": "abc123", "upload_url": "https://...", "storage_key": "uploads/abc123/video.mp4" }
```

**Confirm Upload**
```
POST /api/v1/videos/:id/upload_complete

Response: { "status": "processing_queued" }
```

**Get Video**
```
GET /api/v1/videos/:id

Response: { "id": "abc123", "title": "My Video", "status": "ready", "playback_url": "https://...", "thumbnail_url": "https://..." }
```

**Webhook (Coconut)**
```
POST /webhooks/coconut
{ "job_id": "...", "status": "completed", "outputs": { "hls": "https://...", "thumbnail": "https://..." } }
```

---

## 5. Transcoding Configuration 

```json
{
  "input": { "url": "https://spaces.../uploads/abc123/video.mp4" },
  "storage": { "service": "s3", "bucket": "processed-videos" },
  "notification": { "url": "https://api.example.com/webhooks/coconut" },
  "outputs": {
    "httpstream": [{
      "hls": { "path": "/videos/abc123/" },
      "variants": [
        "mp4:360p::quality=4,maxrate=1000k",
        "mp4:480p::quality=4,maxrate=1500k",
        "mp4:720p::quality=4,maxrate=2000k",
        "mp4:1080p::quality=4,maxrate=4000k"
      ]
    }]
  }
}
```

---

## 6. Operational Notes 

**Fault tolerance:**
- Failed transcoding jobs go to dead-letter queue for manual inspection
- Files failing validation twice are marked as errored and user is notified
- Resumable uploads (multipart) for mobile users and large files

**Storage structure:**
```
/uploads/{video_id}/original.mp4
/processed/{video_id}/playlist.m3u8
/processed/{video_id}/360p/
/processed/{video_id}/720p/
/processed/{video_id}/thumbnail.jpg
```

**Monitoring:**
- Track upload success rate, transcoding latency, error rates
- Alert on webhook failures, queue depth spikes
- 

