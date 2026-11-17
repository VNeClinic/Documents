# openEHR & EHRbase cho nền tảng 3.000 phòng khám — Phân tích và khuyến nghị

Nov 17, 2026 · @toanvu

## Tóm tắt và khuyến nghị

Nên giữ openEHR + EHRbase, nhưng chỉ làm **lớp hồ sơ lâm sàng chuẩn**; mọi nghiệp vụ vận hành phòng khám vẫn chạy trên PostgreSQL của nền tảng. Ở quy mô 3.000 phòng khám, rủi ro lớn nhất không phải tốc độ ghi mà là số lượng template, dung lượng tích lũy và cô lập tenant.

| Câu hỏi | Trả lời ngắn | Điều kiện / việc phải làm |
| --- | --- | --- |
| 1. Xây xoay quanh openEHR template + EHRbase? | **Có, có điều kiện.** Giữ mô hình lai của ADR-0003: bản nháp ở DB vận hành, kết thúc khám mới commit vào EHRbase. | Chuyển từ "mỗi tenant một bộ OPT" sang **thư viện template lõi dùng chung + cấu hình form theo tenant**. Benchmark ở quy mô mục tiêu trước khi chốt ADR-0003. |
| 2. Có lựa chọn tốt hơn EHRbase? | **Mã nguồn mở: không.** EHRbase gần như là CDR openEHR mã nguồn mở duy nhất còn phát triển tích cực. **Thương mại:** Better trưởng thành nhất; HIP EHRbase của vitagroup là bản doanh nghiệp của chính EHRbase. | Chỉ gọi CDR qua openEHR REST + FLAT để giữ đường chuyển sang HIP hoặc Better. Hướng FHIR (Medplum, Aidbox) chỉ hợp lý nếu bỏ yêu cầu template openEHR. |
| 3. Triển khai thế nào cho 3.000 phòng khám? | **Kiến trúc cell:** 8–10 cell, mỗi cell 300–400 phòng khám, có EHRbase + PostgreSQL riêng. Một control plane chung quản lý tenant, template và định tuyến. | Read model cho màn hình khám; kho phân tích tách khỏi CDR; hạ tầng đặt tại Việt Nam; mở cell dần theo lộ trình 1 → 3 → 10. |

Con số định hướng: 300.000 lượt khám/ngày, khoảng 100 triệu lượt/năm, gần gấp 4 lần mục tiêu hiện tại của kiến trúc (80.000 lượt/ngày). Tải ghi lúc cao điểm chỉ khoảng 15–50 composition/giây cho toàn hệ thống. Điều quyết định kiến trúc là khối tích lũy 0,5–1 tỷ composition sau 5 năm.

## Bối cảnh, giả định và bài học từ các hệ thống lớn

Bài toán là SaaS ngoại trú dùng chung cho 3.000 phòng khám, không phải bệnh án bệnh viện. Lượt khám ngắn, số lượng lớn, dữ liệu mỗi lượt nhỏ; phần lớn nghiệp vụ là vận hành (tiếp đón, thu ngân, kho, BHYT, đơn thuốc điện tử).

### Giả định quy mô

| Chỉ số | Giá trị | Cách tính / giả định |
| --- | --- | --- |
| Lượt khám/ngày | 300.000 | 3.000 phòng khám × 100 lượt |
| Lượt khám/năm | \~100 triệu | 330 ngày hoạt động/năm |
| Composition/lượt khám | 1–2 | 1 composition theo template khi kết thúc khám; thêm khi amend hoặc kết quả CLS về muộn (ADR-0011) |
| Composition/năm | 100–200 triệu | 0,5–1 tỷ sau 5 năm |
| Tải ghi cao điểm | 15–50 composition/giây | giờ cao điểm \~15% lượt trong ngày, hệ số đột biến ×2 |
| Người dùng đồng thời | 15.000–25.000 | 5–8 nhân viên trực tuyến mỗi phòng khám |
| Số template (OPT) | 15.000–30.000+ | nếu mỗi tenant có 5–10 mẫu riêng, chưa tính version |

