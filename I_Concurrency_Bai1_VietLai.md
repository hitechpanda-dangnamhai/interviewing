# 🎯 BÀI 1 — Race condition & 4 chiến lược chống race

**Phủ:** I-RACE-001 → I-RACE-009 · Phase 8 (LLMOps/Production — tư duy backend), nhưng đây là **nền móng cho mọi mục I**.

> 💡 **Cách đọc bài này.** Đừng vội học thuộc câu lệnh. Hãy nắm cho được một câu duy nhất: *race condition sinh ra ở **khoảng trống giữa lúc ĐỌC và lúc GHI***. Khi bạn "thấy" được khoảng trống đó trong đầu, mọi cách chống race còn lại chỉ là các cách bịt cái khoảng trống ấy theo những kiểu khác nhau.

---

## ① Mục tiêu & vị trí trong mạch

Học xong bài này bạn sẽ làm được 3 việc:

1. **Định nghĩa** được **race condition** *trên shared data* (dữ liệu chia sẻ như DB/Redis) — và phân biệt nó với race kiểu *event-loop*. Hai thứ này hay bị gộp làm một, nhưng bản chất khác nhau.
2. **Giải thích** được vì sao **Node single-thread** (chạy một luồng) mà *vẫn* dính race — đây là câu hỏi phỏng vấn kinh điển khiến nhiều người ngã.
3. **Chọn đúng 1 trong 4 chiến lược chống race** cho từng tình huống cụ thể, thay vì áp một cách máy móc cho mọi chỗ.

> 🎯 **Vị trí trong mạch học.** Đây là **bài gốc** của cả mục I. Ba bài sau là ba *phạm vi khác nhau* của **cùng một bài toán** này, không phải ba chủ đề rời rạc:
> - **Bài 2 (lock)** — khi atomic một-câu-lệnh không đủ, ta cần khóa tường minh (**pessimistic/optimistic lock**).
> - **Bài 3 (idempotency)** — khi vấn đề là *request lặp lại* (retry, double-click), ta chặn bằng **idempotency key + unique constraint**.
> - **Bài 4 (distributed lock)** — khi cần đồng bộ *across-instance* (giữa nhiều máy/tiến trình), ta cần khóa phân tán qua một **shared store**.

**Ẩn dụ định vị:** hãy hình dung 4 bài này như 4 cái van khác nhau lắp trên *cùng một đường ống nước*. Đường ống là "khoảng trống đọc–ghi". Bài 1 dạy bạn cái van rẻ và nhanh nhất (**atomic write**); ba bài sau là các van cho tình huống khó hơn.

---

## ② Giảng cơ bản → nâng cao

### (a) Trực giác trước

> **Ví dụ trực giác:** Hai nhân viên cùng nhìn một tờ giấy ghi *"còn 1 vé"*. Cả hai cùng đọc thấy "còn 1", cùng nghĩ "ok còn hàng", rồi cùng gạch đi và bán. Kết quả: **bán 2 vé cho 1 chỗ ngồi**.

Lỗi nằm ở đâu? **KHÔNG phải ở thao tác "ghi"** (gạch tờ giấy). Lỗi nằm ở **khoảng trống giữa lúc ĐỌC tờ giấy và lúc GHI lên nó** — trong khoảng trống đó, người kia đã kịp đọc cùng một giá trị cũ.

Định nghĩa chuẩn:

> 💡 **Race condition** = kết quả của chương trình **phụ thuộc vào thứ tự/timing không kiểm soát được** của các thao tác chạy **đồng thời** (concurrent) trên **cùng một tài nguyên**. Cùng một đoạn code, chạy 1000 lần có thể đúng 999 lần và sai 1 lần — vì lần đó timing rơi đúng vào khoảng trống.

**Vì sao đây là loại bug đáng sợ nhất?** Vì nó **không tái hiện ổn định**. Bug thông thường: sai là sai mọi lần, bạn debug được. Race condition: sai *thỉnh thoảng*, phụ thuộc tải và may rủi của bộ lập lịch → rất khó bắt, và (xem **I-RACE-007**) thường *chỉ nổ trên production*.

---

### (b) Cơ chế chi tiết

#### 🔍 Node single-thread mà vẫn race — **I-RACE-001**

