# 🐘 CACHE STAMPEDE / THUNDERING HERD — Chuyên đề mở rộng (Caching · nối tiếp Bài 1)
> **Nhiều request cùng MISS một key → đồng loạt đập nguồn → sập** · Keyword · Concept · Giải pháp · Case study · Staff-level
> (Đọc sau khi đã nắm: *hot key · TTL · stale · multi-tier · jitter · singleflight · cold start · eviction storm*)

---

## ① ĐỊNH VỊ — Stampede là "khoảnh khắc cache phản bội bạn"

Bài 1 dạy: cache sinh ra để **giữ DB QPS xuống, kéo p99 xuống** (gắn cache vào *thứ cần bảo vệ*). **Cache stampede chính là khoảnh khắc sự bảo vệ đó sụp đổ** — và sụp đúng vào lúc tệ nhất.

**Định nghĩa:** **Nhiều request đồng thời cùng hỏi MỘT key, cùng gặp MISS một lúc → tất cả cùng lao xuống nguồn (DB/service) để tính lại CÙNG một thứ → nguồn quá tải.**

Các tên gọi (cùng một hiện tượng, khác sắc thái):
- **Thundering herd** — "đàn thú giẫm đạp": cả đàn cùng xông vào một lúc.
- **Dog-piling** — "chồng chó lên nhau": chồng chất request lên cùng một phép tính.
- **Cache stampede** — tên phổ biến nhất trong ngữ cảnh cache.

> 🔑 **Nghịch lý cốt tử:** **Key càng HOT thì stampede càng KINH KHỦNG.** Key nóng = nhiều request đồng thời = khi nó miss, **N request cùng đập nguồn**. Tức là *đúng cái key đáng được cache bảo vệ nhất lại gây ra cú sập lớn nhất khi cache hụt*.

🎬 *Câu thần chú:* **"Một key, một thời điểm, ngàn kẻ cùng miss → ngàn cú đập DB giống hệt nhau."**

---

## ② TẠI SAO NGUY HIỂM — Vòng xoáy tử thần (metastable failure)

Stampede không chỉ là một spike rồi thôi. Nó có thể **tự khuếch đại thành sập dây chuyền**:

```
Hot key MISS đồng loạt
   → N query giống hệt đập DB cùng lúc
   → DB chậm lại (quá tải)
   → request xếp hàng, timeout
   → client RETRY → thêm tải → DB chậm hơn nữa
   → ... vòng xoáy không tự thoát (METASTABLE FAILURE)
   → có khi cache đã phục hồi rồi mà hệ vẫn sập vì retry storm
```

> ⚠️ Đây là **metastable failure**: một cú kích (stampede) đẩy hệ vào trạng thái sập **tự duy trì** bởi retry, **không tự thoát ra** dù nguyên nhân gốc đã hết. Nên stampede không chỉ làm chậm — nó có thể **giết cả hệ thống**.

---

## ③ NGUYÊN NHÂN / NGÒI NỔ (phải nhận diện)

| Ngòi nổ | Mô tả | Nối với chuyên đề |
|---|---|---|
| **TTL hot key hết hạn đúng cao điểm** | Key nóng vừa hết hạn → ngàn request cùng miss | Kinh điển nhất |
| **TTL đồng bộ (synchronized expiry)** | Nhiều key tạo cùng lúc → TTL bằng nhau → **hết hạn cùng lúc** | Cache warming (warm cả loạt → set cùng TTL) |
| **Cold start / flush / deploy** | Cache rỗng → **rất nhiều key cùng miss** | Chuyên đề Cache warming |
| **Eviction storm** | Cache đầy đuổi hàng loạt → mất key đang nóng | Chuyên đề Eviction |
| **Cache node chết (L2 down)** | Cả không gian key biến mất → mọi node cùng đập DB | Bài 1: Redis là điểm phụ thuộc, cần HA |
| **Viral đột ngột** | Một key chưa từng nóng bỗng được hỏi dồn dập | Hot key mới nổi |
| **Invalidate rồi đọc dồn** | Vừa ghi → xoá key → đúng lúc đọc nhiều → miss đồng loạt | Bài 1: ghi phải invalidate |

🎬 *Phỏng vấn: "Stampede xảy ra khi nào?" → đọc đúng 7 ngòi nổ này, đừng chỉ nói "TTL hết hạn".*

---

## ④ ⭐ TAXONOMY STAFF-LEVEL — 3 kiểu sự cố cache hay bị gộp nhầm

