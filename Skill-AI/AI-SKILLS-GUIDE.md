# Hướng dẫn tạo Claude Code Skills cho dự án TLCN

> Tài liệu này **chỉ là đặc tả/hướng dẫn** — mô tả 4 skill nên có và cách chúng phải hoạt động. **Chưa cài đặt (chưa tạo file skill thật)**. Khi nào cần dùng, yêu cầu Claude đọc tài liệu này rồi tạo file skill thật theo đúng đặc tả bên dưới.
>
> Cập nhật: 2026-09-06.

---

## 1. Skill là gì, vì sao cần cho dự án này

Một **Claude Code Skill** là 1 bộ hướng dẫn đóng gói (file `SKILL.md` + tài nguyên đi kèm nếu có), đặt tại `.claude/skills/<tên-skill>/SKILL.md` trong repo. Khi làm việc, Claude tự nhận diện ngữ cảnh phù hợp (dựa vào phần `description` khai báo) và tự áp dụng đúng quy trình đã đóng gói đó, thay vì phải giải thích lại từ đầu mỗi lần.

**Lý do dự án TLCN cần**: bộ tài liệu `0.docs` hiện có (`BUSINESS-REQUIREMENTS.md`, `SKILLS-PRINCIPLES.md`, `DEVELOPMENT_RULES.md`, `SKILLS-BACKEND.md`, `SKILLS-FRONTEND-WEB.md`, `SKILLS-FRONTEND-MOBILE.md`) đã quy định rất rõ cấu trúc/nguyên tắc, nhưng **không có gì bắt buộc đọc đúng file, đúng thứ tự** mỗi khi code/sửa lỗi — dễ xảy ra tình trạng viết code lệch quy ước (như 2 lỗi `Double`/link sai đã phát hiện trước đó). Skill giải quyết đúng vấn đề này: mỗi lần code/sửa lỗi Mobile hoặc Web, quy trình đọc tài liệu → xác định đúng lớp → code/sửa đúng lớp → tự kiểm tra lại được thực hiện tự động, nhất quán.

## 2. Danh sách 4 skill cần tạo

| Skill | Mục đích | Trigger (khi nào dùng) | File `TECH-*.md` bắt buộc đọc |
|---|---|---|---|
| `develop-mobile` | Code tính năng **mới** cho app Mobile (Flutter) | User yêu cầu thêm màn hình/chức năng mới ở Mobile | `TECH-MOBILE.md`, `TECH-API.md`, `TECH-AUTH.md` |
| `develop-web` | Code tính năng **mới** cho app Web (React) | User yêu cầu thêm trang/chức năng mới ở Web | `TECH-WEB.md`, `TECH-API.md`, `TECH-AUTH.md` |
| `fix-mobile` | Debug + sửa lỗi trên app Mobile (Flutter) | User báo bug/lỗi hành vi sai trên Mobile | `TECH-MOBILE.md`, `TECH-API.md`, `TECH-AUTH.md` |
| `fix-web` | Debug + sửa lỗi trên app Web (React) | User báo bug/lỗi hành vi sai trên Web | `TECH-WEB.md`, `TECH-API.md`, `TECH-AUTH.md` |

> Chỉ 4 skill này theo đúng yêu cầu hiện tại. Backend (Java) chưa nằm trong phạm vi — nếu sau này cần `develop-backend`/`fix-backend`, tạo thêm theo đúng khuôn mẫu ở mục 4, tài liệu tham chiếu cấu trúc là `Backend/SKILLS-BACKEND.md`, tài liệu công nghệ đã có sẵn ở `TECH-BACKEND.md`, `TECH-DATABASE.md`, `TECH-ORM.md`, `TECH-BUILD.md`, `TECH-CONTAINER.md` (xem mục 4.1-4.4 làm khuôn để viết bước "Quy trình" tương ứng) — không cần chờ viết thêm tài liệu công nghệ mới.

## 3. Nguyên tắc chung cho cả 4 skill (áp dụng khi cài đặt thật)

