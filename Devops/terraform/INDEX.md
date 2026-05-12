# Terraform Knowledge Base — Chỉ Mục Đầy Đủ

> Hướng dẫn toàn diện về Terraform — Infrastructure as Code — Hạ Tầng Dưới Dạng Mã

## 📁 Cấu Trúc Thư Mục

```
Devops/terraform/
├── README.md                               [BẮT ĐẦU TẠI ĐÂY] Lộ trình & tổng quan
│
├── 01-fundamentals/
│   ├── README.md                           Nền tảng Terraform, HCL, vòng đời tài nguyên
│   ├── what-is-iac.md                      IaC là gì, lợi ích, so sánh tools
│   ├── hcl-syntax.md                       HCL — HashiCorp Configuration Language — cú pháp
│   ├── providers-resources.md              Provider & Resource cơ bản
│   ├── variables-outputs.md                Variables, Locals, Outputs
│   └── lifecycle.md                        init, plan, apply, destroy — vòng đời
│
├── 02-state-management/
│   ├── README.md                           Quản lý state, backend, locking
│   ├── state-explained.md                  State file là gì, chứa gì, tại sao quan trọng
│   ├── remote-backend.md                   S3+DynamoDB, GCS, Azure Blob, Terraform Cloud
│   ├── state-locking.md                    State Locking — Khoá trạng thái — và deadlock
│   ├── state-commands.md                   terraform state list/show/mv/rm/pull/push
│   └── state-recovery.md                   Phục hồi khi state bị corrupt — hỏng
│
├── 03-modules/
│   ├── README.md                           Thiết kế module, best practices
│   ├── module-structure.md                 Cấu trúc module chuẩn
│   ├── input-output.md                     Variables, Outputs, Type Constraints
│   ├── module-registry.md                  Terraform Registry — public & private
│   ├── module-versioning.md                Semantic versioning — Đánh số phiên bản
│   └── composition-patterns.md             Flat, Nested, Wrapper module patterns
│
├── 04-workspaces-environments/
│   ├── README.md                           Quản lý đa môi trường
│   ├── workspaces.md                       Terraform Workspaces — giới hạn và dùng khi nào
│   ├── environment-separation.md           dev/staging/prod strategy — chiến lược phân tách
│   ├── tfvars-management.md                .tfvars file per environment
│   ├── terragrunt-intro.md                 Terragrunt — DRY multi-env management
│   └── backend-per-env.md                  Backend riêng cho từng môi trường
│
├── 05-security/
│   ├── README.md                           Bảo mật Terraform toàn diện
│   ├── secrets-management.md               Vault, AWS Secrets Manager, SOPS
│   ├── iam-roles.md                        IAM — Identity Access Management — role tối thiểu
│   ├── sensitive-variables.md              Sensitive vars, output masking
│   ├── static-analysis.md                  tfsec, Checkov, Terrascan
│   └── audit-logging.md                    Nhật ký kiểm tra — CloudTrail, audit logs
│
├── 06-cicd/
│   ├── README.md                           Tích hợp CI/CD với Terraform
│   ├── github-actions.md                   GitHub Actions workflow cho Terraform
│   ├── gitlab-ci.md                        GitLab CI/CD pipeline
│   ├── atlantis.md                         Atlantis — PR-based automation — tự động qua PR
│   ├── terraform-cloud.md                  Terraform Cloud / HCP Terraform
│   └── rollback-strategy.md                Chiến lược rollback khi có sự cố
│
├── 07-testing/
│   ├── README.md                           Kiểm thử hạ tầng Terraform
│   ├── validate-fmt.md                     terraform validate, fmt, lint
│   ├── tflint.md                           TFLint — Kiểm tra lỗi và best practices
│   ├── terratest.md                        Terratest — Unit & Integration testing
│   ├── checkov-opa.md                      Checkov & OPA — Open Policy Agent — compliance
│   └── test-strategy.md                    Chiến lược kiểm thử toàn diện
│
├── 08-monitoring/
│   ├── README.md                           Giám sát & quan sát hạ tầng Terraform
│   ├── drift-detection.md                  Drift Detection — Phát hiện lệch cấu hình
│   ├── infracost.md                        Infracost — Ước tính chi phí trong CI/CD
│   ├── change-audit.md                     Kiểm tra ai thay đổi gì, khi nào
│   ├── resource-tagging.md                 Tagging strategy — Chiến lược gán nhãn
│   └── alerting.md                         Cảnh báo khi có thay đổi ngoài dự kiến
│
├── 09-troubleshooting/
│   ├── README.md                           Xử lý sự cố & incident response
│   ├── state-corruption.md                 State file hỏng — cách phát hiện và phục hồi
│   ├── dependency-issues.md                Dependency Cycle — Vòng phụ thuộc
│   ├── provider-errors.md                  Lỗi provider — timeout, rate limit, auth
│   ├── import-moved.md                     terraform import & moved block
│   ├── debug-mode.md                       TF_LOG=DEBUG — chế độ gỡ lỗi chi tiết
│   └── production-checklist.md             Checklist trước khi apply vào production
│
├── 10-advanced/
│   ├── README.md                           Chủ đề Terraform nâng cao
│   ├── dynamic-blocks.md                   Dynamic Blocks — Khối động
│   ├── meta-arguments.md                   depends_on, lifecycle, provisioner
│   ├── for-each-count.md                   for_each vs count — so sánh và dùng khi nào
│   ├── custom-providers.md                 Viết Custom Provider — Provider tùy chỉnh
│   ├── terraform-cdk.md                    CDK — Cloud Development Kit — Python/TypeScript
│   ├── opentofu.md                         OpenTofu — Nhánh open-source của Terraform
│   └── multi-region-account.md             Kiến trúc đa vùng & đa tài khoản
│
├── 11-interview-prep/
│   ├── README.md                           Tổng quan chuẩn bị phỏng vấn
│   ├── INTERVIEW_GUIDE.md                  Top 20 câu hỏi phỏng vấn Terraform
│   ├── star-stories.md                     Câu chuyện sự cố theo phương pháp STAR
│   ├── system-design-scenarios.md          Thiết kế hệ thống với IaC
│   ├── technical-questions.md              Q&A kỹ thuật tổng hợp
│   └── 90-day-study-plan.md                Kế hoạch học 90 ngày có cấu trúc
│
├── ROADMAP.md                              Lộ trình học chi tiết theo tuần
├── GLOSSARY.md                             Thuật ngữ Terraform & IaC
├── RESOURCES.md                            Sách, blog, công cụ, khoá học
└── CHECKLIST.md                            Checklist trước phỏng vấn & trước khi deploy
```

