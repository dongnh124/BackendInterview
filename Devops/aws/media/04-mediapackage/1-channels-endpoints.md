# MediaPackage — Channel & Endpoint (Kênh Nhận & Điểm Phát)

> Channel là điểm nhận TS stream từ MediaLive; Endpoint là điểm phát nội dung đã đóng gói đến CDN và viewer. Hiểu rõ hai khái niệm này là nền tảng làm chủ MediaPackage.

## 📚 Mục Lục

1. [Channel — Kênh Nhận](#1-channel--kênh-nhận)
2. [Endpoint — Điểm Phát](#2-endpoint--điểm-phát)
3. [Cấu Hình CDN Authorization](#3-cấu-hình-cdn-authorization)
4. [Input Redundancy — Dự Phòng Đầu Vào](#4-input-redundancy--dự-phòng-đầu-vào)
5. [Thực Hành: Tạo Channel & Endpoint Qua CLI](#5-thực-hành-tạo-channel--endpoint-qua-cli)
6. [Câu Hỏi Phỏng Vấn](#6-câu-hỏi-phỏng-vấn)

---

## 1. Channel — Kênh Nhận

### 1.1 Channel Là Gì?

**Channel** trong MediaPackage là thực thể nhận TS — Transport Stream từ MediaLive (hoặc encoder khác). Channel **không** thực hiện encoding, transcoding hay packaging — nhiệm vụ duy nhất là nhận và lưu tạm stream vào rolling buffer.

```
MediaLive Pipeline A ──▶ Ingest URL 1 ┐
                                        ├──▶ MediaPackage Channel ──▶ Rolling Buffer
MediaLive Pipeline B ──▶ Ingest URL 2 ┘                              (TS segments)
```

### 1.2 Ingest Endpoints (Điểm Nhập)

Mỗi Channel có **2 Ingest Endpoints** (WebDAV URLs), tương ứng với 2 pipelines của MediaLive Standard Channel:

```
Channel: my-live-channel
├── Ingest Endpoint 1: https://xxxxx.mediapackage.us-east-1.amazonaws.com/in/v2/abc123/abc123/channel
│   Username: xxxxxxxx
│   Password: xxxxxxxx  (auto-generated, lưu trong Secrets Manager)
│
└── Ingest Endpoint 2: https://yyyyy.mediapackage.us-east-1.amazonaws.com/in/v2/def456/def456/channel
    Username: yyyyyyyy
    Password: yyyyyyyy
```

> **Quan trọng:** MediaLive Pipeline A gửi vào Endpoint 1, Pipeline B gửi vào Endpoint 2. MediaPackage tự động xử lý redundancy — nếu một nguồn mất tín hiệu, tự chuyển sang nguồn kia.

### 1.3 Ingest Protocol (Giao Thức Nhận)

**WebDAV** — Web-based Distributed Authoring and Versioning là giao thức mặc định của MediaPackage v1:

```
WebDAV PUT request:
PUT /in/v2/{channelId}/{channelId}/channel
Content-Type: video/MP2T          ← MPEG-2 Transport Stream
Authorization: Basic base64(user:pass)

Body: raw TS stream (binary)
```

MediaPackage v2 dùng **HTTP PUT** thuần, đơn giản hơn WebDAV.

### 1.4 Rolling Buffer (Bộ Đệm Cuộn)

Channel lưu trữ TS segments trong **rolling buffer** — bộ nhớ trượt theo thời gian:

```
Timeline:  [12:00]──[12:01]──[12:02]──[12:03]──[12:04]──[12:05] (LIVE)
                                │
                         Window: 7 ngày
                         (segments cũ hơn 7 ngày bị xoá)

Viewer request live  → nhận segments mới nhất
Viewer request 12:01 → nhận segments tại 12:01 (nếu trong window)
```

**Cấu hình Rolling Buffer:**
- **Startover window** (cửa sổ xem lại): 0 đến 1,209,600 giây (14 ngày)
- Mặc định: 0 (không lưu, chỉ live)
- Khuyến nghị cho catch-up TV: 86,400 giây (24 giờ) đến 604,800 giây (7 ngày)

---

## 2. Endpoint — Điểm Phát

### 2.1 Endpoint Là Gì?

**Endpoint** là URL CloudFront (hoặc viewer) gọi để lấy manifest và segments. Mỗi Endpoint ánh xạ một định dạng output và có cấu hình riêng biệt.

```
Channel: my-live-channel
├── Endpoint: hls-main
│   URL: https://xxxxx.mediapackage.us-east-1.amazonaws.com/out/v1/abc/index.m3u8
│   Format: HLS
│
├── Endpoint: dash-main
│   URL: https://xxxxx.mediapackage.us-east-1.amazonaws.com/out/v1/def/index.mpd
│   Format: DASH-ISO
│
└── Endpoint: cmaf-ll
    URL: https://xxxxx.mediapackage.us-east-1.amazonaws.com/out/v1/ghi/index.m3u8
    Format: CMAF (CMAF — Common Media Application Format)
```

### 2.2 Các Loại Endpoint

#### HLS Endpoint — HTTP Live Streaming

```
Cấu hình điển hình:
┌─────────────────────────────────────────────┐
│ HLS Packaging Configuration                  │
│                                              │
│ Segment duration:    6 giây                  │
│ Playlist window:     60 giây (10 segments)   │
│ Use audio rendition: true                    │
│ Include Iframe-only: false                   │
│ Ad markers:          DATERANGE (từ SCTE-35)  │
│ Manifest name:       index                   │
│ Program date time:   INCLUDE                 │
└─────────────────────────────────────────────┘

Output manifest: index.m3u8
#EXTM3U
#EXT-X-VERSION:3
#EXT-X-STREAM-INF:BANDWIDTH=5000000,RESOLUTION=1920x1080
1080p/index.m3u8
#EXT-X-STREAM-INF:BANDWIDTH=3000000,RESOLUTION=1280x720
720p/index.m3u8
#EXT-X-STREAM-INF:BANDWIDTH=800000,RESOLUTION=640x360
360p/index.m3u8
```

#### DASH Endpoint — Dynamic Adaptive Streaming over HTTP

```
Cấu hình điển hình:
┌─────────────────────────────────────────────┐
│ DASH Packaging Configuration                 │
│                                              │
│ Segment duration:        2 giây              │
│ Manifest window duration: 60 giây            │
│ Min buffer time:          2 giây             │
│ Profile:                  NONE / HBBTV_1_5   │
│ Segment template format:  NUMBER_WITH_TIMELINE│
└─────────────────────────────────────────────┘

Output manifest: index.mpd (MPD — Media Presentation Description)
```

#### CMAF Endpoint — Common Media Application Format

CMAF kết hợp ưu điểm HLS và DASH: dùng fragmented MP4 (fMP4) thay vì TS, tương thích cả iOS (HLS) và Android (DASH):

```
CMAF với HLS wrapper:
  index.m3u8  ← HLS manifest nhưng segment là .m4s (fMP4)
  1080p/segment-001.m4s
  1080p/segment-002.m4s

CMAF với DASH wrapper:
  index.mpd   ← DASH manifest nhưng segment là .m4s (fMP4)
  1080p/segment-001.m4s   ← cùng file với HLS!
```

> **Lợi ích CMAF:** Một bộ segments phục vụ cả iOS (qua HLS) và Android (qua DASH), giảm chi phí storage và bandwidth.

### 2.3 Stream Selection — Lọc Luồng

Endpoint có thể **lọc bớt** các renditions từ channel để phù hợp use case:

```
Channel có bitrate ladder: 1080p, 720p, 480p, 360p, 270p

Endpoint "mobile":
  Max video bitrate: 1,500,000 bps
  → Chỉ phát: 480p, 360p, 270p (bỏ 1080p và 720p)

Endpoint "fullhd":
  Min video bitrate: 2,000,000 bps
  → Chỉ phát: 1080p, 720p
```

### 2.4 Encryption (Mã Hoá) Trên Endpoint

Cấu hình DRM trực tiếp trên từng endpoint:

```
HLS Endpoint Encryption:
├── Method: AES-128 (cơ bản, dễ cấu hình)
│   hoặc SAMPLE-AES (cho FairPlay)
│
├── SPEKE configuration:
│   ├── Resource ID: unique ID cho content này
│   ├── System IDs: [FairPlay SystemID]
│   ├── URL: SPEKE Key Provider URL
│   └── Role ARN: IAM role để gọi SPEKE
│
└── Key rotation: mỗi 86,400 giây (24 giờ)

DASH Endpoint Encryption:
└── SPEKE với Widevine + PlayReady System IDs
```

---

## 3. Cấu Hình CDN Authorization

### Tại Sao Cần CDN Authorization?

Không có CDN Authorization, ai cũng có thể gọi thẳng vào MediaPackage endpoint bỏ qua CloudFront, dẫn đến:
- **Tăng chi phí** MediaPackage (tính phí theo GB output)
- **Bypass caching** của CloudFront → tăng tải MediaPackage
- **Bypass security** (Signed URL/Cookies của CloudFront)

### Cách Hoạt Động

```
Luồng hợp lệ:
Viewer ──▶ CloudFront (thêm header bí mật) ──▶ MediaPackage
                                                 │
                                                 ├── Kiểm tra header ✓
                                                 └── Trả content

Luồng bị chặn:
Viewer ──▶ MediaPackage trực tiếp (không có header)
               │
               └── Từ chối 403 Forbidden
```

### Cấu Hình CDN Authorization

**Bước 1:** Tạo secret trong AWS Secrets Manager:
```json
{
  "MediaPackageCDNIdentifier": "my-secret-value-12345"
}
```

**Bước 2:** Cấu hình Endpoint với CDN Authorization:
```json
{
  "Authorization": {
    "CdnIdentifierSecret": "arn:aws:secretsmanager:us-east-1:123456789:secret:MediaPackageCDN",
    "SecretsRoleArn": "arn:aws:iam::123456789:role/MediaPackageSecretsRole"
  }
}
```

**Bước 3:** Cấu hình CloudFront Origin Custom Headers:
```
Header Name:  X-MediaPackage-CDNIdentifier
Header Value: my-secret-value-12345
```

---

## 4. Input Redundancy — Dự Phòng Đầu Vào

### Cơ Chế Tự Động Chuyển Đổi

MediaPackage tự động chọn input "khoẻ mạnh" nhất trong số các ingest endpoint:

```
Trạng thái bình thường:
MediaLive A ──▶ Endpoint 1 (active)   ┐
MediaLive B ──▶ Endpoint 2 (standby)  ├──▶ MediaPackage chọn Endpoint 1
                                       ┘

Khi Pipeline A lỗi:
MediaLive A ──✕ Endpoint 1 (mất tín hiệu) ┐
MediaLive B ──▶ Endpoint 2 (active)        ├──▶ MediaPackage chuyển sang Endpoint 2
                                            ┘   (< 10 giây gián đoạn)
```

### Điều Kiện Chuyển Đổi Input

MediaPackage chuyển input khi:
1. Endpoint không nhận được data trong **> 10 giây**
2. Encoder báo lỗi qua header HTTP

### Input Loss Behavior (Hành Vi Khi Mất Đầu Vào)

Khi **cả hai** input mất tín hiệu, MediaPackage:
- Tiếp tục phát manifest (không xoá)
- Viewer nhận lỗi 404 khi request segment mới
- Player thường hiển thị "buffering" hoặc error

---

## 5. Thực Hành: Tạo Channel & Endpoint Qua CLI

### Tạo Channel

```bash
aws mediapackage create-channel \
  --id "my-live-channel" \
  --description "Live channel cho sự kiện thể thao" \
  --region us-east-1
```

**Response trả về 2 ingest URLs:**
```json
{
  "Arn": "arn:aws:mediapackage:us-east-1:123456789:channels/my-live-channel",
  "Id": "my-live-channel",
  "HlsIngest": {
    "IngestEndpoints": [
      {
        "Id": "abc123",
        "Password": "generatedpassword1",
        "Url": "https://abc123.mediapackage.us-east-1.amazonaws.com/in/v2/my-live-channel/abc123/channel",
        "Username": "user1"
      },
      {
        "Id": "def456",
        "Password": "generatedpassword2",
        "Url": "https://def456.mediapackage.us-east-1.amazonaws.com/in/v2/my-live-channel/def456/channel",
        "Username": "user2"
      }
    ]
  }
}
```

### Tạo HLS Endpoint

```bash
aws mediapackage create-origin-endpoint \
  --channel-id "my-live-channel" \
  --id "hls-main-endpoint" \
  --hls-package '{
    "SegmentDurationSeconds": 6,
    "PlaylistWindowSeconds": 60,
    "AdMarkers": "DATERANGE",
    "IncludeIframeOnlyStream": false,
    "ProgramDateTimeIntervalSeconds": 60
  }' \
  --startover-window-seconds 86400 \
  --time-delay-seconds 0 \
  --region us-east-1
```

### Tạo DASH Endpoint

```bash
aws mediapackage create-origin-endpoint \
  --channel-id "my-live-channel" \
  --id "dash-main-endpoint" \
  --dash-package '{
    "SegmentDurationSeconds": 2,
    "ManifestWindowSeconds": 60,
    "MinBufferTimeSeconds": 2,
    "MinUpdatePeriodSeconds": 2,
    "Profile": "NONE"
  }' \
  --startover-window-seconds 86400 \
  --region us-east-1
```

### Liệt Kê Endpoints Của Channel

```bash
aws mediapackage list-origin-endpoints \
  --channel-id "my-live-channel" \
  --region us-east-1
```

---

## 6. Câu Hỏi Phỏng Vấn

**Q: Tại sao mỗi MediaPackage Channel có 2 Ingest Endpoints?**

> Để nhận stream từ **cả hai pipelines** của MediaLive Standard Channel (Pipeline A và B). Cả hai pipeline encode cùng nội dung và gửi vào 2 endpoint khác nhau. MediaPackage tự động chọn endpoint nào ổn định, đảm bảo stream không bị gián đoạn khi một pipeline lỗi.

**Q: Khác nhau giữa HLS và CMAF endpoint trong MediaPackage?**

> - **HLS endpoint**: segments là MPEG-2 TS (`.ts`), chỉ dùng cho iOS/Safari truyền thống
> - **CMAF endpoint**: segments là fragmented MP4 (`.m4s`), tương thích cả iOS (qua HLS manifest) và Android/Chrome (qua DASH manifest). CMAF chỉ cần lưu một bộ segments cho cả hai platform, tiết kiệm hơn.

**Q: CDN Authorization trong MediaPackage hoạt động thế nào?**

> MediaPackage kiểm tra header `X-MediaPackage-CDNIdentifier` trong mỗi request. CloudFront được cấu hình để tự động thêm header bí mật này vào mọi request gửi về MediaPackage Origin. Request không có header hợp lệ (ví dụ: gọi trực tiếp) sẽ nhận 403 Forbidden. Secret được lưu trong AWS Secrets Manager và rotate định kỳ.

---

**Phần Tiếp Theo:** [2-just-in-time-packaging.md](./2-just-in-time-packaging.md) — JIT Packaging Chi Tiết
