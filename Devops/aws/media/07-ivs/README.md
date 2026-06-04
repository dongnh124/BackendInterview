# Amazon IVS — Interactive Video Service (Dịch Vụ Video Tương Tác)

> **Amazon IVS** — Interactive Video Service — Dịch Vụ Video Tương Tác là dịch vụ live streaming được quản lý hoàn toàn, tối ưu cho **phát trực tiếp tương tác độ trễ thấp** (low-latency interactive live): livestream bán hàng, giáo dục, game show, sự kiện có chat và UI đồng bộ với video. IVS xử lý ingest, transcoding, phân phối và cung cấp **Playback SDK** (bộ SDK phát) cho web/mobile.

## 📚 Mục Lục (Table of Contents)

1. [IVS Là Gì?](#1-ivs-là-gì)
2. [Kiến Trúc Tổng Quan](#2-kiến-trúc-tổng-quan)
3. [Các Khái Niệm Cốt Lõi](#3-các-khái-niệm-cốt-lõi)
4. [Pipeline Interactive Live](#4-pipeline-interactive-live)
5. [Nội Dung Chi Tiết](#5-nội-dung-chi-tiết)
6. [Câu Hỏi Phỏng Vấn Thường Gặp](#6-câu-hỏi-phỏng-vấn-thường-gặp)

---

## 1. IVS Là Gì?

**Amazon IVS** là dịch vụ **managed live streaming** (phát trực tiếp được quản lý) của AWS, xuất phát từ công nghệ **Twitch** (low-latency HLS và hạ tầng phân phối). Khác với **MediaLive + MediaPackage** (hướng broadcast OTT), IVS tập trung vào **tương tác viewer–streamer** với độ trễ thấp, **Timed Metadata** (metadata theo thời gian), và **IVS Chat** tích hợp.

### Vị Trí Trong Pipeline Interactive Live

```
                 ┌──────────── INTERACTIVE LIVE PIPELINE ─────────────────────┐
                 │                                                           │
[OBS / Encoder] → [IVS Channel] → [IVS CDN] → [IVS Player SDK] → [Viewer App]
  RTMPS ingest     Transcode +       Global         HLS / WebRTC      Video +
  Stream Key       ABR ladder        delivery       + Timed Meta      Chat UI
```

### So Sánh IVS vs MediaLive + MediaPackage

| Tiêu Chí | Amazon IVS | MediaLive + MediaPackage |
|----------|------------|--------------------------|
| **Độ trễ** | ~5s (Low) / <1s (Real-time WebRTC) | ~15–30s (HLS standard) |
| **Thiết lập** | Channel + Stream Key trong vài phút | Channel, Input, Output, Endpoint phức tạp |
| **SCTE-35 / broadcast** | Hạn chế | Đầy đủ (ad markers, schedule) |
| **DRM** | Không (use case công khai/interactive) | Widevine, FairPlay qua Speke |
| **Tương tác** | Timed metadata, IVS Chat native | Phải tự xây (WebSocket, API) |
| **Quy mô viewer** | Hàng triệu (IVS CDN) | Hàng triệu (CloudFront) |
| **Use case** | Livestream bán hàng, webinar, game | OTT TV, thể thao, tin tức có quảng cáo |
| **Chi phí** | Input duration + output (data) | Channel hours + MediaPackage + CDN |

> **Khi nào dùng IVS?**
> - Cần độ trễ thấp và chat/metadata đồng bộ với video
> - Không cần DRM phức tạp hay SCTE-35 ad insertion
> - Team nhỏ muốn go-live nhanh với OBS + web player
> - Real-time stage (WebRTC) cho panel, auction, Q&A

> **Khi nào dùng MediaLive?**
> - Kênh OTT có quảng cáo (SCTE-35 → MediaTailor)
> - Multi-DRM, time-shift, catch-up TV
> - Input satellite/RTP, MediaConnect, redundancy A/B chuyên nghiệp

---

## 2. Kiến Trúc Tổng Quan

### Các Thành Phần Chính

```
┌─────────────────────────────────────────────────────────────────────────┐
│                         AMAZON IVS                                      │
│                                                                         │
│  ┌──────────────────┐    ┌──────────────────────────────────────────┐  │
│  │  Channel         │    │  IVS Low-Latency / Real-Time             │  │
│  │                  │    │                                          │  │
│  │  • Type (Basic/  │───▶│  Ingest → Transcode → ABR HLS / WebRTC  │  │
│  │    Standard/     │    │  Playback URL + Stream Key               │  │
│  │    Advanced)     │    └──────────────────────────────────────────┘  │
│  │  • Latency mode  │                    │                              │
│  │  • Recording     │                    ▼                              │
│  └────────┬─────────┘         ┌─────────────────────┐                  │
│           │                   │  IVS Player SDK     │                  │
│           │                   │  JS / iOS / Android │                  │
│           ▼                   └─────────────────────┘                  │
│  ┌──────────────────┐                                                  │
│  │  IVS Chat        │  Room + Messaging API (độc lập region)           │
│  │  (tùy chọn)      │  Token-based auth, moderation hooks               │
│  └──────────────────┘                                                  │
│           │                                                             │
│           ▼                                                             │
│  ┌──────────────────┐    S3 (HLS segments) nếu bật Recording          │
│  │  Auto Recording  │                                                  │
│  └──────────────────┘                                                  │
└─────────────────────────────────────────────────────────────────────────┘
```

### Luồng Dữ Liệu (Data Flow)

```
1. Streamer cấu hình OBS với RTMPS URL + Stream Key từ IVS Console
2. OBS push luồng tới IVS Ingest endpoint
3. IVS transcode thành ABR ladder (tuỳ channel type)
4. Viewer app load Playback URL qua IVS Player SDK
5. (Tuỳ chọn) Backend gửi Timed Metadata → đồng bộ UI (poll, product card)
6. (Tuỳ chọn) IVS Chat: client lấy token → join room → realtime messages
7. (Tuỳ chọn) Recording ghi HLS segments vào S3 sau khi stream kết thúc
```

---

## 3. Các Khái Niệm Cốt Lõi

### 3.1 Channel (Kênh Phát)

**Channel** là tài nguyên trung tâm: mỗi channel có **Ingest endpoint** (điểm nhận luồng), **Playback URL** (URL phát cho viewer), và **Stream Key** (khóa xác thực khi push).

```
Channel: live-shopping-vn
├── Type: STANDARD (hoặc BASIC / ADVANCED-SD / ADVANCED-HD)
├── Latency mode: LOW
├── Authorized: false (public playback) hoặc true (playback token)
├── Recording: ENABLED → arn:aws:s3:::recordings-bucket/ivs/
└── Tags: { "app": "shop-live" }
```

### 3.2 Stream Key (Khóa Luồng)

**Stream Key** gắn với channel — bí mật dùng khi encoder (OBS) kết nối RTMPS. Rotate key khi lộ; có thể tạo nhiều key (primary/backup) cho failover encoder.

### 3.3 Latency Mode (Chế Độ Độ Trễ)

| Mode | Độ trễ điển hình | Giao thức phát |
|------|------------------|----------------|
| **NORMAL** (Standard) | ~15 giây | HLS CMAF |
| **LOW** | ~3–5 giây | Low-latency HLS |
| **REAL_TIME** | < 1 giây | WebRTC (Stage) |

Chi tiết: [2-latency-modes.md](./2-latency-modes.md)

### 3.4 Timed Metadata (Metadata Theo Thời Gian)

Nhúng payload JSON vào luồng tại timestamp cụ thể — player SDK fire event → UI cập nhật (hiện sản phẩm, bắt đầu poll). Khác **ID3 tags** broadcast: API `PutMetadata` gửi từ backend.

### 3.5 IVS Chat

Dịch vụ chat realtime riêng (cùng hệ sinh thái IVS): **Room**, **Messaging API**, token IAM/room. Không đi qua video pipeline nhưng thường triển khai cùng channel.

### 3.6 Recording & Playback SDK

- **Recording**: tự động ghi HLS vào S3 (VOD sau live)
- **Playback SDK**: player tích hợp quality selection, timed metadata events, low-latency buffer tuning

---

## 4. Pipeline Interactive Live

### Livestream Bán Hàng (Typical)

```
┌─────────┐   RTMPS    ┌────────────┐   HLS Low    ┌──────────────┐
│  OBS    │ ────────▶ │ IVS Channel│ ──────────▶ │ Web App      │
│ Streamer│           │ + Metadata │             │ IVS Player   │
└─────────┘           └─────┬──────┘             │ + Product UI │
                            │                    └──────────────┘
                     PutMetadata API
                            │
                     ┌──────▼──────┐
                     │  Backend    │
                     │  (Lambda/   │
                     │   API GW)   │
                     └─────────────┘

IVS Chat Room ◀── viewers (WebSocket-style SDK)
```

### IVS Real-Time (Stage) — Panel / Game

```
Host + Guests ── WebRTC ──▶ IVS Stage
                              │
                              ▼
                         Subscribers (<1s)
```

Dùng **Real-Time Streaming** (Stage API) thay channel RTMPS khi cần nhiều người publish (co-host, panel).

---

## 5. Nội Dung Chi Tiết

```
07-ivs/
├── README.md                 ← [BẠN ĐANG Ở ĐÂY] Tổng quan
├── 1-channel-ingest.md       Channel types, RTMPS, Stream Key, OBS
├── 2-latency-modes.md        NORMAL, LOW, REAL_TIME (WebRTC Stage)
├── 3-timed-metadata.md       PutMetadata, đồng bộ UI tương tác
├── 4-ivs-chat.md             Room, token, moderation
└── 5-recording-playback.md   S3 recording, Player SDK (JS/iOS/Android)
```

### Bản Đồ Học Tập Đề Xuất

```
Bắt đầu tại đây (README)
         │
         ▼
1-channel-ingest.md        ← Tạo channel, cấu hình OBS
         │
         ▼
2-latency-modes.md         ← Chọn NORMAL / LOW / REAL_TIME
         │
         ▼
3-timed-metadata.md        ← UI tương tác đồng bộ video
         │
         ▼
4-ivs-chat.md              ← Chat realtime
         │
         ▼
5-recording-playback.md    ← VOD replay + SDK tích hợp
```

---

## 6. Câu Hỏi Phỏng Vấn Thường Gặp

### Câu hỏi cơ bản

**Q: IVS khác gì MediaLive?**

> **IVS** là managed stack end-to-end cho **interactive low-latency** (ingest → CDN → player SDK), setup nhanh, có Timed Metadata và Chat. **MediaLive** là encoder broadcast chuyên nghiệp, output tới **MediaPackage** cho DRM, SCTE-35, time-shift — độ trễ cao hơn nhưng phù hợp OTT TV.

**Q: Stream Key dùng để làm gì?**

> **Stream Key** xác thực encoder khi push **RTMPS** lên **Ingest endpoint**. Chỉ streamer/backend tin cậy được giữ key; viewer dùng **Playback URL** riêng, không cần stream key.

**Q: Làm sao giảm độ trễ cho viewer?**

> Chọn **latency mode LOW** trên channel; dùng **IVS Player SDK** (không chỉ HLS.js generic); giảm segment duration nếu platform cho phép; với tương tác sub-second dùng **Real-Time (Stage/WebRTC)**.

### Câu hỏi nâng cao

**Q: Timed metadata hoạt động thế nào?**

> Backend gọi `PutMetadata` với payload (≤ 1 KB). IVS nhúng vào luồng tại thời điểm ingest. Player SDK nhận event `PlayerState.METADATA` / tương đương — app map timestamp với UI (flash sale, poll). Độ trễ metadata ≈ độ trễ video mode đang dùng.

**Q: IVS có hỗ trợ DRM không?**

> **IVS** không cung cấp DRM như MediaPackage. Nội dung premium/OTT thường dùng MediaLive + MediaPackage. IVS phù hợp nội dung công khai hoặc bảo vệ bằng **playback authorization** (token) thay DRM.

**Q: Pricing IVS?**

> Tính theo **input duration** (phút stream vào IVS) và **output** (data transfer ra viewer), tuỳ **channel type** (Basic/Standard/Advanced). **IVS Chat** và **Real-Time** có bảng giá riêng. Dùng [Amazon IVS Pricing](https://aws.amazon.com/ivs/pricing/).

---

## 🔗 Tài Liệu Tham Khảo

- [Amazon IVS User Guide](https://docs.aws.amazon.com/ivs/)
- [IVS Low-Latency Streaming](https://docs.aws.amazon.com/ivs/latest/LowLatencyUserGuide/)
- [IVS Real-Time Streaming](https://docs.aws.amazon.com/ivs/latest/RealTimeUserGuide/)
- [IVS Chat Messaging API](https://docs.aws.amazon.com/ivs/latest/ChatUserGuide/)
- [IVS Player SDK](https://docs.aws.amazon.com/ivs/latest/LowLatencyUserGuide/player.html)
- [Amazon IVS Pricing](https://aws.amazon.com/ivs/pricing/)

---

**Cập Nhật Lần Cuối:** 2026-06-04
**Phần Trước:** [06-mediatailor/](../06-mediatailor/README.md) — SSAI & Channel Assembly
**Phần Tiếp Theo:** [08-kinesis-video/](../08-kinesis-video/README.md) — Kinesis Video Streams
