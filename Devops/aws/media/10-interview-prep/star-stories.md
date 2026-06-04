# 🌟 STAR Stories — Câu Chuyện Media Pipeline Theo Phương Pháp STAR

> STAR là phương pháp kể chuyện kinh nghiệm hiệu quả trong phỏng vấn kỹ thuật. Phần này cung cấp **template** và **ví dụ mẫu** về sự cố / dự án media trên AWS — bạn nên **cá nhân hoá** bằng số liệu và chi tiết thật từ công việc của mình.

---

## 📖 Phương Pháp STAR

```
S — Situation (Tình Huống): Bối cảnh hệ thống media
T — Task (Nhiệm Vụ): Trách nhiệm cụ thể của bạn
A — Action (Hành Động): Bước kỹ thuật đã làm (chi tiết, có công cụ AWS)
R — Result (Kết Quả): Số liệu đo lường được
```

**Nguyên tắc:**
- **Action** chiếm ~60% thời gian nói
- **Result** phải có số: downtime, % cost giảm, latency, cache hit
- Tránh "we/team" mơ hồ — nêu rõ **bạn** làm gì

---

## 📋 Câu Hỏi Hành Vi Thường Gặp (Media)

1. "Kể về lần bạn xử lý sự cố live stream bị gián đoạn"
2. "Kể về lần bạn tối ưu chi phí transcoding hoặc CDN"
3. "Kể về lần bạn triển khai DRM hoặc bảo vệ nội dung"
4. "Kể về kiến trúc OTT / video pipeline bạn thiết kế hoặc migrate"
5. "Kể về quyết định kỹ thuật khó (trade-off giữa IVS và MediaLive chẳng hạn)"

---

## 🔴 STAR Story 1: Live Stream Outage — Mất Ingest Trong Sự Kiện

### Kịch bản mẫu: Primary RTMP fail giữa live concert

---

**S — Situation:**

> "Công ty tổ chức live concert OTT qua MediaLive → MediaPackage → CloudFront, peak 80K concurrent. Giữa buổi diễn, viewer báo màn hình đen và player buffering vô hạn. Đây là sự kiện trả phí — mỗi 5 phút gián đoạn ước tính churn ~3% viewer và ticket refund risk."

**T — Task:**

> "Tôi là on-call platform engineer, chịu trách nhiệm khôi phục luồng trong SLA nội bộ 10 phút và điều phối với team production OB (Outside Broadcast — Phát Sóng Ngoài Hiện Trường)."

**A — Action:**

> **Phút 0–3 — Xác định layer lỗi:**
> - CloudFront real-time logs: 200 cho manifest nhưng segment 404 tăng đột biến → không phải CDN block toàn phần
> - MediaPackage CloudWatch: `IngressBytes` về 0 → upstream không còn data
> - MediaLive console: Channel `RUNNING` nhưng **Active Input** = none, alert `RTMP Has No Audio/Video`
>
> **Phút 3–7 — Nguyên nhân:**
> - OB van mất uplink — primary RTMP URL ngừng
> - Automatic Input Failover **chưa bật** (chỉ có một input attachment trong channel cấu hình gấp tuần trước)
>
> **Phút 7–12 — Khắc phục:**
> - Production chuyển sang backup encoder laptop push RTMP backup URL
> - Tôi dùng Schedule Action **Input Switch** sang input attachment thứ hai (đã tạo sẵn nhưng chưa auto-failover)
> - Xác nhận `IngressBytes` phục hồi; sample playback IVS test player + web prod
>
> **Sau sự cố (tuần sau):**
> - Bật **Automatic Input Failover** primary/backup
> - Nâng channel lên **Standard** (2 pipelines) cho sự kiện lớn tiếp theo
> - Runbook + drill 15 phút trước mỗi event

**R — Result:**

