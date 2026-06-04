# MediaStore — Container, Access Policy & CORS

> Container là đơn vị lưu trữ cơ bản của MediaStore. Access policy và CORS configuration kiểm soát ai được đọc/ghi vào container và từ đâu. Hiểu rõ ba thành phần này là nền tảng để bảo mật pipeline live streaming trên AWS.

## 📚 Mục Lục

1. [Container — Thùng Chứa Media](#1-container--thùng-chứa-media)
2. [Access Policy — Chính Sách Truy Cập](#2-access-policy--chính-sách-truy-cập)
3. [CORS — Cross-Origin Resource Sharing](#3-cors--cross-origin-resource-sharing)
4. [Tích Hợp Với IAM](#4-tích-hợp-với-iam)
5. [Ví Dụ Thực Tế End-to-End](#5-ví-dụ-thực-tế-end-to-end)
6. [Câu Hỏi Phỏng Vấn](#6-câu-hỏi-phỏng-vấn)

---

## 1. Container — Thùng Chứa Media

### 1.1 Container Là Gì?

**Container** trong MediaStore là đơn vị lưu trữ cô lập — tương tự S3 Bucket nhưng tối ưu cho media workloads. Mỗi container:

- Có một **endpoint URL** riêng duy nhất để đọc/ghi objects
- Có **namespace** riêng (không chia sẻ với container khác)
- Có **chính sách truy cập** (access policy) gắn trực tiếp
- Có **cấu hình CORS** riêng
- Có **lifecycle policy** riêng

**Endpoint URL format:**
```
https://<container-id>.data.mediastore.<region>.amazonaws.com
```

Ví dụ:
```
https://aaabbbccc111.data.mediastore.ap-southeast-1.amazonaws.com
```

### 1.2 Tạo Container

**Bằng AWS Console:**
```
AWS Console → MediaStore → Containers → Create container
  Container name: my-live-stream   (1–255 ký tự, chữ thường, số, gạch ngang)
  Tags: Environment=Production, Team=MediaOps
```

**Bằng AWS CLI:**
```bash
aws mediastore create-container \
  --container-name "my-live-stream" \
  --region ap-southeast-1
```

**Response trả về:**
```json
{
  "Container": {
    "ContainerARN": "arn:aws:mediastore:ap-southeast-1:123456789012:container/my-live-stream",
    "Endpoint": "https://aaabbbccc111.data.mediastore.ap-southeast-1.amazonaws.com",
    "Name": "my-live-stream",
    "Status": "CREATING"
  }
}
```

> Container ở trạng thái `CREATING` trong vài giây, sau đó chuyển sang `ACTIVE`. Chỉ có thể đọc/ghi khi container ở trạng thái `ACTIVE`.

### 1.3 Cấu Trúc Path Trong Container

MediaStore hỗ trợ **folder hierarchy** — phân cấp thư mục tối đa 10 cấp, dùng `/` phân cấp:

```
Container: my-live-stream
  │
  ├── /sports/                       ← Cấp 1: theo thể loại
  │   ├── football/                  ← Cấp 2: theo môn thể thao
  │   │   ├── channel1/              ← Cấp 3: theo kênh
  │   │   │   ├── master.m3u8        ← Object: master playlist
  │   │   │   ├── 720p/
  │   │   │   │   ├── index.m3u8     ← Object: rendition playlist
  │   │   │   │   ├── seg_001.ts     ← Object: media segment
  │   │   │   │   └── seg_002.ts
  │   │   │   └── 480p/
  │   │   └── channel2/
  │   └── tennis/
  │
  └── /news/
      └── evening/
```

**Thao tác với objects qua CLI:**
```bash
# Ghi object vào container (tương đương HTTP PUT)
aws mediastore-data put-object \
  --endpoint https://aaabbbccc111.data.mediastore.ap-southeast-1.amazonaws.com \
  --path /sports/football/channel1/seg_001.ts \
  --body seg_001.ts \
  --content-type video/MP2T \
  --region ap-southeast-1

# Đọc object từ container (tương đương HTTP GET)
aws mediastore-data get-object \
  --endpoint https://aaabbbccc111.data.mediastore.ap-southeast-1.amazonaws.com \
  --path /sports/football/channel1/seg_001.ts \
  output_seg_001.ts

# Liệt kê nội dung (tương đương HTTP LIST)
aws mediastore-data list-items \
  --endpoint https://aaabbbccc111.data.mediastore.ap-southeast-1.amazonaws.com \
  --path /sports/football/channel1/
```

### 1.4 Giới Hạn Quan Trọng (Limits)

| Giới Hạn | Giá Trị | Ghi Chú |
|----------|---------|---------|
| Số container tối đa / tài khoản | 100 | Có thể request AWS tăng |
| Kích thước object tối đa | **25 MB** | Ảnh hưởng đến segment duration tối đa |
| Cấp folder lồng nhau tối đa | 10 | `/a/b/c/d/e/f/g/h/i/j/object` |
| Tên container | 1–255 ký tự | Chữ thường, số, gạch ngang |
| Throughput | Không giới hạn cứng | Auto-scale theo nhu cầu |

> **Lưu ý segment size**: Với segment duration 6 giây và bitrate 4 Mbps, kích thước segment ≈ 3 MB. Với 25 MB limit, an toàn cho hầu hết use case. Tuy nhiên nếu dùng bitrate rất cao (>25 Mbps) với segment dài, cần giảm segment duration hoặc chia nhỏ.

---

## 2. Access Policy — Chính Sách Truy Cập

### 2.1 Access Policy Là Gì?

**Access policy** (chính sách truy cập) là **resource-based policy** (chính sách dựa trên tài nguyên) gắn trực tiếp vào container — hoạt động tương tự S3 Bucket Policy. Policy được viết theo cú pháp JSON của AWS IAM.

**Hai lớp kiểm soát truy cập:**

```
Lớp 1: IAM Policy (Identity-based)
  → Gắn vào User/Role/Group
  → "Tôi (identity này) được làm gì với resource nào?"

Lớp 2: Container Access Policy (Resource-based)
  → Gắn vào container
  → "Resource này (container) cho phép ai làm gì với nó?"

Quyền truy cập = IAM Policy AND Access Policy đều phải ALLOW
  (trừ trường hợp cross-account: chỉ cần một trong hai ALLOW)
```

### 2.2 Cấu Trúc Access Policy

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "MediaLiveWriteAccess",
      "Effect": "Allow",
      "Principal": {
        "AWS": "arn:aws:iam::123456789012:role/MediaLiveRole"
      },
      "Action": [
        "mediastore:PutObject",
        "mediastore:GetObject",
        "mediastore:DeleteObject",
        "mediastore:DescribeObject"
      ],
      "Resource": "arn:aws:mediastore:ap-southeast-1:123456789012:container/my-live-stream/*"
    }
  ]
}
```

**Các Action quan trọng của MediaStore:**

| Action | Mô Tả | HTTP Method |
|--------|-------|-------------|
| `mediastore:PutObject` | Ghi object vào container | PUT |
| `mediastore:GetObject` | Đọc object từ container | GET |
| `mediastore:DeleteObject` | Xoá object | DELETE |
| `mediastore:DescribeObject` | Lấy metadata object (không lấy nội dung) | HEAD |
| `mediastore:ListItems` | Liệt kê objects/folders tại path chỉ định | GET (list) |
| `mediastore:GetContainerPolicy` | Đọc access policy của container | - |
| `mediastore:PutContainerPolicy` | Ghi/thay thế access policy | - |

### 2.3 Mẫu Access Policy Cho Từng Use Case

#### Mẫu 1: MediaLive Ghi + CloudFront Đọc (Pattern Phổ Biến Nhất)

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "AllowMediaLiveWrite",
      "Effect": "Allow",
      "Principal": {
        "AWS": "arn:aws:iam::123456789012:role/MediaLiveAccessRole"
      },
      "Action": [
        "mediastore:PutObject",
        "mediastore:GetObject",
        "mediastore:DescribeObject",
        "mediastore:ListItems"
      ],
      "Resource": [
        "arn:aws:mediastore:ap-southeast-1:123456789012:container/my-live-stream",
        "arn:aws:mediastore:ap-southeast-1:123456789012:container/my-live-stream/*"
      ]
    },
    {
      "Sid": "AllowCloudFrontRead",
      "Effect": "Allow",
      "Principal": {
        "Service": "cloudfront.amazonaws.com"
      },
      "Action": [
        "mediastore:GetObject",
        "mediastore:DescribeObject"
      ],
      "Resource": "arn:aws:mediastore:ap-southeast-1:123456789012:container/my-live-stream/*",
      "Condition": {
        "StringEquals": {
          "AWS:SourceArn": "arn:aws:cloudfront::123456789012:distribution/EDFDVBD6EXAMPLE"
        }
      }
    }
  ]
}
```

#### Mẫu 2: Public Read (Chỉ Dùng Cho Dev/Test — Không Dùng Production)

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "PublicReadForDev",
      "Effect": "Allow",
      "Principal": "*",
      "Action": "mediastore:GetObject",
      "Resource": "arn:aws:mediastore:ap-southeast-1:123456789012:container/dev-test-stream/*"
    }
  ]
}
```

> ⚠️ **Cảnh báo**: Không bao giờ dùng `Principal: "*"` cho container production. Bất kỳ ai biết endpoint URL đều đọc được nội dung.

#### Mẫu 3: Nhiều Service Cùng Truy Cập

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "AllowEncoders",
      "Effect": "Allow",
      "Principal": {
        "AWS": [
          "arn:aws:iam::123456789012:role/MediaLiveRolePrimary",
          "arn:aws:iam::123456789012:role/MediaLiveRoleBackup",
          "arn:aws:iam::123456789012:role/WowzaEncoderRole"
        ]
      },
      "Action": ["mediastore:PutObject", "mediastore:DescribeObject"],
      "Resource": "arn:aws:mediastore:ap-southeast-1:123456789012:container/my-live-stream/*"
    },
    {
      "Sid": "AllowReadersAndPackagers",
      "Effect": "Allow",
      "Principal": {
        "AWS": [
          "arn:aws:iam::123456789012:role/PackagerRole",
          "arn:aws:iam::123456789012:role/LambdaProcessorRole"
        ],
        "Service": "cloudfront.amazonaws.com"
      },
      "Action": ["mediastore:GetObject", "mediastore:ListItems", "mediastore:DescribeObject"],
      "Resource": [
        "arn:aws:mediastore:ap-southeast-1:123456789012:container/my-live-stream",
        "arn:aws:mediastore:ap-southeast-1:123456789012:container/my-live-stream/*"
      ]
    }
  ]
}
```

