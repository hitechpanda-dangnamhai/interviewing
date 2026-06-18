# 📚 Mục K — Messaging & Microservices · Giáo trình đầy đủ để học

> **Sinh theo WORKFLOW 2 — Bước B (dựng giáo trình ngược từ bộ đề) + Bước C (giảng từng bài theo Hợp đồng 10 mục).**
> Nguồn câu hỏi: `QB_K_messaging.md` (94 câu, 11 mục con). Đây là **tài liệu học để lưu**, không phải phiên phỏng vấn — phần phỏng vấn live là Bước D, làm sau.
> **Bối cảnh kiến thức: tính tại 6/2026.** Mọi mục có *(verify)* là thứ đổi nhanh (config/đời broker, tên tool) → kiểm chứng tại docs chính thức khi thực hành.
> Quy ước nguồn: **📘** = có trong roadmap/QB gốc · **➕** = bổ sung của giảng viên · **⬆️** = làm sâu hơn.

---

## 🔎 Bối cảnh hiện hành đã verify (6/2026) — đọc trước

Những mốc này được dùng xuyên suốt tài liệu (đã search + verify; vẫn nên tự kiểm chứng vì đổi nhanh):

- **Apache Kafka 4.x** (4.0 ra 3/2025, dòng 4.2.x hiện hành giữa 2026): **KRaft-only — ZooKeeper đã bị gỡ hoàn toàn**. Metadata nằm trong topic nội bộ `__cluster_metadata` do nhóm controller (Raft quorum) quản. *(verify: kafka.apache.org)*
- **KIP-848 (cooperative/incremental rebalance) đã GA** trong 4.0 → giảm mạnh "stop-the-world" rebalance; consumer opt-in bằng `group.protocol=consumer`. **Nhớ:** câu phỏng vấn cũ "rebalance là stop-the-world" giờ phải nói thêm "đang dịch sang incremental".
- **KIP-932 Queues for Kafka (share groups)**: early access — Kafka bắt đầu hỗ trợ ngữ nghĩa "queue" (nhiều consumer chia 1 partition). Vẫn EA, đừng coi là chuẩn production. *(verify)*
- **KIP-890** (transactions hardening) củng cố exactly-once trong Kafka 4.x.
- **Debezium 3.4.x** (12/2025): hỗ trợ Kafka 4.x; **exactly-once cho mọi core connector** từ 3.3; có Debezium Server/Engine (chạy CDC không cần Kafka Connect). *(verify: debezium.io)*
- **Saga orchestration tooling**: **Temporal** (durable execution, code-first) và **Camunda 8 / Zeebe** (BPMN, visual) là 2 lựa chọn chủ đạo; thêm Orkes Conductor. "2025–2026 microservices thiên về *workflow-orchestrated* thay vì REST-orchestrated thuần". *(verify)*
- **Service mesh — kỷ nguyên sidecar đang kết thúc**: **Istio ambient mode GA (2025)** dùng `ztunnel` (per-node L4 mTLS) + `waypoint` (per-namespace L7 Envoy) thay sidecar; **Cilium** chạy mesh trong kernel qua eBPF; **Linkerd** giữ sidecar Rust nhẹ (lưu ý: Buoyant đổi mô hình license — stable qua subscription). *(verify)*
- **OpenTelemetry (OTel)** là chuẩn trung lập (CNCF) cho trace/metric/log; **W3C Trace Context** (`traceparent` header) là chuẩn truyền context; backend trace: Jaeger/Tempo/Zipkin/Datadog. *(verify)*

> 🧭 **Câu thần chú của mục K:** *AI viết đúng cú pháp publish/consume nhưng dễ **sai kiến trúc** — publish event trong transaction code (mất event), chọn 2PC thay vì Saga, quên consumer phải idempotent dưới at-least-once. Bắt những chỗ đó là việc của Tech Lead.*

---

# 🧱 BƯỚC B — Giáo trình ngược dựng từ 94 câu hỏi

## Nguyên tắc sắp xếp: **dependency order**, không theo tần suất hỏi

Chuỗi phụ thuộc khái niệm: hiểu *vì sao async* trước → biết *công cụ* (broker) → hiểu *cơ chế Kafka* làm nền cho mọi câu delivery → *delivery semantics & idempotent* là gốc của mọi câu reliability → từ đó mới hiểu được *Outbox* (vá dual-write) → *Saga* (dùng Outbox + idempotent làm baseline) → *Event Sourcing/CQRS* (họ hàng event-driven) → *Resilience* (bảo vệ các call sync còn lại) → *Decomposition* (ranh giới service sinh ra mọi bài trên) → *Gateway/Mesh* (topology nối các service) → *Observability* (nhìn xuyên toàn hệ). Decomposition đặt **sau** phần cơ chế một cách cố ý: để khi bàn "tách service" bạn đã có đủ vũ khí (Saga/Outbox/event) trả lời "tách xong thì đồng bộ dữ liệu kiểu gì".

## Mục lục bài học → ID câu hỏi nó phủ

| Bài | Tên bài | Mục con | ID câu phủ | Số câu |
|---|---|---|---|---|
| **1** | Vì sao Messaging: sync vs async & coupling | K-WHY | K-WHY-001 → 008 | 8 |
| **2** | Broker Landscape & chọn broker | K-BRK | K-BRK-001 → 009 | 9 |
| **3** | Kafka Internals I — partition, consumer group, offset | K-KAFKA | K-KAFKA-001 → 005 | 5 |
| **4** | Kafka Internals II — replication, rebalance, retention, EOS, lag | K-KAFKA | K-KAFKA-006 → 012 | 7 |
| **5** | Delivery Semantics & Idempotent Consumer | K-DELIV | K-DELIV-001 → 009 | 9 |
| **6** | Dual-write, Transactional Outbox & CDC | K-OBX | K-OBX-001 → 008 | 8 |
| **7** | Saga — orchestration vs choreography & compensation | K-SAGA | K-SAGA-001 → 009 | 9 |
| **8** | Event Sourcing & CQRS | K-ES | K-ES-001 → 007 | 7 |
| **9** | Resilience — circuit breaker, bulkhead, retry, timeout | K-RESIL | K-RESIL-001 → 009 | 9 |
| **10** | Service Decomposition & bounded context | K-DECOMP | K-DECOMP-001 → 009 | 9 |
| **11** | API Gateway, Service Mesh & Service Discovery | K-GW | K-GW-001 → 008 | 8 |
| **12** | Distributed Tracing & Observability | K-OBS | K-OBS-001 → 006 | 6 |

**Tổng: 12 bài / 94 câu.** Mỗi bài dưới đây theo **Hợp đồng đầu ra 10 mục**.

---

# 🎓 BƯỚC C — Các bài giảng

---

## Bài 1 — Vì sao Messaging: sync vs async & coupling
*Phủ: K-WHY-001 → 008*

### ① Mục tiêu & vị trí trong mạch
Bài mở đầu Mục K. Học xong bạn trả lời được câu gốc nhất: **"thêm một message broker vào hệ thống để giải bài toán gì, và trả giá gì?"**. Đây là nền cho mọi bài sau — nếu không vững "vì sao async", bạn sẽ nhét queue vào mọi chỗ (anti-pattern) hoặc ngược lại sợ async mà cố giữ mọi thứ sync (cascade failure). Nối tới: Bài 2 (chọn broker nào) và Bài 9 (async không miễn trừ resilience).

### ② Giảng cơ bản → nâng cao

**(a) Trực giác.** Sync call giống **gọi điện thoại**: hai bên phải cùng online, bạn nói xong đứng chờ máy bên kia trả lời mới làm tiếp. Async qua broker giống **gửi thư qua bưu điện**: bạn bỏ thư vào hòm (broker), đi làm việc khác; người nhận lấy thư khi rảnh; nếu họ đi vắng, thư vẫn nằm đó. Bưu điện = broker, nó *tách thời gian* giữa gửi và nhận.

**(b) Cơ chế chi tiết — 5 lớp bài toán message queue giải (📘 K-WHY-001):**
1. **Decoupling** — producer không cần biết consumer là ai, ở đâu, có bao nhiêu; chỉ cần biết "topic/queue". Đổi/thêm consumer không đụng producer.
2. **Async / giảm latency cảm nhận** — trả response cho user ngay sau khi *nhận* yêu cầu, xử lý nặng để worker làm sau (vd gửi email, render video).
3. **Buffering / load-leveling** — broker hấp thụ spike: 10.000 req/s ập tới, downstream xử lý 1.000/s → queue giữ phần dư, downstream rút theo nhịp của nó (shock absorber).
4. **Độ bền (durability)** — message persist; consumer chết giữa chừng, message không mất, xử lý lại khi sống dậy.
5. **Fan-out** — 1 event → N consumer độc lập (order-created → kho, email, analytics, audit), thêm consumer mà không sửa producer.

**Coupling — phân loại (📘 K-WHY-003):**
- **Temporal coupling** = phải cùng online *một lúc*. Sync có; async **gỡ được** (consumer offline vẫn nhận sau).
- **Spatial coupling** = phải biết *địa chỉ* nhau. Async gỡ *một phần* (chỉ cần biết topic, không cần IP/instance).
- ⚠️ Async **tạo coupling mới**: vào **schema của event** và vào **chính broker**. Không có bữa trưa miễn phí — bạn đổi coupling thời-gian/không-gian lấy coupling-vào-contract.

**Command vs Event (📘 K-WHY-004) — phân biệt sống còn:**

| | Command | Event |
|---|---|---|
| Ngữ nghĩa | "Hãy làm X" (ra lệnh) | "X đã xảy ra" (thông báo) |
| Người nhận | **một** owner cụ thể | **bất kỳ ai** quan tâm (fan-out) |
| Đặt tên | mệnh lệnh: `ChargePayment` | quá khứ: `PaymentCharged` |
| Coupling | producer biết ai xử lý | producer **không** quan tâm ai xử lý |
| Mong đợi | side-effect, có thể trả kết quả | không mong đợi gì cụ thể |

Hệ quả thiết kế: command kéo coupling cao hơn (biết owner) nhưng kiểm soát tốt; event decouple cao nhưng "không ai chịu trách nhiệm rõ ràng" → logic phân tán.

**Point-to-point vs pub/sub (📘 K-WHY-006):** P2P = mỗi message tới **đúng một** consumer (work queue — chia tải job render). Pub/sub = mỗi message tới **mọi** subscriber (broadcast — `OrderPlaced` cho 4 phòng ban). Nhận diện nhu cầu: *"chia việc"* → P2P; *"báo nhiều bên"* → pub/sub.

**(c) Mép giới hạn & sai lầm:**
- **"Cho mọi thứ qua queue" (📘 K-WHY-005)** — anti-pattern. Async chỉ đáng khi có lý do trong 5 lớp trên. Khi user cần **read-after-write tức thì** (đặt hàng xong xem ngay đơn), nhét async vào = thêm latency, hạ tầng, eventual consistency, debug khổ. Default phải là **sync**; async là quyết định có chủ đích.
- **Async đổi mô hình nhất quán (📘 K-WHY-007)** — async-hoá một bước (gửi email khi đặt hàng) biến phần đó từ strong → **eventual consistency**. User thấy "Đặt hàng OK" *trước khi* email thật sự gửi. Phải thiết kế **trạng thái trung gian** (`PENDING`/`PROCESSING`) + retry + cơ chế báo lỗi muộn. Sai lầm chết người: coi *"publish xong = đã xong"*.

### ③ ⚠️ Kiến thức cũ / bị thay thế

| Cái cũ (phổ biến) | Thay bằng (hiện nay) | Vì sao |
|---|---|---|
| "Microservice = phải gọi nhau qua HTTP REST" | Sync chỉ cho path cần phản hồi ngay; **event-driven** cho phần còn lại | REST-dây-chuyền = distributed monolith + cascade (Bài 9, 10) |
| "Queue chỉ để chạy nhanh hơn" | Queue = decouple + buffer + durability + fan-out (tốc độ chỉ là 1 trong 5) | Hiểu sai mục đích → đặt sai chỗ |
| "Publish event = việc đã hoàn tất" | Publish = "đã ghi nhận", trạng thái thực vẫn `PENDING` tới khi consumer xong | Eventual consistency (K-WHY-007) |

### ④ Áp dụng thực tế + So sánh bigtech
**Use case thật:** đặt hàng e-commerce. Sync: tạo `Order` (cần phản hồi ngay). Async: trừ kho, tính phí, gửi mail xác nhận, cập nhật analytics → publish `OrderPlaced`, để các service tự xử lý → user không phải chờ email SMTP.

**Kiến trúc tối thiểu:** `API → tạo Order (DB) → publish OrderPlaced → [InventorySvc, EmailSvc, AnalyticsSvc] consume độc lập`.

**Bigtech (pattern phổ biến, không khẳng định tuyệt đối — đổi nhanh):** các hệ quy mô lớn (thương mại điện tử, ride-hailing, fintech) đều chạy "spine" event-driven (thường trên Kafka hoặc broker tương đương) cho fan-out + analytics + CDC, giữ sync cho các call cần độ tươi. Netflix/Uber nổi tiếng với kiến trúc event-driven nội bộ. *(verify chi tiết khi cần con số/tên hệ thống cụ thể.)*

### ⑤ Code thực hành + cấu hình
Minh hoạ "trả response ngay, xử lý sau" bằng pseudo-Node (chưa cần broker thật — Bài 2 chọn broker):

```javascript
// orderController.js — KHÔNG chờ email/inventory đồng bộ
// AI hay viết: gọi sync emailService.send() ngay trong handler → user chờ SMTP.
// Đúng kiến trúc: ghi DB, publish event, trả 202/201 ngay.
async function placeOrder(req, res) {
  const order = await db.orders.insert({ ...req.body, status: 'PENDING' }); // trạng thái trung gian
  await broker.publish('OrderPlaced', { orderId: order.id, ...order });      // fire-and-forget (xem Bài 6: vì sao publish ở đây CHƯA an toàn — dual-write!)
  res.status(201).json({ orderId: order.id, status: 'PENDING' });           // user thấy PENDING, không chờ email
}
```

> ⚠️ **Bẫy đã gài sẵn:** dòng `publish` ngay sau `insert` chính là **dual-write problem** — Bài 6 sẽ vá bằng Outbox. Để đây để bạn thấy "code chạy được nhưng sai kiến trúc".

Cấu hình ở bài này chưa cần broker; chỉ cần quy ước trạng thái: `PENDING → CONFIRMED → ...` trong schema đơn.

### ⑥ Keywords cần nhớ
- 🧠 **Cần HIỂU:** decoupling, temporal vs spatial coupling, load-leveling/buffering, fan-out, eventual consistency, "khi nào KHÔNG dùng queue".
- 📌 **Cần THUỘC:** 5 lớp bài toán MQ; command (mệnh lệnh, 1 owner) vs event (quá khứ, fan-out); P2P vs pub/sub.
- 🛠️ **Cần LÀM ĐƯỢC:** nhìn một flow → quyết sync hay async + nêu lý do; thiết kế trạng thái trung gian khi async-hoá.

### ⑦ Mental model
**"Sync = gọi điện (cùng online, chờ máy); async = gửi thư (broker giữ giùm). Queue không phải để nhanh — để decouple/buffer/bền/fan-out. Async-hoá = đổi strong lấy eventual; phải có trạng thái PENDING."**

