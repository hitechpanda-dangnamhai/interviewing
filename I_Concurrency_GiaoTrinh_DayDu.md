# 🧠 Mục I — Concurrency & Consistency · Giáo trình đầy đủ để học

> Sinh theo **WORKFLOW 2 — Bước B (dựng giáo trình ngược) + Bước C (giảng từng bài)** từ ngân hàng `QB_I_concurrency.md` (71 câu, 7 mục con).
> Mỗi bài theo **Hợp đồng đầu ra 10 mục** (Mục 4 của project instruction).
> Ký hiệu nguồn: **📘** = có trong docs gốc roadmap · **➕** = bổ sung của giảng viên · **⬆️** = nâng cấp/làm sâu.
> **(verify)** = chạm tên/đời sản phẩm, phân loại CAP của DB cụ thể, cú pháp Redis/ZK/etcd, API ORM → đối chiếu docs chính thức tại thời điểm học (đổi nhanh).
>
> **Bối cảnh verify (tính 6/2026):** lý thuyết concurrency/consistency (CAP, PACELC, locking, consensus) **ổn định**. Các điểm đã đối chiếu khi soạn: redis.io docs hiện **khuyến nghị implement fencing token** cho mọi distributed locking system; phân loại CAP/PACELC: MongoDB (majority write concern) = CP / PC+EC, Cassandra default ONE = AP / PA+EL (QUORUM → PC+EC), etcd/ZooKeeper/CockroachDB/Spanner = CP, DynamoDB/Cassandra(low CL)/DNS = AP; TypeORM có `@VersionColumn` nhưng **version check không tự chạy trên mọi `update()`** (xem Bài 2 — bẫy). Vẫn verify lại khi học.

---

## 🗺️ BƯỚC B — Giáo trình ngược (dựng từ bộ đề)

Gom 71 câu thành **7 bài học** theo **dependency order** (cái sau cần cái trước), KHÔNG theo tần suất hỏi. Mạch: *race (vấn đề lõi) → lock (vũ khí cơ bản) → idempotency (an toàn khi lặp) → distributed lock (lên phân tán) → CAP (định luật đánh đổi) → consistency models (phổ nhất quán) → consensus (nền móng đứng sau tất cả)*.

| Bài | Tên bài | Phủ ID câu | Vì sao đặt ở đây |
|---|---|---|---|
| **1** | Race condition & 4 chiến lược chống race | `I-RACE-001…009` (9) | Vấn đề gốc. Mọi bài sau là *cách chống* race ở các phạm vi khác nhau. |
| **2** | Optimistic vs Pessimistic Locking | `I-LOCK-001…012` (12) | Vũ khí cơ bản nhất chống race **trong 1 DB**. Cần hiểu race trước. |
| **3** | Idempotency (góc concurrency/distributed) | `I-IDEM-001…006` (6) | "An toàn khi lặp" — dùng chính unique constraint/version của Bài 2; cầu nối sang phân tán. |
| **4** | Distributed Locks & Fencing Token | `I-DLOCK-001…012` (12) | Khi race vượt khỏi 1 DB. Cần lock (B2) + idempotency (B3); fencing token hé lộ nhu cầu consensus. |
| **5** | CAP & PACELC | `I-CAP-001…011` (11) | Định luật đánh đổi của hệ phân tán — khung tư duy cho mọi quyết định nhất quán. |
| **6** | Consistency Models & Eventual Consistency | `I-CONS-001…012` (12) | Phổ nhất quán chi tiết, mở rộng "C" của CAP thành nhiều mức. |
| **7** | Consensus (Raft/Paxos/Quorum/Leader Election) | `I-CONSENSUS-001…009` (9) | Nền móng đứng sau quorum (B5/B6), fencing token (B4), CP system. Tổng kết cả mục. |

**Cổng hoàn thành mục I (Bước E):** ĐẠT toàn bộ 71 câu khi phỏng vấn (Bước D), có chèn câu cũ ôn ngắt quãng (hôm nay → mai → 3 ngày → 1 tuần).

> 🧭 **Câu thần chú xuyên suốt:** *"Tôi không nhớ để gõ, tôi hiểu để chỉ huy — và tra cứu phần còn lại."* Concurrency là nơi AI viết **đúng cú pháp nhưng sai hành vi** nhiều nhất — bug chỉ hiện dưới tải thật. Người hiểu mới bắt được.

---
---

# 📘 BÀI 1 — Race condition & 4 chiến lược chống race

> Phủ: `I-RACE-001 … I-RACE-009` · Phase 8 (LLMOps/Production tư duy backend) nhưng đây là nền cho mọi mục I.

## ① Mục tiêu & vị trí trong mạch
Học xong bạn sẽ: định nghĩa được race condition trên **shared data** (không phải event-loop), nhận ra vì sao Node single-thread vẫn dính race, và **chọn đúng 1 trong 4 chiến lược** chống race cho từng tình huống. Đây là **bài gốc** — Bài 2 (lock), Bài 3 (idempotency), Bài 4 (distributed lock) đều là *các phạm vi khác nhau của cùng một bài toán này*.

## ② Giảng cơ bản → nâng cao

**(a) Trực giác.** Race condition = kết quả phụ thuộc vào **thứ tự/timing không kiểm soát** của các thao tác chạy đồng thời trên cùng một tài nguyên. Ẩn dụ: hai người cùng nhìn tờ giấy ghi "còn 1 vé", cả hai cùng gạch đi và bán → bán 2 vé cho 1 chỗ. Vấn đề không phải ở "ghi", mà ở **khoảng trống giữa lúc đọc và lúc ghi**.

**(b) Cơ chế chi tiết.**

- **Node single-thread mà vẫn race (`I-RACE-001`):** JS chạy 1 luồng, nhưng mỗi `await` **nhả quyền điều khiển** về event loop. Giữa lúc request A đọc `stock` và lúc A ghi `stock`, request B có thể xen vào đọc cùng giá trị cũ. Race ở đây là trên **shared data ngoài tiến trình** (DB/Redis), không phải tranh chấp biến trong RAM như ngôn ngữ đa luồng. *(Khác G-ASY: G bàn race do "đoán timing" trong event-loop; I bàn race trên dữ liệu chia sẻ.)*
- **Check-then-act / TOCTOU (`I-RACE-002`):** *Time-Of-Check to Time-Of-Use*. Bạn kiểm tra điều kiện (`if stock >= 1`) rồi mới hành động (`stock -= 1`), nhưng giữa hai bước đó state đã cũ. Hai request cùng đọc `stock=1`, cùng thấy "đủ", cùng trừ → **oversell** (bán quá tồn kho).
- **Atomic update cứu thế nào (`I-RACE-003`):** `UPDATE products SET stock = stock - 1 WHERE id = ? AND stock > 0` an toàn hơn "đọc ở app rồi ghi lại" vì DB **tự khóa row trong đúng một câu lệnh** — không có khoảng trống cho ai chen vào. Còn read-modify-write ở app (đọc → tính ở RAM → ghi đè) mở toang cửa **lost update**: hai request đọc cùng giá trị rồi ghi đè kết quả của nhau.
- **Counter < kỳ vọng (`I-RACE-005`):** 1000 request cùng `+1` mà ra < 1000 → kinh điển của lost update. Thủ phạm là counter đặt ở **app/cache** với read-modify-write. Sửa: atomic increment — DB `col = col + 1`, Redis `INCR`. Không cần lock toàn cục.

**(c) Mép giới hạn & sai lầm thường gặp.**

- **`I-RACE-006` — "thêm sleep/retry để né race" là anti-pattern:** fix dựa-timing cực giòn, chỉ làm bug *khó tái hiện hơn* chứ không biến mất. Cần cơ chế đồng bộ thật (atomic op / lock / unique constraint).
- **`I-RACE-007` — chỉ hiện trên prod:** dev 1 máy tải thấp → các request gần như tuần tự → không thấy race. Prod nhiều instance song song → concurrency thật. **In-process mutex / biến local KHÔNG bảo vệ across-instance.**
- **`I-RACE-008` (bẫy ⭐⭐⭐⭐⭐) — `Map`/biến đếm trong RAM 1 instance:** KHÔNG chống được race giữa nhiều instance, vì mỗi instance có state riêng, **không thấy nhau**. Cần một **shared store** (Redis/DB) làm điểm đồng bộ duy nhất. Bẫy kinh điển: "dùng object in-memory làm lock".
- **`I-RACE-004` — double-submit/double-click:** đúng là duplicate do concurrency (retry, đa tab). Chặn ở **server** bằng idempotency key + unique constraint, **KHÔNG** chỉ disable nút ở UI (UI fix không cứu retry mạng / nhiều tab). → cross-ref Bài 3.

**📘 vs ➕:** docs gốc roadmap chạm race ở mức khái niệm; phần TOCTOU, atomic-vs-RMW, across-instance, anti-pattern sleep/retry là **➕ bổ sung** theo chiều sâu Tech Lead.

## ③ ⚠️ Kiến thức cũ / cách làm bị thay thế

| Cái cũ (phổ biến / hay thấy trong code) | Thay bằng (nên dùng) | Vì sao |
|---|---|---|
| Đọc giá trị ở app → tính toán → ghi lại (read-modify-write) | **Atomic / conditional write** (`SET x = x - 1 WHERE x > 0`, Redis `INCR`) | RMW mở khoảng trống → lost update |
| `if (stock >= 1) { stock--; save() }` (check-then-act) | **Conditional update** đẩy điều kiện vào WHERE | TOCTOU bị chen giữa check và act |
| Thêm `sleep()` / retry mù để "né" race | Cơ chế đồng bộ thật (atomic op / lock / unique constraint) | Fix dựa-timing giòn, không trị gốc |
| `Map`/biến đếm in-memory làm "khoá" | **Shared store** (Redis/DB) làm điểm đồng bộ | Biến local không xuyên instance |
| Disable nút submit ở UI để chống duplicate | Idempotency key + unique constraint **ở server** | UI không cứu retry/đa tab/đa thiết bị |

## ④ Áp dụng thực tế + So sánh bigtech
**Use case thật:** flash-sale bán vé/đặt chỗ — 10.000 người cùng bấm mua 100 vé. Naive `if (còn vé) bán` → oversell hàng loạt.
**Kiến trúc tối thiểu:** đẩy quyết định vào DB bằng atomic conditional update `UPDATE ... SET remaining = remaining - 1 WHERE id=? AND remaining > 0`; affected rows = 0 → hết vé, trả lỗi. Không cần lock toàn cục, throughput cao.
**Bigtech pattern (phổ biến, không khẳng định tuyệt đối):** các hệ inventory/ticketing thường ưu tiên **atomic DB op hoặc partition theo key** (mỗi sản phẩm/khoá do một consumer xử lý) thay vì global lock, vì lock toàn cục giết throughput. Với hệ rất lớn, họ dồn về **single-writer per partition** (Kafka partition theo product_id) để biến concurrency thành tuần tự *cục bộ*. *(Pattern chung; chi tiết từng công ty thay đổi — verify khi cần.)*

## ⑤ Code thực hành + cấu hình

```sql
-- ✅ Atomic conditional update: chống oversell trong MỘT câu lệnh, không khoảng trống
-- affected_rows = 1 → trừ thành công; = 0 → hết hàng (KHÔNG ghi đè sai)
UPDATE products
SET    stock = stock - 1
WHERE  id = $1
  AND  stock > 0;
```

