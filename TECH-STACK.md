# TLCN — Tech Stack chính thức

> Đây là **quy định kỹ thuật đã chốt** cho toàn bộ dự án `ute.fit.financemanagement` — mọi tài liệu/skill khác khi hướng dẫn code phải tuân theo đúng bảng này. Nếu phát hiện tài liệu khác ghi khác đi (ví dụ ví dụ code dùng sai công nghệ), coi đó là lỗi cần sửa lại cho khớp, không phải nguồn thay thế.
>
> Cập nhật: 2026-09-06.

| Phần | Công nghệ | Lý do |
|---|---|---|
| Backend | **Java + Spring Boot** | Mạnh, phổ biến, phù hợp OOP/REST API |
| Web | **React + TypeScript** | Dễ phát triển UI, ecosystem lớn |
| Mobile | **Flutter + Dart** | Một codebase cho Android/iOS |
| Database | **PostgreSQL** | Rất hợp với Spring Boot, mạnh và miễn phí |
| ORM | **Spring Data JPA / Hibernate** | Mapping Java Object ↔ DB |
| API | **REST API + JSON** | Web/mobile dùng chung backend |
| Authentication | **Spring Security + JWT** | Phù hợp hệ thống login/role |
| Build | **Maven** | Chuẩn và dễ quản lý dependency |
| Container | **Docker** | Dễ deploy toàn bộ hệ thống |

---

## Ghi chú đối chiếu với tài liệu hiện có (cần lưu ý khi code thật)

- `Backend/SKILLS-BACKEND.md` mô tả đúng Java + Spring Boot, khớp bảng trên — nhưng **chưa từng nói rõ build tool (Maven) hay database (PostgreSQL)**; khi khởi tạo project thật, dùng `pom.xml` (Maven), driver/dialect PostgreSQL.
- `Frontend-Web/SKILLS-FRONTEND-WEB.md` **hiện toàn bộ ví dụ code đang là JavaScript thuần** (`.jsx`, `.js`, không có kiểu dữ liệu) — **lệch với quy định TypeScript ở bảng trên**. Đây là điểm cần chốt lại: có chuyển toàn bộ ví dụ/quy ước sang `.tsx`/`.ts` + khai báo type/interface hay không. Đang để mở, chưa tự sửa vì ảnh hưởng nhiều ví dụ code trong file đó — báo lại khi bạn muốn cập nhật.
- Chưa có tài liệu nào mô tả schema PostgreSQL cụ thể hay cấu hình Docker (Dockerfile/docker-compose) — nếu cần, đây là 2 việc nên làm tiếp theo khi bắt đầu code thật.

---

*Tài liệu liên quan: [`README.md`](README.md), [`Backend/SKILLS-BACKEND.md`](Backend/SKILLS-BACKEND.md), [`Frontend-Web/SKILLS-FRONTEND-WEB.md`](Frontend-Web/SKILLS-FRONTEND-WEB.md).*
