# 📚 J — CACHING · Giáo trình đầy đủ (Bước B + C)

> **Mục J** trong roadmap Tech Lead Backend · Sinh theo **WORKFLOW 2 — Bước B (dựng giáo trình ngược) + Bước C (giảng từng bài)**.
> Nguồn câu hỏi: `QB_J_caching.md` (88 câu, 11 mục con). Mỗi bài bám đúng các ID câu nó phủ.
> **Cách dùng tài liệu này:** đọc để **hiểu** (không học vẹt cú pháp) → sau đó mở `QB_J_caching.md` tự test theo **Bước D** (hỏi xong DỪNG, tự trả lời, rồi đối chiếu dòng *"dò cái gì"*).
> **Thời điểm biên soạn:** 6/2026. Mọi dòng đánh dấu *(verify)* là **cú pháp/lệnh/đời sản phẩm dễ đổi** → đối chiếu docs chính thức (redis.io, bullmq.io, MDN/RFC 9111) khi học.

---

## 🧭 Câu thần chú khi học mục này

> *"Tôi không nhớ để gõ lệnh Redis — tôi hiểu để **chỉ huy** (chọn đúng strategy/policy) và **kiểm tra** (bắt được khi AI/đồng đội chọn sai)."*

Caching là mục mà AI rất hay viết **đúng cú pháp nhưng sai kiến trúc**: dùng write-back cho tiền, cùng một key cho mọi user, quên TTL, update cache thay vì delete. Toàn bộ giáo trình này nhấn vào **chỗ AI sai** — đó là phần giá trị của một Tech Lead.

---

## 📋 BƯỚC B — GIÁO TRÌNH NGƯỢC TỪ BỘ ĐỀ

88 câu được gom thành **11 BÀI HỌC**, sắp theo **dependency order** (mỗi bài cần bài trước làm nền), KHÔNG theo tần suất hỏi:

| Bài | Tên bài | Phủ ID (tag) | Số câu | Vì sao đứng ở đây |
|---|---|---|---|---|
| **1** | Vì sao cache · các tầng cache · khi nào KHÔNG cache | `J-WHY-001→007` | 7 | Nền: hiểu *vì sao* trước khi học *cách* |
| **2** | Cache strategies (aside/read-through/write-through/write-back/refresh-ahead) | `J-STRAT-001→010` | 10 | Khung tư duy cho mọi bài sau |
| **3** | Invalidation · TTL · key design · negative caching | `J-INVAL-001→010` | 10 | "Hai việc khó nhất CS" — phải nắm trước consistency |
| **4** | Cache↔DB consistency (dual-write, delete vs update, race, CDC) | `J-CONSIST-001→009` | 9 | Cần strategy + invalidation làm nền |
| **5** | Bộ ba kinh điển: Penetration / Breakdown(Stampede) / Avalanche + hot/big key | `J-PROB-001→011` | 11 | Hệ quả khi cache gặp tải thật |
| **6** | Eviction & memory (LRU/LFU, maxmemory-policy, working set) | `J-EVICT-001→007` | 7 | Vì sao key biến mất ngoài ý muốn |
| **7** | Redis as cache — bản chất (single-thread, data structures, RDB/AOF, vs Memcached) | `J-REDIS-001→009` | 9 | Bây giờ mới mở "hộp đen" Redis |
| **8** | Redis ngoài cache (rate limit, leaderboard, session, pub/sub, HLL) | `J-BEYOND-001→006` | 6 | Mở rộng cùng một hạ tầng |
| **9** | Redis scaling (replica, cluster/hash slots, sentinel, RediSearch) | `J-CLUSTER-001→006` | 6 | Khi 1 node hết đủ |
| **10** | Background jobs: BullMQ / Redis Streams (vs Kafka) | `J-QUEUE-001→007` | 7 | Redis làm queue — cần hiểu eviction (Bài 6) trước |
| **11** | HTTP / CDN caching (Cache-Control, ETag, CDN, edge) | `J-HTTP-001→006` | 6 | Tầng cache xa nhất, gói lại bức tranh tầng (Bài 1) |

**Mạch xuyên suốt:** *vì sao → strategy → invalidation → consistency → các "bệnh" khi tải cao → tại sao key mất (eviction) → Redis nội tại → Redis đa năng → scale → queue → tầng HTTP/CDN.*

---

## 🗺️ OUTDATED MAP — Caching (đã verify 6/2026)

> Bảng "Cũ → Mới" ổn định cho mục J. Các dòng có *(verify)* vẫn nên đối chiếu khi học.

| Cái cũ / hiểu lầm phổ biến | Hiện nay nên dùng / biết | Vì sao |
|---|---|---|
| "Redis là BSD, free vô tư" | Redis 8.x **tri-license RSALv2 / SSPLv1 / AGPLv3**; **Valkey** là fork BSD (Linux Foundation, AWS/GCP hậu thuẫn) | License đổi 2024 (→SSPL) rồi 2025 (Redis 8 thêm AGPLv3). Cloud provider làm managed service phải để ý license — nhiều team chọn **Valkey** hoặc **DragonflyDB** *(verify)* |
| "Redis hoàn toàn single-thread" | Command execution vẫn 1 luồng, nhưng **I/O threads** (Redis 6+, mạnh hơn ở 8.x / Valkey 8.x) | Lý do nhanh vẫn đúng, nhưng "1 core" không còn tuyệt đối *(verify)* |
| `KEYS *` để quét key | **`SCAN`** (cursor, non-blocking) | `KEYS` O(n) block single-thread → giết latency mọi client |
| `DEL` một big key | **`UNLINK`** (giải phóng async) | `DEL` big key block; `UNLINK` trả ngay |
| Ép JSON "không tồn tại" → cứ đập DB | **Negative caching** (cache null TTL ngắn) + **Bloom filter** chống penetration | Bảo vệ DB khỏi id rác / tấn công |
| Naive RAG/cache top-k + TTL cố định | TTL **jitter** + single-flight/lock cho hot key + L1/L2 | Tránh avalanche & stampede |
| "Update cache khi write" | **Delete (invalidate) cache** khi write | Update-cache dễ race → stale bền |
| Bull (cũ) | **BullMQ 5.x** (TypeScript-native, dựa Redis Streams) *(verify 5.78.x — 6/2026)* | Bull đã được kế nhiệm |
| Chung 1 Redis cho cache + session/queue, cùng `allkeys-lru` | **Tách instance**: cache (`allkeys-lru`) vs store bền (`noeviction`/persistence) | Eviction xoá nhầm session/job |
| `Cache-Control` mơ hồ / lẫn `no-cache`↔`no-store` | Hiểu rõ `max-age`/`s-maxage`/`no-store`/`no-cache`/`private` theo **RFC 9111** | Đặt sai → rò cache giữa user hoặc miss tràn |


---

# 🎯 BÀI 1 — Vì sao cache · Các tầng cache · Khi nào KHÔNG cache

> Phủ: `J-WHY-001 → J-WHY-007`

## ① Mục tiêu & vị trí trong mạch
Bài mở màn của Phase Caching. Học xong bạn **hiểu cache giải bài toán gì**, **vẽ được các tầng cache** trong một request web, biết **đánh đổi local vs distributed**, và quan trọng nhất: biết **khi nào KHÔNG nên cache** — vì cache thêm sai (stale) và thêm phức tạp. Bài này là nền cho mọi bài sau: mỗi strategy/policy về sau đều là một cách quản lý đánh đổi *tốc độ ↔ độ tươi ↔ bộ nhớ* mà bài này đặt ra.

## ② Giảng cơ bản → nâng cao

**(a) Trực giác.** Cache = giữ một **bản sao** dữ liệu ở chỗ **truy cập nhanh hơn nguồn gốc**, để lần sau khỏi phải tính lại / đọc lại từ chỗ chậm. Ẩn dụ: bạn không chạy xuống thư viện mỗi lần cần một con số — bạn ghi nó lên tờ giấy dán cạnh màn hình. Tờ giấy = cache. Nhanh vì nó **gần** và **đã sẵn**.

Tốc độ đến từ hai nguồn:
- **Locality** — *temporal* (thứ vừa dùng dễ được dùng lại) và *spatial* (thứ gần thứ vừa dùng dễ được dùng tiếp). Cache đặt cược vào locality.
- **Tránh I/O / compute đắt** — RAM nhanh hơn disk/DB/network/LLM-call nhiều bậc độ lớn.

Đổi lại bạn trả: **thêm bộ nhớ** + **rủi ro stale** (bản sao lệch nguồn) + **thêm một thứ phải vận hành**.

**(b) Cơ chế — các tầng cache (J-WHY-002).** Một request đi từ trình duyệt tới DB xuyên qua nhiều tầng cache, mỗi tầng đổi *tốc độ ↔ phạm vi chia sẻ ↔ độ tươi*:

```
Browser cache         (riêng từng user, nhanh nhất, dễ stale nhất)
   ↓
CDN / Edge cache      (gần user theo địa lý — cắt latency vùng miền)
   ↓
Reverse proxy         (Nginx/Varnish — cache response trước app)
   ↓
App in-process (L1)   (in-memory ngay trong tiến trình — không network)
   ↓
Distributed cache (L2)(Redis/Memcached — share giữa mọi instance)
   ↓
DB buffer pool        (chính DB cũng cache page trong RAM)
```

**(c) Local vs Distributed (J-WHY-003).**

| | Local (in-process) | Distributed (Redis) |
|---|---|---|
| Tốc độ | Cực nhanh (không network hop) | Nhanh nhưng tốn 1 network round-trip |
| Chia sẻ | ❌ Mỗi instance một bản → dễ lệch | ✅ Mọi instance thấy chung |
| Invalidate đồng loạt | Khó (phải báo mọi node) | Dễ (1 nơi) |
| Điểm phụ thuộc | Không thêm phụ thuộc | Thêm một service phải HA |

Thực tế hay **multi-tier (J-WHY-004)**: **L1 local** cho hot key (cắt latency + giảm tải L2) + **L2 Redis** làm nguồn chia sẻ. Rủi ro: L1 ở mỗi node có thể stale **khác nhau** khi L2/DB đổi → giải bằng **TTL ngắn cho L1** + **pub/sub broadcast invalidate**, hoặc chấp nhận lệch nhỏ.

**(d) Khi nào KHÔNG cache (J-WHY-005) — ➕ phần TL hay bỏ qua.** Cache **hại nhiều hơn lợi** khi:
- Data **đổi liên tục** hoặc **đọc một lần** (hit ratio thấp → cache chỉ tốn RAM + sinh stale).
- Path cần **strong consistency tuyệt đối**: số dư tiền, tồn kho, hạn mức — đọc cache cũ = quyết định sai.
- **Write-heavy, ít đọc** (invalidate liên tục, hiếm khi hit).
- Data **rẻ để tính lại** (cache không cứu được gì).

> Nguyên tắc: **chỉ cache khi đo được lợi**. Cache "cho chắc" là cách phổ biến để đẻ bug stale.

**(e) Đo cache có đáng không (J-WHY-006).** Metrics: **hit ratio**, **latency p50/p99**, **mức giảm tải nguồn** (DB QPS giảm bao nhiêu), **cost/req**. Bẫy: *hit ratio cao chưa chắc tốt* — nếu bạn toàn cache thứ rẻ/ít gọi thì hit ratio đẹp mà chẳng bảo vệ được gì. Phải gắn cache vào **thứ bạn muốn bảo vệ** (DB khỏi sập, latency đuôi p99), không tối ưu hit ratio mù.

**(f) Cache vs precompute/materialized view (J-WHY-007).** Cache = **lazy**, theo nhu cầu, có thể **miss**. MV/precompute = **chủ động** tính sẵn, **luôn có** nhưng tốn ghi/refresh. Chọn theo *độ tươi cần* + *pattern đọc*. (Cross-ref mục E: MV vs cột denorm vs cache.)

## ③ ⚠️ Kiến thức cũ / bị thay thế

| Cái cũ (trong docs / phổ biến) | Thay bằng (hiện nay) | Vì sao |
|---|---|---|
| "Cache cho mọi thứ để nhanh" | Cache có chọn lọc, đo hit ratio + tải nguồn | Cache bừa → stale bug + tốn RAM, không bảo vệ đúng chỗ |
| "Tối ưu hit ratio là mục tiêu" | Tối ưu **thứ cần bảo vệ** (DB QPS, p99) | Hit ratio cao trên thứ rẻ là vô nghĩa |
| "Local cache là đủ cho cluster" | Multi-tier L1+L2, hiểu local không share | Mỗi node một bản → lệch giữa instance |

## ④ Áp dụng thực tế + So sánh bigtech
**Use case:** trang sản phẩm e-commerce. Thông tin sản phẩm (đổi vài lần/ngày) → cache mạnh. Tồn kho realtime → **không** cache (hoặc TTL cực ngắn + đọc DB khi checkout). Giá sau khuyến mãi cá nhân hoá → cache theo key gồm user.

**Bigtech (pattern phổ biến, không tuyệt đối — *verify* khi cần số liệu):** mọi hệ thống quy mô lớn đều multi-tier: browser → CDN (Cloudflare/Akamai/CloudFront) → app cache → Redis/Memcached. Facebook nổi tiếng với **Memcached** ở quy mô khổng lồ + tier riêng (TAO cho social graph). Netflix dùng **EVCache** (lớp trên Memcached). Điểm chung: họ **đo** và chấp nhận stale ở chỗ rẻ, **không** cache ở path tiền/quyền.

## ⑤ Code thực hành + cấu hình
Minh hoạ multi-tier (L1 in-process LRU + L2 Redis) trong Node.js:

```js
// cache.js — L1 (in-process, TTL ngắn) + L2 (Redis, share)
// Pin version: ioredis@5, lru-cache@11 (verify bản mới nhất khi học)
import Redis from "ioredis";
import { LRUCache } from "lru-cache";

const redis = new Redis(process.env.REDIS_URL); // KHÔNG hardcode URL/secret
const L1 = new LRUCache({ max: 10_000, ttl: 5_000 }); // L1 chỉ 5s → giảm rủi ro stale giữa node

export async function getProduct(id, loadFromDb) {
  const key = `product:${id}`;

  // 1) L1 (nhanh nhất, không network)
  const l1 = L1.get(key);
  if (l1 !== undefined) return l1;

  // 2) L2 (Redis, share giữa instance)
  const l2 = await redis.get(key);
  if (l2 !== null) {
    const val = JSON.parse(l2);
    L1.set(key, val);
    return val;
  }

  // 3) Miss cả hai → nguồn (DB), rồi điền ngược lên
  const val = await loadFromDb(id);
  if (val != null) {
    await redis.set(key, JSON.stringify(val), "EX", 300); // L2 TTL 5 phút (verify cú pháp SET..EX)
    L1.set(key, val);
  }
  return val;
}
```
Cấu hình: `REDIS_URL` qua `.env` (không commit); `requirements`/`package.json` pin `ioredis@5`. ⚠️ Đừng để L1 TTL dài — sẽ lệch giữa các node.

## ⑥ Keywords cần nhớ (glossary)
- 🧠 **Cần HIỂU:** locality (temporal/spatial), tầng cache, đánh đổi tốc độ↔tươi↔RAM, multi-tier L1/L2, khi nào không cache, hit ratio gắn vào "thứ cần bảo vệ".
- 📌 **Cần THUỘC:** thứ tự tầng cache (browser→CDN→proxy→L1→L2→DB buffer); 4 trường hợp không nên cache; cache=lazy còn MV=eager.
- 🛠️ **Cần LÀM ĐƯỢC:** vẽ sơ đồ tầng cache cho một request; quyết định 1 field nên/không nên cache; đo hit ratio + DB QPS để biện minh.

