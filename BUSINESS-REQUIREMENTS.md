# FinanceManagement — Đặc tả nghiệp vụ (Business Requirements)

> Nguồn: `Document/images/ClassDiagram_0310206.png` — class diagram **hiện hành** (2026-10-03, bản cập nhật có `amountOfApproval`). Đây là **bản viết lại hoàn toàn** so với các bản trước: dùng `Membership` (thay `Relationship`), 3 ngưỡng nằm thẳng trên `FinancialCategory` (không còn entity `Threshold`), `SupportTicket` (thay `Feedback`), `ActivityLog` (thay `Log` + `Loggable`), `Person` có thêm `avatar`.
>
> Các quy tắc dưới đây đã được **người dùng chốt trực tiếp (2026-10-03)**: xem mục 9 (bảng "Đã chốt" và phần còn cần xác nhận). `Document/SRS.docx` **tạm thời không dùng** làm nguồn. Tài liệu Word tương ứng: `Document/BusinessRequirements_FinanceManagement_v2.docx`; luồng theo vai trò: `Document/LuongNghiepVu_Admin_Owner_Member.docx`.

---

## 1. Giới thiệu

**FinanceManagement** — ứng dụng quản lý tài chính gia đình (package gốc `ute.fit.financemanagement`, xem [README.md](README.md)). Một `Person` có đúng 1 `Account` (mang `accountRole`: Admin hoặc User). `Person` tham gia nhiều `Family` qua `Membership` (mang `familyMemberRole`: Owner/Member và `permissions`: Read/Write/Approval). Mỗi `Family` có nhiều `FinancialCategory` (danh mục tài chính) với số dư ban đầu và **3 ngưỡng**: `amountOfWarning` (cảnh báo), `amountOfEmergency` (khẩn cấp), `amountOfApproval` (mức phải duyệt). Mỗi danh mục có nhiều `Transaction` (thu/chi): giao dịch dưới `amountOfApproval` **tự động được duyệt**, từ mức đó trở lên phải đi qua luồng duyệt. Giao dịch có thể nhập bằng 6 hình thức (Manual, OCR, Voice, Message, Announcement, ScanAI), dữ liệu gốc lưu ở `rawData`. Người dùng gửi `SupportTicket` để được hỗ trợ, Admin phản hồi. Các hành động quan trọng được ghi vào `ActivityLog`.

## 2. Đối tượng người dùng (Actor)

| Actor | Xác định bằng | Vai trò |
|---|---|---|
| **Admin** | `Account.accountRole = Admin` | Khóa/mở `Account` người dùng, tiếp nhận và phản hồi `SupportTicket`, tra cứu toàn bộ `ActivityLog` |
| **User — Owner** | `accountRole = User` + `Membership.familyMemberRole = Owner` (trong 1 `Family` cụ thể) | Chủ hộ: quản lý thành viên và quyền, tạo/cấu hình/khóa/mở `FinancialCategory`, duyệt giao dịch, chia sẻ quyền duyệt và mở khóa danh mục |
| **User — Member** | `accountRole = User` + `Membership.familyMemberRole = Member` | Thành viên thường; quyền chi tiết theo `Membership.permissions` (Read/Write/Approval) do Owner cấp |

- Vai trò hệ thống (Admin/User) nằm ở `Account.accountRole`, áp dụng toàn hệ thống. Vai trò trong gia đình (Owner/Member) nằm ở `Membership.familyMemberRole`, chỉ có hiệu lực trong từng `Family`. 1 `Person` có thể là Owner của gia đình này và Member của gia đình khác.
- Mỗi `Family` có **đúng 1 Owner tại mọi thời điểm**. Owner không được tự rời nhóm khi còn là Owner duy nhất — phải chuyển vai trò Owner cho 1 Member khác (`changeFamilyMemberRole()`) trước.

## 3. Phạm vi chức năng theo nhóm nghiệp vụ

