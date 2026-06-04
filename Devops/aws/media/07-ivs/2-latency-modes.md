# Amazon IVS — Latency Modes (Chế Độ Độ Trễ)

> **Latency** (độ trễ) là khoảng thời gian từ khi streamer nói/hành động đến khi viewer nhìn thấy trên màn hình. IVS cung cấp **NORMAL** (chuẩn HLS), **LOW** (low-latency HLS), và **REAL_TIME** (WebRTC qua IVS Stage).

## 📚 Mục Lục

1. [Tổng Quan Ba Mức Độ Trễ](#1-tổng-quan-ba-mức-độ-trễ)
2. [NORMAL — Standard Latency](#2-normal--standard-latency)
3. [LOW — Low-Latency HLS](#3-low--low-latency-hls)
4. [REAL_TIME — WebRTC Stage](#4-real_time--webrtc-stage)
5. [Chọn Mode & Trade-offs](#5-chọn-mode--trade-offs)
6. [Câu Hỏi Phỏng Vấn](#6-câu-hỏi-phỏng-vấn)

---

## 1. Tổng Quan Ba Mức Độ Trễ

```
Timeline (streamer nói "Mua ngay!" → viewer nghe thấy)

NORMAL (~15s):     ████████████████░░░░░░░░  buffer lớn, ổn định CDN
LOW (~3-5s):       ██████░░░░░░░░░░░░░░░░  LL-HLS — Low-Latency HLS
REAL_TIME (<1s):   ██░░░░░░░░░░░░░░░░░░░░  WebRTC — peer/subscriber path
```

| Mode | API `latencyMode` | Công nghệ | Độ trễ điển hình |
|------|-------------------|-----------|------------------|
| Standard | `NORMAL` | HLS CMAF segment dài hơn | 10–20 giây |
| Low | `LOW` | LL-HLS partial segments | 3–5 giây |
| Real-time | Stage (riêng) | WebRTC | < 1 giây |

> **Channel RTMPS** chỉ chọn `NORMAL` hoặc `LOW`. **Real-Time** dùng **IVS Real-Time** (Stage), không set trên cùng channel RTMP.

---

## 2. NORMAL — Standard Latency

### Cách Hoạt Động

**HLS** — HTTP Live Streaming — chia video thành **segments** (thường 2–6 giây). Player buffer vài segment trước khi play → độ trễ = vài × segment duration + CDN + encode.

```
Encoder → IVS → [seg1][seg2][seg3]... → CDN → Player buffer 3 segs → Play
                      └── ~6s mỗi segment → ~15-18s glass-to-glass
```

### Khi Nào Dùng NORMAL

- Không cần tương tác realtime (phát sóng một chiều, delay chấp nhận được)
- Viewer mạng yếu — buffer lớn giảm rebuffer
- Tiết kiệm chi phí bandwidth (ít request partial segment hơn LOW mode)

### Hạn Chế

Chat và **Timed Metadata** vẫn hoạt động nhưng **lệch ~15s** so với hành động streamer — không phù hợp auction countdown chính xác giây.

---

## 3. LOW — Low-Latency HLS

### LL-HLS — Low-Latency HTTP Live Streaming

**LL-HLS** dùng **partial segments** (CMAF chunks nhỏ) và **blocking playlist reload** — player tải manifest thường xuyên hơn, giảm buffer tối thiểu cần thiết.

```
Standard HLS:  [──── 6s segment ────][──── 6s ────]  buffer ≥ 2-3 segments
LL-HLS:        [─1s─][─1s─][─1s─]...               buffer ~ 3-5 giây tổng
```

### Yêu Cầu Player

- Dùng **Amazon IVS Player SDK** (web/iOS/Android) — tối ưu cho LL-HLS IVS
- **HLS.js** generic có thể play nhưng latency và ổn định kém hơn SDK chính thức
- **AVPlayer** (iOS) / **ExoPlayer** (Android) với SDK wrapper IVS

### Cấu Hình Channel

```bash
aws ivs create-channel \
  --latency-mode LOW \
  --type STANDARD \
  ...
```

### Trade-offs LOW Mode

| Ưu | Nhược |
|----|-------|
| Tương tác gần realtime | Viewer mạng kém dễ rebuffer hơn NORMAL |
| Phù hợp live shopping, Q&A | CDN/request manifest tăng → chi phí output có thể cao hơn |
| Vẫn scale hàng triệu viewer | Không đạt <1s như WebRTC |

---

## 4. REAL_TIME — WebRTC Stage

### IVS Real-Time Streaming

**Real-Time** không đi qua pipeline RTMPS → HLS channel. Thay vào đó:

```
Participants (Host, Guest)
        │ WebRTC publish
        ▼
┌───────────────────┐
│  IVS Stage        │  SFU — Selective Forwarding Unit
│  (media server)   │  (chuyển tiếp luồng có chọn lọc)
└─────────┬─────────┘
          │ WebRTC subscribe
          ▼
    Viewers / Co-hosts (< 1s latency)
```

**WebRTC** — Web Real-Time Communication — giao thức P2P/browser realtime; IVS quản lý signaling và media server.

### Khái Niệm Stage

| Tài nguyên | Mô tả |
|------------|-------|
| **Stage** | Không gian session realtime |
| **Participant token** | Quyền publish/subscribe |
| **Participant** | User với role publisher hoặc subscriber |
| **Composition** (tuỳ phiên bản) | Layout nhiều video (grid, spotlight) |

### Use Cases

- Panel thảo luận nhiều người camera
- Game show với host + khách mời
- Đấu giá / bidding cần đồng bộ sub-second
- Remote production (thay Zoom embed)

### So Sánh LOW vs REAL_TIME

| | LOW (HLS) | REAL_TIME (WebRTC) |
|---|-----------|-------------------|
| Ingest | OBS RTMPS | Browser/SDK WebRTC |
| Scale viewer | Rất cao | Cao (nhưng khác kiến trúc SFU) |
| Latency | 3–5s | < 1s |
| Độ phức tạp app | Trung bình (1 publisher) | Cao (multi participant) |
| Recording | S3 HLS recording channel | Cần thiết kế riêng / composite |

---

## 5. Chọn Mode & Trade-offs

### Decision Tree

```
Cần độ trễ < 1 giây?
├── Có → IVS Real-Time (Stage) + WebRTC
└── Không
    └── Cần tương tác chat/metadata < 10s?
        ├── Có → LOW + IVS Player SDK
        └── Không → NORMAL (ổn định nhất)
```

### Ảnh Hưởng Tới Timed Metadata & Chat

```
LOW mode:     Metadata event ~ cùng độ trễ video (3-5s)
NORMAL mode:  Metadata lệch ~15s — vẫn đúng thứ tự trong timeline
REAL_TIME:    Metadata/API riêng — thiết kế app sync qua Stage events
```

### MediaLive So Sánh Latency

```
MediaLive + MediaPackage (HLS 6s):     15-30+ giây
IVS LOW:                              3-5 giây
IVS Real-Time:                        < 1 giây
```

Không thể chỉ "bật low latency" trên MediaPackage tương đương IVS LOW mà không thiết kế CMAF/LL-HLS end-to-end — lý do nhiều app interactive chọn IVS.

---

## 6. Câu Hỏi Phỏng Vấn

**Q: Tại sao LOW latency vẫn dùng HLS mà không WebRTC cho mọi viewer?**

> **HLS** scale CDN cực tốt (HTTP cache, millions viewers), OBS ingest chuẩn. **WebRTC** tốt cho latency nhưng mỗi viewer có chi phí media server khác HLS broadcast. IVS tách: **LOW** = HLS scale + latency chấp nhận được; **Real-Time** = WebRTC khi cần sub-second.

**Q: Đổi latency mode sau khi tạo channel?**

> Có thể **update channel** `latencyMode` giữa NORMAL và LOW (API `update-channel`). Stream đang live có thể cần restart ingest để viewer nhận hiệu ứng đầy đủ. Không đổi RTMP channel thành Stage — hai sản phẩm khác nhau.

**Q: Viewer ở VN xem IVS region Tokyo — latency thêm bao nhiêu?**

> Thêm **RTT** mạng (50–100ms+) cộng buffer mode. Chọn **region** IVS gần streamer ingest và đa số viewer; IVS CDN global nhưng glass-to-glass vẫn chịu geography.

---

**Cập Nhật Lần Cuối:** 2026-06-04
**Xem thêm:** [3-timed-metadata.md](./3-timed-metadata.md) — đồng bộ UI với LOW mode
