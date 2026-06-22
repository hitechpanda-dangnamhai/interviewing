# 🥶 CHUYÊN ĐỀ — COLD START (Khởi động lạnh) trong Caching
> Đào sâu một keyword mở rộng của **Bài 1 — Vì sao cache · Các tầng cache · Khi nào KHÔNG cache**
> Mục tiêu: hiểu *bản chất*, *nhận diện*, *đo*, và *xử lý* cold start ở mức **staff engineer**.
> (✅ = đã có trong Bài 1 · ➕ = mở rộng thêm)

---

## ① Mục tiêu & vị trí trong mạch

Trong Bài 1, **Cold start** mới chỉ xuất hiện ở **bảng keyword mở rộng (A2)** với một câu rất ngắn:

> ➕ **Cold start** — *Cache rỗng lúc khởi động, miss nhiều* — *"Sau deploy, cache **cold** nên p99 tăng vọt vài phút đầu."*

Chuyên đề này "mở hộp" câu đó. Học xong bạn sẽ:

- Hiểu **chính xác cold start là gì**, và phân biệt hai biến thể dễ lẫn (cache cold start vs serverless cold start).
- Nhìn ra **cơ chế tổn thương**: vì sao cache rỗng không chỉ làm *chậm* mà còn có thể làm *SẬP* nguồn (cascading failure).
- Nắm **thang giải pháp** từ cơ bản → staff/principal: warming, rolling restart, readiness gating, load shedding, request coalescing, probabilistic early expiration…
- Kết nối cold start với **stampede, thrashing, working set, p99, capacity planning** — để không học rời rạc.

> 💡 **Một câu để gắn vào mạch Bài 1:** Cold start chính là lúc bạn **trả trước cái giá của cache** — toàn bộ "thuế miss" dồn vào vài phút đầu. Nó là **mặt tối của locality**: khi cache còn rỗng, *chưa có gì để đặt cược*, nên mọi request đều rơi xuống nguồn.

---

## ② Trực giác — Cold start là gì?

**Cold start (khởi động lạnh) = trạng thái cache đang RỖNG (hoặc gần rỗng), nên gần như mọi request đều MISS và phải rơi xuống nguồn gốc (DB / service / LLM).**

Quay lại **ẩn dụ tờ giấy dán màn hình** ở Bài 1:

- Cache "ấm" (*warm*) = tờ giấy đã chi chít các con số bạn hay tra → liếc mắt là có.
- Cache "lạnh" (*cold*) = **tờ giấy trắng tinh**. Mọi con số đều phải **đi xuống thư viện** (nguồn) lấy lần đầu.

Cold start **không phải là một bug** — nó là một **giai đoạn (phase)** mà *mọi* cache đều phải đi qua: lúc mới sinh ra, nó chưa biết gì cả. Vấn đề không phải "có cold start hay không" (chắc chắn có), mà là:

> **Hệ của bạn có SỐNG SÓT qua giai đoạn lạnh đó không?**

Có hai mức độ hậu quả, cần tách bạch ngay từ đầu:

| Mức | Tên gọi | Biểu hiện | Mức nguy hiểm |
|---|---|---|---|
| **Nhẹ** | *Latency degradation* | p99 tăng vọt vài phút, user thấy chậm rồi tự hồi | 🟡 Khó chịu nhưng không chết |
| **Nặng** | *Cascading failure* | Miss đồng loạt → nguồn quá tải → **nguồn sập** → cache không bao giờ kịp ấm → **outage kéo dài** | 🔴 Mất dịch vụ |

Sự khác biệt giữa hai mức này là **trọng tâm của toàn bộ chuyên đề**. Mức nặng xảy ra khi **cache là load-bearing** (chịu lực) chứ không chỉ là "tối ưu cho nhanh" — sẽ nói kỹ ở mục ④.

---

## ③ Cold start xảy ra KHI NÀO? (các nguồn gốc)

Cache trở nên "lạnh" trong nhiều tình huống. Liệt kê theo **tầng cache của Bài 1** (`browser → CDN → proxy → L1 → L2 → DB buffer`):

| # | Sự kiện | Tầng bị ảnh hưởng | Vì sao lạnh |
|---|---|---|---|
| 1 | **Deploy / restart app** | L1 (in-process) | Process bị giết → toàn bộ RAM của process **bay sạch**. L1 luôn lạnh sau mỗi deploy. |
| 2 | **Scale-out (thêm instance mới)** | L1 của node mới | Node mới sinh ra với L1 trắng tinh, nhưng **đã nhận traffic** từ load balancer. |
| 3 | **Restart / failover Redis** | L2 (distributed) | Nếu Redis không bật persistence (hoặc dữ liệu chưa kịp ghi đĩa) → restart xong là **rỗng**. |
| 4 | **Flush cache thủ công** | L2 | Một lệnh `FLUSHALL` (vô tình hoặc để "fix" bug) → xoá sạch trong 1 giây. **Kinh điển gây outage.** |
| 5 | **TTL hết hạn đồng loạt** | bất kỳ | Hàng loạt key được nạp cùng lúc → cùng TTL → **cùng hết hạn** → cold cục bộ theo đợt (*synchronized expiry*). |
| 6 | **Cache eviction ồ ạt** | L1/L2 | Cache đầy, **eviction** đẩy ra nhanh hơn nạp vào → working set không nằm trong cache nữa. |
| 7 | **Region/cluster mới** | toàn cụm | Bật một cụm cache mới ở region mới → cả cụm lạnh (bài toán "cold cluster" của Facebook). |
| 8 | **CDN cache mới / purge** | CDN/Edge | Purge toàn bộ CDN, hoặc một edge node mới → miss dồn về origin. |
| 9 | **Schema/key đổi version** | bất kỳ | Đổi format key (`product:v1` → `product:v2`) → **tất cả key cũ vô dụng** = cold trá hình. |

