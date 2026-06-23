# 🧪 QB_E — Ngân hàng câu hỏi: Database & Data Modeling

> **Mục E** trong roadmap Tech Lead Backend (🔴 ~70%) · Sinh theo **WORKFLOW 2 — Bước A**.
> Đây là **mục LỚN** → bộ đề phủ HẾT 13 mục con + nhóm data-modeling/TL để đủ chiều sâu.
> **Chỉ câu hỏi — KHÔNG kèm đáp án.** Mỗi câu có dòng *"dò cái gì"* = tiêu chí ĐẠT, dùng để chấm **live** ở Bước D.
> **Chống trùng:** đã đọc `QB_G_nodejs.md` + `QB_H_nestjs.md`. E giữ phần *DB engine/data modeling* (index, MVCC, isolation, sharding, Postgres specifics...), KHÔNG lặp connection-pool ở góc Node runtime của G hay TypeORM/`forFeature` wiring của H — ở đây pooling/N+1 nhìn từ **phía database**.
> **Tổng: 97 câu.**

## Cách đọc
- **ID:** `E-<tag>-<số>`. Tag mục con liệt kê ở mục lục dưới.
- **Độ khó:** ⭐ định nghĩa cơ bản → ⭐⭐⭐⭐⭐ phân biệt senior với TL thật.
- **"Dò cái gì":** điều người trả lời PHẢI chạm tới mới tính ĐẠT (không phải đáp án — chỉ là mốc chấm).

## Bối cảnh phiên bản (tính tại 6/2026 — *verify lại tại postgresql.org khi học*)
- **PostgreSQL 18.4** = bản stable mới nhất (5/2026). **PostgreSQL 17** = bản ổn định trưởng thành → cả 17 và 18 đều hợp production.
- **PostgreSQL 14** sẽ **EOL 12/11/2026** → lên kế hoạch nâng cấp. **PostgreSQL 13** trở về trước đã **EOL** → tránh.
- **PostgreSQL 19 Beta 1** ra 6/2026, bản chính thức dự kiến ~Q3–Q4/2026 → **không dùng beta cho production**.
- Chu kỳ: ~1 major/năm, mỗi major được support **5 năm** — *kiểm chứng tại postgresql.org/support/versioning.*
- ⚠️ Mọi câu chạm tên/đời DB, distributed-SQL, tooling đổi nhanh đều có dấu *(verify)*.

---

## Mục lục mục con (ID prefix → số câu)

| Tag | Mục con | Số câu |
|---|---|---|
| **E-SQLNO** | SQL vs NoSQL & chọn data model | 8 |
| **E-IDX** | Index & B-tree | 9 |
| **E-CIDX** | Composite index | 5 |
| **E-NP1** | N+1 query | 6 |
| **E-ACID** | ACID & transaction | 8 |
| **E-ISO** | Isolation levels · MVCC · locking · deadlock | 9 |
| **E-NORM** | Normalization vs denormalization | 6 |
| **E-SHRD** | Sharding & partitioning | 8 |
| **E-REP** | Replication & lag | 7 |
| **E-PG** | PostgreSQL specifics (JSONB/VACUUM/MVCC...) | 8 |
| **E-OPT** | Query optimization tools (EXPLAIN, pg_stat_statements) | 7 |
| **E-POOL** | Connection pooling | 6 |
| **E-NEW** | NewSQL / distributed SQL | 5 |
| **E-RT** | Data modeling depth / quyết định TL | 5 |

---

## E-SQLNO — SQL vs NoSQL & chọn data model

**E-SQLNO-001** ⭐⭐
"Chọn SQL hay NoSQL dựa trên gì? Vì sao 'NoSQL nhanh hơn SQL' là câu trả lời sai?"
> *Dò cái gì:* chọn theo **access pattern + yêu cầu nhất quán + mô hình dữ liệu**, không phải 'tốc độ' chung chung; nhận ra Postgres là mặc định an toàn cho phần lớn ca.

**E-SQLNO-002** ⭐⭐⭐
"Bốn họ NoSQL (document, key-value, wide-column, graph) — mỗi loại hợp use case nào, kèm ví dụ DB cụ thể."
> *Dò cái gì:* document=MongoDB (đọc theo aggregate), key-value=Redis/DynamoDB (truy cập theo key), wide-column=Cassandra/ScyllaDB (ghi lớn theo partition key), graph=Neo4j (quan hệ nhiều bậc).

**E-SQLNO-003** ⭐⭐⭐
"Trong MongoDB, khi nào nên embed document lồng nhau, khi nào reference (tách collection)?"
> *Dò cái gì:* embed khi đọc cùng nhau + quan hệ bounded; reference khi unbounded/chia sẻ/cập nhật độc lập; trade-off duplicate vs phải 'join' tay.

