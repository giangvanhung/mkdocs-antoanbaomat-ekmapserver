# Hướng dẫn dùng tab "Giao diện" (Interface)

**Ngày:** 2026-07-09

**Phạm vi:** Tab **Giao diện** trong trang Cấu hình của app Administrator — ảnh hưởng tới trang Đăng nhập

**Đối tượng:** Người quản trị điền cấu hình, không cần biết lập trình

---

Hướng dẫn nhanh: mỗi ô trong tab **Giao diện** điền được **giá trị gì**, và khi điền thì
**trang Đăng nhập (Login) thay đổi ra sao**. Sau khi bấm **Lưu tất cả**, tải lại trang Login
là thấy thay đổi.

!!! danger "⛔ LƯU Ý QUAN TRỌNG — điền thiếu sẽ mất ảnh"

    Giá trị mặc định **chỉ dùng khi bạn chưa từng lưu cấu hình lần nào**. Cơ chế là
    **"tất cả hoặc không có gì"**, không phải từng ô riêng.

    - Khi bấm **Lưu** — dù chỉ điền 1 ô, để trống các ô còn lại — hệ thống lưu **đúng những gì
      đang có trên form**. Ô để trống = **rỗng**.
    - Kết quả: **logo / ảnh nền / slogan biến mất**, **không** tự quay về mặc định.
    - 👉 Vì vậy dòng "*(để trống)*" trong các bảng dưới **chỉ đúng khi bạn chưa từng lưu**. Đã lưu
      rồi thì ô trống = rỗng.
    - Muốn lấy lại mặc định → bấm nút **Khôi phục mặc định**.

    💡 **Mẹo:** khi chỉ muốn đổi vài ô, **đừng để trống** các ô khác — điền đầy đủ, hoặc bấm
    **Khôi phục mặc định** để nạp sẵn giá trị mặc định rồi chỉnh ô cần đổi.

---

## 0. Bố cục tab & khung Xem trước

Tab **Giao diện** chia làm 2 cột:

- **Cột trái** — các ô nhập, gom thành 5 nhóm: *Thông tin chung*, *Logo*, *Ảnh nền đăng nhập*,
  *Slogan*, *Màu sắc & Favicon*.
- **Cột phải** — khung **Xem trước trang đăng nhập**, cập nhật **ngay khi bạn gõ**, chưa cần bấm Lưu.

Khung xem trước mô phỏng: tiêu đề tab trình duyệt, vị trí ảnh nền / form (theo *Đảo bố cục*),
logo, và slogan. Nó **cố ý gắn sẵn class `text-white`** giống trang Login thật, nên đây là chỗ
để **thử ngay** xem giá trị Style bạn gõ có thắng được màu trắng mặc định hay không (xem mục 8.1).

> Xem trước chỉ mô phỏng bố cục, không phản ánh chính xác 100% trang thật. Muốn chắc chắn: bấm
> **Lưu tất cả** rồi tải lại trang Login.

Các ô **chưa có tác dụng** lên trang Login được gom chung vào nhóm *Màu sắc & Favicon* và gắn nhãn
🏷️ **Chưa áp dụng**.

---

## 1. Tất cả các ô trong tab

