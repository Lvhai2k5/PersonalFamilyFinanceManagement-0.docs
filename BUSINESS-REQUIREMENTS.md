# FinanceManagement — Đặc tả nghiệp vụ (Business Requirements)

> Nguồn: `Document/FinanceManagement.png` (class diagram hiện hành, đã qua nhiều vòng chỉnh sửa cùng người dùng — bản đọc gần nhất). `Document/ClassDiagram.eapx` là file dự án Enterprise Architect (binary, chỉ mở được bằng EA) mô tả cùng thiết kế, không đối chiếu được bằng công cụ text. Các file từng dùng làm nguồn ở bản rất đầu (`FinalClassDiagram.drawio`, `FamJar-Architecture.html`) **không còn tồn tại trong repo** — không dùng lại nội dung của chúng.
>
> Tài liệu này là bản **chốt sau nhiều vòng trao đổi nghiệp vụ trực tiếp** với người dùng (không chỉ suy ra từ hình vẽ) — mỗi quy tắc dưới đây đều đã được xác nhận rõ. Mục 7 chỉ còn 2 lưu ý triển khai (không phải lỗi thiết kế) — toàn bộ điểm lệch giữa diagram và nghiệp vụ đã chốt từ các vòng review trước đó **đã được sửa hết** trên diagram mới nhất.

---

## 1. Giới thiệu

**FinanceManagement** — ứng dụng quản lý tài chính gia đình (package gốc `ute.fit.financemanagement`, xem [README.md](README.md)). Một `User` tạo `FamilySpace`, quản lý thành viên qua `FamilyMember` (đúng 1 `Owner` duy nhất tại mọi thời điểm, còn lại là `Member`). Tiền được chia vào nhiều `ExpenseJar`; mỗi hũ có thể được Owner tự cấu hình (không bắt buộc) 3 ngưỡng số: `warningThreshold` (chỉ cảnh báo), `emergencyThreshold` (cảnh báo **và tự khóa hũ**), `approveThreshold` (từ mức này giao dịch phải chờ Owner duyệt, dưới mức tự động hợp lệ). Giao dịch (`TransactionHistory`) được nhập qua nhiều kênh (`Record`: Manual/OCR/Voice/Message/Announcement/ScanAI), luôn biết rõ **thành viên nào tạo ra** và **thành viên nào đã đổi trạng thái** nó. `Admin` giám sát tài khoản (`Account`) và xử lý `Feedback`. Mọi hành động quan trọng đều được ghi `Log`, biết rõ cả **"cái gì bị đổi"** (`Loggable`: Account/FamilySpace/TransactionHistory/Feedback) lẫn **"ai gây ra"** (`Person`).

## 2. Đối tượng người dùng (Actor)

| Actor | Vai trò | Ghi chú |
|---|---|---|
| **Admin** | Giám sát tài khoản (`Account`), xử lý `Feedback`, quản lý `Log` | Kế thừa `Person`; có `Account` riêng của chính mình (qua `Person`–`Account`), **đồng thời** giám sát được nhiều `Account` khác (`Admin`–`supervises`–`Account`) |
| **User — Owner** | Chủ `FamilySpace`: quản lý thành viên, tạo `ExpenseJar`, cấu hình ngưỡng, duyệt/khóa/mở hũ | `FamilyMember.role = Owner`; **đúng 1 Owner/FamilySpace tại mọi thời điểm**, full quyền bất kể `permissions` |
| **User — Member** | Thành viên thường, tạo giao dịch | `FamilyMember.role = Member`; quyền chi tiết theo `permissions: List<Permission>` (Read/Write) |

Không có cấp bậc trung gian (không có CoOwner). Owner **không được tự rời nhóm** khi đang là Owner duy nhất — phải `changeRole()` chuyển 1 Member khác thành Owner mới trước, sau đó mới được rời/lùi xuống Member (không cần method `transferOwnership()` riêng — `changeRole()` sẵn có đã đủ, theo xác nhận của người dùng).