**E-SQLNO-004** ⭐⭐⭐⭐
"MongoDB đã có multi-document transaction từ lâu — vậy câu 'NoSQL không có ACID' còn đúng không? Nói cho chính xác."
> *Dò cái gì:* nhiều NoSQL hiện đại đã hỗ trợ ACID (multi-doc tx) nhưng có chi phí/giới hạn; không vơ đũa cả nắm, phân biệt theo từng DB.

**E-SQLNO-005** ⭐⭐⭐
"'Schema-less' có thật sự là 'không cần thiết kế schema'? Rủi ro thực tế là gì?"
> *Dò cái gì:* schema vẫn tồn tại ngầm ở tầng application; schema drift, phải tự validate, query khó/khó tối ưu khi dữ liệu không đồng nhất.

**E-SQLNO-006** ⭐⭐⭐⭐
"Khi nào wide-column store (Cassandra) thắng PostgreSQL? 'Query-first / thiết kế bảng quanh truy vấn' nghĩa là gì?"
> *Dò cái gì:* ghi cực lớn, multi-region, truy vấn theo partition key biết trước; thiết kế bảng theo query đã biết thay vì normalize, chấp nhận duplicate.

**E-SQLNO-007** ⭐⭐⭐⭐
"Polyglot persistence là gì? Đánh đổi khi dùng nhiều loại DB trong một hệ?"
> *Dò cái gì:* chọn store theo từng workload (vd Postgres + Redis + Elasticsearch); trả giá: vận hành phức tạp, đồng bộ/nhất quán giữa các store.

**E-SQLNO-008** ⭐⭐⭐⭐⭐
"TL: team đề xuất chuyển sang MongoDB 'để scale cho dễ'. Bạn kiểm tra/hỏi những gì TRƯỚC khi đồng ý?"
> *Dò cái gì:* truy access pattern thật, nhu cầu transaction/join, chi phí migration & rủi ro, liệu Postgres (JSONB/partition/replica) đã đủ chưa; quyết định bằng dữ liệu, không theo hype.

---

## E-IDX — Index & B-tree

**E-IDX-001** ⭐⭐
"Index là gì và vì sao nó tăng tốc đọc?"
> *Dò cái gì:* cấu trúc phụ cho tra cứu nhanh (như mục lục sách); tránh full table scan, dùng thuật toán tìm nhanh.

**E-IDX-002** ⭐⭐⭐
"Vì sao index làm CHẬM ghi (insert/update/delete)?"
> *Dò cái gì:* mỗi thao tác ghi phải cập nhật cả index; càng nhiều index càng đắt; đây là trade-off read vs write kinh điển.

**E-IDX-003** ⭐⭐⭐
"Vì sao đa số RDBMS dùng B-tree/B+tree cho index mặc định thay vì hash hay binary tree?"
> *Dò cái gì:* B+tree cân bằng, hỗ trợ equality + range + sort + prefix, đọc theo block đĩa hiệu quả; hash chỉ phục vụ equality.

**E-IDX-004** ⭐⭐⭐⭐
"Clustered index vs non-clustered (secondary) khác nhau gì? Postgres có clustered index không?"
> *Dò cái gì:* clustered quyết định thứ tự vật lý của row; Postgres KHÔNG có clustered index thật (heap + `CLUSTER` chỉ sắp 1 lần), khác MySQL InnoDB (PK = clustered).

**E-IDX-005** ⭐⭐⭐
"Covering index / index-only scan là gì? Khi nào nó cứu một query chậm?"
> *Dò cái gì:* index chứa đủ cột query cần → không phải đọc heap/table; ở Postgres dùng `INCLUDE` columns.

**E-IDX-006** ⭐⭐⭐⭐
"Vì sao index trên cột selectivity thấp (vd boolean `is_active`) thường vô dụng?"
> *Dò cái gì:* planner bỏ qua index khi phải đọc phần lớn bảng (full scan rẻ hơn random I/O); khái niệm cardinality/selectivity.

**E-IDX-007** ⭐⭐⭐⭐
"Có index rồi mà query vẫn full scan / không dùng index — kể các nguyên nhân thường gặp."
> *Dò cái gì:* function trên cột (không sargable), type mismatch/cast, `LIKE '%x'` leading wildcard, selectivity thấp, thống kê cũ, `OR`; hiểu khái niệm sargable.

**E-IDX-008** ⭐⭐⭐⭐
"Partial index và expression (functional) index giải quyết bài toán gì? Cho ví dụ."
> *Dò cái gì:* partial = index một phần bảng theo `WHERE` (vd chỉ rows active); expression = index trên `lower(email)` khớp query; tiết kiệm + đúng truy vấn thực.

**E-IDX-009** ⭐⭐⭐⭐⭐
"Một bảng có 12 index, ghi chậm dần. Là TL bạn quyết định bỏ index nào — dựa trên dữ liệu gì?"
> *Dò cái gì:* dùng `pg_stat_user_indexes` (idx_scan=0 = không dùng), tìm index trùng/redundant (là prefix của composite khác), đo write amplification; quyết định bằng usage stats, không cảm tính.

---

## E-CIDX — Composite index

