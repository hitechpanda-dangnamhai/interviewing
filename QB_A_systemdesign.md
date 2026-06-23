# 🧪 QB_A — Ngân hàng câu hỏi: System Design & Software Architecture

> **Mục A** trong roadmap Tech Lead Backend (🔴 ~90% — vòng nặng nhất) · Sinh theo **WORKFLOW 2 — Bước A**.
> Đây là **mục LỚN** → bộ đề phủ HẾT 14 mục con của A + nhóm phán đoán TL ("hiểu để chỉ huy & kiểm tra").
> **Chỉ câu hỏi — KHÔNG kèm đáp án.** Mỗi câu có dòng *"dò cái gì"* = tiêu chí ĐẠT, dùng để chấm **live** ở Bước D.
> **Chống trùng:** A là mục *tổng hợp* (ráp Phase 1–4). Đã đọc `QB_E/F/I/J/K/N/O`. Ở đây mọi câu giữ ở **góc quyết định thiết kế** (chọn gì, đặt ở đâu, đánh đổi gì), KHÔNG lặp câu engine-level đã có ở các mục kia — chỗ nào đụng, ghi *(giao với X — đào sâu ở mục đó)* thay vì hỏi lại cơ chế.
> **Tổng: 89 câu.**

## Cách đọc
- **ID:** `A-<tag>-<số>`. Tag mục con liệt kê ở mục lục dưới.
- **Độ khó:** ⭐ định nghĩa cơ bản → ⭐⭐⭐⭐⭐ phân biệt senior với TL thật.
- **"Dò cái gì":** điều người trả lời PHẢI chạm tới mới tính ĐẠT (không phải đáp án — chỉ là mốc chấm).

## Bối cảnh (tính tại 6/2026)
- System design chủ yếu là **khái niệm ổn định** (building block, trade-off, pattern) → ít cần verify.
- Nhưng chỗ nào chạm **tên/đời sản phẩm cụ thể, giá, con số latency tuyệt đối, hay "bigtech X hiện làm Y"** → đánh dấu *(verify)* và search trước khi khẳng định ở Bước C/D. Số latency trong bộ đề chỉ là **bậc độ lớn** để biện luận, không phải con số chính xác.
- Pattern bigtech (fan-out, ABR, adaptive rate limiting…) nêu như **pattern phổ biến**, không khẳng định chi tiết nội bộ của hãng.

---

## Mục lục mục con (ID prefix → số câu)

| Tag | Mục con | Số câu |
|---|---|---|
| **A-PROC** | Quy trình làm bài (clarify→estimate→design→scale→trade-off) | 7 |
| **A-CAP** | Capacity / back-of-envelope estimation | 6 |
| **A-LB** | Load balancing & horizontal scaling | 7 |
| **A-BB** | Building blocks & kỹ thuật lõi (CDN, consistent hashing, bloom filter, geo…) | 9 |
| **A-URL** | Design URL shortener / pastebin | 5 |
| **A-RL** | Design rate limiter | 6 |
| **A-FEED** | Design news feed / fan-out | 6 |
| **A-VID** | Design video/feed platform quy mô lớn (TikTok/Instagram/Netflix) | 5 |
| **A-CHK** | Design e-commerce checkout (chống oversell) | 5 |
| **A-CHAT** | Design chat real-time | 5 |
| **A-NOTI** | Design notification system | 4 |
| **A-BOT** | Nhận diện bottleneck | 6 |
| **A-TO** | Trade-off articulation & CAP | 7 |
| **A-DEC** | Decomposition / service boundaries | 4 |
| **A-SVL** | Serverless & edge computing | 5 |
| **A-RT** | Phán đoán TL — hiểu để chỉ huy & kiểm tra | 2 |

---

## A-PROC — Quy trình làm bài system design

**A-PROC-001** ⭐⭐⭐
"Tiếp cận một bài system design từ con số 0 thế nào — kể các bước theo đúng thứ tự."
> *Dò cái gì:* clarify requirements (functional + non-functional) → ước lượng → high-level design → API/data model → scale & deep-dive → bottleneck/trade-off; KHÔNG nhảy vào vẽ box ngay.

**A-PROC-002** ⭐⭐⭐
"Functional vs non-functional requirements khác gì, vì sao phải chốt non-functional TRƯỚC khi vẽ kiến trúc?"
> *Dò cái gì:* functional = hệ làm gì; non-functional = scale/latency/availability/consistency/durability; non-functional định hình kiến trúc (vd cần strong consistency thì thiết kế khác hẳn).

**A-PROC-003** ⭐⭐⭐⭐
"Vì sao 'clarify trước khi thiết kế' là tín hiệu senior, và những câu clarify quan trọng nhất là gì?"
> *Dò cái gì:* thu hẹp scope tránh thiết kế sai bài; hỏi DAU/QPS, read:write ratio, latency budget, mức consistency cần, đặc thù dữ liệu; thể hiện không vội.