## 3. Phạm vi chức năng theo nhóm nghiệp vụ

### 3.1 Auth & Account
- Đăng ký/đăng nhập cho `User` và `Admin`; `Account.changePassword()`.
- `Account` có 1 quan hệ 1–1 duy nhất **"has"** với `Person` (lớp cha của `Admin`/`User`) — 1 Account gắn đúng 1 Person, không tách 2 association riêng cho User/Admin.
- **`Admin "1" —— "0..*" Account`, nhãn "supervises"**: Admin giám sát/khóa được nhiều Account (của User khác), **tách biệt** với quan hệ `Person`–`Account` (đó là tài khoản của chính Admin để tự đăng nhập). 2 quan hệ không thay thế nhau.
- `Account.lockAccount()` / `unlockAccount()` — method thực thi việc Admin khóa/mở tài khoản vi phạm, đi kèm `changePassword()` sẵn có.
- `Account` hiện thực `Loggable` → mọi hành động trên Account (login/logout, đổi mật khẩu, bị khóa/mở...) đều sinh `Log`, áp dụng cho cả Admin lẫn User.

### 3.2 FamilySpace & Thành viên
- `User.createFamilySpace()` — 1 User tạo được nhiều `FamilySpace`, tự động thành `Owner` của space đó.
- Quan hệ thành viên qua `FamilyMember` (association class, không lưu trực tiếp trên `User`/`FamilySpace`): `User "1" — participates in — "0..*" FamilyMember`; `FamilyMember "1..*" — belongs to — "1" FamilySpace`.
- `FamilyMember.changeMemberStatus()` / `changePermission()` / `changeRole()` quản lý `memberStatus` (Active/Inactive), `permissions` (Read/Write) và `role` (Owner/Member).
- **Owner luôn full quyền**, bỏ qua `permissions` — thuộc tính này chỉ có tác dụng hạn chế với Member.
- `FamilySpace` hiện thực `Loggable` → tạo/xóa thành viên, đổi thông tin FamilySpace đều được ghi Log.

### 3.3 ExpenseJar & Threshold (3 ngưỡng độc lập)
- `FamilySpace.createExpenseJar()` — 1 FamilySpace có 0..* `ExpenseJar`.
- `baseExpense: BigDecimal` — **chi phí gốc (ngân sách ban đầu) của hũ**, do Owner thiết lập khi tạo hũ; không phải số dư hiện tại. Số dư hiện tại của hũ = `baseExpense` trừ tổng các `TransactionHistory.expense` đã ở trạng thái `Approve` — đây là giá trị `checkBalance()` dùng để so sánh với `warningThreshold`/`emergencyThreshold`.
- 3 ngưỡng dạng số (`BigDecimal`), **đều do Owner tự cấu hình, không bắt buộc**:
  - `warningThreshold` — cảnh báo nhẹ, **không khóa hũ**.
  - `emergencyThreshold` — số dư **≥ ngưỡng này** → cảnh báo **và tự động khóa hũ** (`lockJar()`), chỉ Owner mới `unlockJar()` được.
  - `approveThreshold` — giao dịch có `expense` **≥ ngưỡng này** mới cần Owner duyệt; **dưới ngưỡng thì tự động hợp lệ**, không cần ai duyệt.
- `jarStatus: JarStatus` (enum `JarStatus`: Active/Locked, 1 giá trị đơn) — biểu diễn **trạng thái thao tác hiện tại của chính cái hũ** (Active = cho Read+Write bình thường, Locked = chỉ Read, chặn tạo giao dịch mới), đi theo `lockJar()`/`unlockJar()`. **Đây không phải phân quyền theo từng thành viên** — khác hoàn toàn với `FamilyMember.permissions` (quyền của 1 thành viên trong gia đình). 2 khái niệm dễ nhầm vì trước đó từng đặt tên là `jarPermission` dùng chung enum `Permission` — đã đổi sang enum `JarStatus` riêng, giá trị đơn (không phải List), cho rõ nghĩa.
- `checkBalance()` — kiểm tra số dư so với `warningThreshold`/`emergencyThreshold`, tự kích hoạt cảnh báo + khóa khi tới ngưỡng khẩn cấp.
- `approveTransaction()` / `rejectTransaction()` — đặt ở `ExpenseJar` (không phải `TransactionHistory`): Owner duyệt/từ chối giao dịch của chính hũ đó, hoàn toàn chủ quan, hệ thống chỉ **đề xuất** cần duyệt khi vượt `approveThreshold`.
- `syncOfflineTransactions()` — đồng bộ các giao dịch được tạo lúc mất mạng (xem 3.4).