**E-CIDX-001** ⭐⭐⭐
"Composite index `(a,b,c)` — thứ tự cột quan trọng thế nào? Quy tắc leftmost prefix là gì?"
> *Dò cái gì:* chỉ dùng được khi query có `a` (rồi `a,b`...); `(a,b)` ≠ `(b,a)`; query chỉ trên `b` không tận dụng được index này.

**E-CIDX-002** ⭐⭐⭐⭐
"Index `(a,b)` có phục vụ được `WHERE a=? ORDER BY b` không? Vì sao điều đó quý?"
> *Dò cái gì:* có — tránh bước sort riêng vì row đã sẵn thứ tự theo `b` trong mỗi nhóm `a`; equality-prefix kết hợp order.

**E-CIDX-003** ⭐⭐⭐⭐
"Sắp thứ tự cột trong composite index theo nguyên tắc nào (equality vs range)?"
> *Dò cái gì:* cột dùng equality đặt trước, cột dùng range/order đặt sau; một cột range sẽ 'cắt' khả năng dùng các cột sau nó.

**E-CIDX-004** ⭐⭐⭐
"Hai index riêng `(a)` và `(b)` có tương đương một index `(a,b)` không? Index merge/bitmap là gì?"
> *Dò cái gì:* không tương đương; DB có thể bitmap-AND hai index nhưng thường kém hơn một composite phù hợp.

**E-CIDX-005** ⭐⭐⭐⭐⭐
"Cho 4 truy vấn phổ biến trên một bảng, thiết kế bộ index TỐI THIỂU để phủ mà không thừa."
> *Dò cái gì:* gộp theo leftmost prefix, dùng covering/`INCLUDE`, loại bỏ index là prefix của index khác; tư duy tối ưu cả tập index chứ không từng cái.

---

## E-NP1 — N+1 query

**E-NP1-001** ⭐⭐⭐
"N+1 query là gì? Cho ví dụ cụ thể nó phát sinh thế nào với ORM."
> *Dò cái gì:* 1 query lấy list + N query lấy quan hệ cho từng phần tử; do lazy loading được gọi trong vòng lặp.

**E-NP1-002** ⭐⭐⭐
"Phát hiện N+1 trong production bằng cách nào (không đoán)?"
> *Dò cái gì:* bật query logging/đếm số query mỗi request, APM/tracing, hoặc `pg_stat_statements` thấy một query lặp N lần.

**E-NP1-003** ⭐⭐⭐⭐
"Các cách fix N+1 và trade-off mỗi cách (JOIN/eager, batch `IN`, DataLoader)?"
> *Dò cái gì:* JOIN eager (1 query nhưng có thể nhân dòng), tách 2 query với `WHERE IN`, DataLoader gom theo tick; chọn theo ngữ cảnh, không có một đáp án đúng duy nhất.

**E-NP1-004** ⭐⭐⭐⭐
"Eager load tất cả 'để tránh N+1' có hại gì? Khi nào lazy lại tốt hơn?"
> *Dò cái gì:* over-fetch dữ liệu không dùng, JOIN nặng/duplicate rows, ngốn RAM; lazy tốt khi quan hệ ít khi cần.

**E-NP1-005** ⭐⭐⭐
"Vì sao GraphQL đặc biệt dễ dính N+1, và pattern chuẩn để chống là gì?"
> *Dò cái gì:* resolver lồng theo field gọi DB cho từng node; DataLoader batch + cache per-request. *(liên hệ mục F roadmap.)*

**E-NP1-006** ⭐⭐⭐⭐⭐
"JOIN một query lớn vs nhiều query nhỏ (batch) — khi nào nhiều query nhỏ lại NHANH hơn?"
> *Dò cái gì:* JOIN nhân dòng/khó cache/giữ lock rộng; nhiều query nhỏ tận dụng index + cache + truyền ít data; phải đo thực tế, không giáo điều 'JOIN luôn tốt'.

---

## E-ACID — ACID & transaction

**E-ACID-001** ⭐⭐⭐
"ACID là gì? Giải thích từng chữ bằng ví dụ chuyển tiền."
> *Dò cái gì:* Atomicity, Consistency, Isolation, Durability; ví dụ all-or-nothing giữa hai tài khoản.

**E-ACID-002** ⭐⭐⭐
"'Consistency' trong ACID khác 'Consistency' trong CAP thế nào? (bẫy thuật ngữ)"
> *Dò cái gì:* ACID-C = invariant/constraint luôn hợp lệ; CAP-C = mọi node thấy dữ liệu mới nhất; hai khái niệm hoàn toàn khác nhau.

**E-ACID-003** ⭐⭐⭐
"Durability đảm bảo bằng cơ chế gì ở mức engine (WAL/redo log)?"
> *Dò cái gì:* ghi log trước (write-ahead log) + fsync; commit nghĩa là log đã bền, có thể replay khi crash.

