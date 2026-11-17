# Đánh giá thực tế: Hệ thống lớn có triển khai mô hình Template dùng chung và cấu hình theo cơ sở hay không?

**Phiên bản:** 1.0  
**Ngày:** 17/11/2026  
**Bối cảnh áp dụng:** HIS/EMR đa khoa, SaaS Multi-Tenant, .NET 10, React, PostgreSQL, định hướng openEHR

---

## 1. Kết luận điều hành

**Có, nhiều nền tảng EHR/EMR lớn đã triển khai các nguyên lý tương tự**, gồm:

- **Clinical model reuse (Tái sử dụng mô hình lâm sàng)**.
- **Template-based documentation (Ghi chép dựa trên mẫu)**.
- **Metadata-driven forms (Form điều khiển bằng siêu dữ liệu)**.
- **Conditional sections/fields (Phần và trường hiển thị có điều kiện)**.
- **Organization-specific configuration (Cấu hình theo tổ chức/cơ sở)**.
- **Specialty-specific workflow (Quy trình theo chuyên khoa)**.

Tuy nhiên, cần sửa một điểm quan trọng trong cách hiểu ban đầu:

> Các hệ thống lớn thường **không tạo một Master Template (Mẫu tổng) khổng lồ chứa toàn bộ trường của mọi chuyên khoa**, rồi cho từng phòng khám tùy ý tắt các node của chính template đó.

Cách làm bền vững hơn là:

1. Xây dựng **Reusable Clinical Models (Mô hình lâm sàng tái sử dụng)**.
2. Tổ chức thành **Section Library (Thư viện phần nội dung)** hoặc **Component Library (Thư viện thành phần)**.
3. Tạo nhiều **Purpose-specific Templates (Mẫu theo mục đích sử dụng)** bằng cách lắp ghép và ràng buộc các thành phần.
4. Dùng **Configuration/Profile (Cấu hình/Hồ sơ cấu hình)** để điều khiển tenant, chuyên khoa, vai trò và giao diện.
5. Không để cấu hình giao diện làm thay đổi tùy tiện **Clinical semantics (Ngữ nghĩa lâm sàng)** của dữ liệu đã lưu.

Vì vậy, nhận định chính xác là:

> **Có những hệ thống lớn thực hiện theo kiến trúc tương tự, nhưng mô hình trưởng thành thường là “một kho mô hình lâm sàng dùng chung + nhiều template có phạm vi rõ ràng + cấu hình giao diện/quy trình theo cơ sở”, không phải “một template duy nhất cho mọi khoa”.**

---

## 2. Phân biệt bốn lớp thường bị nhầm lẫn

### 2.1 Archetype (Mô hình dữ liệu lâm sàng)

Định nghĩa ý nghĩa và cấu trúc của một khái niệm lâm sàng có thể tái sử dụng, ví dụ:

- Blood Pressure (Huyết áp).
- Body Temperature (Nhiệt độ cơ thể).
- Problem/Diagnosis (Vấn đề/Chẩn đoán).
- Medication Order (Y lệnh thuốc).

### 2.2 Operational Template - OPT (Mẫu vận hành)

Là cấu trúc có ràng buộc, được tạo cho một use case cụ thể và có thể gửi lên openEHR CDR (Kho dữ liệu lâm sàng openEHR). EHRbase xác nhận template được tải lên máy chủ dưới dạng OPT.

### 2.3 Form definition (Định nghĩa biểu mẫu)

Quy định cách các node trong template được trình bày:

- Trang, tab và section (phần).
- Nhãn hiển thị.
- Điều kiện ẩn/hiện.
- Validation (Kiểm tra hợp lệ).
- Widget (Thành phần giao diện).
- Read-only/required (Chỉ đọc/bắt buộc).

Form không nên được coi là đồng nhất với template lâm sàng.

### 2.4 Tenant/Workflow Configuration (Cấu hình phòng khám/quy trình)

Quy định:

