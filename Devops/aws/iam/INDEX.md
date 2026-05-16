# AWS Security, Identity & Compliance — Chỉ Mục Đầy Đủ

> Hướng dẫn toàn diện về Bảo Mật, Định Danh và Tuân Thủ trên AWS

## 📁 Cấu Trúc Thư Mục

```
Devops/aws/iam/
├── README.md                                   [BẮT ĐẦU TẠI ĐÂY] Lộ trình & tổng quan
├── INDEX.md                                    Chỉ mục đầy đủ (file này)
│
├── 01-iam-fundamentals/
│   ├── README.md                               Nền tảng IAM, khái niệm cốt lõi
│   ├── users-groups-roles.md                   (Cần tạo) Users, Groups, Roles chi tiết
│   ├── policy-types.md                         (Cần tạo) Identity/Resource/SCPs/Permissions Boundary
│   ├── iam-conditions.md                       (Cần tạo) Condition keys, operators, examples
│   ├── permission-boundaries.md                (Cần tạo) Giới hạn quyền, use cases
│   └── iam-best-practices.md                   (Cần tạo) Checklist bảo mật IAM
│
├── 02-identity-federation/
│   ├── README.md                               (Cần tạo) Federation, SSO, Cognito overview
│   ├── iam-identity-center.md                  (Cần tạo) AWS SSO đa tài khoản
│   ├── saml-federation.md                      (Cần tạo) SAML 2.0 với AD/Okta/Azure AD
│   ├── oidc-federation.md                      (Cần tạo) OIDC với GitHub Actions, Google
│   ├── cognito-user-pools.md                   (Cần tạo) Auth cho ứng dụng web/mobile
│   └── cross-account-roles.md                  (Cần tạo) Mô hình hub-and-spoke
│
├── 03-organizations/
│   ├── README.md                               (Cần tạo) Multi-account strategy
│   ├── organizations-setup.md                  (Cần tạo) Tạo OU hierarchy
│   ├── service-control-policies.md             (Cần tạo) SCPs — cú pháp, ví dụ, giới hạn
│   ├── control-tower.md                        (Cần tạo) Landing zone tự động
│   └── account-vending.md                      (Cần tạo) Tự động tạo tài khoản mới
│
├── 04-encryption-kms/
│   ├── README.md                               (Cần tạo) Encryption strategy trên AWS
│   ├── kms-key-types.md                        (Cần tạo) CMK vs AWS-managed vs data keys
│   ├── envelope-encryption.md                  (Cần tạo) Mã hóa phong bì, DEK lifecycle
│   ├── key-policies.md                         (Cần tạo) Key policy vs IAM policy
│   ├── kms-grants.md                           (Cần tạo) Grants — ủy quyền dùng key tạm thời
│   ├── cloudhsm.md                             (Cần tạo) CloudHSM — FIPS 140-2 Level 3
│   └── s3-encryption-options.md                (Cần tạo) SSE-S3 vs SSE-KMS vs SSE-C vs CSE
│
├── 05-secrets-certificates/
│   ├── README.md                               (Cần tạo) Secrets lifecycle management
│   ├── secrets-manager.md                      (Cần tạo) Lưu trữ, xoay vòng, truy xuất bí mật
│   ├── parameter-store.md                      (Cần tạo) Hierarchy, SecureString, versions
│   ├── secrets-vs-parameter.md                 (Cần tạo) So sánh và hướng dẫn chọn lựa
│   ├── acm-certificates.md                     (Cần tạo) TLS cert lifecycle, auto-renewal
│   └── acm-private-ca.md                       (Cần tạo) Internal PKI trên AWS
│
├── 06-network-security/
│   ├── README.md                               (Cần tạo) Defense-in-depth network design
│   ├── security-groups.md                      (Cần tạo) Stateful firewall, best practices
│   ├── nacls.md                                (Cần tạo) Stateless rules, subnet protection
│   ├── vpc-endpoints.md                        (Cần tạo) Gateway vs Interface endpoints
│   ├── privatelink.md                          (Cần tạo) Private service connectivity
│   ├── waf-setup.md                            (Cần tạo) WAF rules, managed rule groups
│   ├── shield-ddos.md                          (Cần tạo) DDoS protection tiers
│   └── network-firewall.md                     (Cần tạo) Stateful firewall có thể lập trình
│
├── 07-monitoring-auditing/
│   ├── README.md                               (Cần tạo) Security observability strategy
│   ├── cloudtrail-setup.md                     (Cần tạo) Trail config, S3 integrity, Lake
│   ├── cloudtrail-analysis.md                  (Cần tạo) Phân tích log, Athena queries
│   ├── aws-config-rules.md                     (Cần tạo) Managed rules, custom rules, remediation
│   ├── access-analyzer.md                      (Cần tạo) Phát hiện external access
│   └── security-dashboard.md                   (Cần tạo) CloudWatch dashboards cho security
│
├── 08-threat-detection/
│   ├── README.md                               (Cần tạo) Threat detection ecosystem
│   ├── guardduty.md                            (Cần tạo) Findings types, severity, suppression
│   ├── security-hub.md                         (Cần tạo) Aggregation, standards, custom actions
│   ├── inspector.md                            (Cần tạo) EC2/ECR/Lambda vulnerability scanning
│   ├── macie.md                                (Cần tạo) S3 data classification, PII detection
│   ├── detective.md                            (Cần tạo) Graph-based investigation
│   └── incident-response.md                    (Cần tạo) Runbook xử lý sự cố bảo mật AWS
│
├── 09-compliance-governance/
│   ├── README.md                               (Cần tạo) Compliance strategy on AWS
│   ├── audit-manager.md                        (Cần tạo) Tự động thu thập bằng chứng
│   ├── conformance-packs.md                    (Cần tạo) CIS, PCI-DSS, HIPAA conformance packs
│   ├── firewall-manager.md                     (Cần tạo) Centralized WAF/SG/Shield management
│   ├── pci-dss-aws.md                          (Cần tạo) Payment Card Industry compliance
│   └── hipaa-aws.md                            (Cần tạo) Healthcare data compliance
│
├── 10-advanced/
│   ├── README.md                               (Cần tạo) Advanced security patterns
│   ├── abac.md                                 (Cần tạo) Attribute-Based Access Control
│   ├── zero-trust.md                           (Cần tạo) Zero Trust Architecture trên AWS
│   ├── data-perimeter.md                       (Cần tạo) Vành đai dữ liệu, resource policies
│   ├── security-automation.md                  (Cần tạo) Auto-remediation với EventBridge + Lambda
│   └── supply-chain-security.md                (Cần tạo) CodeArtifact, Sigstore, SBOM
│
├── 11-interview-prep/
│   ├── README.md                               (Cần tạo) Tổng quan chuẩn bị phỏng vấn
│   ├── top-questions.md                        (Cần tạo) Top 25 câu hỏi AWS Security
│   ├── iam-troubleshooting.md                  (Cần tạo) Debug Access Denied, policy conflicts
│   ├── system-design-security.md               (Cần tạo) Bài toán thiết kế kiến trúc bảo mật
│   ├── star-stories.md                         (Cần tạo) Mẫu câu chuyện sự cố STAR
│   └── 90-day-study-plan.md                    (Cần tạo) Kế hoạch học tập 90 ngày
│
├── GLOSSARY.md                                 (Cần tạo) Thuật ngữ AWS Security
├── RESOURCES.md                                (Cần tạo) Sách, blog, công cụ, chứng chỉ
└── CHECKLIST.md                                (Cần tạo) Checklist pre-deployment & pre-interview
```

