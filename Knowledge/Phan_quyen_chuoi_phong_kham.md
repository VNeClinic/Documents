# Hiểu kiến trúc phân quyền cho chuỗi phòng khám — Phiên bản dễ hiểu

> **Dành cho:** lập trình viên mới hoặc ở mức middle, chưa từng thiết kế hệ thống phân quyền lớn.
>
> **Cách đọc:** đọc lần lượt từ Phần 1 đến Phần 7. Phần 7 (luồng dữ liệu) là phần quan trọng nhất. Gặp thuật ngữ lạ thì tra ở **Phần 11 – Bảng thuật ngữ**.
>
> **Cách xem sơ đồ:** tài liệu dùng sơ đồ Mermaid. Sơ đồ hiển thị tự động khi mở bằng GitHub, GitLab, Obsidian, Typora, hoặc VS Code (cài extension *Markdown Preview Mermaid Support*). Nếu mở bằng trình xem không hỗ trợ, bạn sẽ thấy mã nguồn sơ đồ; có thể dán vào https://mermaid.live để xem.

---

## Mục lục

1. Ý tưởng cốt lõi trong 1 phút
2. Hệ thống được tổ chức như thế nào
3. Các nhân vật dùng làm ví dụ xuyên suốt
4. Ba viên gạch nền móng: Permission, Role, Role Assignment
5. Phạm vi (Scope) và kế thừa quyền
6. Hai lớp quyền: "Bạn là ai" và "Bạn đang làm gì"
7. **Luồng dữ liệu — từng bước một**
8. Ba lớp bảo vệ
9. Cache — vì sao cần và dùng thế nào
10. Một ngày làm việc: tổng hợp các tình huống
11. Bảng thuật ngữ
12. Câu hỏi thường gặp
13. Gợi ý thứ tự triển khai

---

## Phần 1. Ý tưởng cốt lõi trong 1 phút

Hãy tưởng tượng hệ thống phần mềm là **một tòa nhà phòng khám**.

Khi bạn bước vào, có hai việc hoàn toàn khác nhau xảy ra:

| Câu hỏi | Ví dụ ngoài đời | Tên kỹ thuật | Ai làm |
|---|---|---|---|
| **Bạn là ai?** | Bảo vệ kiểm tra thẻ nhân viên ở cổng | **Authentication** (xác thực, viết tắt AuthN) | Hệ thống đăng nhập |
| **Bạn được làm gì?** | Thẻ của bạn mở được những cánh cửa nào | **Authorization** (phân quyền, viết tắt AuthZ) | Hệ thống phân quyền |

Tài liệu này chủ yếu nói về câu hỏi thứ hai: **phân quyền**.

Toàn bộ kiến trúc có thể tóm gọn trong một câu:

> **Mỗi khi ai đó muốn làm gì, hệ thống hỏi: "Người này có được làm việc này, với dữ liệu này, ở nơi này, vào lúc này không?" — và câu trả lời phải đến từ một nơi duy nhất, nhất quán.**

Bốn chữ "việc này", "dữ liệu này", "nơi này", "lúc này" chính là bốn thứ mà các phần sau sẽ giải thích.

---

## Phần 2. Hệ thống được tổ chức như thế nào

### 2.1 Ví dụ: Chuỗi Phòng khám An Tâm

```mermaid
flowchart TD
    T["TENANT<br/>Chuỗi Phòng khám An Tâm"]
    R1["REGION - Miền Bắc"]
    R2["REGION - Miền Nam"]
    F1["FACILITY - Cơ sở Cầu Giấy"]
    F2["FACILITY - Cơ sở Hà Đông"]
    F3["FACILITY - Cơ sở Quận 1"]
    U1["UNIT - Phòng khám Nội 3"]
    U2["UNIT - Phòng khám Nội 5"]
    U3["UNIT - Phòng khám Nhi 1"]
    U4["UNIT - Quầy tiếp đón"]
    U5["UNIT - Quầy thu ngân"]
    U6["UNIT - Kho dược"]

    T --> R1
    T --> R2
    R1 --> F1
    R1 --> F2
    R2 --> F3
    F1 --> U1
    F1 --> U2
    F1 --> U3
    F1 --> U4
    F1 --> U5
    F1 --> U6
```

### 2.2 Giải thích từng cấp

| Cấp | Tên trong code | Nghĩa đời thường | Ví dụ |
|---|---|---|---|
| **Tenant** | `Tenant` | "Khách hàng" sử dụng phần mềm, thường là một công ty/thương hiệu. Dữ liệu giữa hai tenant **tuyệt đối không được nhìn thấy nhau**. | Chuỗi Phòng khám An Tâm |
| **Region** *(không bắt buộc)* | `Region` | Vùng/cụm, dùng khi có người quản lý nhiều cơ sở cùng lúc | Miền Bắc |
| **Facility** | `Facility` | **Cơ sở vật lý**: có địa chỉ, có giấy phép, có nhân viên | Cơ sở Cầu Giấy |
| **Unit** | `Unit` | **Đơn vị bên trong cơ sở**: phòng khám chuyên khoa, quầy, kho, phòng xét nghiệm | Phòng khám Nội 3, Kho dược |

### 2.3 Hai lưu ý quan trọng về tên gọi

**Lưu ý 1 — Tránh chữ "phòng khám" mơ hồ.** Trong tiếng Việt, "phòng khám" vừa là cả cơ sở ("Phòng khám Đa khoa X") vừa là một phòng bên trong ("Phòng khám Nội số 3"). Trong code, ta dùng `Facility` cho cơ sở và `Unit` cho phòng, để không ai hiểu nhầm.

**Lưu ý 2 — Vì sao gộp mọi loại phòng vào "Unit"?** Phòng khám, quầy tiếp đón, kho dược, phòng xét nghiệm đều là "một chỗ bên trong cơ sở có người làm việc". Gộp chung thành `Unit` và phân biệt bằng cột `unit_type` giúp:

- thêm loại phòng mới (ví dụ phòng vật lý trị liệu) chỉ cần thêm một giá trị `unit_type`, không cần thêm bảng mới;
- logic phân quyền chỉ phải xử lý một khái niệm.

```text
units
┌──────┬─────────────┬──────────────┬───────────┬────────────────────┐
│ id   │ facility_id │ unit_type    │ specialty │ name               │
├──────┼─────────────┼──────────────┼───────────┼────────────────────┤
│ U1   │ F1          │ exam_room    │ noi       │ Phòng khám Nội 3   │
│ U2   │ F1          │ exam_room    │ noi       │ Phòng khám Nội 5   │
│ U4   │ F1          │ reception    │ -         │ Quầy tiếp đón      │
│ U6   │ F1          │ pharmacy     │ -         │ Kho dược           │
└──────┴─────────────┴──────────────┴───────────┴────────────────────┘
```

### 2.4 Dữ liệu nằm ở cấp nào?

Đây là điểm người mới hay nhầm. **Không phải mọi dữ liệu đều thuộc về một cơ sở.**

