# 📋 Top 20 Câu Hỏi Phỏng Vấn AWS Media Services — Hướng Dẫn Đầy Đủ

> Bộ câu hỏi + đáp án mẫu cho phỏng vấn DevOps, Backend, Platform và Solutions Architect liên quan video streaming trên AWS. Mỗi câu trả lời nhấn trade-offs và liên kết tới module chi tiết trong knowledge base.

---

## 🏷️ Phân Loại Câu Hỏi

| Danh Mục                         | Câu Số | Mức Độ   |
| -------------------------------- | ------ | -------- |
| Nền tảng (Codec, Protocol)       | 1-4    | Junior+  |
| VOD & MediaConvert               | 5-7    | Mid+     |
| Live & MediaLive / IVS           | 8-11   | Mid+     |
| Packaging, DRM & Ads             | 12-15  | Mid+     |
| CDN, Scale & Cost                | 16-18  | Senior+  |
| System Design & Tổng Hợp         | 19-20  | Senior+  |

---

## 🔵 Nền Tảng Media

---

### Câu 1: Codec khác Container như thế nào? Giải thích H.264 trong MP4 và HLS.

**Câu trả lời mẫu:**

**Codec** — Coder-Decoder (Bộ Mã Hoá/Giải Mã) — là thuật toán **nén** video/audio (H.264, AAC). **Container** — Định Dạng Đóng Gói — là "hộp" chứa các track đã mã hoá (MP4, MKV, TS).

```
H.264 (codec) + AAC (codec) → đóng trong MP4 (container) → file .mp4 đơn (progressive download)

H.264 + AAC → chia thành segment .ts hoặc .m4s → playlist .m3u8 (HLS — HTTP Live Streaming)
              → container ở cấp segment, không phải một file MP4 duy nhất
```

**Điểm phỏng vấn cần nói:**
- Cùng codec H.264 có thể xuất MP4 (VOD) hoặc HLS (streaming)
- Player HLS tải manifest rồi tải từng segment — phù hợp ABR
- Nhầm "HLS là codec" là lỗi phổ biến — HLS là **protocol/delivery format**

**Follow-up:** "H.265 so với H.264?" → HEVC — High Efficiency Video Coding — tiết kiệm ~50% bitrate cùng chất lượng; trade-off: encode chậm hơn, một số thiết bị cũ không decode.

📖 Chi tiết: [01-fundamentals/1-codec-container.md](../01-fundamentals/1-codec-container.md)

---

### Câu 2: HLS và DASH khác nhau thế nào? Khi nào dùng CMAF?

**Câu trả lời mẫu:**

| Tiêu Chí        | HLS — HTTP Live Streaming              | DASH — Dynamic Adaptive Streaming over HTTP |
| --------------- | -------------------------------------- | --------------------------------------------- |
| Manifest        | `.m3u8` (text)                         | `.mpd` (XML)                                  |
| Segment         | `.ts` (MPEG-TS) hoặc `.m4s` (fMP4)     | thường `.m4s` (fMP4)                          |
| Hệ sinh thái    | Apple (Safari, iOS native)             | Android, Smart TV, tiêu chuẩn mở              |
| DRM             | FairPlay (HLS) phổ biến trên iOS       | Widevine, PlayReady trên DASH                  |

**CMAF** — Common Media Application Format — Định Dạng Ứng Dụng Media Chung — dùng **một bộ segment fMP4** cho cả HLS và DASH → giảm storage và transcoding cost khi cần đa nền tảng.

**Khi nào chọn:**
- Chỉ web + iOS → HLS đủ (đơn giản hơn)
- Android TV + web + mobile đa nền → DASH hoặc **dual manifest** từ CMAF
- MediaConvert output group **CMAF** khi target OTT đa platform

📖 Chi tiết: [01-fundamentals/2-streaming-protocols.md](../01-fundamentals/2-streaming-protocols.md)

---

### Câu 3: ABR — Adaptive Bitrate Streaming hoạt động như thế nào?

**Câu trả lời mẫu:**

ABR cho phép player **tự chọn** rendition (phiên bản chất lượng) dựa trên bandwidth và buffer.

```
Master playlist (.m3u8)
├── 1080p @ 5 Mbps
├── 720p  @ 2.5 Mbps
├── 480p  @ 1 Mbps
└── 360p  @ 500 Kbps

Player logic (ví dụ HLS.js):
  buffer đầy + bandwidth cao → switch lên rendition cao
  buffer cạn / packet loss   → switch xuống rendition thấp
```

