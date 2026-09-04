# Cấu trúc Frontend Mobile chuẩn (Flutter)

> File này đi cùng bộ 3 tài liệu kiến trúc dự án:
> - [`SKILLS-BACKEND.md`](./SKILLS-BACKEND.md) — Backend Java Spring Boot
> - [`SKILLS-FRONTEND.md`](./SKILLS-FRONTEND.md) — Frontend Web React
> - `SKILLS-FRONTEND-MOBILE.md` (file này) — Frontend Mobile Flutter
>
> Cả 3 cùng theo **1 nguyên tắc kiến trúc thống nhất**: mỗi lớp một trách nhiệm, luồng phụ thuộc một chiều, cùng "nói chung ngôn ngữ" qua DTO khi giao tiếp API.

State management dùng trong tài liệu này: **Provider** (đơn giản, dễ tương ứng với `hooks/` bên React; nếu app lớn hơn có thể thay bằng Riverpod/Bloc mà không đổi cấu trúc thư mục).

---

## 1. Cấu trúc thư mục (package structure) chuẩn

```
lib/
├── main.dart                    # Điểm khởi chạy app
├── app.dart                     # MaterialApp, theme, routes gốc
├── config/
│   ├── env.dart                  # Đọc biến môi trường (API_BASE_URL...)
│   └── theme.dart                # Theme màu, font toàn app
├── api/                         # Gọi API tới backend
│   ├── api_client.dart            # Dio instance, interceptor gắn token
│   ├── user_api.dart
│   └── transaction_api.dart
├── models/                       # Model dữ liệu (fromJson/toJson)
│   ├── user_model.dart
│   └── transaction_model.dart
├── providers/                    # State + logic nghiệp vụ phía client
│   ├── auth_provider.dart
│   └── transaction_provider.dart
├── screens/                      # Từng màn hình/nghiệp vụ (feature-based)
│   ├── login/
│   │   └── login_screen.dart
│   ├── dashboard/
│   │   └── dashboard_screen.dart
│   └── transaction/
│       ├── transaction_list_screen.dart
│       └── transaction_form_screen.dart
├── widgets/                      # Widget dùng chung toàn app
│   ├── common/                     # AppButton, AppInput, AppModal...
│   └── layout/                      # AppBarCustom, BottomNavBar
├── routes/
│   └── app_routes.dart            # Định nghĩa route + navigation guard
├── utils/                         # Hàm tiện ích thuần (không gọi API)
│   ├── format_currency.dart
│   └── format_date.dart
└── constants/                     # Hằng số (message, key, enum)
    └── app_constants.dart
```

> Quy tắc luồng phụ thuộc: **Screen → Provider → Api → HTTP**. Lớp trên gọi xuống lớp dưới, không có chiều ngược lại (`api/` không được import từ `providers/` hay `screens/`).

---

## 2. Ý nghĩa chính xác từng lớp

### `api/` — Data Access Layer (phía client)
Nơi **duy nhất** được phép gọi HTTP request tới backend. Chỉ lo gửi/nhận dữ liệu thô (JSON), không biết dữ liệu đó sẽ hiển thị ra sao. `api_client.dart` cấu hình `Dio` instance dùng chung (baseURL, interceptor gắn token); mỗi file theo module (`user_api.dart`, `transaction_api.dart`) export các hàm gọi endpoint tương ứng.

### `models/` — Data Model Layer (tương đương DTO)
Định nghĩa cấu trúc dữ liệu trao đổi với backend, có `fromJson()` (parse response) và `toJson()` (serialize request). **Không chứa logic nghiệp vụ**, chỉ là "hình dạng" dữ liệu — tương đương DTO bên backend/React.

### `providers/` — Business Logic + State Layer (tương đương hook)
Đóng gói logic có state + side effect: gọi `api/`, xử lý kết quả, thông báo cho UI cập nhật qua `notifyListeners()`. Đây là nơi **duy nhất** chứa logic nghiệp vụ thật phía client (validate, tính toán, điều phối gọi nhiều API).

### `screens/` — Container Layer (tương đương page)
Lớp **"ráp nối"** — mỗi thư mục con là 1 màn hình hoàn chỉnh. Screen lắng nghe `provider` (qua `Consumer`/`context.watch`), gọi hành động khi user thao tác, rồi truyền dữ liệu xuống `widgets/common` để hiển thị. Screen **không tự gọi `api/` trực tiếp**, luôn đi qua `provider`.

### `widgets/common/` — Presentational Widget
Widget UI **thuần túy, tái sử dụng**, không biết gì về nghiệp vụ. Chỉ nhận dữ liệu qua constructor và callback (`onPressed`, `onChanged`...). Ví dụ: `AppButton`, `AppInput`, `AppCard`.

### `widgets/layout/` — Khung giao diện cố định
Phần khung sườn lặp lại nhiều màn hình: `AppBarCustom`, `BottomNavBar`, `Drawer`.