## ⑦ Mental model
> **Cache = bản sao gần & nhanh, đặt cược vào locality, trả bằng RAM + rủi ro stale. Đừng cache thứ rẻ/đổi-liên-tục/cần-đúng-tuyệt-đối.**

## ⑧ Câu hỏi phỏng vấn & thách đố
1. *(dễ)* Cache giải bài toán gì, tốc độ đến từ đâu? → locality + tránh I/O; đổi bằng RAM + stale.
2. *(TB)* Kể các tầng cache trong một web request. → browser→CDN→proxy→L1→L2→DB buffer.
3. *(TB)* Local vs distributed cache khác gì, khi nào dùng cái nào? → local nhanh không share; distributed share nhưng +network; thực tế multi-tier.
4. *(khó)* Hit ratio 99% có chắc cache tốt? → không, nếu cache thứ rẻ/ít gọi; phải gắn vào DB QPS/p99.
5. *(khó)* Cho 3 trường hợp KHÔNG nên cache. → tồn kho/tiền (cần strong consistency), data đổi liên tục, đọc-một-lần.

**Thách đố:** *"Team thêm Redis cache trước DB, p99 vẫn không cải thiện — vì sao có thể vẫn vậy?"* → có thể hit ratio thấp (data đổi liên tục/đọc-một-lần), hoặc nút thắt không nằm ở DB (nằm ở compute/serialize/network), hoặc cache miss vẫn đập DB ở p99. Phải đo trước khi thêm tầng.

## ⑨ Bài tập + tiêu chí tự chấm
- **BT1:** Vẽ sơ đồ tầng cache cho trang chi tiết sản phẩm; với mỗi field (tên SP, tồn kho, giá KM theo user) ghi cache ở tầng nào + TTL. *Đạt khi:* tồn kho không cache (hoặc TTL cực ngắn), giá-theo-user có user trong key, tên SP cache dài.
- **BT2:** Cho 5 loại data, phân loại "nên cache / không nên" + lý do. *Đạt khi:* phân loại đúng theo tiêu chí hit ratio + yêu cầu consistency, không cache bừa.

## ⑩ Đọc thêm
- redis.io/docs — *Introduction / Use cases*.
- AWS Whitepaper — *Database Caching Strategies Using Redis* (phần tầng cache).
- Bài "scaling Memcached at Facebook" (khái niệm multi-tier ở quy mô lớn).

> 🔎 **AI hay sai ở đây:** AI mặc định "thêm cache = nhanh hơn" và bê Redis vào mọi path, kể cả tồn kho/tiền. Hãy kiểm tra: *path này có chịu được stale không?* Nếu không → đừng cache, hoặc cache phần public tách khỏi phần cần đúng.

---

# 🎯 BÀI 2 — Cache Strategies

> Phủ: `J-STRAT-001 → J-STRAT-010`

## ① Mục tiêu & vị trí trong mạch
Sau khi biết *vì sao* cache (Bài 1), bài này cho bạn **khung 5 chiến lược** đọc/ghi cache — bộ từ vựng dùng suốt phần còn lại. Học xong bạn mô tả được luồng đọc/ghi của từng strategy, biết **vì sao cache-aside là mặc định công nghiệp**, và bắt được lỗi kinh điển **dùng write-back cho dữ liệu tiền**.

## ② Giảng cơ bản → nâng cao

**(a) Trực giác.** 5 strategy khác nhau ở **ai chịu trách nhiệm đọc/ghi giữa cache và DB** và **ghi đồng bộ hay async**.

**(b) Cơ chế từng strategy.**

**1. Cache-aside / lazy loading (J-STRAT-001) — phổ biến nhất.** *App tự quản, cache không biết DB.*
- **Đọc:** app check cache → **miss** → đọc DB → ghi vào cache → trả. **Hit** → trả luôn.
- **Ghi:** cập nhật DB → **invalidate (delete)** cache (chi tiết delete-vs-update ở Bài 4).
- Nhược: lần đầu **luôn miss** (cold) + có **window stale**.

```
Đọc:  app → cache? --miss--> DB → set cache → trả
Ghi:  app → DB (update) → DEL cache
```

**2. Read-through (J-STRAT-002).** Giống cache-aside nhưng **cache layer/lib tự load từ DB khi miss**; app **chỉ nói chuyện với cache**. Gọn code hơn nhưng cần provider/abstraction hỗ trợ (vd cache library có "loader").

**3. Write-through (J-STRAT-003).** Mỗi write ghi **đồng bộ cả cache lẫn DB** (qua cache layer). Cache **luôn tươi** → đọc nhất quán hơn. Trả giá: **write latency cao hơn** + cache chứa cả data **không bao giờ được đọc lại** (phí RAM).

**4. Write-back / write-behind (J-STRAT-004) — ⚠️ nguy hiểm.** Write vào **cache trước**, flush xuống DB **async/batch sau**. Write cực nhanh + gộp ghi. **Rủi ro chí mạng: mất data** nếu cache chết trước khi flush. Chỉ hợp khi **chấp nhận mất mát nhỏ** (vd đếm view, metrics) + có persistence/replication.

**5. Refresh-ahead (J-STRAT-005).** Chủ động **làm mới key nóng *trước* khi hết hạn** (dự đoán sắp được đọc) → tránh miss/latency spike lúc expiry. Hạn chế: refresh nhầm key **không còn được đọc** → phí tài nguyên.

**(c) So sánh & lựa chọn (J-STRAT-006, 007).**

| | Cache-aside | Write-through |
|---|---|---|
| Độ tươi cho read | Có window stale | Tươi hơn |
| Write latency | Bình thường (ghi DB) | Cao hơn (ghi cả 2) |
| Cache chết thì sao | **Degrade** (chỉ tăng miss, đọc thẳng DB) | App phụ thuộc cache hơn |
| Độ phức tạp | Đơn giản | Cần cache layer hỗ trợ |

**Vì sao cache-aside là mặc định công nghiệp (J-STRAT-007):** (1) **đơn giản**, (2) **resilient** — cache down thì degrade chứ không chết (đọc thẳng DB), (3) **không khoá app vào provider**. Nhược (miss đầu, stale window) **chấp nhận được** với TTL hợp lý.

**(d) Cold start & warming (J-STRAT-008).** Cache **rỗng sau deploy/restart** → loạt miss đập DB (cold cache). **Warming** = preload hot key trước khi nhận traffic. Hạn chế: không biết hết hot key + không cứu **mass expiry** về sau (cần thêm TTL jitter — Bài 5).

**(e) Cache cái gì: row / computed / fragment (J-STRAT-009).** Càng gần **kết quả cuối** (rendered fragment) càng **cắt nhiều compute** nhưng càng **dễ vỡ** khi 1 phần đổi (invalidate khó) + trùng lặp. Cache **thấp** (raw row) tái dùng cao nhưng **còn compute**. Chọn theo *cost-tính-lại vs tần suất đổi*.

## ③ ⚠️ Kiến thức cũ / bị thay thế

| Cái cũ | Thay bằng | Vì sao |
|---|---|---|
| Write-back "cho nhanh" ở path quan trọng | Write-through / ghi-DB-trước cho path correctness | Write-back mất data khi cache chết |
| "Write-through luôn nhất quán tuyệt đối" | Hiểu vẫn stale ở môi trường phân tán (nhiều cache node/replica) | Consistency còn phụ thuộc replication & path ghi khác (Bài 4) |
| Cache-aside "update cache khi write" | **Delete cache khi write** | Update dễ race → stale bền (Bài 4) |

## ④ Áp dụng thực tế + So sánh bigtech
**Use case:** feed/profile đọc nhiều ghi ít → **cache-aside** + TTL. Đếm lượt xem video (chấp nhận lệch nhỏ, ghi cực nhiều) → **write-back/batch** flush xuống DB mỗi N giây. Trang chủ hot → **refresh-ahead** để không ai gặp miss lúc key hết hạn.

**Bigtech (pattern phổ biến):** đa số dùng **cache-aside** làm mặc định vì tính resilient. Write-back chỉ xuất hiện ở metrics/counter nơi mất vài bản ghi không chết người.

## ⑤ Code thực hành + cấu hình
Cache-aside chuẩn (Node.js + ioredis), có TTL + delete-on-write:

```js
// productCache.js — cache-aside (lazy loading)
import Redis from "ioredis";
const redis = new Redis(process.env.REDIS_URL);
const TTL = 300; // giây

export async function getProduct(id, db) {
  const key = `product:${id}`;
  const cached = await redis.get(key);
  if (cached !== null) return JSON.parse(cached);     // HIT

  const row = await db.findProduct(id);               // MISS → đọc DB
  if (row) {
    // EX = TTL; "NX" optional. Verify cú pháp SET tại redis.io
    await redis.set(key, JSON.stringify(row), "EX", TTL);
  }
  return row;
}

export async function updateProduct(id, patch, db) {
  await db.updateProduct(id, patch);  // 1) ghi DB TRƯỚC
  await redis.del(`product:${id}`);   // 2) DELETE cache (KHÔNG update) — Bài 4 giải thích vì sao
}
```
⚠️ Tuyệt đối **không** dùng write-back ở đây cho dữ liệu cần đúng. ⚠️ Quên TTL = key sống mãi → memory phình (Bài 6).

## ⑥ Keywords cần nhớ
- 🧠 **Cần HIỂU:** 5 strategy & luồng đọc/ghi; vì sao cache-aside resilient; trade-off tươi↔latency; rủi ro write-back; cache row vs computed vs fragment.
- 📌 **Cần THUỘC:** cache-aside đọc = check→miss→DB→set; ghi = DB→DEL; write-through = ghi cả 2 đồng bộ; write-back = cache trước, flush sau.
- 🛠️ **Cần LÀM ĐƯỢC:** code cache-aside có TTL + delete-on-write; chọn strategy đúng cho một use case; nhận ra write-back sai chỗ.

## ⑦ Mental model
> **Cache-aside là default (app tự quản, cache chết thì degrade). Write-through = tươi nhưng chậm. Write-back = nhanh nhưng có thể MẤT data — cấm dùng cho tiền.**

## ⑧ Câu hỏi phỏng vấn & thách đố
1. *(dễ)* Mô tả luồng đọc/ghi cache-aside. → check→miss→DB→set; ghi=DB→DEL.
2. *(TB)* Read-through khác cache-aside ở đâu? → cache layer tự load, app chỉ gọi cache.
3. *(TB)* Write-through đánh đổi gì? → tươi hơn nhưng write chậm + cache cả data không đọc lại.
4. *(khó)* Write-back lợi/rủi ro chí mạng? → nhanh+gộp ghi / mất data khi cache chết.
5. *(khó)* Vì sao cache-aside là mặc định công nghiệp dù có nhược điểm? → đơn giản + resilient + không khoá provider.

**Thách đố (J-STRAT-010, ⭐⭐⭐⭐⭐):** *"AI sinh code dùng write-back cho dữ liệu thanh toán 'cho nhanh'. Vì sao là lỗi kiến trúc, sửa sao?"* → Write-back có thể **mất write khi cache chết** → mất giao dịch tiền (không chấp nhận). Path correctness phải **write-through / ghi-DB-trước**. "Nhanh" không bù rủi ro mất tiền. *Đây đúng là chỗ TL phải bắt — hiểu để kiểm tra.*

## ⑨ Bài tập + tiêu chí tự chấm
- **BT1:** Viết hàm `getUser` cache-aside (TTL 60s) + `updateUser` (delete-on-write). *Đạt khi:* miss đọc DB rồi set; update ghi DB trước rồi DEL (không update cache).
- **BT2:** Cho 4 use case (giỏ hàng, số dư ví, đếm view, profile), gán strategy + biện minh. *Đạt khi:* ví=không write-back/đọc DB; view=write-back ok; profile=cache-aside.

## ⑩ Đọc thêm
- AWS Whitepaper *Database Caching Strategies Using Redis* — chương Caching Patterns.
- redis.io/docs — *Client-side caching*, *Caching patterns*.
- microservices.io — *Caching patterns* (aside vs through).

> 🔎 **AI hay sai ở đây:** AI thích write-back/"ghi cache xong trả luôn" vì nó nhanh trong demo. Kiểm tra: *nếu Redis chết ngay sau đó, có mất dữ liệu không thể tái tạo không?* Có → cấm.

---

# 🎯 BÀI 3 — Invalidation · TTL · Key Design · Negative Caching

> Phủ: `J-INVAL-001 → J-INVAL-010`

## ① Mục tiêu & vị trí trong mạch
"Có hai việc khó nhất trong khoa học máy tính: đặt tên biến, **cache invalidation**, và lỗi off-by-one." Bài này mổ xẻ vì sao invalidation khó, các công cụ kiểm soát stale (TTL, explicit invalidate, versioned key, negative caching, stale-while-revalidate) và **thiết kế cache key đúng** — nền trực tiếp cho Bài 4 (consistency).

## ② Giảng cơ bản → nâng cao

**(a) Vì sao invalidation khó (J-INVAL-001).** Khó ở chỗ biết **KHI NÀO** + **CÁI GÌ** cần xoá khi nguồn đổi — đặc biệt với data **dẫn xuất/aggregate** phụ thuộc nhiều nguồn (một row đổi có thể làm sai một list, một count, một trang). Xoá **thiếu** → stale; xoá **thừa** → mất hiệu quả cache. Lại còn phải phối hợp giữa **nhiều writer/instance**.

**(b) Hai cơ chế gốc (J-INVAL-002, 003).**
- **TTL-based expiry:** đặt thời hạn, key tự hết. Đơn giản, tự dọn; stale tối đa = TTL.
- **Explicit invalidation:** xoá key khi write. Tươi hơn nhưng phải **bắt được MỌI đường ghi** + **đúng key**.
- **Thực tế kết hợp:** explicit (tươi) **+ TTL làm lưới an toàn** (chặn stale vĩnh viễn nếu lỡ một đường ghi).

TTL ngắn vs dài: **ngắn** → tươi hơn nhưng hit ratio thấp + tải DB cao; **dài** → hiệu quả cache cao nhưng stale lâu. Chọn theo *độ chấp nhận stale của data* + *chi phí miss*.

**(c) Thiết kế cache key (J-INVAL-004) — cực kỳ quan trọng.** Key phải **xác định duy nhất** kết quả và **gồm MỌI tham số ảnh hưởng** output: `user/tenant`, `locale`, `filter`, `version schema`. 

> **Key tồi kinh điển:** `products` (không kèm tenant/filter/locale) → trả nhầm data giữa tenant/người dùng → **rò dữ liệu / sai quyền**. 
> **Key tốt:** `tenant:42:products:lang=vi:filter=active:v3`. Dùng **namespace rõ** (prefix có dấu `:`).

**(d) Versioned key / cache busting (J-INVAL-005).** Nhúng **version** (hoặc đổi prefix) vào key. Đổi version = **vô hiệu toàn bộ bản cũ NGAY** mà không cần `DEL` từng key; bản cũ tự rơi theo TTL/eviction. Tránh được race "xoá-rồi-đọc-lại". Vd: bump `schema_v=3 → 4` khi đổi format.

**(e) Invalidate list/aggregate (J-INVAL-006).** Một phần tử đổi có thể ảnh hưởng **nhiều key dẫn xuất** (list, count, page). Cách: **tag/group key**, **versioned namespace** (bump version của cả nhóm), hoặc **TTL ngắn cho aggregate**. Tránh phải truy mọi key bằng tay.

