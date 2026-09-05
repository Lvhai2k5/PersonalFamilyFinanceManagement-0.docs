# Nguyên tắc thiết kế: Clean Architecture, SOLID, OOP

> File này quy định **cách viết code bên trong từng lớp**, bổ sung cho 3 file cấu trúc đã có:
> - [`SKILLS-BACKEND.md`](./SKILLS-BACKEND.md) — cấu trúc Backend (nơi file gì đặt ở đâu)
> - [`Frontend-Web/SKILLS-FRONTEND-WEB.md`](./Frontend-Web/SKILLS-FRONTEND-WEB.md) — cấu trúc Web React
> - [`SKILLS-FRONTEND-MOBILE.md`](./SKILLS-FRONTEND-MOBILE.md) — cấu trúc Mobile Flutter
>
> 3 file trên trả lời **"đặt code ở đâu"**. File này trả lời **"viết code trong đó như thế nào cho đúng"**.

---

## 1. Clean Architecture — áp dụng vào dự án

### Ý tưởng gốc (Robert C. Martin)

```
        ┌─────────────────────────────┐
        │   Frameworks & Drivers       │  ← DB, Web framework, UI
        │  ┌─────────────────────┐    │
        │  │ Interface Adapters   │    │  ← Controller, Repository impl, Mapper
        │  │  ┌───────────────┐  │    │
        │  │  │  Use Cases     │  │    │  ← Service (logic nghiệp vụ)
        │  │  │  ┌─────────┐  │  │    │
        │  │  │  │ Entities │  │  │    │  ← Model nghiệp vụ cốt lõi
        │  │  │  └─────────┘  │  │    │
        │  │  └───────────────┘  │    │
        │  └─────────────────────┘    │
        └─────────────────────────────┘
```

**Dependency Rule (nguyên tắc quan trọng nhất):** mũi tên phụ thuộc **luôn hướng vào trong**. Vòng trong không được biết gì về vòng ngoài.

### Map vào cấu trúc thực tế của dự án

| Vòng Clean Architecture | Lớp tương ứng trong dự án |
|---|---|
| Entities | `entity/` (Backend), `models/` (Flutter/React — phần lõi dữ liệu) |
| Use Cases | `service/` (logic nghiệp vụ thật) |
| Interface Adapters | `controller/`, `mapper/`, `repository/` (interface) |
| Frameworks & Drivers | Spring Boot, Database, React/Flutter UI framework |

### 3 hệ quả bắt buộc tuân theo trong dự án

1. **`Entity` không được import `Repository` hay `Service`.** Entity chỉ chứa dữ liệu + logic thuần liên quan chính nó (`isExpired()`, `calculateTotal()`...), không được gọi ngược lên tầng trên.
2. **`Service` không được import bất cứ thứ gì thuộc `Controller`.** Service không biết mình đang được gọi từ REST API, GraphQL hay CLI — nó chỉ nhận input, trả output.
3. **`Service` phụ thuộc vào `Repository` qua interface, không phụ thuộc class cài đặt cụ thể** (đây chính là nguyên tắc D trong SOLID — xem mục 2.5).

> **Vì sao quan trọng?** Nếu vi phạm (ví dụ Entity gọi thẳng Repository), khi đổi công nghệ lưu trữ (MySQL → MongoDB) hoặc đổi framework, bạn phải sửa cả logic nghiệp vụ lẫn tầng dữ liệu — thay vì chỉ sửa 1 lớp Adapter.

---

## 2. Nguyên tắc SOLID

### 2.1 S — Single Responsibility Principle (1 class = 1 lý do để thay đổi)

Một class chỉ nên có **đúng 1 trách nhiệm**, đúng 1 lý do khiến nó phải sửa đổi.

