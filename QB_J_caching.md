# 🧪 QB_J — Ngân hàng câu hỏi: Caching

> **Mục J** trong roadmap Tech Lead Backend (🟡 ~45%) · Sinh theo **WORKFLOW 2 — Bước A**.
> Đây là **mục LỚN (A–R)** → bộ đề phủ HẾT 6 mục con của roadmap (cache strategies, invalidation & TTL, cache stampede, Redis ngoài cache, Redis Cluster/RediSearch, BullMQ/Redis Streams) + mở rộng đúng chiều sâu TL: cache↔DB consistency, penetration/breakdown/avalanche, eviction/memory, HTTP/CDN cache.
> **Chỉ câu hỏi — KHÔNG kèm đáp án.** Mỗi câu có dòng *"dò cái gì"* = tiêu chí ĐẠT, dùng để chấm **live** ở Bước D.
> **Chống trùng:** đã đọc `QB_E_database.md`, `QB_F_apidesign.md`, `QB_G_nodejs.md`, `QB_H_nestjs.md`, `QB_I_concurrency.md`, `QB_L_testing.md`, `QB_M_security.md`, `QB_Q_designpatterns.md`.
> - **I-DLOCK** đã phủ TRỌN distributed lock (SET NX, release-by-token, Redlock/Kleppmann, fencing token, failover) → J **không lặp**; `J-BEYOND-005` chỉ nhận diện Redis-as-lock là một use case và **trỏ về I-DLOCK**.
> - **F-RL** đã giữ góc *thuật toán + thiết kế API* của rate limiting (token bucket, sliding window, gateway vs app) → J chỉ chạm Redis ở **góc store** (`INCR`/zset atomic), cross-ref F/M; không lặp thuật toán.
> - **E** đã giữ *materialized view vs cột denorm vs cache* (góc freshness) + read-from-cache khi replication lag → J nhìn cache ở **góc cache layer** (chiến lược/invalidation/eviction), cross-ref E khi đụng.
> - **G** đã giữ *unbounded in-memory cache = memory leak* + rate-limit store dùng chung Redis → J không lặp wiring Node; **I-RACE/I-IDEM** giữ race shared-data & idempotency → `J-CONSIST` chỉ chạm race ở **góc cache-aside**, cross-ref I.
> - **M** đã giữ session store Redis ở *góc security* → `J-BEYOND-004` nhìn từ *góc cache/eviction*, cross-ref M.
> **Tổng: 88 câu.**

## Cách đọc
- **ID:** `J-<tag>-<số>`. Tag mục con liệt kê ở mục lục dưới.
- **Độ khó:** ⭐ định nghĩa cơ bản → ⭐⭐⭐⭐⭐ phân biệt senior với TL thật.
- **"Dò cái gì":** điều người trả lời PHẢI chạm tới mới tính ĐẠT (không phải đáp án — chỉ là mốc chấm).
- *(verify)* = câu chạm cú pháp/tên/đời sản phẩm (lệnh Redis, danh sách eviction policy, version BullMQ, cơ chế Cluster/Sentinel/module, header HTTP) → **kiểm chứng tại docs chính thức** khi học (đổi nhanh).

## Bối cảnh (tính tại 6/2026)
- Lý thuyết caching (strategies, invalidation, stampede, eviction, consistency) **ổn định** — ít đổi. Phần dễ đổi: **cú pháp & danh sách `maxmemory-policy`**, **lệnh Redis** (SET options, Streams `XADD/XREADGROUP/XACK`), **cơ chế Redis Cluster/Sentinel**, **module RediSearch**, **API/đời BullMQ**.
- **Bộ ba kinh điển hay hỏi** (đặc biệt phỏng vấn theo phong cách Trung Quốc): cache **penetration / breakdown(=stampede) / avalanche** — *nguồn tổng hợp: redis.io/docs + nhiều bài interview (Medium/Javarevisited, HireMe AI).* Phương Tây hay gọi breakdown là **dogpile / thundering herd / cache stampede**.
- **Eviction policy hiện hành** (verify tại redis.io): `noeviction`, `allkeys-lru/-lfu/-random`, `volatile-lru/-lfu/-random/-ttl` (bản mới thêm `allkeys-lrm`). Default thường **volatile-lru**; ElastiCache/Memorystore cũng mặc định volatile-lru.
- **BullMQ** là chuẩn de-facto cho Redis-backed job queue trong Node (kế nhiệm Bull, TypeScript-native); nội bộ tận dụng **Redis Streams**. Khuyến nghị production: **Redis của queue tách khỏi Redis của cache** (eviction need khác nhau). Cần event-streaming + replay + throughput rất lớn → **Kafka**. *(verify version, vd BullMQ 5.x.)*

---

## Mục lục mục con (ID prefix → số câu)

| Tag | Mục con | Số câu |
|---|---|---|
| **J-WHY** | Vì sao cache · tầng cache · local vs distributed · khi nào KHÔNG cache | 7 |
| **J-STRAT** | Cache strategies (aside / read-through / write-through / write-back / refresh-ahead) | 10 |
| **J-INVAL** | Invalidation, TTL, key design, negative caching, stale-while-revalidate | 10 |
| **J-CONSIST** | Cache↔DB consistency (dual-write, delete vs update, ordering, race) | 9 |
| **J-PROB** | Penetration / breakdown(stampede) / avalanche / hot key / big key | 11 |
| **J-EVICT** | Eviction & memory (LRU/LFU, maxmemory-policy, working set) | 7 |
| **J-REDIS** | Redis as cache (single-thread, fast, data structures, RDB/AOF, vs Memcached) | 9 |
| **J-BEYOND** | Redis ngoài cache (rate limit, leaderboard, session, pub/sub, HLL) — cross-ref I/F/M | 6 |
| **J-CLUSTER** | Redis scaling (replica, cluster/hash slots, sentinel, RediSearch) | 6 |
| **J-QUEUE** | BullMQ / Redis Streams (jobs, queue vs streams, vs Kafka) | 7 |
| **J-HTTP** | HTTP / CDN caching (Cache-Control, ETag, CDN, edge) | 6 |

