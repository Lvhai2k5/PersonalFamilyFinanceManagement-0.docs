# Cấu trúc Backend chuẩn (Java - Spring Boot)

## 1. Cấu trúc thư mục (package structure) chuẩn

```
ute.fit.financemanagement
├── config/              # Cấu hình ứng dụng
├── controller/          # Tiếp nhận HTTP request
├── dto/
│   ├── request/         # Dữ liệu client gửi lên
│   └── response/        # Dữ liệu trả về client
├── entity/              # Ánh xạ bảng database
├── enums/                # Hằng số dạng liệt kê
├── exception/           # Exception tùy chỉnh + xử lý lỗi tập trung
├── mapper/              # Chuyển đổi Entity <-> DTO
├── repository/          # Truy vấn database
├── security/            # Xác thực, phân quyền (JWT, filter...)
├── service/
│   ├── (interface)      # Khai báo hành vi nghiệp vụ
│   └── impl/            # Cài đặt logic nghiệp vụ
├── validator/           # Validate dữ liệu tùy chỉnh
├── constant/            # Hằng số cố định (message, key...)
├── filter/               # Interceptor/Filter xử lý request trước Controller
├── specification/        # Query động (Spring Data JPA Specification)
└── ProjectApplication.java   # Điểm khởi chạy ứng dụng
```

> Quy tắc luồng phụ thuộc: **Controller → Service (interface) → ServiceImpl → Repository → Entity**. Lớp trên chỉ gọi xuống lớp dưới, không được gọi ngược lại.

---

## 2. Vai trò chính xác của từng lớp

### Controller (Presentation Layer)
- **Chỉ làm nhiệm vụ điều phối HTTP**: nhận request, deserialize JSON → DTO, gọi Service, serialize kết quả → response.
- **Không chứa** logic nghiệp vụ, không truy vấn trực tiếp Repository, không xử lý transaction.
- Annotation: `@RestController`, `@RequestMapping`, `@GetMapping/@PostMapping/@PutMapping/@DeleteMapping`, `@Valid` (kích hoạt validate DTO).

### DTO — Request/Response
- **Request DTO**: định nghĩa đúng field client được phép gửi lên (tránh mass-assignment — không dùng Entity làm input).
- **Response DTO**: định nghĩa đúng field trả về client, ẩn dữ liệu nội bộ/nhạy cảm.
- Chứa annotation validate: `@NotNull`, `@Size`, `@Email`...
- **Không chứa logic nghiệp vụ, không có annotation `@Entity`.**

### Service — interface + Impl
- **Service (interface)**: khai báo "hợp đồng" — ứng dụng có thể làm gì (ví dụ `UserService.createUser(...)`), giúp Controller không phụ thuộc vào cách cài đặt cụ thể (dễ mock khi test, dễ thay đổi implementation).
- **ServiceImpl**: nơi **duy nhất** chứa logic nghiệp vụ thật — validate nghiệp vụ (khác với validate format ở DTO), tính toán, điều phối gọi nhiều Repository, quản lý transaction.
- Annotation: `@Service`, `@Transactional`.
- **Nguyên tắc quan trọng**: transaction boundary luôn đặt ở Service, không đặt ở Controller hay Repository.

### Repository (Data Access Layer)
- **Chỉ làm nhiệm vụ truy vấn/lưu dữ liệu**, không chứa logic nghiệp vụ (không tính toán, không validate nghiệp vụ).
- Kế thừa `JpaRepository<Entity, ID>` cho CRUD cơ bản; viết thêm method theo convention (`findByEmail`) hoặc `@Query` cho truy vấn phức tạp.
- Annotation: `@Repository` (Spring Data JPA tự động implement, thường không cần khai báo tường minh).

### Entity (Domain Layer)
- Ánh xạ 1-1 với bảng trong database, đại diện cho **trạng thái dữ liệu lưu trữ**, không phải dữ liệu trao đổi qua API.
- Có thể chứa logic thuần liên quan đến chính đối tượng đó (ví dụ method `isExpired()`), nhưng **không gọi Repository/Service khác** (tránh entity phụ thuộc ngược lên tầng trên).
- Annotation: `@Entity`, `@Table`, `@Id`, `@Column`, `@OneToMany`, `@ManyToOne`...

