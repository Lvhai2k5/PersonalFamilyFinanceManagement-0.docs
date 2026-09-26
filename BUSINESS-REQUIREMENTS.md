# FinanceManagement — Đặc tả nghiệp vụ (Business Requirements)

> Nguồn: `Document/images/TLCN.drawio (9).png` — class diagram **mới nhất** (2026-09-26), thay thế hoàn toàn bản `FinanceManagement.png` trước đó. So với bản trước, đây là **một vòng tái cấu trúc lớn**, không chỉ đổi tên: bỏ phân lớp `Admin`/`User` kế thừa từ `Person` (vai trò giờ nằm trên `Account.accountRole`), bỏ toàn bộ cây kế thừa `Record` (Manual/OCR/Voice/Message/Announcement/ScanAI) để thay bằng 1 field `rawData: String` trên `Transaction`, tách 3 ngưỡng số ra thành entity `Threshold` riêng, và đổi tên hàng loạt: `FamilySpace`→`Family`, `FamilyMember`→`Relationship`, `ExpenseJar`→`FinancialCategory`, `TransactionHistory`→`Transaction`. `Document/SRS.docx` (2026-09-26) vẫn là mô tả nghiệp vụ dạng văn bản dùng để đối chiếu ý nghĩa nghiệp vụ đằng sau các entity. `Document/ClassDiagram.eapx` (Enterprise Architect, binary) không dùng để đối chiếu được bằng công cụ text.
>
> **Một số multiplicity nhỏ trên ảnh PNG khó đọc tuyệt đối chính xác** (ví dụ số lượng giữa `Loggable` và `Log`) — nếu cần độ chính xác tuyệt đối khi sinh entity/migration, nên mở lại file `.drawio` gốc để xác nhận. Mục 9 liệt kê toàn bộ điểm cần xác nhận thêm với người dùng, phát sinh từ những thay đổi lớn ở vòng tái cấu trúc này.

---

## 1. Giới thiệu

**FinanceManagement** — ứng dụng quản lý tài chính cá nhân và gia đình (package gốc `ute.fit.financemanagement`, xem [README.md](README.md)). Một `Person` có đúng 1 `Account` (mang `accountRole`: Admin hoặc User — vai trò không còn là phân lớp kế thừa như bản cũ). `Person` tạo được nhiều `Family`; trong mỗi `Family`, một `Person` tham gia qua `Relationship` (association class) với `memberRole` (Owner/Member) và `customPermission` (danh sách quyền chi tiết: Read/Write/Approval/Share/Invitation). Tiền được chia vào nhiều `FinancialCategory` (trước gọi là "hũ chi tiêu"); mỗi category có 1..* `Threshold` (loại Approve/Warning/Emergency, có thể bật/tắt và đặt số tiền riêng). Giao dịch (`Transaction`) có `content`, `amount`, `transactionType` (Income/Expense), `transactionStatus`, và `rawData` (dữ liệu thô nhập vào — không còn phân theo 6 kênh nhập liệu như bản thiết kế trước, xem mục 9.2). `Person` gửi/nhận `Feedback` (không còn tách rõ "User tạo" / "Admin trả lời" bằng 2 association riêng — giờ dùng chung `Person — answers — Feedback`, phân biệt qua `accountRole`). Mọi hành động quan trọng trên `Account`, `Family`, `Transaction`, `Feedback` (đều hiện thực `Loggable`) được ghi vào `Log`.

## 2. Đối tượng người dùng (Actor)

| Actor | Xác định bằng | Vai trò |
|---|---|---|
| **Admin** | `Account.accountRole = Admin` | Khóa/mở `Account` của người khác, quản lý `Log` toàn hệ thống, xem và phản hồi `Feedback`. **Không còn** là 1 quan hệ tường minh trên diagram (kiểu "supervises" ở bản cũ) — hoàn toàn dựa vào kiểm tra `accountRole` ở tầng Service (xem mục 7, mục 9.4) |
| **User — Owner** | `Account.accountRole = User` + `Relationship.memberRole = Owner` (trong 1 `Family` cụ thể) | Chủ 1 `Family`: quản lý thành viên, tạo `FinancialCategory`, cấu hình `Threshold` | 
| **User — Member** | `Account.accountRole = User` + `Relationship.memberRole = Member` | Thành viên thường; quyền chi tiết theo `Relationship.customPermission` |

