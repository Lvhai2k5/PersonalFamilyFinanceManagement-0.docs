# Quy ước và quy trình làm việc với Git

## Mục đích

Tài liệu này quy định quy trình làm việc với Git, chiến lược nhánh, quy ước đặt tên nhánh và quy ước viết thông điệp commit nhằm đảm bảo:

- Sự phối hợp nhất quán giữa các thành viên phát triển.
- Khả năng truy vết rõ ràng giữa các thay đổi mã nguồn và yêu cầu nghiệp vụ.
- Quy trình rà soát mã nguồn và phát hành hiệu quả.
- Giảm xung đột khi hợp nhất và cải thiện khả năng bảo trì kho mã nguồn.

---

# Quy trình làm việc

## Trách nhiệm của nhà phát triển

### 1. Đồng bộ các thay đổi mới nhất
Trước khi bắt đầu công việc mới:

```bash
git checkout develop
git pull origin develop
```

### 2. Tạo nhánh làm việc
Tạo nhánh từ `develop` và sử dụng mã User Story được giao cho bạn. Phần `<US_ID>` phải được thay bằng mã User Story thực tế.

```bash
git checkout -b feature/<US_ID>
```

Ví dụ với mã User Story `US114`:

```bash
git checkout -b feature/US114
```

### 3. Commit
Trong quá trình triển khai:

```bash
git add .
git commit -m "feat(scope): description"
git push origin <branch-name>
```

- Tạo commit thường xuyên với thông điệp mô tả chính xác thay đổi.
- Mỗi commit chỉ nên tập trung vào một thay đổi logic.
- Đảm bảo đã kiểm thử trên máy cục bộ trước khi đẩy mã nguồn lên kho lưu trữ.


### 4. Tạo Pull Request
Sau khi hoàn tất việc phát triển:

- Đẩy mã nguồn mới nhất lên kho lưu trữ từ xa.
- Tạo Pull Request (PR).
- Đặt nhánh đích của PR là `develop`.
- Chỉ định ít nhất một người rà soát.
- Liên kết User Story và Task tương ứng.
- Viết và chạy unit test cho các chức năng đã xác định trước khi tạo PR.
- Xử lý xung đột khi hợp nhất và chủ động thông báo cho các thành viên liên quan nếu phát sinh vấn đề.

### 5. Xử lý phản hồi rà soát
- Giải quyết các nhận xét từ người rà soát.
- Đẩy các cập nhật lên cùng nhánh.
- Đánh dấu nhận xét rà soát là đã xử lý khi phù hợp.

---

## Trách nhiệm của người rà soát (người được giao PR)

### 1. Rà soát chất lượng mã nguồn
Kiểm tra:

- Các tiêu chuẩn viết mã được tuân thủ.
- Giải pháp đáp ứng các yêu cầu.
- Không phát sinh độ phức tạp không cần thiết.
- Không có vấn đề về bảo mật hoặc hiệu năng.

### 2. Rà soát phạm vi chức năng
Kiểm tra:

- Phần triển khai đáp ứng User Story được liên kết.
- Các tiêu chí chấp nhận được đáp ứng.
- Các trường hợp biên đã được xem xét.

### 3. Rà soát việc tuân thủ quy ước Git
Kiểm tra:

- Tên nhánh tuân thủ quy ước.
- Thông điệp commit tuân thủ quy ước.
- Không bao gồm các thay đổi không liên quan.

### 4. Phê duyệt hoặc yêu cầu thay đổi
- Phê duyệt khi các yêu cầu đã được đáp ứng.
- Yêu cầu thay đổi khi cần cải thiện hoặc sửa lỗi.

### 5. Chính sách hợp nhất
- Chỉ các PR đã được phê duyệt mới được hợp nhất.
- PR phải vượt qua tất cả bước kiểm tra bắt buộc trước khi hợp nhất.

---

# Chiến lược nhánh

## Cấu trúc nhánh

```text
main
└── develop
    ├── feature/*
    ├── bugfix/*
    ├── hotfix/*
    ├── refactor/*
    └── release/*
```

### main

- Nhánh chứa phiên bản sẵn sàng đưa vào môi trường thực tế.
- Luôn phải duy trì trạng thái ổn định.
- Không được phép commit trực tiếp.

