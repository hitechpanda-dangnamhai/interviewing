# 🎯 BÀI 4 — Distributed Locks & Fencing Token

**Phủ:** I-DLOCK-001 → I-DLOCK-012 · **Khi race vượt khỏi một DB transaction.**

> 💡 **Cách đọc bài này.** Đây là bài *khó nhất và "senior nhất"* của mục I — chỗ **phân biệt senior với Tech Lead thật**. Hai ý sẽ theo bạn suốt đời làm hệ phân tán: ***(1) TTL chống deadlock nhưng KHÔNG chống "holder zombie" — chỉ fencing token làm được; (2) lock cho "efficiency" rất khác lock cho "correctness".*** Nếu chỉ nhớ được hai câu đó thôi, bài này đã đáng giá.

---

## ① Mục tiêu & vị trí trong mạch

Học xong bài này bạn sẽ làm được 5 việc:

1. **Biết khi nào cần distributed lock** (và *khi nào nên tránh*).
2. **Viết lock Redis tối thiểu đúng** — `SET NX PX` + **release-if-token-match**.
3. **Hiểu fencing token cứu cái gì mà TTL không cứu được.**
4. **Nắm tranh luận Redlock** (Kleppmann ⇄ antirez).
5. **Phân biệt lock "for efficiency" vs "for correctness".**

> 🎯 **Vị trí trong mạch học.**
> - **Nâng cấp từ Bài 2:** Bài 2 khóa *trong một DB*; ở đây race **vượt ra ngoài** một transaction → cần điểm khóa chung *bên ngoài*.
> - **Dùng quorum** (sẽ đào ở **Bài 5/7**) và là **động lực dẫn tới consensus** (**Bài 7**).

**Ẩn dụ định vị:** Bài 1–3 lo race *trong nhà* (một DB). Bài 4 là khi tài nguyên *nằm rải ra ngoài đường* — gọi API bên ngoài, ghi vào nhiều store, cron chạy trên nhiều máy. Lúc đó cái khóa cửa nhà (`FOR UPDATE`) không với tới được; bạn cần một *"trạm điều phối" chung* mà mọi máy đều tin.

---

## ② Giảng cơ bản → nâng cao

### (a) Trực giác — **I-DLOCK-001**

> **Ví dụ trực giác:** Ba chi nhánh ngân hàng cùng muốn *in một tờ báo cáo tổng* lên một máy in chung ở trụ sở. Không chi nhánh nào "sở hữu" máy in. Cần một *quy ước chung*: ai cầm được *thẻ điều phối* thì mới được in, in xong trả thẻ.

**Distributed lock** = **mutual exclusion** (loại trừ tương hỗ — *chỉ một bên được vào vùng tới hạn tại một thời điểm*) **across process / instance / service**, dành cho **tài nguyên KHÔNG nằm gọn trong một DB transaction**, ví dụ:
- **gọi API bên ngoài** (không rollback được),
- **cron "chỉ chạy một lần"** dù deploy nhiều instance,
- **ghi vào nhiều store** khác nhau.

> 🎯 **Chốt:** Khi `FOR UPDATE` (Bài 2) **không phủ được phạm vi** (vì tài nguyên không nằm trong một DB) → cần một **điểm khóa chung bên ngoài** (**Redis / ZooKeeper / etcd**).

---

### (b) Cơ chế chi tiết

#### 🔍 Lock Redis tối thiểu — **I-DLOCK-002**

```
SET key <token> NX PX <ttl>
```

| Thành phần | Nghĩa | Vì sao cần |
|---|---|---|
| **`NX`** (set-if-not-exists) | Chỉ set nếu key **chưa tồn tại** | **Claim atomic** — chỉ *một* client thắng, không ai chen |
| **`PX` / `TTL`** | Tự **hết hạn** sau `ttl` ms | Holder chết thì lock **tự nhả** → tránh **deadlock vĩnh viễn** |
| **`<token>`** | Chuỗi sở hữu (vd UUID) | Để lúc release biết *"lock này có còn là của mình không"* |

> 🔎 **verify cú pháp tại redis.io.**

