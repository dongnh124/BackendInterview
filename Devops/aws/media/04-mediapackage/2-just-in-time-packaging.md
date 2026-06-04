# MediaPackage — Just-In-Time Packaging (Đóng Gói Theo Yêu Cầu Tức Thì)

> JIT — Just-In-Time Packaging là cơ chế cốt lõi của MediaPackage: thay vì đóng gói tất cả formats trước, MediaPackage chỉ đóng gói khi viewer thực sự yêu cầu. Điều này tiết kiệm storage, tăng linh hoạt, và cho phép thay đổi cấu hình mà không cần re-process nội dung.

## 📚 Mục Lục

1. [JIT Packaging Là Gì?](#1-jit-packaging-là-gì)
2. [Luồng Xử Lý Nội Bộ](#2-luồng-xử-lý-nội-bộ)
3. [Bitrate Filtering — Lọc Tốc Độ Bit](#3-bitrate-filtering--lọc-tốc-độ-bit)
4. [Time Delay — Độ Trễ Có Chủ Ý](#4-time-delay--độ-trễ-có-chủ-ý)
5. [Segment Duration & Manifest Window](#5-segment-duration--manifest-window)
6. [Ad Marker Handling — Xử Lý Điểm Quảng Cáo](#6-ad-marker-handling--xử-lý-điểm-quảng-cáo)
7. [Câu Hỏi Phỏng Vấn](#7-câu-hỏi-phỏng-vấn)

---

## 1. JIT Packaging Là Gì?

### So Sánh Pre-packaged vs JIT

```
Pre-packaged (không dùng JIT):
────────────────────────────────────────────────────────
[MediaLive] → [MediaConvert S3] → [S3: HLS files]
                              → [S3: DASH files]
                              → [S3: CMAF files]
                         ↑
               Tốn 3× storage, phải encode 3 lần

JIT Packaging (MediaPackage):
────────────────────────────────────────────────────────
[MediaLive] → [MediaPackage: TS buffer]
                         │
                         ├──▶ HLS request?  → Đóng gói HLS ngay lúc đó
                         ├──▶ DASH request? → Đóng gói DASH ngay lúc đó
                         └──▶ CMAF request? → Đóng gói CMAF ngay lúc đó
                         ↑
               Lưu TS một lần, đóng gói on-demand
```

### Lợi Ích JIT

| Lợi Ích | Giải Thích |
|---------|-----------|
| **Tiết kiệm storage** | Chỉ lưu TS (Transport Stream) một lần, không lưu HLS/DASH/CMAF riêng |
| **Linh hoạt format** | Thêm endpoint CMAF mới không cần re-encode hay re-store nội dung cũ |
| **Thay đổi DRM live** | Bật/tắt DRM trên endpoint mà không ảnh hưởng channel |
| **Bitrate filtering** | Lọc renditions theo endpoint mà không cần job xử lý mới |
| **Chi phí hợp lý** | Trả theo GB đóng gói thực tế, không trả cho format không ai xem |

---

## 2. Luồng Xử Lý Nội Bộ

### Bước 1: Nhận và Lưu TS Segments

```
MediaLive ──WebDAV PUT──▶ MediaPackage Channel
                                 │
                    ┌────────────▼────────────┐
                    │     Internal Buffer      │
                    │  [seg-001.ts] 6 giây     │
                    │  [seg-002.ts] 6 giây     │
                    │  [seg-003.ts] 6 giây     │
                    │  ...                     │
                    │  (rolling window)        │
                    └─────────────────────────┘
```

### Bước 2: Viewer Request Manifest

```
Viewer ──GET /out/v1/{endpointId}/index.m3u8──▶ MediaPackage Endpoint
                                                         │
                                           ┌─────────────▼───────────────┐
                                           │   JIT Manifest Generator    │
                                           │                             │
                                           │  1. Đọc danh sách TS segs   │
                                           │  2. Apply endpoint config:  │
                                           │     - Bitrate filter        │
                                           │     - Window size           │
                                           │     - DRM key URLs          │
                                           │     - Time delay            │
                                           │  3. Tạo .m3u8 manifest      │
                                           └─────────────────────────────┘
                                                         │
                                              Trả về manifest .m3u8
```

### Bước 3: Viewer Request Segment

```
Viewer ──GET /out/v1/{endpointId}/1080p/seg-003.ts──▶ MediaPackage
                                                              │
                                              ┌───────────────▼──────────────┐
                                              │    JIT Segment Packager      │
                                              │                              │
                                              │  1. Đọc seg-003.ts từ buffer │
                                              │  2. Nếu HLS endpoint:        │
                                              │     → Trả .ts nguyên vẹn     │
                                              │  3. Nếu DASH/CMAF endpoint:  │
                                              │     → Chuyển TS → fMP4 (.m4s)│
                                              │  4. Nếu DRM enabled:         │
                                              │     → Mã hoá segment         │
                                              └──────────────────────────────┘
                                                              │
                                                   Trả về segment đã xử lý
```

### Phân Biệt TS vs fMP4

| Tiêu Chí | TS — Transport Stream | fMP4 — Fragmented MP4 |
|---------|----------------------|----------------------|
| **Extension** | `.ts` | `.m4s` |
| **Container** | MPEG-2 TS | ISO Base Media (MP4) |
| **Dùng bởi** | HLS truyền thống | CMAF, DASH, HLS mới |
| **Hiệu quả** | Thấp hơn (overhead lớn) | Cao hơn (overhead nhỏ) |
| **DRM** | AES-128 / SAMPLE-AES | CENC — Common Encryption |
| **Random access** | Không hiệu quả | Hiệu quả (moov atom) |

---

## 3. Bitrate Filtering — Lọc Tốc Độ Bit

### Khái Niệm

MediaPackage cho phép **lọc renditions** (các phiên bản độ phân giải) trên từng endpoint mà không thay đổi channel. Ví dụ: channel nhận 5 renditions từ MediaLive, nhưng endpoint mobile chỉ phát 3 renditions thấp nhất.

### Cấu Hình Stream Selection

```json
{
  "StreamSelection": {
    "MinVideoBitsPerSecond": 0,
    "MaxVideoBitsPerSecond": 1500000,
    "StreamOrder": "ORIGINAL"
  }
}
```

### Ví Dụ Thực Tế

```
Channel nhận từ MediaLive:
  Rendition 1: 1920×1080 @ 5,000,000 bps
  Rendition 2: 1280×720  @ 3,000,000 bps
  Rendition 3:  960×540  @ 1,500,000 bps
  Rendition 4:  640×360  @   800,000 bps
  Rendition 5:  480×270  @   400,000 bps

Endpoint "desktop":
  Min: 0, Max: 9,999,999
  Kết quả: tất cả 5 renditions ✓

Endpoint "mobile":
  Min: 0, Max: 1,500,000
  Kết quả: Rendition 3, 4, 5 (bỏ 1080p và 720p)

Endpoint "low-bandwidth":
  Min: 0, Max: 800,000
  Kết quả: Rendition 4, 5 (chỉ 360p và 270p)
```

### StreamOrder — Thứ Tự Hiển Thị Trong Manifest

| Giá Trị | Ý Nghĩa |
|---------|---------|
| `ORIGINAL` | Giữ nguyên thứ tự từ encoder |
| `VIDEO_BITRATE_ASCENDING` | Sắp xếp từ thấp đến cao (player chọn bitrate tăng dần) |
| `VIDEO_BITRATE_DESCENDING` | Sắp xếp từ cao đến thấp (player ưu tiên chất lượng cao) |

---

## 4. Time Delay — Độ Trễ Có Chủ Ý

### Time Delay Là Gì?

**Time Delay** (trễ có chủ ý) là tính năng làm chậm luồng phát so với thời gian thực. Ví dụ: cấu hình time delay 60 giây → viewer luôn xem nội dung trước 60 giây so với live.

### Use Cases Của Time Delay

```
Use case 1: Kiểm duyệt nội dung (Content moderation)
───────────────────────────────────────────────────────
Thực tế: 12:00:00 (live)
Viewer thấy: 11:59:00 (delay 60 giây)

Trong 60 giây đó, moderator có thể:
  - Cắt nội dung không phù hợp
  - Chèn slate thay thế
  - Trigger input switch sang nguồn khác

Use case 2: Đồng bộ quảng cáo (Ad synchronisation)
───────────────────────────────────────────────────────
Delay đủ để MediaTailor tải và chuẩn bị quảng cáo trước
khi SCTE-35 ad break thực sự phát đến viewer

Use case 3: Multi-region consistency (Nhất quán đa vùng)
───────────────────────────────────────────────────────
Delay nhỏ (5-10 giây) để CloudFront PoPs — Points of Presence
trên toàn cầu có thể cache segment trước khi viewer đầu tiên request
```

### Cấu Hình Time Delay

```bash
aws mediapackage create-origin-endpoint \
  --channel-id "my-channel" \
  --id "delayed-endpoint" \
  --hls-package '{"SegmentDurationSeconds": 6}' \
  --time-delay-seconds 60   # Delay 60 giây
```

**Giới hạn Time Delay:**
- Tối thiểu: 0 giây (không delay)
- Tối đa: 21,600 giây (6 giờ)
- Time delay không thể vượt quá Startover Window

---

## 5. Segment Duration & Manifest Window

### Segment Duration — Thời Lượng Mỗi Đoạn

**Segment duration** ảnh hưởng trực tiếp đến latency và chất lượng player experience:

```
Segment duration = 6 giây (mặc định):
  Latency: ~18-30 giây (3-5 segments buffer)
  Hiệu quả CDN caching: cao (segment lớn hơn)
  Phù hợp: standard live streaming, VOD-quality

Segment duration = 2 giây:
  Latency: ~6-10 giây
  Request rate tăng 3×: tốn nhiều HTTP connections hơn
  Phù hợp: sự kiện thể thao cần low latency

Segment duration = 1 giây (với LL-HLS):
  Latency: ~3-5 giây
  Chỉ khả dụng với MediaPackage v2 + CMAF endpoint
  Phù hợp: interactive live, real-time bidding cho ads
```

### Playlist Window Length — Độ Dài Cửa Sổ Manifest

**Playlist window** xác định bao nhiêu segments hiển thị trong manifest tại một thời điểm:

```
Segment duration: 6 giây
Playlist window: 60 giây → manifest có 10 segments

Manifest tại thời điểm T:
  seg-050.ts  (T - 54s)
  seg-051.ts  (T - 48s)
  seg-052.ts  (T - 42s)
  seg-053.ts  (T - 36s)
  seg-054.ts  (T - 30s)
  seg-055.ts  (T - 24s)
  seg-056.ts  (T - 18s)
  seg-057.ts  (T - 12s)
  seg-058.ts  (T - 6s)
  seg-059.ts  (T)       ← Segment mới nhất
```

**Trade-off Playlist Window:**

| Window ngắn (30-60s) | Window dài (300s+) |
|---------------------|-------------------|
| Manifest nhỏ hơn, load nhanh | Manifest lớn hơn |
| Player cần refresh thường xuyên | Player có nhiều buffer |
| Kém tolerant với network fluctuation | Tolerant với mạng chập chờn |
| Phù hợp: standard live | Phù hợp: catch-up trong cùng session |

---

## 6. Ad Marker Handling — Xử Lý Điểm Quảng Cáo

### SCTE-35 Từ MediaLive → MediaPackage

```
MediaLive ──[SCTE-35 splicing signal]──▶ MediaPackage
                                                │
                                   ┌────────────▼──────────────┐
                                   │  Ad Marker Conversion     │
                                   │                           │
                                   │  SCTE-35 binary signal    │
                                   │         ↓                 │
                                   │  HLS: EXT-X-CUE-OUT       │
                                   │  hoặc EXT-X-DATERANGE     │
                                   │         ↓                 │
                                   │  DASH: EventStream        │
                                   └───────────────────────────┘
                                                │
                                   MediaTailor đọc markers
                                   và chèn quảng cáo phù hợp
```

### Các Chế Độ Ad Markers Trong HLS

```
NONE: Không ghi ad markers vào manifest
  → Dùng khi không có quảng cáo

SCTE35_ENHANCED: Binary SCTE-35 chuẩn
  #EXT-X-CUE-OUT:DURATION=30
  → Dùng cho ad server truyền thống

DATERANGE: Dùng EXT-X-DATERANGE (khuyến nghị)
  #EXT-X-DATERANGE:ID="splice-0",START-DATE="2026-06-04T12:00:00Z",
    PLANNED-DURATION=30,SCTE35-OUT=0xFC00...
  → Dùng cho MediaTailor SSAI

PASSTHROUGH: Chuyển tiếp nguyên vẹn SCTE-35
  → Dùng khi downstream system cần binary SCTE-35 gốc
```

---

## 7. Câu Hỏi Phỏng Vấn

**Q: JIT Packaging trong MediaPackage khác gì so với đóng gói truyền thống?**

> JIT — Just-In-Time Packaging chỉ đóng gói khi viewer thực sự request, thay vì tạo sẵn tất cả formats. Kết quả: lưu TS một lần, không cần lưu HLS + DASH + CMAF riêng → giảm ~3× storage. Ngoài ra, thêm endpoint mới hoặc thay đổi DRM không cần re-process nội dung cũ.

**Q: Khi nào nên dùng segment duration nhỏ (2 giây) thay vì 6 giây?**

> Segment duration nhỏ giảm latency nhưng tăng số HTTP requests:
> - **Dùng 2 giây:** live sport, auction, event cần latency < 10 giây
> - **Dùng 6 giây:** tin tức, concert, streaming thông thường — CDN caching tốt hơn, origin load thấp hơn
> - **Trade-off:** 2 giây tăng origin requests ~3×, tăng CDN miss rate với cache TTL ngắn

**Q: Time Delay trong MediaPackage dùng để làm gì?**

> Time Delay trễ phát nội dung cho viewer so với thực tế:
> 1. **Content moderation**: cho operator 30-60 giây kiểm duyệt trước khi đến viewer
> 2. **Ad preparation**: đảm bảo MediaTailor có đủ thời gian tải quảng cáo trước SCTE-35 break
> 3. **Global CDN warm-up**: segment đủ thời gian được cache tại các PoP trên toàn cầu trước khi viewer request

---

**Phần Tiếp Theo:** [3-drm-speke.md](./3-drm-speke.md) — DRM & SPEKE Protocol
