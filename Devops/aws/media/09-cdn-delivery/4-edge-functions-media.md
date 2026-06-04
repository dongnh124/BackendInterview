# Edge Functions Cho Media — CloudFront Functions & Lambda@Edge

> Xử lý tại **edge** — điểm biên CDN — cho phép rewrite URL manifest, chặn geo, inject header, hoặc validate token mà không round-trip về origin. Bài này so sánh **CloudFront Functions** và **Lambda@Edge**, với pattern thực tế cho HLS/DASH.

## 📚 Mục Lục

1. [Tại Sao Cần Edge Compute Cho Media](#1-tại-sao-cần-edge-compute-cho-media)
2. [CloudFront Functions vs Lambda@Edge](#2-cloudfront-functions-vs-lambdaedge)
3. [Viewer Request / Response](#3-viewer-request--response)
4. [Pattern Media Thường Dùng](#4-pattern-media-thường-dùng)
5. [Giới Hạn & Chi Phí](#5-giới-hạn--chi-phí)
6. [Ví Dụ Code](#6-ví-dụ-code)
7. [Câu Hỏi Phỏng Vấn](#7-câu-hỏi-phỏng-vấn)

---

## 1. Tại Sao Cần Edge Compute Cho Media

### 1.1 Vấn Đề Không Xử Lý Tại Edge

```
Yêu cầu: chặn viewer EU xem nội dung chỉ licensed US
  Không edge: mọi request vẫn hit origin → tốn latency + origin load

Yêu cầu: /hls/device-mobile/index.m3u8 vs /hls/device-tv/index.m3u8
  Không edge: duplicate toàn bộ asset hoặc logic player phức tạp

Yêu cầu: validate JWT ngắn trước khi trả segment
  Không edge: API gateway riêng → thêm hop
```

### 1.2 Vị Trí Trong CloudFront

```
Viewer Request
      │
      ▼
┌─────────────────────┐
│ CloudFront Function │  ← viewer-request (nhẹ, <1ms)
│ hoặc Lambda@Edge    │
└──────────┬──────────┘
           ▼
      [Cache hit?]
           │
     miss  ▼
      Origin (S3 / MediaPackage)
           │
           ▼
┌─────────────────────┐
│ Lambda@Edge         │  ← origin-response (sửa header)
│ (optional)          │
└──────────┬──────────┘
           ▼
      Viewer Response
```

---

## 2. CloudFront Functions vs Lambda@Edge

| Tiêu chí | CloudFront Functions | Lambda@Edge |
|----------|---------------------|-------------|
| **Runtime** | JavaScript subset (nhẹ) | Node.js / Python đầy đủ |
| **Độ trễ** | Sub-millisecond | Tens–hundreds ms (cold start) |
| **Kích thước code** | <10 KB | 50 MB (zipped) |
| **Network** | Không gọi internet | Có (VPC, HTTP call) |
| **Trigger** | viewer-request, viewer-response | viewer-request/response, origin-request/response |
| **Giá** | Rẻ hơn, per-invocation thấp | Đắt hơn + duration |
| **Use case media** | URL rewrite, geo block đơn giản | JWT verify phức tạp, A/B, origin selection |

**Quy tắc chọn:** logic đơn giản, không cần thư viện ngoài → **CloudFront Functions**; cần crypto phức tạp, call API → **Lambda@Edge**.

---

## 3. Viewer Request / Response

### 3.1 Viewer Request

Chạy **trước** cache lookup:

```
Input:  URI, method, headers, querystring, client IP (geo)
Output: modified request HOẶC generate response (403) ngay tại edge
```

Ví dụ: chặn country, redirect manifest path, thêm query cho origin.

### 3.2 Viewer Response

Chạy **sau** nhận object (từ cache hoặc origin):

```
Sửa response headers:
  Cache-Control, Access-Control-Allow-Origin
  X-Device-Type (custom cho analytics)
```

### 3.3 Origin Request / Response (Chỉ Lambda@Edge)

**Origin-request:** đổi URI gửi tới S3 (path mapping).

**Origin-response:** sửa body (hiếm cho video binary — tránh); thường chỉ header.

> **Lưu ý:** Sửa **body manifest** tại edge có thể phá chữ ký HLS — chỉ làm khi hiểu rõ player.

---

## 4. Pattern Media Thường Dùng

### 4.1 Geo Restriction (Chặn Vùng Địa Lý)

```javascript
// CloudFront Function — viewer-request
function handler(event) {
  var request = event.request;
  var country = request.headers['cloudfront-viewer-country']
    ? request.headers['cloudfront-viewer-country'].value
    : 'XX';
  var allowed = ['US', 'CA', 'GB'];
  if (allowed.indexOf(country) === -1) {
    return {
      statusCode: 403,
      statusDescription: 'Forbidden',
      headers: { 'content-type': { value: 'text/plain' } },
      body: 'Not available in your region'
    };
  }
  return request;
}
```

License OTT thường giới hạn territory — có thể dùng **CloudFront geo restriction** built-in thay code.

### 4.2 Device-Based Manifest Routing

```
Request: /hls/index.m3u8
User-Agent: SmartTV

Function rewrite:
  request.uri = '/hls/tv/index.m3u8'

Request từ mobile:
  request.uri = '/hls/mobile/index.m3u8'
```

Giảm bitrate ladder không cần thiết trên mobile (kết hợp MediaConvert output groups).

### 4.3 Token Validation (Lambda@Edge)

```
1. Viewer gửi Authorization: Bearer <JWT>
2. Lambda@Edge viewer-request verify JWT (secret trong KMS/SSM)
3. Invalid → 401 tại edge (không hit S3)
4. Valid → forward request
```

Thay thế hoặc bổ sung **signed cookies** native CloudFront.

### 4.4 A/B Testing Player / Manifest

```
Hash viewer IP → bucket A hoặc B
Rewrite URI → /experiment-a/index.m3u8
```

Đo QoE — Quality of Experience — qua analytics beacon riêng.

### 4.5 CORS Header Injection (Viewer Response)

Khi origin S3 thiếu CORS đúng:

```javascript
function handler(event) {
  var response = event.response;
  response.headers['access-control-allow-origin'] = { value: 'https://app.example.com' };
  return response;
}
```

### 4.6 Không Nên Làm Tại Edge

| Việc | Lý do |
|------|-------|
| Decrypt DRM segment | Không có key; vi phạm security |
| Transcode / repackage | Dùng MediaConvert/MediaPackage |
| SSAI ad insertion | Dùng MediaTailor |
| Cache manifest dài bằng code hack | Dùng cache policy đúng |

---

## 5. Giới Hạn & Chi Phí

### 5.1 CloudFront Functions Limits

- CPU time: ~1 ms effective
- Không `fetch`, không file system
- Chỉ ES5-compatible subset

### 5.2 Lambda@Edge Limits

- Deploy tại **us-east-1** (bắt buộc cho CloudFront association)
- Replication toàn cầu — update function mất vài phút propagate
- Cold start: tránh cho **mọi** segment request — chỉ manifest hoặc path `/auth/*`

### 5.3 Chi Phí Vận Hành

```
Live event 1M viewer, 10 segment/phút, 2h:
  Invocations = 1M × 10 × 120 = 1.2 tỷ (nếu chạy mỗi segment → QUÁ ĐẮT)

Khuyến nghị:
  • Function chỉ trên behavior *.m3u8 (manifest)
  • Hoặc dùng signed cookies backend, không Lambda mỗi request
```

---

## 6. Ví Dụ Code

### 6.1 Rewrite Path (CloudFront Function)

```javascript
function handler(event) {
  var request = event.request;
  if (request.uri.endsWith('/index.m3u8')) {
    var ua = request.headers['user-agent']
      ? request.headers['user-agent'].value.toLowerCase()
      : '';
    if (ua.indexOf('smart-tv') !== -1) {
      request.uri = request.uri.replace('/index.m3u8', '/tv/index.m3u8');
    }
  }
  return request;
}
```

### 6.2 Associate Function (CLI)

```bash
aws cloudfront update-distribution \
  --id E1234567890ABC \
  --if-match ETAGVALUE \
  --distribution-config file://dist-config-with-function.json
```

Trong config JSON, behavior chứa:

```json
"FunctionAssociations": {
  "Quantity": 1,
  "Items": [{
    "FunctionARN": "arn:aws:cloudfront::123456789012:function/media-uri-rewrite",
    "EventType": "viewer-request"
  }]
}
```

### 6.3 Lambda@Edge Viewer Request (Pseudo)

```javascript
const jwt = require('jsonwebtoken'); // bundle in deployment package

exports.handler = async (event) => {
  const request = event.Records[0].cf.request;
  const auth = request.headers.authorization;
  if (!auth || !verify(auth[0].value)) {
    return {
      status: '401',
      statusDescription: 'Unauthorized',
      body: 'Invalid token'
    };
  }
  return request;
};
```

Secret lấy từ **SSM Parameter Store** tại deploy time (không hardcode).

---

## 7. Câu Hỏi Phỏng Vấn

**Q: CloudFront Function hay Lambda@Edge cho chặn geo?**

> **Geo restriction** built-in trên distribution đủ cho allow/deny country list. **CloudFront Function** khi cần logic phức tạp hơn (ví dụ allow US trừ một bang). **Lambda@Edge** khi cần check license database — hiếm realtime tại edge.

**Q: Chạy Lambda@Edge trên mọi .ts request có hợp lý không?**

> **Không.** Chi phí và latency nhân với số segment. Validate tại **manifest request** hoặc dùng **signed cookies** cover cả path `*.ts`.

**Q: CloudFront Function thay MediaTailor SSAI?**

> **Không.** SSAI cần gọi ADS — Ad Decision Server, splice segment — quá nặng cho Function. MediaTailor là dịch vụ chuyên biệt.

**Q: Lambda@Edge deploy region nào?**

> Code upload **us-east-1**; AWS replicate function globally cho CloudFront triggers. Không deploy region viewer.

---

**Cập Nhật Lần Cuối:** 2026-06-04
**Xem thêm:** [1-cloudfront-for-media.md](./1-cloudfront-for-media.md) — Cache behaviors
