# 📚 BỘ HỌC THUỘC — Caching Bài 1
> Keyword · Concept · Từ chuyên ngành + **Hoàn cảnh sử dụng**
> (✅ = có trong tài liệu · ➕ = liên quan, mở rộng thêm)

---

## PHẦN A — KEYWORDS (từ khoá)

### A1. Keyword CỐT LÕI (có trong tài liệu)

| Keyword | Nghĩa nhanh | 🎬 Hoàn cảnh sử dụng (câu mẫu để nhớ) |
|---|---|---|
| ✅ **Cache** | Bản sao dữ liệu đặt ở chỗ truy cập nhanh hơn nguồn | *"Mình **cache** kết quả query này lại, lần sau khỏi đập DB."* |
| ✅ **Stale** | Dữ liệu cache bị cũ, lệch so với nguồn | *"Giá hiển thị sai vì cache bị **stale**, chưa kịp invalidate."* |
| ✅ **Source of truth** | Nguồn gốc luôn đúng (thường là DB) | *"Khi nghi ngờ, đọc thẳng **source of truth** chứ đừng tin cache."* |
| ✅ **Hit / Miss** | Cache có / không có sẵn dữ liệu được hỏi | *"Lần đầu là **miss** nên phải đi DB; các lần sau **hit**."* |
| ✅ **Hit ratio** | Tỉ lệ request được cache trả lời | *"**Hit ratio** 90% nghĩa là 9/10 request không đụng DB."* |
| ✅ **TTL** (Time To Live) | Thời gian sống của một entry trước khi hết hạn | *"Đặt **TTL** 5 giây cho L1 để giới hạn thời gian bị stale."* |
| ✅ **Invalidate** | Xoá / vô hiệu hoá entry cache khi nguồn đổi | *"Mỗi lần ghi DB phải **invalidate** key tương ứng."* |
| ✅ **Back-fill** | Điền ngược kết quả lên các tầng cache sau khi miss | *"Miss cả L1 lẫn L2 → đọc DB → **back-fill** lên L2 rồi L1."* |
| ✅ **Hot key** | Key bị hỏi liên tục, rất nóng | *"Để **hot key** ở L1 local để giảm tải cho Redis."* |
| ✅ **Latency** | Độ trễ (thời gian phản hồi) | *"CDN cắt **latency** cho user ở xa server gốc."* |
| ✅ **Trade-off** | Sự đánh đổi (được cái này mất cái kia) | *"Cache là **trade-off**: nhanh hơn nhưng rủi ro stale + tốn RAM."* |
| ✅ **Multi-tier** | Cache nhiều tầng xếp chồng (L1 + L2…) | *"Hệ lớn nào cũng xài cache **multi-tier**, không chọn một."* |

### A2. Keyword LIÊN QUAN (➕ mở rộng — không có sẵn trong bài)

| Keyword | Nghĩa nhanh | 🎬 Hoàn cảnh sử dụng |
|---|---|---|
| ➕ **Eviction** | Cơ chế "đuổi" entry khi cache đầy | *"Cache đầy thì **eviction policy** quyết định bỏ ai (LRU bỏ thứ lâu không dùng)."* |
| ➕ **Cache warming** | "Hâm nóng" — nạp sẵn cache trước khi traffic tới | *"Trước flash sale, **warm cache** để tránh miss hàng loạt."* |
| ➕ **Cache stampede / thundering herd** | Nhiều request cùng miss một lúc, đồng loạt đập nguồn | *"Key hết hạn đúng lúc cao điểm → **stampede** làm sập DB."* |
| ➕ **Thrashing** | Cache liên tục evict rồi nạp lại, không hiệu quả | *"Cache quá nhỏ so với working set → **thrashing**, hit ratio tụt."* |
| ➕ **Cold start** | Cache rỗng lúc khởi động, miss nhiều | *"Sau deploy, cache **cold** nên p99 tăng vọt vài phút đầu."* |
| ➕ **Working set** | Tập dữ liệu thực sự được dùng trong một khoảng thời gian | *"Cache nên đủ chứa **working set**, không cần chứa toàn bộ data."* |
| ➕ **Negative caching** | Cache cả kết quả "không tồn tại" | *"**Negative cache** để khỏi đập DB lặp lại tìm thứ không có."* |

---

## PHẦN B — CONCEPTS (khái niệm tư duy)

### B1. Concept CỐT LÕI (có trong tài liệu)