Không có phân lớp `Admin extends Person` / `User extends Person` nữa — `Person` giờ là 1 class duy nhất (không có thuộc tính/method riêng cho từng vai trò), vai trò hệ thống (Admin/User) hoàn toàn nằm trên `Account.accountRole`. Vai trò trong 1 gia đình cụ thể (Owner/Member) nằm trên `Relationship.memberRole`, tách biệt với `accountRole`. 1 `Person` vẫn có thể vừa là Owner của `Family` này vừa là Member của nhiều `Family` khác (qua nhiều dòng `Relationship` khác nhau).

## 3. Phạm vi chức năng theo nhóm nghiệp vụ

### 3.1 Auth & Account
- `Account` gắn đúng 1 `Person` (`Person "1" — has — "1" Account`). `accountRole: AccountRole` (Admin/User) và `accountStatus: AccountStatus` (Active/Inactive) nằm trực tiếp trên `Account`.
- `Account.changePassword()` / `lockAccount()` / `unlockAccount()` — đổi mật khẩu, Admin khóa/mở tài khoản vi phạm.
- `Account` hiện thực `Loggable` → login/logout, đăng ký, đổi mật khẩu, khóa/mở đều sinh `Log`.
- Đăng nhập Google, quên mật khẩu qua OTP gmail, quy tắc mật khẩu mạnh (nêu ở `SRS.docx`) **chưa có field/method riêng trên diagram này** — xem mục 7.3 và mục 9.7.

### 3.2 Family & Relationship (thành viên gia đình)
- `Person "1" — creates — "0..*" Family`: 1 Person tạo được nhiều `Family`.
- `Family "1" — has — "1..*" Relationship`: mỗi `Family` có ít nhất 1 thành viên (chính người tạo, với `memberRole = Owner`).
- `Person "1" — belongs to — "0..*" Relationship`: 1 Person tham gia nhiều `Relationship` (ở nhiều `Family` khác nhau).
- `Relationship` (association class giữa `Person` và `Family`): `relationshipId`, `memberRole: MemberRole` (Owner/Member), `customPermission: list<CustomPermission>` — `changeMemberRole()`, `adjustCustomPermission()`.
- **`CustomPermission` mở rộng từ 2 lên 5 giá trị**: `Read`, `Write`, `Approval`, `Share`, `Invitation` — so với bản cũ (chỉ Read/Write), đây là điểm trả lời câu hỏi mở trước đó ("Owner có thể cấp thêm đặc quyền gì khác"); ý nghĩa cụ thể từng quyền mới (`Approval`/`Share`/`Invitation`) cần chốt lại — xem mục 9.6.
- **Owner luôn full quyền, bỏ qua `customPermission`** — quy tắc này vẫn giữ nguyên như bản cũ (không thể hiện được trên diagram, xử lý ở Service).
- `Relationship` **không còn thuộc tính trạng thái riêng** (`memberStatus` như `FamilyMember` bản cũ đã biến mất) — muốn "khóa tạm" 1 thành viên mà không xóa quan hệ thì chưa có chỗ lưu, xem mục 9.3.
- `Family.createFinancialCategory()` — tạo `FinancialCategory` mới trong gia đình. `Family` hiện thực `Loggable`.

### 3.3 FinancialCategory & Threshold
- `Family "1" — has — "0..*" FinancialCategory`. Mỗi category: `financialCategoryId`, `name`, `description`, `initialBalance: BigDecimal` (chi phí gốc ban đầu, không phải số dư hiện tại — vẫn giữ nguyên ý nghĩa như `baseExpense` bản cũ), `categoryStatus: CategoryStatus` (Active/Locked).
- **`Threshold` giờ là 1 entity riêng** (bản cũ là 3 field số cố định `warningThreshold`/`emergencyThreshold`/`approveThreshold` nằm thẳng trên `ExpenseJar`): `FinancialCategory "1" — has — "1..* " Threshold`. Mỗi `Threshold` có `thresholdType: ThresholdType` (Approve/Warning/Emergency), `amount: BigDecimal`, `enabled: boolean` — `modifyThreshold()`.
  - Cách hiểu đề xuất (cần xác nhận, mục 9.1): mỗi `FinancialCategory` khi tạo được seed sẵn 3 dòng `Threshold` (1 dòng/loại), Owner tự bật `enabled` và nhập `amount` cho loại nào cần dùng — khớp với quy tắc cũ "3 ngưỡng đều không bắt buộc".
  - Ý nghĩa từng loại giữ nguyên bản cũ: `Warning` — cảnh báo nhẹ, không khóa; `Emergency` — số dư ≥ ngưỡng → cảnh báo **và tự khóa category** (`categoryStatus = Locked`); `Approve` — giao dịch có `amount` ≥ ngưỡng mới cần Owner duyệt, dưới ngưỡng tự động hợp lệ.