```java
// SAI — UserService vừa lo nghiệp vụ, vừa lo gửi email, vừa lo validate
class UserService {
    void createUser(UserCreateRequest req) {
        // validate
        if (req.getEmail() == null) throw new RuntimeException("...");
        // lưu DB
        userRepository.save(...);
        // gửi email — không liên quan tới "tạo user"
        smtpClient.send(req.getEmail(), "Chào mừng!");
    }
}
```

```java
// ĐÚNG — tách riêng từng trách nhiệm
class UserService {
    private final UserRepository userRepository;
    private final EmailService emailService; // inject, không tự làm

    void createUser(UserCreateRequest req) {
        User user = userRepository.save(toEntity(req));
        emailService.sendWelcomeEmail(user.getEmail());
    }
}

class EmailService {
    void sendWelcomeEmail(String email) { /* chỉ lo gửi email */ }
}
```

**Áp dụng trong dự án:** đây chính là lý do tách `Controller` (điều phối HTTP) / `Service` (logic) / `Repository` (dữ liệu) / `Mapper` (chuyển đổi) thành các lớp riêng như 3 file trước đã mô tả.

### 2.2 O — Open/Closed Principle (mở để mở rộng, đóng để sửa đổi)

Thêm tính năng mới bằng cách **thêm code mới**, không sửa code cũ đang chạy ổn định.

```java
// SAI — mỗi lần thêm loại giảm giá mới phải sửa hàm này
double calculateDiscount(String type, double amount) {
    if (type.equals("STUDENT")) return amount * 0.9;
    if (type.equals("VIP")) return amount * 0.8;
    // thêm loại mới lại phải sửa if-else ở đây
    return amount;
}
```

```java
// ĐÚNG — dùng interface, thêm loại mới = thêm class mới, không sửa code cũ
interface DiscountStrategy {
    double apply(double amount);
}
class StudentDiscount implements DiscountStrategy {
    public double apply(double amount) { return amount * 0.9; }
}
class VipDiscount implements DiscountStrategy {
    public double apply(double amount) { return amount * 0.8; }
}
// thêm SeasonalDiscount mới -> chỉ thêm class, không đụng code cũ
```

**Áp dụng trong dự án:** khi logic có nhiều "biến thể" (nhiều loại giao dịch, nhiều loại thông báo...), ưu tiên interface + nhiều implementation thay vì if-else/switch-case ngày càng phình to.

### 2.3 L — Liskov Substitution Principle (class con phải thay thế được class cha)

Nếu `B extends A`, thì ở bất cứ đâu dùng `A`, thay bằng `B` vẫn phải chạy đúng, không được gây lỗi hay đổi hành vi bất ngờ.

```java
// SAI — Square kế thừa Rectangle nhưng phá vỡ hành vi của cha
class Rectangle {
    void setWidth(int w) { this.width = w; }
    void setHeight(int h) { this.height = h; }
}
class Square extends Rectangle {
    void setWidth(int w) { this.width = w; this.height = w; } // hành vi khác cha!
}
// code dùng Rectangle rect = new Square() sẽ tính sai diện tích
```

**Áp dụng trong dự án:** nếu có `SavingAccount extends Account` và `LoanAccount extends Account`, đảm bảo mọi method dùng chung ở `Account` (ví dụ `withdraw()`) hoạt động đúng logic ở cả 2 lớp con — nếu không thể đảm bảo, dùng interface riêng thay vì ép kế thừa chung.

### 2.4 I — Interface Segregation Principle (không ép class implement thứ nó không dùng)

Nhiều interface nhỏ, chuyên biệt tốt hơn 1 interface lớn "ôm đồm".

```java
// SAI — interface quá lớn, ép mọi loại "worker" implement cả 2 việc không liên quan
interface Worker {
    void code();
    void attendMeeting();
}
class Robot implements Worker {
    public void code() { /* ok */ }
    public void attendMeeting() { throw new UnsupportedOperationException(); } // robot không họp!
}
```

