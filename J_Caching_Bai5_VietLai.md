# 🎯 BÀI 5 — Bộ ba kinh điển: Penetration / Breakdown (Stampede) / Avalanche + Hot/Big key
**Phủ:** J-PROB-001 → J-PROB-011

---

## ① Mục tiêu & vị trí trong mạch

Đây là **bộ ba câu hỏi phỏng vấn kinh điển** (đặc biệt phong cách Trung Quốc) — **ai làm cache ở quy mô thật đều phải nắm**. Bài này gom **mọi thứ đã học** (**TTL**, **key design**, **invalidation**, **consistency**) để trả lời một câu hỏi sống còn: *khi tải thật đập vào, cache hỏng theo những KIỂU NÀO, và phòng thủ NHIỀU LỚP ra sao?*

Ngoài bộ ba bệnh, bài còn thêm **hot key** và **big key** — hai **"bệnh vận hành" (operational issue)** của Redis mà sách lý thuyết hay bỏ qua nhưng production gặp suốt.

> 💡 **Ý chính:** Ba bệnh **trông giống nhau** ("cache miss → DB chịu tải") nhưng **nguyên nhân khác nhau** → **thuốc khác nhau**. Dùng nhầm thuốc = không chữa được bệnh. Cả bài xoay quanh việc **chẩn đoán đúng bệnh trước khi kê đơn**.

---

## ② Giảng cơ bản → nâng cao

### (a) Phân biệt rõ BA khái niệm *(J-PROB-001)* — ⭐ ĐỪNG LẪN

Đây là phần phải thuộc nằm lòng. Ba bệnh khác nhau ở **nguyên nhân**:

| Bệnh | Nguyên nhân | Hệ quả |
|---|---|---|
| **Penetration** (xuyên thủng) | Query data **KHÔNG tồn tại** (không có ở cache lẫn DB) — **id rác / tấn công** | **Mọi lần đều xuống DB** (cache không bao giờ điền được vì chẳng có gì để điền) |
| **Breakdown / Stampede** (đánh sập điểm) — còn gọi **dogpile / thundering herd / cache stampede** | **MỘT hot key hết hạn** | Loạt request **đồng thời cùng miss → cùng nạp lại** → đập DB |
| **Avalanche** (tuyết lở) | **NHIỀU key hết hạn cùng lúc** (hoặc **Redis chết**) | DB **ngập đồng loạt** |

> 🎯 **Mnemonic (3 chữ để nhớ cả đời):**
> - **Penetration = key KHÔNG có thật** (key ma).
> - **Breakdown = MỘT key nóng vỡ.**
> - **Avalanche = NHIỀU key đổ cùng lúc.**

**Ẩn dụ một cửa hàng (để thấy ba bệnh khác nhau):**
> Hình dung **cache = quầy lễ tân**, **DB = kho hàng phía sau** (chậm, dễ quá tải).
> - **Penetration:** khách liên tục hỏi món **không hề có trong cửa hàng**. Lễ tân chẳng có sẵn, *lần nào cũng* phải chạy vào kho kiểm rồi quay ra nói "không có" → kho mệt nhoài vì câu hỏi vô nghĩa.
> - **Breakdown:** **một món hot** (ai cũng hỏi) vừa bị gỡ khỏi quầy đúng giờ cao điểm → **cả đám khách đồng loạt** ùa vào kho hỏi cùng món đó.
> - **Avalanche:** **toàn bộ quầy bị dọn sạch cùng lúc** (hết ca, mất điện) → mọi khách đổ hết vào kho một lượt → kho sập.

---

### (b) Chống **Penetration** *(J-PROB-002, 003, 011)*

Có 3 lớp phòng thủ, dùng phối hợp:

**(1) Negative caching** *(Bài 3):* cache lại **marker `null`/empty với TTL ngắn**. Lần sau hỏi cùng **id rác** → **negative hit** → **không xuống DB**.

