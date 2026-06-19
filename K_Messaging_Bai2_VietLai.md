# 🎯 BÀI 2 — Broker Landscape & chọn broker

**Phủ:** K-BRK-001 → 009

> 💡 **Mục tiêu một câu:** Bài 1 trả lời *"vì sao async"*. Bài này trả lời *"async **bằng cái gì**"* — và quan trọng hơn: **chọn đúng broker theo bài toán**, không chọn theo thương hiệu.

---

## ① Mục tiêu & vị trí trong mạch

**Bài 1** cho bạn cái *quyết định* sync vs async. Nhưng khi đã quyết "chỗ này async", bạn đứng trước một kệ hàng đầy công cụ: **Kafka, RabbitMQ, NATS, SQS/SNS, Redis Streams**... Chọn sai → hoặc *over-engineer* (dựng Kafka cho vài nghìn job/ngày), hoặc *under-engineer* (ép RabbitMQ làm streaming + replay mà nó không sinh ra để làm).

Bài này dạy bạn ba việc:
1. Phân biệt **ba mô hình nền tảng**: **log** vs **queue** vs **pub/sub**.
2. So sánh **Kafka / RabbitMQ / NATS / SQS-SNS / Redis Streams** theo *bản chất*, không theo marketing.
3. Có một **bộ tiêu chí chọn** để biện minh được lựa chọn của mình.

> 🎯 **Chốt vị trí:** đây là cầu nối tới **Bài 3–4** (đào sâu **Kafka** — broker thống trị mảng event-streaming). Góc Node (job queue nhẹ trên Redis) tham chiếu **J-QUEUE** (**BullMQ** / **Redis Streams**).

---

## ② Giảng cơ bản → nâng cao

### (a) Trực giác — log vs queue 📘 K-BRK-001

> 🏪 **Ẩn dụ — Queue truyền thống (RabbitMQ) = quầy phát thư.**
> Lá thư tới quầy, **consumer** lấy đi, nhân viên quầy **gạch sổ (ack) rồi vứt bản gốc**. Đọc xong là *mất*. Nhưng quầy có **bộ định tuyến rất giỏi**: biết thư này nên đưa quầy nào (routing linh hoạt).

> 📼 **Ẩn dụ — Log (Kafka) = cuốn băng ghi append-only.**
> Mọi message được **ghi nối đuôi** vào băng, giữ lại theo **retention** (ví dụ 7 ngày). Mỗi người đọc **tự cầm bút đánh dấu mình đọc tới đâu** (**offset**). Nhiều nhóm đọc *cùng cuốn băng độc lập*, và bất cứ lúc nào cũng **tua lại (replay)** được.

Khác biệt gốc rễ: **queue "đẩy đi rồi xoá"**, còn **log "giữ lại, ai đọc tới đâu tự nhớ"**. Toàn bộ bảng dưới chỉ là hệ quả của hai triết lý này.

| Tiêu chí | **Kafka (log)** | **RabbitMQ (queue/broker)** |
|---|---|---|
| **Lưu trữ** | **append-only log**, giữ theo **retention** | đẩy tới consumer, **xoá sau ack** |
| **Vị trí đọc** | consumer giữ **offset**, **replay** được | broker quản lý, **không replay** mặc định |
| **Nhiều consumer** | nhiều **consumer group** đọc độc lập *cùng topic* | cần **exchange/binding** để **fan-out** |
| **Routing** | đơn giản (**topic** + **partition** theo **key**) | mạnh (**direct/topic/fanout/headers exchange**) |
| **Mạnh ở** | **throughput** rất lớn, **streaming**, **replay**, **CDC** | **routing** phức tạp, **per-message ack/priority**, **task queue** |

> 🔍 **Vì sao "replay" lại là siêu năng lực của log?** Vì khi bạn thêm một consumer *mới* (ví dụ một mô hình ML cần học từ lịch sử đơn hàng), nó có thể **đọc lại từ đầu băng**. Với queue, dữ liệu *đã bị xoá sau ack* — không còn gì để học lại.

---

### (b) Cơ chế