**(f) Event-based invalidation (J-INVAL-007).** Thay vì xoá cache ngay trong code ghi, **phát event khi DB đổi** → consumer xoá cache. Lợi: **tách concern** + bắt được cả **ghi từ nguồn khác** (batch job, service khác, không qua app). Gắn với **Outbox/CDC** để tránh "ghi DB ok nhưng publish fail" (chi tiết Bài 4 + mục K).

**(g) Negative caching (J-INVAL-008) — ➕ chống penetration.** Cache **kết quả "không tồn tại"** (null/empty marker) để chặn request id-rác liên tục đập DB. Rủi ro: nếu sau đó data **được tạo**, null cache **che mất** → cần **TTL ngắn cho null** + **invalidate khi create**. (Cross-ref Bài 5: penetration.)

**(h) Đừng bỏ TTL hoàn toàn (J-INVAL-009).** "Không bao giờ TTL, chỉ invalidate thủ công" rất nguy hiểm: lỡ **một** đường ghi không invalidate → **stale vĩnh viễn** (không lưới an toàn). **Luôn có TTL backstop** kể cả khi đã explicit invalidate.

**(i) Stale-while-revalidate (J-INVAL-010).** Trả **bản stale ngay** cho client **trong khi background làm mới** → không ai phải chờ DB lúc expiry. Chấp nhận stale ngắn để **bỏ latency spike**. Phổ biến ở HTTP/CDN (Bài 11) & app cache. *(verify cú pháp header `stale-while-revalidate`.)*

## ③ ⚠️ Kiến thức cũ / bị thay thế

| Cái cũ | Thay bằng | Vì sao |
|---|---|---|
| `DEL` từng key khi đổi format | **Versioned key/namespace** (bump version) | Vô hiệu cả nhóm tức thì, tránh race xoá-đọc |
| "Chỉ invalidate thủ công, khỏi TTL" | Explicit **+ TTL backstop** | Lỡ 1 đường ghi → stale vĩnh viễn |
| Xoá cache ngay trong code ghi | **Event/CDC-based invalidation** (Bài 4) | Bắt cả ghi ngoài app, tách concern |
| Key thiếu chiều (user/tenant/locale) | Key gồm **mọi tham số ảnh hưởng** | Tránh rò dữ liệu giữa user |

## ④ Áp dụng thực tế + So sánh bigtech
**Use case:** danh mục sản phẩm theo tenant + ngôn ngữ → key `tenant:{id}:catalog:{lang}:v{n}`; đổi schema → bump `v`. Trang "kết quả tìm kiếm không có" bị spam id rác → **negative cache** null 30s + Bloom filter (Bài 5).

**Bigtech (pattern phổ biến):** versioned/content-hash key là chuẩn cho static asset (CDN cache bust bằng đổi tên file `app.a1b2c3.js`). Hệ thống lớn thường **CDC (Debezium) → invalidate cache** thay vì rải lệnh DEL khắp code.

## ⑤ Code thực hành + cấu hình
Versioned key + negative cache:

```js
// catalogCache.js
import Redis from "ioredis";
const redis = new Redis(process.env.REDIS_URL);

const SCHEMA_V = 3;                 // bump số này = vô hiệu toàn bộ bản cũ
const NEG_TTL = 30;                 // null cache ngắn
const TTL = 600;

const key = (tenant, lang) => `tenant:${tenant}:catalog:${lang}:v${SCHEMA_V}`;

export async function getCatalog(tenant, lang, db) {
  const k = key(tenant, lang);
  const cached = await redis.get(k);
  if (cached === "__NULL__") return null;          // negative hit
  if (cached !== null) return JSON.parse(cached);  // hit

  const data = await db.loadCatalog(tenant, lang);
  if (data == null) {
    await redis.set(k, "__NULL__", "EX", NEG_TTL);  // negative cache TTL ngắn
    return null;
  }
  await redis.set(k, JSON.stringify(data), "EX", TTL);
  return data;
}

// Khi tạo data mới phải xoá negative cache để tránh che mất:
export async function onCatalogCreated(tenant, lang) {
  await redis.del(key(tenant, lang));
}
```
⚠️ `SCHEMA_V` đặt ở config để bump không cần sửa code rải rác. ⚠️ Negative TTL phải ngắn + invalidate khi create.

## ⑥ Keywords cần nhớ
- 🧠 **Cần HIỂU:** vì sao invalidation khó (khi nào+cái gì, aggregate); TTL vs explicit trade-off; vì sao versioned key tốt; negative caching & rủi ro che data; stale-while-revalidate.
- 📌 **Cần THUỘC:** key phải gồm mọi tham số ảnh hưởng; explicit + TTL backstop; negative TTL ngắn + invalidate-on-create.
- 🛠️ **Cần LÀM ĐƯỢC:** thiết kế cache key đúng namespace; implement versioned busting + negative cache.

## ⑦ Mental model
> **Key gồm MỌI thứ ảnh hưởng kết quả. Explicit invalidate + TTL backstop. Đổi format → bump version, đừng DEL từng key. Cache "không tồn tại" để chặn id rác.**

## ⑧ Câu hỏi phỏng vấn & thách đố
1. *(dễ)* Vì sao cache invalidation nổi tiếng khó? → biết khi nào+cái gì xoá, nhất là aggregate phụ thuộc nhiều nguồn.
2. *(TB)* TTL vs explicit invalidation, khi nào dùng gì? → TTL đơn giản/stale≤TTL; explicit tươi nhưng phải bắt mọi đường ghi; kết hợp.
3. *(TB)* Một cache key tồi và hậu quả? → thiếu tenant/user → rò data giữa người dùng.
4. *(khó)* Versioned key tốt hơn DEL thủ công chỗ nào? → vô hiệu cả nhóm tức thì, tránh race.
5. *(khó)* Negative caching để làm gì, rủi ro? → chặn penetration; rủi ro che data mới → TTL ngắn + invalidate-on-create.

**Thách đố:** *"Bạn đặt TTL 1h cho cache + invalidate trong code mỗi lần update. Một batch job đêm sửa thẳng DB không qua app. Chuyện gì xảy ra, fix sao?"* → Batch không kích hoạt invalidate → cache stale tới 1h. Fix: **CDC/event-based invalidation** (bắt thay đổi từ DB log) hoặc TTL ngắn hơn cho data đó; đừng dựa vào "mọi ghi đều qua app".

## ⑨ Bài tập + tiêu chí tự chấm
- **BT1:** Thiết kế cache key cho API `GET /reports?tenant=&from=&to=&format=`. *Đạt khi:* key gồm tenant+from+to+format+version; có namespace rõ.
- **BT2:** Thêm negative caching cho `getUserByEmail`. *Đạt khi:* cache null TTL ngắn + xoá null khi user được tạo.

## ⑩ Đọc thêm
- redis.io/docs — *Key expiration*, *Client-side caching invalidation*.
- MDN / RFC 9111 — `stale-while-revalidate`.
- Debezium docs — CDC để invalidate cache (mục K).

> 🔎 **AI hay sai ở đây:** AI sinh cache key kiểu `cache:${queryName}` quên user/tenant/locale → rò data. **Luôn kiểm tra key có gồm đủ chiều phân biệt không** (lặp lại ở Bài 7, J-REDIS-009).

---

# 🎯 BÀI 4 — Cache ↔ DB Consistency

> Phủ: `J-CONSIST-001 → J-CONSIST-009`

## ① Mục tiêu & vị trí trong mạch
Bài "nặng" nhất phần lý thuyết. Dùng strategy (Bài 2) + invalidation (Bài 3) làm nền để trả lời: **vì sao cache + DB không thể nhất quán tuyệt đối miễn phí**, **delete vs update cache**, **thứ tự update-DB-rồi-delete-cache có race gì**, và **CDC/Outbox** giữ đồng bộ tin cậy. Đây là chỗ phân biệt senior với TL thật.

## ② Giảng cơ bản → nâng cao

**(a) Bản chất: dual-write (J-CONSIST-001).** Cache + DB là **hai store**. Cập nhật hai store **không-atomic** → **luôn có cửa sổ** một cái đã đổi, cái kia chưa. Mục tiêu thực tế: **thu nhỏ & kiểm soát window stale**, không phủ nhận nó. Gắn với CAP/eventual consistency. *Không có giải pháp "nhất quán tuyệt đối, miễn phí".*

**(b) Delete vs Update cache (J-CONSIST-002).** Khi write, nên **delete (invalidate)** cache, **không update**:
- **Update-cache** dễ **race**: hai writer ghi cache lệch thứ tự với DB → giá trị cũ đè giá trị mới → **stale bền**. Lại phí ghi giá trị **có thể không bao giờ đọc**.
- **Delete** → lần đọc sau nạp lại từ DB (giá trị mới nhất). **Đơn giản & ít sai hơn.**

**(c) Thứ tự thao tác & race (J-CONSIST-003) — ⭐⭐⭐⭐⭐.**

**"Delete cache TRƯỚC, update DB SAU"** — race xấu:
```
T1: DEL cache
T2 (reader): miss → đọc DB (giá trị CŨ vì T1 chưa commit) → set cache = CŨ
T1: commit DB (giá trị MỚI)
→ cache giữ giá trị CŨ "bền" cho tới TTL  ❌
```

**"Update DB TRƯỚC, delete cache SAU"** — an toàn hơn nhưng **vẫn có race hiếm**:
```
T2 (reader): đọc DB CŨ (trước khi T1 commit)
T1: commit DB MỚI → DEL cache
T2: set cache = CŨ (sau khi T1 đã DEL)
→ stale  ❌ (xác suất thấp hơn nhiều: reader phải đọc-DB-trước-commit và set-cache-sau-DEL)
```
**Kết luận:** *update-DB-trước-delete-cache* là lựa chọn tiêu chuẩn (Cache-Aside của AWS), **nhưng KHÔNG kín tuyệt đối**.

**(d) Delayed double delete (J-CONSIST-004).** Xoá cache **trước + sau** update, lần 2 **sau một delay ngắn** → dọn giá trị cũ mà reader có thể đã nạp vào trong cửa sổ race. Hạn chế: **chọn delay khó**, vẫn còn xác suất, thêm phức tạp → chỉ **giảm** chứ không triệt tiêu.

**(e) Race read-miss đồng thời write (J-CONSIST-005).** Reader miss đọc DB (cũ) → trước khi nó ghi cache, writer update DB + delete cache → reader **ghi giá trị cũ đè lên** → stale. Giảm bằng: **versioned/CAS khi set cache** (chỉ set nếu version ≥), **double-delete**, hoặc **TTL ngắn**. (Cross-ref I-RACE check-then-act.)

**(f) Write-through vẫn stale ở phân tán (J-CONSIST-006).** Write-through nhất quán cache↔DB **qua cùng layer**, nhưng **nhiều cache node** / **read replica DB** / **đường ghi không qua cache** vẫn gây lệch. Consistency còn phụ thuộc **replication** & **mọi path ghi**.

**(g) Khi nào chấp nhận stale (J-CONSIST-007).** Hiển thị phụ (đếm view, gợi ý) → **stale OK**. Quyết định **tiền/tồn kho/quyền** → **đọc nguồn (DB)** hoặc **bỏ cache** cho path đó. Chọn theo **hậu quả sai**, không cache đồng loạt.

**(h) Read-your-own-writes (J-CONSIST-008).** User vừa update nhưng đọc trúng cache cũ → "không thấy thay đổi của mình" (UX tệ). Fix: **invalidate ngay sau write của họ**, hoặc **bypass cache** cho read ngay sau write của chính user (vd đọc DB trong N giây sau khi user ghi).

**(i) CDC/Outbox (J-CONSIST-009) — ⭐⭐⭐⭐⭐ chuẩn production.** Bắt thay đổi từ **commit log của DB** (Debezium) → phát event → invalidate cache. Đảm bảo **MỌI commit** (kể cả ghi ngoài app) đều kích hoạt invalidate → tránh "commit DB ok nhưng quên/lỗi xoá cache". Outbox: ghi event vào cùng transaction DB → relay publish → không mất event. *(verify; cross-ref K Outbox/CDC.)*

## ③ ⚠️ Kiến thức cũ / bị thay thế

| Cái cũ | Thay bằng | Vì sao |
|---|---|---|
| Update cache khi write | **Delete cache** | Update race → stale bền |
| Delete-cache-trước-update-DB | **Update-DB-trước-delete-cache** (+TTL/double-delete) | Thứ tự kia có race nạp giá trị cũ |
| "Write-through = nhất quán tuyệt đối" | Vẫn stale ở multi-node/replica | Dual-write + replication |
| Xoá cache rải rác trong code | **CDC/Outbox-based invalidation** | Bắt mọi commit, không mất |

## ④ Áp dụng thực tế + So sánh bigtech
**Use case:** đổi giá sản phẩm → update DB → DEL cache → (option) double-delete sau 500ms. Hồ sơ user vừa sửa → bypass cache 5s cho chính user đó (read-your-writes). Hệ thống nhiều service ghi chung bảng → **CDC** invalidate thay vì tin mỗi service tự xoá.

**Bigtech (pattern phổ biến):** Facebook mô tả vấn đề cache↔DB ở quy mô lớn trong paper Memcached (dùng "leases" chống stale set & thundering herd). Hệ thống hiện đại nghiêng về **CDC/Debezium → invalidate** vì nó bắt mọi đường ghi.

## ⑤ Code thực hành + cấu hình
Update-DB-trước-delete-cache + double-delete + versioned set chống race:

```js
// consistency.js
import Redis from "ioredis";
const redis = new Redis(process.env.REDIS_URL);
const TTL = 300;

export async function updateProduct(id, patch, db) {
  await db.updateProduct(id, patch);          // 1) DB trước
  await redis.del(`product:${id}`);           // 2) DEL cache
  // 3) double-delete: dọn giá trị reader có thể vừa nạp trong race window
  setTimeout(() => redis.del(`product:${id}`).catch(() => {}), 500);
}

// Set cache có version để chống ghi-đè-cũ (J-CONSIST-005)
// Lua: chỉ set nếu version mới >= version đang lưu (atomic, tránh check-then-act race)
const SET_IF_NEWER = `
local cur = redis.call('HGET', KEYS[1], 'v')
if (cur == false) or (tonumber(ARGV[2]) >= tonumber(cur)) then
  redis.call('HSET', KEYS[1], 'v', ARGV[2], 'data', ARGV[1])
  redis.call('EXPIRE', KEYS[1], ARGV[3])
  return 1
end
return 0`;  // verify Lua/HSET tại redis.io

export async function setVersioned(id, data, version) {
  await redis.eval(SET_IF_NEWER, 1, `product:${id}`,
                   JSON.stringify(data), version, TTL);
}
```
⚠️ `setTimeout` chỉ là minh hoạ double-delete; production nên dùng job/queue để không mất khi process chết. ⚠️ Với tiền/tồn kho: **đừng cache** giá trị quyết định, đọc DB.

## ⑥ Keywords cần nhớ
- 🧠 **Cần HIỂU:** dual-write không atomic; vì sao delete > update; race của 2 thứ tự; write-through vẫn stale; read-your-writes; CDC/Outbox.
- 📌 **Cần THUỘC:** update-DB-trước-delete-cache; double-delete = xoá 2 lần (lần 2 delay); negative/versioned set chống race.
- 🛠️ **Cần LÀM ĐƯỢC:** code update-then-delete đúng thứ tự; nhận diện path cần bypass cache; mô tả luồng CDC→invalidate.

## ⑦ Mental model
> **Cache+DB = dual-write → luôn có window stale, chỉ thu nhỏ được. DELETE (không update), DB trước cache sau, TTL backstop. Tiền/quyền thì đọc DB. Đồng bộ tin cậy = CDC/Outbox.**