```typescript
// Node/TypeScript (ví dụ với 'pg' — verify version tại docs chính thức)
// package.json: "pg": "^8.13.0"   // pin version thật khi dùng
import { Pool } from 'pg';
const pool = new Pool(); // cấu hình qua biến môi trường PG* (KHÔNG hardcode)

async function buyOne(productId: string): Promise<boolean> {
  // Đẩy điều kiện vào WHERE → DB tự serialize trên row, không TOCTOU
  const res = await pool.query(
    `UPDATE products SET stock = stock - 1
     WHERE id = $1 AND stock > 0`,
    [productId],
  );
  // rowCount = 0 nghĩa là điều kiện stock > 0 không còn đúng → hết hàng
  return res.rowCount === 1;
}

// ❌ ANTI-PATTERN (đừng làm) — read-modify-write ở app, dính lost update:
// const { stock } = await getStock(productId);   // đọc
// if (stock > 0) await setStock(productId, stock - 1); // ghi đè → race
```

```typescript
// ❌ Bẫy I-RACE-008: in-memory "lock" KHÔNG xuyên instance
const inFlight = new Set<string>(); // chỉ tồn tại trong RAM của 1 process
// Trên 2 instance, mỗi cái có Set riêng → KHÔNG thấy nhau → vẫn race.
// → Phải dùng Redis/DB làm điểm đồng bộ (xem Bài 4).
```

**Cấu hình:** `.env` chứa `PGHOST/PGUSER/PGPASSWORD/PGDATABASE`; nạp bằng `dotenv`/secret manager; `requirements`/`package.json` pin version; chạy `node app.js`. Tuyệt đối không commit credential.

## ⑥ Keywords cần nhớ (glossary)
- 🧠 **Cần HIỂU:** race condition, TOCTOU / check-then-act, lost update, read-modify-write vs atomic write, across-instance vs in-process, "shared store làm điểm đồng bộ".
- 📌 **Cần THUỘC:** `UPDATE ... SET x = x - 1 WHERE x > 0`; Redis `INCR`; "affected rows = 0 → điều kiện không còn đúng".
- 🛠️ **Cần LÀM ĐƯỢC:** viết conditional update chống oversell; nhận diện counter-in-app là thủ phạm; chỉ ra vì sao in-memory lock vô dụng đa instance.

## ⑦ Mạch tư duy cần nhớ (mental model)
**"Khoảng trống giữa ĐỌC và GHI là nơi race sinh ra → bịt nó bằng một thao tác atomic, đừng vá bằng timing."** Và: *race là thuộc tính của phạm vi tài nguyên — biến local không bảo vệ được tài nguyên chia sẻ.*

## ⑧ Câu hỏi phỏng vấn & thách đố
1. *(dễ)* Race condition là gì? Node single-thread sao vẫn dính? → *gợi ý:* await nhả control, race trên shared data DB/Redis.
2. *(TB)* Vì sao `UPDATE SET bal = bal - 10 WHERE id=?` an toàn hơn đọc-rồi-ghi ở app? → atomic, không khoảng trống vs lost update.
3. *(khó)* 1000 request `+1` ra < 1000, debug từ đâu? → lost update do RMW; atomic increment; counter app/cache là thủ phạm.
4. *(khó)* Liệt kê 4 chiến lược chống race và chọn theo gì? → atomic write / pessimistic lock / optimistic lock+retry / unique constraint; chọn theo contention + chi phí retry + phạm vi tài nguyên.
- 🎯 **Thách đố/trick:** *"Tôi để một `Map` đếm request trong RAM để rate-limit, sao prod vẫn vượt giới hạn?"* → mỗi instance có Map riêng, không xuyên instance; cần Redis. · *"Thêm `await sleep(50)` thì hết lỗi rồi mà?"* → bug chỉ bị giấu, sẽ tái phát dưới tải khác; chưa trị gốc.

## ⑨ Bài tập thực hành + tiêu chí tự chấm
- **BT1:** Viết endpoint `POST /buy` chống oversell bằng conditional update. **Đạt khi:** chạy 500 request đồng thời trên stock=100 → đúng 100 thành công, 400 trả "hết hàng", DB `stock` không âm.
- **BT2:** Tái hiện lost update bằng read-modify-write rồi sửa bằng atomic increment. **Đạt khi:** chứng minh được con số sai → sửa → con số đúng, giải thích nguyên nhân bằng lời.

## ⑩ Đọc thêm / nguồn chuẩn
- PostgreSQL docs — *Concurrency Control / Row Locking*. · *Designing Data-Intensive Applications* (Kleppmann) ch.7 (race conditions, lost update). · Redis docs — `INCR` atomicity.

---
> 🔚 **Hết Bài 1.** Theo WORKFLOW, đây là chỗ chuyển sang **Bước 3 — Chốt ghi nhớ** (3 câu free-recall, hỏi xong DỪNG). Khi học live, đừng đọc tiếp Bài 2 vội — tự trả lời recall trước.

---

# 📘 BÀI 2 — Optimistic vs Pessimistic Locking

> Phủ: `I-LOCK-001 … I-LOCK-012` · Vũ khí cơ bản nhất chống race **trong một DB**.

## ① Mục tiêu & vị trí trong mạch
Học xong bạn sẽ: phân biệt **triết lý** hai loại lock, hiện thực được cả hai (version column / `FOR UPDATE`), chọn đúng theo **workload**, thiết kế retry an toàn cho optimistic, và biết khi nào lock là *thừa*. Bài này nối tiếp Bài 1: lock là cách *bịt khoảng trống đọc-ghi* khi atomic conditional update đơn lẻ không đủ (sửa nhiều field, cần báo conflict). Bài 4 sẽ nâng khái niệm này lên **across-instance** (distributed lock).

## ② Giảng cơ bản → nâng cao

**(a) Trực giác (`I-LOCK-001`).** Hai triết lý đối lập:
- **Pessimistic** ("bi quan"): *giả định conflict hay xảy ra* → **khóa trước** khi đụng data, ai đến sau phải chờ. Như mượn chìa khóa phòng họp: bạn giữ chìa, người khác đứng ngoài.
- **Optimistic** ("lạc quan"): *giả định conflict hiếm* → cho mọi người chạy song song, **chỉ kiểm tra lúc commit** xem có ai sửa chen vào không; nếu có → fail và làm lại. Như sửa Google Doc rồi lúc lưu mới báo "ai đó vừa sửa, tải lại".

**(b) Cơ chế chi tiết.**

- **Optimistic = version column / CAS (`I-LOCK-002`):** mỗi row có cột `version` (hoặc timestamp). Update kèm điều kiện:
  `UPDATE t SET ..., version = version + 1 WHERE id = ? AND version = ?`.
  Nếu **affected rows = 0** → có người tăng version trước bạn → conflict → fail + retry. Đây chính là **Compare-And-Swap** ở tầng row. (TypeORM `@VersionColumn` — *verify*, xem bẫy ở Mục ③.)
- **Pessimistic = lock ở DB (`I-LOCK-003`):** `SELECT ... FOR UPDATE` (exclusive) / `FOR SHARE`. Khóa row đến hết transaction; tx khác đụng row đó phải **chờ**. *(Không lặp lý thuyết isolation của mục E — ở đây nhìn từ góc chiến lược.)*
- **`SELECT ... FOR UPDATE SKIP LOCKED` (`I-LOCK-009`):** giải bài "hàng đợi trong DB". Nhiều worker cùng quét bảng job: mỗi worker **bỏ qua** row đang bị khóa thay vì chờ → mỗi worker lấy job *khác nhau*, giảm contention. Nền của nhiều job-queue-trên-Postgres.

**(c) Chọn cái nào & mép giới hạn.**

- **Theo workload (`I-LOCK-004`):** contention **thấp** + read-heavy + retry rẻ/idempotent → **optimistic**. Contention **cao** + ghi nóng cùng row + retry đắt → **pessimistic**. Nguyên tắc: **đo conflict rate**, đừng chọn cảm tính.
- **Bẫy ngược (`I-LOCK-005`):** optimistic dưới **contention cao** có thể *tệ hơn* pessimistic — nhiều tx cùng đụng 1 row → đa số fail version-check → **retry dồn dập** (wasted work, gần livelock). Pessimistic serialize nhưng mỗi tx tiến đều, không phí công.
- **ABA problem (`I-LOCK-006`, ⭐⭐⭐⭐⭐):** giá trị đổi A→B→A. Nếu check "bằng giá trị cũ" thì tưởng *không đổi* → bỏ sót thay đổi giữa chừng. **Version đơn điệu tăng (monotonic)** phát hiện được vì version đã nhảy dù giá trị quay về như cũ. → đây là lý do version column ăn đứt "so sánh giá trị".
- **Deadlock với pessimistic (`I-LOCK-007`):** nhiều `FOR UPDATE` khóa chéo theo thứ tự ngược → deadlock. Phòng: khóa theo **thứ tự nhất quán** (luôn lock id nhỏ trước), tx **ngắn**, lock **granular** (row > table). *(cross-ref E-ISO.)*
- **Lạm dụng `FOR UPDATE` (`I-LOCK-010`):** "cứ FOR UPDATE cho chắc" trên mọi read → serialize không cần thiết, giữ lock + connection lâu → block, deadlock, **cạn connection pool**. Chỉ khóa đúng điểm nóng read-modify-write.
- **Retry đúng cách cho optimistic (`I-LOCK-008`):** retry phải **có giới hạn** (N lần) + **jittered backoff** (tránh các retry đồng pha lại đụng nhau); mỗi lần retry phải **đọc lại state mới** rồi tính lại; fail hẳn sau N lần → trả lỗi rõ ràng thay vì retry vô hạn.

**(d) Khi nào lock là THỪA.**
- **Version column vs atomic conditional update (`I-LOCK-011`):** với counter/stock đơn → `stock = stock - 1 WHERE stock > 0` *gọn nhất*, khỏi version. Version column hợp khi **sửa nhiều field** hoặc cần **báo conflict cho user** (form edit). 
- **Multi-instance có cần distributed lock? (`I-LOCK-012`, ⭐⭐⭐⭐⭐):** Nếu đã dùng optimistic version column thì **không** cần thêm distributed lock — version check ở DB **đã là điểm đồng bộ atomic across-instance**. Thêm distributed lock thường thừa và giảm throughput. Nhận thức cốt lõi: **DB constraint chính là một cơ chế concurrency control.**

**📘 vs ➕:** docs gốc chạm optimistic/pessimistic ở mức khái niệm; ABA, retry design, SKIP LOCKED, "lock thừa khi đã có version" là **➕/⬆️**.

## ③ ⚠️ Kiến thức cũ / cách làm bị thay thế

| Cái cũ | Thay bằng (nên dùng) | Vì sao |
|---|---|---|
| So sánh "bằng giá trị cũ" để phát hiện thay đổi | **Version column đơn điệu tăng** (hoặc timestamp) | Tránh ABA problem |
| `FOR UPDATE` trên mọi read "cho chắc" | Chỉ khóa đúng điểm nóng read-modify-write | Tránh cạn pool, block, deadlock |
| Optimistic + retry **vô hạn**, không backoff | Retry **có giới hạn** + jittered backoff + đọc lại state | Tránh livelock/retry storm |
| Thêm distributed lock khi đã có optimistic version | Tin vào version check ở DB (đã atomic across-instance) | Lock thừa, giảm throughput |
| Pessimistic lock theo thứ tự tùy tiện | Lock theo **thứ tự nhất quán** + tx ngắn | Tránh deadlock |
| Worker queue: nhiều worker `FOR UPDATE` chờ nhau | `FOR UPDATE SKIP LOCKED` | Mỗi worker lấy job khác nhau, hết block |

## ④ Áp dụng thực tế + So sánh bigtech
**Use case:** chỉnh sửa hồ sơ/đơn hàng nhiều field từ nhiều admin. Dùng **optimistic version**: ai lưu sau thấy version lệch → "bản ghi vừa được người khác sửa, tải lại". UX rõ ràng, không khóa người dùng.
**Use case 2:** trừ số dư ví tài chính dưới contention cao trên *cùng một ví* → **pessimistic** `FOR UPDATE` (hoặc atomic update) để mỗi tx tiến chắc, tránh retry storm.
**Bigtech pattern:** ORM/framework lớn (Hibernate, EF Core, TypeORM, ActiveRecord) đều cung cấp optimistic locking qua version/timestamp column như **default cho web app** (contention thường thấp). Job-queue-trên-DB hiện đại (nhiều hệ Postgres-backed queue) dựa `SKIP LOCKED`. *(Pattern phổ biến; verify API cụ thể.)*

