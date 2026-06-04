# 🎯 Interview Prep — Chuẩn Bị Phỏng Vấn AWS Media Services

> Hướng dẫn toàn diện giúp bạn tự tin trả lời câu hỏi về AWS Media Services — từ lý thuyết codec/protocol đến system design OTT và câu chuyện xử lý sự cố live stream.

## 📁 Cấu Trúc Thư Mục

```
10-interview-prep/
├── README.md                    Tổng quan & checklist (file này)
├── INTERVIEW_GUIDE.md           Top 20 câu hỏi AWS Media kèm đáp án chi tiết
├── architecture-scenarios.md    Bài toán thiết kế: OTT, live streaming, ads
└── star-stories.md              Mẫu câu chuyện thực tế theo phương pháp STAR
```

---

## 🎯 Mục Tiêu Của Section Này

Sau khi hoàn thành `10-interview-prep/`, bạn có thể:

- ✅ Trả lời tự tin **top 20 câu hỏi AWS Media Services** thường gặp
- ✅ Kể **2–3 câu chuyện** theo định dạng STAR về media pipeline
- ✅ Thiết kế **kiến trúc OTT / live streaming** trên giấy trong 30–45 phút
- ✅ So sánh **trade-offs** giữa MediaLive, IVS, MediaConvert, KVS
- ✅ Ước tính **chi phí** và latency cho workload media cụ thể

---

## 📋 Danh Sách Kiểm Tra Trước Phỏng Vấn

### Kiến Thức Lý Thuyết — Cần Nắm Chắc

- [ ] Codec: H.264/AVC, H.265/HEVC, AV1 — khi nào dùng từng loại
- [ ] Container vs streaming format: MP4, HLS (.m3u8), DASH (.mpd), CMAF
- [ ] ABR — Adaptive Bitrate Streaming — Phát Trực Tuyến Thích Ứng Tốc Độ Bit
- [ ] Pipeline: Ingest → Encode → Package → Deliver
- [ ] DRM — Digital Rights Management — Widevine, FairPlay, PlayReady + Speke
- [ ] SCTE-35 — Society of Cable Telecommunications Engineers 35 — ad markers
- [ ] SSAI — Server-Side Ad Insertion — chèn quảng cáo phía máy chủ
- [ ] Latency: Standard HLS vs Low-latency vs WebRTC real-time

### Dịch Vụ AWS — Cần Phân Biệt Rõ

- [ ] **MediaConvert** — VOD transcoding (file-based)
- [ ] **MediaLive** — live encoding broadcast-grade
- [ ] **MediaPackage** — JIT packaging, DRM, time-shift
- [ ] **MediaTailor** — SSAI, Channel Assembly
- [ ] **IVS** — Interactive Video Service — live tương tác độ trễ thấp
- [ ] **KVS** — Kinesis Video Streams — IoT / camera / ML
- [ ] **CloudFront** — CDN cho manifest & segment

### Kỹ Năng Thực Hành — Nên Đã Làm

- [ ] Tạo MediaConvert job (Console hoặc CLI) với HLS output group
- [ ] Chạy MediaLive channel (OBS → RTMP) output tới MediaPackage
- [ ] Cấu hình CloudFront distribution với S3 origin + OAC
- [ ] Tạo IVS channel và phát bằng OBS + IVS player
- [ ] Đọc manifest HLS (.m3u8) và hiểu master vs media playlist

### Câu Chuyện Thực Tế — Cần Chuẩn Bị

- [ ] Sự cố live stream (input loss, encoding error, CDN cache)
- [ ] Tối ưu chi phí transcoding hoặc CDN egress
- [ ] Triển khai DRM hoặc bảo vệ nội dung trả phí
- [ ] Thiết kế / migration pipeline VOD hoặc OTT
- [ ] Latency hoặc chất lượng video (QoE — Quality of Experience)

---

## 🗺️ Lộ Trình Chuẩn Bị

### 2 Tuần Trước Phỏng Vấn

```
Tuần 1: Ôn lý thuyết & dịch vụ cốt lõi
- Ngày 1-2: 01-fundamentals/ (codec, HLS/DASH, ABR, DRM)
- Ngày 3-4: 02-mediaconvert/ + 09-cdn-delivery/ (VOD pipeline)
- Ngày 5-6: 03-medialive/ + 04-mediapackage/ (live pipeline)
- Ngày 7: 07-ivs/ + so sánh MediaLive vs IVS

Tuần 2: Practice & Polish
- Ngày 1-2: Đọc INTERVIEW_GUIDE.md — luyện trả lời nói to
- Ngày 3-4: Vẽ sơ đồ từ architecture-scenarios.md
- Ngày 5: Chuẩn bị stories từ star-stories.md (cá nhân hoá)
- Ngày 6: Mock interview (45 phút system design + 30 phút Q&A)
- Ngày 7: Ôn nhẹ checklist, nghỉ ngơi
```

### 1 Tuần Trước Phỏng Vấn

```
- Ôn top 20 câu hỏi — tập trung role (VOD vs Live vs Platform)
- Vẽ pipeline VOD và Live từ bộ nhớ (không nhìn tài liệu)
- Chuẩn bị 3 STAR stories (outage, cost, architecture)
- Review bảng so sánh dịch vụ trong INTERVIEW_GUIDE.md
```

### 1 Ngày Trước Phỏng Vấn

