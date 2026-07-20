# Cập nhật DB đang chạy lên schema hiện tại

**Ngày:** 2026-07-17 · **Script:** `Test by SQL/update_db_to_current.sql` · **Trạng thái:** 🔶 Đã viết, **chưa chạy** trên DB nào

---

## 1. Bài toán

```mermaid
flowchart LR
    A["🗄️ DB production<br/>đã có sẵn, ĐANG PHỤC VỤ THẬT"] --> B{"Làm sao đưa<br/>schema mới vào?"}
    B --> C["❌ Dựng lại DB mới"]
    B --> D["❌ dotnet ef database update"]
    B --> E["✅ Script SQL chỉ THÊM"]

    style C fill:#ffcdd2,stroke:#c62828
    style D fill:#ffcdd2,stroke:#c62828
    style E fill:#c8e6c9,stroke:#2e7d32
```

---

## 2. Những gì được thêm mới

**Tổng: 3 cột + 1 bảng.** Chỉ `ADD` và `CREATE` — không `ALTER COLUMN`, không `DROP`, không rebuild bảng nào.

```mermaid
flowchart TD
    subgraph U["✏️ Bảng AbpUsers (ĐANG CÓ DỮ LIỆU THẬT)"]
        C1["ChangePassWordForLogInNext<br/><b>bit NOT NULL</b><br/>❌ KHÔNG có migration"]
        C2["LastPasswordChangeTime<br/><b>datetime2 NULL</b><br/>⚠️ PHẢI backfill"]
        C3["LastFailedLoginAttemptTime<br/><b>datetime2 NULL</b><br/>✅ không cần gì"]
    end
    subgraph T["🆕 Bảng mới (rủi ro = 0)"]
        T1["tenantBranding<br/>27 cột"]
    end
    subgraph H["📒 Sổ migration"]
        H1["__EFMigrationsHistory<br/>9 row — chỉ GHI SỔ, không thực thi"]
    end

    style C1 fill:#ffcdd2,stroke:#c62828
    style C2 fill:#ffe0b2,stroke:#e65100
    style C3 fill:#c8e6c9,stroke:#2e7d32
    style T fill:#e8f5e9,stroke:#2e7d32
```

### KHÔNG đụng tới