**E-ACID-004** ⭐⭐⭐⭐
"Transaction nên 'dài' hay 'ngắn'? Vì sao long-running transaction nguy hiểm?"
> *Dò cái gì:* giữ lock/giữ snapshot lâu → block tx khác, bloat (MVCC version cũ không dọn được), tăng deadlock; mở tx muộn, commit sớm.

**E-ACID-005** ⭐⭐⭐⭐
"Savepoint / subtransaction để làm gì? Postgres hỗ trợ tới đâu?"
> *Dò cái gì:* rollback một phần bên trong tx lớn bằng `SAVEPOINT`; Postgres có subtransaction nhưng không 'nested commit' độc lập.

**E-ACID-006** ⭐⭐⭐
"Khi nào KHÔNG cần transaction tường minh? Đa số ORM ngầm làm gì cho một câu lệnh đơn?"
> *Dò cái gì:* single statement đã atomic (autocommit); chỉ cần tx tường minh khi nhiều thao tác phải nguyên tử cùng nhau.

**E-ACID-007** ⭐⭐⭐⭐
"Transaction qua nhiều service/DB (distributed) — vì sao tránh 2PC, dùng gì thay?"
> *Dò cái gì:* 2PC khóa + kém chịu lỗi ở microservices; dùng **Saga** + **Outbox**, chấp nhận eventual consistency. *(liên hệ mục K roadmap.)*

**E-ACID-008** ⭐⭐⭐⭐⭐
"Bạn mở transaction rồi gọi một API HTTP bên ngoài ngay giữa tx — vì sao đây là anti-pattern?"
> *Dò cái gì:* giữ tx mở chờ I/O mạng → giữ lock + connection lâu, dễ timeout, gây bloat; tách side-effect ra ngoài tx (outbox / sau commit).

---

## E-ISO — Isolation levels · MVCC · locking · deadlock

**E-ISO-001** ⭐⭐⭐
"Bốn isolation level chuẩn SQL là gì, sắp theo độ chặt tăng dần?"
> *Dò cái gì:* Read Uncommitted < Read Committed < Repeatable Read < Serializable.

**E-ISO-002** ⭐⭐⭐⭐
"Ba anomaly dirty read / non-repeatable read / phantom read là gì, và level nào chặn cái nào?"
> *Dò cái gì:* định nghĩa đúng từng anomaly + map: RC chặn dirty, RR chặn non-repeatable, Serializable chặn phantom.

**E-ISO-003** ⭐⭐⭐⭐
"Default isolation của PostgreSQL là gì? 'Repeatable Read' của Postgres khác chuẩn SQL ở điểm nào?"
> *Dò cái gì:* default **Read Committed**; RR của Postgres là snapshot isolation, mạnh hơn mức tối thiểu chuẩn (chặn được phần lớn phantom).

**E-ISO-004** ⭐⭐⭐⭐⭐
"MVCC hoạt động thế nào và vì sao 'reader không block writer, writer không block reader'?"
> *Dò cái gì:* mỗi row có nhiều version (xmin/xmax), reader thấy snapshot phù hợp nên không cần read lock; chỉ writer–writer trên cùng row mới đụng nhau.

**E-ISO-005** ⭐⭐⭐⭐
"Serializable trong Postgres (SSI) hoạt động kiểu gì, và vì sao nó có thể abort tx với serialization_failure?"
> *Dò cái gì:* kiểu optimistic — phát hiện vòng phụ thuộc giữa các tx và rollback một cái; ứng dụng phải retry.

**E-ISO-006** ⭐⭐⭐
"`SELECT ... FOR UPDATE` làm gì? Khi nào cần?"
> *Dò cái gì:* khóa các row được chọn để cập nhật, chặn tx khác sửa/khóa chúng; dùng cho read-modify-write tránh lost update.

**E-ISO-007** ⭐⭐⭐⭐
"Lost update là gì? Hai request cùng trừ tồn kho — minh họa và cách chặn."
> *Dò cái gì:* hai tx đọc cùng giá trị rồi ghi đè nhau; chặn bằng `FOR UPDATE` (pessimistic), version/CAS (optimistic), hoặc `UPDATE ... SET stock = stock - 1` atomic. *(cross-ref mục I.)*

**E-ISO-008** ⭐⭐⭐⭐
"Deadlock trong DB xảy ra thế nào? Postgres xử lý ra sao và bạn phòng/giảm bằng cách gì?"
> *Dò cái gì:* hai tx khóa chéo theo thứ tự ngược; DB detect và abort một nạn nhân; phòng: khóa theo thứ tự nhất quán, tx ngắn, thu hẹp phạm vi lock.

**E-ISO-009** ⭐⭐⭐⭐⭐
"Nâng isolation lên Serializable 'cho an toàn' có miễn phí không? TL cân nhắc gì?"
> *Dò cái gì:* chi phí abort/retry, throughput giảm, contention tăng; thường giải đúng bài toán ở RC + locking/atomic op cho điểm nóng, không nâng toàn cục.

---

## E-NORM — Normalization vs denormalization