### `routes/` — Routing Layer
Khai báo bản đồ tên route ↔ Screen, xử lý điều hướng và chặn truy cập khi chưa đăng nhập (tương đương `PrivateRoute` bên React).

### `utils/` — Pure Helper Functions
Hàm thuần túy: nhận input, trả output, không side effect, không gọi API, không phụ thuộc `BuildContext`. Ví dụ: `formatCurrency(500000)` → `"500.000 ₫"`.

### `constants/` — Hằng số dùng chung
Giá trị cố định lặp lại nhiều nơi: message lỗi mặc định, key lưu `secure storage`, enum trạng thái. **Nên đặt tên giống hệt** với `constants/` bên React và `constant/`/`enums/` bên backend để dễ đối chiếu.

### `config/` — Cấu hình môi trường & giao diện
`env.dart` đọc biến môi trường (URL backend khác nhau giữa dev/staging/prod — tương đương `.env` bên React); `theme.dart` định nghĩa màu sắc/font dùng xuyên suốt app.

---

## 3. Bảng tóm tắt vai trò

| Lớp | Có logic nghiệp vụ? | Gọi tới lớp nào? | Nhận/trả gì? |
|---|---|---|---|
| `screens/` | Không (chỉ điều phối) | `providers/`, `widgets/common` | Render UI |
| `providers/` | **Có** | `api/` | State + hàm xử lý, `notifyListeners()` |
| `api/` | Không | Backend (HTTP qua Dio) | JSON thô → parse qua `models/` |
| `models/` | Không (chỉ định hình dữ liệu) | — | `fromJson`/`toJson` |
| `widgets/common` | Không | — | Nhận props (constructor), trả UI |
| `routes/` | Không | `screens/`, `providers/` (kiểm tra auth) | Cho phép/chặn điều hướng |
| `utils/` | Hàm thuần, không side effect | — | Input → Output |

---

## 4. Ví dụ end-to-end: chức năng "Danh sách giao dịch"

```dart
// models/transaction_model.dart
class TransactionModel {
  final int id;
  final double amount;
  final String category;
  final String? note;
  final DateTime createdAt;

  TransactionModel({
    required this.id,
    required this.amount,
    required this.category,
    this.note,
    required this.createdAt,
  });

  factory TransactionModel.fromJson(Map<String, dynamic> json) {
    return TransactionModel(
      id: json['id'],
      amount: (json['amount'] as num).toDouble(),
      category: json['category'],
      note: json['note'],
      createdAt: DateTime.parse(json['createdAt']),
    );
  }

  Map<String, dynamic> toJson() => {
        'amount': amount,
        'category': category,
        'note': note,
      };
}
```

```dart
// api/api_client.dart
import 'package:dio/dio.dart';
import 'package:flutter_secure_storage/flutter_secure_storage.dart';
import '../config/env.dart';

final Dio apiClient = Dio(BaseOptions(baseUrl: Env.apiBaseUrl))
  ..interceptors.add(InterceptorsWrapper(
    onRequest: (options, handler) async {
      final token = await const FlutterSecureStorage().read(key: 'accessToken');
      if (token != null) {
        options.headers['Authorization'] = 'Bearer $token';
      }
      handler.next(options);
    },
  ));
```

```dart
// api/transaction_api.dart
import 'api_client.dart';
import '../models/transaction_model.dart';

class TransactionApi {
  Future<List<TransactionModel>> getAll() async {
    final res = await apiClient.get('/transactions');
    return (res.data as List)
        .map((json) => TransactionModel.fromJson(json))
        .toList();
  }

  Future<TransactionModel> create(TransactionModel data) async {
    final res = await apiClient.post('/transactions', data: data.toJson());
    return TransactionModel.fromJson(res.data);
  }
}
```

```dart
// providers/transaction_provider.dart
import 'package:flutter/foundation.dart';
import '../api/transaction_api.dart';
import '../models/transaction_model.dart';

class TransactionProvider with ChangeNotifier {
  final TransactionApi _api = TransactionApi();
  List<TransactionModel> transactions = [];
  bool loading = false;

  Future<void> fetchAll() async {
    loading = true;
    notifyListeners();
    transactions = await _api.getAll();
    loading = false;
    notifyListeners();
  }
}
```

```dart
// screens/transaction/transaction_list_screen.dart
import 'package:flutter/material.dart';
import 'package:provider/provider.dart';
import '../../providers/transaction_provider.dart';
import '../../widgets/common/app_list_tile.dart';

class TransactionListScreen extends StatefulWidget {
  @override
  State<TransactionListScreen> createState() => _TransactionListScreenState();
}

class _TransactionListScreenState extends State<TransactionListScreen> {
  @override
  void initState() {
    super.initState();
    Future.microtask(() => context.read<TransactionProvider>().fetchAll());
  }

  @override
  Widget build(BuildContext context) {
    return Consumer<TransactionProvider>(
      builder: (context, provider, _) {
        if (provider.loading) return const CircularProgressIndicator();
        return ListView.builder(
          itemCount: provider.transactions.length,
          itemBuilder: (context, index) =>
              AppListTile(transaction: provider.transactions[index]),
        );
      },
    );
  }
}
```

