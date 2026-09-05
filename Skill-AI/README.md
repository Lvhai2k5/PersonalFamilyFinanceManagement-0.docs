# Skill-AI — Mục lục

> Toàn bộ tài liệu phục vụ việc tạo/dùng Claude Code Skill (devkit) cho dự án TLCN. Nguồn công nghệ chốt ở [`../TECH-STACK.md`](../TECH-STACK.md) — mỗi dòng trong bảng đó có 1 file `TECH-*.md` tương ứng ở đây, quy định cụ thể cách dùng đúng công nghệ đó trong dự án.

## Đặc tả skill

- [`AI-SKILLS-GUIDE.md`](./AI-SKILLS-GUIDE.md) — đặc tả 4 skill `develop-mobile`/`develop-web`/`fix-mobile`/`fix-web` (mục đích, trigger, quy trình). Đọc file này trước khi cài skill thật.

## File công nghệ (`TECH-*.md`) — đối chiếu bảng Tech Stack

| Dòng trong `TECH-STACK.md` | File tương ứng | Skill nào cần đọc |
|---|---|---|
| Backend — Java + Spring Boot | [`TECH-BACKEND.md`](./TECH-BACKEND.md) | `develop-backend`, `fix-backend` (khi tạo) |
| Web — React + TypeScript | [`TECH-WEB.md`](./TECH-WEB.md) | `develop-web`, `fix-web` |
| Mobile — Flutter + Dart | [`TECH-MOBILE.md`](./TECH-MOBILE.md) | `develop-mobile`, `fix-mobile` |
| Database — PostgreSQL | [`TECH-DATABASE.md`](./TECH-DATABASE.md) | `develop-backend`, `fix-backend` |
| ORM — Spring Data JPA / Hibernate | [`TECH-ORM.md`](./TECH-ORM.md) | `develop-backend`, `fix-backend` |
| API — REST + JSON | [`TECH-API.md`](./TECH-API.md) | cả 4 skill (hợp đồng dùng chung) |
| Authentication — Spring Security + JWT | [`TECH-AUTH.md`](./TECH-AUTH.md) | cả 4 skill (hợp đồng dùng chung) |
| Build — Maven | [`TECH-BUILD.md`](./TECH-BUILD.md) | `develop-backend`, `fix-backend` |
| Container — Docker | [`TECH-CONTAINER.md`](./TECH-CONTAINER.md) | `develop-backend`, `fix-backend` |

> `develop-backend`/`fix-backend` chưa nằm trong 4 skill đã đặc tả ở `AI-SKILLS-GUIDE.md` (xem mục 2 file đó) — 5 file `TECH-*.md` liên quan Backend vẫn tạo sẵn ở đây để dùng ngay khi cần mở rộng thêm 2 skill đó, không phải chờ viết lại từ đầu.

## Quan hệ với các file `SKILLS-*.md` gốc

`TECH-*.md` **không thay thế** `Backend/SKILLS-BACKEND.md`, `Frontend-Web/SKILLS-FRONTEND-WEB.md`, `Frontend-Mobile/SKILLS-FRONTEND-MOBILE.md` — 3 file gốc đó quy định **cấu trúc thư mục & vai trò từng lớp** (kiến trúc), còn `TECH-*.md` quy định **phần đặc thù công nghệ** (version, cấu hình, convention, lệnh chạy) mà 3 file gốc không nêu. Khi code/sửa, đọc cả 2 loại — không loại nào thay được loại nào.

---

*Cập nhật: 2026-09-06.*