| # | Nhóm | Ô nhập | Điền được gì | Giới hạn | Đổi được giao diện? |
|---|---|---|---|---|---|
| 1 | Thông tin chung | Company Name | Tên tổ chức | 200 ký tự | ✅ Có |
| 2 | Thông tin chung | Đảo bố cục *(Reverse Layout)* | Bật / tắt | — | ✅ Có |
| 3 | Thông tin chung | Chỉ hiện form *(Form Only)* | Bật / tắt | — | ✅ Có |
| 4 | Logo | Logo URL | Đường dẫn ảnh logo | 500 ký tự | ✅ Có |
| 5 | Logo | Logo Style | Chuỗi CSS (kích thước, canh lề…) | 500 ký tự | ✅ Có |
| 6 | Logo | Logo Class | Tên class CSS | 500 ký tự | ✅ Có |
| 7 | Ảnh nền | Background URL | Đường dẫn ảnh nền | 500 ký tự | ✅ Có |
| 8 | Ảnh nền | Background Style | Chuỗi CSS cho ảnh nền | 500 ký tự | ✅ Có |
| 9 | Ảnh nền | Background Class | Tên class CSS | 500 ký tự | ✅ Có |
| 10 | Slogan | Slogan | Câu khẩu hiệu (nhiều dòng) | 1000 ký tự | ✅ Có |
| 11 | Slogan | Slogan Style | Chuỗi CSS cho slogan | 500 ký tự | ✅ Có |
| 12 | Slogan | Slogan Class | Tên class CSS | 500 ký tự | ✅ Có |
| 13 | Màu sắc & Favicon | Favicon URL | Đường dẫn ảnh icon | 500 ký tự | ⛔ Chưa hiển thị |
| 14 | Màu sắc & Favicon | Primary Color | Chọn màu (bảng màu hoặc gõ mã hex) | — | ⛔ Chưa hiển thị |
| 15 | Màu sắc & Favicon | Secondary Color | Chọn màu (bảng màu hoặc gõ mã hex) | — | ⛔ Chưa hiển thị |
| 16 | Màu sắc & Favicon | Font Family | Tên font chữ | 100 ký tự | ⛔ Chưa hiển thị |

> Các ô ⛔ **Chưa hiển thị**: bạn điền và lưu được, nhưng hiện **chưa làm đổi trang Login**.
> Hai ô màu có **2 cách nhập**: bấm ô vuông màu để mở bảng chọn, hoặc gõ thẳng mã hex (`#187DE4`)
> vào ô bên cạnh — hai ô luôn đồng bộ với nhau.

---

## 2. Ba loại ô lặp lại (Url / Style / Class)

Logo, ảnh nền, slogan đều có 3 ô cùng mẫu:

| Ô | Điền gì | Ví dụ |
|---|---|---|
| `...URL` | Đường dẫn tới ảnh có sẵn (không upload file ở đây) | `/resources/images/logo.png` · `https://cdn.abc.vn/bg.jpg` |
| `...Style` | Chuỗi CSS chỉnh trực tiếp | `width:150px;` · `background-size:cover;` · `color:#fff;` |
| `...Class` | Tên class có sẵn của giao diện (cách nhau bằng dấu cách) | `max-h-90px` · `text-center font-weight-bold` |

---

## 3. Company Name (Tên tổ chức)

| Giá trị điền | Trên trang Login |
|---|---|
| `eKMap Server` | Tên tab trình duyệt: `Đăng nhập - eKMap Server` |
| `Công ty ABC` | Tên tab: `Đăng nhập - Công ty ABC` |
| *(để trống)* | Chưa từng lưu → mặc định `eKMapServer`. Đã lưu → tab thành `Đăng nhập -` (trống). |

---

## 4. Reverse Layout (Đảo bố cục)

| Giá trị điền | Trên trang Login |
|---|---|
| ○ tắt (mặc định) | Form **bên trái** – ảnh nền **bên phải** |
| ● bật | Ảnh nền **bên trái** – form **bên phải** |

> Chỉ đổi trên màn hình lớn; màn hình nhỏ (điện thoại) luôn xếp dọc.

### 4.1. Chỉ hiện form (Form Only)

| Giá trị điền | Trên trang Login |
|---|---|
| ○ tắt (mặc định) | Hiện đủ **form + ảnh nền + slogan** |
| ● bật | **Chỉ** hiện khung đăng nhập, **ẩn** ảnh nền và slogan |

> Khi bật Form Only, các ô Background và Slogan vẫn lưu được nhưng **không hiển thị**.
> Bật thử công tắc này rồi nhìn khung **Xem trước** ở cột phải để thấy ngay khác biệt.

---

## 5. Logo (phía trên form đăng nhập)

| Ô | Ví dụ điền | Trên trang Login |
|---|---|---|
| Logo URL | `/resources/images/abc-logo.png` | Đổi ảnh logo |
| Logo URL | *(để trống)* | Chưa từng lưu → logo mặc định. Đã lưu → **logo trống/hỏng**. |
| Logo Style | `width:180px;` | Ép chiều rộng logo 180px |
| Logo Style | `max-width:180px; margin:0 auto;` | Giới hạn rộng + canh giữa |
| Logo Class | `max-h-90px` | Giới hạn chiều cao 90px |
| Logo Class | `max-h-90px text-center` | Giới hạn cao + canh giữa |

