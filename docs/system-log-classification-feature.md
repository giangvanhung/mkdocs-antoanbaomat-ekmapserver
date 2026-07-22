# Phân loại nhật ký hệ thống theo 05 nhóm

> Gom nhật ký sẵn có thành **5 nhóm** cho màn hình tra cứu. Không tạo bảng log mới, không migration.

## 1. Năm nhóm nhật ký

| # | Nhóm | Trả lời câu hỏi | Nguồn dữ liệu |
|---|---|---|---|
| i | Truy cập phần mềm | Ai gọi gì, lúc nào, từ đâu | `AbpAuditLogs` |
| ii | Đăng nhập quản trị | Ai đăng nhập admin, thành/bại | `AbpUserLoginAttempts` |
| iii | Lỗi phát sinh | Lỗi gì, ở đâu | `AbpAuditLogs` (có lỗi) + `Logs.txt` |
| iv | Quản lý tài khoản | Tài khoản/quyền đổi **từ gì → gì** | `AbpEntityChanges` |
| v | Thay đổi cấu hình | Tham số đổi **từ gì → gì** | `AbpEntityChanges` |

## 2. Cách hoạt động

```mermaid
flowchart LR
    subgraph GHI["✍️ GHI — ABP tự làm (không viết code)"]
        A1["Gọi dịch vụ"] --> S1[("AbpAuditLogs")]
        A2["Đăng nhập"] --> S2[("AbpUserLoginAttempts")]
        A3["Sửa User / Role / Cấu hình"] --> S3[("AbpEntityChanges")]
    end
    subgraph DOC["👓 ĐỌC — phần mình viết"]
        S1 --> Q["SystemLogAppService<br/>gắn nhãn nhóm i..v"]
        S2 --> Q
        S3 --> Q
    end
    Q --> NG["🅰️ Màn hình 5 tab + biểu đồ"]

    style S3 fill:#fff3cd,stroke:#f39c12
    style Q fill:#c8e6c9,stroke:#2e7d32
```

**Hai điều cốt lõi:**

- **Không viết code ghi log** — ABP đã ghi sẵn ở tầng khung. Chỉ cần **bật theo dõi thay đổi** cho `User / Role / UserRole / Setting` (cho nhóm iv, v).
- **Phân nhóm lúc ĐỌC**, không phải lúc ghi. CSDL **không có bảng log hợp nhất, không có cột "nhóm"** — mỗi nhóm là một truy vấn trên nguồn tương ứng, cùng quy về một DTO.

## 3. File nào — sửa gì

> ➕ thêm mới · ✏️ sửa

```mermaid
flowchart TB
    subgraph CORE["📁 Nền tảng — Core"]
        c1["📄 eKMapServerCoreModule.cs<br/>✏️ bật theo dõi thay đổi<br/>(User / Role / UserRole / Setting)"]
    end
    subgraph APP["📁 Nghiệp vụ — Application"]
        a1["📄 SystemLogAppService.cs<br/>➕ truy vấn phân 5 nhóm + số liệu biểu đồ"]
        a2["📄 SystemLogItemDto + enum nhóm<br/>➕ DTO chung cho cả 5 nhóm"]
    end
    subgraph NG["📁 Giao diện — Administrator"]
        n1["📄 system-log-categories.component<br/>➕ màn hình 5 tab + bảng + biểu đồ"]
        n2["📄 system-log-detail / charts<br/>➕ popup chi tiết + xem biểu đồ đầy đủ"]
    end

    style CORE fill:#fff3e0,stroke:#fb8c00
    style APP fill:#e8f5e9,stroke:#43a047
    style NG fill:#e3f2fd,stroke:#1e88e5
```

> Quyền: **dùng lại** `Administration.Audit` sẵn có — không tạo quyền mới, không đụng phân quyền vai trò.

## 4. Cần nhớ

!!! question "Bảng lưu 5 nhóm ở đâu? — KHÔNG có bảng nào cả"
    Có chủ đích. Log đã nằm ở các bảng ABP sẵn có; chúng chỉ **thiếu nhãn nhóm**, mà nhãn đó suy ra 100% từ nội dung. Tạo bảng hợp nhất = nhân đôi dung lượng + phải sửa 3 chỗ ghi của ABP (vỡ khi nâng cấp).

!!! note "Nhóm iv/v ghi được 'từ gì → gì'"
    Nhờ bật **theo dõi thay đổi (EntityHistory)**: so giá trị cũ ↔ mới mỗi khi lưu. `AbpAuditLogs` thường chỉ có tham số **mới**, không có giá trị cũ để đối chiếu.

!!! warning "Giới hạn: chỉ bắt thay đổi qua ứng dụng"
    Theo dõi thay đổi móc lúc lưu qua ứng dụng, **không phải** ở tầng database. Sửa thẳng DB (SSMS/`psql`) hay sửa tay `appsettings.json` **không để lại vết**. → Ghi rõ giới hạn này khi nghiệm thu, kiểm soát bù bằng **phân quyền truy cập DB**.

!!! danger "Không hồi tố — bật càng sớm càng tốt"
    Nhóm iv/v **rỗng** cho tới khi có thay đổi mới sau khi bật; lịch sử trước đó không dựng lại được. Nên deploy bước "bật theo dõi thay đổi" **riêng và sớm** để tích lũy dữ liệu.

!!! tip "Không migration"
    3 bảng theo dõi thay đổi đã có sẵn trong schema (do ABP sinh). Chỉ bật cấu hình + build lại + restart, **không cần `dotnet ef`**.