> 🔑 **Insight #1:** L1 (in-process) **lạnh sau MỌI lần deploy** — không thể tránh, chỉ có thể *làm ấm nhanh* hoặc *chịu được lúc lạnh*. L2 (Redis) thì **có thể giữ ấm qua restart** nếu cấu hình persistence đúng. Đây là một lý do nữa để dùng **multi-tier**: L2 ấm "đỡ" cho L1 lạnh.

> 🔑 **Insight #2:** Nguy hiểm nhất không phải lúc cache lạnh *từ từ*, mà lúc nó **lạnh ĐỘT NGỘT và ĐỒNG LOẠT** — flush, synchronized expiry, restart cả cụm. Lạnh đột ngột = miss đồng loạt = nguồn ăn nguyên cú đấm.

---

## ④ Cold start GÂY HẠI thế nào? (cơ chế tổn thương)

Đây là phần quan trọng nhất. Hãy đi từ số liệu.

### (a) Toán đơn giản về "thuế miss"

Giả sử ở trạng thái ấm (steady-state):

- Tổng traffic: **10.000 req/s**.
- **Hit ratio: 95%** → DB chỉ nhận **5% × 10.000 = 500 QPS**. DB của bạn được sizing cho ~500–1.000 QPS. Mọi thứ êm.

Bây giờ cache **lạnh đột ngột** (flush / restart):

- Hit ratio rơi tạm thời về **~0%** → DB phải nhận **gần như 10.000 QPS**.
- Đó là **20× tải bình thường**.

> ❗ Nếu DB chỉ chịu được 1.000 QPS, thì **20× = sập**. Và đây là điểm chí mạng:

### (b) Vòng xoáy tử thần (death spiral)

```
Cache lạnh
   ↓
Miss đồng loạt → DB QPS tăng 20×
   ↓
DB quá tải → query chậm dần (p99 DB tăng)
   ↓
Request giữ kết nối lâu hơn → connection pool cạn
   ↓
App timeout → user retry → THÊM tải
   ↓
DB càng chậm → cache CÀNG không kịp được back-fill (vì back-fill cần DB trả lời)
   ↓
Cache mãi mãi không ấm lên → OUTAGE kéo dài
```

> 🔥 **Đây là lý do cold start nguy hiểm hơn nó *trông có vẻ*:** nó không tự hồi nếu nguồn đã sập. Cache cần nguồn để ấm lên, mà nguồn lại đang chết *vì* cache lạnh. Hệ kẹt trong vòng lặp.

### (c) Khái niệm then chốt: Cache "tối ưu" vs cache "chịu lực" (load-bearing)

| | **Cache tối ưu (optimization)** | **Cache chịu lực (load-bearing)** |
|---|---|---|
| Nếu cache biến mất | Nguồn **vẫn sống**, chỉ chậm hơn | Nguồn **SẬP** — không gánh nổi traffic thật |
| Cold start gây | p99 tăng tạm rồi hồi | Outage |
| Bạn được phép | Coi nhẹ warming | **Bắt buộc** có chiến lược cold start |

> 🔑 **Insight #3 (câu hỏi vàng của staff engineer):**
> **"Nếu toàn bộ cache biến mất ngay bây giờ, nguồn của tôi có sống không?"**
> - Nếu **CÓ** → cache của bạn là *optimization*, cold start chỉ là khó chịu.
> - Nếu **KHÔNG** → cache của bạn là *load-bearing*, và **cold start là một sự cố sản xuất chờ xảy ra**. Bạn phải thiết kế cho nó.

### (d) Vì sao p99 đau hơn p50

Nhắc lại từ Bài 1: **p99 là thứ user cảm nhận đau nhất.** Trong cold start:

- p50 có thể trông "ổn" (đa số request vẫn nhanh nếu một phần key đã ấm).
- Nhưng **p99/p999 nổ tung**: 1% request xui xẻo rơi đúng key lạnh + DB đang nghẽn → chờ vài giây hoặc timeout.
- Nếu một trang gọi 50 micro-services, chỉ cần **1 trong 50** dính p99 → **cả trang chậm**. Đây là hiệu ứng "tail amplification" — p99 của từng service trở thành p50 của trang.

---

## ⑤ Phân biệt với các khái niệm "anh em"

Cold start hay bị lẫn với mấy thứ trong bảng A2/C4 của Bài 1. Tách rõ:

| Khái niệm | Bản chất | Khác cold start ở chỗ |
|---|---|---|
| ➕ **Cold start** | Cache **rỗng từ đầu** → miss vì *chưa từng có* | Trạng thái *khởi đầu* |
| ➕ **Cache stampede / thundering herd** | Nhiều request **cùng miss MỘT key** một lúc, đồng loạt đập nguồn | Có thể xảy ra cả khi cache đã ấm (1 key hot vừa hết hạn). Cold start làm stampede *tệ hơn* vì miss khắp nơi |
| ➕ **Thrashing** | Cache **liên tục evict rồi nạp lại**, không bao giờ ấm ổn định | Cold start *qua đi*; thrashing *kéo dài mãi* vì cache quá nhỏ so với working set |
| ➕ **Cold cluster** | Cả một **cụm cache** mới lạnh | Là cold start ở quy mô *cụm*, không phải 1 node |
| ✅ **Cache warming** | **Giải pháp** chủ động làm ấm cache trước | Warming là *thuốc*, cold start là *bệnh* |

> 🧠 **Cách nhớ:**
> - **Cold start** = "cache mới sinh, chưa biết gì".
> - **Stampede** = "một key hot vừa chết, cả đám lao xuống nguồn cùng lúc".
> - **Thrashing** = "cache quá bé, nạp vào lại bị đẩy ra ngay, không đời nào ấm".
>
> Ba thứ này **chồng lấn**: một cold start tệ thường **kéo theo stampede**, và nếu cache lại còn quá nhỏ thì **mãi thrashing không bao giờ thoát cold**.

---

## ⑥ Ví dụ minh hoạ

### Ví dụ 1 — Deploy buổi trưa (kinh điển)

Bạn deploy lúc 12h trưa (giờ cao điểm). Rolling restart **tất cả 20 instance cùng lúc** để "cho nhanh".

- 20 process mới → 20 L1 cache **trắng tinh**.
- Traffic giờ cao điểm vẫn 10.000 req/s đổ vào.
- L1 miss 100% → dồn xuống L2 (Redis). Nếu Redis cũng vừa restart cùng đợt → dồn thẳng xuống DB.
- DB QPS nhảy từ 500 → ~10.000. p99 từ 80ms → 4s. Một số request timeout, user F5 → retry storm.

**Bài học:** *thời điểm* + *đồng loạt* là kẻ thù. (Xem giải pháp: rolling/staggered restart, deploy giờ thấp điểm.)

### Ví dụ 2 — Synchronized expiry (TTL trùng nhau)

Lúc khởi động, code nạp 10.000 sản phẩm vào cache, **tất cả cùng `TTL = 300s`** trong cùng một vòng lặp.

```
t=0s    : nạp 10.000 key, mỗi key EX 300
t=300s  : 10.000 key CÙNG hết hạn trong cùng một khoảnh khắc
        → 10.000 miss đồng loạt → DB ăn cú đấm y như cold start
        → cứ mỗi 300s lại "lạnh lại" một lần
```

Đây là **cold start tái diễn theo chu kỳ** — và là lý do của kỹ thuật **jitter** (mục ⑧).

### Ví dụ 3 — Cold start *ẩn* khi đổi version key

```js
// Trước:  key = `product:${id}`
// Sau (đổi schema): key = `product:v2:${id}`
```

Deploy cái này lên → **tất cả key `product:*` cũ trong Redis trở nên vô dụng** (không ai đọc nữa, chờ TTL chết). Cache *trông* đầy nhưng **hit ratio rơi về 0** với traffic mới = cold start trá hình, nhưng nhìn dashboard Redis memory vẫn thấy "đầy" nên dễ bị bỏ sót.

### Ví dụ 4 — Code minh hoạ vòng xoáy + bounded warming

Phiên bản **NGÂY THƠ** (gây death spiral khi cold):

```js
// ❌ Mỗi miss tự do đập DB — cold start = 10.000 query song song
export async function getProduct(id, loadFromDb) {
  const key = `product:${id}`;
  const cached = await redis.get(key);
  if (cached !== null) return JSON.parse(cached);

  const val = await loadFromDb(id);          // không giới hạn concurrency!
  if (val != null) await redis.set(key, JSON.stringify(val), "EX", 300);
  return val;
}
```

Phiên bản **CÓ PHÒNG THỦ** (singleflight + jitter — chi tiết ở mục ⑧/⑨):

```js
import pLimit from "p-limit";              // giới hạn concurrency xuống nguồn
const dbLimit = pLimit(50);                // tối đa 50 query DB đồng thời
const inflight = new Map();                // request coalescing (singleflight)

function jitter(base) {                    // TTL = base ± 10% → tránh synchronized expiry
  return base + Math.floor(base * (Math.random() * 0.2 - 0.1));
}

export async function getProduct(id, loadFromDb) {
  const key = `product:${id}`;
  const cached = await redis.get(key);
  if (cached !== null) return JSON.parse(cached);

  // Gộp mọi request cùng key đang miss thành MỘT lần đi nguồn
  if (inflight.has(key)) return inflight.get(key);

  const p = dbLimit(() => loadFromDb(id))    // chặn không quá 50 query song song
    .then(async (val) => {
      if (val != null) {
        await redis.set(key, JSON.stringify(val), "EX", jitter(300));
      }
      return val;
    })
    .finally(() => inflight.delete(key));

  inflight.set(key, p);
  return p;
}
```

> Hai dòng "bùa hộ mệnh" cho cold start: **`pLimit` (chặn concurrency xuống DB)** + **`inflight Map` (singleflight, gộp miss trùng key)** + **`jitter` (TTL lệch nhau)**.

---

## ⑦ Case study thực tế

> ⚠️ Các case dưới đây tóm tắt theo tài liệu/bài báo công khai và pattern phổ biến trong ngành. Khi cần số liệu chính xác để trích dẫn, hãy verify lại nguồn gốc.

