# Cấu trúc Frontend chuẩn (React + TypeScript)

> Công nghệ chính thức: **React + TypeScript** (xem [`../TECH-STACK.md`](../TECH-STACK.md)) — mọi file `.jsx`/`.js` bên dưới là quy ước **cũ, đã thay bằng `.tsx`/`.ts`**. Toàn bộ props, state, dữ liệu API phải khai báo `type`/`interface` rõ ràng, hạn chế tối đa `any`.

## 1. Cấu trúc thư mục (package structure) chuẩn

```
src/
├── api/                      # Gọi API tới backend
│   ├── axiosClient.ts          # Cấu hình axios instance (baseURL, interceptor)
│   ├── userApi.ts              # Các hàm gọi API theo module
│   └── transactionApi.ts
├── assets/                   # Ảnh, icon, font, media tĩnh
├── components/                # Component dùng chung toàn app
│   ├── common/                 # Button, Input, Modal, Table... (không gắn nghiệp vụ)
│   └── layout/                  # Header, Sidebar, Footer, MainLayout
├── pages/                     # Từng trang/nghiệp vụ (feature-based)
│   ├── Login/
│   │   ├── Login.tsx
│   │   ├── Login.module.css
│   │   └── useLogin.ts          # hook riêng cho trang này
│   ├── Dashboard/
│   └── Transaction/
│       ├── TransactionList.tsx
│       ├── TransactionForm.tsx
│       └── useTransaction.ts
├── hooks/                     # Custom hook dùng chung nhiều nơi
│   └── useAuth.ts
├── context/  (hoặc store/)     # Quản lý state toàn cục
│   └── AuthContext.tsx          # (hoặc Redux/Zustand store nếu app lớn)
├── routes/                    # Định nghĩa route
│   ├── AppRoutes.tsx
│   └── PrivateRoute.tsx         # Chặn route cần đăng nhập
├── types/                      # Type/interface dùng chung (tương đương DTO phía backend)
│   └── transaction.ts
├── utils/                      # Hàm tiện ích thuần (không gọi API)
│   ├── formatCurrency.ts
│   └── formatDate.ts
├── constants/                  # Hằng số (message, key, enum)
├── styles/                     # CSS/SCSS toàn cục, biến theme
├── App.tsx
└── main.tsx                    # Điểm khởi chạy (Vite)
```

> Quy tắc luồng phụ thuộc: **Page → Hook → Api → HTTP**. Lớp trên gọi xuống lớp dưới, không có chiều ngược lại (`api/` không được import từ `hooks/` hay `components/`).
>
> **`types/` mới thêm khi chuyển sang TypeScript**: định nghĩa `interface`/`type` cho dữ liệu trao đổi qua API (tương đương DTO backend) — dùng chung giữa `api/`, `hooks/`, `pages/`, tránh mỗi nơi tự khai báo type riêng rồi lệch nhau.

---

## 2. Ý nghĩa chính xác từng lớp

### `api/` — Data Access Layer (phía client)
Là nơi **duy nhất** được phép gọi HTTP request tới backend. Chỉ lo việc gửi/nhận dữ liệu thô (raw JSON), không biết dữ liệu đó sẽ hiển thị ra sao. Chứa `axiosClient` (cấu hình chung: baseURL, interceptor gắn token) và các file theo module (`userApi.ts`, `transactionApi.ts`) — mỗi file export các hàm gọi endpoint tương ứng.

### `components/common/` — Presentational/Dumb Components
Component UI **thuần túy, tái sử dụng**, không biết gì về nghiệp vụ hay dữ liệu tới từ đâu. Chỉ nhận dữ liệu qua `props` và render, kèm callback (`onClick`, `onChange`...) báo lại cho cha khi có tương tác. Ví dụ: `Button`, `Input`, `Modal`, `Table`.

### `components/layout/` — Khung giao diện cố định
Chứa phần khung sườn lặp lại ở nhiều trang: `Header`, `Sidebar`, `Footer`, `MainLayout`. Tách riêng khỏi `common/` vì đây là bố cục tổng thể, không phải UI nhỏ lẻ tái sử dụng.

### `pages/` — Container/Smart Components
Lớp **"ráp nối"** — mỗi thư mục con tương ứng 1 màn hình hoàn chỉnh. Page gọi hook để lấy dữ liệu/logic, rồi truyền xuống cho component thuần ở `components/common` để hiển thị. Page là nơi **duy nhất** biết "màn hình này cần gọi API nào, xử lý gì khi submit form".

### `hooks/` — Reusable Logic Layer
Đóng gói logic có state + side effect (gọi API, subscribe, debounce...) để nhiều page dùng lại được mà không copy code. Ví dụ `useAuth()`, `useDebounce()`. Khác `utils/` ở chỗ hook được dùng React state/effect, utils thì không.

