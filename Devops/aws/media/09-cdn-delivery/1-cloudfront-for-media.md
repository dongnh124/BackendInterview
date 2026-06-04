# CloudFront Cho Media — Cache Behaviors, TTL & Origin Groups

> Cấu hình **CloudFront** đúng cho HLS/DASH quyết định chất lượng phát (live vs VOD), chi phí origin, và khả năng chịu spike viewer. Bài này tập trung **cache behaviors**, **TTL** — Time To Live, **Origin Groups** failover, và các header media quan trọng.

## 📚 Mục Lục

1. [Cache Behavior Cho Streaming](#1-cache-behavior-cho-streaming)
2. [TTL & Cache Policy](#2-ttl--cache-policy)
3. [Cache Key & Query String](#3-cache-key--query-string)
4. [Origin Groups & Failover](#4-origin-groups--failover)
5. [Header & Range GET Cho Player](#5-header--range-get-cho-player)
6. [Thực Hành: Cấu Hình Distribution](#6-thực-hành-cấu-hình-distribution)
7. [Câu Hỏi Phỏng Vấn](#7-câu-hỏi-phỏng-vấn)

---

## 1. Cache Behavior Cho Streaming

### 1.1 Cấu Trúc Behavior

Mỗi **cache behavior** gồm:

```
Path pattern     →  Origin ID
Cache policy     →  TTL min/default/max, cache key
Origin request   →  Headers gửi tới origin (Host, OAC signing)
Response headers →  CORS, Cache-Control override (tùy chọn)
```

**Thứ tự ưu tiên:** CloudFront match behavior **cụ thể nhất** trước. Behavior mặc định `*` là fallback.

### 1.2 Path Pattern Điển Hình

| Pattern | Ví dụ file | Ghi chú |
|---------|------------|---------|
| `*.m3u8` | `index.m3u8`, `720p/index.m3u8` | Master & media playlist HLS |
| `*.mpd` | `index.mpd` | DASH manifest |
| `*.ts` | `seg00001.ts` | HLS MPEG-TS segment (legacy) |
| `*.m4s` | `chunk.m4s` | CMAF/fMP4 segment |
| `*.mp4` | `movie.mp4` | VOD progressive hoặc CMAF init |
| `/keys/*` | License, key rotation | Thường **không cache** hoặc TTL=0 |

### 1.3 Sơ Đồ Request Live HLS

```
Timeline live (segment 6s):

T0: manifest M1 lists seg1
T6: manifest M2 lists seg1, seg2
T12: manifest M3 lists seg1, seg2, seg3

Player poll manifest mỗi 2–6s:
  → CloudFront TTL manifest phải ≤ polling interval (hoặc stale policy hợp lý)

Player tải seg1 một lần:
  → seg1 URL không đổi → cache 24h+ tại edge
```

---

## 2. TTL & Cache Policy

### 2.1 Managed Cache Policies (Khuyến Nghị)

AWS cung cấp **managed cache policies** thay cho legacy `Forwarded Values`:

| Policy | Use case media |
|--------|----------------|
| `CachingOptimized` | Segment VOD, static assets |
| `CachingDisabled` | API, license, debug |
| `Elemental-MediaPackage` | Origin MediaPackage (nếu dùng preset AWS) |
| Custom | Live manifest TTL 2s, VOD manifest 300s |

### 2.2 TTL Theo Loại Nội Dung

| Loại | Live streaming | VOD |
|------|----------------|-----|
| Master manifest | 2–6 giây | 300–3600 giây |
| Media playlist | 2–6 giây | 60–300 giây |
| Segment (.ts/.m4s) | = segment duration → 86400s+ | 7 ngày (immutable) |
| Thumbnail poster | 3600s | 86400s |

**Minimum TTL = 0** cho phép respect `Cache-Control` từ origin nếu có.

### 2.3 Cache-Control Từ Origin

Origin (S3 hoặc MediaPackage) có thể trả header:

```http
Cache-Control: max-age=6, public          # manifest live
Cache-Control: max-age=31536000, immutable # segment VOD CMAF
```

CloudFront **cache policy** quyết định có **honor origin headers** hay dùng TTL cố định. Media pipeline thường **override tại CloudFront** để đồng nhất giữa nhiều origin.

### 2.4 Stale Content & Live

```
Vấn đề: TTL manifest = 6s nhưng player poll mỗi 2s

Giải pháp:
1. Giảm TTL manifest xuống 2s
2. Hoặc dùng Origin Shield + short TTL
3. Low-latency HLS: manifest/part có cơ chế blocking playlist — TTL cực ngắn

Không nên: TTL manifest 60s cho live → viewer lag 1 phút so với thực tế
```

---

## 3. Cache Key & Query String

### 3.1 Query String Trong Media URL

MediaPackage time-shift và một số SSAI URL có query:

```
/hls/index.m3u8?startTime=2026-06-04T12:00:00Z
/hls/index.m3u8?aws.manifestfilter=...
```

**Cache key** phải **bao gồm** query string liên quan nếu không viewer catch-up sẽ nhận manifest live của người khác.

| Cấu hình | Hậu quả |
|----------|----------|
| Query string: **All** | Cache đúng từng biến thể, nhiều key hơn |
| Query string: **Whitelist** `startTime` | Chỉ phân biệt time-shift |
| Query string: **None** | Sai cho time-shift / personalized manifest |

### 3.2 Headers Trong Cache Key

Thường **không** đưa `Authorization` vào cache key cho public segment. Với **signed URL**, chữ ký nằm trong query (`Expires`, `Signature`) → phải nằm trong cache key hoặc dùng **signed cookies** thay URL params.

---

## 4. Origin Groups & Failover

### 4.1 Origin Group Là Gì?

**Origin Group** gom **primary** + **secondary** origin. CloudFront chuyển sang secondary khi primary trả **5xx** hoặc **timeout**.

```
Origin Group: vod-failover
├── Primary:   S3 bucket us-east-1 (OAC)
└── Secondary: S3 bucket eu-west-1 (replica CRR — Cross-Region Replication)

Behavior /movies/* → Origin Group vod-failover
```

### 4.2 Khi Nào Dùng Cho Media

| Scenario | Primary | Secondary |
|----------|---------|-----------|
| DR — Disaster Recovery | S3 region A | S3 region B replica |
| Blue/green deploy | Bucket prod | Bucket staging (hiếm) |
| MediaPackage | Không dùng group thường — MP đã HA managed | |

> **Lưu ý:** Failover **không** sync real-time giữa hai bucket; CRR có độ trễ. Live stream failover thường dùng **MediaLive redundancy** + **một** MediaPackage origin, không phải dual S3.

### 4.3 Origin Shield

```
Bật Origin Shield tại region gần origin (ví dụ us-east-1):

[Edge PoP Tokyo] ──┐
[Edge PoP Sydney]──┼──▶ [Shield us-east-1] ──▶ [S3 / MediaPackage]
[Edge PoP Mumbai]──┘

Lợi ích live event: giảm concurrent connections tới MediaPackage từ hàng trăm PoP xuống 1 lớp trung gian
Chi phí: thêm data transfer qua Shield — cần tính trong TCO — Total Cost of Ownership
```

---

## 5. Header & Range GET Cho Player

### 5.1 CORS — Cross-Origin Resource Sharing

Web player (HLS.js, Shaka) từ `https://app.example.com` gọi `https://dxxx.cloudfront.net`:

```
Response Headers Policy cần:
  Access-Control-Allow-Origin: https://app.example.com
  Access-Control-Allow-Methods: GET, HEAD, OPTIONS
  Access-Control-Expose-Headers: Content-Length, Content-Range
```

S3 origin cũng cần CORS rule khớp; CloudFront có thể **thêm** header tại edge.

### 5.2 Range GET

VOD MP4 progressive:

```http
GET /movie.mp4 HTTP/1.1
Range: bytes=0-1048575

HTTP/1.1 206 Partial Content
Content-Range: bytes 0-1048575/500000000
```

CloudFront cache **từng range** riêng hoặc full object tùy kích thước — với segment HLS nhỏ (2–6 MB) thường tải full object.

### 5.3 HTTP/3 & TCP

Bật **HTTP/3 (QUIC)** trên distribution giúp mobile viewer giảm handshake latency khi tải nhiều segment song song.

---

## 6. Thực Hành: Cấu Hình Distribution

### 6.1 VOD HLS Từ S3 (Console Checklist)

```
1. Tạo distribution
   - Origin: S3 bucket, OAC enabled
   - Default root object: (để trống cho HLS path sâu)

2. Behavior 1: Path *.m3u8
   - Cache policy: Custom — Default TTL 3600, Min 0, Max 86400
   - Compress: Bật (manifest text nhỏ)

3. Behavior 2: Path *.ts hoặc *.m4s
   - Cache policy: CachingOptimized hoặc TTL max 7 ngày

4. Error pages: 403/404 → custom JSON (optional)

5. Logging: standard log → S3; hoặc real-time log → Kinesis
```

### 6.2 Live MediaPackage Origin

```
Origin domain: xxxxx.egress.mediapackage.us-east-1.amazonaws.com
Protocol: HTTPS only
Origin path: /out/v1/xxxx/

Behaviors:
  *.m3u8 → TTL 2–6s
  *.ts   → TTL 3600s+

Origin custom header (CDN Authorization):
  X-MediaPackage-CDN-Identifier: cloudfront
  + signing theo doc MediaPackage
```

### 6.3 CLI: Tạo Cache Policy Tùy Chỉnh (Ví Dụ)

```bash
aws cloudfront create-cache-policy \
  --cache-policy-config '{
    "Name": "MediaLiveManifestShortTTL",
    "Comment": "Manifest HLS live TTL ngan",
    "DefaultTTL": 2,
    "MaxTTL": 10,
    "MinTTL": 0,
    "ParametersInCacheKeyAndForwardedToOrigin": {
      "EnableAcceptEncodingGzip": true,
      "EnableAcceptEncodingBrotli": true,
      "QueryStringsConfig": { "QueryStringBehavior": "all" },
      "HeadersConfig": { "HeaderBehavior": "none" },
      "CookiesConfig": { "CookieBehavior": "none" }
    }
  }'
```

---

## 7. Câu Hỏi Phỏng Vấn

**Q: Cache manifest live 60 giây có được không?**

> **Không** cho live thông thường. Viewer sẽ phát nội dung cũ tới 60 giây. Live cần TTL manifest **bằng hoặc nhỏ hơn** segment duration / player refresh interval (thường 2–6 giây). VOD có thể cache manifest lâu hơn vì playlist tĩnh.

**Q: Segment đã cache, live vẫn chạy — có conflict không?**

> Không. Mỗi segment có **URL unique** (`seg00042.ts`). Playlist mới chỉ **thêm** URL mới; segment cũ vẫn valid trong window DVR. Player không request lại segment đã buffer trừ khi seek.

**Q: Origin Group thay thế MediaLive failover?**

> **Không.** Origin Group là failover **giữa origin storage** (S3). MediaLive failover là **giữa pipeline encode** (A/B). Hai lớp bổ sung nhau trong kiến trúc DR toàn diện.

**Q: Làm sao đo cache hit ratio?**

> Bật **CloudFront real-time logs** hoặc **standard logs** → field `x-edge-result-type` (Hit, Miss, RefreshHit). Metric **CacheHitRate** trong CloudWatch. Mục tiêu VOD segment: >90%; live manifest: hit rate thấp hơn là bình thường do TTL ngắn.

---

**Cập Nhật Lần Cuối:** 2026-06-04
**Xem thêm:** [3-oac-origin-security.md](./3-oac-origin-security.md) — Bảo mật S3 origin