| Dữ liệu | Thuộc cấp | Lý do |
|---|---|---|
| Bệnh nhân (`patients`) | **Tenant** | Bà Mai khám ở Cầu Giấy hôm nay, tháng sau tái khám ở Hà Đông — vẫn là một hồ sơ |
| Lượt khám (`encounters`) | **Facility** | Mỗi lượt khám diễn ra ở một cơ sở cụ thể |
| Lịch hẹn, hóa đơn | **Facility** | Gắn với nơi cung cấp dịch vụ |
| Hàng đợi khám (`queue_entries`) | **Unit** | Bệnh nhân xếp hàng chờ một phòng cụ thể |
| Tồn kho thuốc | **Unit** (kho) | Mỗi kho có số lượng riêng |

---

## Phần 3. Các nhân vật dùng làm ví dụ xuyên suốt

Toàn bộ tài liệu sẽ dùng những người này. Hãy nhớ tên họ.

| Nhân vật | Công việc | Làm việc ở |
|---|---|---|
| **BS Minh** | Bác sĩ nội khoa | Cơ sở Cầu Giấy, xoay giữa phòng Nội 3 và Nội 5 theo lịch |
| **Y tá Lan** | Điều dưỡng | Cơ sở Cầu Giấy, xoay phòng theo lịch |
| **Hộ lý Hùng** | Hộ lý, hỗ trợ di chuyển bệnh nhân | Cơ sở Cầu Giấy |
| **Chị Hoa** | Nhân viên tiếp đón | Quầy tiếp đón, Cơ sở Cầu Giấy |
| **Anh Tú** | Dược sĩ cấp phát | Kho dược, Cơ sở Cầu Giấy |
| **Anh Nam** | Quản lý cơ sở | Cơ sở Cầu Giấy |
| **Chị Thu** | Quản lý vùng | Toàn bộ Miền Bắc |
| **Bà Mai** | Bệnh nhân | Đang chờ khám ở phòng Nội 3 |

---

## Phần 4. Ba viên gạch nền móng: Permission, Role, Role Assignment

### 4.1 So sánh với chùm chìa khóa

| Khái niệm | So sánh đời thường | Ví dụ |
|---|---|---|
| **Permission** (quyền) | **Một chiếc chìa khóa** mở đúng một loại cửa | `patient.clinical.read` = chìa mở "xem hồ sơ bệnh án" |
| **Role** (vai trò) | **Một chùm chìa khóa** đã được gom sẵn | Role "Bác sĩ" = chùm gồm chìa xem bệnh án, chìa kê đơn, chìa chỉ định xét nghiệm |
| **Role Assignment** (giao vai trò) | **Giao chùm chìa cho ai, dùng ở tòa nhà nào** | Giao chùm "Bác sĩ" cho BS Minh, dùng ở Cơ sở Cầu Giấy |

```mermaid
flowchart LR
    P1["Permission<br/>patient.clinical.read"]
    P2["Permission<br/>prescription.create"]
    P3["Permission<br/>lab_order.create"]
    R["ROLE<br/>Bác sĩ"]
    A["ROLE ASSIGNMENT<br/>BS Minh + Bác sĩ<br/>tại Cơ sở Cầu Giấy"]
    U["USER<br/>BS Minh"]

    P1 --> R
    P2 --> R
    P3 --> R
    R --> A
    A --> U
```

### 4.2 Cách đặt tên permission

Quy ước: `tài_nguyên.hành_động` (có thể thêm cấp giữa).

| Permission | Nghĩa |
|---|---|
| `patient.demographics.read` | Xem thông tin hành chính bệnh nhân (tên, SĐT, địa chỉ) |
| `patient.clinical.read` | Xem thông tin lâm sàng (chẩn đoán, kết quả) |
| `patient.sensitive.read` | Xem dữ liệu nhạy cảm (HIV, tâm thần...) |
| `queue.view` | Xem hàng đợi khám |
| `prescription.create` | Kê đơn thuốc |
| `prescription.dispense` | Cấp phát thuốc theo đơn |
| `inventory.adjust` | Điều chỉnh tồn kho |
| `inventory.adjust.approve` | Duyệt phiếu điều chỉnh tồn kho |
| `invoice.create` | Tạo hóa đơn |

### 4.3 Ai sở hữu cái gì

| Thành phần | Ai tạo/sửa | Sửa có phải deploy code không? |
|---|---|---|
| Permission | **Lập trình viên**, viết trong code | Có — vì permission mới = tính năng mới |
| Role | **Admin**, qua màn hình quản trị | Không |
| Role Assignment | **Admin**, qua màn hình quản trị | Không |

### 4.4 Quy tắc vàng: code chỉ hỏi PERMISSION, không bao giờ hỏi tên ROLE

**Cách làm sai:**

```csharp
// ❌ SAI: code biết tên role
if (user.Role == "Doctor" || user.Role == "Nurse")
{
    return patient.Diagnosis;
}
```

Vấn đề: ngày mai admin tạo role mới "Bác sĩ thực tập". Code trên không biết role này tồn tại → bác sĩ thực tập bị chặn → phải sửa code, deploy lại.

**Cách làm đúng:**

```csharp
// ✅ ĐÚNG: code chỉ hỏi "có chìa khóa này không"
if (await authz.HasPermission(user, "patient.clinical.read", patient))
{
    return patient.Diagnosis;
}
```

Admin tạo role "Bác sĩ thực tập", tick chọn `patient.clinical.read` → code tự hoạt động đúng, **không cần sửa một dòng nào**.

```mermaid
flowchart LR
    subgraph ADMIN["Admin làm trên màn hình - không cần lập trình viên"]
        A1["Tạo role Bác sĩ thực tập"] --> A2["Tick chọn các permission có sẵn"]
        A2 --> A3["Giao role cho BS Khoa tại Cầu Giấy"]
    end
    subgraph CODE["Code - không thay đổi"]
        C1["API hỏi: BS Khoa có<br/>patient.clinical.read không?"]
    end
    A3 --> C1
    C1 --> OK["Có - cho phép"]
```

---

## Phần 5. Phạm vi (Scope) và kế thừa quyền

### 5.1 Scope là gì?

**Scope = "chùm chìa khóa này dùng được ở đâu"**. Cùng là role "Quản lý", nhưng:

- Anh Nam là Quản lý **ở Cơ sở Cầu Giấy** → chỉ quản lý Cầu Giấy.
- Chị Thu là Quản lý **ở Miền Bắc** → quản lý mọi cơ sở thuộc Miền Bắc.

### 5.2 Kế thừa: gán ở trên, tự có quyền ở dưới

Khi gán quyền ở một cấp, người đó **tự động có quyền ở mọi cấp con**. Giống như chìa khóa tổng của tòa nhà mở được mọi phòng trong tòa nhà.

```mermaid
flowchart TD
    T["Tenant An Tâm"]
    R1["Miền Bắc<br/>⭐ Chị Thu được gán ở đây"]
    F1["Cầu Giấy<br/>✅ Chị Thu tự có quyền<br/>⭐ Anh Nam được gán ở đây"]
    F2["Hà Đông<br/>✅ Chị Thu tự có quyền<br/>❌ Anh Nam không có quyền"]
    U1["Nội 3<br/>✅ Chị Thu<br/>✅ Anh Nam"]
    U2["Kho dược<br/>✅ Chị Thu<br/>✅ Anh Nam"]

    T --> R1
    R1 --> F1
    R1 --> F2
    F1 --> U1
    F1 --> U2
```

**Lợi ích:** Chị Thu quản lý 50 cơ sở, nhưng chỉ cần **1 dòng** gán quyền thay vì 50 dòng. Khi mở cơ sở thứ 51 ở Miền Bắc, chị Thu tự động có quyền, không ai phải làm gì.

