# AWS Elemental MediaPackage — Đóng Gói & Bảo Vệ Nội Dung (Packaging & Origin)

> AWS Elemental MediaPackage là dịch vụ đóng gói và bảo vệ nội dung video được quản lý hoàn toàn bởi AWS. Dịch vụ nhận luồng video từ MediaLive (hoặc encoder khác), đóng gói thành các định dạng phát trực tuyến (HLS, DASH, CMAF) theo thời gian thực, tích hợp DRM — Digital Rights Management — Quản Lý Quyền Kỹ Thuật Số, và đóng vai trò là Origin Server — Máy Chủ Nguồn cho CloudFront CDN.

## 📚 Mục Lục (Table of Contents)

1. [MediaPackage Là Gì?](#1-mediapackage-là-gì)
2. [Kiến Trúc Tổng Quan](#2-kiến-trúc-tổng-quan)
3. [Các Khái Niệm Cốt Lõi](#3-các-khái-niệm-cốt-lõi)
4. [Luồng Xử Lý](#4-luồng-xử-lý)
5. [Nội Dung Chi Tiết](#5-nội-dung-chi-tiết)
6. [Câu Hỏi Phỏng Vấn Thường Gặp](#6-câu-hỏi-phỏng-vấn-thường-gặp)

---

## 1. MediaPackage Là Gì?

**AWS Elemental MediaPackage** là dịch vụ **just-in-time packaging** (đóng gói theo yêu cầu tức thì) và **origin server** (máy chủ nguồn) cho video streaming. Nằm giữa encoder (MediaLive) và CDN (CloudFront) trong pipeline live streaming.

### Vị Trí Trong Pipeline Live Streaming

```
                 ┌──────────────────── LIVE STREAMING PIPELINE ────────────────────────┐
                 │                                                                      │
[Encoder/OBS] → [MediaLive] → [MediaPackage] → [CloudFront] → [Viewer]
  RTMP/RTP        Encode          Đóng gói          CDN toàn cầu    HLS/DASH player
  nguồn live       multi-bitrate   JIT: HLS/DASH/    phân phối       trình duyệt
                   output          CMAF + DRM         adaptive        / mobile
                                   origin server      bitrate
```

### Chức Năng Chính Của MediaPackage

| Chức Năng | Mô Tả |
|-----------|-------|
| **JIT Packaging** (Đóng gói tức thì) | Chuyển đổi TS — Transport Stream sang HLS/DASH/CMAF theo yêu cầu viewer |
| **Multi-protocol delivery** (Phân phối đa giao thức) | Một channel, nhiều endpoint HLS + DASH + CMAF đồng thời |
| **DRM integration** (Tích hợp bảo vệ nội dung) | SPEKE — Secure Packager and Encoder Key Exchange, Widevine, FairPlay, PlayReady |
| **Time-shift viewing** (Xem dịch chuyển thời gian) | Startover — Xem lại từ đầu, Catch-up TV — Bắt kịp chương trình đã phát |
| **CDN Authorization** (Xác thực CDN) | Ký xác nhận yêu cầu từ CloudFront, ngăn truy cập trực tiếp |
| **Scalability** (Khả năng mở rộng) | Tự động scale, không giới hạn concurrent viewer |

### So Sánh MediaPackage vs Các Giải Pháp Khác

| Tiêu Chí | MediaPackage | Nginx + packager tự dựng | MediaStore |
|----------|-------------|------------------------|------------|
| **Quản lý hạ tầng** | Không cần | Phải tự quản lý | Không cần |
| **JIT packaging** | Có | Có (tự cấu hình) | Không (chỉ lưu trữ) |
| **DRM tích hợp** | SPEKE sẵn có | Phải tự tích hợp | Không |
| **Time-shift** | Tích hợp sẵn | Phải tự xây | Phải tự xây |
| **Scalability** | Tự động | Phải tự scale | Tự động |
| **CDN Authorization** | Có | Phải tự xây | Có |
| **Chi phí** | Theo GB đóng gói | Chi phí EC2 cố định | Theo GB lưu trữ |

> **Khi nào dùng MediaPackage thay vì MediaStore?**
> - MediaPackage: cần JIT packaging, DRM, time-shift, multi-format delivery từ một nguồn
> - MediaStore: chỉ cần lưu trữ tốc độ cao cho live fragments, không cần packaging

---

## 2. Kiến Trúc Tổng Quan

### Các Thành Phần Chính

```
┌────────────────────────────────────────────────────────────────────┐
│                    AWS ELEMENTAL MEDIAPACKAGE                      │
│                                                                    │
│  ┌──────────────┐    ┌─────────────────────────────────────────┐  │
│  │   Channel    │    │             Endpoints                   │  │
│  │              │    │                                         │  │
│  │  WebDAV/REST │    │  ┌────────────┐  ┌────────────────────┐ │  │
│  │  ingest      │───▶│  │ HLS        │  │ DASH               │ │  │
│  │  endpoint    │    │  │ Endpoint   │  │ Endpoint           │ │  │
│  │              │    │  └────────────┘  └────────────────────┘ │  │
│  │  (nhận TS    │    │  ┌────────────┐  ┌────────────────────┐ │  │
│  │   stream từ  │    │  │ CMAF       │  │ MS Smooth          │ │  │
│  │   MediaLive) │    │  │ Endpoint   │  │ Endpoint           │ │  │
│  └──────────────┘    │  └────────────┘  └────────────────────┘ │  │
│                      └─────────────────────────────────────────┘  │
│                                      │                             │
│              ┌───────────────────────┤                             │
│              ▼                       ▼                             │
│  ┌─────────────────┐    ┌─────────────────────────────────────┐   │
│  │  DRM / SPEKE    │    │        Time-shift Buffer             │   │
│  │  Key Provider   │    │   (Live → Startover / Catch-up)      │   │
│  └─────────────────┘    └─────────────────────────────────────┘   │
└────────────────────────────────────────────────────────────────────┘
                                │
                    ┌───────────┴────────────┐
                    ▼                        ▼
             CloudFront CDN           Direct viewer
             (khuyến nghị)            (dev/test only)
```

### Luồng Dữ Liệu (Data Flow)

```
1. MediaLive gửi TS stream vào MediaPackage Channel qua WebDAV
2. MediaPackage lưu trữ tạm TS segments trong rolling buffer
3. Viewer request manifest (HLS: .m3u8 / DASH: .mpd)
4. MediaPackage JIT-đóng gói TS → format yêu cầu (HLS/DASH/CMAF)
5. Nếu có DRM: lấy encryption key từ SPEKE Key Provider
6. Trả manifest + segments cho viewer (qua CloudFront)
7. Player ABR — Adaptive Bitrate tự chọn rendition phù hợp
```

---

## 3. Các Khái Niệm Cốt Lõi

### 3.1 Channel (Kênh Nhận)

**Channel** là điểm nhận luồng video từ MediaLive hoặc encoder khác. Mỗi Channel có hai **ingest endpoints** (điểm nhập) để hỗ trợ redundancy — dự phòng:

```
MediaPackage Channel
├── Ingest Endpoint 1 (Primary)    — MediaLive Pipeline A gửi vào đây
└── Ingest Endpoint 2 (Secondary)  — MediaLive Pipeline B gửi vào đây
```

**Giao thức nhận (Ingest protocols):**
- **WebDAV** — Web-based Distributed Authoring and Versioning: giao thức mặc định, dùng cho live stream
- **HLS ingest**: nhận HLS stream (chủ yếu cho MediaPackage v2)

**Lưu ý quan trọng:**
> Channel **không** thực hiện encoding hay transcoding. Channel chỉ nhận raw TS — Transport Stream và đưa vào rolling buffer. Việc đóng gói xảy ra tại Endpoint.

### 3.2 Endpoint (Điểm Phát)

**Endpoint** là URL phát nội dung đến viewer hoặc CDN. Mỗi Endpoint ánh xạ đến một định dạng output:

| Loại Endpoint | Định Dạng | Use Case |
|--------------|----------|---------|
| **HLS** — HTTP Live Streaming | `.m3u8` + `.ts` | iOS, Safari, Apple TV |
| **DASH** — Dynamic Adaptive Streaming over HTTP | `.mpd` + `.m4s` | Android, Web (Chrome/Firefox) |
| **CMAF** — Common Media Application Format | `.m3u8`/`.mpd` + `.m4s` | Cross-platform, low-latency |
| **MS Smooth** — Microsoft Smooth Streaming | `.ism` | Microsoft/Xbox devices |

**Cấu hình Endpoint điển hình:**
```
Endpoint: HLS Production
├── Manifest name: index.m3u8
├── Segment duration: 6 giây (hoặc 2s cho low-latency)
├── Playlist window length: 60 giây (10 segments)
├── Startover window: 7 ngày (time-shift buffer)
├── Ad markers: DATERANGE (từ SCTE-35)
├── DRM: SPEKE (Widevine + FairPlay)
└── CDN Authorization: enabled
```

### 3.3 JIT Packaging — Just-In-Time Packaging (Đóng Gói Theo Yêu Cầu Tức Thì)

**JIT packaging** nghĩa là MediaPackage **không** đóng gói trước — nó chỉ đóng gói khi có viewer request:

```
Không có JIT (pre-packaged):
  MediaLive ──▶ Tạo HLS + DASH + CMAF trước ──▶ Lưu S3 ──▶ CloudFront

Có JIT (MediaPackage):
  MediaLive ──▶ Lưu TS một lần ──▶ Viewer request HLS → tạo HLS ngay lúc đó
                                    Viewer request DASH → tạo DASH ngay lúc đó
```

**Lợi ích JIT:**
- Tiết kiệm storage: chỉ lưu TS một lần, không cần lưu nhiều format
- Linh hoạt: thêm/bớt DRM, thay đổi cấu hình endpoint mà không cần re-encode
- Hiệu quả: chỉ đóng gói những segment được request

### 3.4 Origin Server (Máy Chủ Nguồn)

MediaPackage đóng vai trò **Origin Server** cho CloudFront. CloudFront cache manifest và segments, khi cache miss mới forward request về MediaPackage.

```
Viewer ──▶ CloudFront (cache hit?) ──▶ HIT: trả từ edge cache
                                    ──▶ MISS: lấy từ MediaPackage Origin
                                            MediaPackage JIT-tạo segment
                                            Trả cho CloudFront + cache lại
```

**CDN Authorization — Xác Thực CDN:**
Để ngăn truy cập trực tiếp vào MediaPackage bỏ qua CloudFront, cấu hình CDN Authorization:
- CloudFront thêm header `X-MediaPackage-CDNIdentifier` vào mỗi request
- MediaPackage kiểm tra header, từ chối request không hợp lệ
- Secret lưu trong AWS Secrets Manager

---

## 4. Luồng Xử Lý

### Use Case 1: Live Streaming Cơ Bản

```
OBS Studio (RTMP)
        │
        ▼
MediaLive Channel (Standard — 2 pipelines)
  Pipeline A ──▶ MediaPackage Ingest Endpoint 1
  Pipeline B ──▶ MediaPackage Ingest Endpoint 2
        │
        ▼
MediaPackage Channel
  Rolling buffer: 7 ngày (startover window)
        │
        ├──▶ HLS Endpoint   (iOS + Safari viewers)
        ├──▶ DASH Endpoint  (Android + Chrome viewers)
        └──▶ CMAF Endpoint  (cross-platform, low-latency)
                │
                ▼
        CloudFront CDN
                │
                ▼
        Viewer (ABR player tự chọn rendition)
```

### Use Case 2: Live Streaming Có DRM

```
MediaLive ──▶ MediaPackage Channel
                      │
                      ▼
              HLS Endpoint (DRM enabled)
                  │
                  ├── Request key từ SPEKE Key Provider
                  │   (AWS Elemental MediaPackage ↔ DRM vendor)
                  │
                  ├── Widevine  → Android, Chrome
                  ├── FairPlay  → iOS, macOS, tvOS
                  └── PlayReady → Windows, Xbox
                      │
                      ▼
              CloudFront + Signed URL/Cookies
                      │
                      ▼
              Viewer (player xin license trước khi decrypt)
```

### Use Case 3: Live + Time-shift (Catch-up TV)

```
MediaLive ──▶ MediaPackage Channel
  Rolling buffer: 7 ngày

Viewer 1 (đang xem live):
  Request: /hls/index.m3u8 → manifest với segments mới nhất

Viewer 2 (bỏ lỡ, muốn xem lại từ 30 phút trước):
  Request: /hls/index.m3u8?startTime=2026-06-04T12:30:00Z
         → MediaPackage tạo manifest từ buffer tại thời điểm đó
         → Viewer xem lại từ 12:30 (catch-up)

Viewer 3 (muốn xem từ đầu chương trình):
  Request: /hls/index.m3u8?startTime=2026-06-04T12:00:00Z
         → Startover: xem từ 12:00 dù stream vẫn đang live
```

---

## 5. Nội Dung Chi Tiết

```
04-mediapackage/
├── README.md                    ← [BẠN ĐANG Ở ĐÂY] Tổng quan
├── 1-channels-endpoints.md      Channel ingest, Endpoint config, CDN Authorization
├── 2-just-in-time-packaging.md  JIT packaging chi tiết, bitrate filtering, time-delay
├── 3-drm-speke.md               SPEKE protocol, multi-DRM setup, key rotation
├── 4-time-shift-viewing.md      Startover, catch-up TV, windowed manifest
└── 5-mediapackage-v2.md         V2: harvest jobs, LL-CMAF, scalability improvements
```

### Bản Đồ Học Tập Đề Xuất

```
Bắt đầu tại đây (README)
         │
         ▼
1-channels-endpoints.md      ← Hiểu Channel và Endpoint trước
         │
         ▼
2-just-in-time-packaging.md  ← Cơ chế JIT là trọng tâm của MediaPackage
         │
         ▼
3-drm-speke.md               ← Bảo vệ nội dung với DRM
         │
         ▼
4-time-shift-viewing.md      ← Startover và catch-up TV
         │
         ▼
5-mediapackage-v2.md         ← Tính năng mới nhất của V2
```

---

## 6. Câu Hỏi Phỏng Vấn Thường Gặp

### Câu hỏi cơ bản

**Q: MediaPackage khác gì MediaStore?**

> **MediaPackage** là dịch vụ **packaging & origin**: nhận TS stream, JIT-đóng gói thành HLS/DASH/CMAF, tích hợp DRM, hỗ trợ time-shift. Phù hợp cho pipeline live streaming đầy đủ.
>
> **MediaStore** là dịch vụ **lưu trữ tốc độ cao** cho media fragments, độ trễ thấp hơn S3 đáng kể. Không có JIT packaging hay DRM. Phù hợp khi cần lưu live fragments để encoder/packager khác đọc.

**Q: Tại sao MediaPackage cần 2 Ingest Endpoint?**

> Vì **MediaLive Standard Channel** chạy 2 pipelines (A và B) song song để dự phòng. Cả hai pipeline đều gửi luồng giống nhau vào MediaPackage, nhưng qua 2 endpoint riêng biệt. MediaPackage tự động chọn endpoint nào ổn định hơn. Nếu một pipeline MediaLive lỗi, MediaPackage tự chuyển sang pipeline kia mà không gián đoạn stream đến viewer.

**Q: JIT Packaging là gì và tại sao quan trọng?**

> JIT — Just-In-Time Packaging là cơ chế MediaPackage chỉ đóng gói content **khi có viewer yêu cầu**, thay vì đóng gói trước và lưu hết. Điều này:
> - **Tiết kiệm chi phí storage**: chỉ lưu TS một lần
> - **Linh hoạt**: thêm DRM hay đổi định dạng không cần re-process
> - **Hiệu quả**: không lãng phí tài nguyên cho các format không ai xem

### Câu hỏi nâng cao

**Q: Cách thiết lập Multi-DRM cho cross-platform?**

> Dùng **SPEKE** — Secure Packager and Encoder Key Exchange với nhiều DRM system IDs:
> 1. Cấu hình SPEKE Key Provider (AWS hoặc DRM vendor như Irdeto, EZDRM)
> 2. Trên HLS Endpoint: thêm **FairPlay** (System ID: `94ce86fb-07bb-4b43-adf0-...`) cho iOS/tvOS
> 3. Trên DASH Endpoint: thêm **Widevine** (System ID: `edef8ba9-79d6-4ace-a3c8-...`) cho Android/Chrome
> 4. Optionally **PlayReady** (System ID: `9a04f079-9840-4286-ab92-...`) cho Windows
> 5. Player xin license từ DRM License Server trước khi decrypt content

**Q: Time-shift viewing hoạt động như thế nào?**

> MediaPackage duy trì một **rolling buffer** — bộ đệm cuộn lưu TS segments trong khoảng thời gian cấu hình (tối đa 336 giờ = 14 ngày). Khi viewer request manifest với `startTime` parameter:
> - MediaPackage tra cứu buffer tại timestamp đó
> - Tạo manifest chỉ có segments từ thời điểm đó trở đi
> - Viewer xem lại nội dung đã phát mà không cần lưu vào S3 riêng
>
> **Startover**: xem từ đầu một sự kiện đang live
> **Catch-up TV**: bắt kịp nội dung đã bỏ lỡ trong khoảng window

**Q: MediaPackage v1 vs v2 khác nhau điểm gì?**

> **V2** cải tiến chính:
> - **LL-HLS / LL-DASH** — Low-Latency: hỗ trợ CMAF Chunked Transfer, giảm latency xuống 3–5 giây
> - **Harvest Jobs**: xuất một đoạn VOD từ live stream trực tiếp ra S3 (không cần lưu toàn bộ)
> - **Packaging Groups**: quản lý nhiều channels + endpoints theo nhóm
> - **Improved scalability**: kiến trúc mới scale tốt hơn ở workload lớn
> - **HTTP PUT ingest**: thay thế WebDAV, đơn giản và hiệu quả hơn

---

## 🔗 Tài Liệu Tham Khảo

- [MediaPackage User Guide](https://docs.aws.amazon.com/mediapackage/latest/ug/)
- [MediaPackage v2 User Guide](https://docs.aws.amazon.com/mediapackage/latest/userguide/)
- [MediaPackage Pricing](https://aws.amazon.com/mediapackage/pricing/)
- [SPEKE Specification](https://docs.aws.amazon.com/speke/latest/documentation/)
- [Low-Latency HLS with MediaPackage](https://aws.amazon.com/blogs/media/low-latency-hls-with-aws-elemental-mediapackage/)

---

**Cập Nhật Lần Cuối:** 2026-06-04
**Phần Trước:** [03-medialive/](../03-medialive/README.md) — Live Encoding
**Phần Tiếp Theo:** [05-mediastore/](../05-mediastore/README.md) — Media Storage
