# 🧩 CHUYÊN ĐỀ — CACHE-ASIDE (Lazy Loading) trong Caching
> Đào sâu pattern nền tảng (C4) của **Bài 1 — Vì sao cache · Các tầng cache · Khi nào KHÔNG cache**
> Mục tiêu: làm chủ pattern caching phổ biến nhất — *chính là code trong Bài 1* — và mọi cạm bẫy của nó ở mức **staff engineer**.
> (✅ = đã có trong Bài 1 · ➕ = mở rộng thêm)

---

## ① Mục tiêu & vị trí trong mạch

Trong Bài 1, **Cache-aside** xuất hiện ở **bảng từ chuyên ngành mở rộng (C4)**:

> ➕ **Cache-aside (lazy loading)** — *App tự đọc cache trước, miss thì đọc DB rồi nạp lại* — *"Pattern phổ biến nhất: cache-aside (chính là code trong bài)."*

Đây **không phải một keyword phụ** — nó là **xương sống** của Bài 1: đoạn code `getProduct` multi-tier (L1+L2) ở mục (e) **chính là** cache-aside. Và nó là **nơi mọi vấn đề** của tám chuyên đề trước hội tụ thành code thật:
- **Eventual Consistency** — stale-set race (Ví dụ 4 của chuyên đề đó *là* race của cache-aside).
- **Cold Start** — mỗi key lần đầu đều miss; hot key miss → stampede.
- **Cache Coherence** — TTL backstop, invalidate, CDC.
- **Idempotency** — back-fill phải idempotent.
- **Working Set** — chỉ cache cái được hỏi (lazy = tự nhiên theo working set).

Chuyên đề này đưa cache-aside ra sân khấu chính. Học xong bạn sẽ:

- Vẽ được **luồng đọc & ghi** của cache-aside, hiểu vì sao gọi là "lazy".
- Phân biệt nó với cả **họ pattern**: read-through, write-through, write-back, write-around.
- Biết vì sao nó **phổ biến nhất** (đơn giản, dùng cache "ngu", sống sót khi cache chết) — và **cạm bẫy** của nó.
- Nắm bộ giải pháp staff-level: **write→delete đúng thứ tự, versioning/lease, singleflight, TTL backstop, CDC, negative caching**.

> 💡 **Một câu để gắn vào mạch Bài 1:** Cache-aside là pattern mà **ứng dụng tự cầm trịch** — cache chỉ là một kho key-value "ngu", app lo việc đọc-trước, miss-thì-nạp. Nó "lazy" vì **chỉ nạp cái được hỏi**. Làm chủ cache-aside = làm chủ toàn bộ cụm chủ đề caching, vì mọi vấn đề khó đều **biểu hiện cụ thể ở chính pattern này**.

---

## ② Trực giác — Cache-aside là gì?

### Định nghĩa

**Cache-aside (còn gọi look-aside, lazy loading) = ỨNG DỤNG tự quản lý cache.** Cache nằm "bên cạnh" (aside) đường dữ liệu chính; app điều phối:

- **Đọc:** hỏi cache trước → **hit** thì trả; **miss** thì app **tự đọc nguồn (DB), tự nạp vào cache**, rồi trả.
- **"Lazy"** vì dữ liệu **chỉ được nạp vào cache khi có người hỏi** (on-demand), không nạp sẵn trước.

### Luồng đọc (read path) — phải thuộc

```
        ┌─────────┐   1. GET key    ┌─────────┐
        │   App   │ ──────────────► │  Cache  │
        │         │ ◄────────────── │ (Redis) │
        └─────────┘   2. hit/miss   └─────────┘
             │
   miss ─────┤ 3. đọc DB
             ▼
        ┌─────────┐
        │   DB    │  4. trả value
        └─────────┘
             │
             ▼
        5. App NẠP value vào Cache (back-fill)  →  6. trả cho caller
```

- **Hit:** 1 chặng (chỉ cache). Nhanh.
- **Miss:** 3 chặng (hỏi cache → đọc DB → ghi cache). Đây là **"miss penalty"**.

### Ẩn dụ thủ thư tự phục vụ

Bạn (app) cần một cuốn sách. Bạn **tự** ra giá sách gần (cache) xem có không. Có → lấy luôn. Không → bạn **tự** đi xuống kho (DB) lấy, rồi **tự** đặt một bản lên giá gần cho lần sau. **Cái giá sách không biết gì cả — bạn làm hết.** Đó là cache-aside: *cache "ngu", app thông minh*.