---

## 6. Favicon ⛔ (điền được nhưng chưa hiển thị)

| Ô | Ví dụ điền | Trên trang Login |
|---|---|---|
| Favicon URL | `/resources/images/favicon.ico` | **Hiện chưa có tác dụng** — icon tab chưa đổi theo ô này. |

---

## 7. Ảnh nền Login

| Ô | Ví dụ điền | Trên trang Login |
|---|---|---|
| Background URL | `https://cdn.abc.vn/bg-login.jpg` | Đổi ảnh nền |
| Background URL | *(để trống)* | Chưa từng lưu → ảnh nền mặc định. Đã lưu → **nền trống**. |
| Background Style | `background-size:cover; background-position:center;` | Ảnh phủ kín, canh giữa |
| Background Style | `background-size:contain; background-repeat:no-repeat;` | Ảnh vừa khung, không lặp |
| Background Class | `bgi-size-cover bgi-position-center` | Phủ/canh ảnh bằng class có sẵn |

---

## 8. Slogan (chữ lớn trên ảnh nền)

| Ô | Ví dụ điền | Trên trang Login |
|---|---|---|
| Slogan | `GIS Software`⏎`Make in`⏎`Viet Nam` | 3 dòng chữ (mỗi lần Enter = 1 dòng) |
| Slogan | `Chuyển đổi số`⏎`cho doanh nghiệp` | 2 dòng chữ |
| Slogan | *(để trống)* | Chưa từng lưu → slogan mặc định. Đã lưu → **không có slogan**. |
| Slogan Style | `color:#fff; font-size:2.5rem;` | Chữ trắng, cỡ lớn |
| Slogan Style | `letter-spacing:2px; text-shadow:0 2px 6px #000;` | Giãn chữ + đổ bóng |
| Slogan Class | `text-center font-weight-bolder` | Canh giữa + in đậm |

> Slogan chỉ nhận **chữ thường** — không chèn được thẻ HTML.

### 8.1. ⚠️ Đổi màu chữ Slogan — vì sao `color:` không ăn?

Trang Login **gán cứng** class `text-white` cho slogan. Trong Bootstrap, class này là:

```css
.text-white { color: #fff !important; }
```

Chữ `!important` khiến nó **thắng mọi giá trị `color`** bạn gõ ở ô Slogan Style. Vì vậy
`color:#FFD54F;` **không có tác dụng** — chữ vẫn trắng.

**Cách xử lý — dùng `-webkit-text-fill-color` thay cho `color`:**

Đây là thuộc tính quyết định **màu tô của chữ**, và nó được áp dụng **sau** `color`, nên nó
thắng kể cả khi `color` có `!important`.

| Muốn | Dán vào ô **Slogan Style** | Kết quả |
|---|---|---|
| ✅ Chữ vàng | `-webkit-text-fill-color:#FFD54F;` | **Chạy được** |
| ✅ Chữ đỏ | `-webkit-text-fill-color:#E4572E;` | **Chạy được** |
| ❌ Chữ vàng (cách cũ) | `color:#FFD54F;` | Bị `text-white` đè → vẫn trắng |

Ba cách, xếp theo mức độ chắc chắn:

| # | Cách làm | Dán vào ô Slogan Style | Đánh giá |
|---|---|---|---|
| 1 | `-webkit-text-fill-color` | `-webkit-text-fill-color:#FFD54F;` | ⭐ **Nên dùng.** Luôn thắng, chạy trên Chrome / Edge / Cốc Cốc / Firefox / Safari |
| 2 | Thêm `!important` vào `color` | `color:#FFD54F !important;` | Thường chạy, nhưng tuỳ cách trang Login gắn style — nếu không ăn thì dùng cách 1 |
| 3 | Nhờ lập trình viên bỏ `text-white` | — | Triệt để nhất, nhưng phải sửa mã nguồn |