### Mapper
- **Chỉ chuyển đổi dữ liệu** giữa Entity và DTO (2 chiều), không chứa logic nghiệp vụ, không gọi Repository/Service.
- Viết tay hoặc dùng MapStruct (`@Mapper` — sinh code compile-time, hiệu năng tốt hơn reflection).

### Exception + Global Exception Handler
- **Custom Exception** (`UserNotFoundException`, `InvalidRequestException`...): định nghĩa lỗi nghiệp vụ rõ ràng, ném (`throw`) từ Service.
- **GlobalExceptionHandler**: bắt toàn bộ exception ở một nơi duy nhất, map sang HTTP status + response lỗi chuẩn hóa (tránh mỗi Controller tự try-catch riêng lẻ).
- Annotation: `@RestControllerAdvice`, `@ExceptionHandler`.

### Config
- Nơi khai báo Bean cấu hình toàn cục: `DataSourceConfig`, `SecurityConfig`, `CorsConfig`, `SwaggerConfig`, `WebConfig`...
- **Không chứa logic nghiệp vụ**, chỉ khởi tạo/cấu hình thành phần hạ tầng.
- Annotation: `@Configuration`, `@Bean`.

### Security
- Xử lý authentication (xác thực danh tính) và authorization (phân quyền truy cập).
- Thành phần thường gặp: `JwtTokenProvider` (tạo/giải mã token), `JwtAuthenticationFilter` (chặn request kiểm tra token), `UserDetailsServiceImpl`, `SecurityConfig` (khai báo endpoint nào public/private).
- Quan trọng với hệ thống có nhiều client (React + Flutter) vì không dùng session cookie truyền thống mà dùng token (JWT) stateless.

### Validator (tùy chọn, cho validate phức tạp)
- Dùng khi validate vượt quá khả năng annotation có sẵn (`@NotNull`, `@Size`...), ví dụ kiểm tra định dạng số điện thoại theo vùng, kiểm tra business rule đơn giản ngay tại DTO.
- Annotation: `@Constraint`, custom annotation (ví dụ `@ValidPhoneNumber`).

### Constant / Enums
- **Constant**: giá trị cố định dùng nhiều nơi (message lỗi, key cấu hình...), tránh hardcode chuỗi rải rác.
- **Enum**: tập giá trị cố định có ý nghĩa nghiệp vụ (ví dụ `OrderStatus { PENDING, PAID, CANCELLED }`), giúp thay thế "magic string/number".

### Filter / Interceptor
- Xử lý request **trước khi** vào tới Controller (ví dụ: log request, kiểm tra token, đo thời gian xử lý).
- Khác Security Filter ở chỗ đây thường dùng cho mục đích chung (logging, CORS thủ công...), không nhất thiết liên quan xác thực.
- Annotation/Interface: `OncePerRequestFilter`, `HandlerInterceptor`.

### Specification (tùy chọn)
- Dùng khi cần xây dựng query động (filter theo nhiều điều kiện tùy ý từ client, ví dụ tìm kiếm nâng cao), tránh viết hàng chục method `findByXAndY...` trong Repository.
- Dùng cùng Spring Data JPA Specification API.

---

## 3. Bảng tóm tắt vai trò

| Lớp | Có logic nghiệp vụ? | Gọi tới lớp nào? | Trả về |
|---|---|---|---|
| Controller | Không | Service | DTO (response) |
| DTO | Không (chỉ validate format) | — | — |
| Service (interface) | — (khai báo hành vi) | — | — |
| ServiceImpl | **Có** | Repository, Mapper | DTO hoặc Entity |
| Repository | Không | Database | Entity |
| Entity | Chỉ logic nội tại đối tượng | — | — |
| Mapper | Không (chỉ chuyển đổi) | — | DTO ↔ Entity |
| Exception Handler | Không | — | Response lỗi chuẩn hóa |
| Config | Không | — | Bean cấu hình |
| Security | Xác thực/phân quyền | Service (UserDetails) | Cho phép/chặn request |

---

## 4. Phân biệt DTO - Entity - Mapper

| Lớp | Vai trò | Ví dụ |
|---|---|---|
| **Entity** | Ánh xạ trực tiếp bảng DB, chứa mọi cột kể cả dữ liệu nhạy cảm | `User { id, name, email, passwordHash }` |
| **DTO** | Dữ liệu gửi/nhận qua API, chỉ chứa field cần thiết | `UserResponseDTO { id, name, email }` |
| **Mapper** | Cầu nối chuyển đổi Entity ↔ DTO | `UserMapper.toDTO(User user) -> UserResponseDTO` |