> 🔑 **Insight #1:** Điểm cốt lõi: trong cache-aside, **logic nằm ở app, không ở cache**. Cache chỉ là kho key-value. Điều này làm nó (a) **dùng được với bất kỳ cache nào** (Redis/Memcached đều là kho ngu), và (b) **app vẫn chạy khi cache chết** (chỉ cần fallback đọc DB). Đây là lý do nó phổ biến nhất.

---

## ③ Luồng GHI — phần khó & gây bug nhất

Cache-aside **không quy định** cách ghi, nhưng cách ghi *đi kèm* mới là chỗ sinh bug. Hai lựa chọn:

### Lựa chọn A — Ghi DB → XOÁ cache key (invalidate) ✅ khuyến nghị

```
1. UPDATE DB (ghi nguồn)
2. DELETE cache key (xoá, KHÔNG cập nhật)
→ Lần đọc sau: miss → nạp lại bản mới từ DB
```

### Lựa chọn B — Ghi DB → CẬP NHẬT cache (set value mới) ⚠️ nhiều vấn đề

```
1. UPDATE DB
2. SET cache key = value mới
```

### Vì sao **XOÁ tốt hơn CẬP NHẬT**?

| Lý do | Giải thích |
|---|---|
| **Tránh cache đồ không ai đọc** | Cập nhật = nhồi value mới dù có thể chẳng ai hỏi lại (phí RAM, ngược working set) |
| **Giảm stale-set race** | Cập nhật bằng value tính ở app dễ bị ghi đè bởi value cũ trong race; xoá thì lần sau nạp tươi từ DB |
| **Idempotent & đơn giản** | DELETE lặp lại vô hại (Idempotency); SET value cần đúng value + version |
| **Khớp "lazy"** | Lazy = chỉ nạp khi hỏi; xoá giữ đúng tinh thần đó |

> 🔑 **Insight #2:** **Quy tắc vàng cache-aside khi ghi: "Ghi DB trước, XOÁ cache key sau" (không cập nhật).** Đây là mặc định an toàn. (Nhưng vẫn còn stale-set race tinh vi — mục ⑥ & ⑧.)

### ⚠️ Thứ tự cũng quan trọng

```
❌ XOÁ cache → rồi GHI DB:  khe hở giữa hai bước, một đọc-miss có thể nạp lại ĐỒ CŨ từ DB.
✅ GHI DB → rồi XOÁ cache:  an toàn hơn (nhưng vẫn cần versioning cho race hiếm — mục ⑧).
```

---

## ④ Cache-aside trong HỌ pattern (so sánh đầy đủ)

Cache-aside là một trong **năm pattern** caching cốt lõi. Phân biệt rạch ròi:

| Pattern | Đọc nguồn lúc miss | Ghi nguồn | Đặc trưng |
|---|---|---|---|
| **Cache-aside** (look-aside) | **App** tự đọc | App ghi DB, rồi xoá/cập nhật cache | App điều phối; cache "ngu"; sống khi cache chết |
| **Read-through** | **Cache** tự đọc (qua loader) | — | Logic load nằm trong cache/thư viện; app chỉ hỏi cache |
| **Write-through** | — | Ghi qua cache → DB **đồng bộ** | Cache luôn tươi; ghi chậm hơn |
| **Write-back** (write-behind) | — | Ghi cache → DB **bất đồng bộ** (sau) | Ghi cực nhanh; **rủi ro mất data** nếu cache chết |
| **Write-around** | App tự đọc | Ghi **thẳng DB**, bỏ qua cache | Không nạp cache lúc ghi; hợp data ghi-một-lần-ít-đọc |

> 🔑 **Insight #3 — read-through vs cache-aside (cặp hay bị hỏi):**
> - **Cache-aside:** app cầm logic → **kiểm soát tối đa**, nhưng **logic rải khắp** mọi chỗ đọc (dễ trùng lặp/sai sót), và app **chịu trách nhiệm** khi cache miss/chết.
> - **Read-through:** cache (hoặc thư viện như Caffeine `LoadingCache`) **giấu** logic load → **gọn, tập trung một chỗ**, nhưng cần cache **hỗ trợ** và app **mất kiểm soát chi tiết**.
> Thực tế: cache-aside phổ biến hơn vì Redis/Memcached là kho ngu; read-through hay thấy ở **in-process cache library**.