```java
// ĐÚNG — tách interface nhỏ theo đúng khả năng
interface Coder { void code(); }
interface MeetingAttendee { void attendMeeting(); }
class Robot implements Coder { public void code() { /* ok */ } }
class Developer implements Coder, MeetingAttendee { ... }
```

**Áp dụng trong dự án:** ví dụ `UserService` không nên gộp chung cả nghiệp vụ "quản lý user" lẫn "quản lý quyền admin" vào 1 interface — tách `UserService` và `AdminUserService` riêng nếu 2 nhóm client khác nhau dùng.

### 2.5 D — Dependency Inversion Principle (phụ thuộc vào interface, không phụ thuộc class cụ thể)

Đây là nguyên tắc **quan trọng nhất với cấu trúc dự án đã thiết kế** — chính là lý do `service/` tách interface riêng khỏi `service/impl/`.

```java
// SAI — Service phụ thuộc trực tiếp class cài đặt cụ thể
class UserServiceImpl {
    private MySqlUserRepository repository = new MySqlUserRepository(); // gắn chặt vào MySQL
}
```

```java
// ĐÚNG — Service phụ thuộc interface, Spring tự inject implementation cụ thể
interface UserRepository extends JpaRepository<User, Long> { ... }

class UserServiceImpl implements UserService {
    private final UserRepository userRepository; // interface, không quan tâm DB gì bên dưới
    UserServiceImpl(UserRepository userRepository) { this.userRepository = userRepository; }
}
```

**Áp dụng trong dự án:**
- `Controller` phụ thuộc `Service` (interface), không phụ thuộc `ServiceImpl`.
- `ServiceImpl` phụ thuộc `Repository` (interface Spring Data JPA sinh ra), không tự `new` implementation.
- Đây cũng là lý do dùng **Dependency Injection** (constructor injection) thay vì tự khởi tạo (`new`) dependency bên trong class.

---

## 3. Nguyên tắc OOP

### 3.1 Bốn tính chất cốt lõi

| Tính chất | Ý nghĩa | Ví dụ trong dự án |
|---|---|---|
| **Encapsulation** (đóng gói) | Giấu dữ liệu nội bộ, chỉ expose qua method có kiểm soát | `Entity` dùng `private` field + getter/setter, validate trong setter nếu cần |
| **Abstraction** (trừu tượng hóa) | Chỉ expose "cái gì làm được", giấu "làm như thế nào" | `Service` interface khai báo `createUser()`, không lộ chi tiết cài đặt |
| **Inheritance** (kế thừa) | Class con tái sử dụng thuộc tính/hành vi class cha | Dùng khi thật sự có quan hệ "is-a" rõ ràng (xem cảnh báo bên dưới) |
| **Polymorphism** (đa hình) | Cùng 1 lời gọi method, hành vi khác nhau tùy đối tượng thực thi | Nhiều `DiscountStrategy` cùng có `apply()` nhưng tính khác nhau |

### 3.2 Quy định định hướng OOP cho dự án

**1. Ưu tiên Composition hơn Inheritance ("has-a" thay vì "is-a" khi không chắc chắn)**

```java
// Tránh: ép kế thừa khi quan hệ không thật sự "is-a"
class ReportGenerator extends EmailSender { ... } // ReportGenerator không "là một" EmailSender

// Nên: dùng composition
class ReportGenerator {
    private final EmailSender emailSender; // "có một" EmailSender để dùng
}
```
Kế thừa tạo ràng buộc chặt (class con phụ thuộc toàn bộ implementation class cha), chỉ dùng khi quan hệ "is-a" thật sự rõ ràng và ổn định lâu dài (ví dụ `SavingAccount is-a Account`).

**2. Program to an interface, not an implementation**
Khai báo biến/tham số bằng interface (`UserService`, `List`), không bằng class cụ thể (`UserServiceImpl`, `ArrayList`) — trừ lúc khởi tạo.

