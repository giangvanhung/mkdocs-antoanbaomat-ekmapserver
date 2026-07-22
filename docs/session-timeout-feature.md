# Tự động đóng phiên khi không hoạt động

> Bỏ máy quá lâu → **hiện cảnh báo đếm ngược** → không thao tác thì **đóng phiên, bắt đăng nhập lại**. Chỉ áp dụng app **Administrator**.

## 1. Cách hoạt động

```mermaid
flowchart TD
    L["Đăng nhập Administrator"] --> R{"Bật tính năng?"}
    R -->|"tắt"| X["Không làm gì"]
    R -->|"bật"| T["⏱️ Đếm thời gian nhàn rỗi<br/>(chuột/phím reset về 0)"]
    T -->|"nhàn rỗi tới ngưỡng cảnh báo"| D["⚠️ Hộp thoại đếm ngược<br/>[ Tiếp tục phiên ] [ Đăng nhập lại ]"]
    D -->|"bấm Tiếp tục"| T
    D -->|"bấm Đăng nhập lại / hết đếm ngược"| O["🚪 Đăng xuất (xóa cookie)"]
    O --> LG["Về trang đăng nhập<br/>'Phiên đã hết hạn'"]

    style O fill:#ffcdd2,stroke:#c62828
    style X fill:#eeeeee,stroke:#9e9e9e
```

## 2. Cấu hình (Host + Tenant)

| Tham số | Ý nghĩa | Mặc định |
|---|---|---|
| Bật/tắt | Có áp dụng không | Tắt |
| Tổng thời gian chờ | Nhàn rỗi tối đa trước khi đóng | 30 giây |
| Cảnh báo trước khi đóng | Hiện hộp thoại trước bao nhiêu giây | 30 giây |

→ Hộp thoại hiện tại mốc **(tổng − cảnh báo)** rồi đếm ngược.

## 3. File nào — sửa gì

> ✏️ sửa · ➕ thêm mới

```mermaid
flowchart TB
    subgraph APP["📁 Nghiệp vụ — Application"]
        a1["📄 SessionTimeOutInfoDto.cs<br/>➕ gói 3 tham số"]
        a2["📄 SessionAppService.cs<br/>➕ trả cấu hình cho mọi user"]
        a3["📄 Host + Tenant SettingsAppService<br/>✏️ đọc/ghi cấu hình"]
    end
    subgraph NG["📁 Giao diện — Administrator"]
        n1["📄 session-timeout.service.ts<br/>➕ đồng hồ nhàn rỗi + đồng bộ đa tab"]
        n2["📄 session-timeout-dialog<br/>➕ hộp thoại đếm ngược"]
        n3["📄 topbar.component.ts<br/>✏️ khởi động đồng hồ"]
        n4["📄 settings.component.html<br/>➕ form 3 ô cấu hình"]
    end
    subgraph MVC["📁 Web — MVC"]
        m1["📄 Startup.cs<br/>✏️ hạ trần cookie (lớp chặn dự phòng)"]
    end
    subgraph CORE["📁 Nền tảng"]
        c1["📄 Localization<br/>➕ nhãn hộp thoại + form"]
    end

    style APP fill:#e8f5e9,stroke:#43a047
    style NG fill:#e3f2fd,stroke:#1e88e5
    style MVC fill:#f3e5f5,stroke:#8e24aa
    style CORE fill:#fff3e0,stroke:#fb8c00
```

## 4. Cần nhớ

!!! warning "Phiên thật là COOKIE của server"
    Đóng phiên = điều hướng tới **`/Account/Logout`** để server xóa cookie — **không** phải xóa localStorage (xóa localStorage thì cookie vẫn sống). Vì cookie dùng chung nên đăng xuất là **toàn hệ** (cả MVC + Administrator + Manager).

!!! note "Ba tầng thời gian, không xung đột"
    Đồng hồ client (phát hiện nhàn rỗi + cảnh báo) chỉ **kích hoạt** luồng đăng xuất có sẵn; không đụng hạn cookie (~14 ngày, giữ làm chốt chặn cuối). `Startup.cs` hạ trần cookie xuống vài giờ làm lớp dự phòng khi client (JS) chết.

!!! tip "Đổi cấu hình phải reload"
    Cấu hình đọc lúc vào app, nên user đang mở phải **F5** mới nhận giá trị mới. Mặc định **tắt** → build xong chưa đổi gì cho tới khi admin bật.
