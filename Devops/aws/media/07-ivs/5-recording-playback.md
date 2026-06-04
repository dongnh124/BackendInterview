# Amazon IVS — Recording & Playback SDK

> **Recording** (ghi hình) tự động lưu live stream thành **HLS** segments trên **S3** — Simple Storage Service — Lưu Trữ Đối Tượng Đơn Giản. **Playback SDK** (bộ SDK phát) cung cấp player tối ưu cho **LOW latency**, **Timed Metadata**, và quality switching trên web/iOS/Android.

## 📚 Mục Lục

1. [Recording Configuration](#1-recording-configuration)
2. [S3 Output & VOD Replay](#2-s3-output--vod-replay)
3. [IVS Player SDK](#3-ivs-player-sdk)
4. [Tích Hợp Ứng Dụng](#4-tích-hợp-ứng-dụng)
5. [So Sánh Player Khác](#5-so-sánh-player-khác)
6. [Câu Hỏi Phỏng Vấn](#6-câu-hỏi-phỏng-vấn)

---

## 1. Recording Configuration

### Recording Configuration Resource

**Recording Configuration** định nghĩa **đích S3** và quyền — gắn vào **Channel** qua `recordingConfigurationArn`.

```bash
aws ivs create-recording-configuration \
  --name "ivs-recordings-prod" \
  --destination-configuration '{
    "s3": {
      "bucketName": "my-ivs-recordings",
      "prefix": "live/"
    }
  }' \
  --region ap-northeast-1
```

### Gắn Vào Channel

```bash
aws ivs create-channel \
  --name "shop-channel" \
  --recording-configuration-arn arn:aws:ivs:region:account:recording-configuration/xxx \
  ...
```

Hoặc `update-channel` để bật recording sau.

### IAM Role IVS → S3

IVS cần **role** tin cậy S3:

```
Trust: ivs.amazonaws.com
Policy: s3:PutObject trên bucket/prefix
```

Thiếu quyền → recording **thất bại** im lặng hoặc status `FAILED` trong metrics — monitor **RecordingStartFailure**.

### Hành Vi Recording

```
Stream start (ingest healthy)
        │
        ▼
IVS ghi segments HLS vào S3 (theo session)
        │
        ▼
Stream stop
        │
        ▼
Manifest master + segments hoàn chỉnh trong prefix
```

| Trạng thái | Mô tả |
|------------|-------|
| Recording | Đang ghi trong khi live |
| Idle | Không có stream |
| Failed | Lỗi S3/IAM — cần alarm |

---

## 2. S3 Output & VOD Replay

### Cấu Trúc Object S3

```
s3://my-ivs-recordings/live/
└── ivs/v1/account_id/channel_id/YYYY/MM/DD/HH/MM/
    ├── index.m3u8          (master hoặc media playlist)
    ├── media_00001.ts
    ├── media_00002.ts
    └── ...
```

Prefix chính xác theo [IVS Recording docs](https://docs.aws.amazon.com/ivs/latest/LowLatencyUserGuide/record-to-s3.html) — dùng list API hoặc EventBridge khi recording end để index VOD catalog.

### Phát Lại Recording

```
Option A: S3 + CloudFront
  S3 bucket (OAC — Origin Access Control) → CloudFront → HLS player

Option B: Copy sang MediaConvert pipeline
  Recording → clip highlights → VOD library

Option C: IVS Player load recorded m3u8 URL (signed nếu private)
```

### Thumbnail & Clip

IVS không tự tạo thumbnail — dùng **MediaConvert** hoặc **Lambda@FFmpeg** trên file đầu tiên sau recording. **Clip** (highlight) cắt từ playlist bằng MediaConvert input HLS.

### Retention

- **S3 Lifecycle** — Tự Động Xoá/Chuyển Tier: xoá recording sau N ngày
- Tuân thủ GDPR: xoá theo user request nếu VOD gắn personal data

---

## 3. IVS Player SDK

### Tại Sao Dùng SDK Chính Thức

| Tính năng | IVS Player SDK | HLS.js thuần |
|-----------|----------------|--------------|
| LOW latency buffer | Tối ưu | Cần tune thủ công |
| Timed Metadata events | Native | Khó/không đầy đủ |
| Quality switching | Built-in | Manual |
| Rebuffer handling | IVS-tuned | Generic |
| DRM | Không (IVS không DRM) | N/A |

### Web — Cài Đặt

```html
<script src="https://player.live-video.net/1.x.x/amazon-ivs-player.min.js"></script>
```

```javascript
import { create, PlayerState } from 'amazon-ivs-player';

const player = create();
player.attachHTMLVideoElement(document.querySelector('#video'));

player.addEventListener(PlayerState.PLAYING, () => {
  console.log('Live playing');
});

player.addEventListener(PlayerState.REBUFFERING, () => {
  showSpinner();
});

player.load(playbackUrl);
player.play();
```

### iOS / Android

- **iOS:** Swift Package / CocoaPods `AmazonIVSPlayer`
- **Android:** Maven `com.amazonaws:ivs-player`

Pattern: `playerView` + `load(path)` + delegate metadata/quality.

### Quality & Bitrate

SDK tự **ABR** — Adaptive Bitrate Streaming — chọn rendition theo bandwidth. Developer có thể set **initial quality** hoặc cap max quality (tiết kiệm data mobile).

---

## 4. Tích Hợp Ứng Dụng

### Kiến Trúc Viewer App

```
┌─────────────────────────────────────────────────────────┐
│                    Viewer Web App                        │
│  ┌─────────────┐  ┌──────────────┐  ┌───────────────┐ │
│  │ IVS Player  │  │ IVS Chat SDK │  │ Product UI    │ │
│  │ (video)     │  │              │  │ (metadata)    │ │
│  └──────┬──────┘  └──────┬───────┘  └───────▲───────┘ │
│         │                │                   │         │
└─────────┼────────────────┼───────────────────┼─────────┘
          │                │                   │
          ▼                ▼            PutMetadata path
     playbackUrl      /api/chat-token    (host backend)
```

### Authorized Playback

```javascript
// Backend trả signed playback URL hoặc token
const { playbackUrl } = await fetch('/api/ivs-playback').then(r => r.json());
player.load(playbackUrl);
```

IAM backend: `ivs:GetStreamSession` hoặc pattern **participant token** tùy phiên bản API.

### Error Handling

| Lỗi | Xử lý UX |
|-----|----------|
| Stream offline | "Streamer chưa bắt đầu" + retry |
| Rebuffering | Spinner, không panic disconnect |
| 403 playback | Refresh auth token |
| Recording pending | "Replay sẽ có sau 5 phút" |

---

## 5. So Sánh Player Khác

### IVS Player vs Video.js + HLS.js

- **Video.js:** ecosystem plugin, không IVS-specific
- Dùng Video.js khi cần UI skin phức tạp — embed IVS tech qua SDK riêng cho video element

### IVS Player vs Native ExoPlayer HLS

- ExoPlayer play IVS URL được nhưng **metadata/low-latency** kém hơn **AmazonIVSPlayer**

### Recording + MediaPackage

Nếu cần **DRM VOD** sau live, copy S3 → MediaConvert → MediaPackage VOD endpoint — IVS recording alone không có Speke DRM.

---

## 6. Câu Hỏi Phỏng Vấn

**Q: Recording có ghi cả khi stream không healthy?**

> Chỉ ghi khi có **active stream session** hợp lệ. Gap ingest (starved) có thể tạo hole trong timeline — xử lý post-production hoặc slate.

**Q: Chi phí recording?**

> **S3 storage** + **PUT requests** + không tính riêng "recording fee" lớn ngoài IVS output (data đã phát live vẫn tính). Ước tính lifecycle S3 cho archive.

**Q: Có thể dùng CloudFront thay IVS CDN cho playback?**

> Live playback URL do **IVS CDN** cung cấp — không thay origin bằng CloudFront cho low-latency path chuẩn. **Recording** trên S3 thì **nên** CloudFront phía trước VOD replay.

**Q: Player SDK có bắt buộc cho REAL_TIME Stage không?**

> **Real-Time** dùng **Amazon IVS Broadcast/WebRTC SDK** khác family — không dùng cùng HLS Player cho subscribe Stage.

---

## 🔗 Tài Liệu Tham Khảo

- [Record to Amazon S3](https://docs.aws.amazon.com/ivs/latest/LowLatencyUserGuide/record-to-s3.html)
- [IVS Player SDK Web](https://docs.aws.amazon.com/ivs/latest/LowLatencyUserGuide/player.html)
- [IVS Player iOS](https://docs.aws.amazon.com/ivs/latest/LowLatencyUserGuide/player-ios.html)
- [IVS Player Android](https://docs.aws.amazon.com/ivs/latest/LowLatencyUserGuide/player-android.html)

---

**Cập Nhật Lần Cuối:** 2026-06-04
**Quay lại:** [README.md](./README.md) — tổng quan IVS
