# 🧪 QB_I — Ngân hàng câu hỏi: Concurrency & Consistency

> **Mục I** trong roadmap Tech Lead Backend (🟡 ~50%) · Sinh theo **WORKFLOW 2 — Bước A**.
> Đây là **mục LỚN (A–R)** → bộ đề phủ HẾT 5 mục con của roadmap (race condition, optimistic/pessimistic locking, CAP, eventual consistency, idempotency) + mở rộng đúng chiều sâu TL: distributed locks, consistency models, consensus.
> **Chỉ câu hỏi — KHÔNG kèm đáp án.** Mỗi câu có dòng *"dò cái gì"* = tiêu chí ĐẠT, dùng để chấm **live** ở Bước D.
> **Chống trùng:** đã đọc `QB_E_database.md`, `QB_F_apidesign.md`, `QB_G_nodejs.md`, `QB_H_nestjs.md`, `QB_L_testing.md`, `QB_M_security.md`, `QB_Q_designpatterns.md`.
> - **E** đã giữ góc *DB-engine*: isolation levels, MVCC, deadlock, `FOR UPDATE`, lost-update ở mức storage, replication lag, 2PC↔Saga → I **không lặp** lý thuyết isolation; I nhìn từ góc **chiến lược điều khiển đồng thời & nhất quán phân tán**.
> - **F** đã giữ góc *HTTP*: `Idempotency-Key` header, method semantics (RFC 9110), claim key bằng unique constraint → I-IDEM chỉ chạm idempotency ở **góc concurrency/distributed** (retry, at-least-once, dedup dưới song song) + cross-ref F.
> - **G** đã giữ góc *Node runtime*: event loop, worker_threads, "race do đoán timing" → I bàn **race trên shared data (DB/Redis) across-instance**, không lặp event-loop phases.
> **Tổng: 71 câu.**

## Cách đọc
- **ID:** `I-<tag>-<số>`. Tag mục con liệt kê ở mục lục dưới.
- **Độ khó:** ⭐ định nghĩa cơ bản → ⭐⭐⭐⭐⭐ phân biệt senior với TL thật.
- **"Dò cái gì":** điều người trả lời PHẢI chạm tới mới tính ĐẠT (không phải đáp án — chỉ là mốc chấm).
- *(verify)* = câu chạm tên/đời sản phẩm, phân loại CAP của DB cụ thể, cú pháp Redis/ZK/etcd → **kiểm chứng tại docs chính thức** khi học (đổi nhanh).

## Bối cảnh (tính tại 6/2026)
- Lý thuyết concurrency/consistency (CAP, PACELC, locking, consensus) **ổn định** — ít đổi. Phần dễ đổi: **phân loại CAP/PACELC của từng DB**, **cú pháp & guarantee của Redis/Redlock/ZooKeeper/etcd**, **API `@VersionColumn` của ORM**.
- Tranh luận **Redlock** (Kleppmann ⇄ antirez) vẫn là tài liệu chuẩn để bàn distributed lock — *nguồn: martin.kleppmann.com + redis.io/docs.*
- **Raft** là chuẩn de-facto cho consensus (etcd/Consul); **Paxos** là nền lý thuyết. *(verify landscape khi học.)*

---

## Mục lục mục con (ID prefix → số câu)

| Tag | Mục con | Số câu |
|---|---|---|
| **I-RACE** | Race condition (góc shared-data / across-instance) | 9 |
| **I-LOCK** | Optimistic vs pessimistic locking | 12 |
| **I-DLOCK** | Distributed locks (Redis/Redlock/ZK/etcd, fencing) | 12 |
| **I-CAP** | CAP theorem & PACELC | 11 |
| **I-CONS** | Consistency models & eventual consistency | 12 |
| **I-IDEM** | Idempotency (góc concurrency/distributed) | 6 |
| **I-CONSENSUS** | Consensus (Raft/Paxos/quorum/leader election) | 9 |

---

## I-RACE — Race condition (góc shared-data / across-instance)

**I-RACE-001** ⭐⭐
"Race condition là gì? Node single-thread rồi sao vẫn dính race?"
> *Dò cái gì:* race = kết quả phụ thuộc thứ tự/timing không kiểm soát của các thao tác đồng thời; Node single-thread JS nhưng `await` nhả control → hai request xen kẽ giữa lúc *read* và lúc *write* trên cùng tài nguyên (DB/Redis) → vẫn race. *(khác G-ASY: ở đây race trên shared data, không phải event-loop.)*