> 🔑 **Insight #4 — kết hợp đọc & ghi:** Cache-aside (đọc) thường **ghép với write-around hoặc invalidate-on-write** (ghi). Một số hệ ghép **cache-aside đọc + write-through ghi** để cache luôn tươi. Bạn **chọn pattern đọc và pattern ghi độc lập**, miễn nhất quán.

---

## ⑤ Vì sao cache-aside PHỔ BIẾN NHẤT — và nhược điểm

### Ưu điểm

1. **Dùng được với cache "ngu"** (Redis/Memcached) — không cần tích hợp đặc biệt.
2. **Sống sót khi cache chết** — cache down → app vẫn đọc DB (degrade, không sập). Resilient.
3. **Lazy = tự khớp working set** — chỉ cache cái được hỏi; data lạnh không tốn RAM (nối Working Set).
4. **Kiểm soát tối đa** — app quyết cache cái gì, TTL bao lâu, key ra sao.

### Nhược điểm (toàn những thứ ta đã học!)

| Nhược điểm | Chuyên đề liên quan |
|---|---|
| **Stale-set race** (cache kẹt đồ cũ) | Eventual Consistency |
| **Miss penalty** (3 chặng lúc miss) | — |
| **Cold start** (mọi key lần đầu miss) | Cold Start |
| **Stampede** (hot key miss → đập nguồn đồng loạt) | Cold Start |
| **Logic rải khắp** (mỗi chỗ đọc tự cài pattern) | (read-through giải) |
| **Cửa sổ stale theo TTL** | Eventual Consistency, Cache Coherence |
| **Invalidate có thể sót** → drift | Cache Coherence (CDC giải) |

> 🔑 **Insight #5:** Cache-aside **đơn giản để bắt đầu, khó để làm đúng ở quy mô lớn.** Toàn bộ tám chuyên đề trước thực ra là **"cách vá các nhược điểm của cache-aside"**. Đó là lý do nó là pattern trung tâm.

---

## ⑥ Ví dụ minh hoạ

### Ví dụ 1 — Cache-aside cơ bản (một tầng)

```js
export async function getUser(id, db) {
  const key = `user:${id}`;
  // 1) Hỏi cache
  const cached = await redis.get(key);
  if (cached !== null) return JSON.parse(cached);   // HIT

  // 2) MISS → đọc DB
  const user = await db.findUser(id);

  // 3) Back-fill cache (lazy load) + TTL backstop
  if (user != null) {
    await redis.set(key, JSON.stringify(user), "EX", 300);
  }
  return user;
}

// Ghi: GHI DB → XOÁ cache (Lựa chọn A)
export async function updateUser(id, data, db) {
  await db.updateUser(id, data);     // 1) ghi nguồn
  await redis.del(`user:${id}`);     // 2) xoá cache (không cập nhật)
}
```

### Ví dụ 2 — Cache-aside multi-tier (chính là code Bài 1)

Đoạn `getProduct` L1+L2 ở Bài 1 mục (e) **là cache-aside hai tầng**: hỏi L1 → miss hỏi L2 → miss đọc DB → back-fill ngược lên L2 rồi L1. Cùng pattern, chỉ thêm tầng.

### Ví dụ 3 — Stale-set race (cạm bẫy kinh điển, nối Eventual Consistency)

```
Luồng A (đọc):                         Luồng B (ghi):
1. GET cache → MISS
2. đọc DB → v1 (cũ)
                                        3. UPDATE DB → v2
                                        4. DELETE cache
5. SET cache = v1  ← GHI ĐÈ ĐỒ CŨ!
   → cache kẹt v1 tới khi TTL hết
```
> Đây là lý do **dù làm đúng "ghi DB→xoá cache"**, vẫn còn khe hở. Vá ở mục ⑧ (versioning/lease/double-delete). **TTL backstop** (Ví dụ 1) là lưới an toàn cuối: dù kẹt, cũng chỉ kẹt tối đa = TTL.

### Ví dụ 4 — Cache-aside chống stampede + race (bản "đủ giáp", gộp cả cụm)