Đây là khung phân loại **rất nhiều staff engineer dùng** (gốc từ cộng đồng SRE Trung Quốc, nay phổ biến toàn cầu). **Ba thứ khác nhau, thuốc chữa khác nhau:**

| Kiểu | Tên (TQ) | Bản chất | Thuốc chữa |
|---|---|---|---|
| **Cache breakdown** (đánh thủng) | 缓存击穿 | **MỘT hot key** hết hạn → stampede vào đúng key đó | **Lock/singleflight**, never-expire + refresh nền |
| **Cache penetration** (xuyên thủng) | 缓存穿透 | Hỏi key **KHÔNG TỒN TẠI** → luôn miss → luôn đập DB (có thể là tấn công) | **Negative caching**, **Bloom filter** |
| **Cache avalanche** (tuyết lở) | 缓存雪崩 | **NHIỀU key** hết hạn cùng lúc / **node cache chết** → miss diện rộng | **Jitter TTL**, **HA + multi-tier**, circuit breaker |

> 🔑 Phân biệt nhanh:
> - **Breakdown** = thủng **một điểm** (1 hot key). → coalesce/lock.
> - **Penetration** = thủng vì **rỗng thật** (key không có). → cache cả "không có" + Bloom filter.
> - **Avalanche** = **đổ cả mảng** (loạt key / node). → rải TTL + HA.

🎬 *Trả lời được "phân biệt击穿/穿透/雪崩" là tín hiệu staff rõ rệt.*

---

## ⑤ CÁC GIẢI PHÁP (phần thịt) — 4 nhóm tư duy

Mọi giải pháp stampede quy về **4 hướng**, nhớ theo khung này:

```
A. ĐỪNG để chúng cùng miss      → jitter TTL · refresh-ahead · probabilistic early recompute (XFetch)
B. CHỈ cho MỘT kẻ qua nguồn      → singleflight (in-proc) · distributed lock / lease (cross-proc)
C. ĐƯA tạm đồ cũ trong lúc tính  → serve-stale / stale-while-revalidate · stale-if-error
D. ĐỪNG lazy-cache nữa           → precompute / materialized view · cache warming (cold start)
   + bọc nguồn: negative cache · Bloom filter · rate limit / load shedding · circuit breaker
```

### Nhóm A — Đừng để chúng cùng miss

**A1. Jitter (TTL ngẫu nhiên)** — nối thẳng Bài 1 ➕. Chữa **avalanche do TTL đồng bộ**:
```javascript
// Thay vì TTL cố định 300s (mọi key warm cùng lúc → hết hạn cùng lúc):
const ttl = 300 + Math.floor(Math.random() * 60); // 300–360s → rải đều, không "vách đá"
await redis.set(key, val, "EX", ttl);
```
> Rẻ, đơn giản, **luôn nên bật**. Là tuyến phòng thủ đầu tiên.

**A2. Refresh-ahead** (đã có ở chuyên đề Cache warming) — làm tươi **trước** khi hết hạn → hot key **không bao giờ thật sự miss** → không có cú "vách đá".

**A3. ⭐ XFetch — Probabilistic early recomputation (giải pháp học thuật đẹp nhất).**
Thay vì để key hết hạn rồi cả đàn miss, **mỗi request có một xác suất nhỏ tự nguyện tính lại SỚM**, xác suất này **tăng dần khi gần hết hạn** và **lớn hơn với key đắt tính lại**:
```javascript
// XFetch (Vattani et al., VLDB 2015): tính lại sớm theo xác suất
// delta = thời gian tính lại lần trước (giá compute) ; beta >= 1 (chỉnh "sớm cỡ nào")
function shouldEarlyRecompute(deltaMs, expiryMs, beta = 1) {
  return Date.now() - deltaMs * beta * Math.log(Math.random()) >= expiryMs;
}
// Khi true → MỘT request đi tính lại NỀN trước khi key chết; số còn lại vẫn dùng bản hiện tại
```
> 🎯 Vẻ đẹp: **trải đều việc tính lại lên nhiều request trước thời điểm hết hạn** → không còn "vách đá" tại đúng giây TTL=0. Key càng **đắt** (delta lớn) càng được tính lại **sớm hơn**. Đây là cách "không vách đá" tối ưu về mặt toán học.

### Nhóm B — Chỉ cho MỘT kẻ qua nguồn (request coalescing)