| Concept | Ý cốt lõi | 🎬 Hoàn cảnh / câu thần chú |
|---|---|---|
| ✅ **Locality** | Cache "đặt cược" rằng thứ vừa/gần dùng sẽ được dùng lại | *"Không có **locality** → cache vô dụng."* |
| ✅ **Temporal locality** | Vừa dùng → dễ dùng lại ngay | *"User vừa mở trang A, vài giây sau refresh A → **temporal**."* |
| ✅ **Spatial locality** | Nằm gần thứ vừa dùng → dễ dùng tiếp | *"Đọc dòng 1 của mảng → sắp đọc dòng 2,3 → **spatial**."* |
| ✅ **Ba thứ đánh đổi** | Tốc độ ↔ Độ tươi (freshness) ↔ Bộ nhớ (RAM) | *"Mọi policy cache chỉ là cách cân **3 cái này**."* |
| ✅ **Các tầng cache** (cache layers) | Request xuyên qua nhiều tầng, mỗi tầng cơ hội trả lời sớm | *"Vẽ được sơ đồ tầng cache là hiểu được toàn cảnh."* |
| ✅ **Local vs Distributed** | Nhanh-ích-kỷ vs Chậm-hơn-đoàn-kết | *"**Local** không share; **distributed** share nhưng +network."* |
| ✅ **Strong consistency** | Đọc luôn ra giá trị mới nhất, tuyệt đối đúng | *"Path tiền / tồn kho cần **strong consistency** → đừng cache."* |
| ✅ **Cache = lazy / MV = eager** | Cache tính khi có người hỏi; Materialized View tính sẵn trước | *"Đọc thưa → **lazy cache**; cần luôn-sẵn → **eager MV**."* |
| ✅ **"Đo trước khi thêm tầng"** | Đừng tin cảm giác, phải đo lợi ích thực | *"p99 không cải thiện ư? **Đo** xem nút thắt có ở DB không đã."* |
| ✅ **Gắn cache vào "thứ cần bảo vệ"** | Tối ưu mục tiêu thật (DB QPS, p99), không tối ưu hit ratio mù quáng | *"Hit ratio 99% mà toàn thứ rẻ thì **vô nghĩa**."* |

### B2. Concept LIÊN QUAN (➕ mở rộng)

| Concept | Ý cốt lõi | 🎬 Hoàn cảnh sử dụng |
|---|---|---|
| ➕ **Eventual consistency** | Cuối cùng sẽ đồng nhất, nhưng có độ trễ | *"Like count xài **eventual consistency** được, lệch vài giây không sao."* |
| ➕ **CAP theorem** | Hệ phân tán chỉ chọn 2/3: Consistency, Availability, Partition-tolerance | *"Distributed cache buộc bạn đối mặt đánh đổi kiểu **CAP**."* |
| ➕ **Cache coherence** | Giữ nhiều bản sao đồng nhất với nhau | *"L1 ở mỗi node lệch nhau → vấn đề **coherence**, xử bằng pub/sub."* |
| ➕ **Idempotency** | Làm lại nhiều lần kết quả vẫn như một | *"Back-fill phải **idempotent** để retry an toàn."* |
| ➕ **Amdahl's law** | Tối ưu phần không phải nút thắt thì vô ích | *"Cache DB nhưng nút thắt ở serialize → vô dụng (kiểu **Amdahl**)."* |
| ➕ **Denormalization** | Nhân bản dữ liệu để đọc nhanh, đánh đổi ghi phức tạp | *"**Denormalize** cột thay vì cache khi đọc cực nhiều."* |

---

## PHẦN C — TỪ CHUYÊN NGÀNH (technical / industry terms)

### C1. Tầng cache & hạ tầng (có trong tài liệu)

| Thuật ngữ | Là gì | 🎬 Hoàn cảnh sử dụng |
|---|---|---|
| ✅ **Browser cache** | Bản sao ngay trên máy user (ảnh, CSS, JS) | *"Đặt header cache cho ảnh tĩnh → trình duyệt khỏi tải lại."* |
| ✅ **CDN / Edge cache** | Server rải khắp thế giới, phục vụ gần user | *"User VN được phục vụ từ node VN thay vì bay sang Mỹ."* |
| ✅ **Reverse proxy** | Đứng trước app, trả response đã cache (Nginx/Varnish) | *"**Nginx** trả thẳng trang đã cache, app khỏi chạy lại logic."* |
| ✅ **In-process / L1 cache** | Cache ngay trong tiến trình app (RAM của process) | *"Một `LRUCache` trong process — **không tốn network hop**."* |
| ✅ **Distributed cache / L2** | Service riêng mọi instance cùng nối (Redis/Memcached) | *"**Redis** làm L2 để mọi node thấy chung một sự thật."* |
| ✅ **DB buffer pool** | Bản thân DB cũng cache "page" nóng trong RAM | *"Đập thẳng DB không phải lúc nào cũng chậm — có **buffer pool**."* |
| ✅ **Network round-trip / hop** | Một vòng đi-về qua mạng | *"L2 tốn 1 **round-trip**; L1 thì không."* |
| ✅ **Page** | Đơn vị dữ liệu DB nạp vào bộ nhớ | *"DB cache các **page** nóng trong RAM."* |

