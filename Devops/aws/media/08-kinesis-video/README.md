# Amazon Kinesis Video Streams — Luồng Video IoT & Phân Tích (KVS)

> **Amazon Kinesis Video Streams** — KVS — Luồng Video Kinesis là dịch vụ nhận, lưu trữ và phát lại video từ **thiết bị IoT** (Internet of Things — Mạng Vạn Vật), camera IP, drone, robot — tối ưu cho **phân tích realtime** (real-time analytics), **ML** (Machine Learning — Học Máy), và **WebRTC** hai chiều (two-way video). Khác **IVS** (interactive broadcast) hay **MediaLive** (OTT broadcast), KVS tập trung vào **ingest liên tục từ nhiều edge device** và **consumer đọc frame** cho pipeline AI.

## 📚 Mục Lục (Table of Contents)

1. [KVS Là Gì?](#1-kvs-là-gì)
2. [Kiến Trúc Tổng Quan](#2-kiến-trúc-tổng-quan)
3. [Các Khái Niệm Cốt Lõi](#3-các-khái-niệm-cốt-lõi)
4. [Pipeline IoT Video Analytics](#4-pipeline-iot-video-analytics)
5. [Nội Dung Chi Tiết](#5-nội-dung-chi-tiết)
6. [Câu Hỏi Phỏng Vấn Thường Gặp](#6-câu-hỏi-phỏng-vấn-thường-gặp)

---

## 1. KVS Là Gì?

**Kinesis Video Streams (KVS)** là dịch vụ **managed video ingestion & storage** (nhập và lưu video được quản lý) của AWS. Video được ghi dưới dạng **fragment** (mảnh) có timestamp, cho phép consumer đọc theo thời gian thực hoặc **playback** (phát lại) qua HLS/DASH.

### Vị Trí Trong Hệ Sinh Thái AWS Media

```
                 ┌──────────── IOT / SECURITY / ML PIPELINE ─────────────────┐
                 │                                                          │
[Camera/Device] → [KVS Producer SDK] → [KVS Stream] → [Consumer / Rekognition]
  H.264/MKV         PutMedia API            Lưu fragment      GetMedia / ML
  WebRTC ingest     hoặc GStreamer          + retention       Lambda / KDA
```

### So Sánh KVS vs IVS vs MediaLive

| Tiêu Chí | Kinesis Video Streams | Amazon IVS | MediaLive + MediaPackage |
|----------|----------------------|------------|--------------------------|
| **Nguồn ingest** | Camera IoT, embedded, GStreamer | OBS/encoder RTMPS | Broadcast RTMP/RTP/HLS |
| **Số luồng** | Hàng nghìn camera/device | Một vài channel/event | Vài kênh live chất lượng cao |
| **Mục tiêu** | Lưu trữ + phân tích ML | Phát tới hàng triệu viewer | OTT phát sóng TV |
| **Consumer** | GetMedia, Rekognition, Lambda | IVS Player SDK | CDN + player HLS/DASH |
| **WebRTC** | Có (signaling channel) | Real-time Stage | Không native |
| **DRM / ads** | Không | Hạn chế | Đầy đủ |
| **Độ trễ viewer** | Không phải focus chính | ~5s / <1s | ~15–30s |
| **Use case** | An ninh, nhà máy, telemedicine | Livestream bán hàng | Thể thao, OTT TV |

> **Khi nào dùng KVS?**
> - Nhiều camera/edge device gửi video liên tục lên cloud
> - Cần **Rekognition** face/label/object detection trên luồng
> - **WebRTC** hai chiều (doorbell, telemedicine, robot điều khiển)
> - Lưu buffer ngắn (giờ → vài ngày) rồi xử lý hoặc archive S3

> **Khi nào KHÔNG dùng KVS?**
> - Phát live cho hàng triệu người xem → **IVS** hoặc **MediaLive + CloudFront**
> - VOD transcoding file lớn → **MediaConvert**
> - Cần DRM, SCTE-35, SSAI → **MediaPackage + MediaTailor**

---

## 2. Kiến Trúc Tổng Quan

### Các Thành Phần Chính

```
┌────────────────────────────────────────────────────────────────────────────┐
│                    AMAZON KINESIS VIDEO STREAMS                            │
│                                                                            │
│  ┌──────────────────┐         ┌─────────────────────────────────────┐   │
│  │  Video Stream    │         │  Storage Layer                      │   │
│  │                  │         │  • Fragment theo timestamp          │   │
│  │  • Stream name   │────────▶│  • Retention (1h → unlimited*)      │   │
│  │  • Data retention│         │  • HLS/DASH endpoint (playback)     │   │
│  │  • KMS encryption│         └─────────────────────────────────────┘   │
│  └────────┬─────────┘                        │                            │
│           │                                  ▼                            │
│  ┌────────▼─────────┐    ┌──────────────────────────────────────────┐   │
│  │  Producer        │    │  Consumers                               │   │
│  │  • C++ / Java SDK  │    │  • GetMedia API (raw MKV)                │   │
│  │  • PutMedia        │    │  • HLS/DASH session URL                  │   │
│  │  • GStreamer plugin│    │  • Rekognition Video stream processor    │   │
│  └──────────────────┘    │  • Kinesis Data Streams → Lambda/KDA     │   │
│                            └──────────────────────────────────────────┘   │
│  ┌──────────────────┐                                                    │
│  │  Signaling       │  WebRTC: STUN/TURN + SDP exchange                 │
│  │  Channel         │  Master (viewer/app) ↔ Viewer (device)             │
│  └──────────────────┘                                                    │
└────────────────────────────────────────────────────────────────────────────┘
```

### Luồng Dữ Liệu (Data Flow)

```
1. Tạo KVS Stream (Console/CLI) — cấu hình retention, KMS
2. Device cài Producer SDK → PutMedia gửi fragment H.264 trong container MKV
3. KVS lưu fragment có server timestamp + producer timestamp
4a. Consumer GetMedia → đọc byte stream MKV → decode → ML inference
4b. Hoặc tạo HLS/DASH streaming session → player web/mobile xem lại
4c. Hoặc gắn Rekognition Stream Processor → label/face events → SNS/Lambda
5. Retention hết hạn → fragment tự xoá (không archive trừ khi copy sang S3)
```

---

## 3. Các Khái Niệm Cốt Lõi

### 3.1 Video Stream (Luồng Video)

**Stream** là tài nguyên logic — mỗi camera hoặc nguồn thường map **1 stream**. ARN dùng cho IAM policy và API.

```
Stream: factory-line-camera-03
├── Data retention: 168 giờ (7 ngày)
├── Media type: video/h264 (trong MKV)
├── KMS key: alias/aws/kinesisvideo (hoặc CMK)
└── Tags: { "site": "hanoi-factory", "line": "3" }
```

### 3.2 Fragment (Mảnh Video)

**Fragment** là đơn vị lưu trữ nhỏ (thường vài giây), có **Producer Timestamp** (thời gian thiết bị) và **Server Timestamp** (thời gian KVS nhận). Consumer đọc theo khoảng thời gian hoặc **continuation token** (token tiếp nối).

### 3.3 Producer vs Consumer

| Vai trò | API / Công cụ | Mô tả |
|---------|---------------|-------|
| **Producer** | `PutMedia`, Producer SDK | Gửi video từ device lên stream |
| **Consumer** | `GetMedia`, `GetHLSStreamingSessionURL` | Đọc raw hoặc phát HLS/DASH |

Chi tiết: [1-producer-consumer.md](./1-producer-consumer.md)

### 3.4 WebRTC & Signaling Channel

**Signaling Channel** — Kênh Báo Hiệu — trung gian trao đổi **SDP** — Session Description Protocol — Mô Tả Phiên và **ICE** — Interactive Connectivity Establishment — Thiết Lập Kết Nối Tương Tác cho **peer-to-peer** WebRTC. KVS cung cấp managed signaling; media có thể đi P2P hoặc qua TURN.

Chi tiết: [2-webrtc-signaling.md](./2-webrtc-signaling.md)

### 3.5 Rekognition Integration

**Amazon Rekognition Video** đọc trực tiếp từ KVS stream qua **Stream Processor** — Bộ Xử Lý Luồng — phát hiện face, label, celebrity, content moderation.

Chi tiết: [3-rekognition-integration.md](./3-rekognition-integration.md)

### 3.6 Retention & Playback

**Data retention** (thời gian giữ dữ liệu): từ 1 giờ đến không giới hạn (theo region/policy). **GetMedia** cho xử lý batch/realtime; **HLS/DASH session** cho xem trên player.

Chi tiết: [4-retention-lifecycle.md](./4-retention-lifecycle.md)

---

## 4. Pipeline IoT Video Analytics

### Camera An Ninh + Cảnh Báo

```
┌──────────┐  PutMedia   ┌─────────────┐  Stream Processor  ┌──────────────┐
│ IP Camera│ ──────────▶ │ KVS Stream  │ ─────────────────▶ │ Rekognition  │
│ + SDK    │             │ (24h retain)│                    │ Face / Label │
└──────────┘             └──────┬──────┘                    └──────┬───────┘
                                │                                  │
                                │ GetMedia (optional archive)      ▼
                                ▼                           ┌──────────────┐
                         ┌─────────────┐                    │ Lambda → SNS │
                         │ S3 (archive)│                    │ Alert app    │
                         │ long-term   │                    └──────────────┘
                         └─────────────┘
```

### Telemedicine / Doorbell (WebRTC)

```
Patient App (Master) ◀── Signaling Channel ──▶ Device (Viewer)
         │                                              │
         └──────── WebRTC media (STUN/TURN) ────────────┘
                    Optional: ghi vào KVS Stream
```

### Nhà Máy — Giám Sát + KDA

```
100 cameras → 100 KVS Streams → Kinesis Data Streams (metadata tags)
                                      │
                                      ▼
                              Kinesis Data Analytics / Lambda
                              → Dashboard anomaly detection
```

---

## 5. Nội Dung Chi Tiết

```
08-kinesis-video/
├── README.md                 ← [BẠN ĐANG Ở ĐÂY] Tổng quan
├── 1-producer-consumer.md    Producer SDK, PutMedia, GetMedia, GStreamer
├── 2-webrtc-signaling.md     Signaling channel, Master/Viewer, STUN/TURN
├── 3-rekognition-integration.md  Stream processor, face/label detection
└── 4-retention-lifecycle.md  Retention, HLS/DASH playback, lifecycle
```

### Bản Đồ Học Tập Đề Xuất

```
Bắt đầu tại đây (README)
         │
         ▼
1-producer-consumer.md     ← Gửi và đọc video từ stream
         │
         ▼
2-webrtc-signaling.md      ← Hai chiều realtime với WebRTC
         │
         ▼
3-rekognition-integration.md ← ML phân tích trên luồng
         │
         ▼
4-retention-lifecycle.md   ← Retention, playback, vận hành
```

---

## 6. Câu Hỏi Phỏng Vấn Thường Gặp

### Câu hỏi cơ bản

**Q: KVS khác IVS thế nào?**

> **KVS** nhận video từ **nhiều thiết bị IoT/camera**, lưu fragment để **phân tích ML** hoặc playback nội bộ — không tối ưu phát cho hàng triệu viewer. **IVS** là **managed live broadcast** tới audience lớn với Player SDK và độ trễ thấp cho tương tác.

**Q: Producer và Consumer trong KVS là gì?**

> **Producer** ghi video lên stream qua **PutMedia** (SDK C++/Java hoặc GStreamer). **Consumer** đọc qua **GetMedia** (raw MKV) hoặc tạo **HLS/DASH streaming session** để player phát lại.

**Q: Video lưu ở đâu trong KVS?**

> KVS quản lý storage phía sau — bạn không chọn bucket S3 trực tiếp. Dữ liệu là **fragment** có retention; muốn lưu lâu dài phải **copy sang S3** (Lambda, custom consumer) hoặc dùng **S3 integration** pattern.

### Câu hỏi nâng cao

**Q: Tích hợp Rekognition với KVS?**

> Tạo **Rekognition Stream Processor** gắn ARN KVS stream + IAM role. Rekognition đọc luồng, emit events (face detected, label) tới **Kinesis Data Streams** hoặc **SNS**. Không cần tự decode H.264 trong Lambda nếu dùng processor có sẵn.

**Q: WebRTC trên KVS hoạt động ra sao?**

> **Signaling Channel** trao đổi SDP/ICE giữa **Master** (thường là app/web) và **Viewer** (device). Media peer-to-peer; KVS có thể **ghi song song** vào Video Stream. Cần **STUN/TURN** endpoint do AWS cung cấp theo region.

**Q: Pricing KVS?**

> Tính theo **data ingested** (GB ghi vào), **data consumed** (GB đọc GetMedia/HLS), **signaling messages**, **WebRTC TURN** minutes, và **days stored** (retention). Xem [Kinesis Video Streams Pricing](https://aws.amazon.com/kinesis/video-streams/pricing/).

---

## 🔗 Tài Liệu Tham Khảo

- [Kinesis Video Streams Developer Guide](https://docs.aws.amazon.com/kinesisvideostreams/latest/dg/)
- [KVS Producer SDK](https://docs.aws.amazon.com/kinesisvideostreams/latest/dg/producer-sdk.html)
- [KVS WebRTC](https://docs.aws.amazon.com/kinesisvideostreams-webrtc-dg/latest/devguide/)
- [Rekognition Video Stream Processor](https://docs.aws.amazon.com/rekognition/latest/dg/streaming-video.html)
- [Kinesis Video Streams Pricing](https://aws.amazon.com/kinesis/video-streams/pricing/)

---

**Cập Nhật Lần Cuối:** 2026-06-04
**Phần Trước:** [07-ivs/](../07-ivs/README.md) — Interactive Video Service
**Phần Tiếp Theo:** [09-cdn-delivery/](../09-cdn-delivery/README.md) — CloudFront & CDN Delivery