**A-PROC-004** ⭐⭐⭐
"Phỏng vấn ~45 phút, bạn phân bổ thời gian các bước ra sao và vì sao?"
> *Dò cái gì:* không sa lầy 1 phần; đủ thời gian cho high-level + 1–2 deep-dive + trade-off cuối; quản lý thời gian là kỹ năng được chấm.

**A-PROC-005** ⭐⭐⭐⭐
"'Drive the interview' nghĩa là gì — vì sao ngồi chờ interviewer dẫn lại là điểm trừ?"
> *Dò cái gì:* chủ động đề xuất scope, nêu giả định ra tiếng, tự chỉ bottleneck & chọn phần deep-dive; thể hiện ownership thay vì bị hỏi mới nói.

**A-PROC-006** ⭐⭐⭐
"API contract & data model nên xuất hiện ở bước nào, vì sao không bỏ qua?"
> *Dò cái gì:* sau high-level, trước scale; định nghĩa vài endpoint chính + schema cốt lõi để neo thiết kế, tránh nói chung chung treo lơ lửng.

**A-PROC-007** ⭐⭐⭐⭐⭐
"Sai lầm về quy trình hay khiến trượt vòng system design dù biết nhiều kiến thức?"
> *Dò cái gì:* nhảy vào chi tiết quá sớm, không hỏi requirement, không nêu trade-off, im lặng không nghĩ thành tiếng, over-engineer; quy trình kém che lấp kiến thức tốt.

---

## A-CAP — Capacity / back-of-envelope estimation

**A-CAP-001** ⭐⭐⭐
"10M DAU — ước lượng QPS trung bình và peak thế nào?"
> *Dò cái gì:* DAU × số request/ngày ÷ 86400 ≈ avg QPS; peak ≈ 2–3× avg; nêu giả định rõ ràng thay vì bịa số.

**A-CAP-002** ⭐⭐⭐
"Ước lượng dung lượng lưu trữ cho hệ sinh ~500M item/năm (vd ảnh) — tính ra sao?"
> *Dò cái gì:* item/năm × kích thước trung bình × hệ số (replication + metadata); quy ra GB/TB/PB; dự trù tăng trưởng nhiều năm.

**A-CAP-003** ⭐⭐⭐
"Read:write ratio ảnh hưởng kiến trúc thế nào — vì sao phải ước lượng nó sớm?"
> *Dò cái gì:* read-heavy → cache + read replica + CDN; write-heavy → sharding/queue/LSM store; ratio quyết định ĐẶT nỗ lực tối ưu ở đâu.

**A-CAP-004** ⭐⭐⭐⭐
"Vài con số latency 'nên thuộc bậc độ lớn' nào hữu ích để biện luận nhanh khi thiết kế?"
> *Dò cái gì:* memory (ns) ≪ SSD (µs) ≪ disk/network (ms) ≪ cross-region (~chục–trăm ms); dùng để biện luận đặt cache/replica ở đâu. *(verify — đây là bậc độ lớn, không phải số tuyệt đối)*

**A-CAP-005** ⭐⭐⭐
"Bandwidth/throughput estimation (QPS × payload) cho ra gì và dùng làm gì?"
> *Dò cái gì:* ra bytes/s → biết cần CDN không, egress bao nhiêu, chọn instance/network, ước lượng chi phí; nối ước lượng với quyết định.

**A-CAP-006** ⭐⭐⭐⭐
"Khi nào estimation là lãng phí thời gian trong phỏng vấn, khi nào bắt buộc?"
> *Dò cái gì:* chỉ ước lượng con số *dẫn tới quyết định kiến trúc* (cần shard? cần cache? 1 hay nhiều DC?); bỏ ước lượng trang trí không đổi thiết kế.

---

## A-LB — Load balancing & horizontal scaling

**A-LB-001** ⭐⭐
"Vì sao scale ngang (horizontal) thường ưu tiên hơn scale dọc (vertical)?"
> *Dò cái gì:* vertical có trần phần cứng + SPOF + downtime khi nâng; horizontal mở rộng gần vô hạn + chịu lỗi, đổi lại cần stateless + LB + phối hợp.

**A-LB-002** ⭐⭐⭐
"Stateless service nghĩa là gì, vì sao là điều kiện để scale ngang?"
> *Dò cái gì:* instance không giữ state phiên; bất kỳ instance phục vụ được request bất kỳ; state đẩy ra Redis/DB; nhờ đó thêm/bớt node tự do.

**A-LB-003** ⭐⭐⭐
"Sticky session là gì, vì sao thường nên TRÁNH?"
> *Dò cái gì:* ghim user vào 1 instance cố định; cản rebalance, mất session khi node chết, lệch tải; thay bằng external session store (Redis).

