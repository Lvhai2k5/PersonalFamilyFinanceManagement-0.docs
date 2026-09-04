# FamJar — Đặc tả nghiệp vụ (Business Requirements)

> Tổng hợp từ `Document/FinalClassDiagram.drawio` (nguồn chính — mới nhất), đối chiếu với `Document/FinanceManagement.png` (bản trước) và `Document/FamJar-Architecture.html` (kiến trúc kỹ thuật đã có). `Document/ClassDiagram.eapx` (Enterprise Architect, dạng binary) chưa đọc được — mở bằng EA nếu cần đối chiếu thêm.
>
> ⚠️ Tài liệu này **chưa hoàn toàn chốt** — mục 7 liệt kê các mâu thuẫn thật giữa các nguồn, cần bạn xác nhận trước khi bắt đầu code entity ở [Backend/SKILLS-BACKEND.md](Backend/SKILLS-BACKEND.md).

---

## 1. Giới thiệu

**FamJar** — ứng dụng quản lý tài chính gia đình. Gia đình tạo **FamilySpace** (không gian chung), trong đó chia tiền vào nhiều **MoneyJar** (hũ chi tiêu) có ngưỡng cảnh báo. Thành viên ghi nhận giao dịch (**TransactionHistory**) qua nhiều hình thức nhập liệu (Manual/OCR/Voice), giao dịch cần duyệt trước khi tính vào số dư. Hệ thống tách 2 cụm độc lập: **Admin** (giám sát, xử lý phản hồi) và **Mobile/User** (nghiệp vụ tài chính hằng ngày), dùng chung 1 database.

## 2. Đối tượng người dùng (Actor)

| Actor | Vai trò | Ghi chú |
|---|---|---|
| **Admin** | Giám sát hệ thống, khóa/mở tài khoản, xử lý Feedback, tra cứu Log | Kế thừa `Person`, có `Account` riêng |
| **User — Owner** | Chủ FamilySpace, toàn quyền: thêm/xóa thành viên, tạo MoneyJar, duyệt giao dịch | `FamilyMembership.role = Owner` |
| **User — CoOwner** | Đồng sở hữu, quyền gần như Owner | `FamilyMembership.role = CoOwner` |
| **User — Member** | Thành viên thường, tạo giao dịch nhưng không tự duyệt | `FamilyMembership.role = Member` |

> ⚠️ Class diagram chưa mô tả rõ **CoOwner khác Owner ở quyền cụ thể nào** — hiện chỉ có 3 giá trị enum `Role`, không có bảng phân quyền chi tiết theo action. Xem mục 6 và 7.

## 3. Phạm vi chức năng theo nhóm nghiệp vụ

### 3.1 Auth & Account
- Đăng ký/đăng nhập cho `User` và `Admin`, đổi mật khẩu.
- Mỗi `Account` (username/password) gắn với đúng 1 `Person` (Admin hoặc User) — xem mâu thuẫn ở mục 7.1.

### 3.2 FamilySpace & Thành viên
- `User` tạo `FamilySpace` (1 User có thể tạo nhiều FamilySpace).
- `FamilySpace.addMember()` / `removeMember()` — thêm/xóa thành viên.
- Quan hệ thành viên lưu qua `FamilyMembership` (role + trạng thái), không lưu trực tiếp trên `User`/`FamilySpace` — cho phép 1 User tham gia nhiều FamilySpace với vai trò khác nhau ở mỗi nơi.

### 3.3 MoneyJar & Threshold (ngưỡng cảnh báo)
- `FamilySpace.createMoneyJar()` — 1 FamilySpace có nhiều MoneyJar.
- Mỗi MoneyJar có `checkBalance()` và được gắn với ngưỡng cảnh báo (Threshold) — có 2 nhóm ngưỡng: **Static** (cố định) và **Dynamic** (theo %), mỗi nhóm có 2 loại con. Chi tiết cấu trúc + mâu thuẫn xem mục 4.4 và 7.2.