**B1. Singleflight / request collapsing (in-process).** Nhiều caller cùng key → **chỉ một** đi nguồn, số còn lại **chờ chung kết quả**:
```javascript
// Cache cả PROMISE đang bay, không chỉ value → caller đồng thời await CHUNG một promise
const inflight = new Map();
async function getCoalesced(key, loader) {
  const cached = L1.get(key);
  if (cached !== undefined) return cached;       // HIT
  if (inflight.has(key)) return inflight.get(key); // đã có kẻ đang tính → BÁM theo, không đập thêm
  const p = loader(key).finally(() => inflight.delete(key));
  inflight.set(key, p);                           // kẻ đầu tiên đi nguồn
  const val = await p; L1.set(key, val); return val;
}
```
> ⚠️ Bẫy singleflight: (1) nếu kẻ-đầu-tiên **fail**, tất cả waiter **cùng fail** → cân nhắc retry; (2) một call **chậm** treo mọi waiter → phải có **timeout**. (3) Chỉ gộp **trong một tiến trình** — nhiều instance vẫn mỗi instance một lần đi nguồn → cần B2.

**B2. Distributed lock / lease (cross-process).** Nhiều instance → dùng khoá phân tán để **chỉ một instance toàn cụm** được tính lại:
```javascript
// Redis lock: chỉ kẻ chiếm được lock mới đi DB; số còn lại chờ ngắn rồi đọc lại / dùng stale
async function getWithLock(key, loader) {
  const hit = await redis.get(key);
  if (hit) return JSON.parse(hit);
  const lock = await redis.set(`lock:${key}`, "1", "NX", "PX", 5000); // SET NX PX = chiếm khoá 5s
  if (lock) {
    try {
      const val = await loader(key);                       // CHỈ kẻ này đập DB
      await redis.set(key, JSON.stringify(val), "EX", 300);
      return val;
    } finally { await redis.del(`lock:${key}`); }
  }
  await sleep(50);                  // kẻ thua khoá: chờ ngắn rồi thử lại (hoặc trả stale nếu có)
  return getWithLock(key, loader);
}
```

### Nhóm C — Đưa tạm đồ cũ trong lúc tính (chấp nhận stale — Bài 1: "stale ở chỗ rẻ thì OK")

**C1. Serve-stale / stale-while-revalidate.** Key hết hạn → **trả ngay bản cũ** cho user, **làm tươi nền**. User không phải chờ, nguồn không bị đàn đập:
```
Cache-Control: max-age=60, stale-while-revalidate=600
→ 60s đầu: tươi. Sau đó tới 600s: trả STALE ngay + refresh nền. User KHÔNG chờ, DB KHÔNG bị đàn.
```
**C2. Stale-if-error.** Nguồn lỗi → **trả bản cũ** thay vì lỗi → tăng độ bền (resilience).
> ➜ Đây thường là giải pháp **đơn giản & mạnh nhất** *nếu path chịu được lệch vài giây* (like count, feed, mô tả SP — Bài 1). Đừng dùng cho tiền/tồn kho.

### Nhóm D — Đừng lazy-cache nữa + bọc nguồn

- **Precompute / Materialized View** (Bài 1 g): key **luôn nóng** và stampede-prone → **đừng** để nó lazy; tính sẵn eager. **Không miss = không stampede.**
- **Cache warming** (chuyên đề trước): chữa stampede **do cold start**.
- **Negative caching** (Bài 1 ➕): chữa **penetration** — cache cả kết quả "không tồn tại" để khỏi đập DB lặp lại.
- **Bloom filter**: chặn **penetration**/tấn công — hỏi nhanh "key này có khả năng tồn tại không?"; nếu Bloom nói "chắc chắn không" → **khỏi đập DB**.
- **Rate limit / load shedding / circuit breaker** ở nguồn: dù mọi cách trên thủng, **giới hạn số query đồng thời xuống DB** để DB không chết; ngắt mạch khi nguồn đang lỗi.

---

## ⑥ CASE STUDY THỰC TẾ

**1. ⭐ Facebook Memcached — "leases" (CHÍNH bài trong reading list Bài 1: *Scaling Memcached at Facebook*).**
Khi một key **miss**, Memcached **chỉ cấp cho MỘT client một "lease" (token 64-bit)** để đi tính và set lại; các client khác được bảo **chờ ngắn rồi thử lại** hoặc **dùng bản stale**. Memcached còn **rate-limit cấp lease** (cỡ một lease/key mỗi ~10s) → bóp thundering herd. Cơ chế này **một công đôi việc**: vừa chống **stampede** (thundering herd) vừa chữa **race "set đè bản cũ"** (stale set). ➜ Đây là giải pháp B2 (lease) kinh điển nhất.