```js
import pLimit from "p-limit";
const dbLimit = pLimit(50);            // chặn concurrency xuống DB (Cold Start)
const inflight = new Map();            // singleflight (Cold Start)
const jitter = b => b + Math.floor(b * (Math.random()*0.2 - 0.1)); // chống synchronized expiry

export async function getProduct(id, db) {
  const key = `product:${id}`;
  const cached = await redis.get(key);
  if (cached !== null) return JSON.parse(cached);

  if (inflight.has(key)) return inflight.get(key);   // gộp miss trùng key
  const p = dbLimit(() => db.findProduct(id))
    .then(async (val) => {
      if (val != null) {
        // SET kèm version để chống stale-set race (mục ⑧)
        await redis.set(key, JSON.stringify(val), "EX", jitter(300));
      }
      return val;
    })
    .finally(() => inflight.delete(key));
  inflight.set(key, p);
  return p;
}
```
> Một hàm gói gọn: **lazy load + singleflight + jitter + TTL** — cache-aside "trưởng thành".

---

## ⑦ Case study thực tế

> ⚠️ Tóm tắt theo tài liệu/bài báo công khai và pattern phổ biến. Verify số liệu cụ thể khi cần.

### CS1 — Facebook Memcached: "demand-filled look-aside cache" (NSDI 2013)

Facebook mô tả cache của họ đúng là **cache-aside** ("demand-filled look-aside"): app đọc Memcached, miss thì đọc DB rồi nạp lại. Họ gặp **đúng** hai vấn đề ta liệt kê — **stale set** và **thundering herd** — và giải bằng **leases** (token chỉ-cho-fill-hợp-lệ + gộp miss). Đây là cache-aside ở quy mô khổng lồ + bản vá chuẩn mực.

> 🎯 **Bài học:** Cache-aside là pattern *mặc định* kể cả ở bigtech; cái phân biệt là **bộ vá** (lease/version/singleflight) cho race & stampede.

### CS2 — AWS ElastiCache: "Lazy Loading" là chiến lược khuyến nghị đầu tiên

Tài liệu AWS gọi cache-aside là **"lazy loading"** và đặt nó làm chiến lược nền tảng, thường **ghép với TTL** và (tuỳ nhu cầu) **write-through**. Họ nêu đúng ưu điểm (chỉ cache cái được hỏi, resilient) và nhược (cache miss penalty, dữ liệu có thể stale → dùng TTL).

> 🎯 **Bài học:** Khuyến nghị ngành = **cache-aside + TTL backstop** làm mặc định, thêm write-through nếu cần tươi hơn.

### CS3 — Web app điển hình + Redis

Tuyệt đại đa số web app dùng **cache-aside với Redis**: `get → miss → query DB → set`. Đơn giản, không khoá vào vendor, cache chết vẫn chạy. Đây là vì sao nó là pattern bạn gặp đầu tiên và nhiều nhất.

### CS4 — Netflix EVCache

EVCache (trên Memcached) phục vụ các pattern cache-aside ở quy mô streaming, kèm replication/đa vùng. Vẫn là look-aside ở cốt lõi, gia cố cho HA.

---

## ⑧ Giải pháp staff engineer — cache-aside "đúng"

Toàn bộ là **vá các nhược điểm ở ⑤** (mỗi cái nối một chuyên đề):

### G1. Write path kỷ luật (mục ③)
**Ghi DB → XOÁ cache key** (không cập nhật), **đúng thứ tự** (DB trước). Mặc định an toàn nhất.

### G2. Vá stale-set race (nối Eventual Consistency)
- **Versioning/CAS:** SET cache chỉ khi version ≥ version đang có → fill mang v1 cũ bị từ chối.
- **Leases (Facebook):** token chỉ-cho-fill-hợp-lệ.
- **Delayed double-delete:** xoá cache → ghi DB → đợi chút → **xoá lần nữa** (dọn bản lỡ stale).

### G3. TTL backstop LUÔN bật (nối Cache Coherence)
Kể cả khi có invalidate, **luôn đặt TTL** → nếu invalidate bị sót/race, cache chỉ stale tối đa = TTL. **Lưới an toàn không thể thiếu.**

### G4. Chống stampede trên hot key (nối Cold Start)
- **Singleflight/coalescing:** miss trùng key → 1 lần đi nguồn.
- **Jitter TTL:** chống synchronized expiry.
- **Probabilistic early expiration (XFetch):** làm mới hot key *trước* khi hết hạn.

### G5. Negative caching
Cache cả kết quả **"không tồn tại"** (TTL ngắn) → tránh miss lặp lại đập DB cho key rác/không có.

### G6. Back-fill idempotent (nối Idempotency)
Nhiều request cùng back-fill một key → vô hại (ghi gán-tuyệt-đối cùng giá trị). Đảm bảo đúng đắn dù lặp.

