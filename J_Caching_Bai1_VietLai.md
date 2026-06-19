# 🎯 BÀI 1 — Vì sao cache · Các tầng cache · Khi nào KHÔNG cache
**Phủ:** J-WHY-001 → J-WHY-007

---

## ① Mục tiêu & vị trí trong mạch

Đây là **bài mở màn của Phase Caching**. Học xong bài này bạn sẽ:

- Hiểu **cache** thực sự giải **bài toán** gì (và *không* giải bài toán gì).
- Vẽ được **các tầng cache** (*cache layers*) mà một request web đi xuyên qua.
- Phân biệt được hai kiểu cache nền tảng: **local cache** (in-process) và **distributed cache** (Redis).
- Quan trọng nhất — và là phần dễ bị bỏ qua nhất — biết **khi nào KHÔNG nên cache**.

> 💡 **Ý chính xuyên suốt cả Phase:** Cache *không miễn phí*. Mỗi lần bạn thêm một bản sao dữ liệu, bạn vừa thêm **tốc độ**, vừa thêm **rủi ro sai (stale)** và **độ phức tạp vận hành**. Mọi *strategy* và *policy* ở các bài sau chỉ là những **cách quản lý ba thứ đánh đổi nhau**: **tốc độ ↔ độ tươi (freshness) ↔ bộ nhớ (memory)**. Bài 1 đặt nền cho cái khung đánh đổi đó.

---

## ② Giảng cơ bản → nâng cao

### (a) Trực giác — Cache là gì?

**Cache = giữ một bản sao của dữ liệu ở một chỗ truy cập NHANH HƠN nguồn gốc**, để lần sau khỏi phải tính lại hay đọc lại từ chỗ chậm.

**Ẩn dụ tờ giấy dán màn hình:** Bạn đang làm việc và cần tra một con số nằm trong một cuốn sách ở thư viện cuối hành lang. Lần đầu bạn buộc phải đi xuống thư viện (chậm). Nhưng khi đã có con số rồi, bạn **chép nó lên một tờ giấy dán cạnh màn hình**. Những lần sau, bạn chỉ liếc mắt là thấy — **không cần đi lại**.

- **Tờ giấy = cache.** Nó nhanh vì hai lý do: nó **ở gần** (không phải đi xa) và nó **đã sẵn** (không phải tính lại).
- **Thư viện = nguồn gốc** (*source of truth*): chậm hơn nhưng luôn đúng.

**Rủi ro ngay lập tức:** nếu ai đó sửa con số trong sách mà bạn không biết, **tờ giấy của bạn bị cũ** → đây chính là vấn đề **stale** (dữ liệu lệch so với nguồn) mà cả Phase Caching xoay quanh.

#### Tốc độ của cache đến từ ĐÂU?

Có đúng **hai nguồn tạo ra tốc độ**, cần thuộc lòng:

**1. Locality (tính cục bộ) — cache "đặt cược" vào đây.** Có hai dạng:

| Loại locality | Nghĩa | Ví dụ |
|---|---|---|
| **Temporal locality** (theo *thời gian*) | Thứ **vừa dùng** thì dễ **được dùng lại** ngay sau đó | Bạn vừa mở trang sản phẩm A → có thể bạn sẽ refresh / xem lại A trong vài giây tới |
| **Spatial locality** (theo *không gian*) | Thứ **nằm gần** thứ vừa dùng thì dễ được dùng tiếp | Vừa đọc dòng 1 của một mảng → khả năng cao sẽ đọc dòng 2, 3 |

> Nếu dữ liệu **không có locality** (mỗi lần truy cập một thứ hoàn toàn khác, chẳng cái nào lặp lại), thì cache **vô dụng** — vì cache sống nhờ "thứ này sẽ được hỏi lại".

**2. Tránh I/O / compute đắt.** Các tầng lưu trữ có tốc độ chênh nhau **nhiều bậc độ lớn** (*orders of magnitude*):

```
RAM         ~ nanosecond  ← cache thường ở đây
SSD/disk    ~ microsecond–millisecond
Network/DB  ~ millisecond+
LLM call    ~ trăm ms → vài giây   ← cực kỳ đáng cache
```

Cache "ăn" được chính khoảng chênh lệch này: đọc **RAM** thay vì gọi **DB / network / LLM**.

#### Cái giá phải trả

Đổi lại tốc độ, bạn **luôn** trả ba thứ:

1. **Thêm bộ nhớ** (cache chiếm RAM).
2. **Rủi ro stale** — bản sao có thể lệch nguồn.
3. **Thêm một thứ phải vận hành** — thêm component nghĩa là thêm chỗ có thể hỏng, thêm cấu hình, thêm bug.

> **Quy tắc vàng:** Cache là một **vụ trao đổi (trade-off)**, không phải "nút tăng tốc free". Nếu bạn không nói được mình *đang trả gì* để *được gì*, bạn chưa hiểu cache.

---

### (b) Cơ chế — Các tầng cache *(J-WHY-002)*

Một **request** đi từ trình duyệt người dùng xuống tận **database** sẽ **xuyên qua nhiều tầng cache xếp chồng**. Mỗi tầng là một cơ hội "trả lời sớm" để khỏi phải đi sâu hơn. Mỗi tầng đánh đổi khác nhau giữa **tốc độ ↔ phạm vi chia sẻ ↔ độ tươi**.

**Thứ tự PHẢI THUỘC (từ gần user nhất → gần DB nhất):**

```
①  Browser cache          → riêng từng user · nhanh nhất · DỄ stale nhất
        ↓
②  CDN / Edge cache        → đặt gần user theo ĐỊA LÝ · cắt latency vùng miền
        ↓
③  Reverse proxy           → Nginx / Varnish · cache cả response TRƯỚC khi vào app
        ↓
④  App in-process (L1)     → in-memory NGAY trong tiến trình · KHÔNG tốn network
        ↓
⑤  Distributed cache (L2)  → Redis / Memcached · CHIA SẺ giữa mọi instance
        ↓
⑥  DB buffer pool          → chính DB cũng cache "page" trong RAM
```

Giải thích từng tầng cho dễ nhớ:

- **① Browser cache** — bản sao nằm ngay trên máy người dùng (ảnh, CSS, JS, đôi khi cả response API). **Nhanh nhất** vì không đi đâu cả, nhưng **bạn khó kiểm soát nhất** → dễ phục vụ đồ cũ.
- **② CDN / Edge** — máy chủ đặt rải rác khắp thế giới (**Cloudflare, Akamai, CloudFront**). Người dùng ở Việt Nam được phục vụ từ node ở Việt Nam thay vì bay sang Mỹ → **cắt latency theo địa lý**.
- **③ Reverse proxy** — **Nginx / Varnish** đứng trước app, có thể **trả thẳng response đã cache** mà app không cần chạy lại logic.
- **④ App in-process (L1)** — cache nằm **bên trong chính tiến trình app** (ví dụ một `Map` / `LRUCache` trong RAM của process). **Không tốn một network hop nào** → cực nhanh. Nhược điểm: **mỗi instance một bản riêng**.
- **⑤ Distributed cache (L2)** — **Redis / Memcached**, một service riêng mà **mọi instance app đều nối tới**. Tốn 1 lần đi mạng (*network round-trip*) nhưng **mọi node thấy chung một sự thật**.
- **⑥ DB buffer pool** — đừng quên: **bản thân database cũng cache** các trang dữ liệu (*page*) nóng trong RAM. Nên đôi khi "đập thẳng DB" không chậm như bạn tưởng.

---

### (c) Local vs Distributed *(J-WHY-003)*

Đây là **hai loại cache nền tảng** ở tầng app, cần phân biệt rõ:

| Tiêu chí | **Local** (in-process / L1) | **Distributed** (Redis / L2) |
|---|---|---|
| **Tốc độ** | ⚡ Cực nhanh — **không có network hop** | Nhanh, nhưng tốn **1 network round-trip** |
| **Chia sẻ** | ❌ **Mỗi instance một bản** → dễ lệch nhau | ✅ **Mọi instance thấy chung** |
| **Invalidate đồng loạt** | 😣 Khó — phải đi báo **từng node** | 😌 Dễ — xoá ở **một nơi** là xong |
| **Điểm phụ thuộc** | Không thêm service phụ thuộc | Thêm một service **phải HA** (*high availability*) |

**Cách nhớ một câu:**
- **Local = nhanh nhất nhưng ích kỷ** (không share, khó báo nhau khi đổi).
- **Distributed = chậm hơn chút nhưng đoàn kết** (share chung, dễ xoá đồng loạt) — đổi lại bạn phải nuôi thêm một service không được phép chết.