#### RabbitMQ exchange 📘 K-BRK-004

> 📮 **Ví dụ trực giác:** producer *không* gửi thư thẳng vào hòm (queue). Nó gửi tới một **bưu cục trung tâm (exchange)**; bưu cục nhìn **routing key** trên phong bì + các **binding** đã khai báo để quyết định nhét thư vào ≥0 hòm.

Bốn loại **exchange** (phải thuộc):

| Loại **exchange** | Cách định tuyến | Ví dụ cụ thể |
|---|---|---|
| **fanout** | **broadcast** mọi queue đã **bind** (bỏ qua routing key) | 1 event → tất cả phòng ban |
| **direct** | **routing key** khớp **chính xác** binding key | `payment.failed` → đúng queue xử lý lỗi |
| **topic** | khớp **wildcard** | `order.*.created` bắt mọi `order.vn.created`, `order.us.created` |
| **headers** | khớp theo **header** thay vì routing key | định tuyến theo `{region: "vn", tier: "vip"}` |

> 🎯 **Điểm mạnh cốt lõi của RabbitMQ:** **tách publish khỏi routing**. Producer chỉ "ném vào exchange", còn *ai nhận* là do binding quyết định → đổi luồng định tuyến **không sửa** producer.

---

#### Push vs Pull 📘 K-BRK-003

> ⚡ **Ẩn dụ:** **Push** = nhân viên *dúi* việc vào tay bạn liên tục. **Pull** = bạn *tự ra lấy* việc khi làm xong cái cũ.

| Tiêu chí | **Push (RabbitMQ)** | **Pull (Kafka)** |
|---|---|---|
| Ai chủ động | **broker đẩy** message tới consumer | **consumer kéo** theo nhịp của nó |
| Latency | **thấp** (gửi ngay) | thêm **một nhịp poll** |
| Consumer chậm | dễ bị **ngộp** → cần **prefetch/QoS limit** giới hạn số message **in-flight** | **tự backpressure** — tự bảo vệ mình |
| Batch | khó hơn | dễ **batch** |

> ⚠️ **Điều tệ nhất nếu quên prefetch (push):** broker dúi 10.000 message vào một consumer chỉ xử lý nổi 100 → consumer *phình RAM rồi chết*, message in-flight mất theo. **`prefetch`** chính là cái van chặn dòng.

```
PUSH không prefetch — consumer ngộp:
  Broker ──message──► Consumer (đang xử lý msg cũ)
         ──message──►   "khoan đã..."
         ──message──►   RAM phình ❌
         ──message──►   💥 OOM, chết

PULL (Kafka) — consumer tự nhịp:
  Consumer ──poll(lấy 20)──► Broker
     (xử lý xong 20)
  Consumer ──poll(lấy 20 tiếp)──► Broker   ✅ không bao giờ quá tải
```

---

#### AWS SQS/SNS 📘 K-BRK-005

| Dịch vụ | Bản chất | Đặc tính chính |
|---|---|---|
| **SQS standard** | queue managed | **at-least-once** + **best-effort ordering** + **throughput** rất cao |
| **SQS FIFO** | queue đúng thứ tự | **đúng thứ tự** + **dedup** (theo **message group id** / **deduplication id**), nhưng **throughput giới hạn** (🔎 *verify giới hạn TPS hiện tại*) |
| **SNS** | pub/sub | **fan-out** tới nhiều subscriber (**SQS, Lambda, HTTP**...) |

> 🎯 **Pattern kinh điển — SNS → nhiều SQS:** vừa **fan-out**, vừa cho **mỗi consumer một buffer (queue) riêng**.

```
                 ┌──► SQS-Inventory ──► InventorySvc  (buffer riêng)
  SNS:OrderPlaced┼──► SQS-Email     ──► EmailSvc      (buffer riêng)
                 └──► SQS-Analytics ──► AnalyticsSvc  (buffer riêng)
  ✅ EmailSvc chậm/ngộp → CHỈ SQS-Email dồn lại, Inventory & Analytics không bị ảnh hưởng
```

