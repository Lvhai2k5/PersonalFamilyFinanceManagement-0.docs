# Quy định kỹ thuật: Container — Docker

> Dùng làm tài liệu tham chiếu cho skill backend (xem [`TECH-BACKEND.md`](./TECH-BACKEND.md)) khi cần đóng gói/chạy hệ thống bằng Docker. **Chỉ áp dụng cho Backend + Database** — Web build ra static file (deploy riêng, không nhất thiết cần container cho phạm vi khoá luận), Mobile không đóng container (build APK/IPA theo `TECH-MOBILE.md`).

## 1. Dockerfile Backend — multi-stage build

```dockerfile
# Stage 1: build bằng Maven + JDK đầy đủ
FROM maven:3.9-eclipse-temurin-17 AS build
WORKDIR /app
COPY pom.xml .
RUN mvn dependency:go-offline
COPY src ./src
RUN mvn clean package -DskipTests

# Stage 2: runtime chỉ cần JRE, image nhẹ
FROM eclipse-temurin:17-jre-alpine
WORKDIR /app
COPY --from=build /app/target/*.jar app.jar
ENTRYPOINT ["java", "-jar", "app.jar"]
```

Multi-stage giúp image cuối cùng không mang theo Maven/source code, chỉ có `.jar` + JRE — nhẹ hơn và không lộ source trong image chạy thật.

## 2. `docker-compose.yml` cho local dev

```yaml
services:
  postgres:
    image: postgres:16-alpine
    environment:
      POSTGRES_DB: financemanagement
      POSTGRES_USER: ${DB_USER}
      POSTGRES_PASSWORD: ${DB_PASSWORD}
    volumes:
      - pgdata:/var/lib/postgresql/data
    ports:
      - "5432:5432"

  backend:
    build: .
    environment:
      SPRING_PROFILES_ACTIVE: dev
      SPRING_DATASOURCE_URL: jdbc:postgresql://postgres:5432/financemanagement
      SPRING_DATASOURCE_USERNAME: ${DB_USER}
      SPRING_DATASOURCE_PASSWORD: ${DB_PASSWORD}
    ports:
      - "8080:8080"
    depends_on:
      - postgres

volumes:
  pgdata:
```

- Biến `${DB_USER}`/`${DB_PASSWORD}` đọc từ file `.env` (Docker Compose tự đọc `.env` cùng thư mục) — commit `.env.example`, **không commit `.env` thật**.
- `volumes: pgdata` giữ dữ liệu Postgres qua các lần `docker compose down/up`, tránh mất dữ liệu test khi restart container.

## 3. Lệnh chuẩn

- `docker compose up -d` — chạy cả Backend + Postgres local, giống cấu hình dùng khi deploy thật hơn là cài Postgres trực tiếp lên máy.
- `docker compose down` — dừng, thêm `-v` nếu muốn xóa luôn volume (mất dữ liệu, chỉ dùng khi cố ý reset).

## 4. Không hardcode bất kỳ giá trị nhạy cảm nào trong `Dockerfile`/`docker-compose.yml`

Mọi secret (password DB, JWT secret) đi qua biến môi trường/`.env` — khớp nguyên tắc đã nêu ở `TECH-BACKEND.md` mục 2 và `TECH-DATABASE.md` mục 5.
