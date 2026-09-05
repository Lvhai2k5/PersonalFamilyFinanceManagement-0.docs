# TLCN — FinanceManagement — Developer Guide

> Tài liệu kỹ thuật cho dự án `ute.fit.financemanagement` (Backend Java, Web React, Mobile Flutter).

## Đọc gì trước

| Bạn đang làm gì | Đọc file |
|---|---|
| Mới bắt đầu, muốn hiểu tổng thể | File này → [SKILLS-PRINCIPLES.md](SKILLS-PRINCIPLES.md) |
| Muốn biết công nghệ chính thức dùng ở mỗi phần | [TECH-STACK.md](TECH-STACK.md) |
| Hiểu nghiệp vụ, actor, entity, ma trận quyền | [BUSINESS-REQUIREMENTS.md](BUSINESS-REQUIREMENTS.md) |
| Code Backend (Java/Spring Boot) | [Backend/SKILLS-BACKEND.md](Backend/SKILLS-BACKEND.md) |
| Code Web (React + TypeScript) | [Frontend-Web/SKILLS-FRONTEND-WEB.md](Frontend-Web/SKILLS-FRONTEND-WEB.md) |
| Code Mobile (Flutter) | [Frontend-Mobile/SKILLS-FRONTEND-MOBILE.md](Frontend-Mobile/SKILLS-FRONTEND-MOBILE.md) |
| Trước khi commit — tự review code có đúng nguyên tắc không | [SKILLS-PRINCIPLES.md](SKILLS-PRINCIPLES.md) mục 4 (Checklist) |
| Gặp lỗi, muốn ghi lại bài học / tránh lặp lại | [DEVELOPMENT_RULES.md](DEVELOPMENT_RULES.md) |
| Muốn cài Claude Code Skill hỗ trợ code/fix Mobile/Web | [Skill-AI/AI-SKILLS-GUIDE.md](Skill-AI/AI-SKILLS-GUIDE.md) |

## Hệ thống là gì

**FinanceManagement** — ứng dụng quản lý tài chính cá nhân, gồm:

```
Backend (Java Spring Boot)  — REST API, package gốc: ute.fit.financemanagement
        ↑                ↑
   Web (React)      Mobile (Flutter)
```

Cả Web và Mobile cùng gọi chung 1 backend qua REST API (JWT stateless).

## Nguyên tắc cốt lõi (áp dụng cho cả 3 phía)

- Mỗi lớp/tầng có đúng 1 trách nhiệm, luồng phụ thuộc 1 chiều (chi tiết: [SKILLS-PRINCIPLES.md](SKILLS-PRINCIPLES.md)).
- DTO là "hợp đồng" chung giữa Backend ↔ Web ↔ Mobile — đổi field 1 bên phải cập nhật đồng bộ các bên còn lại.
- Không tin dữ liệu từ client: validate cả ở Frontend (UX) lẫn Backend (bắt buộc, vì đó mới là nơi đảm bảo an toàn).

## Mục lục toàn bộ docs

- [TECH-STACK.md](TECH-STACK.md) — Công nghệ chính thức từng phần (Backend/Web/Mobile/DB/Auth/Build/Container) + lý do chọn
- [BUSINESS-REQUIREMENTS.md](BUSINESS-REQUIREMENTS.md) — Đặc tả nghiệp vụ: actor, entity, use case, vấn đề mở
- [SKILLS-PRINCIPLES.md](SKILLS-PRINCIPLES.md) — Clean Architecture, SOLID, OOP
- [DEVELOPMENT_RULES.md](DEVELOPMENT_RULES.md) — Luật lệ + bài học rút ra khi code thật
- [Backend/SKILLS-BACKEND.md](Backend/SKILLS-BACKEND.md) — Cấu trúc Backend Java
- [Frontend-Web/SKILLS-FRONTEND-WEB.md](Frontend-Web/SKILLS-FRONTEND-WEB.md) — Cấu trúc Web React + TypeScript
- [Frontend-Mobile/SKILLS-FRONTEND-MOBILE.md](Frontend-Mobile/SKILLS-FRONTEND-MOBILE.md) — Cấu trúc Mobile Flutter
- [Skill-AI/AI-SKILLS-GUIDE.md](Skill-AI/AI-SKILLS-GUIDE.md) — Đặc tả 4 Claude Code Skill hỗ trợ code/fix Mobile & Web (chưa cài đặt, chỉ là hướng dẫn)

---

*Cập nhật lần cuối: 2026-09-06.*
