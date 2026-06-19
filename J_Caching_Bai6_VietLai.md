# 🎯 BÀI 6 — Eviction & Memory (Trục xuất key & Quản lý bộ nhớ)
**Phủ:** J-EVICT-001 → J-EVICT-007

---

## ① Mục tiêu & vị trí trong mạch

Bài 5 cho thấy **key có thể mất do Redis chết**. Bài này tiết lộ một sự thật khiến nhiều dev *sốc*: **key còn biến mất do eviction (bị trục xuất) — NGAY CẢ KHI chưa hết TTL**. Và vì sao điều đó **nguy hiểm** nếu Redis **vừa làm cache vừa giữ session/job**.

Đây là **nền vận hành** trực tiếp cho **Bài 7 (Redis nội tại)** và **Bài 10 (queue)**.

> 💡 **Ý chính:** **TTL không phải là lời hứa "key sẽ sống tới hết hạn".** Khi bộ nhớ đầy, Redis có quyền **xoá key đi trước hạn** để lấy chỗ. Hiểu điều này = hiểu vì sao **không được nhét cache chung instance với dữ liệu "không được phép mất"**.

---

## ② Giảng cơ bản → nâng cao

### (a) Eviction vs Expiration *(J-EVICT-001)* — hai thứ ĐỘC LẬP

Đây là cặp khái niệm dễ lẫn nhất của cả bài:

| | **Expiration** (hết hạn) | **Eviction** (trục xuất) |
|---|---|---|
| Ai kích hoạt | **Thời gian** — TTL bạn đặt | **Bộ nhớ đầy** — chạm `maxmemory` |
| Xoá key nào | Key **đã hết hạn** | Key **bị chọn theo policy** (có thể **chưa hết hạn**!) |
| Bạn kiểm soát | Có (đặt TTL) | Gián tiếp (qua policy + dung lượng) |

> 🚨 **Chỗ dev hay sốc:** *"Tôi đặt TTL 1 giờ, sao key mất sau 10 phút?"* → vì nó **bị evict do memory đầy**, hoàn toàn **không liên quan tới TTL còn lại**. **Eviction và expiration chạy độc lập nhau.**

**Ẩn dụ tủ lạnh đầy:**
> **Expiration** = món hết hạn sử dụng → bạn vứt theo ngày. **Eviction** = tủ **chật cứng**, bạn cần nhét đồ mới vào → buộc phải **vứt bớt món cũ** *dù chúng còn hạn*. Tủ chật không quan tâm món còn dùng được hay không, nó cần *chỗ trống*.

---

### (b) maxmemory-policy *(J-EVICT-002)* — 8 chính sách trục xuất

Khi đầy bộ nhớ, Redis hành xử theo **`maxmemory-policy`**. **Redis 8.x có 8 policy** (đã verify 6/2026 — *vẫn nên verify lại tại redis.io*):

| Policy | Phạm vi xét | Tiêu chí xoá |
|---|---|---|
| **`noeviction`** | — | **Không xoá**, trả **lỗi OOM** khi ghi thêm |
| **`allkeys-lru`** | **mọi key** | **Least Recently Used** (lâu không dùng nhất) |
| **`allkeys-lfu`** | mọi key | **Least Frequently Used** (Redis 4+) |
| **`allkeys-random`** | mọi key | Ngẫu nhiên |
| **`volatile-lru`** | **chỉ key có TTL** | LRU |
| **`volatile-lfu`** | chỉ key có TTL | LFU |
| **`volatile-random`** | chỉ key có TTL | Ngẫu nhiên |
| **`volatile-ttl`** | chỉ key có TTL | **TTL ngắn nhất bị xoá trước** |

**Default thường là `volatile-lru`** (cả **ElastiCache** và **Memorystore** cũng mặc định `volatile-lru`).

> 🎯 **Cách chọn policy — 3 câu hỏi:**
> 1. **Có chịu được eviction không?** Không → **`noeviction`** (nhưng nhớ: rồi sẽ **OOM error**).
> 2. **Mọi key có TTL không?** Nếu **có key KHÔNG TTL** thì các policy **`volatile-*`** có thể **không giải phóng được gì** (vì chúng chỉ đụng key có TTL) → kẹt.
> 3. **Pattern là recency hay frequency?** → chọn **LRU** vs **LFU**.
>
> **Pure cache** → **`allkeys-lru`** / **`allkeys-lfu`**. *(verify)*