**(2) Bloom filter** — vũ khí chính:
> **Bloom filter** là một **cấu trúc membership xác suất** (probabilistic), trả lời câu hỏi *"phần tử này có nằm trong tập không?"* bằng cực ít bộ nhớ. Đặc tính cốt lõi:
> - Trả **"CHẮC CHẮN KHÔNG có"** → **chặn sớm**, không cần đụng DB.
> - Hoặc **"CÓ THỂ có"** → cho qua (đi kiểm tiếp).
> - ⚠️ Có **false positive** (cho qua nhầm thứ không có — **chấp nhận được**, chỉ tốn 1 lần kiểm thừa).
> - ✅ **KHÔNG bao giờ false negative** (đã nói "không có" thì *chắc chắn* không có — nên chặn an toàn).
>
> **Nhược:** **khó xoá phần tử** (muốn xoá phải dùng **counting bloom filter**). *(verify.)*

**Ẩn dụ Bloom filter:** như **danh sách "khách VIP" ở cửa**. Bảo vệ liếc danh sách: tên *không có* → **chắc chắn đuổi** (không cho vào quấy rầy DB). Tên *có* → cho vào kiểm kỹ (có thể trùng tên, nhưng không sao). Bảo vệ không bao giờ đuổi nhầm VIP thật.

**(3) Validate input** (kiểm format id) **+ rate limit theo client** — chặn ngay từ vòng ngoài.

> 🚨 **Tấn công cố ý *(J-PROB-011)* — vì sao null-cache KHÔNG đủ:**
> Một **adversary** chủ động **sinh ra vô số key đa dạng** (mỗi request một id rác khác nhau). Nếu chỉ dùng **negative caching**, bạn sẽ **đẻ ra vô số null key** → **null-cache phình to, ngốn RAM**. → Phải có **Bloom filter** (chặn *trước khi* kịp tạo null key) **+ validate format + rate limit**. Một mình null-cache thua cuộc tấn công này.

---

### (c) Chống **Breakdown / Stampede** *(J-PROB-004, 005)*

**(1) Mutex / Single-flight** — ý tưởng "chỉ một người đi lấy":
> Khi **hot key miss**, **chỉ cho 1 request lấy lock** (`SET NX`) để **rebuild** (nạp lại từ DB); **số còn lại CHỜ ngắn rồi đọc lại cache** (hoặc đọc bản stale). → Giảm từ **N query DB xuống còn 1**.

**Ẩn dụ single-flight:** cả văn phòng hết cà phê. Thay vì **20 người cùng chạy ra quán** (DB nghẽn), **1 người xung phong đi mua cho cả nhóm**, 19 người còn lại ngồi đợi vài phút rồi có cà phê chung.

```
hot key MISS (N request cùng lúc)
   │
   ├─ request #1: SET NX lock OK → đi rebuild (1 query DB) → set cache → trả
   └─ request #2..N: lock FAIL → chờ ~50ms → đọc lại cache (đã có) ✅
─────────────────────────────────────────
KẾT QUẢ: N request nhưng chỉ 1 lần đập DB
```

*(verify cú pháp; lock ĐÚNG — token, fencing, Lua release — xem I-DLOCK.)*

**(2) Probabilistic early expiration / logical TTL *(J-PROB-005)* — thường LỢI HƠN mutex:**
> Thay vì đợi key chết hẳn rồi tranh nhau rebuild, ta **refresh sớm một cách NGẪU NHIÊN trước hạn** — **xác suất refresh tăng dần khi càng gần expiry**. Kết quả: **một request lẻ chủ động làm mới ở nền TRƯỚC KHI key thật sự hết** → **không bao giờ có khoảnh khắc tất cả cùng miss**.
>
> ✅ **Hơn mutex ở chỗ:** **không cần lock toàn cục** → **tránh nghẽn cổ chai tại chính cái mutex**.
>
> **Biến thể:** *"never expire + async refresh"* — key **không đặt TTL cứng**, một tiến trình nền **tự làm mới** định kỳ.

**Ẩn dụ probabilistic early refresh:** quán cà phê thấy bình **sắp cạn** → **pha bình mới TRƯỚC** khi cạn hẳn, nên khách không bao giờ phải đứng đợi lúc bình rỗng (so với single-flight: đợi cạn rồi cử 1 người đi mua).