## ⑧ Câu hỏi phỏng vấn & thách đố
1. *(TB)* Vì sao cache+DB không nhất quán tuyệt đối miễn phí? → dual-write không atomic → window stale.
2. *(khó)* Delete vs update cache, vì sao delete an toàn hơn? → update race lệch thứ tự → stale bền.
3. *(rất khó)* "Update DB rồi delete cache" vs "delete rồi update" — race mỗi cái? → (trình bày 2 sơ đồ ở mục ②c).
4. *(khó)* Read-your-own-writes vỡ vì cache thế nào, giữ sao? → đọc trúng cache cũ; invalidate/bypass ngay sau write của user.
5. *(rất khó)* CDC/Outbox giữ cache đồng bộ thế nào, vì sao tin cậy hơn xoá trong code? → bắt mọi commit từ DB log → invalidate; không miss ghi ngoài app.

**Thách đố:** *"Delayed double delete đặt delay = 1ms. Vẫn stale. Vì sao?"* → Delay phải **dài hơn** thời gian reader đọc-DB-rồi-set-cache (thường vài chục–trăm ms tuỳ tải). 1ms quá ngắn → lần xoá thứ 2 chạy *trước khi* reader kịp set giá trị cũ → không dọn được. Nhưng đặt dài thì lại tăng window stale → đó là lý do double-delete chỉ **giảm** chứ không **triệt tiêu**; muốn chắc → versioned/CAS hoặc bỏ cache cho path đó.

## ⑨ Bài tập + tiêu chí tự chấm
- **BT1:** Sửa hàm update sai (delete-trước-update) thành đúng thứ tự + giải thích race cũ. *Đạt khi:* đổi thành update-DB→DEL-cache + nêu đúng race nạp-giá-trị-cũ.
- **BT2:** Cho path "trừ tồn kho khi đặt hàng": quyết định cache hay không + lý do. *Đạt khi:* KHÔNG cache giá trị tồn kho quyết định; đọc/giảm trên DB (hoặc atomic store) — không tin cache.

## ⑩ Đọc thêm
- AWS Whitepaper *Database Caching Strategies Using Redis* — Cache-Aside consistency.
- Facebook *Scaling Memcache at Facebook* (leases, thundering herd).
- Debezium / *Transactional Outbox pattern* (microservices.io).

> 🔎 **AI hay sai ở đây:** AI viết "update cache = giá trị mới" hoặc "delete cache rồi update DB" — cả hai sinh stale. Kiểm tra **thứ tự thao tác** và **delete-not-update** mỗi khi review code cache write.

---

# 🎯 BÀI 5 — Bộ ba kinh điển: Penetration / Breakdown (Stampede) / Avalanche + Hot/Big key

> Phủ: `J-PROB-001 → J-PROB-011`

## ① Mục tiêu & vị trí trong mạch
Đây là **bộ ba câu hỏi phỏng vấn kinh điển** (đặc biệt phong cách Trung Quốc) — ai làm cache ở quy mô thật đều phải nắm. Bài này dùng mọi thứ trước đó (TTL, key, invalidation, consistency) để trả lời: khi **tải thật** đập vào, cache hỏng theo những kiểu nào và **phòng thủ nhiều lớp** ra sao. Thêm **hot key** và **big key** — hai "bệnh vận hành" Redis.

## ② Giảng cơ bản → nâng cao

**(a) Phân biệt rõ ba khái niệm (J-PROB-001) — ĐỪNG LẪN.**

| Bệnh | Nguyên nhân | Hệ quả |
|---|---|---|
| **Penetration** (xuyên thủng) | Query data **KHÔNG tồn tại** (cache lẫn DB) — id rác / tấn công | **Mọi lần** đều xuống DB (cache không bao giờ điền) |
| **Breakdown / Stampede** (đánh sập điểm) = *dogpile / thundering herd / cache stampede* | **MỘT hot key** hết hạn | **Loạt request đồng thời** cùng miss → cùng nạp lại → đập DB |
| **Avalanche** (tuyết lở) | **NHIỀU key** hết hạn cùng lúc (hoặc **Redis chết**) | **DB ngập** đồng loạt |

Mnemonic: *penetration = key không có thật · breakdown = một key nóng vỡ · avalanche = nhiều key đổ cùng lúc.*

**(b) Chống Penetration (J-PROB-002, 003, 011).**
- **(1) Negative caching:** cache **null/empty marker TTL ngắn** (Bài 3) → id rác lần sau hit cache null, không xuống DB.
- **(2) Bloom filter:** cấu trúc **membership xác suất** — trả "**chắc chắn không có**" (chặn sớm) hoặc "**có thể có**" (cho qua). Có **false positive** (cho qua nhầm — chấp nhận được), **KHÔNG false negative**. Tốn ít bộ nhớ. Nhược: **khó xoá phần tử** (cần counting bloom). *(verify.)*
- **(3) Validate input** (format id) + **rate limit** theo client.

> **Tấn công cố ý (J-PROB-011):** adversary sinh key **đa dạng** → **null-cache có thể phình** (đẻ vô số null key tốn RAM) → **null-cache đơn thuần KHÔNG đủ**. Cần **Bloom filter** (chặn trước khi tạo null key) + validate format + rate limit.

**(c) Chống Breakdown/Stampede (J-PROB-004, 005).**
- **Mutex / single-flight:** khi hot key miss, **chỉ 1 request** lấy lock (`SET NX`) để rebuild, **số còn lại chờ / đọc stale** rồi lấy từ cache → giảm từ **N query DB còn 1**. *(verify cú pháp; lock đúng xem I-DLOCK.)*
- **Probabilistic early expiration / logical TTL (J-PROB-005) — lợi hơn mutex:** refresh **sớm ngẫu nhiên trước hạn** (xác suất refresh tăng khi gần expiry) → **một request lẻ** làm mới nền **trước khi key thật sự hết** → không ai cùng miss. **Không cần lock toàn cục** → tránh nghẽn ở mutex. Biến thể: "**never expire + async refresh**" (key không TTL cứng, background tự làm mới).

**(d) Chống Avalanche.**
- **Do mass expiry (J-PROB-006):** nhiều key set cùng TTL (warm cùng lúc / TTL cố định) → hết hạn đồng loạt → DB sốc. Chống bằng **TTL jitter**: `TTL = base + random(offset)` để **rải thời điểm hết hạn**, tránh đồng pha.
- **Do Redis sập (J-PROB-007) — phòng thủ NHIỀU LỚP:** (1) **HA** (replica/sentinel/cluster), (2) **circuit breaker + rate limit** bảo vệ DB khi cache mất, (3) **local L1 fallback**, (4) **graceful degradation** (trả stale/partial). **Không để DB nhận full traffic trần.** (Cross-ref M DDoS, K circuit breaker.)

**(e) Hot key (J-PROB-008).** Một single key nhận **quá nhiều traffic** → **shard/node Redis giữ key đó thành bottleneck** (CPU/băng thông node đó) **dù hit 100%**. Giảm: **đọc từ replica**, **local cache hot key ở app** (L1), **key splitting** (chia N bản `key#0..key#N`, đọc ngẫu nhiên 1), **client-side cache**.

**(f) Big key (J-PROB-009).** Value quá lớn / collection khổng lồ → thao tác trên nó **block single-thread** (O(n)) → tăng latency **mọi client**; tốn mạng khi truyền; lệch memory giữa shard; nguy hiểm khi `DEL`/expire (blocking). Giải: **`UNLINK`** (xoá async) thay `DEL`, **scan-từng-phần** (`HSCAN`/`SSCAN`), **tách nhỏ** key. *(verify.)*

**(g) Stampede vs Avalanche cần phương án khác nhau (J-PROB-010).** Stampede = **1 key** → giải bằng **single-flight/lock per-key**. Avalanche = **nhiều key/đồng loạt** → giải bằng **rải TTL + HA + bảo vệ DB tổng thể**. Nhầm phương án sẽ **không chữa đúng bệnh** (đặt mutex cho từng key không cứu nổi avalanche; rải TTL không cứu nổi 1 hot key).

## ③ ⚠️ Kiến thức cũ / bị thay thế

| Cái cũ | Thay bằng | Vì sao |
|---|---|---|
| TTL cố định cho cả batch key | **TTL jitter** (base + random) | Tránh mass-expiry avalanche |
| Để N request cùng rebuild hot key | **Single-flight/mutex** hoặc **probabilistic early refresh** | Giảm N→1 query DB lúc expiry |
| `DEL` big key | **`UNLINK`** + tách nhỏ | DEL block single-thread |
| Null-cache là đủ chống penetration | Null-cache **+ Bloom filter** (+ validate + rate limit) | Tấn công đa dạng key làm phình null-cache |
| `KEYS *` để tìm hot/big key | `SCAN` + `redis-cli --bigkeys` / `--hotkeys` | KEYS block (Bài 7) |

## ④ Áp dụng thực tế + So sánh bigtech
**Use case:** flash sale — trang sản phẩm khuyến mãi là **hot key** + nhiều cache **warm cùng lúc** lúc mở sale. Phòng: **TTL jitter** cho catalog + **single-flight** cho key sản phẩm hot + **L1 local** + **circuit breaker** trước DB. Bot quét id sản phẩm ngẫu nhiên → **Bloom filter** + negative cache + rate limit.

**Bigtech (pattern phổ biến):** thuật ngữ "thundering herd"/"dogpile" rất phổ biến phương Tây; Facebook giải bằng **leases** (paper Memcached). Probabilistic early expiration được mô tả trong paper "Optimal Probabilistic Cache Stampede Prevention" (XFetch). *(verify khi cần trích chính xác.)*

## ⑤ Code thực hành + cấu hình

**TTL jitter (chống avalanche):**
```js
const jitter = (base) => base + Math.floor(Math.random() * base * 0.2); // ±20%
await redis.set(key, val, "EX", jitter(600)); // 600s ± 20% → rải expiry
```

**Single-flight (chống stampede) — phác thảo:**
```js
// Chỉ 1 request rebuild hot key; số còn lại chờ ngắn rồi đọc lại cache.
async function getWithSingleFlight(key, rebuild) {
  let cached = await redis.get(key);
  if (cached !== null) return JSON.parse(cached);

  const lockKey = `lock:${key}`;
  // SET NX PX = chỉ set nếu chưa có, TTL 5s (lock tự hết nếu rebuilder chết)
  const got = await redis.set(lockKey, "1", "NX", "PX", 5000); // verify; lock ĐÚNG xem I-DLOCK
  if (got === "OK") {
    try {
      const fresh = await rebuild();                 // chỉ 1 request đập DB
      await redis.set(key, JSON.stringify(fresh), "EX", jitter(600));
      return fresh;
    } finally {
      await redis.del(lockKey); // production: release-by-token (Lua) — xem I-DLOCK
    }
  }
  // Không lấy được lock → chờ ngắn rồi thử cache lại
  await new Promise(r => setTimeout(r, 50));
  cached = await redis.get(key);
  return cached !== null ? JSON.parse(cached) : rebuild(); // fallback
}
```

**Probabilistic early expiration (XFetch) — ý tưởng:**
```js
// Lưu kèm thời điểm tính (delta = thời gian rebuild). Refresh sớm với xác suất tăng dần.
// shouldRefresh = now - delta*beta*ln(rand()) >= expiry
// → gần expiry thì 1 request lẻ chủ động refresh nền, không ai cùng miss.
```
⚠️ Lock ở đây là phác thảo; **distributed lock đúng** (token, Lua release, fencing, Redlock caveats) nằm ở mục I (I-DLOCK). ⚠️ `redis-cli --bigkeys` / `--hotkeys` để soi *(verify)*.

## ⑥ Keywords cần nhớ
- 🧠 **Cần HIỂU:** 3 bệnh khác nhau & cách chống tương ứng; vì sao probabilistic > mutex; hot key bottleneck dù hit; big key block single-thread; tấn công penetration làm phình null-cache.
- 📌 **Cần THUỘC:** penetration=key không tồn tại / breakdown=1 hot key expiry / avalanche=nhiều key|Redis chết; jitter chống avalanche; single-flight chống stampede; Bloom filter (FP, không FN).
- 🛠️ **Cần LÀM ĐƯỢC:** thêm TTL jitter; viết single-flight; cấu hình Bloom filter + negative cache; dùng `UNLINK`/`SCAN` cho big key.

## ⑦ Mental model
> **Penetration = key ma → Bloom + null-cache. Breakdown = 1 key nóng vỡ → single-flight / early refresh. Avalanche = nhiều key đổ → TTL jitter + HA + circuit breaker. Hot/big key → split/L1/UNLINK.**

## ⑧ Câu hỏi phỏng vấn & thách đố
1. *(TB)* Phân biệt penetration / breakdown / avalanche. → (bảng ②a).
2. *(khó)* 2 cách chống penetration? → negative cache + Bloom filter (+validate, rate limit).
3. *(khó)* Bloom filter chống penetration thế nào, đánh đổi? → "chắc chắn không / có thể có"; FP có, FN không; khó xoá.
4. *(khó)* Chống stampede bằng single-flight thế nào? → 1 request lock rebuild, còn lại chờ/đọc stale → N→1.
5. *(khó)* Avalanche do mass-expiry chống bằng gì? → TTL jitter rải thời điểm hết hạn.
6. *(rất khó)* Hot key gây vấn đề gì dù hit 100%, giảm sao? → node giữ key thành bottleneck; replica/L1/key-splitting.

**Thách đố:** *"Bạn đặt mutex single-flight cho mọi key để 'chống stampede'. Lúc Redis restart (mọi key mất), hệ thống vẫn sập DB. Vì sao?"* → Đó là **avalanche do cache mất**, không phải stampede 1-key. Mutex per-key không cứu khi **hàng nghìn key cùng miss** (mỗi key vẫn cho 1 request xuống DB → tổng vẫn ngập). Cần **HA + circuit breaker + L1 fallback + warming có jitter** — đúng bệnh đúng thuốc (J-PROB-010).

## ⑨ Bài tập + tiêu chí tự chấm
- **BT1:** Thêm jitter + single-flight cho `getHotProduct`. *Đạt khi:* TTL có random; chỉ 1 request rebuild lúc miss; còn lại không đập DB.
- **BT2:** Thiết kế phòng thủ penetration cho `GET /user/:id` bị bot quét id ngẫu nhiên. *Đạt khi:* Bloom filter chặn id không tồn tại + negative cache TTL ngắn + rate limit, nêu rõ vì sao null-cache đơn thuần không đủ.

## ⑩ Đọc thêm
- redis.io/docs — *Bloom filter (Redis probabilistic)*, *Client-side caching*.
- Paper *Optimal Probabilistic Cache Stampede Prevention* (XFetch).
- Facebook *Scaling Memcache at Facebook* (leases / thundering herd).

> 🔎 **AI hay sai ở đây:** AI gộp ba bệnh làm một ("cache miss thì DB chịu"), và dùng một thuốc cho mọi bệnh. Kiểm tra: *đây là key-ma, một-key-nóng, hay nhiều-key-đổ?* — chọn phòng thủ tương ứng.

---

# 🎯 BÀI 6 — Eviction & Memory

> Phủ: `J-EVICT-001 → J-EVICT-007`

## ① Mục tiêu & vị trí trong mạch
Bài 5 cho thấy key có thể **mất do Redis chết**; bài này giải thích key còn **biến mất do eviction** ngay cả khi **chưa hết TTL** — và vì sao điều đó nguy hiểm nếu Redis vừa làm cache vừa giữ session/job. Nền vận hành trực tiếp cho Bài 7 (Redis nội tại) và Bài 10 (queue).