So với mục tiêu hiện tại trong `docs/architecture/00-tong-quan.md` (1.000 khách hàng, 80.000 lượt/ngày), yêu cầu mới lớn gần gấp 4 lần. Nếu "3.000 phòng khám" thuộc ít tenant hơn (chuỗi nhiều cơ sở), số template giảm mạnh nhưng tải và dung lượng giữ nguyên.

### Các hệ thống lớn đang làm gì

| Hệ thống | Quy mô | Lớp dữ liệu lâm sàng | Bài học cho dự án |
| --- | --- | --- | --- |
| [athenaOne](https://casrai.org/guides/athenahealth) (Mỹ, ngoại trú) | Một instance dùng chung cho mọi nhóm hành nghề | Mô hình riêng; dữ liệu phân vùng logic theo `practiceid` | SaaS ngoại trú lớn chạy một nền tảng chung, cô lập theo phòng khám |
| [EMIS](https://www.enliviohealth.com/news-insights/emis-x-now-available-for-all-gp-practices-in-england) (Anh, bác sĩ gia đình) | 3.745 phòng khám, 232.000 người dùng trên nền tảng cloud EMIS-X (5/2024) | Mô hình riêng, không dùng openEHR | Quy mô tương đương dự án, vận hành tập trung |
| [CDR Catalonia](https://discourse.openehr.org/t/region-of-catalonia-award-of-the-tender-for-the-service-of-cdr-platform/3910) (Tây Ban Nha, vùng) | 7–8 triệu dân; chạy thật từ 30/11/2023 | CDR openEHR; [theo Nirmitee](https://nirmitee.io/blog/openehr-cdr-vendor-comparison-2026/) là HIP EHRbase | Kho giao dịch chỉ nhận AQL đã lưu sẵn; AQL tự do chạy trên bản sao. Mục tiêu p95: 40 ms truy vấn một EHR, 60 ms ghi |
| [OneLondon Universal Care Plan](https://nirmitee.io/blog/openehr-cdr-vendor-comparison-2026/) (Anh, vùng) | 10 triệu người, 5 ICS | CDR openEHR của Better | openEHR mạnh nhất ở hồ sơ dùng chung nhiều cơ sở |
| [DIPS Arena](https://nirmitee.io/blog/openehr-cdr-vendor-comparison-2026/) (Na Uy, bệnh viện) | Hơn 85% bệnh viện Na Uy | CDR openEHR nhúng trong sản phẩm | openEHR đã chạy ổn ở quy mô quốc gia khi có đội mô hình hóa mạnh |

Hệ thống phòng khám lớn nhất (athenaOne, EMIS) không dùng openEHR làm kho vận hành. openEHR thành công ở hồ sơ lâm sàng dùng chung và bệnh viện, luôn tách khỏi lớp vận hành. Tôi không tìm thấy tiền lệ công khai nào về SaaS ngoại trú cỡ 3.000 phòng khám chạy trên một CDR openEHR, nên benchmark riêng là bắt buộc.

## Câu hỏi 1 — Có nên xây dựng xoay quanh openEHR template + EHRbase?

**Có**, nếu openEHR là lớp hồ sơ lâm sàng chuẩn và template được quản trị tập trung. **Không**, nếu định dùng nó làm nền cho toàn bộ ứng dụng, hoặc để mỗi phòng khám tự tạo OPT tuỳ ý. Kiến trúc lai của ADR-0003 đã đúng hướng; cần đổi chiến lược template và kế hoạch mở rộng.

### Lợi ích thật ở quy mô 3.000 phòng khám

- **Thêm biểu mẫu không cần migration hay deploy.** Template mới chạy ngay ở chế độ dynamic (ADR-0004).
- **Truy vấn xuyên template.** Các template cùng dùng một archetype (ví dụ `OBSERVATION.blood_pressure.v2`), nên một câu AQL đọc được huyết áp từ mọi biểu mẫu. Đây là lợi thế lớn nhất so với form tự do hoặc FHIR Questionnaire.
- **Hồ sơ có version và vết sửa.** Mỗi lần amend là một version mới, hợp với yêu cầu lưu trữ, truy xuất, phục hồi của [Thông tư 13/2025/TT-BYT](https://luatvietnam.vn/y-te/thong-tu-13-2025-tt-byt-huong-dan-trien-khai-ho-so-benh-an-dien-tu-tu-bo-y-te-402396-d1.html).
- **Có đường thoát nhà cung cấp.** openEHR REST + FLAT dùng chung giữa EHRbase và Better.
- **Liên thông FHIR có công cụ mở.** [openFHIR](https://github.com/openFHIR/openfhir) (Apache 2.0) ánh xạ hai chiều openEHR ↔ FHIR theo đặc tả FHIR Connect; bản mở chưa dành cho production, bản Enterprise có xác thực và tối ưu hiệu năng.

### Rủi ro phải chấp nhận

| Rủi ro | Mức | Vì sao | Giảm thiểu |
| --- | --- | --- | --- |
| EHRbase mã nguồn mở không có multi-tenancy | Cao | [Multi-tenancy là tính năng doanh nghiệp](https://docs.ehrbase.org/docs/EHRbase/Enterprise-Features/Overview) của HIP EHRbase. Trên bản mở, [AQL trả dữ liệu mọi tenant](https://discourse.openehr.org/t/tenant-feature-in-ehrbase-version-23/3756) | Không mở EHRbase ra ngoài; backend kiểm tra quyền sở hữu `ehr_id` trước mọi lời gọi; chỉ dùng AQL trong catalog đã duyệt; chia cell |
| Bùng nổ template | Cao | 15.000–30.000 OPT nếu mỗi tenant có bộ riêng; EHRbase nạp cache template khi cần, [không nạp sẵn theo mặc định](https://github.com/ehrbase/ehrbase/blob/develop/CHANGELOG.md) | Thư viện template lõi + cấu hình form theo tenant (bên dưới) |
| AQL quần thể chậm khi dữ liệu lớn | Trung bình–cao | [Chưa có benchmark chính thức](https://discourse.openehr.org/t/aql-performance-benchmarks-in-ehrbase/1588); một truy vấn cohort một ngày trên hệ thống 150 GB (v2.5) [mất khoảng 20 phút](https://discourse.openehr.org/t/ehrbase-performance-issue/11953) | Read model cho màn hình khám; kho phân tích riêng; AQL tự do chỉ chạy trên bản sao |
| Thiếu nhân lực openEHR tại Việt Nam | Cao | Cần người mô hình hóa archetype/template, AQL, vận hành CDR Java | Đội mô hình lâm sàng 2–3 người; hợp đồng hỗ trợ nhà cung cấp |
| Hai nguồn dữ liệu (DB vận hành + CDR) | Trung bình | Phải đối soát commit, retry, amend | Outbox/inbox (ADR-0008) và màn hình Đồng bộ EHRbase đã có |
| CDR Java trong đội .NET | Trung bình | EHRbase là Spring Boot; hệ .NET hầu như không có thư viện openEHR được duy trì (ước đoán, chưa kiểm chứng) | Vận hành EHRbase như hộp đen trong container; giữ FlatBuilder/WebTemplateIndex tự viết |
| Bản dịch và bảng mã | Trung bình | Archetype CKM chủ yếu tiếng Anh; ICD-10 bản Việt do Bộ Y tế ban hành | Dịch archetype dùng chung một lần; bảng mã quản lý ở module Catalog |

### Điều kiện để đi tiếp

1. **Kiểm soát archetype, hạn chế OPT riêng.** Archetype là đơn vị ngữ nghĩa và chỉ lấy từ thư viện đã duyệt. Mỗi chuyên khoa có một bộ template lõi dùng chung (`core.{key}.v{n}`).
2. **Tenant tuỳ biến bằng form profile, không bằng OPT.** Profile chỉnh ẩn/hiện, thứ tự, nhãn, giá trị mặc định, danh sách chọn, bắt buộc thêm. Profile chỉ được chặt hơn template, nên dữ liệu vẫn hợp lệ khi commit theo template lõi.
3. **OPT riêng `{tenant}.{key}.v{n}` chỉ cho tenant có nhu cầu thật** (chuỗi lớn, chuyên khoa đặc thù), qua quy trình duyệt hiện có.
4. **Cô lập tenant trong CDR do nền tảng đảm bảo**: theo cell, quyền sở hữu `ehr_id` và catalog AQL; hoặc mua plugin multi-tenancy của HIP.
5. **Chỉ dùng openEHR REST chuẩn**, không dùng API riêng của EHRbase, để có thể thay bằng HIP hoặc Better.
6. **Benchmark một cell ở quy mô thật** trước khi chốt ADR-0003 (chi tiết ở mục Rủi ro và lộ trình).

Chỉ nên bỏ openEHR nếu phần lớn khách hàng chỉ cần biểu mẫu đơn giản, không cần truy vấn lâm sàng xuyên tenant, và dự án không duy trì được đội mô hình hóa. Điểm 1–3 thay đổi ADR-0003 và ADR-0004, nên cần một ADR mới.

## Câu hỏi 2 — Ngoài EHRbase còn lựa chọn nào tốt hơn?

Với yêu cầu "template openEHR theo tenant", không có CDR mã nguồn mở nào tốt hơn EHRbase. Lựa chọn đáng cân nhắc là **mua bản doanh nghiệp hoặc hỗ trợ của chính EHRbase** (HIP, vitagroup), và dùng **Better** làm phương án so sánh. Không nhà cung cấp thương mại nào công bố bảng giá, nên phải so tổng chi phí 5 năm qua báo giá.

### CDR openEHR

| Sản phẩm | Giấy phép | Tham chiếu công khai | Nhận xét cho dự án |
| --- | --- | --- | --- |
| [EHRbase](https://github.com/ehrbase/ehrbase) | Apache 2.0, miễn phí | Phát triển trong các bệnh viện đại học Đức (HiGHmed); bản 2.38.0 | **Lựa chọn hiện tại.** Đủ dùng nếu nền tảng tự cô lập tenant. Không có multi-tenancy, không có SLA |
| [HIP EHRbase](https://docs.ehrbase.org/docs/EHRbase/Enterprise-Features/Overview) (vitagroup, Đức) | Thương mại, có SLA | CDR Catalonia, 7–8 triệu dân | Cùng API và dữ liệu với EHRbase. Thêm multi-tenancy (schema riêng mỗi tenant), YugabyteDB, event trigger, audit ATNA, gộp EHR. Nâng cấp ít rủi ro nhất |
| [Better Platform](https://nirmitee.io/blog/openehr-cdr-vendor-comparison-2026/) (Slovenia) | Thương mại | OneLondon 10 triệu người; chương trình quốc gia Slovenia, Hy Lạp, Ireland; hãng nêu hơn 40 triệu bệnh nhân | Trưởng thành nhất, có Studio low-code và kho FHIR. Chi phí cao; form/ứng dụng làm bằng Studio không mang đi được |
| [Ocean Health Systems](https://nirmitee.io/blog/openehr-cdr-vendor-comparison-2026/) (Úc) | Thương mại | Multiprac tại hơn 130 cơ sở bệnh viện công Queensland | Mạnh về mô hình hóa; ít tài liệu kỹ thuật công khai |
| [DIPS EHR Store](https://nirmitee.io/blog/openehr-cdr-vendor-comparison-2026/) (Na Uy) | Chỉ bán kèm DIPS Arena | Hơn 85% bệnh viện Na Uy | Không mua riêng được |
| [Atomik](https://nirmitee.io/blog/openehr-cdr-vendor-comparison-2026/) (CaboLabs, Uruguay) | Thương mại | Chưa có tham chiếu công khai | Nhỏ; EHRServer mã nguồn mở cùng hãng ngừng cập nhật từ 3/2023 |
| [Cadasto / CODE24](https://nirmitee.io/blog/openehr-cdr-vendor-comparison-2026/) (Hà Lan) | Thương mại | Ra mắt đầu 2026 | Quá mới để đặt cược |
| [ArcEHR](https://nirmitee.io/blog/openehr-cdr-vendor-comparison-2026/) (Slovenia) | Beta đóng, dự kiến Apache 2.0 | Chưa có khách hàng công khai | Chỉ nên theo dõi |

### Hướng FHIR (chỉ khi bỏ yêu cầu template openEHR)

| Sản phẩm | Giấy phép | Điểm mạnh | Điểm yếu với dự án |
| --- | --- | --- | --- |
| [Medplum](https://www.medplum.com/open-source) | Phần lớn Apache 2.0; một số dịch vụ cần hợp đồng | Headless EHR trên PostgreSQL; biểu mẫu bằng Questionnaire/SDC; cô lập tenant theo Project ở tầng ứng dụng | Không có archetype: dữ liệu form khó truy vấn xuyên mẫu nếu không gắn mã cho từng câu hỏi |
| [Aidbox](https://www.health-samurai.io/docs/aidbox/overview/licensing-and-support) (Health Samurai) | Thương mại, theo báo giá | PostgreSQL; bản Multibox cho multi-tenant | Phí bản quyền, khóa nhà cung cấp |
| [HAPI FHIR JPA](https://hapifhir.io/hapi-fhir/docs/server_jpa_partitioning/partitioning.html) | Mã nguồn mở (Java) | Phổ biến nhất; phân vùng (partitioning) cho multi-tenant | Vẫn là Java; tự xây toàn bộ lớp biểu mẫu |
| [Microsoft FHIR Server](https://github.com/microsoft/fhir-server) | MIT, .NET | Cùng stack .NET; lưu trên SQL Server | Thiết kế hướng Azure; Cosmos DB hết hỗ trợ 30/9/2026 |

### Kết luận

- **Giai đoạn 1–2 (≤ 1.000 phòng khám):** giữ EHRbase mã nguồn mở, cô lập tenant ở tầng nền tảng.
- **Trong giai đoạn 2, trước Cổng 2:** xin báo giá HIP EHRbase (hỗ trợ + multi-tenancy) và Better cho cùng kịch bản 3.000 phòng khám, so tổng chi phí 5 năm với chi phí tự vận hành.
- **FHIR là lớp liên thông, không phải kho chính:** xuất FHIR từ composition bằng openFHIR khi cần kết nối hệ thống ngoài; bản mở cần tự bổ sung xác thực hoặc mua bản Enterprise.

## Câu hỏi 3 — Phương án triển khai cho 3.000 phòng khám

Phương án tốt nhất là **kiến trúc cell**: 8–10 cell giống hệt nhau, mỗi cell là một bản đầy đủ của nền tảng phục vụ 300–400 phòng khám. Một control plane mỏng quản lý tenant, template và định tuyến. Tốc độ ghi không phải vấn đề; cell giải quyết dung lượng, cô lập sự cố, số template mỗi CDR và nâng cấp an toàn.

### Dung lượng là ràng buộc chính

&#91;embedded content: ước tính từ giả định trong tài liệu · 3.000 phòng khám, 5 năm vận hành\]

Tải ghi chỉ 15–50 composition/giây, nhưng khối dữ liệu tăng tuyến tính và không giảm vì hồ sơ phải lưu lâu dài. Chia cell giữ mỗi PostgreSQL ở mức dễ sao lưu, khôi phục và nâng cấp.

### So sánh bốn phương án

| Phương án | Mô tả | Ưu điểm | Nhược điểm | Kết luận |
| --- | --- | --- | --- | --- |
| A. Một EHRbase dùng chung | Một cụm EHRbase + một PostgreSQL cho mọi tenant pooled (ADR-0003 hiện tại) | Đơn giản nhất | 10–20 TB trong một DB; một sự cố dừng 3.000 phòng khám; mọi OPT dồn vào một cache | Chỉ cho giai đoạn đầu, ≤ 400 phòng khám |
| B. EHRbase riêng mỗi tenant | Silo hoàn toàn | Cô lập tuyệt đối | 3.000 cụm EHRbase + DB: chi phí và vận hành không khả thi | Chỉ cho vài chuỗi lớn trả phí silo |
| C. Cell | 8–10 cell, mỗi cell 300–400 phòng khám | Mỗi CDR 1–2 TB sau 5 năm; sự cố chỉ ảnh hưởng một cell; thêm năng lực bằng cách thêm cell | Cần control plane, định tuyến, công cụ chuyển tenant giữa cell | **Chọn** |
| D. Một CDR phân tán | HIP EHRbase trên YugabyteDB, mở rộng ngang | Một hệ thống logic | Phụ thuộc bản thương mại; vận hành DB phân tán khó | Dự phòng nếu cell không đủ |

### Kiến trúc cell

&#91;embedded content: kiến trúc cell đề xuất · 1 control plane, 8–10 cell\]

Người dùng đi qua cổng API tới đúng cell của tenant; control plane chỉ phân phối template, form profile và cấu hình. Dữ liệu lâm sàng không rời cell, trừ bản đã khử định danh gửi sang kho phân tích.

1. **Tenant thuộc đúng một cell.** Bệnh nhân và EHR thuộc tenant, nên không có truy vấn chéo cell. Chuỗi lớn có thể có cell riêng (silo theo ADR-0002).
2. **Cell không gọi nhau.** Control plane chỉ giữ cấu hình, không giữ dữ liệu lâm sàng.
3. **Cell có kích thước cố định**, khoảng 400 phòng khám hoặc 40.000 lượt/ngày. Cell đầy thì mở cell mới, không phình cell cũ.
4. **Nâng cấp EHRbase kiểu canary.** EHRbase 2.0 từng đổi toàn bộ cấu trúc dữ liệu và [cần công cụ migration riêng](https://github.com/ehrbase/ehrbase/blob/develop/UPDATING.md); một cell nâng trước, các cell khác theo sau.
5. **Có công cụ chuyển tenant giữa cell** (xuất/nhập EHR và composition qua openEHR REST), dùng khi cân bằng tải hoặc khi khách hàng chuyển sang silo.

### Quản trị template ở quy mô lớn

| Tầng | Ai sở hữu | Triển khai ở đâu | Số lượng (ước tính) |
| --- | --- | --- | --- |
| Archetype (CKM + archetype riêng cho Việt Nam) | Đội mô hình lâm sàng của nền tảng | Git + control plane | 100–200 |
| Template lõi theo chuyên khoa `core.{key}.v{n}` | Đội mô hình lâm sàng | Tải vào mọi cell | 50–150 |
| OPT riêng của tenant `{tenant}.{key}.v{n}` | Tenant lớn, đội mô hình duyệt | Chỉ tải vào cell của tenant | Vài trăm, có hạn mức theo gói |
| Form profile của tenant | Quản trị viên tenant tự cấu hình | DB, có version | Không giới hạn; không tạo OPT |

- Chế độ generated (codegen) chỉ dành cho template lõi; OPT riêng của tenant chạy dynamic (ADR-0004).
- Bật nạp sẵn cache cho template lõi (`ehrbase.cache.template-init-on-startup`) và đo bộ nhớ trong benchmark.
- Template đã có dữ liệu là bất biến; version mới là `template_id` mới, như quy trình hiện tại.

### Đường đọc dữ liệu

| Nhu cầu | Đọc từ | Lý do |
| --- | --- | --- |
| Màn hình khám: dị ứng, chẩn đoán, thuốc đang dùng, sinh hiệu gần nhất, dòng thời gian | Read model trong DB vận hành, cập nhật sau mỗi lần commit | Màn hình khám không phụ thuộc CDR lúc đông bệnh nhân |
| Xem chi tiết hồ sơ một bệnh nhân | AQL trong catalog, luôn có `ehr_id` | Catalonia đặt mục tiêu p95 40 ms cho loại truy vấn này |
| Truy vấn quần thể, nghiên cứu | Bản sao CDR hoặc kho phân tích | Cohort một ngày mất \~20 phút trên 150 GB |
| Báo cáo vận hành, BHYT, doanh thu | DB vận hành | Dữ liệu nghiệp vụ không nằm trong CDR |

### Hạ tầng tại Việt Nam

- **Dữ liệu bệnh nhân đặt trong nước, hai trung tâm dữ liệu** (Hà Nội, TP.HCM) để dự phòng thảm họa. [Luật Bảo vệ dữ liệu cá nhân 91/2025/QH15](https://vneconomy.vn/nhung-diem-moi-trong-luat-bao-ve-du-lieu-ca-nhan-tu-112026.htm) có hiệu lực từ 01/01/2026 và yêu cầu báo vi phạm trong 72 giờ. Nghĩa vụ lưu trữ trong nước theo Nghị định 53/2022 cần tư vấn pháp lý.
- **Sao lưu liên tục + PITR cho cả DB vận hành và CDR**, diễn tập khôi phục định kỳ. Thông tư 13/2025 yêu cầu lưu trữ dự phòng và khả năng phục hồi thông tin.
- **Kubernetes + PostgreSQL tự quản hoặc DBaaS của nhà cung cấp trong nước.** Điểm xuất phát cho benchmark mỗi cell: EHRbase 2–3 pod, mỗi pod 2–4 CPU và 4–16 GB, như [một hệ thống đang chạy](https://discourse.openehr.org/t/ehrbase-performance-issue/11953); PostgreSQL CDR có primary + replica.

## Rủi ro, kiểm chứng và lộ trình

Chưa nên chốt ADR-0003 với điều kiện hiện tại (5 triệu composition). Cần benchmark một cell ở 100 triệu composition, tức dung lượng một cell sau 5 năm, trước khi mở rộng. Việc cần làm ngay: xác minh mốc 31/12/2026 của Thông tư 13/2025 có áp dụng cho phòng khám khách hàng hay không.

### Benchmark một cell (giai đoạn 0)

| Hạng mục | Mục tiêu đề xuất | Ghi chú |
| --- | --- | --- |
| Dữ liệu nạp sẵn | 100 triệu composition trên 5–10 template lõi thật | Bằng một cell sau 5 năm (ước tính) |
| Ghi | ≥ 30 composition/giây/cell, p95 < 500 ms | Đủ xả hàng đợi của 4 giờ EHRbase gián đoạn trong dưới 15 phút |
| Đọc một EHR (AQL trong catalog) | p95 < 300 ms như ADR-0003 | Catalonia đặt 40 ms |
| Số template | 2.000 OPT trong một cell | Đo RAM, thời gian khởi động, độ trễ khi cache miss |
| Kích thước | KB/composition kể cả index | Cập nhật lại biểu đồ dung lượng |
| Sao lưu/khôi phục | Khôi phục CDR 2 TB trong ≤ 4 giờ | RTO đề xuất, cần chốt với kinh doanh |
| Nâng cấp | Nâng một bản minor của EHRbase trên 100 triệu composition | Đo thời gian migration DB |

Công cụ có sẵn: lệnh `/spike-ehrbase` của repo cho phần chức năng; bộ [testehr/JMeter](https://discourse.openehr.org/t/aql-performance-benchmarks-in-ehrbase/1588) của cộng đồng cho phần tải.

### Lộ trình

&#91;embedded content: lộ trình đề xuất · 4 giai đoạn, 3 cổng; ngưỡng của cổng là đề xuất, cần chốt\]

Giai đoạn 0 trả lời câu hỏi lớn nhất: EHRbase có giữ được hiệu năng ở dung lượng một cell sau 5 năm hay không. Không đạt thì thu nhỏ cell hoặc chuyển sang CDR thương mại trước khi mở rộng.

### Rủi ro triển khai

| Rủi ro | Phương án dự phòng |
| --- | --- |
| EHRbase không đạt mục tiêu ở 100 triệu composition | Thu nhỏ cell (200 phòng khám); hoặc HIP EHRbase + YugabyteDB; hoặc Better |
| EHRbase đổi lớn như bản 2.0 | Khóa phiên bản, nâng canary từng cell, chỉ dùng openEHR REST chuẩn |
| Thiếu người mô hình hóa, hàng đợi duyệt template dài | Hạn mức OPT riêng theo gói; đẩy tuỳ biến sang form profile; thuê tư vấn openEHR |
| Chuyển tenant giữa cell phức tạp hơn dự kiến | Thiết kế từ giai đoạn 1, thử với tenant nội bộ trước khi cần thật |
| Phụ thuộc một nhà phát triển EHRbase (vitagroup) | Hợp đồng hỗ trợ; giữ đường chuyển sang Better qua chuẩn REST + FLAT |

## Nguồn và điểm cần xác minh

### Cần xác minh

- [ ] Mốc 31/12/2026 (Điều 4 khoản 2, Thông tư 13/2025/TT-BYT) có áp dụng cho phòng khám của khách hàng hay không.
- [ ] "3.000 phòng khám" tương ứng bao nhiêu tenant; con số này quyết định số OPT riêng.
- [ ] Kích thước thực của một composition (KB, kể cả index) với template lõi.
- [ ] Sản phẩm CDR của Catalonia: Nirmitee ghi HIP EHRbase, diễn đàn openEHR không xác nhận do điều khoản bảo mật.
- [ ] Báo giá HIP EHRbase và Better cho kịch bản 3.000 phòng khám trong 5 năm.
- [ ] Nghĩa vụ lưu trữ trong nước (Nghị định 53/2022) và Điều 26 Luật 91/2025 về dữ liệu sức khỏe: cần tư vấn pháp lý.

### Nguồn

Trong repo: `docs/adr/ADR-0002-multi-tenancy.md`, `docs/adr/ADR-0003-openehr-ehrbase.md`, `docs/architecture/00-tong-quan.md`, `docs/architecture/03-openehr-template-pipeline.md`.

- [EHRbase CHANGELOG](https://github.com/ehrbase/ehrbase/blob/develop/CHANGELOG.md) và [UPDATING.md](https://github.com/ehrbase/ehrbase/blob/develop/UPDATING.md)
- [EHRbase Docs — Enterprise Features](https://docs.ehrbase.org/docs/EHRbase/Enterprise-Features/Overview)
- openEHR Discourse: [multi-tenancy trên EHRbase (3/2023)](https://discourse.openehr.org/t/tenant-feature-in-ehrbase-version-23/3756), [benchmark AQL (6/2021)](https://discourse.openehr.org/t/aql-performance-benchmarks-in-ehrbase/1588), [hiệu năng EHRbase v2.5 (4/2026)](https://discourse.openehr.org/t/ehrbase-performance-issue/11953), [gói thầu CDR Catalonia (2023)](https://discourse.openehr.org/t/region-of-catalonia-award-of-the-tender-for-the-service-of-cdr-platform/3910)
- [Nirmitee — openEHR CDR vendor comparison 2026](https://nirmitee.io/blog/openehr-cdr-vendor-comparison-2026/) (blog của một công ty tư vấn, không phải nguồn độc lập)
- [Enlivio (EMIS) — EMIS-X, 14/5/2024](https://www.enliviohealth.com/news-insights/emis-x-now-available-for-all-gp-practices-in-england); [CASRAI — athenahealth](https://casrai.org/guides/athenahealth)
- [Medplum — Open source](https://www.medplum.com/open-source), [Medplum — Tenanting](https://www.medplum.com/solutions/compliance-security/tenanting-access)
- [Aidbox — Licensing](https://www.health-samurai.io/docs/aidbox/overview/licensing-and-support), [Aidbox — Multibox](https://www.health-samurai.io/docs/aidbox/tutorials/security-access-control-tutorials/run-multibox-locally.md)
- [HAPI FHIR — Partitioning](https://hapifhir.io/hapi-fhir/docs/server_jpa_partitioning/partitioning.html), [Microsoft FHIR Server](https://github.com/microsoft/fhir-server), [openFHIR](https://github.com/openFHIR/openfhir)
- [Thông tư 13/2025/TT-BYT (LuatVietnam)](https://luatvietnam.vn/y-te/thong-tu-13-2025-tt-byt-huong-dan-trien-khai-ho-so-benh-an-dien-tu-tu-bo-y-te-402396-d1.html), [Luật Bảo vệ dữ liệu cá nhân 2025 (VnEconomy)](https://vneconomy.vn/nhung-diem-moi-trong-luat-bao-ve-du-lieu-ca-nhan-tu-112026.htm)