---

## ✅ Những Gì Đã Tạo

| Chủ Đề                           | File                        | Trạng Thái | Chất Lượng     |
| -------------------------------- | --------------------------- | ---------- | -------------- |
| **Tổng quan & Lộ trình**         | README.md                   | ✅         | Toàn diện      |
| **Chỉ mục đầy đủ**               | INDEX.md                    | ✅         | Toàn diện      |

---

## 🎯 Cần Tạo Tiếp (Theo Thứ Tự Ưu Tiên)

### Ưu Tiên Cao (Kỹ năng cốt lõi)

- [ ] `01-fundamentals/README.md` — Nền tảng HCL, providers, lifecycle
- [ ] `02-state-management/README.md` — Remote state, locking
- [ ] `02-state-management/remote-backend.md` — S3, GCS, Terraform Cloud
- [ ] `03-modules/README.md` — Module design patterns
- [ ] `11-interview-prep/INTERVIEW_GUIDE.md` — Top 20 câu hỏi phỏng vấn
- [ ] `ROADMAP.md` — Kế hoạch học 90 ngày chi tiết

### Ưu Tiên Trung (Kỹ năng vận hành)

- [ ] `06-cicd/README.md` — CI/CD integration
- [ ] `05-security/README.md` — Bảo mật hạ tầng
- [ ] `09-troubleshooting/README.md` — Xử lý sự cố
- [ ] `04-workspaces-environments/README.md` — Quản lý môi trường
- [ ] `11-interview-prep/star-stories.md` — Câu chuyện STAR

### Ưu Tiên Thấp (Tham khảo)

- [ ] `07-testing/README.md` — Kiểm thử hạ tầng
- [ ] `08-monitoring/README.md` — Giám sát & drift detection
- [ ] `10-advanced/README.md` — Chủ đề nâng cao
- [ ] `GLOSSARY.md` — Thuật ngữ
- [ ] `RESOURCES.md` — Tài liệu tham khảo

---

## 🚀 Cách Dùng Knowledge Base Này

### Cho Tự Học

```
1. Bắt đầu với README.md
2. Chọn Lộ Trình (Mới bắt đầu / Trung cấp / Nâng cao)
3. Học từng phần theo thứ tự
4. Làm bài tập thực hành (thiết lập lab thực tế)
5. Xây dựng dự án portfolio
```

### Cho Chuẩn Bị Phỏng Vấn

```
1. Đọc 11-interview-prep/INTERVIEW_GUIDE.md
2. Tập trung vào State Management (luôn được hỏi)
3. Nắm vững Module Design (luôn được hỏi)
4. Học 06-cicd/ — cách tích hợp Terraform vào team
5. Chuẩn bị câu chuyện sự cố hạ tầng (STAR)
6. Mock interview với đồng nghiệp
```

### Cho Công Việc Thực Tế