## ② Giảng cơ bản → nâng cao

**(a) Eviction vs Expiration (J-EVICT-001).**
- **Expiration (TTL):** key **tự hết hạn** theo thời gian bạn đặt.
- **Eviction:** khi memory chạm **`maxmemory`**, Redis **chủ động xoá key** (theo policy) để nhường chỗ cho data mới. 
- **Độc lập nhau:** eviction có thể xoá **cả key chưa hết hạn**. Đây là chỗ dev hay sốc: "tôi đặt TTL 1h, sao key mất sau 10 phút?" → bị evict do đầy memory.

**(b) maxmemory-policy (J-EVICT-002) — *verify danh sách tại redis.io*.** Redis 8.x có **8 policy** (đã verify 6/2026):

| Policy | Phạm vi xét | Tiêu chí xoá |
|---|---|---|
| `noeviction` | — | **Không xoá**, trả lỗi OOM khi ghi thêm |
| `allkeys-lru` | mọi key | Least Recently Used |
| `allkeys-lfu` | mọi key | Least Frequently Used (Redis 4+) |
| `allkeys-random` | mọi key | Ngẫu nhiên |
| `volatile-lru` | key **có TTL** | LRU |
| `volatile-lfu` | key có TTL | LFU |
| `volatile-random` | key có TTL | Ngẫu nhiên |
| `volatile-ttl` | key có TTL | TTL ngắn nhất bị xoá trước |

**Default thường `volatile-lru`** (ElastiCache, Memorystore cũng mặc định `volatile-lru`). Chọn theo: *có chịu được eviction không* (không → `noeviction`) + *mọi key có TTL không* (`volatile-*` chỉ đụng key có TTL — nếu có key không TTL thì `volatile-*` có thể không giải phóng được gì) + *pattern recency/frequency*. **Pure cache** → `allkeys-lru`/`allkeys-lfu`. *(verify)*

**(c) LRU vs LFU (J-EVICT-003).** **LRU** đuổi "**lâu không dùng**" (recency). **LFU** đuổi "**ít dùng**" (frequency). **LFU thắng** khi hot key bị một **burst đọc-một-lần** đẩy ra oan (vd một lượt scan/crawler quét hết key → LRU coi chúng là "vừa dùng" và đẩy hot key thật ra = **cache pollution**). *(verify Redis LFU approx — dùng Morris counter + decay.)*

**(d) Vì sao "approximated" (J-EVICT-004).** Redis **không giữ danh sách LRU/LFU toàn cục** (quá tốn RAM/CPU) mà **sample ngẫu nhiên vài key** rồi chọn nạn nhân (eviction pool). **`maxmemory-samples`** (default 5): tăng → **chính xác hơn** (gần true LRU) nhưng **tốn CPU**. Đánh đổi *accuracy ↔ overhead*. (True LRU cần doubly-linked list mọi key — Redis từ chối vì memory.)

**(e) Trộn cache + store bền cùng instance (J-EVICT-005) — ⚠️ nguy hiểm.** Nếu Redis vừa làm **cache** vừa giữ **session/job** với cùng policy `allkeys-*` → có thể **xoá nhầm key "không được mất"** (session/job). Cache cần evict; store bền cần `noeviction`/persistence → **tách instance** hoặc tách logic. (Cross-ref J-QUEUE-007, J-BEYOND-004.)

**(f) Hit ratio tụt sau khi bật LRU (J-EVICT-006).** Triệu chứng: bật eviction → hit ratio rơi mạnh. Chẩn đoán: **working set > memory khả dụng** → **cache thrash** (đuổi key sắp dùng lại). Giải: **tăng memory**, **giảm thứ cache** (TTL/đừng cache thứ rẻ), **tách nóng/lạnh**, hoặc **shard thêm**. Đo `evicted_keys` cao + `keyspace_hits` thấp = under-sized.

**(g) Memory overhead/fragmentation (J-EVICT-007).** "Data 1GB" chiếm **hơn 1GB RAM** vì: **overhead per-key** (metadata, expire, pointer), **encoding nội bộ**, **fragmentation** của allocator (jemalloc). Quy tắc: **đệm dung lượng** (đặt maxmemory ~70–80% RAM, **không sát mép**), theo dõi `INFO memory` (`used_memory` vs `used_memory_rss` + `mem_fragmentation_ratio`). *(verify)*

## ③ ⚠️ Kiến thức cũ / bị thay thế

| Cái cũ | Thay bằng | Vì sao |
|---|---|---|
| "TTL đặt rồi key chắc chắn sống tới hết hạn" | Hiểu **eviction xoá cả key chưa hết hạn** | maxmemory đầy → evict bất kể TTL |
| `noeviction` cho cache (mặc định nghĩ vậy) | `allkeys-lru/lfu` cho **pure cache** | noeviction → OOM error khi đầy |
| Chung 1 Redis cache + session, `allkeys-*` | Tách instance / `volatile-*` + protect | Tránh evict nhầm session/job |
| Đặt maxmemory = 100% RAM | ~70–80% RAM | Overhead + fragmentation + buffer |
| "Redis LRU là true LRU" | **Approximated** (sampling), tinh chỉnh `maxmemory-samples` | True LRU quá tốn |

## ④ Áp dụng thực tế + So sánh bigtech
**Use case:** Redis cache catalog → `allkeys-lru`, maxmemory 80% RAM, monitor `evicted_keys`. Redis session/queue riêng → `noeviction` + AOF. Phát hiện crawler làm cache pollution (hit ratio tụt) → cân nhắc `allkeys-lfu`.

**Bigtech (pattern phổ biến):** ElastiCache (Redis/Valkey) & Memorystore mặc định `volatile-lru`; AWS khuyến nghị LRU cho cache cơ bản, LFU khi popular key bị evict oan; luôn để RAM dư cho buffer replication/AOF. *(verify defaults tại docs nhà cung cấp.)*

## ⑤ Code thực hách + cấu hình
Cấu hình Redis (redis.conf / `CONFIG SET`):
```conf
# redis.conf — instance làm PURE CACHE
maxmemory 4gb
maxmemory-policy allkeys-lru     # pure cache: evict thoải mái
maxmemory-samples 10             # gần true LRU hơn (tốn CPU hơn chút) — verify

# instance làm SESSION/QUEUE (store bền) → file/instance riêng:
# maxmemory-policy noeviction
# appendonly yes                 # AOF persistence
```
```bash
# Soi vận hành:
redis-cli CONFIG GET maxmemory-policy
redis-cli INFO memory   | grep -E 'used_memory:|used_memory_rss:|mem_fragmentation_ratio:|maxmemory:'
redis-cli INFO stats    | grep -E 'evicted_keys|keyspace_hits|keyspace_misses'
redis-cli --bigkeys     # tìm big key (verify)
```
⚠️ Đừng để cache instance dùng `noeviction` (sẽ OOM-error khi đầy). ⚠️ Đừng để store-bền dùng `allkeys-*` (mất session/job).

## ⑥ Keywords cần nhớ
- 🧠 **Cần HIỂU:** eviction ≠ expiration; LRU vs LFU & cache pollution; vì sao approximated; thrash khi working set > RAM; overhead/fragmentation; vì sao tách instance.
- 📌 **Cần THUỘC:** 8 maxmemory-policy + default volatile-lru; allkeys-lru cho pure cache, noeviction cho store bền; maxmemory ~70–80% RAM; `maxmemory-samples` default 5.
- 🛠️ **Cần LÀM ĐƯỢC:** chọn policy đúng theo use case; đọc `INFO memory`/`INFO stats` để chẩn đoán thrash/fragmentation; tách instance cache vs bền.

## ⑦ Mental model
> **maxmemory đầy → Redis evict theo policy, xoá cả key chưa hết TTL. Pure cache = allkeys-lru. Store bền (session/job) = noeviction + persistence + instance RIÊNG. LRU/LFU là xấp xỉ (sampling).**

## ⑧ Câu hỏi phỏng vấn & thách đố
1. *(dễ)* Eviction khác expiration thế nào? → expiration tự hết theo TTL; eviction xoá khi đầy memory, bất kể TTL.
2. *(TB)* Kể các maxmemory-policy chính + ý nghĩa. → (bảng ②b).
3. *(TB)* LRU vs LFU, khi nào LFU thắng? → recency vs frequency; LFU thắng khi burst đọc-một-lần (cache pollution).
4. *(khó)* Vì sao Redis LRU là approximated, `maxmemory-samples` ảnh hưởng gì? → sampling thay vì list toàn cục; tăng samples = chính xác hơn, tốn CPU.
5. *(khó)* Data 1GB sao chiếm >1GB RAM? → overhead per-key + encoding + fragmentation.

**Thách đố:** *"Bạn set TTL 1h cho mọi cache key, policy `noeviction`, maxmemory 2GB. Một ngày app bắt đầu báo lỗi 'OOM command not allowed'. Vì sao, sửa sao?"* → Memory đầy (key tích lại nhanh hơn TTL dọn), `noeviction` → Redis **từ chối ghi** thay vì evict → app lỗi. Sửa: đổi sang `allkeys-lru`/`volatile-lru` (cho phép evict), **và/hoặc** tăng RAM, **và/hoặc** giảm TTL/thứ cache. Bài học: cache instance **không nên** `noeviction`.

## ⑨ Bài tập + tiêu chí tự chấm
- **BT1:** Cho một Redis vừa cache vừa giữ session, đang `allkeys-lru`. Chỉ ra rủi ro + đề xuất sửa. *Đạt khi:* nhận ra session bị evict nhầm → tách instance (cache `allkeys-lru`, session `noeviction`+TTL).
- **BT2:** Hit ratio tụt sau khi bật eviction. Liệt kê 3 hướng chẩn đoán/sửa. *Đạt khi:* nêu working-set > RAM (thrash) + tăng RAM/giảm cache/tách nóng-lạnh/shard.

## ⑩ Đọc thêm
- redis.io/docs — *Key eviction* (maxmemory & policies).
- AWS *Database Caching Strategies Using Redis* — Evictions.
- redis.io — *Memory optimization*, `INFO memory` fields.

> 🔎 **AI hay sai ở đây:** AI cấu hình một Redis cho mọi việc (cache + session + queue) cùng `allkeys-lru` "cho tiện". Kiểm tra: *có key nào không-được-mất nằm chung instance với cache có evict không?* Có → tách.

---

# 🎯 BÀI 7 — Redis as Cache (bản chất)

> Phủ: `J-REDIS-001 → J-REDIS-009`

## ① Mục tiêu & vị trí trong mạch
Đến giờ mới "mở hộp đen" Redis: **vì sao nhanh dù single-thread**, **hệ quả vận hành** của single-thread, **data structures**, **persistence RDB/AOF**, **pipelining vs transaction**, **Lua**, và lỗi cache kinh điển AI hay tạo (cùng key cho mọi user). Bài này tổng hợp mọi vận hành Redis làm nền cho Bài 8–10.

## ② Giảng cơ bản → nâng cao

**(a) Vì sao Redis nhanh dù single-thread (J-REDIS-001).** *Command execution* chạy **1 luồng** nhưng vẫn nhanh vì:
- **In-memory** — không disk I/O trên đường nóng.
- **Single-thread → không lock/context-switch**, thao tác **atomic** không cần khoá.
- **I/O multiplexing** (epoll) xử lý nhiều kết nối hiệu quả.
- **Data structure tối ưu** (C, encoding gọn).
- Phần có thể chậm là **mạng** & **persistence**, không phải execution.
> *(verify)* Redis 6+ có **I/O threads** (đọc/ghi socket song song), Redis 8.x / **Valkey 8.x** mạnh hơn → "1 core" không còn tuyệt đối, nhưng *thực thi lệnh* vẫn tuần tự (giữ tính atomic).

**(b) Hệ quả vận hành single-thread (J-REDIS-002) — dev hay quên.** Một command **chậm O(n)** sẽ **block MỌI client**:
- `KEYS *` → **dùng `SCAN`** (cursor, non-blocking).
- big-key op → tránh (Bài 5).
- Lua nặng → giữ ngắn.
- Throughput giới hạn ~**1 core** cho execution → **scale = shard** (Bài 9).

**(c) Redis vs Memcached (J-REDIS-003).**

| | Memcached | Redis |
|---|---|---|
| Mô hình | Đơn giản, **multi-thread**, chỉ key-value string | Giàu **data structure**, persistence, replication, pub/sub, scripting |
| Hợp khi | Cache thuần đơn giản, cần multi-core thuần cho string | Đa năng, cần structure/HA/persistence |
| Lựa chọn mới | Ít dùng hơn | **Phần lớn case mới chọn Redis** (hoặc Valkey/DragonflyDB) |
*(verify)*

**(d) Data structures + use case (J-REDIS-004).**

| Structure | Use case cache/đời thật |
|---|---|
| **String** | giá trị serialize (JSON), counter (`INCR`) |
| **Hash** | object theo field (`HSET user:1 name ... age ...`) |
| **Sorted Set (zset)** | leaderboard, rate-limit theo time, top-N |
| **Set** | unique/tag, kiểm tra membership |
| **List** | queue đơn giản (`LPUSH`/`BRPOP`) |
| **Stream** | event/job log (Bài 10) |
| **Bitmap / HyperLogLog** | đếm/flag, đếm xấp xỉ unique (Bài 8) |
Chọn đúng structure **cắt được nhiều logic** (đừng tự sort trong app khi zset làm sẵn). *(verify)*

**(e) Persistence RDB vs AOF (J-REDIS-005).**
- **RDB** = snapshot định kỳ — **gọn**, restore nhanh, **mất data từ snapshot cuối**.
- **AOF** = ghi log mỗi write — **bền hơn**, file lớn/chậm hơn (có `appendfsync everysec/always`).
- **Cache thuần** → có thể **tắt persistence** (data tái tạo được từ DB) để nhanh nhất. **Store bền** (session/queue) → cần AOF/RDB. *(verify)*

**(f) Pipelining vs Transaction (J-REDIS-006).**
- **Pipelining** = gộp nhiều command **gửi 1 lượt** → cắt **round-trip mạng (RTT)**. **KHÔNG** đảm bảo atomic (lệnh khác có thể xen).
- **MULTI/EXEC** (transaction) = nhóm **atomic** (không xen giữa) nhưng **không rollback** như SQL (lệnh lỗi runtime vẫn chạy các lệnh khác). 
- Hai mục đích **khác nhau**: pipeline tối ưu throughput, MULTI đảm bảo atomic. *(verify)*

**(g) Lua script (J-REDIS-007).** Chạy nhiều thao tác **atomic phía server** (vd "compare-token rồi DEL" khi release lock, rate-limit atomic) → **tránh race check-then-act**. Rủi ro: script chạy lâu **block single-thread** → **giữ ngắn**. (Cross-ref I-DLOCK-003.)

**(h) Đặt TTL & lỗi "quên TTL" (J-REDIS-008).** `SET key val EX <giây>` / `PX <ms>` hoặc `EXPIRE key <giây>`. **Quên TTL** → key **tồn mãi** → memory phình, dễ chạm maxmemory (Bài 6) + giữ data stale. **Mọi cache value nên có TTL** trừ khi có lý do rõ. *(verify cú pháp)*

## ③ ⚠️ Kiến thức cũ / bị thay thế

