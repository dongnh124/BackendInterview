# Amazon IVS — Timed Metadata (Metadata Theo Thời Gian)

> **Timed Metadata** cho phép backend nhúng payload (JSON, trigger ID) vào luồng live tại thời điểm ingest — **IVS Player SDK** phát event tới app để đồng bộ UI (sản phẩm, poll, hiệu ứng) với video mà không cần websocket riêng cho từng trigger.

## 📚 Mục Lục

1. [Timed Metadata Là Gì?](#1-timed-metadata-là-gì)
2. [PutMetadata API](#2-putmetadata-api)
3. [Luồng End-to-End](#3-luồng-end-to-end)
4. [Tích Hợp Player SDK](#4-tích-hợp-player-sdk)
5. [Thiết Kế & Giới Hạn](#5-thiết-kế--giới-hạn)
6. [Câu Hỏi Phỏng Vấn](#6-câu-hỏi-phỏng-vấn)

---

## 1. Timed Metadata Là Gì?

### Khác Chat và API Riêng

| Cơ chế | Đồng bộ video | Hướng |
|--------|---------------|-------|
| **Timed Metadata** | Gắn timeline luồng IVS | Server → tất cả viewer cùng lúc (theo latency mode) |
| **IVS Chat** | Không gắn frame chính xác | Viewer → room (có thể trễ network) |
| **WebSocket app** | Tự implement | Khó khớp với buffer HLS |

**Timed Metadata** tương tự **ID3 tags** trong broadcast HLS nhưng điều khiển qua API `PutMetadata` thay vì inject tại encoder.

### Use Cases

```
Live shopping:   hiện card sản phẩm khi streamer giới thiệu
Game show:       mở cửa vote / poll đúng nhịp chương trình
Education:       hiện quiz khi giáo viên bấm "Câu hỏi"
Sports overlay:  cập nhật tỷ số (kết hợp data provider)
```

---

## 2. PutMetadata API

### API Overview

```bash
aws ivs put-metadata \
  --channel-arn arn:aws:ivs:region:account:channel/xxx \
  --metadata '{"type":"product","sku":"SKU-001","price":199000}' \
  --region ap-northeast-1
```

| Tham số | Giới hạn / ghi chú |
|---------|-------------------|
| `metadata` | Chuỗi UTF-8, tối đa **1 KB** |
| `channelArn` | Channel đang **live** (có active stream) |
| IAM | `ivs:PutMetadata` trên channel ARN |

### Payload Khuyến Nghị

```json
{
  "v": 1,
  "type": "SHOW_PRODUCT",
  "id": "prod-8821",
  "title": "Áo thun limited",
  "durationMs": 30000
}
```

- Giữ payload **nhỏ** — tránh vượt 1 KB
- Có **version** (`v`) để app backward compatible
- Dùng `type` enum — client switch handler rõ ràng

### Ai Gọi PutMetadata?

```
┌─────────────┐     webhook / button      ┌──────────────┐
│  Streamer   │ ────────────────────────▶ │  Backend API │
│  Dashboard  │     "Show product X"      │  Lambda/API  │
└─────────────┘                           └──────┬───────┘
                                                   │
                                          PutMetadata
                                                   ▼
                                            IVS Channel
```

Streamer app **không** gọi trực tiếp từ browser (lộ IAM). Luồng chuẩn: dashboard → backend có credential → IVS.

---

## 3. Luồng End-to-End

```
T0: Streamer nói "Sản phẩm A"
T1: Backend nhận event → PutMetadata({ type: SHOW_PRODUCT, id: A })
T2: IVS nhúng metadata vào ingest pipeline
T3: Segment/chunk chứa metadata đi qua CDN
T4: Player SDK parse → fire Metadata event
T5: App UI hiện card sản phẩm A

Độ trễ T1→T5 ≈ glass-to-glass latency (LOW ~3-5s, NORMAL ~15s)
```

### Sơ Đồ

```
Backend ──PutMetadata──▶ IVS Ingest Pipeline
                              │
                              ▼ (mux vào stream)
                         IVS CDN
                              │
                              ▼
                    IVS Player SDK
                              │
                              ▼
                         onMetadata → React/Vue UI
```

---

## 4. Tích Hợp Player SDK

### Web (IVS Player SDK for Web)

```javascript
import { create } from 'amazon-ivs-player';

const player = create();
player.attachHTMLVideoElement(document.getElementById('video'));

player.addEventListener('PlayerEventType.TEXT_METADATA_CUE', (cue) => {
  const payload = JSON.parse(cue.text);
  if (payload.type === 'SHOW_PRODUCT') {
    showProductCard(payload.id);
  }
});

player.load('https://xxx.playback.live-video.net/.../index.m3u8');
player.play();
```

> Event name có thể theo phiên bản SDK — tham khảo [Player Events](https://docs.aws.amazon.com/ivs/latest/LowLatencyUserGuide/player-events.html).

### Mobile

- **iOS:** `IVSPlayer` delegate `player(_:didOutputCue:)`
- **Android:** `Player.Listener` metadata callbacks

Cùng nguyên tắc: parse JSON từ cue → cập nhật UI thread-safe.

### Đồng Bộ Với Chat

```
Timed Metadata:  "Sản phẩm A lên sóng" — mọi viewer cùng thấy UI (trễ video)
Chat:            Viewer hỏi "còn size M không?" — không đảm bảo sync frame

Pattern: Metadata cho hành động host; Chat cho tương tác audience
```

---

## 5. Thiết Kế & Giới Hạn

### Rate Limit

- **PutMetadata** có giới hạn TPS (transactions per second) theo channel — burst quá cao bị throttle
- Thiết kế **debounce** phía backend (ví dụ tối đa 2 metadata/giây cho UI flash)

### Idempotency & Ordering

- Metadata **có thứ tự** theo thời gian gửi
- App nên **ignore duplicate** `id` nếu retry API
- Không dùng metadata cho dữ liệu lớn (ảnh, HTML) — chỉ **reference ID**, client fetch chi tiết qua REST

### Khi Stream Offline

`PutMetadata` trên channel không live → lỗi. Queue event phía backend hoặc báo streamer "chưa live".

### So Với SCTE-35

| | Timed Metadata (IVS) | SCTE-35 (MediaLive) |
|---|---------------------|---------------------|
| Mục đích | UI tương tác app | Ad breaks broadcast |
| API | PutMetadata | Schedule / splice |
| Player | IVS SDK | HLS player + SSAI |

---

## 6. Câu Hỏi Phỏng Vấn

**Q: Timed metadata có đảm bảo tất cả viewer nhận cùng lúc không?**

> Mọi viewer nhận **cùng timeline** trong luồng, nhưng **thời điểm hiển thị** lệch một chút do buffer mạng khác nhau (vài giây). Chấp nhận được cho live shopping; không thay **atomic clock** cho auction pháp lý — dùng **Real-Time Stage** nếu cần.

**Q: Có thể gửi metadata từ OBS không?**

> IVS **PutMetadata** là API server-side. OBS không inject trực tiếp; streamer bấm hotkey → app gọi backend → PutMetadata. Một số team dùng **on-stream-data** RTMP (không phải IVS native) — không chuẩn IVS.

**Q: Metadata có ghi vào recording S3 không?**

> Recording HLS có thể chứa cue tương ứng — replay VOD cần player xử lý metadata cue tương tự live. Kiểm tra phiên bản SDK và format recording khi build replay feature.

---

**Cập Nhật Lần Cuối:** 2026-06-04
**Xem thêm:** [4-ivs-chat.md](./4-ivs-chat.md) — chat song song với metadata