Đây là nghịch lý làm nhiều người bối rối: *"JavaScript chạy một luồng (**single-thread**) cơ mà, làm gì có hai thứ chạy song song để mà tranh nhau?"*

Mấu chốt: **mỗi `await` nhả quyền điều khiển** (yield) trở về **event loop**. Trong lúc request A đang "chờ" DB trả lời ở một câu `await`, event loop **không ngồi không** — nó nhảy sang phục vụ request B. Thế là *giữa lúc A đọc `stock` và lúc A ghi `stock`*, request B đã kịp chen vào và đọc **cùng một giá trị cũ**.

**Timeline — vì sao single-thread vẫn race:**

```
Tài nguyên chia sẻ (DB): stock = 1

T1  Request A:  const s = await getStock()   // đọc → s = 1
       ↓  (await nhả control về event loop)
T2  Request B:  const s = await getStock()   // ❌ B chen vào, đọc → s = 1 (giá trị CŨ)
       ↓
T3  Request A:  await setStock(s - 1)        // ghi 0   (A nghĩ "1 → 0")
       ↓
T4  Request B:  await setStock(s - 1)        // ghi 0   (B cũng nghĩ "1 → 0")

Kết quả: stock = 0, nhưng đã bán 2 đơn.  ❌ oversell
```

> 🔎 **Lưu ý sâu — đây là race trên _shared data_, KHÔNG phải tranh chấp biến RAM.** Trong ngôn ngữ đa luồng (Java, Go...), race thường là hai thread cùng giành một biến trong bộ nhớ. Ở Node, biến cục bộ trong một request là *riêng tư*, không ai giành. Cái bị giành là **dữ liệu chia sẻ ngoài tiến trình** — hàng nằm trong **DB/Redis**. Đó mới là "tài nguyên" mà các request đua nhau.

> ⚠️ **Phân biệt với G-ASY (cross-ref):** Mục **G-ASY** bàn race kiểu *"đoán timing trong event-loop"* (ví dụ giả định callback chạy theo thứ tự nào đó). Mục **I** ở đây bàn race **trên dữ liệu chia sẻ**. Đừng nhầm hai loại — cách chống cũng khác.

---

#### 🔍 Check-then-act / **TOCTOU** — **I-RACE-002**

**TOCTOU** = *Time-Of-Check To Time-Of-Use* (khoảng cách thời gian giữa lúc *kiểm tra* điều kiện và lúc *dùng* kết quả kiểm tra đó).

> **Ẩn dụ:** Bạn mở tủ lạnh, thấy *còn 1 hộp sữa* (CHECK). Bạn quay đi lấy ly. Lúc quay lại định rót (USE), em bạn đã uống mất rồi. Cái "thấy còn sữa" đã **cũ** ngay khoảnh khắc bạn quay đi.

Trong code, **check-then-act** trông rất ngây thơ và *có vẻ* đúng:

```javascript
// ❌ Mẫu check-then-act điển hình — DÍNH TOCTOU
if (stock >= 1) {     // CHECK: kiểm tra điều kiện
   stock -= 1;        // ACT:   hành động dựa trên điều kiện
   await save(stock);
}
```

**Vì sao sai?** Vì *giữa* dòng CHECK và dòng ACT, có một khoảng trống (đặc biệt nếu có `await` xen vào, hoặc đơn giản là hai request chạy đồng thời). Trong khoảng đó, **state đã cũ**: cả hai request cùng đọc `stock = 1`, cùng thấy điều kiện "đủ hàng" là *true*, cùng trừ đi 1 → **oversell** (bán vượt tồn kho).

> 🎯 **Chốt:** Mọi bug oversell, double-spend, trừ tiền hai lần... gần như đều quy về **check-then-act trên một state đang cũ dần**.

---

#### ✅ Atomic update cứu thế nào — **I-RACE-003**

> **Ví dụ trực giác:** Thay vì "nhìn tờ giấy rồi mới gạch" (hai bước, có khoảng trống), ta yêu cầu quầy vé làm **một thao tác duy nhất, không chia tách được**: *"Nếu còn vé thì trừ 1, làm liền tay, không ai chen vào giữa được."*

