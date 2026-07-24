# Xác thực 2 bước (2FA) — nghiên cứu & thiết kế

**Ngày:** 2026-07-24 · **Trạng thái:** 📐 Đề xuất — chốt thiết kế trước khi code · **Mức độ:** 🔴 Cao (đụng thẳng luồng đăng nhập)

!!! abstract "Tóm tắt một dòng"
    Hạ tầng 2FA của ABP **đã có sẵn gần đủ** (cột DB, setting, service sinh/kiểm mã), nhưng **cổng đăng nhập đã bị viết lại và bỏ mất bước thử thách mã**. Việc cần làm không phải "xây từ đầu" mà là **nối lại 3 mắt xích còn thiếu** — theo đúng bản mẫu có sẵn trong dự án là luồng `ForceChangePassword`.

!!! info "Đây là hạng mục BẮT BUỘC"
    2FA nằm trong tài liệu bảo vệ dữ liệu của dự án ⇒ phải triển khai, không phải tùy chọn. **Áp dụng cho mọi tài khoản, gồm cả admin** — hạng mục tuân thủ bắt buộc không thể miễn cho tài khoản quyền cao nhất; miễn admin là điểm trừ nặng khi đánh giá. Đường khôi phục cho admin (§8, §10 bước 1) là cái làm cho việc "admin cũng phải bật" trở nên an toàn.

---

## 1. 2FA là gì (nhắc nhanh)

Xác thực dựa trên việc chứng minh 2 **yếu tố khác loại**:

```mermaid
flowchart LR
    A["🔑 Yếu tố 1: BIẾT<br/>mật khẩu"] --> C{"Đăng nhập"}
    B["📱 Yếu tố 2: CÓ<br/>mã từ điện thoại"] --> C
    C --> D["✅ Vào hệ thống"]

    style B fill:#e3f2fd,stroke:#1565c0
```

Kẻ trộm được mật khẩu vẫn chưa vào được vì thiếu điện thoại. Ba phương thức phổ biến, xếp theo độ ưu tiên đề xuất:

| Phương thức | Cần gì | Đánh giá cho dự án này |
|---|---|---|
| **Google Authenticator (TOTP)** | App trên điện thoại, không cần mạng | ⭐ Nên chọn đầu tiên — miễn phí, không phụ thuộc hạ tầng ngoài |
| **Email OTP** | SMTP hoạt động | Khả thi, nhưng SMTP hiện trỏ `127.0.0.1:25` — chưa chắc gửi được thật |
| **SMS OTP** | Cổng SMS trả phí | Chưa có nhà cung cấp — để sau |

---

## 2. Hiện trạng: khung có sẵn, dây bị đứt

### Cái ĐÃ có

```mermaid
flowchart TD
    subgraph DB["🗄️ Tầng dữ liệu — KHÔNG cần migration"]
        A["AbpUser.IsTwoFactorEnabled<br/><i>cờ user đã bật 2FA chưa — LÀ cột thật</i>"]
        B["Khoá TOTP lưu ở token store Identity<br/>(bảng AbpUserTokens), KHÔNG phải cột riêng<br/><i>truy cập qua *AuthenticatorKeyAsync</i>"]
    end
    subgraph SET["⚙️ Tầng cấu hình — vừa thêm UI ở Settings"]
        C["twoFactorLogin.isEnabled + các provider<br/>HostSettingsAppService.cs:567 lưu/đọc đủ"]
    end
    subgraph SVC["🔧 Tầng service — ABP 6.0 ZeroCore có sẵn"]
        D["UserManager: sinh mã, kiểm mã TOTP<br/>SignInManager: TwoFactorSignInAsync…"]
    end

    style A fill:#c8e6c9,stroke:#2e7d32
    style B fill:#c8e6c9,stroke:#2e7d32
    style C fill:#c8e6c9,stroke:#2e7d32
    style D fill:#c8e6c9,stroke:#2e7d32
```

`User` kế thừa `AbpUser` (`User.cs:8`) nên `IsTwoFactorEnabled` có sẵn trong bảng `AbpUsers`, còn khoá TOTP nằm ở bảng `AbpUserTokens` (đã có sẵn trong schema ABP.Zero) — **không thêm cột, không migration**.