**E-NORM-001** ⭐⭐
"Normalization là gì? Tóm tắt 1NF/2NF/3NF theo ý nghĩa thực tế (không học vẹt)."
> *Dò cái gì:* loại bỏ redundancy/anomaly; 1NF giá trị atomic, 2NF bỏ partial dependency, 3NF bỏ transitive dependency.

**E-NORM-002** ⭐⭐⭐
"Khi nào nên denormalize? Đánh đổi đọc/ghi là gì?"
> *Dò cái gì:* đọc nhiều/JOIN đắt → nhân bản dữ liệu để đọc nhanh; trả giá: ghi phức tạp, nguy cơ data lệch, phải đồng bộ.

**E-NORM-003** ⭐⭐⭐
"Đã denormalize rồi thì giữ dữ liệu nhân bản không lệch bằng cách nào?"
> *Dò cái gì:* cập nhật đồng thời (trigger/app trong cùng transaction) hoặc chấp nhận eventual + job đồng bộ; phải ý thức rủi ro lệch.

**E-NORM-004** ⭐⭐⭐⭐
"Lưu dữ liệu tính sẵn (counter/aggregate): materialized view vs cột denormalized vs cache — chọn cái nào khi nào?"
> *Dò cái gì:* MV refresh định kỳ; cột denorm cập nhật realtime nhưng tốn ghi; cache TTL chấp nhận stale; chọn theo độ tươi cần thiết.

**E-NORM-005** ⭐⭐⭐⭐
"Vì sao OLTP thường chuẩn hóa còn OLAP/warehouse thường denormalize (star schema)?"
> *Dò cái gì:* OLTP tối ưu ghi + nhất quán; OLAP tối ưu đọc/aggregate, ít JOIN → star/snowflake schema.

**E-NORM-006** ⭐⭐⭐⭐⭐
"Reviewer chê DB của bạn 'chưa chuẩn 3NF'. Khi nào đó là vấn đề thật, khi nào là giáo điều?"
> *Dò cái gì:* chuẩn hóa là default tốt, nhưng denormalize **có chủ đích vì performance** là hợp lệ; phán xét theo access pattern + đo đạc, không theo 'điểm thi 3NF'.

---

## E-SHRD — Sharding & partitioning

**E-SHRD-001** ⭐⭐⭐
"Partitioning vs sharding khác nhau thế nào?"
> *Dò cái gì:* partition = chia bảng trong CÙNG một instance (logical); shard = chia data qua NHIỀU instance/node.

**E-SHRD-002** ⭐⭐⭐
"Ba kiểu partition (range / list / hash) — mỗi kiểu hợp gì?"
> *Dò cái gì:* range theo khoảng (thời gian), list theo tập giá trị (region), hash phân tán đều; chọn theo query + phân bố data.

**E-SHRD-003** ⭐⭐⭐⭐
"Chọn shard key sai gây hậu quả gì? Cho ví dụ hot shard."
> *Dò cái gì:* phân bố lệch (hotspot), không scale, cross-shard query đắt; vd shard theo ngày → shard hôm nay nóng, các shard cũ nằm im.

**E-SHRD-004** ⭐⭐⭐⭐
"Vì sao cross-shard query và cross-shard JOIN/transaction là cơn ác mộng?"
> *Dò cái gì:* scatter-gather qua nhiều node, không có JOIN tự nhiên, distributed transaction phức tạp; phải thiết kế shard key để né.

**E-SHRD-005** ⭐⭐⭐⭐
"Consistent hashing giải bài toán gì khi thêm/bớt shard?"
> *Dò cái gì:* giảm tối thiểu số key phải di chuyển khi đổi số node (so với `hash % N` phải rehash gần như toàn bộ). *(cross-ref mục A.)*

**E-SHRD-006** ⭐⭐⭐⭐
"Khi nào CHƯA nên shard? Các bước scale nên thử trước."
> *Dò cái gì:* shard là phương án cuối; trước đó: tuning index/query, vertical scale, read replica, caching, partitioning; shard tăng độ phức tạp rất lớn.

**E-SHRD-007** ⭐⭐⭐
"Declarative partitioning của Postgres giúp gì cho bảng cực lớn, ngoài tốc độ query?"
> *Dò cái gì:* partition pruning; `DROP`/`DETACH` partition để xóa data cũ nhanh (vs `DELETE` nặng); bảo trì/VACUUM theo từng partition.

**E-SHRD-008** ⭐⭐⭐⭐⭐
"TL: đang 1 Postgres, bảng `orders` ~2 tỷ dòng, tải tăng. Vạch lộ trình mở rộng theo thứ tự nào, mốc nào mới shard?"
> *Dò cái gì:* đo bottleneck trước → index/partition + read replica + cache → cân nhắc Citus/NewSQL → shard ở tầng app chỉ khi thật cần; có lộ trình, quyết định theo dữ liệu.

---

## E-REP — Replication & lag