### C2. Công cụ & sản phẩm thực tế (có trong tài liệu)

| Tên | Loại | 🎬 Hoàn cảnh sử dụng |
|---|---|---|
| ✅ **Redis** | Distributed cache / in-memory store | *"Dùng **Redis** làm L2, `SET key val EX 300`."* |
| ✅ **Memcached** | Distributed cache (đơn giản, thuần key-value) | *"**Memcached** nhẹ, Facebook xài ở quy mô khổng lồ."* |
| ✅ **Nginx / Varnish** | Reverse proxy có cache | *"**Varnish** chuyên cache HTTP response."* |
| ✅ **Cloudflare / Akamai / CloudFront** | Nhà cung cấp CDN | *"Bật **Cloudflare** để cache asset tĩnh ở edge."* |
| ✅ **LRUCache** | Cấu trúc cache loại bỏ "ít dùng gần đây nhất" | *"`new LRUCache({ max, ttl })` cho L1 in-process."* |
| ✅ **ioredis / lru-cache** | Thư viện Node.js (Redis client / LRU) | *"Pin `ioredis@5`, `lru-cache@11` trong package.json."* |
| ✅ **TAO** (Facebook) | Tier riêng cho social graph | *"FB thêm **TAO** cho đồ thị xã hội bên cạnh Memcached."* |
| ✅ **EVCache** (Netflix) | Lớp cache xây trên Memcached | *"Netflix dùng **EVCache** cho quy mô streaming."* |

### C3. Metric & vận hành (có trong tài liệu)

| Thuật ngữ | Là gì | 🎬 Hoàn cảnh sử dụng |
|---|---|---|
| ✅ **p50 / p99 latency** | Độ trễ trung vị / "đuôi" 1% chậm nhất | *"**p99** là thứ user cảm nhận đau nhất, không phải trung bình."* |
| ✅ **DB QPS** | Số query/giây xuống DB | *"Cache để kéo **DB QPS** xuống, giữ DB khỏi sập."* |
| ✅ **Cost / request** | Chi phí mỗi request | *"Cache LLM call vì **cost/request** rất cao."* |
| ✅ **High availability (HA)** | Service phải luôn sống, không được chết | *"Redis là điểm phụ thuộc → phải cấu hình **HA**."* |
| ✅ **Pub/sub broadcast** | Phát thông báo cho mọi node tự xoá L1 | *"Dữ liệu đổi → **pub/sub** báo các node invalidate L1."* |
| ✅ **`.env` / secret** | Cấu hình nhạy cảm, không commit vào code | *"`REDIS_URL` để trong **.env**, đừng hardcode."* |
| ✅ **Materialized View (MV)** | Bảng tính sẵn kết quả, refresh định kỳ | *"Báo cáo tổng hợp đọc nhiều → dùng **MV** thay cache."* |
| ✅ **Orders of magnitude** | Chênh nhau nhiều bậc độ lớn (×10, ×100…) | *"RAM nhanh hơn network nhiều **orders of magnitude**."* |

### C4. Từ chuyên ngành LIÊN QUAN (➕ mở rộng)

