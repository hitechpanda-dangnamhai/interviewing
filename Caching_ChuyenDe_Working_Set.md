# 🎯 CHUYÊN ĐỀ — WORKING SET (Tập làm việc) trong Caching
> Đào sâu một keyword/concept mở rộng của **Bài 1 — Vì sao cache · Các tầng cache · Khi nào KHÔNG cache**
> Mục tiêu: hiểu *bản chất*, biết *đo*, biết *sizing cache* theo working set ở mức **staff engineer**.
> (✅ = đã có trong Bài 1 · ➕ = mở rộng thêm)

---

## ① Mục tiêu & vị trí trong mạch

Trong Bài 1, **Working set** mới xuất hiện ở **bảng mở rộng (A2)** với một câu ngắn:

> ➕ **Working set** — *Tập dữ liệu thực sự được dùng trong một khoảng thời gian* — *"Cache nên đủ chứa **working set**, không cần chứa toàn bộ data."*

Và ở chuyên đề **Cold Start** ta đã chốt một insight: *"cold start chỉ qua nhanh nếu cache đủ chứa working set; nếu không → thrashing, mãi không thoát lạnh."* Chuyên đề này "mở hộp" khái niệm gốc đó.

Học xong bạn sẽ:

- Hiểu **working set là gì** (và *không phải* là gì), từ gốc lý thuyết bộ nhớ ảo của **Denning** đến caching hiện đại.
- Biết vì sao **cache chỉ cần bằng working set**, không cần bằng toàn bộ data — và điều gì xảy ra khi cache *nhỏ hơn* / *lớn hơn* working set.
- Biết **ĐO** working set thật: **reuse distance**, **miss ratio curve (MRC)**, các thuật toán **SHARDS / Counter Stacks**.
- Biết **sizing cache** đúng (tìm "khuỷu" của đường cong hit ratio), và xử lý khi working set **dịch chuyển** hay **không vừa cache**.

> 💡 **Một câu để gắn vào mạch Bài 1:** Nếu **locality** (B1 Bài 1) là *lý do* cache hoạt động, thì **working set** chính là **hình hài đo được của locality** — nó trả lời câu hỏi *"cache cần TO bao nhiêu thì đủ?"*. Không nắm working set thì việc sizing cache chỉ là **đoán mò theo RAM còn trống**.

---

## ② Trực giác — Working set là gì?

**Working set (tập làm việc) = tập dữ liệu THỰC SỰ được truy cập trong một CỬA SỔ THỜI GIAN nhất định.**

Hai từ khoá cần khắc cốt:
- **"Thực sự được truy cập"** — không phải *toàn bộ* data bạn có, chỉ phần đang *được dùng*.
- **"Cửa sổ thời gian"** — working set luôn gắn với một khoảng thời gian τ (tau). Đổi τ → đổi kích thước working set.

### Ẩn dụ cái bàn làm việc

Bạn có cả một **thư viện** (toàn bộ dataset) trong nhà. Nhưng để làm việc *hôm nay*, trên **mặt bàn** bạn chỉ cần khoảng chục cuốn sách đang dùng — đó là **working set**.

- **Mặt bàn = cache.** Nó nhỏ, nhanh, nhưng đủ để chứa những cuốn bạn đang lật qua lật lại.
- **Thư viện = toàn bộ data** (nguồn / DB). To, chậm, nhưng đầy đủ.
- **Bí quyết:** mặt bàn **không cần to bằng thư viện** — chỉ cần **to bằng working set** (chục cuốn đang dùng) là bạn gần như không phải đứng dậy đi lấy sách.

> 🔑 **Câu thần chú gốc (Bài 1 A2):** *"Cache nên đủ chứa working set, không cần chứa toàn bộ data."* Cả chuyên đề này là phần chứng minh + cách *đo* câu đó.

### Gốc lý thuyết — Denning (1968)

Khái niệm này ra đời từ **bộ nhớ ảo (virtual memory)** của hệ điều hành. Peter Denning định nghĩa:

> **W(t, τ)** = tập các *page* khác nhau được tham chiếu trong khoảng thời gian **(t − τ, t]**.

Tức: nhìn lại τ đơn vị thời gian gần nhất, đếm xem có bao nhiêu *trang dữ liệu khác nhau* đã bị đụng tới. Đó là working set tại thời điểm t với cửa sổ τ.

- τ **nhỏ** → working set nhỏ (chỉ vài thứ vừa chạm).
- τ **lớn** → working set lớn dần, nhưng **plateau** (chững lại) — vì qua một ngưỡng, bạn chỉ gặp lại đồ cũ chứ ít gặp đồ mới. **Điểm plateau đó chính là "kích thước working set thật".**