### ⑧ Câu hỏi phỏng vấn & thách đố
1. *(dễ)* Kể ≥3 bài toán message queue giải. → Gợi ý: decouple, async, buffer, durability, fan-out.
2. *(TB)* Sync vs async đánh đổi gì, chọn theo tiêu chí nào? → cần phản hồi tức thì? chịu eventual? có spike?
3. *(khó)* Temporal vs spatial coupling — async gỡ loại nào, *tạo* coupling gì mới? → gỡ temporal + một phần spatial; tạo coupling vào schema event + broker.
4. *(khó)* Command vs event khác gì về coupling và ai sở hữu logic?
5. *(rất khó)* Async-hoá "gửi email khi đặt hàng" — hệ thống nhất quán đổi thế nào, user thấy gì? → strong→eventual, thấy PENDING trước khi email xong, cần retry/báo lỗi muộn.
- 🎯 **Thách đố:** *"Sếp bảo: cứ cho tất cả call nội bộ qua Kafka cho 'scalable'. Bạn phản biện sao?"* → Đáp: async thêm latency + eventual + hạ tầng + debug khó; call cần read-after-write/phản hồi ngay nên giữ sync; queue chỉ cho chỗ có lý do (decouple/buffer/fan-out/bền). "Scalable" không phải lý do tự thân.

### ⑨ Bài tập + tiêu chí tự chấm
Cho luồng "đăng ký user → gửi email verify → tặng coupon chào mừng". Vẽ phần nào sync, phần nào async, đặt tên event (quá khứ), và chỉ trạng thái trung gian.
- **Đạt khi:** tạo user là sync (cần userId ngay); gửi email + coupon là async qua `UserRegistered`; có trạng thái `email_verified=false`; nêu được rủi ro nếu coi "publish = đã gửi".

### ⑩ Đọc thêm
- *Enterprise Integration Patterns* (Hohpe & Woolf) — kinh điển về messaging patterns.
- Microsoft Learn / AWS Prescriptive Guidance: "messaging patterns", "choreography vs orchestration" (khái niệm nền, ổn định).

---

## Bài 2 — Broker Landscape & chọn broker
*Phủ: K-BRK-001 → 009*

### ① Mục tiêu & vị trí trong mạch
Bài 1 trả lời "vì sao async". Bài này trả lời **"async bằng cái gì"** — phân biệt mô hình **log vs queue vs pub/sub**, so sánh Kafka/RabbitMQ/NATS/SQS-SNS/Redis Streams, và **chọn đúng broker** theo bài toán. Là cầu nối tới Bài 3–4 (đào sâu Kafka — broker thống trị event-streaming). Cross-ref J-QUEUE (BullMQ/Redis góc Node).

### ② Giảng cơ bản → nâng cao

**(a) Trực giác — log vs queue (📘 K-BRK-001):**
- **Queue truyền thống (RabbitMQ)** = **hàng đợi ở quầy**: message tới, consumer lấy, broker **xoá sau khi ack**. Đọc xong là mất. Routing linh hoạt (chọn quầy nào).
- **Log (Kafka)** = **băng ghi append-only**: message ghi nối đuôi, **giữ theo retention** (vd 7 ngày), consumer tự nhớ mình đọc tới đâu (**offset**). Nhiều nhóm đọc **độc lập**, tua lại (replay) được.

| | Kafka (log) | RabbitMQ (queue/broker) |
|---|---|---|
| Lưu trữ | append-only log, giữ theo retention | đẩy tới consumer, **xoá sau ack** |
| Vị trí đọc | consumer giữ **offset**, replay được | broker quản, không replay mặc định |
| Nhiều consumer | nhiều **consumer group** đọc độc lập cùng topic | cần exchange/binding để fan-out |
| Routing | đơn giản (topic + partition theo key) | **mạnh** (direct/topic/fanout/headers exchange) |
| Mạnh ở | throughput rất lớn, streaming, replay, CDC | routing phức tạp, per-message ack/priority, task queue |

**(b) Cơ chế — RabbitMQ exchange (📘 K-BRK-004):** producer **không** gửi thẳng vào queue mà gửi vào **exchange**; exchange theo **type** + **binding** định tuyến (qua *routing key*) vào ≥0 queue:
- `fanout` = broadcast tất cả queue đã bind (bỏ qua routing key).
- `direct` = routing key khớp **chính xác** binding key.
- `topic` = khớp **wildcard** (`order.*.created`).
- `headers` = khớp theo header thay vì routing key.
Tách publish khỏi routing là điểm mạnh của RabbitMQ.

**Push vs pull (📘 K-BRK-003):**
- **Push (RabbitMQ)** — broker đẩy message tới consumer: latency thấp, nhưng dễ làm **ngộp** consumer chậm → cần `prefetch`/QoS limit để giới hạn message in-flight.
- **Pull (Kafka)** — consumer **tự kéo** theo nhịp của nó: tự **backpressure**, dễ batch, đổi lại thêm một nhịp poll. Consumer chậm tự bảo vệ mình.

**AWS SQS/SNS (📘 K-BRK-005):**
- **SQS standard** = at-least-once + best-effort ordering + throughput rất cao.
- **SQS FIFO** = đúng thứ tự + dedup (theo *message group id* / *deduplication id*) nhưng **throughput giới hạn** (verify giới hạn TPS hiện tại).
- **SNS** = pub/sub fan-out tới nhiều subscriber (SQS, Lambda, HTTP...).
- **Pattern kinh điển: SNS → nhiều SQS** = fan-out nhưng mỗi consumer có **buffer riêng** (1 consumer chậm không ảnh hưởng consumer khác).

**(c) Mép giới hạn:**
- **Durability (📘 K-BRK-007):** broker bền = persist ra disk + **replicate** sang nhiều node trước khi coi "đã nhận", + **producer ack** để biết broker thật sự giữ (Kafka `acks=all` chờ ISR; RabbitMQ *publisher confirms*). Thiếu các cơ chế này → message bay khi node chết. *(verify config — Bài 4.)*
- **Broker là SPOF/bottleneck? (📘 K-BRK-008):** có thể. HA cần: cluster + **replication factor ≥3** (Kafka) / quorum queue (RabbitMQ), partition/shard để scale ngang, client **retry + reconnect**, và **Outbox phía producer** (Bài 6) để không mất event khi broker tạm sập, + giám sát **lag/disk**.
- **Schema tiến hoá (📘 K-BRK-009):** event là "API contract". Dùng **schema registry** + versioning + quy tắc compatibility (backward/forward): thêm field **optional**, không đổi nghĩa field cũ, consumer **bỏ qua field lạ**, đổi lớn thì tách **version** event. *(verify: Avro/Protobuf/JSON Schema + Confluent/Apicurio registry.)*

### ③ ⚠️ Cũ → Mới

| Cái cũ | Nên dùng nay | Ghi chú |
|---|---|---|
| "Kafka thay được RabbitMQ cho mọi thứ" | Chọn theo nhu cầu: Kafka cho streaming/replay/throughput; RabbitMQ cho routing/task queue | Hai mô hình khác nhau (log vs queue) |
| Kafka + ZooKeeper | **Kafka KRaft-only** (4.x, không còn ZooKeeper) | Đỡ một hệ phải vận hành *(verify)* |
| "Kafka không làm được queue (P2P chia 1 partition)" | **KIP-932 Queues/share groups** (early access 4.x) bắt đầu cho phép | Vẫn EA, chưa thay RabbitMQ *(verify)* |
| Tự build retry/delay trên Redis thủ công | BullMQ/Redis Streams có sẵn (góc Node) → lên Kafka khi cần bền/replay lớn | cross-ref J-QUEUE |

### ④ Áp dụng thực tế + So sánh bigtech
**Tiêu chí chọn (📘 K-BRK-002):**
- Cần **replay / lịch sử / nhiều consumer độc lập / throughput cực lớn / CDC / analytics** → **Kafka**.
- Cần **routing phức tạp / per-message priority / task queue / RPC-style** → **RabbitMQ**.
- Cần **pub/sub siêu nhẹ, latency thấp** (control plane, IoT) → **NATS** (JetStream khi cần bền).
- Muốn **managed, ít vận hành trên AWS** → **SQS** (queue) + **SNS** (fan-out).
- Job queue trong app Node, hạ tầng nhẹ (đã có Redis) → **BullMQ/Redis Streams**; vượt ngưỡng → Kafka *(cross-ref J-QUEUE-006)*.

**Bigtech (pattern, verify chi tiết):** LinkedIn sinh ra Kafka và chạy event spine quy mô khổng lồ; nhiều hệ AWS-native dùng SNS→SQS cho fan-out managed; hệ cần routing nghiệp vụ phức tạp hay dùng RabbitMQ. NATS phổ biến ở control-plane/edge.

### ⑤ Code thực hành + cấu hình
RabbitMQ topic exchange (Node `amqplib`) — minh hoạ tách publish/routing:

```javascript
// publisher.js — gửi vào EXCHANGE, không gửi thẳng queue
const amqp = require('amqplib'); // pin version khi dùng thật, vd "amqplib@0.10.x" // verify
const conn = await amqp.connect(process.env.RABBITMQ_URL); // KHÔNG hardcode URL
const ch = await conn.createConfirmChannel();              // confirm channel = publisher confirms (durability)
await ch.assertExchange('orders', 'topic', { durable: true });
ch.publish('orders', 'order.vn.created', Buffer.from(JSON.stringify(evt)), { persistent: true }); // persistent = ghi disk
await ch.waitForConfirms(); // chờ broker xác nhận đã giữ -> tránh mất message
```

```javascript
// consumer.js — bind queue với wildcard, prefetch chống ngộp
await ch.assertQueue('vn-orders', { durable: true });
await ch.bindQueue('vn-orders', 'orders', 'order.vn.*'); // topic wildcard
ch.prefetch(20); // QoS: tối đa 20 message in-flight (backpressure cho push-based)
ch.consume('vn-orders', async (msg) => {
  try { await handle(JSON.parse(msg.content)); ch.ack(msg); }   // ack SAU khi xử lý xong (Bài 5)
  catch (e) { ch.nack(msg, false, false); }                     // nack -> DLQ (cấu hình dead-letter-exchange)
});
```

`requirements`/`package.json` (ví dụ — **verify đời mới nhất khi học**):
```
amqplib        // RabbitMQ client (verify)
kafkajs        // Kafka client Node (verify) — dùng ở Bài 3-4
// Hạ tầng: docker compose rabbitmq:3-management ; apache/kafka:4.x (KRaft) // verify tag
```

### ⑥ Keywords
- 🧠 **Cần HIỂU:** log vs queue vs pub/sub; push vs pull + backpressure; durability (replicate+ack); broker là SPOF → HA; schema evolution.
- 📌 **Cần THUỘC:** 4 loại exchange RabbitMQ; SQS standard vs FIFO; SNS→SQS fan-out; tiêu chí chọn Kafka/RabbitMQ/NATS/SQS.
- 🛠️ **Cần LÀM ĐƯỢC:** chọn broker cho 1 bài toán + biện minh; cấu hình publisher confirms + prefetch + DLQ cơ bản.

### ⑦ Mental model
**"Kafka = băng ghi tua lại được (log, offset, replay); RabbitMQ = quầy phát thư với bộ định tuyến mạnh (exchange, xoá sau ack). Chọn broker = chọn mô hình, không chọn thương hiệu."**

### ⑧ Phỏng vấn & thách đố
1. *(dễ)* Kafka vs RabbitMQ khác nhau lõi ở đâu? → log+offset+replay vs queue+ack+xoá.
2. *(TB)* Push vs pull, hại gì với consumer chậm? → push ngộp (cần prefetch); pull tự backpressure.
3. *(TB)* 4 exchange RabbitMQ + routing key để làm gì?
4. *(khó)* Khi nào Kafka, khi nào RabbitMQ, khi nào SQS/SNS, khi nào NATS?
5. *(khó)* Broker đảm bảo durability bằng gì? `acks=all` + replicate đóng vai trò gì?
- 🎯 **Thách đố:** *"Team chọn Kafka vì 'ai cũng dùng', nhưng nhu cầu là gửi job render video có priority + retry/delay + vài nghìn job/ngày. Sai ở đâu?"* → Kafka không có per-message priority, delay/retry phải tự dựng (retry topic), partition là đơn vị song song thô; nhu cầu này hợp **RabbitMQ** (priority queue, DLX) hoặc **BullMQ** (delay/retry sẵn). Throughput vài nghìn/ngày không cần Kafka.

### ⑨ Bài tập + tiêu chí
Cho 3 nhu cầu: (a) đồng bộ DB → search index realtime; (b) gửi SMS có retry + ưu tiên SMS OTP; (c) phát `PriceChanged` cho 5 service. Chọn broker mỗi cái + 1 câu lý do.
- **Đạt khi:** (a) Kafka + CDC (replay, nhiều consumer) ; (b) RabbitMQ/BullMQ (priority + retry/DLQ) ; (c) Kafka hoặc SNS→SQS (fan-out, mỗi consumer buffer riêng). Lý do bám đúng mô hình, không "vì quen".

### ⑩ Đọc thêm
- Docs chính thức: kafka.apache.org, rabbitmq.com, docs.nats.io, AWS SQS/SNS Developer Guide. *(verify config/giới hạn tại đây.)*
- Confluent: "Kafka vs RabbitMQ" (đọc phê phán, lưu ý thiên vị vendor).

---

## Bài 3 — Kafka Internals I: partition, consumer group, offset
*Phủ: K-KAFKA-001 → 005*

### ① Mục tiêu & vị trí trong mạch
Kafka là broker thống trị event-streaming → hai bài Kafka (3 & 4) là **trục xương sống** của cả Mục K: mọi câu về delivery, ordering, Outbox, Saga đều dựa trên cơ chế ở đây. Bài 3 dựng nền **partition/offset/consumer group** — sau bài này bạn hết coi Kafka là hộp đen. Bài 4 sẽ thêm replication/rebalance/EOS.

### ② Giảng cơ bản → nâng cao

**(a) Trực giác.** Hình dung **topic** = một chủ đề (vd `orders`). Nó được chia thành nhiều **partition** = nhiều băng ghi append-only song song. Mỗi message rơi vào *một* partition và nhận một **offset** = số thứ tự tăng dần *trong partition đó*. Nhiều **broker** (server) giữ các partition; tập broker = **cluster**.

**(b) Cơ chế (📘 K-KAFKA-001 → 005):**
- **Offset không duy nhất across partition** — offset 5 của partition 0 khác message offset 5 của partition 1. Duy nhất chỉ *trong* một partition.
- **Partition vừa là đơn vị song song vừa là đơn vị ordering (📘 K-KAFKA-002):** Kafka **chỉ đảm bảo thứ tự trong một partition**, KHÔNG across partition. Vì thế:
  - Số partition = **trần song song** của một consumer group (xem dưới).
  - Tăng partition → tăng throughput **nhưng** phá thứ tự global + nhiều file/overhead + có thể **đổi mapping key→partition** (vì `hash(key) % N` đổi khi N đổi). Đây là trade-off **ordering ↔ throughput** kinh điển.
- **Consumer group (📘 K-KAFKA-003):** một nhóm consumer chia nhau các partition để xử lý song song. Quy tắc vàng: **mỗi partition gán cho đúng 1 consumer trong group**. Hệ quả:
  - Nếu `#consumer > #partition` → consumer thừa **ngồi không** (idle). Muốn scale tiêu thụ → phải **tăng partition**, không chỉ thêm consumer.
  - Nhiều **group khác nhau** đọc **độc lập** cùng topic (đây chính là pub/sub: mỗi group có offset riêng).
- **Producer chọn partition (📘 K-KAFKA-004):**
  - Có **key** → `hash(key) % #partition` → **cùng key luôn vào cùng partition** → giữ ordering theo thực thể (vd key = `orderId` ⇒ mọi event của một order cùng partition, xử lý đúng thứ tự).
  - Không key → phân bổ round-robin/sticky. *(verify default partitioner — đã đổi qua các đời.)*
  - ⇒ **Key là công cụ giữ ordering per-entity.** Đây là câu trả lời cho "làm sao event của cùng order đúng thứ tự" (gặp lại ở Bài 5).
