# Tài liệu kỹ thuật eKMapServer

Nơi tập hợp báo cáo tính năng và sự cố kỹ thuật của hệ thống.

## Tính năng

### [Hết hạn mật khẩu & Đổi mật khẩu bắt buộc](password-expiration-feature.md)

Ép người dùng đổi mật khẩu ở lần đăng nhập đầu tiên (tài khoản mặc định) và khi mật khẩu quá hạn (`PasswordExpirationDays`, mặc định 90 ngày), kèm cảnh báo trước từ mốc `PasswordChangeReminderDays` (mặc định 80 ngày).

Tài liệu gồm: yêu cầu, bối cảnh kiến trúc, các quyết định thiết kế kèm lý do, luồng hoạt động, danh sách file, quy trình triển khai và kịch bản kiểm thử.

!!! warning "Còn điểm chặn trước khi lên môi trường thật"
    Migration **chưa được apply**, và **phương án backfill chưa được chốt**. Nếu apply migration mà không backfill, mọi user được tạo cách đây hơn `PasswordExpirationDays` ngày sẽ bị ép đổi mật khẩu ngay ở lần đăng nhập kế tiếp — kể cả người vừa đổi mật khẩu tuần trước. Xem [mục 6 bước 2](password-expiration-feature.md#backfill).

### [Khóa tài khoản tạm thời khi đăng nhập sai](login-lockout-feature.md)

Khóa quyền truy cập tạm thời khi một tài khoản nhập sai mật khẩu quá số lần quy định trong một cửa sổ thời gian (`FailedAttemptWindowSeconds`, mặc định 10 phút), tận dụng cơ chế lockout sẵn có của ABP và bổ sung phần cửa sổ thời gian mà ABP còn thiếu.

Tài liệu gồm: yêu cầu, khoảng trống so với ABP, các quyết định thiết kế (chokepoint `UserManager.AccessFailedAsync`, ngữ nghĩa cửa sổ), luồng hoạt động, danh sách file, triển khai và kiểm thử.

!!! note "Migration chưa apply"
    Code đã hoàn thành và build sạch. Migration `20260715022326_Add_LastFailedLoginAttemptTime` còn ở trạng thái `Pending` — chạy `dotnet ef database update` trước khi khởi động code mới.

### [Tự động đóng phiên khi không hoạt động (Session Timeout)](session-timeout-feature.md)

Đóng phiên và yêu cầu đăng nhập lại khi người dùng không thao tác quá `TimeOutSecond`, có thông báo cảnh báo trước từ mốc `ShowTimeOutNotificationSecond`. Idle-timer chạy ở app Angular Administrator, nhưng phiên thật là **cookie do server giữ** — hết giờ thì điều hướng `/Account/Logout` để server hủy cookie; admin cấu hình thời gian, user thường chỉ chịu tác dụng.

Tài liệu gồm: yêu cầu, bối cảnh phiên cookie (Angular ăn theo cookie MVC, không phải JWT), các quyết định thiết kế (phạm vi, đường đọc cấu hình qua `GetCurrentLoginInformations`, ngữ nghĩa timeout, đồng bộ đa tab), luồng hoạt động, danh sách file dự kiến, triển khai và kiểm thử.

!!! warning "Đề xuất — chưa triển khai"
    Backend đã có sẵn phần lưu/đọc 3 tham số cấu hình, nhưng **chưa có nơi tiêu thụ** (idle timer) và **chưa có ô nhập** trên trang Settings. Tài liệu chốt thiết kế trước khi code.

### [Giới hạn địa chỉ mạng được phép quản trị từ xa (Admin IP Allowlist)](admin-ip-restriction-feature.md)

Chỉ cho phép quản trị Phần mềm từ những địa chỉ/dải mạng đã khai báo (IP đơn hoặc CIDR), cưỡng chế bằng middleware trong pipeline `Web.Mvc`, cấu hình qua bảng CRUD ở trang Settings. Chính sách **chỉ áp lên hành vi quản trị** — dịch vụ GIS (`/ekmapserver`, xác thực bằng apikey) và người dùng Manager **không bị ảnh hưởng**.

Báo cáo mô tả chức năng, **trình bày bằng sơ đồ**: bản đồ code (cái gì nằm ở đâu), luồng cưỡng chế end-to-end (một request đi qua những trạm nào), luồng cấu hình (bấm Lưu thì chuyện gì xảy ra), cơ chế khớp IP trên bit, ba lớp chống tự khoá, và hai cạm bẫy mạng (reverse proxy, NAT).

✅ **Đã triển khai** — 52 unit test pass, không migration, không cần chạy SQL.

!!! danger "Một câu hỏi chưa có lời đáp — phải hỏi bên vận hành trước khi bật"
    **Hệ thống có chạy sau reverse proxy không?** Nếu có nginx/IIS ARR mà không khai `ForwardedHeaders:KnownProxies`, mọi request đều mang IP của proxy ⇒ allowlist hoặc chặn sạch, hoặc cho qua tất cả — **mà vẫn báo cáo là đã bật**. Mặc định trong repo là mảng rỗng (an toàn cho triển khai không proxy). Xem [§9.1](admin-ip-restriction-feature.md#cam-bay).

!!! warning "Rủi ro tự khoá là cái giá của mô hình allowlist"
    Vì mặc định là **chặn** (default-deny), khai thiếu IP của chính mình ⇒ mất quyền quản trị. Đã có ba lớp chống: guard ở server khi lưu, loopback luôn mở, van trong `appsettings.json` — [§7](admin-ip-restriction-feature.md#self-lockout).

### [Phân loại nhật ký hệ thống theo 05 nhóm](system-log-classification-feature.md)

Phân loại nhật ký thành 5 nhóm: truy cập, đăng nhập quản trị, lỗi phát sinh, quản lý tài khoản, thay đổi cấu hình. Kiểm kê 6 nguồn log đang có, đối chiếu với yêu cầu, và chốt thiết kế: phân nhóm bằng **truy vấn lúc đọc** chứ không tạo bảng log hợp nhất.

Tài liệu gồm: hiện trạng từng nguồn log, khoảng trống so với yêu cầu, các quyết định thiết kế kèm lý do, ánh xạ từng nhóm về nguồn dữ liệu, API/UI đề xuất, danh sách file, rủi ro và kiểm thử.

!!! warning "Đề xuất — chưa triển khai"
    3/5 nhóm đã đủ dữ liệu thô. Hai nhóm còn lại (quản lý tài khoản, thay đổi cấu hình) cần **khai báo `EntityHistory.Selectors`** — toàn bộ đường ống ABP đã lắp sẵn, 3 bảng `AbpEntityChange*` đã có trong DB, **không cần migration**. Xem [§4.1](system-log-classification-feature.md#entityhistory).

!!! danger "Hai điều phải chốt trước khi bắt tay"
    **1. EntityHistory không hồi tố.** Bật hôm nay thì nhóm iv/v rỗng cho tới khi có thay đổi mới — nên [§10](system-log-classification-feature.md#order) xếp việc này làm **trước tiên**, độc lập với phần còn lại.

    **2. Đây là audit tầng ứng dụng, không phải tầng DB.** Hook ở `SaveChanges`, nên sửa thẳng bằng SSMS/`ExecuteSqlRaw` **không để lại vết**. Phải ghi rõ giới hạn này trong hồ sơ nghiệm thu — xem [§2.3](system-log-classification-feature.md#entityhistory-limit).

## Hướng dẫn sử dụng

### [Tab Giao diện — tuỳ biến trang Đăng nhập](interface-tab-guide.md)

Hướng dẫn cho **người quản trị** (không cần biết lập trình): mỗi ô trong tab **Giao diện** điền được giá trị gì, và trang Đăng nhập đổi ra sao. Kèm thư viện CSS dán sẵn (bo tròn, đổ bóng, xoay, ẩn/hiện theo màn hình, chữ gradient…) và hướng dẫn dùng ảnh động/GIF.

!!! danger "Cơ chế lưu là \"tất cả hoặc không có gì\""
    Bấm **Lưu** sẽ ghi **đúng những gì đang có trên form** — ô để trống = **rỗng**, *không* tự quay về mặc định. Chỉ điền 1 ô rồi lưu ⇒ **mất logo/ảnh nền/slogan**. Muốn đổi vài ô: điền đầy đủ, hoặc bấm **Khôi phục mặc định** trước rồi chỉnh.

!!! note "Bốn ô điền được nhưng chưa có tác dụng"
    Favicon URL, Primary Color, Secondary Color, Font Family — lưu được nhưng **chưa làm đổi trang Login**. Đã gắn nhãn 🏷️ *Chưa áp dụng* trong giao diện.

## Vận hành

### [Cập nhật DB đang chạy lên schema hiện tại](db-update-existing-database.md)

Liệt kê đúng những gì code thêm mới, kèm script idempotent `Test by SQL/update_db_to_current.sql` (kiểm tra chỉ-đọc → cập nhật trong transaction → hậu kiểm) để đưa vào DB thật **mà không đụng dữ liệu đang có**.

Delta rất nhỏ: **3 cột trên `AbpUsers` + bảng `tenantBranding`**. Chỉ `ADD`/`CREATE`, không `ALTER COLUMN`, không `DROP`. `AbpSettings` không đổi schema.

!!! danger "Hai cái bẫy — cả hai đều vô hình nếu chỉ nhìn danh sách migration"
    **1. `AbpUsers.ChangePassWordForLogInNext` không có migration.** Nó bị nướng thẳng vào `InitialBaseline` — mà baseline thì chỉ được *đóng dấu*, không *chạy*. `AccountController.cs:153` đọc cột này ở mỗi lần đăng nhập ⇒ thiếu nó là **không ai đăng nhập được**. `dotnet ef database update` không cứu được, vì theo sổ migration thì mọi thứ "đã up to date".

    **2. `LastPasswordChangeTime` phải backfill.** `NULL` rơi về `CreationTime` (`PasswordExpirationChecker.cs:37`), mặc định hết hạn 90 ngày và bật sẵn ⇒ thêm cột mà không backfill thì **toàn bộ user bị ép đổi mật khẩu** ngay lần đăng nhập kế tiếp.

    Đây là lý do phải soát delta từ **code**, không chỉ từ migration.

## Sự cố kỹ thuật

### [Migration thêm 1 cột nhưng sinh thêm 41 lệnh `AlterColumn` thừa](migration-identity-columns-issue.md)

`dotnet ef migrations add` sinh ra migration đụng vào cột `Id` của gần như mọi bảng trong database, do file `ModelSnapshot` trong repo bị lệch so với schema thật.

Báo cáo phân tích nguyên nhân gốc, rủi ro khi chạy trên production, cách đã xử lý và khuyến nghị phòng ngừa.

## Chạy tài liệu này

```bash
mkdocs serve      # xem tại http://127.0.0.1:8000
mkdocs build      # xuất HTML tĩnh ra thư mục site/
```