---

## 5. Quy tắc giữ "sạch"

- **Dumb vs Smart widget**: `widgets/common` chỉ nhận dữ liệu qua constructor và render (không gọi API); `screens/` mới "thông minh" — lắng nghe provider, xử lý điều hướng.
- **Screen không gọi thẳng `api/`**: luôn đi qua `providers/` để logic tập trung một chỗ, dễ test và tái sử dụng giữa nhiều màn hình.
- **1 file = 1 trách nhiệm**: không nhét cả gọi API lẫn UI lẫn xử lý logic vào chung 1 widget lớn.
- **Không hardcode URL/secret**: dùng `config/env.dart` (đọc từ `--dart-define` hoặc gói `flutter_dotenv`) để tách biệt cấu hình dev/staging/prod.
- **Lưu token bằng `flutter_secure_storage`**, không dùng `SharedPreferences` cho dữ liệu nhạy cảm (khác với web dùng `localStorage`).

---

## 6. Sự tương ứng 3 chiều: Backend ↔ Web (React) ↔ Mobile (Flutter)

```
Backend (Java)        Web (React)          Mobile (Flutter)
───────────────        ─────────────        ─────────────────
Controller       ←──→  pages/         ←──→  screens/            (điều phối)
Service/Impl      ←──→  hooks/         ←──→  providers/           (logic thật)
Repository        ←──→  api/           ←──→  api/                 (gọi HTTP thô)
Entity/DTO         ←──→  (object JS)   ←──→  models/              (fromJson/toJson)
Mapper             ←──→  (map trong hook) ←→ (fromJson trong model)
constant/enums     ←──→  constants/     ←──→  constants/           (đặt tên giống nhau)
Security/JWT Filter ←──→ routes/PrivateRoute ←→ routes/ + secure storage
Config             ←──→  .env           ←──→  config/env.dart
```

| Vai trò | Backend | Web (React) | Mobile (Flutter) |
|---|---|---|---|
| Điều phối request/UI | Controller | `pages/` | `screens/` |
| Logic nghiệp vụ thật | Service/Impl | `hooks/` | `providers/` |
| Gọi/truy vấn dữ liệu thô | Repository | `api/` | `api/` |
| Định hình dữ liệu trao đổi | DTO (request/response) | object JS theo shape DTO | `models/` (`fromJson`/`toJson`) |
| Chuyển đổi dữ liệu | Mapper | xử lý trong `hooks/` | xử lý trong `models/` |
| Hằng số/enum | `constant/`, `enums/` | `constants/` | `constants/` |
| Bảo vệ truy cập | Security/JWT Filter | `routes/PrivateRoute` | `routes/` + kiểm tra token |
| Cấu hình môi trường | `application.yml`/`.properties` | `.env` (Vite) | `config/env.dart` |

### Ví dụ 1 DTO dùng chung cho cả 3 phía — "Tạo giao dịch"

```java
// Backend: dto/request/TransactionCreateRequest.java
public class TransactionCreateRequest {
    @NotNull Double amount;
    @NotNull String category;
    String note;
}
```

```javascript
// Web React: payload gửi lên phải khớp field trên
const payload = { amount: 500000, category: "FOOD", note: "Ăn trưa" };
transactionApi.create(payload);
```

```dart
// Mobile Flutter: model.toJson() phải khớp field trên
final data = TransactionModel(amount: 500000, category: "FOOD", note: "Ăn trưa");
await transactionApi.create(data); // gọi data.toJson() bên trong
```

**→ Cả 3 phía cùng "nói chung 1 ngôn ngữ dữ liệu"**: đổi field ở backend (thêm/bớt/đổi tên) thì phải cập nhật đồng thời cả `models/` bên Flutter lẫn cấu trúc payload bên React, nếu không sẽ vỡ luồng dữ liệu ở phía không được cập nhật.

### Điểm khác biệt quan trọng giữa Web và Mobile (dù cùng là "frontend")

| Vấn đề | Web (React) | Mobile (Flutter) |
|---|---|---|
| Lưu token | `localStorage` (dễ bị XSS đọc) | `flutter_secure_storage` (mã hóa, an toàn hơn) |
| Gọi HTTP | `axios` | `dio` |
| Biến môi trường | `.env` + Vite (`import.meta.env`) | `--dart-define` hoặc `flutter_dotenv` |
| State toàn cục | Context API/Redux/Zustand | Provider/Riverpod/Bloc |
| Điều hướng | React Router (URL-based) | Navigator (stack-based, không có URL thật) |

Về bản chất kiến trúc (4 lớp: điều phối → logic → gọi API → dữ liệu thô), **Web và Mobile giống hệt nhau** — chỉ khác công cụ/thư viện cụ thể do đặc thù nền tảng.
