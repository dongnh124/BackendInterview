# MediaPackage — Time-shift Viewing (Xem Dịch Chuyển Thời Gian)

> Time-shift viewing cho phép viewer xem lại nội dung đã phát mà không cần lưu trữ riêng biệt. MediaPackage duy trì rolling buffer của stream live, viewer có thể "tua về quá khứ" trong khoảng thời gian window cấu hình. Đây là nền tảng cho Startover — Xem Lại Từ Đầu và Catch-up TV — Bắt Kịp Chương Trình Đã Phát.

## 📚 Mục Lục

1. [Time-shift Viewing Là Gì?](#1-time-shift-viewing-là-gì)
2. [Rolling Buffer — Bộ Đệm Cuộn](#2-rolling-buffer--bộ-đệm-cuộn)
3. [Startover — Xem Lại Từ Đầu](#3-startover--xem-lại-từ-đầu)
4. [Catch-up TV — Bắt Kịp Chương Trình](#4-catch-up-tv--bắt-kịp-chương-trình)
5. [Windowed Manifest — Manifest Theo Cửa Sổ Thời Gian](#5-windowed-manifest--manifest-theo-cửa-sổ-thời-gian)
6. [Kết Hợp Với DRM](#6-kết-hợp-với-drm)
7. [Chi Phí và Giới Hạn](#7-chi-phí-và-giới-hạn)
8. [Thực Hành: Cấu Hình Time-shift](#8-thực-hành-cấu-hình-time-shift)
9. [Câu Hỏi Phỏng Vấn](#9-câu-hỏi-phỏng-vấn)

---

## 1. Time-shift Viewing Là Gì?

### Bài Toán Thực Tế

```
Tình huống: Người dùng bắt đầu xem chương trình thể thao 30 phút sau khi phát sóng.
            Hoặc: Người dùng muốn xem lại bàn thắng vừa xảy ra.
            Hoặc: Người dùng bỏ lỡ tin tức buổi sáng, muốn xem lại lúc trưa.

Không có Time-shift:
  → Viewer chỉ có thể xem live (thời điểm hiện tại)
  → Bỏ lỡ = bỏ luôn

Với Time-shift (MediaPackage):
  → Viewer request nội dung tại bất kỳ thời điểm nào trong window
  → MediaPackage tạo manifest từ buffer tại thời điểm đó
  → Không cần infrastructure thêm, không cần S3 riêng cho catch-up
```

### Ba Tính Năng Time-shift Chính

| Tính Năng | Mô Tả | Ví Dụ |
|-----------|-------|-------|
| **Startover** (Xem lại từ đầu) | Xem từ đầu sự kiện đang live | Bắt đầu xem trận đấu từ phút 1 dù đang ở phút 60 |
| **Catch-up TV** (Bắt kịp chương trình) | Xem nội dung đã phát trong quá khứ | Xem tin tức 9 giờ sáng vào buổi trưa |
| **Time-delay playback** (Phát lại trễ) | Xem live nhưng trễ so với thực tế | Tua đi tua lại trong vòng 1 giờ qua |

---

## 2. Rolling Buffer — Bộ Đệm Cuộn

### Cơ Chế Lưu Trữ

MediaPackage lưu TS segments trong **rolling buffer** được quản lý nội bộ — không phải S3. Buffer tự động xoá các segments cũ hơn **startover window**:

```
Timeline:  ←────────────────────────────────────────────→
           [cũ bị xoá]  [trong window]              [LIVE]
                        ↑                           ↑
                   startover window start      current time

Ví dụ: startover window = 7 ngày
  Ngày 1 00:00 → seg-0001.ts, seg-0002.ts, ...
  Ngày 1 12:00 → seg-7200.ts, seg-7201.ts, ...
  Ngày 7 23:00 → seg-100800.ts (live segs)
                 Buffer vẫn giữ segments từ ngày 1 00:00

  Ngày 8 00:00 → Ngày 1 00:00 bị xoá khỏi buffer (vượt 7 ngày)
```

### Startover Window Configuration

```
Cấu hình trên Endpoint (không phải Channel):
  startoverWindowSeconds: 0 → 1,209,600 (14 ngày tối đa)

Mặc định: 0 (không có startover, chỉ live)

Ví dụ thực tế:
  Kênh tin tức 24/7:    86,400  (24 giờ) → xem lại tin trong ngày
  Kênh thể thao:        604,800 (7 ngày)  → replay trận đấu cả tuần
  Kênh sự kiện live:    3,600   (1 giờ)   → chỉ startover, không catch-up lâu dài
```

---

## 3. Startover — Xem Lại Từ Đầu

### Startover Là Gì?

**Startover** cho phép viewer bắt đầu xem một chương trình từ đầu, ngay cả khi chương trình đang phát sóng live. Viewer "bắt đầu lại" (start over) từ thời điểm chương trình bắt đầu.

### Cách Viewer Request Startover

```
Live stream đang ở: 14:45
Chương trình bắt đầu: 14:00

Viewer muốn xem từ 14:00 (startover):
GET /hls/index.m3u8?startTime=2026-06-04T14:00:00Z

MediaPackage:
  1. Tìm segments trong buffer tại 14:00
  2. Tạo manifest HLS với EXT-X-PLAYLIST-TYPE:EVENT
     (không phải LIVE — manifest có điểm bắt đầu cố định)
  3. Viewer xem từ 14:00 → bắt kịp live dần dần
```

### HLS Manifest Cho Startover vs Live

```
Live manifest (không startover):
#EXTM3U
#EXT-X-VERSION:3
#EXT-X-TARGETDURATION:6
#EXT-X-MEDIA-SEQUENCE:7200         ← sequence number luôn tiến về phía trước
#EXTINF:6.0,
seg-007200.ts
#EXTINF:6.0,
seg-007201.ts
                                   ← Không có EXT-X-ENDLIST (đang live)

Startover manifest:
#EXTM3U
#EXT-X-VERSION:3
#EXT-X-TARGETDURATION:6
#EXT-X-PLAYLIST-TYPE:EVENT         ← Type EVENT: có điểm bắt đầu, chưa có điểm kết thúc
#EXT-X-MEDIA-SEQUENCE:0
#EXTINF:6.0,
seg-000001.ts                      ← Bắt từ 14:00
#EXTINF:6.0,
seg-000002.ts
...
#EXTINF:6.0,
seg-007201.ts                      ← Và tiếp tục thêm segments mới (đang live)
```

---

## 4. Catch-up TV — Bắt Kịp Chương Trình

### Catch-up TV Là Gì?

**Catch-up TV** cho phép viewer xem lại nội dung đã phát trong **khoảng thời gian startover window**. Viewer chọn một chương trình đã phát và xem lại toàn bộ như VOD.

### So Sánh Startover vs Catch-up

```
Startover:
  Chương trình: 14:00 → (đang live 14:45)
  Viewer request lúc: 14:30
  Viewer xem: 14:00 → 14:30 → rồi bắt kịp live
  → Xem chương trình ĐANG LIVE từ đầu

Catch-up:
  Chương trình: 09:00 → 10:00 (đã kết thúc, hiện là 16:00)
  Viewer request lúc: 16:00
  Viewer xem: 09:00 → 10:00 (VOD-like, có điểm kết thúc)
  → Xem lại chương trình ĐÃ KẾT THÚC
```

### EPG — Electronic Program Guide Tích Hợp

Để catch-up TV hoạt động tốt, cần tích hợp EPG — Electronic Program Guide — Hướng Dẫn Chương Trình Điện Tử:

```
EPG cung cấp:
  Chương trình A: 09:00–10:00 "Tin tức buổi sáng"
  Chương trình B: 10:00–11:00 "Phim tài liệu"
  Chương trình C: 11:00–12:00 "Thời sự quốc tế"

Viewer chọn "Tin tức buổi sáng" → App tạo URL:
  startTime=2026-06-04T09:00:00Z&endTime=2026-06-04T10:00:00Z
  (endTime chỉ dùng để giới hạn manifest, không phải MediaPackage feature gốc)
```

### Giới Hạn Manifest Bằng endTime

MediaPackage không có tham số `endTime` gốc, nhưng có thể giả lập qua CloudFront Lambda@Edge:

```javascript
// Lambda@Edge: Thêm EXT-X-ENDLIST vào manifest khi đến endTime
exports.handler = async (event) => {
  const response = event.Records[0].cf.response;
  const endTime = parseEndTimeFromRequest(event);

  if (isCurrentTimePastEndTime(endTime)) {
    // Chèn EXT-X-ENDLIST vào cuối manifest
    response.body = appendEndList(response.body);
  }

  return response;
};
```

---

## 5. Windowed Manifest — Manifest Theo Cửa Sổ Thời Gian

### Manifest Window Trong Live vs Time-shift

```
Live manifest (playlistWindowSeconds = 60):
  Luôn hiển thị 10 segments mới nhất (60s / 6s = 10)
  Player thấy 10 segments ← manifest "trượt" về phía trước

Time-shift manifest (startTime=T):
  Hiển thị segments từ T đến T + playlistWindowSeconds
  Nếu T + 60s < live time: player thấy 10 segments cũ
  Nếu T + 60s >= live time: merge với live (EXT-X-PLAYLIST-TYPE:EVENT)
```

### Program-Date-Time — Timestamp Tuyệt Đối

Để time-shift hoạt động, manifest phải có **EXT-X-PROGRAM-DATE-TIME** — timestamp tuyệt đối cho mỗi segment:

```
HLS manifest với Program-Date-Time:
#EXTINF:6.0,
#EXT-X-PROGRAM-DATE-TIME:2026-06-04T14:00:00.000Z
seg-000001.ts
#EXTINF:6.0,
#EXT-X-PROGRAM-DATE-TIME:2026-06-04T14:00:06.000Z
seg-000002.ts
...
```

MediaPackage tự động thêm EXT-X-PROGRAM-DATE-TIME khi cấu hình:
```json
{
  "HlsPackage": {
    "ProgramDateTimeIntervalSeconds": 60
  }
}
```

---

## 6. Kết Hợp Với DRM

### Time-shift + DRM: Vấn Đề Key Rotation

Khi nội dung có DRM và key rotation, viewer xem catch-up có thể gặp phải segments được mã hoá với nhiều key khác nhau:

```
Timeline:
  12:00 → 12:00 + 24h: encrypted với Key-1
  13:00 (ngày 2): encrypted với Key-2

Viewer xem catch-up tại 11:58 ngày 1:
  seg tại 11:58 → Key-1 → OK
  seg tại 12:01 (key rotation!) → Key-2 → Player cần xin license mới
```

**Giải pháp:** Player hiện đại (ExoPlayer, AVPlayer, Shaka Player) tự xử lý multi-key playback — đọc manifest, phát hiện key change, xin license mới tự động.

### Persistent License vs Online License

| Loại License | Mô Tả | Dùng Cho |
|-------------|-------|---------|
| **Online License** | Player xin license mỗi lần phát, cần internet | Live + Catch-up TV |
| **Persistent License** | License lưu offline, phát không cần internet | Download để xem offline |

Time-shift với MediaPackage thường dùng **Online License** — mỗi lần viewer request catch-up, player xin license online.

---

## 7. Chi Phí và Giới Hạn

### Chi Phí Time-shift

MediaPackage tính phí theo **GB đóng gói (packaged output)**, không phải GB lưu trữ buffer. Rolling buffer nội bộ không tốn phí storage riêng.

```
Phí = GB đóng gói × $0.0XX/GB

Ví dụ: 1 kênh HD, 1 triệu viewer xem startover 1 giờ
  1 giờ × 5 Mbps × 1,000,000 viewers
  = 5,000,000 Mbps·giờ = rất lớn → CloudFront cache giảm đáng kể
  
Thực tế với CDN cache:
  Segment phổ biến → cache tại CloudFront → MediaPackage chỉ đóng gói 1 lần
  → Chi phí thực tế thấp hơn nhiều lý thuyết
```

### Giới Hạn (Limits)

| Giới Hạn | Giá Trị |
|---------|---------|
| Startover window tối đa | 1,209,600 giây (14 ngày) |
| Time delay tối đa | 21,600 giây (6 giờ) |
| Số channels tối đa (mặc định) | 30 (có thể request tăng) |
| Số endpoints tối đa per channel | 10 |

---

## 8. Thực Hành: Cấu Hình Time-shift

### Tạo Endpoint Với Startover 7 Ngày

```bash
aws mediapackage create-origin-endpoint \
  --channel-id "live-sports-channel" \
  --id "hls-catchup-endpoint" \
  --hls-package '{
    "SegmentDurationSeconds": 6,
    "PlaylistWindowSeconds": 300,
    "ProgramDateTimeIntervalSeconds": 60,
    "AdMarkers": "DATERANGE",
    "IncludeIframeOnlyStream": false
  }' \
  --startover-window-seconds 604800 \
  --time-delay-seconds 0 \
  --region us-east-1
```

### Test Startover URL

```bash
# URL live bình thường
ENDPOINT_URL="https://xxxxx.mediapackage.us-east-1.amazonaws.com/out/v1/abc/index.m3u8"

# URL startover (xem từ 1 giờ trước)
START_TIME=$(date -u -d "1 hour ago" +"%Y-%m-%dT%H:%M:%SZ")
STARTOVER_URL="${ENDPOINT_URL}?startTime=${START_TIME}"

echo "Startover URL: $STARTOVER_URL"
# https://xxxxx.mediapackage.../index.m3u8?startTime=2026-06-04T12:00:00Z

# Test với curl
curl -I "$STARTOVER_URL"
# Kỳ vọng: HTTP/1.1 200 OK
# Content-Type: application/x-mpegURL
```

### Kiểm Tra Manifest Trả Về

```bash
curl "$STARTOVER_URL" | head -20

# Kỳ vọng:
# #EXTM3U
# #EXT-X-VERSION:3
# #EXT-X-PLAYLIST-TYPE:EVENT           ← Dấu hiệu startover
# #EXT-X-TARGETDURATION:6
# #EXT-X-MEDIA-SEQUENCE:0
# #EXT-X-PROGRAM-DATE-TIME:2026-06-04T12:00:00.000Z
# #EXTINF:6.0,
# seg-000001.ts
# ...
```

---

## 9. Câu Hỏi Phỏng Vấn

**Q: Time-shift viewing trong MediaPackage hoạt động như thế nào?**

> MediaPackage duy trì **rolling buffer** lưu tất cả TS segments trong khoảng thời gian **startover window** (tối đa 14 ngày). Khi viewer request manifest với tham số `startTime=<ISO8601>`, MediaPackage tra buffer và tạo manifest bắt đầu từ thời điểm đó. Không cần S3 riêng, không cần pipeline riêng — tất cả tích hợp trong MediaPackage.

**Q: Startover khác Catch-up TV như thế nào?**

> - **Startover**: xem chương trình **đang live** từ đầu. Viewer request `startTime` là thời điểm chương trình bắt đầu, manifest có type `EVENT` (có điểm đầu, chưa có điểm cuối, segments mới vẫn thêm vào).
> - **Catch-up TV**: xem chương trình **đã kết thúc**. Viewer chọn từ EPG, request `startTime=<program_start>`. MediaPackage trả segments cũ từ buffer, không thêm segments mới sau khi đến `endTime`.

**Q: Rolling buffer trong MediaPackage có tốn phí storage không?**

> **Không tốn phí storage riêng.** MediaPackage quản lý rolling buffer nội bộ và không tính phí lưu trữ buffer. Chi phí time-shift chỉ tính khi viewer thực sự request nội dung (tính theo GB đóng gói output), tương tự như live streaming thông thường.

---

**Phần Tiếp Theo:** [5-mediapackage-v2.md](./5-mediapackage-v2.md) — MediaPackage V2 & Tính Năng Mới