1. **Luôn đọc `BUSINESS-REQUIREMENTS.md` trước** nếu việc code/sửa liên quan tới nghiệp vụ (entity, luồng trạng thái, quyền theo role...) — không tự suy đoán nghiệp vụ, không dùng lại ví dụ minh họa (`amount/category/note`) làm entity thật.
2. **Đọc đúng 1 file cấu trúc theo layer** trước khi động tay: Mobile → `Frontend-Mobile/SKILLS-FRONTEND-MOBILE.md`; Web → `Frontend-Web/SKILLS-FRONTEND-WEB.md`. Xác định đúng code thuộc lớp nào (`screens/providers/api/models` hay `pages/hooks/api`) trước khi viết, không nhét logic nghiệp vụ vào lớp điều phối (screen/page).
3. **Đọc `SKILLS-PRINCIPLES.md` mục 4 (Checklist)** trước khi báo hoàn thành — tự rà lại code vừa viết/sửa có đúng nguyên tắc SOLID/Clean Architecture không.
4. **Đọc `DEVELOPMENT_RULES.md`** — đặc biệt bắt buộc với 2 skill `fix-*` (tránh lặp lại lỗi đã từng gặp); sau khi fix xong 1 lỗi mới có giá trị lặp lại, cân nhắc ghi thêm bài học vào file này.
5. **Không tự bịa entity/field** — nếu tài liệu cấu trúc có ví dụ minh họa (đã ghi chú rõ "không phải entity thật"), phải tra `BUSINESS-REQUIREMENTS.md` mục 3.4/4.5 để lấy đúng field.
6. **Đọc thêm file `TECH-*.md` cùng thư mục này** ứng với công nghệ liên quan đến việc đang làm — mỗi file quy định phần đặc thù công nghệ (version, cấu hình, convention) mà các file `SKILLS-*.md` không nêu. Xem bảng ánh xạ ở [`README.md`](./README.md) (mục lục `Skill-AI/`) để biết skill nào cần đọc file `TECH-*.md` nào.

## 4. Đặc tả chi tiết từng skill

### 4.1 `develop-mobile`

- **description đề xuất** (dùng làm YAML frontmatter khi cài đặt thật): "Code tính năng mới cho app Mobile Flutter + Dart của dự án FinanceManagement, đúng cấu trúc screens/providers/api/models và đúng quy định kỹ thuật Flutter/REST API/JWT đã chốt. Dùng khi user yêu cầu thêm màn hình, chức năng, hoặc luồng nhập liệu mới ở phía Mobile."
- **Quy trình**:
  1. Đọc `BUSINESS-REQUIREMENTS.md` phần liên quan tới tính năng được yêu cầu (entity, enum, luồng trạng thái).
  2. Đọc `Frontend-Mobile/SKILLS-FRONTEND-MOBILE.md` — xác định tính năng cần thêm ở đâu: `models/` (dữ liệu), `api/` (gọi endpoint), `providers/` (logic + state), `screens/` (UI lắp ráp), `widgets/common` (UI tái dùng).
  3. Đọc `Skill-AI/TECH-MOBILE.md` — áp đúng version/state management (Provider), lint, build flavor, `flutter_secure_storage` cho dữ liệu nhạy cảm.
  4. Đọc `Skill-AI/TECH-API.md` — endpoint gọi đúng convention `/api/v1/...`, xử lý đúng status code trả về, đọc đúng shape phân trang nếu là danh sách.
  5. Đọc `Skill-AI/TECH-AUTH.md` — gắn access token vào header đúng cách, xử lý refresh token khi hết hạn, lưu token bằng `flutter_secure_storage` (không dùng `SharedPreferences`).
  6. Viết theo đúng chiều phụ thuộc **Screen → Provider → Api → HTTP** — screen không được gọi thẳng `api/`.
  7. Đối chiếu bảng "Điểm khác biệt quan trọng giữa Web và Mobile" (mục 6 của `SKILLS-FRONTEND-MOBILE.md`) — dùng `dio`, `flutter_secure_storage`, `config/env.dart`, không copy nguyên cách làm bên Web.
  8. Tự kiểm tra theo Checklist ở `SKILLS-PRINCIPLES.md` trước khi báo xong.

### 4.2 `develop-web`

- **description đề xuất**: "Code tính năng mới cho app Web React + TypeScript của dự án FinanceManagement, đúng cấu trúc pages/hooks/api/components và đúng quy định kỹ thuật TypeScript/REST API/JWT đã chốt. Dùng khi user yêu cầu thêm trang, chức năng, hoặc luồng nhập liệu mới ở phía Web."
- **Quy trình**:
  1. Đọc `BUSINESS-REQUIREMENTS.md` phần liên quan.
  2. Đọc `Frontend-Web/SKILLS-FRONTEND-WEB.md` — xác định thêm ở `api/`, `hooks/`, `pages/`, hay `components/common`.
  3. Đọc `Skill-AI/TECH-WEB.md` — khai báo type/interface ở `types/` cho dữ liệu mới (không dùng `any`), giữ `tsconfig.json`/`vite.config.ts` alias khớp nhau nếu thêm import mới.
  4. Đọc `Skill-AI/TECH-API.md` — endpoint gọi đúng convention `/api/v1/...`, xử lý đúng status code trả về, đọc đúng shape phân trang nếu là danh sách.
  5. Đọc `Skill-AI/TECH-AUTH.md` — gắn access token vào header qua interceptor `axios`, xử lý khi token hết hạn, lưu token ở `localStorage`.
  6. Viết theo đúng chiều phụ thuộc **Page → Hook → Api → HTTP**.
  7. Nếu tính năng cần đồng bộ dữ liệu với Backend/Mobile (DTO dùng chung) — kiểm tra field khớp đúng theo `BUSINESS-REQUIREMENTS.md`, không tự đặt tên field khác đi.
  8. Tự kiểm tra theo Checklist ở `SKILLS-PRINCIPLES.md` trước khi báo xong.