> 🔍 **Vì sao tách buffer riêng quan trọng?** Nếu *chung* một queue, một consumer chậm sẽ làm message dồn ứ ảnh hưởng tất cả. Tách ra → lỗi/độ chậm được **cô lập (isolation)**.

---

### (c) Mép giới hạn

#### Durability 📘 K-BRK-007

**Broker bền** không phải chỉ là "có nhận message". Bền thật cần **ba lớp**:
1. **persist** ra **disk** (không giữ trên RAM),
2. **replicate** sang nhiều node *trước khi* coi là "đã nhận",
3. **producer ack** để producer *biết chắc* broker đã giữ.

Cụ thể: **Kafka `acks=all`** chờ các bản sao trong **ISR** (in-sync replicas); **RabbitMQ publisher confirms** báo về khi đã giữ chắc.

> 🚨 **Thiếu các cơ chế này → message *bay hơi* khi một node chết.** (🔎 *verify config — Bài 4 đào sâu*.)

---

#### Broker là SPOF / bottleneck? 📘 K-BRK-008

> ⚠️ **Câu trả lời: CÓ THỂ.** Đưa mọi thứ qua một broker đơn lẻ = tạo một **SPOF** (single point of failure) mới.

Muốn **HA** (high availability) cần đủ bộ:

| Biện pháp | Vì sao cần |
|---|---|
| **cluster** + **replication factor ≥3** (Kafka) / **quorum queue** (RabbitMQ) | mất 1 node vẫn còn bản sao |
| **partition/shard** | **scale ngang** throughput, không nghẽn 1 node |
| client **retry** + **reconnect** | sống sót qua trục trặc mạng tạm thời |
| **Outbox** phía producer (**Bài 6**) | **không mất event** khi broker tạm sập |
| giám sát **lag** / **disk** | thấy nghẽn *trước khi* vỡ |

---

#### Schema tiến hoá 📘 K-BRK-009

> 🧠 **Mental model:** **event là một "API contract"** — không phải dữ liệu vứt đi. Nhiều consumer đang dựa vào hình dạng của nó.

Dùng **schema registry** + **versioning** + quy tắc **compatibility** (**backward** / **forward**):
- **thêm field optional** ✅ (consumer cũ bỏ qua được),
- **không đổi nghĩa field cũ** ✅,
- consumer **bỏ qua field lạ** ✅,
- đổi *lớn* (breaking) → **tách version event** mới.

> 🔍 *verify khi triển khai:* **Avro / Protobuf / JSON Schema** + registry như **Confluent** / **Apicurio**.

---

## ③ ⚠️ Cũ → Mới

| Cái cũ | Nên dùng nay | Ghi chú |
|---|---|---|
| **"Kafka thay được RabbitMQ cho mọi thứ"** | Chọn theo nhu cầu: **Kafka** cho **streaming/replay/throughput**; **RabbitMQ** cho **routing/task queue** | Hai mô hình khác nhau (**log** vs **queue**) |
| **Kafka + ZooKeeper** | **Kafka KRaft-only** (4.x, **không còn ZooKeeper**) | Đỡ một hệ phải vận hành (🔎 verify) |
| **"Kafka không làm được queue (P2P chia 1 partition)"** | **KIP-932 Queues / share groups** (early access 4.x) bắt đầu cho phép | Vẫn **EA**, chưa thay RabbitMQ (🔎 verify) |
| **Tự build retry/delay trên Redis thủ công** | **BullMQ / Redis Streams** có sẵn (góc Node) → lên **Kafka** khi cần bền/replay lớn | cross-ref **J-QUEUE** |

---

## ④ Áp dụng thực tế + So sánh bigtech

### Tiêu chí chọn 📘 K-BRK-002

