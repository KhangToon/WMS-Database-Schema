# 🔄 CHI TIẾT QUY TRÌNH: TỪ ASN ĐẾN NHẬP KHO HOÀN TẤT (ASN → GR → INVENTORY)

Dưới đây là mô tả từng bước của quy trình nhập kho sử dụng ví dụ xuyên suốt về việc nhập mặt hàng **Áo thun** từ nhà cung cấp May 10 theo đơn mua hàng PO-1001.

**Ví dụ:**
- **PO:** PO-1001 (Mua của NCC May 10)
- **Hàng hóa:** 100 cái Áo thun (Mã: SHIRT-M)

---

### Giai đoạn 1: Từ Source (PO/RTR/SRR/ODR) đến ASN (System)
> **Trách nhiệm:** Hệ thống (Tự động) & Nhân viên Mua hàng/Quản lý kho

1. **Source Document được Duyệt (Approved)**
   - **User:** Trưởng phòng Mua hàng duyệt PO-1001.
   - **Hệ thống:** Trigger sự kiện "PO Approved".

2. **Tạo ASN Header (Tự động)**
   - **Bảng:** `ASN`
   - **Status:** 'Chờ xử lý' (Open)
   - **Dữ liệu:**
     - ASN Type: 'Đơn hàng PO'
     - Tham chiếu: PO-1001
     - Ngày dự kiến: 12/01/2026

3. **Tạo ASN Item (Tự động)**
   - **Bảng:** `ASN_Item`
   - **Dữ liệu:**
     - Item: SHIRT-M
     - Expected Quantity: 100
     - Actual Received: 0
     - Actual Received: 0
     - Status: 'Mới'

   **Example DB Record:**
   ```sql
   INSERT INTO ASN (ASN_Code, ASN_Type, PO_Id, ASN_Status) 
   VALUES ('ASN-2601-001', 'PO', 'PO-1001', 'NEW');
   
   INSERT INTO ASN_Item (ASN_Id, Item_Code, Expected_Qty)
   VALUES ('ASN-2601-001', 'SHIRT-M', 100);
   ```

---

### Giai đoạn 2: Tiếp nhận hàng & Tạo GR (Wait for Goods)
> **Trách nhiệm:** Nhân viên Kho (Nhận hàng tại cửa kho)

4. **Xe hàng đến & Tạo GR**
   - **User:** Xe tải May 10 đến giao hàng. Nhân viên kho mở ASN của PO-1001, nhấn **"Tạo phiếu nhập"**.
   - **Hệ thống:** Đọc dữ liệu từ ASN, tạo bản ghi GR mới.

5. **Tạo GR Header & GR Item Detail (Draft)**
   - **Bảng:** `GR` (Status: 'Nhập'), `GR_Item`
   - **Dữ liệu:**
     - GR Code: GR-2601-001
     - Item: SHIRT-M | SL Dự kiến: 100 | SL Thực nhận: 0 (chờ nhập)

   **Example DB Record:**
   ```sql
   INSERT INTO GR (GR_Code, GR_Status, ASN_Id) 
   VALUES ('GR-2601-001', 'DRAFT', 'ASN-2601-001');

   INSERT INTO GR_Item (GR_Id, Item_Code, Expected_Qty, Received_Qty)
   VALUES ('GR-2601-001', 'SHIRT-M', 100, 0);
   ```

6. **Kiểm đếm sơ bộ & Nhập số lượng thực tế**
   - **User:** Đếm thùng hàng.
     - *Thực tế:* Thấy có 1 lỗi thùng rách, nhưng tổng vẫn đủ 100 cái.
     - *Nhập liệu:* Vào chi tiết dòng SHIRT-M, nhập **Received Quantity = 100**.
   - **Note:** Có thể nhập SL ít nhơn (giao thiếu) hoặc nhiều hơn (giao thừa - cần quy trình duyệt riêng).

