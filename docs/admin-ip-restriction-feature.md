# Giới hạn địa chỉ mạng truy cập khu vực quản trị

> Chỉ các IP / dải mạng **đã khai trước** mới vào được khu vực quản trị. Mọi IP khác bị chặn (**default-deny**). Không migration, không cần SQL.

## 1. Làm gì — ai bị ảnh hưởng

```mermaid
flowchart LR
    A["👤 Admin TRONG danh sách"] -->|"✅ vào được"| S["🖥️ Khu quản trị"]
    B["👤 Admin NGOÀI danh sách"] -->|"❌ 403"| S
    C["👤 User Manager"] -->|"✅ không bị đụng"| S
    D["🤖 Dịch vụ GIS gọi bằng apikey"] -->|"✅ không bị đụng"| S

    style B fill:#ffcdd2,stroke:#c62828
    style A fill:#c8e6c9,stroke:#2e7d32
    style C fill:#c8e6c9,stroke:#2e7d32
    style D fill:#c8e6c9,stroke:#2e7d32
```

!!! danger "Danh sách ĐƯỢC PHÉP, không phải danh sách BỊ CHẶN"
    Không khớp gì cả → **CHẶN**. Lý do: kẻ tấn công đến từ **bất kỳ IP nào**, không liệt kê hết được; người được phép thì **hữu hạn, biết trước**. → Hệ quả: **khai thiếu IP của chính mình = tự khóa** (xem §5).

## 2. File nào — sửa gì

> ➕ thêm mới · ✏️ sửa

```mermaid
flowchart TB
    subgraph CORE["📁 Nền tảng — Core (không dính ASP.NET)"]
        c1["📄 AdminIpPolicy.cs<br/>➕ luật cho phép/chặn (dùng chung)"]
        c2["📄 AdminIpNetwork.cs<br/>➕ khớp IP theo bit"]
        c3["📄 AppSettings + Provider<br/>✏️ 3 tham số cấu hình"]
    end
    subgraph FILTER["📁 Cưỡng chế — Web.Filter"]
        f1["📄 AdminIpRestrictionMiddleware.cs<br/>➕ chặn request ngoài danh sách"]
    end
    subgraph APP["📁 Nghiệp vụ — Application"]
        a1["📄 HostSettingsAppService.cs<br/>✏️ lưu + chống tự khóa"]
        a2["📄 SessionAppService.cs<br/>✏️ trả IP hiện tại cho form"]
    end
    subgraph MVC["📁 Web — MVC"]
        m1["📄 Startup.cs + appsettings.json<br/>✏️ cắm middleware + van cứu hộ"]
    end
    subgraph NG["📁 Giao diện — Administrator"]
        n1["📄 settings.component<br/>➕ bảng thêm/xóa/bật‑tắt rule"]
    end

    style CORE fill:#fff3e0,stroke:#fb8c00
    style FILTER fill:#ffe0b2,stroke:#e65100
    style APP fill:#e8f5e9,stroke:#43a047
    style MVC fill:#f3e5f5,stroke:#8e24aa
    style NG fill:#e3f2fd,stroke:#1e88e5
```

## 3. Cách hoạt động — 5 cửa gạt

Mỗi cửa gạt ra là **cho qua luôn**; chỉ số ít request tới bước so IP.

```mermaid
flowchart TD
    R["📥 Request"] --> G1{"1️⃣ Van cứu hộ bật?"}
    G1 -->|"có"| P1["✅ cho qua"]
    G1 -->|"không"| G2{"2️⃣ Là dịch vụ GIS<br/>/ekmapserver?"}
    G2 -->|"có"| P2["✅ cho qua (lớp token lo)"]
    G2 -->|"không"| G3{"3️⃣ Tính năng tắt?"}
    G3 -->|"tắt"| P3["✅ cho qua"]
    G3 -->|"bật"| G4{"4️⃣ Vào khu quản trị?<br/>(đường /Administrator HOẶC role Admin)"}
    G4 -->|"không"| P4["✅ cho qua (user Manager)"]
    G4 -->|"có"| G5{"5️⃣ IP khớp rule đang bật?"}
    G5 -->|"khớp"| P5["✅ cho qua"]
    G5 -->|"không"| D["❌ 403 + ghi log"]

    style D fill:#ffcdd2,stroke:#c62828,stroke-width:3px
    style P1 fill:#c8e6c9,stroke:#2e7d32
    style P2 fill:#c8e6c9,stroke:#2e7d32
    style P3 fill:#c8e6c9,stroke:#2e7d32
    style P4 fill:#c8e6c9,stroke:#2e7d32
    style P5 fill:#c8e6c9,stroke:#2e7d32
```