> 🚨 **Cấm tách `SETNX` rồi `EXPIRE` riêng (2 lệnh).** **SET NX của Redis hiện đại đã gộp set+expire atomic.** Nếu tách ra mà **crash giữa hai lệnh** → key đã set nhưng *chưa kịp đặt TTL* → **lock không bao giờ hết hạn** → kẹt vĩnh viễn.

---

#### 🔍 Release phải "delete-if-token-matches" — **I-DLOCK-003**

> **Ví dụ trực giác:** Bạn thuê phòng khách sạn tới 12h trưa. Quá giờ, lễ tân *đã cho người khác thuê*. Nếu bạn quay lại "trả phòng" bằng cách *quăng chìa vào ổ và bảo dọn phòng* — bạn vừa **đuổi nhầm người khách mới**. Phải *kiểm tra "phòng này còn đứng tên mình không"* rồi mới trả.

> 🚨 **KHÔNG `DEL` thẳng.** Nếu lock của bạn **đã hết TTL** và client khác **đã chiếm**, `DEL` thẳng sẽ **xóa nhầm lock người khác**.

**Timeline — vì sao DEL thẳng nguy hiểm:**

```
T1  A: SET lock token=AAA NX PX 5000  → A giữ lock 🔒 (token AAA)
       ↓  (A xử lý chậm, quá 5s → TTL hết, lock TỰ NHẢ)
T2  B: SET lock token=BBB NX PX 5000  → B giữ lock 🔒 (token BBB) ✅
       ↓
T3  A: xong việc → DEL lock            → ❌ XÓA NHẦM lock của B!
T4  C: SET lock ... NX → thành công    → giờ B và C cùng tưởng mình giữ lock ❌❌
```

**Cách đúng — release atomic bằng Lua script** (vì *get-rồi-del* là hai thao tác → lại **TOCTOU**):

```lua
if redis.call('get', KEYS[1]) == ARGV[1]   -- token còn là của mình?
   then return redis.call('del', KEYS[1])   -- thì mới xóa
   else return 0 end
```

---

### (c) TTL, failover & kẻ thù timing

#### 🔍 TTL ngắn vs dài — **I-DLOCK-004**

| TTL | Rủi ro | Hệ quả tệ nhất |
|---|---|---|
| **Ngắn** | Lock hết hạn **giữa critical section** | **2 holder cùng lúc** → vi phạm mutual exclusion |
| **Dài** | Holder chết thì tài nguyên **kẹt lâu** | Cả hệ chờ vô ích tới khi TTL hết |

> 💡 **Cách thoát thế lưỡng nan:** dùng **watchdog / lease-renewal** (*gia hạn lock định kỳ* khi còn sống), hoặc **ước lượng TTL đúng** (đo critical section thực tế) **+ fencing**.

---

#### 🚨 GC pause / process pause là kẻ thù — **I-DLOCK-008**

> **Ví dụ trực giác:** Bạn đang giữ thẻ điều phối thì *ngất xỉu 8 giây* (không hề hay biết). Trong lúc đó thẻ hết hạn, người khác nhận thẻ và bắt đầu in. Bạn *tỉnh dậy, tưởng mình vẫn cầm thẻ*, lao vào in tiếp → **hai người cùng in lên một tờ giấy**.

**Cơ chế:** holder bị **pause** (GC, **stop-the-world**, scheduler treo) **lâu hơn TTL** → lock hết hạn → client khác chiếm → **holder tỉnh dậy vẫn tưởng còn giữ lock** và ghi tiếp → **vi phạm mutual exclusion**.

**Timeline — holder zombie:**

```
TTL = 5s

T0   A: acquire lock 🔒, vào critical section
T1   A: ⏸️ GC pause 8 giây (A "đứng hình", không biết gì)
T5   lock TTL hết → tự nhả
T6   B: acquire lock 🔒 ✅, B ghi dữ liệu
T8   A: 😵 tỉnh dậy, TƯỞNG còn giữ lock → ghi ĐÈ lên dữ liệu của B ❌
```

> 🎯 **Chốt (rất quan trọng):** **TTL KHÔNG chặn được điều này** — vì process tỉnh dậy *không biết mình đã mất lock*. Đây chính là lý do **fencing token** ra đời (xem mục (d)).

---

#### 🚨 Redis master–replica failover phá lock — **I-DLOCK-011**