### 3.4 Giao dịch & Ghi nhận (TransactionHistory / Record)
- 1 `ExpenseJar` có 0..* `TransactionHistory`. Mỗi `TransactionHistory` có đúng 1 `Record`.
- **`Record.baseRecord: String`** là **dữ liệu thô** — link ảnh hóa đơn (OCR), link file mp3 (Voice), nội dung tin nhắn thô (Message)... lưu ở nơi khác (storage/CDN). `processBaseRecord()` xử lý dữ liệu thô này thành **nội dung đầy đủ**, đổ vào `TransactionHistory.content`.
- 6 lớp con của `Record`, cùng override `processBaseRecord()` theo cách riêng:
  - **Manual** — nhập tay. **OCR** — quét ảnh hóa đơn. **Voice** — ghi âm. **ScanAI** — quét bằng AI.
  - **Message** — giao dịch tạo từ 1 tin nhắn **User chủ động share vào app** từ ứng dụng khác (Zalo, SMS...), ví dụ "mẹ ơi con mua 100k cái áo rồi nha mẹ".
  - **Announcement** — app **chủ động đọc notification** phát sinh trên máy, nhưng **chỉ từ 1 vài app được User cấu hình trước (whitelist)**, không quét tràn lan mọi thông báo.
- **`FamilyMember "1" — create — "0..*" TransactionHistory`**: biết rõ **thành viên nào tạo** giao dịch.
- **`FamilyMember "1" — changes status — "0..*" TransactionHistory`**: biết rõ **thành viên nào (Owner) đã đổi trạng thái** giao dịch — tách riêng khỏi người tạo, phục vụ audit chính xác.
- **Luồng trạng thái `TransactionStatus`, rẽ theo `expense` so với `ExpenseJar.approveThreshold`:**
  - `expense < approveThreshold` (hoặc hũ không cấu hình ngưỡng) → **tự động Approve**, bỏ qua Pend/Review.
  - `expense ≥ approveThreshold` → đi đủ `Draft` (nháp, chưa nộp) → `Pend` (đã nộp, chưa xem) → `Review` (đã xem, chờ xử lý) → `Approve`/`Reject` (Owner quyết định qua `ExpenseJar`).
  - **`Offline`** — trạng thái tạm khi thiết bị mất mạng (lưu ở database cục bộ, không xác định Draft/Pend/Review lúc đó). Khi có mạng lại, `ExpenseJar.syncOfflineTransactions()` áp lại đúng quy tắc trên: dưới ngưỡng → thẳng `Approve`; từ ngưỡng trở lên → về `Pend` chờ duyệt bình thường.
- `TransactionHistory.changeStatus()` — method chung thực hiện các chuyển trạng thái trên; nút quyết định cuối (Approve/Reject khi vượt ngưỡng) vẫn nằm ở `ExpenseJar`.
- `TransactionHistory` hiện thực `Loggable` → mọi thay đổi giao dịch được ghi Log.

#### 3.4.1 Luồng nghiệp vụ chi tiết theo từng hình thức nhập liệu (`Record`)

