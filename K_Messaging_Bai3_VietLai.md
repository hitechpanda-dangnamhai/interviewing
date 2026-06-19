# 🎯 BÀI 3 — Kafka Internals I: partition, consumer group, offset

**Phủ:** K-KAFKA-001 → 005

> 💡 **Mục tiêu một câu:** Sau bài này, bạn **hết coi Kafka là hộp đen**. Bạn hiểu *vì sao* event của cùng một đơn hàng giữ đúng thứ tự, *vì sao* thêm consumer không phải lúc nào cũng nhanh hơn, và *vì sao* commit đặt sai chỗ làm **mất message**.

---

## ① Mục tiêu & vị trí trong mạch

**Kafka** là **broker** thống trị mảng **event-streaming**, nên **hai bài Kafka (3 & 4)** là *trục xương sống* của cả **Mục K**. Mọi câu hỏi về **delivery** (giao nhận), **ordering** (thứ tự), **Outbox**, **Saga** ở các bài sau đều *dựa trên cơ chế dựng ở đây*. Nếu phần này lỏng, các bài sau sẽ thành học vẹt.

- **Bài 3 (bài này):** dựng nền **partition / offset / consumer group** — bộ ba "vật lý" của Kafka.
- **Bài 4:** thêm **replication / rebalance / EOS** (exactly-once semantics).

> 🎯 **Chốt vị trí:** học xong bài này, bạn nhìn vào một topic là *thấy được* dữ liệu nằm ở đâu, ai đọc cái gì, và đảm bảo thứ tự tới mức nào.

---

## ② Giảng cơ bản → nâng cao

### (a) Trực giác

> 📼 **Ẩn dụ — topic là một "chủ đề", chia thành nhiều băng ghi song song.**
> Hình dung **topic** = một chủ đề (ví dụ `orders`). Nó được xẻ thành nhiều **partition** = *nhiều cuốn băng ghi **append-only** chạy song song*. Mỗi message rơi vào **đúng một** partition và được dán một **offset** = *số thứ tự tăng dần trong cuốn băng đó*. Nhiều **broker** (server) giữ các partition; tập hợp broker = **cluster**.

```
topic "orders"
 ┌─ partition 0:  [off0][off1][off2][off3]──►  (băng ghi 1, có thứ tự)
 ├─ partition 1:  [off0][off1][off2]──►        (băng ghi 2, có thứ tự)
 └─ partition 2:  [off0][off1][off2][off3][off4]──►  (băng ghi 3, có thứ tự)
   → thứ tự CHỈ tồn tại trong từng băng, KHÔNG có thứ tự "chung" giữa 3 băng
```

---

### (b) Cơ chế 📘 K-KAFKA-001 → 005

#### Offset không duy nhất across partition

> ⚠️ **Bẫy nhận thức đầu tiên:** **offset 5** của **partition 0** *khác* message **offset 5** của **partition 1**. **Offset** chỉ **duy nhất trong một partition**, không duy nhất toàn topic.

🔍 *Vì sao điều này quan trọng?* Vì nếu bạn tưởng offset là "ID toàn cục", bạn sẽ tin có một "dòng thời gian chung" — nhưng Kafka *không* có. Mỗi partition tự đếm từ 0.

---

#### Partition vừa là đơn vị song song, vừa là đơn vị ordering 📘 K-KAFKA-002

Đây là **hai vai trò gộp vào một** — gốc rễ của mọi trade-off Kafka:

> 🎯 **Khẩu quyết:** *Kafka chỉ đảm bảo **thứ tự trong MỘT partition**, **KHÔNG** đảm bảo thứ tự **across partition**.*

Từ đó suy ra hai hệ quả:

1. **Số partition = trần song song** của một **consumer group** (giải thích ở phần dưới).
2. **Tăng partition** → tăng **throughput**, NHƯNG có cái giá:

| Tiêu chí | Ít partition | Nhiều partition |
|---|---|---|
| **Throughput** | thấp (ít luồng song song) | cao (nhiều luồng) |
| **Thứ tự global** | gần hơn (ít băng) | ❌ vỡ — không có thứ tự chung |
| **Overhead** | ít file/metadata | nhiều file, nhiều overhead |
| **Mapping key→partition** | ổn định | 🚨 **có thể đổi** vì `hash(key) % N` đổi khi **N** đổi |