> **Ví dụ trực giác:** Sổ ghi "ai đang giữ thẻ" được chép từ *quyển chính* sang *quyển dự phòng*, nhưng việc chép có độ trễ. Quyển chính cháy *trước khi chép kịp*; mọi người giở quyển dự phòng ra thấy "trống" → phát thẻ cho người mới, trong khi người cũ vẫn cầm thẻ.

**Cơ chế:** **replication async** (sao chép bất đồng bộ) — lock `SET` trên **master** *chưa kịp sang* **replica**, master chết, replica lên *không có lock* → **2 client cùng giữ**.

> 🔎 Đây chính là **động cơ đẻ ra Redlock** (và Redlock *cũng không kín* — xem mục (d)). **(verify.)**

---

### (d) Fencing token & Redlock — phần lõi senior/TL

#### 🚨 Fencing token — **I-DLOCK-006** (⭐⭐⭐⭐⭐)

> **Ví dụ trực giác:** Mỗi lần phát thẻ điều phối, đánh **số thứ tự tăng dần**: thẻ #33, #34, #35... Máy in **ghi nhớ số lớn nhất nó từng phục vụ**. Ai cầm thẻ tới mà số *nhỏ hơn* số máy in đã thấy → **từ chối in**. Anh "zombie" tỉnh dậy cầm thẻ #33 cũ, nhưng máy in đã phục vụ #34 → #33 bị chặn.

**Fencing token** = một **số đơn điệu tăng** (*monotonic*) cấp mỗi lần **acquire**. **Resource** (DB/storage) **từ chối write mang token NHỎ HƠN token lớn nhất nó từng thấy** → chặn **holder zombie** (bị pause) ghi đè *sau khi lock đã chuyển sang người khác*.

**Timeline — fencing token cứu holder-zombie:**

```
T0   A: acquire → token = 33, vào critical section
T1   A: ⏸️ GC pause...
T5   lock hết hạn
T6   B: acquire → token = 34, B ghi (kèm token 34) → storage ghi nhận max_token = 34 ✅
T8   A: 😵 tỉnh dậy, ghi (kèm token 33)
        → storage thấy 33 < 34 → TỪ CHỐI ✅ (A bị "fence" ra ngoài)
```

> 🎯 **Chốt:** Đây là thứ **TTL không làm được** — *TTL chỉ giải phóng lock, không ngăn process cũ tỉnh dậy ghi bậy*. **Fencing cần resource biết kiểm tra token** (không phải lock nào cũng tự có).

---

#### 🚨 Redlock & phê phán Kleppmann — **I-DLOCK-005** (⭐⭐⭐⭐⭐)

**Redlock** = acquire lock trên **N Redis độc lập** (*không replicate giữa nhau*), cần **majority** (**N/2 + 1**) đồng ý *trong thời gian < TTL*.

**Kleppmann phê phán hai điểm:**

| Điểm phê phán | Vì sao nguy hiểm |
|---|---|
| (a) **Không tự sinh fencing token** | Không chặn được holder-zombie (mục I-DLOCK-008) |
| (b) **Phụ thuộc giả định timing** (clock skew, GC pause, network delay) | Khi giả định vỡ → **vẫn có thể 2 client cùng giữ lock** |

> 🔎 **redis.io docs hiện cũng nhắc** đọc kỹ phần phản biện và *khuyên implement fencing token*; lưu ý **Redis dùng wall-clock** (không **monotonic**) cho TTL nên **dịch giờ (clock skew)** có thể gây lỗi.

> ⚠️ **(verify; nguồn martin.kleppmann.com ⇄ antirez ⇄ redis.io docs.)**

---

#### 🔍 Efficiency vs Correctness — **I-DLOCK-007**

> **Kleppmann/antirez đồng thuận thực dụng** ở chỗ này — hiếm khi hai bên tranh luận lại đồng ý, nên hãy nhớ kỹ:

| Mục đích lock | Hậu quả nếu lock sai | Giải pháp đủ |
|---|---|---|
| **for efficiency** (tránh làm trùng; sai chút *không chết* — vd né chạy cron 2 lần) | Tốn thêm chút CPU | **single-node Redis `SET NX`** là đủ |
| **for correctness** (sai = **mất tiền / hỏng data**) | Thảm họa | **consensus system** (**ZooKeeper / etcd**) + **bắt buộc fencing token** |