---

## ③ Working set quan hệ thế nào với kích thước cache?

Đây là phần "ăn tiền". Có **ba vùng** kích thước cache:

```
Hit ratio
  ▲
1 │                         ┌──────────────  ← cache ≫ working set: PHÍ RAM (gần như phẳng)
  │                      ┌──┘
  │                 ┌────┘   ◄── "KHUỶU" (knee): điểm cache ≈ working set — sizing ở đây
  │            ┌────┘
  │      ┌─────┘
  │  ┌───┘                   ← cache < working set: THRASHING (hit ratio thấp, dốc)
  └──┴────────────────────────────────►  Kích thước cache
```

| Vùng | Quan hệ | Hậu quả |
|---|---|---|
| **Quá nhỏ** | cache **<** working set | **Thrashing** (Bài 1 A2): nạp vào rồi bị evict ngay, hit ratio thấp, cold start không bao giờ thoát |
| **Vừa đủ (khuỷu)** | cache **≈** working set | **Điểm vàng**: hit ratio cao, RAM dùng hiệu quả nhất |
| **Quá to** | cache **≫** working set | Hit ratio gần như **không tăng thêm** nhưng tốn RAM/tiền (diminishing returns) |

> 🔑 **Insight #1:** Mục tiêu sizing cache **không phải** "to nhất có thể" mà là **"tìm cái khuỷu (knee) của đường cong hit-ratio"** — nơi cache vừa khớp working set. To hơn = đốt tiền vô ích; nhỏ hơn = thrashing.

> 🔑 **Insight #2 (nối với Bài 1):** Câu *"hit ratio cao chưa chắc tốt"* (B1 Bài 1) và working set là hai mặt một đồng xu. Working set cho bạn biết hit ratio **có thể** cao tới đâu với một kích thước cache cho trước — đó là **trần lý thuyết**, còn bạn vẫn phải gắn nó vào *thứ cần bảo vệ* (DB QPS, p99).

---

## ④ Vì sao working set thường NHỎ hơn nhiều so với toàn bộ data?

Vì **traffic thực tế lệch nặng (skewed)** — không phải mọi key được hỏi đều nhau.

### Quy luật Zipf / power law (➕ kiến thức nền)

Trong rất nhiều hệ thực tế (web, mạng xã hội, e-commerce), tần suất truy cập tuân theo **phân phối Zipf**: key xếp hạng thứ *r* có tần suất ∝ **1/r^s**.

- Một **số ít key cực nóng** chiếm phần lớn lượt truy cập.
- "Đuôi dài" (long tail) gồm vô số key hiếm khi được hỏi.

Hệ quả thực tế (quy luật 80/20, đôi khi 90/10):

> **~10–20% data nóng** có thể chiếm **~80–90% lượt truy cập.**

⇒ Cache chỉ cần chứa cái phần 10–20% đó (≈ working set) là đã "bắt" được 80–90% request. **Đó là lý do sâu xa cache hoạt động được với RAM nhỏ.**

> 🔑 **Insight #3:** Cache "ăn lời" được là nhờ **skew (lệch)**. Nếu workload **đều tăm tắp** (uniform, mọi key xác suất như nhau) thì working set = toàn bộ data ⇒ cache gần như **vô dụng** (đây chính là trường hợp *"không có locality → cache vô dụng"* ở B1 Bài 1, nhìn dưới góc working set).

---

## ⑤ Working set là thứ ĐỘNG (thay đổi theo thời gian)

Sai lầm phổ biến: coi working set là một con số cố định. **Không.** Nó dịch chuyển:

| Kiểu dịch chuyển | Ví dụ |
|---|---|
| **Theo giờ trong ngày** | Ban ngày working set là sản phẩm/feed người dùng active; ban đêm khác hẳn |
| **Theo mùa / sự kiện** | Black Friday: working set phình to + đổi nội dung (đồ giảm giá) |
| **Theo xu hướng (trend)** | Một bài viral → bỗng vào working set; vài giờ sau rớt ra |
| **Phase change (đổi pha)** | Job batch ban đêm quét *toàn bộ* data → working set tạm = cả dataset (kẻ thù của cache!) |

> 🔑 **Insight #4:** Working set "ban ngày" và "ban đêm" có thể khác nhau cả về **kích thước** lẫn **nội dung**. Cache cố định kích thước sẽ **thừa lúc thấp điểm, thiếu lúc cao điểm**. Staff engineer hoặc **size theo đỉnh**, hoặc dùng cache **co giãn / admission policy thông minh** (mục ⑨).