- `FinancialCategory.checkBalance()` — so số dư hiện tại (`initialBalance` trừ tổng `Transaction.amount` đã `Approve`) với `Threshold` loại Warning/Emergency đang `enabled`.
- `lockCategory()` / `unlockCategory()` — khóa/mở category (chỉ Owner mở được, khóa có thể tự động qua `checkBalance()`).
- `syncOfflineTransaction()` — đồng bộ giao dịch tạo lúc mất mạng (xem 3.4).

### 3.4 Giao dịch (Transaction)
- `FinancialCategory "1" — has — "0..*" Transaction`. Mỗi `Transaction`: `content`, `amount: BigDecimal`, `transactionType: TransactionType` (Income/Expense — enum hóa "loại giao dịch tiền vào/tiền ra" từ SRS.docx, trước đây bản cũ chưa đặt tên enum chính thức), `transactionStatus: TransactionStatus` (Draft/Pend/Review/Approve/Reject/Offline), `rawData: String`.
- **`approveTransaction()` / `rejectTransaction()` giờ nằm ngay trên `Transaction`** (bản cũ đặt ở `ExpenseJar` với lý giải rõ ràng "Owner duyệt của hũ, không phải của chính giao dịch") — đây là đảo ngược có chủ đích so với thiết kế trước, cần xác nhận lại (mục 9.5).
- `changeTransactionType()` — cho sửa lại Income/Expense sau khi tạo (ví dụ nhập nhầm loại).
- **Không còn method `changeStatus()` chung** như `TransactionHistory` bản cũ — các bước chuyển trạng thái tự động theo thao tác người dùng (nộp → `Pend`, mở xem → `Review`) cần cài đặt ở tầng Service dựa theo hành vi UI, không có method riêng trên diagram cho việc này.
- Quy tắc rẽ nhánh theo ngưỡng `Approve` **vẫn giữ nguyên tinh thần bản cũ**: `amount < Threshold(Approve).amount` (hoặc `Threshold(Approve)` không `enabled`) → tự động `Approve`; ngược lại → đi đủ `Draft → Pend → Review → Approve/Reject`. `Offline` là trạng thái tạm khi mất mạng, thoát ra bằng `syncOfflineTransaction()`.
- `Transaction` hiện thực `Loggable`.
- **Không còn field nào ghi nhận "ai tạo giao dịch" / "ai đổi trạng thái giao dịch"** như `FamilyMember — create/changes status — TransactionHistory` ở bản cũ — quan hệ `Person`/`Relationship` với `Transaction` **không xuất hiện trực tiếp** trên diagram mới; việc audit "thành viên nào tạo/duyệt giao dịch" chỉ còn suy ra gián tiếp qua `Log` (`Person — has — Log`, với `description` ghi rõ). Đây là điểm mất chi tiết so với bản cũ, cần xác nhận có chủ đích không (mục 9.8).

#### 3.4.1 Về 6 kênh nhập liệu (Manual/OCR/Voice/Message/Announcement/ScanAI)

Toàn bộ cây kế thừa `Record` (`baseRecord`, `confidenceScore`, `recordStatus`, 6 lớp con) ở bản thiết kế trước **không còn xuất hiện** trên class diagram mới — thay vào đó `Transaction` chỉ có 1 field `rawData: String`. Luồng nghiệp vụ chi tiết cho từng kênh nhập liệu (đã mô tả rất kỹ ở `SRS.docx` và bản BUSINESS-REQUIREMENTS.md trước) **chưa biết còn giữ lại hay bị cắt phạm vi** — xem mục 9.2 để xác nhận. Nếu vẫn giữ nghiệp vụ 6 kênh, khả năng cao nó sẽ được xử lý ở tầng Service/DTO (ví dụ 1 enum `sourceType` + `rawData`) thay vì tách entity riêng — nhưng hiện diagram còn thiếu cả field `sourceType`, nên coi đây là gap cần hỏi lại, không tự suy diễn thêm.