### 3.1 Tài khoản & Người dùng (Account / Person)
- `Person "1" — has — "1" Account` (quan hệ 1–1).
- `Account.accountRole: AccountRole` (Admin/User), `accountStatus: AccountStatus` (Active/Inactive).
- Đăng ký bằng `username`, `password` và thông tin cá nhân (`name`, `gmail`, `phone`, `avatar`).
- Chỉ `Account` đang `Active` mới đăng nhập được; `Inactive` bị từ chối đăng nhập.
- `Account.changePassword()` — người dùng đổi mật khẩu của chính mình; mật khẩu lưu dạng đã mã hóa.
- `Account.lockAccount()` / `unlockAccount()` — Admin khóa (→ Inactive) / mở (→ Active) tài khoản vi phạm.
- Đăng ký, đăng nhập, đăng xuất, đổi mật khẩu, khóa/mở đều sinh `ActivityLog` (kể cả khi thất bại, ví dụ đăng nhập sai mật khẩu → `Login`/`Failed`).

### 3.2 Gia đình & Thành viên (Family / Membership)
- `Person` — `Family`: nhiều-nhiều, hiện thực qua `Membership` (`Person "1" — "0..*" Membership`; diagram vẽ `Person "1..*" — has — "0..*" Family`, nghĩa là 1 Family có ít nhất 1 Person).
- Người tạo `Family` tự động có 1 `Membership` với `familyMemberRole = Owner`.
- `Membership`: `membershipId`, `familyId`, `familyMemberRole: FamilyMemberRole` (Owner/Member), `permissions: List<FamilyPermission>` — `changeFamilyMemberRole()`, `adjustPermission()`.
- `FamilyPermission`:
  - **Read** — xem dữ liệu gia đình (danh mục, giao dịch).
  - **Write** — tạo/sửa giao dịch của mình.
  - **Approval** — duyệt/từ chối giao dịch **và mở khóa danh mục**. **Mặc định chỉ Owner có**; Member chỉ có khi Owner chủ động chia sẻ (`adjustPermission()`).
- **Owner luôn full quyền, bỏ qua `permissions`**; `permissions` chỉ hạn chế Member.
- **Tạm ẩn thành viên**: khi `permissions` của 1 Member **rỗng (không còn quyền nào)**, thành viên đó bị **tạm ẩn** — không xem/thao tác được gì trong gia đình đó. `Membership` **vẫn được giữ, không bị xóa**; Owner cấp lại quyền (`adjustPermission()`) để khôi phục. Không áp dụng cho Owner.
- Chỉ Owner thêm/xóa thành viên, đổi vai trò, chỉnh quyền.
- `Family.createFinancialCategory()` — tạo danh mục tài chính trong gia đình (chỉ Owner).
- Tạo `Family`, thêm/xóa thành viên, đổi vai trò/quyền đều ghi `ActivityLog`.

### 3.3 Danh mục tài chính & 3 ngưỡng (FinancialCategory)
- `Family "1" — has — "0..*" FinancialCategory`.
- Thuộc tính: `financialCategoryId`, `name`, `description`, `initialBalance` (số dư ban đầu — không phải số dư hiện tại), `categoryStatus: CategoryStatus` (Active/Locked), `amountOfWarning`, `amountOfEmergency`, `amountOfApproval`.
- **Số tiền đã sử dụng** của danh mục = tổng `Transaction.amount` loại `Expense` đã `Approve`. **Số dư hiện tại** = `initialBalance` + tổng `Income` đã `Approve` − tổng `Expense` đã `Approve`. Giao dịch chưa `Approve` không được tính.
- **3 ngưỡng là bắt buộc**: Owner phải nhập đủ cả 3 khi tạo danh mục (và chỉnh sau bằng `modifyThreshold()`), nên không có trường hợp danh mục thiếu ngưỡng:

| Ngưỡng | Xét trên | Điều kiện | Hệ quả |
|---|---|---|---|
| `amountOfWarning` | **Cả danh mục** — số tiền đã sử dụng | đã sử dụng ≥ ngưỡng | **Chỉ push thông báo cảnh báo tới toàn bộ thành viên gia đình**, **không khóa** |
| `amountOfEmergency` | **Cả danh mục** — số tiền đã sử dụng | đã sử dụng ≥ ngưỡng | **Push thông báo tới toàn bộ thành viên gia đình và tự động khóa danh mục** (`Locked`) |
| `amountOfApproval` | **Từng giao dịch** — `Transaction.amount` | amount ≥ ngưỡng | Giao dịch **phải chờ duyệt**; dưới ngưỡng thì **tự động được duyệt** (xem 3.4) |