---

## ✅ Trạng Thái Nội Dung

| Chủ Đề | File | Trạng Thái | Mức Độ |
|---|---|---|---|
| **Tổng Quan & Lộ Trình** | README.md | ✅ Hoàn Thành | Toàn Diện |
| **Chỉ Mục Đầy Đủ** | INDEX.md | ✅ Hoàn Thành | Đầy Đủ |
| **IAM Fundamentals** | 01-iam-fundamentals/ | 🚧 Đang Tạo | - |
| **Identity Federation** | 02-identity-federation/ | 🚧 Đang Tạo | - |
| **Organizations & Multi-Account** | 03-organizations/ | 🚧 Đang Tạo | - |
| **Encryption & KMS** | 04-encryption-kms/ | 🚧 Đang Tạo | - |
| **Secrets & Certificates** | 05-secrets-certificates/ | 🚧 Đang Tạo | - |
| **Network Security** | 06-network-security/ | 🚧 Đang Tạo | - |
| **Monitoring & Auditing** | 07-monitoring-auditing/ | 🚧 Đang Tạo | - |
| **Threat Detection** | 08-threat-detection/ | 🚧 Đang Tạo | - |
| **Compliance & Governance** | 09-compliance-governance/ | 🚧 Đang Tạo | - |
| **Advanced Topics** | 10-advanced/ | 🚧 Đang Tạo | - |
| **Interview Prep** | 11-interview-prep/ | 🚧 Đang Tạo | - |

