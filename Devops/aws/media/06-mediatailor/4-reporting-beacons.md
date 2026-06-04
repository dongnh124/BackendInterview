# MediaTailor — Reporting & Beacons (Theo Dõi Quảng Cáo)

> **Beacons** — tín hiệu HTTP tracking — báo cho ADS và analytics biết viewer đã **nhìn thấy** (impression), **xem đến** quartile (25%/50%/75%), hay **hoàn thành** (complete) quảng cáo. MediaTailor proxy và fire beacons trong quá trình SSAI stitch.

## 📚 Mục Lục

1. [Tại Sao Cần Ad Tracking](#1-tại-sao-cần-ad-tracking)
2. [Các Loại Beacon](#2-các-loại-beacon)
3. [Luồng Beacon Trong SSAI](#3-luồng-beacon-trong-ssai)
4. [MediaTailor Reporting Configuration](#4-mediatailor-reporting-configuration)
5. [CloudWatch & Metrics](#5-cloudwatch--metrics)
6. [Tích Hợp Analytics Bên Thứ Ba](#6-tích-hợp-analytics-bên-thứ-ba)
7. [Thực Hành & Troubleshooting](#7-thực-hành--troubleshooting)
8. [Câu Hỏi Phỏng Vấn](#8-câu-hỏi-phỏng-vấn)

---

## 1. Tại Sao Cần Ad Tracking

### Mô Hình Tính Tiền Quảng Cáo

| Metric | Ý nghĩa kinh doanh |
|--------|-------------------|
| **Impression** | Ad đã được "deliver" — thường tính CPM — Cost Per Mille (chi phí mỗi 1000 lần hiển thị) |
| **Quartile** | Viewer thực sự xem (chống gian lận viewability) |
| **Complete** | View-through — quan trọng cho brand campaigns |
| **Click** (nếu có) | Performance ads — CSAI phổ biến hơn SSAI |

Không có beacon chính xác → không chứng minh delivery → mất doanh thu quảng cáo.

### SSAI vs CSAI Tracking

```
CSAI:  Player SDK fire beacon trực tiếp tới ADS
SSAI:  MediaTailor fire beacon khi stitch/serve segment
       (player có thể fire thêm — dual tracking cần dedupe)
```

---

## 2. Các Loại Beacon

### 2.1 VAST Tracking Events

Trong VAST `<TrackingEvents>`:

| Event | Thời điểm |
|-------|------------|
| `start` | Bắt đầu phát ad |
| `firstQuartile` | 25% duration |
| `midpoint` | 50% |
| `thirdQuartile` | 75% |
| `complete` | 100% |
| `mute` / `unmute` | Tương tác (ít dùng SSAI TV) |
| `skip` | Bỏ qua (nếu skippable) |

### 2.2 Impression URL

```xml
<Impression>https://ads.example.com/pixel?event=impression&cid=123</Impression>
```

Fire **một lần** khi ad bắt đầu được coi là delivered (MediaTailor khi ad segment vào playlist).

### 2.3 MediaTailor Beacon Proxy

MediaTailor có thể **rewrite** beacon URLs qua endpoint proxy:

```
Gốc:  https://ads.example.com/track/complete?id=1
Proxy: https://logs.mediatailor.region.amazonaws.com/v1/tracking/...?redirect=...
```

Lợi ích: ghi log tập trung, retry, correlate với `sessionId`.

---

## 3. Luồng Beacon Trong SSAI

```
1. ADS trả VAST kèm Impression + TrackingEvents URLs
2. MediaTailor stitch ad vào manifest
3. Player request ad segment từ CDN
4. MediaTailor (hoặc player theo config) fire:
   - Impression khi ad segment bắt đầu được serve
   - Quartile theo playback progress
   - Complete khi hết ad segment
5. ADS / analytics nhận HTTP GET pixel
```

### Timeline 30s Ad

```
0s     ── Impression + start
7.5s   ── firstQuartile (25%)
15s    ── midpoint (50%)
22.5s  ── thirdQuartile (75%)
30s    ── complete
```

> **Viewability:** Một số chuẩn IAB yêu cầu 50% pixels visible ≥ 2 giây — TV/OTT SSAI thường coi fullscreen playback = viewable.

---

## 4. MediaTailor Reporting Configuration

### 4.1 Playback Configuration Reporting

Gắn **Reporting Configuration** với Playback Configuration:

```json
{
  "Name": "ott-reporting-main",
  "ReportingConfiguration": {
    "ReportingConfigurationName": "ott-reporting-main"
  }
}
```

### 4.2 Các Chế Độ Reporting

| Tính năng | Mô tả |
|-----------|-------|
| **AWS reporting** | Metrics trong CloudWatch (AdDelivery, etc.) |
| **CDN reporting** | Tích hợp CloudFront real-time logs |
| **Client-side reporting** | Player gửi thêm events (supplement) |

### 4.3 Ví Dụ Tạo Reporting Config (CLI)

```bash
aws mediatailor put-reporting-configuration \
  --reporting-configuration-name ott-reporting-main \
  --region ap-southeast-1

aws mediatailor put-playback-configuration \
  --name ott-live-hls-main \
  --reporting-configuration-name ott-reporting-main \
  --region ap-southeast-1
  # ... các trường khác giữ nguyên
```

### 4.4 Correlation Với Session

Mọi beacon/event nên gắn:

- `sessionId` — phiên MediaTailor
- `availId` — avail cụ thể trong break
- `creativeId` — từ VAST Ad `@id`

Giúp debug: "viewer X thấy creative Y tại break Z".

---

## 5. CloudWatch & Metrics

### Metrics MediaTailor Quan Trọng

| Metric | Ý nghĩa |
|--------|---------|
| `AdDecisionServer.Latency` | Thời gian gọi ADS |
| `AdDecisionServer.Errors` | ADS fail / timeout |
| `AdDelivery` | Số ad insertion thành công |
| `Avail.Duration` | Thời lượng avail xử lý |
| `Manifest.Stitch.Errors` | Lỗi ghép manifest |

### Alarm Khuyến Nghị

```yaml
# Ví dụ ý tưởng CloudWatch Alarm
- ADS error rate > 5% trong 5 phút → PagerDuty
- AdDecisionServer p95 > 1s → Warning
- Manifest stitch errors > 0 → Critical (live event)
```

### Logs

Bật log delivery tới Kinesis Firehose → S3 → Athena cho phân tích ad fill theo campaign.

---

## 6. Tích Hợp Analytics Bên Thứ Ba

### 6.1 ADS-Side Reporting

Hầu hết doanh thu báo cáo từ **ADS dashboard** (Google Ad Manager, FreeWheel) — nhận beacon từ MediaTailor/proxy.

### 6.2 Data Warehouse Pipeline

```
MediaTailor logs / CloudFront logs
        │
        ▼
   Kinesis Firehose
        │
        ▼
   S3 (Parquet) → Glue → Athena / Redshift
        │
        ▼
   BI dashboard: fill rate, eCPM, by channel/geo
```

### 6.3 Dedupe Dual Tracking

Nếu **vừa** SSAI beacon **vừa** player SDK:

- Chỉ tin một nguồn cho billing
- Hoặc dedupe bằng `sessionId + event + timestamp` trong ETL

---

## 7. Thực Hành & Troubleshooting

### Kiểm Tra Beacon Reach ADS

```bash
# Trong staging, dùng ADS trả beacon tới httpbin
<Impression>https://httpbin.org/get?evt=impression</Impression>

# Sau ad break test, query httpbin logs hoặc webhook.site
```

### Vấn Đề Thường Gặp

| Triệu chứng | Nguyên nhân | Khắc phục |
|-------------|-------------|-----------|
| Impression cao, complete thấp | Viewer skip / buffer | Kiểm tra ad transcode, bitrate |
| Không có impression | Beacon URL sai, bị firewall | Validate VAST XML |
| Double impression | Player + SSAI cùng fire | Tắt một phía |
| Quartile lệch thời gian | Player seek trong ad | Disable seek trong ad break (player policy) |

### Player Policy Cho OTT

```javascript
// Khuyến nghị UX — pseudo-code
player.on('ad-break-start', () => {
  disableSeek(true);
  disablePause(false); // tùy product
});
player.on('ad-break-end', () => {
  disableSeek(false);
});
```

---

## 8. Câu Hỏi Phỏng Vấn

**Q: Ai fire beacon trong SSAI — player hay MediaTailor?**

> Chủ yếu **MediaTailor** khi serve stitched segments (server-side delivery proof). Một số setup **bổ sung** player beacons cho tương tác. Production nên thống nhất một nguồn billing để tránh double-count.

**Q: Impression và Complete khác gì?**

> **Impression**: ad đã được inserted và bắt đầu delivery (đầu break). **Complete**: viewer xem hết 100% duration (hoặc theo rule viewability). CPM thường tính impression; brand campaigns optimize complete/view-through rate.

**Q: Làm sao audit fill rate cho live event?**

> So sánh: (số SCTE-35 avails từ MediaLive logs) vs (AdDelivery metric + impression beacons). Gap → ADS timeout, slate rate cao, hoặc stitch errors. Dashboard real-time trong event operations center.

---

**Cập Nhật Lần Cuối:** 2026-06-04
**Phần Trước:** [3-channel-assembly.md](./3-channel-assembly.md)
**Phần Tiếp Theo:** [5-prefetch-ads.md](./5-prefetch-ads.md)