---

### (d) Chống **Avalanche**

Avalanche có **hai nguyên nhân**, mỗi cái một thuốc:

**(1) Do mass expiry *(J-PROB-006)*:** nhiều key được set **cùng một TTL** (vì *warm cùng lúc* hoặc *TTL cố định*) → **hết hạn đồng loạt** → DB sốc một lượt.
> **Thuốc — TTL jitter:** đặt **`TTL = base + random(offset)`** để **rải thời điểm hết hạn**, tránh các key **đồng pha** (cùng chết một giây).
>
> **Ẩn dụ:** đừng để **cả lớp cùng nộp bài đúng 12h00**; cho mỗi người một deadline lệch vài phút → phòng văn thư không bị dồn cục.

**(2) Do Redis sập *(J-PROB-007)* — phòng thủ NHIỀU LỚP:**
> 1. **HA** (high availability): **replica / sentinel / cluster** để Redis không phải "single point of failure".
> 2. **Circuit breaker + rate limit** bảo vệ DB khi cache mất (ngắt mạch, không cho full traffic dội thẳng).
> 3. **Local L1 fallback** (cache in-process còn đỡ được phần nào).
> 4. **Graceful degradation** (trả **stale / partial** thay vì sập hẳn).
>
> 🎯 **Nguyên tắc:** **KHÔNG bao giờ để DB nhận full traffic trần** khi cache biến mất. *(Cross-ref M DDoS, K circuit breaker.)*

---

### (e) Hot key *(J-PROB-008)*

> **Một single key** nhận **quá nhiều traffic** → **shard/node Redis giữ key đó trở thành bottleneck** (nghẽn CPU / băng thông *của riêng node đó*) — **dù hit ratio = 100%**.

**Vì sao hit 100% vẫn nghẽn?** Vì vấn đề **không phải miss**, mà là **một node phải gánh toàn bộ request cho key nóng đó**, trong khi các node khác rảnh. Cache "trúng" hết nhưng *một cánh tay phải bê hết tạ*.

**Cách giảm:**
- **Đọc từ replica** (san tải đọc).
- **Local cache hot key ở app (L1)** — request không cần ra Redis.
- **Key splitting:** chia thành **N bản** `key#0..key#N`, mỗi request **đọc ngẫu nhiên 1 bản** → tải rải ra N node.
- **Client-side cache.**

---

### (f) Big key *(J-PROB-009)*

> **Value quá lớn** / **collection khổng lồ** (một list/hash chứa hàng triệu phần tử).

**Vì sao nguy hiểm?** Redis xử lý **đơn luồng (single-thread)**:
- Thao tác trên big key là **O(n)** → **block single-thread** → **tăng latency cho MỌI client** (không chỉ client đụng key đó).
- **Tốn mạng** khi truyền value to.
- **Lệch memory giữa các shard** (một shard phình vì ôm big key).
- **Nguy hiểm khi `DEL`/expire** — xoá một big key cũng là thao tác **blocking**.

**Giải:**
- **`UNLINK`** (xoá **async**, không block) **thay cho `DEL`**.
- **Scan từng phần** (**`HSCAN` / `SSCAN`**) thay vì lấy/duyệt cả cục.
- **Tách nhỏ key** ra nhiều key con. *(verify.)*

---

### (g) Stampede vs Avalanche cần PHƯƠNG ÁN KHÁC NHAU *(J-PROB-010)* — ⭐⭐⭐⭐⭐

Đây là cái bẫy mà câu thách đố ở mục ⑧ nhắm vào:

| | **Stampede** | **Avalanche** |
|---|---|---|
| Quy mô | **1 key** nóng vỡ | **Nhiều key** / đồng loạt |
| Thuốc đúng | **Single-flight / lock per-key** | **Rải TTL (jitter) + HA + bảo vệ DB tổng thể** |