Đó chính là **atomic update** (cập nhật nguyên tử — *atomic* nghĩa là "không thể chia nhỏ hơn, làm trọn gói một phát"). Cụ thể:

```sql
UPDATE products
SET    stock = stock - 1
WHERE  id = ? AND stock > 0;
```

**Vì sao an toàn hơn "đọc ở app rồi ghi lại"?** Vì **DB tự khóa row trong đúng một câu lệnh** — điều kiện kiểm tra (`stock > 0`) và hành động (`stock - 1`) nằm **trong cùng một thao tác**, *không tách ra* được, nên **không còn khoảng trống** cho ai chen vào.

So sánh với **read-modify-write (RMW)** ở app — *đọc ra RAM → tính toán → ghi đè lại*:

> 🚨 **read-modify-write mở toang cửa cho lost update.** **Lost update** = hai request đọc cùng một giá trị, mỗi đứa tính toán riêng, rồi *ghi đè lên kết quả của nhau* → một bản cập nhật **biến mất không dấu vết**.

**Timeline — atomic vs read-modify-write:**

```
❌ read-modify-write (RMW) ở app — DÍNH lost update
   DB: stock = 5

   A: đọc 5 ──┐                         B: đọc 5 ──┐
              ↓ tính 5-1=4                         ↓ tính 5-1=4
   A: ghi 4 ──┘                         B: ghi 4 ──┘
   → DB còn 4, nhưng đã bán 2 đơn (đúng ra phải còn 3).  ❌ mất 1 lần trừ

────────────────────────────────────────────────────────────

✅ atomic conditional update — AN TOÀN
   DB: stock = 5

   A: UPDATE ... stock = stock-1 WHERE stock>0   → DB khóa row, 5→4 ✅
   B: UPDATE ... stock = stock-1 WHERE stock>0   → đợi A xong, 4→3 ✅
   → DB còn 3, bán 2 đơn.  ✅ đúng, không khoảng trống
```

---

#### 🔍 Counter ra < kỳ vọng — **I-RACE-005**

**Triệu chứng kinh điển:** 1000 request cùng `+1` vào một bộ đếm, nhưng kết quả cuối ra **< 1000** (ví dụ 873). *"Tôi gọi tăng đủ 1000 lần cơ mà?!"*

> 💡 Đây là **lost update** mặc áo khác. Mỗi lần "mất" một bản cập nhật là một đơn vị đếm bốc hơi. Mất 127 lần → ra 873.

**Thủ phạm:** bộ đếm đặt ở **app hoặc cache** và cập nhật theo kiểu **read-modify-write** (`x = x + 1` thực chất là *đọc x → cộng → ghi lại*).

**Cách sửa — atomic increment** (tăng nguyên tử), KHÔNG cần lock toàn cục:

| Nơi đặt counter | Cách atomic đúng | Ghi chú |
|---|---|---|
| **DB column** | `UPDATE t SET col = col + 1 WHERE id = ?` | DB tự serialize trên row |
| **Redis** | `INCR key` | Lệnh **INCR** của Redis là atomic sẵn |

> 🎯 **Chốt:** Thấy bộ đếm ra thiếu → nghi ngay **read-modify-write**, sửa bằng **atomic increment** (`col = col + 1` hoặc Redis `INCR`). Đừng vội nghĩ tới lock — phần lớn trường hợp *không cần*.

---

### (c) Mép giới hạn & sai lầm thường gặp

#### 🚨 "Thêm sleep/retry để né race" là **anti-pattern** — **I-RACE-006**

> **Ẩn dụ:** Hai người hay va nhau ở cửa, bạn "sửa" bằng cách bảo một người *đếm tới 3 rồi mới đi*. Hôm nay hết va. Mai có người thứ ba, hoặc người kia đi nhanh hơn → va lại. Bạn **không sửa cái cửa**, bạn chỉ đang cầu may về timing.

**Fix dựa-timing cực kỳ giòn (brittle).** Thêm `sleep()` hay `retry` mù chỉ làm bug **khó tái hiện hơn**, chứ **không biến mất**. Tệ hơn: nó tạo cảm giác *"đã fix"* giả, để rồi bug tái phát dưới một mức tải khác — lúc đó khó debug gấp bội.