---

## J-WHY — Vì sao cache · tầng cache · khi nào KHÔNG cache

**J-WHY-001** ⭐⭐
"Cache là gì và giải quyết bài toán cốt lõi nào? Tốc độ đến từ đâu?"
> *Dò cái gì:* lưu bản sao dữ liệu ở tầng truy cập nhanh hơn (RAM gần app) để tránh tính lại / đọc lại từ nguồn chậm (DB/disk/API/compute); tốc độ từ locality (temporal/spatial) + tránh I/O; đánh đổi: tốn thêm bộ nhớ + rủi ro stale.

**J-WHY-002** ⭐⭐
"Các tầng cache trong một request web đi từ client tới DB gồm những đâu?"
> *Dò cái gì:* browser cache → CDN/edge → reverse proxy (Nginx/Varnish) → app in-process (in-memory) → distributed cache (Redis/Memcached) → DB buffer pool; mỗi tầng đổi giữa tốc độ ↔ phạm vi chia sẻ ↔ độ tươi.

**J-WHY-003** ⭐⭐⭐
"Local (in-process) cache vs distributed cache (Redis): đánh đổi gì, khi nào dùng cái nào?"
> *Dò cái gì:* local cực nhanh (không network) nhưng KHÔNG share giữa instance → mỗi node một bản, khó invalidate đồng loạt, dễ lệch; distributed share + nhất quán hơn nhưng tốn network hop + thành điểm phụ thuộc; thực tế hay multi-tier (L1 local + L2 Redis). *(cross-ref I-RACE-008 'Map in-RAM không share'.)*

**J-WHY-004** ⭐⭐⭐
"Multi-level cache (L1 in-process + L2 Redis): lợi gì, rủi ro nhất quán gì?"
> *Dò cái gì:* L1 cắt latency & giảm tải L2 cho hot key; rủi ro L1 ở mỗi node có thể stale khác nhau khi L2/DB đổi → cần TTL ngắn cho L1 + cơ chế invalidate (pub/sub broadcast) hoặc chấp nhận lệch nhỏ.

**J-WHY-005** ⭐⭐⭐
"Khi nào KHÔNG nên cache? Cho vài trường hợp cache hại nhiều hơn lợi."
> *Dò cái gì:* data đổi liên tục / đọc-một-lần (hit ratio thấp), path cần strong consistency tuyệt đối (tiền/tồn kho), write-heavy ít đọc, data rẻ để tính lại; cache thêm phức tạp + sinh bug stale → chỉ cache khi đo được lợi.

**J-WHY-006** ⭐⭐⭐⭐
"Chỉ số nào đo 'cache có đáng không'? Hit ratio cao có luôn tốt không?"
> *Dò cái gì:* hit ratio, latency p50/p99, mức giảm tải nguồn (DB QPS), cost/req; hit ratio cao chưa chắc tốt nếu toàn cache thứ rẻ/ít gọi; phải gắn vào thứ mình muốn bảo vệ (DB, latency đuôi), không tối ưu hit ratio mù.

**J-WHY-007** ⭐⭐⭐⭐
"Phân biệt caching với precomputation / materialized view. Khi nào dùng cái nào?"
> *Dò cái gì:* cache = lazy, theo nhu cầu, có thể miss; MV/precompute = chủ động tính sẵn, luôn có nhưng tốn ghi/refresh; chọn theo độ tươi cần + pattern đọc. *(cross-ref E 'MV vs cột denorm vs cache'.)*

---

## J-STRAT — Cache strategies

**J-STRAT-001** ⭐⭐⭐
"Cache-aside (lazy loading) hoạt động thế nào? Mô tả luồng đọc và luồng ghi."
> *Dò cái gì:* đọc: app check cache, miss → đọc DB → ghi vào cache → trả; ghi: cập nhật DB rồi invalidate/update cache; app tự quản (cache không biết DB); phổ biến nhất; nhược: lần đầu luôn miss + có window stale.

**J-STRAT-002** ⭐⭐⭐
"Read-through cache khác cache-aside ở đâu?"
> *Dò cái gì:* read-through để cache layer/lib tự load từ DB khi miss, app chỉ nói chuyện với cache; cache-aside app tự load; read-through gọn code nhưng cần provider/abstraction hỗ trợ.

**J-STRAT-003** ⭐⭐⭐
"Write-through cache là gì? Đánh đổi?"
> *Dò cái gì:* mỗi write ghi đồng bộ cả cache lẫn DB (qua cache layer) → cache luôn tươi, đọc nhất quán hơn; trả giá: write latency cao hơn + cache chứa cả data không bao giờ đọc lại.

**J-STRAT-004** ⭐⭐⭐⭐
"Write-back (write-behind) là gì? Lợi và rủi ro chí mạng?"
> *Dò cái gì:* write vào cache trước, flush xuống DB async/batch sau → write nhanh, gộp ghi; rủi ro **mất data** nếu cache chết trước khi flush → cần persistence/replication; chỉ hợp khi chấp nhận mất mát nhỏ.

**J-STRAT-005** ⭐⭐⭐⭐
"Refresh-ahead (refresh-on-expiry) giải quyết vấn đề gì? Hạn chế?"
> *Dò cái gì:* chủ động làm mới key nóng *trước* khi hết hạn (dự đoán sắp được đọc) → tránh miss/latency spike lúc expiry; hạn chế: refresh nhầm key không còn được đọc → phí tài nguyên.

**J-STRAT-006** ⭐⭐⭐⭐
"So sánh cache-aside vs write-through về độ tươi và độ phức tạp — chọn cái nào khi nào?"
> *Dò cái gì:* cache-aside đơn giản, chịu được cache chết (chỉ tăng miss) nhưng có window stale; write-through tươi hơn cho read nhưng ghi chậm + phụ thuộc cache; chọn theo read/write ratio + yêu cầu tươi.