### G7. Tập trung pattern (giảm logic rải khắp)
Bọc cache-aside vào **một helper/decorator** (`cached(key, ttl, loader)`) thay vì copy-paste `get/miss/set` ở mọi chỗ đọc → giảm bug, dễ thêm singleflight/metric một chỗ.

---

## ⑨ Giải pháp NÂNG CAO (staff / principal)

### A1. CDC-based invalidation thay vì app tự xoá (nối Cache Coherence)
App tự `del()` ở mọi đường ghi **dễ sót** (một service ghi DB mà quên xoá cache → drift). Nâng cấp: **bắt binlog (CDC) → phát invalidate** → không bao giờ sót, kể cả ghi từ nhiều nguồn. Consumer idempotent (xoá hai lần vô hại).

### A2. Refresh-ahead cho hot key
Chủ động **làm mới key nóng trước khi hết hạn** (dựa trên dự đoán key sẽ còn được hỏi) → hot key gần như **không bao giờ miss** → cắt cả miss penalty lẫn stampede. Đổi lại tốn công refresh thứ có thể không cần.

### A3. Cache-aside + write-through lai
Đọc cache-aside (lazy), nhưng **ghi write-through** (ghi cache + DB đồng bộ) cho path **đọc-ngay-sau-ghi** → cache luôn tươi cho path đó, không cần chờ miss nạp lại. Chọn lai theo từng loại dữ liệu.

### A4. Phân tầng nhất quán theo path
Không phải mọi key cùng một TTL/cơ chế. Path tiền/quyền: TTL cực ngắn + CDC + versioning (gần strong). Path đếm/feed: TTL dài, chấp nhận eventual. **Cache-aside cho phép vặn từng path** — tận dụng điều đó.

### A5. Đo trước khi cache-aside (nối Amdahl)
Trước khi bọc một đường đọc bằng cache-aside, **đo** xem DB/đường đọc đó có phải **nút thắt** không. Cache-aside thêm độ phức tạp (race, stampede, TTL) — chỉ đáng nếu đọc thật sự thống trị. Đừng cache-aside mọi thứ "cho nhanh".

### A6. Quyết định "cache-aside hay denormalize/MV" (nối Denormalization)
Nếu một aggregate/JOIN đọc **cực nhiều** và cần **luôn-sẵn/đúng-trong-transaction**, **denormalized column hoặc MV** có thể tốt hơn cache-aside (eager, strong, bền). Cache-aside hợp **đọc thưa hơn, chịu miss, không động schema**. Chọn đúng công cụ trong họ "dữ liệu dẫn xuất".

---

## ⑩ Trường hợp MỞ RỘNG (look-aside ở mọi tầng)

| Bối cảnh | Cache-aside thể hiện thế nào |
|---|---|
| **App + Redis/Memcached** (điển hình) | get → miss → DB → set (chính chủ) |
| **Multi-tier L1+L2** (Bài 1) | Cache-aside lồng nhau qua các tầng |
| **HTTP/browser** | Trình duyệt/CDN hỏi cache, miss → fetch origin → lưu (look-aside, có ETag) |
| **DNS resolver** | Hỏi cache local, miss → query upstream → lưu theo TTL |
| **ORM second-level cache** | Đôi khi read-through, đôi khi cache-aside |
| **Application memoization** | Cache-aside in-process (memo map, miss → tính → lưu) |
| **DB query cache** | Look-aside cho kết quả query |
| **CPU cache** | **KHÁC** — phần cứng làm trong suốt (giống read-through), không phải app-managed |

> 🔑 **Insight #6:** Cùng một vũ điệu "hỏi cache → miss thì đi nguồn → nạp lại" lặp ở mọi tầng. Điểm phân biệt: **AI điều phối** — app (cache-aside) hay cache/phần cứng (read-through). CPU cache là read-through (trong suốt); app cache thường là cache-aside (app cầm trịch).

---

## ⑪ Kiến thức MỞ RỘNG (kết nối cả cụm — 10 tài liệu)

