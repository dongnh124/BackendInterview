# 🏗️ Architecture Scenarios — Bài Toán Thiết Kế Media Trên AWS

> 5 kịch bản thiết kế kiến trúc video streaming thường gặp trong phỏng vấn Mid/Senior và Solutions Architect. Mỗi kịch bản có requirements, sơ đồ, trade-offs và điểm cần nhấn khi trình bày.

---

## 📋 Framework Tiếp Cận System Design (Media)

Khi interviewer đưa bài toán, áp dụng thứ tự sau:

```
1. Clarify (2–3 phút)
   → VOD / Live / cả hai? Concurrent viewers? Regions?
   → DRM? Ads? Latency target? Budget / team size?

2. Success criteria
   → Availability (99.9% vs 99.99%)
   → Startup time (< 2s?), rebuffer rate
   → Concurrent peak, ingest sources

3. High-level pipeline (vẽ boxes)
   Ingest → Encode → Package → Origin → CDN → Player

4. Drill-down từng hop + failure mode

5. Trade-offs + cost drivers (egress, MediaLive hours, transcoding minutes)
```

---

## 🔴 Scenario 1: Nền Tảng OTT VOD — Netflix-Style (Startup → Scale)

### Yêu Cầu

```
Business: Nền tảng học trực tuyến, 50K MAU, tăng trưởng nhanh
Content: 10K giờ VOD, upload từ CMS, format đa dạng (MOV, MP4)
Devices: Web (Chrome/Safari), iOS, Android
Security: Một số khoá học trả phí — cần signed URL
Scale: Peak 5K concurrent viewers, 95% xem VOD
Latency: Startup < 3s, không yêu cầu live
Budget: Startup — tối ưu chi phí, team 2 engineer
```

### Kiến Trúc Đề Xuất

```
┌────────────── CMS / Admin API ──────────────┐
│  Upload metadata → RDS                      │
│  Upload file → S3 presigned URL (source)    │
└────────────────────┬────────────────────────┘
                     │ S3 Event
                     ▼
            ┌─────────────────┐
            │ Lambda / SFN    │  Step Functions — Điều Phối Workflow
            │ Trigger Job     │
            └────────┬────────┘
                     ▼
            ┌─────────────────┐
            │ MediaConvert    │  HLS + CMAF, ladder 360p–1080p
            │ On-demand Queue │
            └────────┬────────┘
                     ▼
            ┌─────────────────┐
            │ S3 Output       │  /courseId/videoId/*.m3u8
            └────────┬────────┘
                     ▼
            ┌─────────────────┐
            │ CloudFront      │  OAC, cache segment dài, manifest ngắn
            │ Signed URL      │
            └────────┬────────┘
                     ▼
            ┌─────────────────┐
            │ Player          │  HLS.js / native AVPlayer / ExoPlayer
            └─────────────────┘
```

### Trade-offs

| Quyết Định | Lựa Chọn | Lý Do |
| ---------- | -------- | ----- |
| MediaPackage vs S3 origin | S3 trực tiếp | VOD tĩnh, không cần JIT; đơn giản hơn |
| CMAF vs HLS-only | CMAF nếu cần Android+Web; HLS-only nếu chỉ web+iOS giai đoạn 1 | Giảm storage khi đa platform |
| Orchestration | Step Functions | Retry job, parallel profiles, audit trail |
| DRM giai đoạn 1 | Signed URL thay DRM | Đủ cho "trả phí" MVP; DRM khi có app native bắt buộc |

### Điểm Cần Nói Trong Phỏng Vấn

1. **Idempotent jobs:** Cùng `videoId` không tạo duplicate job — dùng DynamoDB job status
2. **Thumbnail / preview:** MediaConvert Frame Capture hoặc Lambda@Edge
3. **Cost:** Transcoding một lần; egress CloudFront dominate ở scale — theo dõi cache hit ratio
4. **Evolution:** Khi cần offline mobile DRM → thêm MediaPackage VOD endpoints + Speke

📖 Tham chiếu: [02-mediaconvert/](../02-mediaconvert/README.md), [09-cdn-delivery/](../09-cdn-delivery/README.md)

---

## 🟠 Scenario 2: Live Sports OTT — 1 Triệu Concurrent Viewers