---

## 🎯 Thứ Tự Ưu Tiên Tạo Nội Dung

### Ưu Tiên Cao (Kỹ Năng Cốt Lõi)

- [ ] `01-iam-fundamentals/README.md` — IAM users, roles, policies cơ bản
- [ ] `04-encryption-kms/README.md` — KMS, envelope encryption
- [ ] `07-monitoring-auditing/cloudtrail-setup.md` — audit logging thiết yếu
- [ ] `08-threat-detection/guardduty.md` — phát hiện mối đe dọa tự động
- [ ] `11-interview-prep/top-questions.md` — Top 25 câu hỏi phỏng vấn

### Ưu Tiên Trung Bình (Kỹ Năng Nâng Cao)

- [ ] `02-identity-federation/iam-identity-center.md` — SSO đa tài khoản
- [ ] `03-organizations/service-control-policies.md` — SCPs chi tiết
- [ ] `05-secrets-certificates/secrets-manager.md` — auto-rotation
- [ ] `08-threat-detection/security-hub.md` — tổng hợp findings
- [ ] `09-compliance-governance/audit-manager.md` — evidence collection

### Ưu Tiên Thấp Hơn (Tài Liệu Tham Khảo)

- [ ] `10-advanced/zero-trust.md` — kiến trúc zero trust
- [ ] `10-advanced/data-perimeter.md` — vành đai dữ liệu
- [ ] `06-network-security/waf-setup.md` — WAF chi tiết
- [ ] `GLOSSARY.md` — thuật ngữ
- [ ] `RESOURCES.md` — tài liệu học tập

---

## 🚀 Cách Sử Dụng Knowledge Base Này

### Tự Học

```
1. Đọc README.md — hiểu toàn cảnh
2. Chọn lộ trình phù hợp với mục tiêu nghề nghiệp
3. Học từng module theo thứ tự giai đoạn
4. Thực hành trên AWS (dùng account lab riêng)
5. Xây dựng dự án portfolio minh chứng kỹ năng
```

### Chuẩn Bị Phỏng Vấn

```
1. Đọc 11-interview-prep/top-questions.md
2. Ôn kỹ 01-iam-fundamentals/ (luôn được hỏi)
3. Ôn 04-encryption-kms/ (câu hỏi kiến trúc phổ biến)
4. Chuẩn bị 2-3 câu chuyện sự cố theo STAR
5. Luyện debug IAM với iam-troubleshooting.md
```

### Kỹ Sư Vận Hành (Operations)

```
Dùng như tài liệu tham khảo:
- Trước deployment: Xem CHECKLIST.md
- Xử lý sự cố: Đến 08-threat-detection/incident-response.md
- Audit: Dùng 07-monitoring-auditing/ và 09-compliance-governance/
- Thiết kế hệ thống: Tham khảo 03-organizations/ cho multi-account
- Rotation: Theo hướng dẫn 05-secrets-certificates/
```

### Thiết Kế Kiến Trúc (Architecture)

```
1. Đọc 03-organizations/ cho multi-account design
2. Dùng 01-iam-fundamentals/ để thiết kế least privilege
3. Theo 06-network-security/ cho defense-in-depth
4. Tham chiếu 04-encryption-kms/ cho data protection
5. Kiểm tra 09-compliance-governance/ cho regulatory requirements
```

---

## 📊 Ước Tính Thời Gian Học

| Module | Thời Gian | Độ Khó | Mức Ưu Tiên |
|---|---|---|---|
| IAM Fundamentals | 6–8 giờ | ⭐⭐ | Bắt Buộc |
| Identity Federation & SSO | 4–6 giờ | ⭐⭐ | Bắt Buộc |
| Organizations & Multi-Account | 4–6 giờ | ⭐⭐⭐ | Bắt Buộc |
| Encryption & KMS | 6–8 giờ | ⭐⭐⭐ | Bắt Buộc |
| Secrets & Certificates | 3–4 giờ | ⭐⭐ | Bắt Buộc |
| Network Security | 5–7 giờ | ⭐⭐ | Bắt Buộc |
| Monitoring & Auditing | 4–6 giờ | ⭐⭐ | Bắt Buộc |
| Threat Detection | 5–7 giờ | ⭐⭐ | Bắt Buộc |
| Compliance & Governance | 6–8 giờ | ⭐⭐⭐ | Nên Học |
| Advanced Topics | 10–15 giờ | ⭐⭐⭐ | Nice-to-Have |

