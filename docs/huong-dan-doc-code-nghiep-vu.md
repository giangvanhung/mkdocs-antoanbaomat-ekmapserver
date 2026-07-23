# Hướng dẫn đọc & xử lý code nghiệp vụ

Tài liệu này là **cẩm nang thực hành** để đọc hiểu và sửa code nghiệp vụ eKMapServer mà không bị ngợp. Không cần nhớ cả hệ thống — chỉ cần **một tấm bản đồ + một công thức lặp lại**.

!!! tip "Câu thần chú"
    **Hệ này chỉ có 2 nửa: QUẢN LÝ bản đồ và PHỤC VỤ bản đồ.**

    Gặp bất kỳ tính năng nào, hỏi đúng 1 câu: *"Cái này thuộc nửa nào?"* → biết ngay đi tìm ở đâu.

---

## 1. Sơ đồ 2 nửa {#hai-nua}

```mermaid
flowchart TB
    subgraph QL["🗂️ NỬA QUẢN LÝ — app Manager"]
        direction TB
        H1["Nút bấm (.html)<br/>(click)=&quot;tênHàm&quot;"]
        H2["Component (.ts)<br/>abp.services.app.gSV___.method()"]
        H3["GSV___AppService.cs<br/>(mỏng: map DTO + kiểm quyền)"]
        H4["___Manager.cs<br/>(Business = LOGIC THẬT)"]
        H5["IRepository (EF Core)"]
        H1 --> H2 --> H3 --> H4 --> H5
    end
    subgraph PV["🌐 NỬA PHỤC VỤ — /rest/services/..."]
        direction TB
        P1["Mapbox / OpenLayers<br/>(trình duyệt)"]
        P2["Controller (Web.Services)"]
        P3["Business/GISServices"]
        P4["Provider.Core<br/>ĐỌC dữ liệu không gian"]
        P5["Render.Core<br/>VẼ ảnh PNG / vector tile"]
        P1 -->|"URL tile"| P2 --> P3
        P3 --> P4
        P3 --> P5
    end
    H5 --> DB[("DB metadata<br/>GSVMap, GSVLayer...")]
    DB -.->|"đọc qua cache"| P3
    P4 --> SRC[("Nguồn dữ liệu:<br/>PostGIS / Shapefile / MBTiles")]
    P5 -->|"tile"| P1
    style QL fill:#e3f2fd,stroke:#1565c0
    style PV fill:#e8f5e9,stroke:#2e7d32
```

| Nửa | Là gì | Chạm vào |
|---|---|---|
| **QUẢN LÝ** (app Manager) | tạo / sửa / xóa / publish bản đồ | **DB (bảng)** qua EF Core |
| **PHỤC VỤ** (`/rest/services/...`) | trả ảnh/tile cho người xem | **Provider** (đọc) + **Render** (vẽ) + **đĩa** (cache) |

**Provider** = đọc dữ liệu không gian. **Render** = vẽ ảnh. **Business** đứng trên, điều phối cả hai.

---

## 2. Công thức 4 bước — trace MỌI tính năng {#cong-thuc}

Không cần nghĩ, cứ đi 4 bước này mỗi lần, y hệt nhau:

```mermaid
flowchart LR
    B1["1️⃣ Nút ở đâu?<br/><b>.html</b><br/>tìm (click)=tênHàm"]
    B2["2️⃣ Hàm gọi gì?<br/><b>.ts</b><br/>abp.services.app.gSV___.method"]
    B3["3️⃣ Backend nào?<br/><b>GSV___AppService.cs</b><br/>tìm Method"]
    B4["4️⃣ Logic thật?<br/><b>___Manager.*.cs</b><br/>(Business)"]
    B1 --> B2 --> B3 --> B4
    style B1 fill:#fff3e0,stroke:#e65100
    style B4 fill:#e8f5e9,stroke:#2e7d32
```

!!! success "Món quà lớn nhất của codebase: TÊN KHỚP XUYÊN SUỐT"
    `gSVMap` (JS) → `GSVMapAppService` (backend) → `MapManager` (business). Cứ theo chữ mà lần, **không cần đoán**.

    Mẹo: đọc **ngược từ bước 4 → 1**. Hiểu logic trước, rồi mới xem ai gọi nó.

---

## 3. Đọc TÊN FILE thay vì đọc hết file {#duoi-file}

File được tách sẵn theo **hành động** (`partial class`) — nhìn đuôi tên là biết vào đâu, khỏi đọc cả đống:

| Muốn sửa... | Mở file đuôi... |
|---|---|
| Danh sách / tìm kiếm | `*.Select.cs` |
| Sửa thông tin | `*.Update.cs` |
| Xóa | `*.Delete.cs` |
| Xuất bản | `*.Publish.cs` |
| Cache | `*.Cache.cs` |

> Sửa "xóa bản đồ"? → chỉ mở `MapManager.Delete.cs`. Khỏi đụng file khác.

---

## 4. Bản đồ domain {#domain}

Tiền tố **`GSV`** = GIS Service. Frontend và backend chia domain trùng khớp:

| Domain | Là gì | Frontend module |
|---|---|---|
| `GSVMap` | Bản đồ (CRUD, publish, share) | `modules/manager` |
| `GSVLayer` | Lớp dữ liệu trong bản đồ | (trong manager) |
| `GSVCollection` | Chuyên đề / thư mục nhóm | `modules/manager` |
| `GSVPackage` | Gói dữ liệu upload | `modules/manager/mapPackage` |
| `GSVProject` | Dự án (nhóm token/API key) | `modules/project` |
| `GSVToken` | API key để consumer dùng dịch vụ | `modules/apikey` |

