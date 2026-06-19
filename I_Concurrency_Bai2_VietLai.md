# 🎯 BÀI 2 — Optimistic vs Pessimistic Locking

**Phủ:** I-LOCK-001 → I-LOCK-012 · **Vũ khí cơ bản nhất chống race trong MỘT database.**

> 💡 **Cách đọc bài này.** Bài 1 dạy bạn bịt khoảng trống đọc–ghi bằng *một câu lệnh atomic*. Nhưng đời không phải lúc nào cũng gọn một câu — đôi khi bạn phải *sửa nhiều field*, hoặc cần *báo cho user biết có người vừa sửa chen vào*. Lúc đó ta cần **lock** (khóa). Cả bài chỉ xoay quanh một lựa chọn: **khóa-trước-rồi-làm** (pessimistic) hay **làm-trước-rồi-kiểm-tra-lúc-commit** (optimistic). Nắm được "khi nào chọn cái nào" là nắm được 80% giá trị bài này.

---

## ① Mục tiêu & vị trí trong mạch

Học xong bài này bạn sẽ làm được 5 việc:

1. **Phân biệt triết lý** hai loại lock: **pessimistic** ("bi quan") và **optimistic** ("lạc quan").
2. **Hiện thực** được cả hai — bằng **version column** (cho optimistic) và `FOR UPDATE` (cho pessimistic).
3. **Chọn đúng** theo **workload** (đặc tính tải): tranh chấp cao hay thấp, ghi nóng hay đọc nhiều.
4. **Thiết kế retry an toàn** cho optimistic (giới hạn số lần + **jittered backoff** + đọc lại state).
5. **Biết khi nào lock là THỪA** — đây là phần Tech Lead, tránh "khóa cho chắc" rồi giết hiệu năng.

> 🎯 **Vị trí trong mạch học.**
> - **Nối tiếp Bài 1:** lock là cách bịt **khoảng trống đọc–ghi** *khi atomic conditional update đơn lẻ không đủ* (cần sửa nhiều field, hoặc cần báo **conflict** cho user).
> - **Mở đường cho Bài 4:** Bài 4 sẽ nâng khái niệm khóa này lên phạm vi **across-instance** (giữa nhiều máy) — tức **distributed lock**.

**Ẩn dụ định vị:** nếu Bài 1 là "trừ tiền liền tay không ai chen được", thì Bài 2 là khi giao dịch phức tạp tới mức bạn cần **giữ quyền** trong một khoảng — hoặc *giữ chìa khóa phòng* (pessimistic), hoặc *cứ làm rồi tới lúc nộp mới kiểm tra có ai đụng không* (optimistic).

---

## ② Giảng cơ bản → nâng cao

### (a) Trực giác — hai triết lý đối lập — **I-LOCK-001**

> **Ví dụ trực giác — Pessimistic ("bi quan"):** Như **mượn chìa khóa phòng họp**. Bạn cầm chìa, vào phòng, *khóa cửa lại*. Ai muốn dùng phòng phải **đứng ngoài chờ** tới khi bạn ra. Triết lý: *"Conflict chắc chắn hay xảy ra → khóa trước cho yên tâm."*

> **Ví dụ trực giác — Optimistic ("lạc quan"):** Như **sửa Google Doc**. Mọi người cùng gõ song song, không ai bị chặn. Chỉ *đến lúc lưu* hệ thống mới báo: *"Ai đó vừa sửa, tải lại đi."* Triết lý: *"Conflict hiếm khi xảy ra → cứ chạy thoải mái, đụng thì làm lại."*

| Tiêu chí | **Pessimistic** (bi quan) | **Optimistic** (lạc quan) |
|---|---|---|
| Giả định về **conflict** | Hay xảy ra | Hiếm khi xảy ra |
| Thời điểm xử lý | **Khóa trước** khi đụng data | **Kiểm tra lúc commit** |
| Người đến sau | Phải **chờ** (xếp hàng) | Chạy song song, **fail + làm lại** nếu đụng |
| Ẩn dụ | Chìa khóa phòng họp | Google Doc báo "tải lại" |

---

### (b) Cơ chế chi tiết

#### 🔍 Optimistic = version column / CAS — **I-LOCK-002**