> "Tổng downtime viewer-facing: **~9 phút**. Khôi phục ingest, không cần recreate channel. Sau tháng đó:
> - **0** sự cố input-loss không tự heal ở 4 event tiếp theo
> - MTTR — Mean Time To Recovery — Thời Gian Phục Hồi Trung Bình cho ingest giảm từ ~25 phút (manual) xuống **< 30 giây** (auto failover)
> - Post-mortem được dùng trong phỏng vấn nội bộ training."

**Follow-up trả lời:**

- *"Tại sao không dùng IVS?"* → Event cần MediaPackage + multi-bitrate broadcast và contract với CDN sponsor; IVS không fit lúc đó.
- *"CloudFront có cần invalidate không?"* → Không — live segment mới tiếp tục; manifest tự refresh.

---

## 🟡 STAR Story 2: Cost Optimization — Giảm Chi Phí MediaConvert + CDN

### Kịch bản mẫu: Thư viện VOD 50K giờ, bill transcoding + egress tăng gấp đôi

---

**S — Situation:**

> "Nền tảng edtech có catalog 50K giờ VOD, mỗi upload tạo 6 renditions H.264 + 1 H.265 UHD 'cho tương lai'. Bill MediaConvert và CloudFront egress tăng 110% sau 6 tháng mà MAU chỉ tăng 30%."

**T — Task:**

> "Tôi được giao phân tích và đề xuất giảm **≥ 35%** media infra cost trong 1 sprint mà không làm giảm measurable QoE."

**A — Action:**

> **Phân tích:**
> - Cost Explorer + job metadata: 40% phút xử lý là **UHD H.265** từ nguồn chỉ 1080p
> - CloudFront: cache hit ratio manifest path thấp (TTL 0) nhưng segment chỉ **72%** — query string policy sai
>
> **Thay đổi:**
> 1. Job template mới: cap max **1080p**, H.265 chỉ cho tier premium (flag DB)
> 2. Chuyển batch re-transcode backlog sang **Spot Queue** — chấp nhận delay đêm
> 3. **CMAF** thay dual job HLS+DASH riêng → giảm ~45% processing cho catalog migrate
> 4. CloudFront: tách behavior, segment `Cache-Control: max-age=86400`, bỏ query string khỏi cache key
> 5. Reserved capacity negotiation với AWS account team cho baseline 5000 phút/tháng
>
> **Đo lường:** So sánh startup time và rebuffer rate 2 tuần A/B — không đổi có ý nghĩa thống kê.

**R — Result:**

> "Tháng sau triển khai:
> - MediaConvert cost **-42%**
> - CloudFront egress **-28%**
> - Tổng media line item **-35%** (đạt mục tiêu)
> - p95 startup time giữ nguyên ±50ms"

---

## 🟢 STAR Story 3: DRM Rollout — Multi-Platform Launch

### Kịch bản mẫu: Launch app mobile premium với FairPlay + Widevine

---

**S — Situation:**

> "Dịch vụ streaming premium chuẩn bị launch iOS/Android với yêu cầu studio: **Widevine L1** + **FairPlay**, không cho screen recording. Trước đó chỉ web HLS clear (signed URL)."

**T — Task:**

> "Tôi thiết kế và triển khai luồng DRM end-to-end với MediaPackage + Speke + nhà cung cấp license bên thứ ba, phối hợp mobile team tích hợp player."

**A — Action:**

> 1. POC Speke: API Gateway + Lambda template AWS → EZDRM sandbox
> 2. MediaPackage: 2 endpoints (HLS/FairPlay, DASH/Widevine) cùng channel VOD
> 3. MediaConvert output encryption enabled trong job settings
> 4. CloudFront **Signed Cookies** cho session (tránh ký từng segment URL)
> 5. Test matrix: Safari iOS, Chrome Android, Samsung TV (PlayReady)
> 6. Runbook key rotation và incident "license denied 403"

**R — Result:**

> "Launch đúng deadline, **< 0.5%** playback errors DRM-related tuần đầu (chủ yếu certificate staging mismatch, đã fix).
> Support ticket DRM giảm 90% sau tuần 2 nhờ FAQ player integration."

