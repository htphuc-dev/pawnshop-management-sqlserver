# pawnshop-management-sqlserver
Pawnshop management database using SQL Server and T-SQL
# 🏦 CSDL Quản Lý Cầm Đồ — Bài Tập số 03

> **Môn học:** Hệ quản trị CSDL | **Lớp:** 59KMT | **SV** Hoàng Trường Phúc

---

## 📋 Mục lục

1. [Mô tả bài toán & Giả định hệ thống](#1-mô-tả-bài-toán--giả-định-hệ-thống)
2. [Thiết kế ERD](#2-thiết-kế-erd)
3. [Chi tiết các bảng (Schema)](#3-chi-tiết-các-bảng-schema)
4. [CHECK Constraints đầy đủ](#4-check-constraints-đầy-đủ)
5. [Chiến lược FK — ON DELETE / ON UPDATE](#5-chiến-lược-fk--on-delete--on-update)
6. [Quy tắc nghiệp vụ & Vòng đời trạng thái](#6-quy-tắc-nghiệp-vụ--vòng-đời-trạng-thái)
7. [Thuật toán tính lãi](#7-thuật-toán-tính-lãi)
8. [Tối ưu truy vấn bằng INDEX](#8-tối-ưu-truy-vấn-bằng-index)
9. [Transaction Consistency](#9-transaction-consistency)
10. [Giải thích thiết kế](#10-giải-thích-thiết-kế)
11. [Chuẩn hóa 3NF](#11-chuẩn-hóa-3nf)
12. [Hướng dẫn chạy Script](#12-hướng-dẫn-chạy-script)

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

### Pseudocode T-SQL (`fn_CalcMoneyContract`)

```sql
-- Bước 1: Tính lãi đơn đầy đủ đến Deadline1
SET @NgayDenDL1 = DATEDIFF(DAY, @NgayVay, @Deadline1)
SET @LaiDon     = @SoTienVayGoc * 0.005 * @NgayDenDL1

-- Bước 2: Phân nhánh theo TargetDate
IF @TargetDate <= @Deadline1
BEGIN
    -- Chưa qua deadline: chỉ tính lãi đơn đến TargetDate
    SET @SoNgay = DATEDIFF(DAY, @NgayVay, @TargetDate)
    RETURN @SoTienVayGoc + (@SoTienVayGoc * 0.005 * @SoNgay)
END
ELSE
BEGIN
    -- Đã quá hạn: lãi phạt trên gốc mới
    SET @GocMoi   = @SoTienVayGoc + @LaiDon
    SET @NgayPhat = DATEDIFF(DAY, @Deadline1, @TargetDate)
    SET @LaiPhat  = @GocMoi * 0.005 * @NgayPhat
    RETURN @SoTienVayGoc + @LaiDon + @LaiPhat
END
```

---

## 8. Tối ưu truy vấn bằng INDEX

Phần này thể hiện **tư duy thiết kế thực tế** — biết bảng sẽ bị query theo pattern nào, tạo index phù hợp.

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

## 12. Hướng dẫn chạy Script

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



*Sinh viên thực hiện — Lớp 59KMT*
