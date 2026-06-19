# 🎯 BÀI 5 — Delivery Semantics & Idempotent Consumer

**Phủ:** K-DELIV-001 → 009

> 💡 **Mục tiêu một câu:** Đây là **bài lõi reliability** của cả **Mục K**. Học xong, bạn hiểu *vì sao* hệ thống thực tế mặc định để message **lặp** (chứ không mất), và *vì sao* consumer **bắt buộc** phải **idempotent** — nếu không, kho bị trừ hai lần, khách bị tính tiền hai lần.

---

## ① Mục tiêu & vị trí trong mạch

Đây là **bài lõi reliability** — mọi câu về **Outbox / Saga / resilience** ở các bài sau đều *dựa vào nó*. Học xong bạn nắm:
- ba mức **at-most-once / at-least-once / exactly-once**,
- *vì sao* **at-least-once** là **mặc định thực tế** và consumer **bắt buộc idempotent**,
- cách xử lý **ordering**,
- **DLQ / poison message**,
- **head-of-line blocking** khi retry.

> 🎯 **Chốt vị trí:** cross-ref **I-IDEM** (đã học idempotency ở góc concurrency) — ở đây ta nhìn *cùng vấn đề* nhưng từ **góc cơ chế messaging** (id gắn vào message, không phải vào request HTTP).

---

## ② Giảng cơ bản → nâng cao

### (a) Trực giác — 3 mức delivery 📘 K-DELIV-001

> 📦 **Ẩn dụ — gửi bưu kiện quan trọng:**
> - **At-most-once** = *"gửi một lần rồi thôi, mất kệ"* — bạn bỏ bưu kiện vào hòm, không quan tâm có tới không. **Không bao giờ trùng**, nhưng **có thể mất**.
> - **At-least-once** = *"gửi lại tới khi chắc chắn họ nhận"* — bạn gửi đi gửi lại tới khi có ký nhận. **Không mất**, nhưng **có thể tới hai lần** (người nhận thấy hai bưu kiện giống hệt).
> - **Exactly-once** = *"đúng một lần, không hơn không kém"* — lý tưởng đẹp, nhưng qua biên mạng thì *gần như bất khả*.

| Mức | Cơ chế | Mất? | Trùng? | Thực tế |
|---|---|---|---|---|
| **At-most-once** | **commit/ack TRƯỚC** khi xử lý | ❌ có thể mất | ✅ không trùng | hiếm dùng cho việc quan trọng |
| **At-least-once** | **xử lý TRƯỚC rồi mới ack/commit** | ✅ không mất | ⚠️ có thể trùng | 🎯 **mặc định phổ biến** |
| **Exactly-once delivery** qua biên mạng | — | gần như **bất khả** (**FLP / two-generals**) | — | thay bằng **at-least-once + dedup = effectively-once** |

> 🚨 **Sự thật cốt lõi:** **Exactly-once *delivery* qua biên mạng = gần như bất khả** (bài toán **FLP** / **two-generals**). Cái bạn *làm được* trong thực tế là **at-least-once + dedup = "effectively-once"**. (cross-ref **I-IDEM-002**.)

---

### (b) Cơ chế

#### Vì sao consumer BẮT BUỘC idempotent dưới at-least-once 📘 K-DELIV-002

> ⚠️ Dưới **at-least-once**, **broker sẽ redeliver** (do **timeout**, **rebalance**, **retry**, **crash trước commit**) → **side-effect chạy 2 lần**.

```
TIMELINE — vì sao redeliver xảy ra:
  T1  consumer nhận message (trừ kho)
  T2  xử lý xong: kho 100 → 99  ✅
  T3  💥 crash TRƯỚC khi commit offset
  T4  restart → offset chưa tiến → broker REDELIVER cùng message
  T5  xử lý LẠI: kho 99 → 98  ❌ trừ NHẦM lần hai!
  → nếu KHÔNG idempotent, mỗi crash = một lần trừ kho sai
```

**Cơ chế dedup messaging-specific:**
- lưu **`messageId` / `eventId`** đã xử lý vào **dedup store** (thấy lại → **bỏ qua**),
- hoặc dùng **unique constraint** khi ghi DB.

> 🔍 **Khác với HTTP Idempotency-Key (F-IDEM):** ở đây **id gắn vào message** (eventId trong payload), *không phải* header request.

---

#### Ack/nack & redelivery 📘 K-DELIV-003

> 🎯 **Quy tắc vàng:** *ack/commit **SAU** khi xử lý + ghi side-effect xong.*

