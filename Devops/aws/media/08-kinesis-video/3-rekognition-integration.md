# Kinesis Video Streams — Tích Hợp Amazon Rekognition

> **Amazon Rekognition Video** phân tích nội dung video bằng **ML** — Machine Learning. Với **KVS**, **Stream Processor** — Bộ Xử Lý Luồng — đọc trực tiếp từ **Video Stream** để phát hiện **face**, **label**, **celebrity**, **content moderation** realtime mà không cần tự decode H.264 trong Lambda.

## 📚 Mục Lục

1. [Tại Sao KVS + Rekognition?](#1-tại-sao-kvs--rekognition)
2. [Kiến Trúc Stream Processor](#2-kiến-trúc-stream-processor)
3. [Các Loại Processor](#3-các-loại-processor)
4. [IAM & Thiết Lập](#4-iam--thiết-lập)
5. [Xử Lý Kết Quả (Events)](#5-xử-lý-kết-quả-events)
6. [So Sánh Với GetMedia + Custom ML](#6-so-sánh-với-getmedia--custom-ml)
7. [Thực Hành & Best Practices](#7-thực-hành--best-practices)
8. [Câu Hỏi Phỏng Vấn](#8-câu-hỏi-phỏng-vấn)

---

## 1. Tại Sao KVS + Rekognition?

### Bài Toán

```
Camera fleet → cần biết:
  • Có người lạ trong khu vực cấm không?     → Face detection / search
  • Có lửa / xe / vật nguy hiểm không?       → Label detection
  • Nội dung video có vi phạm policy không?  → Content moderation
```

Tự xây pipeline **GetMedia → decode → inference** tốn engineering và GPU. **Rekognition Stream Processor** là managed consumer của KVS.

### Pipeline Tổng Quan

```
┌────────────┐  PutMedia   ┌─────────────┐  Read stream   ┌──────────────────┐
│  Camera    │ ──────────▶ │ KVS Stream  │ ─────────────▶ │ Rekognition      │
│  + Producer│             │             │                │ Stream Processor │
└────────────┘             └─────────────┘                └────────┬─────────┘
                                                                 │
                    ┌────────────────────────────────────────────┼────────────┐
                    ▼                    ▼                       ▼            ▼
            Kinesis Data Streams      SNS                  S3 (optional)   CloudWatch
                    │
                    ▼
              Lambda → alert / DynamoDB / Step Functions
```

---

## 2. Kiến Trúc Stream Processor

### Stream Processor Là Gì?

**Stream Processor** là tài nguyên Rekognition gắn **một KVS stream ARN** làm input và **một hoặc nhiều output** (Kinesis Data Stream, KVS stream khác, SNS).

```
StreamProcessor: factory-face-detector
├── Input: KVS ARN (camera-line-3)
├── Output: Kinesis Data Stream (rekognition-events)
├── Status: RUNNING | STOPPED | FAILED
└── Settings: FaceDetection | ConnectedHome (label) | ...
```

### Vòng Đời Processor

| Trạng thái | Mô tả |
|------------|-------|
| `CREATE` | Đang khởi tạo |
| `START` / `RUNNING` | Đang đọc KVS và phân tích |
| `STOP` | Dừng — không billing inference |
| `FAILED` | Lỗi IAM, stream không tồn tại, quota |

> **Lưu ý:** Processor **phải START** sau khi tạo; thay đổi output thường cần stop → update → start.

### Rekognition Là Consumer Của KVS

Rekognition dùng quyền đọc stream (tương tự GetMedia nội bộ) — bạn **không** trả thêm KVS data out cho path này theo cùng pattern consumer app (vẫn có chi phí Rekognition riêng).

---

## 3. Các Loại Processor

### Face Detection

| Tính năng | Mô tả |
|-----------|-------|
| **Detect faces** | Bounding box, confidence trên frame |
| **Face search** | So khớp với **Collection** — Bộ Sưu Tập Khuôn Mặt đã index |
| **User match** | `FaceId` nếu khớp collection |

**Use case:** access control, known person alert, retail analytics.

### Label Detection (Connected Home / General)

**Label detection** nhận diện object/scene: `Person`, `Car`, `Package`, `Pet`, v.v.

```
Event ví dụ (khái niệm):
{
  "Label": "Person",
  "Confidence": 98.2,
  "Timestamp": 1717500000000,
  "BoundingBox": { ... }
}
```

**Use case:** smart home, warehouse safety (forklift + person).

### Celebrity Recognition

Nhận diện người nổi tiếng có trong model — ít dùng industrial, nhiều media.

### Content Moderation

Phát hiện nội dung nhạy cảm (explicit) — moderation pipeline UGC — User-Generated Content.

### So Sánh Nhanh

| Processor type | Input KVS | Output phổ biến | Độ trễ |
|----------------|-----------|-----------------|--------|
| Face detection | H.264 stream | Kinesis Data Streams | Vài giây |
| Label detection | H.264 stream | SNS / Lambda | Vài giây |
| Custom ML (SageMaker) | GetMedia tự decode | Tự thiết kế | Tuỳ model |

---

## 4. IAM & Thiết Lập

### Role Rekognition → KVS

**IAM Role** gắn với Rekognition service principal, trust policy cho `rekognition.amazonaws.com`:

```json
{
  "Effect": "Allow",
  "Action": [
    "kinesisvideo:GetDataEndpoint",
    "kinesisvideo:GetMedia",
    "kinesisvideo:DescribeStream"
  ],
  "Resource": "arn:aws:kinesisvideo:*:*:stream/camera-*/*"
}
```

### Role Rekognition → Kinesis Data Streams (Output)

```json
{
  "Effect": "Allow",
  "Action": [
    "kinesis:PutRecord",
    "kinesis:PutRecords"
  ],
  "Resource": "arn:aws:kinesis:region:account:stream/rekognition-output"
}
```

### Tạo Processor (CLI — khái niệm)

```bash
# Bước 1: Tạo processor (face detection example)
aws rekognition create-stream-processor \
  --input-kinesis-video-stream arn:aws:kinesisvideo:region:acct:stream/my-stream/... \
  --output-kinesis-data-stream arn:aws:kinesis:region:acct:stream/events \
  --name face-processor-01 \
  --role-arn arn:aws:iam::acct:role/RekognitionKVSRole \
  --settings '{"FaceSearch": {"CollectionId": "employees", "FaceMatchThreshold": 85}}'

# Bước 2: Start processor
aws rekognition start-stream-processor --name face-processor-01
```

### Face Collection (Cho Search)

Trước khi **Face search** hoạt động:

```
1. create-collection (CollectionId)
2. index-faces từ ảnh S3 (nhân viên đã đăng ký)
3. Stream processor FaceSearch trỏ CollectionId
```

**Collection** lưu vector khuôn mặt — không lưu ảnh raw trong processor.

---

## 5. Xử Lý Kết Quả (Events)

### Output Qua Kinesis Data Streams

```
Rekognition → PutRecord → Kinesis Data Stream
                                │
                                ▼
                         Lambda (event source mapping)
                                │
                    ┌───────────┼───────────┐
                    ▼           ▼           ▼
                 SNS alert   DynamoDB    Step Functions
```

**Lambda** parse JSON event — filter `Confidence > threshold` — gửi **SNS** — Simple Notification Service — hoặc ticket PagerDuty.

### Output Qua SNS (Trực Tiếp)

Một số cấu hình gửi thẳng **SNS Topic** — phù hợp ít xử lý, nhiều subscriber email/SMS.

### Event Structure (Khái Niệm Face)

```json
{
  "InputInformation": {
    "KinesisVideo": {
      "StreamArn": "arn:aws:kinesisvideo:..."
    }
  },
  "FaceSearchResponse": {
    "DetectedFaces": [...],
    "MatchedFaces": [
      {
        "Face": { "FaceId": "uuid", "Confidence": 99.1 },
        "Similarity": 92.5
      }
    ]
  }
}
```

### Idempotency & Duplicate

Processor có thể emit **nhiều event** cho cùng đối tượng qua nhiều frame — Lambda nên **dedupe** (window time + FaceId) tránh alert spam.

---

## 6. So Sánh Với GetMedia + Custom ML

| Tiêu chí | Rekognition Stream Processor | GetMedia + SageMaker/OpenCV |
|----------|------------------------------|-----------------------------|
| **Setup** | Vài API call | Build decode, scale, model hosting |
| **Model** | AWS pretrained | Custom model |
| **Chi phí** | Per-minute analyzed video | EC2/GPU + KVS data out |
| **Latency** | Vài giây | Có thể thấp hơn nếu tối ưu edge |
| **Privacy** | Video gửi AWS Rekognition | Có thể on-prem inference |

**Khi dùng custom:** defect detection sản xuất, model độc quyền, cần inference trên edge trước khi gửi cloud.

---

## 7. Thực Hành & Best Practices

### Thiết Kế Hệ Thống

| Khuyến nghị | Lý do |
|-------------|-------|
| **1 processor / 1 stream** (hoặc nhóm logic rõ) | Debug và IAM least privilege |
| **Filter confidence ở Lambda** | Giảm false positive alert |
| **Separate dev/prod collection** | Tránh nhầm FaceId |
| **Monitor processor FAILED** | CloudWatch alarm |
| **Retention KVS ≥ thời gian điều tra** | Replay sau sự cố |

### Chi Phí

- **Rekognition Video**: tính theo **phút video phân tích**
- **KVS**: ingest + storage retention
- **Kinesis Data Streams**: shard hours + PUT payload
- **Lambda**: invocation khi có event

Dùng **AWS Pricing Calculator** với số camera × giờ hoạt động/ngày.

### Giới Hạn & Compliance

- Video có khuôn mặt có thể thuộc **GDPR** / luật bảo vệ dữ liệu cá nhân — cần consent, retention policy
- **Rekognition** không dùng cho quyết định pháp lý tự động không giám sát con người (AWS AUP — Acceptable Use Policy)
- Region availability: một số tính năng Rekognition không có ở mọi region

### Kết Hợp Với Các Dịch Vụ Khác

```
KVS + Rekognition (realtime alert)
         │
         └── Clip sự cố → Lambda GetMedia slice → S3 archive (bằng chứng)
         └── MediaConvert (nếu cần transcode clip cho review portal)
```

---

## 8. Câu Hỏi Phỏng Vấn

**Q: Rekognition đọc KVS như thế nào?**

> Tạo **Stream Processor** với input là **KVS Stream ARN** và IAM role cho Rekognition quyền đọc stream. Sau **start-stream-processor**, Rekognition consume fragment tương tự **GetMedia**, chạy model, ghi kết quả ra **Kinesis Data Streams** hoặc **SNS**.

**Q: Có cần Lambda decode video không?**

> **Không** nếu dùng processor có sẵn (face/label/moderation). **Có** nếu custom model — thường **GetMedia** hoặc **S3** (sau archive) làm input SageMaker.

**Q: Face search khác face detection?**

> **Detection** tìm khuôn mặt trong frame. **Search** so sánh với **Collection** đã **index-faces** — trả về **FaceId** nhân viên đã đăng ký nếu similarity > threshold.

**Q: Processor dừng khi camera offline?**

> Processor vẫn **RUNNING** nhưng không có fragment mới — không event. Nên **stop processor** khi bảo trì fleet để tránh chi phí idle tùy pricing model; hoặc automation theo schedule.

---

**Cập Nhật Lần Cuối:** 2026-06-04
**Phần Trước:** [2-webrtc-signaling.md](./2-webrtc-signaling.md)
**Tiếp Theo:** [4-retention-lifecycle.md](./4-retention-lifecycle.md)