---

### (c) LRU vs LFU *(J-EVICT-003)*

| | **LRU** (Least Recently Used) | **LFU** (Least Frequently Used) |
|---|---|---|
| Đuổi cái | **"Lâu không dùng"** (recency) | **"Ít dùng"** (frequency) |
| Câu hỏi | *Lần cuối đụng là khi nào?* | *Đụng bao nhiêu lần?* |

**Vì sao LFU đôi khi thắng — vấn đề cache pollution:**
> Tưởng tượng một **crawler / scan** quét **một lượt qua TẤT CẢ key** (mỗi key đọc đúng 1 lần). Với **LRU**, những key vừa-bị-quét đó được coi là **"vừa dùng"** → LRU **đẩy hot key THẬT ra ngoài** để giữ lại đám key rác vừa quét. Đây gọi là **cache pollution** (ô nhiễm cache) — đồ quý bị đồ rác chen mất chỗ.
>
> **LFU miễn nhiễm hơn:** vì nó đếm **tần suất**, key chỉ-được-đọc-1-lần có **frequency thấp** → bị đuổi trước, **hot key thật (đọc nhiều lần) được giữ lại**.

*(verify: Redis LFU là approx — dùng **Morris counter** + **decay**.)*

**Ẩn dụ:** LRU như dọn bàn theo *"cái nào lâu rồi không sờ tới"* — nhưng nếu có người vừa lướt tay qua mọi thứ một lượt, LRU tưởng tất cả đều "vừa dùng". LFU dọn theo *"cái nào ít khi dùng"* — lướt một lần không đánh lừa được nó.

---

### (d) Vì sao "approximated" (xấp xỉ) *(J-EVICT-004)*

> Redis **KHÔNG giữ danh sách LRU/LFU toàn cục** (làm vậy **quá tốn RAM + CPU**). Thay vào đó nó **sample (lấy mẫu) ngẫu nhiên vài key** rồi chọn "nạn nhân" tệ nhất trong mẫu — bỏ vào **eviction pool**.

- **`maxmemory-samples`** (mặc định **5**): **tăng** số mẫu → **chính xác hơn** (gần *true LRU*) nhưng **tốn CPU hơn**.
- Đây là một **đánh đổi accuracy ↔ overhead**.

> 🔍 **Vì sao không làm true LRU?** True LRU cần một **doubly-linked list cho MỌI key** để biết thứ tự dùng chính xác → tốn bộ nhớ khủng khiếp. **Redis từ chối** cái giá đó, chấp nhận xấp xỉ.

**Ẩn dụ sampling:** thay vì xếp hàng *toàn bộ* nhân viên để tìm người đến muộn nhất (tốn thời gian), quản lý **bốc ngẫu nhiên 5 người** rồi đuổi người tệ nhất trong 5 người đó. Không hoàn hảo, nhưng **đủ tốt và rẻ**.

---

### (e) Trộn cache + store bền cùng instance *(J-EVICT-005)* — ⚠️⚠️ NGUY HIỂM

> Nếu một Redis **vừa làm cache, vừa giữ session/job**, mà đặt policy **`allkeys-*`** → khi đầy bộ nhớ, Redis có thể **xoá nhầm key "KHÔNG ĐƯỢC PHÉP MẤT"** (session đăng nhập, job đang chờ xử lý).

**Vì sao chí mạng:** `allkeys-*` đối xử **mọi key như nhau** — nó *không biết* key nào là cache vứt được, key nào là session sống còn. User đang đăng nhập bỗng **bị đá ra**, job trong hàng đợi **biến mất không dấu vết**.

> 🎯 **Nguyên tắc:** **Cache cần evict; store bền cần `noeviction` + persistence.** Hai nhu cầu trái ngược → **TÁCH instance** (hoặc tách logic). *(Cross-ref J-QUEUE-007, J-BEYOND-004.)*

---

### (f) Hit ratio tụt sau khi bật LRU *(J-EVICT-006)*

**Triệu chứng:** bật eviction → **hit ratio rơi mạnh**.

**Chẩn đoán:** **working set > memory khả dụng** → **cache thrash** (cache *giật* — vừa đuổi key thì lại cần dùng ngay key đó, đuổi nhầm thứ sắp dùng).

