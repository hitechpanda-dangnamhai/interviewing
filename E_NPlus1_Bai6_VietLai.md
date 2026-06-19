# 🎯 BÀI 6 — N+1 query

**Phủ:** E-NP1-001 → 006

> 💡 Đây là **lỗi performance phổ biến nhất ở tầng app/ORM** — bug mà gần như mọi backend dev đều dính ít nhất một lần. Tin tốt: phát hiện và fix nó không khó, *nếu* bạn biết nó trông như thế nào.

---

## ① Mục tiêu & vị trí

Bài này về **N+1 query** — lỗi performance số một sinh ra ở **tầng app/ORM** (lớp code/thư viện ánh xạ object ↔ bảng).

Để **fix đúng**, bạn cần kiến thức **JOIN + index** từ **Bài 4–5**. Bài này cũng liên hệ:
- **mục F** (GraphQL) — nơi N+1 đặc biệt dễ xảy ra.
- **mục G/H** (ORM trong Node/Nest) — nơi lazy loading âm thầm tạo ra N+1.

> 🎯 **Chốt định vị:** **N+1** không phải lỗi của DB — DB chạy đúng từng query. Nó là lỗi *cách app gọi DB*: thay vì hỏi một câu lớn, app hỏi N câu nhỏ trong vòng lặp.

---

## ② Giảng cơ bản → nâng cao

### (a) N+1 là gì  `E-NP1-001`

> **Ẩn dụ:** Bạn cần mua 50 món cho 50 người. Thay vì ghi một danh sách rồi đi siêu thị **một chuyến**, bạn chạy ra siêu thị **50 lần riêng lẻ**, mỗi lần mua một món. Cộng thêm chuyến đầu để lấy danh sách → **51 chuyến**. Mỗi "chuyến" là một **round-trip** tới DB.

**N+1 query** = **1 query** lấy danh sách (N phần tử) + **N query** lấy quan hệ cho *từng* phần tử = **N+1 round-trip**.

**Nguyên nhân:** **lazy loading** bị gọi *bên trong vòng lặp*:
```
for order in orders:        # 1 query lấy orders
    print(order.customer.name)   # ❌ mỗi vòng lặp lại 1 query lấy customer
```

Timeline minh họa:

```text
N+1 với 50 orders:
  T1: SELECT * FROM order WHERE user_id=1   → ra 50 dòng   ← 1 query
       ↓ (vào vòng lặp)
  T2: SELECT customer WHERE id=101          ← query #1
  T3: SELECT customer WHERE id=102          ← query #2
  ...
  T51: SELECT customer WHERE id=150         ← query #50
       ↓
  ❌ Tổng = 1 + 50 = 51 round-trip → mỗi round-trip tốn độ trễ mạng
     → endpoint chậm gấp chục lần dù dữ liệu chẳng nhiều
```

> 🚨 **Điều tệ nhất:** với danh sách lớn (N = 1000), bạn bắn **1001 query** cho một request — DB và mạng quá tải, request treo, mà nhìn code thì "trông vô hại".

---

### (b) Phát hiện (không đoán)  `E-NP1-002`

> ⚠️ **Đừng đoán** N+1 ở đâu — hãy **đo**. N+1 ẩn rất kỹ vì mỗi query lẻ đều nhanh; chỉ khi *đếm số lượng* mới lộ.

Ba cách phát hiện:

| Cách | Làm gì | Dấu hiệu N+1 |
|------|--------|--------------|
| **query logging** | bật log query + **đếm số query mỗi request** | một request bắn ra hàng chục/trăm query |
| **APM / tracing** | công cụ giám sát (theo dõi từng request) | thấy "chùm" query giống nhau lặp lại |
| **pg_stat_statements** | xem thống kê query trong Postgres | **một query lặp N lần** với tham số khác nhau |

> 💡 Dấu hiệu kinh điển trong **pg_stat_statements**: cùng một câu `SELECT ... WHERE id = $1` có số lần gọi cực cao — đó là N query trong vòng lặp.

---

### (c) Cách fix + trade-off  `E-NP1-003`

> **Ẩn dụ:** Quay lại chuyện siêu thị — thay vì 50 chuyến, bạn **gom danh sách** rồi đi *một* (hoặc *hai*) chuyến. Đó là tinh thần của mọi cách fix: **gom round-trip lại**.

| Cách fix | Số query | Ưu | Nhược |
|----------|----------|----|-------|
| **JOIN / eager load** | 1 | gọn nhất | ❌ có thể **nhân dòng** (**cartesian** khi join nhiều **one-to-many**) + **over-fetch** |
| **Batch `WHERE IN (...)`** | 2 | ✅ thường **cân bằng tốt nhất** | cần gom id rồi map lại |
| **DataLoader** | gom theo "tick" | ✅ chuẩn cho **GraphQL**, **cache per-request** | thêm một lớp thư viện |