**J-STRAT-007** ⭐⭐⭐⭐
"Vì sao cache-aside là mặc định công nghiệp dù có nhược điểm?"
> *Dò cái gì:* đơn giản, resilient (cache down → degrade chứ không chết, đọc thẳng DB), không khoá app vào provider; nhược (miss đầu, stale window) chấp nhận được với TTL hợp lý.

**J-STRAT-008** ⭐⭐⭐
"Cold start / cache lạnh là gì? Cache warming giải quyết thế nào, hạn chế gì?"
> *Dò cái gì:* cache rỗng sau deploy/restart → loạt miss đập DB; warming = preload hot key trước khi nhận traffic; hạn chế: không biết hết hot key, không cứu mass expiry về sau (cần thêm TTL jitter/lock).

**J-STRAT-009** ⭐⭐⭐⭐
"Nên cache cái gì: raw row, computed result, hay rendered fragment? Cân nhắc gì?"
> *Dò cái gì:* cache càng gần kết quả cuối càng cắt nhiều compute nhưng càng dễ vỡ khi 1 phần đổi (invalidate khó hơn) + trùng lặp; cache thấp (row) tái dùng cao nhưng còn compute; chọn theo cost-tính-lại vs tần suất đổi.

**J-STRAT-010** ⭐⭐⭐⭐⭐
"AI sinh code dùng write-back cho dữ liệu thanh toán 'cho nhanh'. Vì sao là lỗi kiến trúc, sửa sao?"
> *Dò cái gì:* write-back có thể mất write khi cache chết → mất giao dịch tiền (không chấp nhận); path correctness phải write-through / ghi-DB-trước; nhận ra 'nhanh' không bù rủi ro mất tiền — đúng chỗ TL phải bắt. *(nguyên tắc hiểu-để-kiểm-tra.)*

---

## J-INVAL — Invalidation, TTL, key design

**J-INVAL-001** ⭐⭐
"Vì sao 'cache invalidation' nổi tiếng là một trong hai việc khó nhất? Khó ở đâu?"
> *Dò cái gì:* khó biết *khi nào* + *cái gì* cần xoá khi nguồn đổi, nhất là với data dẫn xuất/aggregate phụ thuộc nhiều nguồn; xoá thiếu → stale, xoá thừa → mất hiệu quả; phối hợp giữa nhiều writer/instance.

**J-INVAL-002** ⭐⭐⭐
"TTL-based expiry vs explicit invalidation: đánh đổi gì, khi nào dùng cái nào?"
> *Dò cái gì:* TTL đơn giản, tự dọn, chấp nhận stale tối đa = TTL; explicit (xoá khi write) tươi hơn nhưng phải bắt được mọi đường ghi + đúng key; thực tế kết hợp: explicit + TTL làm lưới an toàn.

**J-INVAL-003** ⭐⭐⭐
"Đặt TTL ngắn vs dài: đánh đổi gì?"
> *Dò cái gì:* ngắn → tươi hơn nhưng hit ratio thấp + tải DB cao hơn; dài → hiệu quả cache cao nhưng stale lâu; chọn theo độ chấp nhận stale của data + chi phí miss.

**J-INVAL-004** ⭐⭐⭐⭐
"Thiết kế cache key đúng cần lưu ý gì? Cho ví dụ một key tồi."
> *Dò cái gì:* key xác định duy nhất + gồm mọi tham số ảnh hưởng kết quả (user/tenant, locale, filter, version schema); ví dụ tồi: quên kèm tenant/user → trả nhầm data giữa người dùng (rò dữ liệu/quyền); cần namespace rõ.

**J-INVAL-005** ⭐⭐⭐⭐
"Versioned key / cache busting là gì? Vì sao thường tốt hơn xoá key thủ công?"
> *Dò cái gì:* nhúng version (hoặc đổi prefix) vào key → đổi version = vô hiệu toàn bộ bản cũ NGAY mà không cần DEL từng key; bản cũ tự rơi theo eviction/TTL; tránh race xoá-rồi-đọc-lại.

**J-INVAL-006** ⭐⭐⭐⭐
"Invalidate một list/aggregate khi một phần tử đổi — vì sao khó, cách tiếp cận?"
> *Dò cái gì:* một thay đổi có thể ảnh hưởng nhiều key dẫn xuất (list, count, page); cách: tag/group key, versioned namespace, hoặc TTL ngắn cho aggregate; tránh phải truy mọi key bằng tay.

**J-INVAL-007** ⭐⭐⭐⭐
"Event-based invalidation (qua message/CDC) khác invalidate ngay trong code ghi thế nào?"
> *Dò cái gì:* phát event khi DB đổi → consumer xoá cache; tách concern + bắt được cả ghi từ nguồn khác (không qua app); gắn với Outbox/CDC để không 'ghi DB ok nhưng publish fail'. *(cross-ref K; J-CONSIST-009.)*

**J-INVAL-008** ⭐⭐⭐⭐⭐
"Negative caching (cache kết quả 'không tồn tại') để làm gì, rủi ro gì?"
> *Dò cái gì:* cache null/empty marker để chặn penetration (query data không tồn tại đập DB); rủi ro: nếu sau đó data được tạo, null cache che mất → cần TTL ngắn cho null + invalidate khi create. *(cross-ref J-PROB-002.)*

**J-INVAL-009** ⭐⭐⭐
"Vì sao 'không bao giờ đặt TTL, chỉ invalidate thủ công' là chiến lược nguy hiểm?"
> *Dò cái gì:* lỡ một đường ghi không invalidate → stale vĩnh viễn (không có lưới an toàn); TTL đảm bảo chặn stale tối đa; luôn nên có TTL backstop kể cả khi đã explicit invalidate.

**J-INVAL-010** ⭐⭐⭐⭐
"Stale-while-revalidate là gì? Lợi ích cho latency đuôi?"
> *Dò cái gì:* trả bản stale ngay cho client trong khi background làm mới → không ai phải chờ DB lúc expiry; chấp nhận stale ngắn để bỏ latency spike; phổ biến ở HTTP cache/CDN & app cache. *(verify cú pháp header.)*