### 5.3 Bảng `role_assignments` trông như thế nào

```text
role_assignments
┌───────────┬──────────────┬────────────┬──────────────┬─────────────┐
│ user      │ role         │ scope_type │ scope        │ valid_to    │
├───────────┼──────────────┼────────────┼──────────────┼─────────────┤
│ BS Minh   │ Bác sĩ       │ facility   │ Cầu Giấy     │ (không hạn) │
│ Y tá Lan  │ Điều dưỡng   │ facility   │ Cầu Giấy     │ (không hạn) │
│ Hộ lý Hùng│ Hộ lý        │ facility   │ Cầu Giấy     │ (không hạn) │
│ Chị Hoa   │ Tiếp đón     │ unit       │ Quầy tiếp đón│ (không hạn) │
│ Anh Tú    │ Dược sĩ      │ unit       │ Kho dược     │ (không hạn) │
│ Anh Nam   │ Quản lý      │ facility   │ Cầu Giấy     │ (không hạn) │
│ Chị Thu   │ Quản lý vùng │ region     │ Miền Bắc     │ (không hạn) │
│ BS Khoa   │ BS thực tập  │ facility   │ Cầu Giấy     │ 31/12/2026  │
└───────────┴──────────────┴────────────┴──────────────┴─────────────┘
```

Cột `valid_to` cho phép quyền **tự hết hạn**: BS Khoa thực tập đến hết năm, sau đó tự mất quyền mà không ai cần nhớ đi thu hồi.

---

## Phần 6. Hai lớp quyền: "Bạn là ai" và "Bạn đang làm gì"

### 6.1 Vấn đề

BS Minh có role Bác sĩ ở Cầu Giấy. Câu hỏi: **BS Minh có được xem bệnh án của MỌI bệnh nhân đang chờ ở Cầu Giấy không?**

Không nên. Cơ sở Cầu Giấy có 15 phòng khám. BS Minh chỉ nên thấy bệnh nhân **của phòng mình đang ngồi**. Nhưng BS Minh lại **đổi phòng theo lịch**:

| Thời gian | BS Minh ngồi ở |
|---|---|
| Thứ Hai, 7h–11h30 | Phòng Nội 3 |
| Thứ Hai, 13h–17h | Phòng Nội 5 |
| Thứ Ba | Nghỉ |

Nếu mỗi lần đổi phòng lại phải admin vào sửa role thì 1.000 cơ sở × hàng chục bác sĩ × 2 ca/ngày = **hàng chục nghìn lần sửa quyền mỗi ngày**. Không khả thi.

### 6.2 Giải pháp: tách làm hai lớp

| | **Lớp 1 — Năng lực** | **Lớp 2 — Ngữ cảnh vận hành** |
|---|---|---|
| Trả lời câu hỏi | "Bạn là ai, làm nghề gì, ở cơ sở nào?" | "Lúc này bạn đang làm gì, với bệnh nhân nào?" |
| Ví dụ | BS Minh là Bác sĩ ở Cầu Giấy | BS Minh đang trực phòng Nội 3, ca sáng thứ Hai |
| Thay đổi | Hiếm (khi tuyển dụng, chuyển công tác, nghỉ việc) | Liên tục (mỗi ca) |
| Ai nhập | Admin / nhân sự | Bộ phận điều phối xếp lịch (việc họ vốn đã làm) |
| Lưu ở bảng | `role_assignments` | `duty_rosters` (lịch trực), `encounters` (lượt khám) |

**Quyết định cuối cùng = Lớp 1 VÀ Lớp 2 đều đồng ý.**

```mermaid
flowchart TD
    Q["BS Minh muốn xem bệnh án của Bà Mai"]
    L1{"LỚP 1 - Năng lực<br/>BS Minh có permission<br/>patient.clinical.read<br/>tại Cầu Giấy?"}
    L2{"LỚP 2 - Ngữ cảnh<br/>Có ít nhất 1 lý do hợp lệ?"}
    R1["Bà Mai đang chờ ở phòng<br/>mà BS Minh đang trực"]
    R2["BS Minh đã từng khám<br/>Bà Mai gần đây"]
    R3["BS Minh nằm trong<br/>nhóm điều trị của Bà Mai"]
    YES["✅ CHO PHÉP"]
    NO["❌ TỪ CHỐI"]

    Q --> L1
    L1 -- Không --> NO
    L1 -- Có --> L2
    R1 -.-> L2
    R2 -.-> L2
    R3 -.-> L2
    L2 -- Có --> YES
    L2 -- Không --> NO
```

### 6.3 Kết quả theo thời gian

Bà Mai đang chờ ở phòng **Nội 3**.

| Lúc | BS Minh đang trực | Lớp 1 | Lớp 2 | Kết quả |
|---|---|---|---|---|
| Thứ Hai 9h00 | Nội 3 | ✅ | ✅ Bà Mai chờ ở Nội 3 | **Cho phép** |
| Thứ Hai 14h00 | Nội 5 | ✅ | ❌ Bà Mai không ở Nội 5 | **Từ chối** |
| Thứ Hai 14h00, nhưng BS Minh đã khám Bà Mai lúc sáng | Nội 5 | ✅ | ✅ Đã khám gần đây | **Cho phép** (để xem lại kết quả xét nghiệm) |
| Thứ Ba | Nghỉ | ✅ | ❌ | **Từ chối** |

Không ai phải sửa quyền. **Lịch thay đổi thì quyền tự thay đổi theo.**

### 6.4 Bảng lịch trực

```text
duty_rosters
┌───────────┬──────────┬───────────────────┬──────────────────┐
│ user      │ unit     │ shift_start       │ shift_end        │
├───────────┼──────────┼───────────────────┼──────────────────┤
│ BS Minh   │ Nội 3    │ 09/11/2026 07:00  │ 09/11/2026 11:30 │
│ BS Minh   │ Nội 5    │ 09/11/2026 13:00  │ 09/11/2026 17:00 │
│ Y tá Lan  │ Nội 3    │ 09/11/2026 07:00  │ 09/11/2026 17:00 │
│ Hộ lý Hùng│ Nội 3    │ 09/11/2026 07:00  │ 09/11/2026 11:30 │
└───────────┴──────────┴───────────────────┴──────────────────┘
```

### 6.5 Ai dùng lớp nào?

| Vai trò | Lớp 1 (role) | Lớp 2 (lịch/lượt khám) | Vì sao |
|---|---|---|---|
| Bác sĩ, Y tá, Hộ lý | ✅ | ✅ | Xoay phòng theo ca |
| Tiếp đón, Thu ngân, Dược sĩ, Thủ kho | ✅ | ❌ không cần | Ngồi cố định một chỗ, gán role thẳng vào Unit là đủ |
| Quản lý cơ sở, Quản lý vùng | ✅ | ❌ không cần | Xem báo cáo tổng hợp; xem bệnh án chỉ qua "truy cập khẩn cấp" |

---

## Phần 7. Luồng dữ liệu — từng bước một

Đây là phần quan trọng nhất. Mỗi luồng gồm: **tình huống → sơ đồ → giải thích từng bước**.

### 7.1 Bản đồ tổng thể: các thành phần trong hệ thống

