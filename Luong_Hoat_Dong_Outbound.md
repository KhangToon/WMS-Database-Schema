# 🔄 CHI TIẾT QUY TRÌNH: TỪ YÊU CẦU ĐẾN XUẤT KHO HOÀN TẤT (SOURCE → ODR → GI → ACTUAL SHIP)

Dưới đây là mô tả từng bước của quy trình xuất kho (Outbound), sử dụng ví dụ xuyên suốt về việc xuất mặt hàng **Áo thun** cho phòng ban Sales (Nội bộ).

**Ví dụ:**
- **Nhu cầu:** Phòng Sales xin lãnh 10 cái Áo thun (Mã: SHIRT-M) để làm mẫu tặng khách hàng.
- **Chứng từ nguồn:** ISR-2026-001 (Internal Stock Request).

---

### Giai đoạn 1: Tiếp nhận yêu cầu & Tạo ODR (Request Processing)
> **Trách nhiệm:** Hệ thống (Tự động) hoặc Nhân viên tạo thủ công.

1. **Source Document được Duyệt (Approved)**
   - **User:** Trưởng phòng Sales duyệt phiếu xin lãnh ISR-2026-001.
   - **Hệ thống:** Trigger sự kiện "ISR Approved" -> Tự động tạo ODR.

2. **Tạo ODR Header (Tự động)**
   - **Bảng:** `ODR`
   - **Trạng thái:** `NEW` (Mới tạo)
   - **Dữ liệu:**
     - Code: ODR-2601-001
     - Source Type: 'ISR'
     - Tham chiếu: ISR-2026-001
     - Kho xuất: WAREHOUSE_HCM
     - Người nhận: Sales Dept

3. **Tạo ODR Item (Tự động)**
   - **Bảng:** `ODR_Item`
   - **Dữ liệu:**
     - Item: SHIRT-M
     - Requested Qty: 10
     - Allocated Qty: 0
     - Shipped Qty: 0
     - Condition: 'Mới' (NEW)

   **Example DB Record:**
   ```sql
   INSERT INTO ODR (ODR_Code, Source_Type, Reference_Code, ODR_Status)
   VALUES ('ODR-2601-001', 'ISR', 'ISR-2026-001', 'NEW');

   INSERT INTO ODR_Item (ODR_Id, Item_Code, Requested_Qty, Condition_Type)
   VALUES ('ODR-2601-001', 'SHIRT-M', 10, 'NEW');
   ```

---

### Giai đoạn 2: Phân bổ tồn kho (Inventory Allocation)
> **Trách nhiệm:** Nhân viên Kho (Admin/Thủ kho)
> **Mục tiêu:** Giữ chỗ hàng hóa (Soft Allocation) để đảm bảo có hàng cho phiếu này.

4. **Kiểm tra tồn kho khả dụng**
   - **User:** Mở ODR-2601-001, nhấn **"Phân bổ"**.
   - **Hệ thống:** Check bảng `Inventory` (Tổng hợp).
     - *Hiện tại:* SHIRT-M (Mới) có `Available_Qty` = 80.
     - *Yêu cầu:* 10. -> **Đủ hàng.**

5. **Thực hiện Phân bổ (Giữ chỗ)**
   - **User:** Xác nhận phân bổ 10 cái.
   - **Hệ thống:**
     - Tạo 1 dòng trong `ODR_Allocation`: Map ODR với Inventory ID.
     - Cập nhật `Inventory`:
       - `Available_Qty`: 80 - 10 = **70** (Giảm khả dụng).
       - `Allocated_Qty`: 0 + 10 = **10** (Tăng giữ chỗ).
     - Cập nhật `ODR_Item`: `Allocated_Qty` = 10.
     - Cập nhật `ODR Status`: `ALLOCATED`.

   **Example DB Record:**
   ```sql
   -- 1. Create Allocation Record
   INSERT INTO ODR_Allocation (ODR_Id, ODR_Item_Id, Inventory_Id, Allocated_Qty)
   VALUES ('ODR-2601-001', 'ITEM-01', 'INV-001', 10);

   -- 2. Update Inventory (Soft Allocation)
   UPDATE Inventory 
   SET Available_Qty = Available_Qty - 10, 
       Allocated_Qty = Allocated_Qty + 10
   WHERE Inventory_Id = 'INV-001'; -- (SHIRT-M, NEW)

   -- 3. Update Status
   UPDATE ODR SET ODR_Status = 'ALLOCATED' WHERE ODR_Code = 'ODR-2601-001';
   ```