> 🎯 **Nguyên tắc vàng:** Race phải trị bằng **cơ chế đồng bộ thật** — **atomic op** / **lock** / **unique constraint** — chứ không phải bằng cách *điều chỉnh timing*.

---

#### 🔍 "Chỉ nổ trên prod" — **I-RACE-007**

**Vì sao máy dev không thấy bug, prod thì có?**

| Môi trường | Đặc điểm | Hệ quả |
|---|---|---|
| **Dev (1 máy, tải thấp)** | Request đến lẻ tẻ, gần như **tuần tự** (sequential) | Hầu như không có hai request rơi vào *cùng* khoảng trống → **không thấy race** |
| **Prod (nhiều instance, tải cao)** | Nhiều **instance** chạy **song song**, request dồn dập | **Concurrency thật** → khoảng trống bị chen liên tục → race nổ |

> ⚠️ **Cảnh báo:** **In-process mutex** (khóa trong một tiến trình) hay **biến local** **KHÔNG bảo vệ được across-instance** (giữa nhiều máy/tiến trình). Trên dev một process thì "có vẻ chạy"; lên prod nhiều process thì vô dụng. Đây là cái bẫy dẫn thẳng tới I-RACE-008 dưới đây.

---

#### 🚨 Map/biến đếm in-RAM một instance — **I-RACE-008** (bẫy ⭐⭐⭐⭐⭐)

> **Ví dụ trực giác:** Ba quầy vé, mỗi quầy có *sổ riêng* ghi "còn bao nhiêu vé". Quầy A bán xong ghi vào sổ A; quầy B **không nhìn thấy** sổ A. Ba quầy cùng tưởng "còn 100" → bán tổng 300 vé cho 100 chỗ.

**Bản chất kỹ thuật:** một `Map` / `Set` / biến đếm để **trong RAM của một instance** thì **không chống được race giữa nhiều instance** — vì *mỗi instance có state riêng, không thấy nhau*.

```javascript
// ❌ Bẫy I-RACE-008 — "khóa" bằng object in-memory
const inFlight = new Set();   // chỉ tồn tại trong RAM của 1 process
// 2 instance → mỗi cái có Set riêng → KHÔNG thấy nhau → vẫn race
```

> 🎯 **Chốt vàng (đắt giá nhất bài):** Muốn đồng bộ across-instance, **phải có một *shared store* (Redis/DB) làm điểm đồng bộ DUY NHẤT**. "Dùng object in-memory làm lock" là **bẫy kinh điển** — nghe thì gọn, nhưng chết ngay khi scale ra >1 instance. → chi tiết cơ chế khóa across-instance ở **Bài 4 (distributed lock)**.

---

#### 🔍 Double-submit / double-click — **I-RACE-004**

**Tình huống:** người dùng bấm nút "Thanh toán" hai lần (lỡ tay), hoặc mở hai tab, hoặc mạng chậm nên client **retry** → server nhận **hai request giống hệt** → tạo hai đơn / trừ tiền hai lần. Đây đúng là **duplicate do concurrency**.

> 🚨 **Sai lầm phổ biến: chỉ disable nút submit ở UI.** UI fix **KHÔNG cứu** được:
> - **Retry mạng** (client tự gửi lại khi timeout) — UI không biết.
> - **Nhiều tab / nhiều thiết bị** — mỗi cái một nút riêng.
> - Người dùng cố ý gọi API trực tiếp.

**Cách đúng — chặn ở *server*:** **idempotency key** (mỗi thao tác mang một khóa duy nhất do client sinh) **+ unique constraint** (ràng buộc duy nhất ở DB) → request thứ hai mang cùng key sẽ bị DB từ chối, *không* tạo đơn trùng.

> → **cross-ref Bài 3** để học sâu **idempotency key + unique constraint**.

---

### 🧩 Tổng hợp: 4 chiến lược chống race (đúng như tên bài)

Bốn chiến lược dưới đây *đều* bịt cái "khoảng trống đọc–ghi", nhưng theo cách khác nhau và cho tình huống khác nhau. Đây là cái bạn cần **chọn đúng** theo tình huống (xem tiêu chí ở ⑧):