> 🚨 **Nhầm thuốc = không chữa đúng bệnh:**
> - Đặt **mutex cho từng key** **KHÔNG cứu nổi avalanche** (vì *mỗi key vẫn cho 1 request xuống DB* → hàng nghìn key = hàng nghìn query → vẫn ngập).
> - **Rải TTL KHÔNG cứu nổi một hot key** đơn lẻ (chỉ 1 key thì jitter chẳng giải quyết gì).

---

## ③ ⚠️ Kiến thức cũ / bị thay thế

| Cái CŨ | Thay bằng (hiện nay) | Vì sao |
|---|---|---|
| **TTL cố định** cho cả batch key | **TTL jitter** (base + random) | Tránh **mass-expiry avalanche** |
| Để **N request cùng rebuild** hot key | **Single-flight / mutex** hoặc **probabilistic early refresh** | Giảm **N → 1** query DB lúc expiry |
| **`DEL` big key** | **`UNLINK`** + tách nhỏ | `DEL` **block single-thread** |
| **Null-cache là đủ** chống penetration | Null-cache **+ Bloom filter** (+ validate + rate limit) | Tấn công đa dạng key làm **phình null-cache** |
| **`KEYS *`** để tìm hot/big key | **`SCAN`** + `redis-cli --bigkeys / --hotkeys` | `KEYS` **block** (Bài 7) |

---

## ④ Áp dụng thực tế + So sánh bigtech

**Use case: flash sale (mở bán giảm giá).** Đây là *bộ ba bệnh hội tụ một chỗ*:
- Trang sản phẩm khuyến mãi = **hot key** + **nhiều cache warm cùng lúc** lúc mở sale.
- **Phòng thủ:** **TTL jitter** cho catalog + **single-flight** cho key sản phẩm hot + **L1 local** + **circuit breaker** trước DB.
- Bot quét **id sản phẩm ngẫu nhiên** → **Bloom filter + negative cache + rate limit**.

**Pattern ở bigtech:**
- Thuật ngữ **"thundering herd" / "dogpile"** rất phổ biến ở phương Tây; **Facebook** giải bằng **leases** (paper **Memcached**).
- **Probabilistic early expiration** được mô tả trong paper *"Optimal Probabilistic Cache Stampede Prevention"* (**XFetch**). *(verify khi cần trích chính xác.)*

---

## ⑤ Code thực hành + cấu hình

**TTL jitter (chống avalanche):**

```javascript
// rải thời điểm hết hạn để tránh mass-expiry
const jitter = (base) => base + Math.floor(Math.random() * base * 0.2); // ±20%
await redis.set(key, val, "EX", jitter(600)); // 600s ± 20% → expiry rải đều
```

**Single-flight (chống stampede) — phác thảo:**

