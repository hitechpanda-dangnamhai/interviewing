# 🎯 BÀI 2 — Cache Strategies (5 chiến lược đọc/ghi cache)
**Phủ:** J-STRAT-001 → J-STRAT-010

---

## ① Mục tiêu & vị trí trong mạch

Bài 1 trả lời câu hỏi **"vì sao cache"**. Bài 2 trả lời câu hỏi tiếp theo: **"khi đã quyết cache rồi thì ĐỌC và GHI giữa cache với DB diễn ra THEO LUỒNG NÀO?"**

Có đúng **5 chiến lược (strategy)** kinh điển. Đây không phải 5 thứ rời rạc để học thuộc — chúng là **bộ từ vựng nền** mà cả phần còn lại của giáo trình sẽ dùng đi dùng lại. Học xong bài này bạn sẽ:

- **Mô tả được luồng đọc/ghi** của từng strategy (vẽ ra giấy được).
- Hiểu **vì sao `cache-aside` là mặc định công nghiệp** (*industry default*) — không phải vì nó "xịn nhất" mà vì một lý do sâu hơn.
- **Bắt được lỗi kinh điển:** dùng **`write-back`** cho **dữ liệu tiền** — một lỗi *kiến trúc*, không phải lỗi cú pháp.

> 💡 **Khung tư duy của cả bài:** 5 strategy chỉ khác nhau ở **đúng 2 câu hỏi**:
> 1. **AI chịu trách nhiệm** đọc/ghi giữa cache và DB? (app tự lo, hay cache layer lo hộ?)
> 2. Ghi **đồng bộ** (*synchronous* — chờ DB xong mới trả) hay **bất đồng bộ** (*asynchronous* — trả trước, ghi DB sau)?
>
> Nắm 2 trục này, bạn tự suy ra được mọi strategy mà không cần học vẹt.

---

## ② Giảng cơ bản → nâng cao

### (a) Trực giác — 5 strategy khác nhau ở chỗ nào?

Tưởng tượng bạn là **thủ thư** (app), có một **cuốn sổ tay trên bàn** (cache) và một **kho lưu trữ ở tầng hầm** (DB). Khi có người hỏi một thông tin, hoặc khi thông tin cần được cập nhật, bạn phải quyết định:

- **Ai đi xuống tầng hầm?** Bạn tự đi (*cache-aside*), hay bạn có một trợ lý tự động đi giúp (*read-through*)?
- **Khi cập nhật, ghi vào sổ tay rồi mới thủng thẳng xuống kho sau** (*write-back* — nhanh nhưng nếu sổ tay cháy thì mất), hay **ghi đồng thời cả hai cho chắc** (*write-through* — chậm hơn nhưng an toàn)?

Năm strategy chính là năm cách trả lời khác nhau cho mấy câu hỏi đó. Giờ đi vào từng cái.

---

### (b) Cơ chế từng strategy

#### 1️⃣ Cache-aside / Lazy loading *(J-STRAT-001)* — **phổ biến nhất**

**Đặc trưng:** **app tự quản lý** mọi thứ. **Cache hoàn toàn "ngây thơ"** — nó *không biết DB tồn tại*, chỉ là một cái kho key-value câm. App là người duy nhất nói chuyện với cả hai.

**Tên gọi `lazy loading` (nạp lười):** dữ liệu **chỉ được nạp vào cache KHI có người hỏi** (lần đầu). Không ai hỏi thì cache trống — "lười", không làm gì trước.

**Luồng ĐỌC:**
```
app → hỏi cache?
        ├── HIT  → trả luôn ✅ (nhanh)
        └── MISS → đọc DB → ghi (set) vào cache → trả ❌→✅
```

**Luồng GHI:**
```
app → update DB (ghi nguồn TRƯỚC)
    → DEL cache (xoá key, KHÔNG update)
```

**Hai nhược điểm cố hữu — phải hiểu rõ:**

- **Cold miss (miss lần đầu):** key chưa từng được hỏi → lần đầu **luôn miss** → phải đi DB. Không tránh được, chỉ giảm nhẹ bằng *warming* (mục d).
- **Stale window (cửa sổ cũ):** giữa lúc DB đã đổi và lúc cache được cập nhật lại, **cache có thể trả đồ cũ**. Độ dài cửa sổ này thường bị chặn bởi **TTL**.

