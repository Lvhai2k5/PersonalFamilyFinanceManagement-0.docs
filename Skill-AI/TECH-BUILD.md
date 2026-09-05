# Quy định kỹ thuật: Build — Maven

> Dùng làm tài liệu tham chiếu cho skill backend (xem [`TECH-BACKEND.md`](./TECH-BACKEND.md)) mỗi khi thêm dependency hoặc cấu hình build.

## 1. `pom.xml`

- Kế thừa `spring-boot-starter-parent` (quản lý version dependency Spring Boot đồng bộ, tránh tự chọn version lẻ tẻ xung đột nhau).
- Khai báo `<java.version>17</java.version>` (khớp `TECH-BACKEND.md` mục 1).
- Version dependency: khai báo **tường minh, cố định** (ví dụ `2.x.y`), không dùng `LATEST`/range — đảm bảo build lặp lại giống nhau (reproducible build) giữa các máy/lần build.

## 2. Dependency tối thiểu cần có

- `spring-boot-starter-web` (REST API), `spring-boot-starter-data-jpa` (ORM), `spring-boot-starter-security` (auth), `spring-boot-starter-validation` (`@Valid`), driver `postgresql`, `flyway-core` (migration — xem `TECH-DATABASE.md`), `jjwt` hoặc thư viện JWT tương đương, `mapstruct` (Mapper, nếu chọn dùng thay vì viết tay), `lombok` (giảm boilerplate getter/setter — tùy chọn).

## 3. Lệnh chuẩn

- `mvn clean install` — build + chạy test trước khi coi 1 thay đổi là xong (không chỉ chạy `mvn compile`).
- `mvn spring-boot:run -Dspring-boot.run.profiles=dev` — chạy local với profile `dev`.
- Build ra 1 file `.jar` duy nhất (Spring Boot fat jar, plugin `spring-boot-maven-plugin` mặc định đã làm việc này) — dùng file này làm input cho `TECH-CONTAINER.md` (Docker image).

## 4. Không dùng Gradle song song

Chỉ 1 build tool cho toàn bộ Backend — tránh có cả `pom.xml` lẫn `build.gradle` gây nhầm lẫn công cụ nào là nguồn thật khi CI/CD hoặc người khác clone project về.
