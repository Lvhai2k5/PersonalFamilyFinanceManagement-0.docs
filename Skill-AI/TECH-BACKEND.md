# Quy định kỹ thuật: Backend — Java + Spring Boot

> Dùng làm tài liệu tham chiếu cho skill `develop-backend`/`fix-backend` (xem [`AI-SKILLS-GUIDE.md`](./AI-SKILLS-GUIDE.md)). Cấu trúc thư mục/vai trò từng lớp đã có đầy đủ ở [`../Backend/SKILLS-BACKEND.md`](../Backend/SKILLS-BACKEND.md) — file này **không lặp lại**, chỉ quy định phần đặc thù công nghệ (version, cấu hình, testing) mà file kia chưa nêu.

## 1. Version

- **Java 17 (LTS)** — bắt buộc vì Spring Boot 3.x yêu cầu tối thiểu Java 17.
- **Spring Boot 3.x** (bản ổn định mới nhất tại thời điểm khởi tạo project — ghi rõ version cụ thể vào `pom.xml` khi tạo project, không dùng range).

## 2. Cấu hình theo môi trường (profile)

- 3 file cấu hình: `application.yml` (mặc định, chỉ chứa giá trị chung không đổi theo môi trường), `application-dev.yml`, `application-prod.yml`.
- Kích hoạt qua biến môi trường `SPRING_PROFILES_ACTIVE=dev|prod`, không hardcode trong code.
- **Không commit secret thật** (password DB, JWT secret) vào `application-prod.yml` — chỉ commit `application-prod.yml.example`, giá trị thật truyền qua biến môi trường (khớp cách Docker/`TECH-CONTAINER.md` truyền `.env`).

## 3. Logging

- Dùng SLF4J + Logback có sẵn trong Spring Boot Starter, không thêm thư viện log khác.
- Level `DEBUG` ở `dev`, `INFO` ở `prod` — cấu hình qua `application-{profile}.yml`, không dùng `System.out.println`.

## 4. Testing

- **Unit test** Service layer: JUnit 5 + Mockito, mock `Repository`/`Mapper` — test đúng logic nghiệp vụ trong `ServiceImpl`, không đụng DB thật.
- **Integration test** (khi cần test cả luồng Controller→DB): `@SpringBootTest` + Testcontainers chạy PostgreSQL thật trong container test — **không dùng H2** (khác dialect với PostgreSQL ở prod, dễ pass test nhưng lỗi khi chạy thật, đặc biệt với `BigDecimal`/`DECIMAL(15,3)` đã chốt ở `BUSINESS-REQUIREMENTS.md` mục 7.1).

## 5. Điểm cần nhớ khi code (liên kết các quy định khác)

- Entity/DTO/Mapper/Exception: theo đúng `Backend/SKILLS-BACKEND.md`.
- ORM (JPA/Hibernate): xem [`TECH-ORM.md`](./TECH-ORM.md).
- REST API convention: xem [`TECH-API.md`](./TECH-API.md).
- Auth (Spring Security + JWT): xem [`TECH-AUTH.md`](./TECH-AUTH.md).
- Build (Maven): xem [`TECH-BUILD.md`](./TECH-BUILD.md).
- Database (PostgreSQL): xem [`TECH-DATABASE.md`](./TECH-DATABASE.md).