- `checkBalance()` — chạy sau mỗi lần 1 giao dịch `Expense` chuyển sang `Approve`: so số tiền đã sử dụng của danh mục với `amountOfWarning` / `amountOfEmergency` để gửi push và/hoặc khóa.
- `lockCategory()` / `unlockCategory()` — khóa tự động khi chạm `amountOfEmergency`; Owner có thể khóa thủ công bất kỳ lúc nào. **Owner hoặc Member có quyền `Approval` mở khóa** (chốt 2026-10-14). Danh mục `Locked`: **không tạo được giao dịch mới nhưng vẫn xem được**. Nếu số tiền đã sử dụng vẫn ≥ `amountOfEmergency` thì cần nâng ngưỡng (`modifyThreshold()`, **chỉ Owner**) trước khi mở khóa, nếu không lần `Approve` kế tiếp sẽ khóa lại.
- `syncOfflineTransaction()` — đồng bộ giao dịch tạo lúc mất mạng (xem 3.4).
- Tạo/sửa ngưỡng/khóa/mở danh mục đều ghi `ActivityLog`.

### 3.4 Giao dịch (Transaction)
- `FinancialCategory "1" — has — "0..*" Transaction`.
- Thuộc tính: `transactionId`, `content`, `amount: BigDecimal` (**phải > 0**; chiều thu/chi do `transactionType` quyết định, không dùng số âm), `transactionType: TransactionType` (Income/Expense), `transactionStatus: TransactionStatus`, `createdPersonId` (người tạo), `rawData: String` (dữ liệu gốc — xem 3.4.1).
- Chỉ thành viên có quyền `Write` (hoặc Owner) tạo giao dịch, và chỉ trong danh mục `Active`.
- **Rẽ nhánh theo `amountOfApproval` của danh mục, xét trên đúng giao dịch đó:**
  - `amount < amountOfApproval` → **tự động `Approve`**, bỏ qua Pend/Review.
  - `amount ≥ amountOfApproval` → đi đủ `Draft` (nháp) → `Pend` (đã nộp, chờ xử lý) → `Review` (đang được xem xét) → `Approve` | `Reject`.
  - Chỉ so sánh **số tiền của đúng giao dịch đó** với `amountOfApproval`, không cộng dồn các giao dịch khác.
- **Người duyệt**: **Owner**, hoặc Member được Owner chia sẻ quyền `Approval`. `approveTransaction()` / `rejectTransaction()` chỉ 2 nhóm này thực hiện được, và **được duyệt mọi giao dịch cần duyệt, kể cả giao dịch do chính mình tạo**. `Reject` là trạng thái kết thúc, không tính vào số dư; muốn gửi lại thì tạo giao dịch mới.
- `changeTransactionType()` — đổi Income ↔ Expense khi giao dịch chưa `Approve`.
- **`Offline`**: khi thiết bị mất mạng, giao dịch lưu cục bộ với trạng thái `Offline`; khi có mạng, `syncOfflineTransaction()` đẩy lên máy chủ và **áp lại đúng quy tắc trên**: dưới `amountOfApproval` → thẳng `Approve`; từ ngưỡng trở lên → vào `Pend` chờ duyệt.
- Mọi thay đổi giao dịch (tạo, sửa, duyệt, từ chối, đổi loại, đồng bộ) ghi `ActivityLog`.

#### 3.4.1 Sáu hình thức nhập giao dịch (`rawData`)

Giao dịch được nhập bằng **6 hình thức**: **Manual, OCR, Voice, Message, Announcement, ScanAI**. Dữ liệu gốc của giao dịch lưu ở `rawData: String` — có thể là link ảnh, link video/ghi âm, nội dung tin nhắn/thông báo thô… (dữ liệu lớn lưu ở storage/CDN, `rawData` giữ đường dẫn).

- **Chỉ thể hiện ở source backend, không thể hiện ở database**: không có trường/bảng riêng cho hình thức nhập; việc xử lý `rawData` nằm ở tầng Service/DTO.
- Nguyên tắc chung: xử lý `rawData` **không tự đổi `transactionStatus`**; kết quả được điền sẵn vào màn hình xác nhận để thành viên xem lại, chỉnh sửa, và giao dịch chỉ được tạo khi thành viên xác nhận (rồi đi theo luồng ở 3.4).
- **Luồng chi tiết từng hình thức: người dùng sẽ trình bày sau** — bổ sung vào mục này khi có.

