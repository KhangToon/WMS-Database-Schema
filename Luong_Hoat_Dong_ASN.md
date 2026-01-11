# 🔄 LUỒNG HOẠT ĐỘNG HỆ THỐNG ASN - WMS (ĐA NGUỒN)

> **Module:** ASN (Advanced Shipping Notice - Kế hoạch nhận hàng)
> **Tích hợp:** PO (Mua hàng), RTR (Trả hàng), SRR (Đổi NCC), ODR (Chuyển kho)
> **Version:** 2.0 (Cập nhật Multi-source)
> **Ngày cập nhật:** 2026-01-11

---

## 🎯 MỤC TIÊU
Hệ thống tự động tạo ASN từ 4 nguồn chứng từ khác nhau khi chúng được **DUYỆT (APPROVED)** hoặc **CHỜ XUẤT (WAITING FOR EXPORT)**.

| Nguồn | Tên đầy đủ | Điều kiện kích hoạt (Trigger) | Loại ASN |
| :--- | :--- | :--- | :--- |
| **PO** | Purchase Order (Đơn hàng mua) | Trạng thái chuyển sang **Approved** | Đơn hàng PO |
| **RTR** | Return to Warehouse (Đề nghị trả kho) | Trạng thái chuyển sang **Approved** | Nhập trả về kho |
| **SRR** | Supplier Return Request (Đề nghị đổi trả NCC) | Trạng thái chuyển sang **Approved** (*) | Đổi hàng NCC |
| **ODR** | Outbound Delivery Request (Yêu cầu xuất kho) | Trạng thái chuyển sang **Waiting for Export** (**) | Nhập chuyển kho |

> (*) SRR chỉ tạo ASN cho các dòng có loại là "ĐỔI".
> (**) Chỉ áp dụng với ODR loại "Xuất chuyển kho".

---

## 📋 CHI TIẾT TỪNG CASE (TRƯỜNG HỢP)

### **CASE 1: TẠO ASN TỪ PO (Đơn hàng mua)**
> **Logic:** Nhập hàng mới từ Nhà cung cấp.

**Bước 1: Trigger**
- PO được duyệt (Status: `Approved`).

**Bước 2: Kiểm tra (Validation)**
- `PO.ASN_Created` = FALSE (Chưa tạo ASN trước đó).
- PO có ít nhất 1 dòng item.

**Bước 3: Tạo ASN Header**
- `ASN_Type` = 'Đơn hàng PO'.
- `PO_Id` = PO hiện tại.
- `Supplier_Id` = Supplier của PO.
- `Warehouse_Id`, `LegalEntity_Id` lấy từ PO.

**Bước 4: Tạo ASN Details**
- Copy toàn bộ `PO_Item` sang `ASN_Item`.
- `Transaction_Quantity` = `PO_Item.Quantity`.

**Bước 5: Hậu xử lý**
- Cập nhật `PO.ASN_Created` = TRUE.
- Ghi Log: "Tạo ASN từ PO [Mã PO]".

---

### **CASE 2: TẠO ASN TỪ RTR (Đề nghị trả kho)**
> **Logic:** Các bộ phận/nhân viên trả lại hàng thừa hoặc hỏng về kho.

**Bước 1: Trigger**
- RTR được duyệt (Status: `Approved`).

**Bước 2: Kiểm tra**
- Chưa tạo ASN cho RTR này.

**Bước 3: Tạo ASN Header**
- `ASN_Type` = 'Nhập trả về kho'.
- `RTR_Id` = RTR hiện tại.
- `Supplier_Id` = NULL.
- `Warehouse_Id` = Kho nhận trong RTR.
- `Reference_Code` = Mã RTR.

**Bước 4: Tạo ASN Details**
- Copy item từ RTR sang `ASN_Item`.
- `Owner_Reference` = Tên Nhân viên/Bộ phận trả (để biết hàng của ai).
- `Transaction_Quantity` = SL trả duyệt.

**Bước 5: Hậu xử lý**
- Đánh dấu RTR đã tạo ASN.
- Ghi Log.

---

### **CASE 3: TẠO ASN TỪ SRR (Đề nghị đổi trả NCC)**
> **Logic:** Đổi hàng lỗi với NCC. Gửi hàng lỗi đi, nhận hàng mới về. ASN dùng để nhận hàng mới.

**Bước 1: Trigger**
- SRR được duyệt (Status: `Approved`).

**Bước 2: Kiểm tra (Quan trọng)**
- SRR phải có ít nhất 1 dòng item có loại là **"ĐỔI"**.
- Nếu toàn bộ là "TRẢ" (không đổi) -> **KHÔNG** tạo ASN.