> ⚠️ **Phase change là cạm bẫy:** một job quét toàn bảng (full scan) làm working set tức thời = cả dataset → đẩy hết hot key đang nóng ra khỏi cache (*cache pollution*) → sau job, cache **lạnh trở lại** = cold start trá hình. ⇒ cần **scan resistance** (mục ⑨).

---

## ⑥ Ví dụ minh hoạ

### Ví dụ 1 — E-commerce: catalog 10 triệu SP nhưng working set chỉ ~50k

- Tổng catalog: **10.000.000** sản phẩm.
- Trong một giờ cao điểm, thực tế chỉ **~50.000** sản phẩm được xem (hàng hot, đang chạy quảng cáo, trang chủ).
- ⇒ Working set ≈ 50k. Cache chỉ cần chứa ~50k–100k entry là **hit ratio ~90%+**, KHÔNG cần chứa 10 triệu.
- Cố nhét cả 10 triệu vào Redis = đốt RAM cho phần "đuôi dài" gần như không ai hỏi.

### Ví dụ 2 — Twitter/X timeline: chỉ user active + tweet gần đây

- Hàng trăm triệu user, hàng tỉ tweet (toàn bộ data).
- Working set tại một thời điểm: **timeline của user đang online** + **tweet gần đây của người họ follow**.
- ⇒ Cache (Redis) chứa working set "đang sống", không phải toàn bộ lịch sử.

### Ví dụ 3 — Đo τ làm đổi kích thước working set

```
Cùng một stream truy cập, đo working set với các τ khác nhau:

τ = 1 giây   → WSS ≈ 2.000 key   (chỉ thứ vừa chạm)
τ = 1 phút   → WSS ≈ 40.000 key
τ = 10 phút  → WSS ≈ 90.000 key
τ = 1 giờ    → WSS ≈ 110.000 key  ◄── bắt đầu PLATEAU
τ = 6 giờ    → WSS ≈ 125.000 key  (tăng rất ít → đã chạm trần)
```
⇒ "Kích thước working set thật" ở đây ≈ **~110k–125k**. Cache ~150k entry là đủ với biên an toàn. To hơn = phí.

### Ví dụ 4 — Code: ước lượng working set bằng sampling (HyperLogLog)

Đếm **số key DISTINCT** được hỏi trong cửa sổ τ — chính là kích thước working set. Dùng **HyperLogLog** để đếm distinct với bộ nhớ tí hon:

```js
import Redis from "ioredis";
const redis = new Redis(process.env.REDIS_URL);

// Mỗi request: ghi nhận key vào HLL của "phút hiện tại"
export async function trackAccess(key) {
  const bucket = `wss:${Math.floor(Date.now() / 60000)}`; // 1 bucket / phút
  await redis.pfadd(bucket, key);          // HyperLogLog add (xấp xỉ distinct)
  await redis.expire(bucket, 7200);        // giữ 2 giờ
}

// Ước lượng working set của cửa sổ τ phút gần nhất = đếm distinct hợp nhất các bucket
export async function estimateWSS(windowMinutes) {
  const now = Math.floor(Date.now() / 60000);
  const buckets = Array.from({ length: windowMinutes }, (_, i) => `wss:${now - i}`);
  return redis.pfcount(...buckets);        // PFCOUNT hợp nhất nhiều HLL
}
// estimateWSS(60) ≈ số key distinct trong 60 phút qua = working set ~1h
```

> Đây là cách *rẻ* để có một đường "WSS theo τ" như Ví dụ 3, từ đó tìm điểm plateau để sizing cache. Muốn chính xác hơn (hit ratio theo từng mức cache) thì cần **MRC** ở mục ⑧.

---

## ⑦ Case study thực tế

> ⚠️ Tóm tắt theo tài liệu/bài báo công khai và pattern phổ biến. Verify số liệu cụ thể khi cần trích dẫn.

### CS1 — VMware: SHARDS & đo working set của VM để chia RAM (FAST 2015)

VMware cần biết **mỗi máy ảo (VM) thực sự cần bao nhiêu RAM** để chia bộ nhớ hợp lý (memory ballooning). Đo working set bằng **miss ratio curve** truyền thống quá tốn (O(N·M)). Họ công bố **SHARDS** — lấy mẫu tham chiếu theo *spatial hashing* với tỉ lệ cố định, dựng được **MRC xấp xỉ** chỉ với **bộ nhớ hằng số (vài chục KB)** thay vì gigabyte.

> 🎯 **Bài học:** Working set/MRC **đo được online, rẻ** nếu chấp nhận lấy mẫu. Không cần lưu toàn bộ trace. SHARDS là công cụ chuẩn để "vẽ đường cong hit-ratio" trong thực tế.

### CS2 — Coho/Counter Stacks: working set của storage thay đổi theo pha (OSDI 2014)