### 3.5 Hỗ trợ người dùng (SupportTicket)
- Thuộc tính: `supportTicketId`, `content`, `noteOfRespondent`, `createdPersonId` (người gửi), `answeredPersonId` (người phản hồi), `ticketStatus: TicketStatus`.
- **Luồng trạng thái**: `Draft` (nháp) → `Submitted` (đã gửi) → `Viewed` (Admin đã xem) → `Responded` (đã phản hồi).
- `handleWithTicket()` / `changeTicketStatus()` — Admin xử lý phiếu và đổi trạng thái; khi phản hồi **bắt buộc** điền `noteOfRespondent` và `answeredPersonId`.
- Người dùng chỉ xem phiếu của chính mình; Admin xem toàn bộ.
- Tạo/gửi/xem/phản hồi phiếu ghi `ActivityLog`.

### 3.6 Nhật ký hoạt động (ActivityLog)
- Thuộc tính: `logId`, `actionType: ActionType` (Login/Logout/Register/Create/Modify/Delete), `activityStatus: ActivityStatus` (Success/Failed), `activityDescription`, `ipAddress`, `logTime`, `performedPersonId` (người thực hiện).
- Ghi cả hành động thành công lẫn thất bại.
- **Không có `ActionType` riêng cho Approve/Reject/Lock/Unlock** — dùng chung `Modify` và **bắt buộc điền `activityDescription`** rõ ràng (ví dụ `"Approve transaction #123"`, `"Auto lock category #5 do chạm amountOfEmergency"`, `"Admin lock account #7"`).
- ActivityLog **chỉ ghi thêm, không sửa/xóa**. Admin tra cứu toàn bộ log.

## 4. Luồng nghiệp vụ chính theo vai trò

### 4.1 Admin
**Bắt đầu**: đăng nhập bằng `Account` `accountRole = Admin`, `Active` → (1) hệ thống xác thực, ghi log `Login/Success` → (2) xem danh sách tài khoản; khóa tài khoản vi phạm (`Inactive`) hoặc mở lại (`Active`) → (3) mở danh sách `SupportTicket` `Submitted`, mở phiếu → `Viewed` → (4) `handleWithTicket()`, nhập `noteOfRespondent`, đổi sang `Responded` → (5) tra cứu `ActivityLog` khi cần → (6) mọi khóa/mở/phản hồi sinh log `Modify` kèm mô tả. **Kết thúc**: đăng xuất (`Logout`); tài khoản và phiếu đã xử lý ở trạng thái mới.

### 4.2 Owner
**Bắt đầu**: người dùng đăng ký, đăng nhập, tạo `Family` → (1) hệ thống tạo `Membership` `Owner` → (2) thêm Person vào gia đình (`Member`), cấp `permissions`; có thể **chia sẻ quyền `Approval`** cho Member tin tưởng; Member nào bị gỡ hết quyền thì bị tạm ẩn → (3) tạo `FinancialCategory` (`initialBalance`, `amountOfWarning`, `amountOfEmergency`, `amountOfApproval`, `Active`) → (4) tự tạo giao dịch hoặc duyệt/từ chối các giao dịch đang chờ (từ `amountOfApproval` trở lên) → (5) sau mỗi giao dịch `Approve`, hệ thống `checkBalance()`: nhận push cảnh báo khi chạm `amountOfWarning`; khi chạm `amountOfEmergency` danh mục **tự khóa**, Owner nâng ngưỡng nếu cần rồi `unlockCategory()` → (6) muốn rời gia đình thì chuyển Owner cho Member khác trước → (7) mọi thao tác ghi log. **Kết thúc**: đăng xuất; gia đình đủ thành viên/danh mục, giao dịch đã `Approve`/`Reject`, số dư đã cập nhật.