---

## 5. Dịch vụ nào dùng Render, dịch vụ nào không {#render-hay-khong}

!!! note "Quy tắc"
    **Xuất ẢNH/TILE → có Render. Xuất DỮ LIỆU THÔ → không Render (chỉ Provider).**

| Dịch vụ | Render? | Ai vẽ |
|---|:---:|---|
| WMS, WMTS, MapServer (tile ảnh) | ✅ | Server (PNG) |
| VectorTileServer (MVT) | ✅ | Client (Mapbox) |
| WFS, FeatureServer (GeoJSON/GML) | ❌ | — (chỉ data) |

Cả **OGC** (chuẩn mở: WMS/WFS/WMTS) và **OGS** (định dạng riêng eKMap: MapServer/FeatureServer/VectorTileServer) đều dùng chung lõi Render khi cần xuất ảnh.

---

## 6. Quy tắc vàng để KHÔNG bị ngợp khi sửa {#quy-tac-vang}

1. **Đọc theo 1 sợi dây, không đọc ngang.** Trace tính năng A thì kệ B. Bỏ qua 90% còn lại.
2. **Sửa ở tầng thấp nhất.** Đổi logic → sửa trong `Manager` (Business), đừng đụng frontend nếu không cần.
3. **Đổi giao diện/nút → chỉ frontend. Đổi cách tính/lọc → chỉ Business.** Hiếm khi phải sửa cả hai.
4. **Copy tính năng có sẵn thay vì tự nghĩ.** Hệ này rất đều tay — cái mới gần như luôn giống cái cũ.

---

## 7. Thứ tự học đề nghị (dễ → khó) {#thu-tu-hoc}

```mermaid
flowchart LR
    L1["1. Chỉ đọc<br/>*.Select.cs"] --> L2["2. Ghi<br/>*.Update / *.Delete"] --> L3["3. Publish<br/>Provider đọc package"] --> L4["4. Phục vụ / Render<br/>(để cuối)"]
    style L1 fill:#e8f5e9,stroke:#2e7d32
    style L4 fill:#ffebee,stroke:#c62828
```

!!! warning "Đừng học Render/Provider trước"
    Đó là bẫy gây nản. **80% việc thực tế nằm ở nửa QUẢN LÝ đơn giản.**

---

## 8. Ví dụ mẫu đã trace sẵn: nút "Xem bản đồ" (View) {#vi-du-view}

```mermaid
sequenceDiagram
    autonumber
    participant U as Người dùng
    participant HTML as manager.component.html
    participant TS as manager-view-map.component.ts
    participant API as GSVMapAppService
    participant MB as Mapbox GL (client)
    participant BE as Backend phục vụ

    U->>HTML: Click "View"
    HTML->>TS: onViewMapService(mapId) → mở popup
    TS->>API: getServiceInfoOfMap(mapId)
    API-->>TS: statusService + token
    Note over TS: nếu "Running" → cảnh báo "đang xử lý", đóng
    TS->>API: getPublishingInfo(mapId)
    API-->>TS: layers, provider, tâm/zoom
    TS->>MB: new mapboxgl.Map() trỏ URL /rest/services/{mapId}/WMS
    loop mỗi ô tile
        MB->>BE: GET tile (WMS GetMap)
        Note over BE: Cache → Provider đọc dữ liệu → Render vẽ PNG
        BE-->>MB: ảnh PNG
    end
    MB-->>U: bản đồ hiện lên (loading = false)
```

!!! info "Điểm chốt"
    Nút View **không tự vẽ** — nó chỉ **đấu Mapbox vào URL dịch vụ**. Backend mới vẽ từng tile khi Mapbox gọi tới.

---

## 9. Ví dụ mẫu: nút "Xóa Cache" {#vi-du-xoa-cache}

**Vì sao có nút này:** mỗi ô tile PNG được vẽ 1 lần rồi **lưu vào đĩa** (cache) để lần sau khỏi vẽ lại. Khi bạn sửa style/dữ liệu, tile cũ vẫn còn → người xem thấy bản đồ **CŨ**. Nút "Xóa Cache" dọn tile cũ, buộc server vẽ lại theo dữ liệu mới.

**Bấm vào ảnh hưởng gì:**

| Câu hỏi | Trả lời |
|---|---|
| Xóa bảng nào trong DB? | **Không.** Chỉ `Directory.Delete` / `File.Delete` trên đĩa. |
| Mất dữ liệu bản đồ / layer / feature? | **Không.** Dữ liệu gốc ở DB/PostGIS/file — không đụng. |
| Sau khi xóa? | Lần đầu xem lại **chậm hơn chút** vì server vẽ lại tile, rồi tự cache lại. |
| An toàn không? | **Có.** Cache tự sinh lại khi có người xem. |

> Đường đi: `edit.component.html` → `onDeleteCache()` → `gSVMap.deleteCache(mapId)` → `MapManager.DeleteCache(map)` (`MapManager.Delete.cs`).
>
> **Xóa Cache = "làm mới" hình ảnh bản đồ, KHÔNG phải xóa bản đồ.** Đây là chức năng thuộc nửa **PHỤC VỤ** (chạm đĩa), dù nút nằm trong màn hình sửa.