- ✅ **Code Bài 1 (e):** `getProduct` multi-tier *chính là* cache-aside. Đây là pattern gốc.
- ✅ **Back-fill (B2 Bài 1):** bước nạp-ngược của cache-aside; phải idempotent.
- ➕ **Eventual Consistency:** stale-set race của cache-aside; TTL = bounded staleness.
- ➕ **Cold Start:** mọi key lần đầu miss; hot key miss → stampede → singleflight/XFetch.
- ➕ **Cache Coherence:** TTL backstop + invalidate; CDC để không sót.
- ➕ **Idempotency:** back-fill & invalidate idempotent → retry/at-least-once an toàn.
- ➕ **Working Set:** lazy load chỉ cache cái được hỏi → tự khớp working set.
- ➕ **Amdahl:** đo trước khi cache-aside một đường đọc.
- ➕ **Denormalization:** cache-aside (lazy, ngoài DB) vs denormalized/MV (eager) — chọn theo nhu cầu.
- ➕ **Họ pattern:** read-through/write-through/write-back/write-around — bản đồ đầy đủ.

---

## ⑫ Anti-patterns — AI & người mới hay sai

| ❌ Sai lầm | Vì sao nguy hiểm | ✅ Đúng |
|---|---|---|
| **Cập nhật** cache lúc ghi (thay vì xoá) | Cache đồ không ai đọc + stale-set race | **Ghi DB → XOÁ key** (Lựa chọn A) |
| **Xoá cache trước, ghi DB sau** | Khe hở → đọc nạp lại đồ cũ | **Ghi DB trước, xoá sau** |
| **Không đặt TTL** (chỉ dựa invalidate) | Invalidate sót → stale **vĩnh viễn** | **TTL backstop** luôn bật |
| Không chống **stampede** hot key | Hot key miss → đập nguồn đồng loạt | Singleflight + jitter + XFetch |
| **Logic cache-aside rải khắp** app | Mỗi chỗ một kiểu, dễ bug | Bọc **một helper** `cached(key,ttl,loader)` |
| Bỏ qua **stale-set race** | Cache kẹt đồ cũ trong concurrency | Versioning/CAS / lease / double-delete |
| App tự `del()` ở mọi nơi | Sót một đường ghi → drift | **CDC invalidation** |
| **Cache-aside mọi thứ** "cho nhanh" | Tối ưu phần không-nút-thắt (Amdahl) | Đo trước; cache đường đọc thống trị |
| Cache-aside cho **aggregate cần strong** | Eventual + race | Cân nhắc **denormalize/MV** (eager, strong) |

> 🔎 **AI hay sai (nối Bài 1):** AI viết cache-aside thường **cập nhật cache lúc ghi, quên TTL, không chống stampede, không xử race**. Luôn ép kiểm: *"Ghi thì xoá hay cập nhật? Thứ tự nào? Có TTL backstop chưa? Hot key có singleflight chưa? Stale-set race xử thế nào?"*

---

## ⑬ Khung quyết định / Playbook cache-aside

### D-CA-1 — Có nên dùng cache-aside cho đường đọc này?
```
1. Đường đọc này có phải NÚT THẮT không? (Amdahl — đo p99/DB QPS)
   → Không → đừng thêm cache-aside (đừng thêm độ phức tạp vô ích).
2. Đọc ≫ ghi? Có locality (key được hỏi lại)? → hợp cache-aside.
3. Cần luôn-sẵn/strong/aggregate nặng? → cân nhắc denormalize/MV thay vì cache-aside.
```

### D-CA-2 — Checklist triển khai cache-aside đúng
```
□ Đọc: get → miss → load DB → back-fill (idempotent) → return.
□ Ghi: GHI DB trước → XOÁ key sau (không cập nhật).
□ TTL backstop LUÔN bật (lưới an toàn cho invalidate sót/race).
□ Hot key: singleflight + jitter (+ XFetch nếu cần).
□ Stale-set race: versioning/CAS hoặc lease hoặc delayed double-delete.
□ Negative caching cho key không tồn tại (TTL ngắn).
□ Invalidate đáng tin: CDC nếu nhiều đường ghi.
□ Bọc pattern vào MỘT helper (đừng rải logic).
```

### D-CA-3 — Khi cache CHẾT (resilience)
```
□ Miss/lỗi cache → fallback đọc DB (đừng để cache-down = app-down).
□ Nhưng coi chừng: cache chết = cold start = stampede → bật bounded concurrency + shedding.
```

---

## ⑭ Mental model (câu thần chú gốc)