> 🔍 **Vì sao GHI lại là `DEL` chứ không phải `update cache`?** (sẽ đào sâu ở Bài 4)
> Vì **update cache khi ghi dễ tạo race condition** (hai request đan xen nhau) dẫn tới một **bản stale "bền"** nằm lì trong cache. **Xoá (`DEL`) thì an toàn hơn**: lần đọc kế tiếp sẽ tự miss → tự nạp lại bản mới từ DB. Nhớ nguyên tắc: **"ghi DB trước, xoá cache sau"**.

---

#### 2️⃣ Read-through *(J-STRAT-002)*

**Giống `cache-aside` về kết quả, nhưng khác về AI làm việc.** Ở đây **cache layer (thư viện/provider) TỰ đi load từ DB khi miss**, app **chỉ nói chuyện với cache** và không bao giờ chạm trực tiếp vào DB cho đường đọc.

**Ẩn dụ:** thay vì bạn (thủ thư) tự đi tầng hầm, bạn có một **trợ lý tự động**: bạn chỉ hỏi trợ lý, nếu trợ lý không có thì *nó tự đi lấy* rồi đưa bạn.

**Luồng ĐỌC:**
```
app → cache (loader)
         ├── HIT  → trả
         └── MISS → cache TỰ đọc DB → tự set → trả
```

- ✅ **Code app gọn hơn** (logic miss-thì-load nằm trong cache layer, không lặp lại ở mọi nơi).
- ⚠️ **Đổi lại:** cần **provider / abstraction hỗ trợ "loader"** (không phải cache nào cũng có). App bị **gắn chặt hơn vào provider** đó.

---

#### 3️⃣ Write-through *(J-STRAT-003)*

**Đặc trưng:** mỗi lần ghi, **ghi ĐỒNG BỘ cả cache lẫn DB** (thường thông qua cache layer). "Đồng bộ" nghĩa là **chờ cả hai xong mới coi là ghi thành công**.

**Luồng GHI:**
```
app → cache layer → ghi cache  ┐  (cả hai phải xong
                  → ghi DB     ┘   thì write mới done)
```

- ✅ **Cache LUÔN tươi** (vừa ghi DB là cache đã có bản mới) → **đọc nhất quán hơn**, gần như không có stale window.
- ⚠️ **Trả giá 1 — write latency cao hơn:** mỗi write phải đợi *hai* nơi ghi xong.
- ⚠️ **Trả giá 2 — phí RAM:** cache chứa **cả những data có thể không bao giờ được đọc lại**. Bạn ghi gì là nó cache nấy, kể cả thứ chẳng ai hỏi tới.

> ⚠️ **Hiểu lầm cần dập:** *"Write-through thì nhất quán tuyệt đối."* **Không hẳn.** Trong môi trường **phân tán** (nhiều cache node, nhiều replica DB), vẫn có thể stale do **độ trễ replication** hoặc do **đường ghi khác** chạm DB không qua cache layer. (Bài 4 nói kỹ.)

---

#### 4️⃣ Write-back / Write-behind *(J-STRAT-004)* — ⚠️ **NGUY HIỂM**

**Đặc trưng:** ghi vào **cache TRƯỚC** rồi trả về ngay, còn việc **flush (đổ) xuống DB làm async / theo batch SAU**.

**Luồng GHI:**
```
app → ghi cache → TRẢ LUÔN ✅ (cực nhanh)
                    ⋮ (sau đó, async/batch)
              cache → flush xuống DB
```

- ✅ **Write cực nhanh** (không chờ DB).
- ✅ **Gộp ghi (batching):** 1000 lần tăng bộ đếm có thể gộp thành 1 lần ghi DB → giảm tải ghi khủng khiếp.
- ☠️ **RỦI RO CHÍ MẠNG — MẤT DATA:** nếu **cache chết (crash) TRƯỚC khi kịp flush**, phần dữ liệu đang nằm chờ trong cache **biến mất vĩnh viễn** — DB chưa bao giờ thấy nó.

> 🚨 **Quy tắc sống còn:** **`write-back` CHỈ dùng được khi bạn CHẤP NHẬN mất mát nhỏ.** Ví dụ hợp lệ: **đếm lượt xem (view count), metrics, counter** — mất vài bản ghi không ai chết. Để bớt rủi ro, kèm thêm **persistence / replication** cho cache.
> **TUYỆT ĐỐI KHÔNG** dùng cho **tiền, đơn hàng, tồn kho, giao dịch** — mất là không tái tạo được.