```
Dùng làm tài liệu tham khảo:
- Trước deploy: Đọc 09-troubleshooting/production-checklist.md
- Khi có sự cố: Vào 09-troubleshooting/ để chẩn đoán
- Thiết kế module: Theo 03-modules/ best practices
- CI/CD setup: Theo 06-cicd/ hướng dẫn
- Bảo mật: Kiểm tra theo 05-security/ checklist
```

### Cho System Design

```
1. Đọc README.md để hiểu tổng quan
2. Dùng 04-workspaces-environments/ cho multi-env design
3. Dùng 05-security/ cho security architecture
4. Dùng 06-cicd/ cho deployment pipeline design
5. Dùng 10-advanced/ cho multi-region architecture
```

---

## 📊 Ước Tính Thời Gian Học

| Phần                                   | Thời Gian   | Độ Khó | Ưu Tiên |
| -------------------------------------- | ----------- | ------ | ------- |
| Nền tảng (Fundamentals)                | 4-6 giờ     | ⭐     | Bắt buộc |
| State Management — Quản lý trạng thái | 6-8 giờ     | ⭐⭐   | Bắt buộc |
| Modules — Mô-đun                       | 8-10 giờ    | ⭐⭐   | Bắt buộc |
| Workspaces & Environments              | 4-6 giờ     | ⭐⭐   | Bắt buộc |
| Security — Bảo mật                     | 6-8 giờ     | ⭐⭐   | Bắt buộc |
| CI/CD Integration                      | 8-10 giờ    | ⭐⭐⭐ | Nên có   |
| Testing — Kiểm thử                     | 6-8 giờ     | ⭐⭐⭐ | Nên có   |
| Monitoring — Giám sát                  | 4-6 giờ     | ⭐⭐   | Nên có   |
| Advanced Topics — Nâng cao             | 15-20 giờ   | ⭐⭐⭐ | Tốt hơn  |

**Tổng cộng: 60-90 giờ để có kiến thức Terraform toàn diện**

---

## 🎓 Cấp Độ Kỹ Năng Được Hỗ Trợ

### Mới Bắt Đầu (0-1 năm kinh nghiệm)

- [ ] Hiểu IaC là gì và tại sao cần
- [ ] Viết được HCL — HashiCorp Configuration Language — cơ bản
- [ ] Tạo và xóa tài nguyên trên AWS/GCP
- [ ] Hiểu state file và cách không commit lên git
- [ ] Dùng module từ Terraform Registry

**Thời gian để thành thạo:** 2-3 tháng

### Trung Cấp (1-3 năm kinh nghiệm)

- [ ] Thiết kế module tái sử dụng tốt
- [ ] Quản lý remote state với locking
- [ ] Tích hợp Terraform vào CI/CD
- [ ] Quản lý nhiều môi trường hiệu quả
- [ ] Xử lý secrets an toàn

**Thời gian để nâng cấp:** 2-3 tháng để đào sâu

### Nâng Cao (3-5+ năm kinh nghiệm)

- [ ] Kiến trúc multi-account, multi-region
- [ ] Policy as Code — Chính Sách Dưới Dạng Mã
- [ ] Incident command cho hạ tầng
- [ ] Custom provider development
- [ ] Compliance frameworks — Khung tuân thủ

**Thời gian:** Học liên tục

---

## 🔗 Điều Hướng Nhanh

| Nhu Cầu                          | Vị Trí                                                                         |
| -------------------------------- | ------------------------------------------------------------------------------ |
| Tổng quan nhanh                  | [README.md](README.md)                                                         |
| Remote backend setup             | [02-state-management/remote-backend.md](02-state-management/remote-backend.md) |
| Module design guide              | [03-modules/README.md](03-modules/README.md)                                   |
| CI/CD setup                      | [06-cicd/README.md](06-cicd/README.md)                                         |
| Security checklist               | [05-security/README.md](05-security/README.md)                                 |
| Câu hỏi phỏng vấn                | [11-interview-prep/INTERVIEW_GUIDE.md](11-interview-prep/INTERVIEW_GUIDE.md)   |

---

## 📈 Theo Dõi Tiến Độ Học

Sao chép và theo dõi tiến độ của bạn:

```markdown
## Hoàn Thành Terraform Knowledge

### Giai Đoạn 1: Nền Tảng (Tuần 1-2)

- [ ] IaC là gì, tại sao dùng Terraform
- [ ] HCL syntax — cú pháp cơ bản
- [ ] Providers & Resources
- [ ] Variables, Locals, Outputs
- [ ] Vòng đời: init, plan, apply, destroy

### Giai Đoạn 2: Kỹ Năng Cốt Lõi (Tuần 3-6)

- [ ] State file — hiểu và quản lý
- [ ] Remote backend với locking
- [ ] Module design & reuse
- [ ] Workspaces và multi-env strategy
- [ ] Secrets management — quản lý bí mật

### Giai Đoạn 3: Vận Hành (Tuần 7-10)

- [ ] CI/CD pipeline với Terraform
- [ ] Drift detection — phát hiện lệch
- [ ] Testing infrastructure — kiểm thử
- [ ] Security scanning — quét bảo mật
- [ ] Incident response cho hạ tầng

### Giai Đoạn 4: Chuyên Sâu (Tuần 11+)

- [ ] Multi-account architecture
- [ ] Policy as Code với Sentinel/OPA
- [ ] Custom providers
- [ ] Terragrunt mastery
- [ ] Mock interviews
```

---

## 🎯 Tiêu Chí Thành Công

Sau khi hoàn thành knowledge base này, bạn có thể:

### ✅ Năng Lực Nền Tảng

- [ ] Giải thích IaC và lợi ích không cần nhìn notes
- [ ] Thiết kế cấu trúc Terraform cho dự án thực tế
- [ ] Hiểu trade-off giữa các cách quản lý state
- [ ] Đọc và debug HCL code của người khác
- [ ] Thiết kế module tái sử dụng tốt

### ✅ Năng Lực Vận Hành

- [ ] Thiết lập remote backend an toàn cho team
- [ ] Tích hợp Terraform vào CI/CD pipeline
- [ ] Quản lý nhiều môi trường không bị nhầm lẫn
- [ ] Phát hiện và xử lý drift — lệch cấu hình
- [ ] Áp dụng security best practices — thực hành bảo mật tốt nhất

### ✅ Sẵn Sàng Phỏng Vấn

- [ ] Trả lời được top 20 câu hỏi Terraform
- [ ] Kể được 2-3 câu chuyện sự cố hạ tầng (STAR)
- [ ] Thiết kế hệ thống với IaC considerations
- [ ] Thảo luận trade-off và constraints tự tin
- [ ] Nắm vững ít nhất 1 cloud provider sâu

---

## 🚀 Bước Tiếp Theo

### Ngay Bây Giờ (Tuần Này)

1. Đọc kỹ README.md
2. Chọn lộ trình học phù hợp với mình
3. Cài Terraform và thiết lập môi trường lab
4. Tạo tài nguyên đầu tiên trên AWS Free Tier / GCP Free Tier

### Ngắn Hạn (2 Tuần Tới)

1. Hoàn thành `01-fundamentals/`
2. Thiết lập remote backend thực tế
3. Bắt đầu `03-modules/` — viết module đầu tiên
4. Làm lab thực hành cho từng chủ đề

### Trung Hạn (4 Tuần Tới)

1. Hoàn thành tất cả core topics (01-06)
2. Deep dive vào một cloud provider
3. Tích hợp Terraform vào một CI/CD pipeline thực
4. Chuẩn bị 2-3 câu chuyện sự cố

### Dài Hạn (3 Tháng Tới)

1. Thành thạo một cloud provider hoàn toàn
2. Hiểu trade-off giữa các patterns
3. Xây dựng dự án portfolio thực tế
4. Bắt đầu apply vào các vị trí DevOps / Platform Engineer

---

## 💡 Lời Khuyên Từ Thực Tế

1. **Học bằng cách làm:** Đừng chỉ đọc — hãy tạo và destroy tài nguyên thực sự
2. **Commit state vào git là anti-pattern:** Luôn dùng remote backend
3. **`count` vs `for_each`:** Ưu tiên `for_each` để tránh index shifting
4. **Module nhỏ hơn module lớn:** Dễ tái sử dụng và test hơn
5. **Plan trước khi apply:** Luôn review plan output cẩn thận
6. **Đặt tên rõ ràng:** Resource names nên bao gồm environment và purpose
7. **Test state recovery:** Backup state thường xuyên, test restore
8. **Dùng `moved` block:** Khi refactor, không xóa rồi tạo lại tài nguyên

---

## 📞 Đóng Góp

Phát hiện lỗi? Muốn bổ sung nội dung?

Đây là tài liệu sống. Chào mừng đóng góp:

- [ ] Sửa lỗi trong nội dung hiện có
- [ ] Thêm phần cho chủ đề chưa được đề cập
- [ ] Ví dụ thực tế từ kinh nghiệm của bạn
- [ ] Giải thích tốt hơn cho các concepts phức tạp
- [ ] Hướng dẫn cho cloud provider cụ thể

---

## 📄 Giấy Phép

Knowledge base này mở cho việc học tập và sử dụng chuyên nghiệp.

---

**Cập Nhật Lần Cuối:** 2026-05-12
**Phiên Bản:** 1.0
**Trạng Thái:** ✅ README & INDEX hoàn thành | 🚧 Các phần chi tiết đang trong quá trình tạo
