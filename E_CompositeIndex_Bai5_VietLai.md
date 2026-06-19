# 🎯 BÀI 5 — Composite index

**Phủ:** E-CIDX-001 → 005

> 💡 Bài 4 dạy index một cột. Bài này dạy **composite index** (index nhiều cột) — và đây chính là kiến thức phân biệt người *"biết tạo index"* với người *"tạo ĐÚNG index"*. Trái tim của bài: **leftmost prefix** và **thứ tự cột**.

---

## ① Mục tiêu & vị trí

Bài này **xây thẳng trên Bài 4**. Khi đã hiểu index một cột, bước tiếp theo là index nhiều cột — **composite index**.

Hai khái niệm cốt lõi phải nắm:
- **leftmost prefix** — quy tắc "đọc từ trái sang" quyết định query nào dùng được index.
- **thứ tự cột** — sắp sai thứ tự thì index gần như vô dụng.

> 🔎 Đây là ranh giới giữa hai trình độ: người mới tạo index "có cột là được"; người chín tạo index *đúng thứ tự cho đúng query*.

Vị trí: là **nền cho thiết kế bộ index tối thiểu** ở **Bài 10**.

> 🎯 **Chốt định vị:** Một **composite index** sai thứ tự cột tốn y hệt dung lượng và thuế ghi như index đúng — nhưng query không dùng được. Thứ tự cột là *tất cả*.

---

## ② Giảng cơ bản → nâng cao

### (a) Leftmost prefix  `E-CIDX-001`

> **Ẩn dụ:** **composite index** `(a, b, c)` giống **danh bạ điện thoại sắp theo (Họ, Tên đệm, Tên)**. Bạn tra được nhanh khi biết *Họ* (→ rồi Họ+Tên đệm → rồi đủ cả ba). Nhưng nếu chỉ biết mỗi *Tên* mà không biết Họ → danh bạ vô dụng, phải lật từng trang. Đó chính là **leftmost prefix**: phải dùng từ *cột trái nhất* trở đi.

**leftmost prefix** = index `(a, b, c)` **chỉ dùng được khi query lọc từ trái sang**:

| Query lọc theo | Dùng được index `(a,b,c)`? |
|----------------|----------------------------|
| `a` | ✅ (đúng cột trái nhất) |
| `a` và `b` | ✅ (prefix liền từ trái) |
| `a`, `b`, `c` | ✅ (đủ cả ba) |
| **chỉ `b`** | ❌ Bỏ qua cột `a` ở trái → KHÔNG tận dụng được |
| **chỉ `c`** | ❌ Tương tự |
| `b` và `c` (không có `a`) | ❌ Thiếu cột trái nhất |

> 🚨 **`(a,b) ≠ (b,a)`** — thứ tự cột **không** đối xứng. Index `(a,b)` và index `(b,a)` phục vụ những query *hoàn toàn khác nhau*.

**Vì sao phải từ trái sang?** Vì dữ liệu trong index được sắp xếp theo `a` *trước*, rồi trong mỗi nhóm `a` mới sắp theo `b`. Nếu bạn không lọc `a`, các giá trị `b` nằm *rải rác khắp nơi* (mỗi nhóm `a` có lại `b` từ đầu) → không tra liền mạch được.

```text
Index (a,b,c) được sắp như sau:
  a=1 → b=10 → c=...
  a=1 → b=20 → c=...
  a=2 → b=10 → c=...   ← chú ý: b=10 XUẤT HIỆN LẠI ở nhóm a=2
  a=2 → b=30 → c=...

Query WHERE b=10:
  ❌ b=10 nằm rải ở cả nhóm a=1 VÀ a=2 → không tra một dải liền → full scan
Query WHERE a=1:
  ✅ tất cả a=1 nằm liền nhau ở đầu → tra một phát ra hết
```

---

### (b) Equality + order  `E-CIDX-002`

> **Ví dụ trực giác:** Trong danh bạ sắp theo (Họ, Tên), nếu bạn cần "tất cả người họ Nguyễn, xếp theo Tên" → bạn lật tới khúc họ Nguyễn là thấy *Tên đã sẵn thứ tự* rồi, khỏi phải sắp lại.

Index `(a, b)` phục vụ **`WHERE a=? ORDER BY b`** *rất tốt*. **Vì sao quý?** Vì trong mỗi nhóm `a`, các dòng đã **sẵn thứ tự theo `b`** → DB **bỏ được bước sort riêng** — và **sort rất đắt** (phải gom hết kết quả rồi sắp).

Timeline so sánh:

```text
KHÔNG có index (a,b) phù hợp:
  T1: lọc a=5 → ra 10.000 dòng
       ↓
  T2: ❌ phải SORT 10.000 dòng theo b (tốn CPU + bộ nhớ tạm)
  T3: trả kết quả

CÓ index (a, b) — equality + order:
  T1: nhảy tới nhóm a=5 trong index
       ✅ các dòng ĐÃ sẵn thứ tự theo b → đọc lần lượt là xong
       (KHỎI bước sort → nhanh hơn nhiều)
```

> 🎯 **Chốt:** Đây là lý do **composite index quý hơn** vài index đơn: nó vừa **lọc** (equality) vừa **phục vụ sort** trong một cấu trúc.

---

### (c) Thứ tự cột  `E-CIDX-003`

> ⭐ **Nguyên tắc vàng về thứ tự:** **equality đặt trước, range/order đặt sau.**

**Vì sao?** Vì một cột dùng **range** (`>`, `<`, `BETWEEN`) sẽ **cắt khả năng dùng các cột đứng SAU nó** trong index. Sau khi đã quét một *dải* (range), các cột phía sau không còn nằm liền thứ tự nữa.

> **Ví dụ trực giác:** Danh bạ sắp theo (Tuổi, Tên). Bạn hỏi "người 20–30 tuổi, sắp theo Tên" → trong dải tuổi 20–30, Tên *không* còn liền thứ tự (mỗi tuổi lại có Tên từ đầu). Range trên Tuổi đã "phá" thứ tự của Tên phía sau.

Ví dụ cụ thể:
```
Query: WHERE status = ? AND created_at > ?
       (status = equality, created_at = range)
✅ Index đúng: (status, created_at)   ← equality TRƯỚC
❌ Index sai:  (created_at, status)   ← range trước sẽ cắt status phía sau
```

> 🔎 Quy tắc đầy đủ: **các cột equality lên đầu** (theo thứ tự bất kỳ giữa chúng), rồi **đúng MỘT cột range hoặc cột để sort ở cuối**.

---

### (d) Nhiều index đơn vs một composite  `E-CIDX-004`

> **Ẩn dụ:** Có hai cuốn danh bạ riêng — một sắp theo Họ, một sắp theo Nghề — không bằng *một cuốn* sắp theo (Họ, Nghề) khi bạn cần lọc cả hai cùng lúc. Ghép hai cuốn lại để tìm giao điểm thì chậm hơn tra thẳng một cuốn đúng.

**`(a)` + `(b)` KHÔNG tương đương `(a,b)`.**

| Cách | Cách DB làm | Đánh giá |
|------|-------------|----------|
| Hai index đơn `(a)` + `(b)` | **bitmap-AND** hai index rồi giao kết quả | ✅ Có dùng được · ❌ Thường **kém hơn** một composite phù hợp |
| Một composite `(a,b)` | tra thẳng theo cả hai cột | ✅ Nhanh hơn cho query lọc cả `a` và `b` |

**Vì sao bitmap-AND kém hơn?** Vì DB phải quét *hai* index, dựng hai tập kết quả, rồi giao chúng — nhiều bước hơn so với tra một composite đã sắp đúng.

---

## ③ ⚠️ Cũ → Mới (kiến thức bị thay thế)

| Quan niệm CŨ ❌ | Hiện nay ✅ | Vì sao |
|----------------|------------|--------|
| Tạo **nhiều index đơn** cho mọi cột | Một **composite** đúng thứ tự cho **query thực** | Composite phục vụ **equality + order**, gọn hơn |
| Đặt **range column trước** | **Equality trước, range/order sau** | **Range cắt cột phía sau** |

---

## ④ Thực tế + bigtech

### Thiết kế bộ index tối thiểu  `E-CIDX-005`

> **Ví dụ trực giác:** Đừng in 10 cuốn danh bạ chồng chéo. In *ít cuốn nhất* mà vẫn phục vụ được mọi cách tra thường dùng.

Quy trình thiết kế **bộ index tối thiểu**:
1. **Gom các query theo leftmost prefix** — query nào chia sẻ được cùng một composite thì gộp.
2. Dùng **covering** / **`INCLUDE`** (xem Bài 4) để né bước đọc heap.
3. **Loại index là prefix của index khác** — `(a)` **thừa** nếu đã có `(a,b)` (vì `(a,b)` đã phủ luôn query chỉ-lọc-`a` qua leftmost prefix).

> 🎯 **Chốt:** Tư duy **tối ưu cả TẬP index**, không phải từng index lẻ. **Mỗi index thừa là một khoản thuế ghi vĩnh viễn** (nhớ **write amplification** ở Bài 4).

---

## ⑤ Code + cấu hình