### 3.4 Giao dịch & Ghi nhận (TransactionHistory / RecordInput)
- `User.create()` → tạo `TransactionHistory` gắn với 1 MoneyJar, trạng thái khởi tạo `Pending`.
- Giao dịch được nhập qua 1 trong các hình thức `RecordInput`: Manual / OCR / Voice.
- `approveTransaction()` / `rejectTransaction()` — chuyển trạng thái `Pending` → `Approve`/`Reject`. Duyệt thành công → cập nhật số dư MoneyJar → kiểm tra ngưỡng → cảnh báo nếu vượt (theo luồng đã mô tả trong `FamJar-Architecture.html` §08).

### 3.5 Nhật ký (Log)
- Mọi hành động ghi (Create/Modify/Delete/Login/Logout) của `User` và mọi thay đổi trên `MoneyJar` đều tạo 1 `Log`.
- `Admin.check(Log)` — Admin **tra cứu** Log, không tự tạo Log cho hành động của chính mình trong bản Final (khác bản PNG — xem mục 7.4).

### 3.6 Phản hồi (Feedback)
- `User` tạo `Feedback` (nội dung + trạng thái + kết quả xử lý).
- `handleWithFeedback()` — xử lý phản hồi, cập nhật `StatusFeedback` + `result`.
- ⚠️ Ai gọi `handleWithFeedback()` (chắc chắn là Admin theo ngữ cảnh nghiệp vụ + theo `FamJar-Architecture.html`) **không có quan hệ tường minh Admin↔Feedback** trong bản Final — xem mục 7.5.

## 4. Thực thể nghiệp vụ (Entity Catalog — theo `FinalClassDiagram.drawio`)

### 4.1 Person (abstract)
`name`, `phone`, `birthDate`, `address`, `gmail`. 2 lớp con: **Admin** (không thêm thuộc tính), **User** (+ `identificationNumber`).

### 4.2 Account
`username`, `password`. Quan hệ 1-1 riêng với `User` và 1-1 riêng với `Admin` (2 association tách biệt trong bản Final, không phải 1 association chung tại `Person`).

### 4.3 FamilySpace / FamilyMembership
- **FamilySpace**: `name` — có `addMember()`, `removeMember()`, `createMoneyJar()`.
- **FamilyMembership**: `role: Role` (`Owner`/`CoOwner`/`Member`), `permission: Permission` (giá trị enum thực tế: `Active`/`Inactive` — xem 7.3). 1 User có 0..* FamilyMembership.

### 4.4 MoneyJar & Threshold
- **MoneyJar**: `name`, `baseMoney`, `description` — có `checkBalance()`. 1 FamilySpace có 0..* MoneyJar; 1 MoneyJar có 0..* TransactionHistory.
- **StaticThreshold** (abstract): `baseThreshold` — `alertOverThreshold()`.
  - **FixedThreshold** (kế thừa StaticThreshold, không thêm thuộc tính)
  - **FlexibleThreshold** (kế thừa StaticThreshold) — `changeThreshold()`
- **DynamicThreshold** (abstract): `percentageThreshold` — `alertOverThreshold()`.
  - **WarningThreshold** (kế thừa DynamicThreshold) — `pushNotification()`
  - **EmergencyThreshold** (kế thừa DynamicThreshold) — `closeJar()`, `openJar()`
- Có ghi chú tay "**bỏ fixed**" gần `FlexibleThreshold`/`FixedThreshold` trong diagram gốc — ý định chưa rõ, xem 7.2.

### 4.5 TransactionHistory & RecordInput
- **TransactionHistory**: `transactionID`, `content`, `expense`, `statusTransaction: StatusTransaction` (`Pending`/`Approve`/`Reject`) — `approveTransaction()`, `rejectTransaction()`. 1 TransactionHistory có đúng 1 RecordInput.
- **RecordInput** (abstract): `baseRecord` — `processRecord()`. 3 lớp con: **Manual** (không thêm thuộc tính), **OCR** (+ `baseOCRInput`), **Voice** (+ `baseVoiceInput`).

### 4.6 Log
`actionType: ActionType`, `logTime`, `ipAddress`, `description` — `checkLog(user)`. Quan hệ: Admin **check** 1..* Log; User **save account actions** vào 1..* Log; MoneyJar **save jar actions** vào 1..* Log.