| Cái cũ | Thay bằng | Vì sao |
|---|---|---|
| `KEYS *` | **`SCAN`** | KEYS block single-thread |
| `DEL` big key | **`UNLINK`** | DEL block |
| "Redis chỉ single-thread, max 1 core" | I/O threads (6+/8.x/Valkey) — execution vẫn tuần tự | "1 core" không còn tuyệt đối *(verify)* |
| Tự sort/đếm trong app | Dùng **zset/HLL/bitmap** | Structure đúng cắt logic |
| Cache value không TTL | **Luôn TTL** (trừ lý do rõ) | Tránh phình memory + stale |
| Coi MULTI/EXEC như SQL transaction (rollback) | Hiểu **không rollback**, dùng **Lua** cho atomic logic | Ngữ nghĩa khác |

## ④ Áp dụng thực tế + So sánh bigtech
**Use case:** leaderboard game = zset; đếm unique visitor = HLL; rate limit = zset/INCR; release distributed lock = Lua (compare-token→DEL). Cache thuần product = string JSON + TTL, **tắt persistence** instance đó.

**Bigtech (pattern phổ biến):** Facebook quy mô khổng lồ dùng **Memcached** (multi-thread, string thuần) cho lookaside cache; nhiều hệ thống mới chọn **Redis/Valkey** vì data structure + HA. **DragonflyDB** nổi lên như drop-in đa-core hiệu năng cao. *(verify landscape)*

## ⑤ Code thực hành + cấu hình
Pipelining + đúng cú pháp TTL + dùng structure đúng:
```js
import Redis from "ioredis";
const redis = new Redis(process.env.REDIS_URL);

// Pipelining: gộp nhiều lệnh, cắt RTT (KHÔNG atomic)
const pipe = redis.pipeline();
pipe.set("product:1", JSON.stringify(p1), "EX", 300);
pipe.set("product:2", JSON.stringify(p2), "EX", 300);
pipe.incr("metrics:writes");
await pipe.exec();

// Transaction atomic (MULTI/EXEC) — không rollback kiểu SQL
await redis.multi().incr("a").incr("b").exec();

// Leaderboard bằng zset
await redis.zincrby("leaderboard:game1", 10, "userA");
const top10 = await redis.zrevrange("leaderboard:game1", 0, 9, "WITHSCORES");

// SCAN thay KEYS để quét an toàn
let cursor = "0";
do {
  const [next, keys] = await redis.scan(cursor, "MATCH", "product:*", "COUNT", 100);
  cursor = next; /* xử lý keys */
} while (cursor !== "0");
```
⚠️ Không `KEYS *` trên production. ⚠️ Luôn kèm `EX`/`PX` khi set cache. *(verify cú pháp lệnh tại redis.io)*

## ⑥ Keywords cần nhớ
- 🧠 **Cần HIỂU:** vì sao nhanh (in-memory + single-thread atomic + epoll); hệ quả O(n) block; pipeline≠transaction; Lua atomic; RDB vs AOF; structure đúng cắt logic.
- 📌 **Cần THUỘC:** `SCAN` thay `KEYS`, `UNLINK` thay `DEL` big key; `SET..EX/PX`/`EXPIRE`; zset cho leaderboard; MULTI/EXEC không rollback.
- 🛠️ **Cần LÀM ĐƯỢC:** dùng pipeline cắt RTT; chọn structure đúng; viết Lua atomic ngắn; cấu hình persistence theo vai trò.

## ⑦ Mental model
> **Redis nhanh vì in-memory + thực thi tuần tự (atomic, không lock). Hệ quả: 1 lệnh chậm block tất cả → SCAN/UNLINK, giữ Lua ngắn. Chọn đúng data structure. Cache thuần tắt persistence; store bền bật AOF.**

## ⑧ Câu hỏi phỏng vấn & thách đố
1. *(TB)* Vì sao Redis nhanh dù single-thread? → in-memory + atomic không lock + epoll + structure tối ưu.
2. *(TB)* Hệ quả vận hành của single-thread? → lệnh O(n) block mọi client → SCAN/tránh big key/Lua ngắn; ~1 core.
3. *(TB)* Redis vs Memcached chọn khi nào? → Memcached đơn giản multi-thread string; Redis đa năng structure/HA.
4. *(khó)* Pipelining khác MULTI/EXEC? → pipeline cắt RTT không atomic; MULTI atomic không rollback.
5. *(khó)* RDB vs AOF, cache thuần nên thế nào? → snapshot vs log; cache thuần có thể tắt persistence.

**Thách đố (J-REDIS-009, ⭐⭐⭐⭐⭐):** *"AI viết code cache trả về object đã serialize nhưng dùng CÙNG key cho mọi user. Lỗi gì, kiểm tra ở đâu?"* → Thiếu **chiều phân biệt** (user/tenant/locale/permission) trong key → **trả data người này cho người khác** (rò dữ liệu/quyền). Kiểm tra: **key có gồm đủ context ảnh hưởng kết quả không**. Đây là lỗi **correctness/security** AI hay tạo — lặp lại từ Bài 3 (J-INVAL-004). *Hiểu để kiểm tra.*

## ⑨ Bài tập + tiêu chí tự chấm
- **BT1:** Refactor code dùng `KEYS user:*` + `DEL` big hash thành an toàn. *Đạt khi:* đổi `KEYS→SCAN`, `DEL→UNLINK`/`HSCAN`.
- **BT2:** Cài leaderboard top-10 + đếm unique visitor/ngày bằng structure Redis đúng. *Đạt khi:* zset cho leaderboard, HLL (`PFADD`/`PFCOUNT`) cho unique — không lưu mọi phần tử.

## ⑩ Đọc thêm
- redis.io/docs — *Data types*, *Pipelining*, *Transactions*, *Programmability (Lua/Functions)*, *Persistence*.
- redis.io/blog — Redis 8 (AGPLv3, vector sets, I/O).
- valkey.io / dragonflydb.io (lựa chọn thay thế).

> 🔎 **AI hay sai ở đây:** (1) cùng key cho mọi user; (2) `KEYS *` trên prod; (3) quên TTL; (4) coi MULTI/EXEC như rollback SQL. Mỗi cái là một mục review.

---

# 🎯 BÀI 8 — Redis ngoài Cache

> Phủ: `J-BEYOND-001 → J-BEYOND-006`

## ① Mục tiêu & vị trí trong mạch
Redis không chỉ là cache. Dùng data structures (Bài 7) + tính atomic + in-memory, một hạ tầng làm được nhiều việc: rate limiter, distributed lock, session store, leaderboard, pub/sub, đếm xấp xỉ. Bài này **nhận diện** các use case + **lưu ý riêng** mỗi cái (eviction/persistence), và **cross-ref** sang mục I/F/M nơi đào sâu.

## ② Giảng cơ bản → nâng cao

**(a) Vì sao một hạ tầng làm nhiều việc (J-BEYOND-001).** Redis **in-memory nhanh** + **atomic ops** + **đã có sẵn** → tận dụng cho: rate limiter, distributed lock, session store, leaderboard (zset), pub/sub, queue/streams, geospatial, đếm xấp xỉ (HLL). **Nhưng mỗi use case có lưu ý riêng** — đặc biệt **eviction & persistence** (đừng để evict nhầm thứ không-được-mất).

**(b) Rate limiter ở mức store (J-BEYOND-002).** `INCR` + `EXPIRE` (**fixed window**) hoặc **sorted set/Lua** (**sliding window**) để đếm **atomic**, **share state** giữa instance. *Thuật toán (token bucket/sliding window) & đặt ở gateway vs app: đào sâu ở **F-RL, M**.* J chỉ chạm **góc store**. *(verify cú pháp)*

**(c) Leaderboard / top-N (J-BEYOND-003).** **zset** giữ phần tử theo **score có thứ tự**: `ZADD`/`ZINCRBY` **O(log n)**, `ZREVRANGE` lấy top-N nhanh, `ZRANK` lấy hạng. **Không phải sort lại toàn bộ** mỗi lần như DB → realtime leaderboard cực hợp. *(verify)*

**(d) Session store (J-BEYOND-004).** Session **share giữa instance** (không cần sticky session) + nhanh. **Nhưng session "không được mất"** → **KHÔNG** để `allkeys-*` đuổi nhầm (Bài 6). TTL = session timeout + **policy phù hợp / instance riêng**. (Cross-ref M session góc security, J-EVICT-005.)

