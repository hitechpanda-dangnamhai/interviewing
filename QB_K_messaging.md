# 🧪 QB_K — Ngân hàng câu hỏi: Messaging & Microservices

> **Mục K** trong roadmap Tech Lead Backend (🟡 ~40%) · Sinh theo **WORKFLOW 2 — Bước A**.
> Đây là **mục LỚN (A–R)** → bộ đề phủ HẾT 9 mục con của roadmap (vì sao message queue · at-least-once/idempotent consumer · Saga · Outbox · Event Sourcing · Circuit breaker · Service decomposition · API gateway · Distributed tracing) + mở rộng đúng chiều sâu TL: broker landscape (Kafka/RabbitMQ/NATS/SQS-SNS), Kafka internals, CDC, service mesh/discovery, "khi nào KHÔNG microservice".
> **Chỉ câu hỏi — KHÔNG kèm đáp án.** Mỗi câu có dòng *"dò cái gì"* = tiêu chí ĐẠT, dùng để chấm **live** ở Bước D.
> **Chống trùng:** đã đọc `QB_E_database.md`, `QB_F_apidesign.md`, `QB_G_nodejs.md`, `QB_H_nestjs.md`, `QB_I_concurrency.md`, `QB_J_caching.md`, `QB_L_testing.md`, `QB_M_security.md`, `QB_Q_designpatterns.md`.
> - **I-IDEM** đã phủ idempotency từ *góc concurrency/distributed* (at-least-once → idempotent, exactly-once *delivery* là ảo tưởng → effectively-once, naturally-idempotent vs phải-làm-idempotent, parallel retry cùng key, link idempotency↔optimistic↔outbox) → K-DELIV **không lặp lý thuyết nền**; K-DELIV chỉ chạm idempotency ở **góc cơ chế messaging** (dedup store theo messageId, Kafka EOS/transactions, ack vs offset commit) + **cross-ref I-IDEM**.
> - **F-IDEM** giữ `Idempotency-Key` HTTP/method semantics (RFC 9110); **F-TL** giữ vai trò gateway tổng quát + webhook delivery → K-GW nhìn gateway ở **góc topology microservice** (gateway vs service mesh, north-south vs east-west, service discovery, BFF), cross-ref F; không lặp.
> - **J-QUEUE** giữ BullMQ/Redis Streams + DLQ Redis + "khi nào không đủ → Kafka/RabbitMQ" (góc Node/Redis) → K-BRK sở hữu **so sánh broker sâu + chọn broker**; K không lặp wiring BullMQ.
> - **H-MS** giữ `@nestjs/microservices` (transport TCP/Redis/NATS/RabbitMQ/Kafka/gRPC, `@MessagePattern` vs `@EventPattern`, RpcException, hybrid app); **H-CQRS** giữ `@nestjs/cqrs` Saga/ES *kiểu Nest* → K-SAGA/K-ES/K-DELIV ở mức **kiến trúc/pattern không phụ thuộc framework**, cross-ref H khi đụng hiện thực Nest.
> - **G-STR** giữ backpressure *Node stream-level* → K-RESIL chạm backpressure/consumer-lag ở **góc messaging**, cross-ref G.
> - **A (System Design)** sẽ *ráp* các pattern này vào bài thiết kế lớn → K giữ ở mức **pattern + trade-off đơn lẻ**, không lặp bài design tổng.
> **Tổng: 94 câu.**

## Cách đọc
- **ID:** `K-<tag>-<số>`. Tag mục con liệt kê ở mục lục dưới.
- **Độ khó:** ⭐ định nghĩa cơ bản → ⭐⭐⭐⭐⭐ phân biệt senior với TL thật.
- **"Dò cái gì":** điều người trả lời PHẢI chạm tới mới tính ĐẠT (không phải đáp án — chỉ là mốc chấm).
- *(verify)* = câu chạm cú pháp/tên/đời/cấu hình sản phẩm (config Kafka, tên SDK, default partitioner, version Debezium/Temporal...) → **kiểm chứng tại docs chính thức** khi học (đổi nhanh).

## Bối cảnh (tính tại 6/2026)
- **Phần lý thuyết ổn định** (ít đổi): sync vs async coupling, delivery semantics (at-most/at-least/exactly-once), Saga (orchestration↔choreography), Outbox/CDC, Event Sourcing, resilience (circuit breaker/bulkhead/retry/timeout), decomposition theo bounded context. Đây là phần phỏng vấn đào sâu *trade-off* nhất.
- **Phần dễ đổi → verify khi học:** config & default của broker (Kafka `enable.idempotence`, `acks`, transactional API; RabbitMQ exchange types; SQS FIFO vs standard), tên/đời công cụ (Debezium, Temporal/Camunda, Kong, Istio/Linkerd, opossum), API `@nestjs/microservices`.
- **Câu chốt hay bị vặn (Mục 9 roadmap):** "distributed transaction = 2PC" → **Saga + Outbox**; "publish event ngay trong transaction code" → **Outbox + CDC (Debezium) → Kafka**; "polling DB để đồng bộ" → **CDC/event-driven**. Đây là các bẫy "cũ → hiện đại" hay xuất hiện.
- **Quan điểm nền cho mọi câu Kafka:** "Kafka is available, sometimes consistent" — câu hỏi thực tế thường là *consumer* chết (không phải Kafka chết); exactly-once *processing* cần idempotent producer + transactions (KHÔNG bật mặc định, có overhead). *(verify config.)*
- **Câu thần chú áp dụng cho mục này:** AI viết đúng cú pháp publish/consume nhưng dễ **sai kiến trúc** — ví dụ publish event *trong* transaction code (mất event khi commit DB ok nhưng broker fail) thay vì Outbox; chọn distributed transaction 2PC thay vì Saga; quên consumer phải idempotent dưới at-least-once. Bắt những chỗ đó là việc của Tech Lead.

---

## Mục lục mục con (ID prefix → số câu)

| Tag | Mục con | Số câu |
|---|---|---|
| **K-WHY** | Vì sao messaging · sync vs async · coupling · khi nào KHÔNG dùng queue | 8 |
| **K-BRK** | Broker landscape (Kafka/RabbitMQ/NATS/SQS-SNS/Redis Streams) · queue vs log vs pub-sub · push vs pull | 9 |
| **K-KAFKA** | Kafka internals (partition, consumer group, offset, ISR, rebalance, retention) | 12 |
| **K-DELIV** | Delivery semantics · idempotent consumer · ordering · DLQ · poison message (cross-ref I-IDEM) | 9 |
| **K-SAGA** | Saga (orchestration vs choreography, compensating transaction) — cross-ref H-CQRS | 9 |
| **K-OBX** | Outbox + CDC/Debezium · dual-write problem · inbox | 8 |
| **K-ES** | Event Sourcing + CQRS (event store, snapshot, replay, projection) — cross-ref H-CQRS | 7 |
| **K-RESIL** | Circuit breaker · bulkhead · retry/backoff · timeout · backpressure · cascade failure | 9 |
| **K-DECOMP** | Service decomposition · bounded context · data ownership · anti-pattern | 9 |
| **K-GW** | API gateway · service mesh · service discovery · BFF (cross-ref F-TL) | 8 |
| **K-OBS** | Distributed tracing & observability cho microservice (cross-ref O) | 6 |

