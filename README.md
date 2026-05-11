# pawnshop-management-sqlserver
Pawnshop management database using SQL Server and T-SQL
# 🏦 CSDL Quản Lý Cầm Đồ — Bài Tập số 03

> **Môn học:** Hệ quản trị CSDL | **Lớp:** 59KMT | **SV** Hoàng Trường Phúc

---

## 📋 Mục lục



---

## 1. Mô tả bài toán & Giả định hệ thống

Hệ thống quản lý **cửa hàng cầm đồ**, bao gồm:

- Tiếp nhận hợp đồng vay tiền thế chấp tài sản
- Tính lãi linh hoạt (lãi đơn → lãi phạt tuyến tính trên gốc mới sau deadline)
- Quản lý trạng thái tài sản thế chấp
- Xử lý thanh lý đồ khi khách quá hạn
- Ghi lại toàn bộ lịch sử giao dịch và lịch sử thay đổi trạng thái (audit log)

### Giả định hệ thống

```
- Một tài sản chỉ thuộc duy nhất một hợp đồng tại một thời điểm.
- Không xử lý tranh chấp đồng thời (concurrent update) nhiều nhân viên cùng sửa hợp đồng.
- Lãi suất cố định: 0.5%/ngày (5.000đ / 1.000.000đ / ngày), không thay đổi theo hợp đồng.
- Hệ thống dùng đơn vị tiền VNĐ, không có phần thập phân.
- Deadline1 < Deadline2 luôn luôn (ràng buộc CHECK).
- Dữ liệu không bao giờ bị xóa vật lý (soft delete nếu cần).
```

---

## 2. Thiết kế ERD

### Sơ đồ quan hệ (6 bảng sau khi nâng cấp)

```
┌─────────────┐        ┌──────────────────────┐        ┌──────────────┐
│  KhachHang  │ 1    * │       HopDong        │ 1    * │   TaiSan     │
│─────────────│────────│──────────────────────│────────│──────────────│
│ KhachHangID │        │ HopDongID       (PK) │        │ TaiSanID(PK) │
│ HoTen       │        │ KhachHangID     (FK) │        │ HopDongID(FK)│
│ SoDienThoai │        │ NgayVay              │        │ TenTaiSan    │
│ CCCD        │        │ SoTienVayGoc         │        │ MoTa         │
│ DiaChi      │        │ Deadline1            │        │ GiaTriDinhGia│
│ NgayTao     │        │ Deadline2            │        │ TrangThai    │
└─────────────┘        │ TrangThai            │        └──────────────┘
                       └──────────┬───────────┘
                                  │ 1
                        ┌─────────┴──────────┐
                        │                    │
                        │ *                  │ *
           ┌────────────┴───────┐  ┌─────────┴──────────────┐
           │  LichSuGiaoDich    │  │  LichSuTrangThai        │
           │────────────────────│  │─────────────────────────│
           │ GiaoDichID    (PK) │  │ LogID             (PK)  │
           │ HopDongID     (FK) │  │ HopDongID         (FK)  │
           │ NgayGiaoDich       │  │ TrangThaiCu              │
           │ SoTienTra          │  │ TrangThaiMoi             │
           │ DuNoConLai         │  │ ThoiGianThayDoi          │
           │ NhanVienID    (FK)─┼──┼─NhanVienID        (FK)  │
           │ LoaiGiaoDich       │  │ GhiChu                   │
           └────────────────────┘  └─────────────────────────┘
                        │
                        └───────── FK trỏ về ──────────┐
                                                        ▼
                                              ┌──────────────────┐
                                              │    NhanVien      │
                                              │──────────────────│
                                              │ NhanVienID  (PK) │
                                              │ HoTen            │
                                              │ TaiKhoan         │
                                              │ NgayTao          │
                                              └──────────────────┘
```



### Mô tả quan hệ

| Quan hệ | Loại | Diễn giải |
|---------|------|-----------|
| KhachHang → HopDong | 1 - Nhiều | 1 khách hàng có thể có nhiều hợp đồng cầm cố |
| HopDong → TaiSan | 1 - Nhiều | 1 hợp đồng có thể gồm nhiều tài sản thế chấp |
| HopDong → LichSuGiaoDich | 1 - Nhiều | 1 hợp đồng có nhiều lần giao dịch tiền |
| HopDong → LichSuTrangThai | 1 - Nhiều | 1 hợp đồng có nhiều lần đổi trạng thái |
| NhanVien → LichSuGiaoDich | 1 - Nhiều | 1 nhân viên thu nhiều giao dịch |
| NhanVien → LichSuTrangThai | 1 - Nhiều | 1 nhân viên thực hiện nhiều thay đổi trạng thái |

---

## 3. Chi tiết các bảng (Schema)

###  Bảng `KhachHang`

| Cột | Kiểu | Ràng buộc | Mô tả |
|-----|------|-----------|-------|
| `KhachHangID` | INT | PK, IDENTITY(1,1) | Khóa chính tự tăng |
| `HoTen` | NVARCHAR(100) | NOT NULL | Họ và tên đầy đủ |
| `SoDienThoai` | VARCHAR(15) | NOT NULL, UNIQUE | Số điện thoại liên hệ |
| `CCCD` | VARCHAR(20) | NOT NULL, UNIQUE | Căn cước công dân |
| `DiaChi` | NVARCHAR(200) | — | Địa chỉ thường trú |
| `CreatedAt` | DATETIME | DEFAULT GETDATE() | Ngày tạo hồ sơ |
| `UpdatedAt` | DATETIME | — | Lần cập nhật gần nhất |

> **Lý do UNIQUE cho CCCD và SoDienThoai:** Tránh tạo trùng hồ sơ khách hàng — lỗi phổ biến trong hệ thống thực tế.

---

###  Bảng `NhanVien` *(Mới — thay thế NguoiThuTien dạng text)*

| Cột | Kiểu | Ràng buộc | Mô tả |
|-----|------|-----------|-------|
| `NhanVienID` | INT | PK, IDENTITY(1,1) | Khóa chính |
| `HoTen` | NVARCHAR(100) | NOT NULL | Tên nhân viên |
| `TaiKhoan` | VARCHAR(50) | NOT NULL, UNIQUE | Tên đăng nhập hệ thống |
| `CreatedAt` | DATETIME | DEFAULT GETDATE() | Ngày tạo tài khoản |

> **Lý do thêm bảng này:** `NguoiThuTien NVARCHAR` dễ nhập sai tên, không thể JOIN, không thống kê doanh thu theo nhân viên được. Dùng FK là đúng chuẩn.

---

###  Bảng `HopDong`

| Cột | Kiểu | Ràng buộc | Mô tả |
|-----|------|-----------|-------|
| `HopDongID` | INT | PK, IDENTITY(1,1) | Khóa chính |
| `KhachHangID` | INT | FK → KhachHang | Khách hàng ký hợp đồng |
| `NgayVay` | DATE | NOT NULL, DEFAULT hôm nay | Ngày ký hợp đồng |
| `SoTienVayGoc` | DECIMAL(18,0) | NOT NULL, CHECK > 0 | Số tiền vay ban đầu |
| `Deadline1` | DATE | NOT NULL | Hạn cuối tính lãi đơn |
| `Deadline2` | DATE | NOT NULL, CHECK > Deadline1 | Hạn cuối trước khi thanh lý |
| `TrangThai` | NVARCHAR(50) | NOT NULL, CHECK IN (...) | Trạng thái hợp đồng |
| `GhiChu` | NVARCHAR(500) | — | Ghi chú thêm |
| `CreatedAt` | DATETIME | DEFAULT GETDATE() | Ngày tạo |
| `UpdatedAt` | DATETIME | — | Lần cập nhật gần nhất |

---

###  Bảng `TaiSan`

| Cột | Kiểu | Ràng buộc | Mô tả |
|-----|------|-----------|-------|
| `TaiSanID` | INT | PK, IDENTITY(1,1) | Khóa chính |
| `HopDongID` | INT | FK → HopDong | Thuộc hợp đồng nào |
| `TenTaiSan` | NVARCHAR(200) | NOT NULL | Tên tài sản (VD: iPhone 15) |
| `MoTa` | NVARCHAR(500) | — | Màu sắc, tình trạng... |
| `GiaTriDinhGia` | DECIMAL(18,0) | NOT NULL, CHECK > 0 | Giá trị định giá lúc cầm |
| `TrangThai` | NVARCHAR(50) | NOT NULL, CHECK IN (...) | Trạng thái tài sản |
| `CreatedAt` | DATETIME | DEFAULT GETDATE() | Ngày tạo |

**Vòng đời trạng thái tài sản:**

```
'Đang cầm cố'
    ├──→ 'Đã trả khách'       (khi khách thanh toán đủ)
    ├──→ 'Sẵn sàng thanh lý'  (khi quá Deadline2)
    └──→ 'Đã bán thanh lý'    (khi hợp đồng → 'Đã thanh lý tài sản')
```

---

###  Bảng `LichSuGiaoDich` (Audit Log tiền)

| Cột | Kiểu | Ràng buộc | Mô tả |
|-----|------|-----------|-------|
| `GiaoDichID` | INT | PK, IDENTITY(1,1) | Khóa chính |
| `HopDongID` | INT | FK → HopDong | Thuộc hợp đồng nào |
| `NgayGiaoDich` | DATETIME | DEFAULT GETDATE() | Thời điểm giao dịch |
| `SoTienTra` | DECIMAL(18,0) | NOT NULL, CHECK > 0 | Số tiền trả lần này |
| `DuNoConLai` | DECIMAL(18,0) | NOT NULL | Dư nợ gốc còn lại sau khi trả |
| `NhanVienID` | INT | FK → NhanVien, NULL ok | Nhân viên thu tiền |
| `LoaiGiaoDich` | NVARCHAR(50) | NOT NULL, CHECK IN (...) | Loại giao dịch |
| `GhiChu` | NVARCHAR(500) | — | Ghi chú |