**A-LB-004** ⭐⭐⭐
"L4 vs L7 load balancer khác gì, chọn cái nào khi nào?"
> *Dò cái gì:* L4 route theo IP/port (nhanh, không hiểu nội dung); L7 hiểu HTTP (route theo path/header, TLS termination, retry) nhưng tốn hơn.

**A-LB-005** ⭐⭐⭐
"Các thuật toán LB (round-robin, least-connections, hash) — chọn theo gì?"
> *Dò cái gì:* round-robin đơn giản (request đồng đều); least-conn cho request không đều thời lượng; hash theo key giữ affinity/cache locality.

**A-LB-006** ⭐⭐⭐⭐
"Health check & graceful drain trong LB — vì sao quan trọng khi deploy/scale?"
> *Dò cái gì:* LB ngừng route tới node unhealthy; drain connection đang phục vụ trước khi tắt; tránh 5xx khi rolling deploy/scale-in.

**A-LB-007** ⭐⭐⭐⭐
"Bản thân LB là SPOF — làm sao cho tầng LB highly available?"
> *Dò cái gì:* nhiều LB + DNS round-robin/anycast/floating IP/active-passive failover; tránh một LB chết kéo sập toàn hệ.

---

## A-BB — Building blocks & kỹ thuật lõi

**A-BB-001** ⭐⭐⭐
"CDN giải quyết vấn đề gì, đặt cái gì lên CDN và KHÔNG đặt cái gì?"
> *Dò cái gì:* cache static/asset gần user → giảm latency + tải origin; KHÔNG cache dữ liệu cá nhân/động/nhạy cảm; cân nhắc TTL + invalidation.

**A-BB-002** ⭐⭐⭐
"Object storage (vd S3) khác database thế nào, khi nào dùng cho blob (ảnh/video)?"
> *Dò cái gì:* lưu file lớn rẻ/bền/scale; DB chỉ giữ metadata + URL; KHÔNG nhồi blob vào DB; phục vụ qua CDN/presigned URL.

**A-BB-003** ⭐⭐⭐⭐
"Consistent hashing là gì, giải quyết vấn đề gì mà modulo hashing (`hash % N`) không?"
> *Dò cái gì:* phân phối key qua node trên vòng hash; thêm/bớt node chỉ remap *phần nhỏ* key, không reshuffle gần như toàn bộ như modulo; nền tảng cho cache/shard cluster.

**A-BB-004** ⭐⭐⭐⭐
"Virtual nodes trong consistent hashing để làm gì?"
> *Dò cái gì:* rải mỗi physical node thành nhiều điểm trên vòng → phân phối đều hơn, tránh hot node, rebalance mượt khi node chết/thêm.

**A-BB-005** ⭐⭐⭐⭐
"Bloom filter là gì, dùng ở đâu trong thiết kế hệ thống?"
> *Dò cái gì:* cấu trúc xác suất trả 'có thể có / chắc chắn không' — có false positive, KHÔNG false negative; lọc trước để né đụng DB/cache (chống cache penetration, check tồn tại nhanh).

**A-BB-006** ⭐⭐⭐⭐
"Thiết kế tính năng 'tìm xung quanh' (geo/nearby) — index không gian dùng gì?"
> *Dò cái gì:* geohash / quad-tree / S2 để biến tọa độ 2D thành key truy vấn được; query theo ô lưới lân cận; KHÔNG quét toàn bảng tính khoảng cách.

**A-BB-007** ⭐⭐⭐
"Khi nào thêm message queue vào thiết kế — nó tách rời (decouple) cái gì, đánh đổi gì?"
> *Dò cái gì:* decouple producer/consumer, hấp thụ burst (buffer), xử lý async + retry; đổi lại thêm latency + eventual consistency + phức tạp vận hành. *(giao với K — đào sâu ở đó)*

**A-BB-008** ⭐⭐⭐
"Cần full-text search: vì sao không dùng `LIKE '%x%'` trên RDBMS mà thêm Elasticsearch/OpenSearch?"
> *Dò cái gì:* inverted index + relevance + scale; `LIKE %x%` không dùng được index, chậm tuyến tính; đổi lại phải đồng bộ dữ liệu (CDC) + vận hành thêm 1 hệ. *(verify tên/đời)*

**A-BB-009** ⭐⭐⭐⭐⭐
"Trong một thiết kế nhiều tầng, có thể đặt cache ở những tầng nào — đánh đổi từng tầng?"
> *Dò cái gì:* client/CDN/edge → API gateway → app local → distributed (Redis) → DB cache; mỗi tầng giảm tải tầng dưới nhưng thêm bài toán invalidation/stale; chọn theo read pattern. *(giao với J)*

---

