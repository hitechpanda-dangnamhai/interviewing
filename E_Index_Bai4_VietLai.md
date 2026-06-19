# 🎯 BÀI 4 — Index & B-tree

**Phủ:** E-IDX-001 → 009

> 💡 Đây là **trụ cột performance đọc của RDBMS**. Học xong bài này bạn hiểu ba điều cốt lõi: vì sao **index** tăng tốc đọc nhưng **làm chậm ghi**, vì sao **B+tree** là cấu trúc mặc định, và vì sao *có index mà query vẫn full scan*.

---

## ① Mục tiêu & vị trí

Bài này dạy về **index** — công cụ số một để tăng tốc đọc trong **RDBMS**.

Ba câu hỏi bạn phải trả lời được sau bài:
1. Vì sao **index tăng tốc đọc nhưng làm chậm ghi**?
2. Vì sao **B+tree** là mặc định (chứ không phải cấu trúc khác)?
3. Vì sao **có index mà query vẫn full scan** (không dùng được index)?

Vị trí: bài này là **nền cho Bài 5** (composite index — index nhiều cột) và **Bài 10** (đọc **EXPLAIN** để xem planner có dùng index không). Nó cũng nối tiếp khái niệm **B-tree** đã gặp ở Bài 3 (`E-RT-002`).

> 🎯 **Chốt định vị:** Index không phải "phép thuật cứ thêm là nhanh". Nó là một *đánh đổi có giá*, và biết khi nào KHÔNG nên thêm index cũng quan trọng như biết khi nào nên.

---

## ② Giảng cơ bản → nâng cao

### (a) Trực giác  `E-IDX-001`

> **Ẩn dụ:** **index** giống **mục lục cuối sách**. Muốn tìm chữ "caching", bạn không lật từng trang một (đó là **full table scan**) — bạn tra mục lục, thấy "caching → trang 230", rồi nhảy thẳng tới. **index** là một **cấu trúc phụ** giúp tìm nhanh mà không phải quét hết.

**Vì sao cần index?** Vì không có nó, để tìm 1 dòng trong bảng triệu dòng, DB phải đọc *cả triệu dòng* để kiểm tra (**full table scan**). Với index, DB đi thẳng tới đúng chỗ.

---

### (b) Cơ chế

#### 🔍 Vì sao index làm chậm ghi  `E-IDX-002`

> **Ví dụ trực giác:** Mỗi lần thêm một chữ mới vào sách, bạn không chỉ viết nó vào trang nội dung mà còn phải **cập nhật mục lục**. Có 5 mục lục (theo tên, theo chủ đề, theo tác giả...) → mỗi lần thêm phải sửa cả 5.

Mỗi **insert/update/delete** phải **cập nhật cả index**, không chỉ dữ liệu. **Càng nhiều index, ghi càng đắt** — đây là **trade-off read vs write** kinh điển.

Timeline minh họa **write amplification** (một lần ghi nở ra thành nhiều thao tác):

```text
INSERT 1 dòng order vào bảng có 4 index:
  T1: ghi dòng vào heap (dữ liệu thật)        ✅ 1 thao tác
  T2: cập nhật index (user_id)                ← +1
  T3: cập nhật index (status)                 ← +1
  T4: cập nhật index (created_at)             ← +1
  T5: cập nhật index (lower(email))           ← +1
       ↓
  Kết quả: 1 lần INSERT logic = 5 thao tác ghi vật lý
  → càng nhiều index, mỗi ghi càng "nở" ra (write amplification)
```

> 🎯 **Chốt:** Index miễn phí khi đọc, nhưng *đánh thuế mọi lần ghi*. Đây là lý do "cứ thêm index cho mọi cột" là sai.

#### 🔍 Vì sao B+tree là mặc định  `E-IDX-003`

> **Ví dụ trực giác:** **B+tree** giống một danh bạ điện thoại sắp theo thứ tự, các trang nối nhau. Bạn vừa tra được *một cái tên cụ thể* (equality), vừa lật được *một dải tên từ A đến C* (range), vừa đọc theo *thứ tự* (sort) — tất cả nhanh.

**B+tree** là cây cân bằng, được chọn mặc định vì nó hỗ trợ **nhiều kiểu truy vấn** cùng lúc:

| Khả năng | **B+tree** | **Hash index** |
|----------|-----------|----------------|
| **equality** (`=`) | ✅ | ✅ |
| **range** (`>`, `<`, `BETWEEN`) | ✅ | ❌ |
| **sort** (`ORDER BY`) | ✅ | ❌ |
| **prefix** (`LIKE 'abc%'`) | ✅ | ❌ |
| Đọc theo **block đĩa** hiệu quả | ✅ (cây cân bằng) | — |
| **Scan range** | ✅ (lá liên kết với nhau) | ❌ |

**Vì sao B+tree thắng hash?** Vì **hash index** chỉ phục vụ **equality** (tra đúng một giá trị) — không làm được **range** hay **sort**. **B+tree** làm được tất cả nhờ giữ dữ liệu *có thứ tự* và *các lá liên kết* để quét dải liền mạch.

#### 🔍 Clustered vs non-clustered  `E-IDX-004`

> **Ẩn dụ:** **clustered index** = sách được *đóng gáy theo đúng thứ tự mục lục* (thứ tự vật lý của trang = thứ tự index). **non-clustered** = sách để lộn xộn, mục lục chỉ ghi "trang số mấy" để bạn đi tìm.

| Tiêu chí | **clustered** | **non-clustered** |
|----------|---------------|-------------------|
| Ý nghĩa | quyết định **thứ tự vật lý** của row | chỉ là cấu trúc phụ trỏ tới row |
| Ví dụ | **MySQL InnoDB**: PK = **clustered** | index thường |

> 🔎 **Lưu ý sâu về Postgres:** **Postgres KHÔNG có clustered index thật** — nó dùng **heap** (dữ liệu nằm không theo thứ tự cố định). Lệnh **`CLUSTER`** chỉ **sắp xếp một lần**, **không tự duy trì** (ghi mới lại lộn xộn). Đây là điểm khác cốt lõi với **MySQL InnoDB** — đừng nhầm.

#### 🔍 Covering index / index-only scan  `E-IDX-005`

> **Ví dụ trực giác:** Nếu mục lục đã ghi sẵn *cả nội dung tóm tắt* bạn cần, bạn khỏi phải giở vào trang gốc. Đó là **index-only scan** — đọc xong index là đủ, khỏi đụng tới dữ liệu thật (heap).

**covering index** = index **chứa đủ mọi cột query cần** → DB **khỏi đọc heap** (dữ liệu gốc) → nhanh hơn nhiều. Trong **Postgres** dùng cú pháp **`INCLUDE (cols)`** để nhét thêm cột vào index.

Timeline so sánh:

```text
KHÔNG covering — phải đọc heap thêm:
  T1: tra index (user_id) → tìm thấy vị trí dòng
       ↓
  T2: ❌ nhảy vào heap đọc status, total_cents  ← thêm random I/O

COVERING (INCLUDE status, total_cents) — index-only scan:
  T1: tra index → đã có sẵn user_id + status + total_cents
       ✅ trả kết quả luôn, KHỎI đụng heap → nhanh hơn
```

---

### (c) Mép giới hạn

#### 🔍 Selectivity thấp  `E-IDX-006`

> **Ví dụ trực giác:** Mục lục mà 90% trang đều ghi "có nhắc tới chữ *và*" thì vô dụng — vì tra xong vẫn phải đọc gần hết sách. Index cũng vậy: nếu giá trị lặp lại quá nhiều, index không giúp được gì.

Index trên **`is_active`** (boolean) thường **vô dụng**. **Vì sao?** Vì nếu 80% dòng `is_active = true`, **planner** thấy phải đọc *phần lớn bảng* qua index thì **random I/O** còn đắt hơn **full scan** tuần tự → nó chọn full scan.

Hai khái niệm cần nhớ:
- **cardinality** = số lượng giá trị *khác nhau* của cột (boolean chỉ có 2 → cardinality thấp).
- **selectivity** = mức độ một điều kiện *lọc ra ít dòng*. **selectivity cao** (lọc ra ít) → index hữu ích; **selectivity thấp** (lọc ra nhiều) → index vô dụng.

#### 🔍 Có index vẫn full scan  `E-IDX-007`

> ⚠️ Khái niệm chìa khóa: **sargable** (Search-ARGument-ABLE) = điều kiện *viết sao cho index dùng được*. Viết không sargable → index "tê liệt".

Các nguyên nhân **có index mà vẫn full scan**:

| Nguyên nhân | Ví dụ "phá" index | Vì sao |
|-------------|-------------------|--------|
| **function trên cột** | `WHERE lower(email)=...` mà không có **expression index** | index lưu `email` gốc, không lưu `lower(email)` |
| **type mismatch / cast** | so sánh cột số với chuỗi → ép kiểu | cast làm điều kiện không **sargable** |
| **LIKE '%x'** (**leading wildcard**) | `WHERE name LIKE '%abc'` | B+tree không tra được khi không biết phần đầu |
| **selectivity thấp** | lọc ra phần lớn bảng | full scan rẻ hơn random I/O |
| **thống kê cũ** (stale stats) | planner ước lượng sai | cần `ANALYZE` cập nhật thống kê |
| **OR trên nhiều cột** | `WHERE a=1 OR b=2` | khó dùng một index cho cả hai nhánh |

#### 🔍 Partial & expression index  `E-IDX-008`

> **Ẩn dụ:** **partial index** = chỉ làm mục lục cho *chương đang đọc nhiều* (bỏ qua phần ít dùng) → mục lục mỏng, rẻ. **expression index** = làm mục lục theo *cách bạn hay tra* (theo chữ thường thay vì chữ gốc).

- **partial index** = index **một phần bảng** theo điều kiện **`WHERE`** (ví dụ chỉ index các dòng `status = 'open'`). Tiết kiệm dung lượng + đúng truy vấn nóng.
- **expression index** = index trên **biểu thức** (ví dụ `lower(email)`) để khớp đúng query dùng hàm đó.

> 🎯 **Chốt:** Hai loại này giúp index **đúng truy vấn thực** thay vì index tràn lan — vừa nhẹ vừa khớp.

---

## ③ ⚠️ Cũ → Mới (kiến thức bị thay thế)

| Quan niệm CŨ ❌ | Hiện nay ✅ | Vì sao |
|----------------|------------|--------|
| "Cứ thêm **index** cho mọi cột" | Index theo **query thực**; đo bằng **pg_stat_user_indexes** | Mỗi index làm **chậm ghi** + tốn dung lượng |
| Tưởng **Postgres** có **clustered index** | Postgres = **heap**; **CLUSTER** chỉ một lần | Khác **MySQL InnoDB** |
| `WHERE func(col)=...` rồi than chậm | **Expression index** hoặc viết **sargable** | Function **phá khả năng dùng index** |

---

## ④ Thực tế + bigtech  `E-IDX-009`

### Tình huống TL gỡ "12 index, ghi chậm dần"

> 🧩 Một bảng có **12 index**, ghi ngày càng chậm. TL xử lý **dựa trên dữ liệu, không cảm tính**:

1. Dùng **`pg_stat_user_indexes`** tìm index có **`idx_scan = 0`** → **không ai dùng** → ứng viên để bỏ.
2. Tìm **index redundant** — index **là prefix của một composite index khác** (đã được bao trùm → thừa). (Liên quan **Bài 5** về composite.)
3. **Đo write amplification** → bỏ index dựa trên *con số*, không phải cảm giác.

> 🔎 **Quy mô lớn:** dùng **`CREATE INDEX CONCURRENTLY`** để **không khóa bảng** khi tạo index trên production (đánh đổi: chậm hơn, không chạy trong transaction).

> 💡 Nguyên tắc: mỗi index phải *trả lời được* "query nào dùng tao?". Không trả lời được → xóa.

---

## ⑤ Code + cấu hình

```sql
-- Covering index: query chỉ cần (user_id, status, total) → index-only scan
-- INCLUDE nhét status + total_cents vào index để KHỎI đọc heap
CREATE INDEX idx_order_user ON "order" (user_id) INCLUDE (status, total_cents);

-- Partial index: chỉ index đơn hàng đang mở (status='open')
-- → index mỏng, đúng query nóng, không phí chỗ cho đơn đã đóng
CREATE INDEX idx_order_open ON "order" (created_at) WHERE status = 'open';

-- Expression index: khớp WHERE lower(email) = ...
-- → index lưu lower(email) nên query dùng hàm lower() mới dùng được index
CREATE INDEX idx_user_email_lower ON "user" (lower(email));

-- Tìm index vô dụng để bỏ: idx_scan = 0 nghĩa là CHƯA AI dùng
SELECT relname, indexrelname, idx_scan
FROM pg_stat_user_indexes
WHERE idx_scan = 0
ORDER BY relname;

-- Tạo index KHÔNG khóa bảng (production)
-- ⚠️ CONCURRENTLY: không được chạy bên trong một transaction
CREATE INDEX CONCURRENTLY idx_x ON t (col);
```