### Yêu Cầu

```
Event: Trận cup, 1M peak concurrent, toàn cầu
Latency: 20–30s acceptable (standard OTT)
Input: Contribution từ OB van qua MediaConnect + backup RTMP
Ads: National/regional ad pods — SCTE-35
DRM: Không (free-to-air) hoặc optional premium tier
SLA: Không được gián đoạn > 30s trong hiệp đấu
```

### Kiến Trúc

```
[OB Encoder] ──primary──▶ MediaConnect ──▶ MediaLive (Standard, 2 pipeline)
                backup───▶ RTMP Push ────────────────┘
                              │
                              ▼
                    ┌──────────────────┐
                    │ MediaPackage     │  HLS endpoints, 6–8 renditions
                    │ (live origin)    │
                    └────────┬─────────┘
                             │
              ┌──────────────┼──────────────┐
              ▼              ▼              ▼
        (optional)     ┌──────────┐   Multi-region
        MediaTailor    │CloudFront│   viewers
        SSAI + ADS     │ + Shield │   
              │        └────┬─────┘
              └─────────────┘
                            ▼
                      [Players]
```

### Scale & CDN

```
1M viewers × ~2 Mbps average ≈ 2 Tbps aggregate (lý thuyết cao)
Thực tế: CDN cache hit 90–98% trên segment
Origin chỉ phục vụ: live edge + manifest misses

Actions:
- Origin Shield enabled
- Separate cache behaviors: *.m3u8 (TTL thấp) vs *.ts|*.m4s (TTL cao)
- AWS event support: limit review trước game day
```

### Failure Modes

| Sự Cố | Giải Pháp |
| ----- | --------- |
| Primary ingest loss | Automatic input failover → backup RTMP |
| Pipeline A failure | Standard channel failover pipeline B |
| Origin overload | CloudFront scale; pre-warm nếu có thể |
| Ad marker lỗi | Slate + manual SCTE inject qua Schedule Action |

### Trade-offs

| Quyết Định | Lựa Chọn | Lý Do |
| ---------- | -------- | ----- |
| IVS vs MediaLive | MediaLive | SCTE-35, MediaConnect, broadcast SLA |
| Low-latency HLS | Không (giai đoạn 1) | 1M scale + LL-HLS phức tạp hơn |
| Multi-region origin | CloudFront đủ; origin một region nếu AWS advise | Chi phí vs latency |

📖 Tham chiếu: [03-medialive/3-redundancy-failover.md](../03-medialive/3-redundancy-failover.md), [09-cdn-delivery/1-cloudfront-for-media.md](../09-cdn-delivery/1-cloudfront-for-media.md)

---

## 🟡 Scenario 3: FAST Channel — Free Ad-Supported TV

### Yêu Cầu

```
Model: Kênh linear 24/7 từ thư viện VOD + live breaks
Ads: SSAI — Server-Side Ad Insertion cá nhân hoá theo vùng
EPG: Lịch phát chương trình (metadata API)
Compliance: Ad reporting (impression beacons)
```

### Kiến Trúc

```
VOD Library (S3)
    │
    ├── MediaTailor Channel Assembly  ← lắp playlist linear
    │
Live contribution (optional events)
    │
    └── MediaLive (SCTE-35 cues) ──▶ MediaPackage
                │
                ▼
        ┌───────────────┐
        │ MediaTailor   │  Playback configuration + ADS URL
        │ SSAI          │
        └───────┬───────┘
                ▼
        ┌───────────────┐
        │ CloudFront    │  Manifest dynamic — TTL ngắn
        └───────────────┘
```

### Điểm Kỹ Thuật Quan Trọng

1. **SCTE-35** từ live; VOD-only channel cần **ad break markers** nhúng trong timeline Assembly
2. **ADS** — Ad Decision Server — trả VAST; MediaTailor stitch segment
3. **Beacons** — tracking quartile/impression — không được block bởi player
4. **Prefetch ads** giảm black screen khi vào ad break

📖 Tham chiếu: [06-mediatailor/](../06-mediatailor/README.md)

---

## 🟢 Scenario 4: Interactive Live Commerce — IVS-First

### Yêu Cầu