## ⑤ Code thực hành + cấu hình

```sql
-- ✅ OPTIMISTIC: version column + CAS. affected=0 ⇒ conflict ⇒ retry.
UPDATE orders
SET    status = 'paid', version = version + 1
WHERE  id = $1 AND version = $2;
```

```sql
-- ✅ PESSIMISTIC: khóa row tới hết tx. Tx khác đụng row này phải chờ.
BEGIN;
SELECT balance FROM wallets WHERE id = $1 FOR UPDATE;  -- khóa exclusive
-- ... tính toán an toàn ở đây, không ai chen được ...
UPDATE wallets SET balance = balance - $2 WHERE id = $1;
COMMIT;
```

```sql
-- ✅ JOB QUEUE trong DB: mỗi worker lấy job khác nhau, không chờ nhau
SELECT id FROM jobs
WHERE  status = 'pending'
ORDER  BY created_at
LIMIT  1
FOR UPDATE SKIP LOCKED;   -- bỏ qua row đang bị worker khác khóa
```

```typescript
// ✅ Retry an toàn cho optimistic lock (giới hạn + jittered backoff + đọc lại)
async function updateWithOptimisticRetry(
  id: string,
  mutate: (cur: Order) => Partial<Order>,
  maxRetries = 3,                       // CÓ giới hạn — không retry vô hạn
): Promise<Order> {
  for (let attempt = 0; attempt < maxRetries; attempt++) {
    const cur = await getOrder(id);     // ĐỌC LẠI state mới mỗi vòng (quan trọng)
    const patch = mutate(cur);
    const res = await db.query(
      `UPDATE orders SET status = $1, version = version + 1
       WHERE id = $2 AND version = $3`,
      [patch.status, id, cur.version],
    );
    if (res.rowCount === 1) return { ...cur, ...patch, version: cur.version + 1 };
    // conflict → backoff có jitter để tránh các retry đồng pha lại đụng nhau
    await sleep(2 ** attempt * 20 + Math.random() * 20);
  }
  throw new Error('OPTIMISTIC_CONFLICT: hết số lần thử, vui lòng tải lại');
}
```

```typescript
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

**Cấu hình:** transaction phải **ngắn** (đừng gọi API ngoài khi đang giữ `FOR UPDATE` → giữ lock lâu, cạn pool); set `statement_timeout`/`lock_timeout` để không treo vô hạn; secrets qua env.

## ⑥ Keywords cần nhớ
- 🧠 **Cần HIỂU:** pessimistic vs optimistic philosophy, CAS, ABA problem, contention rate, livelock/retry storm, "DB constraint = concurrency control".
- 📌 **Cần THUỘC:** `UPDATE ... SET ..., version=version+1 WHERE id=? AND version=?`; `SELECT ... FOR UPDATE [SKIP LOCKED]`; affected-rows=0 ⇒ conflict.
- 🛠️ **Cần LÀM ĐƯỢC:** triển khai cả hai loại lock; viết retry có giới hạn + jitter + đọc lại; nhận diện lock thừa.

## ⑦ Mạch tư duy cần nhớ (mental model)
**"Pessimistic = khóa-rồi-mới-làm (chờ); Optimistic = làm-rồi-kiểm-tra-lúc-commit (retry). Chọn theo conflict rate, không theo cảm tính."** Và: *version đơn điệu tăng đánh bại ABA; retry phải có trần + jitter + đọc lại.*

## ⑧ Câu hỏi phỏng vấn & thách đố
1. *(dễ)* Optimistic vs pessimistic khác nhau ở triết lý nào?
2. *(TB)* Optimistic hiện thực thế nào? Mô tả version column / CAS.
3. *(khó)* Khi nào optimistic *tệ hơn* pessimistic? → contention cao → retry storm.
4. *(khó)* ABA problem là gì, version column trị thế nào?
5. *(khó)* App multi-instance dùng optimistic version còn cần distributed lock không? → không, version check đã atomic across-instance.
- 🎯 **Thách đố/trick:** *"AI viết `repository.update(id, { status })` với entity có `@VersionColumn` và bảo 'đã có optimistic locking' — đúng không?"* → **Cẩn thận**: cú pháp đúng nhưng hành vi có thể sai — TypeORM **không phải lúc nào cũng tự thêm `WHERE version=?`** trên update một phần; version check tin cậy khi `save()` full entity hoặc `findOne` kèm lock option. Đây đúng kiểu *AI viết đúng cú pháp nhưng sai hành vi* — phải kiểm chứng tại docs và **test bằng update song song**, đừng tin lời cam kết của ORM mặc định. *(verify)* · *"Cùng prompt sao mỗi lần ra khác?"* → liên hệ Bài 1: hành vi không xác định cần kiểm soát; với lock là retry storm khi thiết kế sai backoff.

## ⑨ Bài tập thực hành + tiêu chí tự chấm
- **BT1:** Cài optimistic version cho `orders`, viết retry 3 lần + jitter. **Đạt khi:** 100 update song song trên 1 row → không lost update, log thấy vài lần retry, không retry vô hạn.
- **BT2:** Dựng job-queue trên Postgres với `SKIP LOCKED`, 5 worker. **Đạt khi:** 100 job được xử lý đúng 1 lần mỗi job, không worker nào block chờ worker khác.

## ⑩ Đọc thêm / nguồn chuẩn
- PostgreSQL docs — *Explicit Locking* (`FOR UPDATE`, `SKIP LOCKED`). · TypeORM docs — *Find Options / Locking* (verify `@VersionColumn` behavior). · Hibernate docs — *Optimistic & Pessimistic Locking* (mô hình tham chiếu kinh điển). · Kleppmann, DDIA ch.7.

---
> 🔚 **Hết Bài 2.** → Bước 3 Chốt ghi nhớ (DỪNG chờ trả lời). Nhớ: bài này có **bẫy "AI nói đã có optimistic locking nhưng chưa chắc"** — recall nên kiểm tra điểm đó.

---

# 📘 BÀI 3 — Idempotency (góc concurrency / distributed)

> Phủ: `I-IDEM-001 … I-IDEM-006` · "An toàn khi lặp" — cầu nối từ single-DB sang phân tán.
> *Phạm vi:* I nhìn idempotency từ **retry / đa luồng / distributed**. Cơ chế HTTP (`Idempotency-Key` header, method semantics RFC 9110, TTL/scope) thuộc **F-IDEM** — không lặp ở đây.

## ① Mục tiêu & vị trí trong mạch
Học xong bạn sẽ: hiểu vì sao **retry + at-least-once** bắt buộc consumer idempotent, biết "exactly-once delivery" là ảo tưởng và cách đạt **effectively-once**, thiết kế **dedup key atomic**, phân biệt thao tác *tự* idempotent vs *phải làm cho* idempotent, và nắm điểm chí mạng: **idempotency một mình chưa đủ khi 2 retry chạy song song**. Bài này dùng chính unique constraint/version của Bài 2, và mở đường sang Bài 4 (distributed) + mục K (messaging).

## ② Giảng cơ bản → nâng cao

**(a) Trực giác.** Idempotent = **lặp lại không đổi kết quả**. Bấm thang máy 5 lần vẫn gọi đúng 1 thang. `SET balance = 100` lặp bao nhiêu lần vẫn ra 100; nhưng `balance += 10` lặp 2 lần thì sai gấp đôi.

**(b) Cơ chế chi tiết.**

- **Vì sao bắt buộc (`I-IDEM-001`):** mạng và message queue **sẽ giao trùng** — timeout rồi retry, broker redeliver. Nếu consumer không idempotent → tính tiền/gửi mail/ghi đơn **2 lần**. At-least-once là mặc định an toàn của hầu hết queue, nên consumer *phải* chịu được trùng.
- **Exactly-once delivery là ảo tưởng (`I-IDEM-002`):** vấn đề Two Generals — bên gửi không bao giờ chắc chắn 100% bên nhận đã nhận *đúng một lần* qua mạng không tin cậy. Thực tế đạt **effectively-once = at-least-once + consumer idempotent/dedup**. (Cái mà Kafka gọi "exactly-once" cũng là dedup + transaction nội bộ, không phá định lý này.)
- **Dedup key atomic ở đâu (`I-IDEM-003`):** lưu key bằng **unique constraint / `INSERT ... ON CONFLICT DO NOTHING`** để *claim* key atomic — **một** request thắng, các bản trùng bị từ chối. **KHÔNG** check-then-act ("SELECT xem có chưa → nếu chưa thì INSERT") vì đó lại là TOCTOU (Bài 1). Key theo **business id**, TTL hợp lý.

**(c) Phân loại & bẫy chí mạng.**

- **Naturally vs phải-làm-cho (`I-IDEM-004`):** *tự idempotent* = set tuyệt đối (`PUT state=X`, `SET balance=100`, `DELETE id=5`). *Phải thêm cơ chế* = delta/create (`+10`, `INSERT order`) → gắn **dedup key** hoặc **version/CAS**. Kỹ năng: nhìn một thao tác và biết nó thuộc loại nào.
- **`I-IDEM-005` (⭐⭐⭐⭐⭐) — idempotent NHƯNG 2 retry chạy SONG SONG cùng key:** vẫn race! Idempotency một mình **chưa đủ** — nó nói "lặp *tuần tự* không sao", không nói gì về *song song*. Hai request cùng key chạy đồng thời, cả hai cùng thấy "chưa có" → cùng INSERT/cùng side-effect. Cần **atomic claim (unique constraint)** hoặc lock để **serialize**. → idempotency + concurrency control phải đi cùng nhau.
- **Liên hệ optimistic lock & outbox (`I-IDEM-006`):** version/CAS làm write idempotent *theo state* (Bài 2). **Outbox pattern** + dedup đảm bảo publish event không trùng/không mất (ghi event vào cùng tx với data, relay sau). Cùng một mục tiêu: *an toàn khi lặp*. → cross-ref mục K (Outbox).

**📘 vs ➕:** đây gần như toàn bộ **➕** ở góc concurrency (docs gốc không đào sâu); phần HTTP idempotency là F.

## ③ ⚠️ Kiến thức cũ / cách làm bị thay thế

| Cái cũ | Thay bằng (nên dùng) | Vì sao |
|---|---|---|
| Tin vào "exactly-once delivery" của queue | **at-least-once + consumer idempotent** (effectively-once) | Exactly-once delivery bất khả thi qua mạng |
| Dedup bằng `SELECT rồi INSERT nếu chưa có` | `INSERT ... ON CONFLICT DO NOTHING` / unique constraint | SELECT-then-INSERT là TOCTOU |
| Coi mọi handler là idempotent vì "đã có idempotency key" | Thêm **atomic claim** cho key chạy song song | Idempotency ≠ chống song song |
| Side-effect (charge, email) chạy trước khi claim key | **Claim key atomic trước**, side-effect sau (và idempotent hoá chính side-effect) | Tránh double-charge khi trùng |

## ④ Áp dụng thực tế + So sánh bigtech
**Use case:** webhook thanh toán (Stripe-style) gửi `payment.succeeded` — provider **retry** đến khi nhận 2xx → bạn nhận trùng. Handler phải claim `event_id` qua unique constraint trước, rồi xử lý; lần trùng thấy key đã tồn tại → trả 200 và bỏ qua.
**Bigtech pattern:** các payment API lớn yêu cầu client gửi **idempotency key** và họ dedup ở server (đúng kiểu F-IDEM); message platform (Kafka) cung cấp "exactly-once semantics" *nội bộ* bằng producer idempotent + transaction, nhưng phía consumer vẫn nên idempotent. *(Pattern phổ biến; verify chi tiết API.)*

## ⑤ Code thực hành + cấu hình

```sql
-- ✅ Atomic claim dedup key: 1 thắng, trùng bị từ chối — KHÔNG check-then-act
CREATE TABLE processed_events (
  event_id   TEXT PRIMARY KEY,        -- unique constraint = điểm dedup atomic
  created_at TIMESTAMPTZ DEFAULT now()
);

