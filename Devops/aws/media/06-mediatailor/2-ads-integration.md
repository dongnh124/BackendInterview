# MediaTailor — Tích Hợp ADS (Ad Decision Server)

> ADS — Ad Decision Server — Máy Chủ Ra Quyết Định Quảng Cáo quyết định **quảng cáo nào** hiển thị cho **viewer nào** tại **thời điểm nào**. MediaTailor gọi ADS mỗi khi gặp avail, nhận response **VAST** / **VMAP** và stitch creative vào manifest.

## 📚 Mục Lục

1. [ADS Trong Kiến Trúc SSAI](#1-ads-trong-kiến-trúc-ssai)
2. [VAST / VMAP / VPAID](#2-vast--vmap--vpaid)
3. [Session Parameters & Macros](#3-session-parameters--macros)
4. [Tích Hợp ADS Phổ Biến](#4-tích-hợp-ads-phổ-biến)
5. [Transcode Ad Creative](#5-transcode-ad-creative)
6. [Xử Lý Lỗi & Fill Rate](#6-xử-lý-lỗi--fill-rate)
7. [Thực Hành & Debug](#7-thực-hành--debug)
8. [Câu Hỏi Phỏng Vấn](#8-câu-hỏi-phỏng-vấn)

---

## 1. ADS Trong Kiến Trúc SSAI

```
┌─────────────┐     avail detected      ┌─────────────┐
│ MediaTailor │ ────────────────────────▶ │     ADS     │
│             │  GET + session params   │  (Google /  │
│             │ ◀────────────────────── │   FreeWheel │
│             │  VAST XML + Media URLs  │   custom)   │
└─────────────┘                         └─────────────┘
       │
       ▼ download + transcode ad MP4/HLS
┌─────────────┐
│ CDN origin  │  ad segments cached
└─────────────┘
```

### Trách Nhiệm Phân Tách

| Thành phần | Trách nhiệm |
|------------|-------------|
| **ADS** | Targeting, pacing, frequency cap, chọn creative, trả VAST |
| **MediaTailor** | Gọi ADS, parse VAST, transcode, stitch manifest, fire beacons |
| **Ad CDN / S3** | Lưu creative gốc (thường do ad network host) |
| **CloudFront** | Phân phối stitched stream tới viewer |

---

## 2. VAST / VMAP / VPAID

### 2.1 VAST — Video Ad Serving Template

**VAST** là XML mô tả quảng cáo video: duration, MediaFile URL, tracking events.

```xml
<?xml version="1.0" encoding="UTF-8"?>
<VAST version="4.2">
  <Ad id="ad-12345">
    <InLine>
      <AdSystem>Example ADS</AdSystem>
      <AdTitle>30s Pre-roll</AdTitle>
      <Impression>https://ads.example.com/track/impression?id=12345</Impression>
      <Creatives>
        <Creative>
          <Linear>
            <Duration>00:00:30</Duration>
            <MediaFiles>
              <MediaFile delivery="progressive" type="video/mp4" width="1920" height="1080">
                https://cdn.ads.example.com/creative/30s-hd.mp4
              </MediaFile>
            </MediaFiles>
            <TrackingEvents>
              <Tracking event="firstQuartile">https://ads.example.com/track/q1</Tracking>
              <Tracking event="midpoint">https://ads.example.com/track/mid</Tracking>
              <Tracking event="thirdQuartile">https://ads.example.com/track/q3</Tracking>
              <Tracking event="complete">https://ads.example.com/track/complete</Tracking>
            </TrackingEvents>
          </Linear>
        </Creative>
      </Creative>
    </InLine>
  </Ad>
</VAST>
```

MediaTailor đọc `MediaFile` → tải creative → transcode nếu cần → chèn vào HLS playlist.

### 2.2 VMAP — Video Multiple Ad Playlist

**VMAP** định nghĩa **vị trí** và **loại** ad break trên timeline (pre-roll, mid-roll, post-roll):

```xml
<vmap:VMAP xmlns:vmap="http://www.iab.net/videosuite/vmap" version="1.0">
  <vmap:AdBreak timeOffset="start" breakType="linear" breakId="preroll-1">
    <vmap:AdSource id="preroll" allowMultipleAds="false">
      <vmap:AdTagURI templateType="vast3">
        <![CDATA[https://ads.example.com/vast?type=preroll&session=[session]]]>
      </vmap:AdTagURI>
    </vmap:AdSource>
  </vmap:AdBreak>
  <vmap:AdBreak timeOffset="00:15:00.000" breakType="linear" breakId="midroll-1">
    ...
  </vmap:AdBreak>
</vmap:VMAP>
```

Dùng nhiều cho **VOD SSAI** khi manifest gốc không có SCTE-35 markers.

### 2.3 VPAID — Video Player-Ad Interface Definition

**VPAID** — quảng cáo tương tác chạy trong player (HTML/JS overlay). MediaTailor SSAI **không hỗ trợ VPAID** trực tiếp vì stitch yêu cầu file video cố định (MP4/HLS). Interactive ads cần CSAI hoặc giải pháp hybrid.

| Định dạng | MediaTailor SSAI | Ghi chú |
|-----------|------------------|---------|
| VAST 2.x/3.x/4.x (Linear MP4/HLS) | ✅ Hỗ trợ | Phổ biến nhất |
| VMAP | ✅ (VOD) | Định nghĩa break positions |
| VPAID | ❌ | Dùng CSAI hoặc non-linear overlay |
| Wrapper VAST | ✅ | ADS chain nhiều tầng (redirect) |

---

## 3. Session Parameters & Macros

### 3.1 Placeholder Trong AdDecisionServerUrl

```text
https://ads.example.com/vast?
  session=[session]
  &app=[player_params.app_bundle]
  &geo=[player_params.geo]
  &content=[player_params.content_id]
  &break=[avail.duration]
```

| Macro | Nguồn |
|-------|-------|
| `[session]` | MediaTailor session ID |
| `[player_params.*]` | Trường gửi trong Session Initialization |
| `[avail.*]` | Metadata avail hiện tại (duration, type) |

### 3.2 Tham Số ADS Điển Hình (OTT)

```json
{
  "playerParams": {
    "app_bundle": "com.ott.app",
    "device_type": "firetv",
    "user_id": "hashed-user-id",
    "content_id": "episode-s01e05",
    "content_genre": "sports",
    "gdpr_consent": "1",
    "us_privacy": "1YNN",
    "ifa": "advertising-id-optional",
    "width": "1920",
    "height": "1080"
  }
}
```

> **Privacy:** Không gửi PII — Personal Identifiable Information thô; hash `user_id`, tuân thủ GDPR/CCPA. ADS và legal team định nghĩa whitelist params.

### 3.3 Live vs VOD ADS Request

| Loại stream | ADS trigger |
|-------------|-------------|
| **Live** | Mỗi SCTE-35 / CUE-OUT avail → realtime VAST request |
| **VOD** | Mỗi VMAP break hoặc embedded avail trong manifest |

Live ADS phải respond **< 500ms** để tránh buffer underrun.

---

## 4. Tích Hợp ADS Phổ Biến

### 4.1 Custom ADS (Tự Xây)

```
API Gateway / ALB → Lambda hoặc microservice
  Input: session, content_id, avail_duration, geo
  Logic: frequency cap, campaign rules, A/B test
  Output: VAST XML (MediaFile URLs trên S3/CloudFront)
```

Phù hợp FAST platform nội bộ, house ads + programmatic waterfall.

### 4.2 Third-Party SSP/DSP

Tích hợp qua ad server hỗ trợ server-side:

- **Google Ad Manager** (với partner setup SSAI)
- **FreeWheel**, **Magnite**, **SpotX** (broadcast/OTT)

Yêu cầu: ADS endpoint trả VAST linear, latency SLA, hỗ trợ macros MediaTailor.

### 4.3 Waterfall (Thác Quảng Cáo)

```
1. Gọi Premium ADS (CPM cao)
   └── Empty? → 2. Gọi Programmatic ADS
                  └── Empty? → 3. House ad (slate / promo nội bộ)
```

Implement ở tầng ADS (không phải MediaTailor). MediaTailor chỉ nhận **một** VAST response cuối cùng mỗi avail.

---

## 5. Transcode Ad Creative

### 5.1 Tại Sao Cần Transcode?

Content ladder: 1080p / 720p / 480p / 360p. ADS thường trả **một** MP4 1080p. MediaTailor **transcode** xuống các rendition để ABR không bị switch lỗi khi vào ad break.

### 5.2 TranscodeProfile

```json
"TranscodeProfileName": "ott-ad-profile-720p"
```

Profile định nghĩa codec (H.264), max bitrate, resolution cap. Chi phí transcode tính vào MediaTailor workload.

### 5.3 Best Practice

- ADS trả creative **đúng aspect ratio** content (16:9)
- Chuẩn hoá duration: 15s / 30s / 60s (khớp avail)
- Host creative gần region MediaTailor (giảm tải latency)
- Pre-transcode ladder gửi sẵn trong VAST (nhiều MediaFile) → giảm transcode realtime

---

## 6. Xử Lý Lỗi & Fill Rate

### 6.1 Các Tình Huống Lỗi

| Tình huống | Hành vi MediaTailor |
|------------|---------------------|
| ADS timeout | Slate hoặc bỏ avail (tùy config) |
| VAST rỗng | Slate |
| MediaFile 404 | Slate / skip ad |
| Duration ad > avail | Cắt bớt hoặc slate phần dư |
| Duration ad < avail | Slate pad hoặc nhiều ads từ VAST pod |

### 6.2 Fill Rate (Tỷ Lệ Lấp Đầy Avail)

```
Fill Rate = (avail filled by paid ads) / (total avails) × 100%
```

Monitor qua:

- CloudWatch MediaTailor metrics (`AdDecisionServer.Duration`, errors)
- ADS reporting dashboard
- Beacon `complete` rate vs `impression`

### 6.3 SLA Khuyến Nghị

| Metric | Target production |
|--------|-------------------|
| ADS p95 latency | < 300ms |
| Stitch ready before CUE-OUT | Prefetch + Package delay |
| Paid fill rate | > 85% (tùy thị trường) |
| Slate rate | < 15% |

---

## 7. Thực Hành & Debug

### Test ADS Stub (Local)

```python
# Flask stub trả VAST cố định — dùng staging
from flask import Flask, Response

app = Flask(__name__)

VAST_30S = """<?xml version="1.0"?>
<VAST version="3.0">
  <Ad><InLine>
    <Impression>https://httpbin.org/get?event=impression</Impression>
    <Creatives><Creative><Linear>
      <Duration>00:00:30</Duration>
      <MediaFiles>
        <MediaFile type="video/mp4">
          https://cdn.example.com/test-ad-30s.mp4
        </MediaFile>
      </MediaFiles>
    </Linear></Creative></Creatives>
  </InLine></Ad>
</VAST>"""

@app.route("/vast")
def vast():
    return Response(VAST_30S, mimetype="application/xml")

if __name__ == "__main__":
    app.run(port=8080)
```

Gắn `AdDecisionServerUrl` staging trỏ tới stub, trigger ad break bằng SCTE-35 test.

### CloudWatch Logs

```bash
# Bật logging playback configuration (nếu chưa)
aws mediatailor put-playback-configuration \
  --name ott-live-hls-main \
  --log-configuration PercentEnabled=100,LogDeliveryStreams=[...]
```

Tìm lỗi: `AdDecisionServer`, `Transcode`, `ManifestStitch`.

---

## 8. Câu Hỏi Phỏng Vấn

**Q: MediaTailor có thể hoạt động không có ADS bên ngoài không?**

> Không cho SSAI thực sự — cần nguồn quyết định creative. Có thể dùng **custom ADS** tối giản (Lambda trả VAST cố định) cho house ads. Không có ADS = chỉ slate hoặc passthrough avail trống.

**Q: VAST Wrapper là gì?**

> VAST trả `<Wrapper>` thay vì `<InLine>`: redirect tới ADS khác (chuỗi waterfall). MediaTailor follow wrapper đến **max depth** cấu hình. Quá sâu → timeout → slate.

**Q: Làm sao đồng bộ tổng duration ads với SCTE-35 break 60s?**

> ADS trả **ad pod** nhiều creative (2×30s hoặc 4×15s). MediaTailor cắt/pad để khớp avail. Nên cấu hình ADS rule: "break_duration=60 → trả pod tổng 60s". Slate pad phần thiếu nếu underfill.

---

**Cập Nhật Lần Cuối:** 2026-06-04
**Phần Trước:** [1-ssai-basics.md](./1-ssai-basics.md)
**Phần Tiếp Theo:** [3-channel-assembly.md](./3-channel-assembly.md)