## A-URL — Design URL shortener / pastebin

**A-URL-001** ⭐⭐⭐
"Thiết kế URL shortener: sinh short key bằng cách nào, đánh đổi gì?"
> *Dò cái gì:* base62 của ID tăng dần (gọn, tuần tự, đoán được) vs hash + xử lý collision (ngẫu nhiên, cần check trùng); độ dài key ↔ không gian địa chỉ.

**A-URL-002** ⭐⭐⭐
"Hệ này read-heavy hay write-heavy, suy ra kiến trúc gì?"
> *Dò cái gì:* read ≫ write (tạo 1 lần, redirect nhiều lần) → cache redirect nóng + read replica + CDN; tối ưu đường đọc.

**A-URL-003** ⭐⭐⭐⭐
"Sinh ID duy nhất ở quy mô phân tán mà không đụng nhau — các cách & đánh đổi?"
> *Dò cái gì:* counter trung tâm / range allocation, Snowflake-like (timestamp+machine+seq), UUID/KGS; tránh auto-increment 1 DB thành bottleneck; cân nhắc tính tuần tự vs phân tán.

**A-URL-004** ⭐⭐⭐
"Redirect nên dùng HTTP 301 hay 302, ảnh hưởng gì tới analytics & cache?"
> *Dò cái gì:* 301 permanent → browser cache, mất lần đếm sau; 302 temporary → mỗi lần qua server, đếm được; chọn theo nhu cầu tracking.

**A-URL-005** ⭐⭐⭐⭐
"Thêm custom alias + expiration + analytics làm phức tạp thiết kế ở đâu?"
> *Dò cái gì:* custom alias cần check unique; expiration cần TTL/cleanup job; analytics đẩy event async qua queue để KHÔNG làm chậm redirect.

---

## A-RL — Design rate limiter

**A-RL-001** ⭐⭐⭐
"Token bucket vs fixed window vs sliding window — đặc tính & đánh đổi từng cái?"
> *Dò cái gì:* token bucket cho phép burst + smoothing average rate; fixed window đơn giản nhưng boundary burst; sliding window log chính xác nhưng tốn memory; sliding counter dung hòa.

**A-RL-002** ⭐⭐⭐⭐
"'Boundary burst' của fixed window là gì, vì sao nguy hiểm?"
> *Dò cái gì:* dồn request cuối cửa sổ này + đầu cửa sổ kế → tới ~2× limit trong khoảng ngắn; sliding window khắc phục bằng cách trượt liên tục.

**A-RL-003** ⭐⭐⭐⭐
"Rate limiter trên 50 app server: vì sao state cục bộ mỗi server là sai, fix sao?"
> *Dò cái gì:* mỗi server chỉ thấy phần traffic → tổng vượt limit; cần state trung tâm (Redis) HOẶC local counter + sync định kỳ; nhận ra đây là bài toán consistency.

**A-RL-004** ⭐⭐⭐⭐
"Race condition khi 2 server cùng đọc-ghi 1 counter — xử lý thế nào?"
> *Dò cái gì:* atomic op / Lua script / INCR / transaction trong Redis; KHÔNG read-modify-write rời rạc; thừa nhận race là điểm cộng, lờ đi là red flag. *(giao với I)*

**A-RL-005** ⭐⭐⭐⭐
"Redis (nguồn state) chết thì rate limiter nên làm gì — fail-open hay fail-closed?"
> *Dò cái gì:* fail-open (cho qua, ưu tiên availability — đa số API ngoài) vs fail-closed (chặn, ưu tiên bảo vệ — endpoint tài chính/security); nêu trade-off có chủ đích, không trả lời cứng.

**A-RL-006** ⭐⭐⭐⭐⭐
"Ở throughput cực lớn, ngay cả Redis cũng thành bottleneck / hot key — giảm tải sao?"
> *Dò cái gì:* local in-memory counter sync Redis mỗi ~100ms (chấp nhận sai số nhỏ), shard key qua slot, per-user dedicated key cho super-user; consistent hashing để phân tán. *(pattern phổ biến — verify chi tiết)*

---

## A-FEED — Design news feed / fan-out

**A-FEED-001** ⭐⭐⭐⭐
"Fan-out-on-write vs fan-out-on-read khác gì, đánh đổi?"
> *Dò cái gì:* write (đẩy vào feed mỗi follower lúc post — đọc nhanh, ghi nặng, tốn storage) vs read (gom lúc đọc — ghi nhẹ, đọc chậm); chọn theo read:write & phân bố follower.

**A-FEED-002** ⭐⭐⭐⭐⭐
"'Celebrity problem' (user triệu follower) phá fan-out-on-write thế nào, fix ra sao?"
> *Dò cái gì:* 1 post phải ghi triệu feed → bão ghi/độ trễ; giải pháp hybrid: fan-out-on-write cho user thường + pull (fan-out-on-read) cho celebrity, merge lúc đọc.