| Hành động | Hệ quả |
|---|---|
| **Ack sớm** (trước khi xử lý) | crash giữa chừng = **mất message** ❌ |
| **Nack / không-ack** | broker **redeliver** (**RabbitMQ requeue**; **Kafka** đơn giản là **không commit offset**) |

> 🧠 **Câu thần chú (gặp lại từ Bài 3):** **ack = "tôi đã xử lý xong"**, KHÔNG phải **"tôi đã nhận"**.

---

#### Ordering 📘 K-DELIV-004

> 🎯 Đảm bảo event của **cùng một thực thể** đúng thứ tự bằng cách **route theo key**:

| Broker | Cách giữ ordering per-entity |
|---|---|
| **Kafka** | **cùng key → cùng partition** |
| **RabbitMQ** | **consistent-hash** hoặc **single-queue-per-key** |
| **SQS FIFO** | **message group id** |

> 💡 Thường **không cần global order**, chỉ cần **per-entity**. **Trade-off: ordering ↔ song song.**

---

#### Nghịch lý scale ↔ ordering 📘 K-DELIV-009

> 🧩 **Ví dụ trực giác:** bạn thuê thêm 5 nhân viên đóng gói cho nhanh, nhưng giờ bưu kiện của *cùng một khách* bị 5 người xử lý lẫn lộn → ra sai thứ tự.

```
TĂNG CONSUMER để giảm lag → vỡ thứ tự GLOBAL:
  Không key (round-robin):
    order A.created ──► partition 0 ──► consumer X
    order A.paid    ──► partition 2 ──► consumer Z   ❌ paid xử lý trước created?!

  Route theo key=orderId:
    order A.* (mọi event) ──► partition 1 ──► consumer Y  ✅ đúng thứ tự
    order B.* ──► partition 0 ──► consumer X  (song song với A ✅)
```

> 🎯 **Dung hoà:** **ordering chỉ cần per-key** (1 key → 1 partition → 1 consumer), **song song** giữa các key khác nhau. **Câu hỏi đúng luôn là: "ordering ở scope nào?"** — không phải "có cần ordering không".

---

### (c) Mép giới hạn — DLQ, poison, head-of-line

#### Poison message + DLQ 📘 K-DELIV-005

> ☠️ **Ví dụ trực giác:** một lá thư bị rách nát, máy đọc mãi không ra. Nếu cứ *đọc đi đọc lại* lá đó → cả dây chuyền đứng. Phải *bỏ riêng nó ra một ngăn* để người xử lý thủ công.

**Poison message** = message **luôn fail** (data hỏng / bug) → **retry vô hạn** làm **kẹt queue / chặn partition**.

> 🎯 **Giải:** sau **N lần fail** → đẩy sang **Dead Letter Queue (DLQ)** để điều tra thủ công, **không chặn luồng**.

> ⚠️ **Retry phải có:** **giới hạn** + **(jittered) backoff** (chờ tăng dần, có nhiễu ngẫu nhiên để tránh đồng loạt dồn lại); **alert** khi **DLQ** tăng.

---

#### In-line retry vs retry-topic 📘 K-DELIV-006

> 🚂 **Ví dụ trực giác:** Kafka xử lý **tuần tự per-partition** — như một toa tàu một làn. Một hành khách kẹt ở cửa = **cả toa phía sau đứng chờ**.

**Head-of-line blocking:** **block-and-retry** 1 message = **chặn mọi message sau nó** trong partition.

| Cách | Mô tả | Vấn đề / lợi |
|---|---|---|
| **In-line retry** (block) | retry ngay tại chỗ, tuần tự | 🚨 **head-of-line blocking** — chặn cả partition |
| **Retry topic** | đẩy message lỗi sang topic riêng, xử lý sau với **delay tăng dần** | ✅ không chặn luồng chính |
| **Delay queue** | hàng đợi trễ riêng | ✅ tương tự |

> 🔍 **Phân biệt:** lỗi **transient** (tạm thời — mạng chập, DB bận → **retry**) vs lỗi **vĩnh viễn** (data hỏng → **DLQ ngay**, đừng phí lần retry).

---

#### Consumer chậm + visibility timeout 📘 K-DELIV-007

```
TIMELINE — visibility timeout hết hạn giữa lúc đang xử lý:
  T1  worker A nhận message, visibility timeout = 30s
  T2  worker A xử lý CHẬM (> 30s)...
  T3  ⏰ visibility timeout/lease HẾT → broker tưởng A chết
  T4  broker REDELIVER → worker B nhận CÙNG message
  T5  A và B cùng xử lý 1 message  ❌ double-process
  → BẮT BUỘC idempotent + atomic claim
```