> ⚠️ Đây chính là **trade-off ordering ↔ throughput kinh điển**. Muốn nhanh hơn (nhiều partition) thì hy sinh thứ tự; muốn thứ tự chặt thì ít partition lại.

---

#### Consumer group 📘 K-KAFKA-003

> 👥 **Ví dụ trực giác:** một **consumer group** giống *một đội nhân viên chia nhau các quầy*. Có 3 quầy (partition) thì tối đa 3 người làm thật; người thứ 4 vào đội chỉ *đứng nhìn*.

> 🎯 **Quy tắc vàng:** *Mỗi **partition** được gán cho **đúng 1 consumer** trong cùng group.*

Hai hệ quả phải thuộc:

```
TRƯỜNG HỢP #consumer > #partition (3 partition, 4 consumer):
  partition 0 ──► consumer A   ✅ làm việc
  partition 1 ──► consumer B   ✅ làm việc
  partition 2 ──► consumer C   ✅ làm việc
                  consumer D   ❌ IDLE (ngồi không!)
  → Muốn scale tiêu thụ phải TĂNG PARTITION, không chỉ thêm consumer
```

- **#consumer > #partition** → consumer thừa **idle** (ngồi không). ⚠️ Muốn **scale** tiêu thụ thì **tăng partition**, *không* chỉ thêm consumer.
- **Nhiều group khác nhau** đọc **độc lập** cùng một topic → *đây chính là* **pub/sub**: mỗi group giữ **offset riêng**.

```
PUB/SUB qua nhiều group — mỗi group có offset riêng:
  topic "orders"
     ├──► group "inventory-svc"  (đang đọc tới offset 100)
     ├──► group "email-svc"      (đang đọc tới offset 80)
     └──► group "analytics-svc"  (đang đọc tới offset 5 — đọc lại từ đầu)
  ✅ ba group hoàn toàn không ảnh hưởng nhau
```

---

#### Producer chọn partition 📘 K-KAFKA-004

| Trường hợp | Cách chọn partition | Hệ quả |
|---|---|---|
| **Có key** | `hash(key) % #partition` | **cùng key → luôn cùng partition** → giữ **ordering theo thực thể** |
| **Không key** | **round-robin / sticky** (🔎 *verify default partitioner — đã đổi qua các đời*) | trải đều, không đảm bảo entity-ordering |

> 💡 **Ví dụ cụ thể — `key = orderId`:** mọi event của *một* order (`created → paid → shipped`) đều rơi *cùng một partition*, nên **xử lý đúng thứ tự**. Trong khi đó, các order *khác nhau* trải đều nhiều partition → vẫn **song song**.

> 🎯 **Chốt:** *Key là công cụ giữ **ordering per-entity**.* Đây chính là câu trả lời cho *"làm sao event của cùng một order đúng thứ tự"* (gặp lại ở **Bài 5**).

---

#### Offset commit 📘 K-KAFKA-005 — điểm AI hay sai

> 🚨 **Điểm AI hay sai ở đây:** đặt **commit** *sai chỗ* quyết định bạn rơi vào **at-most-once** hay **at-least-once**.

Vẽ timeline cho rõ:

```
COMMIT TRƯỚC khi xử lý  →  at-most-once (rủi ro MẤT message):
  T1  nhận message
  T2  commit offset        ✅ Kafka tưởng "đã xong"
  T3  bắt đầu xử lý...
  T4  💥 crash giữa chừng
  T5  restart → đọc từ offset đã commit → BỎ QUA message này  ❌ MẤT

COMMIT SAU khi xử lý  →  at-least-once (rủi ro XỬ LÝ LẠI):
  T1  nhận message
  T2  xử lý + ghi side-effect  ✅
  T3  💥 crash TRƯỚC khi commit
  T4  restart → offset chưa tiến → đọc LẠI message này  ⚠️ xử lý 2 lần
  → cần IDEMPOTENT (Bài 5) để xử lý lại không gây hại
```

