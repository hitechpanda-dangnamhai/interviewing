# 🎯 BÀI 10 — Background Jobs: BullMQ / Redis Streams
**Phủ:** J-QUEUE-001 → J-QUEUE-007

---

## ① Mục tiêu & vị trí trong mạch

Redis còn làm **job queue** (hàng đợi xử lý nền). Bài cuối của mạch J này trả lời:

- **Vì sao cần background queue?**
- **BullMQ** — chuẩn de-facto của Node (dựa trên **Redis Streams**).
- **Streams vs list-based queue**.
- Làm sao **đảm bảo không mất job**.
- **DLQ / retry** an toàn.
- Khi nào BullMQ **không đủ → Kafka / RabbitMQ**.
- Và **vì sao phải tách Redis queue khỏi Redis cache** (nối Bài 6).

Bài này **mở đường sang mục K (Messaging)**.

> 💡 **Ý chính:** Một queue chạy đúng phải bám hai sự thật: **job là thứ "KHÔNG ĐƯỢC MẤT"** (nên không được nằm chung instance với cache có eviction), và **mọi queue thực tế đều là at-least-once** (job *có thể chạy lại*) → **consumer BẮT BUỘC phải idempotent**.

---

## ② Giảng cơ bản → nâng cao

### (a) Vì sao cần job queue *(J-QUEUE-001)* — 3 lớp vấn đề

**1. Tách việc chậm khỏi request.** Việc nặng (**transcode video**, **gửi email**) không nên chạy trong vòng đời HTTP. Đẩy vào queue → **HTTP không treo, trả `200` ngay**, việc nặng xử lý nền.

**2. Làm phẳng spike (buffer).** Khi traffic **burst** vượt sức **downstream** (service phụ thuộc), queue **giữ tạm** rồi nhả ra từ từ → **không sập** service yếu hơn.

**3. Retry việc lỗi tạm thời.** Lỗi chốc lát (mạng chập chờn) → **thử lại job** thay vì **fail cả request** của user.

Cộng thêm: **decouple producer/consumer** (bên tạo việc và bên làm việc tách rời) + **scale worker riêng**.

**Ẩn dụ:** queue như **tờ phiếu order dán lên dây chuyền bếp**. Khách (request) gọi món xong là **đi ngồi ngay (trả 200)**, không đứng đợi ở quầy. Bếp (worker) lần lượt làm theo phiếu; món cháy thì **làm lại phiếu đó** (retry), bếp đông khách thì **phiếu xếp hàng chờ** (buffer) chứ không vỡ trận.

---

### (b) BullMQ *(J-QUEUE-002)*

> **BullMQ** = thư viện queue cho **Node** chạy trên **Redis** (**kế nhiệm Bull**, **TypeScript-native**; nội bộ tận dụng **Redis Streams**). *(verify version — 6/2026 là 5.78.x.)*

Các **primitive** chính:
- **Job states:** `waiting` / `active` / `delayed` / `completed` / `failed`.
- **Retry + backoff**, **delay**, **priority**, **rate limit**, **repeatable**.
- **Parent-child flow** (job phụ thuộc nhau).
- **Dashboard** (**Bull Board**).

---

### (c) Redis Streams vs list-based queue *(J-QUEUE-003)*

| Tiêu chí | **List** (`LPUSH`/`BRPOP`) | **Streams** |
|---|---|---|
| **Mô hình** | Queue đơn giản, **lấy ra là mất** | **Append-only log** có **ID** |
| **Replay** (đọc lại lịch sử) | ❌ | ✅ |
| **Nhiều consumer độc lập** | ❌ | ✅ (**consumer group** đọc độc lập) |
| **At-least-once** | Tự xử | **`XACK` + pending list + reclaim** job worker chết |

**Lệnh Streams cốt lõi:** **`XADD`** thêm message, **`XREADGROUP`** đọc theo group, **`XACK`** xác nhận đã xử lý.

> **Consumer group** làm 3 việc: **chia message giữa các worker** + **theo dõi pending** (cho **at-least-once**) + **reclaim job của worker chết**. *(verify lệnh)*

**Ẩn dụ List vs Streams:** **List** như **giỏ vé "lấy là hết"** — rút vé ra là không còn dấu vết, ai đã làm gì không biết. **Streams** như **sổ ghi chép có đánh số dòng** — đọc xong vẫn còn lưu (replay được), nhiều người có thể đọc theo nhóm, và có cột "đã xử lý xong chưa" (`XACK`).

---

### (d) Không mất job khi worker crash *(J-QUEUE-004)*

> Job phải được **claim atomic** + **đánh dấu đang xử lý**, và **ACK CHỈ KHI XONG**. Worker chết giữa chừng → job **pending** sẽ được **reclaim / retry** (qua **visibility timeout** / **stalled-job check**).

**Vì sao ACK phải ở cuối?** Vì nếu ACK *trước* khi làm xong rồi worker chết → job coi như xong nhưng *chưa làm* → **mất job thật**. ACK ở cuối đảm bảo: chưa xong = chưa ACK = sẽ được làm lại.