---

#### 5️⃣ Refresh-ahead *(J-STRAT-005)*

**Đặc trưng:** **chủ động làm mới (refresh) một key NÓNG TRƯỚC KHI nó hết hạn**, dựa trên dự đoán "key này sắp lại được đọc".

**Vấn đề nó giải:** với cache-aside thường, đúng **khoảnh khắc một hot key hết hạn (expire)**, request kế tiếp sẽ **miss và phải đi DB** → tạo một **latency spike** (gai trễ) ngay tại lúc đông người. **Refresh-ahead** làm mới key *trước* thời điểm đó, nên **không ai gặp miss lúc expiry**.

**Ẩn dụ:** quán cà phê thấy bình cà phê sắp cạn vào giờ cao điểm → **pha sẵn bình mới trước** thay vì đợi cạn hẳn rồi mới pha (lúc đó khách đang xếp hàng).

- ⚠️ **Hạn chế:** nếu đoán sai — **refresh một key thực ra không còn được đọc nữa** → **phí tài nguyên** (làm mới đồ chẳng ai cần).

---

### (c) So sánh & lựa chọn *(J-STRAT-006, 007)*

**Cuộc đấu kinh điển: `cache-aside` vs `write-through`.**

| Tiêu chí | **Cache-aside** | **Write-through** |
|---|---|---|
| **Độ tươi cho read** | Có **stale window** | **Tươi hơn** (gần như không stale) |
| **Write latency** | Bình thường (chỉ ghi DB) | **Cao hơn** (ghi cả 2) |
| **Cache chết thì sao?** | 😌 **Degrade** — chỉ tăng miss, vẫn **đọc thẳng DB** được | 😟 App **phụ thuộc cache nhiều hơn** |
| **Độ phức tạp** | ✅ Đơn giản | Cần **cache layer hỗ trợ** |

#### ⭐ Vì sao `cache-aside` là MẶC ĐỊNH CÔNG NGHIỆP? *(J-STRAT-007)*

Đây là câu hỏi phỏng vấn rất hay. Lý do **không phải** vì nó nhanh nhất hay tươi nhất, mà vì **3 tính chất**:

1. **Đơn giản (simple):** app tự quản, không cần cache layer thông minh, không cần loader đặc biệt.
2. **Resilient (chịu lỗi tốt) — quan trọng nhất:** khi **cache down**, hệ thống **chỉ degrade** (tăng miss, đọc chậm hơn vì đập thẳng DB) chứ **KHÔNG chết**. Cache là "lớp tăng tốc tuỳ chọn", không phải "đường sống còn".
3. **Không khoá vào provider (no vendor lock-in):** logic nằm ở app, đổi Redis sang Memcached hay gì khác đều dễ.

> 🎯 **Câu chốt:** Hai nhược điểm của cache-aside (**cold miss đầu** + **stale window**) là **chấp nhận được** miễn là đặt **TTL hợp lý**. Đổi lại bạn được sự **đơn giản + an toàn khi cache sập** — và trong sản xuất, **"cache sập mà app vẫn sống"** là tính chất đáng giá hơn vài phần trăm độ tươi.

---

### (d) Cold start & Cache warming *(J-STRAT-008)*

**Cold cache (cache nguội):** ngay sau khi **deploy / restart**, cache **rỗng trơn**. Mọi request đầu tiên đều **miss** → **một loạt request đồng loạt đập thẳng DB** → DB có thể bị **sốc tải** đúng lúc vừa khởi động.

**Cache warming (làm ấm cache):** **preload (nạp trước) các hot key** *trước khi* mở cửa nhận traffic — để khi traffic vào thì cache đã có sẵn đồ.

**Hạn chế của warming:**
- Bạn **không biết hết hot key** là những key nào (đoán có thể trật).
- Warming **không cứu được mass expiry về sau** — tức tình huống **nhiều key hết hạn cùng lúc** rồi cùng miss một loạt. Cái đó cần **TTL jitter** (rải lệch thời điểm hết hạn — Bài 5).

---