Mỗi row gắn thêm một cột **version** (hoặc timestamp). Khi update, bạn *đính kèm điều kiện "version chưa đổi"*:

```sql
UPDATE t
SET    ..., version = version + 1
WHERE  id = ? AND version = ?;   -- ? = version mà bạn đã đọc lúc đầu
```

**Đọc kết quả:**
- `affected rows = 1` → version lúc ghi vẫn đúng như lúc đọc → **không ai chen** → ✅ thành công.
- `affected rows = 0` → đã có người **tăng version trước bạn** → ❌ **conflict** → fail + **retry**.

> 💡 Đây chính là **Compare-And-Swap (CAS)** ở tầng row: *"chỉ ghi nếu giá trị (version) vẫn đúng như tôi nhìn thấy lúc đầu"*. CAS là một viên gạch nền tảng của lập trình concurrent.

**Timeline — optimistic phát hiện conflict:**

```
orders(id=7): status='pending', version=4

T1  A: đọc order → version = 4
T2  B: đọc order → version = 4   (cùng đọc 4)
       ↓
T3  B: UPDATE ... version=5 WHERE id=7 AND version=4   → affected=1 ✅ (B thắng, version giờ là 5)
       ↓
T4  A: UPDATE ... version=5 WHERE id=7 AND version=4   → affected=0 ❌
       (vì version DB đã là 5, không còn =4) → A biết bị chen → retry: đọc lại version=5, tính lại
```

> 🔎 **Lưu ý:** **TypeORM `@VersionColumn`** hỗ trợ cơ chế này — nhưng **verify**, và xem **bẫy ở Mục ③** (ORM không phải lúc nào cũng tự thêm `WHERE version=?`).

---

#### 🔍 Pessimistic = lock ở DB — **I-LOCK-003**

Khóa row tường minh, *giữ tới hết transaction*:

```sql
SELECT ... FOR UPDATE;   -- khóa exclusive (độc quyền): chỉ mình bạn sửa được
SELECT ... FOR SHARE;    -- khóa share (chia sẻ): nhiều người cùng đọc-giữ, nhưng không ai ghi
```

**`FOR UPDATE`** khóa row tới khi transaction kết thúc; transaction khác đụng *đúng row đó* **phải chờ**.

**Timeline — pessimistic serialize:**

```
wallets(id=9): balance=100

T1  tx-A: BEGIN; SELECT ... FOR UPDATE  → A khóa row 9 🔒
T2  tx-B: BEGIN; SELECT ... FOR UPDATE  → B phải CHỜ (row đang bị A khóa) ⏳
       ↓
T3  tx-A: UPDATE balance=balance-30; COMMIT  → mở khóa, balance=70 ✅
       ↓
T4  tx-B: (mới được chạy) đọc balance=70 mới, UPDATE balance=70-20=50; COMMIT ✅
```

> 🔎 Không lặp lại lý thuyết **isolation** của **mục E** — ở đây ta nhìn lock từ **góc chiến lược** (chọn cái nào), không phải từ góc isolation level.

---

#### 🔍 `SELECT ... FOR UPDATE SKIP LOCKED` — **I-LOCK-009**

> **Ví dụ trực giác:** 5 nhân viên cùng bốc phiếu việc từ một khay. Thay vì *chen nhau giành cùng một phiếu rồi 4 người phải chờ*, mỗi người **bỏ qua phiếu ai đó đã cầm** và lấy phiếu kế tiếp. Không ai đứng chờ ai.

Đây là lời giải kinh điển cho bài **"hàng đợi trong DB" (job queue on DB)**. Nhiều **worker** cùng quét bảng job: `SKIP LOCKED` khiến mỗi worker **bỏ qua row đang bị khóa** *thay vì chờ* → mỗi worker lấy được **một job khác nhau**, giảm **contention** (tranh chấp).

**Timeline — SKIP LOCKED chia việc:**

```
jobs: [#1 pending] [#2 pending] [#3 pending]

T1  worker-A: SELECT ... LIMIT 1 FOR UPDATE SKIP LOCKED → lấy #1, khóa #1 🔒
T2  worker-B: SELECT ... LIMIT 1 FOR UPDATE SKIP LOCKED → #1 đang khóa → BỎ QUA → lấy #2 ✅
T3  worker-C: SELECT ... LIMIT 1 FOR UPDATE SKIP LOCKED → #1,#2 khóa → BỎ QUA → lấy #3 ✅
→ 3 worker lấy 3 job KHÁC nhau, KHÔNG ai chờ ai.
```