**A-FEED-003** ⭐⭐⭐
"Feed lưu ở đâu để đọc nhanh — cấu trúc & vì sao?"
> *Dò cái gì:* precomputed feed trong cache/Redis theo user; chỉ lưu ID post rồi hydrate nội dung sau; giới hạn độ dài feed (vd vài trăm mục gần nhất).

**A-FEED-004** ⭐⭐⭐⭐
"Ranking feed (không chỉ theo thời gian) thêm vào làm phức tạp gì?"
> *Dò cái gì:* cần feature/score → không thể chỉ sort timestamp; thường tách ranking service, tính async/precompute; trade-off realtime vs chất lượng xếp hạng.

**A-FEED-005** ⭐⭐⭐
"Feed nhất quán tới mức nào là đủ — vì sao eventual consistency chấp nhận được ở đây?"
> *Dò cái gì:* thấy post trễ vài giây không gây hại (khác tiền/đặt vé) → ưu tiên availability/latency hơn strong consistency.

**A-FEED-006** ⭐⭐⭐⭐
"Pagination feed cuộn vô tận: vì sao cursor chứ không offset?"
> *Dò cái gì:* offset chậm + lệch/nhảy khi có post mới chèn; cursor (timestamp/id) ổn định + hiệu quả ở list lớn realtime. *(giao với F)*

---

## A-VID — Design video/feed platform quy mô lớn

**A-VID-001** ⭐⭐⭐⭐
"Phục vụ video (Netflix/TikTok): vì sao không stream từ origin mà qua CDN, và adaptive bitrate là gì?"
> *Dò cái gì:* CDN đẩy nội dung gần user (giảm latency + tải/băng thông gốc); ABR (HLS/DASH) chia nhiều độ phân giải, client chọn theo mạng; pre-encode nhiều bitrate sẵn. *(verify chi tiết hãng)*

**A-VID-002** ⭐⭐⭐⭐⭐
"Pipeline upload → xử lý → sẵn sàng phát của video gồm các bước nào, vì sao async?"
> *Dò cái gì:* upload tới object storage → transcoding nhiều bitrate/format qua queue + worker → đẩy CDN → cập nhật metadata khi xong; nặng/lâu nên async, không chặn user.

**A-VID-003** ⭐⭐⭐⭐
"TikTok 'For You' khác Instagram follow-feed ở chỗ nào về kiến trúc?"
> *Dò cái gì:* recommendation-driven (không chỉ theo follow) → cần candidate generation + ranking + serving riêng; mix precompute/realtime; nặng phần ML serving. *(pattern phổ biến)*

**A-VID-004** ⭐⭐⭐
"View count / like ở quy mô tỷ lượt: vì sao không `UPDATE` trực tiếp mỗi lượt?"
> *Dò cái gì:* ghi nóng vào 1 row → lock/bottleneck; gom batch / approximate counter / đẩy qua stream rồi aggregate; chấp nhận đếm xấp xỉ + trễ.

**A-VID-005** ⭐⭐⭐⭐
"Thumbnail/preview ở quy mô lớn sinh & phục vụ thế nào?"
> *Dò cái gì:* sinh offline lúc upload (nhiều size), lưu object storage, phục vụ qua CDN; KHÔNG sinh on-the-fly mỗi request.

---

## A-CHK — Design e-commerce checkout (chống oversell)

**A-CHK-001** ⭐⭐⭐⭐
"Thiết kế checkout chống oversell (bán quá tồn kho): cơ chế giữ hàng?"
> *Dò cái gì:* reserve/lock tồn kho lúc checkout (atomic conditional decrement, optimistic/pessimistic lock) + TTL reservation để nhả khi bỏ giỏ; KHÔNG 'check rồi trừ' rời rạc.

**A-CHK-002** ⭐⭐⭐⭐⭐
"Vì sao 'đọc tồn kho rồi trừ' kiểu naive sai trong môi trường đồng thời, fix sao?"
> *Dò cái gì:* race giữa 2 đơn tranh món cuối → cả 2 cùng thành công; cần atomic conditional update (`UPDATE ... WHERE qty>0`) / `SELECT FOR UPDATE` / Redis atomic. *(giao với I)*

**A-CHK-003** ⭐⭐⭐⭐
"Idempotency cho thanh toán: vì sao bắt buộc, đảm bảo thế nào?"
> *Dò cái gì:* client retry / double-click không được tính tiền 2 lần; idempotency key + dedup store ở payment service. *(giao với F/I)*

**A-CHK-004** ⭐⭐⭐⭐
"Checkout chạm nhiều service (tồn kho, thanh toán, đơn, thông báo) — nhất quán xuyên service sao?"
> *Dò cái gì:* Saga (orchestration/choreography) + compensating action thay vì distributed transaction/2PC; Outbox để publish event chắc chắn. *(giao với K/I)*

