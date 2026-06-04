# Amazon IVS Chat — Phòng Chat Realtime

> **IVS Chat** là dịch vụ messaging realtime độc lập, thiết kế đi kèm live stream IVS: **Room** (phòng), token xác thực, **Messaging API**, và hook moderation. Không trộn vào video pipeline nhưng latency thấp và scale theo số viewer chat.

## 📚 Mục Lục

1. [IVS Chat Là Gì?](#1-ivs-chat-là-gì)
2. [Kiến Trúc Room & Token](#2-kiến-trúc-room--token)
3. [Messaging API & SDK](#3-messaging-api--sdk)
4. [Moderation & Bảo Mật](#4-moderation--bảo-mật)
5. [Thực Hành Triển Khai](#5-thực-hành-triển-khai)
6. [Câu Hỏi Phỏng Vấn](#6-câu-hỏi-phỏng-vấn)

---

## 1. IVS Chat Là Gì?

### Tách Biệt Video Pipeline

```
┌─────────────────┐          ┌─────────────────┐
│  IVS Channel    │          │  IVS Chat Room  │
│  (video HLS)    │          │  (WebSocket)    │
└────────┬────────┘          └────────┬────────┘
         │                            │
         ▼                            ▼
    IVS Player SDK              Chat SDK (JS/mobile)
         │                            │
         └──────────┬─────────────────┘
                    ▼
              Viewer Application
```

| Thành phần | Region | Billing |
|------------|--------|---------|
| IVS Low-Latency | ví dụ `ap-northeast-1` | input/output video |
| IVS Chat | **riêng** (tạo room cùng region khuyến nghị) | messages, connections |

> Tạo **Room** cùng region với channel để giảm latency chat–video perceived.

### So Với Tự Xây WebSocket

| | IVS Chat | API Gateway WebSocket + Lambda |
|---|----------|-------------------------------|
| Scale | Managed | Tự thiết kế |
| Moderation API | Có sẵn | Tự xây |
| Tích hợp IVS | Official patterns | Tự map session |
| Chi phí | Theo message/connection | EC2/Lambda + ops |

---

## 2. Kiến Trúc Room & Token

### Room

**Room** là container cho messages trong một live session:

```bash
aws ivschat create-room \
  --name "live-shopping-room-001" \
  --maximum-message-rate-per-second 10 \
  --maximum-message-length 500 \
  --region ap-northeast-1
```

| Cấu hình | Ý nghĩa |
|----------|---------|
| `maximumMessageRatePerSecond` | Chống spam flood |
| `maximumMessageLength` | Giới hạn ký tự mỗi tin |
| `loggingConfigurationIdentifiers` | Gửi log tới CloudWatch/S3 (tuỳ chọn) |

### Token Flow (Không Dùng IAM Trên Client)

```
┌──────────┐  1. Login app     ┌──────────────┐
│  Viewer  │ ────────────────▶ │   Backend    │
└──────────┘                   │  (IAM role)  │
       ▲                       └──────┬───────┘
       │  3. Chat token               │
       │  (CreateChatToken)           │ 2. ivschat:CreateChatToken
       └──────────────────────────────┘
       │
       4. Chat SDK connect Room với token
```

**CreateChatToken** gắn:

- `userId` — id ứng dụng (không PII nhạy cảm nếu có thể)
- `attributes` — role (`viewer`, `moderator`), display name
- `capabilities` — `SEND_MESSAGE`, `DISCONNECT_USER`, ...
- TTL — thời gian sống token ngắn (vài phút–giờ)

### EventBridge (Tùy Chọn)

Room có thể gửi events tới **EventBridge** — AWS event bus — để Lambda xử lý moderation async, analytics, archive.

---

## 3. Messaging API & SDK

### Gửi / Nhận Message

Client dùng **IVS Chat Messaging SDK** (JavaScript, iOS, Android):

```javascript
// Pseudocode — pattern chính thức từ AWS docs
const chatRoom = new ChatRoom({
  regionOrUrl: 'wss://edge.ivschat.ap-northeast-1.amazonaws.com',
  tokenProvider: async () => {
    const res = await fetch('/api/chat-token');
    return res.json(); // { token: "..." }
  },
});

chatRoom.connect();
chatRoom.addListener('message', (msg) => {
  renderChatLine(msg.sender.userId, msg.content);
});

chatRoom.sendMessage('Xin chào shop!');
```

### Message Structure

```
Message
├── id
├── content (text)
├── sender { userId, attributes }
├── sendTime
└── type (SYSTEM | USER | ...)
```

**System messages** — thông báo moderator kick, room closed.

### Disconnect User

Moderator token có capability `DISCONNECT_USER` — ngắt kết nối user vi phạm mà không đợi client tự ngắt.

---

## 4. Moderation & Bảo Mật

### Chiến Lược Moderation

```
Lớp 1: Rate limit (room config)
Lớp 2: Token — chỉ user đã login app mới chat
Lớp 3: Lambda + Comprehend / custom filter (pre-send qua API)
Lớp 4: Human moderator dashboard (disconnect, delete via API)
Lớp 5: Logging → audit trail
```

### Logging

Gắn **logging configuration** để lưu messages:

- Phục vụ compliance, tranh chấp
- Replay chat cùng VOD recording
- ML training moderation (cẩn thận privacy)

### Bảo Mật Token

- **Không** embed IAM access key trong mobile app
- Backend ký **CreateChatToken** sau khi auth user
- Rotate và TTL ngắn; bind `userId` với session app

### Khác Timed Metadata

| Chat | Timed Metadata |
|------|----------------|
| Viewer-generated content | Host/backend triggered |
| Cần moderation mạnh | Payload kiểm soát server |
| Không sync frame-perfect | Sync video timeline |

---

## 5. Thực Hành Triển Khai

### Tạo Room + Token (CLI)

```bash
# Tạo room
aws ivschat create-room --name "shop-live-1" --region ap-northeast-1

# Token (thường làm từ SDK/backend — ví dụ AWS CLI với role)
aws ivschat create-chat-token \
  --room-identifier arn:aws:ivschat:ap-northeast-1:123456789012:room/AbCd \
  --user-id "user-42" \
  --attributes displayName=Viewer42 \
  --capabilities SEND_MESSAGE \
  --region ap-northeast-1
```

### Gắn Room Với Live Session

```
1 live event = 1 room (tạo mới) hoặc reuse room cố định
Khi stream end: disconnect clients, optional delete room
VOD replay: đọc chat log từ S3 thay vì live room
```

### IAM Policy Mẫu (Backend)

```json
{
  "Effect": "Allow",
  "Action": [
    "ivschat:CreateChatToken",
    "ivschat:DisconnectUser",
    "ivschat:DeleteMessage"
  ],
  "Resource": "arn:aws:ivschat:ap-northeast-1:123456789012:room/*"
}
```

---

## 6. Câu Hỏi Phỏng Vấn

**Q: IVS Chat có đi chung gói với IVS video không?**

> **Billing tách** — video (input/output) và chat (connections, messages). Triển khai có thể chỉ video không chat; chat cần tạo **Room** riêng trong IVS Chat service.

**Q: Làm sao chống bot spam chat?**

> Rate limit room + CAPTCHA trước khi cấp token + **CreateChatToken** chỉ sau auth + Lambda filter từ khóa + disconnect/kick. Không có silver bullet — layered defense.

**Q: Chat có sync với Timed Metadata không?**

> Không tự động. App có thể: host action → `PutMetadata` (UI chung) + optional system chat message qua backend. Tránh double-trigger UX.

---

**Cập Nhật Lần Cuối:** 2026-06-04
**Xem thêm:** [5-recording-playback.md](./5-recording-playback.md) — replay video + chat log
