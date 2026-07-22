# Khóa tài khoản tạm thời khi đăng nhập sai

> Sai mật khẩu **quá số lần** trong **một khoảng thời gian** → khóa tạm thời, hết giờ **tự** mở lại.

## 1. Cách hoạt động

```mermaid
timeline
    title Ví dụ: 5 lần sai trong 10 phút → khóa 15 phút
    0 phút : Sai lần 1
    2 phút : Sai lần 2
    5 phút : Sai lần 3
    7 phút : Sai lần 4
    9 phút : Sai lần 5 → 🔒 KHÓA tới phút 24
    24 phút : Hết khóa → tự đăng nhập lại
```

```mermaid
flowchart TD
    A["Sai mật khẩu"] --> B{"Chính sách bật?"}
    B -->|"tắt"| Z["Không đếm"]
    B -->|"bật"| C{"Lần sai trước<br/>cách đây quá cửa sổ?"}
    C -->|"rồi"| D["Đếm lại từ 0"]
    C -->|"chưa"| E["Giữ số đếm"]
    D --> F["số đếm +1"]
    E --> F
    F --> G{"đủ số lần tối đa?"}
    G -->|"đủ"| H["🔒 Khóa trong N phút"]
    G -->|"chưa"| I["Chưa khóa"]

    style H fill:#ffcdd2,stroke:#c62828
    style Z fill:#eeeeee,stroke:#9e9e9e
```

## 2. Cấu hình (Host + Tenant)

| Tham số | Ý nghĩa | Mặc định |
|---|---|---|
| Bật/tắt | Có áp dụng chính sách không | Bật |
| Số lần sai tối đa | Sai bao nhiêu lần thì khóa | 5 |
| Cửa sổ thời gian | Khoảng đếm số lần sai | 600 giây (10 phút) |
| Thời gian khóa | Khóa bao lâu | 900 giây (15 phút) |

## 3. File nào — sửa gì

> ✏️ sửa · ➕ thêm mới

```mermaid
flowchart TB
    subgraph CORE["📁 Nền tảng — Core"]
        c1["📄 User.cs<br/>➕ mốc 'lần sai gần nhất'"]
        c2["📄 UserManager.cs<br/>➕ logic cửa sổ thời gian"]
        c3["📄 AppSettings + Provider<br/>➕ tham số cửa sổ (600s)"]
        c4["📄 Localization<br/>✏️ nhãn + 3 lỗi cấu hình"]
    end
    subgraph APP["📁 Nghiệp vụ — Application"]
        a1["📄 UserLockOutSettingsEditDto.cs<br/>➕ trường cửa sổ"]
        a2["📄 Host + Tenant SettingsAppService<br/>✏️ đọc/ghi + kiểm tra hợp lệ"]
    end
    subgraph EF["📁 Cơ sở dữ liệu"]
        e1["📄 Migration<br/>➕ cột mốc lần sai (datetime2 NULL)"]
    end
    subgraph NG["📁 Giao diện — Administrator"]
        n1["📄 settings.component.html<br/>➕ ô nhập cửa sổ thời gian"]
    end

    style CORE fill:#fff3e0,stroke:#fb8c00
    style APP fill:#e8f5e9,stroke:#43a047
    style EF fill:#eceff1,stroke:#546e7a
    style NG fill:#e3f2fd,stroke:#1e88e5
```

## 4. Cần nhớ

!!! warning "Chặn ở đúng một điểm để phủ cả 2 đường đăng nhập"
    Cả form web (MVC) lẫn API (`/api/TokenAuth`) đều dồn về **một** hàm đếm sai. Logic đặt ở đó nên **không đường nào lọt lưới**. Đặt ở controller thì đường API thành lỗ hổng.

!!! note "Ý nghĩa cửa sổ"
    Mỗi lần sai, nếu **cách lần sai trước** quá cửa sổ thì số đếm về 0. Đăng nhập **đúng** cũng reset về 0. Đăng nhập bằng OpenID (không mật khẩu) **không** bị tính.

!!! danger "Phải chạy migration trước khi bật code mới"
    Code đọc cột mốc lần sai; chưa có cột → **hỏng cả đăng nhập** (`Invalid column name`). Chạy `dotnet ef database update` (đúng 1 lệnh `ADD COLUMN`). **Không cần** backfill (user cũ = chưa có lần sai nào).