> 🚨 **AI hay sai ở đây:** AI hay sinh `WHERE func(col)=...` rồi than chậm mà không tạo **expression index**; hoặc đề xuất thêm index cho mọi cột bất kể **selectivity**; hoặc quên **`CONCURRENTLY`** khiến tạo index khóa bảng production.

---

## ⑥ Keywords

> 🧠 **HIỂU** (nắm bản chất):
> **trade-off read/write** · **selectivity** / **cardinality** · **sargable** · vì sao **B+tree** · **index-only scan**.

> 📌 **THUỘC** (nhớ chính xác):
> **Postgres không có clustered index** · **INCLUDE** = **covering** · **partial** = **WHERE** · **CONCURRENTLY** không trong transaction.

> 🛠️ **LÀM ĐƯỢC** (kỹ năng tay):
> Viết **partial** / **expression** / **covering index** đúng · dùng **pg_stat_user_indexes** để quyết định bỏ index.

---

## ⑦ Mental model

> 🎯 **NGUYÊN TẮC VÀNG CỦA BÀI:**
> *"**Index = đổi tốc độ ghi lấy tốc độ đọc**. Mỗi index phải trả lời được: **query nào dùng nó?** Không ai dùng thì xóa."*

Cách dùng: trước khi tạo index, hỏi "query cụ thể nào sẽ dùng nó?". Định kỳ chạy **`pg_stat_user_indexes`** để dọn index `idx_scan = 0`. Đừng để index "mọc hoang".

---

## ⑧ Phỏng vấn & thách đố

| Mức | Câu hỏi | Trả lời cốt lõi |
|-----|---------|-----------------|
| **(dễ)** | **Index** tăng tốc đọc thế nào? | **mục lục**, tránh **full scan** |
| **(TB)** | Vì sao index **làm chậm ghi**? | mỗi ghi cập nhật cả index (**write amplification**) |
| **(khá)** | Vì sao **B+tree** thay **hash**? | **equality** + **range** + **sort** + **prefix** |
| **(khá)** | Index trên boolean **is_active** vô dụng vì? | **selectivity thấp** |
| **(khó)** | Có index mà vẫn **full scan** — kể nguyên nhân | **function**/**cast**/**leading wildcard**/**stats cũ**/**OR** (**sargable**) |

> 🧩 **Thách đố tình huống (TL):**
> *12 index, ghi chậm — bỏ cái nào, dựa vào gì?*
>
> Trả lời chuẩn: dùng **`pg_stat_user_indexes`** (bỏ `idx_scan = 0`) + tìm **redundant prefix** (index bị composite khác bao trùm) + đo **write amplification**. Quyết định **dựa dữ liệu, không cảm tính**.

---

## ⑨ Bài tập + tiêu chí tự chấm

### BT1 — Query chậm dù đã có index

`SELECT * FROM users WHERE lower(email)='a@b.com'` chậm dù đã có index `(email)`. Hãy sửa.

> ✅ **Tiêu chí Đạt:**
> - Nhận ra index `(email)` không dùng được vì query bọc cột trong hàm `lower()` (không **sargable**).
> - Sửa bằng **expression index** `lower(email)` **hoặc** chuẩn hóa email (lưu sẵn lowercase) lúc ghi.
> - ❌ Trượt nếu chỉ "thêm index (email) nữa" mà không hiểu vì sao nó không ăn.

### BT2 — Dọn index dựa trên thống kê

Cho `pg_stat_user_indexes` với **3 index `idx_scan = 0`** và **1 index là prefix của một composite khác**. Quyết định bỏ cái nào.

> ✅ **Tiêu chí Đạt:**
> - Bỏ **3 index 0-scan** (không ai dùng) + **1 index redundant prefix** (đã bị composite bao trùm).
> - Nêu **rủi ro**: kiểm tra xem index có phục vụ **constraint** (unique/FK) phụ thuộc không trước khi xóa.
> - ❌ Trượt nếu bỏ bừa mà không kiểm tra ràng buộc phụ thuộc.

---

## ⑩ Đọc thêm

- **PostgreSQL — Indexes** (types, partial, expression, **INCLUDE**) — `postgresql.org/docs/current/indexes.html`
- **Markus Winand, *Use The Index, Luke!*** — `use-the-index-luke.com`

> 🎯 **Một câu mang về:** Index không phải càng nhiều càng tốt. Index tốt là index có *một query cụ thể cần nó*. Mọi index khác chỉ là thuế đánh lên mỗi lần ghi.