> 💡 **Cách kiểm tra nhanh:** gõ vào ô Slogan Style rồi nhìn khung **Xem trước** ở cột phải.
> Khung này cũng gắn sẵn `text-white` y như trang thật, nên **thấy đổi màu ở đó = sẽ đổi màu thật**.

**Vì sao chữ gradient (mục 13.7) lại chạy được?** Vì công thức đó cũng đi qua
`-webkit-text-fill-color:transparent` — nó vô hiệu hoá `color:#fff !important` đúng theo cách 1.

**Các hiệu ứng chữ khác không bị ảnh hưởng** bởi `text-white` (vì chúng không dùng `color`):
`text-shadow`, `-webkit-text-stroke`, `font-size`, `letter-spacing`, `text-transform`,
`font-style`, `transform`, `opacity`, `background`.

---

## 9. Màu & Font ⛔ (điền được nhưng chưa hiển thị)

| Ô | Ví dụ điền | Trên trang Login |
|---|---|---|
| Primary Color | `#FF5722` | **Hiện chưa có tác dụng** |
| Secondary Color | `#263238` | **Hiện chưa có tác dụng** |
| Font Family | `Segoe UI` · `Roboto, sans-serif` | **Hiện chưa có tác dụng** |

---

## 10. Muốn đổi gì thì điền ô nào

| Muốn đổi… | Điền ô | Ví dụ |
|---|---|---|
| Tên trên tab trình duyệt | Company Name | `Công ty ABC` |
| Vị trí form / ảnh nền | Đảo bố cục | ● bật |
| Ẩn hẳn ảnh nền & slogan | Chỉ hiện form | ● bật |
| Ảnh logo | Logo URL | `/resources/images/abc-logo.png` |
| Kích thước logo | Logo Style | `max-width:180px;` |
| Ảnh nền trang | Background URL | `https://cdn.abc.vn/bg-login.jpg` |
| Cách hiển thị ảnh nền | Background Style | `background-size:cover;` |
| Câu khẩu hiệu | Slogan | `Chuyển đổi số`⏎`cho doanh nghiệp` |
| **Màu chữ slogan** | Slogan Style | `-webkit-text-fill-color:#FFD54F;` *(xem mục 8.1)* |
| Cỡ chữ slogan | Slogan Style | `font-size:2.5rem;` |

---

## 11. Hai ví dụ điền sẵn

**A — Đổi thương hiệu, form sang phải**

| Ô | Giá trị điền |
|---|---|
| Company Name | `Công ty ABC` |
| Logo URL | `/resources/images/abc-logo.png` |
| Logo Style | `max-width:180px;` |
| Background URL | `https://cdn.abc.vn/bg-login.jpg` |
| Background Style | `background-size:cover; background-position:center;` |
| Slogan | `Chuyển đổi số`⏎`cho doanh nghiệp` |
| Slogan Style | `font-size:2.5rem;` |
| Đảo bố cục | ● bật |

**B — Chỉ đổi slogan**

| Ô | Giá trị điền |
|---|---|
| Slogan | `Bản đồ số`⏎`cho mọi nhà` |
| *(các ô khác)* | ⚠️ Đừng để trống — sẽ mất logo/ảnh nền. Điền lại URL cũ hoặc bấm **Khôi phục mặc định** trước rồi chỉ sửa Slogan. |

**C — Slogan chữ vàng, nền tối cho dễ đọc**

| Ô | Giá trị điền |
|---|---|
| Background Style | `background-size:cover; background-color:rgba(0,0,0,.45); background-blend-mode:multiply;` |
| Slogan | `Bản đồ số`⏎`cho mọi nhà` |
| Slogan Style | `-webkit-text-fill-color:#FFD54F; font-size:2.5rem; text-shadow:0 2px 8px rgba(0,0,0,.7);` |
| Slogan Class | `text-center font-weight-bolder` |

> Dùng `-webkit-text-fill-color` chứ không phải `color` — lý do ở mục 8.1.

---

## 12. Các nút & khung xem trước

