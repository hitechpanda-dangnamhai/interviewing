# 🎯 BÀI 10 — Query optimization tools

**Phủ:** E-OPT-001 → 007

> 💡 Đây là **bộ công cụ "đo trước, sửa sau"** — vũ khí chống lại thói "đoán mò". Bài 4–5 dạy *tạo* index; bài này dạy *biết chắc* query nào chậm, vì sao chậm, và sửa cái gì. Đây là **kỹ năng debug performance hằng ngày** của backend.

---

## ① Mục tiêu & vị trí

Bài này về **bộ công cụ đo trước, sửa sau** — **chống "đoán mò"**.

Cần kiến thức nền: **index** (**Bài 4–5**), **scan types** và **statistics** (**Bài 9**).

> 🎯 **Chốt định vị:** Tối ưu query mà không đo giống chữa bệnh không khám — bạn có thể "thêm index" vô ích trong khi thủ phạm thật là **stats cũ**. **Đo, đừng đoán** là toàn bộ tinh thần của bài.

---

## ② Giảng cơ bản → nâng cao

### (a) Quy trình debug query chậm  `E-OPT-001`

> **Ẩn dụ:** Sửa query chậm như bác sĩ khám bệnh: *xét nghiệm* (tìm query chậm) → *chụp X-quang* (EXPLAIN ANALYZE) → *đọc kết quả* (scan/join/sort) → *kê đơn* (thêm index/sửa query). Không ai kê thuốc trước khi khám.

Quy trình chuẩn (4 bước, **không đoán mò**):

```text
1. TÌM query chậm     → pg_stat_statements / slow log
        ↓
2. EXPLAIN ANALYZE    → chạy thật để đo
        ↓
3. ĐỌC plan           → scan types / join / sort
        ↓
4. SỬA                → thêm index / viết lại query
```

---

### (b) EXPLAIN vs EXPLAIN ANALYZE  `E-OPT-002`

> **Ví dụ trực giác:** **EXPLAIN** = *đọc bản đồ* dự đoán đường đi (không thật sự lái). **EXPLAIN ANALYZE** = *lái thử thật* và bấm giờ từng chặng. Cái sau cho số liệu thực, nhưng vì *chạy thật* nên nguy hiểm với câu ghi.

| Tiêu chí | **EXPLAIN** | **EXPLAIN ANALYZE** |
|----------|-------------|----------------------|
| Có chạy query không? | ❌ chỉ **ước lượng** | ✅ **chạy thật** để đo **actual** |
| Cho số liệu | dự đoán của planner | thời gian/rows thực tế |
| Nguy hiểm với INSERT/UPDATE/DELETE? | không | 🚨 **CÓ — đổi data thật** |

> 🚨 **CẢNH BÁO:** với **INSERT/UPDATE/DELETE**, **EXPLAIN ANALYZE chạy thật** nên *thực sự thay đổi dữ liệu*. **Phải bọc transaction + ROLLBACK**:

```text
EXPLAIN ANALYZE trên câu ghi — vì sao phải ROLLBACK:
  T1: BEGIN
  T2: EXPLAIN ANALYZE UPDATE ... → ❌ câu UPDATE CHẠY THẬT → data đã đổi
       ↓
  T3: ROLLBACK → ✅ hủy thay đổi → data trở lại như cũ (chỉ giữ lại số đo)
```

---

### (c) Đọc scan types  `E-OPT-003`

> **Ví dụ trực giác:** 4 kiểu **scan** là 4 cách tìm sách trong thư viện — từ "lật từng cuốn" đến "tra mục lục và lấy được luôn".

| Scan type | Cách hoạt động | Khi nào tốt |
|-----------|----------------|-------------|
| **Seq Scan** | **quét cả bảng** | bảng nhỏ / lấy phần lớn dòng |
| **Index Scan** | tra index rồi **lấy từng row từ heap** | lấy ít dòng |
| **Bitmap Index Scan** | **gom nhiều row** / **kết hợp nhiều index** trước khi đọc heap | lấy số dòng trung bình / nhiều điều kiện |
| **Index Only Scan** | đủ cột trong index → **không chạm heap** | có **covering** (Bài 4) + **visibility map** (Bài 9) |

> 🔎 **Index Only Scan** nhanh nhất vì *khỏi đọc heap* — nhưng cần **covering index** và **visibility map** được cập nhật (tức bảng đã được **VACUUM** đủ, Bài 9).

---

### (d) Mép giới hạn

#### 🔍 Estimated vs actual rows lệch lớn  `E-OPT-004`

> ⚠️ Đây là **dấu hiệu xấu** quan trọng nhất khi đọc plan: số **planner estimate** lệch xa số **actual**.