### 2.4 Gắn Access Policy Qua CLI

```bash
# Gắn access policy vào container
aws mediastore put-container-policy \
  --container-name "my-live-stream" \
  --policy file://container-policy.json \
  --region ap-southeast-1

# Đọc access policy hiện tại
aws mediastore get-container-policy \
  --container-name "my-live-stream" \
  --region ap-southeast-1

# Xoá access policy (container trở về trạng thái private hoàn toàn)
aws mediastore delete-container-policy \
  --container-name "my-live-stream" \
  --region ap-southeast-1
```

---

## 3. CORS — Cross-Origin Resource Sharing (Chia Sẻ Tài Nguyên Đa Nguồn Gốc)

### 3.1 Tại Sao Cần CORS?

Browser áp dụng **Same-Origin Policy** (Chính sách cùng nguồn gốc): JavaScript tại `https://myapp.com` mặc định **không thể** gửi request đến domain khác (`https://aaabbbccc111.data.mediastore.amazonaws.com`).

Khi web player (HLS.js, Shaka Player) chạy trong browser và request segments trực tiếp từ MediaStore endpoint, browser sẽ chặn nếu không có CORS headers trong response.

**CORS flow:**
```
Browser (myapp.com)
  │
  │ 1. Preflight request: OPTIONS /live/seg_001.ts
  │    Origin: https://myapp.com
  ▼
MediaStore Container
  │ 2. Response với CORS headers:
  │    Access-Control-Allow-Origin: https://myapp.com
  │    Access-Control-Allow-Methods: GET, HEAD
  │    Access-Control-Max-Age: 3000
  ▼
Browser
  │ 3. CORS check: Origin được allow → tiếp tục
  │
  │ 4. Actual request: GET /live/seg_001.ts
  ▼
MediaStore
  │ 5. Response với data + CORS headers
  ▼
Browser → Player nhận được segment, phát video
```