---

### (d) Thực tế: Multi-tier (L1 + L2) *(J-WHY-004)*

Trong thực tế các hệ lớn **không chọn một trong hai** — họ **dùng cả hai chồng lên nhau (multi-tier)**:

- **L1 = local** cho **hot key** (vài key bị hỏi liên tục): cắt latency tối đa + **giảm tải cho L2**.
- **L2 = Redis** làm **nguồn chia sẻ** giữa mọi instance.

**Cách hoạt động (đọc):** hỏi **L1** trước → trượt thì hỏi **L2** → trượt nữa mới xuống **DB**, rồi **điền ngược (back-fill)** kết quả lên L2 và L1.

**Rủi ro cốt tử của multi-tier:** vì **L1 của mỗi node là độc lập**, khi dữ liệu ở **L2/DB thay đổi**, các bản L1 ở những node khác nhau **có thể stale khác nhau** (node A đã cập nhật, node B vẫn giữ đồ cũ). Hai cách xử lý phổ biến:

1. **TTL ngắn cho L1** (ví dụ vài giây) → giới hạn thời gian một bản có thể bị sai.
2. **Pub/sub broadcast invalidate** → khi dữ liệu đổi, **phát một thông báo** cho mọi node tự xoá L1.
3. Hoặc đơn giản là **chấp nhận một độ lệch nhỏ** — *nếu* path đó chịu được stale.

---

### (e) Khi nào KHÔNG cache *(J-WHY-005)* — ⭐ phần dễ bị bỏ qua nhất

> Đây là phần **TL;DR hay bỏ qua**, nhưng lại là thứ phân biệt người hiểu cache thật sự. **Cache có hại nhiều hơn lợi** trong các trường hợp sau:

**1. Data đổi liên tục, hoặc chỉ đọc một lần** → **hit ratio thấp**. Cache chỉ kịp lưu thì đã cũ, hoặc chẳng ai hỏi lại → bạn tốn RAM mà chẳng được gì, lại còn sinh **stale**.

**2. Path cần strong consistency (nhất quán mạnh) tuyệt đối.** Ví dụ kinh điển: **số dư tiền · tồn kho · hạn mức tín dụng**. Ở đây đọc một bản cache cũ = **ra quyết định sai** (bán hàng đã hết, tiêu quá hạn mức). **Đừng cache** những path này — hoặc tách phần public ra cache riêng.

**3. Write-heavy, ít đọc** (ghi nhiều hơn đọc). Mỗi lần ghi bạn phải **invalidate** cache, mà lại hiếm khi đọc trúng (*hit*) → công invalidate nhiều hơn lợi ích.

**4. Data rẻ để tính lại.** Nếu tính lại nhanh ngang việc đọc cache, thì cache **chẳng cứu được gì** — chỉ thêm phức tạp.

> 🔑 **Nguyên tắc:** **Chỉ cache khi ĐO ĐƯỢC lợi.** Cache "cho chắc" / "để nhanh hơn tí" là **cách phổ biến nhất để đẻ ra bug stale**.

---

### (f) Đo cache có ĐÁNG không *(J-WHY-006)*

Đừng tin cảm giác — hãy **đo**. Các **metric** cần nhìn:

- **Hit ratio** — tỉ lệ request được cache trả lời.
- **Latency p50 / p99** — độ trễ ở mức trung vị và mức "đuôi" (1% chậm nhất). **p99 thường là thứ người dùng cảm nhận đau nhất.**
- **Mức giảm tải nguồn** — ví dụ **DB QPS** (số query/giây xuống DB) giảm được bao nhiêu.
- **Cost / request** — chi phí mỗi request.

> ⚠️ **Bẫy hit ratio:** **Hit ratio cao CHƯA chắc tốt.** Nếu bạn toàn cache những thứ **rẻ / ít được gọi**, hit ratio nhìn rất đẹp nhưng **chẳng bảo vệ được gì**. Phải **gắn cache vào THỨ bạn muốn bảo vệ** — ví dụ "giữ DB khỏi sập" hay "kéo p99 xuống" — chứ **không tối ưu hit ratio một cách mù quáng**.

---

### (g) Cache vs Precompute / Materialized View *(J-WHY-007)*

Hai cách "có sẵn dữ liệu cho nhanh", nhưng triết lý ngược nhau:

| | **Cache** | **Materialized View / Precompute** |
|---|---|---|
| Kiểu | **Lazy** (lười) — tính khi *có người hỏi* | **Eager** (chủ động) — tính sẵn *trước* |
| Có thể trượt? | ✅ Có thể **miss** (lần đầu phải đi nguồn) | ❌ Luôn có sẵn |
| Chi phí | Trả khi miss | Trả ở khâu **ghi / refresh** liên tục |

**Chọn thế nào?** Tuỳ **độ tươi cần** + **pattern đọc**: cần luôn-có-sẵn và đọc nhiều kiểu tổng hợp → nghiêng về **MV**; đọc thưa, chấp nhận lần đầu chậm → **cache**. *(Cross-ref mục E: MV vs cột denormalized vs cache.)*

---

## ③ ⚠️ Kiến thức cũ / bị thay thế

| Cái CŨ (trong docs / quan niệm phổ biến) | Thay bằng (hiện nay) | Vì sao |
|---|---|---|
| *"Cache cho mọi thứ để nhanh"* | **Cache có chọn lọc**, đo **hit ratio** + **tải nguồn** | Cache bừa → **bug stale** + tốn RAM, không bảo vệ đúng chỗ |
| *"Tối ưu hit ratio là mục tiêu"* | Tối ưu **thứ cần bảo vệ** (**DB QPS**, **p99**) | Hit ratio cao trên thứ rẻ là **vô nghĩa** |
| *"Local cache là đủ cho cluster"* | **Multi-tier L1 + L2**, hiểu rõ local **không share** | Mỗi node một bản → **lệch giữa các instance** |

---

## ④ Áp dụng thực tế + So sánh bigtech

**Use case: trang sản phẩm e-commerce.** Cùng một trang nhưng mỗi field có chính sách cache khác nhau:

- **Thông tin sản phẩm** (tên, mô tả — đổi vài lần/ngày) → **cache mạnh**, TTL dài.
- **Tồn kho realtime** → **KHÔNG cache** (hoặc **TTL cực ngắn**); khi **checkout** thì đọc thẳng DB. Lý do: cần **strong consistency** — bán đồ đã hết là thảm hoạ.
- **Giá sau khuyến mãi cá nhân hoá** → cache được, nhưng **key phải gồm `user`** (vì mỗi user một giá), nếu không bạn sẽ phục vụ nhầm giá của người khác.

**Pattern ở bigtech** *(phổ biến, không tuyệt đối — verify khi cần số liệu cụ thể):*

- Mọi hệ thống quy mô lớn đều **multi-tier**: **browser → CDN (Cloudflare/Akamai/CloudFront) → app cache → Redis/Memcached**.
- **Facebook** nổi tiếng dùng **Memcached** ở quy mô khổng lồ, cộng thêm tier riêng (**TAO** cho *social graph*).
- **Netflix** dùng **EVCache** (một lớp xây trên Memcached).
- **Điểm chung quan trọng:** họ **đo**, và **chấp nhận stale ở những chỗ rẻ**, nhưng **không cache ở path tiền / quyền**.

---

## ⑤ Code thực hành + cấu hình

Minh hoạ **multi-tier**: **L1 in-process (LRU)** + **L2 Redis** trong **Node.js**.

```javascript
// cache.js — L1 (in-process, TTL ngắn) + L2 (Redis, share)
// Pin version: ioredis@5, lru-cache@11 (verify bản mới nhất khi học)
import Redis from "ioredis";
import { LRUCache } from "lru-cache";

const redis = new Redis(process.env.REDIS_URL);        // KHÔNG hardcode URL/secret
const L1 = new LRUCache({ max: 10_000, ttl: 5_000 });  // L1 chỉ 5s → giảm rủi ro stale giữa các node

export async function getProduct(id, loadFromDb) {
  const key = `product:${id}`;

  // 1) L1 — nhanh nhất, KHÔNG network
  const l1 = L1.get(key);
  if (l1 !== undefined) return l1;

  // 2) L2 — Redis, SHARE giữa các instance
  const l2 = await redis.get(key);
  if (l2 !== null) {
    const val = JSON.parse(l2);
    L1.set(key, val);          // back-fill lên L1
    return val;
  }

  // 3) Miss CẢ HAI → xuống nguồn (DB), rồi điền ngược lên L2 và L1
  const val = await loadFromDb(id);
  if (val != null) {
    await redis.set(key, JSON.stringify(val), "EX", 300); // L2 TTL 5 phút (verify cú pháp SET..EX)
    L1.set(key, val);
  }
  return val;
}
```