**JOIN / eager load:** một query duy nhất, nhưng cẩn thận **nhân dòng** — khi JOIN nhiều quan hệ **one-to-many**, kết quả nở ra tổ hợp (**cartesian**) và **over-fetch** dữ liệu thừa.

**Batch `WHERE IN (...)`:** chia 2 query — một lấy danh sách, một gom hết quan hệ theo danh sách id. Thường là điểm cân bằng tốt nhất.

```text
Fix bằng batch WHERE IN — chỉ 2 round-trip:
  T1: SELECT * FROM order WHERE user_id=1        → ra 50 dòng + 50 customer_id
       ↓ (gom id lại: 101,102,...,150)
  T2: SELECT * FROM customer WHERE id IN (101..150)   ← 1 query gom hết
       ✅ Tổng = 2 round-trip (thay vì 51) → nhanh hơn ~25 lần
```

**DataLoader:** gom các request lẻ phát sinh trong cùng một "**tick**" (vòng sự kiện) thành một batch, kèm **cache per-request** (trong một request, hỏi cùng id hai lần chỉ tốn một query). Đây là **pattern chuẩn cho GraphQL**.

---

### (d) Mép giới hạn

#### 🔍 Eager load tất cả "để tránh N+1" cũng hại  `E-NP1-004`

> ⚠️ Đừng đu sang thái cực ngược: **eager load mọi thứ** để "diệt N+1" lại sinh lỗi khác.

- ❌ **over-fetch** — kéo về dữ liệu không bao giờ dùng.
- ❌ **JOIN nặng / duplicate rows** — nhân dòng tốn băng thông + xử lý.
- ❌ **ngốn RAM** — gom hết vào bộ nhớ.

> 🎯 **Chốt:** **lazy tốt hơn khi quan hệ ít khi cần.** Nếu 90% request không đụng tới `order.customer`, eager load nó mỗi lần là lãng phí.

#### 🔍 GraphQL dễ dính N+1  `E-NP1-005`

> **Ví dụ trực giác:** GraphQL cho client hỏi dữ liệu lồng nhau ("posts → author → avatar"). Mỗi tầng lồng (**resolver**) tự gọi DB cho *từng node* → N+1 nở theo từng tầng.

**resolver lồng theo field** gọi DB cho từng node → **pattern chuẩn là DataLoader** (**batch** + **cache per-request**).

#### 🔍 JOIN lớn vs nhiều query nhỏ  `E-NP1-006`

> 🚨 **Đừng giáo điều "JOIN luôn tốt".** Đôi khi **nhiều query nhỏ NHANH hơn** một JOIN lớn.

| Tiêu chí | **JOIN lớn** | **Nhiều query nhỏ** |
|----------|--------------|---------------------|
| Số round-trip | ít (1) | nhiều |
| Nhân dòng | ❌ **nhân dòng** (cartesian) | ✅ không |
| Khả năng **cache** | ❌ khó cache cả kết quả JOIN | ✅ từng query nhỏ dễ cache |
| **Lock** | ❌ giữ **lock rộng** hơn | ✅ lock hẹp |
| Tận dụng **index** | tùy | ✅ mỗi query nhỏ ăn index riêng |
| Lượng data truyền | có thể nhiều (duplicate) | ✅ truyền ít |

> 🎯 **Chốt:** **Phải đo, không giáo điều.** Có trường hợp JOIN thắng, có trường hợp batch nhỏ thắng — số đo trên dữ liệu thật mới quyết định.

---

## ③ ⚠️ Cũ → Mới (kiến thức bị thay thế)

| Quan niệm CŨ ❌ | Hiện nay ✅ | Vì sao |
|----------------|------------|--------|
| "**Eager load** mọi thứ để hết **N+1**" | **Eager** đúng quan hệ cần; **batch IN**/**DataLoader** | Eager bừa gây **over-fetch** + **nhân dòng** |
| "**JOIN** luôn nhanh hơn nhiều query" | **Đo** từng case; **batch nhỏ** có khi nhanh hơn | JOIN **nhân dòng**, **khó cache** |

---

## ④ Thực tế + bigtech

> 🔎 **Pattern ngành (verify):**
> - **GraphQL ở quy mô lớn** (Facebook khai sinh **DataLoader**): **batch** + **cache per-request** là **mặc định**.
> - **REST API** thường fix bằng **eager**/**IN** ở **tầng repository**.
> - **Nguyên tắc TL:** **đặt ngân sách số query/request** và **alert khi vượt** — biến N+1 từ "bug ẩn" thành "có cảnh báo tự động".

---

## ⑤ Code + cấu hình

```sql
-- N+1 (XẤU): 1 query orders + N query customer
SELECT * FROM "order" WHERE user_id = 1;          -- 1 query lấy list
SELECT * FROM customer WHERE id = ?;              -- ❌ lặp N lần trong vòng lặp app

-- FIX batch IN: 2 query, gom quan hệ theo danh sách id
SELECT * FROM "order" WHERE user_id = 1;          -- 1 (lấy list + thu thập customer_id)
SELECT * FROM customer WHERE id IN (101,102,...); -- 1 (gom hết một lần)

-- FIX JOIN eager: 1 query (⚠️ cẩn thận nhân dòng nếu nhiều one-to-many)
SELECT o.*, c.name
FROM "order" o JOIN customer c ON c.id = o.customer_id
WHERE o.user_id = 1;
```

