# Kiểm thử các chức năng bảo mật bằng Postman & SQL

Tài liệu gộp cách **kiểm thử tay** cho 5 chức năng bảo mật, dùng **Postman** (gọi API) và **SQL** (kiểm/chuẩn bị dữ liệu). Dành cho môi trường **dev local**.

Mỗi mục có: cách chuẩn bị, bước chạy, kết quả mong đợi, và câu SQL để xác nhận.

!!! warning "Chỉ chạy trên DB dev"
    Các câu `UPDATE`/`INSERT` bên dưới sửa thẳng dữ liệu để dựng tình huống test. **Không** chạy trên DB thật. DB dev hiện tại: `eKMapServer_dev` (xem `appsettings.json`).

---

## 0. Chuẩn bị chung {#setup}

### 0.1. Đăng nhập lấy phiên (bắt buộc cho hầu hết test)

App dùng **login cookie qua MVC**, không phải JWT. Trong Postman:

1. Tab **Settings** của request → **HTTP version = HTTP/1.x**. Settings (⚙) → **SSL verification = OFF**.
2. **GET** `https://localhost:5000/Account/Login` → Postman tự lưu cookie `XSRF-TOKEN`. Bấm **Cookies** → copy value của `XSRF-TOKEN`.
3. **POST** `https://localhost:5000/Account/Login`
      - Header `X-XSRF-TOKEN` = value vừa copy
      - Body **x-www-form-urlencoded**: `UsernameOrEmailAddress=admin`, `Password=<mật khẩu>`, `RememberMe=false`
      - **Không** tự set header `Cookie` — để Postman quản lý.
4. Thành công → nhận cookie `.AspNetCore.Identity.Application`. Từ đây mọi request dùng chung phiên; **POST** nào cũng kèm lại header `X-XSRF-TOKEN`.

### 0.2. Kết nối SQL

Trỏ SSMS/Azure Data Studio tới DB `eKMapServer_dev`. Cấu hình lưu ở `AbpSettings`; dữ liệu người dùng ở `AbpUsers`.

**Xem nhanh mọi setting bảo mật đang có trong DB:**
```sql
SELECT Name, Value, UserId, TenantId FROM AbpSettings
WHERE Name LIKE 'App.Security.%'
   OR Name LIKE 'App.UserManagement.%'
   OR Name LIKE 'App.Password%'
   OR Name LIKE 'Abp.Zero.UserManagement.UserLockOut.%'
ORDER BY Name;
```
> Không có row ⇒ đang dùng **giá trị mặc định** trong code (`AppSettingProvider`), chưa ai chỉnh qua UI.

---

## 1. Khóa tài khoản khi đăng nhập sai {#lockout}

Chi tiết cơ chế: [Khóa tài khoản khi đăng nhập sai](login-lockout-feature.md).

**Tham số** (AbpSettings):

| Setting | Ý nghĩa | Mặc định |
|---|---|---|
| `Abp.Zero.UserManagement.UserLockOut.IsEnabled` | Bật khóa | `true` |
| `Abp.Zero.UserManagement.UserLockOut.MaxFailedAccessAttemptsBeforeLockout` | Số lần sai tối đa | `5` |
| `Abp.Zero.UserManagement.UserLockOut.DefaultAccountLockoutSeconds` | Thời gian khóa | `900` (15 phút) |
| `App.UserManagement.UserLockOut.FailedAttemptWindowSeconds` | Cửa sổ gộp các lần sai | *(cấu hình riêng)* |

### Postman