### 3.5 Nhật ký (Log)
- `Loggable` — interface được `Account`, `Family`, `Transaction`, `Feedback` hiện thực → trả lời **"cái gì bị đổi"**.
- `Loggable "1" — saves action — "..." Log` — 1 đối tượng `Loggable` có thể phát sinh nhiều `Log` theo thời gian (số nhiều phía `Log`, dù số chính xác trên ảnh khó đọc tuyệt đối — xem ghi chú đầu file).
- **`Person "1" — has — "0..*" Log`** thay cho cặp quan hệ `performs`/`manages` tách biệt ở bản cũ. Nghĩa là: mỗi `Log` gắn với đúng 1 `Person` gây ra hành động đó (giống "performs" cũ); riêng khả năng **Admin xem được Log của mọi Person khác** (giống "manages" cũ) **không còn là 1 association riêng** — phải suy ra bằng quyền hạn (`accountRole = Admin` thì query không giới hạn theo `Person`), xem mục 9.4.
- `Log`: `actionType: ActionType` (Login/Logout/Register/Create/Modify/Delete), `logTime`, `ipAddress`, `description`. Vẫn giữ quy tắc cũ: không thêm `ActionType` riêng cho Approve/Reject/Lock/Unlock, dùng chung `Modify` + `description` mô tả rõ.

### 3.6 Phản hồi (Feedback)
- **`Person "1" — answers — "0..*" Feedback`** thay cho 2 quan hệ tách biệt `User creates Feedback` / `Admin response Feedback` ở bản cũ — hợp lý vì `Person` không còn phân lớp Admin/User, cả 2 vai trò đều là `Person`, phân biệt qua `Account.accountRole` của người tạo `Feedback` đó.
- `Feedback`: `feedbackId`, `content`, `feedbackStatus: FeedbackStatus`, `noteByAdmin: String` (đổi tên từ `result`) — không còn method `handleWithFeedBack()` tường minh trên diagram (có thể gộp vào thao tác đổi `feedbackStatus` + set `noteByAdmin` ở tầng Service).
- **`FeedbackStatus` đổi hẳn giá trị**: `Draft, Submitted, Viewed, Responded` (4 giá trị, khớp đúng với cách diễn đạt "nháp, đã nộp, đã xem, đã phản hồi" trong `SRS.docx`) — **thay thế** bộ giá trị cũ `Draft/Pend/Review/Approve/Reject`. Điều này giải quyết dứt điểm câu hỏi mở ở phiên bản tài liệu trước.
- `Feedback` hiện thực `Loggable`.

## 4. Thực thể nghiệp vụ (Entity Catalog — theo `Document/images/TLCN.drawio (9).png`)

### 4.1 Person
`personId: int`, `name: String`, `gmail: String`, `phone: String`. **Không có method nào trên diagram.** So với bản cũ: mất `address`, `birthDate`, `identificationNumber` (thuộc `User` cũ) và mất method `createFamilySpace()` (giờ chỉ còn là association "creates" tới `Family`, không có method tường minh) — xem mục 9.7.

### 4.2 Account
`accountId: int`, `username: String`, `password: String`, `accountRole: AccountRole`, `accountStatus: AccountStatus` — `changePassword(): void`, `lockAccount(): void`, `unlockAccount(): void`. Quan hệ: `Person "1"——"1" Account` (has). Hiện thực `Loggable`. **Không còn quan hệ "supervises" tới nhiều Account khác** như bản cũ (Admin giám sát Account của người khác giờ hoàn toàn là logic tầng Service dựa vào `accountRole`, không phải association trên diagram).

### 4.3 Relationship (association class Person–Family)
`relationshipId: int`, `memberRole: MemberRole`, `customPermission: list<CustomPermission>` — `changeMemberRole(): void`, `adjustCustomPermission(): void`. Quan hệ: `Person "1"——"0..*" Relationship` (belongs to); `Family "1"——"1..*" Relationship` (has). Ràng buộc nghiệp vụ giữ nguyên tinh thần bản cũ (không thể hiện trên diagram): đúng 1 `memberRole = Owner`/Family tại mọi thời điểm; Owner full quyền bỏ qua `customPermission`.

### 4.4 Family
`familyId: int`, `name: String` — `createFinancialCategory(): void`. Hiện thực `Loggable`. Quan hệ: `Person "1"——"0..*" Family` (creates); `Family "1"——"0..*" FinancialCategory` (has).