**A-CHK-005** ⭐⭐⭐
"Flash sale (1 món hot, triệu người tranh): chịu tải đột biến thế nào?"
> *Dò cái gì:* queue / virtual waiting room, pre-load tồn kho vào Redis, atomic decrement, reject sớm khi hết; tách luồng nóng khỏi hệ chính.

---

## A-CHAT — Design chat real-time

**A-CHAT-001** ⭐⭐⭐⭐
"Vì sao chat dùng WebSocket thay vì polling, đánh đổi gì?"
> *Dò cái gì:* kết nối 2 chiều bền, push realtime, ít overhead hơn long-poll; nhưng connection stateful → khó scale/LB, cần connection registry.

**A-CHAT-002** ⭐⭐⭐⭐
"Hàng triệu WebSocket đồng thời: scale tầng connection sao?"
> *Dò cái gì:* nhiều gateway giữ connection, map user→server (registry/pub-sub qua Redis), route message tới đúng server đang giữ connection; connection là state phải quản lý.

**A-CHAT-003** ⭐⭐⭐
"Lưu & phân phối message: đảm bảo thứ tự + không mất tin nhắn thế nào?"
> *Dò cái gì:* persist trước khi ack, sequence/timestamp cho ordering trong room, offline → lưu rồi push khi online; at-least-once + dedup.

**A-CHAT-004** ⭐⭐⭐⭐
"Presence (online/typing) ở quy mô lớn tốn kém vì sao, làm nhẹ sao?"
> *Dò cái gì:* cập nhật liên tục + fan-out tới nhiều người → tải lớn; heartbeat + TTL trong Redis, throttle cập nhật, chấp nhận trễ/xấp xỉ.

**A-CHAT-005** ⭐⭐⭐
"Group chat fan-out khác chat 1-1 thế nào?"
> *Dò cái gì:* 1 message phải tới N thành viên (fan-out như feed); nhóm rất lớn → cân nhắc pull/giới hạn; vẫn cần ordering theo room.

---

## A-NOTI — Design notification system

**A-NOTI-001** ⭐⭐⭐⭐
"Thiết kế hệ notification đa kênh (email/push/SMS): kiến trúc tối thiểu?"
> *Dò cái gì:* ingestion → queue → per-channel worker/provider adapter → retry/DLQ; tách kênh, template service, user preference, observability gửi/nhận.

**A-NOTI-002** ⭐⭐⭐⭐
"Đảm bảo không gửi trùng (dedup) & retry khi provider lỗi?"
> *Dò cái gì:* idempotency key / dedup store, exponential backoff + DLQ, at-least-once nên cần dedup; theo dõi delivery status.

**A-NOTI-003** ⭐⭐⭐
"Throttle & user preference (opt-out, quiet hours) đặt ở đâu trong luồng?"
> *Dò cái gì:* áp TRƯỚC khi đẩy provider; tôn trọng preference + chống spam + giới hạn quota provider; tách policy khỏi delivery.

**A-NOTI-004** ⭐⭐⭐⭐
"Fan-out notification cho sự kiện chạm hàng triệu user (vd 'live started'): chịu tải sao?"
> *Dò cái gì:* đẩy event vào queue, worker scale ngang, batch theo kênh, throttle provider; KHÔNG gửi đồng bộ inline trong request.

---

## A-BOT — Nhận diện bottleneck

**A-BOT-001** ⭐⭐⭐⭐
"'Bottleneck của thiết kế anh vừa vẽ là gì?' — bạn soi từ đâu trước?"
> *Dò cái gì:* tầng có tải/đồng thời cao nhất hoặc state tập trung (DB ghi, single LB, cache, hot key/partition); chỉ rõ + cách giảm, không nói chung chung 'cần scale'.

**A-BOT-002** ⭐⭐⭐⭐
"Database thường là bottleneck đầu tiên — vì sao, và các nấc giảm tải theo thứ tự?"
> *Dò cái gì:* ghi/đọc tập trung; nấc: index → cache → read replica → sharding → CQRS/queue; đi rẻ→đắt, KHÔNG nhảy thẳng vào shard.

**A-BOT-003** ⭐⭐⭐⭐
"Hot partition / hot key là gì, phát hiện & xử lý sao?"
> *Dò cái gì:* 1 shard/key nhận tải lệch (celebrity, popular item, key xấu); fix: shard key tốt hơn, salting, dedicated cache, replicate read; nhận ra phân phối lệch.