Nguyên tắc chung cho cả 6 luồng: `processBaseRecord()` chỉ có nhiệm vụ chuẩn hóa dữ liệu thô (`baseRecord`) thành nội dung giao dịch có cấu trúc — **không luồng nào được tự ý đổi `transactionStatus`**. Kết quả xử lý được phản ánh qua `recordStatus` (Success/Failed) và `confidenceScore` trên chính `Record` (xem 4.5), rồi đổ vào **cùng 1 màn hình xác nhận giao dịch** (nơi thành viên xem lại, chỉnh sửa nếu cần) trước khi giao dịch được ghi nhận chính thức; giao dịch chỉ thật sự được tạo (và bắt đầu đi theo luồng trạng thái ở mục 3.4) tại thời điểm thành viên xác nhận. **`Record` chỉ được lưu (persist) cùng lúc với `TransactionHistory` khi thành viên xác nhận** — ở 2 nhánh "bỏ qua hoàn toàn" của Message/Announcement (nêu bên dưới), không có `Record`/`recordStatus` nào được lưu, đó chỉ là kết quả xử lý tạm thời trong bộ nhớ.

- **Manual**: Thành viên mở màn hình tạo giao dịch → tự nhập đầy đủ thông tin (số tiền, hũ chi tiêu, mô tả, ngày phát sinh) → xác nhận tạo → hệ thống kiểm tra tính hợp lệ của thông tin đã nhập → giao dịch được ghi nhận và đi theo luồng trạng thái đã quy định. Không qua bước AI nên `recordStatus` luôn `Success`, `confidenceScore` không áp dụng (không có ý nghĩa).

- **OCR**: Thành viên cung cấp 1 ảnh hóa đơn → hệ thống đọc nội dung chữ trong ảnh → suy ra các thông tin giao dịch (số tiền, ngày, nơi phát sinh) từ nội dung đã đọc được → các thông tin này được điền sẵn vào màn hình xác nhận giao dịch → thành viên xem lại, chỉnh sửa nếu thông tin suy ra chưa đúng → xác nhận tạo → giao dịch được ghi nhận (`recordStatus = Success`, `confidenceScore` phản ánh độ tin cậy đọc được). Nếu hệ thống không đọc được nội dung ảnh hoặc không suy ra được thông tin giao dịch hợp lệ, màn hình xác nhận được mở ở trạng thái trống để thành viên tự nhập (`recordStatus = Failed`); `Record` vẫn được lưu khi thành viên hoàn tất nhập tay và xác nhận, giữ lại dấu vết đã thử qua OCR.

- **Voice**: Thành viên cung cấp 1 đoạn ghi âm → hệ thống chuyển giọng nói thành văn bản → suy ra các thông tin giao dịch (số tiền, danh mục, ghi chú) từ văn bản đã chuyển đổi → điền sẵn vào màn hình xác nhận giao dịch → thành viên xem lại, chỉnh sửa nếu cần → xác nhận tạo → giao dịch được ghi nhận (`recordStatus = Success`). Nếu không nhận diện được giọng nói hoặc không suy ra được thông tin hợp lệ, xử lý tương tự OCR: mở màn hình xác nhận trống (`recordStatus = Failed`), `Record` vẫn được lưu nếu thành viên hoàn tất nhập tay và xác nhận.

- **Message**: Thành viên chủ động chuyển tiếp 1 đoạn tin nhắn (đang trò chuyện ở nơi khác) vào hệ thống → hệ thống phân tích nội dung đoạn tin nhắn đó → suy ra các thông tin giao dịch (số tiền, danh mục gợi ý) → điền sẵn vào màn hình xác nhận giao dịch → thành viên xem lại, chỉnh sửa nếu cần → xác nhận tạo → giao dịch được ghi nhận (`recordStatus = Success`). Nếu nội dung tin nhắn không chứa thông tin giao dịch hợp lệ, hệ thống không tạo giao dịch nháp mà thông báo cho thành viên biết để tự nhập tay — **không lưu `Record`/`recordStatus` nào**, toàn bộ nội dung tin nhắn bị bỏ qua trước khi tới bước lưu.