**I-RACE-002** ⭐⭐⭐
"Check-then-act / TOCTOU là gì? Minh họa bằng 'kiểm tra tồn kho rồi trừ'."
> *Dò cái gì:* khoảng trống giữa lúc kiểm tra điều kiện và lúc hành động bị request khác chen vào → quyết định dựa trên state đã cũ; 2 request cùng đọc `stock=1` rồi cùng trừ → oversell.

**I-RACE-003** ⭐⭐⭐
"Vì sao `UPDATE ... SET balance = balance - 10 WHERE id=?` an toàn hơn 'đọc balance ở app rồi ghi lại'?"
> *Dò cái gì:* atomic update để DB tự khóa row trong một câu lệnh, không có khoảng trống; read-modify-write ở app mở cửa lost update vì hai request đọc cùng giá trị rồi ghi đè nhau.

**I-RACE-004** ⭐⭐⭐⭐
"Double-click / double-submit tạo 2 đơn hàng giống hệt — có phải race không, chặn ở tầng nào?"
> *Dò cái gì:* đúng là duplicate do concurrency; chặn bằng idempotency key + unique constraint ở server, KHÔNG chỉ disable nút ở UI (UI fix không cứu được retry/đa tab). *(cross-ref F-IDEM, I-IDEM.)*

**I-RACE-005** ⭐⭐⭐⭐
"1000 request cùng `+1` một counter nhưng kết quả < 1000. Debug từ đâu?"
> *Dò cái gì:* lost update do read-modify-write; sửa bằng atomic increment (DB `col = col + 1`, Redis `INCR`), không cần lock toàn cục; nhận ra counter ở app/cache là thủ phạm.

**I-RACE-006** ⭐⭐⭐
"Vì sao 'thêm sleep/retry để né race' là anti-pattern?"
> *Dò cái gì:* fix dựa-timing rất giòn, không giải nguyên nhân; cần cơ chế đồng bộ thật (atomic op / lock / unique constraint). *(đồng nhịp với G-ASY 'race do đoán timing'.)*

**I-RACE-007** ⭐⭐⭐⭐
"Race chỉ xuất hiện trên prod (nhiều instance) mà dev 1 máy không thấy — vì sao?"
> *Dò cái gì:* prod chạy concurrency thật (nhiều process/instance song song), dev tải thấp/gần tuần tự; in-process mutex/biến local KHÔNG bảo vệ across-instance → cần DB lock / distributed lock.

**I-RACE-008** ⭐⭐⭐⭐⭐
"Một `Map`/biến đếm trong RAM của 1 instance Node có chống được race giữa nhiều instance không?"
> *Dò cái gì:* không — mỗi instance có state riêng, không thấy nhau; cần shared store (Redis/DB) làm điểm đồng bộ duy nhất; bẫy kinh điển 'dùng object in-memory làm lock'.

**I-RACE-009** ⭐⭐⭐
"Liệt kê 4 chiến lược chống race condition và khi nào dùng cái nào."
> *Dò cái gì:* (a) atomic/conditional write, (b) pessimistic lock (`FOR UPDATE`/distributed lock), (c) optimistic lock (version/CAS + retry), (d) unique constraint/idempotency key; chọn theo mức contention + chi phí retry + phạm vi tài nguyên.

---

## I-LOCK — Optimistic vs pessimistic locking

**I-LOCK-001** ⭐⭐
"Optimistic vs pessimistic locking khác nhau ở triết lý nào?"
> *Dò cái gì:* pessimistic giả định conflict hay xảy ra → **khóa trước** khi đụng data; optimistic giả định conflict hiếm → cho chạy song song, chỉ **kiểm tra conflict lúc commit**.

**I-LOCK-002** ⭐⭐⭐
"Optimistic locking hiện thực bằng gì? Mô tả version column / CAS."
> *Dò cái gì:* mỗi row có version (hoặc timestamp); `UPDATE ... SET ..., version=version+1 WHERE id=? AND version=?`; nếu affected rows = 0 → có người sửa trước → fail + retry. (TypeORM `@VersionColumn` — *verify*.)