> ⚠️ **Nguyên tắc:** Bảng này chỉ INSERT, **không bao giờ UPDATE hoặc DELETE**. Mọi dòng tiền phải có dấu vết.

---

### 📋 Bảng `LichSuTrangThai` *(Mới — audit thay đổi trạng thái)*

| Cột | Kiểu | Ràng buộc | Mô tả |
|-----|------|-----------|-------|
| `LogID` | INT | PK, IDENTITY(1,1) | Khóa chính |
| `HopDongID` | INT | FK → HopDong | Thuộc hợp đồng nào |
| `TrangThaiCu` | NVARCHAR(50) | NOT NULL | Trạng thái trước khi đổi |
| `TrangThaiMoi` | NVARCHAR(50) | NOT NULL | Trạng thái sau khi đổi |
| `ThoiGianThayDoi` | DATETIME | DEFAULT GETDATE() | Thời điểm thay đổi |
| `NhanVienID` | INT | FK → NhanVien, NULL ok | Ai thay đổi (NULL nếu do Trigger tự động) |
| `GhiChu` | NVARCHAR(500) | — | Lý do thay đổi |

> **Lý do cần bảng này:** Trigger tự động đổi trạng thái (Event 5) — nếu không có bảng này, không biết hợp đồng bị chuyển "Quá hạn" lúc nào, do ai hay do hệ thống.

---

## 4. CHECK Constraints đầy đủ

Phần này bảo vệ tính toàn vẹn dữ liệu **ở tầng database**, không phụ thuộc vào logic ứng dụng.

```sql
-- ===== Bảng HopDong =====
CONSTRAINT CHK_HopDong_SoTien    CHECK (SoTienVayGoc > 0),
CONSTRAINT CHK_HopDong_Deadline  CHECK (Deadline2 > Deadline1),
CONSTRAINT CHK_HopDong_TrangThai CHECK (TrangThai IN (
    N'Đang vay',
    N'Đang trả góp',
    N'Quá hạn (nợ xấu)',
    N'Đã thanh toán',
    N'Đã thanh lý tài sản'
)),

-- ===== Bảng TaiSan =====
CONSTRAINT CHK_TaiSan_GiaTri    CHECK (GiaTriDinhGia > 0),
CONSTRAINT CHK_TaiSan_TrangThai CHECK (TrangThai IN (
    N'Đang cầm cố',
    N'Đã trả khách',
    N'Sẵn sàng thanh lý',
    N'Đã bán thanh lý'
)),

-- ===== Bảng LichSuGiaoDich =====
CONSTRAINT CHK_GiaoDich_SoTien CHECK (SoTienTra > 0),
CONSTRAINT CHK_GiaoDich_LoaiGD CHECK (LoaiGiaoDich IN (
    N'Trả nợ',
    N'Gia hạn',
    N'Thanh toán đủ'
)),
```

> **Tại sao đặt tên constraint?** Khi vi phạm, SQL Server báo đúng tên (VD: `CHK_HopDong_Deadline`) thay vì thông báo chung chung — debug nhanh hơn nhiều.

---

## 5. Chiến lược FK — ON DELETE / ON UPDATE



| FK | ON DELETE | ON UPDATE | Lý do |
|----|-----------|-----------|-------|
| HopDong → KhachHang | `NO ACTION` | `NO ACTION` | Không cho xóa khách đã có hợp đồng — mất hồ sơ tài chính |
| TaiSan → HopDong | `NO ACTION` | `NO ACTION` | Không cho xóa hợp đồng còn tài sản thế chấp |
| LichSuGiaoDich → HopDong | `NO ACTION` | `NO ACTION` | Không xóa hợp đồng đã phát sinh giao dịch — vi phạm audit |
| LichSuTrangThai → HopDong | `NO ACTION` | `NO ACTION` | Log lịch sử không được mất |
| LichSuGiaoDich → NhanVien | `SET NULL` | `NO ACTION` | Nhân viên nghỉ việc: log vẫn giữ, chỉ NhanVienID = NULL |
| LichSuTrangThai → NhanVien | `SET NULL` | `NO ACTION` | Tương tự |

```sql
-- Ví dụ khai báo:
CONSTRAINT FK_HopDong_KhachHang
    FOREIGN KEY (KhachHangID) REFERENCES KhachHang(KhachHangID)
    ON DELETE NO ACTION
    ON UPDATE NO ACTION,

CONSTRAINT FK_GiaoDich_NhanVien
    FOREIGN KEY (NhanVienID) REFERENCES NhanVien(NhanVienID)
    ON DELETE SET NULL   -- Nhân viên nghỉ: log vẫn còn, NhanVienID = NULL
    ON UPDATE NO ACTION
```

---

## 6. Quy tắc nghiệp vụ & Vòng đời trạng thái

### Quy trình vòng đời hợp đồng

```
[Tạo hợp đồng]
      │
      ▼
 ┌─────────┐
 │Đang vay │ ← Lãi đơn: 0.5%/ngày trên gốc
 └────┬────┘
      │
      ├── Khách trả một phần ──→ [Đang trả góp] ──→ trả hết ──→ [Đã thanh toán] ✅
      │
      │ (vượt Deadline1, chưa trả đủ) ← Trigger tự động
      ▼
 ┌──────────────────┐
 │Quá hạn (nợ xấu) │ ← Lãi phạt: 0.5%/ngày trên (Gốc + Lãi đơn tích lũy)
 └────────┬─────────┘
          │
          ├── Khách đến trả đủ ──→ [Đã thanh toán] ✅
          │
          │ (vượt Deadline2) ← Trigger tự động
          ▼
   Tài sản → [Sẵn sàng thanh lý]
          │
          │ (bán xong)
          ▼
   [Đã thanh lý tài sản] — Tài sản → [Đã bán thanh lý]
```

### Quy tắc trả hàng

```
Điều kiện được lấy lại tài sản X:

  SUM(GiaTriDinhGia của tài sản 'Đang cầm cố' còn lại, SAU KHI bỏ X ra)
  >= DuNoHienTai (gốc + lãi tính đến hôm nay)
```

---

## 7. Thuật toán tính lãi

### ⚠️ Làm rõ thuật ngữ "lãi kép" trong bài này

> Bài toán dùng cụm "lãi kép" nhưng **không phải compound interest** theo công thức `A = P(1+r)^n`.
>
> Compound interest thật sự phải tính lũy thừa theo từng kỳ.
>
> **Bài này thực chất là:** lãi tuyến tính với gốc tính lãi được nâng lên sau Deadline1.
>
> **Cách diễn đạt chính xác hơn:**
> *Sau Deadline1, hệ thống áp dụng lãi phạt tuyến tính tính trên (Gốc vay + Tổng lãi đơn đã tích lũy). Đây là cơ chế lãi phạt, không phải lãi kép theo nghĩa tài chính.*

---

### Giai đoạn 1 — Lãi đơn (từ NgayVay đến Deadline1)

```
LaiDon/ngay = SoTienVayGoc × 0.005
LaiDon      = LaiDon/ngay × DATEDIFF(DAY, NgayVay, MIN(TargetDate, Deadline1))
```

**Ví dụ:** Vay 5.000.000đ, Deadline1 sau 30 ngày
```
LaiDon/ngay = 5.000.000 × 0.005 = 25.000đ
LaiDon      = 25.000 × 30       = 750.000đ
```

---

### Giai đoạn 2 — Lãi phạt (từ Deadline1 đến TargetDate, nếu quá hạn)

```
GocMoi  = SoTienVayGoc + LaiDon     ← gốc tính lãi được nâng lên
LaiPhat = GocMoi × 0.005 × DATEDIFF(DAY, Deadline1, TargetDate)
```

**Ví dụ tiếp:** Quá hạn thêm 10 ngày
```
GocMoi  = 5.000.000 + 750.000 = 5.750.000đ
LaiPhat = 5.750.000 × 0.005 × 10 = 287.500đ
```

---

### Tổng tiền phải trả tại ngày X

```
Nếu X <= Deadline1:
  Tổng = Gốc + LãiĐơn(NgayVay → X)

Nếu X > Deadline1:
  Tổng = Gốc
       + LãiĐơn(NgayVay → Deadline1)    ← lãi đơn đầy đủ
       + LãiPhạt(Deadline1 → X)         ← lãi phạt trên gốc mới
```



---

## 8. Tối ưu truy vấn bằng INDEX

Phần này để biết bảng sẽ bị query theo pattern nào, tạo index phù hợp.

### Index kết hợp cho truy vấn nợ xấu (Event 4)

```sql
-- Event 4 lọc: WHERE Deadline1 < GETDATE() AND TrangThai = N'Đang vay'
-- Index kết hợp giúp tránh Full Table Scan
CREATE INDEX IX_HopDong_Deadline1_TrangThai
ON HopDong(Deadline1, TrangThai);
```

### Index cho tra cứu trạng thái hợp đồng

```sql
-- Trigger Event 5 liên tục check TrangThai
CREATE INDEX IX_HopDong_TrangThai
ON HopDong(TrangThai);
```

### Index cho log giao dịch

```sql
-- Function fn_CalcMoneyContract JOIN LichSuGiaoDich theo HopDongID
-- DESC vì thường cần "giao dịch gần nhất"
CREATE INDEX IX_LichSuGiaoDich_HopDongID
ON LichSuGiaoDich(HopDongID, NgayGiaoDich DESC);
```

### Index cho lịch sử trạng thái

```sql
CREATE INDEX IX_LichSuTrangThai_HopDongID
ON LichSuTrangThai(HopDongID, ThoiGianThayDoi DESC);
```

### Tổng hợp

