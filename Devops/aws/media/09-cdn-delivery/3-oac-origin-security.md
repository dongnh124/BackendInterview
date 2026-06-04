# OAC — Origin Access Control — Bảo Mật Origin S3 & Media

> **OAC** — Origin Access Control — Kiểm Soát Truy Cập Origin thay thế **OAI** — Origin Access Identity — cho phép chỉ **CloudFront** đọc object S3, hỗ trợ **SSE-KMS**, và tích hợp chặt với **MediaPackage CDN Authorization**. Bài này hướng dẫn thiết lập và so sánh các mô hình bảo mật origin media.

## 📚 Mục Lục

1. [Tại Sao Cần Bảo Mật Origin](#1-tại-sao-cần-bảo-mật-origin)
2. [OAC vs OAI vs Public Bucket](#2-oac-vs-oai-vs-public-bucket)
3. [Thiết Lập OAC Cho S3 VOD](#3-thiết-lập-oac-cho-s3-vod)
4. [Bucket Policy Mẫu](#4-bucket-policy-mẫu)
5. [MediaPackage & MediaTailor](#5-mediapackage--mediatailor)
6. [SSE-KMS & Block Public Access](#6-sse-kms--block-public-access)
7. [Câu Hỏi Phỏng Vấn](#7-câu-hỏi-phỏng-vấn)

---

## 1. Tại Sao Cần Bảo Mật Origin

### 1.1 Rủi Ro Bucket Public

```
S3 bucket public-read:
  https://my-bucket.s3.amazonaws.com/movies/index.m3u8

Hậu quả:
  • Bỏ qua CloudFront → không cache, không signed URL, không geo block
  • Egress S3 đắt hơn CloudFront
  • Dễ bị scrape toàn bộ thư viện VOD
```

### 1.2 Mô Hình Mong Muốn

```
Viewer ──▶ CloudFront (signed URL/cookies, WAF) ──▶ S3 (chỉ CloudFront OAC)

Viewer ──X──▶ S3 trực tiếp (403 Access Denied)
```

---

## 2. OAC vs OAI vs Public Bucket

| Tiêu chí | Public bucket | OAI (legacy) | OAC (khuyến nghị) |
|----------|---------------|--------------|-------------------|
| Truy cập trực tiếp S3 | Cho phép | Chặn | Chặn |
| SSE-KMS trên S3 | Có | Hạn chế / phức tạp | **Hỗ trợ đầy đủ** |
| SigV4 signing | Không | Cũ | **Có** |
| Nhiều distribution / account | Khó | Trung bình | **Linh hoạt** |
| AWS khuyến nghị mới | Không | Migrate sang OAC | **Có** |

---

## 3. Thiết Lập OAC Cho S3 VOD

### 3.1 Các Bước (Console)

```
1. S3 bucket: Block all public access = ON
2. CloudFront → Origin → Create OAC
   - Name: vod-bucket-oac
   - Signing: Sign requests (recommended)
   - Origin type: S3
3. Gắn OAC vào origin của distribution
4. CloudFront hiển thị "Copy policy" → dán vào S3 bucket policy
5. Origin access: Origin access control settings (OAC)
6. Không dùng "Public" hoặc legacy OAI nếu tạo mới
```

### 3.2 Origin Request Policy

Với OAC, CloudFront tự ký request tới S3. Dùng **managed origin request policy** `CORS-S3Origin` hoặc custom nếu cần header đặc biệt.

```
S3 origin settings:
  Origin domain: my-vod-bucket.s3.us-east-1.amazonaws.com
  OAC: vod-bucket-oac
  Origin path: /output/hls   (optional prefix)
```

### 3.3 Sơ Đồ Luồng

```
CloudFront edge nhận GET /movies/index.m3u8
        │
        ▼
Origin fetch ký SigV4 với OAC IAM role/service principal
        │
        ▼
S3 GetObject → 200 + body
        │
        ▼
Edge cache + trả viewer
```

---

## 4. Bucket Policy Mẫu

### 4.1 Policy Do AWS Generate (OAC)

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "AllowCloudFrontServicePrincipal",
      "Effect": "Allow",
      "Principal": {
        "Service": "cloudfront.amazonaws.com"
      },
      "Action": "s3:GetObject",
      "Resource": "arn:aws:s3:::my-vod-bucket/*",
      "Condition": {
        "StringEquals": {
          "AWS:SourceArn": "arn:aws:cloudfront::123456789012:distribution/E1234567890ABC"
        }
      }
    }
  ]
}
```

**Condition `AWS:SourceArn`:** chỉ distribution ID cụ thể được đọc — tránh distribution khác trong account truy cập nhầm.

### 4.2 Nhiều Distribution (VOD + Staging)

```json
"Condition": {
  "StringEquals": {
    "AWS:SourceArn": [
      "arn:aws:cloudfront::123456789012:distribution/EPROD",
      "arn:aws:cloudfront::123456789012:distribution/ESTAGING"
    ]
  }
}
```

### 4.3 MediaConvert Ghi S3 + CloudFront Đọc

MediaConvert cần **IAM role** `s3:PutObject` riêng; OAC chỉ **GetObject** cho CloudFront:

```
MediaConvert Role:  s3:PutObject, s3:PutObjectAcl (bucket output)
CloudFront OAC:     s3:GetObject (via bucket policy Principal cloudfront.amazonaws.com)
Human admin:        không cần public read
```

---

## 5. MediaPackage & MediaTailor

### 5.1 MediaPackage CDN Authorization

MediaPackage endpoint URL có thể bị lộ. **CDN Authorization** yêu cầu request có header/chữ ký từ CloudFront đã cấu hình:

```
Viewer → CloudFront → (thêm signed headers) → MediaPackage origin
Viewer → MediaPackage trực tiếp → 403 (nếu bật authorization)
```

Cấu hình trên **MediaPackage Endpoint** + **CloudFront origin custom headers** theo [AWS doc](https://docs.aws.amazon.com/mediapackage/latest/ug/cdn-auth.html).

### 5.2 MediaTailor

**MediaTailor** playback endpoint thường đặt sau CloudFront:

```
Origin: abc.mediatailor.us-east-1.amazonaws.com
Behaviors: manifest TTL ngắn (SSAI cá nhân hoá)
```

OAC **không áp dụng** cho custom HTTP origin như MediaPackage/MediaTailor — dùng **CDN Authorization**, **signed URL**, hoặc **secret header** thay vì S3 OAC.

### 5.3 Bảng Origin Type

| Origin | Cơ chế bảo vệ |
|--------|----------------|
| S3 VOD | **OAC** + Block Public Access |
| MediaPackage | **CDN Authorization** + CloudFront only |
| MediaTailor | CloudFront + signed cookies + TTL |
| ALB / API | Security group + WAF; không OAC |

---

## 6. SSE-KMS & Block Public Access

### 6.1 Encryption At Rest

```
S3 bucket default encryption: SSE-KMS (CMK — Customer Master Key)

OAC + KMS:
  CloudFront service cần quyền kms:Decrypt qua bucket key policy
  (AWS hướng dẫn cập nhật KMS key policy khi dùng OAC)
```

### 6.2 Checklist Bảo Mật Media Origin

- [ ] Block Public Access: bật cả 4 option
- [ ] Không dùng ACL public-read trên object
- [ ] OAC thay OAI cho distribution mới
- [ ] Bucket policy giới hạn `AWS:SourceArn`
- [ ] CloudFront **HTTPS only** viewer protocol
- [ ] Signed cookies/URL cho subscriber content
- [ ] WAF rate limit trên path `*.m3u8`
- [ ] MediaPackage CDN Authorization khi live

---

## 7. Câu Hỏi Phỏng Vấn

**Q: OAC có chặn user tải trộm qua CloudFront không?**

> **Không hoàn toàn.** OAC chỉ chặn **bypass CloudFront** tới S3. User vẫn có thể dùng URL CloudFront hợp lệ. Cần thêm **signed URL/cookies** và **DRM** cho nội dung trả phí.

**Q: Migrate OAI sang OAC làm thế nào?**

> Tạo OAC mới → attach distribution origin → update S3 bucket policy (thay Principal OAI bằng `cloudfront.amazonaws.com` + SourceArn) → test → remove OAI. AWS Console có wizard **Migrate to OAC**.

**Q: Lambda trigger S3 event khi MediaConvert xong — conflict OAC?**

> **Không.** Lambda đọc S3 qua IAM role riêng, không qua CloudFront. OAC chỉ ảnh hưởng **viewer path** CloudFront → S3.

**Q: Cùng bucket làm source và output MediaConvert?**

> Tách prefix: `source/` (Convert đọc) và `output/` (CloudFront phục vụ). OAC policy có thể giới hạn `Resource` chỉ `arn:aws:s3:::bucket/output/*` nếu muốn source không qua CDN.

---

**Cập Nhật Lần Cuối:** 2026-06-04
**Xem thêm:** [2-signed-url-cookies.md](./2-signed-url-cookies.md) — Lớp authorization viewer
