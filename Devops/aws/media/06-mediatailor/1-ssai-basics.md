# MediaTailor — SSAI Cơ Bản & Playback Configuration

> SSAI — Server-Side Ad Insertion — Chèn Quảng Cáo Phía Máy Chủ là mô hình ghép quảng cáo vào manifest trước khi player nhận. **Playback Configuration** là tài nguyên AWS định nghĩa cách MediaTailor kết nối content origin, ADS, và CDN.

## 📚 Mục Lục

1. [SSAI vs CSAI](#1-ssai-vs-csai)
2. [Playback Configuration](#2-playback-configuration)
3. [Session Initialization](#3-session-initialization)
4. [Avail & Ad Markers](#4-avail--ad-markers)
5. [Manifest Stitching](#5-manifest-stitching)
6. [Thực Hành: Tạo Playback Configuration](#6-thực-hành-tạo-playback-configuration)
7. [Câu Hỏi Phỏng Vấn](#7-câu-hỏi-phỏng-vấn)

---

## 1. SSAI vs CSAI

### CSAI — Client-Side Ad Insertion

```
Player timeline:
[Content] ──▶ Player SDK gọi IMA/Google Ads ──▶ [Ad request riêng] ──▶ [Ad play] ──▶ [Content]

Đặc điểm:
- Request quảng cáo từ domain khác (dễ bị ad-blocker chặn)
- Cần SDK riêng trên từng platform (web: IMA SDK, iOS: IMA, Android: IMA)
- Player phải xử lý chuyển cảnh content ↔ ad
```

### SSAI — Server-Side Ad Insertion

```
Player timeline:
[Content + Ad trong cùng manifest HLS] ──▶ Player chỉ request một luồng

Đặc điểm:
- Ad segments cùng origin/CDN với content
- Player HLS/DASH chuẩn (AVPlayer, ExoPlayer, HLS.js) không cần ad SDK
- MediaTailor cá nhân hoá manifest theo session
```

| Khía cạnh | SSAI | CSAI |
|-----------|------|------|
| Ad-blocker resistance | Cao | Thấp |
| Player đơn giản | Có | Không (cần ad SDK) |
| Broadcast SCTE-35 | Tự nhiên | Khó đồng bộ |
| Độ trễ ad start | Phụ thuộc stitch + prefetch | Thường thấp hơn nếu SDK cache sẵn |
| Vận hành | MediaTailor + ADS + markers | Player team + ad SDK |

---

## 2. Playback Configuration

### 2.1 Các Trường Quan Trọng

**Playback Configuration** (`aws mediatailor create-playback-configuration`) gồm:

| Trường | Mô tả |
|--------|-------|
| `Name` | Tên config (unique trong account/region) |
| `VideoContentSourceUrl` | URL manifest gốc (MediaPackage endpoint hoặc CloudFront) |
| `AdDecisionServerUrl` | URL ADS — có placeholder `[session]` hoặc query params |
| `ContentSegmentUrlPrefix` | Prefix cho segment nội dung (nếu khác manifest host) |
| `AdSegmentUrlPrefix` | Prefix CDN cho ad segments sau transcode |
| `CdnConfiguration` | `AdSegmentUrlPrefix`, `ContentSegmentUrlPrefix` |
| `AvailSuppression` | Bỏ qua avail trong khoảng thời gian (ví dụ: pre-roll đầu chương trình) |
| `SlateAdUrl` | Video filler khi không có ad |
| `TranscodeProfile` | Profile transcode ad creative không khớp ladder |
| `ManifestProcessingRules` | `AdMarkerPassthrough`, `ManifestHlsOnDemand` |

### 2.2 Ví Dụ Cấu Hình JSON

```json
{
  "Name": "ott-live-hls-main",
  "VideoContentSourceUrl": "https://d111111abcdef8.cloudfront.net/out/v1/abc123/ott-live/index.m3u8",
  "AdDecisionServerUrl": "https://ads.partner.com/vast?session=[session]&app=[player_params.app]",
  "CdnConfiguration": {
    "AdSegmentUrlPrefix": "https://d222222abcdef8.cloudfront.net/",
    "ContentSegmentUrlPrefix": "https://d111111abcdef8.cloudfront.net/"
  },
  "SlateAdUrl": "https://cdn.example.com/slate/30s-black.mp4",
  "ManifestProcessingRules": {
    "AdMarkerPassthrough": {
      "Enabled": true
    }
  },
  "Tags": {
    "Environment": "production",
    "Channel": "sports-1"
  }
}
```

### 2.3 Ad Marker Passthrough

Khi `AdMarkerPassthrough.Enabled = true`, MediaTailor **giữ nguyên** các tag `#EXT-X-CUE-OUT`, `#EXT-X-DATERANGE` trong manifest output — hữu ích cho:

- Player analytics đọc cue points
- Downstream system debug ad timing
- Compliance logging

Khi `false`, manifest output chỉ chứa stitched segments, không lộ marker gốc.

---

## 3. Session Initialization

### 3.1 Tại Sao Cần Session?

Mỗi viewer cần **session riêng** để:

- ADS chọn quảng cáo theo `user_id`, geo, device, content metadata
- MediaTailor track beacons (impression, quartile) đúng session
- Tránh cache manifest cá nhân hoá chéo viewer (CDN phải cache theo session ID)

### 3.2 Luồng Khởi Tạo

```
┌──────────┐    1. POST session init     ┌──────────────┐
│  App /   │ ──────────────────────────▶ │ MediaTailor  │
│  Player  │    playerParams, origin      │  API         │
└──────────┘                             └──────┬───────┘
       ▲                                        │
       │    2. manifestUrl + sessionId          │
       └────────────────────────────────────────┘

3. Player load manifestUrl (qua CloudFront)
4. Mọi segment request gắn aws.sessionId=...
```

### 3.3 Ví Dụ Session Init (HTTP)

```http
POST /v1/session/{account-id}/ott-live-hls-main HTTP/1.1
Host: api.mediatailor.ap-southeast-1.amazonaws.com
Content-Type: application/json

{
  "origin": "https://ott.example.com",
  "manifestLayout": "MULTI_PERIOD",
  "playerParams": {
    "app_bundle": "com.example.ott",
    "device_type": "smarttv",
    "user_id": "user-abc-123",
    "content_id": "match-final-2026"
  }
}
```

Response (rút gọn):

```json
{
  "manifestUrl": "https://d222.cloudfront.net/v1/master/.../index.m3u8?aws.sessionId=abc-def-123",
  "sessionId": "abc-def-123",
  "dashManifestUrl": "https://d222.cloudfront.net/.../index.mpd?aws.sessionId=abc-def-123"
}
```

> **Lưu ý CDN:** CloudFront cache key phải bao gồm `aws.sessionId` (query string forward) — nếu không, viewer A có thể nhận manifest của viewer B.

---

## 4. Avail & Ad Markers

### 4.1 Avail Là Gì?

**Avail** — khoảng thời gian trong timeline stream được đánh dấu để chèn quảng cáo. Độ dài avail thường do SCTE-35 `break_duration` hoặc `#EXT-X-CUE-OUT` quy định.

### 4.2 HLS Ad Markers Từ MediaPackage

MediaPackage chuyển SCTE-35 thành HLS tags:

```m3u8
#EXTM3U
#EXT-X-VERSION:6
...
#EXT-X-CUE-OUT:60
#EXT-X-CUE-OUT-CONT:CAI=0,ElapsedTime=0,Duration=60
...
#EXTINF:6.000,
segment_001.ts
...
#EXT-X-CUE-IN
```

MediaTailor đọc `CUE-OUT` → biết avail 60 giây → gọi ADS lấy creative tổng ~60 giây.

### 4.3 SCTE-35 DATERANGE (Chi Tiết Hơn)

```m3u8
#EXT-X-DATERANGE:ID="splice-1001",START-DATE="2026-06-04T12:00:00.000Z",
  SCTE35-OUT=0xFC003C0000000000000000000000000000000000000000000000000000000000,
  DURATION=60.0,PLANNED-DURATION=60.0
```

Phù hợp broadcast chuẩn; MediaTailor hỗ trợ cả CUE tags và DATERANGE tùy cấu hình origin.

### 4.4 Avail Suppression (Ẩn Avail)

Trong Playback Configuration, **AvailSuppression** bỏ qua avail ở:

- Đầu chương trình (tránh double pre-roll)
- Khoảng thời gian blackout (tin tức khẩn)
- Nội dung trẻ em / premium tier không có ads

```json
"AvailSuppression": {
  "Mode": "BEHIND_LIVE_EDGE",
  "Value": "00:00:30"
}
```

---

## 5. Manifest Stitching

### 5.1 Quy Trình Stitch

```
1. Fetch master manifest từ VideoContentSourceUrl
2. Fetch variant manifests (1080p, 720p, ...)
3. Tại mỗi avail:
   a. Gọi ADS → VAST MediaFile URLs
   b. Transcode ad nếu cần (bitrate/resolution không khớp)
   c. Chèn #EXTINF segments của ad vào media playlist
4. Rewrite segment URLs với AdSegmentUrlPrefix / ContentSegmentUrlPrefix
5. Trả manifest đã stitch cho player
```

### 5.2 Multi-Bitrate Ad Stitching

Player ABR — Adaptive Bitrate cần ad ở **mọi rendition** (1080p, 720p, 360p):

```
Variant 1080p playlist:  [content segs] + [ad segs 1080p] + [content segs]
Variant 720p playlist:   [content segs] + [ad segs 720p]  + [content segs]
```

MediaTailor transcode mỗi ad creative theo **TranscodeProfile** hoặc ladder có sẵn từ ADS (nếu ADS trả multi-bitrate VAST).

### 5.3 MULTI_PERIOD vs SINGLE_PERIOD (DASH)

| Layout | Mô tả |
|--------|-------|
| `SINGLE_PERIOD` | Một Period, đơn giản, tương thích player cũ |
| `MULTI_PERIOD` | Period riêng cho content và ad — chuẩn DASH SSAI hiện đại |

Session init chọn `manifestLayout` phù hợp player target (ExoPlayer khuyến nghị MULTI_PERIOD).

---

## 6. Thực Hành: Tạo Playback Configuration

### CLI — Tạo Config

```bash
aws mediatailor put-playback-configuration \
  --name ott-live-hls-main \
  --region ap-southeast-1 \
  --video-content-source-url "https://d111.cloudfront.net/out/v1/abc/ott/index.m3u8" \
  --ad-decision-server-url "https://ads.example.com/vast?session=[session]" \
  --cdn-configuration AdSegmentUrlPrefix=https://d222.cloudfront.net/,ContentSegmentUrlPrefix=https://d111.cloudfront.net/ \
  --slate-ad-url "https://cdn.example.com/slate/30s.mp4"
```

### CLI — Liệt Kê & Mô Tả

```bash
aws mediatailor list-playback-configurations --region ap-southeast-1

aws mediatailor get-playback-configuration \
  --name ott-live-hls-main \
  --region ap-southeast-1
```

### Kiểm Tra Stitch (Staging)

```bash
# Khởi tạo session test
curl -X POST "https://api.mediatailor.ap-southeast-1.amazonaws.com/v1/session/123456789012/ott-live-hls-main" \
  -H "Content-Type: application/json" \
  -d '{"playerParams":{"device":"test"}}'

# Tải manifest, tìm ad segments
curl -s "MANIFEST_URL_FROM_RESPONSE" | grep -E "EXT-X-CUE|EXTINF|amazonaws"
```

### Checklist Production

- [ ] CloudFront forward query string `aws.sessionId`
- [ ] MediaPackage endpoint có ad markers (SCTE-35 enabled)
- [ ] ADS timeout < stitch deadline
- [ ] Slate URL hoạt động (failover)
- [ ] Transcode profile khớp content ladder
- [ ] Không cache manifest cá nhân hoá cross-user

---

## 7. Câu Hỏi Phỏng Vấn

**Q: Playback Configuration khác gì Channel trong MediaPackage?**

> **MediaPackage Channel** nhận TS và tạo HLS origin với ad markers. **MediaTailor Playback Configuration** không nhận TS — chỉ đọc manifest URL đã có markers và **thay nội dung avail** bằng ads. Hai dịch vụ bổ sung nhau: Package = origin + markers; Tailor = personalization + stitch.

**Q: Tại sao phải Session Initialization thay vì dùng URL cố định?**

> Manifest sau stitch **khác nhau theo viewer** (ads khác nhau). Session ID đảm bảo ADS nhận đúng context và CDN không trả manifest của người khác. URL cố định chỉ phù hợp demo không có cá nhân hoá.

**Q: Slate ad dùng khi nào?**

> Khi ADS trả VAST rỗng, timeout, creative lỗi transcode, hoặc fill rate thấp. Slate giữ timeline liên tục, tránh "đen màn hình" trong ad break — quan trọng cho broadcast SLA.

---

**Cập Nhật Lần Cuối:** 2026-06-04
**Phần Trước:** [README.md](./README.md)
**Phần Tiếp Theo:** [2-ads-integration.md](./2-ads-integration.md)