**2. nginx — `proxy_cache_lock` + `proxy_cache_use_stale` (tầng reverse proxy của Bài 1).**
`proxy_cache_lock on;` → khi một phần tử cache đang được điền, **chỉ một request đi origin**, số còn lại **chờ** (coalescing — nhóm B). `proxy_cache_use_stale updating;` → trong lúc cập nhật, **trả bản cũ** (serve-stale — nhóm C). Hai dòng cấu hình chống stampede ngay ở proxy.

**3. Go — `singleflight` / groupcache (Brad Fitzpatrick).**
Gói `golang.org/x/sync/singleflight` gộp các call trùng key thành một; `groupcache` dựng cache phân tán chống thundering herd. Rất nhiều dịch vụ Go dùng — ví dụ điển hình của nhóm B1.

**4. CDN — request coalescing / origin shielding (Cloudflare, Akamai, Fastly).**
Sau khi purge một asset hot, nếu mọi edge cùng miss về origin → origin lãnh đòn. CDN dùng **tiered cache / origin shield**: chỉ **một** request mỗi object được phép tới origin, phần còn lại bám theo → coalescing ở quy mô toàn cầu.

**5. Metastable failures (nghiên cứu, HotOS 2021).**
Stampede là một **ngòi nổ kinh điển** đẩy hệ phân tán vào **metastable failure** (sập tự duy trì bởi retry). Bài học vận hành: chống stampede **đi kèm** với **retry có backoff + jitter** và **load shedding**, nếu không retry storm sẽ tự nuôi cú sập.

---

## ⑦ CÁC TRƯỜNG HỢP MỞ RỘNG

- **Stampede "lần đầu" vs "hết hạn".** Key **chưa từng có** trong cache (cold/viral mới) cần **lock/singleflight + warming**; key **hot vừa hết hạn** hợp với **refresh-ahead/XFetch + serve-stale**. Thuốc khác nhau.
- **Stampede trong multi-tier (Bài 1 d).** L1 miss → L2 miss → DB. Nên **coalesce ở từng tầng**: gộp ở L1 (in-proc singleflight) **và** ở L2/origin (distributed lock) — nếu chỉ gộp L1, mỗi instance vẫn đập L2/DB một lần.
- **Lock cũng có thể thành nút thắt.** Nếu hàng nghìn waiter cùng poll một lock → tải lên Redis lock. Cân nhắc **chờ thông minh** (pub/sub báo khi value sẵn) thay vì poll, hoặc **serve-stale** để waiter khỏi chờ.
- **Penetration do tấn công.** Kẻ xấu hỏi dồn các ID **không tồn tại** (mỗi lần một ID lạ → negative cache không trúng) → **Bloom filter** + rate limit theo client là tuyến chặn.
- **"Warm rồi set cùng TTL" tự tạo avalanche.** Chính việc warm cả loạt (chuyên đề trước) nếu set TTL bằng nhau sẽ đẻ ra avalanche một giờ sau → **bắt buộc jitter** khi warm.

---

## ⑧ KHUNG QUYẾT ĐỊNH STAFF ENGINEER

```
1. Path có CHỊU ĐƯỢC stale không? (Bài 1: tiền/tồn kho thì KHÔNG)
   ├─ CÓ  → SERVE-STALE / stale-while-revalidate  ← đơn giản & mạnh nhất, ưu tiên
   └─ KHÔNG (phải tươi) → sang 2

2. Nhiều key hết hạn cùng lúc? (warm cả loạt / cron cùng TTL)
   └─ CÓ → JITTER TTL (luôn nên bật) — chống avalanche

3. MỘT hot key bị đập (breakdown)?
   ├─ Trong 1 tiến trình   → SINGLEFLIGHT
   ├─ Nhiều instance       → DISTRIBUTED LOCK / LEASE (kiểu Facebook)
   └─ Muốn "không vách đá"  → REFRESH-AHEAD / XFETCH (tính lại sớm theo xác suất)

4. Hỏi key KHÔNG TỒN TẠI (penetration / tấn công)?
   └─ NEGATIVE CACHE + BLOOM FILTER + rate limit theo client

5. Key LUÔN nóng & đắt?
   └─ Đừng lazy → PRECOMPUTE / MATERIALIZED VIEW (không miss = không stampede)

6. Cold start (deploy/scale/failover)?
   └─ CACHE WARMING (chuyên đề trước) + warm-from-peer

7. BỌC CUỐI CÙNG (luôn có): rate limit/load shedding ở DB + circuit breaker
   + retry có EXPONENTIAL BACKOFF + JITTER (chống retry storm → metastable)
```