**I-LOCK-003** ⭐⭐⭐
"Pessimistic locking ở DB hiện thực bằng gì?"
> *Dò cái gì:* `SELECT ... FOR UPDATE` (exclusive) / `FOR SHARE`; khóa row tới hết transaction, tx khác chờ. *(cross-ref E-ISO — không lặp định nghĩa isolation, ở đây là góc chiến lược.)*

**I-LOCK-004** ⭐⭐⭐⭐
"Khi nào chọn optimistic, khi nào pessimistic — theo đặc tính workload nào?"
> *Dò cái gì:* contention thấp + read-heavy + retry rẻ/idempotent → optimistic; contention cao + ghi nóng cùng row + retry đắt → pessimistic; **đo conflict rate**, không chọn cảm tính.

**I-LOCK-005** ⭐⭐⭐⭐
"Optimistic locking dưới contention cao có thể tệ hơn pessimistic — vì sao?"
> *Dò cái gì:* nhiều tx cùng đụng 1 row → đa số fail version-check → retry dồn dập (wasted work, gần livelock); pessimistic serialize nhưng mỗi tx tiến đều, không phí công.

**I-LOCK-006** ⭐⭐⭐⭐⭐
"ABA problem trong optimistic concurrency là gì? Version column giải quyết thế nào?"
> *Dò cái gì:* giá trị đổi A→B→A nên check 'bằng giá trị cũ' tưởng không đổi → bỏ sót thay đổi; version đơn điệu tăng (monotonic) phát hiện đã có update dù giá trị quay về như cũ.

**I-LOCK-007** ⭐⭐⭐⭐
"Pessimistic lock và deadlock liên hệ thế nào? Phòng ra sao?"
> *Dò cái gì:* nhiều `FOR UPDATE` khóa chéo theo thứ tự ngược → deadlock; phòng bằng khóa theo **thứ tự nhất quán**, tx ngắn, lock granular (row > table). *(cross-ref E-ISO-008.)*

**I-LOCK-008** ⭐⭐⭐⭐
"Thiết kế retry cho optimistic lock đúng cách: số lần, backoff, idempotent?"
> *Dò cái gì:* retry có **giới hạn** + (jittered) backoff để tránh đồng pha; mỗi lần retry phải **đọc lại state mới** rồi tính lại (an toàn lặp); fail hẳn sau N lần → trả lỗi rõ thay vì retry vô hạn.

**I-LOCK-009** ⭐⭐⭐
"`SELECT ... FOR UPDATE SKIP LOCKED` giải bài toán gì?"
> *Dò cái gì:* queue/job claim trong DB: nhiều worker lấy các row *khác nhau* không chờ nhau — bỏ qua row đang bị khóa thay vì block; giảm contention ở pattern 'hàng đợi trong DB'.

**I-LOCK-010** ⭐⭐⭐⭐
"'Cứ FOR UPDATE cho chắc' trên mọi read — hại gì?"
> *Dò cái gì:* serialize không cần thiết → giảm throughput, giữ lock + connection lâu → block, deadlock, pool cạn; chỉ khóa đúng điểm nóng read-modify-write.

**I-LOCK-011** ⭐⭐⭐⭐
"Optimistic version column vs atomic conditional update (`stock = stock - 1 WHERE stock > 0`) — khi nào dùng cái nào?"
> *Dò cái gì:* atomic conditional update gọn nhất cho counter/stock đơn; version column hợp khi sửa nhiều field hoặc cần **báo conflict cho user** (vd edit form 'ai đó vừa sửa, tải lại'). *(cross-ref E-ISO-007 lost update.)*

**I-LOCK-012** ⭐⭐⭐⭐⭐
"App multi-instance dùng optimistic version column thì có cần thêm distributed lock không?"
> *Dò cái gì:* không — version check ở DB đã là điểm đồng bộ atomic across-instance; thêm distributed lock thường thừa và giảm throughput; hiểu **DB constraint chính là concurrency control** rồi.

---

## I-DLOCK — Distributed locks (Redis/Redlock/ZK/etcd, fencing)