| Chiến lược | Ý tưởng cốt lõi | Hợp khi | Học sâu ở |
|---|---|---|---|
| **1. Atomic write** (conditional update / `INCR`) | Gộp check + act vào *một* câu lệnh, đẩy điều kiện vào `WHERE` | Thao tác đơn giản (trừ kho, đếm, +/− số dư) | **Bài 1** (bài này) |
| **2. Pessimistic lock** (khóa bi quan) | Khóa row *trước*, ai tới sau phải xếp hàng đợi | Tranh chấp cao, cần đọc-tính-ghi nhiều bước | **Bài 2** |
| **3. Optimistic lock + retry** (khóa lạc quan) | Không khóa trước; ghi kèm điều kiện "version chưa đổi", trượt thì **retry** | Tranh chấp thấp, retry rẻ | **Bài 2** |
| **4. Unique constraint** (+ idempotency key) | Để DB từ chối bản trùng nhờ ràng buộc duy nhất | Chống *request lặp* (double-submit, retry) | **Bài 3** |

> 💡 **Mặc định nên thử trước:** **atomic write**. Nó rẻ nhất, nhanh nhất, *không* giết throughput. Chỉ leo lên lock khi atomic-một-câu-lệnh thật sự không diễn đạt nổi logic của bạn.

> 📘 **vs** ➕ — *docs gốc roadmap* chỉ chạm race ở mức khái niệm; các phần **TOCTOU**, **atomic-vs-RMW**, **across-instance**, **anti-pattern sleep/retry** là phần ➕ bổ sung theo chiều sâu Tech Lead.

---

## ③ ⚠️ Kiến thức cũ / cách làm bị thay thế

| Cái cũ (phổ biến / hay thấy trong code) | Thay bằng (nên dùng) | Vì sao |
|---|---|---|
| Đọc giá trị ở app → tính toán → ghi lại (**read-modify-write**) | **Atomic / conditional write** (`SET x = x - 1 WHERE x > 0`, Redis **INCR**) | **RMW** mở khoảng trống đọc–ghi → **lost update** |
| `if (stock >= 1) { stock--; save() }` (**check-then-act**) | **Conditional update** đẩy điều kiện vào `WHERE` | **TOCTOU**: bị chen *giữa* check và act |
| Thêm `sleep()` / **retry** mù để "né" race | **Cơ chế đồng bộ thật** (**atomic op** / **lock** / **unique constraint**) | Fix **dựa-timing** giòn, không trị gốc |
| **Map/biến đếm in-memory** làm "khóa" | **Shared store** (Redis/DB) làm điểm đồng bộ | Biến local **không xuyên instance** (across-instance) |
| Disable nút submit ở **UI** để chống duplicate | **Idempotency key + unique constraint** ở server | UI không cứu **retry / đa tab / đa thiết bị** |

---

## ④ Áp dụng thực tế + So sánh bigtech

**Use case thật — flash-sale bán vé/đặt chỗ:** 10.000 người cùng bấm mua, chỉ có **100 vé**.

❌ **Naive:** `if (còn vé) { bán }` → đây chính là **check-then-act** → **oversell hàng loạt** (bán mấy trăm vé cho 100 chỗ).

✅ **Kiến trúc tối thiểu đúng:** đẩy quyết định **vào DB** bằng **atomic conditional update**:

```sql
UPDATE tickets
SET    remaining = remaining - 1
WHERE  id = ? AND remaining > 0;
-- affected rows = 1 → mua thành công
-- affected rows = 0 → hết vé, trả lỗi cho user
```

**Vì sao tốt:** không cần **lock toàn cục** (global lock), **throughput cao**, vì DB chỉ serialize *trên đúng row của sản phẩm đó*.

**Bigtech pattern** (phổ biến, *không khẳng định tuyệt đối*): các hệ **inventory/ticketing** lớn thường ưu tiên **atomic DB op**, hoặc **partition theo key** — mỗi sản phẩm/khóa do **một consumer** xử lý — thay vì **global lock**, vì lock toàn cục *giết throughput*.

> 🔎 **Mở rộng quy mô rất lớn:** họ dồn về mô hình **single-writer per partition** — ví dụ **Kafka partition theo `product_id`** — để *biến concurrency thành tuần tự cục bộ*. Nghĩa là: trong một partition, các thao tác lên cùng một sản phẩm được xử lý **lần lượt** bởi một writer duy nhất → không còn đua nhau.