---

## 🔵 STAR Story 4: Architecture Decision — IVS vs MediaLive

### Kịch bản mẫu: Product muốn live commerce, engineering tranh luận stack

---

**S — Situation:**

> "Product yêu cầu live bán hàng trong app trong 6 tuần, 15K peak viewer, chat, flash sale overlay đồng bộ video. Hai engineer tranh luận MediaLive (đã có kinh nghiệm broadcast) vs IVS (mới)."

**T — Task:**

> "Tôi chịu trách nhiệm **ADR** — Architecture Decision Record — và prototype trong 1 tuần."

**A — Action:**

> - POC IVS: OBS → channel, timed metadata → web demo overlay latency **~3s**
> - POC MediaLive→MediaPackage: latency **~18s** với standard HLS — không đạt UX flash sale
> - So sánh ops: IVS **0** server manage; MediaLive cần channel 24/7 hoặc start/stop playbook
> - Document trade-off: IVS thiếu SSAI broadcast — chấp nhận phase 1 client-side promo banner
> - ADR approved: IVS + IVS Chat; roadmap phase 2 FAST channel riêng bằng MediaLive

**R — Result:**

> "Ship đúng **tuần 6**, 15K peak không degradation. Conversion lift +12% so với pre-recorded demo (product metric).
> Team tiết kiệm ~3 engineer-month so với ước tính MediaLive stack."

---

## 🟣 STAR Story 5: CDN Cache Misconfiguration — Manifest Storm

### Kịch bản mẫu: Origin MediaPackage sập do cache miss hàng loạt

---

**S — Situation:**

> "Sau deploy CloudFront behavior mới, live match có 200K viewer, MediaPackage origin CPU 100%, manifest latency > 5s, player rebuffer hàng loạt."

**T — Task:**

> "Tôi debug và khôi phục trong 20 phút, tránh failover sang slate trên toàn mạng."

**A — Action:**

> - Real-time logs: 80% manifest requests **Miss** cùng một path pattern
> - Phát hiện: behavior mới set **TTL=0** cho `*.m3u8` *và* forward tất cả query strings → cache key unique mỗi user (auth token)
> - Rollback behavior; áp dụng **Signed Cookies** ở edge, cache key chỉ theo path
> - Bật **Origin Shield**
> - Giảm MediaPackage endpoint count tạm thời (gộp test endpoint prod nhầm)

**R — Result:**

> "Recovery **14 phút**. Cache hit manifest từ ~20% lên **88%** trong 5 phút sau fix.
> Thêm CI test cho CloudFront Terraform — assert TTL rules cho media paths."

---

## 📝 Template Điền Câu Chuyện Của Bạn

Sao chép và điền:

```markdown
## Story: [Tiêu đề ngắn]

**S:** [Hệ thống, scale, business impact]

**T:** [Vai trò bạn, deadline/SLA]

**A:**
- Bước 1: [Monitoring / triage tool]
- Bước 2: [Root cause]
- Bước 3: [Fix + AWS services cụ thể]
- Bước 4: [Prevention]

**R:** [Số liệu: thời gian, %, $, MTTR, QoE metric]

**Follow-up sẵn sàng:** [1-2 câu hỏi interviewer có thể hỏi]
```

---

## 🎤 Luyện Tập STAR (30 Phút)

1. Chọn 3 stories phù hợp role (DevOps → 1+5; SA → 4+3; Backend → 2+3)
2. Nói to, bấm giờ **3 phút/story** (không quá 5)
3. Ghi âm — kiểm tra có quá nhiều jargon không giải thích
4. Peer mock: interviewer hỏi follow-up 2 vòng

---

**Cập Nhật Lần Cuối:** 2026-06-04  
**Liên quan:** [README.md](./README.md) | [INTERVIEW_GUIDE.md](./INTERVIEW_GUIDE.md) | [architecture-scenarios.md](./architecture-scenarios.md)
