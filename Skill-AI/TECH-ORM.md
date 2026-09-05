# Quy định kỹ thuật: ORM — Spring Data JPA / Hibernate

> Dùng làm tài liệu tham chiếu cho skill backend (xem [`TECH-BACKEND.md`](./TECH-BACKEND.md)) mỗi khi viết/sửa `entity/` hoặc `repository/`. Vai trò tổng quát của Entity/Repository đã có ở [`../Backend/SKILLS-BACKEND.md`](../Backend/SKILLS-BACKEND.md) mục 2 — file này quy định phần đặc thù Hibernate/JPA.

## 1. `ddl-auto`

- **`validate`** ở mọi môi trường có dữ liệu thật (kể cả dev nếu dùng chung schema với người khác) — Hibernate chỉ kiểm tra entity khớp bảng, không tự sửa schema. Schema thật do Flyway quản lý (xem `TECH-DATABASE.md`).
- Chỉ dùng `create-drop` khi chạy unit test với DB tạm (Testcontainers) — không dùng cho bất kỳ môi trường nào giữ dữ liệu qua nhiều lần chạy.

## 2. Quan hệ (relationship)

- Mặc định `FetchType.LAZY` cho mọi quan hệ `@OneToMany`/`@ManyToMany`; `@ManyToOne`/`@OneToOne` cân nhắc `LAZY` nếu không phải lúc nào cũng cần — tránh load thừa dữ liệu.
- Tránh N+1 query: khi cần lấy kèm quan hệ (ví dụ `TransactionHistory` kèm `Record`), dùng `@EntityGraph` hoặc `JOIN FETCH` trong `@Query`, không load rồi lặp gọi lazy trong vòng lặp.
- Quan hệ kế thừa `Record` (abstract) → 6 lớp con (Manual/OCR/Voice/Message/Announcement/ScanAI): dùng chiến lược `InheritanceType.JOINED` hoặc `SINGLE_TABLE` tùy quyết định lúc code entity — `SINGLE_TABLE` đơn giản hơn (1 bảng, cột `discriminator` phân biệt loại) và đủ dùng vì 6 lớp con **không thêm field riêng** trên `Record` (theo `../BUSINESS-REQUIREMENTS.md` mục 4.5, phần thuộc tính phụ trợ đặt ở bảng/entity riêng ngoài `Record`), tránh join nhiều bảng không cần thiết.

## 3. Kiểu tiền (`BigDecimal`)

- Field entity: `@Column(precision = 15, scale = 3)` khớp `DECIMAL(15,3)` ở `TECH-DATABASE.md`.
- Luôn `setScale(3, RoundingMode.HALF_UP)` ngay khi nhận giá trị từ DTO vào entity — không để Hibernate tự làm tròn ngầm, tránh sai lệch số liệu tài chính (đã nêu ở `../BUSINESS-REQUIREMENTS.md` mục 7.1).

## 4. Không để Entity lộ ra ngoài Controller

- `Repository` trả `Entity`, nhưng `Controller` chỉ được thấy `DTO` — `Mapper` là ranh giới bắt buộc (đã quy định ở `SKILLS-BACKEND.md`), nhắc lại ở đây vì đây là lỗi hay gặp nhất khi mới dùng Spring Data JPA (tiện tay trả thẳng Entity cho nhanh).

## 5. Transaction

- `@Transactional` đặt ở `ServiceImpl`, method nào có từ 2 lần gọi Repository trở lên (ví dụ tạo `TransactionHistory` + cập nhật số dư `ExpenseJar`) bắt buộc phải có, để đảm bảo tất cả cùng thành công hoặc cùng rollback.
