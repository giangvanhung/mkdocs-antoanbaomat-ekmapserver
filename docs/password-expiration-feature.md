# Hết hạn mật khẩu & Đổi mật khẩu bắt buộc

> Mật khẩu có tuổi thọ: tới **mốc nhắc** thì nhắc (vẫn cho vào), tới **mốc hết hạn** thì chặn, bắt buộc đổi. Kèm ép đổi mật khẩu ở lần đăng nhập đầu.

## 1. Cách hoạt động

```mermaid
timeline
    title Vòng đời mật khẩu (mặc định 80 / 90 ngày)
    Ngày 0 : Vừa đổi mật khẩu
    Ngày 1-79 : Đăng nhập bình thường
    Ngày 80 : MỐC NHẮC : Vẫn cho vào : Hiện popup nhắc đổi
    Ngày 90 : MỐC HẾT HẠN : Chặn đăng nhập : Bắt buộc đổi mật khẩu
```

Luồng khi đăng nhập:

```mermaid
flowchart TD
    A["Đăng nhập"] --> B{"Mật khẩu đúng?"}
    B -->|"sai"| B1["❌ Từ chối"]
    B -->|"đúng"| C{"Cờ ép đổi lần đầu?"}
    C -->|"có"| G["➡️ Trang đổi mật khẩu bắt buộc"]
    C -->|"không"| D{"Tuổi mật khẩu?"}
    D -->|"≥ mốc hết hạn"| G
    D -->|"≥ mốc nhắc"| F["✅ Vào app + popup nhắc"]
    D -->|"còn trẻ"| F2["✅ Vào app bình thường"]

    style B1 fill:#ffcdd2,stroke:#c62828
    style G fill:#fff3cd,stroke:#f39c12
    style F fill:#c8e6c9,stroke:#2e7d32
    style F2 fill:#c8e6c9,stroke:#2e7d32
```

## 2. Cấu hình (Host + Tenant)

| Mốc | Số ngày mặc định | Khi tới mốc |
|---|---|---|
| 🔔 Nhắc đổi (`PasswordChangeRequestDays`) | 80 | Popup nhắc — **vẫn cho vào** |
| 🔒 Hết hạn (`PasswordValidityDays`) | 90 | **Chặn** — bắt buộc đổi mới vào được |

Ràng buộc: `nhắc < hết hạn`. Đặt **hết hạn = 0** để **tắt** cả tính năng.

## 3. File nào — sửa gì

> ➕ thêm mới · ✏️ sửa

```mermaid
flowchart TB
    subgraph CORE["📁 Nền tảng — Core"]
        c1["📄 User.cs<br/>➕ mốc 'đổi mật khẩu lần cuối'"]
        c2["📄 PasswordExpirationChecker.cs<br/>➕ logic 2 mốc (dùng chung)"]
        c3["📄 AppSettings + Provider<br/>✏️ 2 biến (90 / 80)"]
        c4["📄 Localization<br/>✏️ nhãn hiển thị"]
    end
    subgraph APP["📁 Nghiệp vụ — Application"]
        a1["📄 SecuritySettingsEditDto.cs<br/>✏️ đổi tên thuộc tính"]
        a2["📄 HostSettingsAppService.cs<br/>✏️ đọc/ghi + kiểm tra hợp lệ"]
        a3["📄 SessionAppService.cs<br/>✏️ trả cảnh báo cho app"]
    end
    subgraph MVC["📁 Web — MVC"]
        m1["📄 AccountController.cs<br/>✏️ chặn login + ép đổi"]
        m2["📄 ForceChangePassword (view + lý do)<br/>➕ trang đổi mật khẩu bắt buộc"]
    end
    subgraph EF["📁 Cơ sở dữ liệu"]
        e1["📄 Migration<br/>➕ cột mốc đổi MK (datetime2 NULL)"]
    end
    subgraph NG["📁 Giao diện — Angular"]
        n1["📄 Administrator › settings<br/>✏️ 2 ô nhập số ngày"]
        n2["📄 Manager + Administrator › topbar<br/>➕ popup nhắc đổi mật khẩu"]
    end

    style CORE fill:#fff3e0,stroke:#fb8c00
    style APP fill:#e8f5e9,stroke:#43a047
    style MVC fill:#f3e5f5,stroke:#8e24aa
    style EF fill:#eceff1,stroke:#546e7a
    style NG fill:#e3f2fd,stroke:#1e88e5
```

## 4. Cần nhớ

!!! danger "Đặt 'hết hạn = 0' để tắt — không gộp điều kiện"
    Chỉ `PasswordValidityDays` quyết định bật/tắt. Nếu gộp nhầm điều kiện tắt, đặt hết hạn = 0 mà mốc nhắc vẫn 80 sẽ khiến **mọi user bị chặn và kẹt vĩnh viễn** (đổi xong tuổi = 0 vẫn bị coi là hết hạn). Đây là bug từng gặp — nhánh "tắt" phải đứng riêng.

!!! danger "Phải chạy migration + cân nhắc backfill trước khi bật"
    Code đọc cột mốc đổi MK → chưa có cột thì **hỏng cả đăng nhập**. Sau khi thêm cột, user cũ = `NULL` → tính theo ngày tạo → **có thể ép đổi hàng loạt**. Backfill `= GETDATE()` (coi như vừa đổi lúc nâng cấp). Dùng `GETDATE()` **không** phải `GETUTCDATE()` (server dùng giờ local).

!!! warning "Đường token chưa kiểm hết hạn"
    Việc chặn nằm ở đăng nhập web (MVC). `/api/TokenAuth` cấp JWT **không** kiểm hết hạn mật khẩu. Hiện vô hại (app dùng cookie), nhưng nếu mở API này ra ngoài thì phải cắm thêm cùng bộ kiểm tra.

!!! note "Luật 'MK mới khác MK cũ' đang tắt"
    Đang để tạm tắt → user có thể 'đổi' bằng cách gõ lại mật khẩu cũ; ép đổi lần đầu hiện **không** ngăn được giữ nguyên mật khẩu mặc định. Bật lại phải bật ở **cả hai** nơi (server + client).

!!! check "Mật khẩu không lọt vào nhật ký"
    Các trường mật khẩu gắn `[DisableAuditing]` để không bị ghi thô vào bảng log — vẫn giữ dòng log "ai đổi, lúc nào", chỉ bỏ giá trị.