---

## J-CONSIST — Cache↔DB consistency

**J-CONSIST-001** ⭐⭐⭐
"Vì sao về bản chất cache + DB là bài toán dual-write, không thể nhất quán tuyệt đối miễn phí?"
> *Dò cái gì:* cập nhật hai store không-atomic → luôn có cửa sổ một cái đổi cái kia chưa; mục tiêu thực tế là thu nhỏ & kiểm soát window stale, không phủ nhận nó; gắn với CAP/eventual. *(cross-ref I-CONS.)*

**J-CONSIST-002** ⭐⭐⭐⭐
"Cache-aside khi write: nên 'update cache' hay 'delete cache'? Vì sao delete thường an toàn hơn?"
> *Dò cái gì:* update-cache dễ race (hai writer ghi cache lệch thứ tự với DB → stale bền) + phí ghi giá trị có thể không bao giờ đọc; delete (invalidate) để lần đọc sau nạp lại từ DB; delete đơn giản & ít sai hơn.

**J-CONSIST-003** ⭐⭐⭐⭐⭐
"'Update DB rồi delete cache' vs 'delete cache rồi update DB' — mỗi thứ tự có race gì?"
> *Dò cái gì:* delete-trước-update: giữa lúc xoá và DB commit, một reader nạp lại giá trị CŨ vào cache → stale bền; update-trước-delete an toàn hơn nhưng vẫn có race hiếm (reader đọc DB cũ trước commit, ghi cache sau khi delete); cả hai KHÔNG kín tuyệt đối.

**J-CONSIST-004** ⭐⭐⭐⭐⭐
"Delayed double delete (xoá cache hai lần) giải race nào? Hạn chế?"
> *Dò cái gì:* xoá cache trước + sau update (lần 2 sau một delay ngắn) để dọn giá trị cũ mà reader có thể đã nạp vào trong cửa sổ race; hạn chế: chọn delay khó, vẫn xác suất, thêm phức tạp; chỉ giảm chứ không triệt tiêu.

**J-CONSIST-005** ⭐⭐⭐⭐
"Cache-aside có race kinh điển: read-miss đồng thời với một write. Mô tả và cách giảm."
> *Dò cái gì:* reader miss đọc DB (giá trị cũ) → trước khi nó ghi cache, writer update DB + delete cache → reader ghi giá trị cũ đè lên → stale; giảm bằng versioned/CAS khi set cache, double-delete, hoặc TTL ngắn. *(cross-ref I-RACE check-then-act.)*

**J-CONSIST-006** ⭐⭐⭐⭐
"Vì sao chỉ dựa 'write-through để luôn nhất quán' vẫn có thể stale ở môi trường phân tán?"
> *Dò cái gì:* write-through nhất quán cache↔DB *qua cùng layer*, nhưng nhiều cache node / read replica DB / đường ghi không qua cache vẫn gây lệch; consistency còn phụ thuộc replication & path ghi khác. *(cross-ref I-CONS, E-REP.)*

**J-CONSIST-007** ⭐⭐⭐⭐
"Khi nào chấp nhận stale từ cache, khi nào tuyệt đối không? Gắn vào nghiệp vụ."
> *Dò cái gì:* hiển thị phụ (đếm view, gợi ý) → stale ok; quyết định tiền/tồn kho/quyền → đọc nguồn (DB) hoặc bỏ cache cho path đó; chọn theo hậu quả sai, không cache đồng loạt. *(cross-ref I-CONS-009.)*

**J-CONSIST-008** ⭐⭐⭐⭐
"Read-your-own-writes bị vỡ vì cache như thế nào? Cách giữ?"
> *Dò cái gì:* user vừa update nhưng đọc trúng cache cũ → 'không thấy thay đổi của mình'; fix: invalidate ngay sau write của họ, hoặc bypass cache cho read ngay sau write của chính user. *(cross-ref I-CONS-005.)*

**J-CONSIST-009** ⭐⭐⭐⭐⭐
"CDC/Outbox giữ cache đồng bộ DB thế nào, và vì sao tin cậy hơn xoá cache trong code?"
> *Dò cái gì:* bắt thay đổi từ commit log DB (Debezium) → phát event → invalidate cache; đảm bảo MỌI commit (kể cả ghi ngoài app) đều kích hoạt invalidate, tránh 'commit DB ok nhưng quên/lỗi xoá cache'. *(cross-ref K Outbox/CDC.)* *(verify.)*

---

## J-PROB — Penetration / breakdown(stampede) / avalanche / hot key / big key

**J-PROB-001** ⭐⭐⭐
"Phân biệt rõ cache penetration, cache breakdown (stampede), cache avalanche."
> *Dò cái gì:* penetration = query data KHÔNG tồn tại (cache lẫn DB) → mọi lần đập DB; breakdown/stampede = MỘT hot key hết hạn → loạt request đồng thời nạp lại đập DB; avalanche = NHIỀU key hết hạn cùng lúc (hoặc Redis chết) → DB ngập; đừng lẫn ba khái niệm.

**J-PROB-002** ⭐⭐⭐⭐
"Cache penetration: nguyên nhân và 2 cách phòng chính."
> *Dò cái gì:* request data không tồn tại (id rác / tấn công) không bao giờ điền cache → luôn xuống DB; phòng: (a) cache null/empty marker TTL ngắn, (b) Bloom filter chặn key chắc-chắn-không-tồn-tại trước khi chạm DB; thêm validate input.

**J-PROB-003** ⭐⭐⭐⭐
"Bloom filter chống penetration thế nào? Đánh đổi cố hữu của nó?"
> *Dò cái gì:* membership xác suất: 'chắc chắn không có' (chặn sớm) hoặc 'có thể có' (cho qua); có **false positive** (cho qua nhầm, chấp nhận được), KHÔNG false negative; tốn ít bộ nhớ; nhược: khó xoá phần tử (cần counting bloom). *(verify.)*