**Xử lý:** **tách side-effect nặng** / **đảm bảo ack nhanh** / **chia nhỏ**. (cross-ref **I-IDEM parallel retry**.)

---

#### Effectively-once end-to-end 📘 K-DELIV-008

> 🎯 **Ghép từ 3 mảnh — KHÔNG có nút thần kỳ:**

1. **Producer idempotent / Outbox** → publish **không mất / không trùng** (**Bài 6**).
2. **Broker durable + at-least-once** (**Bài 4**).
3. **Consumer dedup theo eventId** + ghi DB idempotent (**upsert / unique**) **atomic** với việc lưu "đã xử lý" (**Inbox** — **Bài 6**).

```
END-TO-END effectively-once = chuỗi 3 lớp:
  [Producer] idempotent/Outbox ──► [Broker] durable+at-least-once ──► [Consumer] dedup+atomic
       không trùng/mất                  không mất                       không double-process
  ❌ thiếu bất kỳ lớp nào → cả chuỗi KHÔNG còn effectively-once
```

---

## ③ ⚠️ Cũ → Mới

| Cái cũ | Nay | Vì sao |
|---|---|---|
| **"Dùng exactly-once delivery cho chắc"** | **at-least-once + dedup = effectively-once** | **EOD** bất khả ở biên ngoài |
| **Retry ngay tại consumer (block) trên Kafka** | **Retry topic / delay queue** + **DLQ** | Tránh **head-of-line blocking** |
| **"Ack ngay khi nhận cho nhẹ"** | **Ack sau khi xử lý xong** | Ack sớm = **mất message** |
| **Cần global ordering** | **Ordering per-key** là đủ trong hầu hết domain | Mở khoá **song song** |

---

## ④ Áp dụng thực tế + bigtech

**Use case:** consumer **trừ kho** khi nhận **`OrderPlaced`**.

> 🚨 **Vấn đề:** **at-least-once** → có thể nhận **2 lần** → nếu **không idempotent** sẽ **trừ kho gấp đôi**.

**Vá:** bảng **`processed_events(event_id PK)`**, trong **một transaction**:
1. kiểm tra/chèn **`event_id`**,
2. trừ kho;
3. trùng → **`event_id` đã tồn tại** → **skip**.

**Bigtech:** mọi pipeline **at-least-once** nghiêm túc đều có lớp **dedup + DLQ + retry-topic**; **payment systems** coi **idempotency** là **bắt buộc tuyệt đối**. (pattern.)

---

## ⑤ Code thực hành + cấu hình

**Idempotent consumer (dedup atomic)** — Postgres + KafkaJS:

```javascript
// idempotentConsumer.js — dedup theo eventId TRONG cùng transaction với side-effect
await consumer.run({
  autoCommit: false,
  eachMessage: async ({ topic, partition, message }) => {
    const evt = JSON.parse(message.value);
    const client = await pool.connect();
    try {
      await client.query('BEGIN');
      // 1) chèn eventId; nếu đã có -> ON CONFLICT DO NOTHING -> rowCount=0 -> đã xử lý rồi
      const ins = await client.query(
        'INSERT INTO processed_events(event_id) VALUES($1) ON CONFLICT DO NOTHING', [evt.eventId]
      );
      if (ins.rowCount === 1) {
        // 2) side-effect CHỈ chạy khi event MỚI -> idempotent (gặp lại không trừ kho lần nữa)
        await client.query('UPDATE inventory SET qty = qty - $1 WHERE sku=$2', [evt.qty, evt.sku]);
      }
      await client.query('COMMIT'); // ⭐ dedup + side-effect NGUYÊN TỬ cùng nhau
    } catch (e) {
      await client.query('ROLLBACK');
      throw e; // throw -> KHÔNG commit offset -> redeliver (an toàn VÌ đã idempotent)
    } finally {
      client.release();
    }
    await consumer.commitOffsets([{ topic, partition, offset: (Number(message.offset)+1).toString() }]);
  },
});
```

```sql
-- bảng dedup (Inbox tối giản)
CREATE TABLE processed_events ( event_id TEXT PRIMARY KEY, processed_at TIMESTAMPTZ DEFAULT now() );
```

> 🚨 **AI hay sai:** đặt **dedup-check** và **side-effect** ở **2 transaction tách rời** → crash *giữa hai bước* vẫn **double-process**. **Phải nguyên tử** (cùng 1 transaction) hoặc dùng **unique constraint** trên chính bảng nghiệp vụ.