| Hạng mục | Vì sao |
|---|---|
| **`AbpSettings`** | Chỉ là row key–value trong bảng có sẵn — **không đổi schema**. Các setting mới (`SessionTimeOut`, `UserLockOut`, `PasswordExpirationDays`, `AdminIpRestriction`) đều có default trong `AppSettingProvider` ⇒ **không có row nào cũng chạy đúng** |
| **41 lệnh `AlterColumn`** | Từng bị EF sinh ra do lệch ModelSnapshot ([chi tiết](migration-identity-columns-issue.md)) — đã xoá sạch khỏi migration và **lệch snapshot đã sửa tận gốc** ([§6.3](#snapshot-da-sua)). **Không** đưa lại vào script |
| **[Giới hạn IP quản trị](admin-ip-restriction-feature.md)** | Settings thuần — không phát sinh thay đổi DB |

---

## 3. Hai cái bẫy — cả hai đều VÔ HÌNH nếu chỉ nhìn danh sách migration

### 3.1. `InitialBaseline` là ảnh chụp — **không được chạy** {#baseline}

```mermaid
flowchart TD
    A["🗄️ DB production<br/>dựng từ trước khi repo có migration"] -.->|"❌ KHÔNG BAO GIỜ chạy"| B["20260602080713_InitialBaseline<br/>1715 dòng CreateTable<br/><i>chỉ là ẢNH CHỤP model</i>"]
    B -->|"✅ chỉ ĐÓNG DẤU vào<br/>__EFMigrationsHistory"| C["EF coi như đã áp dụng"]

    D["dotnet ef database update"] -->|"cố chạy baseline"| E["💥 There is already an object<br/>named 'AbpUsers'"]

    style B fill:#ffe0b2,stroke:#e65100
    style E fill:#ffcdd2,stroke:#c62828
    style C fill:#c8e6c9,stroke:#2e7d32
```

*(Nếu lỡ chạy: EF chạy mỗi migration trong transaction ⇒ **rollback sạch, DB không đổi gì**. Cứ chạy script này rồi thử lại.)*

### 3.2. `ChangePassWordForLogInNext` không có migration {#khong-co-migration}

```mermaid
timeline
    title Vì sao cột này biến mất khỏi tầm nhìn
    21/05 — commit init : Migrations/ chỉ có ModelSnapshot<br/>KHÔNG có migration nào
    29/05 — commit 228a989 : Thêm ChangePassWordForLogInNext vào User.cs<br/>(tính năng "đổi mật khẩu lần đầu")
    02/06 — sinh InitialBaseline : EF nướng cột này THẲNG vào<br/>CreateTable(AbpUsers) của baseline<br/>thay vì tạo migration riêng
```

```mermaid
flowchart TD
    A["Cột nằm TRONG InitialBaseline"] --> B["Baseline chỉ được ĐÓNG DẤU,<br/>không CHẠY (§3.1)"]
    B --> C["❌ Cột KHÔNG BAO GIỜ được tạo<br/>trên DB production"]
    C --> D["AccountController.cs:153 đọc nó<br/>ở MỖI lần đăng nhập"]
    D --> E["💥 Invalid column name<br/>⇒ KHÔNG AI ĐĂNG NHẬP ĐƯỢC"]

    style C fill:#ffcdd2,stroke:#c62828
    style E fill:#ffcdd2,stroke:#c62828,stroke-width:3px
```

!!! danger "`dotnet ef database update` không cứu được"
    Theo sổ migration thì **mọi thứ "đã up to date"** — trong khi cột vẫn thiếu và đăng nhập vẫn chết.

    → Đây là lý do phải soát delta từ **CODE** (entity + `DbSet`), không chỉ từ danh sách migration.

---

## 4. `LastPasswordChangeTime` phải backfill {#backfill}

```mermaid
flowchart TD
    A["Thêm cột LastPasswordChangeTime<br/>= NULL cho mọi user"] --> B["PasswordExpirationChecker.cs:37<br/><i>LastPasswordChangeTime ?? CreationTime</i>"]
    B --> C["NULL ⇒ rơi về CreationTime"]
    C --> D{"User tạo quá 90 ngày trước?<br/><i>(mặc định, BẬT sẵn)</i>"}
    D -->|"đúng với hầu hết user"| E["💥 Expired ngay<br/>⇒ TOÀN BỘ user bị ép<br/>đổi mật khẩu ở lần đăng nhập kế tiếp"]

    style E fill:#ffcdd2,stroke:#c62828,stroke-width:3px
```

**Xử lý:** backfill `= GETDATE()` — coi như *"vừa đổi mật khẩu lúc nâng cấp"*, đồng hồ 90 ngày đếm lại từ thời điểm deploy.

```mermaid
flowchart LR
    A["Thêm cột"] --> B["UPDATE ... SET = GETDATE()<br/>WHERE ... IS NULL"]
    B --> C["✅ Không ai bị ép đổi mật khẩu"]
    style C fill:#c8e6c9,stroke:#2e7d32
```

!!! bug "Phải `GETDATE()`, KHÔNG phải `GETUTCDATE()`"
    ```mermaid
    flowchart TD
        A["grep Clock.Provider trong repo<br/>= 0 kết quả"] --> B["⇒ ABP dùng mặc định<br/>ClockProviders.Unspecified"]
        B --> C["⇒ Clock.Now = DateTime.Now<br/>= giờ LOCAL của server"]
        C --> D["Backfill bằng UTC sẽ lệch 7 giờ<br/>so với mốc đem ra so sánh"]
        style D fill:#ffe0b2,stroke:#e65100
    ```

Backfill **chỉ chạy đúng lúc cột được tạo lần đầu** ⇒ chạy lại script không bao giờ đè lên mốc thật của user.

---

## 5. Cách chạy trên DB có sẵn

```mermaid
flowchart TD
    P0["🔍 PHẦN 0 — KIỂM TRA<br/><b>CHỈ ĐỌC, không sửa gì</b>"] --> P0R["Báo DB đang thiếu đúng những gì<br/>+ bao nhiêu user sẽ bị ép đổi mật khẩu<br/>nếu không backfill"]
    P0R --> BK["💾 BACKUP DB"]
    BK --> P1["⚙️ PHẦN 1 — CẬP NHẬT<br/><i>trong transaction</i>"]
    P1 --> P1R{"Lỗi bất kỳ chỗ nào?"}
    P1R -->|"có"| RB["↩️ ROLLBACK sạch<br/>DB KHÔNG đổi gì"]
    P1R -->|"không"| CM["✅ COMMIT"]
    CM --> P2["✔️ PHẦN 2 — HẬU KIỂM"]
    P2 --> EF["🧪 dotnet ef database update<br/>phải báo 'already up to date'"]
    EF --> DP["🚀 Deploy code<br/>+ ĐĂNG NHẬP THỬ"]

    style P0 fill:#e3f2fd,stroke:#1565c0
    style BK fill:#fff9c4,stroke:#f57f17
    style RB fill:#ffe0b2,stroke:#e65100
    style CM fill:#c8e6c9,stroke:#2e7d32
```

Script **idempotent** — mọi bước đều `IF NOT EXISTS`, chạy lại nhiều lần vẫn an toàn.

### Log của PHẦN 1

```
[1] Đã đóng dấu InitialBaseline.
[2] Đã thêm ChangePassWordForLogInNext (DEFAULT 0 — không ép ai đổi mật khẩu).
[3] Đã thêm LastPasswordChangeTime + BACKFILL GETDATE() cho toàn bộ user.
[4] Đã thêm LastFailedLoginAttemptTime.
[5] Đã tạo bảng tenantBranding (27 cột).
[6] Đã đóng dấu 8 migration.
=== HOÀN TẤT — ĐÃ COMMIT ===
```

### PHẦN 2 — bốn con số bắt buộc

| Kiểm tra | Bắt buộc |
|---|---|
| `__EFMigrationsHistory` | **9** row |
| `tenantBranding` số cột | **27** |
| User còn `NULL LastPasswordChangeTime` | **0** ⭐ |
| User có `ChangePassWordForLogInNext = 1` | **0** ⭐ |

> Hai dòng ⭐ phân biệt *"nâng cấp êm"* với *"sáng mai cả cơ quan không đăng nhập được"*.

!!! note "Mức độ khoá bảng — nói cho chính xác"
    ```mermaid
    flowchart TD
        A["CREATE TABLE (bảng mới)"] --> R1["✅ 0 rủi ro"]
        B["ADD COLUMN ... NULL<br/>(2 cột datetime2)"] --> R2["✅ metadata-only ở MỌI edition<br/>— tức thì"]
        C["ADD COLUMN ... NOT NULL DEFAULT<br/>(ChangePassWordForLogInNext)"] --> D{"SQL Server edition?"}
        D -->|"Enterprise / Developer"| R3["✅ metadata-only"]
        D -->|"Standard"| R4["⚠️ REBUILD bảng AbpUsers<br/>+ khoá Sch-M"]

        style R1 fill:#c8e6c9,stroke:#2e7d32
        style R2 fill:#c8e6c9,stroke:#2e7d32
        style R3 fill:#c8e6c9,stroke:#2e7d32
        style R4 fill:#ffe0b2,stroke:#e65100
    ```

    Bảng user vài nghìn dòng thì rebuild cũng xong tức thì. **PHẦN 0 in ra Edition + số dòng `AbpUsers`** để tự lượng sức — đừng đoán.

---

## 6. Để lần sau không lặp lại

### 6.1. Thứ tạo bảng là `DbSet`, KHÔNG phải kế thừa ABP

Hiểu nhầm phổ biến: *"chỉ class kế thừa ABP entity mới tạo bảng"* — **sai cả hai chiều**.

```mermaid
flowchart TD
    subgraph SAME["Cùng kế thừa FullAuditedEntity của ABP"]
        A["TenantBranding"]
        B["GSVDataStore"]
    end
    A -->|"CÓ DbSet trong<br/>eKMapServerDbContext"| A2["✅ có bảng"]
    B -->|"KHÔNG có DbSet"| B2["❌ KHÔNG có bảng"]

    style A2 fill:#c8e6c9,stroke:#2e7d32
    style B2 fill:#ffcdd2,stroke:#c62828
```

Bằng chứng trong repo: `GSVDataStore`, `GSVInvoices`, `GSVMapUser`, `GSVTokenKey`, `GSVItem` **đều kế thừa `FullAuditedEntity`/`Entity` nhưng không có bảng nào**.

**Thứ thực sự tạo bảng là được đưa vào EF model**, qua một trong ba đường:

```mermaid
flowchart LR
    A["DbSet&lt;T&gt; trong DbContext"] --> M["📐 EF Model"]
    B["Navigation property từ<br/>entity đã map trỏ tới<br/><i>⚠️ dễ sót nhất</i>"] --> M
    C["modelBuilder.Entity&lt;T&gt;()<br/>trong OnModelCreating"] --> M
    M --> T["🗄️ Tạo bảng"]

    style B fill:#ffe0b2,stroke:#e65100
```

EF Core **không biết ABP là gì**. `Entity<T>`/`FullAuditedEntity` chỉ cho sẵn `Id` + cột audit. `[Table("...")]` chỉ *đặt tên* bảng nếu entity được map, **không** làm entity được map.

!!! warning "Chiều ngược lại mới nguy hiểm"
    Class POCO thuần, **không kế thừa gì của ABP, vẫn tạo bảng** nếu có `DbSet` hoặc bị navigation trỏ tới. *"Không kế thừa ABP"* **không phải tấm khiên**.

    Câu hỏi đúng để tự kiểm: **"class này có vào được EF model không?"** — không phải *"nó có kế thừa ABP không?"*.

    DTO thì an toàn: không nằm trong DbContext, không ai trỏ tới.

### 6.2. Bốn quy tắc cho migration tiếp theo {#quy-tac}

```mermaid
flowchart TD
    R1["1️⃣ ĐỌC KỸ migration EF sinh ra<br/>trước khi commit"] --> R1D["Đã từng có lần migrations add kèm<br/>41 lệnh AlterColumn rác<br/>⇒ rebuild 41 bảng chính.<br/><i>Nay snapshot đã đồng bộ (§6.3)<br/>nhưng vẫn phải đọc</i>"]
    R2["2️⃣ KHÔNG BAO GIỜ sinh lại<br/>InitialBaseline"] --> R2D["Nó nuốt các thay đổi chưa có<br/>migration vào bụng — đúng cách<br/>ChangePassWordForLogInNext biến mất"]
    R3["3️⃣ Cột NOT NULL vào bảng có dữ liệu<br/>⇒ LUÔN kèm DEFAULT"] --> R3D["Và giá trị mặc định phải có<br/>ý nghĩa nghiệp vụ đúng"]
    R4["4️⃣ Code có fallback khi NULL<br/>⇒ LUÔN hỏi: có cần backfill?"] --> R4D["LastPasswordChangeTime là ví dụ:<br/>fallback trông vô hại nhưng<br/>thành sự cố diện rộng"]

    style R2 fill:#ffe0b2,stroke:#e65100
    style R4 fill:#ffe0b2,stroke:#e65100
```

### 6.3. Tin tốt: lệch snapshot đã được sửa TẬN GỐC {#snapshot-da-sua}

```mermaid
flowchart LR
    A["ModelSnapshot.cs<br/><b>42</b> UseIdentityColumn()"] --- B{"="}
    B --- C["Designer của migration<br/>mới nhất<br/><b>42</b> UseIdentityColumn()"]
    B --> D["✅ Snapshot ĐỒNG BỘ<br/>⇒ migration sau KHÔNG còn<br/>đẻ ra 41 lệnh rác nữa"]

    style D fill:#c8e6c9,stroke:#2e7d32
```

Đã kiểm chứng ở thời điểm viết: `ModelSnapshot.cs` và `20260715022326_Add_LastFailedLoginAttemptTime.Designer.cs` **giống hệt nhau** (591 dòng khai báo, diff rỗng). Sự cố 41 `AlterColumn` được xử lý tận gốc từ 14/07 — không phải chỉ vá phần ngọn ([chi tiết](migration-identity-columns-issue.md#cach-xu-ly)).

→ **Không còn nợ kỹ thuật ở đây.** Nhưng vẫn giữ nguyên quy tắc 1 ở [§6.2](#quy-tac): đọc kỹ migration trước khi commit. Cách phát hiện lệch tái diễn: so `ModelSnapshot.cs` với `.Designer.cs` của migration mới nhất — hai file phải mô tả cùng một model.
