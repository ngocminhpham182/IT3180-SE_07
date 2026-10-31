# BlueMoon Frontend — Sprint 1 (US-01, US-02, US-08, US-09, US-10)

Frontend độc lập bằng **React 18 + Vite 6 + React Router 6 + JavaScript + CSS**, giao diện **tiếng Việt**. Tích hợp theo `API_Contract_sprint1_gd1.md` v3. CSDL PostgreSQL nằm phía backend Spring Boot; frontend tuyệt đối không kết nối SQL trực tiếp.

> **Lưu ý khi ghép Git chung:** Chưa có repository frontend hiện tại từ nhóm, vì vậy bản giao nộp là module standalone có cấu trúc rõ ràng, **chưa** được tích hợp/đẩy vào repository thực tế. Không chép đè file đã có trong nhánh chung khi chưa so sánh.

## 1. Khởi chạy

Yêu cầu: Node.js >= 20, npm, backend Spring Boot phục vụ `/api/v1` theo contract. Tại thư mục này:

```bash
npm install
cp .env.example .env       # Windows PowerShell: Copy-Item .env.example .env
npm run dev
```

Truy cập: **http://localhost:5173/**.

- Dev mặc định: `.env` có `VITE_API_BASE_URL=/api/v1`; `vite.config.js` chuyển tiếp `/api/*` đến `http://localhost:8080`. Nếu backend ở địa chỉ khác, **sửa target proxy**.
- Nếu deploy khác domain: đặt `VITE_API_BASE_URL=https://your-backend.example/api/v1` và cấu hình CORS trong backend; không thêm token trong `.env`.
- **Không có tài khoản mẫu** được hardcode. Lấy tài khoản BQT seed từ backend nhóm.
- Chạy `npm run build` trước khi push; output nằm tại `dist/`, không commit `dist/`.
- BrowserRouter cần rewrite mọi URL frontend về `index.html` trên hosting (ngược lại khi refresh `/ho-gia-dinh/:id` sẽ 404).

### Khởi chạy khi chưa có backend

Các màn hình nghiệp vụ dùng dữ liệu thật từ Spring Boot. Nếu chưa chạy API, form đăng nhập sẽ báo lỗi kết nối. **Không có mock-server mặc định**, để tránh vô tình gửi demo giả là dữ liệu thực. Có thể xem wireframe màn hình dạng tĩnh tại `docs/wireframe-preview.html` mà không cần server.

## 2. Các route

| Route | Quyền | User Story | Chức năng |
|---|---|---|---|
| `/dang-nhap` | Công khai | US-08 | JWT, thông báo sai tài khoản/mật khẩu, khóa 15 phút, đếm ngược |
| `/` | Đăng nhập | Điều hướng | Truy cập nhanh theo vai trò, không làm số liệu giả |
| `/ho-gia-dinh` | BQT / ACCOUNTANT | US-01 | Tìm/lọc/phân trang, tạo hộ kèm chủ hộ (BQT) |
| `/ho-gia-dinh/:id` | BQT / ACCOUNTANT | US-01 + US-02 | Chi tiết hộ, đổi chủ hộ, sửa, ngừng hoạt động/xóa, danh sách thành viên và thêm nhân khẩu |
| `/nhan-khau` | BQT / ACCOUNTANT | US-02 | Tìm theo họ tên, lọc trạng thái/quá hạn, phân trang |
| `/nhan-khau/:id` | BQT / ACCOUNTANT | US-02 | Chi tiết, sửa thông tin, xóa nhập nhầm |
| `/tai-khoan` | BQT | US-09 | Cấp tài khoản, liên kết nhân khẩu nếu RESIDENT, sửa quyền, khóa/mở |
| `/ho-so` | Mọi vai trò | US-10 | Xem/sửa thông tin cá nhân, đổi mật khẩu |

`ACCOUNTANT` chỉ xem danh sách và chi tiết hộ/nhân khẩu (không thấy nút thay đổi); `RESIDENT` và `AUTHORITY` chỉ thấy `/` và `/ho-so` do API Sprint 1 không cấp tra cứu nghiệp vụ cho họ.

## 3. Cấu trúc source

```text
src/
  api/
    http.js                  # fetch, Bearer, envelope data, 204, ApiError
    services.js              # endpoint tập trung
  auth/AuthContext.jsx       # JWT, /me, phân quyền, logout
  components/
    UI.jsx                   # modal/form/error/table primitives
    Layout.jsx               # sidebar & header
    HouseholdForm.jsx        # hộ gia đình + chủ hộ
    ResidentForm.jsx         # nhân khẩu
    UserForm.jsx             # cấp tài khoản và vai trò
  pages/
    LoginPage.jsx            # US-08
    HomePage.jsx
    HouseholdsPage.jsx       # US-01
    HouseholdDetailPage.jsx  # US-01 + US-02
    ResidentsPage.jsx        # US-02
    ResidentDetailPage.jsx   # US-02
    UsersPage.jsx            # US-09
    ProfilePage.jsx          # US-10
  utils/
    format.js                # nhãn enum tiếng Việt, định dạng ngày/diện tích
    validation.js            # validate client theo API
  App.jsx                    # router & guards
  styles.css                 # toàn bộ responsive UI
```