**A-BOT-004** ⭐⭐⭐⭐⭐
"'Hệ chậm ở production' — debug bottleneck từ đâu (không đoán mò)?"
> *Dò cái gì:* đo trước (metrics/tracing/p99), tìm tầng latency cao, soi queue depth/CPU/connection pool/DB slow query; data-driven, không sửa theo cảm tính. *(giao với O)*

**A-BOT-005** ⭐⭐⭐⭐
"Connection pool / thread pool cạn kiệt gây nghẽn thế nào?"
> *Dò cái gì:* request xếp hàng chờ connection → latency tăng, timeout lan cả hệ; chỉnh pool size theo DB limit, tách pool, circuit breaker. *(giao với E/I)*

**A-BOT-006** ⭐⭐⭐⭐
"Thundering herd / cache stampede khi cache hết hạn đồng loạt — phòng sao?"
> *Dò cái gì:* nhiều request cùng miss dồn vào DB; fix: lock/single-flight, jittered TTL, stale-while-revalidate, pre-warm. *(giao với J)*

---

## A-TO — Trade-off articulation & CAP

**A-TO-001** ⭐⭐⭐⭐
"CAP theorem nói gì, vì sao cách diễn đạt 'chọn 2 trong 3' gây hiểu lầm?"
> *Dò cái gì:* khi CÓ partition (P luôn có thể xảy ra) phải chọn C hay A; lúc không partition vẫn có C/A; thực tế là PACELC (cả latency); không phải 'vứt hẳn 1 cái vĩnh viễn'.

**A-TO-002** ⭐⭐⭐⭐
"Strong vs eventual consistency: chọn theo gì, ví dụ mỗi loại?"
> *Dò cái gì:* strong cho tiền/tồn kho/đặt vé; eventual cho feed/like/đếm view; đánh đổi latency/availability vs đúng-ngay; phải nêu ví dụ cụ thể.

**A-TO-003** ⭐⭐⭐⭐⭐
"'Vì sao A mà không B, anh đánh đổi gì?' — khung trả lời trade-off thuyết phục?"
> *Dò cái gì:* nêu tiêu chí (latency/cost/consistency/complexity), so 2 phương án theo tiêu chí, gắn với requirement đã chốt, thừa nhận nhược điểm của phương án chọn; KHÔNG 'A tốt hơn' trống rỗng.

**A-TO-004** ⭐⭐⭐⭐
"Latency, throughput, cost, consistency — vì sao không tối ưu được tất cả cùng lúc?"
> *Dò cái gì:* cải thiện cái này thường hy sinh cái kia (strong consistency ↑latency; cache ↓latency nhưng stale); TL chọn theo ưu tiên nghiệp vụ, không 'tốt mọi mặt'.

**A-TO-005** ⭐⭐⭐⭐
"SQL vs NoSQL trong một thiết kế: quyết dựa trên gì (không phải 'NoSQL nhanh hơn')?"
> *Dò cái gì:* access pattern + yêu cầu consistency + mô hình dữ liệu + scale ghi; Postgres mặc định an toàn cho phần lớn ca. *(giao với E)*

**A-TO-006** ⭐⭐⭐⭐
"Monolith vs microservices cho hệ mới: đánh đổi, vì sao 'microservices mặc định' là sai?"
> *Dò cái gì:* micro thêm phức tạp network/vận hành/consistency xuyên service; monolith/modular monolith hợp team nhỏ & giai đoạn sớm; chia khi CÓ lý do (scale độc lập, team boundary).

**A-TO-007** ⭐⭐⭐⭐⭐
"Over-engineering trong system design là gì, vì sao là điểm trừ ở phỏng vấn TL?"
> *Dò cái gì:* thêm shard/microservice/queue/đa vùng khi chưa cần; chi phí phức tạp > lợi ích; bắt đầu đơn giản, scale khi requirement đòi (YAGNI); thể hiện phán đoán chứ không khoe kiến thức.

---

## A-DEC — Decomposition / service boundaries

**A-DEC-001** ⭐⭐⭐⭐
"Chia service theo gì — vì sao 'theo business capability / volatility' hơn là theo tầng kỹ thuật?"
> *Dò cái gì:* chia theo bounded context / cái gì thay đổi cùng nhau, KHÔNG theo layer (controller/service/db); giảm coupling khi thay đổi, mỗi service tự chủ một nghiệp vụ.

**A-DEC-002** ⭐⭐⭐⭐
"Dấu hiệu một boundary service bị vẽ SAI (chia nhầm)?"
> *Dò cái gì:* 2 service luôn deploy/đổi cùng nhau, gọi chéo liên tục (chatty), transaction xuyên service nhiều, chia sẻ DB; → nên gộp/vẽ lại boundary.

**A-DEC-003** ⭐⭐⭐⭐
"Database-per-service: lợi gì, kéo theo vấn đề gì?"
> *Dò cái gì:* tự chủ + deploy độc lập, nhưng mất join + cần đồng bộ dữ liệu (event/CDC) + consistency xuyên service phức tạp. *(giao với K)*