- **Announcement**: Thành viên cấu hình trước danh sách các nguồn thông báo được phép sử dụng để ghi nhận giao dịch (chỉ những nguồn được chọn mới được xử lý, không xử lý toàn bộ thông báo phát sinh trên thiết bị). Khi có 1 thông báo mới phát sinh từ 1 nguồn đã được cho phép, hệ thống tự động đọc nội dung thông báo đó → phân tích để suy ra các thông tin giao dịch → tạo sẵn 1 giao dịch ở trạng thái nháp kèm thông tin đã suy ra (`recordStatus = Success`), đồng thời báo cho thành viên biết có giao dịch mới cần xem → thành viên mở lại, xem/chỉnh sửa thông tin → xác nhận → giao dịch được ghi nhận chính thức. Nếu nội dung thông báo không suy ra được thông tin giao dịch hợp lệ hoặc đến từ nguồn không có trong danh sách cho phép, hệ thống bỏ qua, không tạo giao dịch — **không lưu `Record`/`recordStatus` nào** ở nhánh này (không có khái niệm `Failed` được lưu lại, chỉ là bị loại ngay khi xử lý).

- **ScanAI**: Thành viên cung cấp 1 ảnh chụp trực tiếp của 1 vật dụng thật (không phải hóa đơn/giấy tờ) → hệ thống nhận diện loại vật dụng trong ảnh → suy ra tên và danh mục chi tiêu tương ứng với vật dụng đã nhận diện → điền sẵn tên và danh mục vào màn hình xác nhận giao dịch, **riêng số tiền luôn để trống** → thành viên tự nhập số tiền, xem lại/chỉnh sửa tên và danh mục nếu cần → xác nhận tạo → giao dịch được ghi nhận (`recordStatus = Success`). Nếu hệ thống không nhận diện được vật dụng, các trường tên/danh mục cũng được để trống hoàn toàn cho thành viên tự nhập, không hiển thị kết quả nhận diện chưa xác định (`recordStatus = Failed`); `Record` vẫn được lưu nếu thành viên hoàn tất nhập tay và xác nhận.

### 3.5 Nhật ký (Log) — tách rõ "cái gì" và "ai"
- **`Loggable "1" — save actions — "1..*" Log`**: trả lời **"cái gì bị đổi"** — `Loggable` là interface được `Account`, `FamilySpace`, `TransactionHistory`, `Feedback` cùng hiện thực.
- **`Person "1" — performs — "0..*" Log`**: trả lời **"ai gây ra hành động"** — dùng `Person` (không tách riêng User/Admin) vì actor có thể là 1 trong 2, tận dụng lại quan hệ kế thừa sẵn có.
- **`Admin "1" — manages — "0..*" Log`**: Admin có quyền quản lý/tra cứu toàn bộ Log — quan hệ này **khác** với "performs" (performs = ai gây ra hành động cụ thể đó; manages = Admin có quyền xem/quản lý mọi log, kể cả log không phải do chính Admin đó gây ra). Không trùng lặp.
- `Log` gồm `actionType: ActionType` (Login/Logout/Register/Create/Modify/Delete), `logTime`, `ipAddress`, `description`.
- **Không thêm giá trị `ActionType` riêng cho Approve/Reject/Lock/Unlock** — dùng chung `actionType = Modify`, nhưng **bắt buộc điền `description`** rõ ràng để phân biệt (ví dụ `"Approve transaction #123"`, `"Lock jar #45 do chạm emergencyThreshold"`, `"Unlock jar #45"`, `"Admin lock account #7"`).

### 3.6 Phản hồi (Feedback)
- `User "1" — create — "0..*" Feedback`.
- `Admin "1" — response — "0..*" Feedback`.
- `Feedback.handleWithFeedBack()` — cập nhật `feedbackStatus: FeedbackStatus` (`Draft/Pend/Review/Approve/Reject`) + `result`.
- `Feedback` hiện thực `Loggable` → xử lý feedback cũng được ghi Log (với actor là Admin qua `Person — performs — Log`).

## 4. Thực thể nghiệp vụ (Entity Catalog — theo `Document/FinanceManagement.png`)