### 3.2 Cấu Trúc CORS Policy

```json
{
  "CorsPolicy": [
    {
      "AllowedOrigins": ["https://myapp.com", "https://staging.myapp.com"],
      "AllowedMethods": ["GET", "HEAD"],
      "AllowedHeaders": ["*"],
      "MaxAgeSeconds": 3000,
      "ExposeHeaders": ["ETag", "Content-Length"]
    }
  ]
}
```

**Giải thích từng trường:**

| Trường | Mô Tả | Ví Dụ |
|--------|-------|-------|
| `AllowedOrigins` | Danh sách domain được phép truy cập | `["https://myapp.com"]` hoặc `["*"]` |
| `AllowedMethods` | HTTP methods được cho phép | `["GET", "HEAD"]` — player chỉ cần GET và HEAD |
| `AllowedHeaders` | Headers client được phép gửi | `["*"]` hoặc danh sách cụ thể |
| `MaxAgeSeconds` | Thời gian browser cache kết quả preflight (giây) | `3000` (50 phút) |
| `ExposeHeaders` | Headers trong response mà browser được đọc từ JS | `["ETag"]` — player dùng ETag để validate cache |

### 3.3 Các Mẫu CORS Phổ Biến

#### Mẫu 1: Production — Chỉ Allow Domain Cụ Thể