-- Trả về 1 dòng nếu claim thành công; 0 dòng nếu đã xử lý (trùng)
INSERT INTO processed_events (event_id)
VALUES ($1)
ON CONFLICT (event_id) DO NOTHING
RETURNING event_id;
```

```typescript
// ✅ Webhook handler idempotent + atomic claim (chống cả retry tuần tự lẫn song song)
async function handlePaymentWebhook(eventId: string, payload: Payload) {
  // 1) CLAIM key atomic TRƯỚC khi gây side-effect
  const claimed = await db.query(
    `INSERT INTO processed_events (event_id) VALUES ($1)
     ON CONFLICT (event_id) DO NOTHING RETURNING event_id`,
    [eventId],
  );
  if (claimed.rowCount === 0) {
    return { status: 200, note: 'đã xử lý — bỏ qua bản trùng' }; // idempotent
  }
  // 2) Chỉ ĐẾN ĐÂY khi mình là người thắng claim → làm side-effect đúng 1 lần
  await fulfillOrder(payload);
  // (lý tưởng: gói claim + side-effect trong cùng transaction để all-or-nothing)
}

// ❌ ANTI-PATTERN: check-then-act (TOCTOU) — 2 webhook song song cùng lọt
// const seen = await db.query('SELECT 1 FROM processed_events WHERE event_id=$1', [eventId]);
// if (seen.rowCount === 0) { await db.query('INSERT ...'); await fulfillOrder(); }
```

**Cấu hình:** đặt side-effect *sau* claim; lý tưởng gói trong **một transaction** (claim + ghi data) để all-or-nothing; nếu side-effect là gọi service ngoài (không rollback được) → kết hợp **outbox** (ghi "cần gửi" vào cùng tx, relay sau) thay vì gọi trực tiếp trong tx.

## ⑥ Keywords cần nhớ
- 🧠 **Cần HIỂU:** idempotency, at-least-once vs exactly-once, effectively-once, dedup, "idempotent ≠ chống song song", outbox.
- 📌 **Cần THUỘC:** `INSERT ... ON CONFLICT DO NOTHING RETURNING`; "claim key trước, side-effect sau"; naturally-idempotent = absolute set, delta/create = cần dedup.
- 🛠️ **Cần LÀM ĐƯỢC:** viết webhook handler idempotent; thiết kế dedup key atomic; nhận diện thao tác cần thêm cơ chế.

## ⑦ Mạch tư duy cần nhớ (mental model)
**"Mạng sẽ giao trùng → consumer phải lặp-không-đổi. Nhưng idempotent chỉ cứu lặp TUẦN TỰ; lặp SONG SONG cần thêm atomic claim."** Và: *exactly-once delivery không tồn tại — chỉ có at-least-once + dedup = effectively-once.*

## ⑧ Câu hỏi phỏng vấn & thách đố
1. *(TB)* Vì sao retry + at-least-once bắt buộc consumer idempotent?
2. *(khó)* "Exactly-once delivery" có thật không? Đạt effectively-once bằng gì?
3. *(khó)* Thiết kế dedup key chống trùng dưới concurrency — lưu ở đâu cho atomic?
4. *(khó)* Phân biệt naturally idempotent vs phải-làm-cho-idempotent, cho ví dụ.
- 🎯 **Thách đố/trick (⭐⭐⭐⭐⭐):** *"Handler của tôi đã idempotent rồi, sao 2 webhook đến cùng lúc vẫn tạo 2 order?"* → idempotency không chống **song song**; thiếu atomic claim → cả hai cùng lọt; phải dùng unique constraint/`ON CONFLICT` để serialize. · *"AI viết dedup bằng `SELECT ... if not exists then INSERT` — ổn chứ?"* → **không** — đó là TOCTOU; phải atomic `INSERT ON CONFLICT`. Lại là *AI viết đúng cú pháp nhưng sai hành vi*.

## ⑨ Bài tập thực hành + tiêu chí tự chấm
- **BT1:** Webhook idempotent với `processed_events`. **Đạt khi:** gửi cùng `event_id` 5 lần (kể cả 2 lần đồng thời) → side-effect chạy đúng 1 lần, 4 lần còn lại trả 200 "đã xử lý".
- **BT2:** Phân loại 6 thao tác cho sẵn (`PUT name`, `+10 điểm`, `DELETE`, `INSERT comment`, `SET status='done'`, `gửi email`) thành tự-idempotent vs phải-thêm-cơ-chế. **Đạt khi:** phân đúng + nêu cơ chế cần thêm cho nhóm sau.

## ⑩ Đọc thêm / nguồn chuẩn
- RFC 9110 §9.2 (idempotent methods) — cho góc HTTP (F). · Stripe docs — *Idempotent Requests* (mô hình tham chiếu). · *Two Generals' Problem* (cơ sở của "exactly-once bất khả thi"). · microservices.io — *Transactional Outbox / Idempotent Consumer*.

---
> 🔚 **Hết Bài 3.** → Chốt ghi nhớ. Điểm nóng để recall: **"idempotent nhưng vẫn race khi song song"**.

---

# 📘 BÀI 4 — Distributed Locks & Fencing Token

> Phủ: `I-DLOCK-001 … I-DLOCK-012` · Khi race vượt khỏi một DB transaction.

## ① Mục tiêu & vị trí trong mạch
Học xong bạn sẽ: biết **khi nào cần** distributed lock (và khi nào *tránh*), viết lock Redis tối thiểu đúng (`SET NX PX` + release-if-token-match), hiểu **fencing token** cứu cái gì mà TTL không cứu được, nắm tranh luận **Redlock** (Kleppmann ⇄ antirez), và phân biệt lock "for efficiency" vs "for correctness". Bài này dùng quorum (sẽ đào ở Bài 5/7) và là **động lực dẫn tới consensus** (Bài 7).

## ② Giảng cơ bản → nâng cao

**(a) Trực giác (`I-DLOCK-001`).** Distributed lock = mutual exclusion **across process/instance/service** cho tài nguyên **không nằm gọn trong một DB transaction**: gọi API bên ngoài, cron "chỉ chạy một lần", ghi vào nhiều store. Khi `FOR UPDATE` (Bài 2) không phủ được phạm vi → cần điểm khóa chung bên ngoài (Redis/ZK/etcd).

**(b) Cơ chế chi tiết.**

- **Lock Redis tối thiểu (`I-DLOCK-002`):** `SET key <token> NX PX <ttl>`.
  - **NX** = set-if-not-exists → **claim atomic**, chỉ một client thắng.
  - **PX/TTL** = tự nhả nếu holder chết → tránh **deadlock vĩnh viễn**. *(verify cú pháp tại redis.io.)*
- **Release phải "delete-if-token-matches" (`I-DLOCK-003`):** KHÔNG `DEL` thẳng. Nếu lock của bạn đã hết TTL và **client khác đã chiếm**, `DEL` thẳng sẽ **xóa nhầm lock người khác**. Phải so token sở hữu rồi mới xóa, **atomic bằng Lua script** (vì get-rồi-del là hai thao tác → lại TOCTOU).
- **`SET NX` của Redis hiện đại đã gộp set+expire atomic** — đừng tách `SETNX` rồi `EXPIRE` riêng (nếu crash giữa hai lệnh → lock không bao giờ hết hạn).

**(c) TTL, failover & kẻ thù timing.**

- **TTL ngắn vs dài (`I-DLOCK-004`):** ngắn → lock hết hạn *giữa* critical section → 2 holder cùng lúc. Dài → holder chết thì tài nguyên kẹt lâu. Cần **watchdog/lease-renewal** (gia hạn định kỳ) hoặc ước lượng đúng + **fencing**.
- **GC pause / process pause là kẻ thù (`I-DLOCK-008`):** holder bị pause (GC, stop-the-world, scheduler) lâu hơn TTL → lock hết hạn → client khác chiếm → holder **tỉnh dậy vẫn tưởng còn giữ lock** và ghi tiếp → vi phạm mutual exclusion. **TTL không chặn được điều này** vì process tỉnh dậy không biết mình đã mất lock.
- **Redis master–replica failover phá lock (`I-DLOCK-011`):** replication **async** — lock `SET` trên master chưa kịp sang replica, master chết, replica lên **không có lock** → 2 client cùng giữ. Đây chính là động cơ đẻ ra Redlock (và Redlock *cũng* không kín — xem dưới). *(verify.)*

**(d) Fencing token & Redlock — phần lõi senior/TL.**

- **Fencing token (`I-DLOCK-006`, ⭐⭐⭐⭐⭐):** một **số đơn điệu tăng** cấp mỗi lần acquire. **Resource (DB/storage) từ chối write mang token NHỎ HƠN** token lớn nhất nó từng thấy → chặn holder "zombie" (bị pause) ghi đè sau khi lock đã chuyển sang người khác. Đây là thứ **TTL không làm được**: TTL chỉ giải phóng lock, không ngăn process cũ tỉnh dậy ghi bậy. Fencing cần resource *biết kiểm tra token* (không phải lock nào cũng tự có).
- **Redlock & phê phán Kleppmann (`I-DLOCK-005`, ⭐⭐⭐⭐⭐):** Redlock = acquire trên **N Redis độc lập** (không replicate giữa nhau), cần **majority (N/2+1)** trong thời gian < TTL. Kleppmann phê phán hai điểm: **(a) không tự sinh fencing token**; **(b) phụ thuộc giả định timing** (clock skew, GC pause, network delay) → vẫn có thể 2 client cùng giữ lock. redis.io docs hiện cũng **nhắc đọc kỹ phần phản biện và khuyên implement fencing token**, lưu ý Redis dùng *wall-clock* (không monotonic) cho TTL nên dịch giờ có thể gây lỗi. *(verify; nguồn martin.kleppmann.com ⇄ antirez ⇄ redis.io docs.)*
- **Efficiency vs Correctness (`I-DLOCK-007`):** Kleppmann/antirez đồng thuận thực dụng:
  - **for efficiency** (tránh làm trùng, sai chút không chết — vd né chạy cron 2 lần) → single-node Redis `SET NX` là **đủ**.
  - **for correctness** (sai = mất tiền/hỏng data) → dùng **consensus system (ZooKeeper/etcd)** + **bắt buộc fencing token**.
- **ZK/etcd khác Redis (`I-DLOCK-009`):** dựa **consensus** (ZAB/Raft) + ephemeral node/lease + thứ tự → tự sinh thứ tự đơn điệu (gần như fencing sẵn), tự nhả khi session chết; an toàn hơn cho correctness, đổi lại **nặng/chậm hơn**. *(verify.)*

**(e) Khi nào TRÁNH distributed lock (`I-DLOCK-012`, ⭐⭐⭐⭐⭐).**
Nếu thay được bằng **atomic DB op / unique constraint / idempotency / partition theo key** (mỗi key một consumer) thì **ưu tiên những cái đó**. Distributed lock là "code smell" dễ sai (TTL, failover, fencing, clock). Chỉ dùng khi thật sự cần exclusion *across-resource* mà không cách nào khác.
- **Cron chỉ chạy 1 lần dù nhiều instance (`I-DLOCK-010`):** leader election / distributed lock TTL ngắn + renew, hoặc DB advisory lock / unique row theo schedule-slot. **Đừng giả định "chỉ deploy 1 instance".**

**📘 vs ➕:** docs gốc hầu như không đụng; toàn bộ là **➕** chiều sâu TL. Đây là bài "phân biệt senior với TL thật" — fencing token & Redlock critique là tâm điểm.

## ③ ⚠️ Kiến thức cũ / cách làm bị thay thế

| Cái cũ | Thay bằng (nên dùng) | Vì sao |
|---|---|---|
| `SETNX key` rồi `EXPIRE key` (2 lệnh) | `SET key token NX PX ttl` (1 lệnh atomic) | Crash giữa 2 lệnh → lock vĩnh viễn |
| `DEL key` để nhả lock | **Lua: `if get==token then del`** | DEL thẳng xóa nhầm lock người khác |
| Tin TTL là đủ cho correctness | TTL + **fencing token** (+ consensus nếu cần đúng tuyệt đối) | GC/process pause vượt TTL → 2 holder |
| Redlock = "an toàn cho correctness" | for-correctness → **ZK/etcd + fencing**; Redlock chỉ for-efficiency | Phụ thuộc timing, không fencing |
| Lock trên Redis single master cho việc sống-còn | Consensus store có lease/thứ tự | Async failover mất lock |
| Mặc định distributed lock cho mọi exclusion | Ưu tiên atomic op / unique / idempotency / partition-by-key | Lock dễ sai, đắt |
| "Cron chạy 1 lần vì chỉ deploy 1 instance" | Leader election / advisory lock | Giả định 1-instance sẽ vỡ khi scale |

## ④ Áp dụng thực tế + So sánh bigtech
**Use case efficiency:** tránh hai instance cùng chạy job tổng hợp báo cáo (chạy trùng chỉ tốn CPU) → single-node Redis `SET NX PX` đủ.
**Use case correctness:** chỉ một node được phép ghi vào file/ledger chia sẻ → ZooKeeper/etcd lease + **fencing token** mà storage kiểm tra.
**Bigtech pattern:** Kubernetes dùng **etcd** (Raft) cho coordination/leader election; nhiều hệ dùng ZooKeeper cho controller election (kiểu Kafka cũ). redis.io liệt kê nhiều thư viện Redlock (Redisson/Java, node-redlock, Redsync/Go...) nhưng *kèm cảnh báo fencing*. Khuynh hướng TL: **đẩy correctness về consensus store, chỉ dùng Redis lock cho efficiency.** *(verify landscape.)*

## ⑤ Code thực hành + cấu hình

```typescript
// ✅ Lock Redis tối thiểu ĐÚNG (ioredis) — verify cú pháp tại redis.io
// "ioredis": "^5.x" (pin thật)
import Redis from 'ioredis';
import { randomUUID } from 'crypto';
const redis = new Redis(process.env.REDIS_URL!); // KHÔNG hardcode