- **Offset commit (📘 K-KAFKA-005)** — *điểm AI hay sai*:
  - **Commit TRƯỚC khi xử lý xong** → crash giữa chừng = **mất message** (**at-most-once**).
  - **Commit SAU khi xử lý xong** → crash trước khi commit = **xử lý lại** (**at-least-once** → cần idempotent).
  - **An toàn mặc định:** *process rồi mới commit* (manual commit). Offset lưu ở topic nội bộ `__consumer_offsets`. *(verify.)*

**(c) Mép giới hạn:** auto-commit (`enable.auto.commit=true`) commit theo chu kỳ thời gian, **không** theo "đã xử lý xong" → dễ rơi vào at-most-once *âm thầm* nếu xử lý bất đồng bộ. Hiểu: commit/ack nghĩa là *"tôi đã xử lý xong"*, không phải *"tôi đã nhận"*.

### ③ ⚠️ Cũ → Mới

| Cái cũ | Nay | Vì sao |
|---|---|---|
| ZooKeeper giữ metadata/offset cũ | **KRaft** (4.x); offset consumer ở `__consumer_offsets` | Bớt một hệ vận hành *(verify)* |
| "Thêm consumer là scale được" | Trần = #partition; thừa consumer ngồi không → **tăng partition** | Quy tắc 1 partition–1 consumer/group |
| `enable.auto.commit=true` cho tiện | Manual commit, **process-then-commit** | Auto-commit dễ at-most-once ngầm |

### ④ Áp dụng thực tế + bigtech
**Use case:** xử lý event order theo đúng thứ tự *per order* nhưng vẫn song song giữa các order khác nhau → key = `orderId`. Mỗi order vào 1 partition (đúng thứ tự), nhiều order trải đều nhiều partition (song song). **Bigtech:** đây là cách chuẩn để vừa giữ per-entity ordering vừa scale — dùng rộng rãi trong xử lý giao dịch/ledger. *(pattern phổ biến.)*

### ⑤ Code thực hành + cấu hình
KafkaJS — producer có key + consumer commit thủ công:

```javascript
// producer.js — key = orderId để giữ ordering per-order
const { Kafka } = require('kafkajs'); // pin "kafkajs@2.x" // verify đời mới
const kafka = new Kafka({ clientId: 'orders', brokers: (process.env.KAFKA_BROKERS||'').split(',') });
const producer = kafka.producer();
await producer.connect();
await producer.send({
  topic: 'orders',
  messages: [{ key: order.id, value: JSON.stringify(evt) }], // CÙNG key -> CÙNG partition -> đúng thứ tự
});
```

```javascript
// consumer.js — process THEN commit (at-least-once an toàn)
const consumer = kafka.consumer({ groupId: 'inventory-svc' }); // group riêng = đọc độc lập
await consumer.connect();
await consumer.subscribe({ topic: 'orders', fromBeginning: false });
await consumer.run({
  autoCommit: false, // tắt auto-commit để tự kiểm soát
  eachMessage: async ({ topic, partition, message }) => {
    await handle(JSON.parse(message.value)); // 1) xử lý + ghi side-effect (phải idempotent - Bài 5)
    await consumer.commitOffsets([          // 2) chỉ commit SAU khi xong
      { topic, partition, offset: (Number(message.offset) + 1).toString() },
    ]);
  },
});
```

```
# config broker/topic cần verify khi học (đổi theo đời):
#   num.partitions, retention.ms, enable.auto.commit (client), group.protocol=consumer (KIP-848)
```

### ⑥ Keywords
- 🧠 **Cần HIỂU:** partition = song song + ordering; quy tắc 1 partition–1 consumer/group; key→partition; offset semantics; commit trước/sau.
- 📌 **Cần THUỘC:** topic/partition/offset/broker/cluster; `hash(key)%N`; `__consumer_offsets`; at-most vs at-least theo thời điểm commit.
- 🛠️ **Cần LÀM ĐƯỢC:** chọn key để giữ ordering; cấu hình manual commit process-then-commit; biết khi tăng partition là cần thiết.

### ⑦ Mental model
**"1 partition = 1 băng ghi có thứ tự, 1 consumer/group đọc nó. Key quyết định partition ⇒ key quyết định ordering. Commit nghĩa là 'đã xử lý xong', đặt sau khi xử lý."**

### ⑧ Phỏng vấn & thách đố
1. *(dễ)* Topic/partition/offset/broker quan hệ thế nào? Offset có duy nhất across partition?
2. *(TB)* Vì sao partition vừa là song song vừa là ordering? Tăng partition hại gì?
3. *(TB)* `#consumer > #partition` thì sao?
4. *(khó)* Producer quyết partition bằng gì? Vai trò key?
5. *(khó)* Commit trước vs sau xử lý → semantics nào?
- 🎯 **Thách đố:** *"Đội tăng partition từ 6→12 để 'nhanh hơn', sau đó khách báo event của cùng một đơn xử lý sai thứ tự. Vì sao?"* → `hash(key)%N` đổi khi N đổi → message cùng key có thể rơi partition khác so với trước → mất ordering per-key trong giai đoạn chuyển + giữa message cũ/mới. Đổi số partition không nên làm bừa khi cần ordering theo key.