## 4. API và các giới hạn PHẢI hiểu

- Mọi response thành công có body: `{ success, message, data }`. `src/api/http.js` trả thẳng `data` (tương ứng `response.data.data` với axios). `204 No Content` trả `null`.
- Form hiện lỗi field theo `errors` (kể cả `head.identityNumber`), hiện `message` chung theo API, không suy đoán business error.
- `401 TOKEN_EXPIRED`, `TOKEN_INVALID`, `UNAUTHORIZED` → xóa JWT và về login. `401 INVALID_CREDENTIALS` lúc login **không** coi là hết phiên.
- `423 ACCOUNT_LOCKED` → hiển thị thông báo và đếm ngược theo `details.remainingSeconds`.
- JWT lưu ở **sessionStorage** (chỉ phiên/tab, không nằm trong Git). Không có refresh token theo contract. Backend vẫn chịu trách nhiệm xác thực/kiểm tra phân quyền; route guard chỉ hỗ trợ UX.
- Đổi mật khẩu thành công → 204 → xóa phiên → yêu cầu đăng nhập bằng mật khẩu mới.
- Không triển khai US-03 (thay đổi cư trú/lịch sử) hoặc US-04 (tìm kiếm tổng hợp). US-02 chỉ nhập mới kiểu `entryType` đúng API, không tự sinh endpoint biến động.
- `PUT /households/{id}` **không** đổi chủ hộ; chỉ `PUT /households/{id}/head` mới làm điều đó.
- `PUT /residents/{id}` **không** đổi hộ, trạng thái cư trú hay chủ hộ.
- Khi xóa hộ gặp `HAS_TRANSACTIONS`, đề xuất **ngừng hoạt động**, không tự động xóa. Xóa nhân khẩu gặp `HAS_HISTORY`, `HAS_ACCOUNT`, `INVALID_NEW_HEAD` thì hiển thị hướng xử lý theo business rule.
- Với `RESIDENT`, `POST /users` đòi `residentId` đang `RESIDING` chưa gắn tài khoản; UI dùng `GET /residents?keyword=...&status=RESIDING`. Trong trường hợp sai/đã có tài khoản, backend trả lỗi field.

## 5. Ghép vào Git chung

Nếu nhóm **chưa có frontend**, có thể dùng nguyên thư mục này làm root frontend (sau khi xác nhận router/port/build).

Nếu nhóm **đã có frontend**, nên:
1. Tạo nhánh `feature/fe-us01-02-08-09-10` từ nhánh mới nhất; đọc cấu trúc React hiện tại.
2. So sánh dependency, router, AuthContext và base URL trước khi copy. Chỉ lấy module tương ứng, **không ghi đè package.json/router/CSS chung** nếu nhóm đã thiết kế.
3. Căn chỉnh quy chuẩn code/lint và thiết kế UI của nhóm, chạy `npm run build`, thử tất cả AC của `docs/TEST_PLAN.md`.
4. Commit và tạo Pull Request ghi rõ các endpoint đã tích hợp, các issue backend còn chặn.

## 6. Những thông tin còn cần từ nhóm để hoàn thiện tích hợp

1. Link repository hoặc ZIP frontend hiện tại (nếu đã có), tên branch và quy ước thư mục/React router.
2. Địa chỉ backend chạy được (`http://...`), cách seed tài khoản BQT, và xác nhận API Contract v3 đã triển khai hay chưa.
3. Backend xác nhận xử lý hai khác biệt `gender=OTHER` và `RETURN` trong SQL; xác nhận đã thay `//` bằng `--` trong migration.
4. Nếu nhóm đã có thiết kế chung (Figma/màu sắc/logo), cung cấp để đồng bộ trước khi merge.

Các điểm này **không được tự giả định**; xem `docs/INTEGRATION_BLOCKERS.md`.

## 7. Trạng thái kiểm thử khi bàn giao

Đã kiểm tra cú pháp **20 file JS/JSX** bằng TypeScript transpiler và chạy **8/8 bài test validation PASS** (`npm run check` không phụ thuộc vào package ngoài ở phần test). Việc `npm install` trong môi trường tạo file đã bị chặn bởi npm registry và cache không đủ, vì vậy **chưa kiểm tra được Vite production build hoặc E2E với Spring Boot**. Trước khi push/merge, chạy `npm install && npm run check && npm run build` trên máy nhóm và làm checklist `docs/TEST_PLAN.md`. Xem `docs/VERIFY_STATUS.md`.
