# AWS Elemental MediaStore — Lưu Trữ Media Độ Trễ Thấp (Low-Latency Media Storage)

> AWS Elemental MediaStore là dịch vụ lưu trữ đối tượng (object storage) được tối ưu hoá đặc biệt cho media workloads — đặc biệt là nội dung live streaming. MediaStore cung cấp độ trễ thấp hơn Amazon S3 đáng kể khi đọc/ghi các media fragment nhỏ theo luồng (streaming I/O pattern), đồng thời vẫn duy trì tính nhất quán (consistency) và độ bền (durability) cao cần thiết cho môi trường production.

## 📚 Mục Lục (Table of Contents)

1. [MediaStore Là Gì?](#1-mediastore-là-gì)
2. [Kiến Trúc Tổng Quan](#2-kiến-trúc-tổng-quan)
3. [Các Khái Niệm Cốt Lõi](#3-các-khái-niệm-cốt-lõi)
4. [Luồng Xử Lý](#4-luồng-xử-lý)
5. [Nội Dung Chi Tiết](#5-nội-dung-chi-tiết)
6. [Câu Hỏi Phỏng Vấn Thường Gặp](#6-câu-hỏi-phỏng-vấn-thường-gặp)

---

## 1. MediaStore Là Gì?

**AWS Elemental MediaStore** là dịch vụ lưu trữ đối tượng (object store) chuyên biệt cho media workloads. Nó hoạt động giống S3 về API (HTTP PUT/GET/DELETE) nhưng được tối ưu hoá cho hai pattern truy cập đặc thù của media:

- **High-frequency write** (ghi tần suất cao): encoder ghi liên tục các media fragment (mỗi 2–6 giây một segment)
- **Low-latency read** (đọc độ trễ thấp): player/packager đọc ngay lập tức fragment vừa được ghi

### Vị Trí Trong Media Pipeline

```
                 ┌──────────── LIVE PIPELINE (với MediaStore) ───────────────────┐
                 │                                                                │
[Encoder/OBS] → [MediaLive] → [MediaStore] → [Packager/Origin] → [CloudFront] → [Viewer]
  RTMP/RTP        Encode        Ghi fragment   Đọc & đóng gói     CDN phân phối   Player
  nguồn live       multi-rate    vào container  HLS/DASH/CMAF      toàn cầu        ABR
```

```
                 ┌──────────── LIVE PIPELINE (với MediaPackage) ──────────────────┐
                 │                                                                 │
[Encoder/OBS] → [MediaLive] → [MediaPackage] → [CloudFront] → [Viewer]
  RTMP/RTP        Encode        JIT-đóng gói     CDN phân phối   Player
                               HLS/DASH/CMAF     toàn cầu        ABR
                               + DRM + Origin
```

> **Sự khác biệt cốt lõi:** MediaStore là **lưu trữ thuần tuý** (pure storage), không có packaging hay DRM. MediaPackage là **packaging + origin server** tích hợp đầy đủ. Dùng MediaStore khi bạn có packager riêng hoặc cần buffer trung gian độ trễ thấp.

### Chức Năng Chính Của MediaStore

| Chức Năng | Mô Tả |
|-----------|-------|
| **Low-latency I/O** (Đọc/ghi độ trễ thấp) | Độ trễ nhất quán dưới 10ms cho ghi, dưới 1ms cho đọc từ cache |
| **Container** (Thùng chứa) | Đơn vị lưu trữ cô lập, tương tự S3 Bucket nhưng tối ưu cho media |
| **Object path** (Đường dẫn đối tượng) | Hỗ trợ folder hierarchy (tối đa 10 cấp), dùng `/` phân cấp |
| **HTTP API** (API HTTP thuần) | PUT/GET/DELETE/HEAD qua HTTPS, không cần SDK đặc biệt |
| **Access policy** (Chính sách truy cập) | Resource-based policy + IAM roles kiểm soát quyền truy cập |
| **CORS** (Cross-Origin Resource Sharing — Chia sẻ tài nguyên đa nguồn gốc) | Cấu hình cho phép web player đọc trực tiếp từ trình duyệt |
| **Lifecycle policy** (Chính sách vòng đời) | Tự động xoá object hết hạn, dọn dẹp live buffer |
| **Metrics** (Số liệu theo dõi) | CloudWatch metrics: request count, latency, error rate |

---

## 2. Kiến Trúc Tổng Quan

### Các Thành Phần Chính

```
┌─────────────────────────────────────────────────────────────────────┐
│                    AWS ELEMENTAL MEDIASTORE                         │
│                                                                     │
│  ┌─────────────────────────────────────────────────────────────┐   │
│  │                        Container                             │   │
│  │   my-live-stream-container                                   │   │
│  │                                                              │   │
│  │   /live/                                                     │   │
│  │   ├── index.m3u8          ← Manifest (cập nhật liên tục)     │   │
│  │   ├── segment_001.ts      ← Fragment (ghi bởi encoder)       │   │
│  │   ├── segment_002.ts                                         │   │
│  │   ├── segment_003.ts      (mới nhất, vừa được ghi)           │   │
│  │   └── ...                                                    │   │
│  │                                                              │   │
│  │   Container Endpoint:                                        │   │
│  │   https://<id>.data.mediastore.ap-southeast-1.amazonaws.com  │   │
│  └─────────────────────────────────────────────────────────────┘   │
│                                                                     │
│  ┌─────────────────┐    ┌──────────────────────────────────────┐   │
│  │  Access Policy  │    │         Lifecycle Policy             │   │
│  │  (ai được đọc/  │    │  Xoá objects cũ hơn N giây/phút     │   │
│  │   ghi)          │    │  → Giữ rolling buffer cho live       │   │
│  └─────────────────┘    └──────────────────────────────────────┘   │
│                                                                     │
│  ┌─────────────────┐                                               │
│  │  CORS Policy    │                                               │
│  │  (cho phép web  │                                               │
│  │   player truy   │                                               │
│  │   cập trực tiếp)│                                               │
│  └─────────────────┘                                               │
└─────────────────────────────────────────────────────────────────────┘
         ▲ PUT (ghi)              ▼ GET (đọc)
    MediaLive / Encoder       CloudFront hoặc Packager
```

### Luồng Dữ Liệu (Data Flow) — Live Streaming

```
1. MediaLive encode → ghi fragment vào MediaStore bằng HTTP PUT
   PUT /live/segment_001.ts (mỗi ~2 giây)
   PUT /live/index.m3u8    (cập nhật manifest sau mỗi segment)

2. CloudFront cache manifest & segments từ MediaStore
   GET /live/index.m3u8
   GET /live/segment_001.ts

3. Viewer nhận HLS manifest → player request từng segment
   Player ABR — Adaptive Bitrate tự chọn rendition phù hợp băng thông

4. Lifecycle policy tự xoá segment cũ (ví dụ: cũ hơn 300 giây)
   → Chỉ giữ rolling window, tiết kiệm storage
```

### Luồng Dữ Liệu (Data Flow) — MediaStore Làm Buffer Trung Gian

```
MediaLive
   │ ghi TS fragments
   ▼
MediaStore (buffer trung gian)
   │ packager/origin đọc
   ▼
Packager tự dựng (Nginx + packager, Wowza, v.v.)
   │ JIT-đóng gói HLS/DASH
   ▼
CloudFront CDN
   │
   ▼
Viewer
```

---

## 3. Các Khái Niệm Cốt Lõi

### 3.1 Container (Thùng Chứa)

**Container** là đơn vị lưu trữ cô lập trong MediaStore — tương tự S3 Bucket. Mỗi container có:

- **Endpoint URL** duy nhất: `https://<id>.data.mediastore.<region>.amazonaws.com`
- **Namespace** riêng: path `/` đến tối đa 10 cấp thư mục lồng nhau
- **Chính sách truy cập** (access policy) riêng biệt
- **CORS configuration** riêng biệt
- **Lifecycle policy** riêng biệt

**Giới hạn quan trọng:**
- Tối đa **100 containers** mỗi tài khoản AWS (có thể request tăng)
- Kích thước object tối đa: **25 MB** — quan trọng khi chọn segment duration
- Tên container: 1–255 ký tự, chỉ chữ thường, số, gạch ngang

```
Container: my-live-stream
  │
  ├── /sports/football/         ← Thư mục theo chương trình
  │   ├── manifest.m3u8
  │   ├── seg_001.ts
  │   └── seg_002.ts
  │
  └── /news/evening/            ← Thư mục theo kênh
      ├── manifest.m3u8
      └── seg_001.ts
```

### 3.2 Access Policy (Chính Sách Truy Cập)

**Access policy** là resource-based policy (chính sách dựa trên tài nguyên) gắn trực tiếp vào container — tương tự S3 Bucket Policy. Kết hợp với IAM roles để kiểm soát quyền:

| Loại Quyền | Ví Dụ Use Case |
|-----------|----------------|
| **PutObject** | MediaLive/encoder ghi fragment vào container |
| **GetObject** | CloudFront/packager đọc segment để phân phối |
| **DeleteObject** | Lifecycle rule tự động xoá object cũ |
| **ListItems** | Debug tool liệt kê nội dung container |
| **DescribeObject** | Lấy metadata của object (size, ETag, Content-Type) |

### 3.3 CORS — Cross-Origin Resource Sharing (Chia Sẻ Tài Nguyên Đa Nguồn Gốc)

**CORS** cho phép browser web tại domain `https://myapp.com` đọc trực tiếp từ MediaStore endpoint tại domain khác (`https://<id>.data.mediastore.amazonaws.com`). Nếu không cấu hình CORS, browser sẽ chặn request và player không thể tải segments.

**Cấu hình CORS thường dùng cho web player:**
```json
{
  "CorsPolicy": [
    {
      "AllowedOrigins": ["https://myapp.com"],
      "AllowedMethods": ["GET", "HEAD"],
      "AllowedHeaders": ["*"],
      "MaxAgeSeconds": 3000,
      "ExposeHeaders": ["ETag"]
    }
  ]
}
```

### 3.4 Lifecycle Policy (Chính Sách Vòng Đời)

**Lifecycle policy** tự động xoá objects sau một khoảng thời gian (tính bằng giây). Đây là tính năng thiết yếu cho live streaming vì:

- Encoder liên tục ghi segment mới → container sẽ phình to nếu không dọn dẹp
- Live buffer chỉ cần giữ một cửa sổ thời gian (ví dụ: 5 phút gần nhất)
- Giảm chi phí lưu trữ không cần thiết

**Ví dụ:** Giữ rolling buffer 5 phút (300 giây):
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

### 3.5 Metrics và Monitoring (Giám Sát)

MediaStore tích hợp CloudWatch Metrics — Số Liệu CloudWatch tự động:

| Metric | Ý Nghĩa |
|--------|---------|
| `RequestCount` | Tổng số request (PUT + GET + DELETE) |
| `4xxErrorCount` | Request lỗi do client (xác thực sai, không tìm thấy) |
| `5xxErrorCount` | Request lỗi do server |
| `BytesUploaded` | Tổng byte được ghi vào container |
| `BytesDownloaded` | Tổng byte được đọc từ container |
| `TotalTime` | Tổng thời gian xử lý request (ms) |
| `TurnAroundTime` | Thời gian từ lúc nhận request đến lúc bắt đầu trả response (ms) |

---

## 4. Luồng Xử Lý

### Use Case 1: MediaStore Làm Origin Cho Live HLS

```
[OBS Studio / Encoder phần cứng]
  │ RTMP push
  ▼
[MediaLive Channel]
  │ encode multi-bitrate (720p, 480p, 360p)
  │ ghi HLS segments vào MediaStore
  ▼
[MediaStore Container: my-live-bucket]
  /hls/720p/index.m3u8      ← MediaLive cập nhật mỗi 2–6 giây
  /hls/720p/seg_001.ts
  /hls/480p/index.m3u8
  /hls/480p/seg_001.ts
  /master.m3u8               ← MediaLive ghi master playlist

  Lifecycle: xoá segment cũ hơn 300 giây
  │
  ▼
[CloudFront Distribution]
  Origin: MediaStore container endpoint
  Cache behavior: /hls/*.m3u8 → TTL thấp (2–5s), /hls/*.ts → TTL cao (60s)
  │
  ▼
[Viewer — HLS.js / iOS Player]
  Đọc master.m3u8 → chọn rendition phù hợp → request segments liên tục
```

### Use Case 2: MediaStore Làm Buffer Cho Packager Bên Thứ Ba

```
[MediaLive] → ghi TS fragments → [MediaStore]
                                        │
                                        │ Packager bên thứ ba
                                        │ (Wowza, Nimble Streamer, v.v.) đọc TS
                                        ▼
                                 [Packager tự dựng trên EC2]
                                        │ đóng gói DASH/HLS/CMAF
                                        │ + xử lý DRM riêng
                                        ▼
                                 [CloudFront] → [Viewer]
```

### Use Case 3: MediaStore Cho Live-to-VOD Workflow

```
[Live Event]
  MediaLive → MediaStore (ghi toàn bộ event: 3 giờ)
                    │
                    │ Sau khi event kết thúc
                    ▼
              [Lambda Function]
              Đọc segments từ MediaStore
              Ghép lại thành file MP4 đầy đủ
                    │
                    ▼
              [S3 Bucket] (lưu trữ VOD dài hạn)
                    │
                    ▼
              [MediaConvert] (re-encode nếu cần)
                    │
                    ▼
              [CloudFront] → Viewer xem lại (on-demand)
```

---

## 5. Nội Dung Chi Tiết

```
05-mediastore/
├── README.md                      ← [BẠN ĐANG Ở ĐÂY] Tổng quan
├── 1-container-access-policy.md   Container, IAM policy, CORS configuration
├── 2-lifecycle-policy.md          Lifecycle rules, xoá fragment tự động
└── 3-mediastore-vs-s3.md          So sánh MediaStore và S3 cho media workloads
```

### Bản Đồ Học Tập Đề Xuất

```
Bắt đầu tại đây (README)
         │
         ▼
1-container-access-policy.md    ← Hiểu Container và cách kiểm soát truy cập
         │
         ▼
2-lifecycle-policy.md           ← Cấu hình tự động dọn dẹp buffer cho live
         │
         ▼
3-mediastore-vs-s3.md           ← Khi nào dùng MediaStore, khi nào dùng S3
```

---

## 6. Câu Hỏi Phỏng Vấn Thường Gặp

### Câu hỏi cơ bản

**Q: MediaStore là gì và nó khác S3 ở điểm nào?**

> **MediaStore** là object storage tối ưu cho **media workloads** — đặc biệt là live streaming. So với S3:
> - **Độ trễ thấp hơn**: MediaStore cho consistent low-latency (nhất quán, thường < 10ms) khi ghi/đọc fragment nhỏ tần suất cao. S3 có thể có latency spike đến hàng trăm ms trong điều kiện load cao.
> - **Không hỗ trợ versioning, replication, lifecycle rules phức tạp** như S3
> - **Giới hạn kích thước object**: 25 MB (S3 hỗ trợ đến 5 TB)
> - **Chi phí cao hơn S3** một chút, nhưng đổi lại latency tốt hơn cho live use case
>
> Dùng MediaStore cho **live streaming buffer**; dùng S3 cho **VOD storage**, **archive**, và bất kỳ nội dung nào không cần low-latency ghi liên tục.

**Q: Tại sao cần Lifecycle Policy trong MediaStore?**

> Trong live streaming, encoder (MediaLive) ghi fragment mới liên tục — cứ mỗi 2–6 giây một segment. Nếu không có lifecycle policy, container sẽ tích tụ hàng nghìn segment cũ không còn cần thiết, dẫn đến:
> - **Tốn chi phí lưu trữ** không cần thiết
> - **Khó quản lý** namespace trong container
>
> Lifecycle policy với rule `seconds_since_create > 300` sẽ tự động xoá segment cũ hơn 5 phút — giữ **rolling buffer** — bộ đệm cuộn chỉ chứa nội dung gần đây nhất. Đây là pattern phổ biến nhất khi dùng MediaStore cho live streaming.

**Q: Khi nào nên dùng MediaStore thay vì MediaPackage?**

> Dùng **MediaStore** khi:
> - Bạn đã có **packager tự dựng** (Wowza, Nimble, Nginx + packager) và chỉ cần storage nhanh
> - Cần **buffer trung gian** độ trễ thấp giữa encoder và packager
> - Muốn **kiểm soát hoàn toàn** quá trình packaging, DRM, manifest generation
> - Workload **IoT/edge** cần ghi video fragment từ nhiều nguồn
>
> Dùng **MediaPackage** khi:
> - Cần **managed packaging** (HLS + DASH + CMAF từ một nguồn)
> - Cần **DRM tích hợp** (SPEKE — Secure Packager and Encoder Key Exchange) mà không muốn tự xây
> - Cần **time-shift** (startover, catch-up TV) sẵn có
> - Muốn **giải pháp trọn gói** không phải tự quản lý infrastructure

### Câu hỏi nâng cao

**Q: Cấu hình CloudFront TTL — Time-To-Live (Thời Gian Sống Cache) cho MediaStore origin như thế nào?**

> Cho live HLS từ MediaStore, cần cân bằng freshness vs performance:
> - **Manifest** (`.m3u8`): TTL thấp = 2–5 giây, vì manifest cập nhật liên tục theo mỗi segment mới
> - **Segments** (`.ts`, `.m4s`): TTL cao = 60–300 giây, vì segment đã ghi không bao giờ thay đổi
> - **Master playlist**: TTL trung bình = 30–60 giây
>
> Nếu TTL manifest quá cao, viewer sẽ nhận manifest cũ → không thấy segment mới → stream bị treo.
> Nếu TTL manifest quá thấp, CloudFront sẽ về MediaStore thường xuyên → tăng cost và latency.

**Q: Làm thế nào để bảo mật MediaStore container chỉ cho phép MediaLive ghi và CloudFront đọc?**

> Dùng **resource-based access policy** kết hợp **IAM conditions**:
> 1. **MediaLive ghi**: IAM role của MediaLive channel có `mediastore:PutObject` permission, policy gắn vào container allow role ARN đó
> 2. **CloudFront đọc**: Dùng **OAC** — Origin Access Control — Kiểm Soát Truy Cập Origin của CloudFront; MediaStore access policy chỉ allow `aws:PrincipalServiceName = cloudfront.amazonaws.com` với điều kiện `aws:SourceArn` là distribution ARN cụ thể
> 3. **Block public access**: Container access policy **không** có `Principal: "*"`, tránh truy cập ẩn danh
> 4. Bật **CloudWatch Logs** để audit tất cả request vào container

---

## 🔗 Tài Liệu Tham Khảo

- [MediaStore User Guide](https://docs.aws.amazon.com/mediastore/latest/ug/)
- [MediaStore Pricing](https://aws.amazon.com/mediastore/pricing/)
- [MediaStore Container Access Policy](https://docs.aws.amazon.com/mediastore/latest/ug/policies.html)
- [Setting Up CORS for MediaStore](https://docs.aws.amazon.com/mediastore/latest/ug/cors-policy.html)
- [MediaStore Lifecycle Policy](https://docs.aws.amazon.com/mediastore/latest/ug/policies-object-lifecycle.html)

---

**Cập Nhật Lần Cuối:** 2026-06-04
**Phần Trước:** [04-mediapackage/](../04-mediapackage/README.md) — Packaging & DRM
**Phần Tiếp Theo:** [06-mediatailor/](../06-mediatailor/README.md) — Ad Insertion