### Ví dụ minh họa

```java
// Entity - ánh xạ bảng DB
@Entity
@Table(name = "users")
class User {
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    Long id;
    String name;
    String email;
    String passwordHash; // nhạy cảm, không được lộ ra ngoài
}

// Request DTO - dữ liệu client gửi lên
class UserCreateRequest {
    @NotBlank String name;
    @Email String email;
    @Size(min = 8) String password;
}

// Response DTO - dữ liệu trả về client
class UserResponseDTO {
    Long id;
    String name;
    String email;
    // không có passwordHash
}

// Mapper - chuyển Entity -> DTO
@Mapper(componentModel = "spring")
interface UserMapper {
    UserResponseDTO toDTO(User user);
    User toEntity(UserCreateRequest request);
}

// Service interface
interface UserService {
    UserResponseDTO createUser(UserCreateRequest request);
}

// ServiceImpl - nơi chứa logic nghiệp vụ thật
@Service
@Transactional
class UserServiceImpl implements UserService {
    private final UserRepository userRepository;
    private final UserMapper userMapper;
    private final PasswordEncoder passwordEncoder;

    public UserResponseDTO createUser(UserCreateRequest request) {
        if (userRepository.existsByEmail(request.getEmail())) {
            throw new EmailAlreadyExistsException(request.getEmail());
        }
        User user = userMapper.toEntity(request);
        user.setPasswordHash(passwordEncoder.encode(request.getPassword()));
        User saved = userRepository.save(user);
        return userMapper.toDTO(saved);
    }
}

// Repository
interface UserRepository extends JpaRepository<User, Long> {
    boolean existsByEmail(String email);
}

// Custom Exception
class EmailAlreadyExistsException extends RuntimeException {
    EmailAlreadyExistsException(String email) {
        super("Email đã tồn tại: " + email);
    }
}

// Global Exception Handler
@RestControllerAdvice
class GlobalExceptionHandler {
    @ExceptionHandler(EmailAlreadyExistsException.class)
    ResponseEntity<ErrorResponse> handleEmailExists(EmailAlreadyExistsException ex) {
        return ResponseEntity.status(HttpStatus.CONFLICT)
                .body(new ErrorResponse(ex.getMessage()));
    }
}

// Controller
@RestController
@RequestMapping("/api/users")
class UserController {
    private final UserService userService;

    @PostMapping
    ResponseEntity<UserResponseDTO> create(@Valid @RequestBody UserCreateRequest request) {
        return ResponseEntity.ok(userService.createUser(request));
    }
}
```

---

## 5. Luồng chạy chuẩn

Client (React/Flutter)
→ **Filter/Security** (kiểm tra token nếu cần)
→ **Controller** (nhận request, validate format qua DTO)
→ **Service interface** → **ServiceImpl** (validate nghiệp vụ, xử lý logic, transaction)
→ **Repository** (truy vấn DB) ↔ **Entity**
→ **Mapper** (Entity → DTO)
→ **Controller** trả JSON về client
→ Nếu lỗi ở bất kỳ đâu → ném Exception → **GlobalExceptionHandler** bắt và trả response lỗi chuẩn hóa.

---

## 6. Ghi chú kiến trúc tổng thể

- Backend Java (Spring Boot) expose REST API dùng chung cho cả React (web) và Flutter (mobile).
- Nên thiết kế API chuẩn hóa (REST rõ ràng, versioning `/api/v1/...`, DTO tách biệt khỏi entity) để tránh phải tạo endpoint riêng lẻ cho từng client.
- Dùng JWT (stateless) thay vì session cookie vì có nhiều loại client khác nhau (web, mobile) cùng gọi vào một backend.
- Nếu nhu cầu dữ liệu giữa web và mobile lệch nhau nhiều về sau, có thể cân nhắc GraphQL hoặc BFF (Backend For Frontend).
- MVVM không phù hợp cho backend (dùng cho ứng dụng có giao diện như WPF, Android, iOS). Backend nên theo MVC/MVCS (Controller - Service - Repository) như mô tả ở trên.