```json
{
  "CorsPolicy": [
    {
      "AllowedOrigins": [
        "https://www.mystream.com",
        "https://player.mystream.com",
        "https://app.mystream.com"
      ],
      "AllowedMethods": ["GET", "HEAD"],
      "AllowedHeaders": ["Range", "Origin", "Accept-Encoding"],
      "MaxAgeSeconds": 3000,
      "ExposeHeaders": ["ETag", "Content-Range", "Content-Length"]
    }
  ]
}
```

> `Range` header quan trọng cho **byte-range requests** — player HTTP đôi khi request một phần segment thay vì toàn bộ. Cần cho phép `Range` trong `AllowedHeaders`.

#### Mẫu 2: Development — Allow Tất Cả Origins

```json
{
  "CorsPolicy": [
    {
      "AllowedOrigins": ["*"],
      "AllowedMethods": ["GET", "HEAD", "PUT"],
      "AllowedHeaders": ["*"],
      "MaxAgeSeconds": 600,
      "ExposeHeaders": ["ETag"]
    }
  ]
}
```

> ⚠️ `AllowedOrigins: ["*"]` chỉ phù hợp cho dev/test. Production nên chỉ định domain cụ thể.

#### Mẫu 3: Nhiều Môi Trường (Multi-env) — Nhiều Rule

```json
{
  "CorsPolicy": [
    {
      "AllowedOrigins": ["https://www.production.com"],
      "AllowedMethods": ["GET", "HEAD"],
      "AllowedHeaders": ["Range", "Origin"],
      "MaxAgeSeconds": 86400,
      "ExposeHeaders": ["ETag", "Content-Length"]
    },
    {
      "AllowedOrigins": [
        "https://staging.production.com",
        "http://localhost:3000",
        "http://localhost:8080"
      ],
      "AllowedMethods": ["GET", "HEAD"],
      "AllowedHeaders": ["*"],
      "MaxAgeSeconds": 300,
      "ExposeHeaders": ["ETag"]
    }
  ]
}
```

### 3.4 Gắn CORS Policy Qua CLI

```bash
# Gắn CORS policy vào container
aws mediastore put-cors-policy \
  --container-name "my-live-stream" \
  --cors-policy file://cors-policy.json \
  --region ap-southeast-1

# Đọc CORS policy hiện tại
aws mediastore get-cors-policy \
  --container-name "my-live-stream" \
  --region ap-southeast-1

# Xoá CORS policy
aws mediastore delete-cors-policy \
  --container-name "my-live-stream" \
  --region ap-southeast-1
```