### ⑨ Bài tập + tiêu chí
Topic `payments`, cần: (1) mọi event của cùng `accountId` đúng thứ tự, (2) song song tối đa 8 luồng. Chọn key + số partition tối thiểu + giải thích trần consumer.
- **Đạt khi:** key=`accountId`; partition ≥ 8 (trần song song = #partition); nêu rõ thêm consumer quá 8 sẽ idle; chấp nhận không có global order.

### ⑩ Đọc thêm
- kafka.apache.org/documentation (Design, Implementation). KafkaJS docs. *(verify cấu hình.)*

---

## Bài 4 — Kafka Internals II: replication, rebalance, retention, EOS, lag
*Phủ: K-KAFKA-006 → 012*

### ① Mục tiêu & vị trí trong mạch
Hoàn thiện Kafka: **độ bền** (replication/ISR), **vận hành nhóm** (rebalance), **vòng đời dữ liệu** (retention vs compaction), **exactly-once** (EOS), và 2 lỗi vận hành sống còn (**hot partition**, **consumer lag**). Sau bài này bạn đủ nền cho Bài 5 (delivery) và Bài 6 (Outbox dựa trên producer ack/EOS).

### ② Giảng cơ bản → nâng cao

**(a) Replication & ISR (📘 K-KAFKA-007, 008):** mỗi partition có **1 leader + N-1 follower** (N = replication factor). **ISR (in-sync replicas)** = tập replica đang *theo kịp* leader. 
- `acks=all` (producer) = chờ **mọi ISR** ghi xong mới coi message "đã nhận".
- `min.insync.replicas` = ngưỡng ISR tối thiểu để chấp nhận ghi; nếu ISR tụt dưới ngưỡng → producer bị từ chối (bảo vệ khỏi mất dữ liệu khi leader chết).
- **Đánh đổi:** độ bền ↑ ↔ latency/availability ↓ (chờ nhiều replica hơn). Cấu hình kinh điển bền: RF=3, `min.insync.replicas=2`, `acks=all`. *(verify.)*
- **Leader chết (📘 K-KAFKA-008):** controller bầu một follower **trong ISR** lên làm leader mới; producer/consumer reconnect. Kafka cluster vốn HA → **"available, sometimes consistent"**. Thực tế điểm chết đáng lo hơn là **consumer chết** (xử lý lại từ last committed offset — at-least-once), không phải "Kafka chết".

**(b) Rebalance (📘 K-KAFKA-006):** khi consumer join/leave/chết hoặc đổi #partition → group **reassign** partition cho các consumer. Nguy hiểm vì:
- **Stop-the-world (kiểu cũ, eager):** trong lúc rebalance, consumer **ngừng xử lý** → lag tăng; có thể xử lý lại/đảo thứ tự nếu commit chưa kịp.
- **Cải tiến:** **cooperative/incremental rebalance** (KIP-429 và **KIP-848 GA ở Kafka 4.0**) chỉ chuyển phần partition cần thiết, không revoke tất cả → giảm đau đáng kể. Consumer opt-in bằng `group.protocol=consumer`. *(verify — đây là điểm cập nhật quan trọng: câu cũ "rebalance = stop-the-world" nay phải kèm "đang dịch sang incremental".)*

**(c) Retention vs Compaction (📘 K-KAFKA-009):**
- **Retention** (theo thời gian/dung lượng) → **xoá message cũ** quá hạn. Dùng cho event-log thông thường.
- **Compaction** → giữ **bản ghi mới nhất theo key** (xoá bản cũ cùng key) → topic biến thành **"snapshot trạng thái hiện tại per key"**. Dùng cho changelog/config/materialize state (vd topic giữ trạng thái mới nhất của mỗi `userId`). Chọn theo: cần *lịch sử đầy đủ* (retention) hay chỉ *state hiện tại* (compaction). *(verify `cleanup.policy`.)*

**(d) Exactly-once (EOS) (📘 K-KAFKA-010):** ghép từ 2 mảnh:
- **Idempotent producer** (`enable.idempotence=true`, mặc định bật ở đời mới) — dedup retry phía producer bằng *producer id + sequence number* → không ghi trùng khi producer retry.
- **Transactions** — cho phép *read-process-write* nguyên tử **trong Kafka** (vd Kafka Streams): hoặc cả batch commit, hoặc không gì. 
- **Giới hạn cốt lõi:** EOS chỉ "exactly-once" **trong biên Kafka**. Khi side-effect ra **DB ngoài / API ngoài**, không có phép màu → vẫn cần **idempotent upsert / Outbox** → đạt **"effectively-once"** (Bài 5, 6). KIP-890 (4.x) củng cố transaction protocol. *(verify; cross-ref I-IDEM.)*

**(e) Hai lỗi vận hành:**
- **Hot partition / data skew (📘 K-KAFKA-011):** key phân bố lệch → 1 partition nhận phần lớn traffic → 1 consumer quá tải, số còn lại rảnh. **"Key theo tenantId" nguy hiểm** vì tenant lớn làm nóng partition của nó. Xử lý: chọn key phân tán hơn (composite key `tenantId:entityId`), tách riêng tenant lớn, hoặc đánh đổi ordering. Nhận diện trade-off **ordering ↔ cân tải**.
- **Consumer lag (📘 K-KAFKA-012):** `lag = latest offset − committed offset` = số message **chưa xử lý**. Tăng dần = consumer chậm hơn producer. Xử lý: tăng consumer (≤ #partition) / tăng partition / tối ưu xử lý / batch / đẩy side-effect nặng ra async. Lag là **metric sống còn** của lớp messaging — phải alert. *(verify công cụ đo: `kafka-consumer-groups.sh`, Burrow, hoặc exporter Prometheus.)*

### ③ ⚠️ Cũ → Mới

| Cái cũ | Nay | Vì sao |
|---|---|---|
| "Rebalance luôn stop-the-world" | **Cooperative/incremental** (KIP-848 GA 4.0, `group.protocol=consumer`) | Giảm downtime rebalance *(verify)* |
| "Exactly-once là bất khả / hoặc có sẵn mọi nơi" | EOS có **trong Kafka** (idempotent producer + transactions); ra ngoài → effectively-once | Phân biệt phạm vi *(verify)* |
| `enable.idempotence` phải tự bật | Đời mới **mặc định bật** idempotent producer | Vẫn nên verify theo client/đời |
| ZooKeeper điều phối controller/ISR | **KRaft** controller quorum | 4.x KRaft-only *(verify)* |

### ④ Áp dụng thực tế + bigtech
**Use case:** topic changelog giữ "trạng thái tài khoản mới nhất" → **compaction** (không cần lịch sử, cần state hiện tại để service khởi động đọc lại). Topic giao dịch tài chính → **retention dài + RF=3 + acks=all** (cần lịch sử + bền tuyệt đối). **Bigtech:** Kafka Streams dùng compacted topic làm state store/changelog; EOS dùng trong pipeline streaming nội bộ. *(pattern; verify.)*

### ⑤ Code thực hành + cấu hình
Producer bền + idempotent (KafkaJS/config-style):

```javascript
// producer bền: idempotent + acks=all
const producer = kafka.producer({
  idempotent: true,        // dedup retry phía producer (producerId+seq) // verify tên option theo client
  maxInFlightRequests: 5,  // idempotent cho phép >1 nhưng giữ ordering
  // acks=all được suy ra khi idempotent=true (verify)
});
```

```properties
# topic config (verify theo đời broker)
# Durable log:
replication.factor=3
min.insync.replicas=2
# acks=all đặt phía producer
# Compacted state topic:
cleanup.policy=compact
# Retention log thường:
retention.ms=604800000   # 7 ngày
```

```bash
# Đo lag (verify công cụ/cờ theo đời)
kafka-consumer-groups.sh --bootstrap-server localhost:9092 \
  --describe --group inventory-svc   # cột LAG = latest - current offset
```

### ⑥ Keywords
- 🧠 **Cần HIỂU:** ISR + acks/min.insync (độ bền↔latency); rebalance (eager vs cooperative); retention vs compaction; EOS phạm vi; hot partition; lag.
- 📌 **Cần THUỘC:** RF=3 + min.insync.replicas=2 + acks=all; `cleanup.policy=compact`; `lag = latest − committed`; idempotent producer = producerId+sequence.
- 🛠️ **Cần LÀM ĐƯỢC:** cấu hình topic bền; chọn compaction khi cần state; chẩn đoán lag tăng; nhận diện & xử lý hot partition.

### ⑦ Mental model
**"Bền = RF3 + min.insync 2 + acks=all (ISR là người gác). Rebalance đang chuyển từ stop-the-world sang incremental. EOS chỉ trong Kafka — ra ngoài phải tự idempotent. Lag là nhịp tim của consumer."**

### ⑧ Phỏng vấn & thách đố
1. *(TB)* ISR là gì? `acks=all` + `min.insync.replicas` bảo vệ điều gì, đổi gì?
2. *(TB)* Rebalance kích hoạt khi nào, vì sao nguy hiểm, cải tiến gì? (nhắc KIP-848)
3. *(khó)* Retention vs compaction — khi nào compaction?
4. *(rất khó)* Exactly-once Kafka đạt bằng gì, giới hạn ở đâu khi ghi DB ngoài?
5. *(khó)* Hot partition do "key=tenantId" — chẩn đoán & xử lý?
6. *(khó)* Lag là gì, tăng vô hạn thì làm gì?
- 🎯 **Thách đố:** *"Dev bật Kafka transactions và tuyên bố 'giờ ghi DB cũng exactly-once, bỏ idempotency consumer được rồi'. Sai chỗ nào?"* → EOS chỉ nguyên tử **trong Kafka** (read-process-write giữa các topic). Ghi ra **DB ngoài** nằm ngoài transaction Kafka → vẫn có thể chạy 2 lần khi redeliver/crash → **bắt buộc** giữ idempotent upsert/dedup (effectively-once). Transactions ≠ miễn idempotency ở biên ngoài.

### ⑨ Bài tập + tiêu chí
Thiết kế config cho topic `ledger-entries` (không được mất, cần lịch sử) và topic `account-balance-latest` (chỉ cần số dư hiện tại). Nêu RF, acks, cleanup.policy, lý do.
- **Đạt khi:** ledger: RF=3, min.insync=2, acks=all, retention dài/compact=delete; balance-latest: `cleanup.policy=compact`, key=accountId; giải thích đúng "lịch sử vs state".

### ⑩ Đọc thêm
- kafka.apache.org: Replication, Transactions, Log Compaction; KIP-848, KIP-890 (đọc bản cập nhật 4.x). *(verify.)*

---

## Bài 5 — Delivery Semantics & Idempotent Consumer
*Phủ: K-DELIV-001 → 009*

### ① Mục tiêu & vị trí trong mạch
Đây là **bài lõi reliability** của cả Mục K — mọi câu Outbox/Saga/resilience đều dựa vào nó. Học xong bạn hiểu **at-most/at-least/exactly-once**, vì sao **at-least-once là mặc định thực tế** và consumer **bắt buộc idempotent**, cách xử lý **ordering**, **DLQ/poison message**, và **head-of-line blocking** khi retry. Cross-ref I-IDEM (đã học idempotency góc concurrency) — ở đây ta nhìn **góc cơ chế messaging**.

### ② Giảng cơ bản → nâng cao

**(a) Trực giác — 3 mức delivery (📘 K-DELIV-001):**
- **At-most-once** = "gửi một lần, mất thì thôi" (commit/ack *trước* khi xử lý) → có thể **mất**, không trùng.
- **At-least-once** = "gửi tới khi chắc nhận được" (xử lý *trước* rồi mới ack/commit) → không mất nhưng có thể **trùng**. **Đây là mặc định phổ biến.**
- **Exactly-once** *delivery* qua biên mạng = gần như **bất khả** (FLP/two-generals). Thực tế: **at-least-once + dedup = "effectively-once"**. *(cross-ref I-IDEM-002.)*

**(b) Cơ chế:**
- **Vì sao consumer BẮT BUỘC idempotent dưới at-least-once (📘 K-DELIV-002):** broker sẽ **redeliver** (timeout, rebalance, retry, crash trước commit) → side-effect chạy **2 lần**. Cơ chế dedup **messaging-specific**: lưu `messageId/eventId` đã xử lý vào **dedup store** (thấy lại → bỏ qua), hoặc **unique constraint** khi ghi DB. Khác góc HTTP `Idempotency-Key` (F-IDEM) — ở đây id gắn vào *message*, không phải header request.
- **Ack/nack & redelivery (📘 K-DELIV-003):** quy tắc — **ack/commit SAU khi xử lý + ghi side-effect xong**. Ack sớm → crash giữa chừng = mất. Nack/không-ack → broker redeliver (RabbitMQ requeue; Kafka đơn giản *không commit offset*). Câu thần chú: *ack = "tôi đã xử lý xong", không phải "tôi đã nhận"*.
- **Ordering (📘 K-DELIV-004):** đảm bảo event của **cùng một thực thể** đúng thứ tự bằng **route theo key**: Kafka cùng key→cùng partition; RabbitMQ consistent-hash hoặc single-queue-per-key; SQS FIFO message group id. Thường **không cần global order**, chỉ **per-entity**. Trade-off: ordering ↔ song song.
- **Nghịch lý scale↔ordering (📘 K-DELIV-009):** tăng consumer để giảm lag → nhiều partition xử lý song song → **vỡ thứ tự global**. Dung hoà: ordering chỉ cần **per-key** (1 key→1 partition→1 consumer), song song **giữa các key khác nhau**. Câu hỏi đúng luôn là *"ordering ở scope nào?"*.

**(c) Mép giới hạn — DLQ, poison, head-of-line:**
- **Poison message + DLQ (📘 K-DELIV-005):** message **luôn fail** (data hỏng/bug) → retry vô hạn làm **kẹt queue / chặn partition**. Sau **N lần fail** → đẩy sang **Dead Letter Queue** để điều tra thủ công, không chặn luồng. Retry phải **có giới hạn + (jittered) backoff**; **alert khi DLQ tăng**.
- **In-line retry vs retry-topic (📘 K-DELIV-006):** Kafka xử lý **tuần tự per-partition** → *block-and-retry* 1 message = **chặn mọi message sau nó** trong partition (**head-of-line blocking**). Giải: **retry topic riêng** (đẩy message lỗi sang, xử lý sau với delay tăng dần) hoặc delay queue. Phân biệt lỗi **transient** (retry) vs **vĩnh viễn** (DLQ ngay).
- **Consumer chậm + visibility timeout (📘 K-DELIV-007):** xử lý chậm → visibility timeout/lease hết → broker **redeliver** → **2 worker cùng xử lý 1 message** → bắt buộc idempotent + **atomic claim**. Tách side-effect nặng / đảm bảo ack nhanh / chia nhỏ. *(cross-ref I-IDEM parallel retry.)*
- **Effectively-once end-to-end (📘 K-DELIV-008):** ghép từ 3 mảnh, **không có nút thần kỳ**:
  1. **Producer** idempotent / **Outbox** → publish không mất/không trùng (Bài 6).
  2. **Broker** durable + at-least-once (Bài 4).
  3. **Consumer** dedup theo `eventId` + ghi DB idempotent (upsert/unique) **atomic** với việc lưu "đã xử lý" (Inbox — Bài 6).

### ③ ⚠️ Cũ → Mới

| Cái cũ | Nay | Vì sao |
|---|---|---|
| "Dùng exactly-once delivery cho chắc" | **at-least-once + dedup = effectively-once** | EOD bất khả ở biên ngoài |
| Retry ngay tại consumer (block) trên Kafka | **Retry topic / delay queue** + DLQ | Tránh head-of-line blocking |
| "Ack ngay khi nhận cho nhẹ" | **Ack sau khi xử lý xong** | Ack sớm = mất message |
| Cần global ordering | Ordering **per-key** là đủ trong hầu hết domain | Mở khoá song song |

### ④ Áp dụng thực tế + bigtech
**Use case:** consumer trừ kho khi `OrderPlaced`. At-least-once → có thể nhận 2 lần → nếu không idempotent sẽ **trừ kho gấp đôi**. Vá: bảng `processed_events(event_id PK)`, trong **một transaction**: kiểm tra/chèn event_id + trừ kho; trùng → `event_id` đã tồn tại → skip. **Bigtech:** mọi pipeline at-least-once nghiêm túc đều có lớp dedup + DLQ + retry-topic; payment systems coi idempotency là bắt buộc tuyệt đối. *(pattern.)*

### ⑤ Code thực hành + cấu hình
Idempotent consumer (dedup atomic) — Postgres + KafkaJS:

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
        // 2) side-effect CHỈ chạy khi event mới -> idempotent
        await client.query('UPDATE inventory SET qty = qty - $1 WHERE sku=$2', [evt.qty, evt.sku]);
      }
      await client.query('COMMIT'); // dedup + side-effect nguyên tử cùng nhau
    } catch (e) { await client.query('ROLLBACK'); throw e; } // throw -> không commit offset -> redeliver (an toàn vì idempotent)
    finally { client.release(); }
    await consumer.commitOffsets([{ topic, partition, offset: (Number(message.offset)+1).toString() }]);
  },
});
```

```sql
-- bảng dedup (Inbox tối giản)
CREATE TABLE processed_events ( event_id TEXT PRIMARY KEY, processed_at TIMESTAMPTZ DEFAULT now() );
```

> ⚠️ **AI hay sai:** đặt dedup-check và side-effect ở **2 transaction tách rời** → crash giữa hai bước vẫn double-process. Phải **nguyên tử** (cùng 1 transaction) hoặc dùng unique constraint trên chính bảng nghiệp vụ.

### ⑥ Keywords
- 🧠 **Cần HIỂU:** 3 mức delivery; effectively-once; vì sao idempotent bắt buộc; ordering per-key; head-of-line blocking; poison/DLQ.
- 📌 **Cần THUỘC:** at-most (commit trước) / at-least (commit sau) / exactly (bất khả ở biên); ack = "đã xử lý xong"; dedup theo eventId; retry-topic vs in-line.
- 🛠️ **Cần LÀM ĐƯỢC:** viết idempotent consumer atomic; thiết kế DLQ + retry-topic + alert; chọn key giữ ordering per-entity.

### ⑦ Mental model
**"Mặc định = at-least-once ⇒ consumer PHẢI idempotent (dedup eventId atomic). Ack sau khi xong. Ordering chỉ cần per-key. Retry bằng retry-topic, fail mãi → DLQ. Effectively-once = chuỗi cơ chế, không phải một cờ."**

### ⑧ Phỏng vấn & thách đố
1. *(TB)* Định nghĩa + đánh đổi 3 mức delivery.
2. *(khó)* Vì sao at-least-once bắt buộc idempotent? Dedup messaging-specific thế nào?
3. *(TB)* "Ack trước khi xử lý xong" sai ở đâu?
4. *(khó)* Đảm bảo ordering per-entity bằng gì?
5. *(rất khó)* Effectively-once end-to-end ghép từ những mảnh nào?
- 🎯 **Thách đố 1:** *"Tăng consumer giảm lag xong khách báo trạng thái order nhảy lung tung — giải thích nghịch lý và sửa."* → song song nhiều partition phá global order; sửa: route theo key (orderId→partition), ordering trong key + song song giữa key.
- 🎯 **Thách đố 2:** *"Một message data hỏng làm cả partition đứng. Vì sao và xử lý?"* → head-of-line blocking do retry in-line tuần tự; đẩy sang retry-topic/DLQ sau N lần, không block partition.

### ⑨ Bài tập + tiêu chí
Viết (pseudo hoặc thật) consumer `PaymentCaptured` cập nhật `order.paid=true`, đảm bảo: idempotent, có DLQ sau 5 lần fail, không block partition.
- **Đạt khi:** dedup theo `eventId` atomic với update; phân biệt transient (retry-topic + backoff) vs permanent (DLQ ngay); alert DLQ; giải thích vì sao update `paid=true` còn cần dedup (redeliver).

### ⑩ Đọc thêm
- kafka.apache.org (Delivery Semantics); confluent retry/DLQ patterns; Chris Richardson *microservices.io* "Idempotent Consumer", "Transactional Outbox". *(verify.)*

---

## Bài 6 — Dual-write, Transactional Outbox & CDC
*Phủ: K-OBX-001 → 008*

### ① Mục tiêu & vị trí trong mạch
Bài này vá **bẫy đã gài ở Bài 1**: dòng `insert` rồi `publish`. Đây là **dual-write problem** — và Outbox/CDC là lời giải chuẩn. Là **baseline bắt buộc** cho Saga (Bài 7) và là câu "cũ→hiện đại" kinh điển (2PC vs Saga+Outbox). Cross-ref K-DELIV-008 (effectively-once).

### ② Giảng cơ bản → nâng cao

**(a) Trực giác — dual-write (📘 K-OBX-001):** bạn phải ghi **2 hệ thống** (DB + broker) nhưng chúng **không nằm trong một transaction**. Bốn kịch bản:
1. DB commit OK + publish OK → ✅.
2. DB commit OK + publish **fail** → **mất event** (downstream không biết order tồn tại).
3. DB rollback + publish **OK** → **event ma** (downstream xử lý order không tồn tại).
4. Cả hai fail → ✅ (nhất quán ở trạng thái "chưa làm gì").
Không có atomicity giữa 2 nguồn → **try/catch không cứu được** (publish có thể thành công nhưng app crash trước khi biết). Cần **pattern**.

**(b) Transactional Outbox (📘 K-OBX-002):** ghi **business data** + ghi **event vào bảng `outbox`** trong **cùng một ACID transaction của DB** → **atomic**. Một tiến trình bất đồng bộ đọc `outbox` rồi publish lên broker (đánh dấu đã gửi). **Event chỉ tồn tại nếu DB commit** → loại bỏ kịch bản 2 và 3. Đây là chìa khoá: chuyển "2 hệ thống" thành "1 transaction DB".

**(c) Đẩy outbox ra broker — 2 cách (📘 K-OBX-003, 004):**
- **Polling publisher** — job quét bảng `outbox` theo chu kỳ, publish, đánh dấu `published`. Đơn giản, không thêm hạ tầng; nhưng có **độ trễ** (chu kỳ poll) + **tải DB**.
- **CDC (Change Data Capture)** — đọc **transaction log** của DB (WAL/binlog) bằng **Debezium** → phát event **realtime** cho mọi thay đổi (kể cả ghi từ ngoài app); ít tải hơn poll, độ trễ thấp; nhưng **thêm hạ tầng** (Kafka Connect / Debezium Server). *(verify: Debezium 3.4.x hỗ trợ Kafka 4.x; có Debezium Server/Engine chạy không cần Kafka Connect; EOS cho core connector từ 3.3.)*
- **Vì sao CDC > polling DB để đồng bộ (📘 K-OBX-004):** đọc commit log = **không miss** thay đổi nào (kể cả update ngoài app), **realtime**, **ít tải** hơn `SELECT ... WHERE updated_at > ?` lặp lại; còn dùng để đồng bộ DB→search/cache. Câu "cũ→mới": **"polling DB để đồng bộ" → "CDC/event-driven"**.

**(d) Đầu nhận — Inbox (📘 K-OBX-005, 006):** Outbox đảm bảo **at-least-once publish** → consumer vẫn nhận **trùng**. **Inbox pattern**: consumer ghi `eventId` nhận được vào **bảng `inbox`** trong **cùng transaction** với xử lý → dedup **bền + atomic** "đã xử lý". Đối xứng với Outbox: **Outbox giải "không mất", Inbox/dedup giải "không trùng"** = hai nửa của effectively-once.

**(e) Vận hành outbox (📘 K-OBX-008):** bảng phình → **dọn** (archive/delete row đã publish); publish có thể trùng → consumer dedup; **giữ ordering** bằng publish theo thứ tự ghi (sequence/created_at) + key partition; **giám sát backlog outbox** y như giám sát consumer lag.

**(f) 2PC vs Saga+Outbox (📘 K-OBX-007) — câu kinh điển:**

| | 2PC (distributed transaction) | Saga + Outbox/CDC |
|---|---|---|
| Cơ chế | coordinator khoá tất cả participant, prepare→commit | chuỗi local tx + event, eventual |
| Khi coordinator chết | participant **kẹt** (blocking) | không có coordinator khoá toàn cục |
| Availability/throughput | giảm (khoá, đồng bộ) | cao hơn (không khoá toàn cục) |
| Microservices | **ít dùng** | **chuẩn hiện đại** |

Vì sao 2PC ít dùng: **blocking + kém chịu lỗi + giảm availability**. Microservices thiên về **eventual** qua Saga + Outbox/CDC.

### ③ ⚠️ Cũ → Mới

| Cái cũ | Nay | Vì sao |
|---|---|---|
| `insert(); publish();` trong cùng hàm | **Transactional Outbox** (ghi outbox cùng tx) | Xoá dual-write (mất/ma event) |
| **Polling DB** để đồng bộ service/search/cache | **CDC (Debezium)** đọc WAL/binlog | Realtime, không miss, ít tải *(verify)* |
| **2PC / XA** cho consistency xuyên service | **Saga + Outbox** (eventual) | 2PC blocking, kém chịu lỗi |
| Chỉ Outbox là đủ | Outbox (producer) **+** Inbox/dedup (consumer) | Outbox = không mất; cần thêm để không trùng |

### ④ Áp dụng thực tế + bigtech
**Use case:** `placeOrder` ghi `orders` + ghi `outbox(OrderPlaced)` trong 1 tx; Debezium đọc WAL bảng `outbox` → publish lên Kafka topic `orders`; InventorySvc consume + dedup qua Inbox. **Bigtech:** Outbox + Debezium CDC là pattern phổ biến để "publish event tin cậy" mà không cần 2PC; nhiều hệ dùng Debezium đồng bộ DB→Elasticsearch/cache realtime. *(pattern; verify Debezium docs.)*

### ⑤ Code thực hành + cấu hình
Outbox atomic (Postgres):

```javascript
// placeOrder.js — ghi order + outbox trong CÙNG transaction (atomic)
async function placeOrder(input) {
  const client = await pool.connect();
  try {
    await client.query('BEGIN');
    const { rows } = await client.query(
      'INSERT INTO orders(id, status, payload) VALUES($1,$2,$3) RETURNING *',
      [input.id, 'PENDING', input]
    );
    await client.query(
      `INSERT INTO outbox(id, aggregate_type, aggregate_id, type, payload)
       VALUES($1,'order',$2,'OrderPlaced',$3)`,
      [crypto.randomUUID(), input.id, JSON.stringify(rows[0])]
    ); // event sống/chết CÙNG order -> không bao giờ "DB ok mà mất event"
    await client.query('COMMIT');
    return rows[0];
  } catch (e) { await client.query('ROLLBACK'); throw e; }
  finally { client.release(); }
}
// KHÔNG publish ở đây. Debezium (CDC) hoặc polling job sẽ đẩy outbox -> broker.
```

```sql
CREATE TABLE outbox (
  id UUID PRIMARY KEY,
  aggregate_type TEXT, aggregate_id TEXT, type TEXT,
  payload JSONB, created_at TIMESTAMPTZ DEFAULT now(),
  published BOOLEAN DEFAULT false        -- cho polling; với CDC có thể bỏ
);
```

```json
// Debezium connector (Outbox Event Router) — verify tên/đời connector
{
  "name": "outbox-connector",
  "config": {
    "connector.class": "io.debezium.connector.postgresql.PostgresConnector",
    "table.include.list": "public.outbox",
    "transforms": "outbox",
    "transforms.outbox.type": "io.debezium.transforms.outbox.EventRouter"
  }
}
```

```
# .env (KHÔNG hardcode): DATABASE_URL, KAFKA_BROKERS, CONNECT_URL
# Postgres cần wal_level=logical + replication slot cho Debezium // verify
```

### ⑥ Keywords
- 🧠 **Cần HIỂU:** dual-write 4 kịch bản; Outbox = 1 transaction DB; CDC đọc WAL; Inbox đối xứng; vì sao 2PC bị né.
- 📌 **Cần THUỘC:** Outbox (ghi business+event cùng tx); polling vs CDC; Debezium đọc WAL/binlog; Inbox dedup; "polling→CDC", "2PC→Saga+Outbox".
- 🛠️ **Cần LÀM ĐƯỢC:** viết placeOrder atomic với outbox; cấu hình Debezium EventRouter (verify); giám sát outbox backlog.

### ⑦ Mental model
**"Đừng ghi 2 nơi. Ghi business + event vào outbox trong MỘT transaction DB, để CDC (Debezium) đẩy đi. Outbox = không mất, Inbox/dedup = không trùng. 2PC chết, Saga+Outbox sống."**

### ⑧ Phỏng vấn & thách đố
1. *(khó)* Dual-write là gì, vì sao try/catch không cứu?
2. *(rất khó)* Outbox đảm bảo atomicity thế nào?
3. *(khó)* Polling vs CDC — đánh đổi? Vì sao CDC tốt hơn polling DB?
4. *(khó)* Inbox bổ trợ Outbox ra sao?
5. *(rất khó)* 2PC vs Saga+Outbox — vì sao 2PC ít dùng ở microservices?
- 🎯 **Thách đố:** *"Dev nói: 'tôi publish event trong cùng try với DB commit, có rollback rồi, an toàn'. Phản biện."* → publish lên broker **không** nằm trong DB transaction; nếu DB commit xong rồi app crash trước publish → mất event; nếu publish xong rồi DB rollback → event ma. Rollback DB không undo được message đã rời broker. Phải Outbox.

### ⑨ Bài tập + tiêu chí
Chuyển `placeOrder` (đang `insert` + `publish`) sang Outbox + chọn polling **hoặc** CDC, nêu cách dedup phía consumer.
- **Đạt khi:** business+outbox cùng 1 tx; mô tả đẩy ra broker (polling job đánh dấu published / Debezium EventRouter); consumer dedup theo eventId (Inbox); nêu cách dọn outbox + giám sát backlog.

### ⑩ Đọc thêm
- microservices.io: Transactional Outbox, Polling Publisher, Transaction Log Tailing. debezium.io (Outbox Event Router, CDC). *(verify đời.)*

---

## Bài 7 — Saga: orchestration vs choreography & compensation
*Phủ: K-SAGA-001 → 009*

### ① Mục tiêu & vị trí trong mạch
Saga là cách đạt **nhất quán xuyên nhiều service** khi không có transaction chung (database-per-service — Bài 10). Nó **đứng trên** mọi thứ đã học: dùng **Outbox** (Bài 6) + **idempotent consumer** (Bài 5) làm baseline. Học xong bạn thiết kế được một saga checkout đúng và gọi tên được các chỗ chết người. Cross-ref H-CQRS (Saga kiểu Nest/RxJS).

### ② Giảng cơ bản → nâng cao

**(a) Vì sao Saga (📘 K-SAGA-001):** mỗi service có **DB riêng** → không có ACID transaction chung. 2PC thì khoá + kém chịu lỗi (Bài 6). **Saga** = **chuỗi local transaction**, mỗi bước commit riêng ở service của nó; nếu một bước fail → chạy **compensating transaction** hoàn tác các bước trước → đạt **nhất quán eventual** không cần lock toàn cục.

**(b) Hai kiểu điều phối (📘 K-SAGA-002, 003):**

| | Orchestration | Choreography |
|---|---|---|
| Điều phối | **1 orchestrator** trung tâm ra lệnh từng bước, lưu state | mỗi service **phát event**, service khác phản ứng |
| State | tập trung, **dễ nhìn / dễ test** | phân tán, **khó theo dõi** |
| Coupling | service biết orchestrator | decouple cao |
| Hợp với | saga **dài/phức tạp** (5+ bước, branching, conditional) | saga **ngắn** (2–4 bước), event flow rõ |
| Công cụ | **Temporal**, **Camunda 8/Zeebe**, Orkes Conductor | broker thuần (Kafka/Rabbit) |

Chọn (📘 K-SAGA-003): **không** máy móc "đơn giản→choreography". Xét **cognitive load + visibility**: choreography ngắn & team quen event-driven; orchestration khi cần visibility/branching/test. **Hệ lớn thường dùng cả hai.** *(verify Temporal/Camunda.)*

**(c) Compensation — điểm khó nhất:**
- **Compensation ≠ rollback (📘 K-SAGA-004):** rollback = quay về như chưa xảy ra (trong **1 DB**). Compensation = **hành động nghiệp vụ ngược** trong một thế giới **đã thay đổi** (đã trừ kho, đã gửi mail, đã charge). Có thể **không hoàn tác sạch**: `refund` ≠ "chưa từng charge" (phí, thời gian, lịch sử). Phải thiết kế **semantic compensation**, thứ tự compensate **ngược**, và **idempotent**.
- **Thiếu Isolation (chữ I của ACID) (📘 K-SAGA-005):** giữa các bước, **state trung gian lộ ra** cho giao dịch khác → dirty read nghiệp vụ (thấy đơn `PENDING`), anomalies (lost update, đọc dữ liệu sẽ bị compensate). Giảm thiểu: **semantic lock** (trạng thái `PENDING`/`RESERVED`), versioning, commutative updates, re-read. Đây là điểm khó nhất của saga.
- **Compensation cũng fail (📘 K-SAGA-007):** compensation phải **retry-able + idempotent**; lưu **state saga bền** (đang/đã compensate bước nào); fail mãi → DLQ + alert + can thiệp thủ công. Không để hệ kẹt ở trạng thái nửa vời **im lặng**.

**(d) Baseline bắt buộc cho MỌI saga (📘 K-SAGA-006) — chọn pattern không thay được cái này:**
1. **Atomic tại biên mỗi bước**: ghi DB + publish event nguyên tử (**Outbox/CDC** — Bài 6).
2. **Idempotent consumer** (at-least-once — Bài 5).
3. **Compensation tường minh + DLQ**.
4. **Observability**: correlation id + saga state truy vấn được (Bài 12).

**(e) Đừng nhầm 2 mức (📘 K-SAGA-008):** `@nestjs/cqrs` Saga = phản ứng event **trong một process** bằng RxJS (điều phối command nội bộ). Saga **distributed** = điều phối **qua nhiều service**, **long-running**, cần bền vững → công cụ riêng (Temporal/Camunda) hoặc choreography qua broker. *(verify.)*

### ③ ⚠️ Cũ → Mới

| Cái cũ | Nay | Vì sao |
|---|---|---|
| 2PC/XA xuyên service | **Saga** (local tx + compensation) | 2PC blocking, kém chịu lỗi |
| Tự viết orchestrator + cron + bảng state | **Temporal / Camunda 8** (durable execution, retry/timer/visibility sẵn) | "2026: workflow-orchestrated" *(verify)* |
| "Saga đơn giản thì choreography" (máy móc) | Xét **visibility + cognitive load**; hệ lớn dùng **cả hai** | Tránh kết luận cứng |
| Coi compensation như rollback | **Semantic compensation** (idempotent, có thể không sạch) | Thế giới đã đổi |

### ④ Áp dụng thực tế + bigtech
**Checkout saga (📘 K-SAGA-009):** `reserve inventory → charge payment → confirm order`. Mỗi bước = local tx + event/command. Fail payment → compensate **release inventory**. **Ba chỗ dễ sai:** (a) **không idempotent** → trừ tiền 2 lần khi redeliver; (b) **lộ state PENDING** → oversell / đọc bẩn (thiếu semantic lock); (c) **publish trong tx code** thay vì Outbox → mất event. **Bigtech:** Temporal nổi tiếng cho durable saga (compensation chạy kể cả sau server crash); Camunda 8/Zeebe cho saga BPMN visual; Netflix Conductor/Orkes cho orchestration quy mô lớn. *(verify.)*

### ⑤ Code thực hành + cấu hình
Orchestration saga (pseudo, kiểu Temporal — minh hoạ try/compensate):

```javascript
// checkoutSaga.js — orchestration: gọi activity, compensate ngược khi lỗi
// (Temporal đảm bảo workflow + compensation chạy lại đúng kể cả sau crash) // verify SDK
async function checkout(order) {
  const done = []; // ngăn xếp để compensate ngược
  try {
    await activities.reserveInventory(order); done.push('inventory');   // mỗi activity idempotent
    await activities.chargePayment(order);     done.push('payment');
    await activities.confirmOrder(order);
  } catch (err) {
    // compensate THEO THỨ TỰ NGƯỢC, mỗi compensation idempotent + retry-able
    if (done.includes('payment'))   await activities.refundPayment(order);
    if (done.includes('inventory')) await activities.releaseInventory(order);
    throw err; // ghi state FAILED, alert
  }
}
```

```javascript
// activity reserveInventory — semantic lock (RESERVED) tránh oversell + idempotent
async function reserveInventory({ orderId, sku, qty }) {
  // dedup theo orderId (Bài 5) + chuyển trạng thái sang RESERVED, không trừ "vĩnh viễn" cho tới confirm
  // UPDATE ... WHERE available >= qty  (commutative, re-check)  // verify SQL theo schema
}
```

> ⚠️ **AI hay sai:** viết compensation như "DELETE bản ghi" (rollback) thay vì hành động nghiệp vụ ngược (`refund`, `releaseReservation`); và quên compensation phải idempotent (chạy lại khi retry).

### ⑥ Keywords
- 🧠 **Cần HIỂU:** saga = local tx + compensation eventual; orchestration vs choreography; compensation ≠ rollback; thiếu Isolation → semantic lock; baseline saga.
- 📌 **Cần THUỘC:** 2PC bị né; orchestrator (Temporal/Camunda) vs event-driven; compensate thứ tự ngược + idempotent; 4 baseline.
- 🛠️ **Cần LÀM ĐƯỢC:** thiết kế checkout saga + compensation + gọi tên 3 chỗ sai; phân biệt Nest cqrs saga vs distributed saga.

### ⑦ Mental model
**"Saga = nhiều local transaction nối bằng event, lỗi thì làm hành động nghiệp vụ NGƯỢC (không phải rollback). Baseline = Outbox + idempotent + compensation + observability. Orchestration để nhìn rõ, choreography để decouple."**

### ⑧ Phỏng vấn & thách đố
1. *(khó)* Vì sao không ACID xuyên service? Saga giải gì?
2. *(khó)* Orchestration vs choreography — cơ chế & khi nào chọn?
3. *(rất khó)* Compensation khác rollback chỗ nào, vì sao khó đúng?
4. *(rất khó)* Saga thiếu Isolation — hệ quả & giảm thiểu?
5. *(khó)* Baseline bắt buộc cho mọi saga?
- 🎯 **Thách đố:** *"Thiết kế checkout saga của junior chạy happy-path ngon nhưng production thỉnh thoảng trừ tiền 2 lần và oversell. Hai nguyên nhân gốc?"* → (1) consumer/activity không idempotent dưới at-least-once → charge/trừ kho lặp khi redeliver; (2) thiếu semantic lock (RESERVED) → đọc bẩn state PENDING gây oversell. (Có thể thêm: publish trong tx → mất event.)

### ⑨ Bài tập + tiêu chí
Thiết kế saga "đặt vé": `giữ ghế → thanh toán → xuất vé`. Liệt kê bước, compensation mỗi bước, semantic lock, 2 chỗ idempotency bắt buộc.
- **Đạt khi:** compensation đúng nghiệp vụ ngược (nhả ghế, refund) + idempotent + thứ tự ngược; ghế ở trạng thái `HELD` (semantic lock, có TTL); idempotency ở thanh toán + xuất vé; nêu Outbox cho event.

### ⑩ Đọc thêm
- microservices.io: Saga pattern. Temporal docs (durable execution, saga). Camunda 8 docs. *(verify.)*

---

## Bài 8 — Event Sourcing & CQRS
*Phủ: K-ES-001 → 007*

### ① Mục tiêu & vị trí trong mạch
ES & CQRS là họ hàng "event-driven" nhưng ở **tầng lưu trữ trạng thái**, không phải giao tiếp. Học xong bạn phân biệt ES với "audit log", hiểu snapshot/projection/eventual của read model, và — quan trọng nhất — **khi nào KHÔNG dùng** (over-engineering). Cross-ref H-CQRS (kiểu Nest).

### ② Giảng cơ bản → nâng cao

**(a) Event Sourcing (📘 K-ES-001, 002):** thay vì lưu **state hiện tại** (UPDATE đè), ES lưu **chuỗi event bất biến** (append-only); **state hiện tại = replay/fold** các event. 
- **Được:** audit đầy đủ, time-travel (state tại bất kỳ thời điểm), rebuild view mới từ lịch sử.
- **Mất:** phức tạp, **query state hiện tại khó** (phải fold), eventual, **schema event tiến hoá khó**, cần snapshot.
- **Khác "audit log thường" (📘 K-ES-002):** trong ES, **event LÀ nguồn sự thật chính** (state dựng *từ* event). Audit log thường chỉ là **phụ phẩm** bên cạnh bảng state. Khác nhau ở **"ai là source of truth"**.

**(b) Snapshot (📘 K-ES-003):** replay từ event #0 mỗi lần quá tốn → lưu **snapshot** state tại offset N, chỉ replay event **sau N**. Cần khi stream dài / đọc thường xuyên. Trade-off: bộ nhớ ↔ tốc độ rebuild.

**(c) CQRS (📘 K-ES-004):** tách **write model** (command, tối ưu ghi/nhất quán) khỏi **read model** (query, tối ưu đọc). **ES + CQRS hay đi cùng** vì ES ghi event (write) khó query trực tiếp → **project** ra nhiều read model phù hợp truy vấn. Nhưng **CQRS không bắt buộc ES** và ngược lại — đừng gộp làm một.

**(d) Projection eventual (📘 K-ES-005):** read model cập nhật **bất đồng bộ** → **trễ** so với write → user có thể **không thấy ngay** thay đổi vừa ghi. Xử lý: chấp nhận + **UI optimistic**; đọc-từ-write-model cho path cần fresh; hiển thị "đang xử lý". Phải thiết kế cho độ trễ projection.

**(e) Schema evolution trong ES (📘 K-ES-006) — chi phí dài hạn ít ai lường:** event đã ghi **bất biến**, không sửa được → cần **versioning event** + **upcaster** (chuyển `event v1 → v2` khi đọc) hoặc weak schema (bỏ qua field thiếu). **Không bao giờ rewrite lịch sử** tuỳ tiện.

**(f) Khi nào KHÔNG dùng ES (📘 K-ES-007) — cảnh báo over-engineering:** CRUD đơn giản, không cần audit/time-travel, team chưa quen → ES thêm **phức tạp lớn** (eventual, versioning, rebuild) không tương xứng lợi ích. Chỉ dùng khi domain **thật sự cần lịch sử/audit/temporal**: finance, ledger, kế toán, đặt chỗ. Nhận diện **cost/benefit**.

### ③ ⚠️ Cũ → Mới

| Cái cũ | Nay | Vì sao |
|---|---|---|
| "ES = audit table cho mọi app" | ES chỉ khi cần lịch sử/audit/time-travel **là yêu cầu** | Tránh over-engineering |
| Replay từ #0 mỗi lần | **Snapshot** + replay phần đuôi | Tốc độ rebuild |
| Sửa/migrate event cũ tại chỗ | **Versioning + upcaster** (event bất biến) | Không rewrite lịch sử |
| Gộp ES = CQRS | Tách: CQRS không cần ES & ngược lại | Hai khái niệm độc lập |

### ④ Áp dụng thực tế + bigtech
**Use case hợp ES:** sổ cái tài khoản (mọi giao dịch là event bất biến; số dư = fold; cần audit tuyệt đối). **Use case KHÔNG nên:** CRUD hồ sơ user/profile. **CQRS không-ES:** hệ đọc nặng tách read replica/materialized view để query nhanh mà write vẫn CRUD thường. **Bigtech:** ledger/payment systems dùng ES; nhiều hệ dùng CQRS (read model riêng cho search/feed) mà không ES. *(pattern; verify khi cần ví dụ cụ thể.)*

### ⑤ Code thực hành + cấu hình
Replay + snapshot (pseudo):

```javascript
// rebuildState — fold event; dùng snapshot để khỏi replay từ đầu
function rebuildAccount(accountId) {
  const snap = store.getLatestSnapshot(accountId);          // {state, version} hoặc null
  let state = snap ? snap.state : { balance: 0, version: 0 };
  const events = store.getEventsAfter(accountId, state.version); // chỉ event sau snapshot
  for (const e of events) state = apply(state, e);          // fold: apply thuần, deterministic
  if (events.length > SNAPSHOT_EVERY) store.saveSnapshot(accountId, state); // tạo snapshot mới
  return state;
}
function apply(s, e) {                                       // PHẢI thuần + xử lý version event
  switch (e.type) {
    case 'Deposited':  return { ...s, balance: s.balance + e.amount, version: e.version };
    case 'Withdrawn':  return { ...s, balance: s.balance - e.amount, version: e.version };
    // case 'DepositedV2': ... // hoặc upcast v1->v2 trước khi vào đây
    default: return s; // weak schema: bỏ qua event lạ
  }
}
```

> ⚠️ **AI hay sai:** dùng ES cho CRUD bình thường "cho hiện đại" → gánh eventual + versioning + rebuild vô ích. Và viết `apply` không deterministic (gọi `Date.now()`/random) → replay ra state khác nhau.

### ⑥ Keywords
- 🧠 **Cần HIỂU:** event là source of truth (vs audit log); fold/replay; snapshot; CQRS tách read/write; projection eventual; khi nào KHÔNG dùng.
- 📌 **Cần THUỘC:** ES ≠ audit log; snapshot = state tại offset N; CQRS ⟂ ES; upcaster cho schema evolution.
- 🛠️ **Cần LÀM ĐƯỢC:** viết `apply` thuần/deterministic + snapshot; quyết ES có đáng không cho một domain.

### ⑦ Mental model
**"ES: lưu chuyện đã xảy ra (event bất biến), state = tua lại. Snapshot để tua nhanh. CQRS tách đọc/ghi, không bắt buộc ES. Đừng dùng ES cho CRUD — chỉ khi lịch sử/audit là yêu cầu thật."**

### ⑧ Phỏng vấn & thách đố
1. *(khó)* ES là gì, đánh đổi gì?
2. *(TB)* ES khác audit log ở điểm cốt lõi nào?
3. *(TB)* Snapshot để làm gì, khi nào cần?
4. *(khó)* CQRS là gì, vì sao hay đi với ES? CQRS có cần ES không?
5. *(rất khó)* Schema event tiến hoá xử lý sao khi event bất biến?
- 🎯 **Thách đố:** *"Team định ES hoá toàn bộ app quản lý nhân sự 'để audit'. Bạn cản hay ủng hộ?"* → Phần lớn HR là CRUD; nếu audit là yêu cầu, **audit log/temporal table** thường đủ, rẻ hơn nhiều ES (không gánh eventual/versioning/rebuild). Chỉ ES các aggregate thật sự cần time-travel/replay. Tránh over-engineering toàn cục.

### ⑨ Bài tập + tiêu chí
Cho `Wallet` (nạp/rút). Quyết: ES hay CRUD? Nếu ES, định nghĩa 3 event + hàm `apply` + chính sách snapshot.
- **Đạt khi:** lập luận ES hợp lý cho ví/tiền (audit, lịch sử số dư); event quá khứ (`Deposited`/`Withdrawn`); `apply` thuần deterministic; snapshot mỗi N event; nhắc upcaster khi đổi schema.

### ⑩ Đọc thêm
- martinfowler.com: Event Sourcing, CQRS. microservices.io. EventStoreDB docs. *(verify.)*

---

## Bài 9 — Resilience: circuit breaker, bulkhead, retry, timeout
*Phủ: K-RESIL-001 → 009*

### ① Mục tiêu & vị trí trong mạch
Async đã decouple temporal (Bài 1) nhưng **mọi call sync còn lại** (consumer→downstream, service→service) vẫn cần bảo vệ. Bài này dạy bộ công cụ chống **cascade failure**: timeout → circuit breaker → bulkhead → retry (đúng cách) → fallback. Cross-ref I-IDEM (retry cần idempotent), J (fallback cache), G-STR (backpressure).

### ② Giảng cơ bản → nâng cao

**(a) Cascade failure (📘 K-RESIL-001):** call sync qua mạng → downstream **chậm/treo** → thread/connection của caller bị **giữ chờ** → cạn pool → caller cũng treo → lan **ngược toàn chuỗi** (resource exhaustion). Latency nhỏ ở 1 service **khuếch đại** thành sự cố toàn hệ. Đây là kẻ thù chính.

**(b) Circuit breaker (📘 K-RESIL-002):** 3 trạng thái như cầu dao điện:
- **Closed** = cho qua, **đếm lỗi**. Vượt ngưỡng → chuyển **Open**.
- **Open** = **fail-fast ngay** (không gọi downstream, trả fallback) trong khoảng **cooldown** → cho downstream thời gian hồi + bảo vệ tài nguyên caller.
- **Half-open** = sau cooldown, thử **vài request**; OK → **Closed** lại; fail → **Open** tiếp.
Mục tiêu: *fail fast* thay vì *fail slow*.

**(c) Timeout là tiền đề của breaker (📘 K-RESIL-003):** **không timeout** → request "chậm vô hạn" **không bao giờ tính là lỗi** → breaker **không trip** → thread vẫn bị giữ. Timeout biến "chậm" thành **lỗi đếm được**. ⇒ **timeout + breaker + retry là bộ ba đi cùng**, không tách rời.

**(d) Bulkhead (📘 K-RESIL-004):** ẩn dụ **vách ngăn tàu thuỷ** — chia tài nguyên (thread/connection pool) thành **ngăn riêng cho từng downstream**. 1 downstream hỏng chỉ **cạn ngăn của nó**, không nuốt hết tài nguyên chung (**isolation**). Khác breaker: **bulkhead cô lập** tài nguyên, **breaker chặn** call hỏng → **bổ trợ**, dùng cùng nhau.

**(e) Retry đúng cách (📘 K-RESIL-005):** retry "ngây thơ" (retry ngay, mọi lỗi, đồng loạt) khi downstream đang ngộp → **retry storm** làm nó **sập hẳn** (cộng hưởng). Đúng: **exponential backoff** (giãn cách tăng) + **jitter** (phá đồng bộ giữa các client) + chỉ retry lỗi **transient/idempotent** + **retry budget** (giới hạn tỷ lệ retry). ⚠️ Retry **không idempotent** = **nhân đôi side-effect** (cross-ref Bài 5, I-IDEM).

**(f) Fallback & graceful degradation (📘 K-RESIL-007):** khi downstream chết, trả gì thay lỗi cứng? → **cached/stale**, giá trị mặc định, **kết quả một phần**, hoặc xếp hàng xử lý sau. **Degrade tính năng phụ** để giữ tính năng lõi ("fail soft"). Vd: gợi ý cá nhân hoá chết → trả danh sách phổ biến thay vì lỗi 500.

**(g) Anti-pattern breaker (📘 K-RESIL-006):** **quá chặt** → trip oan; **quá lỏng** → lỗi lọt; **không fallback** → vẫn lỗi tới user; **bọc call nội bộ in-process** (nhẹ) là **thừa** — breaker chỉ cho call **remote/latency cao**; **thiếu visibility** → silent failure. Hiểu cả "khi nào KHÔNG dùng".

**(h) Backpressure (📘 K-RESIL-008):** upstream nhanh hơn downstream → buffer **phình** → ngốn RAM/crash hoặc lag vô hạn. Xử lý: **pull-based** (consumer tự điều nhịp), **prefetch/concurrency limit**, **load shedding** (drop/429), queue có **giới hạn**, **scale consumer**. Broker (queue) vốn là một **buffer có kiểm soát**. *(cross-ref G-STR.)*

**(i) Async ≠ viên đạn bạc (📘 K-RESIL-009):** queue gỡ temporal coupling **nhưng KHÔNG** gỡ: consumer vẫn gọi downstream sync (cần breaker), DLQ/poison cần xử lý, lag cần giám sát, **idempotency vẫn bắt buộc**, **broker cũng có thể sập**. Resilience vẫn cần ở **từng consumer** + **biên ra ngoài**.

### ③ ⚠️ Cũ → Mới

| Cái cũ | Nay | Vì sao |
|---|---|---|
| Retry ngay, mọi lỗi | **Backoff + jitter + retry-budget**, chỉ transient/idempotent | Tránh retry storm |
| Call remote không timeout | **Timeout luôn đặt** (tiền đề breaker) | "Chậm vô hạn" = treo thread |
| Tin "có queue là hết lo resilience" | Vẫn cần breaker/DLQ/idempotent/giám sát lag | Queue chỉ gỡ temporal coupling |
| Resilience nhúng trong code mỗi service | Có thể đẩy lên **service mesh** (Bài 11) — cẩn thận retry amplification | Đồng bộ policy nhưng dễ chồng retry |

### ④ Áp dụng thực tế + bigtech
**Use case:** Checkout gọi PaymentGateway (3rd-party). Đặt **timeout 2s** + **breaker** (mở khi >50% lỗi/10s) + **bulkhead** (pool riêng cho payment) + **retry** 2 lần backoff+jitter chỉ cho lỗi mạng (idempotency-key để không charge lặp) + **fallback** "thanh toán đang xử lý, sẽ xác nhận sau". **Bigtech:** Netflix sinh ra Hystrix (nay bảo trì hạn chế) → khái niệm breaker/bulkhead phổ cập; resilience4j (JVM), **opossum** (Node) là thư viện phổ biến; nhiều hệ đẩy retry/breaker xuống mesh. *(verify thư viện/đời.)*

### ⑤ Code thực hành + cấu hình
Circuit breaker Node với **opossum** + timeout + fallback:

```javascript
// paymentClient.js
const CircuitBreaker = require('opossum'); // pin "opossum@x.y" // verify đời
async function callPayment(req) { /* gọi HTTP tới payment gateway */ }

