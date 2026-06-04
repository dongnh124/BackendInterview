# MediaTailor — Ad Prefetch (Tải Trước Quảng Cáo)

> **Ad Prefetch** — tải trước quảng cáo — cho phép player/ứng dụng báo MediaTailor **sắp có ad break** để dịch vụ gọi ADS và chuẩn bị creative **trước** khi `#EXT-X-CUE-OUT` xảy ra. Giảm độ trễ chuyển cảnh content → ad — pain point lớn nhất của SSAI live.

## 📚 Mục Lục

1. [Vấn Đề Độ Trễ Ad Break](#1-vấn-đề-độ-trễ-ad-break)
2. [Ad Prefetch Hoạt Động Thế Nào](#2-ad-prefetch-hoạt-động-thế-nào)
3. [Prefetch API & Player Integration](#3-prefetch-api--player-integration)
4. [Phối Hợp MediaPackage Time-Delay](#4-phối-hợp-mediapackage-time-delay)
5. [SCTE-35 Early Warning](#5-scte-35-early-warning)
6. [Best Practices & Anti-Patterns](#6-best-practices--anti-patterns)
7. [Câu Hỏi Phỏng Vấn](#7-câu-hỏi-phỏng-vấn)

---

## 1. Vấn Đề Độ Trễ Ad Break

### Timeline Không Prefetch (Live)

```
T=0s    SCTE-35 CUE-OUT xuất hiện trong manifest player đang phát
T=0s    Player buffer sắp hết → cần ad segments NGAY
T=0.2s  MediaTailor nhận request → gọi ADS
T=0.5s  ADS trả VAST
T=1-3s  Tải + transcode ad creative
T=3s+   Ad segments sẵn sàng
        └── Viewer thấy: buffer spinner / freeze / slate kéo dài
```

### Mục Tiêu Với Prefetch

```
T=-30s   Player/ hệ thống gọi Prefetch (biết break sắp tới)
T=-28s   ADS + transcode hoàn tất, ad trên CDN cache
T=0s     CUE-OUT → stitch tức thì → chuyển ad mượt
```

---

## 2. Ad Prefetch Hoạt Động Thế Nào

### Luồng Tổng Quan

```
┌──────────┐  1. POST prefetch      ┌──────────────┐  2. Gọi ADS sớm
│  Player  │ ─────────────────────▶ │ MediaTailor  │ ──────────────▶ ADS
│  / App   │  availOffset, duration  │              │ ◀────────────── VAST
└──────────┘                        │              │  3. Transcode + cache
       │                            └──────┬───────┘
       │  4. CUE-OUT → playback request     │
       └──────────────────────────────────▶│ Stitch dùng ad đã cache
```

### Điều Kiện Prefetch Thành Công

- **Session** đã khởi tạo (cùng `sessionId` với playback)
- **Playback Configuration** bật hỗ trợ prefetch (region/feature)
- **Avail duration** gửi trong prefetch khớp SCTE-35 thực tế (± vài giây)
- ADS respond đủ nhanh trong cửa sổ prefetch

---

## 3. Prefetch API & Player Integration

### 3.1 Prefetch Request (Khái Niệm)

```http
POST /v1/prefetch/session/{session-id}/configuration/{config-name} HTTP/1.1
Host: api.mediatailor.ap-southeast-1.amazonaws.com
Content-Type: application/json

{
  "availOffsetMillis": 30000,
  "availDurationMillis": 60000,
  "userAgent": "OTT-Player/2.0"
}
```

| Trường | Ý nghĩa |
|--------|---------|
| `availOffsetMillis` | Thời gian còn lại đến ad break (ước lượng) |
| `availDurationMillis` | Độ dài break dự kiến (từ SCTE-35 hoặc EPG) |

### 3.2 Nguồn Tín Hiệu "Sắp Có Break"

| Nguồn | Cách dùng |
|-------|-----------|
| **SCTE-35 in-band** | Parser đọc splice countdown trước CUE-OUT |
| **HLS DATERANGE** | `PLANNED-DURATION` + `START-DATE` trong manifest |
| **App metadata** | Sports app biết break ở phút 45 |
| **MediaLive schedule** | Backend schedule → push notification tới app |

### 3.3 Player Logic (Pseudo-code)

```javascript
const PREFETCH_LEAD_MS = 30000; // 30 giây trước break

function onManifestUpdate(manifest) {
  const nextBreak = parseNextCueOut(manifest); // { offsetMs, durationMs }
  if (nextBreak && nextBreak.offsetMs <= PREFETCH_LEAD_MS && !prefetchSent) {
    mediatailorPrefetch(sessionId, {
      availOffsetMillis: nextBreak.offsetMs,
      availDurationMillis: nextBreak.durationMs
    });
    prefetchSent = true;
  }
}

function onCueOutStart() {
  prefetchSent = false; // reset cho break tiếp theo
}
```

### 3.4 HLS.js / Native Player

- **HLS.js**: lắng nghe `FRAG_PARSING_METADATA` hoặc parse `#EXT-X-CUE-OUT-CONT`
- **AVPlayer (iOS)**: đọc interstitials / date range (tùy phiên bản)
- **ExoPlayer (Android)**: SCTE-35 metadata hoặc application-level schedule

---

## 4. Phối Hợp MediaPackage Time-Delay

### Time-Delay (Trễ Phát Manifest)

MediaPackage có thể **delay** manifest vài giây so với live edge:

```
Live thực tế:  12:00:00
Manifest edge: 12:00:10  (delay 10s)
```

**Lợi ích cho SSAI:**

- 10 giây buffer để MediaTailor prefetch + stitch trước khi player thấy CUE-OUT
- Giảm phụ thuộc player prefetch API (vẫn nên có cả hai)

### Công Thức Delay Gợi Ý

```
Time-delay ≥ ADS p95 latency + transcode time + safety margin

Ví dụ:
  ADS 300ms + transcode 2s + margin 2s → delay ≥ 5–10s
```

Trade-off: tăng delay → tăng latency live (viewer thấy "trễ" hơn thực tế).

| Use case | Delay khuyến nghị |
|----------|-------------------|
| Sports live (low latency) | 4–6s + prefetch bắt buộc |
| News / entertainment | 8–15s |
| FAST (pseudo-live) | Có thể 20s+ |

---

## 5. SCTE-35 Early Warning

### Splice Insert Countdown

SCTE-35 **Splice Insert** có thể gửi **early warning** trước break (break sắp diễn ra trong 30s):

```
MediaLive → MediaPackage → manifest DATERANGE với cue advance
         → App đọc countdown → Prefetch 30s trước
```

### Return to Network

Sau ad break, **SCTE-35 Return to Network** báo kết thúc avail — player reset prefetch flag, chuẩn bị break tiếp theo.

### Pipeline Checklist

- [ ] MediaLive output SCTE-35 PID đúng
- [ ] MediaPackage `Ad markers: SCTE-35` enabled trên HLS endpoint
- [ ] MediaTailor `AdMarkerPassthrough` nếu cần debug
- [ ] Player parser SCTE-35/DATERANGE tested trên staging

---

## 6. Best Practices & Anti-Patterns

### Best Practices

1. **Prefetch sớm 20–40 giây** cho live sports (ADS + transcode biến động)
2. **Cache ad segments** trên CloudFront với TTL ngắn, cache key có creative ID
3. **Pre-transcode** creative tại ADS (multi-bitrate VAST) → bỏ bước transcode realtime
4. **Monitor** `AdDecisionServer.Latency` và buffer ratio trong ad break
5. **Staging load test** với concurrent sessions gọi prefetch cùng lúc (halftime break)

### Anti-Patterns

| Anti-pattern | Hậu quả |
|--------------|---------|
| Prefetch quá sớm (> 2 phút) | ADS campaign hết hạn / creative thay đổi |
| Không prefetch, delay = 0 | Buffer stall trên live |
| Sai `availDuration` | Ad bị cắt hoặc slate pad dài |
| CDN cache manifest không có sessionId | Sai ads giữa viewers |
| Chỉ dựa delay, không prefetch | Latency live quá cao để đủ buffer |

### KPI Sau Triển Khai Prefetch

| KPI | Trước | Sau (mục tiêu) |
|-----|-------|----------------|
| Ad start delay | 2–5s | < 500ms |
| Rebuffer trong ad break | > 2% sessions | < 0.5% |
| Slate rate | 20% | < 10% (kèm ADS tuning) |

---

## 7. Câu Hỏi Phỏng Vấn

**Q: Prefetch có bắt buộc cho SSAI live không?**

> Không bắt buộc về mặt kỹ thuật, nhưng **gần như bắt buộc** cho UX production. Không prefetch cần time-delay MediaPackage lớn hoặc chấp nhận stall. Hầu hết OTT sports dùng **prefetch + delay vừa phải**.

**Q: Prefetch khác gì pre-roll caching trong CSAI?**

> CSAI player SDK tải VAST trước trong buffer client. **Prefetch MediaTailor** tải và transcode **phía server**, output vào manifest HLS chung — phù hợp SSAI model, player không cần ad SDK.

**Q: Làm sao test prefetch trước go-live?**

> 1. Staging channel với Schedule SCTE-35 mỗi 5 phút. 2. Player test harness gọi prefetch API log timestamp. 3. Đo thời gian từ CUE-OUT đến first ad frame. 4. So sánh có/không prefetch. 5. Load test ADS với concurrent prefetch bursts.

---

**Cập Nhật Lần Cuối:** 2026-06-04
**Phần Trước:** [4-reporting-beacons.md](./4-reporting-beacons.md)
**Phần Tiếp Theo:** [07-ivs/](../07-ivs/README.md) — Interactive Live Streaming