> 💡 Đây là **nền của nhiều job-queue-trên-Postgres** hiện đại.

---

### (c) Chọn cái nào & mép giới hạn

#### 🔍 Chọn theo workload — **I-LOCK-004**

| Đặc tính workload | Nên chọn | Vì sao |
|---|---|---|
| **Contention thấp** + **read-heavy** + retry rẻ/idempotent | **Optimistic** | Hiếm đụng → ít retry → tận dụng song song tối đa |
| **Contention cao** + ghi nóng cùng row + retry đắt | **Pessimistic** | Serialize để mỗi tx *tiến chắc*, tránh phí công retry |

> 🎯 **Nguyên tắc vàng:** **Đo conflict rate** (tỉ lệ đụng độ thực tế), **đừng chọn cảm tính**. Chọn lock như chọn thuốc — phải theo triệu chứng đo được, không theo "nghe nói".

---

#### 🚨 Bẫy ngược — optimistic dưới contention cao — **I-LOCK-005**

> **Ẩn dụ:** 50 người cùng lao vào một cánh cửa hẹp. Kiểu "lạc quan" (ai cũng cứ thử) → đa số *đập đầu vào nhau rồi lùi ra thử lại* → hành lang hỗn loạn, ít ai qua được. Kiểu "bi quan" (xếp hàng một cửa) chậm mà *đều*, ai cũng qua.

**Optimistic dưới contention cao có thể TỆ HƠN pessimistic.** Vì nhiều transaction cùng đụng **một row** → đa số **fail version-check** → **retry dồn dập** → **wasted work** (phí công vô ích), tiệm cận **livelock** (ai cũng bận rộn nhưng chẳng ai tiến).

**Timeline — retry storm:**

```
1 row nóng, 10 tx cùng đua:
  tx1 thắng (version 4→5).  tx2..tx10 đều affected=0 → ❌ retry
  → tx2 thắng (5→6).        tx3..tx10 lại affected=0 → ❌ retry lần 2
  → ... cứ thế, mỗi vòng chỉ 1 tx tiến, 9 tx phí công.
  → throughput thấp, CPU/DB bận rộn vô ích (livelock-ish).
```

> 🎯 **Chốt:** **Pessimistic** trong cảnh này tuy *serialize* (xếp hàng) nhưng **mỗi tx tiến đều, không phí công**. Contention càng cao, ưu thế pessimistic càng rõ.

---

#### 🚨 ABA problem — **I-LOCK-006** (⭐⭐⭐⭐⭐)

> **Ví dụ trực giác:** Bạn để 100k trong ví, đi ra ngoài. Lúc về vẫn thấy *100k*, bạn yên tâm "không ai đụng". Nhưng thật ra có người đã *rút 100k rồi nạp lại 100k* — giao dịch *đã xảy ra* mà bạn **không hề hay biết** vì con số cuối *trông y hệt*.

**ABA problem:** giá trị đổi **A → B → A**. Nếu bạn kiểm tra kiểu *"có còn bằng giá trị cũ không?"* thì sẽ tưởng **không đổi** → **bỏ sót** thay đổi xảy ra ở giữa.

**Vì sao version column trị được?**

| Cách phát hiện thay đổi | Gặp A→B→A | Vì sao |
|---|---|---|
| **So sánh "bằng giá trị cũ"** | ❌ Bị lừa, tưởng không đổi | Giá trị cuối trùng giá trị đầu |
| **Version column** (đơn điệu tăng — *monotonic*) | ✅ Phát hiện được | Mỗi lần ghi version **luôn nhảy lên** (4→5→6), không bao giờ quay lại — dù giá trị data quay về như cũ |

> 🎯 **Chốt:** Đây chính là lý do **version column ăn đứt "so sánh giá trị"**: version **đơn điệu tăng (monotonic)** nên không bao giờ bị lừa bởi A→B→A.

---

#### 🚨 Deadlock với pessimistic — **I-LOCK-007**