### `context/` (hoặc `store/`) — Global State Layer
Lưu trạng thái **dùng chung nhiều nơi trong app** — thông tin user đăng nhập, theme, giỏ hàng. Nếu chỉ 1-2 trang cần chia sẻ state thì giữ ở page cha là đủ, tránh lạm dụng global state.

### `routes/` — Routing Layer
Khai báo bản đồ URL ↔ Page (`/login` → `Login`, `/dashboard` → `Dashboard`). `PrivateRoute` kiểm tra đã đăng nhập chưa trước khi cho vào trang, nếu chưa thì redirect về `/login`.

### `utils/` — Pure Helper Functions
Hàm **thuần túy**: nhận input, trả output, không side effect, không gọi API, không dùng React hook. Ví dụ: `formatCurrency(1000000)` → `"1.000.000 ₫"`. Vì thuần túy nên dễ unit test.

### `constants/` — Hằng số dùng chung
Giá trị cố định lặp lại nhiều nơi: message lỗi mặc định, key lưu `localStorage`, enum trạng thái (`TRANSACTION_STATUS = { PENDING: 'PENDING', DONE: 'DONE' }`).

### `styles/` — Global Styling
CSS/SCSS dùng chung toàn app: biến màu, font, reset CSS. Style riêng từng page/component nên đặt cạnh file đó (`Login.module.css`), không dồn hết vào đây.

---

## 3. Bảng tóm tắt vai trò

| Lớp | Có logic nghiệp vụ? | Gọi tới lớp nào? | Nhận/trả gì? |
|---|---|---|---|
| `pages/` | Không (chỉ điều phối) | `hooks/`, `components/common` | Render UI |
| `hooks/` | **Có** | `api/` | State + hàm xử lý |
| `api/` | Không | Backend (HTTP) | JSON thô |
| `components/common` | Không | — | Nhận props, trả UI |
| `routes/` | Không | `pages/`, `context/` | Cho phép/chặn truy cập |
| `utils/` | Hàm thuần, không side effect | — | Input → Output |

---

## 4. Ví dụ end-to-end

```typescript
// api/axiosClient.ts
import axios, { type AxiosInstance } from 'axios';

const axiosClient: AxiosInstance = axios.create({
  baseURL: import.meta.env.VITE_API_BASE_URL, // .env: VITE_API_BASE_URL=http://localhost:8080/api
});

axiosClient.interceptors.request.use((config) => {
  const token = localStorage.getItem('accessToken');
  if (token) config.headers.Authorization = `Bearer ${token}`;
  return config;
});

export default axiosClient;
```

```typescript
// types/transaction.ts — type dùng chung, tương đương DTO phía backend
export interface Transaction {
  id: number;
  content: string;
  expense: number;
  transactionStatus: string;
  createdAt: string;
}

export type TransactionCreatePayload = Omit<Transaction, 'id' | 'transactionStatus' | 'createdAt'>;
```

```typescript
// api/transactionApi.ts
import axiosClient from './axiosClient';
import type { Transaction, TransactionCreatePayload } from '../types/transaction';

const transactionApi = {
  getAll: () => axiosClient.get<Transaction[]>('/transactions'),
  create: (data: TransactionCreatePayload) => axiosClient.post<Transaction>('/transactions', data),
};

export default transactionApi;
```

```typescript
// pages/Transaction/useTransaction.ts
import { useState, useEffect } from 'react';
import transactionApi from '../../api/transactionApi';
import type { Transaction } from '../../types/transaction';

export function useTransaction() {
  const [transactions, setTransactions] = useState<Transaction[]>([]);
  const [loading, setLoading] = useState(true);

  useEffect(() => {
    transactionApi.getAll()
      .then((res) => setTransactions(res.data))
      .finally(() => setLoading(false));
  }, []);

  return { transactions, loading };
}
```

```tsx
// pages/Transaction/TransactionList.tsx
import { useTransaction } from './useTransaction';
import Table from '../../components/common/Table';

export default function TransactionList() {
  const { transactions, loading } = useTransaction();

  if (loading) return <p>Đang tải...</p>;
  return <Table data={transactions} />;
}
```

> Ví dụ trên dùng đúng entity thật `TransactionHistory` (field `content`, `expense`, `transactionStatus` — xem [`../BUSINESS-REQUIREMENTS.md`](../BUSINESS-REQUIREMENTS.md) mục 4.5), khác với ví dụ minh họa `amount/category/note` ở mục 6 (mục đó chỉ minh họa nguyên tắc khớp DTO, không phải entity thật).

---

## 5. Quy tắc giữ "sạch"