> ⚠️ Đây là **pattern chung**; chi tiết triển khai từng công ty thay đổi — **verify khi cần**, đừng phát biểu như chân lý tuyệt đối trong phỏng vấn.

---

## ⑤ Code thực hành + cấu hình

**Bước 1 — câu lệnh atomic (trái tim của giải pháp):**

```sql
-- ✅ Atomic conditional update: chống oversell trong MỘT câu lệnh, không khoảng trống
-- affected_rows = 1 → trừ thành công; = 0 → hết hàng (KHÔNG ghi đè sai)
UPDATE products
SET    stock = stock - 1
WHERE  id = $1
  AND  stock > 0;
```

**Bước 2 — gọi từ Node/TypeScript:**

```javascript
// Node/TypeScript (ví dụ với 'pg' — verify version tại docs chính thức)
// package.json: "pg": "^8.13.0"   // pin version thật khi dùng
import { Pool } from 'pg';
const pool = new Pool(); // cấu hình qua biến môi trường PG* (KHÔNG hardcode credential)

async function buyOne(productId: string): Promise<boolean> {
  // Đẩy điều kiện vào WHERE → DB tự serialize trên row → KHÔNG có TOCTOU
  const res = await pool.query(
    `UPDATE products SET stock = stock - 1
     WHERE id = $1 AND stock > 0`,
    [productId],
  );
  // rowCount = 1 → trừ kho thành công (mua được)
  // rowCount = 0 → điều kiện stock > 0 không còn đúng → HẾT HÀNG
  return res.rowCount === 1;
}
```

**Bước 3 — ghi nhớ các anti-pattern để KHÔNG lặp lại:**

```javascript
// ❌ ANTI-PATTERN (đừng làm) — read-modify-write ở app, dính lost update:
// const { stock } = await getStock(productId);          // ĐỌC ra RAM
// if (stock > 0) await setStock(productId, stock - 1);  // GHI đè → có khoảng trống → race

// ❌ Bẫy I-RACE-008: in-memory "lock" KHÔNG xuyên instance
const inFlight = new Set<string>(); // chỉ tồn tại trong RAM của 1 process
// Trên 2 instance, mỗi cái có Set riêng → KHÔNG thấy nhau → vẫn race.
// → Phải dùng Redis/DB làm điểm đồng bộ (xem Bài 4).
```

> 🔧 **Cấu hình:** file `.env` chứa `PGHOST` / `PGUSER` / `PGPASSWORD` / `PGDATABASE`; nạp bằng **dotenv** hoặc **secret manager**; `package.json` **pin version** rõ ràng; chạy `node app.js`.
>
> 🚨 **Tuyệt đối KHÔNG commit credential** vào repo.

---

## ⑥ Keywords cần nhớ (glossary)

**🧠 Cần HIỂU (nắm bản chất):**
- **race condition** — kết quả phụ thuộc timing không kiểm soát trên tài nguyên chia sẻ.
- **TOCTOU / check-then-act** — khoảng trống giữa lúc *kiểm tra* và lúc *hành động* trên state đã cũ.
- **lost update** — hai bản cập nhật ghi đè nhau, một cái biến mất.
- **read-modify-write vs atomic write** — đọc-tính-ghi (có khoảng trống) đối lập với gộp-một-câu-lệnh (không khoảng trống).
- **across-instance vs in-process** — đồng bộ giữa *nhiều máy* đối lập với trong *một tiến trình*.
- **"shared store làm điểm đồng bộ"** — phải có một Redis/DB duy nhất thì nhiều instance mới thấy nhau.

**📌 Cần THUỘC (nhớ nguyên văn):**
- `UPDATE ... SET x = x - 1 WHERE x > 0;`
- Redis **`INCR`** (atomic increment).
- *"affected rows = 0 → điều kiện không còn đúng (hết hàng)"*.

**🛠️ Cần LÀM ĐƯỢC (kỹ năng tay):**
- Viết **conditional update** chống **oversell**.
- Nhận diện **counter-in-app** là thủ phạm khi đếm ra thiếu.
- Chỉ ra **vì sao in-memory lock vô dụng** khi chạy đa instance.

