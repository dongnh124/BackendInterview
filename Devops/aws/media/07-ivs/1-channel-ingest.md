# Amazon IVS — Channel, Ingest & Stream Key

> **Channel** (kênh) là tài nguyên IVS nhận luồng từ encoder và phát tới viewer. **Ingest** (nhập liệu) qua **RTMPS** — RTMP over TLS — Real-Time Messaging Protocol có mã hoá TLS; **Stream Key** (khóa luồng) xác thực nguồn phát.

## 📚 Mục Lục

1. [Channel Types](#1-channel-types)
2. [Ingest Endpoint & RTMPS](#2-ingest-endpoint--rtmps)
3. [Stream Key](#3-stream-key)
4. [Playback URL & Authorization](#4-playback-url--authorization)
5. [Thực Hành: OBS & AWS CLI](#5-thực-hành-obs--aws-cli)
6. [Câu Hỏi Phỏng Vấn](#6-câu-hỏi-phỏng-vấn)

---

## 1. Channel Types

IVS phân loại channel theo **độ phân giải tối đa**, **bitrate ladder**, và **giá**:

| Type | Max resolution | Đặc điểm | Use case |
|------|----------------|----------|----------|
| **BASIC** | 480p | Ladder đơn giản, chi phí thấp | Thử nghiệm, stream nhẹ |
| **STANDARD** | 1080p | Cân bằng chất lượng/chi phí | Livestream thương mại phổ biến |
| **ADVANCED-SD** | 480p | Ladder nâng cao, ốn định hơn Basic | Mobile-first, tiết kiệm bandwidth |
| **ADVANCED-HD** | 1080p | Ladder tối ưu chất lượng | Sự kiện HD, bán hàng cao cấp |

```
Lựa chọn channel type ảnh hưởng:
├── Số rendition ABR — Adaptive Bitrate (phát thích ứng tốc độ bit)
├── Max bitrate ingest chấp nhận
└── Đơn giá input/output trên AWS bill
```

> **Lưu ý:** Sau khi tạo channel, **không đổi được type** — phải tạo channel mới và chuyển ingest.

### Thuộc Tính Channel Quan Trọng

| Thuộc tính | Mô tả |
|------------|-------|
| `name` | Tên hiển thị (không phải ARN) |
| `type` | BASIC / STANDARD / ADVANCED-SD / ADVANCED-HD |
| `latencyMode` | NORMAL / LOW (REAL_TIME dùng Stage riêng) |
| `authorized` | `true` → playback cần token; `false` → public URL |
| `recordingConfigurationArn` | Gắn cấu hình ghi S3 |
| `insecureIngest` | Cho phép RTMP không TLS (không khuyến nghị production) |

---

## 2. Ingest Endpoint & RTMPS

### Luồng Ingest

```
[OBS / Wirecast / Hardware Encoder]
         │ RTMPS (port 443)
         ▼
┌─────────────────────────────┐
│  IVS Ingest Server          │
│  rtmps://xxx.global-contribute.live-video.net:443/app/
└─────────────────────────────┘
         │
         ▼
   Transcode + package ABR
         │
         ▼
   IVS CDN → Playback URL
```

### RTMPS vs RTMP

| Giao thức | Bảo mật | IVS |
|-----------|---------|-----|
| **RTMPS** | TLS mã hoá luồng | Mặc định, khuyến nghị |
| **RTMP** | Không mã hoá | Chỉ khi `insecureIngest: true` |

**RTMP** — Real-Time Messaging Protocol — giao thức push stream phổ biến; **RTMPS** thêm lớp TLS tương tự HTTPS.

### Giới Hạn Ingest Thường Gặp

- **Video:** H.264 (AVC), keyframe interval 1–2 giây khuyến nghị cho ABR ổn định
- **Audio:** AAC-LC stereo
- **Bitrate:** không vượt max của channel type (ví dụ Standard thường ≤ 8.5 Mbps ingest)
- **Resolution:** không upscale — IVS không làm miracle nếu OBS gửi 720p mà kỳ vọng 1080p sắc nét

---

## 3. Stream Key

### Vai Trò

**Stream Key** là secret gắn path ingest:

```
rtmps://a1b2c3d4e5f6.global-contribute.live-video.net:443/app/sk_us-east-1_xxxxxxxxxxxx
                                                              └── Stream Key
```

Ai có stream key có thể **push thay streamer** — coi như mật khẩu publish.

### Quản Lý Stream Key

```
Channel
├── Stream Key (default)     — key chính OBS dùng
├── Stream Key (optional #2) — backup encoder / failover
└── Rotate                   — invalidate key cũ, tạo key mới
```

**Thực hành production:**

1. Không commit stream key vào Git
2. Lưu trong **Secrets Manager** hoặc CI secret; OBS lấy qua script setup
3. Rotate ngay khi nghi ngờ lộ (nhân viên nghỉ, screenshot lộ key)
4. Dùng **separate channel** cho staging vs production

### Stream Health

IVS Console và API báo **Stream health**:

| Trạng thái | Ý nghĩa |
|------------|---------|
| **Stream healthy** | Ingest ổn định, viewer có thể xem |
| **Starved** | Bitrate ingest thấp / mất kết nối tạm thời |
| **Offline** | Không có active ingest |

Monitor qua **CloudWatch** metrics: `LiveInputTime`, `ConcurrentViews`, `IngestBitrate`.

---

## 4. Playback URL & Authorization

### Playback URL

Sau khi tạo channel, IVS cung cấp **Playback URL** (HLS):

```
https://xxx.playback.live-video.net/api/video/v1/us-east-1.123456789012.channel.xxx.m3u8
```

Viewer **không** dùng ingest URL — chỉ playback URL (hoặc URL qua Player SDK).

### Authorized Channel

Khi `authorized: true`:

```
App Backend (có IAM) ──▶ CreateParticipantToken / playback auth API
                              │
                              ▼
                    Player load URL + ?token=...
```

Bảo vệ nội dung không công khai mà không cần DRM — phù hợp webinar trả phí, nội bộ.

---

## 5. Thực Hành: OBS & AWS CLI

### Tạo Channel (CLI)

```bash
aws ivs create-channel \
  --name "live-shopping-demo" \
  --type STANDARD \
  --latency-mode LOW \
  --authorized false \
  --region ap-northeast-1
```

Response gồm `channel`, `streamKey`, `playbackUrl`.

### Cấu Hình OBS Studio

```
Settings → Stream
├── Service:    Custom
├── Server:     rtmps://xxxx.global-contribute.live-video.net:443/app/
└── Stream Key: sk_ap-northeast-1_xxxxxxxxxxxxxxxx

Settings → Output (khuyến nghị)
├── Video Encoder: x264 hoặc NVENC
├── Rate Control:  CBR
├── Bitrate:       4500 Kbps (1080p30) hoặc 2500 (720p)
├── Keyframe:      2 sec
├── Audio:         AAC 160 Kbps
└── Resolution:    1920x1080 hoặc 1280x720
```

### Tạo Stream Key Thứ Hai (Failover)

```bash
aws ivs create-stream-key \
  --channel-arn arn:aws:ivs:ap-northeast-1:123456789012:channel/AbCdEfGh \
  --region ap-northeast-1
```

Encoder backup dùng key mới; primary giữ key cũ — switch nhanh khi primary ingest fail.

### Kiểm Tra Stream Đang Live

```bash
aws ivs get-stream \
  --channel-arn arn:aws:ivs:ap-northeast-1:123456789012:channel/AbCdEfGh \
  --region ap-northeast-1
```

Trả về `streamId`, `health`, `viewerCount`, `startTime`.

---

## 6. Câu Hỏi Phỏng Vấn

**Q: Tại sao IVS dùng RTMPS mà không SRT?**

> **IVS Low-Latency** ingest chuẩn là **RTMPS** — tương thích OBS, Wirecast, phần lớn encoder phần cứng. **SRT** — Secure Reliable Transport thường dùng với **MediaLive/MediaConnect**, không phải ingest path chính của IVS channel. **Real-Time Stage** dùng **WebRTC**, không RTMP.

**Q: Một account có bao nhiêu channel?**

> Giới hạn theo **quota** region (mặc định vài chục channel, tăng qua AWS Support). Mỗi sự kiện/live session có thể 1 channel hoặc tái sử dụng channel cố định.

**Q: Làm sao bảo vệ ingest khỏi hijack?**

> Bảo mật **stream key**, rotate định kỳ, không dùng `insecureIngest`, giới hệ IAM ai được `ivs:CreateStreamKey`. Có thể kết hợp backend chỉ phát key cho streamer đã auth qua app.

---

**Cập Nhật Lần Cuối:** 2026-06-04
**Xem thêm:** [2-latency-modes.md](./2-latency-modes.md) — chọn NORMAL / LOW / REAL_TIME