> 🥇 **Mặc định thực dụng:** path chịu stale → **serve-stale** + **jitter**. Path phải tươi → **singleflight (in-proc) + distributed lock (cross-proc)** + **jitter**. Key luôn nóng → **precompute**. Và **luôn** có **load shedding + backoff-with-jitter** ở dưới cùng.

---

## ⑨ VẬN HÀNH & METRIC

| Dấu hiệu | Nghĩa | Hành động |
|---|---|---|
| **DB QPS spike trùng với cache miss spike** | Đang bị stampede | Tìm hot key vừa hết hạn / node cache chết |
| **p99 nhảy theo CHU KỲ (mỗi giờ/mỗi TTL)** | **TTL đồng bộ** → avalanche định kỳ | **Jitter TTL** |
| **"duplicate origin requests / key"** cao | Thiếu coalescing | Bật singleflight / lock |
| **Lock wait time tăng** | Lock thành nút thắt | Serve-stale cho waiter / pub-sub thay poll |
| **Retry count bùng nổ khi sự cố** | Nguy cơ metastable | Backoff + jitter + load shedding |

> Nối Bài 1: đừng nhìn hit ratio đơn thuần — hãy nhìn **DB QPS có bị spike tại biên TTL không**. Đó là dấu vân tay của stampede.

---

## ⑩ BẪY THƯỜNG GẶP / 🔎 AI HAY SAI

- 🔎 **Chỉ đặt TTL cố định cho cả loạt key** → đồng bộ hết hạn → **avalanche định kỳ**. Quên jitter là lỗi kinh điển nhất.
- 🔎 **Singleflight rồi tưởng xong** — nó chỉ gộp **trong một tiến trình**; nhiều instance vẫn đập nguồn → cần **distributed lock** thêm.
- 🔎 **Lock không có timeout / không tự nhả** → kẻ giữ lock chết → **mọi waiter treo vĩnh viễn** (deadlock). Luôn đặt TTL cho lock.
- 🔎 **Quên negative caching** → hỏi key không tồn tại lặp lại đập DB mãi (**penetration**).
- 🔎 **Serve-stale nhầm chỗ** → trả bản cũ cho **tiền/tồn kho** (cần strong consistency — Bài 1) → bán đồ đã hết.
- 🔎 **Chữa stampede nhưng quên retry storm** → cache phục hồi rồi hệ vẫn sập vì retry không backoff → **metastable**.
- 🔎 **Lazy-cache một thứ LUÔN nóng & đắt** → nên precompute; cứ cố lock/coalesce mãi là chữa ngọn.

---

## ⑪ MENTAL MODEL + CÂU THẦN CHÚ

> **Cache stampede = cache "phản bội" đúng lúc key nóng nhất: ngàn request cùng miss một key → ngàn cú đập nguồn giống hệt.**
> **Bốn lối thoát:** (A) **đừng để cùng miss** (jitter · refresh-ahead · XFetch), (B) **chỉ một kẻ qua** (singleflight · lock/lease), (C) **đưa tạm đồ cũ** (serve-stale), (D) **đừng lazy** (precompute/MV).
> **Penetration → negative cache + Bloom. Breakdown → lock/singleflight. Avalanche → jitter + HA.**
> **Và luôn bọc đáy bằng load shedding + retry backoff-có-jitter** để stampede không hoá metastable.

---

## ⑫ CÂU HỎI PHỎNG VẤN