### 4.7 Feedback
`content`, `statusFeedback: StatusFeedback`, `result` — `handleWithFeedback()`. User **create** 0..* Feedback.

## 5. Enum / trạng thái

| Enum | Giá trị | Dùng ở |
|---|---|---|
| `Role` | Owner, CoOwner, Member | `FamilyMembership.role` |
| `Permission` | Active, Inactive | `FamilyMembership.permission` — ⚠️ tên/giá trị không khớp, xem 7.3 |
| `StatusTransaction` | Pending, Approve, Reject | `TransactionHistory.statusTransaction` |
| `StatusFeedback` | Draft, Pending, Review, Approve, Reject | `Feedback.statusFeedback` |
| `ActionType` | Login, Logout, Create, Modify, Delete | `Log.actionType` |

## 6. Ma trận quyền theo vai trò (suy luận từ ngữ cảnh — CHƯA có trong class diagram)

| Hành động | Owner | CoOwner | Member |
|---|:---:|:---:|:---:|
| Tạo FamilySpace | ✅ (người tạo) | — | — |
| Thêm/xóa thành viên | ✅ | ✅ (giả định) | ❌ |
| Tạo MoneyJar / cấu hình Threshold | ✅ | ✅ (giả định) | ❌ |
| Tạo giao dịch (TransactionHistory) | ✅ | ✅ | ✅ |
| Duyệt/từ chối giao dịch | ✅ | ✅ (giả định) | ❌ |
| Gửi Feedback | ✅ | ✅ | ✅ |

> Bảng này là **suy luận nghiệp vụ hợp lý**, không phải trích xuất từ diagram — class diagram không có bảng phân quyền theo role×action. Cần bạn xác nhận trước khi implement `@PreAuthorize`/guard ở Service layer.

## 7. Vấn đề mở — cần xác nhận trước khi code

### 7.1 Account gắn với Person hay gắn riêng User/Admin?
Bản PNG: `Account` — 1:1 — `Person` (1 association chung ở lớp cha, User/Admin kế thừa). Bản Final (drawio): `Account` có 2 association tách biệt, 1 với `User`, 1 với `Admin`. Nếu implement theo Final, `Account` cần biết nó thuộc loại nào (discriminator hoặc 2 bảng riêng); nếu implement theo PNG, chỉ cần 1 khóa ngoại `person_id`. → **Ảnh hưởng trực tiếp đến schema DB**, cần chốt trước.

### 7.2 Phân cấp Threshold: Static/Dynamic ngược trực giác tên gọi + ghi chú "bỏ fixed"
Trong bản Final, `WarningThreshold` và `EmergencyThreshold` kế thừa **DynamicThreshold** (có `percentageThreshold`), còn `FixedThreshold` và `FlexibleThreshold` kế thừa **StaticThreshold** (có `baseThreshold`) — ngược lại với cách đặt tên trực quan (chữ "Flexible" nghe giống "Dynamic" hơn). Ngoài ra `FamJar-Architecture.html` §04 lại mô tả **1 entity `Threshold` duy nhất** với field `kind: "Fixed|Emergency|Warning|Flexible"` (không dùng kế thừa 6 lớp). Và có ghi chú tay "**bỏ fixed**" trong file gốc chưa rõ ý định (bỏ hẳn class `FixedThreshold`? hay gộp Fixed vào Emergency/Warning?).
→ **Cần chốt 1 trong 3 phương án**: (a) giữ đúng 6 lớp kế thừa như Final, (b) dùng 1 entity + enum `kind` như HTML đã đơn giản hóa, (c) bỏ Fixed theo ghi chú tay.

Ngoài ra, cardinality giữa `MoneyJar` và 2 lớp Threshold **ngược chiều nhau trong chính bản Final**: MoneyJar↔StaticThreshold ghi `0..*`:`1`, còn MoneyJar↔DynamicThreshold ghi `1`:`0..*` — 2 quan hệ có cấu trúc giống nhau nhưng multiplicity đối nghịch, nhiều khả năng là lỗi đặt nhãn khi vẽ tay chứ không phải chủ ý. Cần xác nhận: 1 MoneyJar có bao nhiêu Threshold (1 cái duy nhất, hay có thể nhiều loại cùng lúc)?