**3. Encapsulate what varies (đóng gói phần hay thay đổi)**
Phần logic nào dễ thay đổi theo thời gian (công thức tính phí, quy tắc giảm giá...) nên tách riêng thành interface/strategy để sửa mà không ảnh hưởng phần còn lại.

**4. Tránh God Class**
Một class không nên biết/làm quá nhiều thứ (dấu hiệu: class có >300-400 dòng, tên chung chung như `Manager`, `Helper`, `Util` chứa đủ loại logic không liên quan). Khi thấy vậy, tách theo nguyên tắc S (Single Responsibility).

**5. Immutability khi có thể**
DTO/Model nên hạn chế setter tùy tiện — ưu tiên tạo object đầy đủ dữ liệu ngay khi khởi tạo (constructor/builder), tránh object ở trạng thái "nửa vời" giữa chừng.

**6. Không lộ dữ liệu nội bộ ra ngoài**
```java
// SAI — trả về list nội bộ, bên ngoài có thể sửa trực tiếp làm hỏng trạng thái
class Wallet {
    private List<Transaction> transactions = new ArrayList<>();
    List<Transaction> getTransactions() { return transactions; } // lộ tham chiếu gốc!
}

// ĐÚNG
List<Transaction> getTransactions() { return List.copyOf(transactions); }
```

---

## 4. Checklist áp dụng khi code (dùng để tự review trước khi commit)

- [ ] Class có đúng 1 trách nhiệm rõ ràng, tên class phản ánh đúng trách nhiệm đó (không đặt tên chung chung `Manager`, `Handler` mơ hồ).
- [ ] `Controller` không chứa logic nghiệp vụ, chỉ gọi `Service` và trả response.
- [ ] `Service`/`Provider`/`hooks` không phụ thuộc trực tiếp implementation cụ thể, luôn qua interface (Backend) hoặc qua lớp trừu tượng hóa API (Frontend).
- [ ] `Entity`/`Model` không import ngược lên `Repository`/`Service`.
- [ ] Dependency được inject qua constructor, không `new` trực tiếp bên trong class nghiệp vụ.
- [ ] Không có khối `if-else`/`switch` liệt kê nhiều "loại" mà sẽ còn phình to theo thời gian → cân nhắc Strategy Pattern (interface + nhiều implementation).
- [ ] Method không quá dài (gợi ý: quá ~40-50 dòng nên xem xét tách nhỏ).
- [ ] Không có class kế thừa chỉ để tái sử dụng code khi quan hệ không thật sự "is-a".
- [ ] Field nhạy cảm/nội bộ là `private`, không trả tham chiếu trực tiếp ra ngoài.

---

## 5. Áp dụng tinh thần SOLID cho Frontend (React & Flutter)

SOLID vốn cho OOP nhưng tinh thần áp dụng được cho component/hook/provider:

| Nguyên tắc | Áp dụng ở Frontend |
|---|---|
| **S** | 1 component/hook chỉ lo 1 việc — `useTransaction` chỉ lo dữ liệu giao dịch, không kiêm luôn xử lý auth |
| **O** | Thêm loại filter/loại hiển thị mới bằng cách thêm component/config mới, không sửa component cũ đang chạy ổn |
| **L** | Component con nhận cùng shape props như component cha khai báo thì phải render đúng, không được "bất ngờ" đổi hành vi |
| **I** | Props interface (TypeScript) nên nhỏ, đúng nhu cầu — không bắt component nhận props nó không dùng |
| **D** | `pages/`/`screens/` phụ thuộc vào `hooks/`/`providers/` (đã trừu tượng hóa việc gọi API), không tự gọi thẳng `axios`/`dio` |

Đây chính là lý do cấu trúc `api/` → `hooks/`(hoặc `providers/`) → `pages/`(hoặc `screens/`) trong 2 file frontend đã thiết kế đúng theo tinh thần Dependency Inversion — tầng trên không phụ thuộc trực tiếp chi tiết cài đặt của tầng dưới.