### develop

- Nhánh tích hợp.
- Mọi công việc phát triển được hợp nhất vào nhánh này thông qua Pull Request.

### feature/*

- Dùng cho chức năng mới hoặc cải tiến.
- Được tạo từ `develop`.
- Được hợp nhất trở lại vào `develop`.

### bugfix/*

- Dùng để sửa lỗi thông thường, không phải lỗi nghiêm trọng cần xử lý khẩn cấp.
- Được tạo từ `develop`.
- Được hợp nhất trở lại vào `develop` thông qua Pull Request.

### hotfix/*

- Dùng để sửa các lỗi nghiêm trọng trên môi trường thực tế.
- Khi cần thiết, được tạo từ `main`.
- Sau khi xử lý, thay đổi cần được hợp nhất vào `develop` để các nhánh tiếp tục nhận được bản sửa lỗi.

### refactor/*

- Dùng để tái cấu trúc mã nguồn mà không thay đổi hành vi nghiệp vụ.

### release/*

- Dùng cho các hoạt động chuẩn bị phát hành.

---

# Quy ước đặt tên nhánh

## Định dạng

```text
<type>/<US_ID>
```

Trong đó `<type>` là loại nhánh (ví dụ: `feature`, `bugfix`, `hotfix` hoặc `refactor`), còn `<US_ID>` là mã User Story được giao.

## Quy tắc

- Phần loại nhánh phải viết bằng chữ thường.
- Phần mã User Story phải giữ nguyên cách viết trong hệ thống quản lý công việc, ví dụ `US114`.
- Tên nhánh phải dùng mã User Story, không thay bằng tên công việc.
- Mỗi User Story sử dụng đúng mã được giao; không tự ý đổi định dạng mã.

## Ví dụ

```text
feature/US114
bugfix/US125
hotfix/US130
refactor/US142
```

Các nhánh `release/*` là nhánh chuẩn bị phát hành, không gắn với một User Story cụ thể; phần sau dấu `/` dùng số phiên bản phát hành, ví dụ `release/v1.3.0`.

---

# Quy ước viết thông điệp commit

## Định dạng chuẩn

```text
<type>(scope): <description>
```

## Các loại commit

| Loại | Mục đích |
|--------|---------|
| feat | Tính năng mới |
| fix | Sửa lỗi |
| refactor | Tái cấu trúc mã nguồn |
| docs | Cập nhật tài liệu |
| test | Thêm hoặc cập nhật kiểm thử |
| chore | Công việc bảo trì |
| style | Chỉ chỉnh sửa định dạng |
| perf | Cải thiện hiệu năng |
| build | Cấu hình quá trình build |
| ci | Thay đổi CI/CD |
|

## Ví dụ

```text
feat(search): add customer filtering
fix(auth): handle expired token
refactor(api): simplify request mapping
docs(workflow): update review process
test(payment): add refund test cases
chore(deps): upgrade dependencies
```

---

# Hướng dẫn truy vết

Nhóm sử dụng cả hai thông tin sau:

- User Story 
- Tên công việc

Quy tắc:

- Tên nhánh BẮT BUỘC sử dụng mã User Story theo định dạng `<type>/<US_ID>`.
- Thông điệp commit NÊN đề cập đến phạm vi chức năng.
- Pull Request NÊN bao gồm cả User Story và tên công việc.

Ví dụ:

```text
User Story: Quản lý khách hàng
Task: Tạo màn hình tìm kiếm khách hàng

Branch:
feature/US114

PR Title:
[Quản lý khách hàng] Tạo màn hình tìm kiếm khách hàng
```

---

# Tài liệu tham khảo

Quy ước này dựa trên các thông lệ được áp dụng rộng rãi trong ngành:

1. Conventional Commits (Quy ước commit)
   - https://www.conventionalcommits.org/en/v1.0.0/

2. Tài liệu Pull Request của GitHub
   - https://docs.github.com/en/pull-requests

3. Các phương pháp tốt nhất về đặt tên nhánh Git
   - https://www.atlassian.com/git/tutorials/comparing-workflows

Khi điều chỉnh tài liệu này, luôn trích dẫn tài liệu chính thức hoặc các nguồn được công nhận trong ngành cho mọi thay đổi về quy trình.