7. **Submit GR (Lưu & Chuyển xử lý)**
   - **User:** Nhấn **"Lưu & Xử lý"**.
   - **Hệ thống:**
     - Chuyển `GR Status` → 'Chờ xử lý'.
     - Tạo 1 bản ghi trong `GR_Quality_Check` cho dòng SHIRT-M (Total: 100).
     - **QUAN TRỌNG:**
       - Tạo `Inventory_Transaction` (Type: GR_RECEIVE) → +100 tại Receiving Area.
       - Cập nhật `Inventory` (Tổng hợp): Tăng 100 SL với Status **'Chờ kiểm nhập'**.

   **Example DB Record:**
   ```sql
   -- 1. Update GR Status
   UPDATE GR SET GR_Status = 'SUBMITTED' WHERE GR_Code = 'GR-2601-001';
   
   -- 2. Create QC Record
   INSERT INTO GR_Quality_Check (GR_Id, Item_Code, Total_Received_Qty)
   VALUES ('GR-2601-001', 'SHIRT-M', 100);
   
   -- 3. Create Transaction (Received)
   INSERT INTO Inventory_Transaction (Type, Item_Code, Qty, To_Bin, Status)
   VALUES ('GR_RECEIVE', 'SHIRT-M', 100, 'RECEIVING_AREA', 'QC_PENDING');

   -- 4. Update Inventory Summary
   UPDATE Inventory SET Available_Qty = Available_Qty + 100 
   WHERE Item_Code = 'SHIRT-M' AND Status = 'QC_PENDING';
   ```

---

### Giai đoạn 3: Kiểm tra chất lượng (QC) & Phân bổ (Warehouse Processing)
> **Trách nhiệm:** Nhân viên QC / Thủ kho

8. **Tiếp nhận yêu cầu QC**
   - **User QC:** Thấy GR-2601-001 đang 'Chờ xử lý'. Mở màn hình QC.

9. **Thực hiện kiểm tra & Phân loại (3 Nhánh)**
   - *Tình huống:* Trong 100 cái áo:
     - 80 cái tốt → Cần cất lên kệ.
     - 15 cái cần giao ngay cho đơn hàng gấp (SO-500).
     - 5 cái bị rách → Trả lại nhà cung cấp.

   **Người dùng thao tác phân bổ 100 cái này như sau:**

   **a. Nhánh 1: Lưu kho (Putaway) - 80 cái**
   - **User:** Chọn "Lưu kho" → Hệ thống gợi ý (hoặc User chọn) Bin: **RACK-A-01**.
   - **Bảng:** `GR_Storage_Bin_Allocation` (Bin: RACK-A-01, Qty: 80).
     - Transaction: Tạo `PUTAWAY` transaction.
       - Giảm 80 ở Receiving Area (Status: Chờ kiểm nhập).
       - Tăng 80 ở RACK-A-01 (Status: Mới).
     - Cập nhật `Inventory`: Giảm 80 **'Chờ kiểm nhập'**, Tăng 80 **'Mới'**.
     - Cập nhật `Inventory_Bin`: +80 tại bin RACK-A-01, update `Last_Receipt_Date`.

     **Example DB Record:**
     ```sql
     -- 1. Create Allocation
     INSERT INTO GR_Storage_Bin_Allocation (QC_Id, Bin_Code, Qty)
     VALUES ('QC-001', 'RACK-A-01', 80);
     
     -- 2. Create Transaction (Move)
     INSERT INTO Inventory_Transaction (Type, Item_Code, Qty, From_Bin, To_Bin, Status)
     VALUES ('PUTAWAY', 'SHIRT-M', 80, 'RECEIVING_AREA', 'RACK-A-01', 'NEW');

     -- 3. Update Inventory Summary (Move Stock)
     UPDATE Inventory SET Available_Qty = Available_Qty - 80 WHERE Status = 'QC_PENDING';
     UPDATE Inventory SET Available_Qty = Available_Qty + 80 WHERE Status = 'NEW';

     -- 4. Update Bin Detail
     UPDATE Inventory_Bin SET Available_Qty = Available_Qty + 80, Last_Receipt_Date = GETDATE()
     WHERE Bin_Code = 'RACK-A-01';
     ```

   **b. Nhánh 2: Xuất ngay (Cross-docking) - 15 cái**
   - **User:** Chọn "Xuất ngay" (áp dụng cho PO) → Chọn YC Lãnh kho (Warehouse Request): **REQ-SO-500**.
   - **Bảng:** `GR_Immediate_Export_Allocation` (Request: REQ-SO-500, Qty: 15).
     - Transaction: Tạo `CROSS_DOCK` transaction.
       - Giảm 15 ở Receiving Area.
       - Tăng 15 ở khu vực Outbound Dock (Status: Đã xuất).
     - Cập nhật `Inventory`: Giảm 15 **'Chờ kiểm nhập'**.

     **Example DB Record:**
     ```sql
     -- 1. Create Export Allocation
     INSERT INTO GR_Immediate_Export_Allocation (QC_Id, Req_Code, Qty)
     VALUES ('QC-001', 'REQ-SO-500', 15);
     
     -- 2. Create Transaction (Cross-dock)
     INSERT INTO Inventory_Transaction (Type, Item_Code, Qty, From_Bin, To_Bin)
     VALUES ('CROSS_DOCK', 'SHIRT-M', 15, 'RECEIVING_AREA', 'OUTBOUND_DOCK');

     -- 3. Update Inventory (Reduction)
     UPDATE Inventory SET Available_Qty = Available_Qty - 15 WHERE Status = 'QC_PENDING';
     ```

   **c. Nhánh 3: Hủy/Trả lại (Reject) - 5 cái**
   - **User:** Nhập số lượng Hủy/Trả: 5 cái. Ghi chú: "Rách vai áo".
   - **Bảng:** Update `GR_Quality_Check` (Canceled Qty: 5).
     - Transaction: Tạo `GR_RETURN` transaction.
       - Giảm 5 ở Receiving Area (Status: Lỗi).
     - Cập nhật `Inventory`: Giảm 5 **'Chờ kiểm nhập'**.

     **Example DB Record:**
     ```sql
     -- 1. Update QC Rejected Qty
     UPDATE GR_Quality_Check SET Canceled_Qty = 5 WHERE QC_Id = 'QC-001';

     -- 2. Create Transaction (Return)
     INSERT INTO Inventory_Transaction (Type, Item_Code, Qty, From_Bin)
     VALUES ('GR_RETURN', 'SHIRT-M', 5, 'RECEIVING_AREA');

     -- 3. Update Inventory (Reduction)
     UPDATE Inventory SET Available_Qty = Available_Qty - 5 WHERE Status = 'QC_PENDING';
     ```

