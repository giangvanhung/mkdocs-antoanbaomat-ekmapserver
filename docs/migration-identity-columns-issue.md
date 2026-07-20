# Sự cố: migration thêm 1 cột nhưng EF sinh thêm 41 lệnh `AlterColumn` thừa

**Ngày:** 2026-07-14 · **Migration:** `20260714025730_Add_LastPasswordChangeTime` · **Mức độ:** 🔴 Cao · **Trạng thái:** ✅ Đã xử lý tận gốc

---

## 1. Chuyện gì đã xảy ra

```mermaid
flowchart LR
    A["👤 Muốn: thêm 1 cột<br/>LastPasswordChangeTime<br/>vào AbpUsers"] --> B["$ dotnet ef migrations add"]
    B --> C["📄 Migration dài ~700 dòng"]
    C --> D["✅ 1 lệnh AddColumn<br/><i>đúng ý muốn</i>"]
    C --> E["❌ 41 lệnh AlterColumn<br/><i>đụng cột Id của gần như<br/>MỌI bảng trong DB</i>"]

    style E fill:#ffcdd2,stroke:#c62828,stroke-width:3px
    style D fill:#c8e6c9,stroke:#2e7d32
```

41 bảng bị đụng tới, gồm cả bảng lớn: `AbpUsers`, `AbpTenants`, `AbpAuditLogs`, `AbpSettings`, `gsvMap`, `gsvLayer`, `gsvLog`, `tenantBranding`, `AbpRoles`, `AbpPermissions`…

Nguyên nhân **không phải** do khai báo entity sai.

---

## 2. Vì sao EF làm vậy

```mermaid
flowchart TD
    A["📝 Model hiện tại<br/><i>đọc từ code C#</i>"] --> C{"EF so sánh"}
    B["📸 ModelSnapshot.cs<br/><i>file trong repo</i>"] --> C
    C --> D["Chênh lệch = nội dung migration"]

    E["🗄️ Database THẬT"] -.->|"❌ EF KHÔNG BAO GIỜ<br/>đọc database"| C

    style E fill:#eeeeee,stroke:#9e9e9e
    style B fill:#ffe0b2,stroke:#e65100
```

!!! danger "EF tin tuyệt đối vào ModelSnapshot. Snapshot sai ⇒ migration sai."

### Snapshot đã bị lệch như thế nào

```mermaid
flowchart LR
    subgraph S["Bằng chứng: đếm UseIdentityColumn()"]
        A["Add_FormOnly.Designer.cs<br/><i>migration gần nhất trước đó</i><br/><b>42</b>"]
        B["ModelSnapshot.cs<br/><i>bản đang commit</i><br/><b>0</b> 😱"]
    end
    A -.->|"lẽ ra phải<br/>GIỐNG HỆT NHAU"| B

    style B fill:#ffcdd2,stroke:#c62828
```

Designer của migration cuối **chính là** snapshot tại thời điểm đó. Snapshot mất sạch 42 khai báo identity trong khi Designer vẫn giữ đủ ⇒ file snapshot **đã bị ghi đè bằng bản cũ/sai** (nhiều khả năng: merge conflict giải quyết sai, hoặc một máy dev dùng phiên bản EF tooling khác sinh lại rồi commit đè).

### Hệ quả dây chuyền

```mermaid
flowchart TD
    A["📸 Snapshot nói:<br/>'cột Id KHÔNG phải identity'"] --> C{"EF thấy chênh lệch"}
    B["📝 Model từ code nói:<br/>'cột Id LÀ identity'"] --> C
    C --> D["Sinh 41 lệnh AlterColumn<br/>để 'sửa' cho khớp"]
    D --> E["❌ Nhưng DB THẬT đã có identity<br/>từ InitialBaseline rồi<br/>⇒ sửa một vấn đề KHÔNG TỒN TẠI"]

    style E fill:#ffcdd2,stroke:#c62828,stroke-width:3px
```

---

## 3. Vì sao nguy hiểm