**J-PROB-004** ⭐⭐⭐⭐
"Cache breakdown/stampede (dogpile/thundering herd): mô tả và cách chống bằng mutex/single-flight."
> *Dò cái gì:* hot key expiry → N request cùng miss cùng nạp DB; single-flight: chỉ 1 request lấy lock (`SET NX`) rebuild, số còn lại chờ/đọc stale rồi lấy từ cache → giảm từ N query DB còn 1. *(verify cú pháp; cross-ref I-DLOCK cho lock đúng.)*

**J-PROB-005** ⭐⭐⭐⭐
"Probabilistic early expiration / 'logical TTL' chống stampede ra sao, lợi hơn mutex chỗ nào?"
> *Dò cái gì:* refresh sớm ngẫu nhiên trước hạn (xác suất tăng khi gần expiry) → một request lẻ làm mới nền trước khi key thật sự hết → không ai cùng miss; không cần lock toàn cục, tránh nghẽn ở mutex; biến thể 'never expire + async refresh'.

**J-PROB-006** ⭐⭐⭐⭐
"Cache avalanche do mass expiry: vì sao xảy ra, chống bằng TTL jitter thế nào?"
> *Dò cái gì:* nhiều key set cùng TTL (warm cùng lúc / TTL cố định) → hết hạn đồng loạt → DB sốc; jitter = TTL base + random offset để rải thời điểm hết hạn, tránh đồng pha.

**J-PROB-007** ⭐⭐⭐⭐⭐
"Cache avalanche do Redis sập (không phải expiry): phòng thủ nhiều lớp gồm gì?"
> *Dò cái gì:* HA (replica/sentinel/cluster), circuit breaker + rate limit bảo vệ DB khi cache mất, local L1 fallback, graceful degradation (trả stale/partial); không để DB nhận full traffic trần. *(cross-ref M DDoS, K circuit breaker.)*

**J-PROB-008** ⭐⭐⭐⭐
"Hot key (single key nhận quá nhiều traffic) gây vấn đề gì ngay cả khi cache hit? Giảm sao?"
> *Dò cái gì:* một shard/node Redis giữ hot key thành bottleneck (CPU/băng thông node đó) dù hit 100%; giảm: đọc từ replica, local cache hot key ở app, key splitting (chia N bản), client-side cache.

**J-PROB-009** ⭐⭐⭐⭐
"Big key (value quá lớn / collection khổng lồ) trong Redis gây hại gì?"
> *Dò cái gì:* thao tác trên big key block single-thread (O(n)) → tăng latency mọi client; tốn mạng khi truyền; lệch memory giữa shard; nguy hiểm khi DEL/expire (blocking) → dùng `UNLINK`/scan-từng-phần, tách nhỏ. *(verify.)*

**J-PROB-010** ⭐⭐⭐
"Vì sao chống stampede cho hot key cần khác chống avalanche cho nhiều key?"
> *Dò cái gì:* stampede = 1 key → giải bằng single-flight/lock per-key; avalanche = nhiều key/đồng loạt → giải bằng rải TTL + HA + bảo vệ DB tổng thể; nhầm phương án sẽ không chữa đúng bệnh.

**J-PROB-011** ⭐⭐⭐⭐⭐
"Tấn công penetration cố ý (random id) khác data-not-found ngẫu nhiên thế nào về phòng thủ?"
> *Dò cái gì:* adversary sinh key đa dạng → null-cache có thể phình (đẻ vô số null key) → cần Bloom filter / validate format id / rate limit theo client; null-cache đơn thuần KHÔNG đủ trước tấn công có chủ đích.

---

## J-EVICT — Eviction & memory

**J-EVICT-001** ⭐⭐
"Eviction xảy ra khi nào, khác expiration (TTL) thế nào?"
> *Dò cái gì:* expiration = key tự hết hạn theo TTL; eviction = khi memory chạm `maxmemory`, Redis chủ động xoá key (theo policy) để nhường chỗ; hai cơ chế độc lập; eviction có thể xoá cả key chưa hết hạn.

**J-EVICT-002** ⭐⭐⭐
"Liệt kê các `maxmemory-policy` chính của Redis và ý nghĩa."
> *Dò cái gì:* `noeviction` (lỗi khi đầy), `allkeys-lru/-lfu/-random` (xoá trong mọi key), `volatile-lru/-lfu/-random/-ttl` (chỉ xoá key có TTL); chọn theo: có chịu được eviction không + mọi key có TTL không + pattern recency/frequency. *(verify danh sách tại redis.io.)*

**J-EVICT-003** ⭐⭐⭐
"LRU vs LFU khác nhau ở triết lý? Khi nào LFU thắng?"
> *Dò cái gì:* LRU đuổi 'lâu không dùng' (recency); LFU đuổi 'ít dùng' (frequency); LFU thắng khi hot key bị một burst đọc-một-lần đẩy ra oan (LRU dễ bị cache pollution bởi scan). *(verify Redis LFU approx.)*

**J-EVICT-004** ⭐⭐⭐⭐
"Vì sao Redis LRU/LFU là 'approximated', và `maxmemory-samples` ảnh hưởng gì?"
> *Dò cái gì:* Redis không giữ danh sách LRU toàn cục (tốn) mà sample ngẫu nhiên vài key rồi chọn nạn nhân; tăng samples → chính xác hơn nhưng tốn CPU; đánh đổi accuracy ↔ overhead. *(verify.)*

**J-EVICT-005** ⭐⭐⭐⭐
"Dùng cùng instance Redis vừa làm cache vừa làm store bền (session/queue) với cùng eviction policy — vì sao nguy hiểm?"
> *Dò cái gì:* `allkeys-*` có thể xoá nhầm key 'không được mất' (session/job); cache cần evict, store bền cần `noeviction`/persistence → tách instance hoặc tách logic. *(cross-ref J-QUEUE-007, J-BEYOND-004.)*

**J-EVICT-006** ⭐⭐⭐⭐
"Hit ratio tụt mạnh sau khi bật LRU eviction — chẩn đoán gì?"
> *Dò cái gì:* working set > memory khả dụng → cache thrash (đuổi key sắp dùng lại); giải: tăng memory, giảm thứ cache (TTL/đừng cache thứ rẻ), tách nóng/lạnh, hoặc shard thêm; đo working set vs maxmemory.