**I-DLOCK-001** ⭐⭐⭐
"Distributed lock là gì, khi nào cần (so với DB row lock)?"
> *Dò cái gì:* mutual exclusion across process/instance/service cho tài nguyên **không nằm gọn trong 1 DB tx** (gọi API ngoài, cron chỉ-chạy-một-lần, ghi nhiều store); khi `FOR UPDATE` không phủ được phạm vi.

**I-DLOCK-002** ⭐⭐⭐
"Lock Redis tối thiểu: `SET key token NX PX ttl`. Vì sao cần NX, vì sao cần TTL?"
> *Dò cái gì:* NX = set-if-not-exists để **claim atomic** (1 client thắng); TTL = tự nhả nếu holder chết → tránh deadlock vĩnh viễn. *(verify cú pháp tại redis.io.)*

**I-DLOCK-003** ⭐⭐⭐⭐
"Vì sao release lock phải 'delete-if-token-matches' (Lua/CAS) chứ không `DEL` thẳng?"
> *Dò cái gì:* nếu lock đã hết TTL và bị client khác chiếm, `DEL` thẳng sẽ **xóa nhầm lock của người khác**; phải so token sở hữu rồi mới xóa, atomic bằng Lua script.

**I-DLOCK-004** ⭐⭐⭐⭐
"TTL quá ngắn vs quá dài cho distributed lock — đánh đổi gì?"
> *Dò cái gì:* ngắn → lock hết hạn *giữa* critical section (2 holder cùng lúc); dài → holder chết thì tài nguyên kẹt lâu; cần watchdog/lease-renewal hoặc ước lượng đúng + fencing.

**I-DLOCK-005** ⭐⭐⭐⭐⭐
"Redlock là gì, và Kleppmann phê phán điểm nào?"
> *Dò cái gì:* Redlock = acquire trên N Redis độc lập, cần majority (N/2+1); critique: (a) **không sinh fencing token**, (b) phụ thuộc giả định timing (clock skew, GC pause, network delay) → có thể 2 client cùng giữ lock. *(verify; nguồn Kleppmann ⇄ antirez.)*

**I-DLOCK-006** ⭐⭐⭐⭐⭐
"Fencing token là gì, và nó cứu tình huống nào mà TTL không cứu được?"
> *Dò cái gì:* số đơn điệu tăng cấp mỗi lần acquire; resource từ chối write có token **nhỏ hơn** token đã thấy → chặn holder 'zombie' (bị pause) ghi đè sau khi lock đã chuyển; TTL không chặn vì process tỉnh dậy vẫn tưởng còn giữ lock.

**I-DLOCK-007** ⭐⭐⭐⭐
"Lock 'for efficiency' vs 'for correctness' — Kleppmann khuyên gì cho mỗi loại?"
> *Dò cái gì:* efficiency (tránh làm trùng, sai chút không chết) → single-node Redis `SET NX` là đủ; correctness (sai = mất tiền/hỏng data) → dùng consensus system (ZooKeeper/etcd) + **bắt buộc fencing token**.

**I-DLOCK-008** ⭐⭐⭐⭐
"Vì sao GC pause / process pause là kẻ thù của distributed lock dựa-thời-gian?"
> *Dò cái gì:* holder pause lâu hơn TTL → lock hết hạn, client khác chiếm; holder tỉnh lại vẫn tưởng giữ lock và ghi tiếp → vi phạm mutual exclusion; chỉ fencing token chặn được.

**I-DLOCK-009** ⭐⭐⭐⭐
"ZooKeeper/etcd làm distributed lock khác Redis ở chỗ nào?"
> *Dò cái gì:* dựa **consensus** (ZAB/Raft) + ephemeral node/lease + thứ tự → tự sinh thứ tự đơn điệu (gần fencing), tự nhả khi session chết; an toàn hơn cho correctness, đổi lại nặng/chậm hơn. *(verify.)*

**I-DLOCK-010** ⭐⭐⭐
"'Cron job chỉ được chạy 1 lần dù deploy nhiều instance' — thiết kế thế nào?"
> *Dò cái gì:* leader election / distributed lock TTL ngắn + renew, hoặc DB advisory lock / unique row theo schedule-slot; **tránh giả định 'chỉ deploy 1 instance'**.