### CS1 — Facebook: "Cold Cluster Warmup" (Scaling Memcached at Facebook, NSDI 2013)

Khi Facebook bật **một cụm Memcached mới** (cold cluster), hit ratio ban đầu rất thấp → nếu để client đọc thẳng DB sẽ **đè chết DB**. Giải pháp họ mô tả: trong giai đoạn làm ấm, **client của cụm lạnh được cấu hình để đọc dữ liệu từ một cụm ĐÃ ẤM khác** thay vì từ DB. Cụm lạnh dần dần được lấp đầy từ cụm ấm, **không bắt DB gánh cú sốc cold start**.

> 🎯 **Bài học rút ra:** Khi có **nhiều bản cache**, hãy để bản lạnh **học từ bản ấm**, đừng để nó "tự học từ nguồn gốc đắt đỏ". Đây chính là tinh thần multi-tier: *L1 lạnh đọc L2 ấm thay vì đọc DB*.

### CS2 — Pattern "flush gây outage" (rất nhiều postmortem trong ngành)

Mô típ lặp đi lặp lại trong các báo cáo sự cố công khai: một kỹ sư chạy `FLUSHALL` / xoá namespace cache để "fix" một bug stale, hoặc một lần restart Redis không có persistence. Cache rỗng → DB nhận full traffic → DB sập → outage hàng chục phút đến hàng giờ vì **không ai làm ấm lại cache kịp khi DB đang chết**.

> 🎯 **Bài học:** **`FLUSHALL` trên production là vũ khí huỷ diệt hàng loạt.** Cần (1) rename/disable lệnh nguy hiểm trong Redis config, (2) có **load shedding** để bảo vệ DB, (3) coi "mất cache" là một kịch bản phải diễn tập (game day).

### CS3 — CDN cold edge & origin shielding (Cloudflare/Fastly/Akamai pattern)

Khi một edge node CDN mới hoặc vừa bị purge, mọi request là MISS và dồn về **origin**. Nếu hàng trăm edge node cùng miss một asset hot → origin bị đập. Giải pháp ngành: **origin shield / tiered caching** — đặt một **tầng cache trung gian** giữa các edge và origin, để chỉ tầng đó (chứ không phải hàng trăm edge) đi hỏi origin. (Đây lại là *multi-tier* + *request coalescing* ở quy mô CDN.)

> 🎯 **Bài học:** Cùng một nguyên lý lặp lại ở mọi tầng — **đặt một lớp đệm giữa "đám lạnh" và "nguồn đắt", và gộp các miss lại**.

### CS4 — Serverless cold start (AWS Lambda) — *biến thể KHÁC*, để phân biệt

Khi một function serverless lâu không gọi → môi trường bị thu hồi → lần gọi tiếp theo phải **khởi tạo lại runtime, nạp code, mở kết nối** → độ trễ thêm vài trăm ms đến vài giây. Người ta gọi đây cũng là "cold start", nhưng nó là về **khởi tạo môi trường thực thi**, không phải cache rỗng.

> 🎯 **Vì sao đưa vào đây:** Để bạn **không nhầm hai khái niệm cùng tên**. Trong phỏng vấn, hỏi "cold start" mà không rõ ngữ cảnh thì nên hỏi lại: *"cold start của cache, hay của serverless/runtime?"* — chúng có giải pháp hoàn toàn khác (mục ⑩).

---

## ⑧ Giải pháp — từ CƠ BẢN → nền tảng

Sắp theo độ "trưởng thành". Một hệ tốt thường **kết hợp nhiều cái**, không chỉ một.

### G1. ✅ Cache warming (làm ấm chủ động) — *đã có ở A2 Bài 1*

Nạp sẵn các key **hot/known** vào cache **trước khi mở traffic**.

- **Warm theo danh sách hot key:** giữ một danh sách "top N sản phẩm/khoá hay hỏi nhất" (từ analytics), khởi động xong thì nạp trước.
- **Warm trước sự kiện:** trước flash sale / ra mắt → nạp trước toàn bộ catalog liên quan (Bài 1 A2 đã nhắc: *"Trước flash sale, warm cache để tránh miss hàng loạt"*).
- **Đánh đổi:** warm tốn thời gian và tải nguồn lúc khởi động — nhưng **có kiểm soát** (bạn chọn lúc thấp điểm) thay vì bị động lúc cao điểm.

### G2. ➕ Staggered / rolling restart (khởi động lệch nhau)

**Đừng restart tất cả node cùng lúc.** Restart từng phần (ví dụ 10% một đợt), chờ node ấm rồi mới sang đợt sau.

- Giữ phần lớn cluster vẫn **ấm** trong khi vài node làm ấm.
- DB chỉ chịu "thuế miss" của 10% capacity tại một thời điểm, không phải 100%.

### G3. ➕ Readiness gating (cổng "sẵn sàng")

**Không cho load balancer gửi traffic vào node cho tới khi nó đã ấm.**

- Health check kiểu `/ready` chỉ trả OK khi L1 đã warm tới ngưỡng (ví dụ đã nạp top-1000 key).
- Tránh kịch bản ở Ví dụ 1: node mới *vừa sinh đã ăn full traffic* khi còn lạnh.
- Kubernetes: dùng **readinessProbe** tách khỏi **livenessProbe** đúng cho việc này.