### 4.5 FinancialCategory
`financialCategoryId: int`, `name: String`, `description: String`, `initialBalance: BigDecimal`, `categoryStatus: CategoryStatus` — `lockCategory(): void`, `unlockCategory(): void`, `checkBalance(): void`, `syncOfflineTransaction(): void`. Quan hệ: `Family "1"——"0..*" FinancialCategory` (has); `FinancialCategory "1"——"1..*" Threshold` (has); `FinancialCategory "1"——"0..*" Transaction` (has). *(Không hiện thực `Loggable` — giống bản cũ, `ExpenseJar` trước đây cũng không phải Loggable.)*

### 4.6 Threshold
`thresholdType: ThresholdType`, `amount: BigDecimal`, `enabled: boolean` — `modifyThreshold(): void`. Không có khóa chính riêng thể hiện trên diagram (có thể ngầm định `relationshipId`-style composite hoặc tự sinh `thresholdId` khi code — cần bổ sung khi tạo entity thật).

### 4.7 Transaction
`transactionId: int`, `content: String`, `amount: BigDecimal`, `transactionType: TransactionType`, `transactionStatus: TransactionStatus`, `rawData: String` — `approveTransaction(): void`, `rejectTransaction(): void`, `changeTransactionType(): void`. Hiện thực `Loggable`. Quan hệ: `FinancialCategory "1"——"0..*" Transaction` (has). **Không có quan hệ trực tiếp với `Person`/`Relationship`** (mất "ai tạo/ai duyệt" tường minh so với bản cũ — xem 3.4, mục 9.8).

### 4.8 Log
`logId: int`, `description: String`, `actionType: ActionType`, `logTime: Date`, `ipAddress: String`. Quan hệ: `Loggable "1"——"..." Log` (saves action — target); `Person "1"——"0..*" Log` (has — actor).

### 4.9 Feedback
`feedbackId: int`, `content: String`, `feedbackStatus: FeedbackStatus`, `noteByAdmin: String`. Hiện thực `Loggable`. Quan hệ: `Person "1"——"0..*" Feedback` (answers).

## 5. Enum / trạng thái

| Enum | Giá trị | Dùng ở | Ghi chú thay đổi so với bản trước |
|---|---|---|---|
| `AccountRole` | Admin, User | `Account.accountRole` | **Mới** — thay cho việc `Admin`/`User` là 2 phân lớp kế thừa của `Person` |
| `AccountStatus` | Active, Inactive | `Account.accountStatus` | Đổi tên từ `Status` |
| `MemberRole` | Owner, Member | `Relationship.memberRole` | Đổi tên từ `Role`, giá trị giữ nguyên |
| `CustomPermission` | Read, Write, Approval, Share, Invitation | `Relationship.customPermission` | Đổi tên từ `Permission`, **mở rộng từ 2 lên 5 giá trị** — cần chốt ý nghĩa Approval/Share/Invitation (mục 9.6) |
| `CategoryStatus` | Active, Locked | `FinancialCategory.categoryStatus` | Đổi tên từ `JarStatus`, giá trị giữ nguyên |
| `ThresholdType` | Approve, Warning, Emergency | `Threshold.thresholdType` | **Mới** — thay cho 3 field cố định `approveThreshold`/`warningThreshold`/`emergencyThreshold` |
| `TransactionStatus` | Draft, Pend, Review, Approve, Reject, Offline | `Transaction.transactionStatus` | Giữ nguyên |
| `TransactionType` | Income, Expense | `Transaction.transactionType` | **Mới** — enum hóa chính thức "tiền vào/tiền ra" |
| `FeedbackStatus` | Draft, Submitted, Viewed, Responded | `Feedback.feedbackStatus` | **Đổi hẳn giá trị** (bản cũ: Draft/Pend/Review/Approve/Reject) |
| `ActionType` | Login, Logout, Register, Create, Modify, Delete | `Log.actionType` | Giữ nguyên |
| ~~`RecordStatus`~~ | — | — | **Bị xóa** cùng với cây `Record` (xem 3.4.1) |

## 6. Ma trận quyền theo vai trò

