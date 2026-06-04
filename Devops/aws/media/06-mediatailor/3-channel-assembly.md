# MediaTailor — Channel Assembly (Lắp Ghép Kênh Linear)

> **Channel Assembly** biến thư viện VOD — Video on Demand trên S3 thành kênh phát **linear** (liên tục như truyền hình). Viewer thấy một luồng live 24/7; backend xếp clip theo lịch EPG — Electronic Program Guide.

## 📚 Mục Lục

1. [Channel Assembly Là Gì?](#1-channel-assembly-là-gì)
2. [Kiến Trúc & Thành Phần](#2-kiến-trúc--thành-phần)
3. [Source Location & Program](#3-source-location--program)
4. [Lịch Phát & Slate](#4-lịch-phát--slate)
5. [Kết Hợp SSAI Trên FAST](#5-kết-hợp-ssai-trên-fast)
6. [Thực Hành CLI](#6-thực-hành-cli)
7. [Câu Hỏi Phỏng Vấn](#7-câu-hỏi-phỏng-vấn)

---

## 1. Channel Assembly Là Gì?

### Use Case: FAST — Free Ad-Supported Streaming TV

```
Truyền hình truyền thống:  Lịch cố định → phát sóng tuần tự
FAST (OTT):                VOD clips trên S3 → MediaTailor xếp lịch → HLS linear URL
```

**Lợi ích:**

- Không cần MediaLive 24/7 cho kênh VOD-only
- Thay đổi lịch chương trình bằng API (không restart encoder)
- Mỗi "kênh" = brand riêng (thể thao, phim, tin tức)
- Tích hợp quảng cáo SSAI giữa chương trình

### So Sánh Với Live Pipeline

| Khía cạnh | Live (MediaLive) | Channel Assembly |
|-----------|------------------|------------------|
| Nguồn | RTMP/RTP realtime | File MP4/HLS trên S3 |
| Encoder | MediaLive bắt buộc | Không cần |
| Lịch phát | Schedule Actions | Program / Schedule API |
| Độ trễ | Vài giây (live) | Vài giây (pseudo-live) |
| Chi phí 24/7 | Cao (MediaLive always-on) | Thấp hơn (chỉ Assembly minutes) |

---

## 2. Kiến Trúc & Thành Phần

```
┌─────────────────────────────────────────────────────────────────┐
│              MEDIATAILOR CHANNEL ASSEMBLY                       │
│                                                                 │
│  Source Location (S3)                                           │
│  s3://fast-content/movies/*.mp4                                 │
│         │                                                       │
│         ▼                                                       │
│  ┌─────────────┐    ┌──────────────┐    ┌──────────────────┐ │
│  │   Channel   │───▶│   Program    │───▶│  Schedule / EPG  │ │
│  │  (kênh FAST)│    │  (metadata)  │    │  (thời gian phát)│ │
│  └─────────────┘    └──────────────┘    └──────────────────┘ │
│         │                                                       │
│         ▼                                                       │
│  Manifest generator (HLS master + media playlists rolling)      │
│         │                                                       │
│         ▼                                                       │
│  Output URL → CloudFront → Player (xem như "live")               │
└─────────────────────────────────────────────────────────────────┘
```

### Tài Nguyên Chính

| Tài nguyên | Vai trò |
|------------|---------|
| **Source Location** | Chỉ định S3 bucket/prefix chứa VOD assets |
| **Channel** | Kênh linear output (tên, logo metadata tùy chọn) |
| **Program** | Một title (phim, episode) gắn asset URI + duration |
| **Schedule Entry** | Thời điểm program phát trên timeline kênh |
| **Slate** | Nội dung filler khi gap hoặc giữa chương trình |

---

## 3. Source Location & Program

### 3.1 Source Location (Vị Trí Nguồn)

```json
{
  "SourceLocationName": "fast-vod-library",
  "HttpConfiguration": {
    "BaseUrl": "https://fast-vod-library.s3.ap-southeast-1.amazonaws.com/"
  }
}
```

MediaTailor đọc asset qua HTTP từ S3 (bucket policy cho phép MediaTailor service principal).

**Yêu cầu asset:**

- Container: MP4 (H.264 + AAC phổ biến) hoặc HLS VOD
- Consistent frame rate, keyframe interval đều (giảm glitch khi nối)
- Audio loudness tương đối đồng nhất giữa programs

### 3.2 Program (Chương Trình)

```json
{
  "ProgramName": "movie-action-2024",
  "ChannelName": "fast-action-channel",
  "ScheduleConfiguration": {
    "Transition": {
      "DurationMillis": 0,
      "ClipLimits": {
        "MinimumDurationMillis": 60000
      }
    }
  },
  "SourceLocationName": "fast-vod-library",
  "VodSourceName": "movie-action-2024-mp4"
}
```

**VodSource** — liên kết file cụ thể trong Source Location:

```
s3://fast-vod-library/movies/action-2024.mp4
  → VodSourceName: movie-action-2024-mp4
  → Program: movie-action-2024
```

### 3.3 Transition Giữa Programs

| Transition | Mô tả |
|------------|-------|
| `DurationMillis: 0` | Cắt cứng (hard cut) — phổ biến FAST |
| Fade / bumper | Cần asset bumper riêng làm program bridge |

---

## 4. Lịch Phát & Slate

### 4.1 Schedule Entry (Mục Lịch)

```json
{
  "ScheduleEntry": {
    "ProgramName": "movie-action-2024",
    "ScheduleEntryType": "PROGRAM",
    "ApproximateStartTime": "2026-06-04T14:00:00Z",
    "ApproximateDurationMinutes": 120
  }
}
```

Sau khi program kết thúc, timeline chuyển program tiếp theo trong schedule hoặc **slate**.

### 4.2 Slate (Nội Dung Filler)

**Slate** lấp khoảng trống:

- Giữa hai program khi schedule có gap
- Khi program ngắn hơn slot
- "Please stand by" — broadcast style

```json
{
  "SlateSourceName": "channel-slate-10min-loop",
  "VodSourceName": "slate-loop.mp4"
}
```

### 4.3 Pseudo-Live Timeline

```
Wall clock 14:00 ──▶ Program A (phút 0-90 của file, nếu đã phát từ 12:30)
Wall clock 15:30 ──▶ Program B
Wall clock 17:00 ──▶ Slate loop
```

Viewer vào giữa chừng **không** rewind được (trừ khi player hỗ trợ time-shift riêng) — giống TV broadcast.

---

## 5. Kết Hợp SSAI Trên FAST

### Kiến Trúc FAST Đầy Đủ

```
S3 VOD library
      │
      ▼
Channel Assembly ──▶ HLS linear manifest (content)
      │
      ▼
Playback Configuration (SSAI) ──▶ ADS VAST
      │
      ▼
CloudFront ──▶ Viewer + ads
```

### Ad Break Trên FAST

Hai cách đánh dấu avail:

1. **Scheduled avails** trong manifest Assembly (ad break mỗi 12 phút nội dung)
2. **VMAP** từ ADS khi dùng VOD-style SSAI overlay

```
Chương trình 45 phút:
  00:00  Pre-roll (SSAI)
  12:00  Mid-roll 1
  24:00  Mid-roll 2
  36:00  Mid-roll 3
  45:00  Post-roll (nếu có program tiếp)
```

### Monetization Stack

| Layer | Công nghệ |
|-------|-----------|
| Content | S3 + Channel Assembly |
| Ads | MediaTailor SSAI + ADS |
| Analytics | Beacons + ADS reporting |
| CDN | CloudFront |

---

## 6. Thực Hành CLI

### Tạo Source Location

```bash
aws mediatailor create-source-location \
  --source-location-name fast-vod-library \
  --http-configuration BaseUrl=https://fast-vod-library.s3.ap-southeast-1.amazonaws.com/ \
  --region ap-southeast-1
```

### Tạo Channel

```bash
aws mediatailor create-channel \
  --channel-name fast-action-channel \
  --source-location-name fast-vod-library \
  --region ap-southeast-1
```

### Tạo Vod Source + Program

```bash
aws mediatailor create-vod-source \
  --source-location-name fast-vod-library \
  --vod-source-name movie-action-2024-mp4 \
  --http-package-configuration Path=/movies/action-2024.mp4,Type=GROUP \
  --region ap-southeast-1

aws mediatailor create-program \
  --channel-name fast-action-channel \
  --program-name movie-action-2024 \
  --source-location-name fast-vod-library \
  --schedule-configuration Transition={DurationMillis=0} \
  --vod-source-name movie-action-2024-mp4 \
  --region ap-southeast-1
```

### Lấy Playback URL Kênh

```bash
aws mediatailor describe-channel \
  --channel-name fast-action-channel \
  --region ap-southeast-1 \
  --query "Channel.{PlaybackUrl:PlaybackUrl,State:State}"
```

### Checklist Vận Hành FAST

- [ ] Asset pre-QC (duration, black frames, audio level)
- [ ] Schedule API automation (CMS → MediaTailor)
- [ ] SSAI fill rate monitoring
- [ ] CloudFront cache cho linear manifest (TTL ngắn)
- [ ] DRM (nếu có) — Assembly output có thể qua MediaPackage DRM riêng

---

## 7. Câu Hỏi Phỏng Vấn

**Q: Channel Assembly khác Playback Configuration thế nào?**

> **Channel Assembly** **tạo** luồng linear từ VOD. **Playback Configuration** **chèn ads** vào manifest có sẵn (từ Assembly, MediaPackage, hoặc CloudFront). FAST thường dùng **cả hai**: Assembly tạo content timeline, Playback Config bọc SSAI bên ngoài.

**Q: Tại sao FAST không dùng MediaLive 24/7?**

> MediaLive tính phí theo giờ channel **always-on**. FAST chủ yếu lặp file VOD — Channel Assembly chỉ tính **assembly minutes** phát thực tế, không cần encoder realtime. Tiết kiệm đáng kể cho 10+ kênh VOD.

**Q: Viewer vào giữa phim — thấy gì?**

> **Pseudo-live**: Assembly tính offset trong program dựa trên wall-clock schedule — viewer vào giữa phim sẽ thấy **giữa phim**, không từ đầu (giống bật TV giữa chương trình). Muốn on-demand từ đầu → dùng VOD app riêng, không phải FAST channel.

---

**Cập Nhật Lần Cuối:** 2026-06-04
**Phần Trước:** [2-ads-integration.md](./2-ads-integration.md)
**Phần Tiếp Theo:** [4-reporting-beacons.md](./4-reporting-beacons.md)