**I-DLOCK-011** ⭐⭐⭐⭐
"Redis master–replica + lock: vì sao failover có thể phá tính an toàn của lock single-node?"
> *Dò cái gì:* replication async — lock `SET` trên master chưa kịp sang replica, master chết, replica lên không có lock → 2 client cùng giữ; chính lý do Redlock ra đời (và Redlock cũng không kín). *(verify.)*

**I-DLOCK-012** ⭐⭐⭐⭐⭐
"Khi nào nên TRÁNH distributed lock hoàn toàn và dùng cách khác?"
> *Dò cái gì:* nếu thay được bằng atomic DB op / unique constraint / idempotency / **partition theo key** (mỗi key 1 consumer) thì ưu tiên — distributed lock là 'code smell' dễ sai; lock chỉ khi thật sự cần exclusion across-resource.

---

## I-CAP — CAP theorem & PACELC

**I-CAP-001** ⭐⭐⭐
"Phát biểu CAP theorem. 'Chọn 2 trong 3' gây hiểu lầm gì?"
> *Dò cái gì:* C/A/P, không đạt cả 3 *khi có partition*; hiểu lầm: trong hệ phân tán P là **bắt buộc** (mạng sẽ chia) → thực ra chỉ chọn **C hay A** khi partition xảy ra.

**I-CAP-002** ⭐⭐⭐⭐
"'Consistency' trong CAP nghĩa chính xác là gì?"
> *Dò cái gì:* **linearizability** — read luôn thấy write mới nhất, như chỉ có một bản sao duy nhất; KHÁC 'Consistency' trong ACID. *(cross-ref E-ACID trap — ở đây nói rõ là linearizability.)*

**I-CAP-003** ⭐⭐⭐⭐
"Phân biệt CAP-Consistency vs ACID-Consistency (bẫy thuật ngữ)."
> *Dò cái gì:* CAP-C = các replica đồng ý (replica agreement); ACID-C = transaction giữ invariant/constraint (FK, check); hai khái niệm **trực giao**; có thể ACID local nhưng eventually-consistent global.

**I-CAP-004** ⭐⭐⭐⭐
"Khi partition xảy ra, CP và AP system hành xử khác nhau thế nào? Ví dụ DB mỗi loại."
> *Dò cái gì:* CP **từ chối/chặn** request để giữ nhất quán; AP **vẫn phục vụ** nhưng có thể trả data cũ / ghi conflict; ví dụ CP: etcd, MongoDB majority — AP: Cassandra (default), Dynamo-style. *(verify phân loại.)*

**I-CAP-005** ⭐⭐⭐⭐⭐
"PACELC mở rộng CAP thế nào, vì sao thực tế hơn?"
> *Dò cái gì:* **if Partition → A vs C; Else (không partition, >99% thời gian) → Latency vs C**; nắm được trade-off latency↔consistency tồn tại trên **mỗi request bình thường**, không chỉ lúc sự cố.

**I-CAP-006** ⭐⭐⭐⭐
"Phân loại PACELC của một hệ tunable (vd Cassandra ONE vs QUORUM)."
> *Dò cái gì:* default ONE ≈ **PA/EL**; với QUORUM read+write → **PC/EC**; người dùng chọn per-query → hiểu **tunable consistency**. *(verify.)*

**I-CAP-007** ⭐⭐⭐
"'Chúng tôi chọn availability' — vì sao chưa đủ với interviewer senior?"
> *Dò cái gì:* phải định lượng: **staleness window** chấp nhận được, conflict rate dự kiến, cách resolve conflict (LWW / vector clock / CRDT); trade-off cần con số, không khẩu hiệu.

**I-CAP-008** ⭐⭐⭐⭐
"Partition tolerance 'không clear-cut' nghĩa là gì?"
> *Dò cái gì:* partition không nhị phân — node A coi timeout là chết, node B vẫn chạy; hệ có thể bất đồng về việc *có* partition hay không; CAP đơn giản hóa quá mức thực tế.

**I-CAP-009** ⭐⭐⭐⭐
"Hệ single-region / single-node thì CAP có áp dụng không? Bẫy khi mang CAP vào mọi câu?"
> *Dò cái gì:* CAP chỉ về **phân tán có replica**; một Postgres đơn lẻ không 'chọn C/A' theo CAP; đừng over-apply — phần lớn app vừa & nhỏ không cần lập luận CAP.