- Cơ sở nào được sử dụng chuyên khoa nào.
- Vai trò nào được xem hoặc sửa section nào.
- Form nào áp dụng cho loại encounter (lần khám), khoa, vai trò và trạng thái quy trình.
- Quy tắc mặc định, thứ tự hiển thị và feature flag (cờ tính năng).

---

## 3. Bằng chứng từ các nền tảng thực tế

## 3.1 Better Studio / Better Platform

Better Studio là ví dụ gần nhất với mô hình đang đề xuất:

- Form Builder (Bộ tạo biểu mẫu) dạng low-code kéo thả.
- Form được bind (liên kết) với openEHR template.
- Hỗ trợ multi-page form (form nhiều trang), conditional logic (logic điều kiện), dynamic behaviour (hành vi động), localization (đa ngôn ngữ).
- Form Renderer (Bộ kết xuất biểu mẫu) có thể được nhúng vào ứng dụng riêng.
- Fields lấy từ template được tự động liên kết với cấu trúc openEHR.

**Đánh giá mức độ tương đồng:** Rất cao ở lớp metadata-driven UI (giao diện điều khiển bằng metadata) và tái sử dụng template, nhưng tài liệu công khai không chứng minh rằng Better khuyến nghị một template khổng lồ cho mọi chuyên khoa.

**Bài học áp dụng:** Tách Clinical Template (Mẫu lâm sàng) khỏi Form Layout (Bố cục biểu mẫu), rồi dùng renderer chung.

Nguồn:

- https://docs.better.care/studio/
- https://docs.better.care/studio/build-in-studio/form-builder-overview/
- https://docs.better.care/studio/start-here/start-building/

## 3.2 EHRbase

EHRbase triển khai openEHR server và hỗ trợ:

- Upload Operational Template - OPT (Tải mẫu vận hành).
- WebTemplate / Simplified Data Template - SDT (Mẫu web / mẫu dữ liệu đơn giản hóa).
- Biểu diễn cây template dưới dạng JSON với cardinality như `min`, `max`, `children` và đường dẫn AQL.
- Flat JSON và Structured JSON để ứng dụng client đọc/ghi composition dễ hơn.

**Đánh giá mức độ tương đồng:** Cao ở tầng template-driven API (API điều khiển bằng template). EHRbase là CDR/backend, không tự động trở thành một trình cấu hình tenant hoặc form builder hoàn chỉnh.

**Bài học áp dụng:** Có thể dùng WebTemplate/SDT làm input cho Dynamic Form Engine (Bộ sinh form động), nhưng cần xây thêm lớp UI schema, tenant configuration và authorization.

Nguồn:

- https://docs.ehrbase.org/docs/EHRbase/Explore/Simplified-data-template/WebTemplate
- https://docs.ehrbase.org/docs/EHRbase/openEHR-Introduction/Template

## 3.3 Oracle Health EHR / Cerner PowerForms

Tài liệu công khai của Oracle Health mô tả:

- PowerForms (Biểu mẫu cấu hình).
- Conditional Control (Điều khiển có điều kiện).
- Conditional Section (Phần có điều kiện).
- Required Field (Trường bắt buộc).
- Các control theo kiểu dữ liệu như numeric, free-text, rich-text, grid.
- Các embedded components (thành phần nhúng) như Social History, Family History, Pregnancy History, Allergies và Problems List.
- Dynamic Documentation Templates (Mẫu tài liệu động) và Shared Sections (Phần dùng chung).

**Đánh giá mức độ tương đồng:** Cao ở lớp cấu hình form, section và tài liệu lâm sàng. Đây là bằng chứng rõ rằng một EHR lớn sử dụng các thành phần dùng chung và điều kiện hiển thị. Tuy nhiên, không nên suy ra rằng toàn bộ Oracle Health dùng chính xác một master template duy nhất hoặc cơ chế openEHR.