**Vì sao nguy hiểm?** Vì planner **chọn plan dựa estimate**. Estimate sai → **chọn join sai** (ví dụ **nested loop** khi đáng ra nên **hash join**).

```text
Estimate lệch actual → planner chọn nhầm join:
  Plan: Nested Loop  (estimate rows=2)  (actual rows=900.000)
        ↑ planner tưởng chỉ 2 dòng → chọn nested loop (hợp cho dữ liệu nhỏ)
        nhưng thực tế 900k dòng → nested loop lặp 900k lần   ❌ chậm khủng khiếp
  Nguyên nhân thường gặp: stats cũ HOẶC correlation giữa cột
  ✅ sửa: ANALYZE để cập nhật stats
```

#### 🔍 pg_stat_statements  `E-OPT-005`

> 🚨 **Bẫy tư duy kinh điển:** đừng chỉ tìm query *chậm nhất một lần*. Hãy tìm query tốn **TỔNG thời gian** nhiều nhất.

**pg_stat_statements** **gộp query theo mẫu** và cho biết **total_time**, **calls**, **mean_time**.

> **Ví dụ trực giác:** Một query chạy **2ms** nhưng gọi **1 triệu lần** = **2000 giây** tổng. Một query chạy **5 giây** nhưng gọi **1 lần** = 5 giây. Thủ phạm tải DB thật là cái đầu, dù nhìn riêng lẻ nó "nhanh". **Một query nhanh gọi triệu lần > một query chậm gọi 1 lần.**

#### 🔍 Join algorithms  `E-OPT-006`

| Join | Tốt khi | Vì sao |
|------|---------|--------|
| **nested loop** | **một bên nhỏ** + **có index** | lặp bên nhỏ, tra index bên kia |
| **hash join** | **bảng lớn**, **không sort** | dựng hash table rồi dò |
| **merge join** | **cả hai đã sort** | trộn hai dòng đã thứ tự |

> 💡 **Planner chọn** theo **kích thước** + **index** + **đã sort chưa**. Không có "join tốt nhất" tuyệt đối — tùy hình dạng dữ liệu.

---

## ③ ⚠️ Cũ → Mới (kiến thức bị thay thế)

| Quan niệm CŨ ❌ | Hiện nay ✅ | Vì sao |
|----------------|------------|--------|
| Đoán "chắc thiếu index" | **EXPLAIN ANALYZE** + **pg_stat_statements** rồi mới sửa | **Đo, đừng đoán** |
| Chỉ nhìn query **chậm nhất 1 lần** | Nhìn **TỔNG thời gian** (**calls × mean**) | Thủ phạm tải thật thường là query "vừa" gọi nhiều |
| **EXPLAIN ANALYZE** thẳng câu **UPDATE** | Bọc **`BEGIN; ... ROLLBACK;`** | ANALYZE **chạy thật**, đổi data |

---

## ④ Thực tế + bigtech

### Query "bỗng chậm" dù code/data không đổi nhiều  `E-OPT-007`

> 🧩 Tình huống kinh điển: code không đổi, dữ liệu không tăng mấy, mà query bỗng chậm. Danh sách **nghi phạm**:

| Nghi phạm | Cách xác minh |
|-----------|---------------|
| **stats cũ** | chạy **ANALYZE** (Bài 9) |
| **bloat** | xem `n_dead_tup` (Bài 9) |
| **plan flip** do data lớn dần | so **EXPLAIN** trước/sau |
| **thiếu index** sau khi data tăng | đọc plan thấy **seq scan** |
| **cache lạnh** (cache trống sau restart) | quan sát lần chạy thứ 2 |
| **lock/contention** | xem các tx đang chờ |

> 🔎 **Production thường bật:** **pg_stat_statements** + **auto_explain** + **dashboard slow query** — biến "đoán" thành "có dữ liệu".

---

## ⑤ Code + cấu hình

```sql
-- Bật & dùng pg_stat_statements (cần shared_preload_libraries) -- verify
-- Sắp theo TỔNG thời gian (total_exec_time), KHÔNG sắp theo mean
SELECT query, calls, total_exec_time, mean_exec_time
FROM pg_stat_statements
ORDER BY total_exec_time DESC      -- thủ phạm tải thật lên đầu
LIMIT 10;

-- EXPLAIN ANALYZE AN TOÀN cho câu ghi: bọc BEGIN ... ROLLBACK
BEGIN;
  EXPLAIN (ANALYZE, BUFFERS) UPDATE "order" SET status='x' WHERE id=1;
ROLLBACK;   -- ✅ không đổi data thật, chỉ giữ số đo

-- Đọc plan: chú ý CHÊNH LỆCH rows giữa estimate và actual
-- "Seq Scan ... (rows=1000000) (actual rows=12)" → estimate sai → chạy ANALYZE
```

