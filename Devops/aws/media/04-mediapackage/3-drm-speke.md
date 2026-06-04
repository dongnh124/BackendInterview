# MediaPackage — DRM & SPEKE (Bảo Vệ Nội Dung Kỹ Thuật Số)

> DRM — Digital Rights Management — Quản Lý Quyền Kỹ Thuật Số là hệ thống mã hoá và kiểm soát truy cập nội dung video, ngăn sao chép và phát lại trái phép. SPEKE — Secure Packager and Encoder Key Exchange — Trao Đổi Khoá Bảo Mật Giữa Đóng Gói Và Mã Hoá là giao thức chuẩn AWS để tích hợp DRM vào MediaPackage.

## 📚 Mục Lục

1. [DRM Là Gì và Tại Sao Cần?](#1-drm-là-gì-và-tại-sao-cần)
2. [Các Hệ Thống DRM Phổ Biến](#2-các-hệ-thống-drm-phổ-biến)
3. [SPEKE — Giao Thức Trao Đổi Khoá](#3-speke--giao-thức-trao-đổi-khoá)
4. [Multi-DRM Setup trong MediaPackage](#4-multi-drm-setup-trong-mediapackage)
5. [CENC & CBCS — Chuẩn Mã Hoá](#5-cenc--cbcs--chuẩn-mã-hoá)
6. [Key Rotation — Xoay Vòng Khoá](#6-key-rotation--xoay-vòng-khoá)
7. [Luồng Hoạt Động End-to-End](#7-luồng-hoạt-động-end-to-end)
8. [Câu Hỏi Phỏng Vấn](#8-câu-hỏi-phỏng-vấn)

---

## 1. DRM Là Gì và Tại Sao Cần?

### Vấn Đề Cần Giải Quyết

```
Không có DRM:
  Viewer ──▶ CloudFront ──▶ HLS segments (.ts) ──▶ Player
                                   │
                           Bất kỳ ai download được segment
                           → Sao chép, phát lại miễn phí
                           → Vi phạm bản quyền nội dung premium

Với DRM:
  Viewer ──▶ CloudFront ──▶ HLS segments (encrypted .ts) ──▶ Player
                                                               │
                                               Player cần DRM License
                                               (chứng minh có quyền xem)
                                                               │
                                               ──▶ DRM License Server
                                               ← Nhận decryption key
                                                               │
                                               Giải mã và phát nội dung
```

### DRM Hoạt Động Theo Ba Bước

```
BƯỚC 1: ENCRYPT (Mã hoá) — MediaPackage
  Segment gốc (plaintext) ──▶ Encrypt với Content Key ──▶ Encrypted segment

BƯỚC 2: LICENSE REQUEST (Xin giấy phép) — Player
  Player nhận encrypted segment
  ──▶ Gửi license request lên DRM License Server
  ──▶ Server kiểm tra quyền (đã thanh toán chưa? token hợp lệ?)
  ──▶ Trả về Content Key (nếu có quyền)

BƯỚC 3: DECRYPT & PLAY (Giải mã & Phát) — Player
  Player dùng Content Key giải mã segment
  ──▶ Phát video
```

---

## 2. Các Hệ Thống DRM Phổ Biến

### Bảng So Sánh

| DRM System | Nhà Cung Cấp | Platform Hỗ Trợ | Container |
|-----------|-------------|----------------|-----------|
| **Widevine** | Google | Android, Chrome, Firefox, Chromecast | CENC |
| **FairPlay** | Apple | iOS, macOS, tvOS, Safari | CBCS |
| **PlayReady** | Microsoft | Windows, Xbox, Edge (Trident), Smart TV | CENC |
| **Marlin** | Marlin DA | Một số Smart TV Nhật | CENC |

### System IDs (UUID)

Mỗi DRM system có một UUID duy nhất — System ID — dùng để nhận diện trong DASH MPD và SPEKE requests:

```
Widevine:  edef8ba9-79d6-4ace-a3c8-27dcd51d21ed
FairPlay:  94ce86fb-07bb-4b43-adf0-7a428f0e66b1  (HLS only)
PlayReady: 9a04f079-9840-4286-ab92-e65be0885f95
```

### Chọn DRM Theo Platform

```
Platform Matrix:
─────────────────────────────────────────────────
iOS / macOS / tvOS / Safari  → FairPlay (bắt buộc)
Android                      → Widevine
Chrome / Firefox             → Widevine
Windows / Edge               → PlayReady
Xbox                         → PlayReady
Smart TV (Samsung, LG)       → Widevine hoặc PlayReady
Roku                         → Widevine
─────────────────────────────────────────────────

Multi-DRM tối thiểu cho cross-platform:
  FairPlay + Widevine
  (phủ 95%+ viewer)

Multi-DRM đầy đủ:
  FairPlay + Widevine + PlayReady
  (phủ Smart TV và Windows)
```

---

## 3. SPEKE — Giao Thức Trao Đổi Khoá

### SPEKE Là Gì?

**SPEKE — Secure Packager and Encoder Key Exchange** là giao thức API chuẩn của AWS, định nghĩa cách MediaPackage (packager) giao tiếp với DRM Key Provider (nhà cung cấp khoá mã hoá) để lấy encryption keys.

```
Không có SPEKE (tích hợp tuỳ biến):
  MediaPackage ──??? customAPI ???──▶ DRM Vendor A
  MediaPackage ──??? customAPI ???──▶ DRM Vendor B
  → Mỗi vendor cần tích hợp riêng, tốn công

Với SPEKE (chuẩn hoá):
  MediaPackage ──[SPEKE API]──▶ DRM Vendor A (hỗ trợ SPEKE)
  MediaPackage ──[SPEKE API]──▶ DRM Vendor B (hỗ trợ SPEKE)
  → Một giao thức, nhiều vendor
```

### SPEKE Flow (Luồng Hoạt Động)

```
1. Viewer request segment từ MediaPackage Endpoint (có DRM)
                │
                ▼
2. MediaPackage gửi SPEKE key request:
   POST https://speke-key-provider.example.com/
   Headers:
     Authorization: AWS Signature V4 (IAM Role)
     Content-Type: application/xml
   Body:
     <cpix:CPIX xmlns:cpix="urn:dashif:org:cpix" ...>
       <cpix:ContentKeyList>
         <cpix:ContentKey kid="...UUID..." />
       </cpix:ContentKeyList>
       <cpix:DRMSystemList>
         <cpix:DRMSystem kid="...UUID..." systemId="edef8ba9-...Widevine..." />
         <cpix:DRMSystem kid="...UUID..." systemId="94ce86fb-...FairPlay..." />
       </cpix:DRMSystemList>
     </cpix:CPIX>
                │
                ▼
3. SPEKE Key Provider trả về Content Keys + License Acquisition URLs
                │
                ▼
4. MediaPackage mã hoá segment với Content Key
5. Nhúng License Acquisition URL vào manifest
                │
                ▼
6. Player đọc manifest, biết URL xin license
7. Player gửi license request lên DRM License Server
8. Server kiểm tra entitlement (quyền xem), trả về key
9. Player giải mã và phát
```

### SPEKE Key Provider Options

**Option 1: AWS Elemental MediaPackage SPEKE Reference Implementation**
- AWS cung cấp sẵn Lambda function làm SPEKE Key Provider đơn giản
- Dùng cho dev/test, không phải production

**Option 2: DRM Vendor Third-party**
- **Irdeto**: enterprise-grade, hỗ trợ nhiều platform
- **EZDRM**: cloud-native, dễ tích hợp
- **ExpressPlay**: Intertrust DRM, multi-DRM SaaS
- **Verimatrix**: video security chuyên nghiệp
- **BuyDRM**: KeyOS multi-DRM

**Option 3: Tự xây SPEKE Key Provider**
- Implement SPEKE API specification
- Lưu keys an toàn trong AWS KMS — Key Management Service
- Cần hiểu sâu CPIX — Content Protection Information Exchange Format

---

## 4. Multi-DRM Setup trong MediaPackage

### Cấu Hình DASH Endpoint Với Widevine + PlayReady

```json
{
  "DashPackage": {
    "SegmentDurationSeconds": 2,
    "Encryption": {
      "KeyRotationIntervalSeconds": 86400,
      "SpekeKeyProvider": {
        "ResourceId": "my-content-id-001",
        "RoleArn": "arn:aws:iam::123456789:role/MediaPackageSPEKERole",
        "Url": "https://speke.example.com/v1/keys",
        "SystemIds": [
          "edef8ba9-79d6-4ace-a3c8-27dcd51d21ed",
          "9a04f079-9840-4286-ab92-e65be0885f95"
        ]
      }
    }
  }
}
```

### Cấu Hình HLS Endpoint Với FairPlay

```json
{
  "HlsPackage": {
    "SegmentDurationSeconds": 6,
    "Encryption": {
      "EncryptionMethod": "SAMPLE-AES",
      "KeyRotationIntervalSeconds": 86400,
      "SpekeKeyProvider": {
        "ResourceId": "my-content-id-001",
        "RoleArn": "arn:aws:iam::123456789:role/MediaPackageSPEKERole",
        "Url": "https://speke.example.com/v1/keys",
        "SystemIds": [
          "94ce86fb-07bb-4b43-adf0-7a428f0e66b1"
        ],
        "CertificateArn": "arn:aws:acm:us-east-1:123456789:certificate/abc-123"
      }
    }
  }
}
```

> **Lưu ý FairPlay:** Cần certificate từ Apple Developer Program, lưu trong AWS ACM — AWS Certificate Manager và cung cấp ARN cho SPEKE config.

### IAM Role Cho SPEKE

MediaPackage cần IAM Role để gọi SPEKE Key Provider URL. Role cần trust policy cho MediaPackage và permission gọi API:

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": [
        "sts:AssumeRole"
      ],
      "Principal": {
        "Service": "mediapackage.amazonaws.com"
      }
    }
  ]
}
```

---

## 5. CENC & CBCS — Chuẩn Mã Hoá

### CENC — Common Encryption (Mã Hoá Chung)

**CENC** là chuẩn mã hoá ISO cho MPEG DASH và fMP4. Một nội dung mã hoá CENC có thể được giải mã bởi nhiều DRM system khác nhau (Widevine, PlayReady) dùng cùng key.

```
Content (fMP4)
    │
    ▼ CENC encrypt với Content Key K
    │
Encrypted fMP4
    ├── Widevine header (dùng key K, license từ Widevine server)
    └── PlayReady header (dùng key K, license từ PlayReady server)

→ Một file, hai DRM system → hiệu quả
```

### CBCS — Common Broadcast Encryption Scheme

**CBCS** là phiên bản mới hơn của CENC, được Apple yêu cầu cho FairPlay Streaming trên iOS 11+. Khác với CENC ở chỗ dùng AES-CBC mode thay vì AES-CTR.

```
CENC (CTR mode):  Dùng cho Widevine + PlayReady (DASH/fMP4)
CBCS (CBC mode):  Bắt buộc cho FairPlay (HLS/fMP4 trên Apple devices)
```

### Lựa Chọn Encryption Mode

```
Chỉ cần iOS/Apple:
  HLS Endpoint + SAMPLE-AES (CBCS) + FairPlay → đủ

Chỉ cần Android/Web:
  DASH Endpoint + CENC + Widevine (+ PlayReady) → đủ

Cross-platform (khuyến nghị):
  HLS Endpoint  + SAMPLE-AES + FairPlay        → iOS/macOS/tvOS
  DASH Endpoint + CENC        + Widevine         → Android/Chrome
  DASH Endpoint + CENC        + PlayReady        → Windows/Xbox
  hoặc
  CMAF Endpoint + CBCS + FairPlay + Widevine    → iOS + Android (một endpoint)
```

---

## 6. Key Rotation — Xoay Vòng Khoá

### Tại Sao Cần Key Rotation?

Key Rotation là cơ chế thay đổi encryption key theo định kỳ:
- **Giới hạn thiệt hại**: nếu key bị lộ, chỉ ảnh hưởng nội dung trong khoảng thời gian đó
- **Tuân thủ bảo mật**: nhiều studio yêu cầu key rotation cho premium content
- **Ngăn key reuse**: tránh việc dùng một key cho toàn bộ nội dung

### Cơ Chế Key Rotation

```
T=0:    Sử dụng Key-1 → encrypt segments 1-100
T=24h:  Rotate → sử dụng Key-2 → encrypt segments 101-200
T=48h:  Rotate → sử dụng Key-3 → encrypt segments 201-300
...

Manifest:
  seg-099.ts → encrypted với Key-1
  seg-100.ts → encrypted với Key-1
  seg-101.ts → encrypted với Key-2  (rotation boundary)
  seg-102.ts → encrypted với Key-2
```

### Cấu Hình Key Rotation

```json
{
  "Encryption": {
    "KeyRotationIntervalSeconds": 86400,
    "SpekeKeyProvider": { ... }
  }
}
```

| Interval (giây) | Use Case |
|----------------|---------|
| `0` | Không rotate (một key cho toàn bộ) |
| `3600` (1 giờ) | High-security live content |
| `86400` (24 giờ) | Standard premium content |
| `604800` (7 ngày) | Low-sensitivity content |

> **Lưu ý:** Key rotation xảy ra tại segment boundary — điểm bắt đầu segment, không cắt giữa segment. Player cần request license mới khi gặp segment với key mới.

---

## 7. Luồng Hoạt Động End-to-End

### Luồng Đầy Đủ Cho Viewer Premium Content

```
[Người Dùng] → Vào trang web, chọn nội dung premium
       │
       ▼
[Auth Service] → Kiểm tra subscription, tạo JWT token
       │
       ▼
[CloudFront] → Viewer request manifest với Signed URL
       │         (chứng minh đã auth)
       ▼
[MediaPackage HLS Endpoint]
  → JIT tạo encrypted manifest
  → Manifest chứa:
    - Segment URLs (encrypted)
    - EXT-X-SESSION-KEY với FairPlay License URL
       │
       ▼
[Player — AVPlayer trên iOS]
  → Đọc manifest, thấy content cần FairPlay
  → Gửi SPC — Server Playback Context đến FairPlay License URL
       │
       ▼
[DRM License Server — SPEKE Key Provider]
  → Kiểm tra JWT token (đã authenticated?)
  → Lấy Content Key từ key store
  → Tạo CKC — Content Key Context (encrypted response)
  → Trả CKC về player
       │
       ▼
[Player]
  → Giải mã CKC với device private key
  → Lấy Content Key
  → Giải mã và phát segments
       │
       ▼
[Viewer] → Xem nội dung premium!
```

---

## 8. Câu Hỏi Phỏng Vấn

**Q: SPEKE là gì và tại sao AWS tạo ra nó?**

> **SPEKE — Secure Packager and Encoder Key Exchange** là giao thức API chuẩn hoá việc trao đổi encryption keys giữa AWS media services (MediaPackage, MediaConvert) và các DRM Key Providers của bên thứ ba (Irdeto, EZDRM,...). Trước SPEKE, mỗi DRM vendor có API riêng, buộc AWS phải tích hợp từng vendor. SPEKE chuẩn hoá interface dựa trên CPIX — Content Protection Information Exchange Format của DASH-IF, giúp bất kỳ vendor nào hỗ trợ SPEKE đều tích hợp được ngay.

**Q: Thiết lập Multi-DRM cho iOS và Android như thế nào?**

> Cần ít nhất hai endpoints:
> 1. **HLS Endpoint** với `SAMPLE-AES` + FairPlay System ID → phục vụ iOS/Safari/tvOS
> 2. **DASH Endpoint** với `CENC` + Widevine System ID → phục vụ Android/Chrome
>
> Hoặc dùng **CMAF Endpoint** với `CBCS` mode hỗ trợ cả FairPlay (iOS) lẫn Widevine (Android/Chrome) từ một bộ segments, tiết kiệm hơn.

**Q: CENC và CBCS khác nhau thế nào?**

> - **CENC** (AES-CTR mode): Dùng cho Widevine và PlayReady trên DASH/fMP4. Hầu hết Android và web browsers.
> - **CBCS** (AES-CBC mode): Apple yêu cầu cho FairPlay từ iOS 11+. Dùng cho HLS/fMP4 trên thiết bị Apple.
>
> CMAF hỗ trợ cả CENC lẫn CBCS, cho phép một file segment phục vụ cả Apple (CBCS/FairPlay) và non-Apple (CENC/Widevine) qua dual-key content.

**Q: Key rotation có ảnh hưởng đến viewer đang xem không?**

> Có ảnh hưởng nhỏ nhưng player hiện đại xử lý được: khi player gặp segment với key mới (rotation boundary), nó tự động gửi license request để lấy key mới trước khi segment đó được phát. Nếu license server nhanh (< 1 giây), viewer không cảm nhận được. Vấn đề phát sinh nếu license server chậm hoặc không available → player buffer bị rỗng → rebuffering tạm thời.

---

**Phần Tiếp Theo:** [4-time-shift-viewing.md](./4-time-shift-viewing.md) — Time-shift & Catch-up TV