### 4.3 Member
**Bắt đầu**: được Owner thêm vào `Family`, đăng nhập → (1) xem danh mục và giao dịch (`Read`) → (2) có `Write`: chọn danh mục `Active`, nhập giao dịch bằng 1 trong 6 hình thức (Manual/OCR/Voice/Message/Announcement/ScanAI), kiểm tra ở màn hình xác nhận, rồi xác nhận → (3) **dưới `amountOfApproval`**: giao dịch tự `Approve`; **từ `amountOfApproval` trở lên**: `Draft` → `Pend`, Owner hoặc người được chia sẻ quyền `Approval` xem (`Review`) rồi `Approve`/`Reject` → (4) mất mạng: lưu `Offline`, có mạng thì `syncOfflineTransaction()` áp lại quy tắc ở bước 3 → (5) bị `Reject` thì tạo lại bản đã sửa; danh mục `Locked` thì chỉ xem → (6) sự cố: tạo `SupportTicket` → gửi → chờ `Responded` → (7) mọi thao tác ghi log. **Kết thúc**: giao dịch ở trạng thái cuối `Approve` (tính vào số tiền đã sử dụng/số dư) hoặc `Reject`; phiếu (nếu có) `Responded`; đăng xuất. Nếu bị Owner gỡ hết quyền, Member bị tạm ẩn khỏi gia đình đó.

## 5. Thực thể nghiệp vụ (Entity Catalog — theo `Document/images/ClassDiagram_0310206.png`)

### 5.1 Account
`accountId: int`, `username: String`, `password: String`, `accountRole: AccountRole`, `accountStatus: AccountStatus` — `changePassword()`, `lockAccount()`, `unlockAccount()`. Quan hệ: `Person "1"——"1" Account` (has).

### 5.2 Person
`personId: int`, `name: String`, `gmail: String`, `phone: String`, `avatar: String`. Không có method. Quan hệ: `Account` (1–1), `Membership` (1–0..*), `Family` (qua `Membership`).

### 5.3 Family
`familyId: int`, `name: String` — `createFinancialCategory()`. Quan hệ: `Person "1..*"——"0..*" Family` (has); `Family "1"——"0..*" FinancialCategory` (has).

### 5.4 Membership
`membershipId: int`, `familyId: int`, `familyMemberRole: FamilyMemberRole`, `permissions: List<FamilyPermission>` — `changeFamilyMemberRole()`, `adjustPermission()`. Quan hệ: `Person "1"——"0..*" Membership` (has).

### 5.5 FinancialCategory
`financialCategoryId: int`, `name: String`, `description: String`, `initialBalance: BigDecimal`, `categoryStatus: CategoryStatus`, `amountOfWarning: BigDecimal`, `amountOfEmergency: BigDecimal`, `amountOfApproval: BigDecimal` — `lockCategory()`, `unlockCategory()`, `checkBalance()`, `modifyThreshold()`, `syncOfflineTransaction()`. Quan hệ: `Family "1"——"0..*" FinancialCategory`; `FinancialCategory "1"——"0..*" Transaction` (has).

### 5.6 Transaction
`transactionId: int`, `content: String`, `amount: BigDecimal`, `transactionType: TransactionType`, `transactionStatus: TransactionStatus`, `createdPersonId: int`, `rawData: String` — `approveTransaction()`, `rejectTransaction()`, `changeTransactionType()`.

### 5.7 SupportTicket
`supportTicketId: int`, `content: String`, `noteOfRespondent: String`, `createdPersonId: int`, `answeredPersonId: int`, `ticketStatus: TicketStatus` — `handleWithTicket()`, `changeTicketStatus()`.

### 5.8 ActivityLog
`logId: int`, `actionType: ActionType`, `activityStatus: ActivityStatus`, `activityDescription: String`, `ipAddress: String`, `logTime: Date`, `performedPersonId: int`.

> **Các trường `*Id` kiểu `int` là khóa ngoại có chủ đích** (`Membership.familyId`, `Transaction.createdPersonId`, `SupportTicket.createdPersonId`/`answeredPersonId`, `ActivityLog.performedPersonId`): thiết kế theo OOP/DDD nên các đối tượng tham chiếu nhau bằng ID, không vẽ association trực tiếp trên diagram.

## 6. Enum / trạng thái