**Bitrate ladder** (thang tốc độ bit) thường thiết kế với bước ~1.5–2x giữa các bậc. MediaConvert/MediaLive tạo nhiều output trong một job/channel.

**Điểm cộng phỏng vấn:** Nói thêm **QoE** — Quality of Experience — metrics: rebuffer ratio, startup time, bitrate switches.

📖 Chi tiết: [01-fundamentals/3-adaptive-bitrate.md](../01-fundamentals/3-adaptive-bitrate.md)

---

### Câu 4: DRM là gì? Giải thích luồng Speke với MediaPackage.

**Câu trả lời mẫu:**

**DRM** — Digital Rights Management — Quản Lý Quyền Kỹ Thuật Số — mã hoá nội dung để chỉ player có license mới giải mã được.

| DRM System  | Nền Tảng Chính        |
| ----------- | --------------------- |
| Widevine    | Android, Chrome       |
| FairPlay    | iOS, Safari           |
| PlayReady   | Windows, Xbox, một số TV |

**Luồng điển hình trên AWS:**

```
1. MediaPackage endpoint bật DRM + SPEKE
2. SPEKE — Secure Packager and Encoder Key Exchange — gọi API Gateway → Lambda → key server (Axinom, EZDRM...)
3. Packager nhận content key → mã hoá segment
4. Player: manifest → license server → content key → decrypt & play
```

**Speke** là **chuẩn API** AWS dùng để trao đổi key — không phải DRM engine; bạn mang license provider riêng.

📖 Chi tiết: [01-fundamentals/4-drm-fundamentals.md](../01-fundamentals/4-drm-fundamentals.md), [04-mediapackage/3-drm-speke.md](../04-mediapackage/3-drm-speke.md)

---

## 🟢 VOD & MediaConvert

---

### Câu 5: Mô tả cấu trúc MediaConvert job và vai trò Output Group.

**Câu trả lời mẫu:**

```
Job
├── Role (IAM — Identity and Access Management)
├── Queue (On-demand / Reserved / Spot)
├── Input (S3, HTTP) — file nguồn
├── Settings
│   ├── Video / Audio / Captions selectors
│   └── Output Groups  ← quan trọng
│       ├── HLS Group      → .m3u8 + segments
│       ├── DASH ISO Group → .mpd + segments
│       ├── CMAF Group     → dual HLS+DASH segments
│       ├── File Group     → MP4 đơn
│       └── MS Smooth      → legacy Microsoft
└── Outputs (renditions trong mỗi group)
```

**Output Group** quyết định **định dạng phân phối**, không phải codec — codec nằm trong từng Output (H.264 1080p, 720p...).

**Trigger phổ biến:** S3 event → Lambda → `create-job` → SNS khi complete → invalidate CloudFront.

📖 Chi tiết: [02-mediaconvert/1-jobs-queues.md](../02-mediaconvert/1-jobs-queues.md), [02-mediaconvert/2-output-groups.md](../02-mediaconvert/2-output-groups.md)

---

### Câu 6: Làm sao tối ưu chi phí MediaConvert?

**Câu trả lời mẫu:**

| Chiến Lược | Mô Tả |
| ---------- | ----- |
| **Reserved Queue** | Cam kết throughput/phút — giảm giá cho workload ổn định 24/7 |
| **Spot Queue** | Giá thấp hơn — chấp nhận job có thể bị gián đoạn (batch VOD không gấp) |
| **Right-size outputs** | Không tạo UHD nếu nguồn chỉ HD; bỏ rendition ít người xem |
| **H.265 vs H.264** | H.265 giảm storage/CDN egress — encode cost cao hơn một chút |
| **CMAF dual output** | Một lần encode phục vụ HLS+DASH thay vì hai job riêng |
| **Acceleration** | Pro tier nhanh hơn — dùng khi time-to-market quan trọng hơn tiền |

**Công thức phỏng vấn:** Total cost = processing minutes × rate per resolution tier + S3 storage + CloudFront egress. Egress CDN thường **lớn hơn** transcoding ở scale lớn.

📖 Chi tiết: [02-mediaconvert/5-cost-optimization.md](../02-mediaconvert/5-cost-optimization.md)

---

### Câu 7: Pipeline VOD end-to-end trên AWS là gì?

**Câu trả lời mẫu:**

```
Upload (Console/S3 Transfer) 
    → S3 source bucket
    → EventBridge / Lambda trigger
    → MediaConvert Job (HLS/CMAF outputs)
    → S3 destination bucket
    → CloudFront Distribution (OAC — Origin Access Control)
    → Player (HLS.js / Video.js / native)

Tùy chọn:
    → MediaPackage VOD (nếu cần DRM JIT trên catalog động)
    → Step Functions orchestration nhiều profile (4K + HD + SD)
    → MediaConvert queue priority theo tier subscriber
```