**Bài học áp dụng:** Conditional display (Hiển thị có điều kiện) và shared sections (phần dùng chung) là pattern đã được sử dụng trong hệ thống quy mô lớn.

Nguồn:

- https://docs.oracle.com/en/industries/health/oracle-health-ehr/ehrcg/powerforms.html
- https://docs.oracle.com/en/industries/health/oracle-health-ehr/ehrcg/configure-existing-powerforms-settings.html
- https://docs.oracle.com/en/industries/health/oracle-health-ehr/ehrcg/dynamic-documentation-templates.html

## 3.4 Epic

Epic là hệ thống EHR lớn, hỗ trợ tích hợp quy mô rất cao và có mô hình cấu hình theo tổ chức/chuyên khoa. Tài liệu công khai xác nhận hệ sinh thái API và interface lớn, nhưng không công khai đầy đủ mô hình nội bộ của các công cụ cấu hình hồ sơ lâm sàng.

**Đánh giá mức độ tương đồng:** Có thể dùng Epic như ví dụ về enterprise EHR có cấu hình theo tổ chức và specialty workflow (quy trình chuyên khoa), nhưng **không nên khẳng định từ tài liệu công khai rằng Epic triển khai đúng mô hình “một master template + bật/tắt field theo tenant”**.

**Bài học áp dụng:** Với sản phẩm thương mại đóng, chỉ nên kết luận trong phạm vi tài liệu kiểm chứng được; tránh dùng tên Epic để chứng minh chi tiết kiến trúc nội bộ không công khai.

Nguồn:

- https://open.epic.com/TechnicalSpecifications

---

## 4. Đánh giá lại câu trả lời trước

### 4.1 Những điểm đúng

- Các hệ thống lớn thực sự dùng template, component dùng chung, form động và cấu hình theo ngữ cảnh.
- Better Studio là ví dụ tốt về form liên kết template openEHR và render động.
- EHRbase cung cấp WebTemplate/SDT phù hợp để xây UI động.
- Oracle Health PowerForms là ví dụ mạnh về section và control có điều kiện.
- Việc tách dữ liệu lâm sàng khỏi giao diện là hướng đúng.

### 4.2 Những điểm cần điều chỉnh

Câu “Epic và Cerner vận hành theo đúng mô hình này” là quá mạnh nếu không giới hạn phạm vi. Tài liệu công khai cho thấy các khả năng cấu hình tương tự, nhưng không đủ để chứng minh kiến trúc nội bộ hoàn toàn giống thiết kế đang đề xuất.

Câu “không nên tạo DermatologyTemplate, CardiologyTemplate...” cũng cần sửa. Trong openEHR, nhiều template theo use case là bình thường và cần thiết. Vấn đề không phải số lượng template, mà là:

- Có tái sử dụng archetype hay không.
- Có governance (quản trị) và versioning (quản lý phiên bản) hay không.
- Template có phạm vi và ownership (quyền sở hữu) rõ không.
- Có tránh fork (tách nhánh) vô kiểm soát theo từng tenant không.

### 4.3 Câu trả lời chuyên gia đã hiệu chỉnh

> **Có. Better Studio, EHRbase và Oracle Health EHR thể hiện rõ các pattern tương tự như template-driven data, reusable clinical components, dynamic forms, conditional sections và cấu hình theo ngữ cảnh. Tuy nhiên, hệ thống lớn thường không dùng một master template khổng lồ cho mọi khoa. Họ duy trì một kho mô hình lâm sàng dùng chung, tạo các template có phạm vi rõ theo encounter/use case, sau đó áp dụng form definition và configuration theo cơ sở, khoa, vai trò và quy trình.**

---

## 5. Kiến trúc đề xuất cho hệ thống của bạn

## 5.1 Không dùng một template duy nhất cho toàn bộ chuyên khoa

Không nên:

```text
outpatient-everything.opt
  common
  dermatology
  cardiology
  ophthalmology
  ent
  obstetrics
  pediatrics
  ...
```

Rủi ro:

- Template tree (cây template) quá lớn.
- Payload nặng và validation phức tạp.
- Một thay đổi nhỏ có thể ảnh hưởng nhiều khoa.
- Khó versioning và rollback (quay lui phiên bản).
- Field ẩn trên UI vẫn có thể tồn tại trong template/API.
- Tenant customization dễ làm semantic drift (lệch ngữ nghĩa).

## 5.2 Dùng template theo encounter/use case

Nên có một bộ OPT có phạm vi rõ:

```text
outpatient-core.v1
outpatient-dermatology.v1
outpatient-cardiology.v1
outpatient-ophthalmology.v1
derma-procedure.v1
prescription.v1
lab-order.v1
```

Trong đó các template tái sử dụng archetype chung:

```text
blood_pressure
body_temperature
problem_diagnosis
medication_order
clinical_synopsis
```

## 5.3 Tách Clinical Template và UI Form

```text
Operational Template - OPT
(Mẫu vận hành, đảm bảo dữ liệu và ngữ nghĩa)
        ↓
WebTemplate / SDT
(Cấu trúc đơn giản hóa cho ứng dụng)
        ↓
UI Schema
(Bố cục, widget, điều kiện, nhãn)
        ↓
Tenant Profile
(Cấu hình cơ sở)
        ↓
Role/Workflow Policy
(Chính sách theo vai trò và quy trình)
        ↓
React Form Renderer
(Bộ kết xuất form React)
```

## 5.4 Tenant chỉ override lớp trình bày trong phạm vi cho phép

Cho phép tenant cấu hình:

- Thứ tự section.
- Nhãn địa phương.
- Trường optional (tùy chọn) được ẩn/hiện.
- Default value (giá trị mặc định) an toàn.
- Specialty module (mô-đun chuyên khoa) được kích hoạt.
- Workflow và quyền truy cập.

Không cho phép tenant tự ý:

- Tắt trường bắt buộc về lâm sàng/pháp lý.
- Đổi datatype (kiểu dữ liệu).
- Đổi terminology binding (liên kết thuật ngữ).
- Đổi meaning/code của node.
- Sửa cardinality làm dữ liệu không còn phù hợp OPT.
- Thay đổi template đang dùng cho hồ sơ đã ký mà không tạo version mới.

---

## 6. Ví dụ cấu hình khám da liễu

### 6.1 Template Binding (Liên kết mẫu)

```json
{
  "workflow": "OUTPATIENT_VISIT",
  "specialty": "DERMATOLOGY",
  "templateId": "outpatient-dermatology.v1",
  "uiSchemaId": "dermatology-standard.v3"
}
```

### 6.2 Tenant Profile (Hồ sơ cấu hình phòng khám)

```json
{
  "tenantId": "clinic-a",
  "templateId": "outpatient-dermatology.v1",
  "uiSchemaId": "dermatology-standard.v3",
  "enabledOptionalSections": [
    "bodyMap",
    "clinicalPhotography",
    "pasiAssessment"
  ],
  "disabledOptionalSections": [
    "woodLampAssessment",
    "dlqiAssessment"
  ],
  "requiredFields": [
    "chiefComplaint",
    "lesionLocation",
    "clinicalDiagnosis"
  ]
}
```

### 6.3 Resolution Order (Thứ tự hợp nhất cấu hình)

```text
Platform Default
(Mặc định nền tảng)
    ↓
Template Policy
(Chính sách của template)
    ↓
Tenant Profile
(Cấu hình phòng khám)
    ↓
Department Profile
(Cấu hình khoa)
    ↓
Role Policy
(Chính sách vai trò)
    ↓
Workflow State
(Trạng thái quy trình)
```

Nguyên tắc: lớp sau chỉ được thu hẹp hoặc tùy chỉnh trong phạm vi mà lớp trước cho phép; không được phá vỡ clinical constraint (ràng buộc lâm sàng).

---

## 7. Mô hình dữ liệu cấu hình gợi ý