```
- Đọc lại checklist trong README này
- Ôn 10 câu hỏi quan trọng nhất cho vị trí mục tiêu
- Chuẩn bị câu hỏi hỏi ngược interviewer (xem bên dưới)
- Nghỉ ngơi đầy đủ
```

---

## 📊 Ma Trận Câu Hỏi Theo Cấp Độ

| Cấp Độ     | Loại Câu Hỏi                              | Ví Dụ                                                              |
| ---------- | ----------------------------------------- | ------------------------------------------------------------------ |
| Junior     | Định nghĩa, pipeline cơ bản               | "HLS khác DASH như thế nào?" "MediaConvert dùng cho gì?"           |
| Mid-level  | Cấu hình, troubleshooting, tích hợp       | "Thiết lập MediaLive Standard redundancy?" "DRM qua Speke?"        |
| Senior     | Trade-offs, scale, cost, SLA live         | "1M concurrent viewers — thiết kế CDN cache?"                      |
| Architect  | System design OTT, multi-DRM, FAST channel  | "Thiết kế nền tảng OTT có ads + DRM + time-shift toàn cầu"        |

---

## 🔀 Bảng So Sánh Nhanh — Hay Hỏi Phỏng Vấn

| Câu Hỏi So Sánh              | Chọn A Khi…                                      | Chọn B Khi…                                      |
| ---------------------------- | ------------------------------------------------ | ------------------------------------------------ |
| MediaConvert vs MediaLive    | File VOD, batch processing                       | Luồng live 24/7, SCTE-35, broadcast              |
| MediaLive vs IVS             | OTT broadcast, DRM, MediaPackage, SCTE-35        | Interactive live, chat, <5s latency, SDK sẵn   |
| HLS vs DASH                  | iOS/Safari ưu tiên, ecosystem Apple              | Android/TV đa nền tảng, MPEG-DASH chuẩn          |
| CMAF                         | Một bộ segment cho cả HLS + DASH (giảm storage)  | Legacy players chỉ HLS hoặc chỉ DASH             |
| MediaPackage vs S3 origin    | Live JIT, DRM, startover/catch-up                | VOD tĩnh sau MediaConvert, đơn giản              |
| MediaStore vs S3             | Buffer live fragment độ trễ cực thấp (legacy)    | VOD, archive, chi phí storage thấp, đa dụng      |
| CloudFront vs IVS CDN        | Tự kiến trúc origin (S3/MediaPackage)            | Managed — IVS lo CDN, bạn chỉ embed player       |
| SSAI vs Client-side ads      | Chống ad-blocker, đồng bộ broadcast              | MVP nhanh, quảng cáo overlay đơn giản              |

Chi tiết đáp án: [INTERVIEW_GUIDE.md](./INTERVIEW_GUIDE.md)

---

## 💡 Câu Hỏi Nên Hỏi Ngược Interviewer

1. **"Pipeline media hiện tại là VOD-heavy hay live-heavy?"** — Định hướng ôn tập
2. **"Team có dùng Elemental stack đầy đủ hay IVS-first?"** — Hiểu stack thực tế
3. **"SLA cho live production — RTO khi mất input là bao lâu?"** — Thể hiện ops mindset
4. **"Chiến lược DRM và multi-platform player như thế nào?"** — Security & compliance
5. **"FAST / ad-supported streaming có trong roadmap không?"** — MediaTailor, SCTE-35

---

## 🔗 Liên Kết Tài Liệu Ôn Tập

| Chủ đề phỏng vấn thường gặp | Tài liệu tham chiếu                                      |
| --------------------------- | -------------------------------------------------------- |
| Codec, HLS, ABR, DRM        | [01-fundamentals/](../01-fundamentals/README.md)         |
| VOD transcoding             | [02-mediaconvert/](../02-mediaconvert/README.md)         |
| Live encoding               | [03-medialive/](../03-medialive/README.md)               |
| Packaging & DRM             | [04-mediapackage/](../04-mediapackage/README.md)         |
| SSAI & ads                  | [06-mediatailor/](../06-mediatailor/README.md)           |
| Interactive live            | [07-ivs/](../07-ivs/README.md)                           |
| IoT video                   | [08-kinesis-video/](../08-kinesis-video/README.md)       |
| CDN delivery                | [09-cdn-delivery/](../09-cdn-delivery/README.md)           |
| Top 20 Q&A                  | [INTERVIEW_GUIDE.md](./INTERVIEW_GUIDE.md)                 |
| System design               | [architecture-scenarios.md](./architecture-scenarios.md) |
| STAR stories                | [star-stories.md](./star-stories.md)                       |

---

## 🎯 Theo Loại Role — Ưu Tiên Ôn Tập

### DevOps / Platform Engineer

```
Ưu tiên: MediaLive redundancy, CloudFront cache/OAC, monitoring, IAM roles
Files: INTERVIEW_GUIDE (câu 8, 12, 15, 18), architecture-scenarios (Scenario 2, 4)
```

### Backend / Video API Engineer

```
Ưu tiên: MediaConvert jobs, webhooks, manifest API, DRM license flow
Files: INTERVIEW_GUIDE (câu 3, 6, 11, 14), architecture-scenarios (Scenario 1, 3)
```

### Solutions Architect

```
Ưu tiên: End-to-end OTT, cost, scale, trade-offs toàn pipeline
Files: architecture-scenarios (tất cả), INTERVIEW_GUIDE (câu 17-20)
```

---

**Cập Nhật Lần Cuối:** 2026-06-04  
**Phiên Bản:** 1.0  
**Trạng Thái:** ✅ Hoàn thành
