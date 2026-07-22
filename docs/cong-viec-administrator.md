# Tổng quan công việc — Administrator & Bảo mật mật khẩu

## 1. Nhìn nhanh — đã làm gì

```mermaid
flowchart LR
    R(("Đã làm"))
    R --> F1["🔐 Thời hạn mật khẩu<br/>nhắc đổi + hết hạn khóa"]
    R --> F2["🔔 Popup nhắc trong app"]
    R --> F3["🐞 Sửa lỗi trang Cấu hình"]
    R --> F4["🎨 Sửa hiển thị khi chạy Staging"]
    R --> F5["⚙️ Bộ script chạy / build"]

    style R fill:#00897b,stroke:#004d40,color:#fff
    style F1 fill:#fff3e0,stroke:#fb8c00
    style F2 fill:#e3f2fd,stroke:#1e88e5
    style F3 fill:#ffebee,stroke:#e53935
    style F4 fill:#f3e5f5,stroke:#8e24aa
    style F5 fill:#eceff1,stroke:#546e7a
```

---

## 2. File nào — sửa gì (nhìn là thấy)

> Chú thích:  ✏️ = sửa   ·   ➕ = thêm mới   ·   🐞 = sửa lỗi

```mermaid
flowchart TB
    subgraph CORE["📁 Nền tảng — Core"]
        c1["📄 AppSettings.cs<br/>✏️ đổi tên 2 biến mật khẩu"]
        c2["📄 AppSettingProvider.cs<br/>✏️ mặc định 90 / 80 ngày"]
        c3["📄 PasswordExpirationChecker.cs<br/>✏️ đọc theo tên mới"]
        c4["📄 Localization (vi / en)<br/>✏️ nhãn hiển thị trên form"]
    end
    subgraph APP["📁 Nghiệp vụ — Application"]
        a1["📄 SecuritySettingsEditDto.cs<br/>✏️ đổi tên thuộc tính"]
        a2["📄 HostSettingsAppService.cs<br/>✏️ đọc/ghi + kiểm tra hợp lệ"]
    end
    subgraph NG["📁 Giao diện — Angular"]
        n1["📄 Administrator › Cấu hình<br/>✏️ 2 ô nhập số ngày<br/>🐞 sửa lỗi trắng trang"]
        n2["📄 Manager › Thanh trên<br/>➕ popup nhắc đổi mật khẩu"]
    end
    subgraph MVC["📁 Web — MVC (Razor + đóng gói)"]
        m1["📄 _Layout.cshtml (2 app)<br/>✏️ CSS tự khớp mã băm (Staging)"]
        m2["📄 bundleconfig.json<br/>➕ sinh file nén trang đổi mật khẩu"]
    end
    subgraph DB["📁 Cơ sở dữ liệu"]
        d1["🗄️ AbpSettings<br/>✏️ đổi tên khóa bằng SQL<br/>(không cần migration)"]
    end

    style CORE fill:#fff3e0,stroke:#fb8c00
    style APP fill:#e8f5e9,stroke:#43a047
    style NG fill:#e3f2fd,stroke:#1e88e5
    style MVC fill:#f3e5f5,stroke:#8e24aa
    style DB fill:#eceff1,stroke:#546e7a
```

---

## 3. Tính năng thời hạn mật khẩu

```mermaid
timeline
    title Vòng đời mật khẩu (mặc định)
    Ngày 0 : Vừa đổi mật khẩu
    Ngày 1-79 : Đăng nhập bình thường
    Ngày 80 : MỐC NHẮC : Vẫn cho vào : Hiện popup nhắc đổi
    Ngày 90 : MỐC KHÓA : Chặn đăng nhập : Bắt buộc đổi mật khẩu
```

| Mốc | Số ngày mặc định | Khi tới mốc |
|---|---|---|
| 🔔 Nhắc đổi | 80 | Hiện popup — **vẫn cho vào** |
| 🔒 Hết hạn | 90 | **Chặn** — bắt buộc đổi mới vào được |

!!! note "📷 Ảnh màn hình"
    Chèn ảnh form **Cấu hình hệ thống → Hết hạn mật khẩu** (2 ô nhập số ngày) tại đây.

---

## 4. Ba lỗi đã xử lý (khi chạy Staging)

```mermaid
flowchart LR
    subgraph P1["🐞 Trang Cấu hình"]
        a1["Trước: trắng trang, báo lỗi liên tục"] --> a2["Sau: chờ tải xong mới hiện → OK"]
    end
    subgraph P2["🎨 Giao diện Staging"]
        b1["Trước: link CSS ghi cứng mã cũ → mất giao diện"] --> b2["Sau: dùng mã đại diện → tự khớp"]
    end
    subgraph P3["📄 Trang đổi mật khẩu"]
        c1["Trước: thiếu file nén → hiện chữ JSON thô"] --> c2["Sau: khai báo → tự sinh file nén"]
    end

    style a1 fill:#ffcdd2,stroke:#c62828
    style b1 fill:#ffcdd2,stroke:#c62828
    style c1 fill:#ffcdd2,stroke:#c62828
    style a2 fill:#c8e6c9,stroke:#2e7d32
    style b2 fill:#c8e6c9,stroke:#2e7d32
    style c2 fill:#c8e6c9,stroke:#2e7d32
```

!!! note "📷 Ảnh màn hình"
    Chèn ảnh **trước** (trắng trang / JSON thô) và **sau** (hiển thị đúng) tại đây.

---

## 5. Bộ script chạy / build

```mermaid
flowchart LR
    subgraph DEV["🧪 Phát triển"]
        r1["run-debug-manager.ps1"]
        r2["run-debug-admin.ps1"]
    end
    subgraph STG["🚀 Triển khai"]
        r3["build-angular.ps1<br/>build + copy giao diện"]
        r4["run-mvc-staging.ps1<br/>chỉ chạy web"]
        r5["run-staging.ps1<br/>làm cả hai"]
    end

    style DEV fill:#e3f2fd,stroke:#1e88e5
    style STG fill:#e8f5e9,stroke:#43a047
```