| Nút / khu vực | Ở đâu | Công dụng |
|---|---|---|
| **Lưu tất cả** | Đầu trang | Lưu toàn bộ cấu hình. Báo *Cập nhật thành công*. Trang Login đổi khi tải lại. |
| **Khôi phục mặc định** | Đầu tab Giao diện, trong khung cảnh báo vàng | Hỏi xác nhận → đưa toàn bộ giao diện Login về mặc định và nạp lại giá trị mặc định vào các ô. |
| **Xem trước trang đăng nhập** | Cột phải tab Giao diện | Hiện ngay kết quả khi bạn gõ, **không** cần bấm Lưu. Cuộn trang thì khung này vẫn dính theo. |

> ⚠️ Xem trước **chỉ thay đổi trên màn hình**, chưa ghi vào hệ thống. Phải bấm **Lưu tất cả**
> thì trang Login thật mới đổi.

---

## 13. Thư viện hiệu ứng CSS nâng cao (tròn, xoay, ẩn/hiện, di chuyển, đổ bóng…)

Đây là "kho" giá trị dán sẵn cho các ô **Style** và **Class**. Cứ **copy giá trị ở cột "Dán vào ô"**
đúng vào ô tương ứng.

### 13.0. Đọc trước — 3 quy tắc quan trọng

1. **Ô `...Style` = CSS trực tiếp.** Viết được: `transform`, `filter`, `box-shadow`, `border-radius`,
   `opacity`, `animation`… **KHÔNG viết được** hiệu ứng "khi rê chuột" (`:hover`) hay tự chế
   chuyển động mới.
2. **Muốn hiệu ứng chạy liên tục** (xoay hoài, nhịp đập): dùng `animation` và **gọi đúng tên hiệu
   ứng có sẵn** của hệ thống: `spinner-border`, `animate-wave`, `animation-spinner` (đều là **xoay tròn**),
   `spinner-grow` (**phình to – thu nhỏ**). Ví dụ: `animation: spinner-border 5s linear infinite;`
3. **Ô `...Class` = tên hiệu ứng dựng sẵn** (không phải CSS). Dùng cho **bo tròn nhanh, đổ bóng,
   và ẩn/hiện theo màn hình**.
4. **Một số thuộc tính bị trang Login "khoá cứng"** bằng `!important` — điển hình là **màu chữ
   slogan** (class `text-white`). Gõ `color:` sẽ **không ăn**; phải dùng thuộc tính thay thế
   (xem mục 8.1). Cứ gõ thử rồi nhìn khung **Xem trước** là biết ngay có ăn hay không.

### 13.1. Bo tròn – bo góc – viền (thường dùng cho Logo)

| Muốn | Dán vào ô | Giá trị |
|---|---|---|
| Logo tròn xoe | Logo Style | `border-radius:50%;` |
| Bo góc mềm | Logo Style | `border-radius:18px;` |
| Bo tròn (cách nhanh) | Logo Class | `rounded-circle` |
| Bo góc (cách nhanh) | Logo Class | `rounded-lg` · `rounded-pill` |
| Thêm viền | Logo Style | `border:3px solid #187DE4; padding:6px;` |
| Nền trắng sau logo trong suốt | Logo Style | `background:#fff; padding:10px; border-radius:12px;` |

### 13.2. Đổ bóng

| Muốn | Dán vào ô | Giá trị |
|---|---|---|
| Bóng nhẹ | Logo Style | `box-shadow:0 6px 16px rgba(0,0,0,.2);` |
| Bóng nổi mạnh | Logo Style | `box-shadow:0 15px 40px rgba(0,0,0,.35);` |
| Bóng (cách nhanh) | Logo Class | `shadow` · `shadow-sm` · `shadow-lg` |

### 13.3. Xoay – nghiêng – lật – di chuyển – phóng to (đứng yên)

Dùng `transform` trong ô **Style**. Ghép nhiều hiệu ứng bằng cách viết liền: `transform: rotate(10deg) scale(1.1);`