### 4.1 Person (abstract)
`address`, `birthDate`, `gmail`, `name`, `phone`. 2 lớp con: **Admin** (không thêm thuộc tính), **User** (+ `identificationNumber: int`, + `createFamilySpace(): void`).

### 4.2 Account
`accountStatus: Status`, `createdTime: Date`, `password: String`, `username: String` — `changePassword(): void`, `lockAccount(): void`, `unlockAccount(): void`. Quan hệ: `Person "1"——"1" Account` (has, định danh/đăng nhập); `Admin "1"——"0..*" Account` (supervises, giám sát). Hiện thực `Loggable`.

### 4.3 FamilySpace / FamilyMember
- **FamilySpace**: `name` — `createExpenseJar(): void`. Hiện thực `Loggable`.
- **FamilyMember**: `memberStatus: Status`, `permissions: List<Permission>`, `role: Role` — `changeMemberStatus()`, `changePermission()`, `changeRole()`. Quan hệ: `User "1"——"0..*" FamilyMember` (participates in); `FamilyMember "1..*"——"1" FamilySpace` (belongs to); `FamilyMember "1"——"0..*" TransactionHistory` (create); `FamilyMember "1"——"0..*" TransactionHistory` (changes status). Ràng buộc: đúng 1 `role=Owner`/FamilySpace tại mọi thời điểm; Owner full quyền, bỏ qua `permissions`.

### 4.4 ExpenseJar
`approveThreshold: BigDecimal`, `baseExpense: BigDecimal` (chi phí gốc/ngân sách ban đầu của hũ, không phải số dư hiện tại), `description: String`, `emergencyThreshold: BigDecimal`, `jarStatus: JarStatus`, `name: String`, `warningThreshold: BigDecimal` — `approveTransaction(): void`, `checkBalance(): void`, `lockJar(): void`, `rejectTransaction(): void`, `syncOfflineTransactions(): void`, `unlockJar(): void`. 3 ngưỡng đều do Owner tự cấu hình, không bắt buộc. Quan hệ: `FamilySpace "1"——"0..*" ExpenseJar` (has); `ExpenseJar "1"——"0..*" TransactionHistory` (has).

### 4.5 TransactionHistory & Record
- **TransactionHistory**: `content: String`, `expense: BigDecimal`, `transactionStatus: TransactionStatus` — `changeStatus(): void`. 1 TransactionHistory có đúng 1 `Record`. Hiện thực `Loggable`. Quan hệ với `FamilyMember`: xem 4.3.
- **Record** (abstract): `baseRecord: String` (dữ liệu thô), `createdTime: Date` (thời điểm dữ liệu thô được ghi nhận), `confidenceScore: int` (độ tin cậy AI suy luận từ `baseRecord`, 0–100; chỉ có ý nghĩa với luồng có bước AI, không áp dụng cho Manual), `recordStatus: RecordStatus` (Success/Failed — `processBaseRecord()` có suy ra được thông tin giao dịch hợp lệ hay không; hệ thống dùng `confidenceScore` so với 1 ngưỡng nội bộ để quyết định giá trị này) — `processBaseRecord(): void` (xử lý thành nội dung đầy đủ). 6 lớp con, không thêm thuộc tính: **Manual**, **OCR**, **Voice**, **Message** (User chủ động share), **Announcement** (app tự đọc theo whitelist), **ScanAI**. Chi tiết ý nghĩa `recordStatus`/`confidenceScore` theo từng luồng — xem 3.4.1.

### 4.6 Log
`actionType: ActionType`, `description`, `ipAddress`, `logTime: Date`. Quan hệ: `Loggable "1"——"1..*" Log` (save actions — target); `Person "1"——"0..*" Log` (performs — actor); `Admin "1"——"0..*" Log` (manages — quyền tra cứu/quản lý).

### 4.7 Feedback
`content: String`, `feedbackStatus: FeedbackStatus`, `result: String` — `handleWithFeedBack(): void`. Hiện thực `Loggable`. Quan hệ: `User "1"——"0..*" Feedback` (create); `Admin "1"——"0..*" Feedback` (response).