- **Dumb vs Smart component**: `components/common` chỉ nhận props và render (không gọi API); `pages/` mới "thông minh" — gọi hook, xử lý logic.
- **Alias import**: cấu hình `@/` trỏ về `src/` (trong `vite.config.ts` **và** `tsconfig.json` — 2 nơi phải khớp nhau, thiếu 1 trong 2 sẽ báo lỗi resolve module) để tránh `../../../../` dài dòng.
- **1 file = 1 trách nhiệm**: không nhét cả gọi API lẫn JSX lẫn xử lý logic vào chung 1 component lớn.
- **`.env` cho cấu hình môi trường**: `VITE_API_BASE_URL` khác nhau giữa dev/prod, không hardcode URL backend trong code.
- **`tsconfig.json` bật `strict: true`**: bắt lỗi kiểu dữ liệu ngay lúc code thay vì để runtime mới phát hiện — đúng tinh thần "không tin dữ liệu" đã áp dụng cho validate, nay áp dụng luôn cho type.
- **Không dùng `any` để né lỗi type**: nếu chưa rõ shape dữ liệu, khai báo `interface`/`type` tạm ở `types/` rồi tinh chỉnh sau, thay vì gõ `any` cho nhanh — mất hết lợi ích của TypeScript nếu lạm dụng.

---

## 6. Sự tương ứng với Backend (Java Spring Boot)

```
FRONTEND (React)                          BACKEND (Java Spring Boot)
─────────────────                          ──────────────────────────
components/common/     (UI thuần)
        ↓
pages/                 (điều phối)    ←──→  Controller        (điều phối)
        ↓                                        ↓
hooks/                 (logic thật)   ←──→  Service/Impl       (logic thật)
        ↓                                        ↓
api/                   (gọi HTTP)     ←──→  Repository         (gọi DB)
        ↓                                        ↓
   HTTP Request/Response  ═══════════════   Entity ↔ Database
   (JSON theo DTO)         qua REST API
```

| Frontend (React) | Backend (Java) | Điểm chung |
|---|---|---|
| `pages/` | `Controller` | Điều phối, không chứa logic chi tiết |
| `hooks/` | `Service` (interface + Impl) | Nơi **duy nhất** chứa logic thật |
| `api/` | `Repository` | Chỉ lo lấy/gửi dữ liệu thô |
| Object gửi lên trước khi gọi `api.create(data)` | **Request DTO** | Cùng cấu trúc field — phải khớp chính xác |
| `res.data` nhận về từ axios | **Response DTO** | Cùng cấu trúc field — map thẳng vào state |
| `routes/PrivateRoute` | **Security/JWT Filter** | Cùng chặn truy cập nhưng **không thay thế nhau** — frontend chặn để UX đẹp, backend chặn mới là bảo mật thật |
| `constants/` | `constant/`, `enums/` | Nên đặt tên giống hệt nhau ở cả 2 phía để dễ đối chiếu |
| `context/` (state user đăng nhập) | JWT payload / SecurityContext | Frontend lưu tạm để hiển thị; backend là nguồn sự thật để xác thực |

### Ví dụ khớp DTO giữa 2 phía — chức năng "Tạo giao dịch"

> Đây là **ví dụ minh họa cho nguyên tắc khớp field DTO giữa 2 phía**, field đặt tên đơn giản (`amount`, `category`, `note`) để dễ đọc — **không phải entity thật của dự án**. Entity/field thật (`TransactionHistory`, `Record` và 6 lớp con Manual/OCR/Voice/Message/Announcement/ScanAI, `TransactionStatus`...) xem [`BUSINESS-REQUIREMENTS.md`](../BUSINESS-REQUIREMENTS.md) mục 3.4 & 4.5.

```typescript
// frontend: payload gửi đi (type khai báo tương ứng ở types/, minh họa ngắn gọn ngay tại đây)
interface DemoTransactionPayload {
  amount: number;
  category: string;
  note?: string;
}

const payload: DemoTransactionPayload = {
  amount: 500000,
  category: "FOOD",
  note: "Ăn trưa",
};
transactionApi.create(payload);
```

```java
// backend: dto/request/TransactionCreateRequest.java — PHẢI khớp field với payload trên
public class TransactionCreateRequest {
    @NotNull BigDecimal amount;
    @NotNull String category;
    String note;
}
```

```java
// backend: dto/response/TransactionResponseDTO.java
public class TransactionResponseDTO {
    Long id;
    BigDecimal amount;
    String category;
    String note;
    LocalDateTime createdAt;
}
```

```typescript
// frontend nhận đúng theo Response DTO
const res = await transactionApi.create(payload);
// res.data có shape: { id, amount, category, note, createdAt }
```

### Điểm mấu chốt

- **DTO là "hợp đồng" chung** giữa 2 phía — đổi field một bên mà không báo bên kia sẽ vỡ luồng dữ liệu. Nên định nghĩa DTO trước (API contract) rồi code song song cả 2 bên.
- **Validate ở cả 2 nơi**: frontend validate để UX mượt, backend validate là bắt buộc vì đó mới là nơi đảm bảo an toàn dữ liệu (frontend có thể bị bypass).
- **Không tin dữ liệu từ frontend**: backend luôn phải validate lại toàn bộ (`@Valid` trên DTO) vì request có thể không đến từ giao diện thật (Postman, script khác...).