Gọi `POST /Account/Login` (như [§0.1](#setup)) với **mật khẩu sai** liên tục cho một tài khoản test (`user_test`):

| Lần | Kết quả |
|---|---|
| 1 → (Max−1) | Sai mật khẩu — trả lỗi "sai tên hoặc mật khẩu" |
| Lần thứ Max | Bị khóa — thông báo tài khoản bị khóa |
| Ngay sau đó, **dù nhập ĐÚNG** mật khẩu | Vẫn bị khóa cho tới khi hết `DefaultAccountLockoutSeconds` |

### SQL xác nhận

```sql
SELECT UserName, AccessFailedCount, IsLockoutEnabled,
       LockoutEndDateUtc, LastFailedLoginAttemptTime
FROM AbpUsers WHERE UserName = 'user_test';
```
- `AccessFailedCount` tăng dần theo mỗi lần sai.
- Khi đạt Max: `LockoutEndDateUtc` = thời điểm mở khóa (UTC).
- Đăng nhập **đúng** sau khi hết khóa ⇒ `AccessFailedCount` reset `0`.

### Dựng nhanh / gỡ khóa bằng SQL

```sql
-- Ép khóa ngay để test màn hình bị khóa:
UPDATE AbpUsers SET AccessFailedCount = 5,
       LockoutEndDateUtc = DATEADD(MINUTE, 15, GETUTCDATE())
WHERE UserName = 'user_test';

-- Gỡ khóa:
UPDATE AbpUsers SET AccessFailedCount = 0, LockoutEndDateUtc = NULL,
       LastFailedLoginAttemptTime = NULL
WHERE UserName = 'user_test';
```

---

## 2. Hết hạn mật khẩu & đổi mật khẩu bắt buộc {#password}

Chi tiết: [Hết hạn mật khẩu & Đổi mật khẩu bắt buộc](password-expiration-feature.md).

**Tham số**: `App.PasswordExpirationDays` (mặc định **90**), `App.PasswordChangeReminderDays`.
Cột liên quan trên `AbpUsers`: `LastPasswordChangeTime`, `ChangePassWordForLogInNext`.

### Cách test (dựng bằng SQL rồi login)

Đẩy lùi `LastPasswordChangeTime` quá hạn để ép hết hạn ở lần đăng nhập kế tiếp:

```sql
-- Cho mật khẩu "hết hạn": đặt mốc đổi cách đây 100 ngày (> 90)
UPDATE AbpUsers SET LastPasswordChangeTime = DATEADD(DAY, -100, GETDATE())
WHERE UserName = 'user_test';
```

Rồi `POST /Account/Login` bằng `user_test`:

| Tình huống | Mong đợi |
|---|---|
| `LastPasswordChangeTime` quá `PasswordExpirationDays` | Login trả `targetUrl` = trang **ForceChangePassword** (bị ép đổi) |
| `ChangePassWordForLogInNext = 1` | Cũng bị ép đổi ngay lần đăng nhập đầu |
| Trong hạn | Đăng nhập bình thường |

### Ép đổi mật khẩu lần đăng nhập tới (cách 2)

```sql
UPDATE AbpUsers SET ChangePassWordForLogInNext = 1 WHERE UserName = 'user_test';
```

### SQL xác nhận / trả về bình thường

```sql
SELECT UserName, LastPasswordChangeTime, ChangePassWordForLogInNext
FROM AbpUsers WHERE UserName = 'user_test';

-- Trả về "vừa đổi mật khẩu", hết ép:
UPDATE AbpUsers SET LastPasswordChangeTime = GETDATE(),
       ChangePassWordForLogInNext = 0
WHERE UserName = 'user_test';
```

!!! note "Vì sao NULL nguy hiểm"
    `LastPasswordChangeTime = NULL` rơi về `CreationTime`; nếu user cũ tạo lâu rồi thì bị coi là **đã hết hạn** ngay. Khi test nhớ set mốc rõ ràng.

---

## 3. Tự động đóng phiên khi không hoạt động {#session}

Chi tiết: [Tự động đóng phiên khi không hoạt động](session-timeout-feature.md).

**Tham số** (AbpSettings):

| Setting | Ý nghĩa |
|---|---|
| `App.UserManagement.SessionTimeOut.IsEnabled` | Bật tự đóng phiên |
| `App.UserManagement.SessionTimeOut.TimeOutSecond` | Ngưỡng nhàn rỗi trước khi logout |
| `App.UserManagement.SessionTimeOut.ShowTimeOutNotificationSecond` | Bao lâu trước đó thì cảnh báo |

Đây chủ yếu là hành vi **phía client (Angular)**: đếm nhàn rỗi, cảnh báo, rồi logout. Server chỉ là **lưới an toàn**: cookie xác thực đặt `ExpireTimeSpan = 10 giờ`, `SlidingExpiration`.

### Cách test

- **Đúng luồng (client):** đăng nhập trên browser, hạ `TimeOutSecond` xuống nhỏ (vd 60s) trong Settings, để yên không thao tác → phải thấy cảnh báo rồi bị đăng xuất. Postman không tái hiện được phần này (không có JS đếm nhàn rỗi).
- **Lưới an toàn server (Postman):** sau khi login, dùng cookie gọi một API cần đăng nhập → 200. Kiểm cookie `.AspNetCore.Identity.Application` có thời hạn ~10h (backstop) trong tab Cookies của Postman.

### SQL xác nhận cấu hình

```sql
SELECT Name, Value FROM AbpSettings
WHERE Name LIKE 'App.UserManagement.SessionTimeOut.%';
```

---

## 4. Phân loại nhật ký hệ thống theo 05 nhóm {#syslog}

Chi tiết: [Phân loại nhật ký hệ thống theo 05 nhóm](system-log-classification-feature.md).

Đọc log qua API `SystemLog/GetLogs`. Cần quyền `Administration.Audit` (đăng nhập admin). `category` là bắt buộc, 1 nhóm một truy vấn:

| `category` | Nhóm | Nguồn dữ liệu |
|---|---|---|
| `1` | i. Truy cập phần mềm | `AbpAuditLogs` |
| `2` | ii. Đăng nhập quản trị | `AbpUserLoginAttempts` (lọc user có vai trò admin) |
| `3` | iii. Lỗi phát sinh | `AbpAuditLogs` có `Exception` |
| `4` | iv. Quản lý tài khoản | `AbpEntityChanges` (User/Role/UserRole) |
| `5` | v. Thay đổi cấu hình | `AbpEntityChanges` (`Abp.Configuration.Setting`) |

### Postman

```
POST https://localhost:5000/api/services/app/SystemLog/GetLogs
Header: X-XSRF-TOKEN: <value cookie XSRF-TOKEN>
Body (JSON):
{
  "category": 1,
  "startDate": "2026-07-20T17:00:00.000Z",
  "endDate":   "2026-07-21T16:59:59.999Z",
  "userName": "",
  "clientIpAddress": "",
  "isSuccess": null,
  "includeUnknownUserAttempts": false,
  "maxResultCount": 10,
  "skipCount": 0
}
```
Đổi `category` 1→5 để xem từng nhóm. Với nhóm ii, bật `includeUnknownUserAttempts: true` để thấy cả lần đăng nhập **sai tên tài khoản**.

### Kịch bản sinh dữ liệu rồi kiểm

| Nhóm | Thao tác tạo dữ liệu | Mong đợi khi gọi GetLogs |
|---|---|---|
| ii | Đăng nhập admin đúng, rồi sai mật khẩu | 2 bản ghi (`Success`, `InvalidPassword`) |
| iii | Gọi một API gây exception | Vào nhóm iii, không trùng nhóm i |
| iv | Đổi email một user | Nhóm iv: `Detail` = `EmailAddress: cũ → mới` |
| iv ⭐ | Đăng nhập sai mật khẩu vài lần rồi mở nhóm iv | **Không** thấy bản ghi `AccessFailedCount` (bộ lọc nhiễu hoạt động) |
| v | Đổi một setting ở màn hình Settings | Nhóm v: `Setting` `Value: cũ → mới` |
| Phân quyền | Đăng nhập user **không** có `Administration.Audit` gọi API | **403** |

### SQL xác nhận (dữ liệu thô)

```sql
-- Nhóm ii: đăng nhập (có sẵn từ trước)
SELECT TOP 20 CreationTime, UserNameOrEmailAddress, UserId, Result, ClientIpAddress
FROM AbpUserLoginAttempts ORDER BY CreationTime DESC;

-- Nhóm iv/v: thay đổi thực thể (giá trị cũ → mới)
SELECT TOP 20 cs.CreationTime, cs.UserId, cs.ClientIpAddress,
       c.EntityTypeFullName, c.ChangeType, p.PropertyName, p.OriginalValue, p.NewValue
FROM AbpEntityChangeSets cs
JOIN AbpEntityChanges c ON c.EntityChangeSetId = cs.Id
LEFT JOIN AbpEntityPropertyChanges p ON p.EntityChangeId = c.Id
ORDER BY cs.CreationTime DESC;

-- Nhóm i/iii: audit log
SELECT TOP 20 ExecutionTime, ServiceName, MethodName, UserId, ClientIpAddress,
       CASE WHEN Exception IS NULL THEN 'OK' ELSE 'ERROR' END AS Loai
FROM AbpAuditLogs ORDER BY ExecutionTime DESC;
```

---

## 5. Giới hạn địa chỉ mạng quản trị {#admin-ip}

Chi tiết: [Giới hạn địa chỉ mạng được phép quản trị từ xa](admin-ip-restriction-feature.md).

**Cái bẫy trên localhost:** IP client luôn là loopback, và header `X-Forwarded-For` **bị bỏ qua** để chống giả mạo — trừ khi khai loopback là proxy tin cậy. Vì vậy phải bật `KnownProxies` mới giả được IP.

### Bước 1 — cho phép giả IP (chỉ dev)

`eKMapServer.Web.Mvc/appsettings.json`, rồi **restart**:
```json
"ForwardedHeaders": { "KnownProxies": [ "127.0.0.1", "::1" ] }
```

### Bước 2 — bật chính sách, GIỮ van localhost

Trong Administrator → cài đặt bảo mật: bật chính sách, **giữ "Luôn cho phép localhost"**, thêm rule `203.0.113.0/24` (mô tả `test`, active). Lưu được vì IP thật của bạn là loopback (van localhost cho qua) nên guard chống tự khoá không cản.

### Bước 3 — test bằng `X-Forwarded-For` (Postman)

| Request | `X-Forwarded-For` | Mong đợi |
|---|---|---|
| `GET /Administrator` | `8.8.8.8` | ❌ **403** "Your network address is not allowed…" |
| `GET /Administrator` | `203.0.113.10` | ✅ 200 |
| `POST /api/services/app/SystemLog/GetLogs` | `8.8.8.8` | ❌ 403 JSON |
| `GET /Administrator` | *(không gửi)* | ✅ 200 (IP thật = loopback) |
| `GET /ekmapserver/...` | `8.8.8.8` | ✅ 200 — dịch vụ GIS không vạ lây |

### SQL xác nhận cấu hình

```sql
SELECT Name, Value FROM AbpSettings
WHERE Name LIKE 'App.Security.AdminIpRestriction.%';
```

### Dọn dẹp (quan trọng)

1. Trả `KnownProxies` về `[]` + restart (để `["127.0.0.1","::1"]` trên môi trường thật là **hở đường giả mạo IP**).
2. Tắt chính sách hoặc xoá rule test.
3. Nếu lỡ tự khoá: đặt `"AdminIpRestriction": { "Disabled": true }` + restart để bỏ qua middleware, vào sửa lại, rồi trả về `false`.

---

## 6. Bảng tra nhanh {#cheatsheet}

| Chức năng | Test bằng Postman | Kiểm/dựng bằng SQL |
|---|---|---|
| Khóa tài khoản | Login sai N lần | `AbpUsers`: `AccessFailedCount`, `LockoutEndDateUtc` |
| Hết hạn mật khẩu | Login sau khi lùi mốc | `AbpUsers`: `LastPasswordChangeTime`, `ChangePassWordForLogInNext` |
| Tự đóng phiên | Chủ yếu trên browser | `AbpSettings` `...SessionTimeOut.*` |
| Nhật ký 05 nhóm | `POST SystemLog/GetLogs` (category 1–5) | `AbpUserLoginAttempts`, `AbpEntityChanges`, `AbpAuditLogs` |
| Giới hạn IP quản trị | `X-Forwarded-For` + `KnownProxies` | `AbpSettings` `...AdminIpRestriction.*` |

**Nhớ chung:** Postman để **HTTP/1.x** + **SSL off**; mọi POST kèm header `X-XSRF-TOKEN`; đừng set tay header `Cookie`.