10. **Hoàn tất xử lý dòng hàng**
    - **Hệ thống:** Kiểm tra 80 + 15 + 5 = 100 (Đủ SL thực nhận).
    - **Kết quả:** Đánh dấu dòng SHIRT-M trong GR này đã xử lý xong.

---

### Giai đoạn 4: Hoàn tất & Đồng bộ ngược (Finalize)
> **Trách nhiệm:** Hệ thống (Tự động)

11. **Kiểm tra hoàn thành GR**
    - **Hệ thống:** Quét thấy tất cả các item trong GR-2601-001 đều đã được phân bổ hết.
    - **Hành động:** Chuyển `GR Status` → **'Hoàn thành'**.

12. **Ghi Log & Audit**
    - **Bảng:** `GR_Log` ghi nhận "Hoàn tất xử lý GR-2601-001 bởi User A lúc 10:30".

13. **Đồng bộ ngược về ASN**
    - **Hệ thống:**
      - Update `ASN_Item` (SHIRT-M): `Actual_Received_Quantity` += 100.
      - Tính lại %: 100/100 = 100%.

14. **Đóng ASN**
    - **Hệ thống:** Kiểm tra ASN của PO-1001.
      - Item SHIRT-M đã đủ.
      - Nếu PO còn item khác chưa đủ → ASN vẫn 'Đang xử lý'.
      - Nếu tất cả đã đủ → `ASN Status` → **'Hoàn thành'** (Closed).

---

### Tóm tắt luồng dữ liệu Kho (Inventory Flow Summary)
Với ví dụ 100 cái áo trên, dòng chảy tồn kho diễn ra như sau:

1.  **Lúc xe đến (Bước 7):** +100 ở `Receiving Area` (Kho tạm).
2.  **Lúc xử lý (Bước 9):**
    -   -80 ở `Receiving` → +80 ở `RACK-A-01` (Hàng dùng được).
    -   -15 ở `Receiving` → +15 ở `Outbound` (Hàng đi ngay).
    -   -5 ở `Receiving` → Biến mất khỏi kho (Trả về NCC).
3.  **Kết quả cuối:** Tồn kho thực tế tăng lên đúng 80 cái (tại kệ) và ghi nhận lịch sử cho 15 cái xuất đi và 5 cái trả lại.