**I-CAP-010** ⭐⭐⭐⭐
"AP cho ghi cả 2 phía lúc partition → khi heal phải resolve conflict bằng cách nào?"
> *Dò cái gì:* last-write-wins (timestamp — rủi ro mất ghi), vector clocks/version vectors, CRDT (merge tự động), hoặc app-level merge; mỗi cách đánh đổi đơn giản ↔ rủi ro mất dữ liệu.

**I-CAP-011** ⭐⭐⭐⭐⭐
"Một hệ vừa strong-consistent cho tài chính vừa eventually-consistent cho analytics — đúng hay sai?"
> *Dò cái gì:* đúng — consistency là lựa chọn **per-operation**, không per-system; linearizable cho payment, causal cho chat, eventual cho feed/analytics; tune theo yêu cầu nghiệp vụ thật.

---

## I-CONS — Consistency models & eventual consistency

**I-CONS-001** ⭐⭐⭐
"Consistency là một phổ — kể từ mạnh đến yếu."
> *Dò cái gì:* linearizable → sequential → causal → eventual; mạnh hơn = cần **coordination** nhiều hơn = đắt/chậm hơn.

**I-CONS-002** ⭐⭐⭐⭐
"Linearizability nghĩa là gì (1 câu)? Vì sao đắt?"
> *Dò cái gì:* mọi op như xảy ra tức thời tại một điểm giữa invoke và response, tôn trọng **real-time order**, như chỉ có 1 bản sao; đắt vì cần round-trip tới majority/leader.

**I-CONS-003** ⭐⭐⭐⭐⭐
"Linearizability vs serializability — khác gì?"
> *Dò cái gì:* linearizability = thuộc tính **single-object + real-time ordering** (recency); serializability = thuộc tính **multi-object** về transaction tương đương *một* thứ tự tuần tự nào đó (không bắt buộc real-time); 'strict serializable' = cả hai.

**I-CONS-004** ⭐⭐⭐
"Eventual consistency nghĩa là gì? 'Eventual' là bao lâu?"
> *Dò cái gì:* nếu ngừng ghi, các replica cuối cùng **hội tụ** về cùng giá trị; 'eventual' thường ms→giây, bị chặn bởi replication lag/network; không có đảm bảo thời điểm cụ thể. *(cross-ref E-REP lag.)*

**I-CONS-005** ⭐⭐⭐⭐
"Read-your-own-writes là gì? Vì sao user 'vừa update xong mà không thấy'?"
> *Dò cái gì:* session guarantee đảm bảo đọc thấy ghi *của chính mình*; vi phạm khi read route sang replica chưa cập nhật; fix: đọc từ primary sau write / sticky theo session / chờ version token. *(cross-ref E-REP route nhầm replica.)*

**I-CONS-006** ⭐⭐⭐⭐
"Monotonic reads là gì? Hiện tượng 'thời gian chạy ngược' khi thiếu nó?"
> *Dò cái gì:* đảm bảo lần đọc sau không thấy state **cũ hơn** lần đọc trước; thiếu → user reload trang thấy data biến mất (trúng replica lệch khác nhau); fix sticky replica / version.

**I-CONS-007** ⭐⭐⭐⭐
"Causal consistency giải bài toán gì mà eventual không?"
> *Dò cái gì:* giữ **thứ tự nhân-quả** (reply không xuất hiện trước message gốc); yếu hơn linearizable nhưng đủ cho chat/comment; eventual có thể đảo thứ tự các sự kiện liên quan.

**I-CONS-008** ⭐⭐⭐⭐
"Quorum read/write: công thức R + W > N nghĩa là gì?"
> *Dò cái gì:* nếu tập node đọc (R) + tập node ghi (W) vượt tổng N thì hai tập **giao nhau** → đọc chắc chắn chạm node có ghi mới nhất; tunable: W=N (read nhanh), R=N (write nhanh), QUORUM cân bằng.

**I-CONS-009** ⭐⭐⭐⭐
"Strong hay eventual cho: (a) số dư ví, (b) đếm like, (c) trạng thái đơn hàng?"
> *Dò cái gì:* (a) strong/linearizable (tiền không được sai), (b) eventual (lệch tạm chấp nhận), (c) thường strong cho trạng thái quyết định, eventual cho hiển thị phụ; gắn lựa chọn vào yêu cầu nghiệp vụ.