- **(dễ)** Cache stampede là gì, vì sao key càng hot càng nguy? → ngàn request cùng miss một key → cùng đập nguồn; hot = nhiều request đồng thời = cú đập càng lớn.
- **(TB)** Một hot key vừa hết TTL gây spike DB — chống thế nào? → singleflight/lock (chỉ một kẻ tính lại) + serve-stale + refresh-ahead để khỏi có "vách đá".
- **(TB)** Mỗi giờ p99 lại nhảy một nhịp — vì sao và sửa sao? → **TTL đồng bộ** (loạt key cùng TTL hết hạn cùng lúc) → **jitter TTL**.
- **(khó)** Phân biệt **penetration / breakdown / avalanche**? → key-không-tồn-tại (negative cache + Bloom) / một-hot-key-hết-hạn (lock) / loạt-key-hoặc-node-chết (jitter + HA).
- **(khó)** Singleflight vs distributed lock vs serve-stale — chọn khi nào? → singleflight gộp **trong tiến trình**; lock gộp **toàn cụm** nhưng tốn coordination; serve-stale **đơn giản nhất nếu chịu được stale**.
- **(khó)** Giải thích XFetch / probabilistic early recompute. → mỗi request có xác suất nhỏ tự tính lại **sớm**, tăng dần khi gần hết hạn & với key đắt → **trải đều việc tính lại, xoá "vách đá" tại TTL=0**.
- **(thách đố)** "Đã thêm cache + lock mà sự cố vẫn kéo dài sau khi cache ấm lại — vì sao?" → **retry storm → metastable failure**; cần **backoff + jitter + load shedding/circuit breaker**, không chỉ chống stampede ở tầng cache.

---

## ⑬ GLOSSARY MỞ RỘNG (đóng vào hoàn cảnh)

| Thuật ngữ | Nghĩa nhanh | 🎬 Hoàn cảnh |
|---|---|---|
| **Cache stampede / thundering herd / dog-pile** | Nhiều request cùng miss một key → đập nguồn đồng loạt | *"Hot key hết TTL lúc cao điểm → **stampede** sập DB."* |
| **Cache breakdown** (击穿) | **Một** hot key hết hạn gây stampede | *"Đúng key trang chủ hết hạn → **breakdown**."* |
| **Cache penetration** (穿透) | Hỏi key **không tồn tại** → luôn miss → đập DB | *"Bot hỏi ID rác liên tục → **penetration**."* |
| **Cache avalanche** (雪崩) | Loạt key hết hạn / node chết → miss diện rộng | *"Redis restart → **avalanche** toàn cụm."* |
| **Singleflight / request coalescing** | Gộp nhiều miss cùng key thành 1 lần đi nguồn (in-proc) | *"`singleflight` cho mỗi key chỉ 1 kẻ đi DB."* |
| **Distributed lock / lease** | Khoá toàn cụm: chỉ 1 instance tính lại | *"Redis `SET NX` lock; hoặc **lease** kiểu Facebook."* |
| **Serve-stale / stale-while-revalidate** | Trả bản cũ ngay + làm tươi nền | *"`stale-while-revalidate=600` → user không chờ."* |
| **Stale-if-error** | Nguồn lỗi → trả bản cũ | *"DB lỗi → phục vụ **stale** thay vì 500."* |
| **Refresh-ahead** | Làm tươi **trước** khi hết hạn | *"Hot key không bao giờ thật sự miss."* |
| **XFetch / probabilistic early recompute** | Tính lại sớm theo xác suất, không "vách đá" | *"Key đắt → tính lại sớm hơn theo `delta·beta·log(rand)`."* |
| **Jitter** | TTL ngẫu nhiên chống hết-hạn-đồng-loạt | *"`EX 300 + rand(60)` → tránh avalanche."* |
| **Negative caching** | Cache cả kết quả "không tồn tại" | *"Chống **penetration**, khỏi đập DB tìm thứ không có."* |
| **Bloom filter** | Hỏi nhanh "key có khả năng tồn tại?" | *"Bloom nói 'chắc chắn không' → khỏi đập DB."* |
| **Load shedding / circuit breaker** | Bỏ bớt tải / ngắt mạch để cứu nguồn | *"DB quá tải → **shed** request thừa, **break** mạch."* |
| **Metastable failure** | Sập tự duy trì bởi retry, không tự thoát | *"Cache ấm lại rồi vẫn sập vì **retry storm**."* |
| **Backoff + jitter** | Retry giãn dần + nhiễu, chống retry storm | *"Exponential **backoff** kèm **jitter** khi retry."* |

---

> 🧭 **Một dòng để mang theo:** *Stampede là lúc cache phản bội đúng key nóng nhất. Hỏi đúng một câu — "path này chịu được stale không?": chịu được thì **serve-stale + jitter** là xong; không chịu được thì **singleflight + distributed lock + jitter**; luôn-nóng-và-đắt thì **precompute**; và đừng quên bọc đáy bằng **load shedding + retry backoff-có-jitter** để stampede không hoá metastable.*