> 🚨 **Hệ quả tất yếu — at-least-once → idempotent:** vì job **có thể chạy lại**, **consumer PHẢI idempotent** (chạy 2 lần cho kết quả như 1 lần). *(cross-ref I-IDEM.)* **BullMQ** có cơ chế **stalled-job** tự reclaim.

---

### (e) DLQ / failed jobs + retry an toàn *(J-QUEUE-005)*

> Job fail **quá N lần** → đẩy sang **DLQ (Dead Letter Queue) / failed** để **điều tra**, thay vì **retry vô hạn**.

- **Retry phải có GIỚI HẠN** + **(jittered) backoff** (giãn cách thử lại, có random để tránh dồn cục).
- Job **phải idempotent** vì **retry = chạy lại**.

> ⚠️ **Vì sao cần backoff + giới hạn:** retry vô hạn không giãn cách tạo **retry storm** — hàng loạt job hỏng dội lại liên tục, đập downstream đang yếu càng yếu thêm. *(Cross-ref I-LOCK-008.)*

**Ẩn dụ DLQ:** như **khay "hồ sơ lỗi cần người xem"** — thư không gửi được sau 5 lần thì **đừng cố mãi**, bỏ vào khay riêng cho người phụ trách xử lý tay.

---

### (f) Khi nào BullMQ KHÔNG đủ → Kafka / RabbitMQ *(J-QUEUE-006)*

| Công cụ | Hợp khi |
|---|---|
| **BullMQ** | Job queue Node với **retry/delay**, hạ tầng nhẹ (**đã có Redis**) |
| **Kafka** | Cần **event streaming bền** + **replay lâu dài** + **throughput rất lớn** + **nhiều consumer độc lập** |
| **RabbitMQ** | **Routing phức tạp** / **AMQP** |

*(Đào sâu ở mục K.) (verify landscape)*

> 🎯 **Cách chọn nhanh:** việc nội bộ kiểu "gửi email/transcode" + đã có Redis → **BullMQ**. Cần "dòng sự kiện bền, đọc lại được, cực nhiều" (event sourcing, analytics) → **Kafka**. Cần định tuyến message tinh vi → **RabbitMQ**.

---

### (g) Tách Redis queue khỏi Redis cache *(J-QUEUE-007)* — nối Bài 6

> **Cache** cần **eviction** (`allkeys-lru` — vứt bớt khi đầy là chuyện thường). **Job "KHÔNG ĐƯỢC MẤT"** → cần **`noeviction` / persistence**. **Hai nhu cầu trái ngược.**

**Vì sao chung instance là thảm hoạ (cả hai chiều):**
- **Eviction xoá nhầm job:** policy cache `allkeys-*` đá luôn job key → **mất việc**.
- **Job ngốn memory đẩy cache ra:** queue đầy job → chiếm RAM → **đẩy cache key ra** → hit ratio sập.

Hai thứ còn có **tải & failure mode khác nhau**. → **TÁCH instance.** *(Cross-ref J-EVICT-005.)*

---

## ③ ⚠️ Kiến thức cũ / bị thay thế

| Cái CŨ | Thay bằng (hiện nay) | Vì sao |
|---|---|---|
| **Bull** | **BullMQ 5.x** (TS-native, Redis Streams) *(verify)* | Bull **đã được kế nhiệm** |
| **List-based queue tự chế** (`LPUSH`/`BRPOP`) | **Streams + consumer group** (hoặc BullMQ) | List **không replay / không group / khó at-least-once** |
| **Retry vô hạn không backoff** | Giới hạn N + **jittered backoff** + **DLQ** | Tránh **retry storm**; điều tra job hỏng |
| Chung Redis **cache + queue**, `allkeys-lru` | **Tách instance** (queue `noeviction`/persist) | **Evict nhầm job** |
| Consumer **không idempotent** | **Idempotent** (at-least-once) | Job **có thể chạy lại** (cross-ref I-IDEM) |

---

## ④ Áp dụng thực tế + So sánh bigtech

**Use case:**
- **signup** → **trả `200` ngay**, đẩy job **"welcome email"** vào **BullMQ**; worker riêng xử lý + **retry + DLQ**.
- **Video upload** → **transcode job** (chậm) **tách khỏi request**.
- **Throughput cực lớn + cần replay/streaming bền** (event sourcing, analytics pipeline) → **Kafka**.

**Pattern ở bigtech:**
- **Node shop** thường dùng **BullMQ** cho job nội bộ; hệ thống **event-driven lớn** dùng **Kafka** làm backbone.
- **DragonflyDB** được quảng cáo **tương thích BullMQ** (đa-core).
- **Worker là process RIÊNG**; **dashboard (Bull Board) phải auth** (**đừng phơi ra internet**). *(verify)*

---

## ⑤ Code thực hành + cấu hình

**BullMQ producer + worker** (pin `bullmq@5`, verify bản mới nhất):