**I-CONS-010** ⭐⭐⭐⭐⭐
"Tunable consistency (chọn per-operation) có lợi gì, rủi ro gì?"
> *Dò cái gì:* lợi: trả đúng chi phí cho đúng nhu cầu; rủi ro: dev chọn **sai mức** cho path quan trọng → bug khó tái hiện; cần policy rõ + test cho path nhạy cảm.

**I-CONS-011** ⭐⭐⭐
"'Strong consistency luôn đúng nên cứ chọn nó' — phản biện."
> *Dò cái gì:* strong = latency cao + giảm availability khi partition + throughput thấp; nhiều path không cần; chọn theo yêu cầu, không mặc định strong toàn cục.

**I-CONS-012** ⭐⭐⭐⭐
"Replication lag gây ra lớp lỗi consistency nào ở tầng app? Giảm tác động bằng gì?"
> *Dò cái gì:* stale read, mất read-your-writes, non-monotonic reads; giảm bằng read-from-primary cho path nhạy cảm, sticky session, version/token chờ replica, hoặc chấp nhận + UX phù hợp. *(cross-ref E-REP.)*

---

## I-IDEM — Idempotency (góc concurrency/distributed)

> *Phạm vi:* I-IDEM nhìn idempotency từ **retry/đa luồng/distributed**; cơ chế HTTP (`Idempotency-Key` header, method semantics RFC 9110, TTL/scope) nằm ở **F-IDEM** — không lặp.

**I-IDEM-001** ⭐⭐⭐
"Vì sao retry + at-least-once delivery bắt buộc consumer phải idempotent?"
> *Dò cái gì:* mạng/queue sẽ giao **trùng** (timeout-retry, redelivery); xử lý không idempotent → tính tiền/gửi mail 2 lần; idempotent = lặp không đổi kết quả. *(cross-ref F-IDEM, K messaging.)*

**I-IDEM-002** ⭐⭐⭐⭐
"'Exactly-once delivery' có tồn tại không? Đạt 'effectively-once' bằng gì?"
> *Dò cái gì:* exactly-once *delivery* là ảo tưởng (two-generals); thực tế **at-least-once + idempotent/dedup consumer = effectively-once**. *(cross-ref F.)*

**I-IDEM-003** ⭐⭐⭐⭐
"Thiết kế dedup key chống xử lý trùng dưới concurrency: lưu ở đâu cho atomic?"
> *Dò cái gì:* **unique constraint / `INSERT ... ON CONFLICT`** để claim key atomic (1 thắng) — KHÔNG check-then-act; TTL hợp lý; key theo business id. *(cross-ref F 'hai request song song cùng key'.)*

**I-IDEM-004** ⭐⭐⭐⭐
"Phân biệt 'naturally idempotent' vs 'phải làm cho idempotent'. Cho ví dụ mỗi loại."
> *Dò cái gì:* set tuyệt đối (`PUT state=X`, `SET balance=100`) tự idempotent; delta/create (`+10`, INSERT order) phải gắn dedup key/version; nhận diện thao tác nào cần thêm cơ chế.

**I-IDEM-005** ⭐⭐⭐⭐⭐
"Operation đã idempotent nhưng 2 retry chạy **song song** cùng key — vẫn race không?"
> *Dò cái gì:* idempotency một mình **chưa đủ** — cần atomic claim (unique constraint) hoặc lock để serialize; nếu không, hai lần cùng chèn / cùng side-effect; phải kết hợp idempotency + concurrency control.

**I-IDEM-006** ⭐⭐⭐
"Liên hệ idempotency với optimistic locking và outbox pattern."
> *Dò cái gì:* version/CAS làm write idempotent theo state; outbox + dedup đảm bảo publish không trùng/không mất; cùng mục tiêu 'an toàn khi lặp'. *(cross-ref I-LOCK, K-Outbox.)*

---

## I-CONSENSUS — Consensus (Raft/Paxos/quorum/leader election)

**I-CONSENSUS-001** ⭐⭐⭐
"Consensus là bài toán gì, cần cho việc nào?"
> *Dò cái gì:* nhiều node **đồng ý trên một giá trị/thứ tự** dù có lỗi (crash fault); cần cho leader election, log/state replication, distributed lock, config store (etcd).