**Bước 3: Tạo ASN Header**
- `ASN_Type` = 'Đổi hàng NCC'.
- `SRR_Id` = SRR hiện tại.
- `Supplier_Id` = NCC của SRR.
- `Warehouse_Id` = Kho nhận lại hàng đổi.

**Bước 4: Tạo ASN Details**
- **CHỈ** copy những dòng item có `Type` = 'ĐỔI'.
- Bỏ qua những dòng 'TRẢ'.
- `Transaction_Quantity` = SL đổi.

**Bước 5: Hậu xử lý**
- Đánh dấu SRR đã xử lý.
- Ghi Log.

---

### **CASE 4: TẠO ASN TỪ ODR (Yêu cầu chuyển kho)**
> **Logic:** Kho A xuất chuyển sang Kho B. Kho B cần ASN để nhận hàng.

**Bước 1: Trigger**
- ODR loại "Xuất chuyển kho" chuyển trạng thái sang **Waiting for Export** (Đang chờ xuất).

**Bước 2: Kiểm tra**
- `ODR_Type` = 'Transfer' (Chuyển kho).
- Chưa tạo ASN.

**Bước 3: Tạo ASN Header**
- `ASN_Type` = 'Nhập chuyển kho'.
- `ODR_Id` = ODR hiện tại.
- `Supplier_Id` = NULL.
- `Warehouse_Id` = **Kho Nhập** (Destination Warehouse trong ODR).
- `Source_Warehouse_Id` = **Kho Xuất** (Source Warehouse trong ODR).

**Bước 4: Tạo ASN Details**
- Copy item từ ODR sang `ASN_Item`.
- `Transaction_Quantity` = SL yêu cầu chuyển.

**Bước 5: Hậu xử lý**
- Đánh dấu ODR đã tạo ASN.
- Ghi Log.

---

## 🔄 QUY TRÌNH CHUNG SAU KHI TẠO ASN (ÁP DỤNG MỌI CASE)

**1. Trạng thái Mới (New):**
- ASN vừa tạo có trạng thái 'Mới'.
- % Hoàn tất = 0%.

**2. Nhập kho (GR - Goods Receipt):**
- Nhân viên kho mở ASN, chọn **"Tạo phiếu nhập"**.
- Hệ thống hỗ trợ tạo GR từ ASN (xem tài liệu *Luong_Hoat_Dong_GR.md*).

**3. Cập nhật tiến độ:**
- Khi GR hoàn tất -> Cập nhật `Actual_Received_Quantity` trên `ASN_Item`.
- Tính lại % Hoàn tất.

**4. Hoàn thành:**
- Khi nhập đủ 100% -> ASN chuyển sang 'Hoàn thành'.

---

## ⚠️ CÁC LƯU Ý KỸ THUẬT (TECHNICAL NOTES)

1. **Mapping Dữ liệu:**
   - Các trường `Warehouse`, `LegalEntity`, `User` bắt buộc phải khớp ID giữa các hệ thống (PO, WMS, Request).
   - Nếu Source Document dùng mã (Code), cần lookup ra ID tương ứng trong WMS.

2. **Xử lý bất đồng bộ:**
   - Việc tạo ASN nên chạy **Background Job** để không làm chậm thao tác Duyệt của người dùng.
   - Cần cơ chế **Retry** nếu tạo ASN thất bại.

3. **Idempotency (Tính duy nhất):**
   - Luôn kiểm tra flag `created` hoặc query ASN tồn tại với `Reference_Code` tương ứng trước khi tạo để tránh duplicate.

4. **Reference Keys:**
   - Bảng ASN có các cột `PO_Id`, `RTR_Id`, `SRR_Id`, `ODR_Id`. Chỉ 1 trong các cột này có giá trị (tùy theo loại ASN), các cột còn lại NULL.

---

## 📊 BẢNG TÓM TẮT MAPPING HEADER

| Field ASN | Nguồn PO | Nguồn RTR | Nguồn SRR | Nguồn ODR |
| :--- | :--- | :--- | :--- | :--- |
| **ASN_Type** | 'Đơn hàng PO' | 'Nhập trả về kho' | 'Đổi hàng NCC' | 'Nhập chuyển kho' |
| **Ref Code** | PO Code | RTR Code | SRR Code | ODR Code |
| **Warehouse**| Kho nhận | Kho nhận | Kho nhận | Kho Nhập (Dest) |
| **Supplier** | Supplier PO | NULL | Supplier SRR | NULL |
| **Source WH**| NULL | NULL | NULL | Kho Xuất (Source) |
| **Date** | PO Date | RTR Date | SRR Date | ODR Date |

---
**Hết.**