### 3.5 Khi Nào Không Cần CORS?

Không cần cấu hình CORS nếu:

1. **CloudFront đứng trước MediaStore**: Browser request đến CloudFront domain (ví dụ `d1234abcd.cloudfront.net`), CloudFront request đến MediaStore phía sau. Browser không biết đến MediaStore endpoint. CloudFront tự xử lý CORS headers nếu cần.

2. **Mobile app** (iOS/Android): Không có Same-Origin Policy như browser.

3. **Server-to-server**: Lambda/EC2/ECS đọc từ MediaStore — không phải browser request.

> **Best practice**: Dùng CloudFront trước MediaStore → không cần lo CORS, đồng thời được cache + global distribution. Chỉ cấu hình CORS khi player request trực tiếp đến MediaStore endpoint mà không qua CDN.

---

## 4. Tích Hợp Với IAM

### 4.1 IAM Role Cho MediaLive

Khi tạo MediaLive channel với MediaStore output, cần IAM role có quyền ghi vào container:

**Trust policy của IAM Role** (cho phép MediaLive assume role):
```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Principal": {
        "Service": "medialive.amazonaws.com"
      },
      "Action": "sts:AssumeRole"
    }
  ]
}
```

**Permission policy của IAM Role** (quyền cụ thể):
```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": [
        "mediastore:PutObject",
        "mediastore:GetObject",
        "mediastore:DeleteObject",
        "mediastore:DescribeObject",
        "mediastore:ListItems"
      ],
      "Resource": [
        "arn:aws:mediastore:*:123456789012:container/my-live-stream",
        "arn:aws:mediastore:*:123456789012:container/my-live-stream/*"
      ]
    },
    {
      "Effect": "Allow",
      "Action": "mediastore:DescribeContainer",
      "Resource": "arn:aws:mediastore:*:123456789012:container/my-live-stream"
    }
  ]
}
```

### 4.2 IAM Role Cho Lambda (Đọc/Xử Lý Fragment)

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": [
        "mediastore:GetObject",
        "mediastore:ListItems",
        "mediastore:DescribeObject"
      ],
      "Resource": "arn:aws:mediastore:ap-southeast-1:123456789012:container/my-live-stream/*"
    }
  ]
}
```

### 4.3 Least Privilege — Nguyên Tắc Đặc Quyền Tối Thiểu

| Service | Quyền Cần Thiết | Không Cần |
|---------|-----------------|-----------|
| MediaLive (encoder) | `PutObject`, `DescribeObject` | `DeleteObject`, `ListItems` (trừ khi cần) |
| CloudFront origin | `GetObject`, `DescribeObject` | `PutObject`, `DeleteObject` |
| Lambda processor | `GetObject`, `ListItems` | `PutObject`, `DeleteObject` |
| Admin/DevOps | Tất cả | - |

---

## 5. Ví Dụ Thực Tế End-to-End

### Thiết Lập Pipeline: MediaLive → MediaStore → CloudFront

**Bước 1: Tạo Container**
```bash
aws mediastore create-container \
  --container-name "sports-live-stream" \
  --region ap-southeast-1
```

**Bước 2: Gắn Access Policy**
```bash
# Lưu policy vào file
cat > container-policy.json << 'EOF'
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "AllowMediaLive",
      "Effect": "Allow",
      "Principal": {
        "AWS": "arn:aws:iam::123456789012:role/MediaLiveRole"
      },
      "Action": ["mediastore:PutObject", "mediastore:GetObject", "mediastore:DescribeObject"],
      "Resource": "arn:aws:mediastore:ap-southeast-1:123456789012:container/sports-live-stream/*"
    },
    {
      "Sid": "AllowCloudFront",
      "Effect": "Allow",
      "Principal": {"Service": "cloudfront.amazonaws.com"},
      "Action": ["mediastore:GetObject", "mediastore:DescribeObject"],
      "Resource": "arn:aws:mediastore:ap-southeast-1:123456789012:container/sports-live-stream/*",
      "Condition": {
        "StringEquals": {
          "AWS:SourceArn": "arn:aws:cloudfront::123456789012:distribution/EXXXXXXXXXXXXXX"
        }
      }
    }
  ]
}
EOF