**Bảo mật:** OAC chặn truy cập S3 trực tiếp; **Signed URL** cho nội dung premium.

📖 Chi tiết: [02-mediaconvert/README.md](../02-mediaconvert/README.md), [09-cdn-delivery/](../09-cdn-delivery/README.md)

---

## 🟡 Live Streaming

---

### Câu 8: Khi nào dùng MediaLive thay vì IVS?

**Câu trả lời mẫu:**

| Tiêu Chí | MediaLive + MediaPackage | Amazon IVS |
| -------- | ------------------------ | ---------- |
| Latency | 15–30s (standard HLS) | 5s (low) / <1s (WebRTC real-time) |
| Use case | Broadcast OTT, thể thao, TV | Interactive livestream, chat, bán hàng |
| SCTE-35 | Đầy đủ, schedule actions | Hạn chế hơn |
| DRM / packaging | MediaPackage chuyên sâu | Đơn giản hơn, managed |
| Input | RTMP, RTP, MediaConnect, HLS pull | RTMPS chủ yếu |
| Vận hành | Bạn quản lý nhiều thành phần | Managed end-to-end |

**Rule of thumb:**
- Cần **kênh truyền hình**, ad markers SCTE-35, failover 2 pipeline → **MediaLive**
- Cần **tương tác realtime**, SDK player nhanh, team nhỏ → **IVS**

📖 Chi tiết: [03-medialive/README.md](../03-medialive/README.md), [07-ivs/README.md](../07-ivs/README.md)

---

### Câu 9: Standard channel vs Single-pipeline trong MediaLive?

**Câu trả lời mẫu:**

**Standard channel** — 2 pipelines **A/B** song song:
- Cùng input được xử lý hai lần độc lập
- Output có thể failover tự động khi một pipeline lỗi
- Chi phí ~gấp đôi Single-pipeline
- Bắt buộc cho SLA broadcast production

**Single-pipeline:**
- Một pipeline duy nhất — rẻ hơn
- Phù hợp dev/test hoặc event không critical

**Input redundancy** (tách khỏi channel class):
- **Automatic Input Failover** — hai input (primary/backup RTMP)
- Kết hợp Standard channel → HA — High Availability — cao nhất

📖 Chi tiết: [03-medialive/3-redundancy-failover.md](../03-medialive/3-redundancy-failover.md)

---

### Câu 10: SCTE-35 là gì và vai trò trong live ads?

**Câu trả lời mẫu:**

**SCTE-35** — Society of Cable Telecommunications Engineers 35 — là chuẩn **signaling** (tín hiệu điều khiển) nhúng trong luồng MPEG-TS để đánh dấu:
- Ad break bắt đầu / kết thúc
- Program splice, return to network

```
MediaLive (insert SCTE-35 cue)
    → MediaPackage (passthrough markers trong manifest)
    → MediaTailor (đọc cue → gọi ADS — Ad Decision Server)
    → Manifest có segment quảng cáo cá nhân hoá (SSAI)
```

Không có SCTE-35 → MediaTailor khó đồng bộ ad break với nội dung broadcast.

📖 Chi tiết: [03-medialive/5-schedule-scte35.md](../03-medialive/5-schedule-scte35.md), [06-mediatailor/1-ssai-basics.md](../06-mediatailor/1-ssai-basics.md)

---

### Câu 11: Xử lý input loss trên MediaLive như thế nào?

**Câu trả lời mẫu:**

**Phòng ngừa:**
- Automatic Input Failover (2 URL RTMP/RTP)
- MediaConnect cho contribution ổn định (thay vì public internet RTMP)
- Encoder local buffer trước khi push

**Khi đã mất input:**
- Channel state → `INPUT_FAILURE` hoặc slate (hình placeholder) nếu cấu hình
- CloudWatch alarms: `NetworkIn`, `ActiveAlerts`, `InputLoss`
- Runbook: chuyển Schedule Action sang backup input; thông báo đội production

**Sau sự cố:** Kiểm tra CloudWatch Logs + MediaLive alerts history; xác nhận viewer impact qua CloudFront real-time logs (403/5xx spike).

📖 Chi tiết: [03-medialive/1-channel-input.md](../03-medialive/1-channel-input.md)

---

## 🟠 Packaging, DRM & Ads

---

### Câu 12: MediaPackage JIT packaging là gì?

**Câu trả lời mẫu:**