**E-REP-001** ⭐⭐⭐
"Replication là gì và vì sao cần (read scaling + HA)?"
> *Dò cái gì:* copy data sang node khác; scale read, failover/HA, phục vụ backup/analytics tách tải.

**E-REP-002** ⭐⭐⭐
"Synchronous vs asynchronous replication khác nhau gì, đánh đổi?"
> *Dò cái gì:* sync chờ replica xác nhận (an toàn, chậm hơn, có thể block khi replica chết); async nhanh nhưng có thể mất data khi failover.

**E-REP-003** ⭐⭐⭐⭐
"Replication lag là gì? Đọc từ replica ngay sau khi ghi master gây vấn đề gì?"
> *Dò cái gì:* replica chậm hơn master → 'read-your-own-write' fail (user không thấy data vừa ghi); stale read.

**E-REP-004** ⭐⭐⭐⭐
"Cách xử lý read-after-write khi dùng read replica?"
> *Dò cái gì:* route read quan trọng về master, sticky theo session, chờ LSN/replay position, hoặc đọc từ cache; tùy yêu cầu nhất quán từng read.

**E-REP-005** ⭐⭐⭐
"Failover khi master chết diễn ra thế nào? Split-brain là gì?"
> *Dò cái gì:* promote một replica thành master; split-brain = hai node cùng tưởng mình là master → cần fencing/quorum để tránh.

**E-REP-006** ⭐⭐⭐⭐
"Physical (streaming/WAL) vs logical replication ở Postgres khác nhau gì, dùng khi nào?"
> *Dò cái gì:* physical copy byte WAL (bản sao y hệt, cùng version); logical theo row/table (chọn lọc, cross-version, nền cho CDC + nâng cấp zero-downtime).

**E-REP-007** ⭐⭐⭐⭐⭐
"Bạn thêm read replica để 'giảm tải' nhưng latency tổng không cải thiện, còn có chỗ trả sai data. Chẩn đoán từ đâu?"
> *Dò cái gì:* kiểm replication lag, xem read nào bị route nhầm sang replica, đối chiếu consistency requirement từng read; nhận ra replica KHÔNG giải bài toán write/consistency.

---

## E-PG — PostgreSQL specifics

**E-PG-001** ⭐⭐⭐
"JSONB khác JSON (text) ở Postgres thế nào? Khi nào dùng JSONB thay cột quan hệ?"
> *Dò cái gì:* JSONB lưu nhị phân, query + index được (GIN); hợp dữ liệu bán cấu trúc/linh hoạt, KHÔNG nên thay toàn bộ schema quan hệ bằng JSONB.

**E-PG-002** ⭐⭐⭐⭐
"Index cho truy vấn bên trong JSONB dùng gì? Hạn chế?"
> *Dò cái gì:* GIN index (vd `jsonb_path_ops`), hỗ trợ containment `@>`; không nhanh bằng btree trên cột thường cho mọi loại truy vấn (vd range).

**E-PG-003** ⭐⭐⭐⭐
"VACUUM giải quyết vấn đề gì? Vì sao MVCC khiến nó cần thiết?"
> *Dò cái gì:* MVCC để lại dead tuples (version cũ) → VACUUM thu hồi space + cập nhật visibility map; autovacuum chạy nền.

**E-PG-004** ⭐⭐⭐⭐
"Table/index bloat là gì, do đâu? VACUUM thường khác `VACUUM FULL` thế nào?"
> *Dò cái gì:* dead tuples tích tụ làm phình bảng/index; autovacuum dọn nhưng không trả space về OS; `VACUUM FULL` rewrite bảng (khóa độc quyền, đắt).

**E-PG-005** ⭐⭐⭐⭐⭐
"Transaction ID wraparound là gì và vì sao nó có thể khiến Postgres NGỪNG nhận ghi?"
> *Dò cái gì:* XID 32-bit quay vòng; nếu vacuum freeze không kịp → nguy cơ wraparound → Postgres chuyển read-only để tự bảo vệ; phải đảm bảo autovacuum/freeze.

**E-PG-006** ⭐⭐⭐
"Đặt `max_connections` cao (vd 1000) có phải cách scale connection không? Vì sao mỗi connection tốn?"
> *Dò cái gì:* mỗi connection = một backend process tốn RAM/CPU; quá nhiều connection làm chậm; cần pooler thay vì nâng `max_connections`. *(cross-ref E-POOL.)*

**E-PG-007** ⭐⭐⭐⭐
"`ANALYZE` / planner statistics là gì? Vì sao thống kê cũ làm query chậm đột ngột?"
> *Dò cái gì:* planner chọn plan dựa estimate từ stats; stats lệch → chọn sai plan (seq scan thay vì index); cần `ANALYZE`/autoanalyze.

**E-PG-008** ⭐⭐⭐⭐
"Là TL chọn version Postgres cho dự án mới 6/2026: chọn bản nào, tránh bản nào, vì sao?"
> *Dò cái gì:* chọn bản ổn định mới (**17 hoặc 18** — verify); tránh **14** (EOL 11/2026) và **13** (đã EOL); KHÔNG chạy **19 beta** cho production; tiêu chí: ổn định + còn được vá bảo mật.