---

## ⑦ Mạch tư duy cần nhớ (mental model)

> 🎯 **Khẩu quyết 1 — nơi race sinh ra:**
> *"**Khoảng trống giữa lúc ĐỌC và lúc GHI** là nơi race sinh ra → bịt nó bằng **một thao tác atomic**, đừng vá bằng **timing**."*

> 🎯 **Khẩu quyết 2 — race là thuộc tính của phạm vi:**
> *"Race là **thuộc tính của phạm vi tài nguyên** — **biến local không bảo vệ được tài nguyên chia sẻ**. Muốn đồng bộ nhiều instance, phải có một shared store duy nhất."*

---

## ⑧ Câu hỏi phỏng vấn & thách đố

**(dễ)** **Race condition** là gì? **Node single-thread** sao vẫn dính?
→ *Gợi ý:* mỗi `await` **nhả control** về event loop; race xảy ra trên **shared data** ở DB/Redis, không phải tranh biến RAM.

**(TB)** Vì sao `UPDATE SET bal = bal - 10 WHERE id = ?` an toàn hơn *đọc-rồi-ghi* ở app?
→ *Gợi ý:* nó **atomic**, **không có khoảng trống** đọc–ghi; còn read-modify-write dính **lost update**.

**(khó)** 1000 request `+1` mà ra `< 1000`, debug từ đâu?
→ *Gợi ý:* **lost update** do **read-modify-write**; sửa bằng **atomic increment**; nghi ngay **counter đặt ở app/cache** là thủ phạm.

**(khó)** Liệt kê **4 chiến lược chống race** và chọn theo gì?
→ *Gợi ý:* **atomic write** / **pessimistic lock** / **optimistic lock + retry** / **unique constraint**; **chọn theo** mức **contention** (tranh chấp) + **chi phí retry** + **phạm vi tài nguyên** (một row hay across-instance).

> 🧩 **Thách đố / trick:**
> - *"Tôi để một **Map** đếm request trong RAM để rate-limit, sao prod vẫn vượt giới hạn?"*
>   → mỗi **instance** có Map riêng, **không xuyên instance** → cần **Redis** làm điểm đồng bộ. (I-RACE-008)
> - *"Thêm `await sleep(50)` thì hết lỗi rồi mà?"*
>   → bug chỉ bị **giấu đi**, sẽ tái phát dưới tải khác; chưa **trị gốc**. (I-RACE-006)

---

## ⑨ Bài tập thực hành + tiêu chí tự chấm

**BT1 — Chống oversell bằng conditional update.**
Viết endpoint `POST /buy` chống **oversell** bằng **conditional update**.
> ✅ **Đạt khi:** chạy **500 request đồng thời** trên `stock = 100` → **đúng 100** thành công, **400** trả `"hết hàng"`, và **DB `stock` không âm**.

**BT2 — Tái hiện rồi sửa lost update.**
Tái hiện **lost update** bằng **read-modify-write**, rồi sửa bằng **atomic increment**.
> ✅ **Đạt khi:** *chứng minh được* con số **sai** trước → sửa → con số **đúng** sau, và **giải thích nguyên nhân bằng lời** (chỉ ra đúng khoảng trống đọc–ghi).

---

## ⑩ Đọc thêm / nguồn chuẩn

- **PostgreSQL docs** — *Concurrency Control / Row Locking*.
- **Designing Data-Intensive Applications** (Kleppmann) — **ch.7** (race conditions, lost update).
- **Redis docs** — **INCR** atomicity.

---

> 🔚 **Hết Bài 1.** Theo WORKFLOW, đây là chỗ chuyển sang **Bước 3 — Chốt ghi nhớ** (3 câu **free-recall**, hỏi xong **DỪNG**).
>
> 🎯 Khi học live, **đừng đọc tiếp Bài 2 vội** — hãy tự trả lời recall trước. Gợi ý 3 câu tự hỏi: *(1) Khoảng trống đọc–ghi là gì và vì sao nó sinh race? (2) Vì sao in-memory lock vô dụng đa instance? (3) Kể 4 chiến lược chống race và tiêu chí chọn.*
