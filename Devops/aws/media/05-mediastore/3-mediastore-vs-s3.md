# MediaStore vs S3 — So Sánh Cho Media Workloads

> MediaStore và Amazon S3 đều là object storage trên AWS, nhưng được thiết kế cho hai nhu cầu khác nhau. Hiểu rõ sự khác biệt giúp chọn đúng dịch vụ cho từng use case, tối ưu cả latency lẫn chi phí. Đây là câu hỏi phỏng vấn phổ biến về AWS media architecture.

## 📚 Mục Lục

1. [Tổng Quan So Sánh](#1-tổng-quan-so-sánh)
2. [Latency — Độ Trễ](#2-latency--độ-trễ)
3. [Feature Matrix — Ma Trận Tính Năng](#3-feature-matrix--ma-trận-tính-năng)
4. [Chi Phí (Pricing)](#4-chi-phí-pricing)
5. [Khi Nào Dùng Cái Nào?](#5-khi-nào-dùng-cái-nào)
6. [Kiến Trúc Pipeline Điển Hình](#6-kiến-trúc-pipeline-điển-hình)
7. [Migration — Chuyển Đổi Giữa Hai Dịch Vụ](#7-migration--chuyển-đổi-giữa-hai-dịch-vụ)
8. [Câu Hỏi Phỏng Vấn](#8-câu-hỏi-phỏng-vấn)

---

## 1. Tổng Quan So Sánh

```
┌─────────────────────────────────────────────────────────────────────────┐
│                    MEDIASTORE vs S3 — TỔNG QUAN                         │
├──────────────────────────────┬──────────────────────────────────────────┤
│         MEDIASTORE           │                  S3                       │
├──────────────────────────────┼──────────────────────────────────────────┤
│ Tối ưu cho: LIVE STREAMING   │ Tối ưu cho: GENERAL PURPOSE STORAGE      │
│ (ghi/đọc liên tục, real-time)│ (VOD, archive, backup, static assets)    │
├──────────────────────────────┼──────────────────────────────────────────┤
│ Độ trễ: ~10ms nhất quán      │ Độ trễ: 50–200ms (biến động hơn)         │
│ cho high-frequency I/O       │ spike khi load cao                        │
├──────────────────────────────┼──────────────────────────────────────────┤
│ Object size: tối đa 25 MB    │ Object size: tối đa 5 TB                  │
├──────────────────────────────┼──────────────────────────────────────────┤
│ Tính năng: đơn giản          │ Tính năng: phong phú (versioning,         │
│ (PUT/GET/DELETE + lifecycle) │  replication, lifecycle transitions,       │
│                              │  intelligent tiering, event notifications) │
├──────────────────────────────┼──────────────────────────────────────────┤
│ Pricing: cao hơn S3 một chút │ Pricing: thấp hơn, nhiều storage class    │
│ (~$0.023/GB vs ~$0.023/GB)   │ (Standard, IA, Glacier từ $0.004/GB)      │
└──────────────────────────────┴──────────────────────────────────────────┘
```

> **Tóm tắt một câu**: MediaStore là storage tốc độ cao cho live media fragments; S3 là storage đa năng cho mọi thứ còn lại.

---

## 2. Latency — Độ Trễ

### 2.1 Tại Sao MediaStore Có Latency Thấp Hơn?

MediaStore được thiết kế lại từ đầu cho **streaming I/O pattern** — nghĩa là encoder ghi nhiều object nhỏ (1–25 MB) liên tục với tần suất cao (1 object mỗi 2–6 giây). AWS tối ưu internal storage path và caching layer cho pattern này.

S3 được thiết kế cho **general-purpose storage** với object size lớn hơn và tần suất thấp hơn. Dù S3 cũng rất nhanh, latency có thể spike (tăng đột biến) khi nhiều request đồng thời.

### 2.2 So Sánh Latency Định Lượng

| Chỉ Số | MediaStore | S3 (Standard) | Ghi Chú |
|--------|-----------|---------------|---------|
| Write latency (ghi) | ~10ms (P50) | ~20–50ms (P50) | Với object 1–5 MB |
| Read latency (đọc) | ~1ms từ cache | ~5–20ms | S3 có tiered cache |
| Latency tại P99 | < 50ms | 100–500ms | P99 = 99% request dưới mức này |
| Latency spike | Rất ít | Có thể xảy ra khi hot partition | Do S3 key distribution |
| Consistency (tính nhất quán) | Strong: ghi xong đọc ngay được | Strong: sau 2020, S3 cũng strong | |

> **P99 latency quan trọng cho live streaming**: Nếu 1% request mất 500ms để ghi một segment, segment đó sẽ bị trễ → viewer thấy buffering. Với MediaStore, P99 nhất quán hơn S3.

### 2.3 Khi Nào Latency Thực Sự Quan Trọng?

```
Live Streaming:
  Encoder ghi segment mỗi 2 giây
  → Nếu ghi mất 200ms → OK (còn 1800ms buffer)
  → Nếu ghi mất 1500ms → Có vấn đề, segment bị trễ
  → MediaStore (< 50ms P99) an toàn hơn S3 cho use case này

VOD (Video on Demand):
  Encoder ghi file 500 MB một lần, xong
  → Latency 200ms hay 500ms không ảnh hưởng đáng kể
  → S3 là lựa chọn hợp lý hơn (rẻ hơn, nhiều tính năng hơn)
```

---

## 3. Feature Matrix — Ma Trận Tính Năng

### 3.1 Storage & Access

| Tính Năng | MediaStore | S3 |
|-----------|-----------|-----|
| Object storage (PUT/GET/DELETE) | ✅ | ✅ |
| Folder/prefix hierarchy | ✅ (tối đa 10 cấp) | ✅ (không giới hạn) |
| Object size tối đa | ❗ 25 MB | ✅ 5 TB |
| Multipart upload | ❌ | ✅ (cho object > 100 MB) |
| Presigned URL | ❌ | ✅ |
| Byte-range GET | ✅ | ✅ |
| Conditional GET (If-Match, If-None-Match) | ✅ | ✅ |

### 3.2 Bảo Mật & Kiểm Soát Truy Cập

| Tính Năng | MediaStore | S3 |
|-----------|-----------|-----|
| IAM policy | ✅ | ✅ |
| Resource-based policy | ✅ (Container Policy) | ✅ (Bucket Policy) |
| ACL — Access Control List | ❌ | ✅ (deprecated nhưng vẫn hỗ trợ) |
| Block Public Access settings | ❌ | ✅ |
| Presigned URL (truy cập tạm thời) | ❌ | ✅ |
| VPC Endpoint | ❌ | ✅ |
| Object Lock (WORM — Write Once Read Many) | ❌ | ✅ |
| Server-side encryption (SSE) | ✅ (SSE-S3 tự động) | ✅ (SSE-S3, SSE-KMS, SSE-C) |
| Client-side encryption | ❌ hỗ trợ chính thức | ✅ |

### 3.3 Tính Năng Nâng Cao

| Tính Năng | MediaStore | S3 |
|-----------|-----------|-----|
| Versioning (lưu nhiều phiên bản) | ❌ | ✅ |
| Cross-region replication | ❌ | ✅ |
| Event notifications (SNS/SQS/Lambda) | ❌ | ✅ |
| Lifecycle rules (xoá, transition) | ✅ giây (EXPIRE only) | ✅ ngày (EXPIRE + TRANSITION) |
| Intelligent Tiering (tự chuyển tier) | ❌ | ✅ |
| Storage classes | 1 (Standard) | 8+ (Standard, IA, Glacier, v.v.) |
| S3 Select (query nội dung object) | ❌ | ✅ |
| Batch Operations | ❌ | ✅ |
| Analytics (storage lens, inventory) | ❌ | ✅ |
| Static website hosting | ❌ | ✅ |
| CORS | ✅ | ✅ |
| Object tags | ❌ | ✅ |

### 3.4 Tích Hợp AWS

| Tích Hợp | MediaStore | S3 |
|----------|-----------|-----|
| CloudFront origin | ✅ | ✅ |
| MediaLive output | ✅ (native, tối ưu) | ✅ |
| MediaConvert output | ❌ | ✅ |
| Lambda trigger | ❌ | ✅ (S3 Event) |
| Athena query | ❌ | ✅ |
| Glue catalog | ❌ | ✅ |
| Rekognition | ❌ direct | ✅ |
| Transfer Acceleration | ❌ | ✅ |

---

## 4. Chi Phí (Pricing)

> **Lưu ý**: Giá AWS thay đổi theo region và theo thời gian. Kiểm tra [AWS Pricing Calculator](https://calculator.aws.amazon.com/) cho con số chính xác nhất. Các con số dưới đây là ước tính tương đối tại region us-east-1.

### 4.1 Bảng Giá Tương Đối

| Chi Phí | MediaStore | S3 Standard | S3 Standard-IA | S3 Glacier |
|---------|-----------|-------------|----------------|------------|
| Storage / GB / tháng | ~$0.023 | ~$0.023 | ~$0.0125 | ~$0.004 |
| PUT request (1000 requests) | ~$0.05 | ~$0.005 | ~$0.01 | ~$0.033 |
| GET request (1000 requests) | ~$0.01 | ~$0.0004 | ~$0.001 | ~$0.01 |
| Data transfer ra ngoài | ~$0.09/GB | ~$0.09/GB | ~$0.09/GB | ~$0.09/GB |

**So sánh chi phí PUT request** là điểm đáng chú ý nhất:
- MediaStore: ~$0.05 / 1000 PUT = 10× đắt hơn S3 Standard cho ghi
- Trong live streaming, encoder ghi 1 segment/2 giây = 30 PUT/phút = 1800 PUT/giờ
- 1 kênh live 24/7: 1800 × 24 = 43200 PUT/ngày = 1.296M PUT/tháng
- MediaStore: 1.296M × ($0.05/1000) = **$64.8/tháng** chỉ riêng PUT requests
- S3 Standard: 1.296M × ($0.005/1000) = **$6.48/tháng**

> **Kết luận chi phí**: MediaStore đắt hơn S3 đáng kể về request cost. Nhưng với live streaming, nếu latency spike của S3 gây buffering cho viewer → mất người dùng → cost cao hơn rất nhiều. Trade-off: pay for performance.

### 4.2 Ước Tính Chi Phí Pipeline Thực Tế

**Scenario: 1 kênh live 24/7, 3 renditions, segment 2 giây**

| Thành Phần | Tính Toán | Chi Phí/Tháng |
|-----------|-----------|----------------|
| PUT requests | 3 renditions × 1.296M = 3.888M PUT | ~$194 (MediaStore) vs ~$19 (S3) |
| Storage (buffer 5 phút) | ~0.2 GB × 24/7 = hằng số nhỏ | ~$0.005 |
| GET requests (CloudFront) | CloudFront pull từ origin: ít hơn PUT vì cache | ~$10–50 |
| Data transfer | CloudFront → viewer tính riêng | Tùy viewers |

> Với large-scale live streaming (nhiều kênh), chi phí MediaStore PUT request có thể trở thành factor quan trọng trong quyết định kiến trúc.

---

## 5. Khi Nào Dùng Cái Nào?

### 5.1 Decision Tree — Sơ Đồ Quyết Định

```
Bạn có nhu cầu lưu trữ media?
         │
         ▼
Có phải live streaming với encoder ghi liên tục?
    │              │
   Có             Không
    │              │
    ▼              ▼
Cần JIT         VOD / Archive / Backup
packaging,       → Dùng S3 ✅
DRM, time-
shift?
    │              │
   Có             Không
    │              │
    ▼              ▼
Dùng           Có packager riêng
MediaPackage ✅  (Wowza, Nginx, v.v.)?
                    │              │
                   Có             Không
                    │              │
                    ▼              ▼
              Dùng MediaStore ✅  Cần low-latency
                               write quan trọng?
                                    │              │
                                   Có             Không
                                    │              │
                                    ▼              ▼
                               MediaStore ✅      S3 ✅
                               (nếu latency
                               S3 chấp nhận
                               được → dùng S3
                               cho rẻ hơn)
```

### 5.2 Use Case Theo Từng Dịch Vụ

#### Dùng MediaStore Khi:

| Use Case | Lý Do Chọn MediaStore |
|----------|----------------------|
| Live streaming origin server | Encoder ghi liên tục → cần consistent low-latency write |
| Buffer trung gian giữa MediaLive và packager tự dựng | Low-latency read cho packager đọc fragment ngay lập tức |
| Live-to-VOD buffer (giữ event tạm thời) | Encoder ghi toàn bộ event, Lambda đọc và xử lý sau |
| Thay thế MediaPackage khi đã có packager riêng | Tránh trả tiền cho tính năng không dùng (DRM, time-shift) |

#### Dùng S3 Khi:

| Use Case | Lý Do Chọn S3 |
|----------|--------------|
| VOD output từ MediaConvert | File lớn, ghi một lần, không cần real-time | 
| Archive live stream sau event | Lưu trữ dài hạn → S3 Glacier rẻ hơn nhiều |
| Thumbnails, subtitle, metadata files | Nhỏ, ít thay đổi, không cần real-time |
| Backup transcript/caption | Lưu trữ lâu dài, versioning hữu ích |
| Static website / player HTML | S3 static website hosting |
| Source video gốc (original assets) | File lớn (vài GB–TB), multipart upload cần thiết |
| Output MediaConvert cho DASH/HLS | Ghi một lần, đọc nhiều qua CloudFront |

### 5.3 Có Thể Dùng Cả Hai Trong Cùng Pipeline

```
[MediaLive]
    │
    ├──▶ [MediaStore]       ← LIVE: ghi fragments real-time, CloudFront serve live
    │         │
    │         │ Lifecycle: xoá sau 1 giờ
    │
    └──▶ [Lambda] (trigger từ MediaLive khi kết thúc event)
              │
              ▼
         [MediaConvert] ← re-encode toàn bộ event thành VOD
              │
              ▼
         [S3 Bucket]         ← VOD: lưu trữ dài hạn, CloudFront serve on-demand
              │
              ▼
         [CloudFront] → Viewer xem VOD sau khi event kết thúc
```

---

## 6. Kiến Trúc Pipeline Điển Hình

### Kiến Trúc 1: All-in MediaStore (Đơn Giản)

```
OBS Studio → MediaLive → MediaStore → CloudFront → Viewer
                         (live buffer      (serve HLS
                          5–10 phút)        trực tiếp)
```

**Phù hợp:** Live stream đơn giản, không cần DRM, không cần time-shift, đã có packager trong MediaLive.

### Kiến Trúc 2: MediaStore + S3 Hybrid

```
                   ┌── MediaStore (live buffer) ──▶ CloudFront (live) ──▶ Viewer
MediaLive ─────────┤
                   └── S3 (HLS archive) ──────────▶ CloudFront (VOD)  ──▶ Viewer
```

**Phù hợp:** Sports event — vừa phát live, vừa lưu lại để xem lại sau.

### Kiến Trúc 3: MediaPackage Thay MediaStore (Khi Cần DRM + Time-shift)

```
MediaLive ──▶ MediaPackage ──▶ CloudFront ──▶ Viewer
              (JIT packaging                   (HLS/DASH/CMAF)
               + DRM + time-shift)
```

**Phù hợp:** OTT platform, nội dung trả phí, catch-up TV.

### Kiến Trúc 4: MediaStore Làm Buffer Cho Packager Tự Dựng

```
MediaLive ──▶ MediaStore ──▶ [Wowza / Nimble / Nginx + FFmpeg]
                              (đọc TS fragments, đóng gói DASH, DRM tùy chỉnh)
                                          │
                                          ▼
                                    CloudFront ──▶ Viewer
```

**Phù hợp:** Team đã đầu tư vào packager tự dựng, cần low-latency storage nhưng không muốn lock-in vào MediaPackage.

---

## 7. Migration — Chuyển Đổi Giữa Hai Dịch Vụ

### Từ S3 Sang MediaStore (Khi Latency S3 Gây Vấn Đề)

**Tình huống:** Team đang dùng S3 làm origin cho live streaming nhưng viewer complain buffering → phân tích thấy S3 write latency spike.

**Bước thực hiện:**
1. Tạo MediaStore container mới
2. Cấu hình access policy cho MediaLive role
3. Cập nhật MediaLive output destination từ S3 URL sang MediaStore endpoint
4. Cập nhật CloudFront origin từ S3 sang MediaStore container endpoint
5. Test: chạy song song cả hai, so sánh latency metrics
6. Cutover: chuyển 100% traffic sang MediaStore
7. Thêm lifecycle policy để giữ rolling buffer

### Từ MediaStore Sang S3 (Sau Khi Event Kết Thúc / Chuyển VOD)

```bash
# Script chuyển nội dung từ MediaStore sang S3 sau event
# (MediaStore CLI không có sync command, cần custom script)

#!/bin/bash
CONTAINER_ENDPOINT="https://aaabbbccc111.data.mediastore.ap-southeast-1.amazonaws.com"
S3_BUCKET="s3://my-vod-bucket/events/football-final/"

# Liệt kê tất cả objects trong container
aws mediastore-data list-items \
  --endpoint "$CONTAINER_ENDPOINT" \
  --path /events/football-final/ \
  --query "Items[].Name" \
  --output text | while read ITEM_PATH; do
    # Download từ MediaStore
    aws mediastore-data get-object \
      --endpoint "$CONTAINER_ENDPOINT" \
      --path "$ITEM_PATH" \
      "/tmp/$(basename $ITEM_PATH)"
    # Upload lên S3
    aws s3 cp "/tmp/$(basename $ITEM_PATH)" "${S3_BUCKET}$(basename $ITEM_PATH)"
    rm "/tmp/$(basename $ITEM_PATH)"
done
```

> **Thực tế**: Workflow live-to-VOD thường dùng Lambda được trigger sau khi event kết thúc. Lambda đọc từ MediaStore và ghi vào S3, sau đó tạo MediaConvert job để re-encode nếu cần.

---

## 8. Câu Hỏi Phỏng Vấn

**Q: Câu hỏi kinh điển — MediaStore vs S3: dùng cái nào cho live streaming?**

> **MediaStore** cho live streaming vì:
> - **Consistent low-latency write**: encoder ghi fragment mỗi 2 giây → latency nhất quán quan trọng hơn throughput
> - **Lifecycle policy tính bằng giây**: phù hợp với rolling buffer live (không phải lifecycle tính theo ngày như S3)
> - **Tối ưu cho streaming I/O**: write nhiều object nhỏ liên tục — đúng pattern của live segment
>
> **S3** cho VOD và archive vì:
> - **Giá rẻ hơn**: nhiều storage class, đặc biệt S3-IA và Glacier cho nội dung ít truy cập
> - **Tính năng phong phú**: versioning, replication, event notifications, multipart upload
> - **Object size lớn**: VOD file có thể vài GB — vượt giới hạn 25 MB của MediaStore
>
> **Tóm tắt**: MediaStore = live buffer; S3 = long-term storage.

**Q: Khi nào nên dùng MediaStore, khi nào dùng MediaPackage?**

> Đây là câu hỏi về **packaging vs storage**, không phải storage vs storage:
>
> - **MediaStore**: Pure storage — chỉ PUT/GET/DELETE objects. Không có packaging, không có DRM, không có time-shift. Cần packager riêng (MediaLive HLS output, Wowza, Nginx, v.v.).
>
> - **MediaPackage**: Managed packaging + origin — nhận TS stream, JIT-đóng gói HLS/DASH/CMAF, tích hợp DRM (SPEKE), hỗ trợ startover/catch-up TV. All-in-one solution.
>
> Chọn MediaPackage khi: cần DRM, time-shift, multi-format từ một source, muốn managed service.
> Chọn MediaStore khi: đã có packager, cần kiểm soát packaging, tiết kiệm chi phí (không trả tiền tính năng không dùng).

**Q: Có thể dùng S3 làm origin cho live HLS không?**

> **Có thể**, nhưng cần lưu ý:
> 1. **Latency spike**: S3 write latency có thể tăng đột biến dưới load cao → segment delay → viewer buffering
> 2. **Consistency**: Trước 2020, S3 có eventual consistency cho overwrite. Từ 2020, S3 đã có strong consistency → không còn là vấn đề.
> 3. **Segment size**: S3 không có giới hạn 25 MB → không phải vấn đề
> 4. **Cost**: S3 PUT request rẻ hơn MediaStore ~10× → tiết kiệm chi phí đáng kể
>
> **Kết luận**: Với live stream quy mô nhỏ hoặc khi latency S3 đủ tốt (test thực tế) → S3 là lựa chọn hợp lý vì rẻ hơn. Với live stream production quan trọng (sports, news) → MediaStore giảm rủi ro latency spike. Nên benchmark thực tế trước khi quyết định.

**Q: Giải thích trade-off chi phí giữa MediaStore và S3 cho live streaming?**

> **MediaStore đắt hơn S3** về request cost (~10× cho PUT). Với 1 kênh live 24/7, 3 renditions, segment 2 giây:
> - MediaStore PUT cost: ~$194/tháng
> - S3 Standard PUT cost: ~$19/tháng
> - Chênh lệch: ~$175/tháng
>
> **Nhưng cần tính cả indirect cost**:
> - Nếu S3 latency spike → viewer buffering → viewer rời đi → revenue loss (đặc biệt với live event)
> - Với sports event có 100K viewers, mỗi viewer trả $10/tháng → mất 1% viewers = $10K/tháng
>
> **Kết luận**: $175/tháng extra cho MediaStore rất hợp lý nếu live event có giá trị kinh doanh cao. Với kênh live traffic thấp hoặc nội dung miễn phí → dùng S3 cho tiết kiệm.

---

## 🔗 Tham Khảo

- [AWS MediaStore vs S3 Comparison](https://docs.aws.amazon.com/mediastore/latest/ug/what-is.html)
- [Amazon S3 Documentation](https://docs.aws.amazon.com/s3/)
- [AWS MediaStore Pricing](https://aws.amazon.com/mediastore/pricing/)
- [Amazon S3 Pricing](https://aws.amazon.com/s3/pricing/)
- [Building a Live Streaming Platform on AWS](https://aws.amazon.com/blogs/media/)

---

**Cập Nhật Lần Cuối:** 2026-06-04
**Phần Trước:** [2-lifecycle-policy.md](2-lifecycle-policy.md) — Lifecycle Policy
**Phần Tiếp Theo:** [../06-mediatailor/README.md](../06-mediatailor/README.md) — MediaTailor Ad Insertion