> 🎯 **An toàn mặc định:** **process rồi mới commit** (**manual commit**). Offset được lưu ở topic nội bộ **`__consumer_offsets`**. (🔎 *verify*.)

---

### (c) Mép giới hạn

**Auto-commit** (`enable.auto.commit=true`) commit **theo chu kỳ thời gian**, *không* theo "đã xử lý xong" → dễ rơi vào **at-most-once âm thầm** nếu xử lý **bất đồng bộ** (commit chạy nền trong khi việc thật chưa xong).

> 🧠 **Hiểu cho đúng nghĩa của commit:** *commit/ack nghĩa là **"tôi đã XỬ LÝ XONG"**, KHÔNG phải **"tôi đã NHẬN"**.* Nhầm hai cái này là gốc của hầu hết lỗi mất/lặp message.

---

## ③ ⚠️ Cũ → Mới

| Cái cũ | Nay | Vì sao |
|---|---|---|
| **ZooKeeper** giữ metadata/offset cũ | **KRaft** (4.x); offset consumer ở **`__consumer_offsets`** | Bớt một hệ vận hành (🔎 verify) |
| **"Thêm consumer là scale được"** | Trần = **#partition**; thừa consumer **idle** → **tăng partition** | Quy tắc **1 partition – 1 consumer/group** |
| **`enable.auto.commit=true`** cho tiện | **Manual commit**, **process-then-commit** | Auto-commit dễ **at-most-once** ngầm |

---

## ④ Áp dụng thực tế + bigtech

**Use case:** xử lý event order **đúng thứ tự per order** nhưng vẫn **song song giữa các order khác nhau**.

> 🎯 **Lời giải chuẩn:** **`key = orderId`**.
> - Mỗi order → 1 partition → **đúng thứ tự** (per-entity ordering).
> - Nhiều order → trải đều nhiều partition → **song song** (throughput).

```
key=orderId  →  hash(orderId) % 3
  order A (mọi event) ──► partition 1  (A.created → A.paid → A.shipped đúng thứ tự ✅)
  order B (mọi event) ──► partition 0  (song song với A ✅)
  order C (mọi event) ──► partition 2  (song song ✅)
```

**Bigtech:** đây là cách *chuẩn* để vừa giữ **per-entity ordering** vừa **scale** — dùng rộng rãi trong xử lý **giao dịch / ledger**. (pattern phổ biến.)

---

## ⑤ Code thực hành + cấu hình

**KafkaJS** — producer có **key** + consumer **commit thủ công**:

```javascript
// producer.js — key = orderId để giữ ordering per-order
const { Kafka } = require('kafkajs'); // pin "kafkajs@2.x" // verify đời mới
const kafka = new Kafka({ clientId: 'orders', brokers: (process.env.KAFKA_BROKERS||'').split(',') });
const producer = kafka.producer();
await producer.connect();

await producer.send({
  topic: 'orders',
  // ⭐ CÙNG key (orderId) -> CÙNG partition -> đúng thứ tự cho mọi event của 1 order
  messages: [{ key: order.id, value: JSON.stringify(evt) }],
});
```

```javascript
// consumer.js — process THEN commit (at-least-once an toàn)
const consumer = kafka.consumer({ groupId: 'inventory-svc' }); // group riêng = đọc độc lập (pub/sub)
await consumer.connect();
await consumer.subscribe({ topic: 'orders', fromBeginning: false });

await consumer.run({
  autoCommit: false, // ⚠️ TẮT auto-commit để tự kiểm soát thời điểm commit
  eachMessage: async ({ topic, partition, message }) => {
    // 1) XỬ LÝ + ghi side-effect TRƯỚC (phải idempotent — Bài 5, vì at-least-once có thể lặp)
    await handle(JSON.parse(message.value));

    // 2) Chỉ commit SAU KHI xử lý xong (offset + 1 = vị trí đọc tiếp theo)
    await consumer.commitOffsets([
      { topic, partition, offset: (Number(message.offset) + 1).toString() },
    ]);
  },
});
```