**Tổng cộng: 53–75 giờ cho kiến thức AWS Security toàn diện**

---

## 🎓 Cấp Độ Kỹ Năng Được Hỗ Trợ

### Người Mới Bắt Đầu (0–1 năm kinh nghiệm)

- [ ] Users, Groups, Roles, Policies cơ bản
- [ ] MFA và root account security
- [ ] S3 bucket policies và ACLs
- [ ] Security Groups cơ bản
- [ ] Shared Responsibility Model

**Thời gian đạt được:** 1–2 tháng

### Trung Cấp (1–3 năm kinh nghiệm)

- [ ] IAM role design cho microservices
- [ ] KMS encryption end-to-end
- [ ] Multi-account với Organizations
- [ ] GuardDuty + Security Hub integration
- [ ] Secrets Manager với auto-rotation

**Thời gian nâng cao:** 2–3 tháng thực hành chuyên sâu

### Nâng Cao (3–5+ năm kinh nghiệm)

- [ ] Zero Trust architecture design
- [ ] Data perimeter enforcement
- [ ] Security automation pipelines
- [ ] Compliance program management
- [ ] Advanced threat hunting

**Thời gian:** Học liên tục — lĩnh vực luôn thay đổi

---

## 🔗 Điều Hướng Nhanh

| Nhu Cầu | Vị Trí |
|---|---|
| Tổng quan toàn bộ | [README.md](README.md) |
| IAM cơ bản | [01-iam-fundamentals/README.md](01-iam-fundamentals/README.md) |
| SSO đa tài khoản | [02-identity-federation/iam-identity-center.md](02-identity-federation/iam-identity-center.md) |
| Multi-account design | [03-organizations/README.md](03-organizations/README.md) |
| Mã hóa & KMS | [04-encryption-kms/README.md](04-encryption-kms/README.md) |
| Quản lý bí mật | [05-secrets-certificates/secrets-manager.md](05-secrets-certificates/secrets-manager.md) |
| Bảo mật mạng | [06-network-security/README.md](06-network-security/README.md) |
| CloudTrail & Config | [07-monitoring-auditing/README.md](07-monitoring-auditing/README.md) |
| GuardDuty & incidents | [08-threat-detection/incident-response.md](08-threat-detection/incident-response.md) |
| Compliance | [09-compliance-governance/README.md](09-compliance-governance/README.md) |
| Câu hỏi phỏng vấn | [11-interview-prep/top-questions.md](11-interview-prep/top-questions.md) |

---

## 📈 Theo Dõi Tiến Độ Học Tập

Sao chép và theo dõi tiến độ của bạn:

```markdown
## Tiến Độ AWS Security

### Giai Đoạn 1: Nền Tảng (Tuần 1–2)
- [ ] IAM Users, Groups, Roles
- [ ] Policy types & JSON syntax
- [ ] Permission Boundaries
- [ ] MFA & account security
- [ ] Shared Responsibility Model

### Giai Đoạn 2: Dịch Vụ Định Danh (Tuần 3–5)
- [ ] IAM Identity Center (SSO)
- [ ] SAML federation với IdP
- [ ] Cross-account role setup
- [ ] Cognito User Pools
- [ ] AWS Organizations + SCPs

### Giai Đoạn 3: Hạ Tầng Bảo Mật (Tuần 6–8)
- [ ] KMS key types & policies
- [ ] Envelope encryption
- [ ] Secrets Manager rotation
- [ ] WAF rule groups
- [ ] VPC security architecture

### Giai Đoạn 4: Giám Sát & Phát Hiện (Tuần 9–11)
- [ ] CloudTrail + Lake setup
- [ ] GuardDuty finding types
- [ ] Security Hub standards
- [ ] AWS Config rules
- [ ] Inspector scanning

### Giai Đoạn 5: Tuân Thủ & Nâng Cao (Tuần 12+)
- [ ] Audit Manager setup
- [ ] Conformance packs
- [ ] Zero Trust design
- [ ] Data perimeter
- [ ] Security automation
```

---

## 🎯 Tiêu Chí Thành Công