---

## K-WHY — Vì sao messaging · sync vs async · coupling

**K-WHY-001** ⭐⭐
"Message queue giải quyết những lớp bài toán cốt lõi nào? Kể ≥3."
> *Dò cái gì:* decoupling (producer không cần biết/đợi consumer) · async/giảm latency cảm nhận · buffering/load-leveling (hấp thụ spike) · độ bền (message không mất khi consumer chết) · fan-out (1 event → nhiều consumer). Không chỉ nói "cho nhanh".

**K-WHY-002** ⭐⭐⭐
"Sync (HTTP/gRPC call trực tiếp) vs async (qua broker): đánh đổi gì? Khi nào chọn cái nào?"
> *Dò cái gì:* sync = đơn giản, có phản hồi ngay, nhưng coupling thời-gian (callee chết → caller chết) + cộng dồn latency; async = decouple + chịu lỗi tốt + buffer, nhưng eventual consistency + khó trace + phức tạp vận hành. Chọn theo: cần phản hồi tức thì? chịu được eventual? tải spike?

**K-WHY-003** ⭐⭐⭐
"Phân biệt temporal coupling, spatial coupling. Async messaging gỡ được loại nào?"
> *Dò cái gì:* temporal = phải cùng online một lúc; spatial = phải biết địa chỉ nhau. Broker gỡ **temporal** (consumer offline vẫn nhận sau) và một phần spatial (chỉ cần biết topic/queue, không biết instance); nhưng tạo coupling mới vào *schema event* và *broker*.

**K-WHY-004** ⭐⭐⭐
"Command vs Event (message): khác nhau về ngữ nghĩa và hệ quả thiết kế thế nào?"
> *Dò cái gì:* command = ra lệnh cho **một** consumer cụ thể làm gì (mong đợi side-effect, hay 1 owner); event = thông báo "đã xảy ra việc X" cho **bất kỳ ai quan tâm** (fan-out, producer không quan tâm ai xử lý). Đặt tên: imperative vs past-tense. Ảnh hưởng coupling & ai sở hữu logic.

**K-WHY-005** ⭐⭐⭐⭐
"'Cho mọi thứ qua queue' là anti-pattern thế nào? Khi nào KHÔNG nên dùng message queue?"
> *Dò cái gì:* khi cần phản hồi đồng bộ tức thì cho user (read-after-write tức thời), khi luồng đơn giản chưa cần decouple → thêm broker = thêm latency, hạ tầng, eventual consistency, khó debug. Async chỉ đáng khi có lý do (decouple/buffer/fan-out/độ bền), không phải mặc định.

**K-WHY-006** ⭐⭐⭐
"Point-to-point queue vs publish/subscribe khác nhau gì? Cho ví dụ mỗi loại."
> *Dò cái gì:* P2P = mỗi message tới **đúng một** consumer (work queue, chia tải job); pub/sub = mỗi message tới **mọi** subscriber quan tâm (broadcast event). Nhận diện nhu cầu "chia việc" vs "thông báo nhiều bên".

**K-WHY-007** ⭐⭐⭐⭐
"Async hoá một bước (vd gửi email khi đặt hàng) đổi mô hình nhất quán của hệ thống thế nào? User thấy gì khác?"
> *Dò cái gì:* từ strong → **eventual** consistency cho phần async; user có thể thấy "đặt hàng OK" trước khi email gửi/thật sự xử lý xong; phải thiết kế trạng thái trung gian (PENDING) + retry + thông báo lỗi muộn; không giả định "publish xong = đã xong".

**K-WHY-008** ⭐⭐⭐⭐
"Back-of-envelope: khi nào throughput/spike buộc phải có queue thay vì gọi trực tiếp DB/service?"
> *Dò cái gì:* khi tải đỉnh vượt khả năng xử lý đồng bộ của downstream (DB/3rd-party rate limit) → queue làm shock absorber + xử lý theo nhịp consumer (rate); hoặc khi tác vụ chậm (>vài trăm ms) không nên giữ request HTTP. Nối với capacity estimation (cross-ref A).

---

## K-BRK — Broker landscape · queue vs log vs pub-sub

**K-BRK-001** ⭐⭐⭐
"Kafka (log) vs RabbitMQ (queue/broker truyền thống): khác nhau về mô hình lưu trữ & tiêu thụ thế nào?"
> *Dò cái gì:* Kafka = **append-only log** giữ message theo retention, consumer tự giữ **offset**, đọc lại/replay được, nhiều consumer group đọc độc lập; RabbitMQ = broker đẩy message tới consumer rồi **xoá sau ack**, routing linh hoạt (exchange/binding), không replay mặc định. Log vs queue là khác biệt lõi. *(verify)*

**K-BRK-002** ⭐⭐⭐⭐
"Khi nào chọn Kafka, khi nào RabbitMQ, khi nào NATS, khi nào SQS/SNS? Tiêu chí quyết định."
> *Dò cái gì:* Kafka = event streaming bền + replay + throughput rất lớn + nhiều consumer độc lập + analytics/CDC; RabbitMQ = routing phức tạp (topic/headers exchange), task queue, per-message ack/priority; NATS = pub/sub siêu nhẹ, latency thấp, (JetStream cho bền); SQS/SNS = managed AWS, ít vận hành (SQS queue, SNS fan-out). Quyết theo: cần replay? routing? managed? throughput? *(verify landscape)*

**K-BRK-003** ⭐⭐⭐
"Push-based vs pull-based delivery (RabbitMQ push vs Kafka pull): đánh đổi gì, đặc biệt với consumer chậm?"
> *Dò cái gì:* push = broker đẩy, latency thấp nhưng dễ làm ngộp consumer chậm (cần prefetch/QoS limit); pull = consumer tự kéo theo nhịp của nó → tự backpressure, dễ batch, nhưng thêm 1 nhịp poll. Kafka pull giúp consumer kiểm soát tốc độ.

**K-BRK-004** ⭐⭐⭐⭐
"RabbitMQ: exchange (direct/topic/fanout/headers) + binding + queue hoạt động ra sao? Routing key để làm gì?"
> *Dò cái gì:* producer gửi tới **exchange** (không tới queue trực tiếp); exchange theo **type** + **binding key** định tuyến message (qua routing key) vào ≥0 queue; fanout = broadcast, direct = khớp chính xác, topic = wildcard. Hiểu tách publish khỏi routing. *(verify)*

**K-BRK-005** ⭐⭐⭐⭐
"AWS SQS standard vs FIFO khác gì? SNS vs SQS dùng cho mục đích nào, kết hợp ra sao?"
> *Dò cái gì:* SQS standard = at-least-once + best-effort ordering + throughput cao; FIFO = đúng thứ tự + dedup (message group id) nhưng throughput giới hạn; SNS = pub/sub fan-out tới nhiều subscriber; pattern phổ biến **SNS→nhiều SQS** (fan-out + buffer riêng mỗi consumer). *(verify giới hạn FIFO)*