| Nhu cầu | Chọn | Vì sao (bám mô hình) |
|---|---|---|
| **replay** / lịch sử / nhiều consumer độc lập / **throughput** cực lớn / **CDC** / analytics | **Kafka** | log giữ lại + offset + nhiều consumer group |
| **routing** phức tạp / **per-message priority** / **task queue** / **RPC-style** | **RabbitMQ** | exchange mạnh + priority queue + DLX |
| pub/sub **siêu nhẹ**, latency thấp (**control plane, IoT**) | **NATS** (**JetStream** khi cần bền) | nhẹ, nhanh, ít vận hành |
| **managed**, ít vận hành trên AWS | **SQS** (queue) + **SNS** (fan-out) | AWS lo hạ tầng |
| **job queue** trong app Node, hạ tầng nhẹ (đã có **Redis**) | **BullMQ / Redis Streams**; vượt ngưỡng → **Kafka** | cross-ref **J-QUEUE-006** |

**Bigtech (pattern — 🔎 verify chi tiết):** **LinkedIn** sinh ra **Kafka** và chạy **event spine** quy mô khổng lồ; nhiều hệ AWS-native dùng **SNS→SQS** cho fan-out managed; hệ cần **routing nghiệp vụ phức tạp** hay dùng **RabbitMQ**; **NATS** phổ biến ở **control-plane/edge**.

---

## ⑤ Code thực hành + cấu hình

**RabbitMQ topic exchange (Node `amqplib`)** — minh hoạ *tách publish khỏi routing* + bật durability:

```javascript
// publisher.js — gửi vào EXCHANGE, KHÔNG gửi thẳng queue
const amqp = require('amqplib'); // pin version khi dùng thật, vd "amqplib@0.10.x" // verify
const conn = await amqp.connect(process.env.RABBITMQ_URL); // KHÔNG hardcode URL (đọc từ env)
const ch = await conn.createConfirmChannel();              // confirm channel = bật publisher confirms (durability)

await ch.assertExchange('orders', 'topic', { durable: true }); // exchange bền, sống qua restart broker

// publish với routing key 'order.vn.created' + persistent (ghi xuống disk)
ch.publish('orders', 'order.vn.created', Buffer.from(JSON.stringify(evt)), { persistent: true });

await ch.waitForConfirms(); // ⭐ CHỜ broker xác nhận đã giữ chắc -> nếu không, có thể mất message âm thầm
```

```javascript
// consumer.js — bind queue bằng WILDCARD, prefetch chống ngộp
await ch.assertQueue('vn-orders', { durable: true });
await ch.bindQueue('vn-orders', 'orders', 'order.vn.*'); // topic wildcard: bắt MỌI sự kiện đơn hàng vùng VN

ch.prefetch(20); // QoS: tối đa 20 message IN-FLIGHT cùng lúc (backpressure cho push-based)

ch.consume('vn-orders', async (msg) => {
  try {
    await handle(JSON.parse(msg.content));
    ch.ack(msg);                  // ✅ ack SAU KHI xử lý xong (chi tiết delivery guarantee ở Bài 5)
  } catch (e) {
    ch.nack(msg, false, false);   // ❌ nack -> đẩy sang DLQ (cấu hình dead-letter-exchange)
  }
});
```

**`requirements` / `package.json`** (ví dụ — 🔎 *verify đời mới nhất khi học*):

```
amqplib        // RabbitMQ client (verify)
kafkajs        // Kafka client Node (verify) — dùng ở Bài 3-4
// Hạ tầng: docker compose rabbitmq:3-management ; apache/kafka:4.x (KRaft) // verify tag
```

> 🎯 **Ba thứ "bật durability" phải nhớ trong đoạn code trên:** `createConfirmChannel` → `{ persistent: true }` → `waitForConfirms()`. Thiếu một trong ba, "đã publish" *không* đồng nghĩa "đã giữ".

---

## ⑥ Keywords

**🧠 Cần HIỂU:**
- **log** vs **queue** vs **pub/sub** — ba mô hình nền.
- **push** vs **pull** + **backpressure** — ai chủ động, ai tự bảo vệ.
- **durability** = **replicate** + **ack** (persist disk + nhiều bản sao + producer biết chắc).
- **broker là SPOF → HA** (cluster, replication, partition, retry, Outbox, giám sát lag).
- **schema evolution** — event là contract, dùng registry + compatibility.

