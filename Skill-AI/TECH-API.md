# Quy định kỹ thuật: API — REST + JSON

> Dùng làm tài liệu tham chiếu chung cho cả 3 skill `develop-backend`, `develop-web`, `develop-mobile` (xem [`AI-SKILLS-GUIDE.md`](./AI-SKILLS-GUIDE.md)) — vì đây là "hợp đồng" dùng chung giữa Backend, Web và Mobile, đổi 1 quy định ở đây bắt buộc đồng bộ cả 3 phía.

## 1. Versioning & URL

- Mọi endpoint bắt đầu bằng `/api/v1/...` — đã nêu ở `../Backend/SKILLS-BACKEND.md` mục 6, nhắc lại vì đây là quy định bắt buộc chứ không phải gợi ý.
- Tài nguyên đặt tên số nhiều, danh từ (`/api/v1/transactions`, `/api/v1/expense-jars`), không dùng động từ trong URL (không viết `/api/v1/getTransactions`).

## 2. HTTP method & status code

| Method | Dùng khi | Status thành công |
|---|---|---|
| `GET` | Lấy dữ liệu, không thay đổi state | `200 OK` |
| `POST` | Tạo mới | `201 Created` |
| `PUT`/`PATCH` | Cập nhật (PUT: thay toàn bộ, PATCH: thay 1 phần) | `200 OK` |
| `DELETE` | Xóa | `204 No Content` |

- Lỗi validate (DTO sai format): `400 Bad Request`.
- Chưa đăng nhập / token hết hạn: `401 Unauthorized`.
- Đã đăng nhập nhưng không đủ quyền (ví dụ Member cố duyệt giao dịch — quyền chỉ Owner): `403 Forbidden`.
- Không tìm thấy tài nguyên: `404 Not Found`.
- Xung đột trạng thái (ví dụ duyệt giao dịch đã bị Reject trước đó): `409 Conflict`.
- Lỗi không lường trước: `500 Internal Server Error`, luôn qua `GlobalExceptionHandler`, không để lộ stack trace cho client.

## 3. Response envelope

- **Trả thẳng DTO** (không bọc thêm lớp `{ data: ... }` cho response thành công) — giữ đơn giản, khớp đúng field đã định nghĩa ở DTO, dễ đối chiếu 3 phía.
- Response lỗi dùng 1 format thống nhất do `GlobalExceptionHandler` sinh ra:

```json
{
  "timestamp": "2026-09-06T10:00:00",
  "status": 400,
  "error": "Bad Request",
  "message": "Số tiền không được để trống",
  "path": "/api/v1/transactions"
}
```

## 4. Phân trang (khi danh sách dài — ví dụ lịch sử giao dịch)

- Query param: `?page=0&size=20&sort=createdAt,desc` (0-based, khớp mặc định Spring Data `Pageable`).
- Response dùng shape của `Page<T>` Spring Data trả về (`content`, `totalElements`, `totalPages`, `number`, `size`) — Web/Mobile đọc đúng field này, không tự định nghĩa lại shape phân trang riêng.

## 5. Điểm cần nhớ

- DTO field ở cả 3 phía (Backend DTO, Web `types/`, Mobile `models/`) phải khớp tuyệt đối tên/kiểu — đổi 1 bên phải báo đồng bộ 2 bên còn lại (đã nêu ở `README.md` mục "Nguyên tắc cốt lõi").
- Không tin dữ liệu client gửi lên — luôn validate lại ở Backend (`@Valid`) dù Web/Mobile đã validate UX.
