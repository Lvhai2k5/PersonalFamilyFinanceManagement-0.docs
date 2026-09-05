# Quy định kỹ thuật: Authentication — Spring Security + JWT

> Dùng làm tài liệu tham chiếu chung cho cả 3 skill `develop-backend`, `develop-web`, `develop-mobile` — đây là nguồn quy định canonical duy nhất về auth, `Frontend-Web/SKILLS-FRONTEND-WEB.md` và `Frontend-Mobile/SKILLS-FRONTEND-MOBILE.md` chỉ nói tới cách **lưu token phía client**, không lặp lại toàn bộ ở đây.

## 1. Vì sao JWT (không dùng session cookie)

Hệ thống có nhiều loại client cùng gọi 1 backend (Web + Mobile) — JWT stateless tránh phải đồng bộ session giữa nhiều client, mỗi request tự chứa đủ thông tin xác thực trong token (đã nêu ở `../Backend/SKILLS-BACKEND.md` mục 6 và `../README.md`).

## 2. Cấu trúc token

- **Access token**: thời hạn ngắn (khuyến nghị 15–30 phút), payload chứa `sub` (userId), `role` (Owner/Member/Admin — theo `FamilyMember.role`/`Person` loại nào), `exp`. Dùng để gọi API, gắn vào header `Authorization: Bearer <token>`.
- **Refresh token**: thời hạn dài hơn (khuyến nghị 7 ngày), chỉ dùng để lấy access token mới qua endpoint riêng, **không** dùng trực tiếp để gọi API nghiệp vụ.

## 3. Endpoint auth

- `POST /api/v1/auth/login` — trả cặp access + refresh token.
- `POST /api/v1/auth/refresh` — nhận refresh token, trả access token mới.
- `POST /api/v1/auth/logout` — vô hiệu hóa refresh token hiện tại (nếu lưu refresh token ở DB/Redis để kiểm soát revoke; nếu không lưu, logout chỉ có tác dụng phía client tự xóa token — cân nhắc mức độ cần thiết theo phạm vi khoá luận).

## 4. `SecurityConfig` — endpoint public/private

- Whitelist public (không cần token): `/api/v1/auth/**`, tài liệu Swagger (nếu bật).
- Còn lại **mặc định yêu cầu token hợp lệ** — không whitelist theo kiểu liệt kê ngược (an toàn hơn: chặn hết, mở dần, thay vì mở hết, chặn dần).
- Phân quyền theo `role` ở tầng Service (kiểm tra `Owner` mới được duyệt giao dịch/cấu hình ngưỡng — theo ma trận quyền `../BUSINESS-REQUIREMENTS.md` mục 6), **không chỉ dựa vào `@PreAuthorize` ở Controller** — vì quyền Owner/Member ở đây gắn với từng `FamilySpace` cụ thể (1 user có thể là Owner ở space này, Member ở space khác), không phải role tĩnh toàn hệ thống mà annotation đơn giản xử lý đủ.

## 5. Mật khẩu

- Hash bằng `BCryptPasswordEncoder` — không tự viết thuật toán hash, không lưu plaintext dù chỉ tạm thời trong log.

## 6. Lưu token phía client (tham chiếu, không lặp chi tiết)

| Phía | Nơi lưu | Chi tiết đầy đủ |
|---|---|---|
| Web | `localStorage` | [`../Frontend-Web/SKILLS-FRONTEND-WEB.md`](../Frontend-Web/SKILLS-FRONTEND-WEB.md) |
| Mobile | `flutter_secure_storage` | [`../Frontend-Mobile/SKILLS-FRONTEND-MOBILE.md`](../Frontend-Mobile/SKILLS-FRONTEND-MOBILE.md) |

Web dùng `localStorage` chấp nhận rủi ro XSS đọc được token (đã ghi nhận là điểm khác biệt có chủ đích ở `SKILLS-FRONTEND-MOBILE.md` mục 6) — Mobile bắt buộc dùng secure storage vì có API mã hóa sẵn, không có lý do để dùng `SharedPreferences` cho token.