**(e) Distributed lock (J-BEYOND-005) — chỉ nhận diện.** `SET NX PX` + token, release bằng **compare-token (Lua)**. Cạm bẫy TTL/failover/**fencing token** → **toàn bộ phân tích Redlock/fencing nằm ở mục I (I-DLOCK)**. Ở đây chỉ cần biết: **Redis-as-lock là một use case có cạm bẫy**, đừng tự tin nó "đơn giản". (Cross-ref I-DLOCK-002/005/006.)

**(f) HyperLogLog / bitmap (J-BEYOND-006).** Đếm **unique visitors**: **HLL** ước lượng **cardinality** với bộ nhớ **cố định cực nhỏ** (sai số ~0.81%) thay vì lưu mọi phần tử (set tốn O(n)). **Bitmap** cho đếm/flag theo user-id liên tục (vd daily active). **Chấp nhận xấp xỉ đổi lấy memory.** *(verify)*

## ③ ⚠️ Kiến thức cũ / bị thay thế

| Cái cũ | Thay bằng | Vì sao |
|---|---|---|
| Sticky session ở load balancer | **Session store Redis** share | Không khoá user vào 1 node, scale ngang |
| Đếm unique bằng `SET` (lưu mọi id) | **HyperLogLog** (`PFADD`/`PFCOUNT`) | Set O(n) RAM; HLL cố định nhỏ |
| Sort leaderboard trong DB mỗi lần | **zset** (`ZINCRBY`/`ZREVRANGE`) | O(log n), không re-sort toàn bộ |
| Lock tự chế `SETNX` rồi `DEL` | `SET NX PX` + **compare-token Lua** (chi tiết I-DLOCK) | Tránh xoá nhầm lock của người khác |
| Chung instance cho session + cache `allkeys-lru` | Tách / `volatile-*` + protect | Tránh evict nhầm session |

## ④ Áp dụng thực tế + So sánh bigtech
**Use case:** app scale ngang → session ở Redis riêng (`noeviction`/TTL). Game realtime → leaderboard zset. API public → rate limit Redis (INCR/zset) ở gateway. Analytics → HLL đếm DAU/unique.

**Bigtech (pattern phổ biến):** Redis/Valkey là "dao đa năng" phổ biến cho rate-limit, session, leaderboard ở nhiều công ty; HLL được dùng rộng cho đếm cardinality lớn (vd unique view). *(verify khi cần số liệu cụ thể)*

## ⑤ Code thực hành + cấu hình
```js
import Redis from "ioredis";
const redis = new Redis(process.env.REDIS_URL);

// Rate limit fixed window (góc store; thuật toán xem F-RL)
async function allow(userId, limit = 100, windowSec = 60) {
  const k = `rl:${userId}:${Math.floor(Date.now() / 1000 / windowSec)}`;
  const n = await redis.incr(k);
  if (n === 1) await redis.expire(k, windowSec);
  return n <= limit;
}

// Leaderboard
await redis.zincrby("lb:season1", 50, "userA");
const top = await redis.zrevrange("lb:season1", 0, 9, "WITHSCORES");
const rank = await redis.zrevrank("lb:season1", "userA"); // hạng (0-based)

// Unique visitors bằng HLL
await redis.pfadd("uv:2026-06-18", "userA", "userB");
const approxUnique = await redis.pfcount("uv:2026-06-18"); // xấp xỉ, RAM cố định
```
⚠️ Session/lock/queue: dùng instance/policy phù hợp (đừng `allkeys-*`). ⚠️ Distributed lock đúng → I-DLOCK, không tự chế.

## ⑥ Keywords cần nhớ
- 🧠 **Cần HIỂU:** vì sao 1 hạ tầng đa năng; mỗi use case có lưu ý eviction/persistence; HLL đánh đổi xấp xỉ↔RAM; vì sao zset hợp leaderboard; session không được evict.
- 📌 **Cần THUỘC:** rate-limit store = INCR+EXPIRE / zset; leaderboard = zset; unique = HLL (PFADD/PFCOUNT); lock = SET NX PX + Lua (chi tiết I).
- 🛠️ **Cần LÀM ĐƯỢC:** cài rate-limit store, leaderboard zset, HLL counter; biết khi nào trỏ sang I/F/M.

## ⑦ Mental model
> **Redis = dao đa năng (nhanh + atomic + sẵn có): rate-limit, lock, session, leaderboard, pub/sub, HLL. Nhưng thứ "không được mất" (session/job/lock) cần policy/instance riêng, đừng để cache evict nhầm.**

## ⑧ Câu hỏi phỏng vấn & thách đố
1. *(dễ)* Ngoài cache, Redis hay làm gì? → rate-limit, lock, session, leaderboard, pub/sub, queue, HLL.
2. *(TB)* Vì sao zset hợp leaderboard? → score có thứ tự, ZINCRBY/ZREVRANGE O(log n), không re-sort.
3. *(TB)* Session store Redis lợi gì, eviction phải thế nào? → share không sticky; không `allkeys-*` (mất session).
4. *(khó)* HLL vs set thường để đếm unique? → HLL xấp xỉ RAM cố định; set chính xác nhưng O(n) RAM.
5. *(khó)* Vì sao Redis-as-lock "không đơn giản"? → TTL/failover/fencing → xem I-DLOCK.

**Thách đố:** *"Team dùng cùng một Redis (`allkeys-lru`) cho cache, session đăng nhập, và rate-limit. Người dùng thỉnh thoảng bị đăng xuất ngẫu nhiên. Vì sao?"* → Khi memory đầy, `allkeys-lru` **evict cả session key** (không được mất) → user mất session → bị logout. Sửa: **tách instance** session (`noeviction`/TTL) khỏi cache, hoặc tối thiểu để session key thuộc nhóm protected. (Đúng bệnh J-EVICT-005/J-BEYOND-004.)

## ⑨ Bài tập + tiêu chí tự chấm
- **BT1:** Cài đếm DAU (daily active users) bằng HLL theo ngày. *Đạt khi:* PFADD theo `uv:YYYY-MM-DD`, PFCOUNT, giải thích xấp xỉ + RAM cố định.
- **BT2:** Thiết kế hạ tầng Redis cho app có cache + session + rate-limit. *Đạt khi:* tách session ra policy/instance an toàn, cache `allkeys-lru`, rate-limit store atomic.

## ⑩ Đọc thêm
- redis.io/docs — *Sorted sets*, *HyperLogLog*, *Bitmaps*, *Pub/Sub*.
- Mục **I-DLOCK** (distributed lock), **F-RL / M** (rate limiting), **M** (session security) trong project.

> 🔎 **AI hay sai ở đây:** AI nhét mọi thứ vào một Redis với policy cache. Kiểm tra: *key nào không-được-mất?* → tách khỏi eviction. Và đừng để AI tự viết distributed lock "đơn giản" — trỏ về I-DLOCK.

---

# 🎯 BÀI 9 — Redis Scaling (Replica · Cluster · Sentinel · RediSearch)

> Phủ: `J-CLUSTER-001 → J-CLUSTER-006`

## ① Mục tiêu & vị trí trong mạch
Bài 7 nói execution ~1 core; bài này trả lời: khi **một node hết đủ** thì scale thế nào — **replica** (scale đọc + HA), **cluster/hash slots** (scale ghi + dung lượng), **Sentinel** (failover). Cộng giới hạn **multi-key trong cluster**, hệ quả **replication async** cho cache vs lock, và **RediSearch/modules**.

## ② Giảng cơ bản → nâng cao

**(a) Khi nào scale & các hướng (J-CLUSTER-001).** Một node hết khi: **vượt RAM 1 node** hoặc **throughput > ~1 core / băng thông**. Trước tiên **nhận diện nghẽn** (memory vs CPU vs network). Hướng:
- **Replica** (master-replica): scale **đọc** (đọc từ replica) + **HA**.
- **Cluster / sharding**: scale **ghi** + **dung lượng** (chia data nhiều node).
- **Client-side sharding**: client tự băm key tới node.

**(b) Redis Cluster — hash slots (J-CLUSTER-002).** **16384 hash slot**; key → **CRC16(key) mod 16384** → slot → node giữ slot đó. **Reshard** = di chuyển slot giữa node → kiểm soát phân bố. *(verify số slot/cơ chế tại redis.io)*

**(c) Multi-key trong Cluster & hash tag (J-CLUSTER-003).** Lệnh đụng **nhiều key** phải **cùng slot/node** (không cross-slot) → `MGET`/transaction/Lua đa-key bị giới hạn. **Hash tag `{...}`** ép các key liên quan vào **cùng slot**: `user:{42}:profile` và `user:{42}:settings` cùng slot (chỉ phần trong `{}` được băm). **Thiết kế key phải tính trước.** *(verify)*

**(d) Sentinel vs Cluster (J-CLUSTER-004).**

| | Sentinel | Cluster |
|---|---|---|
| Giải bài toán | **HA/failover** cho master-replica (**không shard**) | **Sharding** + HA cho data vượt 1 node |
| Cơ chế | Bầu master mới khi master chết | Chia 16384 slot + failover per-shard |
| Chọn khi | Chỉ cần failover, data vừa 1 node | Cần shard (data/ghi vượt 1 node) |
*(verify)*

**(e) Replication async — hệ quả (J-CLUSTER-005).** Redis replication là **async** → write trên master có thể **chưa sang replica** khi failover → **mất vài write gần nhất**. **Cache** thường **chấp nhận được** (chỉ tăng miss). **Lock/correctness** thì **nguy hiểm**: 2 client cùng giữ lock sau failover → chính là caveat **Redlock** (Kleppmann). (Cross-ref I-DLOCK-011.) *(verify)*

**(f) RediSearch / modules (J-CLUSTER-006).** Module thêm **secondary index / full-text / vector search / aggregation** trên data đã ở Redis → query phức tạp hơn key-value. Cân nhắc khi cần **search nhanh trên data đang ở Redis**, nhưng **không thay full search engine** (Elasticsearch) cho nhu cầu search lớn. *(verify — Redis 8 đã tích hợp Redis Query Engine/JSON/etc vào core dưới tri-license)*

## ③ ⚠️ Kiến thức cũ / bị thay thế

| Cái cũ | Thay bằng | Vì sao |
|---|---|---|
| "Scale Redis = thêm RAM mãi" | Replica (đọc) + Cluster (ghi/dung lượng) | 1 node có trần RAM/CPU |
| Multi-key tuỳ tiện trong cluster | **Hash tag `{}`** gom cùng slot | Lệnh đa-key cần cùng slot |
| "Redlock chắc chắn đúng" | Hiểu **replication async** → caveat (xem I-DLOCK) | Failover mất write → 2 client giữ lock |
| Dùng Redis làm search engine chính | RediSearch cho vừa phải; lớn → Elasticsearch | Module không thay full search |
| Sentinel = sharding | Sentinel = **failover**, Cluster = sharding | Hai bài toán khác |

## ④ Áp dụng thực tế + So sánh bigtech
**Use case:** đọc nhiều → thêm replica + đọc từ replica (chấp nhận stale nhẹ cho cache). Data vượt RAM 1 node → Cluster, thiết kế key có hash tag cho nhóm cần cùng slot. Cần failover nhưng data vừa → Sentinel.

**Bigtech (pattern phổ biến):** managed Redis/Valkey (ElastiCache/Memorystore) lo replica/cluster/failover. Hệ thống cực lớn shard sớm + đọc-từ-replica; cẩn thận replication lag khi đọc replica (cross-ref E read-from-replica). *(verify)*

## ⑤ Code thực hành + cấu hình
```js
import Redis from "ioredis";
// Redis Cluster client
const cluster = new Redis.Cluster([
  { host: "10.0.0.1", port: 6379 },
  { host: "10.0.0.2", port: 6379 },
]);

// Hash tag: ép cùng slot để dùng lệnh đa-key / transaction
await cluster.mset(
  "user:{42}:profile", JSON.stringify(profile),   // {42} quyết định slot
  "user:{42}:settings", JSON.stringify(settings), // cùng slot → MSET hợp lệ
);
// "user:43:profile" + "user:42:profile" KHÁC slot → MGET cross-slot sẽ lỗi
```
```bash
redis-cli --cluster check 10.0.0.1:6379     # kiểm tra slot coverage (verify)
redis-cli CLUSTER NODES | head
```
⚠️ Đừng dựa replica cho **đọc cần đúng tuyệt đối** (lag). ⚠️ Lock qua Redis cluster: đọc kỹ I-DLOCK (replication async).

## ⑥ Keywords cần nhớ
- 🧠 **Cần HIỂU:** nhận diện nghẽn trước khi scale; replica(đọc/HA) vs cluster(ghi/dung lượng) vs sentinel(failover); replication async → cache ok / lock nguy hiểm; module không thay search engine.
- 📌 **Cần THUỘC:** 16384 hash slot, CRC16 mod; hash tag `{}` gom slot; Sentinel=failover, Cluster=shard; multi-key cần cùng slot.
- 🛠️ **Cần LÀM ĐƯỢC:** thiết kế key có hash tag; chọn replica/cluster/sentinel theo nhu cầu; cấu hình cluster client.

## ⑦ Mental model
> **1 node hết → replica (đọc/HA), cluster (ghi/dung lượng, 16384 slot/hash tag), sentinel (failover). Replication ASYNC: cache chịu được, lock thì KHÔNG (xem I-DLOCK).**

## ⑧ Câu hỏi phỏng vấn & thách đố
1. *(TB)* Khi nào 1 node hết đủ, các hướng scale? → vượt RAM/throughput; replica/cluster/client-shard; nhận diện nghẽn trước.
2. *(khó)* Redis Cluster chia data thế nào? → 16384 hash slot, CRC16 mod, node giữ slot.
3. *(khó)* Multi-key trong cluster bị giới hạn gì, hash tag giải sao? → phải cùng slot; `{}` ép cùng slot.
4. *(TB)* Sentinel vs Cluster khác bài toán gì? → failover (không shard) vs sharding+HA.
5. *(rất khó)* Replication async ảnh hưởng cache vs lock thế nào? → cache chấp nhận mất write; lock → 2 client giữ → nguy hiểm.

**Thách đố:** *"Bạn `MGET user:1:a user:2:b` trên Redis Cluster và bị lỗi CROSSSLOT. Vì sao, fix sao mà vẫn shard được?"* → `user:1:a` và `user:2:b` băm ra **slot khác nhau** → lệnh đa-key cross-slot bị từ chối. Fix: **không** ép mọi thứ về 1 slot (mất shard); thay vào đó **gom theo entity** dùng hash tag khi *thực sự* cần atomic đa-key (`user:{1}:a`, `user:{1}:b`), còn lookup đa-user khác nhau thì **pipeline nhiều GET** thay vì 1 MGET.

## ⑨ Bài tập + tiêu chí tự chấm
- **BT1:** Thiết kế key cho "giỏ hàng + tổng tiền" của 1 user cần cập nhật atomic trong cluster. *Đạt khi:* dùng hash tag `cart:{userId}:items` + `cart:{userId}:total` → cùng slot.
- **BT2:** Quyết định Sentinel hay Cluster cho: (a) 5GB data, cần failover; (b) 200GB data. *Đạt khi:* (a) Sentinel, (b) Cluster + lý do dung lượng.

## ⑩ Đọc thêm
- redis.io/docs — *Scale with Redis Cluster*, *High availability with Sentinel*, *Replication*.
- redis.io — *Redis Query Engine* (RediSearch), Redis 8 modules.
- Kleppmann — *How to do distributed locking* (replication caveat; mục I).

> 🔎 **AI hay sai ở đây:** AI viết `MGET`/transaction đa-key trên cluster không nghĩ tới slot → CROSSSLOT lỗi runtime. Kiểm tra: *các key trong một lệnh đa-key có cùng slot (hash tag) không?*

---

# 🎯 BÀI 10 — Background Jobs: BullMQ / Redis Streams

> Phủ: `J-QUEUE-001 → J-QUEUE-007`

## ① Mục tiêu & vị trí trong mạch
Redis còn làm **job queue**. Bài này: vì sao cần background queue, **BullMQ** (chuẩn de-facto Node, dựa Redis Streams), **Streams vs list-based queue**, đảm bảo **không mất job**, **DLQ/retry**, khi nào **không đủ → Kafka/RabbitMQ**, và vì sao **tách Redis queue khỏi Redis cache** (nối Bài 6). Mở đường sang mục K (Messaging).

## ② Giảng cơ bản → nâng cao

**(a) Vì sao cần job queue (J-QUEUE-001) — 3 lớp vấn đề.**
1. **Tách việc chậm khỏi request** (transcode video, gửi email) → HTTP không treo, trả 200 ngay.
2. **Làm phẳng spike** (buffer khi burst vượt downstream) → không sập service phụ thuộc.
3. **Retry việc lỗi tạm thời** thay vì fail cả request.
+ decouple producer/consumer + **scale worker riêng**.

**(b) BullMQ (J-QUEUE-002).** Thư viện queue **Node trên Redis** (kế nhiệm **Bull**, **TypeScript-native**; nội bộ tận dụng **Redis Streams**). *(verify version — 6/2026 là **5.78.x**.)* Primitives: **job states** (waiting/active/delayed/completed/failed), **retry + backoff**, **delay**, **priority**, **rate limit**, **repeatable**, **parent-child flow**, dashboard (**Bull Board**).

**(c) Redis Streams vs list-based queue (J-QUEUE-003).**

| | List (`LPUSH`/`BRPOP`) | Streams |
|---|---|---|
| Mô hình | Queue đơn giản, lấy ra là mất | **Append-only log** có ID |
| Replay | ❌ | ✅ (đọc lại lịch sử) |
| Nhiều consumer độc lập | ❌ | ✅ (**consumer group** đọc độc lập) |
| At-least-once | Tự xử | **`XACK`** + pending list + **reclaim** job worker chết |

`XADD` thêm message, `XREADGROUP` đọc theo group, `XACK` xác nhận đã xử lý. Consumer group **chia message giữa workers** + theo dõi **pending** cho at-least-once + **reclaim** job của worker chết. *(verify lệnh)*

**(d) Không mất job khi worker crash (J-QUEUE-004).** Job phải được **claim atomic** + đánh dấu **đang xử lý**, **ACK chỉ khi xong**. Worker chết → job **pending** được **reclaim/retry** (visibility timeout / stalled-job check). Vì **at-least-once** → consumer **phải idempotent** (cross-ref I-IDEM). BullMQ có cơ chế stalled-job tự reclaim.

**(e) DLQ / failed jobs + retry an toàn (J-QUEUE-005).** Job fail **quá N lần** → đẩy sang **DLQ/failed** để điều tra thay vì **retry vô hạn**. Retry phải **có giới hạn** + **(jittered) backoff**. Job phải **idempotent** vì retry chạy lại. (Cross-ref I-LOCK-008.)

**(f) Khi nào không đủ → Kafka/RabbitMQ (J-QUEUE-006).**
- **Kafka:** cần **event streaming bền** + **replay lâu dài** + **throughput rất lớn** + nhiều consumer độc lập.
- **RabbitMQ:** **routing phức tạp** / AMQP.
- **BullMQ** hợp **job queue Node** với retry/delay, hạ tầng **nhẹ** (đã có Redis). (Đào sâu ở mục K.) *(verify landscape)*

**(g) Tách Redis queue khỏi Redis cache (J-QUEUE-007) — nối Bài 6.** Cache cần **eviction** (`allkeys-lru`); job **"không được mất"** → cần **`noeviction`/persistence**. Tải & failure mode khác nhau. Chung instance → **eviction xoá nhầm job** hoặc job ngốn memory **đẩy cache** ra. → **Tách instance.** (Cross-ref J-EVICT-005.)

## ③ ⚠️ Kiến thức cũ / bị thay thế

| Cái cũ | Thay bằng | Vì sao |
|---|---|---|
| **Bull** | **BullMQ 5.x** (TS-native, Redis Streams) *(verify)* | Bull đã được kế nhiệm |
| List-based queue tự chế (`LPUSH`/`BRPOP`) | **Streams + consumer group** (hoặc BullMQ) | List không replay/không group/khó at-least-once |
| Retry vô hạn không backoff | Giới hạn N + **jittered backoff** + **DLQ** | Tránh retry storm; điều tra job hỏng |
| Chung Redis cache + queue, `allkeys-lru` | **Tách instance** (queue `noeviction`/persist) | Evict nhầm job |
| Consumer không idempotent | **Idempotent** (at-least-once) | Job có thể chạy lại (cross-ref I-IDEM) |

## ④ Áp dụng thực tế + So sánh bigtech
**Use case:** signup → trả 200 ngay, đẩy job "welcome email" vào BullMQ; worker riêng xử lý + retry + DLQ. Video upload → transcode job (chậm) tách khỏi request. Throughput cực lớn + cần replay/streaming bền (event sourcing, analytics pipeline) → **Kafka**.

**Bigtech (pattern phổ biến):** Node shop thường BullMQ cho job nội bộ; hệ thống event-driven lớn dùng **Kafka** cho backbone. **DragonflyDB** được quảng cáo tương thích BullMQ (đa-core). Chạy worker là **process riêng**, dashboard (Bull Board) **phải auth** (đừng phơi ra internet). *(verify)*

## ⑤ Code thực hành + cấu hình
BullMQ producer + worker (pin `bullmq@5`, verify bản mới nhất):
```js
// queue.js
import { Queue, Worker } from "bullmq";
import IORedis from "ioredis";

// BullMQ yêu cầu maxRetriesPerRequest: null; nên dùng Redis RIÊNG cho queue
const connection = new IORedis(process.env.REDIS_QUEUE_URL, { maxRetriesPerRequest: null });

export const emailQueue = new Queue("email", { connection });

// Producer: enqueue trong route, KHÔNG xử lý trong request
await emailQueue.add(
  "welcome",
  { userId, email },
  { attempts: 5, backoff: { type: "exponential", delay: 1000 }, removeOnComplete: 1000 }
); // attempts + backoff → retry an toàn; fail quá 5 lần → vào 'failed' (DLQ)

// Worker (process RIÊNG)
new Worker(
  "email",
  async (job) => {
    // ⚠️ phải IDEMPOTENT: job có thể chạy lại (at-least-once)
    await sendEmailIdempotent(job.data.userId, job.data.email);
  },
  { connection, concurrency: 10 }
);
```
```conf
# Redis instance của QUEUE (tách khỏi cache):
maxmemory-policy noeviction
appendonly yes
```
⚠️ **REDIS_QUEUE_URL ≠ REDIS_CACHE_URL** (tách instance). ⚠️ Worker chạy process riêng; dashboard phải có auth. ⚠️ Job idempotent (Bài cross-ref I-IDEM).

## ⑥ Keywords cần nhớ
- 🧠 **Cần HIỂU:** 3 lớp lý do dùng queue; Streams vs list (replay/group/ACK); at-least-once → idempotent; DLQ/backoff; khi nào cần Kafka; vì sao tách queue khỏi cache.
- 📌 **Cần THUỘC:** BullMQ trên Redis Streams; `XADD/XREADGROUP/XACK`; attempts+backoff+DLQ; queue instance `noeviction`+persist.
- 🛠️ **Cần LÀM ĐƯỢC:** dựng BullMQ producer/worker với retry/backoff/DLQ; worker idempotent; tách instance.

## ⑦ Mental model
> **Queue = tách việc chậm + làm phẳng spike + retry. BullMQ (trên Redis Streams) cho Node; consumer group + XACK + reclaim = at-least-once → job phải idempotent. Tách Redis queue (noeviction/persist) khỏi cache (allkeys-lru). Quá tải/streaming bền → Kafka.**

## ⑧ Câu hỏi phỏng vấn & thách đố
1. *(TB)* Ba lớp vấn đề job queue giải? → tách việc chậm, phẳng spike, retry lỗi tạm.
2. *(TB)* BullMQ là gì, dựa trên đâu? → queue Node trên Redis (Streams), kế nhiệm Bull, TS-native.
3. *(khó)* Streams khác list-queue thế nào, consumer group cho gì? → append-log+replay; group chia message+pending(XACK)+reclaim.
4. *(khó)* Đảm bảo job không mất khi worker crash? → claim atomic+ACK khi xong+reclaim pending; consumer idempotent.
5. *(rất khó)* Khi nào BullMQ không đủ, dùng Kafka? → streaming bền+replay lâu+throughput rất lớn+nhiều consumer.

**Thách đố:** *"Worker gửi email crash sau khi gửi xong nhưng trước khi ACK. Chuyện gì xảy ra, fix sao?"* → Job chưa ACK → bị **reclaim/retry** → email gửi **lần hai** (at-least-once → có thể trùng). Fix: **idempotent** — lưu "đã gửi cho job-id này" (dedupe key) hoặc dùng idempotency key phía provider email; đừng giả định exactly-once.

## ⑨ Bài tập + tiêu chí tự chấm
- **BT1:** Dựng BullMQ queue gửi email với retry 5 lần + exponential backoff + DLQ. *Đạt khi:* attempts+backoff cấu hình; job idempotent; worker process riêng.
- **BT2:** Giải thích vì sao đặt queue chung Redis với cache `allkeys-lru` là sai + sửa. *Đạt khi:* chỉ ra evict nhầm job → tách instance `noeviction`+persistence.

## ⑩ Đọc thêm
- bullmq.io/docs — Queues, Workers, Flows, Rate limiting (verify version 5.x).
- redis.io/docs — *Streams* (`XADD/XREADGROUP/XACK`), consumer groups.
- Mục **K (Messaging)**: Kafka vs RabbitMQ vs Redis Streams, Outbox/CDC.

> 🔎 **AI hay sai ở đây:** AI giả định "exactly-once" và viết consumer không idempotent; và đặt queue chung Redis cache. Kiểm tra: *job có chạy 2 lần được không hại?* + *queue có nằm trên instance có eviction không?*

---

# 🎯 BÀI 11 — HTTP / CDN Caching

> Phủ: `J-HTTP-001 → J-HTTP-006`

## ① Mục tiêu & vị trí trong mạch
Tầng cache **xa nhất** (gần user nhất) — đóng lại bức tranh "các tầng" mở ở Bài 1. Học `Cache-Control`, `ETag`, **CDN**, vì sao GraphQL/POST khó cache, **purge & `Vary`**, và bẫy **cache response cá nhân hoá** (rò data giữa user). Chuẩn: **RFC 9111**.

## ② Giảng cơ bản → nâng cao

**(a) `Cache-Control` (J-HTTP-001).** Header điều khiển cache HTTP:

| Directive | Ý nghĩa |
|---|---|
| `max-age=N` | TTL (giây) cho **client** cache |
| `s-maxage=N` | TTL cho **shared cache / CDN** (ghi đè max-age ở CDN) |
| `no-store` | **Không lưu gì** (data nhạy cảm) |
| `no-cache` | **Lưu** nhưng **phải revalidate** trước khi dùng (≠ no-store!) |
| `private` | Chỉ **browser** cache (không CDN) |
| `public` | CDN **được** cache |
> **Bẫy kinh điển:** lẫn `no-cache` (được lưu, phải revalidate) với `no-store` (cấm lưu). *(verify)*

**(b) `ETag` + `If-None-Match` (J-HTTP-002).** Server gắn **ETag** (fingerprint nội dung). Client gửi lại `If-None-Match: <etag>` → server **khớp** thì trả **`304 Not Modified`** (**không gửi body**) → tiết kiệm **băng thông**. **Revalidation** giữ tươi mà vẫn cache. (Tương tự `Last-Modified` + `If-Modified-Since`.) Lợi hơn chỉ `max-age`: tươi hơn (kiểm lại nội dung) mà vẫn rẻ (304 không body). *(verify)*

**(c) CDN vs app cache (J-HTTP-003).** **CDN** cache ở **edge gần user** → cắt **latency địa lý** + **offload origin** cho static/asset/response cacheable. **Redis** cache ở data center cho **dynamic/shared state**. Đặt lên CDN: **static/ảnh/JS/CSS** + **response GET công khai**.

**(d) GraphQL/POST khó cache (J-HTTP-004).** HTTP/CDN cache theo **URL + method**; **GET** idempotent/cacheable. GraphQL thường **1 endpoint POST** với **body khác nhau** → **mất HTTP cache** → phải **cache normalized ở client** (Apollo/urql) hoặc **persisted query + GET**. (Cross-ref F GraphQL cache.)

**(e) Purge & `Vary` (J-HTTP-005).** **Purge** toàn cầu nhiều PoP có **độ trễ** + **tốn** → nên **versioned URL** (content hash `app.a1b2.js`) thay vì purge. **`Vary`** (theo header như `Accept-Encoding`/`Authorization`) **chia nhỏ cache key**: đặt sai → **miss nhiều** (Vary quá rộng) hoặc **rò cache giữa user** (thiếu Vary cho data riêng). *(verify)*

**(f) Cache response cá nhân hoá (J-HTTP-006) — ⚠️ nguy hiểm.** Cache **public** một response có **data riêng** → **rò cho user khác** (do `Cache-Control: private` bị đặt sai/bỏ qua, hoặc CDN cache nhầm). Đúng: cá nhân hoá nên **`private`/`no-store`**, hoặc **cache theo key gồm user**, hoặc **tách phần public** (cache) khỏi **phần riêng** (không cache / **ESI** edge-side includes).

## ③ ⚠️ Kiến thức cũ / bị thay thế

| Cái cũ | Thay bằng | Vì sao |
|---|---|---|
| Lẫn `no-cache` = không cache | `no-cache` = **lưu + revalidate**; `no-store` = **cấm lưu** | Nghĩa khác nhau hoàn toàn |
| Purge CDN mỗi lần đổi asset | **Versioned/content-hash URL** | Purge toàn cầu chậm/tốn |
| Cache GraphQL như REST GET ở CDN | Normalized client cache / persisted query + GET | 1 endpoint POST không cache HTTP được |
| Cache response user-specific là `public` | `private`/`no-store` hoặc key gồm user | Tránh rò data giữa user |
| Quên `Vary` cho data theo header | Đặt `Vary` đúng (vừa đủ) | Sai → miss tràn hoặc rò cache |

## ④ Áp dụng thực tế + So sánh bigtech
**Use case:** static asset → `Cache-Control: public, max-age=31536000, immutable` + versioned filename. API GET công khai → `s-maxage` ở CDN + ETag. Trang dashboard cá nhân → `private, no-store`. SPA bundle → content-hash để cache bust.

**Bigtech (pattern phổ biến):** Cloudflare/Akamai/CloudFront/Fastly ở edge; **immutable + versioned URL** là chuẩn cho asset; **stale-while-revalidate** (Bài 3) phổ biến để giữ latency thấp; GraphQL APIs dùng **persisted queries** để lấy lại HTTP/CDN cache. *(verify)*

## ⑤ Code thực hành + cấu hình
Express set header đúng:
```js
import express from "express";
const app = express();

// Static asset versioned → cache cực dài, immutable
app.use("/assets", express.static("public", {
  setHeaders: (res) => res.setHeader("Cache-Control", "public, max-age=31536000, immutable"),
}));

// API GET công khai: CDN cache 60s + revalidate bằng ETag (Express tự sinh ETag)
app.get("/api/products", (req, res) => {
  res.setHeader("Cache-Control", "public, s-maxage=60, stale-while-revalidate=30"); // verify SWR
  res.json(products);
});

// Dashboard cá nhân hoá: KHÔNG cache công khai
app.get("/api/me/dashboard", (req, res) => {
  res.setHeader("Cache-Control", "private, no-store"); // tránh rò data giữa user
  res.json(buildDashboard(req.user));
});
```
⚠️ Đừng để response có data user là `public`. ⚠️ `Vary: Authorization` nếu response đổi theo auth (hoặc tốt hơn: `private`). *(verify header tại MDN/RFC 9111)*

## ⑥ Keywords cần nhớ
- 🧠 **Cần HIỂU:** no-cache≠no-store; ETag/304 revalidation; CDN giải latency địa lý; GraphQL/POST mất HTTP cache; Vary chia cache key; rủi ro cache cá nhân hoá.
- 📌 **Cần THUỘC:** max-age (client) vs s-maxage (CDN); private vs public; versioned URL thay purge; private/no-store cho data riêng.
- 🛠️ **Cần LÀM ĐƯỢC:** set Cache-Control đúng cho static/public-API/private; dùng ETag; dùng content-hash bust; tránh rò cache.

## ⑦ Mental model
> **CDN/HTTP cache ở edge gần user, theo URL+method. Static → public+versioned+immutable. API GET công khai → s-maxage+ETag. Data riêng → private/no-store (đừng public!). no-cache = lưu+revalidate, KHÁC no-store.**

## ⑧ Câu hỏi phỏng vấn & thách đố
1. *(TB)* Phân biệt max-age / s-maxage / no-store / no-cache / private/public. → (bảng ②a).
2. *(khó)* ETag + If-None-Match hoạt động thế nào, lợi gì? → fingerprint→304 không body→tiết kiệm băng thông + revalidate.
3. *(TB)* CDN giải gì mà Redis không? → latency địa lý + offload origin cho static/public.
4. *(khó)* Vì sao GraphQL/POST khó cache HTTP hơn REST GET? → 1 endpoint POST body khác → mất HTTP cache.
5. *(khó)* Cache response cá nhân hoá nguy hiểm sao, làm đúng? → rò data; private/no-store/key gồm user/tách public-riêng.

**Thách đố:** *"Bạn đặt `Cache-Control: public, max-age=300` cho `/api/me/profile`. Vài user báo thấy thông tin của người khác. Vì sao?"* → Response **chứa data riêng** nhưng đánh **`public`** → **CDN/shared cache** lưu bản của user A và **trả cho user B** (cùng URL, không phân biệt auth). Sửa: **`private, no-store`** (hoặc tối thiểu `private` + `Vary: Authorization`). Đây đúng là **bẫy rò cache** J-HTTP-006 — và là lỗi correctness/security TL phải bắt.

## ⑨ Bài tập + tiêu chí tự chấm
- **BT1:** Set header đúng cho 3 endpoint: `/assets/app.<hash>.js`, `/api/news` (public), `/api/me/orders` (riêng). *Đạt khi:* asset immutable+versioned; news public+s-maxage+ETag; orders private/no-store.
- **BT2:** Giải thích cách cache một trang có phần public (tin tức) + phần riêng (tên user) mà không rò. *Đạt khi:* tách phần public cache + phần riêng không cache (ESI/client render), không đánh public cho cả trang.

## ⑩ Đọc thêm
- MDN — *HTTP caching*, *Cache-Control*, *ETag*.
- **RFC 9111** (HTTP Caching) — nguồn chuẩn nhất.
- web.dev — *Love your cache* (HTTP caching best practices).

> 🔎 **AI hay sai ở đây:** AI đặt `public, max-age` cho mọi response "cho nhanh", kể cả endpoint có data user → rò cache. Kiểm tra: *response này có data riêng user không?* Có → `private`/`no-store`.

---

# 🏁 SAU KHI HỌC XONG 11 BÀI — Bước D & E

## Bước D — Tự phỏng vấn (chấm LIVE)
Mở `QB_J_caching.md`, chạy theo đợt **10–15 câu** trộn dễ→khó. Quy tắc vàng: **đọc câu → DỪNG → tự trả lời ra giấy/miệng (Feynman) → mới đối chiếu** dòng *"dò cái gì"* của câu đó. Đạt = chạm đủ các ý trong "dò cái gì".

Gợi ý chia đợt:
- **Đợt 1 (nền):** J-WHY + J-STRAT (017 câu) — vì sao + strategy.
- **Đợt 2 (tươi):** J-INVAL + J-CONSIST (19 câu) — invalidation + consistency.
- **Đợt 3 (tải thật):** J-PROB + J-EVICT (18 câu) — 3 bệnh + eviction.
- **Đợt 4 (Redis):** J-REDIS + J-BEYOND + J-CLUSTER (21 câu).
- **Đợt 5 (mở rộng):** J-QUEUE + J-HTTP (13 câu).
- **Mỗi đợt chèn 1–2 câu CŨ đã đạt** (spaced repetition).

## Bước E — Cổng hoàn thành + ôn ngắt quãng
- **"Học xong mục J" = ĐẠT toàn bộ 88 câu.**
- Câu **chưa đạt** → quay lại đúng Bài phủ nó, vá chỗ yếu, hỏi lại **biến thể**.
- **Lịch ôn:** hôm nay → mai → 3 ngày → 1 tuần.

## 🎯 5 "bẫy AI" của mục J — thuộc lòng để KIỂM TRA
1. **Write-back cho tiền** → mất giao dịch khi cache chết (Bài 2).
2. **Cùng key cho mọi user** → rò data/quyền (Bài 3, 7).
3. **Update cache thay vì delete / sai thứ tự** → stale bền (Bài 4).
4. **Một Redis cho cache + session/queue, `allkeys-lru`** → evict nhầm session/job (Bài 6, 8, 10).
5. **`public, max-age` cho response cá nhân hoá** → rò cache giữa user ở CDN (Bài 11).

> **Câu thần chú:** *"Tôi không nhớ để gõ lệnh Redis — tôi hiểu để chỉ huy (chọn đúng strategy/policy/header) và kiểm tra (bắt 5 bẫy trên)."*

---

## ⏳ Ghi chú "verify" khi học (cú pháp/đời sản phẩm dễ đổi — đối chiếu 6/2026)
- **Redis 8.x**, tri-license **RSALv2/SSPLv1/AGPLv3**; **Valkey** (BSD, Linux Foundation) & **DragonflyDB** là lựa chọn thay thế. Lệnh tương thích phần lớn.
- **8 maxmemory-policy**, default **volatile-lru**; tinh chỉnh `maxmemory-samples` (default 5).
- **BullMQ 5.78.x** (6/2026), dựa **Redis Streams**, TypeScript-native.
- Lệnh Redis (`SET..EX/PX`, `SCAN`, `UNLINK`, `XADD/XREADGROUP/XACK`, zset, `PFADD/PFCOUNT`), cơ chế **Cluster (16384 slot)/Sentinel**, module **Redis Query Engine** → đối chiếu **redis.io**.
- HTTP cache (`Cache-Control`, `ETag`, `Vary`, `stale-while-revalidate`) → **MDN / RFC 9111**.

*Hết tài liệu mục J — Caching. Chúc học chắc & chỉ huy giỏi.*