---

#### 🔍 ZK/etcd khác Redis — **I-DLOCK-009**

| Tiêu chí | **Redis lock** | **ZooKeeper / etcd** |
|---|---|---|
| Nền tảng | Key-value + TTL | **consensus** (**ZAB / Raft**) |
| Thứ tự / fencing | Không tự sinh | **Tự sinh thứ tự đơn điệu** (gần như **fencing sẵn**) |
| Nhả khi holder chết | Dựa TTL | **ephemeral node / lease** — tự nhả khi **session chết** |
| An toàn cho correctness | Yếu | **An toàn hơn** |
| Đánh đổi | Nhẹ, nhanh | **Nặng / chậm hơn** |

> ⚠️ **(verify.)**

---

### (e) Khi nào TRÁNH distributed lock — **I-DLOCK-012** (⭐⭐⭐⭐⭐)

> 🚨 **Nguyên tắc vàng:** **Distributed lock là một "code smell" dễ sai** (TTL, failover, fencing, clock — bốn cái bẫy cùng lúc). **Chỉ dùng khi thật sự cần exclusion across-resource mà không cách nào khác.**

Nếu **thay được** bằng những cách dưới đây thì **ưu tiên chúng**:

| Thay vì distributed lock | Dùng | Nhắc lại từ bài |
|---|---|---|
| Trừ kho / số dư đơn giản | **atomic DB op** | Bài 1 |
| Chống trùng | **unique constraint / idempotency** | Bài 2, 3 |
| Mỗi key một người xử lý | **partition theo key** (mỗi key một consumer) | Bài 1 (single-writer) |

---

#### 🔍 Cron chỉ chạy 1 lần dù nhiều instance — **I-DLOCK-010**

> 🚨 **Đừng giả định "chỉ deploy 1 instance".** Giả định này *sẽ vỡ* ngay khi bạn scale ra nhiều instance — và lúc đó cron chạy trùng.

Cách đúng:
- **leader election** / **distributed lock TTL ngắn + renew**, hoặc
- **DB advisory lock** / **unique row theo schedule-slot** (mỗi khung giờ chỉ một row được insert thành công).

> 📘 **vs** ➕ — *docs gốc hầu như không đụng*; **toàn bộ bài là phần ➕ chiều sâu TL.** Đây là bài *"phân biệt senior với TL thật"* — **fencing token & Redlock critique là tâm điểm.**

---

## ③ ⚠️ Kiến thức cũ / cách làm bị thay thế

| Cái cũ | Thay bằng (nên dùng) | Vì sao |
|---|---|---|
| `SETNX key` rồi `EXPIRE key` (2 lệnh) | `SET key token NX PX ttl` (1 lệnh atomic) | Crash giữa 2 lệnh → **lock vĩnh viễn** |
| `DEL key` để nhả lock | **Lua: `if get==token then del`** | `DEL` thẳng **xóa nhầm lock người khác** |
| Tin **TTL là đủ cho correctness** | **TTL + fencing token** (+ consensus nếu cần đúng tuyệt đối) | **GC / process pause** vượt TTL → **2 holder** |
| **Redlock = "an toàn cho correctness"** | for-correctness → **ZK/etcd + fencing**; Redlock chỉ **for-efficiency** | Phụ thuộc **timing**, không **fencing** |
| Lock trên **Redis single master** cho việc sống-còn | **Consensus store** có **lease / thứ tự** | **Async failover** mất lock |
| Mặc định **distributed lock** cho mọi exclusion | Ưu tiên **atomic op / unique / idempotency / partition-by-key** | Lock **dễ sai, đắt** |
| *"Cron chạy 1 lần vì chỉ deploy 1 instance"* | **Leader election / advisory lock** | Giả định 1-instance **sẽ vỡ khi scale** |

---

## ④ Áp dụng thực tế + So sánh bigtech

**Use case efficiency:** tránh **hai instance cùng chạy job tổng hợp báo cáo** (chạy trùng chỉ *tốn CPU*) → **single-node Redis `SET NX PX`** là đủ.