| Index | Phục vụ cho | Lý do |
|-------|-------------|-------|
| `IX_HopDong_Deadline1_TrangThai` | Event 4 | Lọc nợ xấu theo ngày + trạng thái |
| `IX_HopDong_TrangThai` | Event 5 (Trigger) | Check trạng thái liên tục |
| `IX_LichSuGiaoDich_HopDongID` | fn_CalcMoney | JOIN log trả tiền |
| `IX_LichSuTrangThai_HopDongID` | Audit query | Lịch sử đổi trạng thái |

---

## 9. Transaction Consistency

Khi thực hiện Event 3 (khách trả tiền), hệ thống cần **nhiều bước ghi liên tiếp**:

```
1. INSERT vào LichSuGiaoDich
2. UPDATE TrangThai của HopDong
3. Nếu trả đủ: UPDATE TrangThai từng TaiSan → 'Đã trả khách'
4. INSERT vào LichSuTrangThai
```

Nếu lỗi xảy ra ở bước 3 mà bước 1+2 đã chạy xong → **dữ liệu lệch, không thể rollback thủ công**.

### Giải pháp: bọc trong TRANSACTION

```sql
BEGIN TRY
    BEGIN TRANSACTION

        -- 1. Ghi log giao dịch tiền
        INSERT INTO LichSuGiaoDich (...) VALUES (...)

        -- 2. Cập nhật trạng thái hợp đồng
        UPDATE HopDong
        SET TrangThai = N'Đã thanh toán', UpdatedAt = GETDATE()
        WHERE HopDongID = @HopDongID

        -- 3. Trả tài sản (nếu đủ tiền)
        UPDATE TaiSan
        SET TrangThai = N'Đã trả khách'
        WHERE HopDongID = @HopDongID
          AND TrangThai = N'Đang cầm cố'

        -- 4. Ghi log thay đổi trạng thái
        INSERT INTO LichSuTrangThai (...) VALUES (...)

    COMMIT TRANSACTION
END TRY
BEGIN CATCH
    ROLLBACK TRANSACTION
    THROW   -- Ném lỗi lên cho caller xử lý
END CATCH
```

> **Nguyên tắc:** Mọi Stored Procedure có từ 2 bước ghi trở lên đều phải dùng `BEGIN TRANSACTION ... COMMIT / ROLLBACK`.

---

## 10. Giải thích thiết kế

### Tại sao tách `LichSuGiaoDich` ra bảng riêng?

| Cách sai ❌ | Cách đúng ✅ |
|------------|-------------|
| Ghi đè `DuNo` trong bảng HopDong | INSERT mỗi lần trả vào LichSuGiaoDich |
| Mất hoàn toàn dấu vết giao dịch | Truy vết được từng đồng, từng ngày |
| Không biết ai thu tiền | Ghi `NhanVienID` FK đúng chuẩn |

### Tại sao `NguoiThuTien` phải là FK, không phải text?

| `NguoiThuTien NVARCHAR` ❌ | `NhanVienID INT FK` ✅ |
|--------------------------|----------------------|
| Dễ nhập sai tên | Luôn đúng — chỉ chọn từ bảng NhanVien |
| Trùng tên: không phân biệt được | ID duy nhất cho mỗi người |
| Không thể thống kê theo nhân viên | `GROUP BY NhanVienID` dễ dàng |
| Nhân viên nghỉ việc: log bị mồ côi | `SET NULL` — log vẫn tồn tại |

### Tại sao `GiaTriDinhGia` lưu trong `TaiSan` chứ không trong `HopDong`?

Mỗi tài sản có giá trị khác nhau, cần tính riêng khi xét điều kiện trả hàng:

```sql
-- Kiểm tra: Tài sản còn lại có đủ đảm bảo dư nợ không?
SELECT SUM(GiaTriDinhGia)
FROM TaiSan
WHERE HopDongID = @HopDongID
  AND TrangThai = N'Đang cầm cố'
  AND TaiSanID <> @TaiSanMuonLayLai   -- Bỏ tài sản muốn lấy ra
```

### Tại sao dùng `DECIMAL(18,0)` cho tiền?

- `FLOAT` / `REAL` có sai số dấu phẩy động → **không bao giờ dùng cho tiền**
- `DECIMAL(18,0)` = số nguyên đến 18 chữ số → chính xác tuyệt đối
- Đơn vị: đồng VNĐ — không cần phần thập phân

---

## 11. Chuẩn hóa 3NF

### Kiểm tra từng bảng

**`KhachHang`**
- 1NF ✅: Mọi cột là giá trị nguyên tố, không có cột đa trị
- 2NF ✅: Khóa đơn (KhachHangID) → không thể có phụ thuộc bộ phận
- 3NF ✅: `DiaChi` phụ thuộc trực tiếp `KhachHangID`, không phụ thuộc bắc cầu qua `CCCD`

**`HopDong`**
- 1NF ✅
- 2NF ✅: Mọi cột phụ thuộc đầy đủ vào `HopDongID`
- 3NF ✅: `Deadline1`, `Deadline2` phụ thuộc trực tiếp `HopDongID`, không suy ra từ `KhachHangID`

**`TaiSan`**
- 1NF ✅
- 2NF ✅: `GiaTriDinhGia` phụ thuộc vào `TaiSanID`, **không** phụ thuộc vào `HopDongID`
- 3NF ✅

**`LichSuGiaoDich`**
- 1NF ✅
- 2NF ✅
- 3NF ✅: `DuNoConLai` là snapshot tại thời điểm giao dịch — không suy ra từ cột khác trong bảng

**`LichSuTrangThai`**
- 1NF ✅
- 2NF ✅
- 3NF ✅: `TrangThaiCu`, `TrangThaiMoi` phụ thuộc trực tiếp `LogID`

**`NhanVien`**
- 1NF ✅ | 2NF ✅ | 3NF ✅

---

## 12.  Chạy Script test cơ bản

### Yêu cầu
- SQL Server 2022 trở lên (hoặc Azure SQL)
- SQL Server Management Studio (SSMS) hoặc Azure Data Studio

### Thứ tự tạo bảng *(bắt buộc — sai thứ tự lỗi FK ngay)*

```
1. KhachHang          (không có FK)
2. NhanVien           (không có FK)
3. HopDong            (FK → KhachHang)
4. TaiSan             (FK → HopDong)
5. LichSuGiaoDich     (FK → HopDong, NhanVien)
6. LichSuTrangThai    (FK → HopDong, NhanVien)
```

### Các bước thực hiện

```sql
-- Bước 1: Mở SSMS, kết nối server, chạy file script.sql
-- File bao gồm: CREATE DATABASE → CREATE TABLE → INDEX → Sample Data

-- Bước 2: Kiểm tra dữ liệu mẫu
SELECT * FROM KhachHang;
SELECT * FROM NhanVien;
SELECT * FROM HopDong;
SELECT * FROM TaiSan;
SELECT * FROM LichSuGiaoDich;
SELECT * FROM LichSuTrangThai;

-- Bước 3: Test CHECK constraint hoạt động đúng
-- Thử insert Deadline2 < Deadline1 → phải báo lỗi CHK_HopDong_Deadline
INSERT INTO HopDong (KhachHangID, SoTienVayGoc, Deadline1, Deadline2, TrangThai)
VALUES (1, 5000000, '2025-06-01', '2025-05-01', N'Đang vay');  -- Phải FAIL ✅
```

> ⚠️ Nếu tạo sai thứ tự bảng sẽ lỗi `"FOREIGN KEY constraint"` — lỗi rất phổ biến khi mới học.

---
## Phần 2 — Event 1: Tạo Hợp Đồng

## Tổng quan

Stored Procedure này được xây dựng tuân theo các quy tắc sau:

- Sử dụng **Table-Valued Parameter (TVP)** để truyền danh sách tài sản
- Bọc toàn bộ xử lý trong **Transaction**
- Thực hiện logic **Upsert** khách hàng
- Kiểm tra các **ràng buộc nghiệp vụ (Validation)**

---

## Bước 1: Tạo User-Defined Table Type (TVP)

Để truyền danh sách nhiều tài sản vào một Stored Procedure duy nhất, cần định nghĩa một kiểu dữ liệu bảng trước.

```sql
-- Tạo Type cho danh sách tài sản
CREATE TYPE TaiSanTableType AS TABLE (
    TenTaiSan      NVARCHAR(200)  NOT NULL,
    MoTa           NVARCHAR(500),
    GiaTriDinhGia  DECIMAL(18,0)  NOT NULL
);
GO
```

---

## Bước 2: Viết Stored Procedure Tạo Hợp Đồng Mới

Procedure này thực hiện các nhiệm vụ sau:

1. Logic **Upsert** khách hàng
2. Kiểm tra **giá trị định giá tài sản** so với số tiền vay
3. **Ghi nhận lịch sử** trạng thái ban đầu