**JIT** — Just-In-Time — Đóng Gói Theo Thời Gian Thực — MediaPackage nhận luồng đã mã hoá từ MediaLive, lưu buffer fragment, **tạo manifest HLS/DASH khi viewer request** thay vì lưu sẵn toàn bộ playlist tĩnh.

**Lợi ích:**
- Một luồng ingest → nhiều endpoint (HLS, DASH, CMAF) với DRM khác nhau
- **Time-shift**: startover, catch-up TV qua window trên origin
- Bitrate filtering theo endpoint

**So với VOD tĩnh trên S3:** JIT cho **live**; VOD sau MediaConvert thường origin S3 + CloudFront đơn giản hơn.

📖 Chi tiết: [04-mediapackage/2-just-in-time-packaging.md](../04-mediapackage/2-just-in-time-packaging.md)

---

### Câu 13: SSAI — Server-Side Ad Insertion hoạt động như thế nào?

**Câu trả lời mẫu:**

```
Viewer request manifest
    → MediaTailor
        → Lấy content manifest từ MediaPackage
        → Gọi ADS (VAST/VMAP) lấy quảng cáo phù hợp user
        → Stitch ad segments vào timeline
    → Trả manifest thống nhất (content + ads)
Viewer tải segment — không phân biệt content/ad ở client
```

**Ưu điểm:** Khó bị ad-blocker; trải nghiệm giống TV.
**Nhược điểm:** Phức tạp ops; manifest động → cache CDN cẩn thận (TTL ngắn cho master playlist).

📖 Chi tiết: [06-mediatailor/1-ssai-basics.md](../06-mediatailor/1-ssai-basics.md)

---

### Câu 14: Multi-DRM cho iOS + Android + Web?

**Câu trả lời mẫu:**

1. MediaPackage tạo **nhiều endpoint** hoặc một endpoint hỗ trợ multi-DRM
2. **Speke** gọi key server trả key theo DRM system identifier
3. Player chọn DRM theo platform:
   - iOS → FairPlay (HLS)
   - Android → Widevine (DASH)
   - Web → Widevine hoặc PlayReady

**Lưu ý phỏng vấn:** Certificate FairPlay (Apple), license URL riêng, **key rotation** cho long-form content.

📖 Chi tiết: [04-mediapackage/3-drm-speke.md](../04-mediapackage/3-drm-speke.md)

---

### Câu 15: Time-shift viewing (startover / catch-up) hoạt động ra sao?

**Câu trả lời mẫu:**

MediaPackage giữ **buffer** fragment trong khoảng thời gian cấu hình (ví dụ 2 giờ live + 7 ngày catch-up).

- **Startover:** Xem lại từ đầu chương trình đang phát
- **Catch-up:** Xem chương trình đã phát trong window

Manifest được tạo với **timeline offset** — player seek trong window đó.

**CDN:** Catch-up traffic có thể cache tốt hơn live edge (nội dung "cũ" hơn).

📖 Chi tiết: [04-mediapackage/4-time-shift-viewing.md](../04-mediapackage/4-time-shift-viewing.md)

---

## 🔴 CDN, Scale & Cost

---

### Câu 16: Chiến lược cache CloudFront cho video HLS?

**Câu trả lời mẫu:**

| Loại File | Cache TTL Gợi Ý | Lý Do |
| --------- | ----------------- | ----- |
| Master `.m3u8` | Ngắn (1–10s live) | Chứa URL rendition đổi theo live |
| Media `.m3u8` | Ngắn–trung bình | Playlist segment list cập nhật |
| Segment `.ts`/`.m4s` | Dài (hours–days) | Immutable sau khi publish — cache hit cao |

**Cache key:** Không include query string không cần thiết (trừ signed URL policy).

**Origin shield:** Giảm load MediaPackage khi nhiều edge cùng miss.

**Range GET:** Hỗ trợ partial content cho một số player/progressive MP4.

📖 Chi tiết: [09-cdn-delivery/1-cloudfront-for-media.md](../09-cdn-delivery/1-cloudfront-for-media.md)

---

### Câu 17: Thiết kế hệ thống cho 1 triệu concurrent viewers — điểm cần nói?

**Câu trả lời mẫu:**

**Không phải 1 triệu kết nối vào một server** — scale nhờ CDN:

```
1 triệu viewers
    → CloudFront (450+ PoP — Point of Presence)
    → Cache hit 95%+ trên segment
    → Origin (MediaPackage) chỉ phục vụ cache miss + live edge
```