### G4. ➕ Bật persistence cho L2 (Redis RDB/AOF)

Để Redis **không lạnh sau restart**:

- **RDB** (snapshot định kỳ) + **AOF** (ghi từng lệnh) → restart xong **nạp lại từ đĩa** = ấm sẵn.
- Hoặc dùng **replica**: failover sang replica đã ấm thay vì khởi động một node trống.

> ⚠️ Persistence cứu được **L2**, **không** cứu được **L1** (L1 nằm trong RAM process, chết theo process). Vẫn cần G1–G3 cho L1.

### G5. ➕ Jitter cho TTL (chống synchronized expiry)

Đã thấy ở Ví dụ 2 và code mục ⑥: thêm nhiễu ngẫu nhiên vào TTL (`300s ± 10%`) → key **không hết hạn cùng lúc** → không "lạnh lại" thành đợt.

### G6. ➕ Request coalescing / Singleflight

Khi nhiều request cùng miss **một key**, chỉ cho **một** request đi nguồn, số còn lại **chờ kết quả chung**. (Đã có trong code mục ⑥ qua `inflight Map`.) Đây là vũ khí chính chống **stampede** đi kèm cold start.

### G7. ➕ Đừng cache những thứ Bài 1 đã bảo đừng cache

Cold start "đau" tỉ lệ thuận với **lượng traffic phụ thuộc cache**. Nếu bạn tuân thủ **4 trường hợp KHÔNG nên cache** (D2 Bài 1), bạn đã **giảm bề mặt cold start** từ đầu.

---

## ⑨ Giải pháp NÂNG CAO (staff / principal)

Đây là phần phân biệt người *vận hành cache* với người *thiết kế hệ chịu lỗi*.

### A1. Capacity planning cho kịch bản "cache biến mất"

Trở lại Insight #3. Staff engineer **sizing nguồn với giả định cache có thể mất**:

- Hoặc DB đủ khoẻ để gánh **một phần đáng kể** traffic trần trụi (cache là *optimization*).
- Hoặc — nếu không thể — phải có **cơ chế bảo vệ chủ động** (load shedding, A2) để DB *không bao giờ* nhận quá ngưỡng, kể cả khi cache rỗng.
- Tài liệu hoá rõ: *"DB chịu được X QPS. Steady-state là 500. Cold start có thể đẩy lên 10.000 → vượt X → phải có cơ chế chặn ở Y."*

### A2. Load shedding & backpressure (xả tải có chủ đích)

Khi cache lạnh và nguồn sắp quá tải, **thà từ chối một phần request còn hơn để DB sập kéo cả hệ chết.**

- Đặt **trần concurrency xuống DB** (như `pLimit(50)` ở mục ⑥). Vượt trần → trả lỗi nhanh / phục vụ bản degrade (giá trị mặc định, "đang tải…", dữ liệu hơi cũ).
- **Graceful degradation:** ví dụ trang sản phẩm lạnh thì tạm ẩn phần "gợi ý cá nhân hoá" (đắt) nhưng vẫn render phần lõi.
- Triết lý: **giữ DB sống = cho cache cơ hội ấm lên.** Death spiral bị chặn vì nguồn không bao giờ chết.

### A3. Probabilistic early expiration (XFetch) — chống stampede *trước khi* key chết

Thay vì để key hết hạn rồi cả đám lao xuống nguồn, ta cho **một** request *xui xẻo* được chọn ngẫu nhiên **làm mới key SỚM** (trước hạn) trong khi các request khác vẫn dùng bản cũ. Xác suất "được chọn" tăng dần khi gần hết hạn.

```
Ý tưởng XFetch (Vattani et al.):
  refresh sớm nếu:  now − delta·beta·ln(rand()) ≥ expiry
  (delta = chi phí tính lại; beta ≈ 1; rand() ∈ (0,1])
```

→ key **không bao giờ "chết lạnh" đồng loạt** vì luôn có người làm mới trước. Đây là kỹ thuật "không cho cold start tái diễn theo TTL".

### A4. Warm-from-peer / cross-cluster warming (bài học Facebook CS1)

Node/cluster lạnh **đọc từ peer đã ấm** thay vì từ nguồn gốc:

- L1 lạnh đọc L2 ấm (multi-tier mặc định đã làm điều này — đây là *lý do thiết kế*).
- Cụm cache lạnh đọc cụm cache ấm (Facebook cold cluster warmup).
- Khi failover, **promote replica đã ấm**, đừng spin node rỗng.

### A5. Shadow / replay traffic để pre-warm

Trước khi đưa node mới vào phục vụ thật, **bắn một bản sao traffic thật (shadow)** vào nó (không trả response cho user) để L1 tự ấm theo đúng phân bố thật. Khi readiness probe thấy hit ratio đạt ngưỡng → mở traffic thật. Ấm theo **working set thực tế** thay vì đoán mò danh sách hot key.

### A6. Negative caching để chặn "cold start của thứ không tồn tại"

Nếu nhiều request hỏi key **không có trong nguồn**, mỗi lần miss vẫn đập DB (DB trả "không có" rồi ta không cache → lần sau lại hỏi). Cache **cả kết quả "không tồn tại"** (TTL ngắn) để cold start không bị "rò rỉ" qua các truy vấn rác/quét. (Bài 1 A2 đã có *negative caching*.)

### A7. Tách "control plane" của cache khỏi "data plane"

