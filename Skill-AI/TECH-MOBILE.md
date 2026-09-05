# Quy định kỹ thuật: Mobile — Flutter + Dart

> Dùng làm tài liệu tham chiếu cho skill `develop-mobile`/`fix-mobile` (xem [`AI-SKILLS-GUIDE.md`](./AI-SKILLS-GUIDE.md)). Cấu trúc thư mục/vai trò từng lớp đã có đầy đủ ở [`../Frontend-Mobile/SKILLS-FRONTEND-MOBILE.md`](../Frontend-Mobile/SKILLS-FRONTEND-MOBILE.md) — file này chỉ quy định phần đặc thù công nghệ.

## 1. Version & State management

- Flutter SDK: bản **stable** mới nhất tại thời điểm khởi tạo project — ghi version cụ thể vào `pubspec.yaml`/`.fvmrc` nếu dùng FVM để cố định version giữa các máy.
- State management: **Provider** (đã chốt ở `SKILLS-FRONTEND-MOBILE.md`) — không tự đổi sang Riverpod/Bloc giữa chừng vì sẽ đổi luôn cấu trúc `providers/`.

## 2. Lint/Format

- Bật `flutter_lints` (mặc định khi tạo project bằng `flutter create`) — không tắt rule để code nhanh hơn.
- `dart format .` trước khi coi 1 tính năng là xong.

## 3. Environment & Build flavor

- Không hardcode URL backend — đọc qua `config/env.dart`, giá trị truyền vào lúc build bằng `--dart-define=API_BASE_URL=...` (hoặc `flutter_dotenv` nếu cần đơn giản hơn cho khoá luận).
- Phân biệt `dev`/`prod` bằng build flavor hoặc biến `--dart-define`, không sửa tay `env.dart` trước mỗi lần build.

## 4. Bảo mật dữ liệu cục bộ

- Token đăng nhập: `flutter_secure_storage` — **không** dùng `SharedPreferences` cho dữ liệu nhạy cảm (khác Web dùng `localStorage`, xem `TECH-AUTH.md`).

## 5. Testing (khi cần)

- Unit test cho `providers/` (logic nghiệp vụ phía client) và `utils/` (hàm thuần) — dùng `flutter_test` có sẵn.
- Widget test cho `screens/` quan trọng (luồng nhập liệu, submit form) — không bắt buộc phủ 100%, ưu tiên luồng chính (happy path) + 1-2 trường hợp lỗi theo bảng "bad case" đã liệt kê ở `Document/PLAN-INPUT-METHODS-*.md`.

## 6. Điểm cần nhớ khi code (liên kết các quy định khác)

- Cấu trúc `screens/providers/api/models`: theo đúng `Frontend-Mobile/SKILLS-FRONTEND-MOBILE.md`.
- Entity/field thật khi gọi API: tra `../BUSINESS-REQUIREMENTS.md` mục 3.4/4.5, không tự bịa field.
- REST API convention phía backend phải khớp: xem [`TECH-API.md`](./TECH-API.md).
- Auth (refresh token, thời hạn): xem [`TECH-AUTH.md`](./TECH-AUTH.md).