## 5. Enum / trạng thái

| Enum | Giá trị | Dùng ở |
|---|---|---|
| `Status` | Active, Inactive | `Account.accountStatus`, `FamilyMember.memberStatus` |
| `Role` | Owner, Member | `FamilyMember.role` — không có CoOwner |
| `Permission` | Read, Write | `FamilyMember.permissions` — quyền **của thành viên** |
| `JarStatus` | Active, Locked | `ExpenseJar.jarStatus` — trạng thái thao tác **của chính hũ** (đi theo lock/unlock), không phải quyền thành viên |
| `TransactionStatus` | Draft, Pend, Review, Approve, Reject, Offline | `TransactionHistory.transactionStatus` |
| `FeedbackStatus` | Draft, Pend, Review, Approve, Reject | `Feedback.feedbackStatus` |
| `ActionType` | Login, Logout, Register, Create, Modify, Delete | `Log.actionType` |
| `RecordStatus` | Success, Failed | `Record.recordStatus` |

`TransactionStatus`: rẽ nhánh theo `approveThreshold` (xem 3.4) — dưới ngưỡng tự Approve, từ ngưỡng trở lên đi đủ Draft→Pend→Review→Approve/Reject; `Offline` là trạng thái tạm ngoài luồng, thoát ra bằng `syncOfflineTransactions()` áp lại đúng quy tắc ngưỡng. `FeedbackStatus` dùng chung khuôn Draft→Pend→Review→Approve/Reject, không có Offline. `RecordStatus` là kết quả xử lý `baseRecord` (xem 3.4.1) — **không ảnh hưởng `transactionStatus`**, chỉ phục vụ audit/UX (biết luồng AI có suy luận thành công hay không); với Manual luôn `Success` vì không có bước AI; với Message/Announcement, nhánh thất bại đặc biệt (nội dung không hợp lệ / nguồn ngoài whitelist) không tạo `Record` nào cả nên `Failed` không bao giờ thực sự được lưu ở 2 luồng này.

## 6. Ma trận quyền theo vai trò

| Hành động | Owner | Member (permission Write) | Member (chỉ Read) |
|---|:---:|:---:|:---:|
| Tạo FamilySpace | ✅ (tự động thành Owner) | — | — |
| Thêm/xóa thành viên | ✅ | ❌ | ❌ |
| Chuyển giao quyền Owner (`changeRole()`) | ✅ (bắt buộc trước khi rời nhóm) | ❌ | ❌ |
| Tạo ExpenseJar, cấu hình 3 ngưỡng | ✅ | ❌ | ❌ |
| Tạo giao dịch (TransactionHistory) | ✅ (full quyền) | ✅ | ❌ |
| Giao dịch dưới `approveThreshold` | Tự động Approve | Tự động Approve | ❌ (không tạo được) |
| Duyệt/từ chối giao dịch từ `approveThreshold` trở lên | ✅ (chỉ Owner) | ❌ | ❌ |
| Khóa/mở hũ (`lockJar()`/`unlockJar()`) | ✅ (chỉ Owner mở; khóa có thể tự động qua `checkBalance()`) | ❌ | ❌ |
| Khóa/mở tài khoản User khác (`Account.lockAccount()`/`unlockAccount()`) | Admin (không phải Owner) | — | — |
| Gửi Feedback | ✅ | ✅ | ✅ |

## 7. Lưu ý triển khai (không phải lỗi thiết kế)

1. **Tiền dùng `BigDecimal`, scale cố định 3 chữ số thập phân** (`baseExpense`, `expense`, 3 ngưỡng) — đã đúng kiểu trên diagram. Khi code: chuẩn hóa qua `setScale(3, RoundingMode.HALF_UP)` ngay khi nhận input, so sánh bằng `compareTo()` (không dùng `==`/`equals()`), cộng/trừ dùng `add()`/`subtract()` của `BigDecimal` xuyên suốt — không ép về `double` giữa chừng. Cột DB tương ứng: `DECIMAL(15,3)`.
2. **Các ràng buộc nghiệp vụ đã chốt bằng lời nhưng không thể hiện được trên class diagram** (đúng 1 `role=Owner`/FamilySpace tại mọi thời điểm, whitelist app cho `Announcement`, quy tắc rẽ nhánh `approveThreshold`, `baseExpense` là ngân sách gốc chứ không phải số dư hiện tại...) — đây là giới hạn tự nhiên của UML class diagram (không phải state machine/OCL), **bắt buộc phải validate/tính toán ở tầng Service khi code**, không thể trông chờ diagram tự enforce.