| Enum | Giá trị | Dùng ở |
|---|---|---|
| `AccountRole` | Admin, User | `Account.accountRole` |
| `AccountStatus` | Active, Inactive | `Account.accountStatus` |
| `FamilyMemberRole` | Owner, Member | `Membership.familyMemberRole` |
| `FamilyPermission` | Read, Write, Approval | `Membership.permissions` |
| `CategoryStatus` | Active, Locked | `FinancialCategory.categoryStatus` |
| `TransactionType` | Income, Expense | `Transaction.transactionType` |
| `TransactionStatus` | Draft, Pend, Review, Approve, Reject, Offline | `Transaction.transactionStatus` |
| `TicketStatus` | Draft, Submitted, Viewed, Responded | `SupportTicket.ticketStatus` |
| `ActionType` | Login, Logout, Register, Create, Modify, Delete | `ActivityLog.actionType` |
| `ActivityStatus` | Success, Failed | `ActivityLog.activityStatus` |

## 7. Ma trận quyền theo vai trò

| Hành động | Admin | Owner | Member | Điều kiện của Member |
|---|:---:|:---:|:---:|---|
| Đăng ký / đăng nhập / đổi mật khẩu | ✅ | ✅ | ✅ | — |
| Khóa / mở khóa `Account` | ✅ | — | — | — |
| Tạo `Family` | — | ✅ | ✅ | — |
| Thêm/xóa thành viên, đổi vai trò, chỉnh quyền, chia sẻ `Approval` | — | ✅ | ❌ | — |
| Tạo `FinancialCategory`, đặt/sửa 3 ngưỡng, khóa thủ công | — | ✅ | ❌ | — |
| Mở khóa `FinancialCategory` | — | ✅ | ✅ | có `Approval` (do Owner chia sẻ) |
| Xem danh mục và giao dịch của gia đình | — | ✅ | ✅ | có `Read` |
| Tạo giao dịch (trong danh mục `Active`) | — | ✅ | ✅ | có `Write` |
| Duyệt / từ chối giao dịch (từ `amountOfApproval` trở lên) | — | ✅ | ✅ | có `Approval` (do Owner chia sẻ) |
| Gửi `SupportTicket` | — | ✅ | ✅ | — |
| Phản hồi `SupportTicket` | ✅ | ❌ | ❌ | — |
| Tra cứu toàn bộ `ActivityLog` | ✅ | ❌ | ❌ | — |

Member có `permissions` rỗng bị **tạm ẩn**: không thực hiện được hành động nào trong gia đình đó (xem 3.2). Quyền Admin kiểm tra bằng `Account.accountRole` ở tầng Service; quyền trong gia đình kiểm tra bằng `Membership.familyMemberRole` + `permissions` của đúng `Family` đang thao tác.

## 8. Lưu ý triển khai (không phải lỗi thiết kế)

1. **Tiền dùng `BigDecimal`** (`initialBalance`, 3 ngưỡng, `Transaction.amount`) — diagram ghi `BigDemical` (lỗi chính tả, hiểu là `BigDecimal`). So sánh bằng `compareTo()`, cộng/trừ bằng `add()`/`subtract()`, không ép `double`.
2. **Diagram ghi `amountOfWarming`** (lỗi chính tả) — dùng tên `amountOfWarning` trong code.
3. **Các ràng buộc chỉ chốt bằng lời, không thể hiện trên diagram** — bắt buộc validate ở tầng Service: đúng 1 `Owner`/Family; Owner full quyền bỏ qua `permissions`; `amount > 0`; danh mục `Locked` chặn tạo giao dịch; chỉ giao dịch `Approve` vào số dư; rẽ nhánh duyệt theo `amountOfApproval`; Admin chỉ dựa vào `accountRole`; `ActivityLog` chỉ ghi thêm.
4. **Kiểm tra quyền phải làm ở phía máy chủ** cho mọi thao tác; không tin cờ quyền do client gửi.
5. **Khóa ngoại theo DDD**: các trường `*Id` tham chiếu aggregate khác bằng ID; ràng buộc duy nhất (`personId`, `familyId`) để 1 người không có 2 `Membership` trong cùng 1 gia đình.
6. **Push thông báo** (cảnh báo/khẩn cấp tới toàn bộ thành viên gia đình, giao dịch nháp từ Announcement) cần cơ chế gửi push phía máy chủ tới ứng dụng di động.
7. **Ngoài phạm vi hiện tại**: các nghiệp vụ trong `Document/SRS.docx` (đăng nhập Google, quên mật khẩu OTP, báo cáo thống kê, sửa hồ sơ…) — người dùng tạm không dùng SRS.