SQL Server **không cho phép** thêm/bỏ `IDENTITY` bằng `ALTER COLUMN`. Để thực thi, EF/SQL Server buộc phải rebuild cả bảng:

```mermaid
flowchart LR
    A["1️⃣ Tạo bảng tạm<br/>schema mới"] --> B["2️⃣ Copy TOÀN BỘ<br/>dữ liệu sang"]
    B --> C["3️⃣ Drop bảng cũ"]
    C --> D["4️⃣ Rename bảng tạm"]
    D --> E["5️⃣ Dựng lại index,<br/>FK, constraint"]

    style B fill:#ffe0b2,stroke:#e65100
    style C fill:#ffcdd2,stroke:#c62828
```

**Nhân quy trình đó với 41 bảng** — gồm cả `AbpAuditLogs`, `gsvLog` (những bảng lớn nhất hệ thống):

```mermaid
flowchart TD
    A["41 × rebuild bảng"] --> B["⏱️ Downtime kéo dài"]
    A --> C["🔒 Khoá bảng diện rộng"]
    A --> D["📈 Log transaction phình to"]
    A --> E["💥 Nguy cơ hỏng dữ liệu<br/>nếu migration đứt giữa chừng"]
    B & C & D & E --> F["…tất cả chỉ để đạt được<br/>đúng trạng thái mà DB ĐÃ CÓ SẴN"]

    style F fill:#ffcdd2,stroke:#c62828,stroke-width:3px
```

!!! danger "Cái bẫy độc nhất: nó chạy êm trên máy dev"
    ```mermaid
    flowchart LR
        A["💻 Máy dev<br/>DB nhỏ, không tải"] -->|"chạy êm ru ✅"| B["Yên tâm commit"]
        B --> C["🖥️ Production<br/>DB lớn, đang phục vụ"]
        C -->|"💥"| D["Mới bộc lộ"]

        style D fill:#ffcdd2,stroke:#c62828
    ```

---

## 4. Cách đã xử lý {#cach-xu-ly}

```mermaid
flowchart TD
    S1["1️⃣ GIỮ snapshot do EF sinh mới<br/><i>lần này nó có đủ 42 UseIdentityColumn()</i>"] --> S1R["✅ Snapshot tự nó ĐÃ ĐÚNG<br/>⇒ KHÔNG revert file này"]
    S2["2️⃣ CẮT 41 lệnh AlterColumn<br/>khỏi file migration"] --> S2R["✅ Chỉ giữ 1 lệnh AddColumn thật sự cần"]

    S1R --> K["🎯 Kết quả"]
    S2R --> K
    K --> K1["Migration này an toàn"]
    K --> K2["⭐ Các migration SAU sẽ KHÔNG<br/>sinh lại đám rác này nữa<br/><i>— chặn tận gốc, không vá ngọn</i>"]

    style K2 fill:#c8e6c9,stroke:#2e7d32,stroke-width:3px
```

Migration sau khi cắt gọn:

```csharp
public partial class Add_LastPasswordChangeTime : Migration
{
    protected override void Up(MigrationBuilder migrationBuilder)
    {
        migrationBuilder.AddColumn<DateTime>(
            name: "LastPasswordChangeTime",
            table: "AbpUsers",
            type: "datetime2",
            nullable: true);
    }

    protected override void Down(MigrationBuilder migrationBuilder)
    {
        migrationBuilder.DropColumn(
            name: "LastPasswordChangeTime",
            table: "AbpUsers");
    }
}
```

!!! question "Cắt bỏ 41 lệnh có làm schema lệch không?"
    **Không.** Database thật **vốn đã** có các cột identity đó — do `InitialBaseline` tạo ra. Bỏ qua 41 lệnh `AlterColumn` không làm schema lệch đi đâu cả; chúng vốn là lệnh "sửa" một thứ đã đúng sẵn.

---

## 5. Kiểm chứng

### Lúc xử lý sự cố

```bash
dotnet ef migrations script Add_FormOnly Add_LastPasswordChangeTime
```

