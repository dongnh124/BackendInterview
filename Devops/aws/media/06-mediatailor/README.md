# AWS Elemental MediaTailor — Chèn Quảng Cáo & Lắp Ghép Kênh (SSAI & Channel Assembly)

> AWS Elemental MediaTailor là dịch vụ cá nhân hoá quảng cáo và lắp ghép kênh video được quản lý hoàn toàn bởi AWS. Dịch vụ thực hiện SSAI — Server-Side Ad Insertion — Chèn Quảng Cáo Phía Máy Chủ (ghép quảng cáo vào luồng trước khi tới viewer), tích hợp ADS — Ad Decision Server — Máy Chủ Ra Quyết Định Quảng Cáo, và Channel Assembly — Lắp Ghép Kênh (biến nội dung VOD thành kênh linear giống truyền hình).

## 📚 Mục Lục (Table of Contents)

1. [MediaTailor Là Gì?](#1-mediatailor-là-gì)
2. [Kiến Trúc Tổng Quan](#2-kiến-trúc-tổng-quan)
3. [Các Khái Niệm Cốt Lõi](#3-các-khái-niệm-cốt-lõi)
4. [Luồng Xử Lý SSAI](#4-luồng-xử-lý-ssai)
5. [Nội Dung Chi Tiết](#5-nội-dung-chi-tiết)
6. [Câu Hỏi Phỏng Vấn Thường Gặp](#6-câu-hỏi-phỏng-vấn-thường-gặp)

---

## 1. MediaTailor Là Gì?

**AWS Elemental MediaTailor** là dịch vụ **server-side personalization** (cá nhân hoá phía máy chủ) cho video streaming. Nằm giữa origin server (MediaPackage / CloudFront) và player, MediaTailor sửa manifest HLS/DASH theo từng viewer để chèn quảng cáo phù hợp hoặc lắp ghép chương trình VOD thành kênh phát liên tục.

### Vị Trí Trong Pipeline Live with Ads

```
                 ┌──────────── LIVE + ADS PIPELINE ─────────────────────────────┐
                 │                                                               │
[Encoder] → [MediaLive] → [MediaPackage] → [MediaTailor] → [CloudFront] → [Viewer]
  SCTE-35     Encode +        JIT HLS +         SSAI +           CDN           Player
  markers     ad markers      CUE-OUT/IN         ADS call          phân phối      ABR
```

### Chức Năng Chính Của MediaTailor

| Chức Năng | Mô Tả |
|-----------|-------|
| **SSAI** (Server-Side Ad Insertion) | Ghép quảng cáo vào manifest/segment trước khi viewer nhận — khó bị ad-blocker chặn |
| **Playback Configuration** (Cấu hình phát) | Định nghĩa content source, ADS URL, CDN prefix, transcode profile cho ad |
| **ADS integration** (Tích hợp ADS) | Gọi Ad Decision Server, nhận VAST/VMAP, chọn creative phù hợp từng viewer |
| **Ad marker handling** (Xử lý điểm quảng cáo) | Đọc SCTE-35 / `#EXT-X-CUE-OUT` từ MediaPackage, thay bằng ad segments |
| **Channel Assembly** (Lắp ghép kênh) | Xếp clip VOD theo lịch → kênh linear FAST — Free Ad-Supported Streaming TV |
| **Reporting & Beacons** (Báo cáo) | Tracking impression, quartile, complete qua beacon URLs |
| **Ad Prefetch** (Tải trước quảng cáo) | Tải ad creative trước ad break để giảm độ trễ chuyển cảnh |

### So Sánh SSAI vs CSAI

| Tiêu Chí | SSAI (MediaTailor) | CSAI (Client-Side) |
|----------|-------------------|-------------------|
| **Vị trí chèn ad** | Server (MediaTailor) | Player (SDK trong app/browser) |
| **Ad-blocker** | Khó chặn (cùng origin với content) | Dễ chặn (request riêng tới ad server) |
| **Cá nhân hoá** | Theo session/viewer trên server | Theo client SDK |
| **Độ phức tạp** | Cao (ADS, SCTE-35, transcode ad) | Trung bình (player SDK) |
| **Tương thích player** | HLS/DASH player chuẩn | Cần IMA SDK hoặc tương đương |
| **Use case** | OTT broadcast, FAST, live TV | YouTube-style web, một số app mobile |

> **Khi nào dùng MediaTailor?** Kênh OTT có quảng cáo, FAST platform, live sports với regional ads, hoặc cần một player stack thống nhất (không phụ thuộc Google IMA trên mọi nền tảng).

---

## 2. Kiến Trúc Tổng Quan

### Các Thành Phần Chính

```
┌─────────────────────────────────────────────────────────────────────────┐
│                    AWS ELEMENTAL MEDIATAILOR                            │
│                                                                         │
│  ┌─────────────────────────────┐    ┌─────────────────────────────────┐ │
│  │  Playback Configuration     │    │     Channel Assembly            │ │
│  │  (SSAI)                     │    │     (Linear VOD channel)        │ │
│  │                             │    │                                 │ │
│  │  • Content source URL       │    │  • Source location (S3)         │ │
│  │  • ADS URL + session attrs  │    │  • Program schedule (EPG)       │ │
│  │  • CDN configuration        │    │  • Slate / filler content       │ │
│  │  • Ad transcode profile     │    │  • Output manifest endpoint     │ │
│  └──────────────┬──────────────┘    └─────────────────────────────────┘ │
│                 │                                                       │
│                 ▼                                                       │
│  ┌──────────────────────────────────────────────────────────────────┐  │
│  │  Manifest Stitching (Ghép manifest)                              │  │
│  │  Content segments + Ad segments → Personalized HLS/DASH            │  │
│  └──────────────────────────────────────────────────────────────────┘  │
│                 │                    │                                    │
│                 ▼                    ▼                                    │
│  ┌──────────────────────┐  ┌────────────────────────────────────────┐  │
│  │  ADS (bên thứ ba)    │  │  Reporting: impression / quartile /    │  │
│  │  VAST / VMAP         │  │  complete beacons → ad analytics       │  │
│  └──────────────────────┘  └────────────────────────────────────────┘  │
└─────────────────────────────────────────────────────────────────────────┘
         ▲                                    │
         │ HLS manifest + ad markers          ▼ Personalized manifest
    MediaPackage / CloudFront              CloudFront → Player
```

### Luồng Dữ Liệu (Data Flow) — SSAI Live

```
1. Viewer request playback URL từ ứng dụng (qua CloudFront → MediaTailor)
2. MediaTailor fetch manifest gốc từ Content Source (MediaPackage endpoint)
3. Phát hiện ad break (#EXT-X-CUE-OUT hoặc SCTE-35 DATERANGE)
4. Gọi ADS với session parameters (viewer ID, geo, device, content ID...)
5. ADS trả VAST/VMAP: danh sách ad creative URLs + duration
6. MediaTailor transcode/stitch ad segments vào manifest
7. Trả manifest đã cá nhân hoá cho player
8. Player request ad + content segments (tracking beacons fire theo quartile)
```

---

## 3. Các Khái Niệm Cốt Lõi

### 3.1 Playback Configuration (Cấu Hình Phát SSAI)

**Playback Configuration** là tài nguyên trung tâm của SSAI. Mỗi config gắn:

- **Content segment prefix** — URL gốc nội dung (thường là MediaPackage endpoint hoặc CloudFront trước origin)
- **Ad decision server URL** — endpoint ADS nhận query parameters cá nhân hoá
- **CDN configuration** — prefix CDN cho segment sau khi stitch
- **Avail settings** — cách map ad break (SCTE-35, CUE tags, hoặc schedule)

```
Playback Configuration: ott-live-hls
├── Name: ott-live-hls
├── Video content source URL: https://d111.cloudfront.net/out/v1/.../index.m3u8
├── Ad decision server URL:   https://ads.example.com/vast?session=[session]
├── CDN prefix:               https://d222.cloudfront.net/
├── Slate ad URL:             https://slate.example.com/filler.mp4  (khi ADS không trả ad)
└── Transcode profile:        H.264 720p cho ad creative không khớp bitrate ladder
```

### 3.2 Session Initialization (Khởi Tạo Phiên)

Trước khi phát, player/ứng dụng gọi **Session Initialization** để MediaTailor tạo session ID và nhận playback URL đã gắn session:

```
POST https://api.mediatailor.{region}.amazonaws.com/v1/session/.../configuration-name

Body: {
  "origin": "https://app.example.com",
  "manifestLayout": "MULTI_PERIOD",
  "playerParams": { "device": "ios", "geo": "VN" }
}

Response: {
  "manifestUrl": "https://d222.cloudfront.net/v1/master/.../index.m3u8?aws.sessionId=..."
}
```

Session attributes được forward tới ADS để chọn quảng cáo phù hợp.

### 3.3 Avail (Khung Quảng Cáo)

**Avail** — khung thời gian trong stream dành cho quảng cáo, được đánh dấu bởi:

| Nguồn marker | Mô tả |
|--------------|-------|
| **SCTE-35** | Từ MediaLive → MediaPackage → HLS `#EXT-X-DATERANGE` |
| **HLS CUE tags** | `#EXT-X-CUE-OUT` / `#EXT-X-CUE-IN` trong manifest |
| **Scheduled avails** | VOD: định nghĩa avail theo timeline trong manifest |

### 3.4 Channel Assembly (Lắp Ghép Kênh Linear)

Biến thư viện VOD trên S3 thành **kênh phát liên tục** có lịch chương trình (EPG — Electronic Program Guide):

```
S3: s3://vod-library/movies/...
         │
         ▼
Channel Assembly Program
├── 08:00–10:00  Movie A
├── 10:00–10:30  News clip
├── 10:30–12:00  Movie B
└── 12:00+       Live slate (filler)
         │
         ▼
Output: HLS manifest URL (linear, viewer không biết đang xem VOD)
```

Dùng cho **FAST** — Free Ad-Supported Streaming TV và kênh OTT giả lập broadcast.

---

## 4. Luồng Xử Lý SSAI

### Tích Hợp SCTE-35 End-to-End

```
MediaLive Schedule Action (Splice Insert, 60s)
        │
        ▼
MediaPackage HLS Endpoint (Ad markers: DATERANGE / CUE-OUT)
        │
        ▼
MediaTailor đọc avail → gọi ADS
        │
        ▼
VAST response: 2× 30s pre-roll trong break 60s
        │
        ▼
Stitch: thay 60s "black/slate" bằng 2 ad segments đã transcode
        │
        ▼
Player nhận manifest liền mạch (content → ad → content)
```

### URL Playback Cho Production

```
Khuyến nghị kiến trúc URL:

Viewer → CloudFront (MediaTailor origin path)
      → MediaTailor (stitch + ADS)
      → CloudFront hoặc MediaPackage (content origin)

Không expose MediaPackage URL trực tiếp cho viewer — luôn qua MediaTailor + CDN.
```

---

## 5. Nội Dung Chi Tiết

```
06-mediatailor/
├── README.md                 ← [BẠN ĐANG Ở ĐÂY] Tổng quan
├── 1-ssai-basics.md          Playback Configuration, session, avail, manifest stitching
├── 2-ads-integration.md      ADS, VAST/VMAP/VPAID, session parameters
├── 3-channel-assembly.md     Linear channel từ VOD, EPG, slate
├── 4-reporting-beacons.md     Impression, quartile, complete tracking
└── 5-prefetch-ads.md         Ad prefetch, giảm latency ad break
```

### Bản Đồ Học Tập Đề Xuất

```
Bắt đầu tại đây (README)
         │
         ▼
1-ssai-basics.md           ← Playback Configuration và luồng SSAI
         │
         ▼
2-ads-integration.md       ← ADS và định dạng quảng cáo
         │
         ▼
3-channel-assembly.md      ← FAST / linear VOD channel
         │
         ▼
4-reporting-beacons.md     ← Đo lường hiệu quả quảng cáo
         │
         ▼
5-prefetch-ads.md          ← Tối ưu độ trễ ad break
```

---

## 6. Câu Hỏi Phỏng Vấn Thường Gặp

### Câu hỏi cơ bản

**Q: SSAI là gì và tại sao OTT dùng MediaTailor?**

> **SSAI** — Server-Side Ad Insertion ghép quảng cáo vào stream trên server, không phải trong player SDK. **MediaTailor** thực hiện việc này bằng cách sửa HLS/DASH manifest, gọi **ADS** lấy creative phù hợp từng viewer, và stitch segments. Lợi ích: trải nghiệm đồng nhất trên mọi device, khó bị ad-blocker chặn, phù hợp broadcast workflow có **SCTE-35**.

**Q: MediaTailor đặt ở đâu trong pipeline so với MediaPackage?**

> **MediaPackage** là origin: đóng gói TS → HLS/DASH, nhúng ad markers từ SCTE-35. **MediaTailor** đứng **sau** MediaPackage (hoặc sau CloudFront cache origin): đọc manifest đã có CUE markers, thay phần avail bằng ads. Viewer **không** gọi thẳng MediaPackage — gọi URL MediaTailor (thường qua CloudFront).

**Q: Khác nhau Playback Configuration và Channel Assembly?**

> **Playback Configuration**: SSAI trên stream có sẵn (live hoặc VOD manifest có ad breaks). **Channel Assembly**: tạo stream linear mới từ clip VOD + lịch phát; có thể kết hợp SSAI trên output. Dùng Assembly cho FAST; dùng Playback Config cho live TV có SCTE-35.

### Câu hỏi nâng cao

**Q: ADS không trả quảng cáo thì xử lý thế nào?**

> Cấu hình **slate** — filler ad URL (video tĩnh hoặc house ad) trong Playback Configuration. MediaTailor chèn slate khi ADS timeout, empty VAST, hoặc lỗi transcode. Nên monitor **fill rate** — tỷ lệ avail được lấp đầy bởi paid ads vs slate.

**Q: Làm sao giảm độ trễ khi bắt đầu ad break?**

> 1. **Ad prefetch**: player gọi prefetch API trước CUE-OUT (xem `5-prefetch-ads.md`). 2. Tăng **time-delay** trên MediaPackage nếu cần buffer cho ADS. 3. ADS response time < 200ms (cache, geo-distributed ADS). 4. Ad creative đã transcode đúng profile ladder.

**Q: Pricing MediaTailor tính như thế nào?**

> Theo **ad insertion minutes** (số phút quảng cáo đã chèn thành công) và **channel assembly minutes** (số phút nội dung linear đã phát). Không tính theo số viewer — nhưng số insertion tăng theo số ad break × session. Ước tính bằng [AWS MediaTailor Pricing](https://aws.amazon.com/mediatailor/pricing/).

---

## 🔗 Tài Liệu Tham Khảo

- [MediaTailor User Guide](https://docs.aws.amazon.com/mediatailor/latest/ug/)
- [MediaTailor API Reference](https://docs.aws.amazon.com/mediatailor/latest/apireference/)
- [MediaTailor Pricing](https://aws.amazon.com/mediatailor/pricing/)
- [SSAI with MediaPackage and MediaLive](https://docs.aws.amazon.com/mediatailor/latest/ug/setting-up-ssai.html)
- [Channel Assembly Guide](https://docs.aws.amazon.com/mediatailor/latest/ug/channel-assembly.html)

---

**Cập Nhật Lần Cuối:** 2026-06-04
**Phần Trước:** [05-mediastore/](../05-mediastore/README.md) — Media Storage
**Phần Tiếp Theo:** [07-ivs/](../07-ivs/README.md) — Interactive Live Streaming
