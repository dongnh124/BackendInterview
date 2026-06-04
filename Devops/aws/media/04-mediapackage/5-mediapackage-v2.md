# MediaPackage V2 — Kiến Trúc Mới & Tính Năng Nâng Cao

> MediaPackage V2 là thế hệ tiếp theo của AWS Elemental MediaPackage, giới thiệu kiến trúc mới với Packaging Groups — Nhóm Đóng Gói, hỗ trợ LL-HLS và LL-DASH — Low-Latency, Harvest Jobs — Thu Hoạch Đoạn VOD, và cải thiện đáng kể khả năng mở rộng (scalability). V2 chạy song song với V1 và là hướng đi khuyến nghị cho các triển khai mới.

## 📚 Mục Lục

1. [V1 vs V2 — So Sánh Tổng Quan](#1-v1-vs-v2--so-sánh-tổng-quan)
2. [Packaging Groups — Nhóm Đóng Gói](#2-packaging-groups--nhóm-đóng-gói)
3. [Low-Latency Streaming — Phát Trực Tiếp Độ Trễ Thấp](#3-low-latency-streaming--phát-trực-tiếp-độ-trễ-thấp)
4. [Harvest Jobs — Thu Hoạch Đoạn VOD](#4-harvest-jobs--thu-hoạch-đoạn-vod)
5. [Cải Tiến Scalability — Khả Năng Mở Rộng](#5-cải-tiến-scalability--khả-năng-mở-rộng)
6. [HTTP PUT Ingest — Nhận Luồng Bằng HTTP PUT](#6-http-put-ingest--nhận-luồng-bằng-http-put)
7. [Migration V1 → V2](#7-migration-v1--v2)
8. [Câu Hỏi Phỏng Vấn](#8-câu-hỏi-phỏng-vấn)

---

## 1. V1 vs V2 — So Sánh Tổng Quan

### Bảng So Sánh Chi Tiết

| Tính Năng | MediaPackage V1 | MediaPackage V2 |
|-----------|----------------|----------------|
| **Kiến trúc** | Channel + Endpoints riêng lẻ | Packaging Group → Channel → Origin Endpoint |
| **Ingest protocol** | WebDAV | HTTP PUT (đơn giản hơn) |
| **Low-Latency** | Không hỗ trợ | LL-HLS + LL-DASH (CMAF Chunked) |
| **Harvest Jobs** | Không có | Có — xuất VOD clip từ live |
| **Scalability** | Tốt | Tốt hơn (kiến trúc mới) |
| **Packaging Group** | Không có | Có — quản lý nhiều channels theo nhóm |
| **CMAF** | Có (basic) | Có (full, kể cả low-latency) |
| **DRM** | SPEKE v1 | SPEKE v2 (cải tiến) |
| **Console** | AWS Console cũ | Tích hợp trong Console mới |
| **Khuyến nghị** | Existing workloads | **New deployments (khuyến nghị)** |

### Lý Do Nâng Cấp Lên V2

```
MediaPackage V1:
  Channel ──▶ Endpoint A (HLS)
  Channel ──▶ Endpoint B (DASH)
  Channel ──▶ Endpoint C (CMAF)
  Quản lý phân tán, khó track relationship

MediaPackage V2:
  Packaging Group
  └── Channel
      ├── Origin Endpoint A (HLS)
      ├── Origin Endpoint B (DASH)
      └── Origin Endpoint C (CMAF LL)
  
  Nhóm rõ ràng, policy tập trung, dễ quản lý theo team/use-case
```

---

## 2. Packaging Groups — Nhóm Đóng Gói

### Packaging Group Là Gì?

**Packaging Group** là tầng quản lý mới trong V2, bao gồm một hoặc nhiều Channel và Endpoints. Cho phép áp dụng chung cấu hình (CDN Authorization, DRM policy) cho toàn bộ group.

```
Packaging Group: "sports-live"
├── CDN Authorization Secret: arn:aws:secretsmanager:...
├── Channel: "football-final"
│   ├── Origin Endpoint: "hls-4k"
│   ├── Origin Endpoint: "hls-fullhd"
│   └── Origin Endpoint: "dash-widevine"
├── Channel: "basketball-playoffs"
│   ├── Origin Endpoint: "hls-main"
│   └── Origin Endpoint: "cmaf-ll"
└── Channel: "tennis-open"
    └── Origin Endpoint: "hls-mobile"
```

### Lợi Ích Của Packaging Group

1. **Quản lý tập trung**: CDN Authorization cấu hình một lần cho cả group
2. **Domain thống nhất**: tất cả endpoints trong group dùng cùng domain
3. **Phân quyền IAM**: restrict access theo group thay vì từng channel
4. **Billing visibility**: xem chi phí theo group

### Tạo Packaging Group (V2 API)

```bash
# Tạo Packaging Group
aws mediapackagev2 create-channel-group \
  --channel-group-name "sports-live" \
  --description "Live sports channels" \
  --region us-east-1

# Tạo Channel trong Group
aws mediapackagev2 create-channel \
  --channel-group-name "sports-live" \
  --channel-name "football-final" \
  --region us-east-1

# Tạo Origin Endpoint (V2)
aws mediapackagev2 create-origin-endpoint \
  --channel-group-name "sports-live" \
  --channel-name "football-final" \
  --origin-endpoint-name "hls-main" \
  --container-type HLS \
  --hls-manifests '[{
    "ManifestName": "index",
    "ChildManifestName": "index_1",
    "ManifestWindowSeconds": 60,
    "ProgramDateTimeIntervalSeconds": 60
  }]' \
  --segment '{
    "SegmentDurationSeconds": 6,
    "TsUseAudioRenditionGroup": true,
    "IncludeIframeOnlyStreams": false,
    "TsIncludeDvbSubtitles": false,
    "Scte": {"ScteFilter": ["SPLICE_INSERT", "PROVIDER_AD"]}
  }' \
  --startover-window-seconds 86400 \
  --region us-east-1
```

---

## 3. Low-Latency Streaming — Phát Trực Tiếp Độ Trễ Thấp

### LL-HLS — Low-Latency HLS

**LL-HLS** — Low-Latency HLS là phần mở rộng của Apple cho HLS, cho phép latency 1–3 giây thay vì 15–30 giây của HLS thông thường. Cơ chế dựa trên **Partial Segments** — Đoạn Một Phần và **Preload Hints** — Gợi Ý Tải Trước.

```
HLS thông thường:
  Player phải chờ segment hoàn chỉnh (6 giây) mới có thể request
  Latency = 3 × segment_duration = 3 × 6s = 18 giây (+ CDN delay)

LL-HLS với Partial Segments:
  Server publish partial segment mỗi 0.3-1 giây (không đợi 6 giây)
  Player request partial segment ngay khi có
  Latency ≈ 2-3 giây
```

### Manifest LL-HLS

```
#EXTM3U
#EXT-X-VERSION:9                      ← V9 cho LL-HLS
#EXT-X-TARGETDURATION:6
#EXT-X-SERVER-CONTROL:CAN-BLOCK-RELOAD=YES,PART-HOLD-BACK=1.0
#EXT-X-PART-INF:PART-TARGET=0.33334  ← Partial segment mỗi ~333ms

#EXTINF:6.0,
full-segment-001.ts                   ← Segment đầy đủ (đã hoàn thành)

#EXT-X-PART:DURATION=0.33334,URI="partial-002-001.ts"
#EXT-X-PART:DURATION=0.33334,URI="partial-002-002.ts"
#EXT-X-PART:DURATION=0.33334,URI="partial-002-003.ts"
...                                   ← Partial segments đang tích lũy
#EXT-X-PRELOAD-HINT:TYPE=PART,URI="partial-002-019.ts"
                                      ← Gợi ý partial tiếp theo (chưa có)
```

### LL-DASH — Low-Latency DASH

**LL-DASH** dùng **Chunked Transfer Encoding** — Mã Hoá Truyền Theo Khối: segment được truyền từng chunk nhỏ ngay khi encoder tạo ra, không đợi segment hoàn chỉnh.

```
LL-DASH MPD (Media Presentation Description):
  <MPD type="dynamic" availabilityStartTime="..." suggestedPresentationDelay="PT1.5S">
    <Period>
      <AdaptationSet>
        <SegmentTemplate media="seg-$Number$.cmfv" timescale="90000"
                         duration="270000"        ← 3 giây/segment
                         availabilityTimeOffset="2.5"  ← LL: available 2.5s trước khi segment hoàn thành
                         availabilityTimeComplete="false" />  ← LL flag
```

### Cấu Hình LL-HLS Endpoint Trong MediaPackage V2

```bash
aws mediapackagev2 create-origin-endpoint \
  --channel-group-name "sports-live" \
  --channel-name "football-final" \
  --origin-endpoint-name "hls-low-latency" \
  --container-type HLS \
  --hls-manifests '[{
    "ManifestName": "ll-index",
    "ManifestWindowSeconds": 30,
    "ProgramDateTimeIntervalSeconds": 1
  }]' \
  --low-latency-hls-manifests '[{
    "ManifestName": "ll-hls-index"
  }]' \
  --segment '{
    "SegmentDurationSeconds": 6
  }' \
  --region us-east-1
```

### Latency Comparison

```
Standard HLS:           15–30 giây    (segment 6s, 3 segments buffer)
Low-Latency HLS:        2–3 giây      (partial segments)
LL-DASH:                2–4 giây      (chunked transfer)
CMAF Chunked (V2):      3–5 giây      (common format, cross-platform)
Amazon IVS (Standard):  5–15 giây
Amazon IVS (Low-lat):   2–5 giây
Amazon IVS (Real-time): < 1 giây      (WebRTC — chỉ small-scale)
```

---

## 4. Harvest Jobs — Thu Hoạch Đoạn VOD

### Harvest Jobs Là Gì?

**Harvest Job** — Thu Hoạch Đoạn VOD là tính năng xuất một đoạn nội dung từ live stream đang chạy (hoặc đã phát) thành file VOD lưu vào S3, mà không cần dừng stream hay re-encode.

```
Live stream:  [12:00]──────────────────────[16:00]──▶ (đang live)
                 │              │
           Harvest start    Harvest end
           12:00             13:00

Harvest Job tạo:
  S3: s3://my-bucket/harvest-output/index.m3u8   (HLS VOD)
      s3://my-bucket/harvest-output/seg-001.ts
      s3://my-bucket/harvest-output/seg-002.ts
      ...
      (Các segment từ 12:00 đến 13:00)
```

### Use Cases Harvest Jobs

```
1. Highlight clip từ live sport:
   "Xuất đoạn bàn thắng 79:30 → 80:15"
   startTime: 2026-06-04T19:30:00Z
   endTime:   2026-06-04T20:00:15Z
   → VOD clip 45 giây

2. Lưu chương trình phát sóng:
   "Lưu tin tức buổi tối 19:00-20:00 thành VOD"
   → Phục vụ catch-up sau khi window hết hạn

3. Preview content:
   "Xuất 5 phút đầu buổi phỏng vấn để làm teaser"
   → Marketing content từ live event

4. Compliance recording:
   "Lưu lại đoạn quảng cáo đã phát để kiểm tra"
   → Regulatory requirement
```

### Tạo Harvest Job

```bash
aws mediapackage create-harvest-job \
  --id "football-final-highlights" \
  --origin-endpoint-id "hls-main-endpoint" \
  --s3-destination '{
    "BucketName": "my-media-bucket",
    "ManifestKey": "harvests/football-final/index.m3u8",
    "RoleArn": "arn:aws:iam::123456789:role/MediaPackageHarvestRole"
  }' \
  --start-time "2026-06-04T19:30:00Z" \
  --end-time "2026-06-04T20:00:15Z" \
  --region us-east-1
```

### Kiểm Tra Trạng Thái Harvest Job

```bash
aws mediapackage describe-harvest-job \
  --id "football-final-highlights" \
  --region us-east-1

# Response:
{
  "Status": "SUCCEEDED",
  "Id": "football-final-highlights",
  "S3Destination": {
    "BucketName": "my-media-bucket",
    "ManifestKey": "harvests/football-final/index.m3u8"
  },
  "StartTime": "2026-06-04T19:30:00.000Z",
  "EndTime": "2026-06-04T20:00:15.000Z",
  "CreatedAt": "2026-06-04T21:00:00.000Z",
  "ModifiedAt": "2026-06-04T21:05:30.000Z"
}
```

> **Lưu ý:** Harvest Job chỉ hoạt động trong **startover window**. Nếu window là 7 ngày, chỉ harvest được nội dung trong 7 ngày gần nhất.

---

## 5. Cải Tiến Scalability — Khả Năng Mở Rộng

### Kiến Trúc Mở Rộng Trong V2

MediaPackage V2 thiết kế lại kiến trúc nội bộ để xử lý:
- **Sudden traffic spikes** — đột biến lưu lượng (sự kiện thể thao lớn)
- **Long-tail channels** — nhiều kênh nhỏ chạy đồng thời
- **Global distribution** — phân phối toàn cầu

```
Horizontal scaling trong V2:
  Request đến Origin Endpoint
       │
       ▼
  Load Balancer (tự động, không cần cấu hình)
       │
  ┌────┴────────────────────┐
  ▼            ▼            ▼
Packager-1   Packager-2   Packager-N
  (JIT)        (JIT)        (JIT)
       │            │            │
       └─────────────────────────┘
                    │
              Shared TS Buffer
              (consistent state)
```

### Best Practices Scalability

```
1. CloudFront trước MediaPackage:
   → Cache manifest 5-10 giây
   → Cache segments theo Cache-Control header
   → Giảm 80-90% requests về origin

2. Segment caching:
   CloudFront TTL:
     Manifest (.m3u8): Cache-Control: max-age=3       (3 giây — luôn fresh)
     Segment (.ts):    Cache-Control: max-age=3600     (1 giờ — immutable)

3. Origin Shield:
   Bật CloudFront Origin Shield tại region gần MediaPackage
   → Consolidate cache misses về 1 điểm
   → Giảm concurrent requests đến MediaPackage

4. Channel count:
   Mỗi region có limit channels mặc định = 30
   Request AWS tăng limit nếu cần nhiều hơn
```

---

## 6. HTTP PUT Ingest — Nhận Luồng Bằng HTTP PUT

### V1 WebDAV vs V2 HTTP PUT

```
V1 WebDAV ingest:
  PUT /in/v2/{channelId}/{channelId}/channel HTTP/1.1
  Host: xxxxx.mediapackage.us-east-1.amazonaws.com
  Connection: keep-alive
  Content-Type: video/MP2T
  Authorization: Basic base64(user:pass)
  Transfer-Encoding: chunked     ← WebDAV protocol overhead

  [TS stream binary data...]

V2 HTTP PUT ingest:
  PUT /in/v1/{channelGroupName}/{channelName}/ingest/{ingestEndpointName}
  Host: xxxxx.mediapackagev2.us-east-1.amazonaws.com
  Content-Type: video/MP2T
  Authorization: AWS4-HMAC-SHA256...  ← SigV4 (thay vì Basic Auth)
  Transfer-Encoding: chunked

  [TS stream binary data...]
```

### Lợi Ích HTTP PUT V2

1. **SigV4 Authentication**: bảo mật hơn Basic Auth của V1
2. **IAM-native**: dùng IAM Role thay vì username/password
3. **Standard HTTP**: tương thích với nhiều encoder hơn
4. **Simpler debugging**: dễ trace/debug hơn WebDAV

### Cấu Hình MediaLive Gửi Vào V2

```
MediaLive Output Group → MediaPackage Output:
  MediaPackage V2 Channel:
    Channel Group Name: sports-live
    Channel Name: football-final
    (MediaLive tự resolve ingest URLs từ V2 API)
```

---

## 7. Migration V1 → V2

### Khi Nào Nên Migrate?

- **New deployments**: Luôn dùng V2
- **Existing V1**: Migrate khi cần LL-HLS, Harvest Jobs hoặc quản lý nhiều channels
- **V1 remains supported**: AWS vẫn hỗ trợ V1, không bắt buộc migrate ngay

### Migration Path

```
Bước 1: Tạo V2 Packaging Group + Channel + Endpoints
         (song song với V1 đang chạy)

Bước 2: Test V2 endpoints với traffic nhỏ (dev/staging)

Bước 3: Cập nhật CloudFront Distribution:
         Origin URL → V2 endpoint URL

Bước 4: Migrate MediaLive Output:
         Đổi Output Group từ V1 channel → V2 channel
         (Cần dừng và restart channel để apply)

Bước 5: Monitor metrics trên V2 trong 24-48 giờ

Bước 6: Xoá V1 resources sau khi ổn định
```

### Khác Biệt API V1 vs V2

```bash
# V1
aws mediapackage create-channel ...
aws mediapackage create-origin-endpoint ...

# V2
aws mediapackagev2 create-channel-group ...
aws mediapackagev2 create-channel ...
aws mediapackagev2 create-origin-endpoint ...
```

---

## 8. Câu Hỏi Phỏng Vấn

**Q: MediaPackage V2 cải tiến gì so với V1?**

> V2 giới thiệu 4 cải tiến chính:
> 1. **Packaging Groups**: quản lý nhiều channels theo nhóm, CDN Auth và policy tập trung
> 2. **Low-Latency**: LL-HLS và LL-DASH (CMAF Chunked Transfer), latency 2-5 giây
> 3. **Harvest Jobs**: xuất VOD clip từ live stream trực tiếp ra S3 không cần re-encode
> 4. **HTTP PUT Ingest**: thay WebDAV, dùng SigV4 — bảo mật và đơn giản hơn

**Q: Harvest Job trong MediaPackage là gì và dùng khi nào?**

> Harvest Job cho phép xuất một đoạn nội dung từ live stream (trong rolling buffer) thành file HLS VOD lưu vào S3. Dùng khi:
> - Tạo highlight clip từ live sport ngay sau khi xảy ra
> - Lưu trữ chương trình phát sóng sau khi startover window sắp hết hạn
> - Tạo preview/teaser từ nội dung live
> - Compliance recording — lưu đoạn quảng cáo đã phát để kiểm tra

**Q: LL-HLS hoạt động như thế nào trong MediaPackage V2?**

> LL-HLS dùng **Partial Segments** — đoạn video ngắn (~333ms) thay vì chờ segment đầy đủ (6 giây). Manifest chứa URI của partial segments đang tích lũy và `EXT-X-PRELOAD-HINT` gợi ý partial tiếp theo. Player request partial ngay khi có → latency giảm xuống 2-3 giây. Player hỗ trợ LL-HLS: Safari 13.1+, AVPlayer iOS 14+, HLS.js v1.0+.

---

**Phần Trước:** [4-time-shift-viewing.md](./4-time-shift-viewing.md) — Time-shift & Catch-up TV
**Lên Thư Mục:** [README.md](./README.md) — MediaPackage Tổng Quan