```sql
CREATE PROCEDURE sp_TaoHopDongMoi
    -- Thông tin khách hàng
    @HoTen          NVARCHAR(100),
    @SoDienThoai    VARCHAR(15),
    @CCCD           VARCHAR(20),
    @DiaChi         NVARCHAR(200),

    -- Thông tin hợp đồng
    @SoTienVayGoc   DECIMAL(18,0),
    @Deadline1      DATE,
    @Deadline2      DATE,
    @GhiChu         NVARCHAR(500),
    @NhanVienID     INT,

    -- Danh sách tài sản (TVP)
    @DanhSachTaiSan TaiSanTableType READONLY
AS
BEGIN
    SET NOCOUNT ON;

    BEGIN TRY
        BEGIN TRANSACTION;

        -- =========================================================
        -- 1. Xử lý Khách hàng (Upsert theo CCCD)
        -- =========================================================
        DECLARE @KhachHangID INT;

        IF EXISTS (SELECT 1 FROM KhachHang WHERE CCCD = @CCCD)
        BEGIN
            -- Nếu tồn tại: cập nhật thông tin mới nhất và lấy ID
            UPDATE KhachHang
            SET HoTen        = @HoTen,
                SoDienThoai  = @SoDienThoai,
                DiaChi       = @DiaChi,
                UpdatedAt    = GETDATE()
            WHERE CCCD = @CCCD;

            SELECT @KhachHangID = KhachHangID
            FROM KhachHang
            WHERE CCCD = @CCCD;
        END
        ELSE
        BEGIN
            -- Nếu chưa có: tạo khách hàng mới
            INSERT INTO KhachHang (HoTen, SoDienThoai, CCCD, DiaChi)
            VALUES (@HoTen, @SoDienThoai, @CCCD, @DiaChi);

            SET @KhachHangID = SCOPE_IDENTITY();
        END

        -- =========================================================
        -- 2. Kiểm tra ràng buộc nghiệp vụ (Validation)
        -- =========================================================

        -- Kiểm tra thời hạn hợp đồng
        IF @Deadline1 >= @Deadline2
        BEGIN
            RAISERROR(
                N'Lỗi: Deadline 2 phải lớn hơn Deadline 1.',
                16, 1
            );
        END

        -- Kiểm tra giá trị tài sản bảo đảm: SUM(GiaTri) >= SoTienVay
        DECLARE @TongGiaTriTaiSan DECIMAL(18,0);

        SELECT @TongGiaTriTaiSan = SUM(GiaTriDinhGia)
        FROM @DanhSachTaiSan;

        IF @TongGiaTriTaiSan < @SoTienVayGoc
        BEGIN
            RAISERROR(
                N'Lỗi: Tổng giá trị định giá tài sản không đủ để bảo đảm khoản vay.',
                16, 1
            );
        END

        -- =========================================================
        -- 3. Tạo Hợp đồng
        -- =========================================================
        DECLARE @HopDongID INT;

        INSERT INTO HopDong (
            KhachHangID,
            NgayVay,
            SoTienVayGoc,
            Deadline1,
            Deadline2,
            TrangThai,
            GhiChu
        )
        VALUES (
            @KhachHangID,
            CAST(GETDATE() AS DATE),
            @SoTienVayGoc,
            @Deadline1,
            @Deadline2,
            N'Đang vay',
            @GhiChu
        );

        SET @HopDongID = SCOPE_IDENTITY();

        -- =========================================================
        -- 4. Thêm danh sách Tài sản (bulk insert từ TVP)
        -- =========================================================
        INSERT INTO TaiSan (HopDongID, TenTaiSan, MoTa, GiaTriDinhGia, TrangThai)
        SELECT
            @HopDongID,
            TenTaiSan,
            MoTa,
            GiaTriDinhGia,
            N'Đang cầm cố'
        FROM @DanhSachTaiSan;

        -- =========================================================
        -- 5. Ghi Log lịch sử trạng thái ban đầu
        -- =========================================================
        INSERT INTO LichSuTrangThai (
            HopDongID,
            TrangThaiCu,
            TrangThaiMoi,
            NhanVienID,
            GhiChu
        )
        VALUES (
            @HopDongID,
            NULL,
            N'Đang vay',
            @NhanVienID,
            N'Khởi tạo hợp đồng vay mới'
        );

        COMMIT TRANSACTION;

        -- Trả về kết quả thành công
        SELECT
            @HopDongID AS HopDongID,
            N'Hợp đồng được tạo thành công!' AS Message;

    END TRY

    BEGIN CATCH
        IF @@TRANCOUNT > 0
            ROLLBACK TRANSACTION;

        DECLARE @ErrorMessage NVARCHAR(4000);
        SET @ErrorMessage = ERROR_MESSAGE();
        RAISERROR(@ErrorMessage, 16, 1);
    END CATCH
END;
GO
```

---

## Giải thích các kỹ thuật đã sử dụng

### 1. Tính nguyên tử (Atomicity)

Sử dụng `BEGIN TRANSACTION` để đảm bảo rằng nếu bất kỳ bước nào thất bại — thêm tài sản, tạo hợp đồng, hoặc validation — thì **toàn bộ dữ liệu sẽ bị `ROLLBACK`**, tránh tình trạng dữ liệu bị tạo dở dang.

### 2. Xử lý Khách hàng bằng Upsert

Thay vì báo lỗi khi khách hàng đã tồn tại, logic được xử lý như sau:

- **Nếu tìm thấy khách theo CCCD** → cập nhật thông tin mới nhất
- **Nếu chưa có** → tạo khách hàng mới

**Ưu điểm:**
- Không tạo dữ liệu trùng lặp
- Luôn cập nhật thông tin liên lạc mới nhất
- Dễ đồng bộ dữ liệu khách hàng

### 3. Validation nghiệp vụ

| Ràng buộc | Điều kiện kiểm tra |
|---|---|
| Kiểm tra thời hạn hợp đồng | `Deadline2` phải lớn hơn `Deadline1` |
| Kiểm tra tài sản bảo đảm | `SUM(GiaTriDinhGia) >= SoTienVayGoc` |

### 4. Tối ưu hiệu năng bằng TVP

Thay vì gọi procedure nhiều lần hoặc insert từng tài sản một, sử dụng cú pháp:

```sql
INSERT INTO ... SELECT ... FROM @DanhSachTaiSan;
```

để insert **toàn bộ tài sản trong một lần duy nhất**.

**Ưu điểm:**
- Nhanh hơn, ít round-trip đến server
- Giảm tải đáng kể cho SQL Server

### 5. Bảo mật & Toàn vẹn dữ liệu

Sử dụng `SCOPE_IDENTITY()` thay vì `@@IDENTITY` để đảm bảo lấy **đúng ID vừa tạo trong scope hiện tại**, tránh lỗi sai ID do Trigger gây ra.

---

## Kiểm tra nhanh (Test Script)

Sử dụng đoạn lệnh sau để test Stored Procedure:

```sql
DECLARE @Assets TaiSanTableType;

INSERT INTO @Assets VALUES
    (N'iPhone 15 Pro', N'Màu Titan, 256GB',        20000000),
    (N'MacBook M3',    N'Màu Grey, 16GB RAM',       30000000);

EXEC sp_TaoHopDongMoi 
    @HoTen = N'Hoàng Trường Phúc', 
    @SoDienThoai = '0999888777', -- 
    @CCCD = '123456789012', 
    @DiaChi = N'Thái Nguyên',
    @SoTienVayGoc = 30000000, 
    @Deadline1 = '2026-06-11', 
    @Deadline2 = '2026-07-11', 
    @GhiChu = N'Vay tiêu dùng', 
    @NhanVienID = 1,
    @DanhSachTaiSan = @Assets;
```
<img width="1915" height="1078" alt="image" src="https://github.com/user-attachments/assets/725b5d5d-d52a-4963-913d-1ef30f717aff" />





kiểm tra hợp đồng vừa tạo:


<img width="1917" height="1075" alt="image" src="https://github.com/user-attachments/assets/fde470af-c59f-4031-8978-bac8d9d7e9c6" />



## Phần 3 — Event 2: Tính Toán Công Nợ Thời Gian Thực

## Lựa chọn kỹ thuật

Như đã phân tích ở phần trước,  sẽ sử dụng **Inline Table-Valued Function (Inline TVF)**.
Đây là một kỹ thuật phù hợp vì nó cho phép gọi hàm này bên trong các câu lệnh `SELECT` hoặc
`JOIN` ở Event 4 (Truy vấn nợ xấu) mà không ảnh hưởng đến hiệu năng của cơ sở dữ liệu.

Do quy tắc của Inline TVF là **không được sử dụng lệnh rẽ nhánh `IF...ELSE`
hay khai báo biến `DECLARE`** — toàn bộ logic phải nằm gọn trong một câu `SELECT` —
nên chúng em sẽ áp dụng kỹ thuật **CTE (Common Table Expression — mệnh đề `WITH`)**
để chia nhỏ công thức tính toán thành từng bước một cách rõ ràng và có cấu trúc.

Thay vì viết rời fn_CalcMoneyTransaction để tính từng giao dịch gây chậm hệ thống, em đã thiết kế CTE trong hàm fn_CalcMoneyContract để tự động tổng hợp số dư từ bảng LichSuGiaoDich mới nhất. Điều này giúp tối ưu truy vấn (Chỉ cần gọi 1 hàm duy nhất) mà vẫn đảm bảo tính toán thời gian thực."

---

## Script tạo hàm `fn_CalcMoneyContract`

Dưới đây là script tạo hàm tính công nợ theo đúng các quy tắc lãi suất của hệ thống:

```sql
CREATE FUNCTION fn_CalcMoneyContract
(
    @HopDongID  INT,
    @TargetDate DATE
)
RETURNS TABLE
AS
RETURN
(
    -- SỬ DỤNG CTE ĐỂ CHIA NHỎ LOGIC THEO TỪNG BƯỚC

    WITH BaseInfo AS (
        -- BƯỚC 1: Lấy thông tin Hợp đồng và Giao dịch gần nhất
        SELECT
            HD.HopDongID,
            HD.Deadline1,

            -- TÌM GỐC HIỆN TẠI:
            -- Lấy Dư nợ còn lại từ lần trả tiền gần nhất.
            -- Nếu chưa từng trả, lấy Số tiền vay gốc.
            ISNULL((
                SELECT TOP 1 DuNoConLai
                FROM LichSuGiaoDich
                WHERE HopDongID = HD.HopDongID
                  AND CAST(NgayGiaoDich AS DATE) <= @TargetDate
                ORDER BY NgayGiaoDich DESC
            ), HD.SoTienVayGoc) AS GocHienTai,

            -- TÌM NGÀY BẮT ĐẦU TÍNH LÃI:
            -- Lấy Ngày giao dịch gần nhất.
            -- Nếu chưa từng trả, lấy Ngày vay.
            ISNULL((
                SELECT TOP 1 CAST(NgayGiaoDich AS DATE)
                FROM LichSuGiaoDich
                WHERE HopDongID = HD.HopDongID
                  AND CAST(NgayGiaoDich AS DATE) <= @TargetDate
                ORDER BY NgayGiaoDich DESC
            ), HD.NgayVay) AS NgayBatDauTinh

        FROM HopDong HD
        WHERE HD.HopDongID = @HopDongID
    ),

    DaysCalc AS (
        -- BƯỚC 2: Tính toán Số Ngày chịu Lãi Đơn và Lãi Phạt
        SELECT
            HopDongID,
            GocHienTai,

            -- SỐ NGÀY LÃI ĐƠN:
            -- Đếm từ NgayBatDauTinh đến mốc nhỏ hơn giữa TargetDate và Deadline1
            CASE
                WHEN NgayBatDauTinh < Deadline1 THEN
                    DATEDIFF(DAY, NgayBatDauTinh,
                        CASE WHEN @TargetDate < Deadline1
                             THEN @TargetDate
                             ELSE Deadline1
                        END)
                ELSE 0
            END AS SoNgayLaiDon,

            -- SỐ NGÀY LÃI PHẠT:
            -- Chỉ đếm phần thời gian vượt quá Deadline1
            CASE
                WHEN @TargetDate > Deadline1 THEN
                    DATEDIFF(DAY,
                        CASE WHEN NgayBatDauTinh > Deadline1
                             THEN NgayBatDauTinh
                             ELSE Deadline1
                        END,
                        @TargetDate)
                ELSE 0
            END AS SoNgayLaiPhat

        FROM BaseInfo
    ),

    InterestCalc AS (
        -- BƯỚC 3: Áp dụng công thức tính tiền lãi (Lãi suất 0.5%/ngày = 0.005)
        SELECT
            HopDongID,
            GocHienTai,

            -- Tính tiền Lãi Đơn
            (GocHienTai * 0.005 * SoNgayLaiDon) AS LaiDon,

            -- Tính tiền Lãi Phạt: dựa trên (Gốc + Lãi Đơn đã chốt)
            ((GocHienTai + (GocHienTai * 0.005 * SoNgayLaiDon))
                * 0.005 * SoNgayLaiPhat) AS LaiPhat

        FROM DaysCalc
    )

    -- BƯỚC 4: Tổng hợp và xuất kết quả cuối cùng
    SELECT
        HopDongID,
        GocHienTai                      AS DuNoGoc,
        LaiDon,
        LaiPhat,
        (LaiDon + LaiPhat)              AS TongLai,
        (GocHienTai + LaiDon + LaiPhat) AS TongTienPhaiTra
    FROM InterestCalc
);
GO
```
---
<img width="1912" height="1071" alt="image" src="https://github.com/user-attachments/assets/357e220c-6b44-43eb-943f-35dfaddb7ad9" />

---

## Giải thích thiết kế thuật toán

### 1. Xử lý bài toán "Khách hàng trả nợ nhiều lần"

Một vấn đề thường gặp là nếu khách hàng trả nợ nhiều lần trong quá trình vay
(làm thay đổi dư nợ), việc tính lãi lại từ đầu sẽ cho kết quả sai lệch.

Ở **Bước 1**, vấn đề này được giải quyết bằng cách tách ra hai giá trị:
`GocHienTai` (dư nợ mới nhất) và `NgayBatDauTinh` (ngày giao dịch gần nhất). Nhờ đó:

- Mỗi lần khách hàng đóng tiền, hệ thống tự động **"chốt sổ"** chặng cũ
  và bắt đầu tính lãi mới trên số dư còn lại.
- Đây là cách tính lãi chuẩn được áp dụng trong các phần mềm **Core Banking** thực tế.

### 2. Phân luồng thời gian (Bước 2)

Thay vì viết nhiều câu lệnh `IF` lồng nhau để kiểm tra xem đã quá hạn hay chưa,
chúng em quy bài toán về việc **tính số ngày**.

Thuật toán chia dòng thời gian (từ `NgayBatDauTinh` đến `@TargetDate`) thành **2 đoạn**:

| Đoạn            | Phạm vi                          | Loại lãi   |
|-----------------|----------------------------------|------------|
| Trước Deadline1 | `NgayBatDauTinh` → `Deadline1`   | Lãi đơn    |
| Sau Deadline1   | `Deadline1` → `@TargetDate`      | Lãi phạt   |

Nhờ cách dùng `CASE WHEN`, nếu tính lãi khi chưa đến hạn thì `SoNgayLaiPhat`
sẽ tự động bằng `0`, không cần thêm điều kiện nào khác.

### 3. Công thức lãi phạt kép (Bước 3)

Bước này áp dụng đúng quy tắc nghiệp vụ:

> **Lãi phạt = (Gốc + Lãi đơn) × 0.5% × Số ngày phạt**

Tức là lãi phạt được tính trên **tổng dư nợ đã bao gồm lãi đơn tích lũy**,
không phải chỉ tính trên gốc đơn thuần.

---

## Kiểm thử hàm (Test Script)

Dựa vào `HopDongID = 4` vừa tạo ở Event 1 (số tiền vay 30 triệu,
`Deadline1` là `2026-06-11`), kiểm thử với hai kịch bản:

```sql
-- Kịch bản 1: Truy vấn khi CHƯA ĐẾN HẠN (ngày 2026-05-20)
-- Kết quả mong đợi: LaiPhat = 0, chỉ có LaiDon.
SELECT * FROM fn_CalcMoneyContract(4, '2026-05-20');

-- Kịch bản 2: Truy vấn khi ĐÃ QUÁ HẠN 10 ngày (ngày 2026-06-21)
-- Kết quả mong đợi: Hệ thống tính LaiDon đến ngày 11/06,
-- sau đó cộng vào gốc để tính LaiPhat cho 10 ngày tiếp theo.
SELECT * FROM fn_CalcMoneyContract(4, '2026-06-21');
```

<img width="1910" height="1071" alt="image" src="https://github.com/user-attachments/assets/1da79172-4400-41b2-827e-d1feab7cde38" />

---
## Phần 4 — Event 3: Xử Lý Trả Nợ và Hoàn Trả Tài Sản

## Tổng quan

Đây là thủ tục (Stored Procedure) phức tạp nhất hệ thống vì nó phải tuân thủ nghiêm ngặt
nguyên tắc kế toán tài chính và kết nối trực tiếp với Event 2.

Theo thiết kế ban đầu, thủ tục này phải xử lý được các bài toán sau:

1. **DRY (Don't Repeat Yourself):** Không tự viết lại công thức tính lãi mà phải gọi lại
   hàm `fn_CalcMoneyContract` đã xây dựng ở Event 2.
2. **Nguyên tắc "Lãi trước, Gốc sau":** Khi khách trả tiền, hệ thống phải cấn trừ hết
   vào tiền lãi trước; nếu còn dư mới được trừ vào nợ gốc.
3. **Transaction an toàn:** Ghi log dòng tiền, đổi trạng thái hợp đồng, trả tài sản —
   tất cả phải thành công cùng lúc hoặc thất bại cùng lúc.
4. **Gợi ý tài sản:** Dùng toán học để lọc ra những tài sản khách có thể mang về
   trong khi vẫn đảm bảo giá trị tài sản còn lại đủ thế chấp cho khoản dư nợ mới.

---

## Script tạo Stored Procedure `sp_XuLyTraNo`

```sql
CREATE PROCEDURE sp_XuLyTraNo
    @HopDongID      INT,
    @SoTienKhachTra DECIMAL(18,0),
    @NhanVienID     INT,
    @GhiChu         NVARCHAR(500)
AS
BEGIN
    SET NOCOUNT ON;

    BEGIN TRY

        -- =====================================================
        -- BƯỚC 1: KIỂM TRA TRẠNG THÁI HỢP ĐỒNG
        -- =====================================================
        DECLARE @TrangThaiHD NVARCHAR(50);
        SELECT @TrangThaiHD = TrangThai
        FROM HopDong
        WHERE HopDongID = @HopDongID;

        IF @TrangThaiHD IS NULL
            THROW 50010, N'Lỗi: Không tìm thấy hợp đồng hợp lệ.', 1;

        IF @TrangThaiHD IN (N'Đã thanh lý tài sản', N'Đã thanh toán')
            THROW 50011, N'Lỗi: Hợp đồng đã đóng hoặc tài sản đã bị thanh lý. Hệ thống từ chối thu thêm tiền.', 1;

        -- =====================================================
        -- BƯỚC 2: GỌI HÀM TÍNH NỢ (Tái sử dụng Event 2)
        -- =====================================================
        DECLARE @GocHienTai     DECIMAL(18,0),
                @TongLai        DECIMAL(18,0),
                @TongTienPhaiTra DECIMAL(18,0);

        -- Gọi Inline TVF như một bảng để lấy dữ liệu công nợ tính đến ngày hôm nay
        SELECT
            @GocHienTai      = DuNoGoc,
            @TongLai         = TongLai,
            @TongTienPhaiTra = TongTienPhaiTra
        FROM fn_CalcMoneyContract(@HopDongID, CAST(GETDATE() AS DATE));

        -- Chặn thu dư tiền (tránh lỗi nghiệp vụ)
        IF @SoTienKhachTra > @TongTienPhaiTra
            THROW 50012, N'Lỗi: Số tiền khách đưa LỚN HƠN tổng nợ hiện tại. Vui lòng thối lại tiền thừa và nhập đúng số tiền cần thu.', 1;

        -- =====================================================
        -- BƯỚC 3: XỬ LÝ KẾ TOÁN (Lãi trước → Gốc sau)
        -- =====================================================
        DECLARE @TienTruVaoGoc DECIMAL(18,0) = 0;
        DECLARE @DuNoMoi       DECIMAL(18,0) = @GocHienTai;
        DECLARE @TrangThaiMoi  NVARCHAR(50)  = N'Đang trả góp';

        IF @SoTienKhachTra > @TongLai
        BEGIN
            -- Nếu khách trả dư tiền lãi, phần dư sẽ cấn trừ vào Gốc
            SET @TienTruVaoGoc = @SoTienKhachTra - @TongLai;
            SET @DuNoMoi       = @GocHienTai - @TienTruVaoGoc;
        END
        -- Trường hợp ngược lại: Tiền khách trả <= Lãi thì Gốc giữ nguyên

        -- Nếu dư nợ gốc mới về 0, khách đã trả đủ cả gốc lẫn lãi
        IF @DuNoMoi = 0
            SET @TrangThaiMoi = N'Đã thanh toán';

        -- =====================================================
        -- BƯỚC 4, 5, 6: CẬP NHẬT DATABASE TRONG TRANSACTION
        -- =====================================================
        BEGIN TRANSACTION;

            -- 4.1: Ghi lịch sử dòng tiền (Audit Log)
             INSERT INTO LichSuGiaoDich (HopDongID, NgayGiaoDich, SoTienTra, DuNoConLai, NhanVienID, LoaiGiaoDich, GhiChu)
   VALUES (@HopDongID, GETDATE(), @SoTienKhachTra, @DuNoMoi, @NhanVienID, 
           CASE WHEN @DuNoMoi = 0 THEN N'Thanh toán đủ' ELSE N'Trả nợ' END, 
           @GhiChu);

            -- 4.2: Cập nhật trạng thái Hợp đồng
            UPDATE HopDong
            SET TrangThai = @TrangThaiMoi,
                UpdatedAt = GETDATE()
            WHERE HopDongID = @HopDongID;

            -- 4.3: Ghi Log lịch sử trạng thái (chỉ ghi nếu trạng thái thay đổi)
            IF @TrangThaiMoi != @TrangThaiHD
            BEGIN
                INSERT INTO LichSuTrangThai (
                    HopDongID, TrangThaiCu, TrangThaiMoi, NhanVienID, GhiChu
                )
                VALUES (
                    @HopDongID, @TrangThaiHD, @TrangThaiMoi,
                    @NhanVienID, N'Thay đổi do khách thanh toán tiền'
                );
            END

            -- 4.4: Trả tài sản tự động nếu khách đã thanh toán đủ
            IF @DuNoMoi = 0
            BEGIN
                UPDATE TaiSan
                SET TrangThai = N'Đã trả khách'
                WHERE HopDongID = @HopDongID
                  AND TrangThai  = N'Đang cầm cố';
            END

        COMMIT TRANSACTION;

        -- =====================================================
        -- BƯỚC 7: XUẤT KẾT QUẢ & GỢI Ý TÀI SẢN
        -- =====================================================

        -- Bảng 1: Biên lai điện tử
        SELECT
            N'Giao dịch thành công!' AS ThongBao,
            @TongTienPhaiTra         AS TongNoCu,
            @SoTienKhachTra          AS DaThanhToan,
            @DuNoMoi                 AS DuNoGocConLai,
            @TrangThaiMoi            AS TrangThaiHopDong;

        -- Bảng 2: Gợi ý tài sản có thể rút về (chỉ hiển thị khi khách còn nợ)
        IF @DuNoMoi > 0
        BEGIN
            DECLARE @TongGiaTriDangCamCo DECIMAL(18,0);

            SELECT @TongGiaTriDangCamCo = SUM(GiaTriDinhGia)
            FROM TaiSan
            WHERE HopDongID = @HopDongID
              AND TrangThai  = N'Đang cầm cố';

            -- Điều kiện: (Tổng hiện tại - Giá trị món X) >= Dư nợ mới → Cho phép rút món X
            SELECT
                TaiSanID,
                TenTaiSan,
                GiaTriDinhGia,
                (@TongGiaTriDangCamCo - GiaTriDinhGia) AS GiaTriConLaiSauKhiRut,
                @DuNoMoi                               AS DuNoHienTai,
                N'Đủ điều kiện rút'                    AS TinhTrang
            FROM TaiSan
            WHERE HopDongID = @HopDongID
              AND TrangThai  = N'Đang cầm cố'
              AND (@TongGiaTriDangCamCo - GiaTriDinhGia) >= @DuNoMoi;
        END

    END TRY
    BEGIN CATCH
        IF @@TRANCOUNT > 0 ROLLBACK TRANSACTION;
        THROW;
    END CATCH
END;
GO
```

---
<img width="1909" height="1074" alt="image" src="https://github.com/user-attachments/assets/560d6819-802c-4f00-ac0e-8bc83071dd85" />

---
## Giải thích các điểm kỹ thuật

### 1. Khóa van an toàn bằng `THROW`

Ở Bước 1 và Bước 2, các lỗi nghiệp vụ được thiết lập chặt chẽ:

- Nếu hợp đồng đã thanh lý → **cấm thu tiền**.
- Nếu nhân viên nhập số tiền khách trả lớn hơn tổng nợ hiện tại →
  **báo lỗi ngay lập tức**, yêu cầu nhập lại đúng số để tránh `DuNoConLai`
  bị âm, gây lỗi logic hệ thống về sau.

### 2. Ghi Log Snapshot bất biến

Tại câu lệnh `INSERT INTO LichSuGiaoDich`, giá trị `@DuNoMoi` được lưu trực tiếp,
tạo ra một **"ảnh chụp" vĩnh viễn** về số dư nợ của khách hàng ngay tại thời điểm
thu tiền — đảm bảo tính toàn vẹn cho audit sau này.

### 3. Thuật toán gợi ý tài sản không dùng vòng lặp

Thay vì dùng `CURSOR` để duyệt từng tài sản, bài toán được quy về **phương trình cấp 1**:

> **Điều kiện rút tài sản X:**
> $$\Sigma(\text{Giá trị đang cầm}) - \text{Giá trị X} \geq \text{Dư nợ mới}$$

Chỉ với một câu lệnh `SELECT ... WHERE`, SQL Server trả về chính xác danh sách
tài sản đủ điều kiện cho khách rút về — không cần vòng lặp, không cần con trỏ.

---

## Kiểm thử thực tế (Test Script)

Kịch bản: Khách có `HopDongID = 4` (iPhone 20 triệu + MacBook 30 triệu, vay 30 triệu).
Giả sử nợ gốc 30 triệu + lãi 500 nghìn. Khách mang **5 triệu** đến trả trước.

**Kết quả kỳ vọng:**

1. Bảng 1 (Biên lai giao dịch): Hệ thống nhận 5 triệu của khách. Vì hợp đồng vừa tạo hôm nay nên chưa phát sinh lãi, toàn bộ 5 triệu được cấn trừ thẳng vào nợ gốc (30 triệu - 5 triệu = 25 triệu). Trạng thái hợp đồng tự động chuyển sang "Đang trả góp" cực kỳ chuẩn xác.

2. Bảng 2 (Gợi ý trả đồ): Đây là phần rực rỡ nhất của thuật toán! Tổng tài sản khách cắm là 50 triệu (iPhone 20tr + Mac 30tr). Hệ thống tính nhẩm: Nếu trả chiếc iPhone cho khách, cửa hàng vẫn giữ lại chiếc Mac (30 triệu). Giá trị chiếc Mac (30 triệu) vẫn LỚN HƠN dư nợ hiện tại (25 triệu), nên hệ thống cho phép khách mang iPhone về (Đủ điều kiện rút). Chiếc Mac không hiện ra trong danh sách này vì nếu rút Mac, cửa hàng chỉ còn giữ iPhone 20tr, không đủ bảo đảm cho khoản nợ 25tr.

---
<img width="1912" height="1075" alt="image" src="https://github.com/user-attachments/assets/ca08bc77-b3a9-4e4a-b729-adde3ef39dbc" />

---
```sql
EXEC sp_XuLyTraNo
    @HopDongID      = 4,
    @SoTienKhachTra = 5000000,
    @NhanVienID     = 1,
    @GhiChu         = N'Trả nợ';
```
---
# Phần 5 — Event 4: Truy vấn danh sách nợ xấu (Nợ khó đòi)

**Yêu cầu của phần này:**  
Xuất ra danh sách các khách hàng đã quá hạn **Deadline 1** nhưng chưa thanh toán xong. Báo cáo cần hiển thị số nợ hiện tại và dự phóng **số nợ sau 1 tháng nữa**.

Vì hàm **`fn_CalcMoneyContract`** đã được thiết kế dưới dạng **Inline Table-Valued Function (TVF)**, việc tạo báo cáo này trở nên rất thanh thoát nhờ kỹ thuật **CROSS APPLY**.  
CROSS APPLY hoạt động giống như một vòng lặp JOIN, cho phép truyền từng mã hợp đồng vào hàm tính tiền để lấy kết quả trả về.

---
<img width="1916" height="1078" alt="Screenshot 2026-05-12 003400" src="https://github.com/user-attachments/assets/7fee4b86-224e-41ea-9ddd-5a276a360c1c" />

---

## Script T-SQL tạo hàm báo cáo nợ xấu

```sql
CREATE FUNCTION fn_DanhSachNoXau
(
    @NgayTarget DATE
)
RETURNS TABLE
AS
RETURN
(
    SELECT 
        KH.HoTen AS TenKhachHang,
        KH.SoDienThoai,
        HD.SoTienVayGoc,
        
        -- Tính số ngày đã quá hạn so với Deadline1
        DATEDIFF(DAY, HD.Deadline1, @NgayTarget) AS SoNgayQuaHan,
        
        -- Lấy tổng nợ từ hàm tính toán cho ngày hiện tại (@NgayTarget)
        CongNoHienTai.TongTienPhaiTra AS TongNoHienTai,
        
        -- Lấy tổng nợ từ hàm tính toán cho 1 tháng sau
        CongNoTuongLai.TongTienPhaiTra AS TongNoSau1Thang
        
    FROM HopDong HD
    JOIN KhachHang KH ON HD.KhachHangID = KH.KhachHangID
    
    -- CROSS APPLY 1: Tính nợ cho ngày báo cáo
    CROSS APPLY dbo.fn_CalcMoneyContract(HD.HopDongID, @NgayTarget) CongNoHienTai
    
    -- CROSS APPLY 2: Tính nợ dự phóng 1 tháng sau
    CROSS APPLY dbo.fn_CalcMoneyContract(HD.HopDongID, DATEADD(MONTH, 1, @NgayTarget)) CongNoTuongLai
    
    WHERE 
        -- Chỉ lấy các hợp đồng đã vượt quá Deadline 1
        HD.Deadline1 < @NgayTarget 
        
        -- Bỏ qua các hợp đồng đã đóng
        AND HD.TrangThai IN (N'Đang vay', N'Đang trả góp', N'Quá hạn (nợ xấu)')
);
GO
```
Giải thích các điểm kỹ thuật cốt lõi
1. Sức mạnh của CROSS APPLY
Thông thường, lệnh JOIN chỉ kết nối hai bảng vật lý với nhau. Tuy nhiên, khi cần kết nối một bảng (HopDong) với một hàm (fn_CalcMoneyContract) mà hàm lại nhận tham số là cột từ bảng (HD.HopDongID), thì JOIN thông thường sẽ báo lỗi.
CROSS APPLY được sinh ra chính để giải quyết vấn đề này. Nó duyệt qua từng dòng của bảng HopDong và gọi hàm tính toán tương ứng cho từng dòng.
2. Hàm DATEADD cho dự phóng tài chính
Ở CROSS APPLY thứ hai, thay vì phải viết lại toàn bộ logic tính toán phức tạp cho tháng sau, chỉ cần gọi lại hàm cũ và sử dụng:
SQLDATEADD(MONTH, 1, @NgayTarget)
Điều này giúp code rất sạch (Clean Code) và tuân thủ nguyên tắc DRY (Don’t Repeat Yourself).
3. Khả năng tái sử dụng với tham số @NgayTarget
Hàm được thiết kế nhận tham số @NgayTarget thay vì hard-code GETDATE(). Nhờ đó, hàm cực kỳ linh hoạt:

Có thể xem nợ xấu của ngày hôm nay.
Có thể xem lại nợ xấu của bất kỳ ngày nào trong quá khứ.
Hỗ trợ tốt cho báo cáo lịch sử và kiểm thử.

Cách test thực tế
Hợp đồng số 4 có Deadline1 là 2026-06-11.
Nếu chạy báo cáo vào tháng 5/2026, hợp đồng này sẽ không xuất hiện vì chưa đến hạn.
Để kiểm tra hàm hoạt động đúng, hãy giả lập chạy báo cáo vào ngày 20/06/2026 (lúc này đã quá hạn 9 ngày).
Script test
SQL-- Chạy báo cáo Nợ xấu giả lập vào ngày 2026-06-20
-- Sắp xếp những người nợ lâu nhất lên đầu
```sql
SELECT * 
FROM fn_DanhSachNoXau('2026-06-20')
ORDER BY SoNgayQuaHan DESC;
```

---
<img width="1860" height="913" alt="Screenshot 2026-05-12 003451" src="https://github.com/user-attachments/assets/6e2e6db3-c2b7-4ed7-96ce-d57c18dc0b14" />

---
# Phần 6 — Event 5: Tự động hóa trạng thái & Xử lý nghiệp vụ bổ sung

Phần này chịu trách nhiệm quản lý vòng đời cuối cùng của hợp đồng (**Quá hạn**, **Thanh lý**) và cung cấp nghiệp vụ **"Đảo nợ" (Gia hạn)**.  

Để hệ thống vận hành trơn tru và tối ưu hiệu năng với dữ liệu lớn, kiến trúc được chia thành **4 module** rõ rệt:

## 1. Tối ưu hóa truy vấn bằng Composite Index

Cả chức năng báo cáo nợ xấu (Event 4) và chức năng quét tự động (Event 5) đều thường xuyên lọc dữ liệu trên bảng `HopDong` dựa vào **Trạng thái** và các mốc **Deadline**.  

Việc tạo **Composite Index** giúp SQL Server sử dụng **Index Seek** thay vì **Full Table Scan**, nâng cao đáng kể tốc độ truy vấn.

```sql
CREATE NONCLUSTERED INDEX IX_HopDong_TrangThai_Deadlines 
ON HopDong(TrangThai, Deadline1, Deadline2);
GO
```
2. Event 5.1 — Tự động hóa theo thời gian (Job Quét hệ thống)
Trong thực tế, Trigger không thể tự chạy theo lịch. Cơ chế chuẩn là sử dụng SQL Server Agent Job để gọi Stored Procedure này vào lúc 00:00:00 hàng ngày.
Nhiệm vụ chính:

Quét hợp đồng quá Deadline 1 → Chuyển thành Quá hạn (nợ xấu).
Quét hợp đồng quá Deadline 2 → Đánh dấu tài sản Sẵn sàng thanh lý.
Sử dụng OUTPUT để ghi Audit Log hàng loạt, tránh dùng Cursor gây chậm hệ thống.

---
<img width="1907" height="1072" alt="image" src="https://github.com/user-attachments/assets/4343a7d8-8af5-4990-a28b-0b8f10b3c729" />

---
```SQL
CREATE PROCEDURE sp_QuetTrangThaiNgayMoi
AS
BEGIN
    SET NOCOUNT ON;

    BEGIN TRY
        BEGIN TRANSACTION;

        -- Tạo bảng tạm để hứng log trạng thái thay đổi
        DECLARE @BangTamLog TABLE (
            HopDongID INT, 
            TrangThaiCu NVARCHAR(50), 
            TrangThaiMoi NVARCHAR(50)
        );

        -- [QUY TẮC 1]: Chuyển hợp đồng sang "Quá hạn"
        UPDATE HopDong
        SET TrangThai = N'Quá hạn (nợ xấu)', 
            UpdatedAt = GETDATE()
        OUTPUT inserted.HopDongID, deleted.TrangThai, inserted.TrangThai 
        INTO @BangTamLog
        WHERE TrangThai = N'Đang vay' 
          AND Deadline1 < CAST(GETDATE() AS DATE);

        -- Ghi log cho Quy tắc 1
        INSERT INTO LichSuTrangThai (HopDongID, TrangThaiCu, TrangThaiMoi, NhanVienID, GhiChu)
        SELECT HopDongID, TrangThaiCu, TrangThaiMoi, NULL, N'Hệ thống auto: Quá Deadline 1'
        FROM @BangTamLog;

        DELETE FROM @BangTamLog; -- Reset bảng tạm

        -- [QUY TẮC 2]: Đánh dấu Tài Sản "Sẵn sàng thanh lý"
        UPDATE TaiSan
        SET TrangThai = N'Sẵn sàng thanh lý'
        WHERE TrangThai = N'Đang cầm cố'
          AND HopDongID IN (
              SELECT HopDongID 
              FROM HopDong 
              WHERE TrangThai = N'Quá hạn (nợ xấu)' 
                AND Deadline2 < CAST(GETDATE() AS DATE)
          );

        COMMIT TRANSACTION;
        SELECT N'Hoàn tất quét hệ thống!' AS ThongBao;

    END TRY
    BEGIN CATCH
        IF @@TRANCOUNT > 0 ROLLBACK TRANSACTION;
        THROW;
    END CATCH
END;
GO
```

3. Event 5.2 — Tự động hóa theo Dữ liệu (Trigger Thanh lý)
Trigger này sẽ tự động cập nhật trạng thái tài sản khi nhân viên chuyển hợp đồng sang trạng thái "Đã thanh lý tài sản".

---
<img width="1911" height="1075" alt="Screenshot 2026-05-12 010029" src="https://github.com/user-attachments/assets/239eae94-e23c-4d59-a9c6-effd36a70b1c" />

---

```SQL
CREATE TRIGGER trg_HopDong_ThanhLy 
ON HopDong 
AFTER UPDATE
AS
BEGIN
    SET NOCOUNT ON;

    -- Tối ưu: Chỉ chạy logic nếu cột TrangThai thực sự bị UPDATE
    IF UPDATE(TrangThai)
    BEGIN
        UPDATE TaiSan
        SET TrangThai = N'Đã bán thanh lý'
        FROM TaiSan TS
        INNER JOIN inserted i ON TS.HopDongID = i.HopDongID
        INNER JOIN deleted d ON i.HopDongID = d.HopDongID
        WHERE i.TrangThai = N'Đã thanh lý tài sản' 
          AND d.TrangThai <> N'Đã thanh lý tài sản' 
          AND TS.TrangThai IN (N'Đang cầm cố', N'Sẵn sàng thanh lý');
    END
END;
GO
```
4. Sự kiện bổ sung — Gia hạn hợp đồng (Đảo nợ)
Thủ tục này cho phép khách hàng dời Deadline để tránh lãi phạt, với các quy tắc nghiệp vụ chặt chẽ:

Chặn gian lận: Không cho gia hạn nếu hợp đồng đã thanh toán xong hoặc tài sản đã thanh lý.
Nguyên tắc tài chính: Khách phải trả đủ 100% tiền lãi phát sinh đến ngày gia hạn. Tiền gốc giữ nguyên (hoặc giảm nếu trả dư).
Cứu tài sản: Reset tài sản đang ở trạng thái Sẵn sàng thanh lý về Đang cầm cố.

---
<img width="1910" height="1071" alt="Screenshot 2026-05-12 010237" src="https://github.com/user-attachments/assets/1951e39b-eab0-43a0-8f5b-0a20d6e5504d" />

---

```SQL
CREATE PROCEDURE sp_GiaHanHopDong 
    @HopDongID INT,
    @SoTienKhachTra DECIMAL(18,0),
    @Deadline1_Moi DATE,
    @Deadline2_Moi DATE,
    @NhanVienID INT,
    @GhiChu NVARCHAR(500)
AS
BEGIN
    SET NOCOUNT ON;

    BEGIN TRY
        -- [BƯỚC 1]: KIỂM TRA ĐIỀU KIỆN GIA HẠN
        DECLARE @TrangThaiCu NVARCHAR(50);
        SELECT @TrangThaiCu = TrangThai 
        FROM HopDong 
        WHERE HopDongID = @HopDongID;

        IF @TrangThaiCu IN (N'Đã thanh toán', N'Đã thanh lý tài sản')
            THROW 50023, N'Lỗi: Hợp đồng đã kết thúc hoặc bị thanh lý. Cấm gia hạn!', 1;

        IF @Deadline1_Moi >= @Deadline2_Moi
            THROW 50020, N'Lỗi: Deadline 2 mới phải lớn hơn Deadline 1 mới.', 1;

        -- [BƯỚC 2]: TÍNH CÔNG NỢ HIỆN TẠI
        DECLARE @GocHienTai DECIMAL(18,0), @TongLai DECIMAL(18,0);
        
        SELECT @GocHienTai = DuNoGoc, @TongLai = TongLai
        FROM fn_CalcMoneyContract(@HopDongID, GETDATE());

        IF @SoTienKhachTra < @TongLai
            THROW 50021, N'Lỗi: Khách hàng phải thanh toán tối thiểu toàn bộ số tiền lãi hiện tại.', 1;

        DECLARE @TienTruVaoGoc DECIMAL(18,0) = @SoTienKhachTra - @TongLai;
        DECLARE @DuNoMoi DECIMAL(18,0) = @GocHienTai - @TienTruVaoGoc;

        IF @DuNoMoi <= 0
            THROW 50022, N'Lỗi: Tiền trả đã phủ kín gốc. Vui lòng dùng chức năng Trả Nợ.', 1;

        -- [BƯỚC 3]: CẬP NHẬT GIAO DỊCH (TRANSACTION)
        BEGIN TRANSACTION;

        -- 3.1 Ghi Log giao dịch
        INSERT INTO LichSuGiaoDich (HopDongID, NgayGiaoDich, SoTienTra, DuNoConLai, NhanVienID, LoaiGiaoDich, GhiChu)
        VALUES (@HopDongID, GETDATE(), @SoTienKhachTra, @DuNoMoi, @NhanVienID, N'Gia hạn', @GhiChu);

        -- 3.2 Cập nhật Deadline và Reset Trạng thái Hợp đồng
        UPDATE HopDong
        SET Deadline1 = @Deadline1_Moi,
            Deadline2 = @Deadline2_Moi,
            TrangThai = N'Đang vay', 
            UpdatedAt = GETDATE()
        WHERE HopDongID = @HopDongID;

        -- 3.3 "Cứu" Tài sản (Reset Status)
        UPDATE TaiSan
        SET TrangThai = N'Đang cầm cố'
        WHERE HopDongID = @HopDongID 
          AND TrangThai = N'Sẵn sàng thanh lý';

        -- 3.4 Ghi Log thay đổi trạng thái
        IF @TrangThaiCu <> N'Đang vay'
        BEGIN
            INSERT INTO LichSuTrangThai (HopDongID, TrangThaiCu, TrangThaiMoi, NhanVienID, GhiChu)
            VALUES (@HopDongID, @TrangThaiCu, N'Đang vay', @NhanVienID, N'Reset trạng thái do gia hạn');
        END

        COMMIT TRANSACTION;
        
        SELECT N'Gia hạn thành công! Tài sản đã được khôi phục.' AS ThongBao, 
               @DuNoMoi AS DuNoGocConLai;

    END TRY
    BEGIN CATCH
        IF @@TRANCOUNT > 0 ROLLBACK TRANSACTION;
        THROW;
    END CATCH
END;
GO
```
#  — Kịch bản Test Toàn diện Event 5

Để test chạy mượt mà,  cần "du hành thời gian" bằng cách lùi `NgayVay` của hợp đồng test về đầu tháng (ví dụ: ngày 01/05/2026). Lúc này, hợp đồng đã tồn tại được khoảng 12 ngày, tiền lãi sẽ sinh ra > 0 và các lệnh Insert/Gia hạn sẽ thành công.

Dưới đây là kịch bản test đã được tinh chỉnh hoàn chỉnh:

## Script Test Toàn Diện Event 5

```sql
-- =========================================================================
-- [BƯỚC CHUẨN BỊ]: TẠO NHANH 1 HỢP ĐỒNG TRỄ HẠN ĐỂ TEST
-- =========================================================================
DECLARE @TaiSanTest5 TaiSanTableType;

INSERT INTO @TaiSanTest5 (TenTaiSan, MoTa, GiaTriDinhGia)
VALUES (N'Đồng hồ Rolex Test', N'Bản Fake 1', 5000000);

-- Cài đặt Deadline 1 là ngày 10/05/2026 (Trong quá khứ)
EXEC sp_TaoHopDongMoi 
    @HoTen = N'Khách Test Event 5', 
    @SoDienThoai = '0900555666', 
    @CCCD = '001122334455', 
    @DiaChi = N'TNUT',
    @SoTienVayGoc = 2000000, 
    @Deadline1 = '2026-05-10', 
    @Deadline2 = '2026-06-10', 
    @GhiChu = N'Dữ liệu mồi test Event 5', 
    @NhanVienID = 1,
    @DanhSachTaiSan = @TaiSanTest5;

DECLARE @HD_Test INT = (SELECT MAX(HopDongID) FROM HopDong);

-- FIX DỮ LIỆU: Lùi Ngày Vay về ngày 01/05/2026 để hợp đồng sinh ra tiền lãi > 0
UPDATE HopDong 
SET NgayVay = '2026-05-01' 
WHERE HopDongID = @HD_Test;

SELECT N'TRƯỚC KHI QUÉT' AS GiaiDoan, TrangThai AS TrangThaiHopDong 
FROM HopDong 
WHERE HopDongID = @HD_Test;

-- =========================================================================
-- [TEST 1]: KIỂM TRA JOB QUÉT TỰ ĐỘNG (EVENT 5.1)
-- =========================================================================
EXEC sp_QuetTrangThaiNgayMoi;

SELECT N'SAU KHI QUÉT' AS GiaiDoan, TrangThai AS TrangThaiHopDong 
FROM HopDong 
WHERE HopDongID = @HD_Test;

-- =========================================================================
-- [TEST 2]: KIỂM TRA CHỨC NĂNG GIA HẠN
-- =========================================================================
-- Lúc này, do NgayVay là 01/05, tiền lãi sẽ được tính cho ~12 ngày (chắc chắn > 0)
DECLARE @TienLaiPhaiNop DECIMAL(18,0) = (
    SELECT TongLai 
    FROM fn_CalcMoneyContract(@HD_Test, GETDATE())
);

EXEC sp_GiaHanHopDong 
    @HopDongID = @HD_Test, 
    @SoTienKhachTra = @TienLaiPhaiNop, 
    @Deadline1_Moi = '2026-06-12', 
    @Deadline2_Moi = '2026-07-12', 
    @NhanVienID = 1, 
    @GhiChu = N'Khách đóng lãi xin gia hạn';

SELECT N'SAU KHI GIA HẠN' AS GiaiDoan, 
       TrangThai AS TrangThaiHopDong, 
       Deadline1, 
       Deadline2 
FROM HopDong 
WHERE HopDongID = @HD_Test;

-- =========================================================================
-- [TEST 3]: KIỂM TRA TRIGGER THANH LÝ (EVENT 5.2)
-- =========================================================================
UPDATE HopDong 
SET TrangThai = N'Đã thanh lý tài sản' 
WHERE HopDongID = @HD_Test;

SELECT N'KIỂM TRA TRIGGER TÀI SẢN' AS GiaiDoan, 
       TenTaiSan, 
       TrangThai AS TrangThaiTaiSan 
FROM TaiSan 
WHERE HopDongID = @HD_Test;
```
---
<img width="1911" height="1069" alt="Screenshot 2026-05-12 011111" src="https://github.com/user-attachments/assets/6c8c3244-4ead-4a9a-be89-12df2604b11c" />

---
# Kết quả mong đợi sau khi chạy kịch bản

 Bảng 1 & 2 (Khởi tạo)

Hợp đồng số **7** được tạo thành công với trạng thái ban đầu là **Đang vay**.

Việc lùi ngày vay về **01/05/2026** đã giải quyết triệt để lỗi logic về dòng tiền (lãi phát sinh).

Bảng 3 & 4 (Event 5.1 - Quét tự động)

Stored Procedure `sp_QuetTrangThaiNgayMoi` đã phát hiện hợp đồng trễ hạn (Deadline 1 là **10/05**) và tự động chuyển trạng thái sang **Quá hạn** (nợ xấu).

 Bảng 5 & 6 (Sự kiện Gia hạn)

Khách hàng nộp đủ tiền lãi → Hệ thống báo **"Gia hạn thành công!"**.

Hợp đồng được reset về trạng thái **Đang vay**, đồng thời dời Deadline sang tháng 6 và tháng 7.

 Bảng 7 (Event 5.2 - Trigger Thanh lý)

Khi cập nhật hợp đồng sang trạng thái **"Đã thanh lý tài sản"**, Trigger `trg_HopDong_ThanhLy` tự động chuyển chiếc **Đồng hồ Rolex Test** sang trạng thái **Đã bán thanh lý**.

---

**Kết luận**: Toàn bộ luồng **Event 5** (Tự động hóa trạng thái + Gia hạn + Trigger) đã hoạt động chính xác, không có lỗi logic nào.


*Sinh viên thực hiện —Hoàng Trường Phúc- Lớp 59KMT*