Sau khi hoàn thành knowledge base này, bạn có thể:

### ✅ Năng Lực Cơ Bản

- [ ] Thiết kế IAM policy phức tạp không cần tài liệu
- [ ] Giải thích Shared Responsibility Model chi tiết
- [ ] Debug Access Denied errors một cách có hệ thống
- [ ] Thiết kế kiến trúc multi-account an toàn
- [ ] Cấu hình encryption end-to-end cho dữ liệu nhạy cảm

### ✅ Năng Lực Vận Hành

- [ ] Phân tích CloudTrail log để tìm hoạt động đáng ngờ
- [ ] Xử lý GuardDuty findings theo runbook chuẩn
- [ ] Triển khai Secrets Manager rotation không gián đoạn
- [ ] Thiết lập Security Hub với standards tuân thủ
- [ ] Thực hiện security audit với AWS Config

### ✅ Sẵn Sàng Phỏng Vấn

- [ ] Trả lời tự tin 25 câu hỏi AWS Security phổ biến
- [ ] Kể được 2–3 câu chuyện sự cố bảo mật (STAR)
- [ ] Thiết kế hệ thống an toàn trong bài toán system design
- [ ] Thảo luận được trade-offs giữa các giải pháp bảo mật
- [ ] Nắm vững ít nhất một compliance framework

---

## 🚀 Bước Tiếp Theo

### Ngay Lập Tức (Tuần Này)

1. Đọc toàn bộ README.md
2. Chọn lộ trình phù hợp mục tiêu
3. Bắt đầu `01-iam-fundamentals/` — luôn được hỏi trong phỏng vấn
4. Tạo AWS lab account, bật CloudTrail và GuardDuty

### Ngắn Hạn (2 Tuần Tới)

1. Hoàn thành 01-iam-fundamentals/
2. Thực hành viết IAM policies từ đầu
3. Bắt đầu 04-encryption-kms/
4. Lab: mã hóa S3 bucket với CMK

### Trung Hạn (4 Tuần Tới)

1. Hoàn thành các chủ đề cốt lõi (01–08)
2. Thiết lập GuardDuty + Security Hub trong lab
3. Chuẩn bị 2–3 câu chuyện STAR từ kinh nghiệm thực tế
4. Mock interview với đồng nghiệp

### Dài Hạn (3 Tháng Tới)

1. Hoàn thành toàn bộ knowledge base
2. Xây dựng portfolio project bảo mật thực tế
3. Đăng ký thi AWS Certified Security — Specialty
4. Đóng góp vào open-source security tools (Prowler, ScoutSuite)

---

## 💡 Mẹo Học Tập Hiệu Quả

1. **Luôn dùng lab account riêng:** Không thực hành trên production — tạo AWS account riêng
2. **Bật CloudTrail ngay từ đầu:** Quan sát mọi hành động của bạn để học từ API calls
3. **Đọc IAM Policy Simulator:** Kiểm thử policy trước khi áp dụng thực tế
4. **Nghiên cứu các breach thực tế:** Capital One breach, Tesla crypto-mining — học từ sự cố thật
5. **Biết tại sao, không chỉ cái gì:** Hiểu nguyên lý, không chỉ học thuộc lòng dịch vụ
6. **Follow AWS Security Blog:** Cập nhật tính năng mới hàng tuần
7. **Luyện tập viết policy JSON:** Đây là kỹ năng phân biệt senior với junior
8. **Document sự cố lab của bạn:** Post-mortem ngay cả trong lab là thói quen tốt

---

## 📞 Đóng Góp

Phát hiện lỗi? Muốn thêm nội dung?

Đây là tài liệu sống — đóng góp luôn được chào đón:

- [ ] Sửa lỗi trong nội dung hiện có
- [ ] Thêm ví dụ thực tế từ kinh nghiệm của bạn
- [ ] Cập nhật khi AWS ra tính năng mới
- [ ] Giải thích rõ hơn các khái niệm phức tạp
- [ ] Thêm bài lab thực hành chi tiết

---

## 📄 Giấy Phép

Knowledge base này mở để học tập và sử dụng chuyên nghiệp.

---

**Cập Nhật Lần Cuối:** 2026-05-16
**Phiên Bản:** 1.0 (README + INDEX hoàn thành)
**Trạng Thái:** ✅ README Hoàn Thành | ✅ INDEX Hoàn Thành | 🚧 Các Module Đang Tạo