```javascript
// GraphQL: DataLoader batch + cache per-request (verify dataloader version)
const customerLoader = new DataLoader(async (ids) => {
  // ids = mảng id được GOM trong cùng một tick → chỉ 1 query cho cả batch
  const rows = await db.query('SELECT * FROM customer WHERE id = ANY($1)', [ids]);
  // ⚠️ phải trả về ĐÚNG THỨ TỰ ids đầu vào (DataLoader yêu cầu)
  return ids.map(id => rows.find(r => r.id === id));
});
```

> 🚨 **AI hay sai ở đây:** AI hay "fix" N+1 bằng cách **eager load mọi quan hệ** (gây over-fetch + nhân dòng), hoặc gom thành **một JOIN khổng lồ** mà không đo; hoặc viết **DataLoader quên giữ đúng thứ tự** id (làm map sai dữ liệu).

---

## ⑥ Keywords

> 🧠 **HIỂU** (nắm bản chất):
> **lazy vs eager loading** · vì sao **JOIN nhân dòng** · **DataLoader** **batch** theo **tick**.

> 📌 **THUỘC** (nhớ chính xác):
> **N+1 = 1 + N round-trip** · **GraphQL → DataLoader**.

> 🛠️ **LÀM ĐƯỢC** (kỹ năng tay):
> Phát hiện **N+1** bằng **query count**/log; fix bằng **IN**/**eager**/**DataLoader**; biết khi nào **nhiều query nhỏ tốt hơn**.

---

## ⑦ Mental model

> 🎯 **NGUYÊN TẮC VÀNG CỦA BÀI:**
> *"**N+1 = vòng lặp gọi DB**. **Gom lại** (**IN**/**JOIN**/**DataLoader**). Nhưng gom thành **JOIN khổng lồ** cũng là lỗi — **đo, đừng đoán**."*

Cách dùng: thấy code gọi DB *bên trong* một vòng lặp → cảnh giác ngay. Gom round-trip lại, rồi *đo* lại số query để xác nhận đã hết N+1 mà không tạo JOIN quá nặng.

---

## ⑧ Phỏng vấn & thách đố

| Mức | Câu hỏi | Trả lời cốt lõi |
|-----|---------|-----------------|
| **(dễ)** | **N+1** là gì, ví dụ ORM? | 1 list + N quan hệ do **lazy** trong loop |
| **(TB)** | Phát hiện trong production thế nào? | đếm **query**/log/**APM**/**pg_stat_statements** |
| **(khá)** | Các cách fix + trade-off? | **JOIN**(nhân dòng)/**IN**(cân bằng)/**DataLoader**(GraphQL) |
| **(khá)** | **Eager** hết có hại gì? | **over-fetch**, **nhân dòng**, **RAM** |

> 🧩 **Thách đố tình huống:**
> *Khi nào nhiều query nhỏ NHANH hơn một JOIN lớn?*
>
> Trả lời chuẩn: khi **JOIN nhân dòng** / **khó cache** / **giữ lock rộng**; còn **query nhỏ** tận dụng **index** + **cache** + truyền ít data.
> ⚠️ **Bẫy phải vạch ra:** **không giáo điều "JOIN luôn tốt"** — phải đo từng case.

---

## ⑨ Bài tập + tiêu chí tự chấm

### BT1 — Endpoint 50 bài viết + tên tác giả, log thấy 51 query

Fix và đo lại.

> ✅ **Tiêu chí Đạt:**
> - Giảm còn **2 query** (**batch IN**) hoặc **1 query** (**JOIN**).
> - Giải thích **trade-off** đã chọn (IN cân bằng / JOIN gọn nhưng coi chừng nhân dòng).
> - ❌ Trượt nếu chỉ "thêm cache" mà không gom round-trip.

### BT2 — GraphQL resolver `posts → author → avatar` chậm

Đề xuất giải pháp.

> ✅ **Tiêu chí Đạt:**
> - Dùng **DataLoader** cho `author` và `avatar`, **batch per-request**.
> - Giải thích vì sao resolver lồng gây N+1 và DataLoader gom lại.
> - ❌ Trượt nếu đề xuất eager load toàn bộ graph mà không xét over-fetch.

---

## ⑩ Đọc thêm

- **DataLoader (GitHub)** — `github.com/graphql/dataloader`
- **ORM docs** phần **eager/lazy loading** (Prisma / TypeORM / SQLAlchemy — verify)

> 🎯 **Một câu mang về:** Thấy `for ... in ...` mà bên trong có gọi DB — dừng lại. Đó là 95% các ca **N+1**. Gom round-trip lại, rồi *đếm query* để chắc chắn mình đã thắng.
