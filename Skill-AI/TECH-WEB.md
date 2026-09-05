# Quy định kỹ thuật: Web — React + TypeScript

> Dùng làm tài liệu tham chiếu cho skill `develop-web`/`fix-web` (xem [`AI-SKILLS-GUIDE.md`](./AI-SKILLS-GUIDE.md)). Cấu trúc thư mục/vai trò từng lớp đã có đầy đủ ở [`../Frontend-Web/SKILLS-FRONTEND-WEB.md`](../Frontend-Web/SKILLS-FRONTEND-WEB.md) — file này chỉ quy định phần đặc thù công nghệ.

## 1. Version & công cụ

- **React 18+**, **TypeScript 5+**, bundler **Vite** (đã dùng trong ví dụ `SKILLS-FRONTEND-WEB.md`, không đổi sang Webpack/CRA).
- Package manager: chọn 1 và giữ nhất quán (npm hoặc pnpm) — không trộn lockfile của 2 công cụ trong cùng repo.

## 2. Cấu hình TypeScript

- `tsconfig.json` bật `"strict": true` — bắt buộc, không tắt để né lỗi nhanh.
- Alias `@/` trỏ `src/` phải khai báo khớp ở **cả** `tsconfig.json` (`paths`) và `vite.config.ts` (`resolve.alias`) — thiếu 1 trong 2 sẽ lỗi resolve module dù compile không báo lỗi.
- Hạn chế `any`: nếu chưa rõ shape dữ liệu, khai báo tạm ở `src/types/` rồi tinh chỉnh sau.

## 3. Lint/Format

- ESLint (rule set `eslint-plugin-react` + `@typescript-eslint`) + Prettier — chạy trước khi coi 1 tính năng là xong, không chỉ dựa vào build pass.

## 4. Environment

- `.env` theo Vite (`VITE_API_BASE_URL`...) — khác nhau giữa `dev`/`prod`, không hardcode URL backend trong code (đã nêu ở `SKILLS-FRONTEND-WEB.md` mục 5).
- Không commit `.env` chứa giá trị thật, chỉ commit `.env.example`.

## 5. Testing (khi cần)

- Unit test hàm thuần ở `utils/`: Vitest.
- Test component: React Testing Library — test hành vi (render đúng, click gọi đúng callback), không test chi tiết implementation nội bộ.

## 6. Điểm cần nhớ khi code (liên kết các quy định khác)

- Cấu trúc `api/hooks/pages/components`: theo đúng `Frontend-Web/SKILLS-FRONTEND-WEB.md`.
- Entity/field thật khi gọi API: tra `../BUSINESS-REQUIREMENTS.md` mục 3.4/4.5, không tự bịa field.
- REST API convention phía backend phải khớp: xem [`TECH-API.md`](./TECH-API.md).
- Auth (lưu token, gọi refresh): xem [`TECH-AUTH.md`](./TECH-AUTH.md).