**J-EVICT-007** ⭐⭐⭐⭐
"Memory overhead/fragmentation trong Redis: vì sao 'data 1GB' chiếm hơn 1GB RAM?"
> *Dò cái gì:* overhead per-key (metadata, expire, pointer), encoding nội bộ, fragmentation của allocator (jemalloc); cần đệm dung lượng + theo dõi `INFO memory` (used vs rss + frag ratio); không đặt maxmemory sát mép RAM. *(verify.)*

---

## J-REDIS — Redis as cache

**J-REDIS-001** ⭐⭐⭐
"Vì sao Redis nhanh dù single-threaded (cho command execution)?"
> *Dò cái gì:* in-memory (không disk I/O trên đường nóng), single-thread tránh lock/context-switch + thao tác atomic không cần khoá, I/O multiplexing (epoll), data structure tối ưu; mạng/persistence mới là phần có thể chậm. *(verify: I/O threads ở bản mới.)*

**J-REDIS-002** ⭐⭐⭐⭐
"Redis single-thread có hệ quả vận hành gì mà dev hay quên?"
> *Dò cái gì:* một command chậm O(n) (`KEYS *`, big-key op, Lua nặng) block MỌI client → dùng `SCAN` thay `KEYS`, tránh big key, không chạy lệnh nặng trên hot path; throughput giới hạn ~1 core (scale = shard). *(cross-ref J-PROB-009.)*

**J-REDIS-003** ⭐⭐⭐
"Redis vs Memcached làm cache: chọn cái nào khi nào?"
> *Dò cái gì:* Memcached đơn giản, multi-thread, chỉ key-value string → tốt cho cache thuần đơn giản; Redis giàu data structure (hash/zset/stream), persistence, replication, pub/sub, scripting → đa năng hơn; phần lớn case mới chọn Redis. *(verify.)*

**J-REDIS-004** ⭐⭐⭐⭐
"Kể vài data structure Redis ngoài string và một use case cache/đời thật cho mỗi cái."
> *Dò cái gì:* hash (object/field), sorted set (leaderboard / rate-limit theo time), set (unique/tag), list (queue đơn giản), stream (event/job), bitmap/HyperLogLog (đếm xấp xỉ); chọn structure đúng cắt được nhiều logic. *(verify.)*

**J-REDIS-005** ⭐⭐⭐
"Redis persistence RDB vs AOF: khác gì, ảnh hưởng gì khi Redis chỉ là cache thuần?"
> *Dò cái gì:* RDB = snapshot định kỳ (gọn, mất data từ snapshot cuối), AOF = ghi log mỗi write (bền hơn, file lớn/chậm hơn); cache thuần có thể tắt persistence (data tái tạo được) để nhanh; store bền thì cần AOF/RDB. *(verify.)*

**J-REDIS-006** ⭐⭐⭐⭐
"Pipelining tăng throughput nhờ đâu? Khác transaction (MULTI/EXEC) thế nào?"
> *Dò cái gì:* pipeline gộp nhiều command gửi 1 lượt → cắt round-trip mạng (RTT), KHÔNG đảm bảo atomic; MULTI/EXEC nhóm atomic (không xen) nhưng không rollback như SQL; hai mục đích khác nhau. *(verify.)*

**J-REDIS-007** ⭐⭐⭐⭐
"Lua script trên Redis dùng làm gì trong ngữ cảnh cache/lock? Rủi ro?"
> *Dò cái gì:* chạy nhiều thao tác atomic phía server (vd 'compare-token rồi DEL' khi release lock, rate-limit atomic) → tránh race check-then-act; rủi ro: script chạy lâu block single-thread → giữ ngắn. *(cross-ref I-DLOCK-003.)*

**J-REDIS-008** ⭐⭐⭐
"Đặt TTL cho cache key bằng cách nào, và vì sao 'quên TTL' là bug phổ biến?"
> *Dò cái gì:* `SET key val EX <giây>` / `PX <ms>` hoặc `EXPIRE`; quên TTL → key tồn mãi, memory phình, dễ chạm maxmemory + giữ data stale; mọi cache value nên có TTL trừ khi có lý do rõ. *(verify cú pháp.)*

**J-REDIS-009** ⭐⭐⭐⭐⭐
"AI viết code cache trả về object đã serialize nhưng dùng CÙNG key cho mọi user. Lỗi gì, kiểm tra ở đâu?"
> *Dò cái gì:* thiếu chiều phân biệt (user/tenant/locale/permission) trong key → trả data của người này cho người khác (rò dữ liệu/quyền); kiểm tra: key có gồm đủ context ảnh hưởng kết quả không; đây là lỗi correctness/security AI hay tạo. *(nguyên tắc hiểu-để-kiểm-tra; cross-ref J-INVAL-004.)*

---

## J-BEYOND — Redis ngoài cache (cross-ref I / F / M)

**J-BEYOND-001** ⭐⭐⭐
"Ngoài cache, Redis hay được dùng làm những gì? Vì sao cùng một hạ tầng làm được nhiều việc?"
> *Dò cái gì:* rate limiter, distributed lock, session store, leaderboard (zset), pub/sub, queue/streams, geospatial, đếm xấp xỉ (HLL); vì in-memory nhanh + atomic ops + đã có sẵn; nhưng mỗi mục cần lưu ý riêng (eviction, persistence).

**J-BEYOND-002** ⭐⭐⭐
"Redis làm rate limiter ở mức store thế nào? (thuật toán chi tiết ở mục API)"
> *Dò cái gì:* `INCR` + `EXPIRE` (fixed window) hoặc sorted set/Lua (sliding window) để đếm atomic, share state across instance; *thuật toán token bucket/sliding window & đặt ở gateway vs app: cross-ref F-RL, M.* *(verify cú pháp.)*

