# 🎯 BÀI 3 — Quyết định data modeling thực chiến

**Phủ:** E-RT-001 → 005

> 💡 Bài 1 chọn DB, Bài 2 chuẩn hóa schema. Bài này là về những **quyết định nhỏ mà âm thầm gây hậu quả lớn**: chọn kiểu khóa chính, cách xóa dữ liệu, cách lưu tiền và thời gian. Sai một trong số này từ đầu thì *sửa cực kỳ đắt* vì dữ liệu đã trót lưu theo cách cũ.

---

## ① Mục tiêu & vị trí

Đây là cụm quyết định "nhỏ mà chết người": **kiểu khóa chính**, **cách xóa**, **lưu tiền/thời gian**, **OLTP vs OLAP**.

> 🔎 Đây là phần **tech lead (TL) hay bị hỏi** trong phỏng vấn, vì nó **lộ độ chín** của một kỹ sư. Người mới chỉ quan tâm "code chạy được"; người chín quan tâm "5 năm nữa dữ liệu này còn cứu được không".

Vị trí: bài này **khép lại cụm "thiết kế đúng"** (Bài 1→3) trước khi chuyển sang **performance** ở **Bài 4+**.

> 🎯 **Chốt định vị:** Mọi quyết định ở đây đều là **hợp đồng vĩnh viễn với dữ liệu**. Code sai sửa trong 5 phút; kiểu dữ liệu sai phải migrate hàng trăm triệu dòng.

---

## ② Giảng cơ bản → nâng cao

### (a) OLTP vs OLAP  `E-RT-001`

> **Ẩn dụ:** **OLTP** là quầy thu ngân siêu thị — nhiều giao dịch nhỏ, phải nhanh và chính xác từng đồng. **OLAP** là phòng phân tích cuối tháng — đọc gộp hàng triệu hóa đơn để ra báo cáo. Bắt quầy thu ngân vừa tính tiền khách vừa chạy báo cáo doanh thu năm → khách xếp hàng dài.

| Tiêu chí | **OLTP** | **OLAP** |
|----------|----------|----------|
| Đặc trưng | nhiều **giao dịch nhỏ**, **low latency** | **aggregate lớn**, ưu tiên **throughput** |
| Tổ chức dữ liệu | **chuẩn hóa** (normalize) | **denormalize** / **columnar** |
| Mục đích | ghi/đọc từng bản ghi nhanh | quét + tổng hợp khối lượng lớn |

> 🚨 **Đừng chạy báo cáo nặng trên DB giao dịch chính.** Một câu query OLAP nặng có thể "hút" hết tài nguyên, làm chậm toàn bộ giao dịch của khách thật. Tách kho phân tích ra riêng (xem Bài 2 — **E-NORM-005** về tách OLTP/OLAP).

---

### (b) UUID vs auto-increment  `E-RT-002`

> **Ẩn dụ:** **B-tree** index giống một thư viện sắp sách theo thứ tự. **auto-increment** (ID tăng dần) = sách mới luôn xếp vào *cuối kệ* — gọn gàng, nhanh. **UUIDv4** (ID ngẫu nhiên) = mỗi cuốn mới phải *chen vào giữa kệ ngẫu nhiên* → liên tục phải xê dịch, tách kệ, chừa chỗ → lộn xộn và tốn công.

| Loại khóa | Kích thước | Tính chất | Ưu | Nhược |
|-----------|-----------|-----------|----|-------|
| **bigserial** / **identity** | 8 byte | **tuần tự** | ✅ insert dồn cuối **B-tree**, ít **fragment** | ❌ **lộ thứ tự**/đếm được; khó **merge nhiều nguồn** |
| **UUIDv4** | 16 byte | **ngẫu nhiên** | ✅ không đoán được; sinh ở client | ❌ insert rải khắp B-tree → **page split** + **fragment** + **index phình**; key nặng |
| **UUIDv7** / **ULID** | 16 byte | **time-ordered** | ✅ **dung hòa**: vừa phân tán/khó đoán, vừa tuần tự theo thời gian | ❌ vẫn 16 byte; cần verify hỗ trợ |

**Vì sao UUIDv4 làm phân mảnh B-tree?** Vì **B-tree** thích chèn *theo thứ tự*. Khi ID ngẫu nhiên, mỗi insert rơi vào một trang (page) ngẫu nhiên đang đầy → DB phải **page split** (tách trang) để chừa chỗ → bảng/index phình và phân mảnh.