> 🚨 **AI hay sai ở đây:** AI hay **đoán "thêm index"** mà không **EXPLAIN ANALYZE** trước; hoặc chạy **EXPLAIN ANALYZE thẳng câu UPDATE** (đổi data thật); hoặc sắp **pg_stat_statements theo mean_time** thay vì **total_exec_time** (bỏ sót thủ phạm tải thật).

---

## ⑥ Keywords

> 🧠 **HIỂU** (nắm bản chất):
> **estimate vs actual rows** · vì sao nhìn **TỔNG thời gian** · khi nào **nested**/**hash**/**merge join**.

> 📌 **THUỘC** (nhớ chính xác):
> **EXPLAIN** (ước lượng) vs **ANALYZE** (chạy thật) · **4 scan types** · **ROLLBACK quanh ANALYZE câu ghi**.

> 🛠️ **LÀM ĐƯỢC** (kỹ năng tay):
> Đọc một **plan**, chỉ ra **bottleneck**, đề xuất **index**/sửa query · dùng **pg_stat_statements** tìm thủ phạm.

---

## ⑦ Mental model

> 🎯 **NGUYÊN TẮC VÀNG CỦA BÀI:**
> *"**Đo trước, sửa sau**: **pg_stat_statements** tìm thủ phạm theo **TỔNG thời gian** → **EXPLAIN ANALYZE** đọc plan → **estimate lệch actual = stats cũ**."*

Cách dùng: khi DB chậm, đừng vội thêm index. Mở **pg_stat_statements** sắp theo **total_exec_time** → lấy top query → **EXPLAIN ANALYZE** nó → nếu **estimate lệch actual** thì nghi **stats cũ** trước tiên.

---

## ⑧ Phỏng vấn & thách đố

| Mức | Câu hỏi | Trả lời cốt lõi |
|-----|---------|-----------------|
| **(TB)** | Quy trình debug query chậm? | tìm → **EXPLAIN ANALYZE** → đọc → sửa |
| **(khá)** | **EXPLAIN** vs **ANALYZE** + vì sao nguy hiểm câu ghi? | ước lượng vs **chạy thật**; **ROLLBACK** |
| **(khá)** | 4 **scan types**? | **seq**/**index**/**bitmap**/**index-only** |
| **(khó)** | **Estimate vs actual** lệch lớn nghĩa gì? | **plan sai** do **stats**/**correlation** |
| **(TB)** | **pg_stat_statements** tìm thủ phạm thế nào? | **TỔNG thời gian** |

> 🧩 **Thách đố tình huống:**
> *Query bỗng chậm dù không đổi gì — nghi phạm?*
>
> Trả lời chuẩn: **stats cũ** / **bloat** / **plan flip** / **cache lạnh** / **lock**. Xác minh bằng **EXPLAIN** + **pg_stat_***.

---

## ⑨ Bài tập + tiêu chí tự chấm

### BT1 — Plan có Seq Scan (est=2, actual=900k) + sort tốn

Chẩn đoán + sửa.

> ✅ **Tiêu chí Đạt:**
> - Nhận ra **estimate lệch actual khủng** → nghi **stats cũ** → **ANALYZE**.
> - Cân nhắc **index khớp WHERE + ORDER BY** (composite — Bài 5) để né sort.
> - ❌ Trượt nếu chỉ "thêm index" mà bỏ qua dấu hiệu estimate lệch (stats).

### BT2 — Tìm top query tải DB trong pg_stat_statements

> ✅ **Tiêu chí Đạt:**
> - **Sort theo `total_exec_time`** (không theo `mean_time`).
> - Giải thích vì sao: query "vừa" gọi triệu lần tải DB nhiều hơn query chậm gọi 1 lần.
> - ❌ Trượt nếu sort theo mean và bỏ sót thủ phạm thật.

---

## ⑩ Đọc thêm

- **PostgreSQL — Using EXPLAIN** — `postgresql.org/docs/current/using-explain.html`
- **pg_stat_statements** — `.../pgstatstatements.html`
- **explain.depesz.com / explain.dalibo.com** (visualize plan)

> 🎯 **Một câu mang về:** Người mới *đoán* "chắc thiếu index". Senior *đo*: mở **pg_stat_statements** theo **tổng thời gian**, chạy **EXPLAIN ANALYZE**, đọc chênh lệch **estimate/actual**. Dữ liệu chỉ đường, không phải linh cảm.
