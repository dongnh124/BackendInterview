# Kinesis Video Streams — WebRTC & Signaling Channel

> **WebRTC** — Web Real-Time Communication — Giao Tiếp Thời Gian Thực Trên Web cho phép truyền audio/video **peer-to-peer** (P2P — ngang hàng) độ trễ thấp. **KVS WebRTC** cung cấp **Signaling Channel** — Kênh Báo Hiệu — managed để trao đổi **SDP** và **ICE**, cùng **STUN/TURN** — NAT Traversal — Vượt NAT do AWS vận hành.

## 📚 Mục Lục

1. [Tại Sao WebRTC Trên KVS?](#1-tại-sao-webrtc-trên-kvs)
2. [Kiến Trúc WebRTC KVS](#2-kiến-trúc-webrtc-kvs)
3. [Signaling Channel](#3-signaling-channel)
4. [Master vs Viewer](#4-master-vs-viewer)
5. [STUN, TURN & ICE](#5-stun-turn--ice)
6. [Ghi Song Song Vào Video Stream](#6-ghi-song-song-vào-video-stream)
7. [Thực Hành & Bảo Mật](#7-thực-hành--bảo-mật)
8. [Câu Hỏi Phỏng Vấn](#8-câu-hỏi-phỏng-vấn)

---

## 1. Tại Sao WebRTC Trên KVS?

### Use Case Điển Hình

| Use case | Vai trò WebRTC |
|----------|----------------|
| **Doorbell / camera hai chiều** | Homeowner nói chuyện với visitor |
| **Telemedicine** | Bác sĩ xem + tư vấn bệnh nhân tại nhà |
| **Robot điều khiển từ xa** | Operator xem feed + gửi lệnh realtime |
| **Drone FPV** | Pilot xem video sub-second (kèm latency mạng) |

**KVS WebRTC** không thay **IVS** cho livestream hàng triệu người — mà cho **session 1:1 hoặc vài peer** với độ trễ cực thấp.

### So Sánh WebRTC KVS vs IVS Real-Time

| Tiêu chí | KVS WebRTC | IVS Real-Time (Stage) |
|----------|------------|------------------------|
| **Nguồn** | Embedded camera, custom app | Web/mobile app publish |
| **Signaling** | KVS Signaling Channel | IVS Stage API |
| **Scale viewer** | Nhỏ (vài peer) | Nhiều subscriber |
| **Ghi archive** | Tích hợp KVS Stream | IVS Recording → S3 |
| **Tích hợp ML** | Rekognition trên cùng stream | Tự xây |

---

## 2. Kiến Trúc WebRTC KVS

```
┌─────────────────────────────────────────────────────────────────────────┐
│                     KVS WEBRTC ARCHITECTURE                             │
│                                                                         │
│   ┌──────────────┐                              ┌──────────────┐       │
│   │   Master     │         Signaling          │   Viewer     │       │
│   │  (Web/App)   │◀────── Channel ────────────▶│  (Device)    │       │
│   │              │   SDP offer/answer          │              │       │
│   │              │   ICE candidates            │              │       │
│   └──────┬───────┘                              └──────┬───────┘       │
│          │                                              │               │
│          │         ┌────────────────────────┐           │               │
│          └────────▶│  STUN / TURN (AWS)     │◀──────────┘               │
│                    │  NAT traversal         │                           │
│                    └────────────────────────┘                           │
│          │                    │                    │                   │
│          └────────────────────┼────────────────────┘                   │
│                               ▼                                        │
│                    Media path (SRTP — Secure RTP)                      │
│                    P2P hoặc relay qua TURN                             │
│                               │                                        │
│                               ▼ (optional)                           │
│                    ┌─────────────────────┐                             │
│                    │  KVS Video Stream   │  PutMedia ghi session       │
│                    └─────────────────────┘                             │
└─────────────────────────────────────────────────────────────────────────┘
```

### Luồng Thiết Lập Phiên (Session Setup)

```
1. Master và Viewer connect Signaling Channel (WSS — WebSocket Secure)
2. Master gửi SDP Offer → Viewer nhận qua signaling
3. Viewer trả SDP Answer → Master
4. Hai bên trao đổi ICE candidates
5. ICE connectivity check → chọn path P2P hoặc TURN relay
6. Media flows (H.264/Opus trong SRTP)
7. (Tuỳ chọn) Viewer/Master ghi vào KVS Stream qua Producer
```

---

## 3. Signaling Channel

### Signaling Channel Là Gì?

**Signaling Channel** là tài nguyên KVS riêng biệt với **Video Stream** — chỉ dùng để **báo hiệu** (không chứa video file). Tương tự "phòng chat" cho WebRTC negotiation.

```
Signaling Channel: doorbell-channel-42
├── ARN: arn:aws:kinesisvideo:region:account:channel/doorbell-channel-42/...
├── Message retention: (signaling message TTL)
└── IAM: kinesisvideo:ConnectAsMaster / ConnectAsViewer
```

### API Signaling Chính

| API | Mô tả |
|-----|-------|
| `CreateSignalingChannel` | Tạo channel |
| `GetSignalingChannelEndpoint` | WSS URL để connect |
| `ConnectAsMaster` | Role master join |
| `ConnectAsViewer` | Role viewer join |
| `SendAlexaOfferToMaster` | Pattern tích hợp Alexa (tuỳ ecosystem) |

**WSS** — WebSocket Secure — kết nối persistent giữa client và KVS signaling service.

---

## 4. Master vs Viewer

### Định Nghĩa Role

| Role | Thường là | Hành vi |
|------|-----------|---------|
| **Master** | Web dashboard, mobile app người giám sát | Thường **nhận** offer, có thể gửi lệnh điều khiển |
| **Viewer** | Camera, doorbell, robot | Thường **gửi** video lên master |

> Trong tài liệu KVS, **Viewer** = thiết bị có camera (ngược intuition "người xem" broadcast).

```
Doorbell scenario:
  Viewer = Doorbell firmware (camera + mic)
  Master = Homeowner app trên điện thoại
```

### Single Master Channel

Một signaling channel thường **một Master active** tại một thời điểm — nhiều viewer device có thể pending nhưng master session điều phối pairing.

### Credentials

| Pattern | Mô tả |
|---------|-------|
| **Cognito Identity Pool** | Mobile/web lấy temp creds ConnectAsMaster |
| **IoT certificate** | Device ConnectAsViewer |
| **IAM user** | Chỉ dev/test |

---

## 5. STUN, TURN & ICE

### ICE — Interactive Connectivity Establishment

**ICE** thử nhiều **candidate** — đường kết nối khả dĩ (local IP, public IP qua STUN, relay TURN) để tìm path tốt nhất.

```
Candidates exchange qua Signaling Channel:
  • host candidate: 192.168.x.x (LAN)
  • srflx candidate: public IP qua STUN
  • relay candidate: TURN server relay
```

### STUN vs TURN

| Dịch vụ | Vai trò | KVS |
|---------|---------|-----|
| **STUN** — Session Traversal Utilities for NAT | Khám phá public IP, hole punching P2P | AWS cung cấp endpoint theo region |
| **TURN** — Traversal Using Relays around NAT | Relay media khi P2P thất bại (symmetric NAT, firewall) | Billing theo phút relay |

```
P2P thành công (ưu tiên):
  Master ◀──────── direct ────────▶ Viewer

P2P thất bại:
  Master ◀──▶ TURN ◀──▶ Viewer  (tăng latency + chi phí)
```

### SRTP

Media sau khi ICE connect được mã hoá **SRTP** — Secure Real-time Transport Protocol — không đi qua signaling channel (chỉ metadata SDP/ICE đi qua signaling).

---

## 6. Ghi Song Song Vào Video Stream

### Pattern: WebRTC + Archive

```
WebRTC live view (sub-second)  +  PutMedia ghi KVS Stream (vài giây delay)
                                      │
                                      ▼
                              Retention 24h → Rekognition / playback HLS
```

**Lợi ích:** Người dùng xem realtime qua WebRTC; compliance/ML dùng bản ghi trên stream.

### Cấu Hình

- Device **Viewer** vừa publish WebRTC vừa chạy **Producer SDK** ghi cùng encode pipeline
- Hoặc Master relay (ít phổ biến trên edge vì tốn pin/bandwidth)

---

## 7. Thực Hành & Bảo Mật

### Tạo Signaling Channel (CLI)

```bash
aws kinesisvideo create-signaling-channel \
  --channel-name doorbell-signaling-01 \
  --region ap-southeast-1
```

### Checklist Production

| Mục | Khuyến nghị |
|-----|-------------|
| **Auth** | Cognito/IoT, không hardcode IAM key trên device |
| **TURN** | Monitor usage — symmetric NAT cao → chi phí TURN tăng |
| **HTTPS/WSS only** | Không downgrade signaling |
| **Channel per device** | Tránh cross-tenant signaling |
| **Rate limit** | Retry exponential backoff khi signaling disconnect |

### SDK & Sample

- **KVS WebRTC SDK** (C): tích hợp embedded Linux
- **JavaScript SDK**: web master dashboard
- Sample: [AWS KVS WebRTC repo](https://github.com/awslabs/amazon-kinesis-video-streams-webrtc-sdk)

### Troubleshooting

| Vấn đề | Nguyên nhân |
|--------|-------------|
| ICE failed | Firewall chặn UDP, cần TURN |
| Black screen | Codec mismatch (H.264 profile), SDP không khớp |
| One-way audio | Permission mic, SDP direction |
| Signaling disconnect | Token hết hạn, refresh Cognito creds |

---

## 8. Câu Hỏi Phỏng Vấn

**Q: Signaling Channel khác Video Stream thế nào?**

> **Signaling Channel** chỉ trao đổi **SDP/ICE** để thiết lập WebRTC. **Video Stream** lưu **fragment** qua PutMedia cho playback/ML. Một sản phẩm doorbell dùng **cả hai**: realtime qua WebRTC, archive qua Video Stream.

**Q: KVS WebRTC có scale như CDN không?**

> Không. WebRTC là **mesh P2P/TURN** — phù hợp session nhỏ. Scale broadcast dùng **IVS** hoặc **MediaPackage + CloudFront**.

**Q: Tại sao cần TURN nếu đã có STUN?**

> **STUN** chỉ giúp discovery — không relay media. **Symmetric NAT** hoặc corporate firewall thường chặn P2P → **TURN** bắt buộc để media đi qua relay AWS.

---

**Cập Nhật Lần Cuối:** 2026-06-04
**Phần Trước:** [1-producer-consumer.md](./1-producer-consumer.md)
**Tiếp Theo:** [3-rekognition-integration.md](./3-rekognition-integration.md)