- Có **nút bật/tắt từng cache** và **giảm tải có kiểm soát** (feature flag).
- Khi cold start xấu đi, có thể **tạm hạ TTL / tắt phần đắt / bật chế độ degrade** mà không cần deploy.
- Có **game day**: chủ động flush cache trong môi trường staging (hoặc production có kiểm soát) để **diễn tập** kịch bản cold start trước khi nó xảy ra thật.

---

## ⑩ Trường hợp MỞ RỘNG (cold start ngoài cache thuần)

Cùng tên "cold start" nhưng khác bản chất. Biết để **không nhầm khi phỏng vấn / debug**.

| Loại cold start | Cái gì "lạnh" | Triệu chứng | Giải pháp đặc thù |
|---|---|---|---|
| **Cache cold start** (chủ đề chính) | Cache rỗng | Miss storm, DB QPS vọt | Warming, multi-tier, shedding, coalescing |
| **Serverless cold start** (Lambda/Cloud Functions) | Runtime/môi trường thực thi | Lần gọi đầu chậm vài trăm ms–vài s | Provisioned concurrency, keep-warm ping, giảm package size, runtime nhẹ |
| **JVM / JIT cold start** | Code chưa được JIT tối ưu, JIT cache rỗng | Vài phút đầu sau khởi động chạy chậm | JIT warmup, AOT compilation (GraalVM), warmup requests |
| **Connection pool cold start** | Pool kết nối DB/Redis rỗng | Burst đầu tốn handshake, p99 cao | Min-idle connections, pre-open pool lúc khởi động |
| **ML model cold start** | Model chưa load vào GPU/RAM, KV-cache rỗng | Request đầu chậm (load weights, no KV reuse) | Pre-load model, model warmup, giữ replica nóng |
| **Recommendation cold start** (*nghĩa hoàn toàn khác!*) | **User/item MỚI chưa có dữ liệu lịch sử** | Không gợi ý được gì cho người dùng mới | Content-based fallback, popularity baseline, onboarding hỏi sở thích |

> 🔑 **Insight #4:** "Cold start" trong **hệ gợi ý** (recommendation) nghĩa là **thiếu dữ liệu lịch sử về user/item mới** — KHÔNG liên quan cache. Đây là cái bẫy thuật ngữ kinh điển. Khi đọc/nghe "cold start", luôn xác định **ngữ cảnh**: cache? serverless? recommender? — ba thế giới khác nhau.

---

## ⑪ Kiến thức MỞ RỘNG (kết nối ra ngoài)

- ➕ **Working set & thrashing:** cold start chỉ *qua nhanh* nếu cache **đủ chứa working set** (Bài 1 A2). Nếu cache quá nhỏ → không bao giờ thoát lạnh, rơi vào **thrashing**. ⇒ *Sizing cache theo working set là điều kiện cần để cold start tự kết thúc.*
- ➕ **Eviction policy (LRU/LFU):** trong lúc làm ấm, policy quyết định cái gì được giữ lại. **LFU** giữ key nóng dài hạn tốt hơn khi vừa warm xong (tránh hot key bị đẩy ra bởi traffic quét một lần).
- ➕ **Amdahl's law (Bài 1 B2):** nếu nút thắt **không ở DB**, thì cố warming cache cũng không cứu được cold start về mặt p99 → **đo trước** (Bài 1 B1 "Đo trước khi thêm tầng").
- ➕ **Tail latency & fan-out:** một trang fan-out 50 service → p99 của từng service khuếch đại thành p50 của trang trong lúc cold. ⇒ cold start của *một* dependency lan ra *toàn trang*.
- ➕ **Idempotency (Bài 1 B2):** back-fill và warming phải **idempotent** để retry an toàn khi nhiều request cùng nạp một key trong cơn cold.
- ➕ **CAP / eventual consistency:** trong lúc degrade vì cold, ta thường **tạm chấp nhận eventual consistency** (phục vụ bản hơi cũ / mặc định) để giữ availability — đúng tinh thần đánh đổi của Bài 1.
- ➕ **SRE / error budget:** cold start sau deploy "ăn" vào error budget. ⇒ lý do để **deploy giờ thấp điểm** + **canary** + **rollback nhanh**.

---

## ⑫ Anti-patterns — AI & người mới hay sai

