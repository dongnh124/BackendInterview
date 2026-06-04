# Amazon CloudFront + Media — CDN Phân Phối Video (CDN Delivery)

> **Amazon CloudFront** là CDN — Content Delivery Network — Mạng Phân Phối Nội Dung toàn cầu của AWS. Trong pipeline media, CloudFront đứng sau **Origin** (S3, MediaPackage, MediaTailor) để cache manifest và segment, giảm độ trễ, giảm tải origin, và triển khai **bảo vệ nội dung** (signed URL/cookies), **OAC** — Origin Access Control — Kiểm Soát Truy Cập Origin, và logic tại **edge** (CloudFront Functions / Lambda@Edge).

## 📚 Mục Lục (Table of Contents)

1. [CloudFront Trong Pipeline Media](#1-cloudfront-trong-pipeline-media)
2. [Kiến Trúc Tổng Quan](#2-kiến-trúc-tổng-quan)
3. [Các Khái Niệm Cốt Lõi](#3-các-khái-niệm-cốt-lõi)
4. [Pipeline VOD & Live Với CloudFront](#4-pipeline-vod--live-với-cloudfront)
5. [Nội Dung Chi Tiết](#5-nội-dung-chi-tiết)
6. [Câu Hỏi Phỏng Vấn Thường Gặp](#6-câu-hỏi-phỏng-vấn-thường-gặp)

---

## 1. CloudFront Trong Pipeline Media

### Vai Trò Của CDN

```
Không có CDN:
  Viewer (Hà Nội) ──150–250ms──▶ Origin S3 us-east-1
  Mỗi request segment đều hit origin → chi phí egress cao, origin quá tải khi spike viewer

Có CloudFront:
  Viewer (Hà Nội) ──10–30ms──▶ Edge PoP Singapore
                              │
                    Cache HIT → trả segment từ edge (không gọi origin)
                    Cache MISS → fetch origin một lần, cache cho viewer khác
```

### Vị Trí Trong Các Pipeline AWS Media

| Pipeline | Origin | CloudFront |
|----------|--------|------------|
| **VOD** | S3 bucket (HLS/DASH từ MediaConvert) | Distribution → viewer |
| **Live OTT** | MediaPackage endpoint URL | Distribution + CDN Authorization |
| **Live + Ads** | MediaTailor playback endpoint | Distribution, cache manifest động |
| **IVS** | IVS tự quản lý CDN (không cần CloudFront riêng) | Managed bởi AWS |
| **DRM** | MediaPackage + license server | Signed URL/Cookies + OAC |

### Chức Năng Chính Cho Media

| Chức Năng | Mô Tả |
|-----------|-------|
| **Edge caching** (Cache tại biên) | Lưu `.m3u8`, `.mpd`, `.ts`, `.m4s` tại 450+ PoP — Point of Presence — Điểm Hiện Diện |
| **Cache behaviors** (Hành vi cache) | TTL — Time To Live — Thời Gian Sống khác nhau cho manifest vs segment |
| **Origin Groups** (Nhóm nguồn) | Failover S3 primary → secondary khi origin lỗi |
| **OAC** (Origin Access Control) | Chỉ CloudFront đọc được S3; chặn truy cập trực tiếp bucket |
| **Signed URL / Cookies** | Kiểm soát ai được xem nội dung trả phí |
| **Edge compute** | CloudFront Functions / Lambda@Edge: rewrite URL, geo block, A/B manifest |
| **Real-time logs** | Cache hit ratio, viewer country, status code theo thời gian thực |

---

## 2. Kiến Trúc Tổng Quan

### Distribution Cho Media OTT

```
┌─────────────────────────────────────────────────────────────────────────┐
│                    AMAZON CLOUDFRONT DISTRIBUTION                       │
│                                                                         │
│  Viewer request: https://d111111abcdef8.cloudfront.net/hls/index.m3u8  │
│                                                                         │
│  ┌──────────────┐    ┌─────────────────┐    ┌──────────────────────┐   │
│  │ Cache Policy │    │ Origin Request  │    │ Response Headers     │   │
│  │ (TTL, keys)  │    │ Policy (OAC,    │    │ Policy (CORS,        │   │
│  │              │    │  Host header)   │    │  Cache-Control)      │   │
│  └──────┬───────┘    └────────┬────────┘    └──────────┬───────────┘   │
│         │                     │                        │               │
│         └─────────────────────┼────────────────────────┘               │
│                               ▼                                        │
│                    ┌──────────────────────┐                            │
│                    │   Origin(s)          │                            │
│                    │  • S3 (VOD)          │                            │
│                    │  • MediaPackage      │                            │
│                    │  • MediaTailor       │                            │
│                    │  • Custom HTTP       │                            │
│                    └──────────────────────┘                            │
└─────────────────────────────────────────────────────────────────────────┘
         ▲                              │
         │         Edge locations       │
    [Viewer APAC] ◄─────────────────────┘
    [Viewer EU]
    [Viewer US]
```

### Luồng Request HLS Qua CloudFront

```
1. Player request master manifest: /hls/index.m3u8
2. CloudFront: cache behavior cho *.m3u8 → TTL ngắn (2–10s live, vài phút VOD)
3. MISS → origin MediaPackage trả manifest JIT
4. Player chọn rendition → request /hls/720p/seg001.ts
5. CloudFront: behavior cho *.ts → TTL dài hơn (segment immutable)
6. Hàng triệu viewer cùng segment → HIT tại edge → origin chỉ phục vụ lần đầu mỗi PoP
7. ABR — Adaptive Bitrate: player đổi rendition → request URL khác → cache key khác
```

---

## 3. Các Khái Niệm Cốt Lõi

### 3.1 Distribution (Phân Phối)

**Distribution** là thực thể CloudFront: domain `*.cloudfront.net` hoặc **custom domain** (CNAME) + certificate ACM.

```
Distribution settings quan trọng cho media:
├── Origins: danh sách nguồn (S3, custom domain MediaPackage)
├── Cache behaviors: path pattern → policy
├── Price class: All / 200 / 100 edge locations (ảnh hưởng chi phí & latency)
├── HTTP/2, HTTP/3 (QUIC): giảm latency mobile
├── IPv6: bật cho viewer hiện đại
└── WAF — Web Application Firewall: gắn ACL chống abuse API manifest
```

### 3.2 Cache Behavior (Hành Vi Cache)

Mỗi **cache behavior** map **path pattern** → **origin** + **cache policy** + **origin request policy**:

| Path Pattern | Origin | TTL gợi ý | Lý do |
|--------------|--------|------------|-------|
| `*.m3u8`, `*.mpd` | MediaPackage / S3 | Live: 2–6s; VOD: 300–3600s | Manifest thay đổi liên tục khi live |
| `*.ts`, `*.m4s` | Cùng origin | 86400s+ (immutable segment) | Segment đã phát không đổi nội dung |
| `/api/*` | API Gateway | 0 (no cache) | Không cache API license |

### 3.3 Origin Shield (Khiên Origin)

**Origin Shield** — vùng edge trung gian — gom request từ nhiều PoP về **một** kết nối tới origin:

```
Không Origin Shield:
  50 PoP × cùng segment mới → 50 request tới MediaPackage (spike live event)

Có Origin Shield (ví dụ eu-central-1):
  50 PoP → 1 Shield layer → 1 request tới origin
```

Phù hợp live event lớn, origin MediaPackage/S3 dễ bị throttle.

### 3.4 Range GET (Yêu Cầu Phạm Vi Byte)

Player và một số công cụ dùng **Range GET** — HTTP header `Range: bytes=0-` — để tải một phần file MP4 (VOD progressive) hoặc segment lớn. CloudFront **hỗ trợ Range GET** mặc định; cần origin S3 bật tương thích.

---

## 4. Pipeline VOD & Live Với CloudFront

### Use Case 1: VOD Sau MediaConvert

```
S3 (output HLS)
  ├── /movies/abc/index.m3u8
  ├── /movies/abc/720p/seg00001.ts
  └── ...

CloudFront Distribution
  Origin: S3 bucket (OAC — không public read)
  Behavior 1: *.m3u8 → Cache TTL 1h
  Behavior 2: *.ts   → Cache TTL 24h–7d
  Optional: Signed URL cho subscriber

Viewer → https://video.example.com/movies/abc/index.m3u8
```

### Use Case 2: Live MediaPackage + MediaTailor

```
MediaLive → MediaPackage → MediaTailor (SSAI) → CloudFront → Player

CloudFront origin: MediaTailor playback hostname
Cache:
  - Manifest (có ad markers): TTL rất ngắn hoặc dùng cache key có query string
  - Media segments: TTL theo segment duration

CDN Authorization trên MediaPackage:
  CloudFront ký request → MediaPackage chỉ accept từ CloudFront
```

### Use Case 3: Multi-Origin Failover (VOD DR)

```
Origin Group:
  Primary:   S3 bucket us-east-1
  Secondary: S3 bucket eu-west-1 (replica)

CloudFront failover criteria: 5xx, timeout
→ Viewer không thấy gián đoạn khi primary region sự cố
```

---

## 5. Nội Dung Chi Tiết

```
09-cdn-delivery/
├── README.md                    ← [BẠN ĐANG Ở ĐÂY] Tổng quan CloudFront + Media
├── 1-cloudfront-for-media.md    Cache behaviors, TTL, Origin Groups, Range GET
├── 2-signed-url-cookies.md      Signed URL vs Signed Cookies, key pair, policy
├── 3-oac-origin-security.md     OAC cho S3 & MediaPackage, so sánh OAI
└── 4-edge-functions-media.md    CloudFront Functions vs Lambda@Edge cho media
```

### Bản Đồ Học Tập Đề Xuất

```
Bắt đầu tại đây (README)
         │
         ▼
1-cloudfront-for-media.md      ← Cache strategy là trọng tâm phỏng vấn
         │
         ▼
3-oac-origin-security.md       ← Bảo mật origin (làm trước signed URL trong thực tế)
         │
         ▼
2-signed-url-cookies.md        ← Kiểm soát truy cập subscriber
         │
         ▼
4-edge-functions-media.md      ← Nâng cao: routing, geo, token tại edge
```

---

## 6. Câu Hỏi Phỏng Vấn Thường Gặp

### Câu hỏi cơ bản

**Q: Tại sao cần CloudFront cho video streaming?**

> Video streaming tạo **hàng triệu request nhỏ** (manifest + segment). Không CDN, mọi request đều về origin xa viewer → **latency cao**, **origin overload**, **chi phí egress** S3/MediaPackage tăng tuyến tính theo viewer. CloudFront cache tại edge gần viewer → latency thấp, origin chỉ phục vụ **cache miss**, chi phí và khả năng chịu tải cải thiện rõ rệt.

**Q: TTL manifest và segment khác nhau thế nào?**

> **Manifest** (`.m3u8`, `.mpd`) mô tả danh sách segment **đang thay đổi** khi live → TTL **ngắn** (vài giây) hoặc dùng **stale-while-revalidate**. **Segment** đã phát thường **immutable** → TTL **dài** (giờ đến ngày) vì URL segment không đổi nội dung. Cấu hình sai (cache manifest quá lâu khi live) gây player lag hoặc không bắt kịp live edge.

**Q: OAC khác OAI thế nào?**

> **OAI** — Origin Access Identity — mô hình cũ: identity riêng gắn bucket policy. **OAC** — Origin Access Control — mô hình mới: hỗ trợ **SSE-KMS** trên S3, nhiều distribution, signing **SigV4** chuẩn hơn. AWS khuyến nghị **OAC** cho mọi distribution S3 mới.

### Câu hỏi nâng cao

**Q: Signed URL vs Signed Cookies — chọn cái nào?**

> **Signed URL**: một URL có chữ ký cho **một object** (hoặc wildcard hẹp). Phù hợp link tải xuống, email deep link, API trả URL playback một lần.
>
> **Signed Cookies**: cookie áp dụng cho **nhiều file** trong path (`/premium/*`). Phù hợp web app: user login → set cookie → player request hàng trăm segment không cần ký từng URL.
>
> Cả hai dùng **CloudFront key pair** (hoặc **Key Group** với public key trong KMS/Secrets Manager).

**Q: Làm sao scale cho 1 triệu concurrent viewer live?**

> 1. **CloudFront** scale tự động — không cần provision capacity.
> 2. Bật **Origin Shield** để giảm thundering herd lên MediaPackage.
> 3. **CDN Authorization** + chỉ CloudFront được gọi origin.
> 4. TTL/cache key tối ưu: segment cache dài, manifest ngắn.
> 5. **MediaPackage** / **MediaTailor** đủ quota; monitor 5xx origin.
> 6. **Price class** và **regional edge caches** cân nhắc latency vs chi phí.
> 7. **WAF rate limiting** chống abuse manifest scraping.

**Q: IVS có cần CloudFront không?**

> **Không** — IVS — Interactive Video Service đã embed CDN managed. CloudFront chỉ cần khi bạn tự host **recording HLS trên S3** sau IVS hoặc kết hợp IVS + backend VOD riêng.

---

## 🔗 Tài Liệu Tham Khảo

- [CloudFront Developer Guide](https://docs.aws.amazon.com/AmazonCloudFront/latest/DeveloperGuide/)
- [Serving Video with CloudFront](https://docs.aws.amazon.com/AmazonCloudFront/latest/DeveloperGuide/LiveStreaming.html)
- [Restricting Access with Signed URLs/Cookies](https://docs.aws.amazon.com/AmazonCloudFront/latest/DeveloperGuide/PrivateContent.html)
- [Origin Access Control](https://docs.aws.amazon.com/AmazonCloudFront/latest/DeveloperGuide/private-content-restricting-access-to-s3.html)
- [CloudFront Functions vs Lambda@Edge](https://docs.aws.amazon.com/AmazonCloudFront/latest/DeveloperGuide/lambda-at-edge.html)

---

**Cập Nhật Lần Cuối:** 2026-06-04
**Phần Trước:** [08-kinesis-video/](../08-kinesis-video/README.md) — Kinesis Video Streams
**Phần Tiếp Theo:** [10-interview-prep/](../10-interview-prep/README.md) — Chuẩn Bị Phỏng Vấn