**Cấu hình & cảnh báo:**
- **`REDIS_URL` qua `.env`** — **không commit secret** vào code.
- Pin version trong `package.json`: **`ioredis@5`**.
- ⚠️ **Đừng để L1 TTL dài** — TTL dài = các node giữ bản cũ lâu = **lệch nhau giữa các instance**.

---

## ⑥ Keywords cần nhớ (glossary)

🧠 **Cần HIỂU:** **locality** (*temporal* / *spatial*), **các tầng cache**, **đánh đổi tốc độ ↔ tươi ↔ RAM**, **multi-tier L1/L2**, **khi nào không cache**, **gắn hit ratio vào "thứ cần bảo vệ"**.

📌 **Cần THUỘC:**
- **Thứ tự tầng cache:** `browser → CDN → proxy → L1 → L2 → DB buffer`.
- **4 trường hợp KHÔNG nên cache.**
- **`cache = lazy`** còn **`MV = eager`**.

🛠️ **Cần LÀM ĐƯỢC:** vẽ **sơ đồ tầng cache** cho một request; quyết định **một field nên / không nên cache**; **đo hit ratio + DB QPS** để biện minh.

---

## ⑦ Mental model

> **Cache = một bản sao GẦN & NHANH, đặt cược vào locality, trả giá bằng RAM + rủi ro stale.**
> **Đừng cache** thứ **rẻ** / **đổi-liên-tục** / **cần-đúng-tuyệt-đối**.

---

## ⑧ Câu hỏi phỏng vấn & thách đố

- **(dễ)** Cache giải bài toán gì, tốc độ đến từ đâu? → **locality** + **tránh I/O**; đổi bằng **RAM** + **stale**.
- **(TB)** Kể các tầng cache trong một web request. → `browser → CDN → proxy → L1 → L2 → DB buffer`.
- **(TB)** Local vs distributed khác gì, khi nào dùng cái nào? → **local** nhanh nhưng *không share*; **distributed** share nhưng *+network*; thực tế dùng **multi-tier**.
- **(khó)** Hit ratio 99% có chắc cache tốt? → **Không**, nếu cache thứ rẻ / ít gọi; phải gắn vào **DB QPS / p99**.
- **(khó)** Cho 3 trường hợp KHÔNG nên cache. → **tồn kho / tiền** (cần strong consistency), **data đổi liên tục**, **đọc-một-lần**.

> **🧩 Thách đố:** *"Team thêm Redis cache trước DB, p99 vẫn không cải thiện — vì sao?"*
> Các khả năng: **hit ratio thấp** (data đổi liên tục / đọc-một-lần); hoặc **nút thắt không nằm ở DB** mà ở **compute / serialize / network**; hoặc **cache miss vẫn đập DB đúng ở p99**. → **Phải ĐO trước khi thêm tầng.**

---

## ⑨ Bài tập + tiêu chí tự chấm

**BT1.** Vẽ **sơ đồ tầng cache** cho trang chi tiết sản phẩm; với mỗi field (**tên SP**, **tồn kho**, **giá KM theo user**) ghi cache ở **tầng nào + TTL**.
> ✅ **Đạt khi:** tồn kho **không cache** (hoặc TTL cực ngắn); giá-theo-user **có `user` trong key**; tên SP **cache dài**.

**BT2.** Cho 5 loại data, phân loại **"nên cache / không nên"** + lý do.
> ✅ **Đạt khi:** phân loại đúng theo tiêu chí **hit ratio** + **yêu cầu consistency**, **không cache bừa**.

---

## ⑩ Đọc thêm

- **redis.io/docs** — *Introduction / Use cases*.
- **AWS Whitepaper** — *Database Caching Strategies Using Redis* (phần tầng cache).
- Bài *"Scaling Memcached at Facebook"* (khái niệm **multi-tier** ở quy mô lớn).

---

> 🔎 **AI hay sai ở đây:** AI mặc định **"thêm cache = nhanh hơn"** và bê **Redis** vào **mọi path**, kể cả **tồn kho / tiền**.
> **Hãy luôn tự hỏi:** *Path này có chịu được stale không?* Nếu **không** → **đừng cache**, hoặc **tách phần public** (cache được) **khỏi phần cần đúng tuyệt đối**.