**J-BEYOND-003** ⭐⭐⭐
"Vì sao sorted set (zset) hợp làm leaderboard / top-N realtime?"
> *Dò cái gì:* zset giữ phần tử theo score có thứ tự; `ZADD`/`ZINCRBY` O(log n), `ZREVRANGE` lấy top-N nhanh, `ZRANK` lấy hạng; không phải sort lại toàn bộ mỗi lần như DB. *(verify.)*

**J-BEYOND-004** ⭐⭐⭐
"Redis làm session store cho app scale ngang — lợi gì, eviction policy phải thế nào?"
> *Dò cái gì:* session share giữa instance (không cần sticky), nhanh; nhưng session 'không được mất' → KHÔNG để `allkeys-*` đuổi nhầm; TTL = session timeout + policy phù hợp/instance riêng. *(cross-ref M session, J-EVICT-005.)*

**J-BEYOND-005** ⭐⭐⭐⭐
"Distributed lock bằng Redis — nêu ý chính và vì sao 'không đơn giản'. (chi tiết ở mục Concurrency)"
> *Dò cái gì:* `SET NX PX` + token, release bằng compare-token (Lua); cạm bẫy TTL/failover/fencing → *toàn bộ phân tích Redlock/fencing nằm ở **I-DLOCK***; ở đây chỉ nhận diện Redis-as-lock là một use case và biết nó có cạm bẫy. *(cross-ref I-DLOCK-002/005/006.)*

**J-BEYOND-006** ⭐⭐⭐⭐
"HyperLogLog / bitmap để đếm 'unique visitors' — vì sao dùng thay vì set thường?"
> *Dò cái gì:* HLL ước lượng cardinality với bộ nhớ cố định cực nhỏ (sai số ~%) thay vì lưu mọi phần tử (set tốn O(n)); bitmap cho đếm/flag theo user-id liên tục; chấp nhận xấp xỉ đổi lấy memory. *(verify.)*

---

## J-CLUSTER — Redis scaling

**J-CLUSTER-001** ⭐⭐⭐
"Khi nào một Redis node hết đủ và phải scale? Các hướng scale?"
> *Dò cái gì:* hết khi vượt RAM 1 node hoặc throughput > ~1 core / băng thông; hướng: replica (scale đọc + HA), cluster/sharding (scale ghi + dung lượng), client-side sharding; nhận diện nghẽn (memory vs CPU vs network) trước.

**J-CLUSTER-002** ⭐⭐⭐⭐
"Redis Cluster chia dữ liệu thế nào (hash slots)?"
> *Dò cái gì:* 16384 hash slot; key → CRC16 mod 16384 → slot → node giữ slot đó; reshard = di chuyển slot giữa node; cho phép kiểm soát phân bố. *(verify số slot/cơ chế tại redis.io.)*

**J-CLUSTER-003** ⭐⭐⭐⭐
"Multi-key operation trong Redis Cluster bị giới hạn gì? 'Hash tag' giải quyết sao?"
> *Dò cái gì:* lệnh đụng nhiều key phải cùng slot/node (không cross-slot); hash tag `{...}` ép các key liên quan vào cùng slot để dùng chung transaction/Lua/MGET; thiết kế key phải tính trước. *(verify.)*

**J-CLUSTER-004** ⭐⭐⭐
"Redis Sentinel vs Redis Cluster giải bài toán khác nhau gì?"
> *Dò cái gì:* Sentinel = HA/failover cho mô hình master-replica (không shard) — tự bầu master mới; Cluster = sharding + HA cho dữ liệu vượt 1 node; chọn theo cần shard hay chỉ cần failover. *(verify.)*

**J-CLUSTER-005** ⭐⭐⭐⭐
"Replication Redis là async — hệ quả gì cho cache và cho lock?"
> *Dò cái gì:* write trên master có thể chưa sang replica khi failover → mất vài write gần nhất; cache thường chấp nhận được; với lock/correctness thì nguy hiểm (2 client cùng giữ lock). *(cross-ref I-DLOCK-011.)* *(verify.)*

**J-CLUSTER-006** ⭐⭐⭐
"RediSearch / Redis modules mở rộng Redis làm gì? Khi nào cân nhắc?"
> *Dò cái gì:* thêm secondary index / full-text / vector search / aggregation trên data đã ở Redis → query phức tạp hơn key-value; cân nhắc khi cần search nhanh trên data đang ở Redis, nhưng không thay full search engine. *(verify tên/khả năng module tại redis.io.)*

---

## J-QUEUE — BullMQ / Redis Streams

**J-QUEUE-001** ⭐⭐⭐
"Vì sao cần background job queue? Ba lớp vấn đề nó giải."
> *Dò cái gì:* tách việc chậm khỏi request (transcode/email) để không treo HTTP; làm phẳng spike (buffer khi burst vượt downstream); retry việc lỗi tạm thời thay vì fail cả request; decouple producer/consumer + scale worker riêng.

**J-QUEUE-002** ⭐⭐⭐
"BullMQ là gì, dựa trên đâu, cung cấp primitives gì cho job?"
> *Dò cái gì:* thư viện queue Node trên Redis (kế nhiệm Bull, TypeScript-native); job states (waiting/active/delayed/completed/failed), retry + backoff, delay, priority, rate limit, repeatable, parent-child flow, dashboard (Bull Board). *(verify version, vd BullMQ 5.x.)*

**J-QUEUE-003** ⭐⭐⭐⭐
"Redis Streams khác list-based queue (LPUSH/BRPOP) thế nào? Consumer group cho gì?"
> *Dò cái gì:* stream = append-only log có ID, hỗ trợ replay + nhiều consumer group đọc độc lập; consumer group chia message giữa workers + theo dõi pending (`XACK`) cho at-least-once + reclaim job của worker chết; list đơn giản hơn nhưng không replay/không group. *(verify `XADD/XREADGROUP/XACK`.)*