### (e) Cache CÁI GÌ: row / computed / fragment *(J-STRAT-009)*

Không chỉ chọn *cách* cache, còn phải chọn *tầng dữ liệu* nào để cache. Có một **trục đánh đổi** chạy từ thấp (thô) lên cao (gần kết quả cuối):

| Cache cái gì | Cắt được bao nhiêu compute | Khả năng tái dùng | Độ dễ vỡ khi 1 phần đổi |
|---|---|---|---|
| **Raw row** (hàng DB thô) | Ít — vẫn phải tính lại | ✅ Cao (nhiều nơi dùng chung) | 😌 Ít vỡ |
| **Computed** (kết quả đã tính) | Trung bình | Trung bình | Trung bình |
| **Rendered fragment** (mảnh HTML đã render) | ✅ Nhiều nhất | ❌ Thấp (rất riêng) | 😣 Dễ vỡ — 1 phần đổi là **invalidate khó**, lại hay **trùng lặp** |

> 🎯 **Nguyên tắc chọn:** **càng gần kết quả cuối** (fragment) thì **càng cắt nhiều compute** nhưng **càng dễ vỡ và khó invalidate**; **càng thô** (raw row) thì **tái dùng cao** nhưng **còn phải tính lại nhiều**. Chọn dựa trên: **chi phí tính-lại** vs **tần suất dữ liệu đổi**.

---

## ③ ⚠️ Kiến thức cũ / bị thay thế

| Cái CŨ | Thay bằng (hiện nay) | Vì sao |
|---|---|---|
| Dùng **`write-back`** "cho nhanh" ở path quan trọng | **`write-through`** / **ghi-DB-trước** cho path *correctness* | Write-back **mất data khi cache chết** |
| *"Write-through luôn nhất quán tuyệt đối"* | Hiểu rằng vẫn **stale trong môi trường phân tán** (nhiều node/replica) | Consistency còn phụ thuộc **replication** & **đường ghi khác** (Bài 4) |
| Cache-aside *"update cache khi write"* | **DEL cache khi write** | Update **dễ race → stale bền** (Bài 4) |

---

## ④ Áp dụng thực tế + So sánh bigtech

**Gán strategy theo use case:**

- **Feed / profile** (đọc nhiều, ghi ít) → **`cache-aside` + TTL**. Đây là 90% trường hợp.
- **Đếm lượt xem video** (chấp nhận lệch nhỏ, ghi cực nhiều) → **`write-back` / batch flush** xuống DB mỗi N giây. Mất vài count không sao, đổi lại giảm tải ghi khổng lồ.
- **Trang chủ hot** (key bị hỏi liên tục) → **`refresh-ahead`** để **không ai gặp miss lúc key hết hạn**.

**Pattern ở bigtech:** đa số **chọn `cache-aside` làm mặc định** chính vì tính **resilient**. **`write-back` chỉ xuất hiện ở metrics / counter** — nơi mất vài bản ghi không gây hậu quả.

---

## ⑤ Code thực hành + cấu hình

**Cache-aside chuẩn** (Node.js + ioredis), có **TTL** + **delete-on-write**:

```javascript
// productCache.js — cache-aside (lazy loading)
import Redis from "ioredis";
const redis = new Redis(process.env.REDIS_URL);
const TTL = 300; // giây — chặn độ dài stale window

// ===== ĐỌC: check → miss → DB → set =====
export async function getProduct(id, db) {
  const key = `product:${id}`;
  const cached = await redis.get(key);
  if (cached !== null) return JSON.parse(cached);     // ✅ HIT → trả luôn

  const row = await db.findProduct(id);               // ❌ MISS → đọc DB
  if (row) {
    // EX = TTL. Verify cú pháp SET tại redis.io
    await redis.set(key, JSON.stringify(row), "EX", TTL); // back-fill vào cache
  }
  return row;
}

// ===== GHI: DB trước → DEL cache (KHÔNG update) =====
export async function updateProduct(id, patch, db) {
  await db.updateProduct(id, patch);  // 1) ghi DB TRƯỚC
  await redis.del(`product:${id}`);   // 2) DELETE cache — Bài 4 giải thích vì sao KHÔNG update
}
```