Counter Stacks dùng cấu trúc đếm xác suất (kiểu HyperLogLog) để theo dõi **working set thay đổi theo thời gian** trong hệ lưu trữ, phát hiện **phase change** (lúc workload đổi pha → working set nhảy). Cho phép sizing tier SSD/RAM theo working set *động* thay vì con số tĩnh.

> 🎯 **Bài học:** Working set là **chuỗi thời gian**, không phải một số. Theo dõi nó để bắt **đổi pha** (Insight #4).

### CS3 — Facebook Memcached: sizing theo working set, không theo toàn bộ data

Trong "Scaling Memcached at Facebook", cache được sizing để giữ **phần nóng** của dữ liệu (working set của social graph / lượt đọc), không phải toàn bộ. Kết hợp tier riêng (TAO) cho social graph có pattern truy cập đặc thù.

> 🎯 **Bài học:** Working set **không đồng nhất** — các *lớp dữ liệu* khác nhau có working set khác nhau ⇒ đôi khi tách cache riêng cho từng lớp (mục ⑨ A6).

### CS4 — Caffeine (W-TinyLFU): bảo vệ working set khỏi "one-hit wonder"

Thư viện cache Java **Caffeine** dùng **W-TinyLFU**: trước khi cho một entry mới *chiếm chỗ* (evict entry cũ), nó dùng một **frequency sketch (Count-Min)** để hỏi *"thằng mới này có thật sự nóng hơn thằng sắp bị đuổi không?"*. Nếu không → **không kết nạp**. Nhờ vậy một loạt key "ghé một lần rồi đi" (scan, crawler) **không đẩy được working set thật ra ngoài**.

> 🎯 **Bài học:** Khi cache **không đủ to** cho mọi thứ, thứ quyết định không chỉ là *eviction* mà là **admission** (kết nạp ai vào). Admission policy tốt = bảo vệ working set bằng cache nhỏ hơn.

### CS5 — Database buffer pool (Bài 1 C1) cũng là bài toán working set

DBA chỉnh `innodb_buffer_pool_size` / `shared_buffers` thực chất đang **size cache theo working set của các *page* nóng**. Quy tắc kinh nghiệm "buffer pool ~ working set của index + hot rows" chính là Insight #1 áp vào DB.

---

## ⑧ Cách ĐO working set thật (staff engineer)

Đếm distinct (mục ⑥) cho biết *kích thước* working set, nhưng để biết **hit ratio sẽ là bao nhiêu ứng với mỗi kích thước cache**, ta cần công cụ mạnh hơn.

### (a) Reuse distance / stack distance — nền tảng

**Reuse distance của một lần truy cập** = số *key DISTINCT* được truy cập giữa lần này và **lần trước** cũng truy cập đúng key đó.

```
Chuỗi truy cập:   A  B  C  A  ...
                        └───┘
A xuất hiện lại sau khi đã gặp {B, C} → reuse distance của A (lần 2) = 2
```

**Định lý then chốt (cho LRU):**
> Một truy cập là **HIT** ⇔ **reuse distance < kích thước cache**.

⇒ Nếu biết **phân phối reuse distance** của toàn bộ trace, ta tính được **hit ratio cho MỌI kích thước cache** cùng lúc. Đó là **Miss Ratio Curve (MRC)**.

### (b) Miss Ratio Curve (MRC) / Hit Ratio Curve

MRC = đồ thị **miss ratio theo kích thước cache**. Đây chính là đường cong ở mục ③ (lật ngược). Từ MRC bạn đọc ra:

- **Khuỷu (knee)** → kích thước cache tối ưu (≈ working set).
- **Hit ratio đạt được nếu cấp X GB RAM** → để ra quyết định *chi phí ↔ lợi ích*.
- Mức **diminishing returns** → biết khi nào "thêm RAM là phí".

### (c) Thuật toán dựng MRC

| Thuật toán | Đặc điểm | Khi nào dùng |
|---|---|---|
| **Mattson's stack** (cổ điển) | Chính xác tuyệt đối nhưng **O(N·M)**, tốn bộ nhớ | Trace nhỏ, offline, làm chuẩn |
| **SHARDS** (VMware, CS1) | Lấy mẫu spatial-hash, **bộ nhớ hằng số**, sai số nhỏ | **Online, production** — lựa chọn mặc định |
| **Counter Stacks** (CS2) | Đếm xác suất (HLL), bắt **đổi pha** theo thời gian | Khi cần theo dõi working set *động* |
| **AET / mini-simulation** | Mô phỏng cache thật ở vài kích thước | Khi muốn kiểm chứng nhanh |

> 🔑 **Insight #5:** Đừng sizing cache bằng cảm giác hay "RAM còn trống". Hãy **dựng MRC (SHARDS) → tìm khuỷu → cấp RAM tới đó**. Đây là khác biệt giữa *đoán* và *kỹ thuật*.

### (d) Cách nhanh-bẩn nếu chưa có MRC

- **Thử nghiệm A/B kích thước cache:** chạy hai mức cache, đo hit ratio + DB QPS thực.
- **Đo "WSS theo τ"** (mục ⑥ HLL) tìm plateau.
- **Theo dõi eviction rate:** eviction cao + hit ratio thấp = cache **< working set** (đang thrashing) → cần to hơn.

---

## ⑨ Giải pháp khi working set KHÔNG vừa cache (nâng cao)

Đôi khi working set lớn hơn RAM bạn đủ tiền cấp. Khi đó *không phải* cứ mua thêm RAM — có nhiều vũ khí:

### A1. Admission control (kết nạp có chọn lọc) — W-TinyLFU / TinyLFU

Như CS4: **không kết nạp mọi miss vào cache.** Dùng frequency sketch để chỉ giữ key *đủ nóng*. Cache nhỏ vẫn giữ đúng working set vì rác bị chặn ở cửa. (Caffeine, một số Redis module.)

### A2. Scan resistance — chống full-scan đầu độc cache

Với job quét toàn bảng (Insight #4 phase change): dùng policy **scan-resistant** như **ARC** (Adaptive Replacement Cache) hoặc **SLRU/segmented LRU** — tách "vào lần đầu" khỏi "đã chứng tỏ là nóng", để một lần quét không đẩy hot key ra. (Nhiều DB buffer pool dùng biến thể này.)

### A3. Eviction policy khớp hình dạng working set

- **LRU** — tốt khi working set theo *temporal locality* thuần.
- **LFU / W-TinyLFU** — tốt khi vài key **nóng dài hạn** (Bài 1 C4 đã gợi ý: *"chọn LFU khi vài key luôn nóng dài hạn"*).
- **ARC / hyperbolic** — thích nghi giữa recency và frequency.

### A4. Tiering — đặt phần "ấm" xuống tầng rẻ hơn

Working set lõi (nóng nhất) ở **RAM/L1**; phần "ấm" (working set mở rộng) ở **SSD/Redis-on-flash**; phần lạnh ở **DB**. Mỗi tầng size theo working set tương ứng → giữ chi phí thấp. (Đây là multi-tier Bài 1, nhìn dưới góc working set.)

### A5. Adaptive sizing theo working set động

- **Co giãn theo giờ:** tăng cache lúc cao điểm, thu lúc thấp điểm (Insight #4).
- **Phát hiện đổi pha** (Counter Stacks) → tự điều chỉnh.
- Cloud: autoscale Redis cluster / ElastiCache theo metric eviction + hit ratio.

### A6. Tách working set theo lớp dữ liệu (segmentation)

Trộn chung các loại data có working set khác nhau (catalog ổn định + feed biến động + scan analytics) trong **một** cache làm chúng đá nhau. Tách **cache riêng / namespace riêng** cho từng lớp → mỗi cái size theo working set của nó, không đầu độc lẫn nhau. (Facebook tách TAO khỏi Memcached — CS3.)

### A7. Giảm working set ở GỐC (đôi khi tốt hơn tăng cache)

- **Denormalization** (Bài 1 B2) / precompute để 1 entry phục vụ nhiều request → giảm số key cần nóng.
- **Bớt cardinality của key:** key gồm quá nhiều chiều (user × region × variant…) làm working set **nổ tổ hợp**. Gộp/giảm chiều key khi có thể.
- **Negative caching** hợp lý để key rác không phình working set.

---

## ⑩ Trường hợp MỞ RỘNG (working set ngoài cache app)

Cùng một khái niệm lặp ở mọi tầng của ngành:

| Bối cảnh | "Working set" là gì | Hệ quả nếu cache < WSS |
|---|---|---|
| **OS / Virtual memory** (gốc) | Tập *page* được tham chiếu trong τ | **Thrashing**: page fault liên tục, hệ treo (Denning) |
| **CPU cache (L1/L2/L3)** | Tập cache line nóng của vòng lặp/hàm | Cache miss tăng, CPU stall — tối ưu bằng cách thu nhỏ working set (blocking/tiling) |
| **DB buffer pool** | Tập *page* index + hot rows | Đọc đĩa nhiều, QPS rớt (CS5) |
| **VM memory (cloud)** | RAM thực mỗi VM cần | Swap, ballooning sai → chậm cả host (SHARDS, CS1) |
| **CDN / edge** | Tập asset phổ biến trong vùng | Miss về origin nhiều, latency vùng tăng |
| **LLM serving (KV-cache)** | Tập KV của các sequence đang active | Phải tái tính prefill, GPU mem áp lực, throughput rớt |
| **Filesystem page cache** | Tập file/block đang đọc | I/O đĩa tăng vọt |

> 🔑 **Insight #6:** "Cache phải đủ chứa working set, không cần chứa toàn bộ" là **nguyên lý phổ quát** — từ L1 CPU tới CDN toàn cầu tới KV-cache của LLM. Hiểu nó một lần, áp được ở mọi tầng.

---

## ⑪ Kiến thức MỞ RỘNG (kết nối)

- ✅ **Locality (B1 Bài 1):** working set là **biểu hiện đo được của temporal locality**. Locality mạnh → working set nhỏ gọn → cache hiệu quả. Không locality → working set = toàn bộ data → cache vô dụng.
- ➕ **Thrashing (A2 Bài 1):** trạng thái cache < working set. Working set cho ta *ngưỡng* để tránh thrashing.
- ➕ **Cold start (chuyên đề trước):** cold start chỉ *kết thúc* khi cache đã nạp đủ working set. Cache nhỏ hơn working set ⇒ **cold mãi không thoát** (rơi vào thrashing).
- ➕ **Zipf / power law:** lý do working set ≪ toàn bộ data trong thực tế (skew). Workload uniform = không có "phần nóng" để cache.
- ➕ **Belady's MIN/OPT:** chính sách thay thế *tối ưu lý thuyết* (đuổi thứ dùng xa nhất trong tương lai) — dùng làm **chuẩn trên** để biết policy thật còn cách tối ưu bao xa.
- ➕ **Belady's anomaly:** với FIFO, **tăng cache có thể làm miss TĂNG** (nghịch lý) — một lý do nữa để *đo MRC thật* thay vì giả định "to hơn luôn tốt hơn". (LRU/LFU thuộc lớp "stack algorithm" nên miễn nhiễm anomaly này.)
- ➕ **Eviction LRU/LFU/ARC (C4 Bài 1):** policy quyết định cache *giữ đúng* working set tới đâu khi không đủ chỗ.
- ➕ **Amdahl / "đo trước khi thêm tầng" (B1 Bài 1):** nếu nút thắt không ở chỗ cache bảo vệ, sizing working set chuẩn cũng không cứu p99.

---

## ⑫ Anti-patterns — AI & người mới hay sai

| ❌ Sai lầm | Vì sao sai | ✅ Đúng |
|---|---|---|
| "Cache phải chứa **toàn bộ** data cho chắc" | Đốt RAM cho long-tail gần như không ai hỏi | Size theo **working set** (khuỷu MRC) |
| Sizing cache bằng "**RAM còn trống**" | Không liên quan gì tới nhu cầu thật | **Dựng MRC (SHARDS)** → tìm knee |
| Coi working set là **một con số cố định** | Nó dịch chuyển theo giờ/mùa/pha | Theo dõi **WSS theo thời gian**, size theo đỉnh / adaptive |
| Một cache chung cho **mọi loại data** | Working set khác nhau đá nhau, scan đầu độc hot | **Tách cache theo lớp** + admission/scan resistance |
| Để **full-scan** chạy qua cache chính | Đẩy hot key ra → cold trở lại | **Scan-resistant policy** (ARC/SLRU) hoặc bypass cache cho scan |
| Tin "**hit ratio cao = đủ to**" | Có thể đang cache thứ rẻ/đuôi dài | Gắn vào **DB QPS / p99** (B1 Bài 1) + đọc MRC |
| Key **quá nhiều chiều** | Working set nổ tổ hợp, không cache nổi | Giảm cardinality / denormalize (A7) |

> 🔎 **AI hay sai (nối Bài 1):** AI hay khuyên *"tăng cache lên cho hit ratio cao"* mà không hỏi **working set đang bao to** và **đường cong hit-ratio có còn dốc không**. Luôn ép trả lời: *"Working set ước lượng bao nhiêu? Cache đang ở đoạn nào của MRC — còn dốc hay đã phẳng?"*

---

## ⑬ Khung quyết định / Playbook working set

### D-WSS-1 — Trước khi sizing cache, hỏi 3 câu
```
1. Working set ước lượng bao to? (đếm distinct theo τ → tìm plateau)
2. Cache hiện tại đang ở đoạn nào của MRC? (dốc = thêm RAM lời; phẳng = phí)
3. Working set có DỊCH CHUYỂN không? (giờ/mùa/pha → size theo đỉnh hay adaptive)
```

### D-WSS-2 — Chẩn đoán nhanh "cache có đủ to không"
```
Eviction rate CAO  + hit ratio THẤP   → cache < working set (THRASHING) → tăng to / admission
Eviction rate THẤP + hit ratio CAO    → ổn, có thể đang ở/ qua khuỷu
Eviction ~0        + RAM dùng đầy ≪ cấp → cache ≫ working set → có thể THU bớt, tiết kiệm
Hit ratio rớt theo CHU KỲ              → phase change (job scan?) → scan resistance / tách cache
```

### D-WSS-3 — Khi working set > ngân sách RAM (theo thứ tự rẻ→đắt)
```
1. Admission control (W-TinyLFU)        → giữ đúng hot bằng cache nhỏ hơn.
2. Tách cache theo lớp data             → hết đầu độc lẫn nhau.
3. Giảm working set ở gốc               → denormalize / giảm cardinality key.
4. Tiering (RAM → SSD/Redis-flash)      → phần ấm xuống tầng rẻ.
5. Mới tới: mua thêm RAM / scale-out.    → giải pháp cuối, không phải đầu.
```

---

## ⑭ Mental model (câu thần chú gốc)

> 🎯 **Working set = phần data ĐANG sống trong một cửa sổ thời gian.** Cache chỉ cần to bằng nó — không cần bằng cả kho.
>
> **Cache < working set → thrashing** (cold mãi không thoát).
> **Cache ≈ working set → điểm vàng** (sizing ở "khuỷu" của đường hit-ratio).
> **Cache ≫ working set → đốt tiền** (diminishing returns).
>
> Đừng đoán — **dựng MRC (SHARDS), tìm khuỷu, cấp RAM tới đó.** Và nhớ working set **biết dịch chuyển** (giờ/mùa/pha), nên hoặc size theo đỉnh, hoặc dùng admission + adaptive thay vì cứ mua thêm RAM.

---

## ⑮ Câu hỏi phỏng vấn & thách đố

- **(dễ)** Working set là gì? → Tập data thực sự được truy cập trong một cửa sổ thời gian τ; cache nên đủ chứa nó, không cần chứa toàn bộ data.
- **(dễ)** Vì sao working set thường ≪ toàn bộ data? → Truy cập **lệch (Zipf/power law)**: ~10–20% data nóng chiếm ~80–90% lượt.
- **(TB)** Cache nhỏ hơn working set thì sao? → **Thrashing**: nạp vào bị evict ngay, hit ratio thấp, cold start không thoát.
- **(TB)** Sizing cache to nhất có tốt không? → Không — qua "khuỷu" là **diminishing returns**, chỉ đốt RAM. Mục tiêu là tìm knee ≈ working set.
- **(TB)** Reuse distance liên quan gì tới hit ratio? → Với LRU: hit ⇔ reuse distance < cache size ⇒ phân phối reuse distance cho ra **MRC** (hit ratio ở mọi kích thước cache).
- **(khó)** Đo working set/MRC trên production sao cho rẻ? → Lấy mẫu: **SHARDS** (bộ nhớ hằng số), hoặc **Counter Stacks** để bắt đổi pha; HLL để ước lượng WSS theo τ.
- **(khó)** Working set 200GB, RAM chỉ 64GB, không thêm tiền — làm gì? → admission (W-TinyLFU), tách cache theo lớp, giảm cardinality/denormalize ở gốc, tiering RAM→SSD; mua RAM là cuối.
- **(bẫy)** Tăng cache mà miss lại tăng — sao kỳ vậy? → **Belady's anomaly** (với FIFO); chuyển sang policy "stack" (LRU/LFU) và **đo MRC thật** thay vì giả định "to hơn = tốt hơn".

> 🧩 **Thách đố:** *"Hit ratio tụt đúng 2h sáng mỗi đêm rồi tự hồi — vì sao và xử thế nào?"*
> Khả năng cao: **job batch/ETL quét toàn bảng lúc 2h** làm working set tức thời = cả dataset → đẩy hot key ra (cache pollution) → sau job cache lạnh lại. Xử: **scan-resistant policy** (ARC/SLRU), cho job **bypass cache**, hoặc **tách cache** khỏi đường analytics.

---

## ⑯ Bài tập + tiêu chí tự chấm

**BT1.** Cho catalog 5 triệu SP. Đo được: τ=1ph→WSS 30k, 10ph→70k, 1h→95k, 6h→100k. Hỏi nên cấp cache cho bao nhiêu entry và vì sao?
> ✅ **Đạt khi:** nhận ra plateau ~95k–100k ⇒ cache ~120k (biên an toàn); giải thích "to hơn là phí, nhỏ hơn 95k là rủi ro thrashing".

**BT2.** Cho trace ngắn `A B C A B D A`. Tính reuse distance mỗi lần truy cập và suy ra: cache size = 2 thì hit ratio bao nhiêu?
> ✅ **Đạt khi:** tính đúng reuse distance (A lần 2 = 2, B lần 2 = 2, A lần 3 = 2, …) và áp đúng quy tắc *hit ⇔ reuse distance < cache size*.

**BT3.** Hệ có 3 loại data dùng chung 1 Redis: catalog (ổn định), feed (biến động nhanh), report-scan (quét lớn theo giờ). Hit ratio chung thấp. Đề xuất kiến trúc.
> ✅ **Đạt khi:** tách cache/namespace theo lớp; report-scan **bypass** hoặc cache riêng scan-resistant; size mỗi cache theo working set riêng.

**BT4.** Giải thích vì sao "size cache theo RAM còn trống" là anti-pattern, và quy trình đúng để sizing.
> ✅ **Đạt khi:** nêu được: đo WSS theo τ / dựng MRC bằng SHARDS → tìm knee → cấp RAM tới đó → giám sát eviction + hit ratio để hiệu chỉnh.

---

## ⑰ Glossary (đóng vào hoàn cảnh)

| Thuật ngữ | Nghĩa nhanh | 🎬 Hoàn cảnh sử dụng |
|---|---|---|
| **Working set (WSS)** | Tập data được truy cập trong cửa sổ τ | *"Cache đủ chứa working set là đủ, khỏi nhét cả kho."* |
| **Cửa sổ τ (tau)** | Khoảng thời gian dùng để đo working set | *"Đo WSS theo nhiều τ, tìm chỗ plateau."* |
| **Reuse / stack distance** | Số key distinct giữa hai lần đụng cùng key | *"Hit khi reuse distance < cache size."* |
| **Miss Ratio Curve (MRC)** | Miss ratio theo kích thước cache | *"Đọc MRC, cấp RAM tới khuỷu là tối ưu."* |
| **Knee (khuỷu)** | Điểm cache ≈ working set trên đường cong | *"Sizing đúng = nằm ở knee, không vượt."* |
| **SHARDS** | Thuật toán lấy mẫu dựng MRC bộ nhớ hằng số | *"Bật SHARDS để vẽ MRC trên prod cho rẻ."* |
| **Counter Stacks** | Theo dõi working set động, bắt đổi pha | *"Counter Stacks báo working set nhảy lúc 2h sáng."* |
| **Zipf / power law** | Phân phối truy cập lệch nặng | *"Workload Zipf nên cache nhỏ bắt được 90% hit."* |
| **Thrashing** | Cache < working set, evict-nạp liên hồi | *"Eviction cao + hit thấp = thrashing, cache quá nhỏ."* |
| **Admission control (W-TinyLFU)** | Lọc ai được kết nạp vào cache | *"W-TinyLFU chặn one-hit-wonder, giữ working set."* |
| **Scan resistance (ARC/SLRU)** | Chống full-scan đầu độc cache | *"Bật ARC để job quét đêm không thổi bay hot key."* |
| **Phase change** | Working set đổi pha đột ngột | *"ETL chạy → phase change → working set = cả bảng."* |
| **Diminishing returns** | Tăng cache nhưng hit ratio gần như đứng yên | *"Qua knee rồi, thêm RAM là đốt tiền."* |

---

## ⑱ Đọc thêm

- **P. Denning, "The Working Set Model for Program Behavior" (1968)** — gốc của khái niệm.
- **Waldspurger et al., "Efficient MRC Construction with SHARDS" (FAST 2015)** — đo MRC rẻ, online.
- **Wires et al., "Characterizing Storage Workloads with Counter Stacks" (OSDI 2014)** — working set động / phase.
- **Mattson et al. (1970)** — thuật toán stack & lý thuyết stack distance (nền MRC).
- **Einziger, Friedman, Manes — "TinyLFU / W-TinyLFU"** + thư viện **Caffeine** — admission control.
- **Megiddo & Modha — "ARC: A Self-Tuning Replacement Cache" (IBM)** — scan-resistant.
- **Scaling Memcached at Facebook (NSDI 2013)** — sizing & tách lớp working set ở quy mô lớn.

---

> 🔗 **Nối lại với Bài 1 & chuyên đề Cold Start:** Bài 1 dạy *locality* là *lý do* cache chạy; **working set là thước đo của locality** — nó biến câu hỏi mơ hồ *"cache cần to bao nhiêu"* thành một phép đo (MRC → knee). Và nó khép vòng với Cold Start: ***cache chỉ thoát khỏi trạng thái lạnh khi đã nạp đủ working set — nhỏ hơn thì lạnh mãi.***