| Thuật ngữ | Là gì | 🎬 Hoàn cảnh sử dụng |
|---|---|---|
| ➕ **Cache-aside (lazy loading)** | App tự đọc cache trước, miss thì đọc DB rồi nạp lại | *"Pattern phổ biến nhất: **cache-aside** (chính là code trong bài)."* |
| ➕ **Write-through** | Ghi đồng thời vào cache và DB | *"**Write-through** giữ cache luôn tươi nhưng ghi chậm hơn."* |
| ➕ **Write-back / write-behind** | Ghi vào cache trước, đẩy xuống DB sau | *"**Write-back** nhanh nhưng rủi ro mất data nếu cache chết."* |
| ➕ **Read-through** | Cache tự lo việc đọc nguồn khi miss | *"**Read-through** giấu logic load đi, app chỉ hỏi cache."* |
| ➕ **Eviction policy: LRU / LFU / FIFO** | Quy tắc đuổi entry: ít-dùng-gần-đây / ít-dùng-nhất / vào-trước-ra-trước | *"Chọn **LFU** khi vài key luôn nóng dài hạn."* |
| ➕ **ETag / Last-Modified** | Header HTTP để validate cache trình duyệt | *"Trả **ETag**, lần sau client hỏi 'còn mới không?' → 304."* |
| ➕ **Cache-Control / max-age** | Header chỉ định cách cache HTTP | *"Đặt **Cache-Control: max-age=3600** cho asset tĩnh."* |
| ➕ **CAS (Compare-And-Swap)** | Cập nhật có điều kiện, chống ghi đè race | *"Dùng **CAS** để cập nhật cache an toàn khi đa luồng."* |
| ➕ **Serialization / deserialization** | Đổi object ↔ chuỗi để lưu/gửi (JSON…) | *"`JSON.stringify` khi set Redis là bước **serialize**."* |
| ➕ **Sharding / partitioning** | Chia cache ra nhiều node theo key | *"Redis Cluster **shard** key để mở rộng ngang."* |
| ➕ **Singleflight / request coalescing** | Gộp nhiều miss cùng key thành 1 lần đi nguồn | *"Dùng **singleflight** chống cache stampede."* |
| ➕ **Jitter (TTL ngẫu nhiên)** | Cộng nhiễu vào TTL để key không hết hạn cùng lúc | *"Thêm **jitter** vào TTL → tránh stampede đồng loạt."* |

---

## PHẦN D — KHUNG GHI NHỚ NHANH (đóng vào hoàn cảnh)

### D1. 🪜 Thứ tự tầng cache — PHẢI THUỘC
```
Browser → CDN/Edge → Reverse proxy → L1 (in-process) → L2 (Redis) → DB buffer pool
   gần user nhất ───────────────────────────────────────────► gần DB nhất
   nhanh nhất / dễ stale nh? ──────────────────────────────► chậm hơn / tươi hơn
```
🎬 *Khi được hỏi phỏng vấn "kể các tầng cache" → đọc đúng chuỗi 6 tầng này.*

### D2. ⛔ 4 trường hợp KHÔNG nên cache
1. **Data đổi liên tục / đọc một lần** → hit ratio thấp.
2. **Cần strong consistency** → tiền · tồn kho · hạn mức tín dụng.
3. **Write-heavy, ít đọc** → invalidate nhiều hơn lợi.
4. **Data rẻ để tính lại** → cache chỉ thêm phức tạp.

🎬 *Khi sếp/AI đòi "thêm Redis cho nhanh" → đối chiếu path với 4 mục này trước.*

### D3. 🧠 Mental model (câu thần chú gốc)
> **Cache = bản sao GẦN & NHANH, đặt cược vào locality, trả giá bằng RAM + rủi ro stale.**
> **Đừng cache** thứ **rẻ** / **đổi-liên-tục** / **cần-đúng-tuyệt-đối**.

### D4. 🎯 E-commerce — field nào cache thế nào
| Field | Quyết định | Vì sao |
|---|---|---|
| Tên / mô tả SP | Cache mạnh, TTL dài | Đổi vài lần/ngày |
| Tồn kho realtime | KHÔNG cache (hoặc TTL cực ngắn) | Cần strong consistency |
| Giá KM theo user | Cache được, **key gồm `user`** | Mỗi user một giá |

---

## PHẦN E — TỰ KIỂM TRA (che cột phải lại để học)

| Câu hỏi | Trả lời |
|---|---|
| Tốc độ cache đến từ đâu? | **Locality** + tránh **I/O/compute** đắt |
| Cache trả giá bằng gì? | RAM + rủi ro **stale** + thêm thứ phải vận hành |
| Local khác Distributed? | Local nhanh không share; Distributed share nhưng +network |
| Hit ratio 99% có chắc tốt? | **Không** — phải gắn vào DB QPS / p99 |
| Cache vs MV? | Cache = **lazy**; MV = **eager** |
| Thêm Redis mà p99 không giảm? | Hit ratio thấp / nút thắt không ở DB / miss vẫn đập DB → **phải đo** |