const breaker = new CircuitBreaker(callPayment, {
  timeout: 2000,                 // (c) timeout: biến 'chậm' thành lỗi -> breaker trip được
  errorThresholdPercentage: 50,  // mở khi >50% lỗi
  resetTimeout: 10000,           // cooldown trước khi half-open
  // volumeThreshold: 10,        // tối thiểu request mới tính tỉ lệ (tránh trip oan)
});
breaker.fallback(() => ({ status: 'PENDING', note: 'thanh toán đang xử lý' })); // (f) fail soft

async function pay(req) {
  return breaker.fire(req); // Open -> fail-fast -> trả fallback ngay, không gọi downstream
}
```

```javascript
// retry backoff + jitter, CHỈ cho lỗi transient + idempotency-key (không charge lặp)
async function withRetry(fn, { tries = 3, base = 200 } = {}) {
  for (let i = 0; i < tries; i++) {
    try { return await fn(); }
    catch (e) {
      if (!isTransient(e) || i === tries - 1) throw e;      // lỗi vĩnh viễn -> không retry
      const delay = base * 2 ** i + Math.random() * base;   // exponential + jitter
      await new Promise(r => setTimeout(r, delay));
    }
  }
}
```

> ⚠️ **AI hay sai:** bọc breaker quanh hàm in-process (vô nghĩa); retry mọi exception kể cả lỗi nghiệp vụ 4xx (không bao giờ thành công + tốn tài nguyên); retry POST charge không idempotency-key → charge nhiều lần.

### ⑥ Keywords
- 🧠 **Cần HIỂU:** cascade failure; 3 trạng thái breaker; timeout là tiền đề; bulkhead vs breaker; retry storm; fallback/degradation; backpressure; queue không miễn resilience.
- 📌 **Cần THUỘC:** closed/open/half-open; backoff+jitter+budget; retry chỉ transient+idempotent; bulkhead = pool riêng.
- 🛠️ **Cần LÀM ĐƯỢC:** cấu hình breaker+timeout+fallback (opossum); viết retry backoff+jitter an toàn; nhận diện anti-pattern breaker.

### ⑦ Mental model
**"Timeout → breaker → bulkhead → retry(backoff+jitter, chỉ idempotent) → fallback. Một service chậm có thể kéo sập cả hệ; resilience là để fail FAST và fail SOFT. Queue không cứu bạn khỏi việc này."**

### ⑧ Phỏng vấn & thách đố
1. *(TB)* Cascade failure xảy ra thế nào?
2. *(TB)* 3 trạng thái circuit breaker?
3. *(khó)* Vì sao timeout là tiền đề của breaker?
4. *(khó)* Bulkhead khác breaker, vì sao dùng cùng?
5. *(khó)* Retry ngây thơ hại gì, backoff+jitter+budget cứu sao?
- 🎯 **Thách đố:** *"Sự cố: downstream chậm 5s, toàn hệ sập dù chỉ 1 service lỗi. Trace cho thấy hàng nghìn retry. Hai nguyên nhân gốc và cách chặn?"* → (1) **không timeout** → thread bị giữ → cạn pool → cascade; (2) **retry không backoff/budget** → retry storm dìm downstream. Chặn: đặt timeout + breaker (fail-fast) + bulkhead (cô lập pool) + retry backoff+jitter+budget.

### ⑨ Bài tập + tiêu chí
Bọc một call tới `recommendationService` (không lõi) với resilience đầy đủ; khi nó chết, trang vẫn render.
- **Đạt khi:** timeout + breaker + fallback trả danh sách "phổ biến" (graceful degradation); retry chỉ lỗi transient; nêu vì sao KHÔNG retry cho lỗi 4xx; (tuỳ chọn) bulkhead pool riêng.

### ⑩ Đọc thêm
- *Release It!* (Michael Nygard) — cascade/breaker/bulkhead kinh điển. resilience4j docs; opossum (npm). *(verify.)*

---

## Bài 10 — Service Decomposition & bounded context
*Phủ: K-DECOMP-001 → 009*

### ① Mục tiêu & vị trí trong mạch
Đây là **gốc rễ** sinh ra mọi bài trên: chính việc **tách service** (database-per-service) tạo nhu cầu Saga/Outbox/event. Đặt **sau** phần cơ chế có chủ đích — để khi bàn "tách sao cho đúng", bạn đã biết "tách xong đồng bộ dữ liệu kiểu gì". Học xong bạn tránh được **distributed monolith** và **nano-service**. Cross-ref Q (DDD), A (system design).

### ② Giảng cơ bản → nâng cao

**(a) Microservice thật sự là gì (📘 K-DECOMP-001):** KHÔNG phải "service nhỏ" hay "dùng Docker". Là service **tự trị quanh một business capability / bounded context**, **deploy độc lập**, **sở hữu dữ liệu riêng** (no shared DB), giao tiếp qua API/event, có thể khác stack. Trọng tâm: **ranh giới + autonomy**, không phải kích thước/công cụ.

**(b) Tách theo gì (📘 K-DECOMP-002, 003):** theo **business capability / bounded context (DDD)** + theo volatility/đội sở hữu, sao cho mỗi service **đổi độc lập**. 
- **Tách theo tầng kỹ thuật (UI/logic/DB) là SAI** → mọi feature đụng nhiều service (coupling cao, deploy lệ thuộc) = **distributed monolith**.
- **Bounded context** = phạm vi một mô hình ngôn ngữ nhất quán (cùng từ "Customer" nghĩa khác ở Sales vs Billing). Biên service **trùng** bounded context → ít coupling ngữ nghĩa, mỗi đội sở hữu mô hình của mình.

**(c) Database-per-service (📘 K-DECOMP-004):** vì sao **shared DB là anti-pattern**: coupling schema (đổi bảng vỡ nhiều service), không deploy độc lập, khoá chung. Cái giá khi tách DB: **mất JOIN/transaction xuyên service** → cần **API composition / Saga / event** để đồng bộ + chấp nhận **eventual**. Hiểu cả lợi (autonomy) **và** giá.

**(d) Hai anti-pattern granularity:**
- **Distributed monolith (📘 K-DECOMP-005):** nhiều service nhưng **coupling chặt** (deploy phải đồng bộ, sync call dây chuyền, shared DB/lib, đổi 1 phải đổi nhiều) → gánh **chi phí phân tán** (latency/phức tạp/failure) mà **KHÔNG** được lợi autonomy. Dấu hiệu: *"không deploy 1 service mà không deploy service khác"*. **Tệ hơn cả monolith.**
- **Nano-service (📘 K-DECOMP-007):** quá mịn → 1 thao tác nghiệp vụ gọi **nhiều service** (chatty, latency cộng dồn, transaction phân tán khắp nơi, vận hành nặng). Dấu hiệu: nhiều service **luôn đổi cùng nhau / luôn gọi nhau đồng bộ** → nên **gộp lại**. Granularity là **trade-off**, không "càng nhỏ càng tốt".

**(e) Khi service A cần dữ liệu của B (📘 K-DECOMP-006, 008):**

| Cách | Bản chất | Được | Mất |
|---|---|---|---|
| **Sync call B** (API composition) | gọi runtime | đơn giản, **fresh** | A phụ thuộc B sống, +latency, cascade |
| **Data replication qua event** | A giữ bản sao local từ event của B | decouple, nhanh, chịu lỗi | **eventual**, trùng dữ liệu, phức tạp |
| **CQRS read model** | read model riêng xuyên service | query nhanh | thêm hạ tầng, eventual |

Chọn theo **độ tươi cần thiết + tần suất + độ chịu lỗi**. **Không có default.**

**(f) Di trú monolith → microservices (📘 K-DECOMP-009):** **Strangler Fig** — bọc monolith, dần tách **từng capability** ra service mới + route traffic dần (qua gateway/proxy), giữ monolith chạy **song song** tới khi rút hết. Vì sao không **big-bang rewrite**: rủi ro cao, lâu, dễ thất bại (mất tính năng, không có đường lùi). *(verify tên pattern.)*

### ③ ⚠️ Cũ → Mới

| Cái cũ | Nay | Vì sao |
|---|---|---|
| "Microservice = service nhỏ/Docker" | Tự trị + DB riêng + deploy độc lập quanh **bounded context** | Trọng tâm là ranh giới |
| Tách theo tầng kỹ thuật (UI/logic/DB) | Tách theo **business capability/DDD** | Tránh distributed monolith |
| Shared database cho tiện JOIN | **Database-per-service** + composition/Saga/event | Autonomy + deploy độc lập |
| "Càng nhiều service càng tốt" | Granularity là trade-off; gộp khi nano | Tránh chatty/transaction phân tán |
| Big-bang rewrite monolith | **Strangler Fig** từng phần | Giảm rủi ro *(verify)* |

### ④ Áp dụng thực tế + bigtech
**Use case:** e-commerce tách `Catalog`, `Order`, `Inventory`, `Payment`, `Shipping` theo bounded context, mỗi cái DB riêng; Order cần thông tin sản phẩm → giữ **bản sao tối thiểu** (id, tên, giá tại thời điểm đặt) qua event `ProductUpdated` thay vì gọi Catalog mỗi lần. **Bigtech:** Amazon/Netflix nổi tiếng database-per-service + tách theo capability; Strangler Fig được dùng rộng để thoát monolith dần. *(pattern; verify.)*

### ⑤ Code thực hành + cấu hình
Không phải bài code — là bài thiết kế. Minh hoạ **API composition** vs **replication**:

```javascript
// (A) API composition: Order gọi Catalog runtime (fresh nhưng coupling + latency + cascade)
async function getOrderView(orderId) {
  const order = await orderRepo.find(orderId);
  const product = await catalogClient.get(order.productId); // phụ thuộc Catalog sống -> bọc resilience (Bài 9)!
  return { ...order, productName: product.name };
}