aws mediastore put-container-policy \
  --container-name "sports-live-stream" \
  --policy file://container-policy.json
```

**Bước 3: Gắn CORS Policy**
```bash
cat > cors-policy.json << 'EOF'
{
  "CorsPolicy": [{
    "AllowedOrigins": ["https://sports.mycompany.com"],
    "AllowedMethods": ["GET", "HEAD"],
    "AllowedHeaders": ["Range", "Origin"],
    "MaxAgeSeconds": 3000,
    "ExposeHeaders": ["ETag", "Content-Length"]
  }]
}
EOF

aws mediastore put-cors-policy \
  --container-name "sports-live-stream" \
  --cors-policy file://cors-policy.json
```

**Bước 4: Cấu hình CloudFront Distribution**
```bash
# Tạo CloudFront distribution với MediaStore làm origin
# Origin domain: aaabbbccc111.data.mediastore.ap-southeast-1.amazonaws.com
# Origin protocol: HTTPS only
# Cache behaviors:
#   - *.m3u8 → TTL: min=2, default=5, max=10
#   - *.ts   → TTL: min=60, default=300, max=3600
```

---

## 6. Câu Hỏi Phỏng Vấn

**Q: Container access policy khác IAM policy ở điểm nào?**

> - **IAM policy** là **identity-based** (dựa trên danh tính): gắn vào User/Role/Group, định nghĩa "identity này được làm gì với resource nào".
> - **Container access policy** là **resource-based** (dựa trên tài nguyên): gắn vào container, định nghĩa "container này cho phép ai làm gì với nó".
>
> Khi cả hai đều tồn tại trong cùng tài khoản, quyền cuối = **giao** của cả hai (cả IAM policy VÀ resource policy phải ALLOW). Khi cross-account, chỉ cần **một trong hai** là ALLOW (nhưng cả hai không được có DENY tường minh).

**Q: Tại sao cần cả access policy và CORS? Hai thứ này khác nhau gì?**

> - **Access policy**: Kiểm soát **ai** (AWS identity: role, service) được phép **làm gì** (action: GET, PUT) với container — đây là kiểm soát **authorization** (phân quyền) ở cấp AWS.
> - **CORS**: Kiểm soát **browser từ domain nào** được phép gửi JavaScript request đến container endpoint — đây là cơ chế bảo mật của **trình duyệt web**, không liên quan đến AWS authorization.
>
> Có thể có access policy cho phép nhưng CORS chặn (browser request bị chặn bởi Same-Origin Policy). Hoặc CORS allow nhưng access policy chặn (browser gửi được request nhưng AWS từ chối 403). Cần cấu hình đúng **cả hai** cho web player hoạt động.

**Q: `ExposeHeaders: ["ETag"]` trong CORS dùng để làm gì?**

> Browser theo mặc định chỉ cho JavaScript đọc một số headers cơ bản trong response (Content-Type, Content-Language, Content-Length, Cache-Control, Expires, Last-Modified, Pragma). `ETag` không nằm trong danh sách đó.
>
> Media player (HLS.js, Shaka) dùng `ETag` để **validate cache** — kiểm tra xem segment đã cache có còn mới nhất không. Nếu không expose `ETag`, player không đọc được giá trị này từ JavaScript, có thể dẫn đến behavior không mong muốn (fetch thừa hoặc dùng segment cũ).

---

## 🔗 Tham Khảo

- [MediaStore Container Access Policy](https://docs.aws.amazon.com/mediastore/latest/ug/policies.html)
- [MediaStore CORS Policy](https://docs.aws.amazon.com/mediastore/latest/ug/cors-policy.html)
- [AWS IAM — Resource-based Policy](https://docs.aws.amazon.com/IAM/latest/UserGuide/access_policies.html#policies_resource-based)

---

**Cập Nhật Lần Cuối:** 2026-06-04
**Phần Trước:** [README.md](README.md) — MediaStore Tổng Quan
**Phần Tiếp Theo:** [2-lifecycle-policy.md](2-lifecycle-policy.md) — Lifecycle Policy