> 🧩 **Cache-aside = APP cầm trịch:** hỏi cache → miss thì tự đọc nguồn → tự nạp lại. "Lazy" vì chỉ nạp cái được hỏi. Cache chỉ là kho "ngu".
>
> Phổ biến nhất vì: **dùng cache nào cũng được, sống khi cache chết, tự khớp working set.**
> Nhưng đơn-giản-để-bắt-đầu, khó-để-làm-đúng: **toàn bộ cụm chủ đề caching là cách vá nhược điểm của nó.**
>
> Năm điều phải nhớ:
> 1. **Ghi DB trước, XOÁ key sau** (không cập nhật).
> 2. **TTL backstop luôn bật** (lưới an toàn).
> 3. **Hot key → singleflight + jitter** (chống stampede).
> 4. **Race → versioning/lease** (chống stale-set).
> 5. **Đo trước** (đừng cache-aside cái không phải nút thắt).

---

## ⑮ Câu hỏi phỏng vấn & thách đố

- **(dễ)** Cache-aside là gì, vì sao "lazy"? → App hỏi cache trước, miss thì tự đọc DB rồi nạp lại; lazy vì chỉ nạp cái được hỏi (on-demand).
- **(dễ)** Lúc miss tốn mấy chặng? → 3 (hỏi cache → đọc DB → ghi cache) = miss penalty.
- **(TB)** Cache-aside vs read-through? → Cache-aside: app cầm logic load (kiểm soát, nhưng rải khắp, sống khi cache chết). Read-through: cache/thư viện tự load (gọn, tập trung, nhưng cần cache hỗ trợ).
- **(TB)** Khi ghi nên xoá hay cập nhật cache? Vì sao? → **Xoá** — tránh cache đồ không ai đọc, giảm stale-set race, idempotent, hợp lazy.
- **(bẫy)** "Ghi DB→xoá cache" đã đủ nhất quán chưa? → Chưa — còn **stale-set race** (đọc-miss nạp đồ cũ sau khi xoá). Cần versioning/lease/double-delete + TTL backstop.
- **(TB)** Vì sao cache-aside phổ biến nhất? → Dùng cache ngu (Redis), resilient (cache chết vẫn đọc DB), lazy khớp working set, kiểm soát tối đa.
- **(khó)** Hot key vừa hết hạn, 10k request cùng miss — chuyện gì & xử sao? → **Stampede/thundering herd**; xử: singleflight/coalescing, jitter TTL, XFetch (làm mới trước hết hạn), lease.
- **(khó)** Khi nào KHÔNG dùng cache-aside mà chọn denormalize/MV? → Aggregate/JOIN đọc cực nhiều cần luôn-sẵn/strong/trong-transaction → denormalize/MV (eager) hợp hơn cache-aside (lazy, eventual).

> 🧩 **Thách đố:** *"Hệ dùng cache-aside (ghi DB→xoá cache, TTL 5 phút). Thỉnh thoảng user thấy dữ liệu cũ DÙ đã sửa, kéo dài tận 5 phút. Phân tích đầy đủ."*
> → **Stale-set race**: một đọc-miss đọc DB ra bản cũ *trước* khi ghi commit, rồi **SET cache = bản cũ** *sau* khi ghi đã xoá cache → cache kẹt bản cũ tới khi TTL (5 phút) hết. Sửa theo lớp: (1) **versioning/CAS** — SET chỉ khi version mới hơn → fill cũ bị từ chối; (2) hoặc **lease** (Facebook); (3) hoặc **delayed double-delete**; (4) **hạ TTL** để giảm cửa sổ tệ nhất; (5) **CDC invalidation** nếu nghi ghi từ đường khác không xoá cache. Lưu ý: TTL 5 phút chính là *lưới an toàn* đã giới hạn thiệt hại ở 5 phút — không có nó thì stale **vĩnh viễn**.

---

## ⑯ Bài tập + tiêu chí tự chấm

**BT1.** Viết hàm `getOrder` cache-aside đầy đủ: read path + write path (ghi→xoá), TTL backstop, negative caching.
> ✅ **Đạt khi:** get→miss→DB→set(TTL); update→ghi DB→del key; cache "not found" TTL ngắn; thứ tự ghi đúng.

**BT2.** Vẽ timeline stale-set race và chỉ ra **chính xác** versioning chặn ở bước nào.
> ✅ **Đạt khi:** chỉ đúng bước SET bản cũ bị từ chối vì version < version hiện có.