!!! warning "Đính chính (phát hiện lúc code Bước 1)"
    Bản đầu tài liệu này ghi "`AbpUser.GoogleAuthenticatorKey` là cột lưu khoá" — **SAI** với ABP 6.0 đang dùng. Compiler bắt lỗi ngay khi thử gán `user.GoogleAuthenticatorKey`: property đó không tồn tại. Sự thật: khoá authenticator lưu qua **token store của Identity**, thao tác bằng `GenerateNewAuthenticatorKeyAsync` / `GetAuthenticatorKeyAsync` / `ResetAuthenticatorKeyAsync` (Bước 2 dùng tới). Đây đúng là loại chi tiết mà [note cuối §10](#ke-hoach) đã dặn "phải đối chiếu lại với source khi code" — và compiler chính là cái đối chiếu đó.

### Cái ĐANG thiếu — vì sao 2FA hiện không chạy

Cổng đăng nhập đã được viết lại (thêm ép đổi mật khẩu, kiểm hết hạn) và trong lúc đó **đánh rơi** bước thử thách 2FA:

```mermaid
flowchart TD
    A["Nhập mật khẩu"] --> B["GetLoginResultAsync<br/>mật khẩu đúng?"]
    B --> C["ChangePassWordForLogInNext?<br/>AccountController.cs:153"]
    C --> D["Mật khẩu hết hạn?<br/>:158"]
    D --> X["❌ THIẾU: user có bật 2FA không?<br/>nếu có → thử thách mã"]
    X --> E["SignInAsync — VÀO THẲNG<br/>:164"]

    style X fill:#ffcdd2,stroke:#c62828,stroke-width:3px
    style E fill:#fff9c4,stroke:#f57f17
```

Đã grep toàn bộ `AccountController`: **không có** `TwoFactorSignInAsync`, `RequiresTwoFactor`, hay `GenerateTwoFactorToken`. Công tắc trong Settings hiện là **nút không nối dây** — bật lên chỉ lưu cấu hình.

---

## 3. Ba mắt xích phải nối

```mermaid
flowchart LR
    subgraph E["① ENROLL — user tự bật 2FA"]
        direction TB
        E1["Angular: nút Bật 2FA trong<br/>menu tài khoản → quét QR →<br/>nhập mã xác nhận lần đầu"]
    end
    subgraph C["② CHALLENGE — thử thách khi đăng nhập"]
        direction TB
        C1["MVC: chèn bước hỏi mã vào<br/>AccountController.Login,<br/>giữa 'mật khẩu đúng' và SignInAsync"]
    end
    subgraph R["③ RECOVERY — khôi phục khi mất máy"]
        direction TB
        R1["Admin gỡ 2FA hộ user<br/>(tối thiểu), hoặc mã dự phòng"]
    end

    E --> C
    C -.->|"thiếu ② thì ① vô nghĩa"| R

    style C fill:#ffcdd2,stroke:#c62828,stroke-width:2px
```

!!! danger "Thứ tự làm KHÔNG được tùy tiện"
    **Mảnh ② (challenge) phải xong trước hoặc cùng lúc với ① (enroll).** Nếu chỉ làm trang enroll, user quét QR và tưởng đã được bảo vệ, trong khi đăng nhập vẫn chỉ cần mật khẩu → **an toàn giả**, đúng dạng lỗi "báo thành công nhưng không có tác dụng" như bug đặt lại mật khẩu. Còn ③ là điều kiện bắt buộc để dám bật thật: không có đường khôi phục thì mất điện thoại = mất tài khoản vĩnh viễn.

---

## 4. Vị trí code: enroll ở Angular, challenge ở MVC

Điểm dễ nhầm nhất về kiến trúc. Đăng nhập là **MVC/Razor** (`/Account/Login`), Angular chỉ nạp **sau khi** đã đăng nhập xong. Do đó 2 mảnh nằm ở 2 nơi khác nhau:

```mermaid
flowchart TD
    subgraph M["🌐 MVC — trước khi vào app"]
        L["Trang Login (Razor)"] --> CH["Trang nhập mã 2FA (Razor)<br/><i>MỚI — giống ForceChangePassword.cshtml</i>"]
        CH --> APP
    end
    subgraph A["🅰️ Angular — sau khi đã đăng nhập"]
        APP["Administrator app"] --> EN["Modal Bật 2FA + quét QR<br/><i>MỚI — cạnh nút Đổi mật khẩu trong<br/>user-offcanvas</i>"]
        APP --> UN["Nút admin gỡ 2FA của user<br/><i>MỚI — trang Người dùng</i>"]
    end

    style CH fill:#ffe0b2,stroke:#e65100
    style EN fill:#e3f2fd,stroke:#1565c0
    style UN fill:#e3f2fd,stroke:#1565c0
```

| Mắt xích | Tầng | File dự kiến |
|---|---|---|
| ② Challenge (trang nhập mã) | MVC | `AccountController` + `Views/Account/TwoFactorLogin.cshtml` + `.js` |
| ① Enroll (quét QR, xác nhận) | Angular + App service | `user-offcanvas/` + API mới trong `Application` |
| ③ Recovery (admin gỡ) | Angular + App service | trang Người dùng + API mới |

---

## 5. Bản mẫu có sẵn: luồng `ForceChangePassword`

Đây là chìa khóa khiến mảnh ② **không phải phát minh gì mới**. Dự án đã có sẵn một "bước trung gian dựa trên session chèn vào giữa login" — chính là ép đổi mật khẩu. 2FA đi theo y hệt khuôn đó:

```mermaid
flowchart TD
    A["Mật khẩu đúng"] --> B["StartForceChangePassword<br/>:770 — LƯU userId vào session,<br/>KHÔNG SignInAsync"]
    B --> C["TargetUrl → trang ForceChangePassword"]
    C --> D["User thao tác trên trang trung gian"]
    D --> E["POST về, xác minh, xong<br/>→ mới cho đi tiếp"]

    style B fill:#e8f5e9,stroke:#2e7d32,stroke-width:2px
```

Ánh xạ sang 2FA — cùng bộ khung, chỉ đổi nội dung bước giữa:

| ForceChangePassword (đã có) | TwoFactorLogin (làm mới) |
|---|---|
| `StartForceChangePassword` lưu `ForcePwdUserId` vào session | `StartTwoFactor` lưu `TwoFactorUserId` vào session |
| `TryGetForceChangePasswordUserId` đọc + kiểm hết hạn 10 phút | `TryGetTwoFactorUserId` — copy nguyên |
| Trang `ForceChangePassword.cshtml` + `.js` | Trang `TwoFactorLogin.cshtml` + `.js` — copy khung |
| POST xác minh mật khẩu cũ → `UpdateAsync` | POST xác minh **mã 6 số** → `SignInAsync` |
| `ClearForceChangePasswordSession` | `ClearTwoFactorSession` |

!!! tip "Vì sao dùng session, không dùng cookie đăng nhập"
    Giữa 2 bước, user **chưa được đăng nhập** — mới chỉ chứng minh yếu tố 1. Nếu `SignInAsync` sớm rồi mới hỏi mã thì kẻ tấn công đã có cookie hợp lệ, thử thách thành vô nghĩa. Session giữ trạng thái "đang chờ mã" đúng như `ForcePwdUserId` giữ "đang chờ đổi mật khẩu". Có hạn 10 phút để không kẹt vĩnh viễn.

---

## 6. Luồng đăng nhập sau khi nối 2FA

```mermaid
flowchart TD
    A["POST /Account/Login<br/>user + mật khẩu"] --> B{"Mật khẩu đúng?"}
    B -->|"không"| B1["Báo lỗi / khóa tài khoản"]
    B -->|"đúng"| C{"ChangePassWordForLogInNext?"}
    C -->|"có"| C1["→ ForceChangePassword"]
    C -->|"không"| D{"Mật khẩu hết hạn?"}
    D -->|"có"| D1["→ ForceChangePassword"]
    D -->|"không"| E{"2FA bật ở hệ thống<br/>VÀ user.IsTwoFactorEnabled?"}
    E -->|"không"| G["SignInAsync — vào thẳng<br/><i>hành vi cũ, không đổi</i>"]
    E -->|"có"| F["StartTwoFactor:<br/>lưu userId vào session<br/>→ trang nhập mã"]
    F --> H["User nhập mã 6 số"]
    H --> I{"VerifyTwoFactorToken<br/>đúng?"}
    I -->|"sai"| H
    I -->|"đúng"| J{"Tick 'ghi nhớ<br/>thiết bị này'?"}
    J -->|"có"| J1["Đặt cookie tin cậy<br/>→ lần sau bỏ qua mã"]
    J -->|"không"| G
    J1 --> G

    style F fill:#ffe0b2,stroke:#e65100
    style E fill:#e3f2fd,stroke:#1565c0
    style G fill:#c8e6c9,stroke:#2e7d32
```

Điểm mấu chốt: nhánh `E → không → G` giữ **nguyên hành vi cũ**. User chưa bật 2FA đăng nhập y như trước. 2FA chỉ chen vào với ai đã tự bật — nên có thể triển khai dần, không ép toàn bộ.

---

## 7. Luồng thiết lập (enroll) — mảnh ①

TOTP hoạt động nhờ điện thoại và server **cùng giữ một khóa bí mật**. Bước enroll là lúc trao khóa đó cho điện thoại qua QR:

```mermaid
flowchart TD
    A["User bấm 'Bật 2FA' trong menu tài khoản"] --> B["API: sinh GoogleAuthenticatorKey<br/>lưu tạm, trả về chuỗi + ảnh QR"]
    B --> C["Angular hiện QR"]
    C --> D["User mở Google Authenticator<br/>quét QR → app bắt đầu sinh mã 6 số"]
    D --> E["User nhập mã hiện tại để xác nhận"]
    E --> F{"API kiểm mã đúng?"}
    F -->|"sai"| E
    F -->|"đúng"| G["Lưu khóa chính thức<br/>+ IsTwoFactorEnabled = true"]
    G --> H["✅ Từ giờ đăng nhập sẽ hỏi mã"]

    style E fill:#e3f2fd,stroke:#1565c0
    style G fill:#c8e6c9,stroke:#2e7d32
```

!!! warning "Bắt buộc có bước xác nhận mã trước khi bật"
    Không được bật 2FA ngay khi sinh QR. Phải đợi user nhập đúng một mã — đó là bằng chứng họ **đã quét thành công**. Bỏ bước này thì user quét hụt vẫn bị bật 2FA và tự khóa mình ngay lần đăng nhập sau.

---

## 8. Những quyết định phải chốt trước khi code {#quyet-dinh}

```mermaid
mindmap
  root((2FA<br/>cần chốt))
    Phạm vi bắt buộc
      Chỉ khuyến khích, user tự bật?
      Bắt buộc với role Admin?
      Bắt buộc toàn bộ?
    Provider
      Google Authenticator trước
      Email chờ SMTP thật
      SMS để sau
    Ghi nhớ thiết bị
      Có làm cookie 30 ngày?
      Rủi ro máy dùng chung
    Khôi phục
      Admin gỡ hộ (tối thiểu)
      Mã dự phòng in ra
    Đa tenant
      Bật theo tenant hay toàn hệ thống?
```

| Câu hỏi | Lựa chọn đề xuất | Vì sao |
|---|---|---|
| **Bắt buộc cho ai?** | ✅ ĐÃ CHỐT: **tất cả tài khoản, gồm cả admin** (hạng mục bắt buộc). Có thể bật dần theo nhóm nhưng admin KHÔNG được miễn | Yêu cầu tuân thủ; admin là tài khoản giá trị cao nhất |
| **Provider nào trước?** | ✅ ĐÃ CHỐT: **cả hai** — Google Authenticator + Email OTP | User chọn linh hoạt; lưu ý phải kiểm SMTP gửi thật trước khi bật Email |
| **Ghi nhớ thiết bị?** | Làm sau, mặc định tắt | Thêm bề mặt tấn công; không cần cho bản đầu |
| **Khôi phục?** | ✅ ĐÃ CHỐT: **cả ba** — mã dự phòng (user tự lo) + admin gỡ hộ + van khẩn cấp qua config cho admin | Hệ thống chỉ có 1 admin ⇒ van khẩn cấp là bắt buộc |
| **Đa tenant?** | Theo application (như setting hiện tại) | Khớp với `twoFactorLogin.isEnabled` đang lưu ở mức application |

!!! danger "Hệ thống chỉ có 1 tài khoản admin — van khẩn cấp là BẮT BUỘC, làm TRƯỚC TIÊN"
    Với một admin duy nhất, không có admin thứ hai để gỡ hộ. Nếu admin bật 2FA rồi mất/hỏng điện thoại **trước khi** có van khẩn cấp, toàn hệ thống mất quyền quản trị vĩnh viễn — sửa lại setting cũng là hành vi quản trị. Vì vậy:

    1. **Van khẩn cấp qua `appsettings.json`** (ví dụ `TwoFactor:BypassForAdmin` hoặc cờ tắt 2FA toàn cục) phải tồn tại và kiểm thử **trước khi** admin được phép bật 2FA. Mô hình giống hệt `alwaysAllowLoopback`/van trong `appsettings.json` của [Admin IP Allowlist](admin-ip-restriction-feature.md#self-lockout).
    2. **Mã dự phòng** phải được sinh và bắt admin lưu **ngay tại bước enroll**, không cho bỏ qua.
    3. Cân nhắc **tạo thêm 1 tài khoản admin dự phòng** trước khi bật — rẻ nhất và hiệu quả nhất để phá thế "single point of failure".

---

## 9. Rủi ro

```mermaid
flowchart TD
    R1["🔒 Tự khóa hàng loạt"] --> R1D["Ép 2FA khi user chưa kịp enroll ⇒ không ai vào được.<br/>Giảm thiểu: giai đoạn 1 để tự nguyện; enroll xong mới có tác dụng"]
    R2["📵 Mất điện thoại"] --> R2D["Không có recovery ⇒ mất tài khoản.<br/>Giảm thiểu: làm mảnh ③ (admin gỡ) TRƯỚC khi cho bật rộng"]
    R3["🎭 An toàn giả"] --> R3D["Làm enroll mà quên challenge ⇒ tưởng an toàn nhưng không.<br/>Giảm thiểu: ② trước hoặc cùng ①; test bằng đăng nhập lại thật"]
    R4["⏰ Lệch giờ server"] --> R4D["TOTP phụ thuộc thời gian; server lệch giờ ⇒ mã luôn sai.<br/>Giảm thiểu: đảm bảo NTP đồng bộ; ABP có cửa sổ dung sai ±1 bước"]
    R5["📧 Email OTP không gửi được"] --> R5D["SMTP trỏ 127.0.0.1:25.<br/>Giảm thiểu: chỉ bật provider Email sau khi kiểm 'Gửi thử email' chạy thật"]

    style R1 fill:#ffcdd2,stroke:#c62828
    style R3 fill:#ffcdd2,stroke:#c62828
```

---

## 10. Kế hoạch triển khai đề xuất {#ke-hoach}

```mermaid
flowchart LR
    P0["✅ Bước 0 ĐÃ XONG<br/>Công tắc bật/tắt<br/>trong Settings"] --> P1["Bước 1 — ③ LƯỚI AN TOÀN<br/>Van khẩn cấp appsettings.json<br/>+ API admin gỡ 2FA<br/><i>vì chỉ 1 admin ⇒ BẮT BUỘC trước</i>"]
    P1 --> P2["Bước 2 — ① ENROLL<br/>Sinh QR + xác nhận mã<br/>+ sinh mã dự phòng<br/>(Google Auth + Email)"]
    P2 --> P3["Bước 3 — ② CHALLENGE<br/>Nối vào Login theo mẫu<br/>ForceChangePassword"]
    P3 --> P4["Bước 4<br/>Gỡ cảnh báo vàng,<br/>kiểm thử end-to-end,<br/>tạo admin dự phòng"]

    style P0 fill:#c8e6c9,stroke:#2e7d32
    style P1 fill:#ffcdd2,stroke:#c62828,stroke-width:2px
    style P3 fill:#ffe0b2,stroke:#e65100
```

Thứ tự này chọn có chủ đích: **làm lưới an toàn (③) trước**, rồi mới enroll (①), cuối cùng mới bật thử thách (②). Nhờ vậy ở mọi thời điểm giữa chừng, không ai có thể tự khóa mình mà không có đường gỡ. Với ràng buộc **1 admin duy nhất**, bước 1 không chỉ là "nên làm trước" mà là **điều kiện tiên quyết** — chưa có van khẩn cấp thì tuyệt đối chưa cho admin bật 2FA.

!!! note "Điểm cần xác minh khi bắt tay code"
    Tài liệu này chốt *kiến trúc và luồng*. Tên API chính xác của ABP 6.0 ZeroCore cho phần sinh/kiểm mã TOTP (`GenerateNewAuthenticatorKeyAsync`, `VerifyTwoFactorTokenAsync`, `TwoFactorSignInAsync`…) cần đối chiếu lại với source ABP đang tham chiếu ở bước code — khung luồng thì không đổi dù tên method có khác chút.

---

*Xem thêm: [Hết hạn mật khẩu & Đổi mật khẩu bắt buộc](password-expiration-feature.md) — nơi có luồng `ForceChangePassword` được dùng làm bản mẫu cho mảnh ② ở đây.*