async function withRedisLock<T>(key: string, ttlMs: number, fn: () => Promise<T>) {
  const token = randomUUID();                 // token sở hữu, để release đúng chủ
  // NX = claim atomic; PX = TTL tránh deadlock vĩnh viễn nếu holder chết
  const ok = await redis.set(`lock:${key}`, token, 'PX', ttlMs, 'NX');
  if (ok !== 'OK') throw new Error('LOCK_BUSY');
  try {
    return await fn();
  } finally {
    // ✅ release atomic: chỉ xóa NẾU token còn là của mình (tránh xóa nhầm)
    await redis.eval(
      `if redis.call('get', KEYS[1]) == ARGV[1]
         then return redis.call('del', KEYS[1]) else return 0 end`,
      1, `lock:${key}`, token,
    );
  }
}
// ⚠️ Lock này chỉ AN TOÀN cho "efficiency". Nếu sai = mất tiền/hỏng data
//    → cần consensus store (etcd/ZooKeeper) + FENCING TOKEN ở resource.
```

```text
// 🧠 Fencing token (ý tưởng — TTL không thay thế được):
acquire() → trả token đơn điệu tăng: 33, rồi 34, 35...
Resource (DB/storage) lưu max_token đã thấy.
Ghi kèm token < max_token  →  TỪ CHỐI (chặn holder zombie tỉnh dậy sau pause).
→ Redis SET NX KHÔNG tự sinh chuỗi này; ZooKeeper/etcd có thứ tự sẵn.
```

**Cấu hình:** TTL phải > thời gian critical section *thực tế* (đo, đừng đoán); cân nhắc **lease-renewal/watchdog** cho job dài; đừng gọi API ngoài chậm khi đang giữ lock TTL ngắn; với correctness → chuyển sang etcd/ZK; secrets qua env.

## ⑥ Keywords cần nhớ
- 🧠 **Cần HIỂU:** distributed lock, fencing token, Redlock & critique, efficiency vs correctness lock, GC-pause hazard, async-failover hazard, "tránh lock nếu có cách atomic".
- 📌 **Cần THUỘC:** `SET key token NX PX ttl`; Lua release-if-token-match; Redlock = N Redis độc lập + majority < TTL.
- 🛠️ **Cần LÀM ĐƯỢC:** viết Redis lock đúng (NX/PX + Lua release); thiết kế cron single-run đa instance; chỉ ra chỗ cần fencing.

## ⑦ Mạch tư duy cần nhớ (mental model)
**"Distributed lock = mutual exclusion across-resource. TTL chống deadlock nhưng KHÔNG chống holder-zombie — chỉ fencing token (số đơn điệu tăng) làm được. Lock cho efficiency dùng Redis; cho correctness dùng consensus + fencing. Tốt nhất là tránh lock."**

## ⑧ Câu hỏi phỏng vấn & thách đố
1. *(TB)* Distributed lock là gì, khi nào cần so với DB row lock?
2. *(TB)* `SET key token NX PX ttl` — vì sao cần NX, vì sao cần TTL?
3. *(khó)* Vì sao release phải Lua/CAS chứ không `DEL` thẳng?
4. *(rất khó)* Fencing token là gì, cứu tình huống nào mà TTL không cứu?
5. *(rất khó)* Redlock là gì, Kleppmann phê phán điểm nào? Efficiency vs correctness chọn gì?
- 🎯 **Thách đố/trick:** *"Holder bị GC pause 8 giây, TTL lock là 5 giây — chuyện gì xảy ra, fix sao?"* → lock hết hạn, người khác chiếm, holder tỉnh dậy ghi bậy → cần fencing token (TTL vô dụng ở đây). · *"AI dựng distributed lock bằng Redis cho việc trừ tiền và bảo 'an toàn' — duyệt không?"* → **chặn**: đó là correctness, Redis single-node/Redlock không đủ; *AI viết chạy được nhưng sai kiến trúc* — chỉ huy dùng etcd/ZK + fencing, hoặc tốt hơn là atomic DB op.

## ⑨ Bài tập thực hành + tiêu chí tự chấm
- **BT1:** Viết `withRedisLock` (NX/PX + Lua release), bọc một "cron tổng hợp". **Đạt khi:** chạy 3 instance đồng thời → đúng 1 instance vào critical section, 2 cái nhận `LOCK_BUSY`; kill holder giữa chừng → TTL nhả, instance khác vào được.
- **BT2:** Viết 1 trang phân tích "lock này for efficiency hay correctness?" cho 3 tình huống (cron report, ghi ledger tiền, refresh cache). **Đạt khi:** phân loại đúng + chỉ ra cái nào cần fencing/consensus, cái nào Redis đủ.

## ⑩ Đọc thêm / nguồn chuẩn
- redis.io — *Distributed Locks with Redis* (đọc cả phần *Analysis of Redlock* + khuyến nghị fencing). · Martin Kleppmann — *How to do distributed locking* (bài critique kinh điển) ⇄ antirez phản hồi. · etcd docs — *Distributed locks / lease*; ZooKeeper recipes — *Locks*. · DDIA ch.8–9 (fencing token, unreliable clocks).

---
> 🔚 **Hết Bài 4.** → Chốt ghi nhớ. Tâm điểm recall: **fencing token vs TTL**, **efficiency vs correctness**.

---

# 📘 BÀI 5 — CAP & PACELC

> Phủ: `I-CAP-001 … I-CAP-011` · Định luật đánh đổi của hệ phân tán.

## ① Mục tiêu & vị trí trong mạch
Học xong bạn sẽ: phát biểu CAP **đúng** (và gỡ hiểu lầm "chọn 2 trong 3"), biết "Consistency" trong CAP là **linearizability** (khác ACID-C), phân biệt CP vs AP qua hành vi *khi partition*, hiểu **PACELC** thực tế hơn CAP ở chỗ nào, và biết khi nào **đừng** mang CAP vào câu chuyện. Bài này là khung tư duy cho Bài 6 (đào sâu "C") và Bài 7 (consensus thực thi quorum).

## ② Giảng cơ bản → nâng cao

**(a) Trực giác + gỡ hiểu lầm (`I-CAP-001`).** CAP: khi có **network partition (P)**, hệ phân tán **không thể** vừa Consistent vừa Available — phải chọn một. Hiểu lầm "chọn 2 trong 3" sai vì trong hệ phân tán thật, **P là bắt buộc** (mạng *sẽ* chia, sớm muộn). Nên thực chất chỉ chọn **C hay A khi partition xảy ra**. Lúc mạng lành, có thể có cả C lẫn A.

**(b) Cơ chế & bẫy thuật ngữ.**

- **"Consistency" trong CAP = linearizability (`I-CAP-002`):** read luôn thấy write mới nhất, *như thể chỉ có một bản sao duy nhất*. Đây **KHÁC** chữ "Consistency" trong ACID.
- **CAP-C vs ACID-C (`I-CAP-003`, bẫy kinh điển):** CAP-C = các **replica đồng ý** (replica agreement); ACID-C = transaction giữ **invariant/constraint** (FK, CHECK, business rule). Hai khái niệm **trực giao** — một hệ có thể ACID local nhưng eventually-consistent global. Đừng để interviewer thấy bạn lẫn hai chữ C này.
- **CP vs AP khi partition (`I-CAP-004`):** CP **từ chối/chặn** request (trả lỗi) để giữ nhất quán; AP **vẫn phục vụ** nhưng có thể trả data cũ / ghi conflict. *(verify phân loại — đổi theo cấu hình.)*
  - **CP (ưu C):** etcd, ZooKeeper, MongoDB *với majority write concern*, HBase, CockroachDB, Spanner.
  - **AP (ưu A):** Cassandra *ở consistency level thấp*, DynamoDB (eventually-consistent reads), CouchDB, DNS.
  - Lưu ý: nhiều DB **tunable** — MongoDB có thể nghiêng CP/AP theo write concern; Cassandra chọn **per query**.

**(c) PACELC — thực tế hơn (`I-CAP-005`, ⭐⭐⭐⭐⭐).**
**if Partition → A vs C; Else (không partition, >99% thời gian) → Latency vs C.** Điểm sáng: trade-off **latency ↔ consistency** tồn tại trên **MỖI request bình thường**, không chỉ lúc sự cố. Strong consistency cần round-trip tới majority/leader → **chậm hơn ngay cả khi mạng lành**. CAP bỏ qua chuyện này; PACELC vá đúng chỗ đó.
- **Phân loại PACELC (`I-CAP-006`):** MongoDB (majority) ≈ **PC+EC** (ưu consistency cả hai vế); Cassandra default ONE ≈ **PA+EL**, với QUORUM read+write → **PC+EC**. Người dùng chọn per-query = **tunable consistency**. *(verify.)*

**(d) Sắc thái senior/TL.**

- **"Chọn availability" chưa đủ (`I-CAP-007`):** interviewer senior muốn **con số**: staleness window chấp nhận được bao lâu, conflict rate dự kiến, **cách resolve conflict** (LWW / vector clock / CRDT). Trade-off cần định lượng, không khẩu hiệu.
- **Conflict resolution khi AP (`I-CAP-010`):** ghi cả hai phía lúc partition → khi heal phải merge: **last-write-wins** (theo timestamp — rủi ro *mất* ghi), **vector clocks/version vectors** (phát hiện concurrent), **CRDT** (merge tự động đúng đắn), hoặc app-level merge. Mỗi cách đánh đổi đơn giản ↔ rủi ro mất dữ liệu.
- **Partition không nhị phân (`I-CAP-008`):** node A coi timeout là "B chết", B vẫn chạy → hệ *bất đồng về việc có partition hay không*. CAP đơn giản hóa quá mức thực tế.
- **CAP không áp cho single-node (`I-CAP-009`):** CAP chỉ về **phân tán có replica**. Một Postgres đơn lẻ **không "chọn C/A" theo CAP**. Bẫy over-apply: phần lớn app vừa & nhỏ **không cần** lập luận CAP — mang CAP vào mọi câu là dấu hiệu học vẹt.
- **Consistency là per-operation, không per-system (`I-CAP-011`, ⭐⭐⭐⭐⭐):** một hệ *vừa* strong-consistent cho tài chính *vừa* eventually-consistent cho analytics là **đúng và nên thế**. Tune theo yêu cầu nghiệp vụ thật, không gán một nhãn cho cả hệ.

**📘 vs ➕:** docs gốc roadmap chạm CAP cơ bản; PACELC, CAP-C↔ACID-C, conflict resolution, "per-operation" là **⬆️/➕**.

## ③ ⚠️ Kiến thức cũ / cách làm bị thay thế

| Cái cũ / hiểu lầm | Thay bằng (đúng) | Vì sao |
|---|---|---|
| "CAP = chọn 2 trong 3" | Trong hệ phân tán P bắt buộc → **chọn C hay A khi partition** | Mạng sẽ chia, không bỏ P được |
| "Consistency" CAP = "Consistency" ACID | CAP-C = **linearizability/replica agreement**; ACID-C = invariant | Hai khái niệm trực giao |
| Chỉ dùng CAP để mô tả hệ | **PACELC** (thêm vế Else: Latency vs C) | Trade-off tồn tại cả khi mạng lành |
| Gán một nhãn CP/AP cho cả hệ | **Consistency per-operation** (tunable) | Mỗi path có nhu cầu khác nhau |
| "Chúng tôi chọn AP" (khẩu hiệu) | Định lượng staleness window + conflict resolution | Senior cần con số |
| LWW mặc định để resolve conflict | Cân nhắc vector clock / CRDT khi không được mất ghi | LWW âm thầm mất dữ liệu |
| Mang CAP vào hệ single-node | CAP chỉ cho phân tán có replica | Over-apply = học vẹt |

## ④ Áp dụng thực tế + So sánh bigtech
**Use case:** thiết kế hệ thương mại điện tử toàn cầu — **số dư ví & trạng thái thanh toán** chọn CP (thà báo lỗi còn hơn sai tiền); **feed/đếm like/gợi ý** chọn AP (lệch tạm chấp nhận, ưu tiên luôn phục vụ). Cùng một hệ, hai chế độ.
**Bigtech pattern (đã verify mức landscape):** Kubernetes coordination dùng **etcd (CP, Raft)**; DynamoDB là **AP** (Dynamo paper — eventually consistent + optional strongly-consistent reads); Cassandra cho chọn **per-query** (ONE=EL, QUORUM=EC); Google **Spanner**/CockroachDB cố "vừa C vừa A trên thực tế" bằng đồng bộ đồng hồ (TrueTime) + private network để partition cực hiếm — *không phá CAP, chỉ làm P hiếm đến mức bỏ qua được*. *(verify chi tiết khi học.)*

## ⑤ Code thực hành + cấu hình
> CAP là quyết định **kiến trúc/cấu hình**, không phải đoạn code. "Code" ở đây là *chọn consistency level đúng cho từng path*.

```javascript
// Ví dụ minh hoạ tunable consistency (Cassandra-style pseudo) — verify driver thật
// Path tài chính: cần đọc thấy ghi mới nhất → QUORUM (PC+EC)
await session.execute(query, { consistency: 'QUORUM' });   // chậm hơn, đúng hơn

