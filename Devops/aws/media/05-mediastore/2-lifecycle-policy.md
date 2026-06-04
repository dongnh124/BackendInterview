# MediaStore — Lifecycle Policy (Chính Sách Vòng Đời)

> Lifecycle policy — chính sách vòng đời cho phép MediaStore tự động xoá các object hết hạn theo rule đã định. Đây là tính năng thiết yếu cho live streaming: encoder liên tục ghi segment mới, lifecycle policy đảm bảo chỉ giữ lại **rolling buffer** (bộ đệm cuộn) trong cửa sổ thời gian cần thiết, tránh phình to container và tốn chi phí lưu trữ thừa.

## 📚 Mục Lục

1. [Lifecycle Policy Là Gì?](#1-lifecycle-policy-là-gì)
2. [Cấu Trúc Rule](#2-cấu-trúc-rule)
3. [Định Nghĩa Path — Lọc Đường Dẫn](#3-định-nghĩa-path--lọc-đường-dẫn)
4. [Điều Kiện Hết Hạn](#4-điều-kiện-hết-hạn)
5. [Mẫu Lifecycle Policy Thực Tế](#5-mẫu-lifecycle-policy-thực-tế)
6. [Quản Lý Lifecycle Policy Qua CLI](#6-quản-lý-lifecycle-policy-qua-cli)
7. [Tính Toán Rolling Buffer](#7-tính-toán-rolling-buffer)
8. [Câu Hỏi Phỏng Vấn](#8-câu-hỏi-phỏng-vấn)

---

## 1. Lifecycle Policy Là Gì?

**Lifecycle policy** (chính sách vòng đời) là tập hợp các **rule** (quy tắc) định nghĩa điều kiện để MediaStore tự động xoá objects. Khi object thoả mãn điều kiện trong rule, MediaStore sẽ thực hiện action `EXPIRE` — tức là xoá object đó.

### Tại Sao Live Streaming Cần Lifecycle Policy?

```
Encoder ghi fragment liên tục:
  T=00s: seg_0001.ts (ghi vào container)
  T=02s: seg_0002.ts
  T=04s: seg_0003.ts
  ...
  T=3600s: seg_1800.ts  ← 1 giờ phát → 1800 segments!
  ...
  T=86400s: seg_43200.ts ← 24 giờ phát → 43200 segments!

Không có lifecycle policy:
  Container chứa hàng chục nghìn segment cũ
  → Chi phí storage tăng liên tục
  → Hiệu suất ListItems giảm
  → Quản lý namespace phức tạp

Có lifecycle policy (ví dụ: xoá sau 300 giây):
  Container luôn chỉ có ~150 segments (300s ÷ 2s/segment)
  → Chi phí storage cố định, không tăng theo thời gian
  → Namespace sạch sẽ
  → Đây là "rolling buffer" — bộ đệm cuộn
```

### Hoạt Động Của Lifecycle Policy

```
Container: my-live-stream
  │
  ├── seg_0001.ts  [tuổi: 310s] ← Rule: seconds_since_create > 300 → EXPIRE (xoá)
  ├── seg_0002.ts  [tuổi: 308s] ← EXPIRE
  ├── seg_0003.ts  [tuổi: 306s] ← EXPIRE
  ├── seg_0004.ts  [tuổi: 304s] ← EXPIRE
  ├── seg_0005.ts  [tuổi: 302s] ← Ranh giới xoá
  ├── seg_0006.ts  [tuổi: 298s] ← Giữ lại
  ├── seg_0007.ts  [tuổi: 296s] ← Giữ lại
  ...
  └── seg_0155.ts  [tuổi: 0s]   ← Mới nhất, giữ lại

MediaStore chạy lifecycle evaluation định kỳ
→ Xoá tất cả objects thoả mãn điều kiện
→ Rolling buffer: chỉ giữ segment trong 5 phút gần nhất
```

---

## 2. Cấu Trúc Rule

Lifecycle policy có cấu trúc JSON với một mảng `rules`. Mỗi rule gồm:

```json
{
  "rules": [
    {
      "definition": {
        "path": [ ... ],           // Bộ lọc path — rule áp dụng cho object nào
        "seconds_since_create": [ ... ]  // Điều kiện hết hạn — bao nhiêu giây thì xoá
      },
      "action": "EXPIRE"           // Action duy nhất hiện tại: xoá object
    }
  ]
}
```

**Các thành phần:**

| Thành Phần | Mô Tả | Bắt Buộc |
|-----------|-------|----------|
| `definition.path` | Mảng bộ lọc path, áp dụng theo logic OR | Có |
| `definition.seconds_since_create` | Mảng điều kiện tuổi object (giây kể từ khi tạo) | Có |
| `action` | Hiện tại chỉ hỗ trợ `"EXPIRE"` (xoá object) | Có |

**Giới hạn:**
- Tối đa **10 rules** mỗi lifecycle policy
- Mỗi rule có tối đa **10 path expressions**
- Tuổi xoá tối thiểu: **1 giây** (thực tế nên đặt ít nhất vài phút)

---

## 3. Định Nghĩa Path — Lọc Đường Dẫn

`path` là mảng các bộ lọc xác định object nào rule áp dụng. Nhiều bộ lọc trong mảng kết hợp theo logic **OR** — object thoả mãn bất kỳ bộ lọc nào đều bị áp dụng rule.

### 3.1 Các Loại Path Expression

#### Wildcard — Ký Tự Đại Diện (`wildcard`)

Dùng `*` đại diện cho bất kỳ chuỗi ký tự nào (kể cả `/`):

```json
"path": [{"wildcard": "live/*"}]
```

Khớp với:
- `/live/seg_001.ts` ✅
- `/live/720p/seg_001.ts` ✅
- `/live/720p/2026/06/04/seg_001.ts` ✅
- `/news/seg_001.ts` ❌ (không bắt đầu bằng `live/`)

#### Prefix — Tiền Tố (`prefix`)

Khớp với objects bắt đầu bằng prefix chỉ định:

```json
"path": [{"prefix": "live/720p/"}]
```

Khớp với:
- `/live/720p/seg_001.ts` ✅
- `/live/720p/index.m3u8` ✅
- `/live/480p/seg_001.ts` ❌

#### Equals — Khớp Chính Xác (`equals`)

Khớp với đúng object tại path chỉ định:

```json
"path": [{"equals": "live/temp/processing.ts"}]
```

Ít dùng trong live streaming, thường dùng để xoá object cụ thể.

### 3.2 Kết Hợp Nhiều Path (OR)

```json
"path": [
  {"wildcard": "live/720p/*"},
  {"wildcard": "live/480p/*"},
  {"wildcard": "live/360p/*"}
]
```

Rule áp dụng cho object tại **bất kỳ** path nào trong danh sách (logic OR).

---

## 4. Điều Kiện Hết Hạn

`seconds_since_create` là mảng các điều kiện về tuổi object (tính bằng giây kể từ khi object được tạo/ghi). Nhiều điều kiện trong mảng kết hợp theo logic **AND**.

### 4.1 Toán Tử So Sánh

| Toán Tử | Ý Nghĩa | Ví Dụ |
|---------|---------|-------|
| `>` | Lớn hơn (older than) | `[">", 300]` — cũ hơn 300 giây |
| `>=` | Lớn hơn hoặc bằng | `[">=", 300]` — từ 300 giây trở lên |
| `<` | Nhỏ hơn (newer than) | `["<", 60]` — mới hơn 60 giây |
| `<=` | Nhỏ hơn hoặc bằng | `["<=", 60]` — từ 60 giây trở xuống |
| `=` | Bằng (hiếm dùng) | `["=", 300]` — đúng 300 giây |

### 4.2 Kết Hợp Điều Kiện (AND)

Xoá objects **vừa** cũ hơn 300 giây **vừa** mới hơn 3600 giây (trong cửa sổ 5–60 phút):

```json
"seconds_since_create": [
  {"numeric": [">", 300]},
  {"numeric": ["<", 3600]}
]
```

> Ví dụ này hiếm dùng trong thực tế. Pattern phổ biến nhất là chỉ dùng một điều kiện `> N`.

---

## 5. Mẫu Lifecycle Policy Thực Tế

### Mẫu 1: Rolling Buffer 5 Phút (Pattern Phổ Biến Nhất)

Giữ 5 phút gần nhất — phù hợp với live streaming không cần time-shift:

```json
{
  "rules": [
    {
      "definition": {
        "path": [{"wildcard": "live/*"}],
        "seconds_since_create": [{"numeric": [">", 300]}]
      },
      "action": "EXPIRE"
    }
  ]
}
```

**Kết quả:**
- Encoder ghi segment mỗi 2 giây → rolling buffer ~150 segments
- Sau 300 giây, segment cũ bị xoá tự động
- Container luôn có khoảng 150 × (segment_size ≈ 2 MB) ≈ 300 MB

### Mẫu 2: Buffer Theo Rendition Riêng Biệt

Giữ buffer khác nhau cho từng quality level:

```json
{
  "rules": [
    {
      "definition": {
        "path": [{"wildcard": "live/1080p/*"}],
        "seconds_since_create": [{"numeric": [">", 180]}]
      },
      "action": "EXPIRE"
    },
    {
      "definition": {
        "path": [
          {"wildcard": "live/720p/*"},
          {"wildcard": "live/480p/*"},
          {"wildcard": "live/360p/*"}
        ],
        "seconds_since_create": [{"numeric": [">", 600]}]
      },
      "action": "EXPIRE"
    }
  ]
}
```

**Lý do:** 1080p tốn storage nhiều hơn → giữ buffer ngắn hơn. Các rendition thấp giữ lâu hơn để hỗ trợ viewer có băng thông thấp xem lại.

### Mẫu 3: Multi-Channel Với Buffer Khác Nhau

```json
{
  "rules": [
    {
      "definition": {
        "path": [{"wildcard": "sports/live/*"}],
        "seconds_since_create": [{"numeric": [">", 600]}]
      },
      "action": "EXPIRE"
    },
    {
      "definition": {
        "path": [{"wildcard": "news/live/*"}],
        "seconds_since_create": [{"numeric": [">", 300]}]
      },
      "action": "EXPIRE"
    },
    {
      "definition": {
        "path": [{"wildcard": "temp/*"}],
        "seconds_since_create": [{"numeric": [">", 60]}]
      },
      "action": "EXPIRE"
    }
  ]
}
```

### Mẫu 4: Giữ Toàn Bộ Event (Không Xoá Sớm)

Dùng khi muốn giữ toàn bộ nội dung event (ví dụ: 3 giờ) để xử lý live-to-VOD sau:

```json
{
  "rules": [
    {
      "definition": {
        "path": [{"wildcard": "events/football-final/*"}],
        "seconds_since_create": [{"numeric": [">", 14400]}]
      },
      "action": "EXPIRE"
    }
  ]
}
```

`14400` giây = 4 giờ — giữ toàn bộ event 3 giờ + 1 giờ dự phòng để xử lý.

### Mẫu 5: Xoá Manifest Sau Khi Stream Kết Thúc

Giữ manifest `.m3u8` lâu hơn một chút so với segments, để viewer đang xem có thể finish:

```json
{
  "rules": [
    {
      "definition": {
        "path": [{"wildcard": "live/*.ts"}],
        "seconds_since_create": [{"numeric": [">", 300]}]
      },
      "action": "EXPIRE"
    },
    {
      "definition": {
        "path": [{"wildcard": "live/*.m3u8"}],
        "seconds_since_create": [{"numeric": [">", 600]}]
      },
      "action": "EXPIRE"
    }
  ]
}
```

---

## 6. Quản Lý Lifecycle Policy Qua CLI

### Gắn Policy Vào Container

```bash
# Tạo file policy
cat > lifecycle-policy.json << 'EOF'
{
  "rules": [
    {
      "definition": {
        "path": [{"wildcard": "live/*"}],
        "seconds_since_create": [{"numeric": [">", 300]}]
      },
      "action": "EXPIRE"
    }
  ]
}
EOF

# Gắn vào container
aws mediastore put-lifecycle-policy \
  --container-name "my-live-stream" \
  --lifecycle-policy file://lifecycle-policy.json \
  --region ap-southeast-1
```

### Đọc Policy Hiện Tại

```bash
aws mediastore get-lifecycle-policy \
  --container-name "my-live-stream" \
  --region ap-southeast-1
```

Response:
```json
{
  "LifecyclePolicy": "{\"rules\":[{\"definition\":{\"path\":[{\"wildcard\":\"live/*\"}],\"seconds_since_create\":[{\"numeric\":[\">\" ,300]}]},\"action\":\"EXPIRE\"}]}"
}
```

> Lưu ý: AWS trả về lifecycle policy dưới dạng **JSON string escape** (chuỗi JSON được escape), không phải JSON object trực tiếp. Cần parse thêm nếu xử lý bằng script.

### Xoá Policy

```bash
aws mediastore delete-lifecycle-policy \
  --container-name "my-live-stream" \
  --region ap-southeast-1
```

> Sau khi xoá policy, MediaStore sẽ **không** tự động xoá objects cũ nữa. Tất cả objects trong container được giữ lại vô thời hạn cho đến khi bị xoá thủ công.

### Kiểm Tra Trạng Thái Container Sau Khi Gắn Policy

```bash
aws mediastore describe-container \
  --container-name "my-live-stream" \
  --region ap-southeast-1 \
  --query "Container.{Name:Name,Status:Status,Endpoint:Endpoint}"
```

---

## 7. Tính Toán Rolling Buffer

Để tính buffer size phù hợp, cần nắm:

### 7.1 Công Thức Tính Số Segments Trong Buffer

```
Buffer size (giây)   = seconds_since_create threshold
Segment duration (s) = duration mỗi segment (thường 2–6 giây)
Số renditions       = số quality levels (720p, 480p, 360p, ...)

Số segments / rendition = Buffer size ÷ Segment duration
Tổng segments           = Số segments/rendition × Số renditions
```

**Ví dụ:** Buffer 5 phút (300 giây), segment 2 giây, 3 renditions:
```
Segments/rendition = 300 ÷ 2 = 150 segments
Tổng segments      = 150 × 3 = 450 segments
```

### 7.2 Tính Chi Phí Storage

```
Bitrate 720p  = 3 Mbps → segment 2s = 750 KB ≈ 0.75 MB
Bitrate 480p  = 1.5 Mbps → segment 2s = 375 KB ≈ 0.375 MB
Bitrate 360p  = 0.8 Mbps → segment 2s = 200 KB ≈ 0.2 MB

Storage / rendition = 150 segments × avg_size
720p storage  = 150 × 0.75 MB = 112.5 MB
480p storage  = 150 × 0.375 MB = 56.25 MB
360p storage  = 150 × 0.2 MB = 30 MB
Tổng          ≈ 200 MB (cho rolling buffer 5 phút, 3 renditions)

MediaStore pricing ≈ $0.023/GB/tháng (kiểm tra AWS Pricing Calculator)
Cost = 0.2 GB × $0.023 = ~$0.0046/tháng — rất nhỏ so với network cost
```

### 7.3 Chọn Buffer Size Phù Hợp

| Use Case | Buffer Khuyến Nghị | Lý Do |
|----------|-------------------|-------|
| Live streaming thuần (không time-shift) | 300–600 giây (5–10 phút) | Đủ để recover khi mạng chập chờn |
| Live streaming cần startover nhẹ | 1800–3600 giây (30–60 phút) | Viewer join muộn vẫn xem được từ đầu |
| Live-to-VOD cần giữ toàn bộ event | Thời gian event + 30% buffer | Lambda/worker có đủ thời gian xử lý |
| Thử nghiệm / Dev | 60–120 giây | Giảm cost, dễ debug |

> **Lưu ý quan trọng về time-shift**: MediaStore **không** có tính năng time-shift tích hợp sẵn như MediaPackage. Nếu cần catch-up TV hay startover đầy đủ → dùng MediaPackage. MediaStore chỉ giữ rolling buffer, viewer phải xem trong cửa sổ thời gian buffer còn tồn tại.

---

## 8. Câu Hỏi Phỏng Vấn

**Q: Lifecycle policy trong MediaStore khác lifecycle rules trong S3 như thế nào?**

> - **S3 lifecycle rules**: Rất phong phú — hỗ trợ transition (chuyển storage class: Standard → IA → Glacier), expiration (xoá), abort multipart upload. Filter dựa trên prefix, tags, object size, object age tính theo **ngày**.
> - **MediaStore lifecycle policy**: Đơn giản hơn nhiều — chỉ hỗ trợ `EXPIRE` (xoá). Filter dựa trên path (wildcard, prefix, equals). Tuổi tính theo **giây** (phù hợp với live streaming fragment có lifecycle tính bằng phút, không phải ngày).
>
> MediaStore được thiết kế cho real-time/near-real-time workload với granularity giây. S3 designed cho long-term storage với granularity ngày.

**Q: Khi nào nên dùng buffer 5 phút, khi nào nên dùng buffer 60 phút?**

> - **Buffer 5 phút**: Khi live stream chỉ xem trực tiếp, không cần xem lại. Tiết kiệm storage. Phù hợp với game show, sự kiện thể thao nơi viewer chỉ quan tâm đến nội dung hiện tại.
> - **Buffer 60 phút**: Khi muốn hỗ trợ viewer join muộn xem lại từ đầu chương trình (startover nhẹ), hoặc cần buffer đủ lớn để Lambda/worker xử lý live-to-VOD không bị miss segment.
>
> Cân nhắc thêm: buffer lớn hơn → storage cost cao hơn + latency của ListItems cao hơn (nhiều objects trong container). Cho live streaming production thông thường, 5–10 phút thường đủ.

**Q: Điều gì xảy ra khi lifecycle policy xoá một segment mà viewer đang stream?**

> Nếu viewer đang request segment đó ngay lúc lifecycle evaluation chạy và xoá nó, request sẽ nhận **404 Not Found**. Player thường xử lý 404 bằng cách retry hoặc bỏ qua segment đó và tiến đến segment tiếp theo.
>
> Để tránh vấn đề này trong thực tế:
> 1. **Buffer dư dật**: Đặt threshold lớn hơn playlist window length đủ nhiều. Ví dụ: manifest giữ 60 giây, threshold đặt 300 giây → segment luôn tồn tại ít nhất 4 phút sau khi bị remove khỏi manifest.
> 2. **CloudFront caching**: Segments đã được CloudFront cache sẽ không bị ảnh hưởng dù MediaStore xoá rồi — CloudFront serve từ edge cache.
> 3. **Dùng MediaPackage thay thế** nếu cần time-shift đáng tin cậy: MediaPackage có rolling buffer riêng và xử lý edge case này tốt hơn.

---

## 🔗 Tham Khảo

- [MediaStore Lifecycle Policy](https://docs.aws.amazon.com/mediastore/latest/ug/policies-object-lifecycle.html)
- [MediaStore Object Expiration](https://docs.aws.amazon.com/mediastore/latest/ug/policies-object-lifecycle-add.html)
- [MediaStore Pricing](https://aws.amazon.com/mediastore/pricing/)

---

**Cập Nhật Lần Cuối:** 2026-06-04
**Phần Trước:** [1-container-access-policy.md](1-container-access-policy.md) — Container, Access Policy & CORS
**Phần Tiếp Theo:** [3-mediastore-vs-s3.md](3-mediastore-vs-s3.md) — MediaStore vs S3