```mermaid
flowchart TD
    APP["📱 Ứng dụng<br/>Web / Mobile"]
    IDP["🪪 Hệ thống đăng nhập<br/>IdP - ví dụ Keycloak"]
    GW["🚪 Cổng API<br/>API Gateway"]
    SVC["⚙️ Service nghiệp vụ<br/>Khám bệnh, Dược, Thu ngân..."]
    AUTHZ["⚖️ Dịch vụ phân quyền<br/>Authorization Service"]
    CACHE["⚡ Bộ nhớ đệm<br/>Redis"]
    DB[("🗄️ Cơ sở dữ liệu<br/>PostgreSQL")]
    AUDIT[("📒 Nhật ký truy cập<br/>Audit Log")]

    APP -- "1. Đăng nhập" --> IDP
    IDP -- "2. Trả về token" --> APP
    APP -- "3. Gọi API kèm token" --> GW
    GW -- "4. Chuyển tiếp" --> SVC
    GW -. "kiểm tra sơ bộ" .-> AUTHZ
    SVC -- "5. Hỏi: được phép không?" --> AUTHZ
    AUTHZ <--> CACHE
    AUTHZ <--> DB
    SVC -- "6. Đọc/ghi dữ liệu" --> DB
    SVC -- "7. Ghi lại ai đã làm gì" --> AUDIT
```

| Thành phần | Vai trò đời thường | Nhiệm vụ |
|---|---|---|
| **Hệ thống đăng nhập (IdP)** | Phòng cấp thẻ nhân viên | Kiểm tra mật khẩu, mã OTP; cấp "thẻ" (token) chứng minh bạn là ai |
| **Cổng API (Gateway)** | Bảo vệ ở cổng tòa nhà | Kiểm tra thẻ có hợp lệ không, chặn sơ bộ các yêu cầu rõ ràng không được phép |
| **Service nghiệp vụ** | Nhân viên các phòng ban | Xử lý công việc thật: khám bệnh, kê đơn, thu tiền |
| **Dịch vụ phân quyền** | Phòng pháp chế — nơi duy nhất ra quyết định "được hay không" | Nhận câu hỏi, tra quy định, trả lời CÓ/KHÔNG |
| **Bộ nhớ đệm (Redis)** | Sổ tay ghi nhớ | Lưu tạm câu trả lời để không phải tra lại từ đầu mỗi lần |
| **Cơ sở dữ liệu** | Kho lưu trữ hồ sơ | Lưu mọi dữ liệu; tự chặn nếu ai đó cố đọc dữ liệu tenant khác |
| **Nhật ký truy cập** | Sổ ghi chép ra vào | Ghi lại mọi lượt xem/sửa dữ liệu bệnh nhân |

> 💡 **Điều quan trọng nhất:** các service nghiệp vụ **không tự quyết định** quyền. Chúng luôn **hỏi** Dịch vụ phân quyền. Nhờ vậy, quy tắc chỉ viết ở một chỗ, không bị mỗi service hiểu một kiểu.

---

### 7.2 Luồng đăng nhập

**Tình huống:** BS Minh mở ứng dụng lúc 6h55 sáng thứ Hai.

```mermaid
sequenceDiagram
    actor M as BS Minh
    participant APP as Ứng dụng
    participant IDP as Hệ thống đăng nhập

    M->>APP: Mở ứng dụng
    APP->>IDP: Chuyển sang trang đăng nhập
    M->>IDP: Nhập tên đăng nhập + mật khẩu
    IDP->>M: Yêu cầu mã OTP
    M->>IDP: Nhập mã OTP
    IDP->>IDP: Kiểm tra đúng người
    IDP->>APP: Trả về token
    APP->>M: Chọn cơ sở làm việc hôm nay
    M->>APP: Chọn Cơ sở Cầu Giấy
```

**Token trông như thế nào?** Token là một chuỗi mã hóa. Khi giải mã, bên trong chứa:

```json
{
  "sub": "user-bs-minh",
  "tid": "tenant-an-tam",
  "active_facility": "facility-cau-giay",
  "exp": 1762736100
}
```

| Trường | Nghĩa |
|---|---|
| `sub` | Mã định danh người dùng (BS Minh) |
| `tid` | Thuộc tenant nào (Chuỗi An Tâm) |
| `active_facility` | Đang làm việc ở cơ sở nào |
| `exp` | Thời điểm token hết hạn (thường 15–30 phút, sau đó tự làm mới) |

> ⚠️ **Vì sao token KHÔNG chứa danh sách quyền?**
>
> 1. **Thu hồi không kịp:** nếu quyền nằm trong token, khi admin thu hồi quyền của ai đó, token cũ vẫn mang quyền cũ cho đến khi hết hạn.
> 2. **Token quá to:** Chị Thu có quyền ở 50 cơ sở, mỗi cơ sở hàng chục quyền → token phình to, mỗi request đều phải gửi đi.
> 3. **Lớp 2 (lịch trực) thay đổi theo giờ** — không thể đóng gói sẵn vào token.
>
> → Token chỉ nói "**bạn là ai**". Còn "**bạn được làm gì**" thì hỏi Dịch vụ phân quyền mỗi lần.

---

### 7.3 Luồng kiểm tra quyền cho một yêu cầu

**Tình huống:** 9h00 thứ Hai, BS Minh (đang trực Nội 3) bấm vào hồ sơ của Bà Mai (đang chờ ở Nội 3).

```mermaid
sequenceDiagram
    actor M as BS Minh
    participant APP as Ứng dụng
    participant GW as Cổng API
    participant SVC as Service Khám bệnh
    participant AZ as Dịch vụ phân quyền
    participant C as Redis
    participant DB as Database
    participant LOG as Nhật ký

    M->>APP: Bấm xem hồ sơ Bà Mai
    APP->>GW: GET /patients/ba-mai/clinical kèm token
    GW->>GW: Bước 1 - Token còn hạn và hợp lệ?
    GW->>SVC: Chuyển tiếp yêu cầu
    SVC->>AZ: Bước 2 - BS Minh được patient.clinical.read với Bà Mai không?

    AZ->>C: Bước 3 - Tra sổ tay Lớp 1
    C-->>AZ: BS Minh là Bác sĩ ở Cầu Giấy

    AZ->>DB: Bước 4 - Tra Lớp 2
    Note over AZ,DB: Lúc này BS Minh trực phòng nào?<br/>Bà Mai đang chờ phòng nào?
    DB-->>AZ: BS Minh trực Nội 3 - Bà Mai chờ Nội 3

    AZ-->>SVC: Bước 5 - CHO PHÉP, lý do on_duty_unit
    SVC->>DB: Bước 6 - Đọc hồ sơ Bà Mai
    DB-->>SVC: Dữ liệu hồ sơ
    SVC->>LOG: Bước 7 - Ghi nhật ký
    SVC-->>APP: Trả hồ sơ
    APP-->>M: Hiển thị hồ sơ
```

**Giải thích từng bước:**