### 7.3 Enum `Permission` mang giá trị Active/Inactive — đây là Status, không phải Permission
Xác nhận từ chính bản Final: enum tên `Permission` nhưng 2 giá trị là `Active`/`Inactive` — đúng là **trạng thái thành viên** (active/bị khóa), không phải quyền hạn (đọc/ghi) như tên gọi "Permission" gợi ý. Bản PNG có 2 enum tách biệt: `Status` (Active/Inactive, gắn ở `Account`) và `Permission` (Read/Write, gắn ở `FamilyMember`) — đúng bản chất hơn.
→ **Khuyến nghị**: đổi tên enum trong Final thành `MembershipStatus` (Active/Inactive) và cân nhắc có cần thêm enum `Permission` thật (Read/Write) hay không, hay quyền hạn đã được suy ra hoàn toàn từ `Role`.

### 7.4 Log: Admin có tự tạo Log cho hành động của mình không?
Bản Final: chỉ có `User` và `MoneyJar` "save actions" vào Log; `Admin` chỉ "**check**" (đọc) Log — nghĩa là hành động của Admin (login, khóa account...) **không được ghi log**. Bản PNG gộp chung `Person` (cả Admin lẫn User) đều "save actions" vào Log.
→ Cần xác nhận: có cố ý loại Admin khỏi audit log không? Nếu không, đây là thiếu sót cần bổ sung (đặc biệt vì Admin có quyền khóa Account/xử lý Feedback — hành động nhạy cảm nên có audit).

### 7.5 Feedback: thiếu quan hệ tường minh với Admin (ai xử lý?)
Bản Final chỉ có `User` → create → `Feedback`; không có association nào từ `Admin` tới `Feedback`. Bản PNG có `Feedback` — response — `Admin` (0..*:1). Nghiệp vụ (và cả `FamJar-Architecture.html` §05, endpoint `PATCH /feedback/{id}`) đều giả định Admin là người gọi `handleWithFeedback()`.
→ Đề xuất: bổ sung lại association `Admin` 1 — `Feedback` 0..* vào bản Final (khả năng cao đây là thiếu sót khi vẽ, không phải chủ ý bỏ).

Ngoài ra có 1 edge nối `Feedback` → `Log` (nhãn "relate", 1:1) trong file gốc chưa rõ ý nghĩa — có thể là mỗi Feedback khi xử lý cũng sinh 1 Log tương ứng, cần xác nhận.

### 7.6 Đặt tên không nhất quán giữa các file nguồn
| Khái niệm | Final (drawio) | PNG (cũ) | HTML (kiến trúc) |
|---|---|---|---|
| "Hũ chi tiêu" | `MoneyJar` | `ExpenseJar` | `MoneyJar` |
| Field số dư gốc | `baseMoney` | `baseExpense` | (không nêu) |
| "Ghi nhận nhập liệu" (abstract) | `RecordInput` / `processRecord()` | `Record` / `processBaseRecord()` | (không nêu) |
| Số loại nhập liệu | 3 (Manual/OCR/Voice) | 6 (+ Message/Announcement/ScanAI) | 3 (Manual/OCR/Voice) |
| Tên sản phẩm | (không có) | (không có) | **FamJar** |

→ **Final + HTML đã thống nhất tên `MoneyJar`** và 3 loại RecordInput — nên coi đây là chốt cuối, PNG là bản nháp cũ hơn. Riêng tên sản phẩm "**FamJar**" (từ HTML) khác với tên thư mục/package hiện tại `ute.fit.financemanagement` — cần xác nhận tên chính thức dùng cho báo cáo đồ án.

### 7.7 CoOwner khác Owner ở quyền cụ thể nào?
Enum `Role` có 3 giá trị nhưng không có tài liệu nào (cả 2 diagram lẫn HTML) mô tả rõ ranh giới quyền giữa Owner và CoOwner — bảng ở mục 6 là suy luận tạm, cần bạn xác nhận trước khi code phân quyền.

---

*Dựa trên `Document/FinalClassDiagram.drawio`, `Document/FinanceManagement.png`, `Document/FamJar-Architecture.html`. Cập nhật: 2026-09-03.*