**⚠️ Hai cảnh báo phải khắc cốt:**
- **Tuyệt đối KHÔNG dùng `write-back`** ở đây cho **dữ liệu cần đúng**.
- **Quên `TTL` = key sống mãi → memory phình** (memory leak kiểu cache — Bài 6). Luôn đặt TTL.

---

## ⑥ Keywords cần nhớ (glossary)

🧠 **Cần HIỂU:**
- **5 strategy & luồng đọc/ghi** của từng cái.
- **Vì sao `cache-aside` resilient** (cache chết → degrade, không chết).
- **Trade-off tươi ↔ latency**.
- **Rủi ro `write-back`** (mất data).
- **Cache `row` vs `computed` vs `fragment`**.

📌 **Cần THUỘC (luồng tóm tắt):**
- **`cache-aside` đọc** = `check → miss → DB → set`; **ghi** = `DB → DEL`.
- **`write-through`** = ghi cả 2 **đồng bộ**.
- **`write-back`** = **cache trước, flush sau**.

🛠️ **Cần LÀM ĐƯỢC:** code `cache-aside` có **TTL + delete-on-write**; **chọn strategy đúng** cho một use case; **nhận ra `write-back` đặt sai chỗ**.

---

## ⑦ Mental model

> **`cache-aside` là default** (app tự quản, cache chết thì **degrade** chứ không sập).
> **`write-through`** = **tươi nhưng chậm**.
> **`write-back`** = **nhanh nhưng có thể MẤT data** — **cấm dùng cho tiền**.

---

## ⑧ Câu hỏi phỏng vấn & thách đố

- **(dễ)** Mô tả luồng đọc/ghi `cache-aside`. → đọc `check → miss → DB → set`; ghi `DB → DEL`.
- **(TB)** `read-through` khác `cache-aside` ở đâu? → **cache layer tự load**, app chỉ gọi cache.
- **(TB)** `write-through` đánh đổi gì? → **tươi hơn** nhưng **write chậm** + cache cả data không đọc lại.
- **(khó)** `write-back` lợi / rủi ro chí mạng? → **nhanh + gộp ghi** / **mất data khi cache chết**.
- **(khó)** Vì sao `cache-aside` là mặc định công nghiệp dù có nhược điểm? → **đơn giản + resilient + không khoá provider**.

> **🧩 Thách đố cao cấp *(J-STRAT-010, ⭐⭐⭐⭐⭐)*:**
> *"AI sinh code dùng `write-back` cho dữ liệu thanh toán 'cho nhanh'. Vì sao đây là lỗi KIẾN TRÚC, sửa thế nào?"*
>
> **Trả lời:** `write-back` có thể **mất write khi cache chết trước flush** → **mất giao dịch tiền** — điều **không thể chấp nhận**. Path *correctness* (tiền, đơn hàng) **phải dùng `write-through` / ghi-DB-trước**. Cái lợi **"nhanh" KHÔNG bù nổi rủi ro mất tiền**. → Đây đúng là chỗ một kỹ sư phải **bắt được** khi review code (kể cả code do AI sinh).

---

## ⑨ Bài tập + tiêu chí tự chấm

**BT1.** Viết hàm `getUser` theo **`cache-aside` (TTL 60s)** + `updateUser` (**delete-on-write**).
> ✅ **Đạt khi:** miss thì **đọc DB rồi set**; update thì **ghi DB TRƯỚC rồi `DEL`** (không update cache).

**BT2.** Cho 4 use case (**giỏ hàng, số dư ví, đếm view, profile**), gán **strategy** + biện minh.
> ✅ **Đạt khi:** **ví = KHÔNG `write-back`** (đọc/ghi đúng, ưu tiên correctness); **view = `write-back` ok**; **profile = `cache-aside`**.

---

## ⑩ Đọc thêm

- **AWS Whitepaper** — *Database Caching Strategies Using Redis*, chương **Caching Patterns**.
- **redis.io/docs** — *Client-side caching*, *Caching patterns*.
- **microservices.io** — *Caching patterns* (aside vs through).

---

> 🔎 **AI hay sai ở đây:** AI **thích `write-back` / "ghi cache xong trả luôn"** vì nó **nhanh trong demo**.
> **Hãy luôn tự kiểm:** *Nếu Redis chết NGAY SAU đó, có mất dữ liệu không thể tái tạo không?*
> Nếu **có** → **CẤM** dùng `write-back` ở path đó.
