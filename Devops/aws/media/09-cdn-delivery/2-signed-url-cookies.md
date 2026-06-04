# Signed URL & Signed Cookies — Bảo Vệ Nội Dung Trả Phí

> **Trusted signers** — người ký tin cậy — và **Key Group** cho phép CloudFront chỉ phục vụ nội dung khi request có **chữ ký hợp lệ**. Bài này so sánh **Signed URL** vs **Signed Cookies**, cách tích hợp backend OTT, và lưu ý với player HLS/DASH.

## 📚 Mục Lục

1. [Mô Hình Private Content](#1-mô-hình-private-content)
2. [Key Pair & Key Group](#2-key-pair--key-group)
3. [Signed URL](#3-signed-url)
4. [Signed Cookies](#4-signed-cookies)
5. [Custom Policy vs Canned Policy](#5-custom-policy-vs-canned-policy)
6. [Tích Hợp Player & Backend](#6-tích-hợp-player--backend)
7. [Câu Hỏi Phỏng Vấn](#7-câu-hỏi-phỏng-vấn)

---

## 1. Mô Hình Private Content

### 1.1 Vấn Đề

S3 + CloudFront với OAC chặn truy cập **trực tiếp S3**, nhưng URL CloudFront công khai (`https://dxxx.cloudfront.net/movie/index.m3u8`) vẫn ai cũng gọi được nếu biết path.

**Private content** thêm lớp: CloudFront kiểm tra **chữ ký** trên URL hoặc **cookie** trước khi trả object.

```
[User login] → [Backend API] → ký URL/cookie với private key
                                    │
                                    ▼
[Player] ──GET manifest + segments──▶ [CloudFront] ──verify signature──▶ Origin
```

### 1.2 Khi Nào Cần (Ngoài DRM)

| Cơ chế | Bảo vệ gì |
|---------|-----------|
| **DRM** (Widevine, FairPlay) | Mã hoá **nội dung** segment, license server |
| **Signed URL/Cookies** | Ai được **request** qua CDN (authorization layer) |
| **OAC** | Ai được đọc **S3** (infrastructure layer) |

Thực tế OTT trả phí thường dùng **cả ba**: OAC + signed cookies + DRM.

---

## 2. Key Pair & Key Group

### 2.1 CloudFront Key Pair (Legacy)

```
1. Tài khoản root AWS tạo key pair (chỉ root) → lưu private key .pem
2. Upload public key lên CloudFront "Trusted Key Groups"
3. Gắn Key Group vào cache behavior → "Restrict viewer access"
```

> **Khuyến nghị hiện đại:** dùng **Key Group** với public key; private key lưu **Secrets Manager** / HSM — Hardware Security Module, không commit Git.

### 2.2 Key Group

```
Key Group: ott-playback-keys
├── Public key ID: APKAxxxxxxxx
└── Private key: trong Lambda / ECS / backend (không trên CloudFront)

Distribution behavior /premium/*:
  Trusted key groups: ott-playback-keys
  Trusted signers: (deprecated — dùng key groups)
```

### 2.3 IAM vs Signing Key

| Thành phần | Mục đích |
|------------|----------|
| **IAM** | Admin tạo distribution, quản lý AWS API |
| **CloudFront signing key** | Ký URL/cookie cho **viewer** request |

Không nhầm IAM credential với CloudFront private key.

---

## 3. Signed URL

### 3.1 Cách Hoạt Động

Backend ký URL đầy đủ; viewer dùng đúng URL đó:

```
https://d111111abcdef8.cloudfront.net/movies/ep1/index.m3u8
  ?Expires=1718000000
  &Signature=Base64EncodedRSA==
  &Key-Pair-Id=APKAxxxxxxxx
```

CloudFront verify: thời gian chưa hết, chữ ký khớp resource path, key pair được trust.

### 3.2 Canned Policy (Chính Sách Đóng Hộp)

Giới hạn **một resource** cụ thể:

```json
{
  "Statement": [{
    "Resource": "https://d111111abcdef8.cloudfront.net/movies/ep1/index.m3u8",
    "Condition": {
      "DateLessThan": { "AWS:EpochTime": 1718000000 }
    }
  }]
}
```

Phù hợp: link xem một tập phim, email marketing, download một file.

### 3.3 Custom Policy (Chính Sách Tùy Chỉnh)

Cho phép **wildcard resource**:

```json
{
  "Statement": [{
    "Resource": "https://d111111abcdef8.cloudfront.net/movies/ep1/*",
    "Condition": {
      "DateLessThan": { "AWS:EpochTime": 1718000000 },
      "IpAddress": { "AWS:SourceIp": "203.0.113.0/24" }
    }
  }]
}
```

Một signed URL (custom policy) có thể cover **toàn bộ segment** trong thư mục `ep1/`.

### 3.4 Ưu / Nhược Signed URL

| Ưu | Nhược |
|----|-------|
| Đơn giản, một link chia sẻ | Player HLS request **hàng trăm URL** segment — phải ký wildcard hoặc chuyển sang cookies |
| Kiểm soát IP trong policy | Query string dài → cache key phức tạp |
| Phù hợp API trả link ngắn hạn | Rotate key phải deploy backend ký mới |

---

## 4. Signed Cookies

### 4.1 Ba Cookie CloudFront

| Cookie | Nội dung |
|--------|----------|
| `CloudFront-Policy` | Base64 policy JSON |
| `CloudFront-Signature` | Chữ ký policy |
| `CloudFront-Key-Pair-Id` | ID public key |

Set domain cookie: `.example.com` → mọi request tới `https://video.example.com/...` mang cookie.

### 4.2 Custom Policy Cho Cookies

```json
{
  "Statement": [{
    "Resource": "https://d111111abcdef8.cloudfront.net/premium/*",
    "Condition": {
      "DateLessThan": { "AWS:EpochTime": 1718000000 }
    }
  }]
}
```

Player request:
- `https://video.example.com/premium/show1/index.m3u8`
- `https://video.example.com/premium/show1/720p/seg001.ts`

Cùng cookie → không cần ký từng segment.

### 4.3 CORS & Cookies

```
CloudFront CORS phải:
  Access-Control-Allow-Credentials: true
  Access-Control-Allow-Origin: https://app.example.com (không dùng *)

Player fetch manifest:
  credentials: 'include'   // HLS.js / fetch API
```

### 4.4 Ưu / Nhược Signed Cookies

| Ưu | Nhược |
|----|-------|
| Tốt cho web OTT nhiều segment | Mobile native app phải quản lý cookie jar |
| Giảm tải backend (ký một lần sau login) | Subdomain phải cấu hình đúng |
| Cache behavior dùng chung path | Logout phải clear cookie + invalidate session |

---

## 5. Custom Policy vs Canned Policy

| Tiêu chí | Canned | Custom |
|----------|--------|--------|
| Số resource | Một URL chính xác | Wildcard `*` |
| IP restriction | Không | Có |
| Dùng với cookies | Không (chỉ URL) | Có |
| Độ phức tạp | Thấp | Trung bình |

**OTT HLS/DASH:** hầu hết dùng **custom policy + signed cookies** cho thư mục nội dung.

---

## 6. Tích Hợp Player & Backend

### 6.1 Luồng Đăng Nhập OTT

```
1. User POST /auth/login → JWT session
2. Backend kiểm tra subscription active
3. Backend tạo CloudFront signed cookies (TTL = session hoặc 24h)
4. Set-Cookie trên response API (hoặc redirect page)
5. Web player load https://video.example.com/.../index.m3u8 với credentials
6. CloudFront validate → origin S3 (OAC)
```

### 6.2 Ví Dụ Ký Cookie (Node.js — aws-cloudfront-sign)

```javascript
const cf = require('aws-cloudfront-sign');

const policy = JSON.stringify({
  Statement: [{
    Resource: 'https://d111111abcdef8.cloudfront.net/premium/*',
    Condition: {
      DateLessThan: { 'AWS:EpochTime': Math.floor(Date.now() / 1000) + 3600 }
    }
  }]
});

const cookies = cf.getSignedCookies(policy, {
  keypairId: process.env.CF_KEY_PAIR_ID,
  privateKeyString: process.env.CF_PRIVATE_KEY
});

// res.cookie('CloudFront-Policy', cookies['CloudFront-Policy'], { domain: '.example.com', httpOnly: true, secure: true });
```

### 6.3 MediaPackage CDN Authorization

MediaPackage có cơ chế riêng: CloudFront gửi **header/signature** được MediaPackage trust — **bổ sung** cho signed cookies phía viewer. Xem [04-mediapackage/1-channels-endpoints.md](../04-mediapackage/1-channels-endpoints.md).

### 6.4 Thời Gian Hết Hạn (Expires)

| TTL | Use case |
|-----|----------|
| 5–15 phút | Signed URL one-time playback |
| 1–24 giờ | Session cookie subscriber |
| Vài ngày | Offline download (hiếm qua CDN) |

**Clock skew:** cho phép lệch vài phút giữa server ký và edge.

### 6.5 Key Rotation

```
1. Tạo public key mới trong Key Group (giữ key cũ)
2. Deploy backend ký bằng private key mới
3. Sau TTL cookie cũ hết hạn → remove public key cũ
```

---

## 7. Câu Hỏi Phỏng Vấn

**Q: Signed URL hay Signed Cookies cho app Netflix-style?**

> **Signed Cookies** cho web player vì hàng trăm request segment `.ts`/`.m4s` dùng cùng path prefix. **Signed URL** cho mobile nếu app nhận một **playback URL** đã ký sẵn từ API (custom policy wildcard một show).

**Q: Signed URL có thay DRM không?**

> **Không.** Signed URL chỉ chặn download qua CDN không có chữ ký; user vẫn có thể re-share URL trong thời hạn Expires. **DRM** mã hoá bitstream, key qua license server, chống capture chất lượng cao hơn.

**Q: Cache CloudFront với signed URL thế nào?**

> Chữ ký trong **query string** → mỗi URL khác nhau = cache key khác → hit rate thấp hơn. **Signed cookies** giữ URL sạch → cache segment hiệu quả hơn. Cân nhắc bảo mật vs hiệu năng cache.

**Q: Lambda@Edge có ký URL realtime được không?**

> Có thể generate tại **viewer-request**, nhưng private key tại edge phức tạp bảo mật. Thường ký tại **backend** hoặc Lambda API sau auth; edge chỉ **validate token** tùy chỉnh (JWT) nếu không dùng native CloudFront signing.

---

**Cập Nhật Lần Cuối:** 2026-06-04
**Xem thêm:** [3-oac-origin-security.md](./3-oac-origin-security.md) — OAC S3