| Bước | Việc gì xảy ra | Nếu thất bại thì sao |
|---|---|---|
| 1 | Cổng API kiểm tra token: còn hạn không, có bị làm giả không | Trả lỗi `401 Unauthorized` — "bạn chưa đăng nhập" |
| 2 | Service khám bệnh **không tự quyết**, mà gửi câu hỏi cho Dịch vụ phân quyền | — |
| 3 | Tra Lớp 1 (role, permission) trong Redis cho nhanh. Nếu Redis chưa có, tra database rồi lưu vào Redis | Không có permission → từ chối ngay |
| 4 | Tra Lớp 2: lịch trực hiện tại và hàng đợi | Không có lý do hợp lệ → từ chối |
| 5 | Trả quyết định **kèm lý do** (để ghi nhật ký) | Trả lỗi `403 Forbidden` — "bạn không có quyền" |
| 6 | Đọc dữ liệu. Database **tự kiểm tra thêm một lần nữa** rằng dữ liệu thuộc đúng tenant (xem Phần 8) | Database trả về rỗng |
| 7 | Ghi lại: ai, xem gì, của bệnh nhân nào, lúc nào, vì lý do gì | — |

**Mã giả cho Bước 2 ở service:**

```csharp
[HttpGet("patients/{id}/clinical")]
public async Task<IActionResult> GetClinical(Guid id)
{
    var result = await authz.Check(
        user:       CurrentUser,
        permission: "patient.clinical.read",
        resource:   $"patient:{id}");

    if (!result.Allowed)
        return Forbid();                 // 403

    var record = await db.GetClinicalRecord(id);
    await audit.Log(CurrentUser, "patient.clinical.read", id, result.Reason);
    return Ok(record);
}
```

---

### 7.4 Luồng xem danh sách (hàng đợi khám)

**Tình huống:** 9h00, BS Minh mở màn hình "Bệnh nhân chờ khám".

**Vấn đề:** Cơ sở Cầu Giấy có 300 bệnh nhân đang chờ ở 15 phòng. Nếu lấy cả 300 người ra rồi hỏi Dịch vụ phân quyền 300 lần ("người này được xem không?") thì **rất chậm**.

**Giải pháp: đảo ngược câu hỏi.**

| ❌ Cách chậm | ✅ Cách nhanh |
|---|---|
| Lấy 300 bệnh nhân, hỏi từng người: "BS Minh được xem người này không?" | Hỏi **một lần**: "Lúc này BS Minh được xem bệnh nhân của những phòng nào?" → "Nội 3". Rồi chỉ lấy bệnh nhân phòng Nội 3. |

```mermaid
flowchart LR
    A["BS Minh mở<br/>màn hình hàng đợi"] --> B{"Dịch vụ phân quyền:<br/>BS Minh có<br/>queue.view tại Cầu Giấy?"}
    B -- Không --> X["❌ 403"]
    B -- Có --> C["Database tự tìm:<br/>BS Minh đang trực<br/>phòng nào lúc này?"]
    C --> D["Kết quả: Nội 3"]
    D --> E["Chỉ lấy bệnh nhân<br/>đang chờ ở Nội 3"]
    E --> F["Trả về 20 bệnh nhân<br/>thay vì 300"]
```

**Câu SQL (đã đơn giản hóa):**

```sql
SELECT q.ticket_no, p.full_name, q.arrived_at
FROM   queue_entries q
JOIN   duty_rosters  r ON r.unit_id = q.unit_id      -- nối hàng đợi với lịch trực
JOIN   patients      p ON p.id      = q.patient_id
WHERE  r.user_id = 'user-bs-minh'                    -- lịch trực của BS Minh
  AND  now() BETWEEN r.shift_start AND r.shift_end   -- đúng ca hiện tại
  AND  q.status = 'waiting'
ORDER  BY q.ticket_no;
```

Đọc câu SQL theo tiếng Việt: *"Lấy các bệnh nhân đang chờ, ở những phòng mà BS Minh có lịch trực vào đúng lúc này, sắp xếp theo số thứ tự."*

**Kết quả cùng một màn hình, khác người xem:**

| Người mở màn hình | Thấy gì |
|---|---|
| BS Minh, 9h00 | 20 bệnh nhân chờ ở Nội 3 |
| BS Minh, 14h00 | Bệnh nhân chờ ở Nội 5 |
| Y tá Lan, 9h00 | Bệnh nhân chờ ở Nội 3 (cũng trực Nội 3) |
| Anh Nam (quản lý) | Tổng số chờ ở từng phòng — không thấy tên bệnh nhân |

---

### 7.5 Luồng che bớt dữ liệu: cùng hồ sơ, mỗi người thấy một phần

**Tình huống:** Bà Mai có chẩn đoán "Viêm dạ dày" và một ghi chú nhạy cảm. Ba người cùng mở hồ sơ của Bà Mai.

```mermaid
flowchart TD
    REC["Hồ sơ đầy đủ của Bà Mai<br/>Tên, SĐT, Địa chỉ, BHYT<br/>Chẩn đoán, Kết quả xét nghiệm<br/>Ghi chú nhạy cảm"]

    REC --> F{"Bộ lọc theo permission<br/>của từng người"}

    F --> H["Chị Hoa - Tiếp đón<br/>Có: demographics.read<br/>Thấy: Tên, SĐT, Địa chỉ, BHYT"]
    F --> L["Y tá Lan<br/>Có: demographics.read, clinical.read<br/>Thấy: thêm Chẩn đoán, Kết quả"]
    F --> M["BS Minh - đang khám Bà Mai<br/>Có: thêm sensitive.read qua nhóm điều trị<br/>Thấy: toàn bộ"]
    F --> HU["Hộ lý Hùng<br/>Chỉ có: task.view<br/>Thấy: Tên, Vị trí cần đưa đến"]
```

**Mã giả:**

```csharp
var perms = await authz.GetPermissions(CurrentUser, patient);

var dto = new PatientView
{
    Name      = patient.Name,
    Phone     = perms.Has("patient.demographics.read") ? patient.Phone     : null,
    Diagnosis = perms.Has("patient.clinical.read")     ? patient.Diagnosis : null,
    Sensitive = perms.Has("patient.sensitive.read")    ? patient.Sensitive : null,
};
```

Màn hình chỉ hiển thị những trường không bị `null`.

---

### 7.6 Luồng admin thay đổi quyền

**Tình huống:** Anh Nam (quản lý) muốn Y tá Lan được phép **cấp phát thuốc** tạm thời trong 1 tuần vì Anh Tú (dược sĩ) nghỉ phép.

```mermaid
sequenceDiagram
    actor N as Anh Nam
    participant UI as Màn hình quản trị
    participant AZ as Dịch vụ phân quyền
    participant DB as Database
    participant Q as Hộp thư sự kiện
    participant C as Redis

    N->>UI: Giao role Dược sĩ cho Y tá Lan<br/>tại Kho dược, đến 16/11
    UI->>AZ: Yêu cầu gán quyền
    AZ->>AZ: Anh Nam có quyền gán quyền không?
    AZ->>DB: Ghi role_assignments + ghi sự kiện<br/>trong CÙNG một giao dịch
    Note over DB: Ghi cùng giao dịch để<br/>không bao giờ có chuyện<br/>quyền đã ghi mà sự kiện bị mất
    DB-->>AZ: Thành công
    Q->>Q: Đọc sự kiện: quyền của Lan đã đổi
    Q->>C: Tăng số phiên bản quyền của Lan
    Note over C: Sổ tay cũ của Lan tự hết hiệu lực
    AZ->>DB: Ghi nhật ký: Anh Nam đã gán quyền cho Lan
```

**Sau đó:**