**Use case correctness:** chỉ **một node được phép ghi vào file/ledger chia sẻ** → **ZooKeeper / etcd lease + fencing token** mà **storage kiểm tra**.

**Bigtech pattern:**
- **Kubernetes** dùng **etcd (Raft)** cho **coordination / leader election**.
- Nhiều hệ dùng **ZooKeeper** cho **controller election** (kiểu **Kafka cũ**).
- **redis.io** liệt kê nhiều thư viện **Redlock** (**Redisson/Java**, **node-redlock**, **Redsync/Go**...) nhưng **kèm cảnh báo fencing**.
- **Khuynh hướng TL:** *đẩy **correctness** về **consensus store**, chỉ dùng **Redis lock** cho **efficiency**.*

> ⚠️ **(verify landscape.)**

---

## ⑤ Code thực hành + cấu hình

**Lock Redis tối thiểu ĐÚNG:**

```javascript
// ✅ Lock Redis tối thiểu ĐÚNG (ioredis) — verify cú pháp tại redis.io
// "ioredis": "^5.x" (pin thật)
import Redis from 'ioredis';
import { randomUUID } from 'crypto';
const redis = new Redis(process.env.REDIS_URL!); // KHÔNG hardcode credential

async function withRedisLock<T>(key: string, ttlMs: number, fn: () => Promise<T>) {
  const token = randomUUID();                 // token sở hữu, để release ĐÚNG CHỦ
  // NX = claim atomic (chỉ 1 client thắng); PX = TTL tránh deadlock vĩnh viễn nếu holder chết
  const ok = await redis.set(`lock:${key}`, token, 'PX', ttlMs, 'NX');
  if (ok !== 'OK') throw new Error('LOCK_BUSY'); // không giành được lock → ai đó đang giữ
  try {
    return await fn();                          // critical section: chỉ 1 người vào
  } finally {
    // ✅ release ATOMIC: chỉ xóa NẾU token còn là của mình (tránh xóa nhầm lock người khác)
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

**Fencing token — ý tưởng (TTL không thay thế được):**

```
// 🧠 Fencing token (ý tưởng — TTL không thay thế được):
acquire() → trả token đơn điệu tăng: 33, rồi 34, 35...
Resource (DB/storage) lưu max_token đã thấy.
Ghi kèm token < max_token  →  TỪ CHỐI (chặn holder zombie tỉnh dậy sau pause).
→ Redis SET NX KHÔNG tự sinh chuỗi này; ZooKeeper/etcd có thứ tự sẵn.
```

> 🔧 **Cấu hình:**
> - **TTL phải > thời gian critical section thực tế** (*đo, đừng đoán*).
> - Cân nhắc **lease-renewal / watchdog** cho job dài.
> - **Đừng gọi API ngoài chậm** khi đang giữ lock **TTL ngắn**.
> - Với **correctness** → chuyển sang **etcd / ZooKeeper**.
> - Secrets qua **env**.

---

## ⑥ Keywords cần nhớ (glossary)

**🧠 Cần HIỂU (nắm bản chất):**
- **distributed lock** — mutual exclusion across-resource, ngoài một DB transaction.
- **fencing token** — số đơn điệu tăng, resource từ chối token cũ → chặn holder-zombie.
- **Redlock & critique** — N Redis độc lập + majority; Kleppmann chê thiếu fencing + phụ thuộc timing.
- **efficiency vs correctness lock** — sai-không-chết vs sai-mất-tiền.
- **GC-pause hazard** — holder bị pause vượt TTL → 2 holder.
- **async-failover hazard** — master chết trước khi replicate → mất lock.
- **"tránh lock nếu có cách atomic"** — distributed lock là phương án cuối.

**📌 Cần THUỘC (nhớ nguyên văn):**
- `SET key token NX PX ttl`.
- **Lua release-if-token-match**.
- **Redlock = N Redis độc lập + majority < TTL**.

**🛠️ Cần LÀM ĐƯỢC (kỹ năng tay):**
- Viết **Redis lock đúng** (NX/PX + Lua release).
- Thiết kế **cron single-run** đa instance.
- **Chỉ ra chỗ cần fencing**.

---

## ⑦ Mạch tư duy cần nhớ (mental model)

> 🎯 **Khẩu quyết — gói cả bài trong một hơi:**
> *"**Distributed lock = mutual exclusion across-resource.** **TTL chống deadlock nhưng KHÔNG chống holder-zombie** — chỉ **fencing token** (số đơn điệu tăng) làm được. Lock cho **efficiency** dùng **Redis**; cho **correctness** dùng **consensus + fencing**. **Tốt nhất là TRÁNH lock.**"*

---

## ⑧ Câu hỏi phỏng vấn & thách đố

**(TB)** **Distributed lock** là gì, khi nào cần so với **DB row lock**?
→ *Gợi ý:* khi tài nguyên vượt một DB transaction (API ngoài, nhiều store, cron đa instance).

**(TB)** `SET key token NX PX ttl` — vì sao cần **NX**, vì sao cần **TTL**?
→ *Gợi ý:* NX = claim atomic (một người thắng); TTL = tự nhả nếu holder chết → tránh deadlock.

**(khó)** Vì sao **release phải Lua/CAS** chứ không **`DEL` thẳng**?
→ *Gợi ý:* lock có thể đã hết TTL và người khác chiếm → DEL thẳng xóa nhầm; phải so token.

**(rất khó)** **Fencing token** là gì, cứu tình huống nào mà **TTL không cứu**?
→ *Gợi ý:* holder zombie (GC pause) tỉnh dậy ghi bậy; token cũ < max_token bị resource từ chối.

**(rất khó)** **Redlock** là gì, **Kleppmann phê phán** điểm nào? **Efficiency vs correctness** chọn gì?
→ *Gợi ý:* N Redis + majority; chê thiếu fencing + phụ thuộc timing; correctness → consensus + fencing.

> 🧩 **Thách đố / trick — AI hay sai ở đây:**
> *"Holder bị **GC pause 8 giây**, TTL lock là **5 giây** — chuyện gì xảy ra, fix sao?"*
> → lock hết hạn, người khác chiếm, holder **tỉnh dậy ghi bậy** → cần **fencing token** (**TTL vô dụng** ở đây).
>
> *"AI dựng **distributed lock bằng Redis cho việc trừ tiền** và bảo 'an toàn' — duyệt không?"*
> → ❌ **chặn**: đó là **correctness**, **Redis single-node / Redlock không đủ**; *AI viết chạy được nhưng **sai kiến trúc*** — chỉ huy dùng **etcd / ZK + fencing**, hoặc tốt hơn là **atomic DB op**.

---

## ⑨ Bài tập thực hành + tiêu chí tự chấm

**BT1 — `withRedisLock` (NX/PX + Lua release), bọc một "cron tổng hợp".**
> ✅ **Đạt khi:** chạy **3 instance đồng thời** → **đúng 1 instance** vào critical section, **2 cái nhận `LOCK_BUSY`**; **kill holder giữa chừng** → **TTL nhả**, instance khác vào được.

**BT2 — Phân tích "efficiency hay correctness?" cho 3 tình huống.**
Viết 1 trang cho: **cron report**, **ghi ledger tiền**, **refresh cache**.
> ✅ **Đạt khi:** **phân loại đúng** + chỉ ra cái nào cần **fencing / consensus**, cái nào **Redis đủ**.

---

## ⑩ Đọc thêm / nguồn chuẩn

- **redis.io** — *Distributed Locks with Redis* (đọc cả phần **Analysis of Redlock** + khuyến nghị **fencing**).
- **Martin Kleppmann** — *How to do distributed locking* (bài **critique** kinh điển) ⇄ **antirez** phản hồi.
- **etcd docs** — *Distributed locks / lease*; **ZooKeeper recipes** — *Locks*.
- **DDIA ch.8–9** (**fencing token**, **unreliable clocks**).

---

> 🔚 **Hết Bài 4.** → **Bước 3 — Chốt ghi nhớ.**
>
> 🎯 **Tâm điểm recall:** **fencing token vs TTL**, **efficiency vs correctness**. Gợi ý 3 câu tự hỏi: *(1) Vì sao release phải Lua chứ không DEL thẳng? (2) GC pause 8s + TTL 5s → chuyện gì, fencing cứu thế nào? (3) Việc trừ tiền nên dùng Redis lock hay gì khác, vì sao?*