```javascript
// Chỉ 1 request rebuild hot key; số còn lại chờ ngắn rồi đọc lại cache.
async function getWithSingleFlight(key, rebuild) {
  let cached = await redis.get(key);
  if (cached !== null) return JSON.parse(cached);      // HIT

  const lockKey = `lock:${key}`;
  // SET NX PX = chỉ set nếu CHƯA có, TTL 5s (lock TỰ hết nếu rebuilder chết)
  const got = await redis.set(lockKey, "1", "NX", "PX", 5000); // verify; lock ĐÚNG xem I-DLOCK
  if (got === "OK") {
    try {
      const fresh = await rebuild();                   // ✅ CHỈ 1 request đập DB
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

```javascript
// Lưu kèm thời điểm tính (delta = thời gian rebuild). Refresh sớm với xác suất TĂNG DẦN.
// shouldRefresh = now - delta * beta * ln(rand()) >= expiry
// → gần expiry thì 1 request lẻ CHỦ ĐỘNG refresh nền, không ai cùng miss.
```

> ⚠️ Lock ở đây **chỉ là phác thảo**; **distributed lock đúng** (token, Lua release, fencing, Redlock caveats) nằm ở **mục I (I-DLOCK)**.
> ⚠️ Dùng **`redis-cli --bigkeys / --hotkeys`** để soi (verify).

---

## ⑥ Keywords cần nhớ (glossary)

🧠 **Cần HIỂU:**
- **3 bệnh khác nhau** & cách chống tương ứng.
- **Vì sao probabilistic > mutex** (không nghẽn ở lock toàn cục).
- **Hot key bottleneck dù hit** (một node gánh hết).
- **Big key block single-thread** (O(n) chặn mọi client).
- **Tấn công penetration làm phình null-cache**.

📌 **Cần THUỘC:**
- **penetration = key không tồn tại** / **breakdown = 1 hot key expiry** / **avalanche = nhiều key | Redis chết**.
- **jitter** chống avalanche; **single-flight** chống stampede.
- **Bloom filter** (có **FP**, không **FN**).

🛠️ **Cần LÀM ĐƯỢC:** thêm **TTL jitter**; viết **single-flight**; cấu hình **Bloom filter + negative cache**; dùng **`UNLINK`/`SCAN`** cho big key.

---

## ⑦ Mental model

> **Penetration = key ma** → **Bloom + null-cache**.
> **Breakdown = 1 key nóng vỡ** → **single-flight / early refresh**.
> **Avalanche = nhiều key đổ** → **TTL jitter + HA + circuit breaker**.
> **Hot / big key** → **split / L1 / `UNLINK`**.

---

## ⑧ Câu hỏi phỏng vấn & thách đố

- **(TB)** Phân biệt penetration / breakdown / avalanche. → *(bảng ②a)*.
- **(khó)** 2 cách chống penetration? → **negative cache + Bloom filter** (+ validate, rate limit).
- **(khó)** Bloom filter chống penetration thế nào, đánh đổi? → "chắc chắn không / có thể có"; **FP có, FN không**; khó xoá.
- **(khó)** Chống stampede bằng single-flight thế nào? → 1 request lock rebuild, còn lại chờ/đọc stale → **N → 1**.
- **(khó)** Avalanche do mass-expiry chống bằng gì? → **TTL jitter** rải thời điểm hết hạn.
- **(rất khó)** Hot key gây vấn đề gì dù hit 100%, giảm sao? → **node giữ key thành bottleneck**; **replica / L1 / key-splitting**.

> **🧩 Thách đố:** *"Bạn đặt mutex single-flight cho MỌI key để 'chống stampede'. Lúc Redis restart (mọi key mất), hệ thống vẫn sập DB. Vì sao?"*
>
> **Trả lời:** Đó là **avalanche do cache mất**, **không phải stampede 1-key**. **Mutex per-key không cứu** khi **hàng nghìn key cùng miss** — vì **mỗi key vẫn cho 1 request xuống DB** → tổng cộng vẫn ngập. Cần **HA + circuit breaker + L1 fallback + warming có jitter**. → **Đúng bệnh, đúng thuốc** *(J-PROB-010)*.

---

## ⑨ Bài tập + tiêu chí tự chấm

**BT1.** Thêm **jitter + single-flight** cho `getHotProduct`.
> ✅ **Đạt khi:** TTL có **random**; **chỉ 1 request rebuild** lúc miss; **còn lại không đập DB**.

**BT2.** Thiết kế phòng thủ **penetration** cho `GET /user/:id` bị bot quét id ngẫu nhiên.
> ✅ **Đạt khi:** **Bloom filter** chặn id không tồn tại + **negative cache TTL ngắn** + **rate limit**; **nêu rõ vì sao null-cache đơn thuần không đủ**.

---

## ⑩ Đọc thêm

- **redis.io/docs** — *Bloom filter (Redis probabilistic)*, *Client-side caching*.
- Paper *Optimal Probabilistic Cache Stampede Prevention* (**XFetch**).
- **Facebook** — *Scaling Memcache at Facebook* (**leases / thundering herd**).

---

> 🔎 **AI hay sai ở đây:** AI **gộp ba bệnh làm một** ("cache miss thì DB chịu"), rồi **dùng MỘT thuốc cho mọi bệnh**.
> **Luôn tự chẩn đoán:** *đây là **key-ma**, **một-key-nóng**, hay **nhiều-key-đổ**?* — rồi chọn phòng thủ tương ứng.