- Lần tiếp theo Y tá Lan mở màn hình kho → Dịch vụ phân quyền thấy phiên bản đã đổi → tra lại database → thấy quyền mới → **cho phép**.
- Ngày 16/11 → `valid_to` hết hạn → quyền **tự biến mất**. Anh Nam không cần nhớ thu hồi.

> 💡 **"Hộp thư sự kiện" (Outbox) là gì?** Là một bảng trong database, nơi ghi lại "vừa có thay đổi gì". Một tiến trình chạy nền đọc bảng này và thông báo cho các nơi cần biết (ví dụ: xóa sổ tay cũ trong Redis). Vì được ghi cùng lúc với thay đổi quyền, nó đảm bảo **không có thay đổi nào bị bỏ sót**.

---

### 7.7 Luồng thu hồi quyền khi nhân viên nghỉ việc

**Tình huống:** 10h00, Hộ lý Hùng bị cho nghỉ việc ngay lập tức. Lúc đó Hùng vẫn đang đăng nhập trên điện thoại.

```mermaid
sequenceDiagram
    actor N as Anh Nam
    participant UI as Màn hình quản trị
    participant AZ as Dịch vụ phân quyền
    participant IDP as Hệ thống đăng nhập
    participant GW as Cổng API
    participant C as Redis
    actor H as Hộ lý Hùng

    N->>UI: Khóa tài khoản Hùng
    UI->>AZ: Thu hồi mọi role của Hùng
    UI->>IDP: Hủy mọi phiên đăng nhập của Hùng
    AZ->>C: Tăng phiên bản quyền của Hùng
    IDP->>GW: Đưa token hiện tại của Hùng vào danh sách đen
    Note over GW: Toàn bộ quá trình dưới 5 giây

    H->>GW: 10h00m05s - Hùng bấm vào ứng dụng
    GW-->>H: ❌ 401 - Token đã bị thu hồi
```

Có **ba lớp** cùng chặn cùng lúc: hủy phiên đăng nhập, đưa token vào danh sách đen, và xóa quyền. Kể cả nếu một lớp có trục trặc, hai lớp còn lại vẫn chặn được.

---

### 7.8 Luồng truy cập khẩn cấp (Break-glass)

**Tình huống:** 15h00, một bệnh nhân ngất tại hành lang. BS Minh chạy đến. Bệnh nhân này không nằm trong hàng đợi của phòng BS Minh, nên bình thường BS Minh **không được** xem bệnh án. Nhưng cần biết ngay bệnh nhân có dị ứng thuốc không.

Tên "break-glass" lấy từ hình ảnh **đập kính lấy bình chữa cháy**: ai cũng được làm khi khẩn cấp, nhưng ai làm cũng bị ghi lại và phải giải trình.

```mermaid
sequenceDiagram
    actor M as BS Minh
    participant APP as Ứng dụng
    participant AZ as Dịch vụ phân quyền
    participant DB as Database
    participant ALERT as Cảnh báo
    actor N as Anh Nam

    M->>APP: Mở hồ sơ bệnh nhân
    APP-->>M: ❌ Không có quyền
    M->>APP: Bấm nút Truy cập khẩn cấp
    APP->>M: Bắt buộc nhập lý do
    M->>APP: Bệnh nhân ngất, cần kiểm tra dị ứng
    APP->>AZ: Yêu cầu truy cập khẩn cấp
    AZ->>DB: Thêm BS Minh vào nhóm điều trị<br/>có hạn 4 giờ
    AZ->>ALERT: Gửi cảnh báo ngay
    ALERT->>N: 🔔 BS Minh vừa dùng truy cập khẩn cấp
    AZ-->>APP: ✅ Cho phép trong 4 giờ
    APP-->>M: Hiển thị hồ sơ
    Note over N: Ngày hôm sau, Anh Nam<br/>xem lại và xác nhận hợp lệ
```

---

### 7.9 Luồng ghi nhật ký truy cập

Mọi lượt **xem** và **sửa** dữ liệu bệnh nhân đều được ghi lại. Không chỉ ghi khi sửa, mà cả khi chỉ xem.

```mermaid
flowchart LR
    S1["Service Khám bệnh"] --> Q["Hàng đợi ghi log"]
    S2["Service Dược"] --> Q
    S3["Service Thu ngân"] --> Q
    S4["Dịch vụ phân quyền"] --> Q
    Q --> A[("Kho nhật ký<br/>chỉ được thêm<br/>không được sửa, xóa")]
    A --> R1["Báo cáo: Ai đã xem<br/>hồ sơ của tôi?"]
    A --> R2["Hệ thống giám sát<br/>phát hiện bất thường"]
```

**Một dòng nhật ký trông như thế nào:**

```json
{
  "time": "2026-11-09T09:00:12+07:00",
  "user": "user-bs-minh",
  "action": "patient.clinical.read",
  "patient": "patient-ba-mai",
  "facility": "facility-cau-giay",
  "decision": "allow",
  "reason": "on_duty_unit:noi-3",
  "ip": "10.2.15.33"
}
```

**Ví dụ phát hiện bất thường:** một nhân viên tiếp đón xem 500 hồ sơ trong 10 phút lúc 23h → hệ thống giám sát tự gửi cảnh báo.

---

## Phần 8. Ba lớp bảo vệ

### 8.1 So sánh với tòa nhà

```mermaid
flowchart LR
    R["Yêu cầu từ<br/>người dùng"] --> L1["🚪 LỚP 1<br/>Cổng API<br/>= Bảo vệ cổng tòa nhà"]
    L1 --> L2["🔑 LỚP 2<br/>Service + Dịch vụ phân quyền<br/>= Khóa cửa từng phòng"]
    L2 --> L3["🔒 LỚP 3<br/>Database tự kiểm tra<br/>= Két sắt trong phòng"]
    L3 --> D["Dữ liệu"]
```

| Lớp | Kiểm tra gì | Ví dụ chặn được |
|---|---|---|
| 1. Cổng API | Token hợp lệ? Có quyền gì đó ở cơ sở này không? | Người đã nghỉ việc, token giả |
| 2. Service | Được làm đúng việc này với đúng bệnh nhân này không? | BS Minh xem bệnh nhân phòng khác |
| 3. Database | Dữ liệu này có thuộc đúng tenant/cơ sở không? | Lập trình viên viết code bị lỗi |

### 8.2 Vì sao cần lớp 3 khi đã có lớp 2?

Vì **con người viết code thì có lúc quên**. Hãy xem ví dụ:

```csharp
// Lập trình viên A viết đúng:
db.Invoices.Where(i => i.TenantId == currentTenant && i.Status == "paid");

// Lập trình viên B, 6 tháng sau, viết báo cáo mới và QUÊN điều kiện tenant:
db.Invoices.Where(i => i.Status == "paid");
// → Nếu không có lớp 3: trả về hóa đơn của TẤT CẢ các chuỗi phòng khám!
```

**Lớp 3 dùng tính năng Row-Level Security (RLS) của PostgreSQL.** Hiểu đơn giản: ta dặn database một lần *"mỗi khi ai đọc bảng hóa đơn, tự động chỉ trả về dòng của tenant hiện tại"*. Từ đó, dù code có quên điều kiện, database vẫn tự thêm vào.