!!! danger "Chặn theo ĐƯỜNG DẪN thôi là THỦNG"
    Trang `/Administrator` chỉ là cái vỏ. Chức năng quản trị thật đi qua API `/api/services/app/HostSettings/UpdateAllSettings`... — **không đường nào bắt đầu bằng `/Administrator`**. Vì vậy cửa 4 chặn theo **cả đường dẫn lẫn danh tính Admin** (role trong cookie), nếu không kẻ tấn công gọi thẳng API là lách được.

## 4. Cấu hình (3 tham số, lưu trong AbpSettings)

| Tham số | Mặc định |
|---|---|
| Bật/tắt giới hạn | Tắt (`false`) |
| Danh sách rule (CIDR + mô tả + bật/tắt) | Rỗng (`[]`) |
| Luôn cho phép localhost | Bật (`true`) |

Mô tả **bắt buộc** cho mỗi rule (6 tháng sau không ai nhớ `10.0.0.0/24` là gì → không ai dám xóa). Deploy xong 3 tham số tự có mặc định, bảng vẫn trống — **không cần chạy SQL**.

## 5. Chống tự khóa — 3 lớp

```mermaid
flowchart TD
    X["😱 Khai nhầm allowlist"] --> L1{"🛡️ Lớp 1 — Guard khi lưu<br/>server kiểm IP người đang lưu"}
    L1 -->|"chặn"| OK1["✅ Không lưu, báo lỗi kèm IP hiện tại"]
    L1 -->|"lọt (sửa thẳng DB)"| L2{"🛡️ Lớp 2 — Localhost luôn mở"}
    L2 -->|"cứu"| OK2["✅ RDP vào máy chủ → localhost"]
    L2 -->|"đã tắt"| L3{"🛡️ Lớp 3 — Van file cấu hình"}
    L3 -->|"cứu"| OK3["✅ Disabled=true + restart"]
    L3 -->|"cuối"| SQL["🗄️ UPDATE AbpSettings + restart"]

    style OK1 fill:#c8e6c9,stroke:#2e7d32
    style OK2 fill:#c8e6c9,stroke:#2e7d32
    style OK3 fill:#c8e6c9,stroke:#2e7d32
    style SQL fill:#ffe0b2,stroke:#e65100
```

!!! danger "Nếu lỡ tự khóa — theo thứ tự"
    1. Truy cập từ **chính máy chủ** (localhost luôn mở).
    2. `"AdminIpRestriction": { "Disabled": true }` trong `appsettings.json` → restart.
    3. Đường cuối:
       ```sql
       UPDATE AbpSettings SET Value='false'
       WHERE Name='App.Security.AdminIpRestriction.IsEnabled';
       ```
       **Phải restart** để nhả cache setting.

## 6. Hai bẫy về mạng (hỏi bên vận hành trước khi bật)

!!! warning "Có reverse proxy không?"
    Nếu chạy sau proxy mà **không** khai `ForwardedHeaders:KnownProxies` = IP proxy → server thấy **mọi request đến từ IP proxy** → chính sách vô nghĩa (cho qua tất cả hoặc chặn tất cả). Không proxy thì để **rỗng**. Đây là câu hỏi vận hành, không suy từ code được.

!!! warning "NAT — admin không tự biết IP của mình"
    `ipconfig` cho IP nội bộ (`192.168.1.25`), nhưng nếu server ở ngoài Internet thì server thấy **IP công cộng** của cả văn phòng. Khai nhầm IP nội bộ = tự khóa. → Form **bắt buộc hiện "địa chỉ hiện tại của bạn"**. IP động (cáp quang nhà) đổi thường xuyên → nên dùng **IP tĩnh** hoặc **VPN công ty**.

## 7. Triển khai & kiểm chứng

```mermaid
flowchart LR
    B1["Build MVC + Angular"] --> B2["Deploy<br/>(mặc định TẮT → chưa đổi gì)"]
    B2 --> B3["Bật rule chứa IP của mình<br/>(khi có mặt tại chỗ / có RDP)"]
    B3 --> B4["⭐ Kiểm chứng 2 chiều"]
    style B4 fill:#ffe0b2,stroke:#e65100
```

- Máy **ngoài** allowlist (4G) mở `/Administrator` → phải **403**.
- apikey gọi `/ekmapserver/...` từ IP ngoài → phải **200** (dịch vụ GIS không bị vạ lây).

> Đã có **52 unit test** phủ phép khớp IP (chạy không cần web/DB).
