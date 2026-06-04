# Kinesis Video Streams — Producer & Consumer

> **Producer** (bên ghi) đẩy video từ thiết bị lên **KVS Stream** qua **PutMedia**. **Consumer** (bên đọc) lấy dữ liệu qua **GetMedia** (raw) hoặc **HLS/DASH streaming session** (phát lại). Hiểu hai vai trò này là nền tảng mọi pipeline IoT video trên AWS.

## 📚 Mục Lục

1. [Producer — Gửi Video Lên KVS](#1-producer--gửi-video-lên-kvs)
2. [PutMedia API & Fragment](#2-putmedia-api--fragment)
3. [Producer SDK & GStreamer](#3-producer-sdk--gstreamer)
4. [Consumer — Đọc Video Từ KVS](#4-consumer--đọc-video-từ-kvs)
5. [IAM & Bảo Mật](#5-iam--bảo-mật)
6. [Thực Hành & Troubleshooting](#6-thực-hành--troubleshooting)
7. [Câu Hỏi Phỏng Vấn](#7-câu-hỏi-phỏng-vấn)

---

## 1. Producer — Gửi Video Lên KVS

### Vai Trò

**Producer** là bất kỳ ứng dụng/thiết bị nào gọi **PutMedia** để ghi **fragment** vào **Video Stream**. Mỗi stream thường có **một producer active** tại một thời điểm (ghi đồng thời nhiều producer cùng stream có thể gây xung đột timestamp).

```
┌─────────────────┐     PutMedia (HTTPS)      ┌─────────────────────┐
│  Edge Device    │ ────────────────────────▶ │  Kinesis Video      │
│  • Raspberry Pi │     MKV + H.264 chunks    │  Stream             │
│  • IP Camera    │                           │  factory-cam-01     │
│  • Robot        │                           └─────────────────────┘
└─────────────────┘
```

### Định Dạng Media Khuyến Nghị

| Thành phần | Khuyến nghị | Ghi chú |
|------------|-------------|---------|
| **Container** | MKV — Matroska | KVS Producer SDK đóng gói mặc định |
| **Video codec** | H.264 (AVC) | H.265 hỗ trợ hạn chế theo use case |
| **Audio** | AAC (tuỳ chọn) | Nhiều pipeline ML chỉ cần video |
| **Keyframe** | Đều đặn (1–2s) | Giúp seek và decode ổn định |

**MKV** — Matroska — container chứa H.264 NAL units; **PutMedia** nhận luồng byte MKV liên tục, không phải file MP4 hoàn chỉnh từng lần upload.

---

## 2. PutMedia API & Fragment

### PutMedia Hoạt Động Thế Nào

**PutMedia** là API **streaming HTTP** (không phải single request upload file):

```
Producer                          KVS
   │                                │
   │── POST /putMedia ─────────────▶│ Mở session
   │── chunk MKV fragment 1 ───────▶│ Lưu + timestamp
   │── chunk MKV fragment 2 ───────▶│
   │── ...                          │
   │── close connection ───────────▶│ Kết thúc session
```

### Fragment & Timestamp

Mỗi **fragment** gắn:

| Timestamp | Nguồn | Dùng cho |
|-----------|-------|----------|
| **Producer Timestamp** | Đồng hồ thiết bị | Đồng bộ sự kiện ML, replay theo thời gian thực tế |
| **Server Timestamp** | KVS khi nhận | Billing, latency monitoring |

```
Timeline trên stream:
|---- frag 1 ----|---- frag 2 ----|---- frag 3 ----|
t0              t1              t2              t3
     Producer TS: 10:00:01.000 → 10:00:04.000
```

> **Lưu ý:** Đồng hồ thiết bị lệch (NTP — Network Time Protocol — Giao Thức Thời Gian Mạng không sync) có thể làm consumer seek sai — production nên **sync NTP** trên camera.

### Acknowledgment Event

Producer SDK nhận **AckEvent** — Sự Kiện Xác Nhận — báo fragment đã persist. Dùng để backpressure (giảm tốc gửi nếu mạng yếu) và metrics.

---

## 3. Producer SDK & GStreamer

### KVS Producer SDK

**Producer SDK** — Bộ SDK Producer — có bản **C++** và **Java** (và binding cho embedded):

| Tính năng | Mô tả |
|-----------|-------|
| **Automatic fragmenting** | Chia MKV theo kích thước/thời gian |
| **Storage pressure** | Buffer local khi offline, sync khi có mạng |
| **Credential provider** | Lấy temporary credentials từ **Cognito** hoặc IAM |
| **Callbacks** | Status, error, persisted ACK |

```
Luồng khởi tạo Producer (khái niệm):
1. createKinesisVideoClient (region, credentials)
2. createKinesisVideoStream (stream name, retention, retention period)
3. putKinesisVideoFrame (encoded frame từ camera pipeline)
4. stopKinesisVideoStream khi shutdown
```

### GStreamer Plugin

**GStreamer** — framework pipeline multimedia — có plugin `kvssink` gắn vào pipeline:

```bash
# Ví dụ khái niệm (cần build plugin theo AWS guide)
gst-launch-1.0 v4l2src ! video/x-raw ! x264enc ! kvssink stream-name="my-stream"
```

Phù hợp khi thiết bị đã dùng GStreamer (NVIDIA Jetson, Linux camera).

### AWS IoT vs IAM Credentials

| Cách xác thực | Use case |
|---------------|----------|
| **IAM** — Identity and Access Management | Gateway/server gắn role EC2/ECS |
| **Cognito** | Device fleet — mỗi camera identity riêng |
| **IoT Core + X.509** | Certificate trên thiết bị, assume role ghi KVS |

---

## 4. Consumer — Đọc Video Từ KVS

### Hai Kiểu Consumer Chính

```
                    ┌─────────────────────┐
                    │   KVS Stream        │
                    └──────────┬──────────┘
                               │
              ┌────────────────┼────────────────┐
              ▼                ▼                ▼
       ┌─────────────┐  ┌─────────────┐  ┌─────────────────┐
       │  GetMedia   │  │ HLS Session │  │ Rekognition     │
       │  (raw MKV)  │  │ DASH Session│  │ Stream Processor│
       └─────────────┘  └─────────────┘  └─────────────────┘
              │                │
              ▼                ▼
        Lambda / app      Web player
        decode + ML       (m3u8 / mpd)
```

### GetMedia API

**GetMedia** trả về luồng byte **MKV** tương tự producer gửi:

| Tham số | Mô tả |
|---------|-------|
| `StreamARN` | ARN stream cần đọc |
| `StartSelector` | `StartFragmentNumber` hoặc `ProducerTimestamp` / `ServerTimestamp` |
| `FragmentSelectorType` | Loại timestamp dùng seek |

```
Consumer loop (logic):
1. GetMedia với StartSelector = NOW hoặc timestamp sự kiện
2. Đọc chunk từ HTTP response body
3. Parse MKV → decode H.264 → frame cho OpenCV / custom ML
4. Dùng NextFragmentNumber / continuation để đọc tiếp
```

**Use case GetMedia:** phân tích custom (không dùng Rekognition), archive sang S3, sync multi-region.

### HLS / DASH Streaming Session

Cho **playback** trên trình duyệt/app mà không tự build packager:

| API | Output |
|-----|--------|
| `GetHLSStreamingSessionURL` | URL `.m3u8` tạm thời |
| `GetDASHStreamingSessionURL` | URL `.mpd` tạm thời |

**Streaming session** — Phiên Phát — có TTL (thời gian sống); cần refresh URL trước khi hết hạn. Chi tiết: [4-retention-lifecycle.md](./4-retention-lifecycle.md)

### Kinesis Data Streams (Metadata)

Một số kiến trúc gửi **metadata** (GPS, sensor) song song qua **Kinesis Data Streams** — không nhầm với **Kinesis Video Streams** (cùng họ tên nhưng API và billing khác).

---

## 5. IAM & Bảo Mật

### Policy Mẫu (Khái Niệm)

**Producer** cần quyền ghi:

```json
{
  "Effect": "Allow",
  "Action": [
    "kinesisvideo:PutMedia",
    "kinesisvideo:DescribeStream",
    "kinesisvideo:GetDataEndpoint"
  ],
  "Resource": "arn:aws:kinesisvideo:region:account:stream/stream-name/*"
}
```

**Consumer** cần quyền đọc:

```json
{
  "Effect": "Allow",
  "Action": [
    "kinesisvideo:GetMedia",
    "kinesisvideo:GetHLSStreamingSessionURL",
    "kinesisvideo:DescribeStream"
  ],
  "Resource": "arn:aws:kinesisvideo:region:account:stream/stream-name/*"
}
```

### Mã Hoá

| Lớp | Cơ chế |
|-----|--------|
| **In transit** | TLS 1.2+ cho PutMedia/GetMedia |
| **At rest** | **KMS** — Key Management Service — Dịch Vụ Quản Lý Khóa — SSE với AWS managed hoặc CMK |

---

## 6. Thực Hành & Troubleshooting

### Tạo Stream (CLI)

```bash
aws kinesisvideo create-stream \
  --stream-name demo-camera-01 \
  --data-retention-in-hours 24 \
  --media-type "video/h264" \
  --region ap-southeast-1
```

### Lỗi Thường Gặp

| Triệu chứng | Nguyên nhân | Hướng xử lý |
|-------------|-------------|-------------|
| `ResourceNotFoundException` | Sai stream name/region | `describe-stream` kiểm tra ARN |
| PutMedia disconnect sớm | Network / idle timeout | Giảm bitrate, tăng keyframe, retry SDK |
| GetMedia không có data | StartSelector sau retention | Chọn timestamp trong window retention |
| Video giật / corrupt | Bitrate quá cao, thiếu keyframe | Cấu hình encoder CBR/VBR ổn định |
| Multi-producer conflict | Hai device ghi cùng stream | Mỗi camera một stream |

### Giới Hạn Cần Nhớ

- **Ingest bandwidth** theo region — thiết kế fleet camera phải aggregate bitrate
- **Fragment size** quá lớn tăng latency consumer realtime
- Producer SDK **buffer offline** — cần disk đủ trên edge nếu mạng mất

---

## 7. Câu Hỏi Phỏng Vấn

**Q: PutMedia khác upload S3 thế nào?**

> **PutMedia** là **long-lived HTTP stream** ghi fragment liên tục với timestamp cho **time-indexed playback** và **GetMedia seek**. **S3 PutObject** là object file — không có mô hình fragment realtime của KVS; thường phải tự xây packager.

**Q: Một stream có nhiều consumer không?**

> Có. Nhiều **GetMedia** hoặc **HLS session** đồng thời đọc cùng stream — billing tính **data out** mỗi consumer. **Rekognition processor** cũng là một dạng consumer managed.

**Q: Khi mất mạng producer làm gì?**

> **Producer SDK** buffer fragment local (cấu hình storage), reconnect **PutMedia** khi online — tránh mất video gap nếu buffer đủ. Production cần giám sát buffer full → drop policy.

---

**Cập Nhật Lần Cuối:** 2026-06-04
**Tiếp Theo:** [2-webrtc-signaling.md](./2-webrtc-signaling.md) — WebRTC & Signaling Channel