| Hành động | Owner (Relationship.memberRole) | Member — có `Write`/`Approval`/`Share`/`Invitation` | Member — chỉ `Read` | Admin (Account.accountRole) |
|---|:---:|:---:|:---:|:---:|
| Tạo Family | ✅ (tự động thành Owner qua Relationship) | — | — | — |
| Thêm/xóa thành viên (Relationship) | ✅ | Tùy quyền `Invitation`/`Share` (mục 9.6) | ❌ | — |
| Tạo FinancialCategory, cấu hình Threshold | ✅ | ❌ | ❌ | — |
| Tạo Transaction | ✅ (full quyền) | ✅ (cần `Write`) | ❌ | — |
| Duyệt/từ chối Transaction (`approveTransaction()`/`rejectTransaction()`) | ✅ | Tùy quyền `Approval` (mục 9.6) | ❌ | — |
| Khóa/mở FinancialCategory | ✅ (mở); khóa có thể tự động qua `checkBalance()` | ❌ | ❌ | — |
| Khóa/mở Account của User khác | — | — | — | ✅ (`accountRole = Admin`, kiểm tra ở Service — mục 9.4) |
| Xem toàn bộ Log hệ thống | — | — | — | ✅ (Service check `accountRole`) |
| Gửi/trả lời Feedback | ✅ (gửi) | ✅ (gửi) | ✅ (gửi) | ✅ (trả lời, set `noteByAdmin`) |

## 7. Lưu ý triển khai (không phải lỗi thiết kế)

1. **Tiền dùng `BigDecimal`** (`initialBalance`, `Transaction.amount`, `Threshold.amount`) — diagram ghi `BigDemical` (lỗi chính tả, hiểu là `BigDecimal`). Chuẩn hóa `setScale(3, RoundingMode.HALF_UP)` khi nhận input, so sánh bằng `compareTo()`, cộng/trừ dùng `add()`/`subtract()` xuyên suốt, không ép `double`. Cột DB: `DECIMAL(15,3)`.
2. **Các ràng buộc nghiệp vụ chỉ có thể chốt bằng lời, không thể hiện trên class diagram** (đúng 1 `memberRole = Owner`/Family tại mọi thời điểm; Owner full quyền bỏ qua `customPermission`; quy tắc rẽ nhánh theo `Threshold(Approve)`; `Admin` xem toàn bộ Log/khóa mọi Account chỉ dựa vào `accountRole`, không phải association) — bắt buộc validate ở tầng Service khi code.
3. **`Threshold` cần khóa chính riêng** (`thresholdId`) khi tạo entity thật — diagram chưa liệt kê attribute này.
4. **Đăng nhập Google, quên mật khẩu OTP, quy tắc mật khẩu mạnh, sửa hồ sơ cá nhân, báo cáo chi tiêu theo thời gian/phạm vi** (đã nêu ở `SRS.docx`, xem bản BUSINESS-REQUIREMENTS.md trước) — **chưa có field/method/entity nào trên class diagram mới này**. Cần bổ sung lên diagram trước khi code các luồng đó, hoặc xác nhận đã tạm gác lại ngoài phạm vi đợt này.
5. **6 kênh nhập liệu giao dịch (OCR/Voice/Message/Announcement/ScanAI) và audit "ai tạo/ai duyệt giao dịch"** — đã có trong SRS.docx/bản tài liệu trước nhưng **không còn trên diagram mới** (xem 3.4.1, 3.4, mục 9.2, 9.8). Cần xác nhận rõ trước khi code tầng nhập liệu giao dịch, tránh code thiếu so với ý định thật.

## 8. Trạng thái tổng thể của vòng tái cấu trúc này

Đây là 1 vòng **tái cấu trúc lớn** (không phải chỉnh sửa nhỏ), giải quyết đúng 1 câu hỏi mở từ bản tài liệu trước (mở rộng `Permission`/`CustomPermission`) và 1 phần câu hỏi mở khác (`FeedbackStatus` đổi giá trị khớp SRS.docx). Đổi lại, diagram mới làm **mất một số chi tiết nghiệp vụ đã từng được xác nhận kỹ ở bản trước** (6 kênh nhập liệu giao dịch + audit ai tạo/ai duyệt giao dịch + trạng thái tạm khóa 1 thành viên + quan hệ supervises/manages tường minh của Admin). Trước khi dùng diagram này để bắt đầu code entity, nên đi hết mục 9 với người dùng để biết đây là **cắt phạm vi có chủ đích** (hợp lý cho 1 khóa luận, tránh over-engineering) hay **sót khi vẽ lại diagram**.

## 9. Câu hỏi mở / cần xác nhận thêm (phát sinh từ vòng tái cấu trúc này)