// (B) Replication qua event: Order giữ snapshot product tại thời điểm đặt (decouple, eventual)
//   - subscribe 'ProductUpdated' -> upsert vào bảng product_replica của Order (idempotent - Bài 5)
//   - getOrderView đọc local, KHÔNG gọi Catalog -> không cascade, nhưng dữ liệu có thể trễ
```

> ⚠️ **AI hay sai:** mặc định API composition (gọi nhau sync) khắp nơi → distributed monolith + cascade; hoặc shared DB "cho nhanh" → mất autonomy. Tech Lead phải hỏi: *độ tươi cần bao nhiêu?* trước khi chọn.

### ⑥ Keywords
- 🧠 **Cần HIỂU:** microservice = autonomy + bounded context + DB riêng; distributed monolith; nano-service; sync vs replication; Strangler Fig.
- 📌 **Cần THUỘC:** tách theo capability không theo tầng; shared DB là anti-pattern; 3 cách share data; dấu hiệu distributed monolith ("không deploy độc lập").
- 🛠️ **Cần LÀM ĐƯỢC:** vẽ ranh giới service theo bounded context; chọn cách share data theo độ tươi; nhận diện over/under-decomposition.

### ⑦ Mental model
**"Service = bounded context tự trị, DB riêng, deploy độc lập. Tách theo nghiệp vụ, không theo tầng. Quá chặt = distributed monolith (tệ hơn monolith); quá mịn = nano (chatty). Thoát monolith bằng Strangler, không big-bang."**

### ⑧ Phỏng vấn & thách đố
1. *(dễ)* Điều gì *thật sự* định nghĩa microservice?
2. *(khó)* Tách theo tiêu chí gì? Vì sao tách theo tầng kỹ thuật sai?
3. *(rất khó)* Shared DB là anti-pattern thế nào, cái giá khi tách DB?
4. *(rất khó)* Distributed monolith — dấu hiệu, vì sao tệ hơn monolith?
5. *(khó)* A cần dữ liệu B — sync vs event, chọn sao?
- 🎯 **Thách đố:** *"Hệ có 30 service nhưng mỗi lần release phải deploy đồng loạt, và một thao tác checkout gọi 12 service đồng bộ. Chẩn đoán?"* → **distributed monolith** (deploy lệ thuộc) + **nano/chatty** (12 sync call). Gốc: tách sai (theo tầng / quá mịn / shared lib-DB). Sửa: gộp theo bounded context, thay sync-chain bằng event/replication, làm deploy độc lập được.

### ⑨ Bài tập + tiêu chí
Cho monolith "đặt đồ ăn" (user, nhà hàng, món, đơn, thanh toán, giao hàng). Đề xuất ranh giới service + chọn share-data cho "Đơn cần tên món & giá".
- **Đạt khi:** ranh giới theo bounded context (không theo tầng); DB riêng mỗi service; "Đơn" giữ **snapshot món/giá tại thời điểm đặt** qua event (giải thích vì sao không gọi sync mỗi lần); nêu Strangler nếu tách dần từ monolith.

### ⑩ Đọc thêm
- *Building Microservices* (Sam Newman); *Monolith to Microservices* (Strangler). DDD (Eric Evans) cho bounded context. microservices.io. *(verify.)*

---

## Bài 11 — API Gateway, Service Mesh & Service Discovery
*Phủ: K-GW-001 → 008*

### ① Mục tiêu & vị trí trong mạch
Sau khi đã tách service (Bài 10), bài này lo **topology nối chúng lại**: client vào hệ thống qua đâu (**gateway**), service tìm nhau thế nào (**discovery**), giao tiếp nội bộ an toàn/quan sát được nhờ gì (**service mesh**). Nhìn gateway ở **góc topology** (không lặp F-TL "god component"). Cross-ref F (gateway tổng quát), M (mTLS/zero-trust), K-RESIL (breaker ở hạ tầng).

### ② Giảng cơ bản → nâng cao

**(a) Service discovery (📘 K-GW-001):** service **scale/đổi IP động** (container lên/xuống) → **không hardcode địa chỉ**. Instance **đăng ký** vào **registry** (Consul/Eureka/etcd...), bên gọi **tra cứu**:
- **Client-side discovery** = client tự hỏi registry rồi tự chọn instance (tự load-balance).
- **Server-side discovery** = gateway/LB hỏi registry giúp, client chỉ gọi một địa chỉ ổn định.
*(verify tên công cụ; trong Kubernetes, DNS + Service object làm phần lớn việc này.)*

**(b) API Gateway — góc topology (📘 K-GW-002):** điểm vào **north-south** (client ↔ hệ thống): **routing** tới service, **aggregation** (gộp nhiều call thành một response), **auth/rate-limit/TLS** tập trung, **che cấu trúc nội bộ** khỏi client. Cross-ref F-TL cho cảnh báo gateway **phình to** (đừng nhồi business logic). Trọng tâm K: **vị trí** gateway trong sơ đồ.

**(c) North-south vs east-west (📘 K-GW-003):**
- **North-south** = client ↔ hệ thống → do **API gateway** lo (biên ngoài).
- **East-west** = service ↔ service nội bộ → do **service mesh** (hoặc gọi trực tiếp) lo: mTLS, retry, timeout, traffic shaping nội bộ. **Phân vai rõ**: gateway biên ngoài, mesh nội bộ.

**(d) Service mesh (📘 K-GW-004):** lớp **hạ tầng** cho giao tiếp service-to-service. Mô hình **sidecar**: một proxy (Envoy) chạy **cạnh mỗi pod**, lo **mTLS, retry, timeout, circuit breaking, traffic split, telemetry** — **không cần sửa code** service. Control plane (vd Istiod) cấu hình các proxy. Đổi lại: **độ phức tạp vận hành + latency proxy**.

**(e) BFF — Backend for Frontend (📘 K-GW-005):** gateway/aggregation **riêng cho từng loại client** (web/mobile/3rd-party) → tối ưu payload & số call theo nhu cầu client, tránh 1 gateway "one-size-fits-all" phình to; mỗi đội frontend **sở hữu BFF của mình**. Trade-off: **trùng lặp logic** giữa các BFF.

**(f) mTLS & zero-trust nội bộ (📘 K-GW-006):** xác thực & mã hoá **hai chiều** giữa service → **không tin mạng nội bộ** mặc định (zero-trust); mesh **tự cấp/luân chuyển cert** + áp mTLS **không cần sửa app**; chống **lateral movement** khi 1 service bị chiếm. *(cross-ref M.)*

**(g) Breaker/retry ở hạ tầng — lợi & bất ngờ (📘 K-GW-007):** lợi: **đồng bộ policy resilience** không sửa code. Bất ngờ: **retry ở mesh + retry ở app = nhân số lần gọi** (**retry amplification**); hoặc breaker mesh trip mà app **không biết** → hành vi khó hiểu. Cần **phối hợp policy app↔mesh**, tránh chồng retry. *(cross-ref K-RESIL-005.)*

**(h) Mesh không miễn phí (📘 K-GW-008):** thêm **sidecar** (latency, RAM/CPU mỗi pod), **control plane** phải vận hành, đường cong học **dốc**, **debug khó hơn**. Chỉ đáng khi **nhiều service + cần mTLS/observability/traffic-mgmt thống nhất**; hệ nhỏ → **thư viện resilience (vd opossum) + gateway** là đủ. *(verify.)*

### ③ ⚠️ Cũ → Mới *(cập nhật quan trọng 2025–2026)*

| Cái cũ | Nay | Vì sao |
|---|---|---|
| Hardcode IP/host downstream | **Service discovery** (registry/DNS); trong k8s: Service + DNS | IP động khi scale |
| **Sidecar mesh** (Envoy mỗi pod) là mặc định | **Istio ambient (GA 2025)**: `ztunnel` per-node L4 + `waypoint` per-namespace L7; **Cilium eBPF** (kernel); Linkerd giữ sidecar Rust nhẹ | Sidecar tốn RAM/latency mỗi pod → "kỷ nguyên sidecar đang kết thúc" *(verify)* |
| Một gateway dùng chung mọi client | **BFF** per client khi nhu cầu khác nhau | Tránh gateway phình + payload thừa |
| Mạng nội bộ "tin được" | **Zero-trust + mTLS** (mesh tự áp) | Chống lateral movement |
| Nhồi resilience/auth vào code mỗi service | Đẩy lên **mesh/gateway** (cẩn thận retry amplification) | Đồng bộ policy *(verify)* |

> 📌 **Lưu ý cập nhật:** nếu đọc bài "Istio vs Linkerd" trước giữa-2025, nó bỏ qua **ambient mode** — thay đổi lớn nhất gần đây. Khi học verify: ztunnel (L4 mTLS, DaemonSet) + waypoint (L7 Envoy, optional). Linkerd đổi mô hình license (Buoyant subscription cho stable). Cilium đẩy mesh vào kernel qua eBPF (không sidecar). *(verify từng cái.)*

### ④ Áp dụng thực tế + bigtech
**Use case:** app mobile + web gọi qua **2 BFF** riêng → BFF route vào các service nội bộ; nội bộ chạy **mesh** áp mTLS + telemetry tự động; service discovery qua k8s DNS. **Bigtech:** Istio/Linkerd phổ biến ở hệ k8s lớn; nhiều hệ đang **dịch sang ambient/eBPF** để giảm chi phí sidecar; BFF do Netflix/SoundCloud phổ biến hoá. *(pattern; verify số liệu/đời.)*

### ⑤ Code thực hành + cấu hình
Mesh chủ yếu là **cấu hình hạ tầng**, không phải code app. Ví dụ ý niệm (verify CRD theo đời Istio):

```yaml
# (ý niệm) bật mTLS strict cho namespace qua mesh — KHÔNG sửa code service
apiVersion: security.istio.io/v1
kind: PeerAuthentication
metadata: { name: default, namespace: payments }
spec: { mtls: { mode: STRICT } }   # mọi giao tiếp east-west trong ns này phải mTLS // verify apiVersion
```

```yaml
# (ý niệm) retry ở mesh — CẢNH BÁO: nếu app cũng retry -> amplification
apiVersion: networking.istio.io/v1
kind: VirtualService
metadata: { name: catalog }
spec:
  http:
    - route: [{ destination: { host: catalog } }]
      retries: { attempts: 2, perTryTimeout: 1s }  # phối hợp với app: chỉ retry ở MỘT tầng // verify