```sql
-- Dặn database MỘT LẦN khi tạo bảng
ALTER TABLE invoices ENABLE ROW LEVEL SECURITY;

CREATE POLICY chi_xem_tenant_cua_minh ON invoices
  USING (tenant_id = current_setting('app.tenant_id')::uuid);
```

```csharp
// Mỗi request, service đặt "tenant hiện tại" một lần ở đầu
await db.Execute("SET LOCAL app.tenant_id = @t", currentTenant);

// Sau đó, dù lập trình viên B quên điều kiện:
db.Invoices.Where(i => i.Status == "paid");
// → Database TỰ ĐỘNG chỉ trả hóa đơn của tenant hiện tại ✅
```

---

## Phần 9. Cache — vì sao cần và dùng thế nào

### 9.1 Vì sao cần?

Mỗi lần bấm nút đều phải hỏi quyền. Hệ thống 1 triệu người dùng có thể có **hàng nghìn câu hỏi mỗi giây**. Nếu lần nào cũng tra database thì database quá tải.

**Cache (Redis) giống một cuốn sổ tay:** lần đầu tra database, ghi kết quả vào sổ tay. Lần sau tra sổ tay, nhanh hơn khoảng 50–100 lần.

### 9.2 Vấn đề: sổ tay bị cũ

Nếu Anh Nam vừa thu hồi quyền của ai đó, mà sổ tay vẫn ghi "có quyền" thì sao? **Giải pháp: đánh số phiên bản.**

```mermaid
sequenceDiagram
    participant AZ as Dịch vụ phân quyền
    participant C as Redis

    Note over C: Phiên bản quyền của Lan = 7<br/>Sổ tay: authz:lan:v7 = Điều dưỡng
    AZ->>C: Tra quyền Lan - phiên bản hiện tại?
    C-->>AZ: v7 - có trong sổ tay - dùng luôn ⚡

    Note over C: Anh Nam gán thêm quyền Dược sĩ<br/>→ phiên bản tăng lên 8
    AZ->>C: Tra quyền Lan - phiên bản hiện tại?
    C-->>AZ: v8 - sổ tay chưa có trang v8
    AZ->>AZ: Tra lại database
    AZ->>C: Ghi sổ tay: authz:lan:v8 = Điều dưỡng + Dược sĩ
```

Thay vì phải đi tìm và xóa từng dòng cũ, ta chỉ cần **tăng số phiên bản**. Mọi dòng cũ (`v7`) tự nhiên không còn ai đọc nữa, và sẽ tự hết hạn sau vài phút.

### 9.3 Hai loại sổ tay khác nhau

| | Sổ tay Lớp 1 (role, permission) | Sổ tay Lớp 2 (lịch trực hiện tại) |
|---|---|---|
| Thay đổi | Hiếm | Mỗi ca |
| Giữ trong bao lâu | Đến khi phiên bản đổi (tối đa vài phút) | 1–5 phút |
| Lý do tách riêng | Nếu gộp chung, mỗi lần đổi ca sẽ làm mất luôn sổ tay Lớp 1 |

---

## Phần 10. Một ngày làm việc: tổng hợp các tình huống

Thứ Hai 09/11/2026, Cơ sở Cầu Giấy.

| Giờ | Ai | Muốn làm gì | Kết quả | Vì sao |
|---|---|---|---|---|
| 07:00 | Chị Hoa | Tiếp nhận Bà Mai, xếp vào hàng chờ Nội 3 | ✅ | Có `patient.create`, `queue.add` tại Quầy tiếp đón |
| 07:05 | Chị Hoa | Xem chẩn đoán lần khám trước của Bà Mai | ❌ | Tiếp đón không có `patient.clinical.read` |
| 09:00 | BS Minh | Mở hàng chờ | ✅ Thấy 20 người ở Nội 3 | Lớp 1 + lịch trực Nội 3 |
| 09:00 | BS Minh | Xem bệnh án Bà Mai | ✅ | Bà Mai chờ ở phòng BS Minh đang trực |
| 09:10 | BS Minh | Kê đơn cho Bà Mai | ✅ | Có `prescription.create` + đang khám Bà Mai |
| 09:15 | Hộ lý Hùng | Xem nhiệm vụ đưa Bà Mai đi xét nghiệm | ✅ | Có `task.view`, chỉ thấy tên và nơi cần đến |
| 09:16 | Hộ lý Hùng | Mở chẩn đoán của Bà Mai | ❌ | Luật cấm cứng: hộ lý không bao giờ xem dữ liệu lâm sàng |
| 09:30 | Anh Tú | Cấp phát thuốc theo đơn của BS Minh | ✅ | Có `prescription.dispense` tại Kho dược |
| 10:00 | Anh Tú | Điều chỉnh tồn kho do thuốc vỡ | ✅ | Có `inventory.adjust` |
| 10:01 | Anh Tú | Tự duyệt phiếu điều chỉnh vừa tạo | ❌ | Quy tắc tách nhiệm vụ: không được tự duyệt phiếu của mình |
| 10:30 | Anh Nam | Duyệt phiếu điều chỉnh của Anh Tú | ✅ | Có `inventory.adjust.approve`, không phải người tạo phiếu |
| 14:00 | BS Minh | Xem bệnh nhân mới ở Nội 3 | ❌ | Đã chuyển sang trực Nội 5 |
| 14:05 | BS Minh | Xem kết quả xét nghiệm của Bà Mai | ✅ | Đã khám Bà Mai sáng nay |
| 15:00 | BS Minh | Xem hồ sơ bệnh nhân ngất ở hành lang | ✅ qua khẩn cấp | Break-glass, Anh Nam nhận cảnh báo |
| 16:00 | Chị Thu | Xem báo cáo doanh thu Cầu Giấy và Hà Đông | ✅ | Gán ở Miền Bắc, kế thừa xuống |
| 16:05 | Chị Thu | Xem báo cáo cơ sở Quận 1 | ❌ | Quận 1 thuộc Miền Nam |
| 16:10 | Anh Nam | Xem báo cáo cơ sở Hà Đông | ❌ | Chỉ được gán ở Cầu Giấy |

---

## Phần 11. Bảng thuật ngữ