Kết quả — **đúng một lệnh**, không có thao tác rebuild nào:

```sql
BEGIN TRANSACTION;
ALTER TABLE [AbpUsers] ADD [LastPasswordChangeTime] datetime2 NULL;
COMMIT;
```

### Trạng thái hiện tại (kiểm lại 17/07/2026)

```mermaid
flowchart LR
    A["ModelSnapshot.cs<br/><b>42</b> UseIdentityColumn()<br/>591 dòng khai báo"] --- E{"diff"}
    B["Add_LastFailedLoginAttemptTime<br/>.Designer.cs<br/><b>42</b> · 591 dòng"] --- E
    E --> R["✅ RỖNG — giống hệt nhau<br/>⇒ snapshot đang ĐỒNG BỘ"]

    style R fill:#c8e6c9,stroke:#2e7d32,stroke-width:3px
```

Và **không commit nào** trong lịch sử repo còn chứa `migrationBuilder.AlterColumn` — đã kiểm tra toàn bộ 4 commit từng đụng vào thư mục `Migrations/`.

→ **Sự cố đã đóng.** Migration mới sẽ không tái sinh đám rác này.

---

## 6. Bài học

```mermaid
flowchart TD
    B1["📖 LUÔN đọc migration<br/>trước khi commit"] --> B1D["migrations add KHÔNG phải lệnh<br/>'chạy rồi quên'. Migration đáng lẽ<br/>thêm 1 cột mà dài 700 dòng<br/>= dấu hiệu bất thường RÕ RÀNG"]

    B2["📸 ModelSnapshot là SOURCE CODE,<br/>không phải file rác tự sinh"] --> B2D["Merge conflict ở file này:<br/>TUYỆT ĐỐI không chọn bừa ours/theirs.<br/>Cách đúng: lấy nhánh đích rồi<br/>SINH LẠI migration của mình trên nền đó"]

    B3["🔍 Cách phát hiện snapshot lệch"] --> B3D["So ModelSnapshot.cs với .Designer.cs<br/>của migration mới nhất.<br/>Hai file phải mô tả CÙNG một model.<br/>Khác nhau ⇒ snapshot hỏng"]

    B4["🔧 Thống nhất phiên bản EF tooling<br/>giữa các máy dev"] --> B4D["Version khác nhau sinh snapshot<br/>định dạng khác nhau:<br/>.Annotation('SqlServer:Identity', …)<br/>vs .UseIdentityColumn()<br/>⇒ nguồn gốc phổ biến của lệch.<br/>Nên ghim version trong dotnet-tools.json"]

    B5["🤖 Cân nhắc thêm CI check"] --> B5D["Job chạy migrations script và FAIL<br/>nếu SQL chứa drop/rebuild bảng<br/>ngoài dự kiến ⇒ chặn được<br/>trước khi lên production"]

    style B1 fill:#fff9c4,stroke:#f57f17
    style B2 fill:#fff9c4,stroke:#f57f17
    style B3 fill:#fff9c4,stroke:#f57f17
```

!!! tip "Quy trình an toàn cho mọi migration về sau"
    ```mermaid
    flowchart LR
        A["dotnet ef<br/>migrations add"] --> B["📖 ĐỌC file migration"]
        B --> C{"Có lệnh nào<br/>ngoài dự kiến?"}
        C -->|"có"| D["🛑 DỪNG<br/>snapshot đang lệch"]
        C -->|"không"| E["dotnet ef<br/>migrations script"]
        E --> F{"SQL có DROP CONSTRAINT<br/>hay tạo bảng tạm?"}
        F -->|"có"| D
        F -->|"không"| G["✅ Commit"]

        style D fill:#ffcdd2,stroke:#c62828
        style G fill:#c8e6c9,stroke:#2e7d32
    ```

*Xem thêm: [Cập nhật DB đang chạy lên schema hiện tại](db-update-existing-database.md) — quy trình đưa delta vào DB production mà không đụng dữ liệu.*