## 9. Quyết định đã chốt & điểm còn mở

### 9.1 Đã chốt (người dùng xác nhận 2026-10-03)

| # | Vấn đề | Quyết định |
|---|---|---|
| 1 | Ngưỡng cảnh báo / khẩn cấp | Cảnh báo → chỉ gửi push, không khóa. Khẩn cấp → khóa danh mục |
| 2 | Cách xét ngưỡng | Cảnh báo và khẩn cấp xét trên **số tiền đã sử dụng của cả danh mục**; `amountOfApproval` xét trên **từng giao dịch**. Diagram đã thêm `amountOfApproval` |
| 3 | Ai được duyệt | Chỉ Owner, trừ khi Owner chia sẻ quyền `Approval` cho Member |
| 4 | Người duyệt | Chủ hộ hoặc người có quyền `Approval` |
| 5 | Khi nào cần duyệt | Vượt `amountOfApproval` thì cần duyệt; dưới mức thì tự động duyệt |
| 6 | `rawData` | Chứa link ảnh/video/audio hoặc nội dung thô; phục vụ 6 hình thức nhập đã mô tả ở 3.4.1 |
| 7 | `SRS.docx` | Tạm thời không dùng |
| 8 | Các trường `*Id` | Là khóa ngoại, thiết kế theo OOP/DDD |
| 9 | `permissions` rỗng | Thành viên bị tạm ẩn, không xóa `Membership` |

### 9.2 Chốt thêm (người dùng xác nhận 2026-10-03, lượt 2)

| # | Vấn đề | Quyết định |
|---|---|---|
| A | Cách phân biệt 6 hình thức nhập | Chỉ thể hiện ở source backend, không thể hiện ở database; luồng chi tiết từng hình thức người dùng trình bày sau |
| B | Người có quyền duyệt tự duyệt giao dịch mình tạo | Được — người có quyền duyệt được duyệt hết |
| C | Ngưỡng duyệt | `amount ≥ amountOfApproval` thì cần duyệt; `<` thì tự động `Approve`; chỉ xét trên từng giao dịch |
| D | 3 ngưỡng | Bắt buộc phải nhập đủ khi tạo danh mục |
| E | Push khi vượt mức cảnh báo | Gửi tới toàn bộ thành viên gia đình |

### 9.2b Chốt thêm (người dùng xác nhận 2026-10-14)

| # | Vấn đề | Quyết định |
|---|---|---|
| F | Quyền `FamilyPermission` | Giữ 3 quyền Read/Write/Approval (không rút về 2 quyền) |
| G | Ai mở khóa danh mục | **Owner và Member có quyền `Approval`** (thay cho "chỉ Owner"). Nâng/sửa ngưỡng vẫn **chỉ Owner**. Báo cáo cần sửa QĐ20 và ghi chú dòng 14 Bảng 1.3 cho khớp |

### 9.3 Cách hiểu tạm còn lại (chưa được xác nhận)

- "Số tiền đã sử dụng" của danh mục chỉ tính `Expense` đã `Approve` (Income không trừ đi).
- Push mỗi lần số tiền đã sử dụng vượt ngưỡng lần đầu; push khẩn cấp cũng gửi tới toàn bộ thành viên gia đình như push cảnh báo.

---

*Dựa trên `Document/images/ClassDiagram_0310206.png` (2026-10-03) và các quyết định người dùng chốt cùng ngày.*

*Lịch sử bản trước (không còn là nguồn chốt hiện hành):*
- *2026-09-06 — bản `FinanceManagement.png` (FamilySpace/ExpenseJar/FamilyMember/Record).*
- *2026-09-26 — bản `TLCN.drawio (9).png` (Relationship/Threshold/Log/Feedback, bỏ cây `Record`).*
- *2026-10-03 — bản viết lại theo class diagram mới (Membership, FinancialCategory 3 ngưỡng, SupportTicket, ActivityLog) và chốt quy tắc duyệt/ngưỡng/tạm ẩn.*