---

## E-OPT — Query optimization tools

**E-OPT-001** ⭐⭐⭐
"Debug một query chậm — quy trình của bạn bắt đầu từ đâu?"
> *Dò cái gì:* tìm query chậm (`pg_stat_statements`/slow log) → `EXPLAIN ANALYZE` → đọc scan/join/sort → thêm index/sửa query; không đoán mò.

**E-OPT-002** ⭐⭐⭐⭐
"`EXPLAIN` vs `EXPLAIN ANALYZE` khác nhau gì? Vì sao `ANALYZE` 'nguy hiểm' với câu ghi?"
> *Dò cái gì:* `EXPLAIN` chỉ ước lượng; `ANALYZE` chạy thật để đo → với INSERT/UPDATE/DELETE phải bọc transaction + rollback để không đổi data.

**E-OPT-003** ⭐⭐⭐⭐
"Đọc EXPLAIN: Seq Scan vs Index Scan vs Bitmap Index Scan vs Index Only Scan — mỗi cái nghĩa gì?"
> *Dò cái gì:* phân biệt full table, index lookup từng row, bitmap (nhiều rows / kết hợp index), index-only (không chạm heap).

**E-OPT-004** ⭐⭐⭐⭐
"Trong EXPLAIN ANALYZE, vì sao chênh lệch lớn giữa estimated rows và actual rows là dấu hiệu xấu?"
> *Dò cái gì:* planner estimate sai → chọn plan/join sai (vd nested loop khi nên hash join); thường do stats cũ hoặc correlation giữa cột.

**E-OPT-005** ⭐⭐⭐
"`pg_stat_statements` cho biết gì và dùng nó thế nào để tìm 'thủ phạm' tải DB?"
> *Dò cái gì:* gộp query theo mẫu (total_time, calls, mean_time); tìm query tốn TỔNG thời gian nhiều nhất (không chỉ chậm nhất một lần).

**E-OPT-006** ⭐⭐⭐⭐
"Nested loop vs hash join vs merge join — planner chọn dựa trên gì?"
> *Dò cái gì:* kích thước bảng, index sẵn có, dữ liệu đã sort chưa; nested loop tốt khi một bên nhỏ + có index, hash join cho bảng lớn không sort, merge join khi cả hai đã sort.

**E-OPT-007** ⭐⭐⭐⭐⭐
"Một query bỗng chậm hẳn trong production dù code/data không đổi nhiều. Liệt kê nghi phạm và cách xác minh từng cái."
> *Dò cái gì:* stats cũ (chạy `ANALYZE`), bloat, plan flip do data lớn dần, thiếu index sau khi data tăng, cache lạnh, lock/contention; xác minh bằng `EXPLAIN` + `pg_stat_*`.

---

## E-POOL — Connection pooling

**E-POOL-001** ⭐⭐⭐
"Vì sao cần connection pool? Mở connection mới mỗi request tốn những gì?"
> *Dò cái gì:* handshake + auth + tạo backend process tốn; pool tái dùng connection → giảm latency và bảo vệ DB khỏi quá tải connection.

**E-POOL-002** ⭐⭐⭐⭐
"Đặt pool size bao nhiêu? Vì sao 'càng to càng tốt' là sai?"
> *Dò cái gì:* pool quá lớn → DB quá tải (context switch, lock contention) chậm hơn; quy mô theo capacity của DB (vd ~cores×2), đo thực tế, không theo số connection app muốn.

**E-POOL-003** ⭐⭐⭐⭐
"PgBouncer là gì? Ba pooling mode (session/transaction/statement) khác nhau và đánh đổi?"
> *Dò cái gì:* transaction mode tái dùng tốt nhất nhưng cấm session-state (một số prepared statement/`SET`); session mode an toàn nhưng tái dùng kém.

**E-POOL-004** ⭐⭐⭐⭐
"Vì sao serverless (Lambda) + Postgres dễ 'cháy' connection, và giải pháp?"
> *Dò cái gì:* mỗi instance mở connection riêng, scale ngang bùng nổ số connection; cần external pooler (PgBouncer/RDS Proxy) hoặc Data API.

**E-POOL-005** ⭐⭐⭐
"App có pool, DB cũng có giới hạn — quan hệ giữa (pool size mỗi instance × số instance) và `max_connections`?"
> *Dò cái gì:* tổng connection = pool_size × số instance, phải ≤ `max_connections` trừ dự phòng; nhiều instance rất dễ vượt trần DB.

**E-POOL-006** ⭐⭐⭐⭐⭐
"Triệu chứng 'pool exhausted / timeout acquire connection' dưới tải — điều tra theo hướng nào?"
> *Dò cái gì:* query giữ connection lâu (slow query/long tx/N+1), pool quá nhỏ, connection leak (quên release), DB nghẽn; sửa gốc (query/tx) trước khi tăng pool.