**Checklist phỏng vấn:**
- Đủ **origin capacity** / MediaPackage limits — liên hệ AWS cho soft limit
- **Multi-region** origin hoặc Route 53 latency routing nếu global
- **Manifest TTL** tối ưu — tránh thundering herd origin
- **Monitoring:** CloudFront real-time logs, MediaPackage egress, MediaLive health
- **Cost:** Egress là dòng chi phí lớn — negotiated CloudFront pricing ở enterprise

📖 Chi tiết: [architecture-scenarios.md](./architecture-scenarios.md) Scenario 2

---

### Câu 18: MediaStore vs S3 — khi nào dùng gì?

**Câu trả lời mẫu:**

| Tiêu Chí | MediaStore | S3 |
| -------- | ---------- | -- |
| Độ trễ | Rất thấp cho read fragment | Thấp, đủ cho hầu hết VOD |
| Use case | Live buffer legacy | VOD, archive, IVS recording |
| API | Chuyên media container | Universal |
| Xu hướng | Ít dự án mới | Mặc định cho greenfield |

**Phỏng vấn:** Nhiều kiến trúc mới **bỏ qua MediaStore**, dùng MediaPackage origin + S3 cho VOD.

📖 Chi tiết: [05-mediastore/3-mediastore-vs-s3.md](../05-mediastore/3-mediastore-vs-s3.md)

---

## 🟣 System Design & Tổng Hợp

---

### Câu 19: Kinesis Video Streams (KVS) khác pipeline OTT thế nào?

**Câu trả lời mẫu:**

**KVS** — Kinesis Video Streams — hướng tới **ingest từ thiết bị** (camera IoT), lưu timeline, consumer đọc frame cho ML — **không** thay thế CDN OTT.

```
Camera → Producer SDK → KVS → Lambda / Rekognition / Kinesis Data Streams
                         → (tùy chọn) HLS playback cho xem lại ngắn
```

**OTT:** MediaConvert/MediaLive → CDN → mass viewers.

**Chọn KVS khi:** An ninh, nhà máy, telemedicine — analytics realtime quan trọng hơn broadcast quality.

📖 Chi tiết: [08-kinesis-video/README.md](../08-kinesis-video/README.md)

---

### Câu 20: Liệt kê trade-offs khi chọn stack cho startup OTT nhỏ?

**Câu trả lời mẫu:**

**MVP nhanh (ít ops):**
```
Upload VOD → MediaConvert → S3 → CloudFront
Live tương tác → IVS (không tự ghép MediaLive/MediaPackage)
```

**Broadcast-grade (chi phí + phức tạp cao hơn):**
```
MediaLive → MediaPackage → MediaTailor (nếu ads) → CloudFront
```

| Yếu Tố | Startup MVP | Scale Broadcast |
| ------ | ------------- | ----------------- |
| Time to market | IVS + MediaConvert | +2–3 tháng tích hợp |
| Ops burden | Thấp | MediaLive 24/7 on-call |
| Ads | Client-side hoặc sau | SSAI + SCTE-35 |
| DRM | Có thể trì hoãn | Speke + multi-DRM |

**Câu kết phỏng vấn:** "Em sẽ bắt đầu IVS + VOD S3/CloudFront, thiết kế API và player trước; khi có yêu cầu FAST/DRM broadcast, migrate live sang MediaLive + MediaPackage."

---

## 📌 Bảng Tra Nhanh — Dịch Vụ Theo Use Case

| Use Case | Dịch Vụ AWS Chính |
| -------- | ------------------ |
| Transcode file VOD | MediaConvert |
| Live TV / sports OTT | MediaLive + MediaPackage + CloudFront |
| Interactive live + chat | IVS + IVS Chat |
| DRM + time-shift | MediaPackage + Speke |
| Personalized ads | MediaTailor + SCTE-35 |
| IoT camera analytics | Kinesis Video Streams + Rekognition |
| Bảo vệ S3 VOD | CloudFront OAC + Signed URL |
| Linear channel từ VOD clips | MediaTailor Channel Assembly |

---

## 🎤 Mẹo Trình Bày Trong Phỏng Vấn

1. **Vẽ pipeline trước**, rồi mới đi sâu từng box
2. Nói **latency và cost** chủ động — không đợi interviewer hỏi
3. Dùng số liệu ước lượng (cache hit 90%, egress $X/GB) khi có
4. Thừa nhận điểm chưa biết + cách bạn sẽ verify (AWS docs, PoC)
5. Liên kết kinh nghiệm thật — tránh học thuộc lòng không hiểu

---

**Cập Nhật Lần Cuối:** 2026-06-04  
**Tham Chiếu:** [README.md](./README.md) | [architecture-scenarios.md](./architecture-scenarios.md) | [star-stories.md](./star-stories.md)