**K-BRK-006** ⭐⭐⭐
"Redis Streams / BullMQ so với Kafka: khi nào 'đủ', khi nào phải lên Kafka? (đào sâu từ J-QUEUE)"
> *Dò cái gì:* Redis Streams/BullMQ hợp job queue Node, retry/delay, hạ tầng nhẹ (đã có Redis), throughput vừa; cần event-streaming bền lâu dài + replay xa + throughput rất lớn + nhiều consumer độc lập + retention dài → Kafka. *(cross-ref J-QUEUE-006; verify)*

**K-BRK-007** ⭐⭐⭐⭐
"Message broker đảm bảo độ bền (durability) thế nào? Replication, persist-to-disk, ack producer đóng vai trò gì?"
> *Dò cái gì:* persist message ra disk + replicate sang nhiều node trước khi coi là "đã nhận"; producer ack (vd Kafka `acks=all` chờ ISR; RabbitMQ publisher confirms) để biết broker thật sự giữ; thiếu các cơ chế này → message mất khi node chết. *(verify config)*

**K-BRK-008** ⭐⭐⭐⭐⭐
"Broker có thể là single point of failure / bottleneck không? Thiết kế HA cho lớp messaging thế nào?"
> *Dò cái gì:* broker phải cluster + replication factor ≥3 (Kafka) / mirrored-quorum queue (RabbitMQ); cân nhắc partition/sharding để scale ngang; client cần retry + reconnect + (Outbox phía producer để không mất event khi broker tạm sập); giám sát lag/disk. *(verify)*

**K-BRK-009** ⭐⭐⭐⭐
"Schema của message tiến hoá (thêm/xoá field) mà không vỡ consumer cũ — quản lý thế nào?"
> *Dò cái gì:* schema registry + versioning + quy tắc compatibility (backward/forward), tránh breaking change; thêm field optional, không đổi nghĩa field cũ; tách event version khi đổi lớn; consumer bỏ qua field lạ. Đây là 'API contract' của event. *(verify Avro/Protobuf/JSON Schema)*

---

## K-KAFKA — Kafka internals

**K-KAFKA-001** ⭐⭐
"Topic, partition, offset, broker, cluster — định nghĩa và quan hệ?"
> *Dò cái gì:* topic = luồng logic, chia thành **partition** (đơn vị song song + thứ tự); mỗi message trong partition có **offset** (số thứ tự tăng dần, duy nhất trong partition); **broker** = server lưu/serve partition; cluster = nhiều broker. Offset không duy nhất across partition.

**K-KAFKA-002** ⭐⭐⭐
"Vì sao partition là đơn vị của cả song song lẫn ordering? Hệ quả khi tăng partition?"
> *Dò cái gì:* Kafka chỉ đảm bảo thứ tự **trong một partition**, không across partition; số partition = trần song song của 1 consumer group; tăng partition tăng throughput nhưng phá thứ tự global + nhiều file/overhead + có thể đổi mapping key→partition. Trade-off ordering ↔ throughput.

**K-KAFKA-003** ⭐⭐⭐
"Consumer group là gì? Quy tắc 'mỗi partition tối đa 1 consumer trong group' dẫn tới điều gì?"
> *Dò cái gì:* group = tập consumer chia nhau partition để xử lý song song; mỗi partition gán cho **đúng 1** consumer trong group → nếu #consumer > #partition thì consumer thừa **ngồi không**; nhiều group khác nhau đọc **độc lập** cùng topic (pub/sub). Muốn scale → tăng partition.

**K-KAFKA-004** ⭐⭐⭐
"Producer quyết định message vào partition nào thế nào? Vai trò của message key?"
> *Dò cái gì:* có key → hash(key) % #partition → **cùng key luôn vào cùng partition** (giữ thứ tự theo key, vd theo userId/orderId); không key → phân bổ (round-robin/sticky). Key là công cụ giữ ordering theo thực thể. *(verify default partitioner)*

**K-KAFKA-005** ⭐⭐⭐
"Offset commit: auto vs manual. Commit *trước* khi xử lý xong gây gì? Commit *sau* gây gì?"
> *Dò cái gì:* commit trước xử lý → crash giữa chừng = **mất message** (at-most-once); commit sau xử lý → crash trước commit = **xử lý lại** (at-least-once, cần idempotent); manual commit + 'process rồi mới commit' là mặc định an toàn. *(cross-ref K-DELIV; verify __consumer_offsets)*

**K-KAFKA-006** ⭐⭐⭐⭐
"Rebalance là gì, kích hoạt khi nào, và vì sao nó nguy hiểm (stop-the-world, reorder, lag)?"
> *Dò cái gì:* khi consumer join/leave/chết hoặc đổi #partition → group reassign partition; trong lúc rebalance consumer ngừng xử lý (stop-the-world) → lag; có thể xử lý lại/đảo thứ tự nếu commit chưa kịp; cooperative/incremental rebalance giảm đau. *(verify)*

**K-KAFKA-007** ⭐⭐⭐⭐
"Replication factor, leader/follower, ISR (in-sync replicas) là gì? `acks=all` + `min.insync.replicas` bảo vệ điều gì?"
> *Dò cái gì:* mỗi partition có 1 leader + N-1 follower; ISR = replica theo kịp leader; `acks=all` chờ mọi ISR ghi xong, `min.insync.replicas` đặt ngưỡng tối thiểu → tránh mất dữ liệu khi leader chết. Đánh đổi: độ bền ↑ ↔ latency/availability ↓. *(verify)*

**K-KAFKA-008** ⭐⭐⭐⭐
"Leader của partition chết thì sao? Vì sao consumer thường là điểm chết đáng lo hơn 'Kafka chết'?"
> *Dò cái gì:* follower trong ISR được bầu làm leader mới (controller điều phối) → producer/consumer reconnect; Kafka cluster vốn HA ("available, sometimes consistent") nên realistic hơn là *consumer* chết → xử lý lại từ last committed offset (at-least-once). *(verify)*

**K-KAFKA-009** ⭐⭐⭐⭐
"Retention vs compaction khác nhau gì? Khi nào dùng log compaction?"
> *Dò cái gì:* retention theo thời gian/dung lượng → xoá message cũ; **compaction** giữ **bản ghi mới nhất theo key** (xoá bản cũ cùng key) → biến topic thành 'snapshot trạng thái mới nhất per key' (vd changelog, config, materialize state). Chọn theo: cần lịch sử đầy đủ hay chỉ state hiện tại. *(verify)*

**K-KAFKA-010** ⭐⭐⭐⭐⭐
"Exactly-once trong Kafka đạt bằng gì? Idempotent producer + transactions làm gì, giới hạn ở đâu?"
> *Dò cái gì:* idempotent producer (dedup retry phía producer bằng producer id + sequence) chống ghi trùng; transactions cho phép 'read-process-write' nguyên tử trong Kafka (exactly-once **trong** Kafka, vd Streams); KHÔNG bật mặc định, có overhead; ranh giới ra **DB ngoài** thì cần idempotent upsert/Outbox ('effectively once'). *(verify; cross-ref I-IDEM, K-DELIV)*