**📌 Cần THUỘC:**
- **4 loại exchange RabbitMQ**: **fanout / direct / topic / headers**.
- **SQS standard** vs **SQS FIFO**; **SNS → SQS** fan-out.
- **tiêu chí chọn** Kafka / RabbitMQ / NATS / SQS.

**🛠️ Cần LÀM ĐƯỢC:**
- Chọn broker cho 1 bài toán + **biện minh** bám mô hình.
- Cấu hình **publisher confirms** + **prefetch** + **DLQ** cơ bản.

---

## ⑦ Mental model

> 🧠 **"Kafka = băng ghi tua lại được** (**log**, **offset**, **replay**); **RabbitMQ = quầy phát thư với bộ định tuyến mạnh** (**exchange**, xoá sau **ack**).
> **Chọn broker = chọn mô hình, KHÔNG chọn thương hiệu."**

---

## ⑧ Phỏng vấn & thách đố

**(dễ)** **Kafka vs RabbitMQ** khác nhau ở lõi chỗ nào?
→ **log + offset + replay** vs **queue + ack + xoá**.

**(TB)** **Push vs pull**, hại gì với consumer chậm?
→ **push** dễ **ngộp** (cần **prefetch**); **pull** **tự backpressure**.

**(TB)** 4 **exchange RabbitMQ** + **routing key** dùng để làm gì?
→ fanout/direct/topic/headers; routing key là "địa chỉ" để exchange định tuyến.

**(khó)** Khi nào **Kafka**, khi nào **RabbitMQ**, khi nào **SQS/SNS**, khi nào **NATS**?
→ bám bảng tiêu chí ④ (streaming/replay → Kafka; routing/priority → RabbitMQ; managed AWS → SQS/SNS; siêu nhẹ → NATS).

**(khó)** Broker đảm bảo **durability** bằng gì? **`acks=all`** + **replicate** đóng vai trò gì?
→ persist disk + nhiều bản sao (ISR) + producer ack; `acks=all` ép chờ các bản sao xác nhận trước khi coi là thành công.

> 🧩 **Thách đố:** *"Team chọn **Kafka** vì 'ai cũng dùng', nhưng nhu cầu là gửi **job render video** có **priority** + **retry/delay** + vài nghìn job/ngày. Sai ở đâu?"*
> → Kafka **không có per-message priority**; **delay/retry** phải tự dựng (**retry topic**); **partition** là đơn vị song song *thô*. Nhu cầu này hợp **RabbitMQ** (**priority queue**, **DLX**) hoặc **BullMQ** (**delay/retry** sẵn). Throughput *vài nghìn/ngày* **không cần** Kafka — đó là over-engineering.

---

## ⑨ Bài tập + tiêu chí

**Đề:** Cho 3 nhu cầu, chọn broker cho mỗi cái + 1 câu lý do:
- (a) đồng bộ **DB → search index** realtime;
- (b) gửi **SMS** có **retry** + **ưu tiên SMS OTP**;
- (c) phát **`PriceChanged`** cho 5 service.

> ✅ **Đạt khi:**
> - (a) **Kafka** + **CDC** (replay, nhiều consumer độc lập).
> - (b) **RabbitMQ / BullMQ** (**priority** + **retry/DLQ**).
> - (c) **Kafka** *hoặc* **SNS → SQS** (**fan-out**, mỗi consumer có **buffer riêng**).
> - **Lý do bám đúng mô hình**, KHÔNG "vì quen".

---

## ⑩ Đọc thêm

- **Docs chính thức:** `kafka.apache.org`, `rabbitmq.com`, `docs.nats.io`, **AWS SQS/SNS Developer Guide**. (🔎 *verify config/giới hạn tại đây*.)
- **Confluent:** *"Kafka vs RabbitMQ"* — đọc **phê phán**, lưu ý **thiên vị vendor**.

> 🎯 **Một câu mang về:** *Trước khi gõ tên broker, hãy nói được nhu cầu của bạn là **log**, **queue** hay **pub/sub** — chọn broker chỉ là hệ quả của câu trả lời đó.*