| Muốn | Dán vào ô | Giá trị |
|---|---|---|
| Nghiêng logo 10° | Logo Style | `transform: rotate(-10deg);` |
| Phóng to 20% | Logo Style | `transform: scale(1.2);` |
| Thu nhỏ | Logo Style | `transform: scale(0.8);` |
| Lật gương (trái↔phải) | Logo Style | `transform: scaleX(-1);` |
| Nghiêng đổ (skew) | Logo Style | `transform: skewX(-12deg);` |
| Dịch lên trên | Logo Style | `transform: translateY(-15px);` |
| Dịch sang phải | Logo Style | `transform: translateX(30px);` |
| Vừa xoay vừa phóng | Logo Style | `transform: rotate(8deg) scale(1.15);` |

### 13.4. Chuyển động chạy liên tục (xoay hoài / nhịp đập)

Dùng `animation` + tên hiệu ứng có sẵn. Cú pháp: `animation: <tên> <thời-gian> <kiểu> infinite;`

| Muốn | Dán vào ô | Giá trị |
|---|---|---|
| Logo xoay tròn chậm | Logo Style | `animation: spinner-border 6s linear infinite;` |
| Logo xoay nhanh | Logo Style | `animation: animate-wave 2s linear infinite;` |
| Logo nhịp đập (phình – xẹp) | Logo Style | `animation: spinner-grow 1.8s ease-in-out infinite;` |

> ⚠️ Chỉ 5 tên này chạy được: `spinner-border`, `animate-wave`, `animation-spinner` (xoay),
> `spinner-grow`, `animation-pulse` (nhịp/mờ). Gõ tên khác sẽ **không có gì xảy ra**.

### 13.5. Ẩn / hiện phần tử (kể cả theo kích thước màn hình)

| Muốn | Dán vào ô | Giá trị |
|---|---|---|
| Ẩn hẳn logo | Logo Class | `d-none` |
| Ẩn logo (cách khác) | Logo Style | `display:none;` |
| Làm mờ đi một nửa | Logo Style | `opacity:0.5;` |
| **Ẩn trên điện thoại, chỉ hiện máy tính** | Logo Class | `d-none d-lg-block` |
| **Chỉ hiện trên điện thoại, ẩn máy tính** | Logo Class | `d-lg-none` |
| Ẩn slogan trên điện thoại | Slogan Class | `d-none d-lg-flex` |

### 13.6. Bộ lọc màu cho ảnh (logo)

| Muốn | Dán vào ô | Giá trị |
|---|---|---|
| Chuyển ảnh sang trắng–đen | Logo Style | `filter: grayscale(1);` |
| Tăng sáng / tương phản | Logo Style | `filter: brightness(1.2) contrast(1.1);` |
| Làm mờ ảnh | Logo Style | `filter: blur(2px);` |
| Đổi tông màu (xoay màu) | Logo Style | `filter: hue-rotate(90deg);` |
| Ảnh có bóng theo hình (PNG trong suốt) | Logo Style | `filter: drop-shadow(0 8px 10px rgba(0,0,0,.4));` |

### 13.7. Chữ Slogan "xịn"

| Muốn | Dán vào ô | Giá trị |
|---|---|---|
| **Đổi màu chữ** ⚠️ | Slogan Style | `-webkit-text-fill-color:#FFD54F;` — **không** dùng `color:` (xem mục 8.1) |
| Chữ nền gradient (chuyển màu) | Slogan Style | `background:linear-gradient(90deg,#00c6ff,#0072ff); -webkit-background-clip:text; -webkit-text-fill-color:transparent;` |
| Bóng chữ (dễ đọc trên nền sáng) | Slogan Style | `text-shadow:0 2px 8px rgba(0,0,0,.7);` |
| Viền chữ | Slogan Style | `-webkit-text-stroke:1px #fff;` |
| CHỮ IN HOA | Slogan Style | `text-transform:uppercase;` |
| Giãn khoảng cách chữ | Slogan Style | `letter-spacing:4px;` |
| Chữ nghiêng | Slogan Style | `font-style:italic;` |
| Nghiêng cả dòng | Slogan Style | `transform:rotate(-5deg);` |
| Khối nền mờ sau chữ | Slogan Style | `background:rgba(0,0,0,.35); padding:14px 22px; border-radius:12px;` |
| Canh giữa chữ | Slogan Class | `justify-content-center text-center` |