Timeline minh họa khác biệt khi insert:

```text
bigserial (tuần tự) — luôn dồn cuối:
  T1: insert id=1001 → vào cuối B-tree   ✅
  T2: insert id=1002 → ngay sau 1001     ✅ trang đầy đẹp, không split

UUIDv4 (ngẫu nhiên) — chen giữa khắp nơi:
  T1: insert "f3a9..." → rơi giữa trang A (đang đầy)
       ↓
  T2: ❌ page split: tách trang A làm đôi để chừa chỗ
  T3: insert "0b21..." → rơi giữa trang C (đang đầy)
       ↓
  T4: ❌ lại split → fragment tích lũy, index phình to dần
```

> 🔎 **Lưu ý (verify hỗ trợ native):** **UUIDv7** / **ULID** là **time-ordered** nên insert gần như tuần tự → tránh được vấn đề của v4 mà vẫn giữ được tính phân tán. **Postgres 18 có `uuidv7()`** (verify postgresql.org).

> 🎯 **Chốt:** Cần khóa tuần tự đơn giản → **bigserial**. Cần sinh ID ở nhiều nguồn / không đoán được → ưu tiên **UUIDv7**/**ULID** hơn **UUIDv4**.

---

### (c) Soft delete vs hard delete  `E-RT-003`

> **Ẩn dụ:** **hard delete** = xé tờ giấy bỏ thùng rác (mất hẳn). **soft delete** = đóng dấu "ĐÃ HỦY" lên giấy nhưng vẫn để trong tủ (còn lịch sử, nhưng tủ ngày càng đầy giấy bỏ).

**soft delete** = thêm cột **`deleted_at`**; xóa = ghi thời điểm, không xóa thật. Giữ được lịch sử, **nhưng** có 3 cái giá:

- ❌ **Phình bảng** — dữ liệu "đã xóa" vẫn nằm đó, ngày càng nặng.
- ❌ **Mọi query phải lọc** `WHERE deleted_at IS NULL` — quên một chỗ → hiện cả dữ liệu đã xóa.
- ❌ **Vướng unique constraint** — email "đã xóa" vẫn chiếm chỗ, **chặn người dùng đăng ký lại** bằng email đó.

Timeline minh họa bẫy unique:

```text
SOFT DELETE đụng unique constraint trên email:
  T1: user A (email=a@x.com) bị soft delete → deleted_at = now()
       (dòng vẫn còn trong bảng, chỉ đánh dấu)
  T2: người dùng muốn đăng ký LẠI bằng a@x.com
       ↓
  T3: ❌ INSERT thất bại — unique(email) thấy a@x.com đã tồn tại
       → dù về mặt logic email đó "đã xóa" và nên dùng lại được
  ✅ cách sửa: partial unique index — chỉ ràng buộc dòng CÒN SỐNG
```

> ⚠️ **Cân nhắc `archive table` thay thế:** chuyển bản ghi đã xóa sang một bảng lưu trữ riêng → bảng chính gọn, vẫn giữ lịch sử, không vướng unique. **soft delete nên có chủ đích**, không bật mặc định cho mọi bảng.

---

### (d) Lưu tiền & thời gian đúng  `E-RT-004`

#### 💰 Tiền — KHÔNG dùng float

> **Ví dụ trực giác:** Máy tính lưu số thực dạng nhị phân, mà `0.1` không biểu diễn chính xác được trong nhị phân (giống `1/3` không viết hết được dạng thập phân). Kết quả: **`0.1 + 0.2` ra `0.30000000000000004`**, không phải `0.3`. Với tiền, sai số đó tích lũy thành thâm hụt sổ sách.

| Cách lưu tiền | Đánh giá |
|---------------|----------|
| **float** / **double** | 🚨 CẤM — **sai số nhị phân** làm lệch tiền |
| **numeric** (decimal) | ✅ Chính xác tuyệt đối, hỗ trợ phép tính tài chính |
| **integer cents** (số nguyên đơn vị nhỏ nhất) | ✅ Chính xác, nhẹ — ví dụ `25000` = 250đ |

#### 🕐 Thời gian — dùng timestamptz

| Cách lưu | Đánh giá |
|----------|----------|
| **timestamp** (no tz) | ❌ Không có timezone → sai khi server/người dùng khác múi giờ |
| **timestamptz** | ✅ **Lưu theo UTC**, tránh lỗi múi giờ |

**Vì sao timestamp trần nguy hiểm?** Vì nó lưu một con số "9:00" mà không biết 9:00 ở múi giờ nào. Server ở UTC, user ở Việt Nam (UTC+7) → cùng một thời điểm bị hiểu lệch 7 tiếng. **timestamptz** lưu chuẩn về **UTC** nên ai đọc cũng quy đúng về múi giờ của mình.

---

## ③ ⚠️ Cũ → Mới (kiến thức bị thay thế)

| Quan niệm CŨ ❌ | Hiện nay ✅ | Vì sao |
|----------------|------------|--------|
| **UUIDv4** làm PK "cho chắc" | **bigserial** hoặc **UUIDv7**/**ULID** | v4 **fragment B-tree**, insert chậm |
| **float** lưu tiền | **numeric** / **integer cents** | float **sai số** |
| **timestamp** (no tz) | **timestamptz** | tránh lỗi múi giờ |
| **soft delete** mặc định mọi bảng | Cân nhắc **archive table**; soft delete có chủ đích | tránh **phình** + **bẫy unique** |

