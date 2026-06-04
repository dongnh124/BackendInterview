# Kinesis Video Streams — Retention, GetMedia & Playback

> **Data retention** — Thời Gian Giữ Dữ Liệu — quyết định fragment được lưu bao lâu trên KVS. **GetMedia** đọc raw MKV cho xử lý; **HLS/DASH streaming session** phát lại trên player. Hiểu lifecycle giúp cân bằng **chi phí**, **compliance**, và **khả năng điều tra sự cố**.

## 📚 Mục Lục

1. [Retention Policy](#1-retention-policy)
2. [Lifecycle Fragment Trên Stream](#2-lifecycle-fragment-trên-stream)
3. [GetMedia — Đọc Raw Stream](#3-getmedia--đọc-raw-stream)
4. [HLS & DASH Playback Session](#4-hls--dash-playback-session)
5. [Archive Sang S3](#5-archive-sang-s3)
6. [Giám Sát & Vận Hành](#6-giám-sát--vận-hành)
7. [Chi Phí & Tối Ưu](#7-chi-phí--tối-ưu)
8. [Câu Hỏi Phỏng Vấn](#8-câu-hỏi-phỏng-vấn)

---

## 1. Retention Policy

### Data Retention Là Gì?

Khi tạo **Video Stream**, bạn đặt **DataRetentionInHours** — số giờ KVS giữ fragment. Sau thời gian này, fragment **tự xoá** — không recover (trừ khi đã archive nơi khác).

```
Stream: security-cam-lobby
├── DataRetentionInHours: 72    (3 ngày)
├── Hoặc: 1                     (buffer ngắn, chỉ ML realtime)
├── Hoặc: 8760                  (1 năm — chi phí storage cao)
└── Update: update-data-retention (có thể tăng/giảm)
```

### Chọn Retention Theo Use Case

| Use case | Retention gợi ý | Lý do |
|----------|-----------------|-------|
| ML realtime only | 1–24 giờ | Chỉ cần buffer ngắn cho processor |
| An ninh văn phòng | 72–168 giờ | Điều tra sự cố trong tuần |
| Tuân thủ (compliance) | Archive S3 + retention ngắn trên KVS | KVS đắt cho lưu dài — S3 Glacier rẻ hơn |
| Demo / POC | 24 giờ | Tiết kiệm |

### Thay Đổi Retention

```bash
aws kinesisvideo update-data-retention \
  --stream-name security-cam-lobby \
  --data-retention-in-hours 168 \
  --operation-type UPDATE_DATA_RETENTION \
  --region ap-southeast-1
```

| `OperationType` | Hành vi |
|-----------------|---------|
| `UPDATE_DATA_RETENTION` | Đặt retention mới |
| `INCREASE_DATA_RETENTION` | Tăng — fragment cũ giữ thêm |
| `DECREASE_DATA_RETENTION` | Giảm — fragment cũ hơn window có thể bị xoá sớm |

> **Giảm retention** có thể xoá dữ liệu không khôi phục — cần change management production.

---

## 2. Lifecycle Fragment Trên Stream

### Timeline Fragment

```
Now ─────────────────────────────────────────▶ Past
     │◀────── Retention window (e.g. 72h) ──────▶│
     │  frag N │ frag N-1 │ ... │ frag oldest    │
     │           still readable via GetMedia/HLS │
                                                  │ frag expired → deleted
```

### Fragment Number & Timestamp

Consumer seek bằng:

| Selector | Dùng khi |
|----------|----------|
| **Producer timestamp** | Đồng bộ với sự kiện thiết bị (cảnh báo lúc 14:32:05) |
| **Server timestamp** | Đo latency ingest |
| **Fragment number** | Tiếp tục đọc từ lần GetMedia trước (pagination) |

```
GetMedia pagination:
  Request 1 → fragments 100-150 + NextFragmentNumber
  Request 2 → StartSelector = NextFragmentNumber từ response 1
```

### Không Có S3 Bucket Mặc Định

KVS **không** expose S3 path cho fragment — storage opaque. **Lifecycle** = retention hours, không phải S3 Intelligent-Tiering. Muốn **lifecycle S3** phải **export** chủ động.

---

## 3. GetMedia — Đọc Raw Stream

### Khi Nào Dùng GetMedia

| Scenario | Lý do |
|----------|-------|
| Custom ML decode | Frame-by-frame OpenCV |
| Clip extraction | Cắt đoạn gửi S3 sau alert Rekognition |
| Forensic replay | App desktop nội bộ |
| Transcode downstream | Đưa vào MediaConvert job input |

### GetMedia Request Flow

```
1. describe-stream → lấy StreamARN
2. get-data-endpoint (APIName=GET_MEDIA) → endpoint URL
3. get-media với StartSelector
4. Đọc response body stream (MKV)
5. Parse EBML/MKV → extract H.264 NAL → decode
```

### StartSelector Ví Dụ

```json
{
  "StartSelectorType": "SERVER_TIMESTAMP",
  "StartTimestamp": 1717500000000
}
```

```json
{
  "StartSelectorType": "FRAGMENT_NUMBER",
  "AfterFragmentNumber": "12345678901234567890"
}
```

### Live Edge vs Historical

| Chế độ | StartSelector | Use case |
|--------|---------------|----------|
| **Live tail** | `NOW` hoặc timestamp gần now | Consumer realtime ML |
| **Historical** | Timestamp sự cố | Xem lại sau alert |

**Latency GetMedia live:** thường vài giây sau ingest — không bằng WebRTC sub-second.

---

## 4. HLS & DASH Playback Session

### Tại Sao Cần Streaming Session?

**GetMedia** trả MKV raw — browser player cần **HLS** — HTTP Live Streaming — hoặc **DASH** — Dynamic Adaptive Streaming over HTTP. KVS **packager tạm thời** tạo manifest từ fragment trong retention window.

### API Flow

```
┌──────────┐  1. GetHLSStreamingSessionURL   ┌─────────────┐
│  Web App │ ──────────────────────────────▶ │  KVS        │
│          │  (StreamARN, PlaybackMode)     │  Packager   │
│          │◀────────────────────────────── │             │
│          │  URL: https://.../hls.m3u8     └─────────────┘
│          │  2. Player load m3u8
│          │  3. Player request segments
└──────────┘
```

### PlaybackMode

| Mode | Mô tả |
|------|-------|
| `LIVE` | Phát từ live edge (trễ vài giây) |
| `LIVE_REPLAY` | Live với khả năng seek trong buffer retention |
| `ON_DEMAND` | Phát từ timestamp cụ thể trong retention |

### DASH Session

**GetDASHStreamingSessionURL** tương tự — trả `.mpd` manifest — phù hợp player Android/Shaka.

### Session TTL & Refresh

- URL session **hết hạn** (thường vài phút đến vài giờ — kiểm tra doc region)
- App phải **gọi lại API** trước expiry — pattern giống signed URL ngắn hạn
- **DisplayFragmentTimestamp** — hiển thị timestamp trên player (debug sync)

### So Sánh HLS Session vs CloudFront + MediaPackage

| | KVS HLS session | MediaPackage + CDN |
|--|-----------------|-------------------|
| **Nguồn** | Camera IoT | Broadcast encoder |
| **Scale viewer** | Nhỏ/trung bình | Rất lớn |
| **DRM** | Không | Có |
| **Setup** | Gắn stream ARN | Channel + endpoint |

---

## 5. Archive Sang S3

### Pattern Archive

```
KVS Stream (retention 24h)
        │
        │ GetMedia consumer (Lambda scheduled hoặc on Rekognition alert)
        ▼
   S3 bucket
   ├── clips/2026/06/04/cam-01-incident-143205.mkv
   └── lifecycle → Glacier sau 90 ngày
```

### Lambda On Alert (Khái Niệm)

```
Rekognition event (Person detected)
  → Lambda: GetMedia từ timestamp - 30s đến + 30s
  → Upload MKV hoặc MP4 (ffmpeg) lên S3
  → Metadata DynamoDB (incidentId, streamArn, ttl)
```

### MediaConvert Sau Archive

Clip MKV trên S3 có thể **MediaConvert** sang H.264 MP4 nhiều bitrate cho portal review — tách pipeline ingest (KVS) và distribution (S3 + CloudFront).

### Kinesis Video Streams → S3 Tích Hợp

AWS có pattern **ghi trực tiếp** qua consumer app; không có nút "enable S3 backup" một click như IVS Recording — phải **tự implement** hoặc dùng solution reference architecture.

---

## 6. Giám Sát & Vận Hành

### CloudWatch Metrics (KVS)

| Metric | Ý nghĩa |
|--------|---------|
| `PutMedia.IncomingBytes` | Ingest rate |
| `PutMedia.IncomingFragments` | Fragment rate |
| `GetMedia.OutgoingBytes` | Consumer bandwidth |
| `GetMedia.OutgoingFragments` | Read pattern |
| `ErrorRate` | Lỗi API |

### Alarm Gợi Ý

```
• PutMedia bytes = 0 trong 15 phút (camera offline)
• ErrorRate > threshold
• Rekognition processor status FAILED
```

### Logging

- **CloudTrail** ghi API management (`create-stream`, `update-data-retention`)
- Data plane PutMedia/GetMedia không log nội dung video (privacy)

---

## 7. Chi Phí & Tối Ưu

### Thành Phần Billing

| Thành phần | Driver |
|------------|--------|
| **Ingest** | GB ghi vào stream |
| **Storage** | GB-hours × retention |
| **Consumption** | GB đọc GetMedia / HLS / DASH |
| **WebRTC** | TURN minutes, signaling messages (module khác) |

### Tối Ưu

| Chiến lược | Hiệu quả |
|------------|----------|
| Retention ngắn trên KVS + S3 dài hạn | Giảm KVS storage cost |
| Giảm resolution/bitrate camera | Giảm ingest + consumption |
| Một GetMedia consumer shared | Tránh duplicate read billing |
| Stop Rekognition khi không cần | Giảm phút phân tích |
| HLS session chỉ khi user xem | Tránh tạo session idle |

### Capacity Planning

```
Tổng ingest/day ≈ số camera × bitrate × giây hoạt động / 8
Tổng storage ≈ ingest/day × retention_days (xấp xỉ, có overhead fragment)
```

---

## 8. Câu Hỏi Phỏng Vấn

**Q: Retention hết hạn thì dữ liệu đi đâu?**

> Fragment **bị xoá vĩnh viễn** khỏi KVS. Muốn giữ lâu phải **archive S3** (hoặc export) trước khi hết retention hoặc trong quá trình vẫn còn window.

**Q: GetMedia vs HLS session?**

> **GetMedia** = raw **MKV byte stream** cho app/ML tự xử lý. **HLS session** = URL manifest cho **player** phát trong retention — KVS đóng vai packager tạm.

**Q: Xem video 1 tuần trước trên KVS?**

> Chỉ nếu **retention ≥ 1 tuần** và dùng **ON_DEMAND** / timestamp selector trong window. Quá retention → không đọc được — cần bản archive S3.

**Q: KVS thay S3 cho VOD platform?**

> **Không**. KVS cho **continuous ingest** time-indexed từ device. **VOD** scale dùng **S3 + MediaConvert + CloudFront**.

---

**Cập Nhật Lần Cuối:** 2026-06-04
**Phần Trước:** [3-rekognition-integration.md](./3-rekognition-integration.md)
**Tiếp Theo:** [09-cdn-delivery/](../09-cdn-delivery/README.md) — CloudFront & CDN Delivery