```sql
-- Query nóng: WHERE tenant_id = ? AND status = ? ORDER BY created_at DESC
-- Thứ tự đúng: equality (tenant_id, status) TRƯỚC, cột sort (created_at) SAU
CREATE INDEX idx_order_hot
  ON "order" (tenant_id, status, created_at DESC);
-- → phục vụ CẢ lọc (tenant_id, status) LẪN sort (created_at) trong một index
--   nên KHỎI cần bước sort riêng

-- (a) thừa nếu đã có (a,b): leftmost prefix của (tenant_id, status, ...)
-- đã phủ query chỉ-lọc-tenant_id rồi
-- => KHÔNG tạo thêm idx(tenant_id) khi đã có idx(tenant_id, status, ...)
```

> 🚨 **AI hay sai ở đây:** AI hay sinh **nhiều index đơn** cho từng cột (rồi để DB bitmap-AND), hoặc **đặt range/sort lên trước equality**, hoặc tạo thêm index `(a)` đã thừa vì bị `(a,b)` phủ — tất cả đều tốn thuế ghi mà không thêm tốc độ.

---

## ⑥ Keywords

> 🧠 **HIỂU** (nắm bản chất):
> **leftmost prefix** · vì sao **equality-trước-range-sau** · **bitmap-AND** vs **composite**.

> 📌 **THUỘC** (nhớ chính xác):
> `(a,b)` phục vụ `a` và `(a,b)`, **không phục vụ riêng `b`** · **`(a,b) ≠ (b,a)`**.

> 🛠️ **LÀM ĐƯỢC** (kỹ năng tay):
> Cho 3–4 query → thiết kế **bộ index tối thiểu** không thừa.

---

## ⑦ Mental model

> 🎯 **NGUYÊN TẮC VÀNG CỦA BÀI:**
> *"**Composite index đọc từ trái sang**. **Equality trước, range/sort sau**. Index nào là **prefix của index khác** thì **thừa**."*

Cách dùng: cầm một query lên, hỏi ba câu — *cột nào equality? cột nào range/sort? thứ tự đã equality-trước chưa?* Rồi kiểm tra index mới có bị composite cũ phủ không trước khi tạo.

---

## ⑧ Phỏng vấn & thách đố

| Mức | Câu hỏi | Trả lời cốt lõi |
|-----|---------|-----------------|
| **(TB)** | **leftmost prefix** là gì? `(a,b,c)` phục vụ query nào? | lọc từ trái: `a`, `(a,b)`, `(a,b,c)` |
| **(khá)** | `(a,b)` có phục vụ `WHERE a=? ORDER BY b`? Vì sao quý? | có — **bỏ bước sort** |
| **(khá)** | Sắp thứ tự cột theo nguyên tắc nào? | **equality trước, range sau** |
| **(khó)** | `(a)` + `(b)` có bằng `(a,b)`? | không; **bitmap-AND** kém hơn composite |

> 🧩 **Thách đố tình huống:**
> *Cho 4 query, hãy thiết kế bộ index tối thiểu.*
>
> Trả lời chuẩn: **gom theo leftmost prefix** (query nào dùng chung được một composite thì gộp) + **bỏ redundant** (index là prefix của index khác). Mục tiêu: ít index nhất mà vẫn phủ hết.

---

## ⑨ Bài tập + tiêu chí tự chấm

### BT1 — Thiết kế index cho query lọc + range + sort

`WHERE user_id=? AND created_at BETWEEN ? AND ? ORDER BY created_at`. Thiết kế index.

> ✅ **Tiêu chí Đạt:**
> - Index **`(user_id, created_at)`** — **equality (`user_id`) trước**, **range/sort (`created_at`) sau**.
> - Giải thích được vì sao không đảo thứ tự (range trước sẽ cắt `user_id`).
> - ❌ Trượt nếu đặt `created_at` lên trước hoặc tạo hai index đơn rời.

### BT2 — Tìm index thừa

Bảng có các index: `(a)`, `(a,b)`, `(a,b,c)`, `(b)`. Cái nào **thừa**?

> ✅ **Tiêu chí Đạt:**
> - **`(a)` và `(a,b)` là prefix của `(a,b,c)`** → cân nhắc bỏ (trừ khi cần **covering** hoặc **sort khác**).
> - **`(b)` giữ lại** nếu có query lọc riêng `b` (vì `(a,b,c)` không phục vụ query chỉ-lọc-`b` do leftmost prefix).
> - ❌ Trượt nếu bỏ luôn `(b)` mà không xét query lọc riêng `b`.

---

## ⑩ Đọc thêm

- **PostgreSQL — Multicolumn Indexes** — `postgresql.org/docs/current/indexes-multicolumn.html`
- **Use The Index, Luke!** — chương **"The Where Clause"**

> 🎯 **Một câu mang về:** Tạo composite index không khó — khó là *sắp đúng thứ tự cột*. Nhớ ba chữ: **trái sang phải, equality trước, range/sort sau** — và bạn đã hơn phần lớn người dùng SQL.