### Giai đoạn 2.1: Hoàn nguyên Phân bổ (Allocation Reversal - Optional)
> **Trường hợp:** Người dùng muốn hủy giữ chỗ để dùng hàng cho đơn khác hoặc phân bổ sai.
> **Điều kiện:** Hàng chưa được xuất kho (Chưa tạo GI hoặc GI đang hủy).

*   **User:** Chọn chức năng **"Hoàn nguyên"** trên ODR Item.
*   **Hệ thống:**
    *   Release tồn kho: Move từ `Allocated_Qty` -> `Available_Qty`.
    *   Reset `ODR_Item`: `Allocated_Quantity` về 0 (hoặc giảm tương ứng).
    *   Xóa/Update bản ghi `ODR_Allocation`.
    *   Cập nhật trạng thái ODR về `PENDING_ALLOCATION` (nếu hoàn nguyên hết).

   **Example DB Record:**
   ```sql
   -- 1. Reserve Stock Logic (Reversal)
   UPDATE Inventory 
   SET Available_Qty = Available_Qty + 10,  -- Trả lại khả dụng
       Allocated_Qty = Allocated_Qty - 10   -- Giảm giữ chỗ
   WHERE Inventory_Id = 'INV-001';

   -- 2. Update ODR Item
   UPDATE ODR_Item SET Allocated_Quantity = 0 WHERE ODR_Item_Id = 'ITEM-01';

   -- 3. Log Reversal
   INSERT INTO Outbound_Log (ODR_Id, Action, Description) 
   VALUES ('ODR-2601-001', 'UNALLOCATE', 'Hoàn nguyên 10 cái SHIRT-M');
   ```

---

### Giai đoạn 3: Tạo Phiếu xuất & Soạn hàng (Picking Process)
> **Trách nhiệm:** Nhân viên kho (Soạn hàng)

6. **Tạo Phiếu xuất kho (GI)**
   - **User:** Từ ODR đã phân bổ, nhấn **"Tạo phiếu xuất"**.
   - **Hệ thống:**
     - Tạo `GI` (Status: `DRAFT`).
     - Tạo `GI_Item` (Copy SL phân bổ sang SL yêu cầu xuất): `Requested_Qty` = 10.

7. **Thực hiện Soạn hàng (Picking - Hard Allocation)**
   - **User:** Cầm thiết bị đi lấy hàng. Hệ thống gợi ý Bin **RACK-A-01**.
   - **Thao tác:**
     - Đến RACK-A-01, lấy 10 cái.
     - Scan barcode vị trí và sản phẩm để xác nhận.
   - **Hệ thống:**
     - Tạo `GI_Picking_Location`: Ghi nhận lấy 10 cái từ bin RACK-A-01.
     - Cập nhật `GI_Item`: `Picked_Quantity` = 10.
     - **QUAN TRỌNG (Di chuyển sang STAGING):**
       - Trừ tồn kho tại kệ: `Inventory_Bin` (RACK-A-01) giảm 10.
       - Tăng tồn kho tại khu vực đóng gói: `Inventory_Bin` (STAGING_AREA) tăng 10.
       - Tạo `Inventory_Transaction` (Type: PICK): Move 10 từ RACK-A-01 sang STAGING_AREA.
       - `Inventory` (Tổng hợp): **Không đổi** số lượng tổng (Vì hàng vẫn nằm trong kho, chỉ đổi vị trí).

   **Example DB Record:**
   ```sql
   -- 1. Picking Record
   INSERT INTO GI_Picking_Location (GI_Item_Id, Bin_Code, Picked_Qty)
   VALUES ('GI-ITEM-01', 'RACK-A-01', 10);

   -- 2. Transaction (Move to Staging)
   INSERT INTO Inventory_Transaction (Type, Item_Code, Qty, From_Bin, To_Bin, Action)
   VALUES ('PICK', 'SHIRT-M', 10, 'RACK-A-01', 'STAGING_AREA', 'Move');

   -- 3. Update Bin Details
   UPDATE Inventory_Bin SET Quantity = Quantity - 10 WHERE Bin_Code = 'RACK-A-01';
   
   -- Upsert Staging Bin
   IF EXISTS (SELECT * FROM Inventory_Bin WHERE Bin_Code = 'STAGING_AREA')
       UPDATE Inventory_Bin SET Quantity = Quantity + 10 WHERE Bin_Code = 'STAGING_AREA';
   ELSE
       INSERT INTO Inventory_Bin (Bin_Code, Item_Code, Quantity) VALUES ('STAGING_AREA', 'SHIRT-M', 10);
   ```