**A-DEC-004** ⭐⭐⭐
"Modular monolith là gì, vì sao thường là điểm khởi đầu tốt hơn microservices?"
> *Dò cái gì:* ranh giới module rõ trong 1 deploy; lấy lợi ích boundary mà chưa gánh chi phí phân tán; tách ra micro sau khi seam đã rõ.

---

## A-SVL — Serverless & edge computing

**A-SVL-001** ⭐⭐⭐⭐
"Khi nào serverless (FaaS) hợp, khi nào không — đánh đổi vs service truyền thống?"
> *Dò cái gì:* hợp tải gai/không đều, event-driven, ít vận hành; KHÔNG hợp tải đều cao (cost), latency nhạy, long-running, stateful; trade-off cold start + vendor lock-in.

**A-SVL-002** ⭐⭐⭐⭐
"Cold start là gì, ảnh hưởng thế nào, giảm sao?"
> *Dò cái gì:* khởi tạo runtime ở lần gọi đầu/sau idle → tăng latency; giảm bằng provisioned concurrency/keep-warm/runtime nhẹ; vấn đề với hệ p99 latency-sensitive.

**A-SVL-003** ⭐⭐⭐
"Serverless stateless & ephemeral: hệ quả tới state & DB connection?"
> *Dò cái gì:* không giữ state giữa lần gọi; connection pool khó (mỗi invocation 1 connection) → cần proxy (vd RDS Proxy)/external store; thiết kế khác service thường. *(verify tên dịch vụ)*

**A-SVL-004** ⭐⭐⭐⭐
"Edge computing / CDN compute giải quyết gì, đặt logic gì ở edge?"
> *Dò cái gì:* chạy logic gần user (auth nhẹ, A/B, rewrite, personalization cache) → giảm round-trip tới origin; KHÔNG đặt logic nặng/stateful/data-heavy ở edge.

**A-SVL-005** ⭐⭐⭐⭐⭐
"'Thiết kế này nên serverless hay container/k8s?' — bạn quyết theo tiêu chí nào?"
> *Dò cái gì:* pattern tải, latency budget, độ phức tạp vận hành, cost ở tải dự kiến, statefulness, vendor lock-in; quyết theo requirement, KHÔNG theo hype.

---

## A-RT — Phán đoán TL (hiểu để chỉ huy & kiểm tra)

**A-RT-001** ⭐⭐⭐⭐⭐
"AI/đồng nghiệp đưa kiến trúc 'chạy được' cho chatbot trả lời tài liệu chính sách *hay đổi* — và chọn fine-tune. Là TL bạn phản biện thế nào?"
> *Dò cái gì:* dữ liệu hay đổi → fine-tune sai (phải train lại, tốn, dễ lỗi thời/hallucinate); RAG/lookup mới đúng kiến trúc; *hiểu định hình yêu cầu*, không dừng ở 'chạy được'.

**A-RT-002** ⭐⭐⭐⭐⭐
"Một thiết kế 'đúng trên giấy' nhưng bạn vẫn bác — những thứ chỉ lộ ở production mà sơ đồ giấu là gì?"
> *Dò cái gì:* hot key/partition lệch, cold start, cache stampede, connection pool cạn, network partition, cost ở tải thật, thiếu observability/rollback; *hiểu để kiểm tra* cái sơ đồ không nói.

---

## ✅ Sau Bước A
Đây là **89 câu** phủ HẾT 14 mục con của A + nhóm phán đoán TL. Viết lại bằng lời mình từ tổng hợp nhiều nguồn phỏng vấn cập nhật (không chép danh sách nguồn); mỗi câu có mốc chấm riêng; giữ ở **góc quyết định thiết kế** và cross-reference *(giao với E/F/I/J/K/N/O)* thay vì lặp câu engine-level đã có ở các bank đó.

**Việc của bạn:** lưu file này vào project (kéo vào Project files). Khi bạn muốn, tôi sẽ:
- **Bước B** — dựng giáo trình *ngược* từ bộ đề (gom câu → bài học, sắp cơ bản → nâng cao theo dependency, ánh xạ Bài → ID). Lưu ý: trong dependency order của roadmap, **A học CUỐI** vì nó ráp Phase 1–4 — nên Bước B sẽ gợi ý nền E/F/I/J/K trước.
- **Bước C** — giảng từng bài theo Hợp đồng 10 mục (search + cite chỗ chạm tên/đời/giá/bigtech).
- **Bước D** — phỏng vấn theo đợt, chấm **live** bám đúng dòng *"dò cái gì"*.

> *Câu thần chú:* **"Tôi không nhớ để gõ, tôi hiểu để chỉ huy — và tra cứu phần còn lại."**