---

## ④ Thực tế + bigtech

> 🔎 **Pattern ngành (verify — có thể đã đổi):**
> - **Hệ phân tán nhiều nguồn ghi** thường dùng **UUID**/**ULID** để **sinh ID không cần round-trip DB** (client tự tạo ID ngay, không phải hỏi DB "cho tôi số tiếp theo").
> - **Hệ tài chính** luôn lưu tiền dạng **integer minor unit** + giữ **audit log** thay vì **xóa cứng** — vì lịch sử tiền là bắt buộc về pháp lý.
> - **Tách OLTP/OLAP qua data warehouse** là **chuẩn ngành**.

> 💡 Lý do chung: ở quy mô lớn, các quyết định "nhỏ" này quyết định hệ thống có *scale* và *audit* được hay không.

---

## ⑤ Code + cấu hình

```sql
-- PostgreSQL 18 (uuidv7 native — verify postgresql.org)
CREATE TABLE account (
  id            uuid        PRIMARY KEY DEFAULT uuidv7(),  -- time-ordered → dung hòa (không dùng v4 random)
  balance_cents bigint      NOT NULL DEFAULT 0,            -- tiền = integer cents, KHÔNG float
  created_at    timestamptz NOT NULL DEFAULT now(),        -- timestamptz (UTC), KHÔNG timestamp trần
  deleted_at    timestamptz                                -- soft delete: NULL = còn sống, có giá trị = đã xóa
);

-- Soft delete vướng unique → partial unique index:
-- chỉ ràng buộc email duy nhất TRÊN CÁC DÒNG CÒN SỐNG
-- → cho phép tái dùng email sau khi dòng cũ bị soft delete
CREATE UNIQUE INDEX uq_account_email_live
  ON account (email) WHERE deleted_at IS NULL;
```

> 🚨 **AI hay sai ở đây:** AI sinh migration hay chọn `uuid DEFAULT gen_random_uuid()` (**UUIDv4**) + thiếu **numeric** + **timestamp** trần — **chạy được nhưng sai kiến trúc**.

### 🔍 Checklist TL kiểm trước khi merge migration  `E-RT-005`

> ⭐ Đây là **checklist 5 điểm** một tech lead dùng để review mọi migration:
> 1. **Index có khớp access pattern không?** (index vô dụng nếu không phục vụ query thật)
> 2. **Kiểu money/uuid/tz đúng chưa?** (tiền=integer/numeric, PK=v7/serial, thời gian=timestamptz)
> 3. **nullable / FK / constraint** đã đúng chưa?
> 4. **Migration có khóa bảng lâu không?** (xem BT2 — bảng lớn rất nguy hiểm)
> 5. **Rollback được không?** (nếu hỏng giữa chừng có lùi lại được không)

> 🎯 **Bẫy cốt lõi:** **"chạy được" ≠ "đúng kiến trúc"**. Migration chạy thành công trên máy dev hoàn toàn có thể là quả bom trên production.

---

## ⑥ Keywords

> 🧠 **HIỂU** (nắm bản chất):
> Vì sao **UUIDv4** **fragment B-tree** · vì sao **float** sai cho tiền · **bẫy unique** của **soft delete** · **OLTP vs OLAP**.

> 📌 **THUỘC** (nhớ chính xác):
> Tiền = **numeric** / **integer cents** · thời gian = **timestamptz** · **UUIDv7**/**ULID** = **time-ordered**.