**Ẩn dụ thrash:** bàn làm việc quá nhỏ. Bạn cất tài liệu A đi để lấy chỗ cho B, vừa cất xong thì cần A lại → lôi A ra, cất B đi... **quay cuồng dọn đồ thay vì làm việc**.

**Giải:**
- **Tăng memory.**
- **Giảm thứ cache** (TTL hợp lý / đừng cache thứ rẻ).
- **Tách nóng/lạnh** (hot data riêng).
- **Shard thêm** (chia tải ra nhiều node).

> 🔍 **Dấu hiệu định lượng:** **`evicted_keys` cao** + **`keyspace_hits` thấp** = **cache đang under-sized** (quá nhỏ so với nhu cầu).

---

### (g) Memory overhead / fragmentation *(J-EVICT-007)*

> *"Data của tôi chỉ 1GB, sao Redis chiếm HƠN 1GB RAM?"* — vì RAM thực tế gồm:
> - **Overhead per-key:** mỗi key kèm **metadata, expire info, pointer**.
> - **Encoding nội bộ** của Redis.
> - **Fragmentation** của allocator (**jemalloc**) — bộ nhớ bị "vụn", có khoảng trống không dùng được.

> 🎯 **Quy tắc đệm dung lượng:** đặt **`maxmemory` ~70–80% RAM**, **đừng sát mép**. Theo dõi bằng **`INFO memory`**:
> - **`used_memory`** (data thật) vs **`used_memory_rss`** (RAM hệ điều hành cấp thật).
> - **`mem_fragmentation_ratio`** (rss / used — cao = phân mảnh nhiều). *(verify)*

---

## ③ ⚠️ Kiến thức cũ / bị thay thế

| Cái CŨ | Thay bằng (hiện nay) | Vì sao |
|---|---|---|
| *"TTL đặt rồi key chắc chắn sống tới hết hạn"* | Hiểu **eviction xoá cả key chưa hết hạn** | `maxmemory` đầy → **evict bất kể TTL** |
| **`noeviction` cho cache** (mặc định nghĩ vậy) | **`allkeys-lru` / `lfu`** cho pure cache | `noeviction` → **OOM error** khi đầy |
| Chung 1 Redis cache + session, `allkeys-*` | **Tách instance** / `volatile-*` + protect | Tránh **evict nhầm session/job** |
| Đặt **`maxmemory` = 100% RAM** | **~70–80% RAM** | Overhead + fragmentation + buffer |
| *"Redis LRU là true LRU"* | **Approximated** (sampling), tinh chỉnh `maxmemory-samples` | True LRU **quá tốn** |

---

## ④ Áp dụng thực tế + So sánh bigtech

**Use case:**
- **Redis cache catalog** → **`allkeys-lru`**, `maxmemory` **80% RAM**, monitor **`evicted_keys`**.
- **Redis session/queue** riêng → **`noeviction` + AOF**.
- Phát hiện **crawler gây cache pollution** (hit ratio tụt) → cân nhắc **`allkeys-lfu`**.

**Pattern ở bigtech:**
- **ElastiCache (Redis/Valkey)** & **Memorystore** mặc định **`volatile-lru`**.
- **AWS** khuyến nghị **LRU** cho cache cơ bản, **LFU** khi popular key bị evict oan.
- Luôn **để RAM dư** cho buffer **replication / AOF**. *(verify defaults tại docs nhà cung cấp.)*

---

## ⑤ Code thực hành + cấu hình

**Cấu hình Redis (`redis.conf` / `CONFIG SET`):**

```bash
# redis.conf — instance làm PURE CACHE
maxmemory 4gb
maxmemory-policy allkeys-lru     # pure cache: evict thoải mái
maxmemory-samples 10             # gần true LRU hơn (tốn CPU hơn chút) — verify

# instance làm SESSION/QUEUE (store bền) → file/instance RIÊNG:
# maxmemory-policy noeviction
# appendonly yes                 # AOF persistence
```

```bash
# Soi vận hành:
redis-cli CONFIG GET maxmemory-policy
redis-cli INFO memory | grep -E 'used_memory:|used_memory_rss:|mem_fragmentation_ratio:|maxmemory:'
redis-cli INFO stats  | grep -E 'evicted_keys|keyspace_hits|keyspace_misses'
redis-cli --bigkeys   # tìm big key (verify)
```