// Path hiển thị feed: chấp nhận hơi cũ → ONE (PA+EL)
await session.execute(query, { consistency: 'ONE' });      // nhanh hơn, có thể stale

// 🧠 Bài học chỉ-huy-AI: AI dễ set mặc định một consistency cho mọi query.
//    Người hiểu phải GÁN đúng mức cho đúng path — sai mức ở path tiền = bug ẩn.
```

```text
// MongoDB: cấu hình readConcern/writeConcern theo path — verify docs chính thức
write tiền:   writeConcern: { w: 'majority' }        // nghiêng CP/EC
đọc dashboard: readConcern: 'local'                  // nhanh, có thể stale
```

**Cấu hình:** ghi rõ **policy consistency cho từng nhóm API** trong tài liệu kiến trúc; viết **test cho path nhạy cảm** (tiền, tồn kho) để bắt việc lỡ tay đặt sai mức; giám sát replication lag để biết staleness window thực tế.

## ⑥ Keywords cần nhớ
- 🧠 **Cần HIỂU:** CAP (P bắt buộc), linearizability-as-CAP-C, CAP-C vs ACID-C, PACELC, tunable/per-operation consistency, conflict resolution (LWW/vector clock/CRDT), partition không nhị phân.
- 📌 **Cần THUỘC:** "P → A vs C; E → L vs C"; CP={etcd, ZK, Mongo-majority, Spanner...}, AP={Cassandra-low, DynamoDB, CouchDB, DNS} *(verify)*; Mongo≈PC/EC, Cassandra ONE≈PA/EL.
- 🛠️ **Cần LÀM ĐƯỢC:** chọn consistency level đúng per-path; định lượng staleness window; chọn chiến lược resolve conflict.

## ⑦ Mạch tư duy cần nhớ (mental model)
**"CAP: khi mạng chia, chọn Đúng (C) hay Sống (A). PACELC thêm: khi mạng lành, vẫn chọn Nhanh (L) hay Đúng (C). Và lựa chọn này là PER-OPERATION, không phải nhãn dán cho cả hệ."**

## ⑧ Câu hỏi phỏng vấn & thách đố
1. *(TB)* Phát biểu CAP. "Chọn 2 trong 3" gây hiểu lầm gì?
2. *(khó)* "Consistency" trong CAP nghĩa chính xác là gì? Khác ACID-C ra sao?
3. *(khó)* CP và AP hành xử khác nhau thế nào khi partition? Ví dụ DB mỗi loại.
4. *(rất khó)* PACELC mở rộng CAP thế nào, vì sao thực tế hơn?
5. *(rất khó)* Một hệ vừa strong cho tài chính vừa eventual cho analytics — đúng/sai?
- 🎯 **Thách đố/trick:** *"Postgres đơn lẻ của tôi là CP hay AP?"* → câu hỏi sai — CAP chỉ áp cho hệ phân tán có replica; single-node không "chọn C/A". · *"Chúng tôi chọn availability"* — interviewer: *"staleness window bao lâu thì chấp nhận, conflict resolve bằng gì?"* → nếu không trả lời được con số là chưa đủ.

## ⑨ Bài tập thực hành + tiêu chí tự chấm
- **BT1:** Cho 6 tính năng (số dư ví, đếm view, trạng thái đơn, danh sách bạn bè, log audit, gợi ý sản phẩm) → gán CP/AP + mức consistency + lý do nghiệp vụ. **Đạt khi:** mỗi mục có lựa chọn + định lượng staleness chấp nhận được + cách resolve conflict (nếu AP).
- **BT2:** Phân loại PACELC cho 3 hệ bạn đang dùng (verify docs). **Đạt khi:** gọi đúng PA/PC và EL/EC kèm dẫn chứng cấu hình, không chép nhãn từ trí nhớ.

## ⑩ Đọc thêm / nguồn chuẩn
- Brewer — *CAP Twelve Years Later*. · Abadi — *Consistency Tradeoffs in Modern Distributed Database System Design (PACELC)*. · DDIA ch.5 & 9. · Jepsen.io — phân tích consistency thực nghiệm của DB cụ thể (verify từng hệ). · Docs chính thức của DB bạn dùng (read/write concern, consistency level).

---
> 🔚 **Hết Bài 5.** → Chốt ghi nhớ. Tâm điểm: **CAP-C ≠ ACID-C**, **PACELC vế Else**, **per-operation**.

---

# 📘 BÀI 6 — Consistency Models & Eventual Consistency

> Phủ: `I-CONS-001 … I-CONS-012` · Phổ nhất quán chi tiết — mở rộng chữ "C" của Bài 5.

## ① Mục tiêu & vị trí trong mạch
Học xong bạn sẽ: xếp được **phổ nhất quán** từ mạnh → yếu, định nghĩa linearizability và **phân biệt với serializability**, hiểu các **session guarantee** (read-your-writes, monotonic reads, causal), đọc được công thức **quorum R+W>N**, và gắn đúng mức consistency vào đúng nghiệp vụ. Bài này làm sâu "C" của CAP, dùng quorum sẽ được consensus thực thi ở Bài 7.

## ② Giảng cơ bản → nâng cao

**(a) Phổ nhất quán (`I-CONS-001`).** Từ mạnh → yếu: **linearizable → sequential → causal → eventual**. Càng mạnh càng cần **coordination** nhiều hơn → càng đắt/chậm. Chọn mức **yếu nhất mà nghiệp vụ vẫn đúng**.

**(b) Hai mức mạnh & phân biệt khó.**

- **Linearizability (`I-CONS-002`):** mọi op như xảy ra **tức thời tại một điểm** giữa lúc gọi và lúc trả về, tôn trọng **real-time order** — như thể chỉ có một bản sao. Đắt vì cần round-trip tới majority/leader.
- **Linearizability vs Serializability (`I-CONS-003`, ⭐⭐⭐⭐⭐, bẫy hay nhất):**
  - **Linearizability** = thuộc tính **single-object + real-time ordering** (về *recency*: đọc thấy ghi mới nhất).
  - **Serializability** = thuộc tính **multi-object** về *transaction*: kết quả tương đương *một* thứ tự tuần tự **nào đó** (không bắt buộc khớp real-time).
  - **Strict serializable** = cả hai (serializable + tôn trọng real-time). Đây là điểm nhiều người nhầm — hai chữ "serial..." khác trục hoàn toàn.

**(c) Eventual & session guarantees.**

- **Eventual consistency (`I-CONS-004`):** nếu **ngừng ghi**, các replica cuối cùng **hội tụ** về cùng giá trị. "Eventual" thường ms→giây, bị chặn bởi **replication lag**/network; **không** đảm bảo thời điểm cụ thể.
- **Read-your-own-writes (`I-CONS-005`):** session guarantee — đọc thấy ghi **của chính mình**. Vi phạm khi read route sang replica chưa cập nhật ("vừa update mà không thấy"). Fix: đọc từ primary sau write / sticky theo session / chờ version token. *(cross-ref E-REP.)*
- **Monotonic reads (`I-CONS-006`):** lần đọc sau không thấy state **cũ hơn** lần đọc trước. Thiếu → "thời gian chạy ngược": reload trang thấy data biến mất (trúng replica lệch khác nhau). Fix: sticky replica / version.
- **Causal consistency (`I-CONS-007`):** giữ **thứ tự nhân-quả** — reply không xuất hiện trước message gốc. Yếu hơn linearizable nhưng đủ cho chat/comment. Eventual có thể **đảo thứ tự** các sự kiện liên quan.

**(d) Quorum & quyết định nghiệp vụ.**

- **Quorum R+W>N (`I-CONS-008`):** nếu tập node đọc (R) + tập node ghi (W) **vượt tổng N** thì hai tập **giao nhau** → đọc chắc chắn chạm ít nhất một node có ghi mới nhất. Tunable: W=N (đọc nhanh), R=N (ghi nhanh), QUORUM (R=W=⌈(N+1)/2⌉) cân bằng.
- **Chọn mức theo nghiệp vụ (`I-CONS-009`):** (a) số dư ví → **strong/linearizable** (tiền không được sai); (b) đếm like → **eventual** (lệch tạm OK); (c) trạng thái đơn hàng → thường **strong** cho trạng thái quyết định, eventual cho hiển thị phụ. Gắn lựa chọn vào yêu cầu, không cảm tính.
- **Tunable consistency (`I-CONS-010`):** lợi = trả đúng chi phí cho đúng nhu cầu; rủi ro = dev chọn **sai mức** cho path quan trọng → bug khó tái hiện. Cần **policy rõ + test** cho path nhạy cảm.
- **"Cứ strong cho chắc" — phản biện (`I-CONS-011`):** strong = latency cao + giảm availability khi partition + throughput thấp; nhiều path **không cần**. Đừng mặc định strong toàn cục.
- **Replication lag gây lớp lỗi nào (`I-CONS-012`):** stale read, mất read-your-writes, non-monotonic reads. Giảm bằng read-from-primary cho path nhạy cảm, sticky session, version/token chờ replica, hoặc chấp nhận + UX phù hợp. *(cross-ref E-REP.)*

**📘 vs ➕:** docs gốc chạm "eventual consistency" sơ; phổ đầy đủ, session guarantees, linearizable-vs-serializable, quorum là **➕** chiều sâu.

## ③ ⚠️ Kiến thức cũ / cách làm bị thay thế

| Cái cũ / hiểu lầm | Thay bằng (đúng) | Vì sao |
|---|---|---|
| "Eventual = sẽ đồng bộ trong X giây" (đảm bảo thời điểm) | Eventual = **hội tụ nếu ngừng ghi**, không có deadline | Bị chặn bởi lag/network, không hứa mốc |
| Lẫn linearizability với serializability | single-object/real-time vs multi-object/transaction-order | Hai trục khác nhau; strict-serializable = cả hai |
| "User không thấy update của mình → bug ghi" | Thường là **route sang replica lag** (mất read-your-writes) | Sửa ở tầng routing, không phải ghi |
| Mặc định **strong consistency** toàn hệ | Chọn mức **yếu nhất đủ đúng** per-path | Strong đắt/chậm/giảm availability |
| Đọc 1 node coi là "đủ mới" | Hiểu **R+W>N** để đảm bảo giao điểm | Đọc thiếu quorum → stale |

## ④ Áp dụng thực tế + So sánh bigtech
**Use case:** sau khi user đăng bình luận, họ reload mà **không thấy** → kinh điển mất *read-your-writes* do đọc replica chưa kịp đồng bộ. Fix: route đọc của *chính user đó* về primary trong N giây sau write, hoặc gửi kèm version token để client chờ replica bắt kịp.
**Bigtech pattern:** Dynamo-style (DynamoDB/Cassandra) cho chọn **R/W quorum per-operation**; nhiều hệ social cố tình dùng **eventual** cho feed/counter để đạt throughput khổng lồ, và **causal/strong** cho phần nhạy cảm (tin nhắn, thanh toán). Đồng hồ logic/version vector dùng để giữ thứ tự nhân quả. *(Pattern phổ biến; verify từng hệ.)*

## ⑤ Code thực hành + cấu hình

```typescript
// ✅ Read-your-own-writes: route đọc về primary ngay sau khi user vừa ghi
async function getProfile(userId: string, justWrote: boolean) {
  // Nếu user vừa ghi trong cửa sổ ngắn → đọc PRIMARY để thấy ngay
  const conn = justWrote ? primaryDb : replicaDb;   // sticky-to-primary tạm thời
  return conn.query('SELECT * FROM profiles WHERE user_id = $1', [userId]);
}
// 🧠 Chỉ-huy-AI: AI hay route MỌI read sang replica để "scale". Người hiểu biết
//    phải tách path "vừa-ghi-của-tôi" về primary, nếu không user kêu "mất data".
```

```text
// Quorum R + W > N (minh hoạ N=3):
W=2, R=2  → 2+2=4 > 3  ✅ đảm bảo đọc chạm node có ghi mới nhất (cân bằng, QUORUM)
W=3, R=1  → đọc nhanh, ghi chậm (mọi replica phải nhận write)   (đọc-tối-ưu)
W=1, R=3  → ghi nhanh, đọc chậm                                  (ghi-tối-ưu)
W=1, R=1  → 1+1=2 < 3  ❌ KHÔNG đảm bảo — có thể stale (eventual)
```

**Cấu hình:** tài liệu hoá **mức consistency + routing policy** cho từng nhóm endpoint; test path tiền/tồn kho ở mức strong; theo dõi **replication lag** (đặt alert) để biết staleness window thực tế và chọn cửa sổ sticky-to-primary hợp lý.

## ⑥ Keywords cần nhớ
- 🧠 **Cần HIỂU:** phổ linearizable→sequential→causal→eventual, linearizability vs serializability, read-your-writes, monotonic reads, causal consistency, quorum intersection, "yếu nhất mà vẫn đúng".
- 📌 **Cần THUỘC:** R+W>N; strict-serializable = linearizable + serializable; "eventual = hội tụ nếu ngừng ghi"; QUORUM = ⌈(N+1)/2⌉.
- 🛠️ **Cần LÀM ĐƯỢC:** chẩn đoán "mất read-your-writes" do replica lag; chọn R/W; gán mức consistency per-path.

## ⑦ Mạch tư duy cần nhớ (mental model)
**"Consistency là một phổ, không phải on/off. Mạnh hơn = coordination nhiều hơn = đắt hơn → chọn mức YẾU NHẤT mà nghiệp vụ vẫn đúng. Linearizability nói về recency của 1 object; serializability nói về thứ tự của nhiều transaction — đừng lẫn."**

## ⑧ Câu hỏi phỏng vấn & thách đố
1. *(TB)* Kể phổ consistency từ mạnh đến yếu, cái mạnh đắt hơn vì sao?
2. *(khó)* Linearizability là gì (1 câu)? Vì sao đắt?
3. *(rất khó)* Linearizability vs serializability khác gì? Strict serializable là gì?
4. *(khó)* Read-your-writes là gì? Vì sao "vừa update mà không thấy"?
5. *(khó)* R+W>N nghĩa là gì? Tune thế nào cho đọc nhanh / ghi nhanh?
- 🎯 **Thách đố/trick:** *"Reload trang thấy data lúc có lúc không, 'thời gian chạy ngược' — bug gì?"* → thiếu monotonic reads do trúng các replica lệch nhau; fix sticky replica/version. · *"Cứ chọn strong consistency cho chắc đúng không?"* → không; strong đắt/chậm/giảm availability, nhiều path không cần — đây là *AI hay mặc định strong*, người hiểu phải tiết chế.

## ⑨ Bài tập thực hành + tiêu chí tự chấm
- **BT1:** Tái hiện mất read-your-writes (ghi primary, đọc replica có lag) rồi fix bằng sticky-to-primary. **Đạt khi:** chứng minh được hiện tượng "vừa ghi không thấy" → fix → thấy ngay, giải thích bằng lời tại sao.
- **BT2:** Với N=5, liệt kê các cặp (R,W) thỏa R+W>N và gọi tên trade-off mỗi cặp. **Đạt khi:** đúng điều kiện giao quorum + giải thích cặp nào tối ưu đọc/ghi/cân bằng.

## ⑩ Đọc thêm / nguồn chuẩn
- DDIA ch.5 & 9 (consistency models, linearizability, ordering). · Jepsen — *Consistency Models* (sơ đồ phổ kinh điển). · Dynamo paper (quorum R/W). · Docs DB bạn dùng (read/write concern, consistency level, replica routing).

---
> 🔚 **Hết Bài 6.** → Chốt ghi nhớ. Tâm điểm: **linearizability ≠ serializability**, **R+W>N**, **session guarantees**.

---

# 📘 BÀI 7 — Consensus (Raft / Paxos / Quorum / Leader Election)

> Phủ: `I-CONSENSUS-001 … I-CONSENSUS-009` · Nền móng đứng sau quorum, fencing token, CP system. Tổng kết cả mục I.

## ① Mục tiêu & vị trí trong mạch
Học xong bạn sẽ: hiểu **bài toán consensus** và nó cần cho việc gì, phân biệt **Paxos vs Raft** ở mức thực dụng, nắm **quorum/majority** chống split-brain, mô tả **Raft elect leader** bằng trực giác, hiểu hành vi khi **network partition**, biết vì sao consensus là cách "đúng" để sinh **fencing token** (nối Bài 4), và vì sao consensus **đắt** → không đặt vào hot path. Đây là điểm hội tụ: quorum (Bài 5/6), fencing (Bài 4), CP system (Bài 5) đều quy về đây.

## ② Giảng cơ bản → nâng cao

**(a) Consensus là gì (`I-CONSENSUS-001`).** Nhiều node phải **đồng ý trên một giá trị / một thứ tự** dù có lỗi (crash fault — node chết, mạng trễ). Cần cho: **leader election**, **log/state replication**, **distributed lock** đúng đắn, **config/coordination store** (etcd). Đây là "viên gạch" của mọi hệ CP.

**(b) Paxos vs Raft & quorum.**

- **Paxos vs Raft (`I-CONSENSUS-002`):** Paxos đúng-được-chứng-minh nhưng **khó hiểu/khó implement**. Raft thiết kế cho **dễ hiểu** — tách rõ **leader election** + **log replication** + **safety**; nay là **chuẩn de-facto** (etcd, Consul, nhiều hệ). *Mẹo phỏng vấn:* **đừng diễn Paxos từng bước** — nói được "Raft tách bài toán cho dễ hiểu, là chuẩn thực tế" là đủ điểm.
- **Quorum / majority (`I-CONSENSUS-003`):** chỉ **majority (N/2+1)** mới được elect leader / commit write → tránh **split-brain** (2 leader). Dùng **số lẻ** (3/5/7) để tối ưu khả năng chịu lỗi: 3 node chịu 1 chết, 5 chịu 2, 7 chịu 3. (4 node cũng chỉ chịu 1 như 3 → thừa.)

**(c) Cơ chế Raft (trực giác).**

- **Elect leader (`I-CONSENSUS-004`):** follower không nghe **heartbeat** trong *election timeout* → thành **candidate**, tăng **term**, xin vote. Ai được **majority** vote → thành **leader**, bắt đầu phát heartbeat. **Term** đóng vai **logical clock**: term cao hơn = thông tin mới hơn → nhận ra leader cũ/thông tin cũ.
- **Split-brain (`I-CONSENSUS-005`):** hai node cùng tưởng mình leader → ghi mâu thuẫn. Consensus chặn bằng **majority quorum**: chỉ phe đa số mới commit/elect, phe thiểu số **stall**. Không thể có hai majority trong cùng cluster → không thể hai leader hợp lệ.
- **Network partition (`I-CONSENSUS-006`):** phe **có majority** elect leader mới + tiếp tục phục vụ; phe **thiểu số** không đạt quorum → không commit (block hoặc trả stale tùy cấu hình). Khi **heal**: leader cũ (phe thiểu số) thấy **term cao hơn** từ phe kia → **step down**, đồng bộ lại log. Đây chính là cách CP system "từ chối phục vụ phe thiểu số" trong Bài 5.

**(d) Nối với fencing & chi phí.**

- **Consensus sinh fencing token (`I-CONSENSUS-007`):** fencing token cần **đơn điệu tăng + bền qua node chết**. Đếm trên 1 node → không chịu lỗi; nhiều node tự đếm → lệch nhau. Chỉ **consensus** cho được chuỗi thứ tự nhất quán, bền vững → ZooKeeper (zxid) / etcd (revision) cấp sẵn thứ tự này. Đây là lý do Bài 4 nói "for correctness → ZK/etcd".
- **etcd ở hạ tầng thật (`I-CONSENSUS-008`):** store cấu hình/coordination cho **Kubernetes** (toàn bộ state cluster), service discovery, leader election, distributed lock — vì Raft cho **strong consistency** + cơ chế **watch**. *(verify landscape.)*
- **Consensus ĐẮT, đừng đặt vào hot path (`I-CONSENSUS-009`, ⭐⭐⭐⭐⭐):** mỗi quyết định cần **round-trip tới majority + fsync log** → latency cao, throughput giới hạn. Chỉ dùng cho **metadata / coordination / leader election**, **KHÔNG** cho mọi write data-plane. Nguyên tắc TL: **tách control-plane (consensus) vs data-plane (throughput cao)** — đặt consensus đúng chỗ, không nhét vào đường nóng.

**📘 vs ➕:** docs gốc roadmap mở đầu agentic/coordination; toàn bộ consensus chi tiết là **➕** chiều sâu TL.

## ③ ⚠️ Kiến thức cũ / cách làm bị thay thế

| Cái cũ / hiểu lầm | Thay bằng (đúng) | Vì sao |
|---|---|---|
| Tự implement Paxos | Dùng **Raft** (etcd/Consul) hoặc thư viện đã kiểm chứng | Paxos khó đúng; Raft là chuẩn de-facto |
| Số node **chẵn** cho cluster (4, 6) | Số **lẻ** (3, 5, 7) | Chẵn không tăng khả năng chịu lỗi, tốn tài nguyên |
| Tự đếm fencing token trên app/1 node | Lấy thứ tự từ **consensus store** (zxid/revision) | 1 node không bền; nhiều node tự đếm lệch |
| Đặt consensus vào mọi write (hot path) | Consensus cho **control-plane**; data-plane riêng | Round-trip majority + fsync → quá chậm |
| "Cluster vẫn chạy khi mất quá nửa node" | Mất majority → **không commit được** (đúng theo thiết kế) | An toàn > sống khi đã mất quorum |

## ④ Áp dụng thực tế + So sánh bigtech
**Use case:** một service cần "chỉ một instance là leader xử lý job nhạy cảm". Dùng **etcd lease + leader election**: instance thắng lease là leader, mất lease (chết/partition) thì instance khác lên — an toàn vì etcd (Raft) không cho hai leader.
**Bigtech pattern (verify landscape):** **Kubernetes** lưu toàn bộ state trong **etcd (Raft)**; **Consul** (HashiCorp) dùng Raft cho service mesh coordination; Kafka từng dùng **ZooKeeper** cho controller/broker metadata, nay chuyển sang **KRaft** (Raft nội bộ, bỏ ZooKeeper); Google Spanner dùng Paxos cho replication group. Khuynh hướng: **Raft thắng** vì dễ hiểu/dễ vận hành. *(verify — landscape đổi theo phiên bản.)*

## ⑤ Code thực hành + cấu hình
> Consensus hiếm khi tự viết — bạn **dùng** etcd/ZooKeeper. "Code" là dùng API leader election/lease cho đúng.

```typescript
// ✅ Leader election bằng etcd lease (ý tưởng — verify API client thật)
// "etcd3"/client chính thức: kiểm chứng tại etcd docs
async function runAsLeaderOnly(work: () => Promise<void>) {
  const lease = await etcd.lease(10);                 // lease 10s, tự renew
  const lock = etcd.lock('jobs/leader');
  try {
    await lock.acquire();                              // chỉ 1 instance thắng (Raft đảm bảo)
    // ✅ Đến đây mình là leader duy nhất → làm việc nhạy cảm
    await work();
  } finally {
    await lock.release();
    await lease.revoke();
  }
}
// 🧠 Khác hẳn Redis SET NX (Bài 4): ở đây thứ tự & độc-nhất do CONSENSUS đảm bảo,
//    an toàn cho correctness; đổi lại nặng/chậm hơn → đừng nhét vào hot path.
```

```text
// Vì sao số lẻ: khả năng chịu lỗi f = ⌊N/2⌋
N=3 → f=1   N=5 → f=2   N=7 → f=3
N=4 → f=1 (bằng N=3 nhưng tốn hơn) → CHỌN LẺ
// Quorum để commit/elect = N/2 + 1  (3→2, 5→3, 7→4)
```

**Cấu hình:** cluster consensus đặt **số lẻ**, tách khỏi data-plane; đặt election/heartbeat timeout hợp lý theo độ trễ mạng; KHÔNG để consensus store gánh write throughput cao của ứng dụng; backup/snapshot định kỳ (etcd có thể là single point of failure cho cả cluster K8s).

## ⑥ Keywords cần nhớ
- 🧠 **Cần HIỂU:** consensus (đồng ý dù có crash fault), Paxos vs Raft, quorum/majority, split-brain, term-as-logical-clock, leader election, step-down khi heal, control-plane vs data-plane, "consensus đắt".
- 📌 **Cần THUỘC:** majority = N/2+1; số lẻ 3/5/7 chịu 1/2/3; Raft = election + log replication + safety; etcd=Raft (K8s).
- 🛠️ **Cần LÀM ĐƯỢC:** dùng etcd/ZK cho leader election; giải thích hành vi partition + heal; chỉ ra chỗ KHÔNG nên đặt consensus.

## ⑦ Mạch tư duy cần nhớ (mental model)
**"Consensus = nhiều node đồng ý dù có lỗi, dựa MAJORITY (nên dùng số lẻ) để cấm split-brain. Raft = Paxos-dễ-hiểu, là chuẩn thực tế (etcd/K8s). Nó đúng nhưng ĐẮT → chỉ đặt ở control-plane (coordination/leader/fencing), tránh hot path data."**

## ⑧ Câu hỏi phỏng vấn & thách đố
1. *(TB)* Consensus là bài toán gì, cần cho việc nào?
2. *(TB)* Paxos vs Raft khác biệt thực dụng nào? (và đừng làm gì trong phỏng vấn)
3. *(khó)* Quorum là gì, vì sao thường số node lẻ?
4. *(khó)* Raft elect leader thế nào (trực giác)? Term đóng vai gì?
5. *(rất khó)* Vì sao consensus là cách "đúng" để sinh fencing token? Vì sao đắt, khi nào KHÔNG đặt vào hot path?
- 🎯 **Thách đố/trick:** *"Cluster 5 node mất 3 node, sao không ghi được nữa?"* → mất majority (cần 3) → đúng theo thiết kế, thà block còn hơn split-brain. · *"AI đề xuất đẩy mọi write qua etcd cho 'nhất quán mạnh' — duyệt không?"* → **chặn**: consensus có round-trip majority + fsync, đặt vào data-plane sẽ giết throughput; đây là *AI đúng khái niệm nhưng sai chỗ đặt* — chỉ huy tách control-plane vs data-plane.

## ⑨ Bài tập thực hành + tiêu chí tự chấm
- **BT1:** Dựng leader election bằng etcd (hoặc ZooKeeper) cho 3 instance. **Đạt khi:** đúng 1 leader tại mọi thời điểm; kill leader → instance khác lên trong vài giây; mô tả được điều gì xảy ra khi partition.
- **BT2:** Viết 1 trang "đặt consensus ở đâu trong kiến trúc của tôi". **Đạt khi:** chỉ rõ phần nào là control-plane (đặt consensus) vs data-plane (không), kèm lý do latency/throughput.

## ⑩ Đọc thêm / nguồn chuẩn
- *In Search of an Understandable Consensus Algorithm (Raft)* — Ongaro & Ousterhout. · raft.github.io (visualization tương tác). · etcd docs — *Learning / Why etcd*; ZooKeeper — *Internals (ZAB)*. · DDIA ch.9 (consensus, total order broadcast, membership). · Lamport — *Paxos Made Simple* (đọc để biết, không cần thuộc).

---
---

# ✅ TỔNG KẾT MỤC I & Bản đồ ôn tập 1 trang

## Mạch xuyên suốt (one-liner mỗi bài)
1. **Race:** khoảng trống đọc-ghi sinh race → atomic write, đừng vá bằng timing.
2. **Lock:** pessimistic (khóa-rồi-làm) vs optimistic (làm-rồi-kiểm-tra); chọn theo conflict rate.
3. **Idempotency:** mạng giao trùng → lặp-không-đổi; nhưng song song vẫn cần atomic claim.
4. **Distributed lock:** across-resource; TTL chống deadlock, **fencing token** chống zombie; efficiency→Redis, correctness→consensus.
5. **CAP/PACELC:** partition→C/A; bình thường→L/C; per-operation, không per-system.
6. **Consistency models:** phổ mạnh→yếu; linearizability≠serializability; chọn mức yếu nhất đủ đúng.
7. **Consensus:** majority (số lẻ) cấm split-brain; Raft=chuẩn; đắt→chỉ control-plane.

## Sợi chỉ đỏ nối 7 bài
> **Race** là vấn đề gốc. Chống nó trong 1 DB = **Lock** + **Idempotency**. Vượt khỏi 1 DB = **Distributed lock**, mà muốn *đúng* thì cần **fencing token** — thứ chỉ **Consensus** sinh được an toàn. Còn ở tầng *hệ phân tán có replica*, đánh đổi nhất quán được đóng khung bởi **CAP/PACELC** và chi tiết hóa bởi **Consistency models**, mà mức mạnh nhất (linearizable, CP) lại do **Consensus** thực thi qua **quorum**. → **Tất cả quy về quorum + atomicity.**

## Bảng "dễ nhầm" cần khắc
| Cặp dễ lẫn | Phân biệt 1 dòng |
|---|---|
| CAP-C vs ACID-C | replica agreement (linearizability) vs invariant của transaction |
| Linearizability vs Serializability | single-object/real-time vs multi-object/transaction-order; cả hai = strict serializable |
| Optimistic vs Pessimistic | kiểm-tra-lúc-commit (retry) vs khóa-trước (chờ) |
| TTL vs Fencing token | nhả lock khi chết vs chặn holder-zombie ghi đè |
| at-least-once vs exactly-once | thật & cần idempotent vs ảo tưởng; effectively-once = at-least-once + dedup |
| efficiency-lock vs correctness-lock | Redis SET NX đủ vs cần consensus + fencing |
| Paxos vs Raft | đúng-nhưng-khó vs dễ-hiểu-chuẩn-thực-tế |

## Lịch spaced repetition gợi ý
**Hôm nay** (vừa học): Bài 1–2. · **Mai:** Bài 1–4 + recall các "bẫy ⭐⭐⭐⭐⭐". · **3 ngày:** toàn bộ + bảng dễ-nhầm. · **1 tuần:** phỏng vấn trộn 71 câu (Bước D), chèn câu cũ.

## ⚠️ Nhắc verify khi học (đừng tin trí nhớ model)
Mọi dòng có **(verify)**: phân loại CAP/PACELC của DB cụ thể, cú pháp & guarantee Redis/Redlock/ZK/etcd, API `@VersionColumn` của ORM (và **bẫy TypeORM update một phần không tự version-check**), landscape consensus (Kafka KRaft, etcd version) — **đối chiếu docs chính thức tại thời điểm học**.

## 🧭 Câu thần chú
> *"Tôi không nhớ để gõ, tôi hiểu để chỉ huy — và tra cứu phần còn lại."*
> Concurrency là nơi **AI viết đúng cú pháp nhưng sai hành vi** nhiều nhất. Mỗi bài đều có ít nhất một điểm "AI dễ sai" — đó là nơi giá trị của bạn nằm: **kiểm tra và chỉ huy**.

---
> **Bước tiếp theo (Bước D — Phỏng vấn LIVE):** sẵn sàng thì gõ *"test tôi mục I"* — tôi sẽ ra từng đợt 10–15 câu trộn dễ→khó từ 71 câu, **hỏi xong DỪNG chờ bạn trả lời**, rồi chấm bám đúng dòng "dò cái gì". Hoặc gõ *"giảng Bài X"* để đi sâu một bài, hoặc *"lưu tiến độ"* để xuất snapshot.