```bash
# config broker/topic cần verify khi học (đổi theo đời):
#   num.partitions, retention.ms, enable.auto.commit (client), group.protocol=consumer (KIP-848)
```

> 🎯 **Hai dòng vàng của đoạn consumer:** `autoCommit: false` (tắt tự động) + `commitOffsets` đặt *sau* `handle`. Đảo thứ tự hai cái này = bạn vừa biến hệ thống thành **at-most-once** (mất message khi crash).

---

## ⑥ Keywords

**🧠 Cần HIỂU:**
- **partition** = song song + ordering (hai vai trò gộp một).
- **quy tắc 1 partition – 1 consumer/group**.
- **key → partition** (giữ entity-ordering).
- **offset semantics**; **commit trước/sau** xử lý.

**📌 Cần THUỘC:**
- **topic / partition / offset / broker / cluster**.
- **`hash(key) % N`**.
- **`__consumer_offsets`**.
- **at-most-once** vs **at-least-once** theo *thời điểm commit*.

**🛠️ Cần LÀM ĐƯỢC:**
- Chọn **key** để giữ **ordering**.
- Cấu hình **manual commit** kiểu **process-then-commit**.
- Biết *khi nào* **tăng partition** là cần thiết (và cái giá của nó).

---

## ⑦ Mental model

> 🧠 **"1 partition = 1 băng ghi có thứ tự, 1 consumer/group đọc nó.**
> **Key quyết định partition ⇒ key quyết định ordering.**
> **Commit nghĩa là 'đã xử lý xong', nên đặt SAU khi xử lý."**

---

## ⑧ Phỏng vấn & thách đố

**(dễ)** **topic / partition / offset / broker** quan hệ thế nào? **Offset** có duy nhất across partition?
→ Không — offset chỉ duy nhất *trong* một partition.

**(TB)** Vì sao **partition** vừa là **song song** vừa là **ordering**? Tăng partition hại gì?
→ Một partition là 1 luồng có thứ tự; tăng partition tăng throughput nhưng phá thứ tự global + đổi mapping `hash(key)%N`.

**(TB)** **#consumer > #partition** thì sao?
→ consumer thừa **idle**; phải tăng partition mới scale được.

**(khó)** Producer quyết **partition** bằng gì? Vai trò của **key**?
→ có key thì `hash(key)%N`; key giữ ordering per-entity.

**(khó)** **Commit trước** vs **sau** xử lý → semantics nào?
→ trước = **at-most-once** (mất); sau = **at-least-once** (lặp, cần idempotent).

> 🧩 **Thách đố:** *"Đội tăng partition từ **6→12** để 'nhanh hơn', sau đó khách báo event của cùng một đơn xử lý **sai thứ tự**. Vì sao?"*
> → Vì **`hash(key) % N` đổi khi N đổi** → message cùng key có thể rơi **partition khác** so với trước → **mất ordering per-key** trong giai đoạn chuyển + giữa message cũ/mới (cũ ở partition này, mới ở partition kia, đọc không còn theo trình tự). **Đổi số partition không nên làm bừa khi cần ordering theo key.**

---

## ⑨ Bài tập + tiêu chí

**Đề:** Topic `payments`, cần: (1) mọi event của cùng **`accountId`** đúng thứ tự, (2) **song song tối đa 8 luồng**. Chọn **key** + **số partition tối thiểu** + giải thích **trần consumer**.

> ✅ **Đạt khi:**
> - **`key = accountId`** (giữ ordering per-account).
> - **partition ≥ 8** (vì **trần song song = #partition**).
> - Nêu rõ: thêm consumer *quá 8* sẽ **idle**.
> - Chấp nhận **không có global order** (chỉ có thứ tự per-account).

---

## ⑩ Đọc thêm

- **`kafka.apache.org/documentation`** — phần **Design**, **Implementation**.
- **KafkaJS docs**. (🔎 *verify cấu hình*.)

> 🎯 **Một câu mang về:** *Trong Kafka, **key chọn partition**, **partition giữ thứ tự**, **commit đánh dấu "đã xong"** — nắm ba mệnh đề này là nắm 80% lỗi delivery của hệ thống.*