1. **`Threshold` (1..* bắt buộc)**: mỗi `FinancialCategory` có tự động seed sẵn 3 dòng `Threshold` (Approve/Warning/Emergency, `enabled=false` mặc định) khi tạo, hay Owner tự thêm dần từng dòng?
2. **6 kênh nhập liệu giao dịch (Manual/OCR/Voice/Message/Announcement/ScanAI)**: có chủ đích bỏ hẳn khỏi phạm vi khóa luận (chỉ còn nhập tay + `rawData` tự do), hay vẫn giữ nghiệp vụ này nhưng chuyển qua xử lý ở tầng Service/DTO (cần bổ sung ít nhất 1 field `sourceType` trên `Transaction` mà diagram hiện chưa có)?
3. **Relationship mất `memberStatus`**: muốn tạm khóa 1 thành viên (không cho thao tác) mà không xóa hẳn quan hệ khỏi gia đình thì xử lý thế nào — thêm lại field trạng thái, hay coi "xóa hết `customPermission`" là đủ để vô hiệu hóa?
4. **Admin không còn association "supervises"/"manages" tường minh**: xác nhận cách hiểu "Admin khóa Account bất kỳ / xem Log bất kỳ hoàn toàn dựa vào kiểm tra `accountRole = Admin` ở tầng Service, không giới hạn theo quan hệ nào khác" là đúng ý.
5. **`approveTransaction()`/`rejectTransaction()` chuyển từ `FinancialCategory` sang `Transaction`**: xác nhận đây là thay đổi chủ đích (Owner duyệt trực tiếp trên từng `Transaction`) so với thiết kế trước (duyệt qua `ExpenseJar`).
6. **Ý nghĩa cụ thể của `Approval`, `Share`, `Invitation`** trong `CustomPermission` (ngoài `Read`/`Write` cũ) — ví dụ: `Approval` = Member cũng được duyệt giao dịch thay Owner? `Share` = được chia sẻ category/family ra ngoài? `Invitation` = được tự mời thêm thành viên mới dù không phải Owner?
7. **`Person` mất `address`, `birthDate`, `identificationNumber`, mất method `createFamilySpace()`**: có chủ đích đơn giản hóa hồ sơ cá nhân (bỏ hẳn các trường này khỏi phạm vi), hay chỉ là chưa kịp vẽ lại lên diagram mới, và các nghiệp vụ "sửa hồ sơ cá nhân"/"đăng nhập Google"/"quên mật khẩu OTP" ở `SRS.docx` vẫn cần các trường này?
8. **Mất quan hệ "ai tạo/ai đổi trạng thái giao dịch"** giữa `Relationship`/`Person` và `Transaction`: có chấp nhận việc audit chuyện này chỉ còn suy ra gián tiếp qua `Log.description`, hay cần thêm lại 1 quan hệ trực tiếp như bản cũ để truy vấn nhanh hơn (không phải parse text trong Log)?

---

*Dựa trên `Document/images/TLCN.drawio (9).png` (2026-09-26) và `Document/SRS.docx` (2026-09-26). `Document/ClassDiagram.eapx` (Enterprise Architect, binary) không đối chiếu được bằng công cụ text.*

*Lịch sử các bản trước (giữ lại để tham khảo, không còn là nguồn chốt hiện hành):*
- *2026-09-06 — chốt bản `FinanceManagement.png` lần đầu: `Account` có `lockAccount()`/`unlockAccount()`, `jarStatus` đúng kiểu giá trị đơn, tiền đổi từ `double` sang `BigDecimal` (scale 3).*
- *2026-09-11 — bổ sung `Record.createdTime`/`confidenceScore`/`recordStatus` và luồng nghiệp vụ chi tiết cho 6 kênh nhập liệu.*
- *2026-09-26 (sáng) — đọc `Document/SRS.docx`, bổ sung Báo cáo & Thống kê, Hồ sơ cá nhân, đăng nhập Google, quên mật khẩu OTP.*
- *2026-09-26 (bản này) — đọc class diagram mới `TLCN.drawio (9).png`: tái cấu trúc lớn, bỏ phân lớp Admin/User, bỏ cây `Record`, tách `Threshold` thành entity riêng, đổi tên hàng loạt class/enum, mở rộng `CustomPermission`, đổi giá trị `FeedbackStatus`. Các nghiệp vụ mới thêm sáng nay (Google login/OTP/báo cáo/hồ sơ cá nhân) **chưa được đưa lên diagram này** — vẫn còn là gap, xem mục 7.4 và mục 9.7.*