**BT3.** So sánh cache-aside vs read-through vs write-through cho: (a) catalog đọc nhiều, (b) số dư ví cần tươi sau ghi. Chọn pattern + lý do.
> ✅ **Đạt khi:** catalog → cache-aside (lazy) + TTL; số dư → cân nhắc write-through/denormalize đồng bộ để tươi sau ghi; giải thích đánh đổi.

**BT4.** Liệt kê 5 nhược điểm của cache-aside và bản vá chuẩn cho mỗi cái (gắn chuyên đề).
> ✅ **Đạt khi:** stale-set→versioning/lease; cold start/stampede→singleflight/jitter; sót invalidate→CDC; logic rải→helper; miss penalty/cold→warming/TTL.

---

## ⑰ Glossary (đóng vào hoàn cảnh)

| Thuật ngữ | Nghĩa nhanh | 🎬 Hoàn cảnh sử dụng |
|---|---|---|
| **Cache-aside / look-aside** | App tự quản cache: hỏi trước, miss thì nạp | *"Pattern phổ biến nhất, chính là code Bài 1."* |
| **Lazy loading** | Chỉ nạp cache khi có người hỏi | *"Lazy nên chỉ cache cái được đọc."* |
| **Back-fill** | Nạp ngược kết quả vào cache sau miss | *"Miss DB → back-fill lên cache."* |
| **Miss penalty** | 3 chặng lúc miss (cache→DB→cache) | *"Miss penalty cao thì cần warming."* |
| **Read-through** | Cache tự load nguồn lúc miss | *"Read-through giấu logic load trong cache."* |
| **Write-through** | Ghi qua cache→DB đồng bộ (cache luôn tươi) | *"Write-through cho path đọc-ngay-sau-ghi."* |
| **Write-back** | Ghi cache trước, đẩy DB sau (async) | *"Write-back nhanh nhưng rủi ro mất data."* |
| **Write-around** | Ghi thẳng DB, bỏ qua cache | *"Write-around cho data ghi-một-lần-ít-đọc."* |
| **Invalidate-on-write** | Ghi DB → xoá cache key | *"Cache-aside ghi: DB trước, xoá key sau."* |
| **Stale-set race** | Đọc-miss ghi đè cache bằng đồ cũ | *"Vá bằng versioning/lease/double-delete."* |
| **TTL backstop** | TTL làm lưới an toàn cho invalidate sót | *"Luôn đặt TTL dù đã invalidate."* |
| **Singleflight** | Gộp miss trùng key thành 1 lần đi nguồn | *"Hot key miss → singleflight chống stampede."* |
| **Negative caching** | Cache cả "không tồn tại" | *"Negative cache để khỏi đập DB tìm key rác."* |
| **Refresh-ahead** | Làm mới hot key trước khi hết hạn | *"Refresh-ahead để hot key không bao giờ miss."* |

---

## ⑱ Đọc thêm

- **Scaling Memcached at Facebook (NSDI 2013)** — demand-filled look-aside cache, leases (stale set + thundering herd).
- **AWS — "Caching Strategies" / ElastiCache docs** — Lazy Loading (cache-aside), Write-Through, TTL.
- **Microsoft — "Cache-Aside Pattern" (Azure Architecture Center)** — mô tả pattern chuẩn.
- **Martin Kleppmann, *Designing Data-Intensive Applications*** — derived data, invalidation, races.
- **Caffeine (Java) docs** — LoadingCache (read-through) để đối chiếu với cache-aside.
- **Bài 1 mục (e) & C4** — code multi-tier cache-aside + họ pattern (cross-ref nội bộ cụm).

---

> 🔗 **Nối lại cả cụm (10 tài liệu — khép trọn vẹn):** Cache-aside là **pattern trung tâm** — *chính là code của Bài 1* và là nơi mọi vấn đề của cụm **biểu hiện cụ thể thành code**. Stale-set race → **Eventual Consistency**; mọi-key-miss-lần-đầu & stampede → **Cold Start**; TTL backstop & invalidate & CDC → **Cache Coherence**; back-fill an toàn → **Idempotency**; lazy-chỉ-cache-cái-được-hỏi → **Working Set**; đo-trước-khi-cache → **Amdahl**; cache-aside-hay-denormalize → **Denormalization**; và lựa chọn AP/eventual → **CAP**. Câu chốt của cả cụm: ***cache-aside đơn giản để viết trong năm phút, nhưng viết ĐÚNG nó đòi hỏi cả chín bài học kia — vì mỗi nhược điểm của nó chính là một chuyên đề bạn vừa học.***