```javascript
// queue.js
import { Queue, Worker } from "bullmq";
import IORedis from "ioredis";

// BullMQ yêu cầu maxRetriesPerRequest: null; nên dùng Redis RIÊNG cho queue
const connection = new IORedis(process.env.REDIS_QUEUE_URL, { maxRetriesPerRequest: null });

export const emailQueue = new Queue("email", { connection });

// Producer: enqueue NGAY trong route, KHÔNG xử lý trong request
await emailQueue.add(
  "welcome",
  { userId, email },
  { attempts: 5, backoff: { type: "exponential", delay: 1000 }, removeOnComplete: 1000 }
); // attempts + backoff → retry an toàn; fail quá 5 lần → vào 'failed' (DLQ)

// Worker (process RIÊNG)
new Worker(
  "email",
  async (job) => {
    // ⚠️ phải IDEMPOTENT: job có thể chạy lại (at-least-once)
    await sendEmailIdempotent(job.data.userId, job.data.email);
  },
  { connection, concurrency: 10 }
);
```

```bash
# Redis instance của QUEUE (TÁCH khỏi cache):
maxmemory-policy noeviction
appendonly yes
```

> ⚠️ **`REDIS_QUEUE_URL` ≠ `REDIS_CACHE_URL`** (tách instance).
> ⚠️ **Worker chạy process riêng**; **dashboard phải có auth**.
> ⚠️ **Job idempotent** (cross-ref I-IDEM).

---

## ⑥ Keywords cần nhớ (glossary)

🧠 **Cần HIỂU:**
- **3 lớp lý do dùng queue** (tách việc chậm / phẳng spike / retry).
- **Streams vs list** (replay / group / ACK).
- **at-least-once → idempotent**.
- **DLQ / backoff**.
- **Khi nào cần Kafka**.
- **Vì sao tách queue khỏi cache**.

📌 **Cần THUỘC:**
- **BullMQ trên Redis Streams**.
- **`XADD` / `XREADGROUP` / `XACK`**.
- **attempts + backoff + DLQ**.
- **queue instance `noeviction` + persist**.

🛠️ **Cần LÀM ĐƯỢC:** dựng **BullMQ producer/worker** với **retry/backoff/DLQ**; **worker idempotent**; **tách instance**.

---

## ⑦ Mental model

> **Queue = tách việc chậm + làm phẳng spike + retry.**
> **BullMQ** (trên **Redis Streams**) cho Node; **consumer group + `XACK` + reclaim = at-least-once** → **job phải idempotent**.
> **Tách Redis queue** (`noeviction`/persist) **khỏi cache** (`allkeys-lru`).
> **Quá tải / streaming bền → Kafka.**

---

## ⑧ Câu hỏi phỏng vấn & thách đố

- **(TB)** Ba lớp vấn đề job queue giải? → **tách việc chậm, phẳng spike, retry lỗi tạm**.
- **(TB)** BullMQ là gì, dựa trên đâu? → queue Node trên **Redis (Streams)**, **kế nhiệm Bull**, TS-native.
- **(khó)** Streams khác list-queue thế nào, consumer group cho gì? → **append-log + replay**; group **chia message + pending (`XACK`) + reclaim**.
- **(khó)** Đảm bảo job không mất khi worker crash? → **claim atomic + ACK khi xong + reclaim pending**; consumer **idempotent**.
- **(rất khó)** Khi nào BullMQ không đủ, dùng Kafka? → **streaming bền + replay lâu + throughput rất lớn + nhiều consumer**.

> **🧩 Thách đố:** *"Worker gửi email crash SAU KHI gửi xong nhưng TRƯỚC KHI ACK. Chuyện gì xảy ra, fix sao?"*
>
> **Trả lời:**
> ```
> worker: gửi email XONG ✅
> worker: 💥 CRASH (chưa kịp XACK)
> → job vẫn PENDING → bị reclaim → retry → GỬI EMAIL LẦN HAI ❌ (trùng)
> ```
> Đây là bản chất **at-least-once** (có thể trùng). **Fix — idempotent:** lưu **"đã gửi cho job-id này"** (**dedupe key**), hoặc dùng **idempotency key** phía provider email. **Đừng giả định exactly-once.**

---

## ⑨ Bài tập + tiêu chí tự chấm

**BT1.** Dựng **BullMQ queue** gửi email với **retry 5 lần + exponential backoff + DLQ**.
> ✅ **Đạt khi:** **attempts + backoff** cấu hình; **job idempotent**; **worker process riêng**.

**BT2.** Giải thích vì sao đặt queue **chung Redis với cache `allkeys-lru`** là sai + sửa.
> ✅ **Đạt khi:** chỉ ra **evict nhầm job** → **tách instance `noeviction` + persistence**.

---

## ⑩ Đọc thêm

- **bullmq.io/docs** — *Queues, Workers, Flows, Rate limiting* (verify version 5.x).
- **redis.io/docs** — *Streams* (`XADD`/`XREADGROUP`/`XACK`), *consumer groups*.
- Mục **K (Messaging):** Kafka vs RabbitMQ vs Redis Streams, **Outbox/CDC**.

---

> 🔎 **AI hay sai ở đây:** AI giả định **"exactly-once"** và viết **consumer không idempotent**; và **đặt queue chung Redis cache**.
> **Luôn tự kiểm:** *(1) job chạy 2 lần có hại không?* + *(2) queue có nằm trên instance có eviction không?*