### 4.3 `fix-mobile`

- **description đề xuất**: "Debug và sửa lỗi hành vi sai trên app Mobile Flutter + Dart của dự án FinanceManagement, kể cả lỗi liên quan gọi API/token hết hạn. Dùng khi user báo bug, crash, hoặc dữ liệu hiển thị/gửi sai ở phía Mobile."
- **Quy trình**:
  1. Đọc `DEVELOPMENT_RULES.md` trước — kiểm tra lỗi này đã từng gặp/ghi nhận chưa.
  2. Xác định lỗi thuộc lớp nào theo bảng vai trò ở `SKILLS-FRONTEND-MOBILE.md` mục 3 (`screens`/`providers`/`api`/`models`) — sửa đúng lớp gây lỗi, không sửa tạm ở lớp khác (ví dụ không vá logic nghiệp vụ ngay trong `screen` dù nhanh hơn).
  3. Nếu lỗi liên quan gọi API/đăng nhập (401, token hết hạn, dữ liệu response sai shape) — đối chiếu `Skill-AI/TECH-API.md` và `Skill-AI/TECH-AUTH.md` trước khi sửa, xác định lỗi do phía Mobile hiểu sai hợp đồng API hay do Backend trả sai.
  4. Nếu nghi ngờ lỗi do hiểu sai nghiệp vụ (ví dụ luồng trạng thái, ngưỡng duyệt) — đối chiếu lại `BUSINESS-REQUIREMENTS.md` trước khi sửa code, không đoán.
  5. Sau khi xác nhận đã sửa đúng root cause (không phải chỉ hết triệu chứng), cân nhắc ghi bài học mới vào `DEVELOPMENT_RULES.md` nếu đây là loại lỗi có thể lặp lại.

### 4.4 `fix-web`

- **description đề xuất**: "Debug và sửa lỗi hành vi sai trên app Web React + TypeScript của dự án FinanceManagement, kể cả lỗi liên quan gọi API/token hết hạn. Dùng khi user báo bug hoặc dữ liệu hiển thị/gửi sai ở phía Web."
- **Quy trình**: tương tự `fix-mobile`, thay tài liệu cấu trúc bằng `Frontend-Web/SKILLS-FRONTEND-WEB.md` (bảng vai trò mục 3: `pages`/`hooks`/`api`/`components`) và tài liệu công nghệ bằng `Skill-AI/TECH-WEB.md` — riêng bước đối chiếu API/Auth vẫn dùng chung `Skill-AI/TECH-API.md`/`Skill-AI/TECH-AUTH.md` (hợp đồng dùng chung cả 2 phía).

## 5. Khi nào và cách "cài đặt" (tạo file skill thật)

Tài liệu này **không tự động tạo skill** — đây chỉ là bản đặc tả để tham chiếu. Khi cần dùng thật:

1. Yêu cầu Claude: *"cài skill `develop-mobile`/`develop-web`/`fix-mobile`/`fix-web` theo `0.docs/Skill-AI/AI-SKILLS-GUIDE.md`"*.
2. Claude sẽ tạo `.claude/skills/<tên-skill>/SKILL.md` tương ứng, gồm YAML frontmatter (`name`, `description` lấy từ mục 4) + nội dung quy trình chi tiết hóa từ mục 4.
3. Có thể cài từng skill riêng lẻ (không bắt buộc cài đủ cả 4 cùng lúc) — tùy nhu cầu thực tế lúc đó.

---

*Tài liệu liên quan: [`../README.md`](../README.md) (mục lục docs), [`../SKILLS-PRINCIPLES.md`](../SKILLS-PRINCIPLES.md) (checklist mục 4), [`../BUSINESS-REQUIREMENTS.md`](../BUSINESS-REQUIREMENTS.md) (entity/nghiệp vụ thật).*