Không còn phát hiện mâu thuẫn nghiệp vụ hay lỗi mô hình nào khác. Toàn bộ vấn đề lớn từng nêu qua các vòng review trước — CoOwner, Threshold ngược tên, Permission lẫn Status, Log thiếu actor, Feedback thiếu Admin, TransactionHistory thiếu người tạo/người duyệt, `jarPermission` mơ hồ, Account thiếu method khóa/mở, `jarStatus` khai báo sai kiểu (List thay vì giá trị đơn), tiền dùng `double` — **đều đã được sửa đúng** trên diagram mới nhất.

## 8. Chấm điểm tổng thể (bản class diagram hiện tại)

| Tiêu chí | Điểm /10 | Lý do |
|---|:---:|---|
| Tính đầy đủ nghiệp vụ | **9.5** | Mọi luồng chính (threshold, approve, lock/unlock jar, khóa/mở account, offline sync, audit actor, ai tạo/ai duyệt giao dịch) đều đầy đủ |
| Chuẩn OOP | **9** | `FamilyMember` là association class mẫu mực; `Loggable` tách target/actor rõ ràng; `jarStatus` đúng kiểu giá trị đơn; tiền dùng đúng `BigDecimal` |
| Nhất quán / convention | **9** | Đã sửa hết lỗi đặt tên/cú pháp/kiểu dữ liệu phát hiện qua nhiều vòng (`jarPermission`→`jarStatus`, hết lỗi `()()`, hết List sai chỗ, `double`→`BigDecimal`) |
| Mức độ giải quyết vấn đề đã đặt ra qua các lần review | **10** | Toàn bộ gap từng nêu qua các vòng đều đã được xử lý đúng và có chủ đích |
| **Tổng thể** | **9.5/10** | Thiết kế đã hoàn thiện, đủ điều kiện dùng làm nguồn chốt để bắt đầu code entity. Phần còn lại để tiến gần 10/10 nằm ngoài class diagram: bổ sung State Machine Diagram cho `TransactionStatus`, Sequence Diagram cho luồng duyệt theo ngưỡng và luồng đồng bộ Offline, và note constraint tường minh cho các ràng buộc ở mục 7.2 |

---

*Dựa trên `Document/FinanceManagement.png` (đọc lần cập nhật gần nhất). `Document/ClassDiagram.eapx` (Enterprise Architect, binary) mô tả cùng thiết kế nhưng không đối chiếu được bằng công cụ text. Cập nhật: 2026-09-06 — chốt bản cuối: `Account` đã có `lockAccount()`/`unlockAccount()`, `jarStatus` đã đúng kiểu giá trị đơn (`JarStatus`), toàn bộ tiền đã đổi từ `double` sang `BigDecimal` (scale 3), `baseExpense` được làm rõ là ngân sách gốc của hũ (không phải số dư hiện tại). Toàn bộ các vấn đề nêu qua các vòng review đã được giải quyết — tài liệu này là bản chốt để bắt đầu code entity.*

*Cập nhật: 2026-09-11 — đồng bộ với diagram mới nhất: `Record` bổ sung `createdTime`, `confidenceScore`, `recordStatus: RecordStatus` (enum mới `RecordStatus {Success, Failed}`); mục 3.4.1 bổ sung ý nghĩa `recordStatus`/`confidenceScore` cho từng luồng nhập liệu, kể cả 2 nhánh Message/Announcement không lưu `Record` khi bị loại ngay lúc xử lý.*