**J-QUEUE-004** ⭐⭐⭐⭐
"Đảm bảo job không mất khi worker crash giữa chừng — cơ chế nào?"
> *Dò cái gì:* job phải được claim atomic + đánh dấu đang xử lý, ACK chỉ khi xong; worker chết → job pending được reclaim/retry (visibility timeout / stalled-job check); at-least-once → consumer phải idempotent. *(cross-ref I-IDEM.)*

**J-QUEUE-005** ⭐⭐⭐⭐
"Dead letter queue (DLQ) / failed jobs để làm gì? Thiết kế retry sao cho an toàn?"
> *Dò cái gì:* job fail quá N lần → đẩy sang DLQ/failed để điều tra thay vì retry vô hạn; retry có giới hạn + (jittered) backoff; job phải idempotent vì retry sẽ chạy lại. *(cross-ref I-LOCK-008.)*

**J-QUEUE-006** ⭐⭐⭐⭐⭐
"Khi nào BullMQ/Redis Streams KHÔNG đủ và nên dùng Kafka/RabbitMQ? (mục Messaging đào sâu)"
> *Dò cái gì:* cần event streaming bền + replay lâu dài + throughput rất lớn + nhiều consumer độc lập → Kafka; routing phức tạp/AMQP → RabbitMQ; BullMQ hợp job queue Node với retry/delay, hạ tầng nhẹ (đã có Redis). *(cross-ref K.)* *(verify landscape.)*

**J-QUEUE-007** ⭐⭐⭐⭐
"Vì sao nên tách Redis của queue khỏi Redis của cache?"
> *Dò cái gì:* cache cần eviction (`allkeys-lru`), còn job 'không được mất' → cần `noeviction`/persistence; tải & failure mode khác nhau; chung instance → eviction xoá nhầm job hoặc job ngốn memory đẩy cache. *(cross-ref J-EVICT-005.)*

---

## J-HTTP — HTTP / CDN caching

**J-HTTP-001** ⭐⭐⭐
"`Cache-Control` là gì? Phân biệt `max-age`, `s-maxage`, `no-store`, `no-cache`, `private`/`public`."
> *Dò cái gì:* header điều khiển cache HTTP; `max-age` (TTL client), `s-maxage` (cho shared cache/CDN), `no-store` (không lưu gì), `no-cache` (lưu nhưng phải revalidate trước khi dùng), `private` (chỉ browser) vs `public` (CDN được cache); lẫn no-cache↔no-store là bẫy. *(verify.)*

**J-HTTP-002** ⭐⭐⭐⭐
"`ETag` + `If-None-Match` hoạt động thế nào? Lợi ích so với chỉ `max-age`?"
> *Dò cái gì:* server gắn ETag (fingerprint nội dung); client gửi lại `If-None-Match` → khớp thì server trả `304 Not Modified` (không gửi body) → tiết kiệm băng thông; revalidation giữ tươi mà vẫn cache. *(cross-ref `Last-Modified`/`If-Modified-Since`.)* *(verify.)*

**J-HTTP-003** ⭐⭐⭐
"CDN cache giải quyết gì mà app cache (Redis) không? Đặt cái gì lên CDN?"
> *Dò cái gì:* CDN cache ở edge gần user → cắt latency địa lý + offload origin cho static/asset/response cacheable; Redis cache ở data center cho dynamic/shared state; static/ảnh/JS/CSS/response GET công khai → CDN.

**J-HTTP-004** ⭐⭐⭐⭐
"Vì sao cache HTTP cho GraphQL/POST khó hơn REST GET?"
> *Dò cái gì:* HTTP/CDN cache theo URL+method, GET idempotent/cacheable; GraphQL thường 1 endpoint POST body khác nhau → mất HTTP cache → phải cache normalized ở client hoặc persisted query + GET. *(cross-ref F GraphQL cache.)*

**J-HTTP-005** ⭐⭐⭐⭐
"CDN cache invalidation/purge khó ở đâu? `Vary` ảnh hưởng cache key thế nào?"
> *Dò cái gì:* purge toàn cầu nhiều PoP có độ trễ + tốn → nên versioned URL (content hash) thay vì purge; `Vary` (theo header như Accept-Encoding/Authorization) chia nhỏ cache → đặt sai gây miss nhiều hoặc rò cache giữa user. *(verify.)*

**J-HTTP-006** ⭐⭐⭐⭐
"Cache một response phụ thuộc user (cá nhân hoá) ở CDN — vì sao nguy hiểm, làm sao đúng?"
> *Dò cái gì:* cache public một response có data riêng → rò cho user khác (do `Cache-Control: private` bị đặt sai/bỏ qua); cá nhân hoá nên `private`/`no-store`, hoặc cache theo key gồm user, hoặc tách phần public (cache) khỏi phần riêng (không cache / ESI).

---

## ✅ Sau khi LƯU file này vào project — bước tiếp theo

1. **Bước B (dựng giáo trình ngược):** gom 88 câu thành các BÀI HỌC theo cụm khái niệm, sắp xếp **dependency order** (vì-sao-cache & tầng cache → strategies → invalidation/TTL → consistency → penetration/stampede/avalanche → eviction/memory → Redis nội tại → Redis ngoài cache → scaling → queue → HTTP/CDN), ánh xạ Bài → ID câu.
2. **Bước C:** giảng từng bài theo **Hợp đồng đầu ra 10 mục** (Mục 4).
3. **Bước D:** phỏng vấn theo đợt 10–15 câu, **hỏi xong DỪNG chờ trả lời**, chấm LIVE bám dòng *"dò cái gì"*; trộn dễ→khó.
4. **Bước E:** cổng hoàn thành = ĐẠT toàn bộ; chèn câu cũ ôn ngắt quãng (hôm nay → mai → 3 ngày → 1 tuần).

> ⚠️ Nhắc khi học: lý thuyết J ổn định, nhưng **mọi dòng có *(verify)*** (cú pháp/lệnh Redis, danh sách `maxmemory-policy`, cơ chế Cluster/Sentinel/module, API & version BullMQ, header HTTP) phải đối chiếu **docs chính thức** (redis.io, bullmq.io, MDN/RFC 9111) tại thời điểm học — đừng tin trí nhớ model.