> ⚠️ **Đừng** để cache instance dùng `noeviction` (sẽ **OOM-error** khi đầy).
> ⚠️ **Đừng** để store-bền dùng `allkeys-*` (sẽ **mất session/job**).

---

## ⑥ Keywords cần nhớ (glossary)

🧠 **Cần HIỂU:**
- **eviction ≠ expiration** (hai thứ độc lập).
- **LRU vs LFU** & **cache pollution**.
- **Vì sao approximated** (sampling thay vì list toàn cục).
- **thrash** khi **working set > RAM**.
- **overhead / fragmentation**.
- **Vì sao phải tách instance** cache vs store bền.

📌 **Cần THUỘC:**
- **8 maxmemory-policy** + default **`volatile-lru`**.
- **`allkeys-lru`** cho pure cache, **`noeviction`** cho store bền.
- **`maxmemory` ~70–80% RAM**; **`maxmemory-samples`** default **5**.

🛠️ **Cần LÀM ĐƯỢC:** chọn **policy đúng** theo use case; đọc **`INFO memory` / `INFO stats`** để chẩn đoán **thrash / fragmentation**; **tách instance** cache vs bền.

---

## ⑦ Mental model

> **`maxmemory` đầy → Redis evict theo policy, xoá CẢ key chưa hết TTL.**
> **Pure cache = `allkeys-lru`.**
> **Store bền (session/job) = `noeviction` + persistence + instance RIÊNG.**
> **LRU/LFU là XẤP XỈ (sampling), không phải true.**

---

## ⑧ Câu hỏi phỏng vấn & thách đố

- **(dễ)** Eviction khác expiration thế nào? → **expiration** tự hết theo TTL; **eviction** xoá khi đầy memory, **bất kể TTL**.
- **(TB)** Kể các maxmemory-policy chính + ý nghĩa. → *(bảng ②b)*.
- **(TB)** LRU vs LFU, khi nào LFU thắng? → **recency vs frequency**; LFU thắng khi **burst đọc-một-lần** (**cache pollution**).
- **(khó)** Vì sao Redis LRU là approximated, `maxmemory-samples` ảnh hưởng gì? → **sampling** thay vì list toàn cục; tăng samples = **chính xác hơn, tốn CPU**.
- **(khó)** Data 1GB sao chiếm >1GB RAM? → **overhead per-key + encoding + fragmentation**.

> **🧩 Thách đố:** *"Bạn set TTL 1h cho mọi cache key, policy `noeviction`, `maxmemory` 2GB. Một ngày app báo lỗi 'OOM command not allowed'. Vì sao, sửa sao?"*
>
> **Trả lời:** Memory **đầy** (key **tích lại nhanh hơn** tốc độ TTL dọn), mà **`noeviction`** → Redis **từ chối ghi** thay vì evict → **app lỗi OOM**. **Sửa:** đổi sang **`allkeys-lru` / `volatile-lru`** (cho phép evict), và/hoặc **tăng RAM**, và/hoặc **giảm TTL / thứ cache**. **Bài học:** **cache instance KHÔNG nên `noeviction`**.

---

## ⑨ Bài tập + tiêu chí tự chấm

**BT1.** Cho một Redis vừa cache vừa giữ session, đang `allkeys-lru`. Chỉ ra rủi ro + đề xuất sửa.
> ✅ **Đạt khi:** nhận ra **session bị evict nhầm** → **tách instance** (cache `allkeys-lru`, session `noeviction` + TTL).

**BT2.** Hit ratio tụt sau khi bật eviction. Liệt kê 3 hướng chẩn đoán/sửa.
> ✅ **Đạt khi:** nêu **working-set > RAM (thrash)** + **tăng RAM / giảm cache / tách nóng-lạnh / shard**.

---

## ⑩ Đọc thêm

- **redis.io/docs** — *Key eviction* (maxmemory & policies).
- **AWS** — *Database Caching Strategies Using Redis*, phần **Evictions**.
- **redis.io** — *Memory optimization*, *INFO memory fields*.

---

> 🔎 **AI hay sai ở đây:** AI cấu hình **một Redis cho mọi việc** (cache + session + queue) cùng **`allkeys-lru` "cho tiện"**.
> **Luôn tự kiểm:** *có key nào KHÔNG-ĐƯỢC-MẤT nằm chung instance với cache có evict không?* Có → **TÁCH**.