```

```bash
# Ambient mode (ý niệm) — không sidecar, dùng ztunnel/waypoint // verify lệnh theo đời
istioctl install --set profile=ambient
kubectl label namespace payments istio.io/dataplane-mode=ambient
```

> ⚠️ **AI hay sai:** bật retry ở cả mesh và code → 2×2 = 4 lần gọi khi lỗi (amplification dìm downstream); hoặc khuyên dựng full Istio cho hệ 3 service (over-engineering — dùng opossum + 1 gateway là đủ).

### ⑥ Keywords
- 🧠 **Cần HIỂU:** discovery (client vs server-side); north-south vs east-west; sidecar vs ambient/eBPF; BFF; zero-trust/mTLS; retry amplification; chi phí mesh.
- 📌 **Cần THUỘC:** gateway = north-south; mesh = east-west; sidecar (Envoy) lo cross-cutting; ztunnel(L4)+waypoint(L7); mesh chỉ đáng khi nhiều service.
- 🛠️ **Cần LÀM ĐƯỢC:** quyết có cần mesh không; phân vai gateway/mesh; tránh chồng retry app↔mesh.

### ⑦ Mental model
**"Gateway = cửa trước (north-south), mesh = đường nội bộ (east-west, mTLS/retry/telemetry không sửa code). Discovery thay hardcode IP. Mesh mạnh nhưng đắt — kỷ nguyên sidecar đang nhường chỗ ambient/eBPF. Hệ nhỏ: thư viện + gateway là đủ."**

### ⑧ Phỏng vấn & thách đố
1. *(TB)* Service discovery giải gì? Client vs server-side?
2. *(khó)* North-south vs east-west — gateway lo gì, mesh lo gì?
3. *(khó)* Service mesh & sidecar làm gì? Vì sao tách cross-cutting khỏi code?
4. *(khó)* BFF là gì, khi nào cần?
5. *(rất khó)* Mesh không miễn phí — chi phí gì khiến không phải hệ nào cũng nên dùng?
- 🎯 **Thách đố:** *"Sau khi cài Istio, p99 latency tăng và thỉnh thoảng downstream bị dìm dù traffic không tăng. Hai nghi phạm?"* → (1) **sidecar overhead** (Envoy mỗi pod thêm latency/RAM — cân nhắc ambient/eBPF); (2) **retry amplification** (mesh retry + app retry chồng nhau → nhân số request lên downstream). Sửa: gỡ retry ở một tầng; cân nhắc ambient mode.

### ⑨ Bài tập + tiêu chí
Hệ 4 service trên k8s cần: mTLS nội bộ, telemetry, web+mobile payload khác nhau. Đề xuất: discovery, gateway/BFF, có nên mesh không (và mode nào).
- **Đạt khi:** discovery qua k8s DNS/Service; **BFF** riêng web/mobile; cân nhắc mesh cho mTLS+telemetry nhưng nêu chi phí — gợi ý **ambient** để giảm overhead với hệ vừa, hoặc thư viện resilience nếu chỉ 4 service; tránh chồng retry.

### ⑩ Đọc thêm
- istio.io (Ambient mesh, dataplane modes), linkerd.io, cilium.io. Sam Newman về BFF/gateway. *(verify đời — đổi rất nhanh.)*

---

## Bài 12 — Distributed Tracing & Observability
*Phủ: K-OBS-001 → 006*

### ① Mục tiêu & vị trí trong mạch
Bài chốt Mục K: khi request đi xuyên nhiều service + qua broker (Bài 1–11), **làm sao nhìn xuyên toàn hệ** để debug? Đây cũng là **baseline #4 của Saga** (observability). Cross-ref O (logging/metrics tổng quát).

### ② Giảng cơ bản → nâng cao

**(a) Vì sao khó hơn monolith (📘 K-OBS-001):** 1 request đi qua **nhiều service/queue** → log **rải rác** khắp nơi, không thấy **đường đi end-to-end**. **Distributed tracing** gắn một **trace id** xuyên suốt + chia thành **span** mỗi chặng → dựng lại **timeline** + tìm chặng **chậm/lỗi**. Không có nó = "mò kim đáy bể".

**(b) Trace / span / context propagation (📘 K-OBS-002):**
- **Trace** = toàn bộ hành trình của **1 request**.
- **Span** = **1 đơn vị công việc** (có parent/child, thời lượng, tags). Trace = cây span.
- **Context propagation** = truyền **trace/span id** qua biên: 
  - **HTTP** → **W3C Trace Context** header `traceparent` (chuẩn). 
  - **Message broker** → đính trace context vào **message header** (Kafka headers / AMQP headers) để **nối chặng async**.
- ⚠️ **Thiếu propagation qua broker → trace ĐỨT ở queue** (mỗi bên thành trace riêng). Đây là chỗ AI/dev hay quên: chỉ instrument HTTP mà bỏ broker. *(verify W3C Trace Context.)*

**(c) Correlation ID vs trace id (📘 K-OBS-003):**
- **Correlation/request id** = gắn vào **mọi log** của cùng một luồng nghiệp vụ → `grep` ra toàn bộ hành trình trong log.
- **Trace id** (tracing) + **correlation id** (logging) **bổ trợ** nhau, không thay nhau.
- Cần **structured logging** (JSON + field id) để **truy vấn/aggregate** (không phải log text thuần). *(cross-ref O.)*

**(d) OpenTelemetry (📘 K-OBS-004):** **OTel** = chuẩn + SDK **trung lập** (CNCF) để sinh/truyền **trace–metric–log**, **export tới backend bất kỳ** (Jaeger/Tempo/Zipkin/Datadog...) → **tránh khoá vendor** ("mỗi vendor một SDK"). **Jaeger/Zipkin/Tempo** = backend **lưu/hiển thị** trace. OTel **tách instrumentation khỏi backend**. *(verify.)*

**(e) Sampling (📘 K-OBS-005):** trace **100%** = chi phí lưu/băng thông lớn → **sample**:
- **Head-based** = quyết **ngay đầu** request (rẻ, đơn giản) — nhưng có thể **bỏ lỡ** trace lỗi (vì chưa biết nó sẽ lỗi).
- **Tail-based** = quyết **sau** khi thấy cả trace (giữ được trace **lỗi/chậm**) — nhưng **tốn buffer** (phải giữ span tới khi trace xong). Trade-off **cost ↔ giữ được trace quan trọng**. *(verify.)*

**(f) Three pillars (📘 K-OBS-006):** mỗi cái trả lời câu hỏi khác:
- **Metrics** = *"có gì bất thường, ở đâu"* (rate/error/latency — **RED**; hoặc **USE** cho tài nguyên). Rẻ, aggregate, dùng để **alert**.
- **Traces** = *"request đi đâu chậm/lỗi"* (khoanh **service/chặng** nào).
- **Logs** = *"chi tiết vì sao"* tại chặng đó.
**Quy trình khi alert nổ:** **metric phát hiện → trace khoanh vùng → log soi chi tiết**. Hiểu vai trò từng pillar, không lẫn lộn.

### ③ ⚠️ Cũ → Mới

| Cái cũ | Nay | Vì sao |
|---|---|---|
| Log text rải rác, grep từng service | **Structured logging + correlation id + tracing** | Dựng được hành trình end-to-end |
| Mỗi vendor một SDK instrumentation | **OpenTelemetry** (trung lập) export tới backend bất kỳ | Tránh vendor lock-in *(verify)* |
| Chỉ trace HTTP | **Propagate context qua broker** (message header) | Trace không đứt ở queue |
| Trace 100% | **Sampling** (head/tail tuỳ nhu cầu) | Cân chi phí ↔ giữ trace lỗi |

### ④ Áp dụng thực tế + bigtech
**Use case:** request checkout đi API→Order→(Kafka)→Inventory→(Kafka)→Payment. Inject `traceparent` vào HTTP **và** Kafka headers → một trace liền mạch trong Jaeger/Tempo cho thấy chặng Payment chậm 3s. **Bigtech:** OTel là chuẩn de-facto, hầu hết hệ lớn instrument bằng OTel rồi export tới backend (Tempo/Jaeger/Datadog); tail-based sampling dùng khi cần chắc chắn bắt trace lỗi. *(pattern; verify.)*

### ⑤ Code thực hành + cấu hình
Propagate trace context qua Kafka (ý niệm OTel JS — verify API theo đời SDK):

```javascript
// producer: inject context hiện tại vào message headers (để trace KHÔNG đứt qua broker)
const { propagation, context, trace } = require('@opentelemetry/api'); // verify đời
const headers = {};
propagation.inject(context.active(), headers, {                 // bơm traceparent vào headers
  set: (carrier, k, v) => { carrier[k] = v; },
});
await producer.send({ topic: 'orders', messages: [{ key: order.id, value: JSON.stringify(evt), headers }] });
```

```javascript
// consumer: extract context từ headers -> span con nối đúng trace cha
await consumer.run({ eachMessage: async ({ message }) => {
  const parentCtx = propagation.extract(context.active(), message.headers, {
    get: (carrier, k) => carrier[k]?.toString(),
  });
  const span = trace.getTracer('inventory').startSpan('handle OrderPlaced', {}, parentCtx);
  try { await handle(JSON.parse(message.value)); }
  finally { span.end(); } // span này nối vào trace bắt đầu từ HTTP request gốc
}});
```

```
# requirements/deps (verify đời mới nhất):
#   @opentelemetry/api, @opentelemetry/sdk-node, @opentelemetry/auto-instrumentations-node
#   Backend: Jaeger / Grafana Tempo. Log: structured JSON + correlation id.
```

> ⚠️ **AI hay sai:** chỉ auto-instrument HTTP rồi tưởng "có tracing" — nhưng **trace đứt ở Kafka** vì không inject/extract qua message header. Và log không gắn correlation/trace id → không nối log với trace được.

### ⑥ Keywords
- 🧠 **Cần HIỂU:** vì sao tracing cần ở microservice; trace/span/propagation (HTTP & broker); 3 pillars + thứ tự khoanh vùng; OTel tách instrumentation/backend; sampling head vs tail.
- 📌 **Cần THUỘC:** `traceparent` (W3C); trace id vs correlation id; metrics→trace→log; head-based (rẻ, lỡ lỗi) vs tail-based (giữ lỗi, tốn buffer).
- 🛠️ **Cần LÀM ĐƯỢC:** propagate context qua broker (inject/extract); structured logging + correlation id; chọn sampling.

### ⑦ Mental model
**"Trace id xuyên suốt + span mỗi chặng = bản đồ một request. Đừng để trace đứt ở queue (propagate qua header). Metric phát hiện → trace khoanh vùng → log soi chi tiết. OTel để khỏi khoá vendor."**

### ⑧ Phỏng vấn & thách đố
1. *(TB)* Vì sao debug request ở microservices khó hơn monolith? Tracing giải gì?
2. *(khó)* Trace/span/context propagation — context truyền qua HTTP và broker thế nào?
3. *(TB)* Correlation id khác trace id? Vì sao cần structured logging?
4. *(khó)* OpenTelemetry giải bài "mỗi vendor một SDK" thế nào? Quan hệ Jaeger/Tempo?
5. *(khó)* Head-based vs tail-based sampling đánh đổi gì?
- 🎯 **Thách đố:** *"Đã cài OTel auto-instrumentation, dashboard đẹp, nhưng mọi trace 'kết thúc' ở chỗ publish Kafka và service consumer có trace riêng rời rạc. Vì sao và sửa?"* → **không propagate trace context qua message header** → consumer tạo trace mới thay vì span con. Sửa: inject `traceparent` vào Kafka headers ở producer, extract + startSpan với parentCtx ở consumer.

### ⑨ Bài tập + tiêu chí
Thiết kế observability cho luồng `API → Order → Kafka → Inventory`: nêu trace propagation, correlation id trong log, 1 metric + 1 alert, và quy trình khoanh vùng khi alert nổ.
- **Đạt khi:** inject/extract context qua Kafka header (trace liền mạch); log JSON có trace/correlation id; metric RED (vd error rate / p99 latency) + alert ngưỡng; quy trình metric→trace→log; (tuỳ chọn) nêu sampling.

### ⑩ Đọc thêm
- opentelemetry.io; w3.org/TR/trace-context; grafana.com/oss/tempo; jaegertracing.io. *Distributed Systems Observability* (Cindy Sridharan). *(verify.)*

---

# ✅ Tổng kết Mục K & gợi ý dùng tiếp

## Bản đồ 1 dòng cho mỗi bài (ôn nhanh)
1. **WHY** — async = decouple/buffer/bền/fan-out; đổi strong lấy eventual.
2. **BRK** — Kafka(log/replay) vs RabbitMQ(queue/routing) vs NATS vs SQS-SNS; chọn theo mô hình.
3. **KAFKA-I** — partition = song song+ordering; key→partition; commit sau khi xử lý.
4. **KAFKA-II** — RF3/min.insync2/acks=all; rebalance đang incremental; retention vs compaction; EOS chỉ trong Kafka; lag là nhịp tim.
5. **DELIV** — at-least-once mặc định ⇒ idempotent consumer; ordering per-key; retry-topic/DLQ.
6. **OBX** — đừng dual-write; Outbox (1 tx) + CDC(Debezium); Inbox dedup; 2PC chết.
7. **SAGA** — local tx + compensation(≠rollback); orchestration vs choreography; baseline = Outbox+idempotent+compensation+observability.
8. **ES/CQRS** — event là source of truth; snapshot; CQRS⟂ES; đừng over-engineer.
9. **RESIL** — timeout→breaker→bulkhead→retry(backoff+jitter, idempotent)→fallback.
10. **DECOMP** — bounded context + DB riêng; tránh distributed monolith & nano; Strangler.
11. **GW/MESH** — gateway(north-south) vs mesh(east-west); sidecar→ambient/eBPF; tránh retry amplification.
12. **OBS** — trace id + span; propagate qua broker; metric→trace→log; OTel.

## Lịch spaced repetition đề xuất
- **Hôm nay:** Bài 5 (delivery/idempotent) + Bài 6 (Outbox) — lõi reliability, hay quên nhất.
- **Mai:** Bài 3–4 (Kafka) + Bài 7 (Saga).
- **3 ngày:** Bài 9 (resilience) + Bài 10 (decomposition).
- **1 tuần:** quét lại toàn bộ 12 mental model + làm Bước D (phỏng vấn live 94 câu).

## Bước tiếp theo (ngoài tài liệu này)
- **Bước D — Phỏng vấn live:** mở `QB_K_messaging.md`, mỗi đợt 10–15 câu trộn dễ→khó; tôi hỏi, **bạn trả lời trước**, tôi chấm theo dòng *"dò cái gì"* + chèn 1–2 câu cũ (spaced repetition).
- **Bước E — Cổng hoàn thành:** ĐẠT toàn bộ 94 câu mới tính xong Mục K.

> ⚠️ **Nhắc lại — verify khi thực hành:** config Kafka (`acks`, `min.insync.replicas`, `enable.idempotence`, `group.protocol`, transactional API), KIP-848/932/890, RabbitMQ exchange/confirms, SQS FIFO limits, Debezium 3.4.x (Kafka 4.x, EOS), Temporal/Camunda 8, Istio ambient (ztunnel/waypoint)/Cilium/Linkerd, opossum, W3C Trace Context, OpenTelemetry. **Tên/đời/cấu hình đổi nhanh — luôn tra docs chính thức.**

> 🧭 Câu thần chú: *"Tôi không nhớ để gõ, tôi hiểu để chỉ huy — và tra cứu phần còn lại."*