```
VÌ SAO phải cùng 1 transaction:
  ❌ Tách rời:
     TX1: INSERT event_id  ✅ commit
     💥 crash
     TX2: trừ kho           ← CHƯA chạy → khi redeliver: event_id đã có → SKIP → kho KHÔNG bị trừ ❌ (mất side-effect!)
  ✅ Cùng 1 TX:
     BEGIN → INSERT event_id + trừ kho → COMMIT  (cả hai cùng sống hoặc cùng chết)
```

---

## ⑥ Keywords

**🧠 Cần HIỂU:**
- **3 mức delivery**; **effectively-once**.
- *vì sao* **idempotent** bắt buộc.
- **ordering per-key**; **head-of-line blocking**; **poison / DLQ**.

**📌 Cần THUỘC:**
- **at-most** (commit trước) / **at-least** (commit sau) / **exactly** (bất khả ở biên).
- **ack = "đã xử lý xong"**.
- **dedup theo eventId**.
- **retry-topic** vs **in-line**.

**🛠️ Cần LÀM ĐƯỢC:**
- viết **idempotent consumer atomic**.
- thiết kế **DLQ + retry-topic + alert**.
- chọn **key** giữ **ordering per-entity**.

---

## ⑦ Mental model

> 🧠 **"Mặc định = at-least-once ⇒ consumer PHẢI idempotent** (**dedup eventId atomic**).
> **Ack sau khi xong.** **Ordering chỉ cần per-key.**
> **Retry bằng retry-topic, fail mãi → DLQ.**
> **Effectively-once = chuỗi cơ chế, KHÔNG phải một cờ."**

---

## ⑧ Phỏng vấn & thách đố

**(TB)** Định nghĩa + đánh đổi **3 mức delivery**.
→ at-most (mất, không trùng) / at-least (không mất, trùng) / exactly (bất khả ở biên → effectively-once).

**(khó)** Vì sao **at-least-once** bắt buộc **idempotent**? **Dedup messaging-specific** thế nào?
→ redeliver làm side-effect chạy 2 lần; dedup bằng eventId trong dedup store hoặc unique constraint.

**(TB)** **"Ack trước khi xử lý xong"** sai ở đâu?
→ crash giữa chừng = mất message (at-most-once ngầm).

**(khó)** Đảm bảo **ordering per-entity** bằng gì?
→ route theo key (Kafka cùng partition / RabbitMQ consistent-hash / SQS FIFO message group id).

**(rất khó)** **Effectively-once end-to-end** ghép từ những mảnh nào?
→ producer idempotent/Outbox + broker durable at-least-once + consumer dedup atomic (Inbox).

> 🧩 **Thách đố 1:** *"Tăng consumer giảm lag xong khách báo trạng thái order **nhảy lung tung** — giải thích nghịch lý và sửa."*
> → Song song nhiều partition **phá global order**; sửa: **route theo key** (`orderId → partition`), ordering *trong* key + song song *giữa* các key.

> 🧩 **Thách đố 2:** *"Một message **data hỏng** làm cả partition đứng. Vì sao và xử lý?"*
> → **head-of-line blocking** do **retry in-line tuần tự**; đẩy sang **retry-topic / DLQ** sau N lần, **không block partition**.

---

## ⑨ Bài tập + tiêu chí

**Đề:** Viết (pseudo hoặc thật) consumer **`PaymentCaptured`** cập nhật **`order.paid=true`**, đảm bảo: **idempotent**, có **DLQ sau 5 lần fail**, **không block partition**.

> ✅ **Đạt khi:**
> - **dedup theo eventId** **atomic** với update.
> - phân biệt **transient** (**retry-topic + backoff**) vs **permanent** (**DLQ ngay**).
> - **alert DLQ**.
> - giải thích *vì sao* `update paid=true` **vẫn cần dedup** (vì **redeliver** — dù update có vẻ idempotent sẵn, các side-effect kèm theo như "gửi mail đã thanh toán" thì không).

---

## ⑩ Đọc thêm

- **`kafka.apache.org`** (**Delivery Semantics**).
- **Confluent** retry/DLQ patterns.
- **Chris Richardson** — `microservices.io`: **"Idempotent Consumer"**, **"Transactional Outbox"**. (🔎 *verify*.)

> 🎯 **Một câu mang về:** *Đừng đi tìm cái nút "exactly-once". Hãy chấp nhận **at-least-once**, rồi làm consumer **idempotent** — đó mới là độ tin cậy thật, lắp ráp được, kiểm chứng được.*