| Thuật ngữ | Viết tắt | Giải thích dễ hiểu |
|---|---|---|
| Authentication | AuthN | **Xác thực** — chứng minh "bạn là ai" (đăng nhập) |
| Authorization | AuthZ | **Phân quyền** — quyết định "bạn được làm gì" |
| Identity Provider | IdP | Hệ thống chuyên lo đăng nhập, ví dụ Keycloak, Microsoft Entra ID |
| OpenID Connect | OIDC | Chuẩn giao tiếp đăng nhập giữa ứng dụng và IdP |
| Multi-Factor Authentication | MFA | Đăng nhập nhiều bước: mật khẩu + mã OTP |
| Token / JWT | — | "Thẻ nhân viên điện tử" do IdP cấp, gửi kèm mỗi yêu cầu |
| Tenant | — | Một khách hàng dùng phần mềm; dữ liệu giữa các tenant tách biệt hoàn toàn |
| Multi-tenant | — | Một phần mềm phục vụ nhiều khách hàng cùng lúc |
| Facility | — | Cơ sở vật lý |
| Unit | — | Đơn vị bên trong cơ sở: phòng khám, quầy, kho |
| Permission | — | Một quyền cụ thể — một "chiếc chìa khóa" |
| Role | — | Một nhóm permission — một "chùm chìa khóa" |
| Role Assignment | — | Giao role cho người, ở phạm vi nào, đến khi nào |
| Scope | — | Phạm vi áp dụng quyền: tenant, vùng, cơ sở, hay phòng |
| RBAC | Role-Based Access Control | Phân quyền dựa trên **vai trò** |
| ABAC | Attribute-Based Access Control | Phân quyền dựa trên **thuộc tính/ngữ cảnh**, ví dụ giờ, ca trực, phòng |
| ReBAC | Relationship-Based Access Control | Phân quyền dựa trên **mối quan hệ**, ví dụ "bác sĩ đang điều trị bệnh nhân này" |
| Policy | — | Một quy tắc phân quyền viết thành văn bản/code |
| Policy Engine | — | Phần mềm đọc và thực thi policy, ví dụ Cedar, OPA, OpenFGA |
| PDP | Policy Decision Point | **Nơi ra quyết định** — ở đây là Dịch vụ phân quyền |
| PEP | Policy Enforcement Point | **Nơi thực thi quyết định** — Cổng API, service, database |
| Duty Roster | — | Lịch phân công trực |
| Encounter | — | Một lượt khám bệnh |
| Care Team | — | Nhóm nhân viên y tế đang điều trị cho một bệnh nhân |
| Row-Level Security | RLS | Tính năng database tự lọc dòng dữ liệu theo quy tắc |
| Cache | — | Bộ nhớ tạm để tra cứu nhanh — "cuốn sổ tay" |
| Redis | — | Phần mềm cache phổ biến |
| TTL | Time To Live | Thời gian một dòng trong cache tồn tại trước khi tự xóa |
| Epoch / Version | — | Số phiên bản quyền, tăng lên mỗi khi quyền thay đổi |
| Outbox | — | Bảng "hộp thư sự kiện" ghi lại các thay đổi để xử lý sau |
| Break-glass | — | Truy cập khẩn cấp, được phép nhưng bị ghi lại và cảnh báo |
| Separation of Duties | SoD | Tách nhiệm vụ: người tạo không được tự duyệt |
| Field masking | — | Che bớt một số trường dữ liệu tùy người xem |
| Least Privilege | — | Nguyên tắc: chỉ cấp **vừa đủ** quyền cần thiết cho công việc |
| Zero Trust | — | Nguyên tắc: không tin mặc định, kiểm tra mọi yêu cầu |
| Deny by default | — | Mặc định từ chối; chỉ cho phép khi có quy tắc cho phép rõ ràng |
| Audit Log | — | Nhật ký ghi lại ai đã làm gì, lúc nào |
| SIEM | — | Hệ thống giám sát an ninh, phân tích nhật ký để phát hiện bất thường |
| 401 Unauthorized | — | Mã lỗi HTTP: "bạn chưa đăng nhập / token không hợp lệ" |
| 403 Forbidden | — | Mã lỗi HTTP: "bạn đã đăng nhập nhưng không có quyền" |

---

## Phần 12. Câu hỏi thường gặp

**Hỏi: Tạo role mới có phải sửa code không?**
Không. Admin tạo role trên màn hình và chọn các permission có sẵn. Chỉ khi có **tính năng mới** cần permission mới thì mới sửa code.

**Hỏi: Sao không lưu quyền theo đường dẫn API (ví dụ `GET /api/invoices`)?**
Vì đường dẫn là chi tiết kỹ thuật, hay thay đổi khi sửa code, và không trả lời được câu hỏi "xem bệnh nhân **nào**". Một hành động như "duyệt hoàn tiền" có thể đi qua nhiều đường dẫn khác nhau trên web và mobile. Permission đặt theo nghiệp vụ thì ổn định hơn và admin dễ hiểu hơn.

**Hỏi: Nếu lịch trực nhập sai thì bác sĩ bị chặn à?**
Đúng, đây là rủi ro của Lớp 2. Cách giảm thiểu:
- cho phép **ân hạn** 15 phút trước ca và 30 phút sau ca;
- bác sĩ **tự xác nhận phòng** khi bắt đầu ca nếu lịch chưa có, kèm thông báo cho điều phối;
- trường hợp gấp vẫn có **truy cập khẩn cấp**.

**Hỏi: Một bác sĩ làm ở 2 cơ sở thì sao?**
Tạo 2 dòng `role_assignments`, mỗi dòng một cơ sở. Khi đăng nhập, bác sĩ chọn cơ sở làm việc hôm nay.

**Hỏi: Thêm một chuỗi phòng khám mới (tenant mới) thì làm gì?**
Tạo tenant mới, các role chung (Bác sĩ, Y tá...) đã có sẵn. Tenant mới có thể tự tạo thêm role riêng từ danh sách permission có sẵn. Database tự tách dữ liệu nhờ RLS.

**Hỏi: Vì sao phải ghi nhật ký cả khi chỉ xem?**
Dữ liệu sức khỏe là dữ liệu nhạy cảm theo pháp luật về bảo vệ dữ liệu cá nhân. Bệnh nhân có quyền biết ai đã xem hồ sơ của mình, và khi xảy ra rò rỉ, nhật ký là bằng chứng duy nhất để điều tra.

**Hỏi: Tôi cần học gì để triển khai được?**
Theo thứ tự: (1) cách framework của bạn làm authorization, ví dụ ASP.NET Core Authorization Policy; (2) JWT và OIDC cơ bản; (3) Redis cơ bản; (4) PostgreSQL Row-Level Security; (5) outbox pattern.

---

## Phần 13. Gợi ý thứ tự triển khai

Không cần làm tất cả cùng lúc. Mỗi giai đoạn đều tạo ra một hệ thống **chạy được**.

```mermaid
flowchart TD
    G1["GIAI ĐOẠN 1 - Nền móng<br/>Đăng nhập + OTP<br/>Bảng Tenant, Facility, Unit<br/>Permission, Role, Role Assignment<br/>API chỉ hỏi permission<br/>RLS theo tenant<br/>Ghi nhật ký"]
    G2["GIAI ĐOẠN 2 - Tách dịch vụ<br/>Dịch vụ phân quyền riêng<br/>Cache Redis có phiên bản<br/>Outbox<br/>Thu hồi quyền tức thì"]
    G3["GIAI ĐOẠN 3 - Ngữ cảnh y tế<br/>Lịch trực - Lớp 2<br/>Hàng đợi theo phòng<br/>Che bớt dữ liệu theo người xem<br/>Truy cập khẩn cấp<br/>Tách nhiệm vụ ở kho"]
    G4["GIAI ĐOẠN 4 - Mở rộng<br/>Role riêng cho từng tenant<br/>Màn hình quản trị đầy đủ<br/>Giám sát bất thường"]

    G1 --> G2 --> G3 --> G4
```

**Ba thứ cần làm đúng ngay từ đầu** (sửa về sau rất tốn kém):

1. **API chỉ hỏi permission, không hỏi tên role.**
2. **Role được gán theo phạm vi** (bảng `role_assignments` có `scope_type`, `scope_id`).
3. **Database tự tách dữ liệu theo tenant** (RLS).

Những thứ khác như cache, dịch vụ phân quyền riêng, lịch trực đều có thể thêm dần mà không phải viết lại API.

---

*Hết tài liệu.*