| ❌ Sai lầm | Vì sao nguy hiểm | ✅ Đúng |
|---|---|---|
| "Cứ thêm cache là an toàn" | Bỏ qua việc cache có thể **biến mất** → cold start làm sập nguồn | Hỏi: *"mất cache thì nguồn sống không?"* (Insight #3) |
| Restart/deploy **tất cả node cùng lúc** | 100% cluster lạnh đồng thời | **Rolling/staggered** + readiness gating |
| Nạp toàn bộ key cùng một TTL | **Synchronized expiry** = cold start lặp mỗi chu kỳ | **Jitter** TTL |
| `FLUSHALL` để "fix nhanh" trên prod | Xoá sạch cache → DB ăn full traffic → outage | Invalidate **chọn lọc** theo key; disable lệnh nguy hiểm |
| Mỗi miss tự do đập DB | Cold start = hàng nghìn query song song → death spiral | **Singleflight + bounded concurrency + load shedding** |
| Đổi version key mà không nghĩ tới warming | Cache "đầy" nhưng hit ratio = 0 (cold ẩn) | Warm namespace mới trước / dual-read trong giai đoạn chuyển |
| Tin p50 đẹp là ổn | p99/p999 mới phản ánh đau của cold start | Đo **p99/p999 + DB QPS**, không chỉ trung bình |

> 🔎 **AI hay sai (nối tiếp cảnh báo cuối Bài 1):** AI mặc định *"thêm cache = nhanh hơn"* và **không bao giờ nghĩ tới lúc cache rỗng**. Khi thiết kế, luôn ép nó/ép mình trả lời: *"Cold start xảy ra khi nào, và nguồn có chịu nổi lúc đó không?"*

---

## ⑬ Khung quyết định / Playbook cold start

### D-COLD-1 — Câu hỏi gốc (luôn hỏi đầu tiên)
> **"Nếu cache biến mất ngay lúc cao điểm, nguồn có sống không?"**
> - Sống → cold start là *latency issue*, ưu tiên warming + jitter.
> - Không sống → cold start là *availability issue*, **bắt buộc** load shedding + capacity plan + staggered restart.

### D-COLD-2 — Checklist phòng cold start (đính lên tường)
```
□ L1 có warming / readiness gating không? (đừng nhận traffic khi còn lạnh)
□ Restart có rolling/staggered không? (đừng lạnh đồng loạt)
□ TTL có jitter không? (đừng synchronized expiry)
□ Có singleflight/coalescing cho hot key không? (chống stampede)
□ Có bounded concurrency + load shedding xuống nguồn không? (chống death spiral)
□ Redis có persistence/replica để khỏi lạnh sau restart không?
□ Đã sizing nguồn cho kịch bản "mất cache" chưa?
□ Đã diễn tập (game day) flush cache trong staging chưa?
□ Deploy có chọn giờ thấp điểm + canary + rollback nhanh không?
```

### D-COLD-3 — Khi cold start ĐANG xảy ra (incident)
```
1. Bật load shedding / hạ trần concurrency xuống DB  → chặn death spiral.
2. Bật chế độ degrade (tắt phần đắt, phục vụ default/stale).
3. Warm từ peer ấm nếu có (đừng đập nguồn).
4. KHÔNG restart thêm node (đừng tạo thêm cache lạnh).
5. Theo dõi DB QPS + p99; chỉ mở lại tải khi cache đã ấm tới ngưỡng.
```

---

## ⑭ Mental model (câu thần chú gốc)

> 🥶 **Cold start = lúc trả trước "thuế miss" của cache.** Mọi cache đều phải đi qua nó.
> Câu hỏi sống còn không phải *"có cold start không"* (chắc chắn có), mà *"nguồn có sống sót qua lúc lạnh không"*.
>
> **Nếu mất cache mà nguồn vẫn sống → cache là tối ưu, cold start chỉ chậm.**
> **Nếu mất cache mà nguồn chết → cache là load-bearing, cold start là outage chờ sẵn → PHẢI thiết kế cho nó.**
>
> Ba lá bùa: **đừng lạnh đồng loạt** (staggered + jitter), **đừng để miss tự do đập nguồn** (coalescing + shedding), **học từ peer ấm** (multi-tier + warm-from-peer).

---

## ⑮ Câu hỏi phỏng vấn & thách đố

- **(dễ)** Cold start trong caching là gì? → Cache rỗng lúc khởi động/restart → miss gần 100% → dồn tải xuống nguồn, p99 tăng vọt.
- **(dễ)** L1 hay L2 lạnh sau mỗi deploy? → **L1** (in-process, chết theo process). L2 có thể giữ ấm nếu bật persistence/replica.
- **(TB)** Vì sao cold start có thể gây *outage* chứ không chỉ *chậm*? → Cache load-bearing: mất cache → DB nhận N× tải → DB sập → cache không kịp ấm vì nguồn đã chết = **death spiral**.
- **(TB)** Phân biệt cold start, cache stampede, thrashing. → khởi đầu rỗng / một-key-hot-vừa-chết-đập-đồng-loạt / cache-quá-nhỏ-nạp-rồi-evict-mãi.
- **(TB)** Synchronized expiry là gì, chữa sao? → nhiều key cùng TTL cùng chết → cold theo chu kỳ → **jitter TTL**.
- **(khó)** Hệ bạn cache 95% traffic, DB chịu 1.000 QPS, traffic 10.000 req/s. Ai đó `FLUSHALL`. Chuyện gì xảy ra và bạn làm gì? → DB ăn ~10.000 QPS = 10× → sập → death spiral. Xử: load shedding/bounded concurrency, degrade, warm-from-peer, **không** restart thêm.
- **(khó)** Làm sao đưa một node mới vào cluster mà không gây cold start cho cả hệ? → **readiness gating** (chỉ nhận traffic khi đã warm) + **staggered** + **shadow/replay pre-warm** + warm L1 từ L2.
- **(bẫy)** "Cold start" — bạn hiểu nghĩa nào? → hỏi lại ngữ cảnh: **cache** / **serverless runtime** / **recommendation (user mới)** — ba thứ khác hẳn nhau.

> 🧩 **Thách đố:** *"Team thêm cache warming sau deploy mà p99 vẫn nổ tung mỗi lần restart — vì sao?"*
> Khả năng: (1) **warming danh sách sai** — không khớp working set thật (dùng shadow/replay thay vì đoán); (2) **node nhận traffic TRƯỚC khi warm xong** (thiếu readiness gating); (3) **restart đồng loạt** nên warming của vài node không cứu được lúc 100% lạnh; (4) **nút thắt không ở cache** (Amdahl) — phải **đo** lại.

---

## ⑯ Bài tập + tiêu chí tự chấm

**BT1.** Vẽ timeline của một sự cố cold start do `FLUSHALL` lúc cao điểm: đánh dấu thời điểm DB QPS vọt, p99 nổ, connection pool cạn, và **3 điểm bạn có thể can thiệp** để chặn death spiral.
> ✅ **Đạt khi:** chỉ ra được điểm đặt **load shedding/bounded concurrency**, điểm bật **degrade**, và lý do **không restart thêm node**.

**BT2.** Cho hệ: traffic 20.000 req/s, hit ratio 90%, DB chịu 3.000 QPS. Hỏi: cache là *optimization* hay *load-bearing*? Cần những phòng thủ nào?
> ✅ **Đạt khi:** tính được cold start đẩy DB lên ~20.000 QPS ≫ 3.000 ⇒ **load-bearing** ⇒ bắt buộc shedding + staggered + capacity plan, không chỉ warming.

**BT3.** Viết lại hàm `getProduct` của Bài 1 (mục ⑤ tài liệu gốc) để chịu được cold start: thêm **singleflight + bounded concurrency + jitter TTL**.
> ✅ **Đạt khi:** miss trùng key chỉ đi nguồn 1 lần; số query DB đồng thời bị chặn trần; TTL có nhiễu ngẫu nhiên.

**BT4.** Phân biệt bằng lời 3 nghĩa của "cold start" (cache / serverless / recommendation) và một giải pháp đặc thù cho mỗi nghĩa.
> ✅ **Đạt khi:** không nhầm lẫn; serverless → provisioned concurrency/keep-warm; recommendation → content-based/popularity fallback; cache → warming/shedding.

---

## ⑰ Glossary (đóng vào hoàn cảnh)

| Thuật ngữ | Nghĩa nhanh | 🎬 Hoàn cảnh sử dụng |
|---|---|---|
| **Cold start** | Cache rỗng → miss gần 100% | *"Sau deploy L1 cold, p99 vọt vài phút đầu."* |
| **Warm / cold cache** | Cache đã đầy hot key / còn rỗng | *"Chờ cache warm rồi mới mở traffic."* |
| **Load-bearing cache** | Cache mà nguồn không sống nổi nếu thiếu | *"Cache này load-bearing → mất là sập DB."* |
| **Death spiral** | Vòng xoáy: nguồn quá tải → chậm → retry → càng tải | *"Cold start kéo theo death spiral nếu không shed tải."* |
| **Cache warming / pre-warm** | Nạp sẵn hot key trước khi phục vụ | *"Pre-warm top-1000 trước flash sale."* |
| **Staggered / rolling restart** | Restart lệch nhau từng phần | *"Rolling restart 10% mỗi đợt, giữ phần lớn ấm."* |
| **Readiness gating** | Chỉ nhận traffic khi đã sẵn sàng/ấm | *"Readiness probe đỏ tới khi L1 warm."* |
| **Synchronized expiry** | Nhiều key cùng hết hạn một lúc | *"Cùng TTL → synchronized expiry → cold lại theo chu kỳ."* |
| **Jitter** | Nhiễu ngẫu nhiên cộng vào TTL | *"Thêm jitter để key không chết đồng loạt."* |
| **Singleflight / coalescing** | Gộp miss trùng key thành 1 lần đi nguồn | *"Singleflight chống stampede lúc cold."* |
| **Load shedding / backpressure** | Chủ động từ chối bớt để bảo vệ nguồn | *"Shed tải để DB sống, cho cache kịp ấm."* |
| **Origin shield / tiered cache** | Lớp cache đệm giữa edge và origin | *"Origin shield gộp miss của trăm edge."* |
| **Warm-from-peer** | Node/cụm lạnh đọc từ bản đã ấm | *"Cold cluster đọc từ warm cluster, không đập DB."* |
| **Provisioned concurrency** | Giữ sẵn instance serverless nóng | *"Provisioned concurrency để hết Lambda cold start."* |

---

## ⑱ Đọc thêm

- **Scaling Memcached at Facebook** (NSDI 2013) — phần *cold cluster warmup* (warm-from-peer).
- **"Optimal Probabilistic Cache Stampede Prevention"** (Vattani, Chierichetti, Lowenstein) — thuật toán **XFetch** (probabilistic early expiration).
- **AWS** — *Database Caching Strategies Using Redis* (whitepaper) + tài liệu **Lambda provisioned concurrency** (cho serverless cold start).
- **Cloudflare / Fastly** docs — *tiered caching / origin shield* (cold edge).
- **Redis** docs — *persistence (RDB/AOF)*, *replication*, *renaming dangerous commands*.
- **Google SRE Book** — chương về *handling overload*, *load shedding*, *graceful degradation*, *addressing cascading failures*.

---

> 🔗 **Nối lại với Bài 1:** Cold start là **mặt động (theo thời gian) của cái khung đánh đổi tĩnh** mà Bài 1 dựng lên (*tốc độ ↔ tươi ↔ RAM*). Bài 1 dạy *"chỉ cache khi đo được lợi"*; chuyên đề này thêm một câu phải luôn hỏi: ***"…và hệ vẫn sống khi cache rỗng."***