> **Ẩn dụ:** A cầm chìa phòng 1, chờ phòng 2. B cầm chìa phòng 2, chờ phòng 1. Hai bên **chờ nhau vĩnh viễn** — đó là **deadlock** (kẹt khóa chéo).

**Timeline — deadlock do khóa ngược thứ tự:**

```
tx-A: FOR UPDATE row#1 🔒 ──→ rồi xin row#2 ⏳ (đợi B nhả)
tx-B: FOR UPDATE row#2 🔒 ──→ rồi xin row#1 ⏳ (đợi A nhả)
→ A đợi B, B đợi A → DEADLOCK ❌ (DB phải hủy một tx để gỡ)
```

**Cách phòng:**
- **Khóa theo thứ tự nhất quán** — *luôn lock id nhỏ trước* (cả A và B đều lock #1 trước #2 → không bao giờ chéo).
- **Transaction ngắn** — giữ khóa càng ngắn càng ít cơ hội kẹt.
- **Lock granular** — khóa **row > table** (khóa nhỏ, ít va chạm).

> 🔎 **cross-ref E-ISO** (isolation) để hiểu sâu hơn nền tảng giao dịch.

---

#### 🚨 Lạm dụng `FOR UPDATE` — **I-LOCK-010**

> ⚠️ **Anti-pattern "cứ `FOR UPDATE` cho chắc" trên mọi read.** Hậu quả dây chuyền:
> - **Serialize không cần thiết** → mất song song → chậm.
> - **Giữ lock + connection lâu** → **block** (chặn) các tx khác.
> - Dễ **deadlock**, và **cạn connection pool** (hết kết nối → app treo).

> 🎯 **Chốt:** Chỉ khóa **đúng điểm nóng read-modify-write**, không rải `FOR UPDATE` bừa.

---

#### ✅ Retry đúng cách cho optimistic — **I-LOCK-008**

Một retry tử tế phải có đủ 3 yếu tố:

1. **Giới hạn N lần** — không **retry vô hạn** (nếu không sẽ treo khi conflict kéo dài).
2. **Jittered backoff** — *đợi một khoảng ngẫu nhiên tăng dần*. Vì sao cần **jitter** (nhiễu ngẫu nhiên)? Để **tránh các retry đồng pha** lại đập vào nhau cùng lúc (nếu mọi tx cùng đợi *đúng* 20ms rồi cùng thử lại → lại đụng nhau y như cũ).
3. **Đọc lại state mới mỗi vòng** — retry mà vẫn dùng state cũ thì *chắc chắn fail tiếp*; phải đọc lại version mới rồi tính lại.
4. Sau N lần vẫn fail → **trả lỗi rõ ràng**, đừng retry mãi.

> ⚠️ **Cảnh báo:** retry **không jitter** + **không giới hạn** = công thức tạo **retry storm / livelock**.

---

### (d) Khi nào lock là THỪA

#### 🔍 Version column vs atomic conditional update — **I-LOCK-011**

| Tình huống | Dùng gì | Vì sao |
|---|---|---|
| Counter / stock **đơn giản** (một field) | **Atomic conditional update** (`stock = stock - 1 WHERE stock > 0`) | Gọn nhất — *khỏi cần version*, xem lại Bài 1 |
| Sửa **nhiều field** hoặc cần **báo conflict cho user** (form edit) | **Version column** | Diễn đạt được logic phức tạp + UX rõ "ai đó vừa sửa" |

> 💡 Đừng "nâng cấp" mọi thứ lên version column. Nếu một câu **atomic** của Bài 1 đủ giải → dùng nó, đơn giản và nhanh hơn.

---

#### 🔍 Multi-instance có cần distributed lock? — **I-LOCK-012** (⭐⭐⭐⭐⭐)

> **Câu hỏi bẫy phỏng vấn:** *"App chạy 5 instance, tôi đã dùng optimistic version column. Có cần thêm distributed lock không?"*

**Trả lời: KHÔNG.** Nếu đã dùng **optimistic version column** thì **không cần** thêm **distributed lock** — vì:

> 🎯 **Nhận thức cốt lõi (đắt giá nhất bài):** **version check ở DB ĐÃ LÀ một điểm đồng bộ atomic across-instance**. Mọi instance đều ghi qua *cùng một DB*; câu `WHERE version = ?` được DB serialize trên row, *bất kể request đến từ instance nào*. Thêm **distributed lock** lúc này thường **thừa** và **giảm throughput**.

> 💡 **Bài học lớn:** **DB constraint chính là một cơ chế concurrency control.** Đừng tự dựng lại cái mà DB đã cho sẵn và làm tốt hơn.

> 📘 **vs** ➕ — *docs gốc* chỉ chạm optimistic/pessimistic ở mức khái niệm; các phần **ABA**, **retry design**, **SKIP LOCKED**, *"lock thừa khi đã có version"* là phần ➕/⬆️ bổ sung theo chiều sâu Tech Lead.

---

## ③ ⚠️ Kiến thức cũ / cách làm bị thay thế

| Cái cũ | Thay bằng (nên dùng) | Vì sao |
|---|---|---|
| So sánh **"bằng giá trị cũ"** để phát hiện thay đổi | **Version column** đơn điệu tăng (hoặc timestamp) | Tránh **ABA problem** |
| `FOR UPDATE` trên **mọi read** "cho chắc" | Chỉ khóa **đúng điểm nóng read-modify-write** | Tránh **cạn pool**, **block**, **deadlock** |
| **Optimistic + retry vô hạn**, không backoff | **Retry có giới hạn + jittered backoff + đọc lại state** | Tránh **livelock / retry storm** |
| Thêm **distributed lock** khi đã có **optimistic version** | Tin vào **version check ở DB** (đã atomic across-instance) | **Lock thừa**, giảm **throughput** |
| **Pessimistic lock** theo thứ tự tùy tiện | Lock theo **thứ tự nhất quán** + **tx ngắn** | Tránh **deadlock** |
| Worker queue: nhiều worker `FOR UPDATE` **chờ nhau** | `FOR UPDATE` **SKIP LOCKED** | Mỗi worker lấy job khác nhau, hết **block** |

---

## ④ Áp dụng thực tế + So sánh bigtech

**Use case 1 — chỉnh sửa hồ sơ/đơn hàng nhiều field từ nhiều admin.**
Dùng **optimistic version**: ai lưu sau thấy **version lệch** → báo *"bản ghi vừa được người khác sửa, tải lại"*. **UX rõ ràng**, *không khóa người dùng* trong lúc họ đang gõ form.

**Use case 2 — trừ số dư ví tài chính dưới contention cao trên cùng một ví.**
Dùng **pessimistic `FOR UPDATE`** (hoặc **atomic update** của Bài 1) để **mỗi tx tiến chắc**, *tránh retry storm* mà optimistic gặp khi đụng độ cao.

**Bigtech pattern:**
- Các **ORM/framework lớn** (**Hibernate**, **EF Core**, **TypeORM**, **ActiveRecord**) đều cung cấp **optimistic locking** qua **version/timestamp column** như *default* cho web app — vì contention web app thường thấp.
- **Job-queue-trên-DB hiện đại** (nhiều hệ **Postgres-backed queue**) dựa trên **SKIP LOCKED**.

> ⚠️ Đây là **pattern phổ biến**; **verify API cụ thể** của từng framework trước khi phát biểu chắc nịch.

---

## ⑤ Code thực hành + cấu hình

**OPTIMISTIC — version column + CAS:**

```sql
-- ✅ OPTIMISTIC: version column + CAS. affected=0 ⇒ conflict ⇒ retry.
UPDATE orders
SET    status = 'paid', version = version + 1
WHERE  id = $1 AND version = $2;    -- $2 = version đã đọc lúc đầu
```

**PESSIMISTIC — khóa row tới hết transaction:**

```sql
-- ✅ PESSIMISTIC: khóa row tới hết tx. Tx khác đụng row này phải chờ.
BEGIN;
SELECT balance FROM wallets WHERE id = $1 FOR UPDATE;  -- khóa exclusive (độc quyền)
-- ... tính toán an toàn ở đây, KHÔNG ai chen được ...
UPDATE wallets SET balance = balance - $2 WHERE id = $1;
COMMIT;                                                 -- COMMIT mới nhả khóa
```

**JOB QUEUE — mỗi worker lấy job khác nhau:**

```sql
-- ✅ JOB QUEUE trong DB: mỗi worker lấy job khác nhau, không chờ nhau
SELECT id FROM jobs
WHERE  status = 'pending'
ORDER  BY created_at
LIMIT  1
FOR UPDATE SKIP LOCKED;   -- BỎ QUA row đang bị worker khác khóa (không chờ)
```

**Retry an toàn cho optimistic (giới hạn + jitter + đọc lại):**

```javascript
// ✅ Retry an toàn cho optimistic lock (giới hạn + jittered backoff + đọc lại)
async function updateWithOptimisticRetry(
  id: string,
  mutate: (cur: Order) => Partial<Order>,
  maxRetries = 3,                       // CÓ giới hạn — KHÔNG retry vô hạn
): Promise<Order> {
  for (let attempt = 0; attempt < maxRetries; attempt++) {
    const cur = await getOrder(id);     // ĐỌC LẠI state mới mỗi vòng (RẤT quan trọng)
    const patch = mutate(cur);
    const res = await db.query(
      `UPDATE orders SET status = $1, version = version + 1
       WHERE id = $2 AND version = $3`, // version=$3 = version vừa đọc → CAS
      [patch.status, id, cur.version],
    );
    if (res.rowCount === 1) return { ...cur, ...patch, version: cur.version + 1 }; // ✅ thắng
    // affected=0 ⇒ conflict → backoff CÓ jitter để tránh các retry đồng pha lại đụng nhau
    await sleep(2 ** attempt * 20 + Math.random() * 20);
  }
  throw new Error('OPTIMISTIC_CONFLICT: hết số lần thử, vui lòng tải lại'); // ❌ fail rõ ràng
}
```

**TypeORM `@VersionColumn` — và cái bẫy của nó:**

```javascript
// TypeORM @VersionColumn — verify tại docs chính thức
import { Entity, PrimaryGeneratedColumn, Column, VersionColumn } from 'typeorm'; // "typeorm": "^0.3.x" (pin thật)
@Entity()
export class Order {
  @PrimaryGeneratedColumn() id!: number;
  @Column() status!: string;
  @VersionColumn() version!: number;   // tự tăng mỗi lần save()
}

// ⚠️ BẪY (xem Mục ⑧): version check chỉ chắc chắn khi save() FULL ENTITY
// hoặc dùng findOne với lock option + version. Repository.update(...) một phần
// có thể KHÔNG kích hoạt version check → vẫn lost update.  // kiểm chứng tại docs chính thức
```

> 🔧 **Cấu hình:**
> - **Transaction phải NGẮN** — *đừng gọi API ngoài khi đang giữ `FOR UPDATE`* (giữ lock lâu → cạn pool).
> - Set **`statement_timeout` / `lock_timeout`** để không **treo vô hạn** khi chờ khóa.
> - Secrets nạp qua **env**, không hardcode.

---

## ⑥ Keywords cần nhớ (glossary)

**🧠 Cần HIỂU (nắm bản chất):**
- **pessimistic vs optimistic philosophy** — khóa-trước vs kiểm-tra-lúc-commit.
- **CAS (Compare-And-Swap)** — chỉ ghi nếu giá trị vẫn đúng như lúc đọc.
- **ABA problem** — A→B→A khiến "so sánh giá trị" bị lừa.
- **contention rate** — tỉ lệ tranh chấp thực tế; quyết định chọn loại lock.
- **livelock / retry storm** — ai cũng bận retry nhưng không ai tiến.
- **"DB constraint = concurrency control"** — ràng buộc DB chính là cơ chế đồng bộ.

**📌 Cần THUỘC (nhớ nguyên văn):**
- `UPDATE ... SET ..., version = version + 1 WHERE id = ? AND version = ?;`
- `SELECT ... FOR UPDATE [SKIP LOCKED];`
- *"affected-rows = 0 ⇒ conflict"*.

**🛠️ Cần LÀM ĐƯỢC (kỹ năng tay):**
- Triển khai **cả hai loại lock**.
- Viết **retry có giới hạn + jitter + đọc lại**.
- **Nhận diện lock thừa** (đã có version thì khỏi distributed lock).

---

## ⑦ Mạch tư duy cần nhớ (mental model)

> 🎯 **Khẩu quyết 1 — chọn loại lock:**
> *"**Pessimistic** = khóa-rồi-mới-làm (chờ); **Optimistic** = làm-rồi-kiểm-tra-lúc-commit (retry). **Chọn theo conflict rate, KHÔNG theo cảm tính.**"*

> 🎯 **Khẩu quyết 2 — hai cái bẫy:**
> *"**Version đơn điệu tăng đánh bại ABA**; và **retry phải có trần + jitter + đọc lại** — thiếu một thứ là sinh retry storm."*

---

## ⑧ Câu hỏi phỏng vấn & thách đố

**(dễ)** **Optimistic vs pessimistic** khác nhau ở **triết lý** nào?
→ *Gợi ý:* bi quan = khóa trước (chờ); lạc quan = chạy song song rồi kiểm tra lúc commit (retry).

**(TB)** **Optimistic** hiện thực thế nào? Mô tả **version column / CAS**.
→ *Gợi ý:* `WHERE version = ?`, `affected = 0` ⇒ có người chen ⇒ conflict ⇒ retry.

**(khó)** Khi nào **optimistic tệ hơn pessimistic**?
→ *Gợi ý:* **contention cao** → fail version-check dồn dập → **retry storm** → tiệm cận livelock.

**(khó)** **ABA problem** là gì, **version column** trị thế nào?
→ *Gợi ý:* A→B→A lừa "so sánh giá trị"; version **đơn điệu tăng** nên vẫn phát hiện.

**(khó)** App **multi-instance** dùng **optimistic version** còn cần **distributed lock** không?
→ *Gợi ý:* **không** — **version check đã atomic across-instance** vì mọi instance ghi qua cùng một DB.

> 🧩 **Thách đố / trick — AI hay sai ở đây:**
> *"AI viết `repository.update(id, { status })` với entity có `@VersionColumn` và bảo 'đã có optimistic locking' — đúng không?"*
> → ⚠️ **Cẩn thận: cú pháp đúng nhưng HÀNH VI có thể sai.** **TypeORM không phải lúc nào cũng tự thêm `WHERE version = ?`** trên update *một phần*; version check chỉ **tin cậy khi `save()` full entity** hoặc **`findOne` kèm lock option**. Đây đúng kiểu **AI viết đúng cú pháp nhưng sai hành vi** — phải **kiểm chứng tại docs và test bằng update song song**, đừng tin lời cam kết mặc định của ORM. **(verify)**
>
> *"Cùng prompt sao mỗi lần ra khác?"* → liên hệ **Bài 1**: hành vi **không xác định** cần kiểm soát; với lock, đó là **retry storm** khi thiết kế **backoff** sai.

---

## ⑨ Bài tập thực hành + tiêu chí tự chấm

**BT1 — Optimistic version + retry.**
Cài **optimistic version** cho `orders`, viết **retry 3 lần + jitter**.
> ✅ **Đạt khi:** **100 update song song** trên **1 row** → **không lost update**, log thấy **vài lần retry**, và **không retry vô hạn**.

**BT2 — Job queue với SKIP LOCKED.**
Dựng **job-queue trên Postgres** với **SKIP LOCKED**, **5 worker**.
> ✅ **Đạt khi:** **100 job** được xử lý **đúng 1 lần mỗi job**, **không worker nào block** chờ worker khác.

---

## ⑩ Đọc thêm / nguồn chuẩn

- **PostgreSQL docs** — *Explicit Locking* (`FOR UPDATE`, `SKIP LOCKED`).
- **TypeORM docs** — *Find Options / Locking* (**verify** hành vi `@VersionColumn`).
- **Hibernate docs** — *Optimistic & Pessimistic Locking* (mô hình tham chiếu kinh điển).
- **Kleppmann, DDIA** — **ch.7**.

---

> 🔚 **Hết Bài 2.** → **Bước 3 — Chốt ghi nhớ** (DỪNG chờ trả lời).
>
> 🎯 **Nhớ:** bài này có bẫy *"AI nói đã có optimistic locking nhưng chưa chắc"* — recall của bạn **nên kiểm tra đúng điểm đó**. Gợi ý 3 câu tự hỏi: *(1) Pessimistic vs optimistic — chọn theo gì? (2) ABA là gì và version column trị thế nào? (3) Đã có optimistic version thì có cần distributed lock không, vì sao?*
