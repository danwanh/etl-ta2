# Plan nạp dữ liệu vào Retail_DDS (không staging)

Nguồn: `RetailPlus_Legacy` (ERP_Sales, CustomerMaster, ProductMaster, CustomerMap, ProductMap), `POS.csv`, `Online.xlsx` (sheet OnlineOrders).
Đích: `Retail_DDS` (DimDate, DimCustomer, DimProduct, FactSales, ETL_RejectedRows). Package: `Package.dtsx`.

> **Cập nhật:** ETL đã tách thành `Master` + `Load_DimDate`, `Load_DimCustomer`, `Load_DimProduct`, `Load_FactSales_ERP/POS/Online`.
> Dim: incremental SCD1 (insert mới + update khi đổi). Fact: incremental theo TransactionNo (bỏ qua giao dịch cùng nguồn đã nạp, xoá/ghi lại reject theo nguồn).
> `Master` có tham số `FullReload` (0/1): 1 = TRUNCATE FactSales + ETL_RejectedRows trước khi nạp. Phần "Control flow" bên dưới là thiết kế cũ (một package).

## Control flow (thiết kế cũ, một package)
1. **SQL - Reset Fact va Reject, nap DimDate**: TRUNCATE FactSales + ETL_RejectedRows (full reload, chạy lại được), sinh DimDate 2020-01-01 → 2030-12-31.
2. **DFT - Load DimCustomer**: CustomerMaster → Lookup DimCustomer → chỉ insert mã chưa có.
3. **DFT - Load DimProduct**: tương tự với ProductMaster.
4. **DFT - Fact ERP** → 5. **DFT - Fact POS** → 6. **DFT - Fact Online**.

## Làm sạch trong mỗi DFT Fact (dòng lỗi → ETL_RejectedRows kèm lý do)
| Bước | Áp dụng | Xử lý |
|---|---|---|
| Normalize | cả 3 | TRIM, ép DT_STR, gán SourceSystem |
| Status = Cancelled | ERP, Online | reject |
| Thiếu đơn giá | ERP, Online | reject |
| Parse Quantity `(DT_I4)` | cả 3 | lỗi ép kiểu ("bad", "abc") → reject; Quantity <= 0 → reject |
| Parse Price/Discount | POS | bỏ dấu phẩy "100,000"; lỗi → reject |
| Parse Date | cả 3 | POS dd/MM/yyyy → date; ngày không hợp lệ (32/10) → reject; DateKey = yyyyMMdd |
| Thiếu mã KH / SP | cả 3 | reject (POS T004 rỗng CustID) |
| Map mã web | Online | Lookup CustomerMap / ProductMap (WEB-Cxxx → Cxxx); không map được → reject |
| Lookup DimCustomer / DimProduct | cả 3 | không có trong dim (C999, P999) → reject |
| Tính tiền | cả 3 | Gross = Qty × Price; Discount: ERP/POS là số tiền, Online = Gross × % / 100; Net = Gross − Discount; Net < 0 → reject |
| Chống trùng TransactionNo | cả 3 | Lookup FactSales; đã có → reject. Thứ tự ưu tiên **ERP > POS > Online** |

Vì TransactionNo trùng giữa các nguồn (POS∩Online T011–T016, Online∩ERP T021–T026) và FactSales có UNIQUE(TransactionNo),
bản đúng đầu tiên theo thứ tự ưu tiên được giữ; bản lỗi của nguồn ưu tiên cao không chặn bản đúng của nguồn thấp hơn (vd. POS T011 lỗi P999 → lấy Online T011).

## Kết quả kỳ vọng
FactSales 29 dòng (ERP 13, POS 12, Online 4), ETL_RejectedRows 19 dòng (ERP 3, POS 4, Online 12). Tổng 29 + 19 = 48 = 16 + 16 + 16 dòng nguồn.

## Lưu ý
- Kết nối đang trỏ `localhost` (Windows auth) và `C:\Users\LE HO DAN ANH\Downloads\Data\...` — đổi trong Connection Managers nếu máy khác.
- Đóng `POS.csv` trong Excel trước khi chạy (Excel khoá file).