**K-KAFKA-011** ⭐⭐⭐⭐
"Hot partition / data skew là gì? Vì sao 'key theo tenantId' có thể nguy hiểm? Xử lý sao?"
> *Dò cái gì:* key phân bố lệch → một partition nhận phần lớn traffic → 1 consumer quá tải, các consumer khác rảnh; tenant lớn làm nóng partition của nó; xử lý: chọn key phân tán hơn (composite key), tách tenant lớn, hoặc đánh đổi ordering. Nhận diện trade-off ordering↔balance.

**K-KAFKA-012** ⭐⭐⭐⭐
"Consumer lag là gì, đo bằng gì, và nói lên điều gì? Khi lag tăng vô hạn thì xử lý ra sao?"
> *Dò cái gì:* lag = (latest offset − committed offset) = số message chưa xử lý; tăng dần = consumer chậm hơn producer; xử lý: tăng consumer (≤ #partition) / tăng partition / tối ưu xử lý / batch / async side-effect; là metric vận hành sống còn của lớp messaging. *(verify công cụ đo)*

---

## K-DELIV — Delivery semantics · idempotent consumer · ordering · DLQ

**K-DELIV-001** ⭐⭐⭐
"At-most-once / at-least-once / exactly-once delivery: định nghĩa và đánh đổi mỗi loại?"
> *Dò cái gì:* at-most-once = có thể mất, không trùng (commit trước xử lý); at-least-once = không mất, có thể trùng (xử lý trước commit/ack) — mặc định phổ biến; exactly-once *delivery* gần như bất khả ở ranh giới ngoài → thực tế at-least-once + dedup = **effectively-once**. *(cross-ref I-IDEM-002)*

**K-DELIV-002** ⭐⭐⭐⭐
"Vì sao consumer BẮT BUỘC idempotent dưới at-least-once? Cơ chế messaging-specific để dedup là gì?"
> *Dò cái gì:* broker sẽ redeliver (timeout/rebalance/retry) → side-effect chạy 2 lần; dedup theo **messageId/eventId** lưu vào store (đã xử lý → bỏ qua) hoặc unique constraint khi ghi; KHÁC góc HTTP `Idempotency-Key`. *(cross-ref I-IDEM, F-IDEM — không lặp lý thuyết, chỉ cơ chế messaging)*

**K-DELIV-003** ⭐⭐⭐⭐
"Ack/nack & redelivery hoạt động ra sao? 'Ack trước khi xử lý xong' sai ở đâu?"
> *Dò cái gì:* consumer chỉ ack **sau khi** xử lý + ghi side-effect xong; ack sớm → crash giữa chừng = mất message; nack/không-ack → broker redeliver (RabbitMQ requeue, Kafka không commit offset). Quy tắc: ack/commit là 'tôi đã xử lý xong', không phải 'tôi đã nhận'.

**K-DELIV-004** ⭐⭐⭐⭐
"Ordering guarantee: làm sao đảm bảo các event của CÙNG một thực thể (vd order) được xử lý đúng thứ tự?"
> *Dò cái gì:* route theo key (Kafka: cùng key→cùng partition; RabbitMQ: consistent-hash/single queue per key; SQS FIFO: message group id) + xử lý tuần tự trong phạm vi key; thường KHÔNG cần global order, chỉ per-entity. Trade-off ordering ↔ song song.

**K-DELIV-005** ⭐⭐⭐⭐
"Poison message là gì? Dead Letter Queue (DLQ) để làm gì và thiết kế retry quanh nó thế nào?"
> *Dò cái gì:* message luôn fail (data hỏng/bug) → retry vô hạn làm kẹt queue/blocking partition; sau N lần fail → đẩy sang **DLQ** để điều tra thủ công, không chặn luồng; retry có giới hạn + (jittered) backoff; alert khi DLQ tăng. *(cross-ref J-QUEUE-005)*

**K-DELIV-006** ⭐⭐⭐⭐
"Retry trong messaging: in-line retry vs retry-topic/delay-queue khác nhau? Vì sao retry đồng bộ trong consumer có thể chặn cả partition?"
> *Dò cái gì:* Kafka xử lý tuần tự per-partition → block-and-retry 1 message = chặn các message sau nó (head-of-line blocking); giải: retry topic riêng (đẩy sang, xử lý sau với delay tăng dần) / delay queue; phân biệt lỗi transient (retry) vs vĩnh viễn (DLQ ngay).

**K-DELIV-007** ⭐⭐⭐
"Vì sao consumer chậm không nên giữ message rồi gọi side-effect đồng bộ nặng? Liên hệ idempotency khi timeout."
> *Dò cái gì:* xử lý chậm → lag + visibility timeout hết → broker redeliver → 2 worker cùng xử lý 1 message → cần idempotent + atomic claim; tách side-effect nặng/đảm bảo nhanh-ack hoặc chia nhỏ. *(cross-ref I-IDEM parallel retry)*

**K-DELIV-008** ⭐⭐⭐⭐⭐
"'Effectively-once' end-to-end (producer → broker → consumer → DB) ghép từ những mảnh nào?"
> *Dò cái gì:* (a) producer idempotent/Outbox để publish không mất/không trùng, (b) broker durable + at-least-once, (c) consumer dedup theo eventId + ghi DB idempotent (upsert/unique) trong giao dịch atomic với việc lưu 'đã xử lý'. Không có 1 nút thần kỳ — là chuỗi cơ chế. *(cross-ref K-OBX)*

**K-DELIV-009** ⭐⭐⭐⭐
"Khi tăng số consumer để giảm lag lại làm vỡ ordering — giải thích nghịch lý và cách dung hoà."
> *Dò cái gì:* nhiều consumer xử lý song song nhiều partition → mất thứ tự global, nhưng ordering thường chỉ cần **per-key**; dung hoà: giữ ordering trong key (1 key→1 partition→1 consumer) còn song song giữa các key khác nhau. Hiểu 'ordering ở scope nào'.

---

## K-SAGA — Saga (cross-ref H-CQRS)

**K-SAGA-001** ⭐⭐⭐⭐
"Vì sao không dùng ACID transaction trải nhiều microservice (database-per-service)? Saga giải bài toán gì?"
> *Dò cái gì:* mỗi service có DB riêng → không có transaction chung; 2PC khoá + kém chịu lỗi + coupling; **Saga** = chuỗi local transaction, mỗi bước commit riêng, nếu một bước fail thì chạy **compensating transaction** để hoàn tác các bước trước → đạt nhất quán *eventual* không cần lock toàn cục.

**K-SAGA-002** ⭐⭐⭐⭐
"Orchestration vs Choreography saga: cơ chế điều phối khác nhau thế nào?"
> *Dò cái gì:* orchestration = 1 **orchestrator** trung tâm ra lệnh từng bước, lưu state, quyết bước tiếp/compensate (control tập trung, dễ nhìn state, dễ test); choreography = mỗi service phát **event**, service khác phản ứng (không trung tâm, decouple cao, throughput tốt) nhưng logic phân tán, khó theo dõi state. *(verify Temporal/Camunda)*

**K-SAGA-003** ⭐⭐⭐⭐
"Khi nào chọn orchestration, khi nào choreography?"
> *Dò cái gì:* choreography hợp saga ngắn (2–4 bước), event flow rõ, team quen event-driven; orchestration hợp saga phức tạp (5+ bước), cần visibility/branching/conditional, dễ test & vận hành; nhiều hệ lớn **dùng cả hai**. Không có 'simple→choreography' máy móc — xét cognitive load + visibility.

**K-SAGA-004** ⭐⭐⭐⭐⭐
"Compensating transaction KHÁC rollback ở chỗ nào? Vì sao nó khó đúng?"
> *Dò cái gì:* rollback = quay về như chưa xảy ra (trong 1 DB); compensation = **hành động nghiệp vụ ngược** trong một thế giới đã thay đổi (đã trừ kho/đã gửi mail/đã tính phí) → có thể không hoàn tác sạch (refund ≠ chưa charge), phải thiết kế semantic compensation; thứ tự compensate ngược, phải idempotent.

**K-SAGA-005** ⭐⭐⭐⭐⭐
"Saga thiếu Isolation (chữ I trong ACID). Hệ quả gì và giảm thiểu bằng cách nào?"
> *Dò cái gì:* giữa các bước, state trung gian **lộ ra** cho giao dịch khác (dirty read nghiệp vụ, vd thấy đơn PENDING) → anomalies (lost update, đọc dữ liệu sẽ bị compensate); giảm bằng semantic lock (trạng thái PENDING/reserved), versioning, commutative updates, re-read. Đây là điểm khó nhất của saga.

**K-SAGA-006** ⭐⭐⭐⭐
"Baseline BẮT BUỘC cho mọi saga (bất kể orchestration/choreography) gồm gì?"
> *Dò cái gì:* (1) atomic tại biên mỗi bước: ghi DB + publish event nguyên tử (**Outbox/CDC**), (2) **idempotent consumer** (at-least-once), (3) thiết kế compensation tường minh + DLQ, (4) observability: correlation id + saga state truy vấn được. Chọn pattern không thay được baseline. *(cross-ref K-OBX, K-OBS)*

**K-SAGA-007** ⭐⭐⭐⭐
"Bước compensation cũng có thể fail. Thiết kế chịu lỗi cho chính compensation thế nào?"
> *Dò cái gì:* compensation phải retry-able + idempotent; lưu state saga bền (đã/đang compensate bước nào); fail mãi → DLQ + alert + can thiệp thủ công; không để hệ kẹt ở trạng thái nửa vời mà im lặng. Saga state machine phải bền vững.

**K-SAGA-008** ⭐⭐⭐⭐
"Saga (cross-service, long-running) khác `@nestjs/cqrs` Saga (RxJS in-process) ở mức nào? (cross-ref H-CQRS)"
> *Dò cái gì:* Nest cqrs saga = phản ứng event **trong một process** bằng RxJS (điều phối command nội bộ); saga distributed = điều phối **qua nhiều service**, cần bền vững/long-running → công cụ riêng (Temporal/Camunda) hoặc choreography qua broker. Đừng nhầm 2 mức. *(verify)*

**K-SAGA-009** ⭐⭐⭐⭐⭐
"Thiết kế saga cho checkout: reserve inventory → charge payment → confirm order. Liệt kê bước, compensation, và 2 chỗ dễ sai."
> *Dò cái gì:* mỗi bước local tx + event/command; fail payment → compensate release-inventory; chỗ sai: (a) không idempotent → trừ tiền 2 lần khi redeliver, (b) lộ state PENDING gây oversell/đọc bẩn, (c) publish trong tx code thay vì Outbox. Phải gọi tên trade-off, không chỉ vẽ happy path. *(cross-ref A e-commerce checkout)*

---

## K-OBX — Outbox + CDC · dual-write

**K-OBX-001** ⭐⭐⭐⭐
"Dual-write problem là gì? Vì sao 'ghi DB rồi publish event' trong cùng đoạn code lại không an toàn?"
> *Dò cái gì:* 2 hệ thống (DB + broker) không trong 1 transaction → có thể commit DB OK nhưng publish fail (hoặc ngược lại) → state lệch (mất event / event ma). Không có atomicity giữa 2 nguồn → cần pattern, không 'try/catch' được.

**K-OBX-002** ⭐⭐⭐⭐⭐
"Transactional Outbox pattern hoạt động thế nào? Vì sao nó đảm bảo atomicity?"
> *Dò cái gì:* ghi business data + ghi event vào **bảng outbox** trong **cùng một ACID transaction** của DB → atomic; một tiến trình bất đồng bộ đọc outbox rồi publish lên broker (đánh dấu đã gửi). Event chỉ tồn tại nếu DB commit → không bao giờ 'ghi DB ok nhưng mất event'.

**K-OBX-003** ⭐⭐⭐⭐
"Hai cách đẩy event từ outbox ra broker: polling publisher vs CDC. Đánh đổi?"
> *Dò cái gì:* polling = job quét bảng outbox theo chu kỳ → đơn giản nhưng có độ trễ + tải DB; CDC = đọc **transaction log** của DB (Debezium) đẩy realtime → ít tải, độ trễ thấp, nhưng thêm hạ tầng. *(verify Debezium/Kafka Connect)*

**K-OBX-004** ⭐⭐⭐⭐
"CDC (Change Data Capture) với Debezium là gì? Vì sao đọc commit log tốt hơn polling DB để đồng bộ?"
> *Dò cái gì:* Debezium đọc WAL/binlog → phát event cho **mọi** thay đổi (kể cả ghi từ ngoài app) → publish lên Kafka; realtime + không miss + ít tải hơn poll; hiện thực Outbox hoặc đồng bộ DB→search/cache. *(cross-ref J consistency; verify)*

**K-OBX-005** ⭐⭐⭐⭐
"Outbox đảm bảo at-least-once publish → consumer vẫn có thể nhận trùng. Đầu nhận cần gì?"
> *Dò cái gì:* mỗi event có id ổn định → consumer dedup (Inbox pattern: lưu processed eventId, bỏ qua nếu đã thấy) + ghi side-effect idempotent; Outbox giải 'không mất', dedup giải 'không trùng'. Hai nửa của effectively-once. *(cross-ref K-DELIV-008)*

**K-OBX-006** ⭐⭐⭐
"Inbox pattern là gì, bổ trợ Outbox thế nào?"
> *Dò cái gì:* phía consumer ghi eventId nhận được vào **bảng inbox** trong cùng transaction với xử lý → dedup bền + atomic 'đã xử lý'; nếu redeliver, thấy eventId đã có → skip. Đối xứng với Outbox phía producer.

**K-OBX-007** ⭐⭐⭐⭐
"So sánh: distributed transaction (2PC) vs Saga+Outbox cho consistency qua service. Vì sao 2PC ít dùng ở microservices?"
> *Dò cái gì:* 2PC = coordinator khoá toàn bộ participant chờ commit/abort → blocking, kém chịu lỗi (coordinator chết = kẹt), giảm availability + throughput; microservices thiên về **eventual** qua Saga + Outbox/CDC, không khoá toàn cục. Đây là câu 'cũ→hiện đại' kinh điển.

**K-OBX-008** ⭐⭐⭐⭐
"Outbox table phình to / event trùng / ordering khi publish — các vấn đề vận hành và cách xử lý."
> *Dò cái gì:* dọn (archive/delete) row đã publish; publish có thể trùng (at-least-once) → consumer dedup; giữ ordering bằng cách publish theo thứ tự ghi (sequence/created_at) + key partition; giám sát backlog outbox như giám sát lag.

---

## K-ES — Event Sourcing + CQRS (cross-ref H-CQRS)

**K-ES-001** ⭐⭐⭐⭐
"Event Sourcing là gì? Lưu chuỗi event thay vì state hiện tại đánh đổi gì?"
> *Dò cái gì:* nguồn sự thật = **chuỗi event bất biến** (append-only); state hiện tại = replay/fold các event; được: audit đầy đủ, time-travel, rebuild view; mất: phức tạp, query state hiện tại khó, eventual, schema event tiến hoá khó, cần snapshot. *(cross-ref H-CQRS)*

**K-ES-002** ⭐⭐⭐
"Event Sourcing KHÁC 'chỉ ghi log/audit table' ở điểm cốt lõi nào?"
> *Dò cái gì:* trong ES event **là nguồn sự thật chính** (state được dựng *từ* event); audit log thông thường chỉ là phụ phẩm bên cạnh bảng state. Khác nhau ở 'ai là source of truth'.

**K-ES-003** ⭐⭐⭐⭐
"Snapshot trong Event Sourcing để làm gì? Khi nào cần?"
> *Dò cái gì:* replay từ event #0 mỗi lần quá tốn → lưu **snapshot** state tại offset N rồi chỉ replay event sau N; cần khi stream dài/đọc thường xuyên; trade-off bộ nhớ ↔ tốc độ rebuild.

**K-ES-004** ⭐⭐⭐⭐
"CQRS là gì? Vì sao ES và CQRS hay đi cùng nhau?"
> *Dò cái gì:* CQRS tách **write model** (command) khỏi **read model** (query, tối ưu riêng); ES ghi event (write) → project ra nhiều read model phù hợp truy vấn; ES giải bài 'event khó query trực tiếp' bằng projection. CQRS không bắt buộc ES và ngược lại.

**K-ES-005** ⭐⭐⭐⭐
"Projection / read model được cập nhật bất đồng bộ → read model trễ so với write. Hệ quả & cách xử lý?"
> *Dò cái gì:* eventual consistency giữa write và read view → user có thể không thấy ngay thay đổi vừa ghi; xử lý: chấp nhận + UI optimistic, đọc-từ-write-model cho path cần fresh, hiển thị trạng thái 'đang xử lý'. Phải thiết kế cho độ trễ projection.

**K-ES-006** ⭐⭐⭐⭐⭐
"Schema event tiến hoá trong hệ Event Sourcing (event đã ghi là bất biến) — versioning & upcasting xử lý sao?"
> *Dò cái gì:* event cũ không sửa được → cần **versioning event** + **upcaster** (chuyển event v1→v2 khi đọc) hoặc weak schema (bỏ qua field thiếu); không bao giờ rewrite lịch sử tuỳ tiện. Đây là chi phí dài hạn ít người lường.

**K-ES-007** ⭐⭐⭐⭐⭐
"Khi nào KHÔNG nên dùng Event Sourcing? (cảnh báo over-engineering)"
> *Dò cái gì:* CRUD đơn giản, không cần audit/time-travel, team chưa quen → ES thêm phức tạp lớn (eventual, versioning, rebuild) không tương xứng lợi ích; chỉ dùng khi domain thật sự cần lịch sử/audit/temporal (finance, ledger). Nhận diện cost/benefit.

---

## K-RESIL — Resilience (circuit breaker · bulkhead · retry · timeout)

**K-RESIL-001** ⭐⭐⭐
"Cascade failure trong microservices là gì? Một service chậm có thể kéo sập cả hệ thống thế nào?"
> *Dò cái gì:* call sync qua mạng → downstream chậm/treo → thread/connection của caller bị giữ chờ → cạn pool → caller cũng treo → lan ngược toàn chuỗi (resource exhaustion). Latency nhỏ ở 1 service khuếch đại thành sự cố toàn hệ.

**K-RESIL-002** ⭐⭐⭐
"Circuit breaker là gì? Ba trạng thái closed/open/half-open hoạt động ra sao?"
> *Dò cái gì:* closed = cho qua, đếm lỗi; vượt ngưỡng → **open** = fail-fast ngay (không gọi downstream, trả fallback) trong khoảng cooldown; **half-open** = thử vài request, OK thì đóng lại, fail thì mở tiếp. Mục tiêu: 'fail fast' bảo vệ tài nguyên + cho downstream thời gian hồi.

**K-RESIL-003** ⭐⭐⭐⭐
"Vì sao timeout là tiền đề cho circuit breaker? Không có timeout thì breaker bắt được lỗi latency không?"
> *Dò cái gì:* không timeout → request 'chậm' không bao giờ tính là lỗi → breaker không trip → thread vẫn bị giữ; timeout biến 'chậm vô hạn' thành lỗi đếm được; timeout + breaker + retry là bộ ba đi cùng.

**K-RESIL-004** ⭐⭐⭐⭐
"Bulkhead pattern là gì? Khác circuit breaker thế nào? Vì sao thường dùng cùng nhau?"
> *Dò cái gì:* bulkhead = chia tài nguyên (thread/connection pool) thành ngăn riêng cho từng downstream → 1 downstream hỏng chỉ cạn ngăn của nó, không nuốt hết tài nguyên chung (isolation); breaker = ngừng gọi downstream hỏng (fail-fast). Bulkhead cô lập, breaker chặn — bổ trợ. *(ẩn dụ vách ngăn tàu thuỷ)*

**K-RESIL-005** ⭐⭐⭐⭐
"Retry 'ngây thơ' (retry ngay, mọi lỗi) gây hại gì? Exponential backoff + jitter + retry-budget cứu thế nào?"
> *Dò cái gì:* retry đồng loạt khi downstream đang ngộp → 'retry storm' làm nó sập hẳn (cộng hưởng); backoff giãn cách + **jitter** phá đồng bộ + chỉ retry lỗi **transient/idempotent** + giới hạn budget; retry không idempotent = nhân đôi side-effect. *(cross-ref I-IDEM)*

**K-RESIL-006** ⭐⭐⭐⭐
"Circuit breaker dùng sai chỗ: anti-pattern nào? (quá chặt/quá lỏng, không fallback, bọc cả hàm nội bộ)"
> *Dò cái gì:* quá chặt → trip oan, quá lỏng → lỗi lọt; không fallback → vẫn lỗi tới user; bọc call nội bộ nhẹ (in-process) là thừa — chỉ dùng cho call remote/latency cao; thiếu visibility → silent failure. Hiểu 'khi nào KHÔNG dùng'.

**K-RESIL-007** ⭐⭐⭐⭐
"Fallback & graceful degradation: khi downstream chết, trả gì cho user thay vì lỗi cứng?"
> *Dò cái gì:* trả cached/stale, giá trị mặc định, kết quả một phần, hoặc xếp hàng xử lý sau; degrade tính năng phụ để giữ tính năng lõi; thiết kế 'fail soft'. Nối với resilience tổng thể (cross-ref J fallback khi cache mất).

**K-RESIL-008** ⭐⭐⭐⭐
"Backpressure trong messaging/microservice: producer/upstream nhanh hơn consumer/downstream thì sao, xử lý thế nào?"
> *Dò cái gì:* buffer phình → ngốn RAM/crash hoặc lag tăng vô hạn; xử lý: pull-based để consumer tự điều nhịp, prefetch/concurrency limit, load shedding (drop/429), queue có giới hạn, scale consumer; broker (queue) vốn là một dạng buffer có kiểm soát. *(cross-ref G-STR backpressure)*

**K-RESIL-009** ⭐⭐⭐⭐⭐
"Async qua queue đã 'decouple' rồi thì còn cần resilience không? Nêu sai lầm khi nghĩ queue là viên đạn bạc."
> *Dò cái gì:* queue gỡ temporal coupling nhưng KHÔNG gỡ: consumer vẫn gọi downstream sync (cần breaker), DLQ/poison cần xử lý, lag cần giám sát, idempotency vẫn bắt buộc, broker cũng có thể sập; resilience patterns vẫn cần ở từng consumer + biên ra ngoài.

---

## K-DECOMP — Service decomposition · bounded context · data ownership

**K-DECOMP-001** ⭐⭐
"Microservice là gì? Điều gì *thật sự* định nghĩa nó (không phải 'service nhỏ' hay 'dùng Docker')?"
> *Dò cái gì:* service tự trị quanh **một business capability/bounded context**, **deploy độc lập**, **sở hữu dữ liệu riêng** (no shared DB), giao tiếp qua API/event, có thể khác stack; trọng tâm là *ranh giới + autonomy*, không phải kích thước/công cụ. Tránh liệt kê tool.

**K-DECOMP-002** ⭐⭐⭐⭐
"Tách monolith thành service theo tiêu chí gì? Vì sao 'theo tầng kỹ thuật' (UI/logic/DB) là sai?"
> *Dò cái gì:* tách theo **business capability / bounded context (DDD)** + theo volatility/đội sở hữu, sao cho mỗi service thay đổi độc lập; tách theo tầng kỹ thuật → mọi feature đụng nhiều service (coupling cao, deploy lệ thuộc nhau) = distributed monolith.

**K-DECOMP-003** ⭐⭐⭐⭐
"Bounded context (DDD) là gì và vì sao nó là đường biên tốt cho service?"
> *Dò cái gì:* phạm vi mà một mô hình ngôn ngữ/khái niệm nhất quán (cùng 'Customer' nghĩa khác nhau ở Sales vs Billing); biên service trùng bounded context → ít coupling ngữ nghĩa, mỗi đội sở hữu mô hình của mình. *(cross-ref Q DDD)*

**K-DECOMP-004** ⭐⭐⭐⭐⭐
"Database-per-service: vì sao 'shared database' giữa các service là anti-pattern? Cái giá khi tách DB là gì?"
> *Dò cái gì:* shared DB → coupling schema, đổi bảng làm vỡ nhiều service, không deploy độc lập, khoá chung; tách DB cho autonomy nhưng mất JOIN/transaction xuyên service → cần API composition / Saga / event để đồng bộ + chấp nhận eventual. Hiểu cả lợi và giá.

**K-DECOMP-005** ⭐⭐⭐⭐⭐
"Distributed monolith là gì? Dấu hiệu nhận biết và vì sao nó tệ hơn cả monolith?"
> *Dò cái gì:* nhiều service nhưng **coupling chặt** (deploy phải đồng bộ, sync call dây chuyền, shared DB/lib, đổi 1 phải đổi nhiều) → gánh chi phí phân tán (latency/độ phức tạp/failure) mà KHÔNG được lợi autonomy; dấu hiệu: 'không deploy 1 service mà không deploy service khác'.

**K-DECOMP-006** ⭐⭐⭐⭐
"Sync coupling vs async (event) coupling khi service A cần dữ liệu của B: đánh đổi & khi nào chọn?"
> *Dò cái gì:* sync call B = đơn giản nhưng A phụ thuộc B sống + cộng latency + cascade; async/event (B phát event, A giữ bản sao/projection) = decouple + chịu lỗi nhưng eventual + trùng dữ liệu + phức tạp; chọn theo fresh-ness cần thiết & độ chịu lỗi.

**K-DECOMP-007** ⭐⭐⭐⭐
"Service 'quá nhỏ' (nano-service) gây hại gì? Làm sao biết đã chia quá mịn?"
> *Dò cái gì:* quá mịn → 1 thao tác nghiệp vụ phải gọi nhiều service (chatty, latency cộng dồn, transaction phân tán khắp nơi, vận hành nặng); dấu hiệu: nhiều service luôn đổi cùng nhau / luôn gọi nhau đồng bộ → nên gộp lại. Granularity là trade-off, không 'càng nhỏ càng tốt'.

**K-DECOMP-008** ⭐⭐⭐⭐
"Shared data giữa service: API composition, data replication (qua event), hay CQRS read model — chọn thế nào?"
> *Dò cái gì:* API composition (gọi runtime, fresh nhưng coupling/latency); replication qua event (giữ bản sao local, nhanh/decouple nhưng eventual + duplicate); read model riêng cho query xuyên service; chọn theo độ tươi + tần suất + chịu lỗi. Không có default.

**K-DECOMP-009** ⭐⭐⭐⭐⭐
"Strangler Fig pattern: di trú monolith → microservices từng phần thế nào? Vì sao không 'big-bang rewrite'?"
> *Dò cái gì:* bọc monolith, dần tách từng capability ra service mới + route traffic dần (qua gateway/proxy), giữ monolith chạy song song tới khi rút hết → giảm rủi ro; big-bang rewrite = rủi ro cao, lâu, dễ thất bại. *(verify tên pattern)*

---

## K-GW — API gateway · service mesh · service discovery (cross-ref F-TL)

**K-GW-001** ⭐⭐⭐
"Service discovery giải bài toán gì? Client-side vs server-side discovery khác nhau thế nào?"
> *Dò cái gì:* service scale/đổi IP động → không hardcode địa chỉ; instance **đăng ký** vào registry (Consul/Eureka/etcd...), bên gọi **tra cứu**; client-side = client tự hỏi registry rồi chọn instance; server-side = gateway/LB hỏi registry giúp. *(verify tên công cụ)*

**K-GW-002** ⭐⭐⭐
"API Gateway ở topology microservice đảm nhận gì? (góc topology, không lặp F-TL về 'god component')"
> *Dò cái gì:* điểm vào north-south: routing tới service, aggregation (gộp nhiều call), auth/rate-limit/TLS tập trung, che cấu trúc nội bộ khỏi client; cross-ref F-TL cho cảnh báo gateway phình to. Trọng tâm K: vị trí gateway trong sơ đồ service.

**K-GW-003** ⭐⭐⭐⭐
"North-south traffic vs east-west traffic: khác gì? Cái nào do gateway lo, cái nào do service mesh lo?"
> *Dò cái gì:* north-south = client ↔ hệ thống (qua API gateway); east-west = service ↔ service nội bộ (qua service mesh / direct); gateway lo biên ngoài, mesh lo giao tiếp nội bộ (mTLS, retry, traffic shaping). Phân vai rõ.

**K-GW-004** ⭐⭐⭐⭐
"Service mesh (Istio/Linkerd) là gì? Sidecar proxy làm gì? Vì sao tách cross-cutting ra khỏi code service?"
> *Dò cái gì:* lớp hạ tầng cho giao tiếp service-to-service: sidecar (proxy cạnh mỗi pod) lo mTLS, retry, timeout, circuit breaking, traffic split, telemetry — **không cần sửa code** từng service; đổi lại độ phức tạp vận hành + latency proxy. *(verify Istio/Linkerd)*

**K-GW-005** ⭐⭐⭐⭐
"BFF (Backend for Frontend) là gì? Khi nào cần thay vì 1 gateway dùng chung?"
> *Dò cái gì:* gateway/aggregation riêng cho từng loại client (web/mobile/3rd-party) → tối ưu payload & call theo nhu cầu client, tránh 1 gateway 'one-size-fits-all' phình to; mỗi đội frontend sở hữu BFF của mình. Trade-off: trùng lặp logic.

**K-GW-006** ⭐⭐⭐⭐
"mTLS giữa các service (zero-trust nội bộ) giải quyết gì? Service mesh hỗ trợ thế nào? (cross-ref M)"
> *Dò cái gì:* xác thực & mã hoá hai chiều giữa service → không tin mạng nội bộ mặc định (zero-trust); mesh tự cấp/luân chuyển cert + áp mTLS không cần sửa app; chống lateral movement khi 1 service bị chiếm. *(cross-ref M zero-trust/mTLS)*

**K-GW-007** ⭐⭐⭐⭐
"Gateway/mesh áp circuit breaking & retry ở tầng hạ tầng — khi nào điều này lợi, khi nào gây bất ngờ?"
> *Dò cái gì:* lợi: đồng bộ policy resilience không sửa code; bất ngờ: retry ở mesh + retry ở app = nhân số lần gọi (retry amplification), hoặc breaker mesh trip mà app không biết; cần phối hợp policy app↔mesh, tránh chồng retry. *(cross-ref K-RESIL-005)*

**K-GW-008** ⭐⭐⭐⭐⭐
"Service mesh không miễn phí: chi phí/độ phức tạp gì khiến KHÔNG phải hệ nào cũng nên dùng?"
> *Dò cái gì:* thêm sidecar (latency, RAM/CPU mỗi pod), control plane phải vận hành, đường cong học dốc, debug khó hơn; chỉ đáng khi nhiều service + cần mTLS/observability/traffic-mgmt thống nhất; hệ nhỏ → thư viện resilience (vd opossum) + gateway là đủ. *(verify)*

---

## K-OBS — Distributed tracing & observability cho microservice (cross-ref O)

**K-OBS-001** ⭐⭐⭐
"Vì sao debug một request trong microservices khó hơn monolith? Distributed tracing giải quyết gì?"
> *Dò cái gì:* 1 request đi qua nhiều service/queue → log rải rác, không thấy đường đi end-to-end; tracing gắn **trace id** xuyên suốt + chia thành **span** mỗi chặng → dựng lại timeline + tìm chặng chậm/lỗi. Không có nó → 'mò kim đáy bể'.

**K-OBS-002** ⭐⭐⭐⭐
"Trace, span, context propagation: định nghĩa và 'context' được truyền thế nào qua HTTP và qua message broker?"
> *Dò cái gì:* trace = toàn bộ hành trình 1 request; span = 1 đơn vị công việc (có parent/child); propagation = truyền trace/span id qua **HTTP header** (W3C traceparent) hoặc **message header** (đính vào event) để nối chặng async; thiếu propagation qua broker → trace đứt ở queue. *(verify W3C Trace Context)*

**K-OBS-003** ⭐⭐⭐
"Correlation ID / request ID là gì, khác trace id thế nào, và vì sao cần log có cấu trúc đi kèm?"
> *Dò cái gì:* correlation id gắn vào mọi log của cùng 1 luồng nghiệp vụ → grep ra toàn bộ hành trình; structured logging (JSON + field id) để truy vấn/aggregate; trace id (tracing) + correlation id (logging) bổ trợ. *(cross-ref O logging)*

**K-OBS-004** ⭐⭐⭐⭐
"OpenTelemetry là gì và giải bài toán 'mỗi vendor một SDK' thế nào? Quan hệ với Jaeger/Zipkin/Tempo?"
> *Dò cái gì:* OTel = chuẩn + SDK trung lập để sinh/truyền trace-metric-log, export tới backend bất kỳ (Jaeger/Tempo/Datadog...) → tránh khoá vendor; Jaeger/Zipkin/Tempo = backend lưu/hiển thị trace. Tách instrumentation khỏi backend. *(verify)*

**K-OBS-005** ⭐⭐⭐⭐
"Sampling trong tracing: vì sao không trace 100%? Head-based vs tail-based sampling đánh đổi gì?"
> *Dò cái gì:* trace mọi request = chi phí lưu/băng thông lớn → sample; head-based = quyết ngay đầu (rẻ, nhưng có thể bỏ lỡ trace lỗi); tail-based = quyết sau khi thấy cả trace (giữ được trace lỗi/chậm, nhưng tốn buffer). Trade-off cost ↔ giữ được trace quan trọng. *(verify)*

**K-OBS-006** ⭐⭐⭐⭐⭐
"Three pillars (logs/metrics/traces): mỗi cái trả lời câu hỏi gì? Khi alert nổ, dùng thứ tự nào để khoanh vùng?"
> *Dò cái gì:* metrics = 'có gì bất thường, ở đâu' (rate/error/latency, RED/USE); traces = 'request đi đâu chậm/lỗi' (khoanh service/chặng); logs = 'chi tiết vì sao' tại chặng đó; quy trình: metric phát hiện → trace khoanh vùng → log soi chi tiết. Hiểu vai trò từng pillar, không lẫn lộn. *(cross-ref O)*

---

## Gợi ý dùng tiếp (sau Bước A)

1. **Bước B (dựng giáo trình ngược):** gom 94 câu thành các BÀI HỌC theo cụm khái niệm, sắp **dependency order**:
   *(WHY/sync-async → BRK broker → KAFKA internals → DELIV semantics & idempotent → OBX dual-write/Outbox/CDC → SAGA → ES/CQRS → RESIL → DECOMP → GW/discovery/mesh → OBS tracing)*, ánh xạ Bài → ID câu.
2. **Bước C:** giảng từng bài theo Hợp đồng 10 mục; đụng config/đời broker/tên tool → search + cite.
3. **Bước D (phỏng vấn live):** mỗi đợt 10–15 câu trộn dễ→khó; **hỏi xong DỪNG chờ trả lời**; chấm theo dòng *"dò cái gì"*; chèn 1–2 câu cũ (spaced repetition).
4. **Cổng hoàn thành (Bước E):** ĐẠT toàn bộ 94 câu mới tính xong mục K.

> ⚠️ Khi học, **verify lại** mọi câu có *(verify)*: config Kafka (`acks`, `enable.idempotence`, transactional API, default partitioner, `min.insync.replicas`), RabbitMQ exchange/confirms, SQS FIFO limits, Debezium/Temporal/Camunda, Istio/Linkerd, opossum, W3C Trace Context, OpenTelemetry — tên/đời/cấu hình đổi nhanh.