**I-CONSENSUS-002** ⭐⭐⭐
"Paxos vs Raft — khác biệt thực dụng nào đáng nói?"
> *Dò cái gì:* Paxos đúng-được-chứng-minh nhưng khó hiểu/khó implement; Raft thiết kế cho **dễ hiểu** (tách leader election + log replication), nay là chuẩn de-facto; trong interview **đừng diễn Paxos từng bước**.

**I-CONSENSUS-003** ⭐⭐⭐⭐
"Quorum (majority) là gì, vì sao thường dùng số node lẻ?"
> *Dò cái gì:* chỉ **majority (N/2+1)** mới elect leader / commit write → tránh split-brain 2 leader; số lẻ (3/5/7) tối ưu khả năng chịu lỗi cho cùng số node (3 chịu 1, 5 chịu 2).

**I-CONSENSUS-004** ⭐⭐⭐⭐
"Raft elect leader thế nào (trực giác)?"
> *Dò cái gì:* follower không nghe heartbeat trong election timeout → thành candidate, tăng **term**, xin vote; ai được majority vote thành leader; term đóng vai **logical clock** để nhận ra leader cũ/thông tin cũ.

**I-CONSENSUS-005** ⭐⭐⭐⭐
"Split-brain là gì? Consensus chặn nó bằng đâu?"
> *Dò cái gì:* hai node cùng tưởng mình là leader → ghi mâu thuẫn; chặn bằng **majority quorum** (chỉ phe đa số mới commit/elect), phe thiểu số stall.

**I-CONSENSUS-006** ⭐⭐⭐⭐
"Network partition chia cluster Raft: hai phía hành xử ra sao? Khi heal thì sao?"
> *Dò cái gì:* phe có majority elect leader mới + tiếp tục; phe thiểu số không đạt quorum → không commit (read/write block hoặc stale); heal xong leader cũ thấy term cao hơn → **step down**, đồng bộ lại log.

**I-CONSENSUS-007** ⭐⭐⭐⭐
"Vì sao consensus là cách 'đúng' để sinh fencing token cho distributed lock?"
> *Dò cái gì:* token cần **đơn điệu tăng + bền qua node chết**; đếm trên 1 node không chịu lỗi, nhiều node tự đếm thì lệch nhau → cần consensus; ZooKeeper/etcd cung cấp thứ tự này. *(cross-ref I-DLOCK fencing.)*

**I-CONSENSUS-008** ⭐⭐⭐
"etcd dùng ở đâu trong hạ tầng thật và vì sao?"
> *Dò cái gì:* store cấu hình/coordination cho Kubernetes (state của cluster), service discovery, leader election, distributed lock; vì Raft cho strong consistency + cơ chế watch. *(verify landscape.)*

**I-CONSENSUS-009** ⭐⭐⭐⭐⭐
"Vì sao consensus đắt, và khi nào KHÔNG nên đặt nó vào hot path?"
> *Dò cái gì:* mỗi quyết định cần round-trip tới majority + fsync log → latency cao, throughput giới hạn; chỉ dùng cho **metadata/coordination/leader election**, không cho mọi write data-plane; tách control-plane vs data-plane.

---

## ✅ Sau khi LƯU file này vào project — bước tiếp theo

1. **Bước B (dựng giáo trình ngược):** gom 71 câu thành các BÀI HỌC theo cụm khái niệm, sắp xếp **dependency order** (race/lock nền → distributed lock → CAP/consistency → consensus), ánh xạ Bài → ID câu.
2. **Bước C:** giảng từng bài theo Hợp đồng 10 mục.
3. **Bước D:** phỏng vấn theo đợt 10–15 câu, **hỏi xong DỪNG chờ trả lời**, chấm LIVE bám dòng *"dò cái gì"*.
4. **Bước E:** cổng hoàn thành = ĐẠT toàn bộ; chèn câu cũ ôn ngắt quãng.

> ⚠️ Nhắc khi học: lý thuyết I ổn định, nhưng **mọi dòng có *(verify)*** (phân loại CAP/PACELC của DB, cú pháp Redis/Redlock/ZK/etcd, API ORM) phải đối chiếu **docs chính thức** tại thời điểm học — đừng tin trí nhớ model.