```
Use case: Livestream bán hàng, 20K concurrent, chat, flash sale overlay
Latency: < 5 giây (host nói → viewer nghe gần realtime)
Team: Nhỏ, cần ship trong 4 tuần
Recording: Lưu replay 7 ngày
```

### Kiến Trúc

```
[OBS / Mobile SDK]
    RTMPS ingest
        ▼
┌───────────────────┐
│ IVS Channel       │  Low-latency mode
│ + Timed Metadata  │  ← flash sale events
└─────────┬─────────┘
          │
    ┌─────┴─────┐
    ▼           ▼
IVS Chat    IVS Recording → S3 (HLS)
    │           │
    ▼           ▼
Web/Mobile   Replay player (cùng SDK)
```

### Trade-offs

| Quyết Định | Lựa Chọn | Lý Do |
| ---------- | -------- | ----- |
| MediaLive stack | Không (giai đoạn 1) | IVS managed, latency thấp, SDK sẵn |
| Timed metadata | IVS native | Đồng bộ UI không cần websocket riêng cho cue |
| DRM | Thường không cần | Public live commerce |
| Scale 20K | IVS managed | Không tự tính CDN |

**Khi migrate:** Nếu cần SSAI broadcast quality → tách "showroom" IVS và "main TV" MediaLive.

📖 Tham chiếu: [07-ivs/](../07-ivs/README.md)

---

## 🔵 Scenario 5: Multi-DRM Premium OTT — Global

### Yêu Cầu

```
Subscription streaming, 500K subscribers
Platforms: iOS, Android, Web, Smart TV
DRM: Widevine L1 + FairPlay + PlayReady
Offline download: Có (mobile)
Piracy concern: Cao — watermarking xem xét
```

### Kiến Trúc DRM

```
MediaConvert (hoặc MediaLive cho live linear)
        ▼
MediaPackage Endpoints:
    ├── Endpoint HLS + FairPlay
    ├── Endpoint DASH + Widevine
    └── (TV) DASH + PlayReady
        │
        SPEKE → API Gateway → Lambda → EZDRM / Axinom
        │
CloudFront (signed cookies — session multi-URL)
        ▼
Players với CDM — Content Decryption Module
```

### Điểm Phỏng Vấn

1. **Speke** không thay license server — chỉ trao key cho packager
2. **Certificate FairPlay** — Apple approval process
3. **Key rotation** — policy theo studio content
4. **CloudFront Signed Cookies** thay URL khi player tải nhiều manifest+segment
5. **Forensic watermarking** (optional) — Nagra, Irdeto — ngoài AWS native

📖 Tham chiếu: [04-mediapackage/3-drm-speke.md](../04-mediapackage/3-drm-speke.md), [09-cdn-delivery/2-signed-url-cookies.md](../09-cdn-delivery/2-signed-url-cookies.md)

---

## 📊 Ma Trận Chọn Stack — Tóm Tắt Một Trang

| Scenario | Ingest | Encode/Package | Delivery | Ads | DRM |
| -------- | ------ | -------------- | -------- | --- | --- |
| VOD MVP | S3 | MediaConvert | CloudFront+S3 | - | Signed URL |
| Live mass | MediaConnect/RTMP | MediaLive+MediaPackage | CloudFront | MediaTailor | Speke |
| Interactive | RTMPS | IVS | IVS CDN | Overlay/client | Optional |
| IoT | Producer SDK | KVS | KVS HLS / app | - | IAM |
| FAST | Assembly+Live | MediaTailor+ML+MP | CloudFront | SSAI | Optional |

---

## 🎯 Checklist Trước Khi Kết Thúc Phỏng Vấn Design

- [ ] Đã nêu **latency** và **availability** số liệu
- [ ] Đã vẽ **failure path** (ít nhất 2 scenarios)
- [ ] Đã nhắc **monitoring** (CloudWatch, MediaLive alerts, CF logs)
- [ ] Đã ước lượng **cost driver** (egress, channel hours, TC minutes)
- [ ] Đã nói **migration/evolution** nếu startup → enterprise

---

**Cập Nhật Lần Cuối:** 2026-06-04  
**Liên quan:** [INTERVIEW_GUIDE.md](./INTERVIEW_GUIDE.md) | [star-stories.md](./star-stories.md)