> 🛠️ **LÀM ĐƯỢC** (kỹ năng tay):
> Review một migration AI sinh ra theo **checklist 5 điểm** ở mục ⑤.

---

## ⑦ Mental model

> 🎯 **NGUYÊN TẮC VÀNG CỦA BÀI:**
> *"**Kiểu dữ liệu là hợp đồng vĩnh viễn**. Tiền = **integer**, thời gian = **timestamptz**, PK = **tuần tự / time-ordered** — sai từ đầu thì sửa rất đắt."*

Cách dùng: trước khi tạo bất kỳ bảng nào, dừng lại 30 giây tự hỏi 3 câu — *tiền lưu thế nào? thời gian lưu thế nào? PK chọn kiểu gì?* — vì đây là ba thứ gần như không thể đổi lại rẻ sau này.

---

## ⑧ Phỏng vấn & thách đố

| Mức | Câu hỏi | Trả lời cốt lõi |
|-----|---------|-----------------|
| **(TB)** | **UUID** vs **bigserial** PK — trade-off với index/insert? | **v4** random → **fragment**; **serial** tuần tự; **v7**/**ULID** dung hòa |
| **(TB)** | Vì sao không **float** cho tiền? | **sai số nhị phân**; dùng **numeric**/**cents** |
| **(khá)** | **timestamptz** vs **timestamp**? | tz lưu **UTC** tránh lỗi múi giờ |
| **(khá)** | Cạm bẫy **soft delete**? | **phình** + phải lọc + **vướng unique** → **partial index** |

> 🧩 **Thách đố tình huống (TL):**
> *AI sinh schema chạy được — bạn kiểm gì trước khi merge?*
>
> Trả lời chuẩn: chạy đủ **checklist 5 điểm** ở mục ⑤ (index/access pattern, kiểu money/uuid/tz, nullable/FK/constraint, khóa bảng, rollback).
>
> ⚠️ **Bẫy phải vạch ra:** **"chạy được" ≠ "đúng kiến trúc"**. Đừng để "không báo lỗi" ru ngủ.

---

## ⑨ Bài tập + tiêu chí tự chấm

### BT1 — Thiết kế bảng `payment`

Thiết kế bảng `payment` lưu **tiền** + **thời gian** + **chống xóa cứng**.

> ✅ **Tiêu chí Đạt:**
> - Tiền dùng **integer cents** hoặc **numeric** (không float).
> - Thời gian dùng **timestamptz**.
> - Chống xóa cứng bằng **soft delete** hoặc **archive table** *có chủ đích*.
> - ❌ Trượt nếu dùng float/timestamp trần hoặc hard delete không lưu lịch sử.

### BT2 — Migration trên bảng 500 triệu dòng

Một migration `ALTER TABLE big_table ADD COLUMN ... DEFAULT ...` trên bảng **500 triệu dòng** — rủi ro gì?

> ✅ **Tiêu chí Đạt:**
> - Nhận ra rủi ro **rewrite / khóa bảng lâu** (tùy version DB) → có thể làm sập service.
> - Đề xuất cách an toàn: **thêm cột nullable trước, rồi backfill theo batch** (chia nhỏ cập nhật).
> - ❌ Trượt nếu chỉ nói "chạy là được" mà không thấy nguy cơ khóa bảng.

> 🔎 Timeline rủi ro để hình dung:
> ```text
> ALTER TABLE big_table ADD COLUMN ... DEFAULT ... (500tr dòng):
>   T1: lệnh chạy → (tùy version) DB phải REWRITE toàn bảng
>        ↓
>   T2: ❌ khóa bảng trong nhiều phút/giờ → mọi giao dịch chờ → service treo
>   ✅ an toàn: ADD COLUMN nullable (nhanh) → backfill từng batch → set NOT NULL sau
> ```

---

## ⑩ Đọc thêm

- **PostgreSQL — Date/Time & Numeric Types** — `postgresql.org/docs/current/datatype.html`
- **"UUID vs serial as primary key"** — Cybertec / 2ndQuadrant blogs (verify)

> 🎯 **Một câu mang về:** Những quyết định "nhỏ" về kiểu dữ liệu là thứ phân biệt junior và senior. Junior hỏi "code có chạy không"; senior hỏi "5 năm nữa tôi còn migrate nổi cái này không".