---

### Giai đoạn 4: Xác nhận xuất kho & Hoàn tất (Confirm & Ship)
> **Trách nhiệm:** Thủ kho / Kế toán kho

8. **Xác nhận Xuất kho (Confirm GI)**
   - **User:** Kiểm tra hàng tại khu vực xuất (STAGING), bàn giao cho Sales. Nhấn **"Xác nhận xuất kho"**.
   - **Hệ thống:**
     - **Trừ kho chính thức:**
       - Trừ tồn kho tại `Inventory_Bin` (STAGING_AREA): Giảm 10.
       - Tạo `Inventory_Transaction` (Type: SHIP): -10 tại STAGING_AREA.
       - Cập nhật `Inventory` (Tổng hợp): Giảm `Allocated_Qty` đi 10, Giảm `Total_Qty` đi 10. (Hàng chính thức rời kho).
     - Chuyển `GI Status` -> `COMPLETED`.
     - Cập nhật ngày `Delivery_Date`.
     - Update ngược lại `ODR_Item`: `Shipped_Quantity` += 10.
     - Update `ODR Status`: Nếu Shipped = Requested -> `COMPLETED`.

9. **Ghi Log & Đồng bộ Source**
   - **Hệ thống:**
     - Ghi `Outbound_Log`.
     - Gửi thông báo cho ISR (Source Doc) là đã xuất xong -> ISR chuyển trạng thái `COMPLETED`.

   **Example DB Record:**
   ```sql
   -- 1. Transaction (Ship out)
   INSERT INTO Inventory_Transaction (Type, Item_Code, Qty, From_Bin, Action)
   VALUES ('SHIP', 'SHIRT-M', 10, 'STAGING_AREA', 'Decrease');

   -- 2. Update Inventory Summary (Final Reduction)
   UPDATE Inventory 
   SET Allocated_Qty = Allocated_Qty - 10, -- Release giữ chỗ
       Total_Qty = Total_Qty - 10          -- Giảm tổng tồn
   WHERE Inventory_Id = 'INV-001';

   -- 3. Remove from Staging
   UPDATE Inventory_Bin SET Quantity = Quantity - 10 WHERE Bin_Code = 'STAGING_AREA';

   -- 4. Close GI & ODR
   UPDATE GI SET GI_Status = 'COMPLETED', Delivery_Date = GETDATE() WHERE GI_Code = 'GI-2601-001';
   UPDATE ODR_Item SET Shipped_Quantity = 10 WHERE ODR_Id = 'ODR-2601-001';
   UPDATE ODR SET ODR_Status = 'COMPLETED' WHERE ODR_Code = 'ODR-2601-001';
   ```

---

### Tóm tắt dòng chảy tồn kho (Inventory Flow Summary)

Với ví dụ 10 cái áo xuất cho Sales:

1.  **Lúc Phân bổ (Bước 5):**
    - Kho tổng (`Inventory`): `Available` giảm 10 (80->70), `Allocated` tăng 10 (0->10). Tổng tồn vẫn 80.
    - Kho chi tiết (`Inventory_Bin`): Không đổi (Vẫn nằm trên kệ).

2.  **Lúc Soạn hàng (Bước 7 - Picking):**
    - Kho chi tiết (`Inventory_Bin`): Bin RACK-A-01 giảm 10. Bin STAGING tăng 10.
    - Kho tổng (`Inventory`): **Không đổi**. Vẫn giữ `Allocated` = 10 để đảm bảo không ai khác lấy mất hàng trong lúc đang đóng gói.

3.  **Lúc Xác nhận (Bước 8 - Shipping):**
    - Kho chi tiết (`Inventory_Bin`): Bin STAGING giảm 10 (Về 0).
    - Kho tổng (`Inventory`): `Allocated` giảm 10 (10->0). `Total` giảm 10 (80->70). Hàng chính thức biến mất khỏi hệ thống.