```text
clinical_template
- id
- template_id
- version
- specialty_code
- encounter_type
- status
- opt_content
- checksum

ui_schema
- id
- template_id
- version
- schema_json
- status

tenant_template_profile
- tenant_id
- template_id
- ui_schema_id
- enabled
- effective_from
- effective_to

field_override
- tenant_profile_id
- node_path
- visible
- required_override
- label_override
- display_order
- default_value_json

form_assignment
- tenant_id
- department_id
- role_id
- workflow_code
- workflow_state
- template_id
- ui_schema_id
```

Cần lưu `node_path` theo path ổn định của WebTemplate/openEHR, không dùng tên hiển thị làm khóa.

---

## 8. Governance (Quản trị) bắt buộc

### 8.1 Versioning (Quản lý phiên bản)

- Template và UI schema có version độc lập.
- Hồ sơ đã lưu phải giữ template ID/version đã dùng.
- Không sửa trực tiếp version đã phát hành.
- Migration (chuyển đổi) dữ liệu cũ chỉ thực hiện khi có mapping và kiểm thử.

### 8.2 Lifecycle (Vòng đời)

```text
DRAFT → REVIEW → APPROVED → ACTIVE → RETIRED
```

### 8.3 Testing (Kiểm thử)

Tối thiểu cần:

- Schema validation (kiểm tra cấu trúc).
- Composition validation với OPT.
- Snapshot test cho UI schema.
- Regression test theo tenant.
- Authorization test theo role và department.
- Kiểm thử hồ sơ ở trạng thái draft, signed, amended và cancelled.

### 8.4 Audit (Kiểm toán)

Mọi thay đổi cấu hình phải ghi:

- Ai thay đổi.
- Thời điểm.
- Giá trị trước/sau.
- Tenant/phòng khám bị ảnh hưởng.
- Lý do thay đổi.
- Version được phát hành.

---

## 9. Quyết định kiến trúc khuyến nghị

### Quyết định

Chọn mô hình:

> **Shared Clinical Model Library (Kho mô hình lâm sàng dùng chung) + Use-case Templates (Template theo mục đích) + Metadata-driven UI (UI điều khiển bằng metadata) + Controlled Tenant Profiles (Cấu hình tenant có kiểm soát).**

### Không chọn

- Một OPT duy nhất chứa mọi khoa.
- Mỗi tenant tự fork OPT riêng.
- Dùng form JSON tùy ý mà không liên kết node openEHR.
- Dùng hidden field (trường bị ẩn) như cơ chế bảo mật.

### Lý do

Mô hình đề xuất cân bằng được:

- Tái sử dụng.
- Khả năng cấu hình.
- Hiệu năng.
- An toàn lâm sàng.
- Quản lý phiên bản.
- Cô lập tenant.
- Khả năng phát triển thêm chuyên khoa.

---

## 10. Kết luận cuối cùng

Có đủ bằng chứng thực tế để khẳng định rằng các hệ thống lớn đã sử dụng **những pattern cốt lõi tương tự**: template, thành phần dùng chung, form động, section có điều kiện và cấu hình theo ngữ cảnh/tổ chức.

Nhưng đối với hệ thống HIS/EMR SaaS đang xây dựng, kiến trúc tốt nhất không phải là:

```text
One Master Template + bật/tắt mọi field theo tenant
```

mà là:

```text
Shared Archetypes
(Các archetype dùng chung)
        +
Purpose-specific OPTs
(Các mẫu vận hành theo mục đích)
        +
UI Schemas
(Lược đồ giao diện)
        +
Tenant/Department/Role Profiles
(Cấu hình phòng khám/khoa/vai trò)
        +
React Renderer
(Bộ kết xuất React)
```

Đây là cách tiếp cận gần hơn với các nền tảng trưởng thành, đồng thời tránh được template phình to, phân nhánh theo tenant và mất kiểm soát ngữ nghĩa lâm sàng.