### 13.8. Ảnh nền Login

| Muốn | Dán vào ô | Giá trị |
|---|---|---|
| Phủ kín, canh giữa | Background Style | `background-size:cover; background-position:center; background-repeat:no-repeat;` |
| Ảnh vừa khung, không cắt | Background Style | `background-size:contain;` |
| **Làm tối nền để chữ dễ đọc** | Background Style | `background-color:rgba(0,0,0,.45); background-blend-mode:multiply;` |
| Phủ màu thương hiệu lên ảnh | Background Style | `background-color:#187DE4; background-blend-mode:overlay;` |
| Nền đứng yên khi cuộn | Background Style | `background-attachment:fixed;` |

> 💡 Muốn tối nền mà **không** làm mờ chữ slogan → dùng `background-blend-mode` như trên (đừng dùng
> `filter` cho ảnh nền, vì nó làm mờ/tối luôn cả chữ).

### 13.9. Bảng tra nhanh: hiệu ứng nào làm ở ô nào

| Hiệu ứng | Làm ở ô | Làm được? |
|---|---|---|
| Bo tròn / bo góc / viền | Style hoặc Class | ✅ |
| Đổ bóng | Style hoặc Class | ✅ |
| Xoay / nghiêng / lật / phóng to (đứng yên) | Style (`transform`) | ✅ |
| Xoay hoài / nhịp đập (chuyển động liên tục) | Style (`animation` + tên có sẵn) | ✅ |
| Ẩn / hiện, ẩn-hiện theo màn hình | Class (`d-none`, `d-none d-lg-block`) | ✅ |
| Lọc màu ảnh (xám, sáng, mờ) | Style (`filter`) | ✅ |
| Chữ gradient / bóng / viền | Slogan Style | ✅ |
| **Đổi màu chữ slogan** | Slogan Style (`-webkit-text-fill-color`) | ✅ — nhưng `color:` thì ❌ (mục 8.1) |
| Đổi hiệu ứng **khi rê chuột vào** (`:hover`) | — | ❌ Cần lập trình viên sửa code |
| Tự tạo chuyển động mới (keyframe riêng) | — | ❌ Cần lập trình viên sửa code |

---

## 14. Dùng ảnh động / GIF cho Logo và Ảnh nền

Bạn có thể thay ảnh tĩnh bằng **ảnh động** — chỉ cần dán link ảnh động vào ô **Logo URL** hoặc
**Background URL** như ảnh thường. Trình duyệt **tự phát**, không cần chỉnh gì thêm.

| Định dạng | Logo URL | Background URL | Ghi chú |
|---|---|---|---|
| **GIF động** | ✅ Tự chạy, lặp vô hạn | ✅ Tự chạy, lặp vô hạn | Phổ biến nhất, chắc chắn chạy |
| **APNG** (png động) | ✅ | ✅ | Đuôi file vẫn là `.png` |
| **WebP động** | ✅ | ✅ | Nhẹ hơn GIF nhiều |
| **Video (.mp4 / .webm)** | ❌ | ❌ | Không phát được — cần lập trình viên sửa code |

**Lưu ý khi dùng:**

- **Link phải trỏ thẳng tới file ảnh** (kết thúc bằng `.gif`, `.webp`, `.png`…). Link **trang web**
  chứa ảnh (Giphy, Google Images…) sẽ **không hiện** — phải là link ảnh trực tiếp.
- **Không điều khiển được** tốc độ, số lần lặp, hay dừng ở khung cuối — GIF chạy sao thì hiện vậy.
  Muốn đổi thì phải sửa chính file GIF.
- Ảnh nền động (GIF nặng) có thể làm **chữ slogan khó đọc** và **tải trang login chậm / tốn dữ liệu**.
  Nên chọn file nhẹ, và cân nhắc làm tối nền (xem mục 13.8) để chữ dễ đọc.
- Vẫn áp dụng quy tắc **"tất cả hoặc không có gì"** ở khung cảnh báo đầu tài liệu: đã lưu thì để trống ô = mất ảnh.