---

## E-NEW — NewSQL / distributed SQL

**E-NEW-001** ⭐⭐⭐
"NewSQL / distributed SQL là gì? Nó hứa giải mâu thuẫn nào giữa SQL truyền thống và NoSQL?"
> *Dò cái gì:* SQL + ACID + scale ngang phân tán; tránh phải hi sinh transaction/quan hệ để đổi lấy scale như NoSQL thuần.

**E-NEW-002** ⭐⭐⭐⭐
"Kể vài distributed SQL DB và 'sweet spot' của chúng (verify)."
> *Dò cái gì:* CockroachDB (Postgres-compat, multi-region OLTP), TiDB (MySQL-compat, HTAP), Spanner (TrueTime, GCP), YugabyteDB (Postgres-compat); chọn theo nhu cầu cụ thể. *(verify.)*

**E-NEW-003** ⭐⭐⭐⭐
"Các distributed SQL này đạt strong consistency phân tán bằng gì?"
> *Dò cái gì:* consensus — Raft (CockroachDB/TiDB/YugabyteDB) hoặc Paxos + TrueTime (Spanner); replicate qua consensus để nhất quán mạnh.

**E-NEW-004** ⭐⭐⭐⭐
"Distributed SQL không miễn phí — đánh đổi thực tế so với một Postgres đơn vùng?"
> *Dò cái gì:* latency cao hơn cho transaction cross-node, vận hành phức tạp, chi phí cao; nhiều cluster thực ra không cần multi-region.

**E-NEW-005** ⭐⭐⭐⭐⭐
"TL: khi nào chọn NewSQL thay vì 'Postgres + read replica + sharding ở app'? Tiêu chí quyết định."
> *Dò cái gì:* cần strong consistency ĐA VÙNG + scale ghi vượt 1 node + data residency theo policy; nếu single-region thì managed Postgres đơn giản/rẻ hơn nhiều; không chọn theo hype. *(verify landscape.)*

---

## E-RT — Data modeling depth / quyết định TL

**E-RT-001** ⭐⭐⭐
"Phân biệt OLTP và OLAP về workload và cách thiết kế schema."
> *Dò cái gì:* OLTP nhiều giao dịch nhỏ, chuẩn hóa, low latency; OLAP aggregate lớn, denormalize/columnar, ưu tiên throughput.

**E-RT-002** ⭐⭐⭐⭐
"UUID vs auto-increment (`bigserial`) làm primary key — đánh đổi gì, đặc biệt với index/insert?"
> *Dò cái gì:* UUIDv4 ngẫu nhiên → insert phân tán/fragment B-tree, key to/nặng; `bigserial` gọn + tuần tự; UUIDv7/ULID time-ordered là dung hòa.

**E-RT-003** ⭐⭐⭐⭐
"Soft delete (`deleted_at`) vs hard delete — đánh đổi và cạm bẫy thường gặp?"
> *Dò cái gì:* soft delete giữ lịch sử nhưng phình bảng, mọi query phải lọc, unique constraint vướng; cân nhắc archive table thay thế.

**E-RT-004** ⭐⭐⭐⭐
"Lưu tiền và thời gian đúng cách: vì sao không dùng float cho tiền, và `timestamptz` vs `timestamp`?"
> *Dò cái gì:* float gây sai số → dùng `numeric`/integer (cents); `timestamptz` lưu mốc theo UTC tránh lỗi timezone, `timestamp` không có tz dễ sai.

**E-RT-005** ⭐⭐⭐⭐⭐
"AI sinh schema/migration chạy được nhưng có thể sai kiến trúc — là TL bạn kiểm những gì trước khi merge?"
> *Dò cái gì:* index có khớp access pattern không, kiểu dữ liệu (money/uuid/enum/tz), nullable/constraint/FK, migration có khóa bảng lâu trên bảng lớn không, có rollback được không; *hiểu để kiểm tra*, không chỉ 'chạy được'.

---

## ✅ Sau Bước A
Đây là **97 câu** phủ HẾT 13 mục con của E + nhóm data-modeling/TL. Viết lại bằng lời mình, không chép danh sách nguồn; mỗi câu có mốc chấm riêng; đã chống trùng với `QB_G_nodejs.md` và `QB_H_nestjs.md` (pooling/N+1 nhìn từ phía DB, không lặp Node runtime hay ORM wiring của Nest).

**Việc của bạn:** lưu file này vào project (kéo vào Project files). Khi bạn muốn, tôi sẽ:
- **Bước B** — dựng giáo trình *ngược* từ bộ đề (gom câu → bài học, sắp cơ bản → nâng cao theo dependency order, ánh xạ Bài → ID).
- **Bước C** — giảng từng bài theo Hợp đồng 10 mục.
- **Bước D** — phỏng vấn theo đợt, chấm **live**.

> *Câu thần chú:* **"Tôi không nhớ để gõ, tôi hiểu để chỉ huy — và tra cứu phần còn lại."**
