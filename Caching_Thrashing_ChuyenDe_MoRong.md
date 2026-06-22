# 🌀 THRASHING — Chuyên đề mở rộng (Caching · nối tiếp Bài 1)
> **Cache liên tục evict rồi nạp lại, làm cật lực mà gần như VÔ ÍCH** · Keyword · Concept · Giải pháp · Case study · Staff-level
> (Đọc sau khi đã nắm: *locality · working set · eviction · Miss Ratio Curve · admission control · scan resistance*)

---

## ① ĐỊNH VỊ — Thrashing là "cái máy chạy bộ" của cache

Bài 1 chốt: cache sống nhờ **locality** — "thứ vừa/gần dùng sẽ được dùng lại". **Thrashing là trạng thái cache MẤT đi lợi thế đó dù vẫn chạy hết công suất.**

**Định nghĩa:** **Cache dành phần lớn công sức để ĐUỔI rồi NẠP LẠI chính những entry mà nó sắp cần dùng lại — vì chúng không đủ chỗ để ở lại.** Kết quả: hit ratio sụp, cache biến thành **phí tổn thuần** (tốn RAM + tốn lookup + tốn nạp lại) mà **gần như không đem lại hit nào**.

> 🔑 **Nguyên nhân gốc, một câu:** **Working set > sức chứa cache.** Tập dữ liệu *đang thực sự được dùng* không vừa cache → entry bị đuổi **trước khi** được tái sử dụng → mỗi lần dùng lại đều miss → nạp lại → lại bị đuổi → ... **vòng churn vô tận**.

🎬 *Câu thần chú:* **"Cache thrashing = cái treadmill: chạy mãi tại chỗ, đuổi-rồi-nạp đúng những thứ vừa bỏ đi, tốn sức tối đa cho lợi ích tối thiểu."**

### Gốc gác (để hiểu sâu): thrashing bộ nhớ ảo

Thuật ngữ **thrashing** ra đời từ **hệ điều hành** (Denning, ~1968, *working set model*): khi tập trang cần dùng **lớn hơn RAM vật lý**, hệ thống **dành gần như toàn bộ thời gian để swap page in/out đĩa** thay vì làm việc → **throughput sụp về gần 0**. Cache thrashing là **đúng hiện tượng đó, dịch lên một tầng**: thay vì RAM↔đĩa thì là cache↔nguồn (DB/service).

---

## ② TẠI SAO LÀ "VÁCH ĐÁ", KHÔNG PHẢI "DỐC THOẢI"

Hit ratio **không tụt đều đặn** — nó có thể **sụp đột ngột** ngay khi working set vượt sức chứa. Nối với **Miss Ratio Curve (MRC)** ở chuyên đề Eviction:

```
Miss ratio
 ▲
 │█                        ← working set >> cache: THRASHING (miss gần 100%)
 │ █
 │  █
 │   ██
 │     ████        ← "điểm gãy" (knee): vừa đủ chứa working set
 │         ██████████____  ← cache đủ lớn: thêm RAM gần như vô ích
 └────────────────────────► Cache size
   (bên TRÁI knee = vùng thrashing)
```

> Vận hành **bên trái điểm gãy** = đang ở/gần vùng thrashing. Chỉ cần **vượt knee một chút** là hit ratio bật lên hẳn → đây là lý do thrashing **cực nhạy với kích thước**: thiếu một chút RAM có thể là khác biệt giữa "tuyệt vời" và "thảm hoạ".

### 🧨 Bệnh kinh điển phải biết: "vòng lặp N+1" của LRU

Cho **LRU sức chứa N**, truy cập tuần hoàn **N+1 key khác nhau** (A,B,C,…,A,B,C,…):
- Mỗi lần cần một key, nó **vừa bị đuổi** ngay trước đó → **MISS 100%**, hit ratio = **0**.
- Thêm **đúng 1 slot** (cache size N+1) → **hit gần 100%**. Cực kỳ nhạy.

> 🎯 Bài học: (1) thrashing có thể đến từ **thiếu rất ít** dung lượng; (2) **LRU đặc biệt dễ dính** kiểu quét-vòng này → MRU/Random/CLOCK chịu vòng lặp tốt hơn LRU (nối Eviction doc: chọn policy theo pattern).

---

## ③ PHÂN BIỆT THRASHING với các "chế độ hỏng" anh em

Đây là phần **rất hay bị gộp nhầm** — ba thứ khác nhau về *bản chất thời gian*:

| | **Thrashing** | **Stampede** (chuyên đề trước) | **Cold start** (Cache warming) |
|---|---|---|---|
| Bản chất | **MÃN TÍNH** — trạng thái kéo dài | **SPIKE** — bùng một thời điểm | **TẠM THỜI** — rồi tự ấm |
| Phạm vi | **NHIỀU key** churn liên tục | **MỘT key** bị đàn đập | Nhiều key, nhưng **sẽ lấp đầy** |
| Gốc | Working set **> sức chứa** | Hot key miss đúng lúc đông | Cache **rỗng** ban đầu |
| Tự khỏi? | **KHÔNG** (không bao giờ đủ chỗ để ấm) | Qua đỉnh là hết | **CÓ** (cache fill dần) |
| Thuốc | Tăng cache / giảm working set / tier / admission | Lock · serve-stale · jitter | Warming |

> 🔑 Mấu chốt phân biệt: **Cold start cuối cùng SẼ ấm; thrashing thì KHÔNG BAO GIỜ ấm** vì cache không đủ giữ nổi working set. Đây là chi tiết hay bị nhầm nhất.

**Họ hàng gần — cache pollution (Eviction doc):** entry vô dụng (one-hit-wonder, key cũ-từng-hot) **choán chỗ** → **thu nhỏ sức chứa hiệu dụng** → đẩy hệ vào thrashing dù RAM danh nghĩa vẫn đủ. Pollution **là một nguyên nhân** của thrashing.

---

## ④ NGUYÊN NHÂN / NGÒI NỔ (phải nhận diện)

| Ngòi nổ | Mô tả | Nối với |
|---|---|---|
| **Working set thật > cache** | RAM cấp thiếu so với tập đang dùng | MRC, sizing (Eviction) |
| **Không có locality** | Truy cập ngẫu nhiên trên keyspace khổng lồ, không lặp | Bài 1: "không locality → cache vô dụng" |
| **Scan/batch quét sạch** | Job analytics đọc một-lần cuốn bay working set nóng | Eviction: LRU không scan-resistant |
| **Cache pollution** | One-hit-wonder được nhận bừa, đẩy hot set ra | Eviction: admission control |
| **Cardinality nổ** | Key gồm chiều cao-cardinality (user×time×requestId…) → mỗi key dùng một lần | Bài 1: key gồm `user` — nhưng đừng over-segment |
| **Cache nhỏ dần theo tăng trưởng** | Lúc launch vừa, data/traffic lớn lên mà chưa resize | Vận hành |
| **TTL quá ngắn** | Entry **chết trước khi** được dùng lại → churn giả-thrashing | Bài 1: TTL ↔ độ tươi |
| **Policy sai pattern** | LRU gặp vòng lặp; FIFO dính Belady anomaly | Eviction |
| **Noisy neighbor (multi-tenant)** | Một tenant working set khổng lồ đuổi cache của mọi người | Cô lập pool |

🎬 *Phỏng vấn: "Vì sao thêm cache mà p99/DB QPS không cải thiện?" (chính challenge Bài 1) → một khả năng lớn là **THRASHING**: hit ratio thấp vì working set không vừa / không có locality.*

---

## ⑤ ⭐ KHUNG "3 C's" — phân loại miss để định bệnh (staff framing)

Mô hình kinh điển phân loại **cache miss** thành 3 loại (gốc từ CPU cache, dùng được cho mọi cache):

| Loại miss | Nghĩa | Liên quan thrashing |
|---|---|---|
| **Compulsory** (bắt buộc / cold) | Lần đầu thấy key → **chắc chắn miss** | Cold start (không phải thrashing) |
| **Capacity** (sức chứa) | Working set **không vừa** → bị đuổi trước khi dùng lại | **ĐÂY CHÍNH LÀ THRASHING** |
| **Conflict** (xung đột) | Nhiều key **map cùng một "set"** (do associativity hạn chế) → đụng nhau dù cache chưa đầy | Thrashing kiểu CPU (conflict thrashing) |

> 🔑 Định bệnh nhanh: miss **Capacity** cao → tăng cache / giảm working set. Miss **Conflict** cao (ở tầng CPU/hardware hoặc cache có hashing kém) → tăng associativity / sửa layout. Miss **Compulsory** cao → warming. **Ba C, ba thuốc khác nhau.**

---

## ⑥ PHÁT HIỆN & ĐO (staff phải làm trước khi sửa)

| Tín hiệu | Ý nghĩa |
|---|---|
| **Eviction rate CAO + Hit ratio THẤP, đồng thời** | **Vân tay đặc trưng của thrashing** (nối Eviction doc) |
| **Reload rate ≈ eviction rate** | Đang churn — đuổi cái gì là nạp lại cái đó |
| **Vận hành bên TRÁI knee của MRC** | Thiếu dung lượng → thrashing; MRC cho biết cần thêm bao nhiêu |
| **Reuse distance > cache size** cho phần lớn lần dùng lại | Lý thuyết: chắc chắn miss khi tái sử dụng |
| **Hit ratio ~0 dù "đã có cache"** | Không locality / cardinality nổ |

```javascript
// Cảnh báo thrashing: hit ratio thấp VÀ eviction cao cùng lúc
const hitRatio = hits / (hits + misses);
const evictRate = evictedKeys / windowSeconds;
if (hitRatio < 0.5 && evictRate > THRESHOLD) {
  alert("THRASHING nghi vấn: working set có thể > cache. Đo MRC + reuse distance.");
}
```

**Ước lượng working set rẻ tiền** bằng HyperLogLog (đếm số key *duy nhất* trong cửa sổ):
```bash
# Redis: đếm xấp xỉ số key DUY NHẤT được hỏi trong 1 khung thời gian
redis> PFADD ws:5min  product:1 product:2 product:1 ...
redis> PFCOUNT ws:5min      # ~ kích thước working set → so với sức chứa cache
```
> Nếu **PFCOUNT (working set) ≫ số entry cache giữ nổi** → gần như chắc chắn thrashing.

---

## ⑦ CÁC GIẢI PHÁP (phần thịt) — 4 hướng tư duy

```
A. LÀM CACHE VỪA working set   → tăng RAM tới knee · admission control · scan-resistant policy
B. LÀM WORKING SET NHỎ LẠI     → top-K hot · giảm cardinality key · gom granularity thô hơn · multi-tier
C. CÔ LẬP & TÁCH                → pool theo tenant · tách cache batch vs interactive · bypass cache cho scan
D. CHẤP NHẬN: ĐỪNG CACHE        → nếu KHÔNG có locality, cache là phí tổn thuần → bỏ cache
```

### Nhóm A — Làm cache vừa working set

**A1. Tăng dung lượng — nhưng tới ĐIỂM GÃY thôi.** Dùng MRC để biết cần thêm bao nhiêu là vượt knee; **đừng đốt RAM** quá knee (vô ích — đúng tinh thần "đo trước khi thêm" của Bài 1).

**A2. Admission control (TinyLFU — Eviction doc).** Chặn **one-hit-wonder** chen vào → giữ hot set không bị churn cuốn đi → chữa thrashing **do pollution**. Đây là lý do Caffeine (W-TinyLFU) **không thrash** ở chỗ LRU thrash.

**A3. Policy chống quét (scan-resistant).** InnoDB midpoint-LRU / SLRU / ARC / SIEVE → một job quét **không cuốn bay** working set nóng → tránh thrashing do scan.

### Nhóm B — Làm working set nhỏ lại (thường rẻ & hiệu quả nhất)

**B1. Chỉ cache top-K hot, tier hoá.** L1 nhỏ giữ **vài key nóng nhất**, L2 lớn giữ phần còn lại (Bài 1 multi-tier). Mỗi tầng chỉ giữ thứ nó **đủ chỗ tái sử dụng**.

**B2. ⭐ Giảm cardinality của key — đòn mạnh nhất trong thực tế.** Đừng nhét chiều cao-cardinality vô nghĩa vào key:
```javascript
// ❌ SAI: key gồm requestId/timestamp → MỖI key dùng đúng MỘT lần → không bao giờ hit → thrash
const key = `product:${id}:${requestId}:${Date.now()}`;

// ✅ ĐÚNG: chỉ giữ chiều THỰC SỰ phân biệt kết quả (Bài 1: giá theo user thì cần `user`)
const key = `product:${id}`;            // hoặc `product:${id}:user:${userId}` nếu giá khác theo user
```
> Cardinality nổ là **thủ phạm thầm lặng số một**: hit ratio ~0 không phải vì cache nhỏ mà vì **mỗi key chỉ được hỏi một lần**.

**B3. Gom granularity thô hơn (tăng reuse).** Cache **trang danh mục** thay vì cache **từng tổ hợp filter** hiếm gặp → cùng RAM nhưng **tái sử dụng cao hơn nhiều**.

**B4. Tăng TTL nếu TTL-quá-ngắn gây churn** (cân với độ tươi — Bài 1). Entry chết trước khi dùng lại = lãng phí giống thrashing.

### Nhóm C — Cô lập & tách luồng

**C1. Pool/quota theo tenant** → một tenant không thể đuổi cache của mọi người (chống noisy-neighbor thrashing).
**C2. Tách cache batch vs interactive**, hoặc **bypass cache cho scan / đọc từ read-replica** → job analytics không làm bẩn cache nóng.

### Nhóm D — Đôi khi câu trả lời là "BỎ cache"

> Nếu path **không có locality** (Bài 1: mỗi lần một thứ hoàn toàn khác, chẳng cái nào lặp) → cache **không bao giờ hit**, chỉ thêm RAM + độ phức tạp. **Gỡ cache đi** là giải pháp đúng. Thrashing đôi khi là dấu hiệu **lẽ ra không nên cache chỗ này** (nối Bài 1: "4 trường hợp KHÔNG nên cache").

---

## ⑧ CASE STUDY THỰC TẾ

**1. ⭐ OS Virtual Memory thrashing (gốc của thuật ngữ — Denning, working set model).**
Đa chương quá mức → tổng working set các tiến trình > RAM → CPU dành gần hết thời gian **swap page** → throughput sụp. Thuốc: **working-set scheduling**, kiểm soát **tần suất page-fault**, **giảm mức đa chương**. ➜ Cùng tư duy: làm working set vừa bộ nhớ.

**2. MySQL InnoDB buffer pool thrashing (ops thường gặp).**
Working set (page nóng) > buffer pool → **đuổi page + đọc đĩa liên tục** → DB hoá I/O-bound, p99 vọt. Thuốc: **cấp đủ buffer pool** (rule of thumb ~70–80% RAM trên server DB chuyên dụng — *verify theo phiên bản/khối lượng*), và **midpoint-insertion LRU** giữ full-table-scan khỏi cuốn bay working set (Eviction doc).

**3. Redis / Memcached thrashing.**
`maxmemory` quá thấp so với working set → `evicted_keys` cao, hit ratio thấp. Thuốc: sizing theo MRC, **admission/LFU**, hoặc **shard** để tăng tổng RAM. (Memcached **slab calcification** ở Eviction doc chính là *thrashing trong một slab class* dù tổng RAM còn dư.)

**4. CPU cache — conflict thrashing & false sharing (tầng phần cứng).**
Nhiều địa chỉ map cùng một cache set → **conflict miss** dù cache chưa đầy. Đa lõi: hai core ghi hai biến **cùng một cache line** → **cache-line ping-pong (false sharing)** → line bị giằng qua lại liên tục = thrashing. Thuốc: **padding/alignment**, tăng associativity, `@Contended` (Java). ➜ Thrashing không chỉ ở tầng app.

**5. Caffeine / W-TinyLFU.**
Sinh ra một phần để **chống đúng thrashing kiểu pollution/scan** của LRU: admission control + frequency sketch → giữ hit ratio gần tối ưu ở workload mà LRU **thrash thảm hại**. Ví dụ điển hình "thuật toán đúng cứu được thrashing".

**6. CDN long-tail.**
Catalog video khổng lồ, working set > sức chứa edge → đuổi/nạp triền miên. Thuốc: **size/cost-aware eviction (GDSF)** + **tiered cache** + **admission** để chỉ giữ thứ đủ nóng (Eviction doc).

**7. "Thêm cache mà không nhanh hơn" — chính challenge Bài 1.**
Rất thường là **thrashing**: hit ratio thấp vì working set không vừa / cardinality nổ / không locality → **phải ĐO** (MRC, hit ratio, eviction rate) trước khi kết luận.

---

## ⑨ TRƯỜNG HỢP MỞ RỘNG / KIẾN THỨC NÂNG CAO

- **Reuse distance / stack distance (lý thuyết Mattson).** Với LRU, **miss ratio = tỉ lệ lần truy cập có reuse distance > cache size**. Đây là cách *nghiêm túc* để biết thứ gì đó **có thrash hay không** trước khi cấp RAM — không đoán cảm tính.
- **Performance cliff → nên over-provision NHẸ qua knee.** Vì thrashing là vách đá, giữ cache **nhỉnh hơn working set một chút** an toàn hơn nhiều so với cắt sát mép (một spike working set nhỏ cũng đủ đẩy qua vách).
- **Warming một working set KHÔNG vừa = thrash ngay lập tức.** Cache warming (chuyên đề trước) chỉ có lý nếu working set **vừa cache**; warm thứ không vừa = nạp vào để bị đuổi ngay → vô nghĩa. **Hai chuyên đề kiểm tra lẫn nhau.**
- **Liên hệ Amdahl (Bài 1 ➕).** Nếu cache thrash, "speedup" kỳ vọng **không bao giờ tới**; tệ hơn, bạn còn thêm overhead — tối ưu nhầm chỗ kiểu Amdahl.
- **Adaptive sizing / autoscale cache.** Hệ trưởng thành có **vòng phản hồi**: theo dõi hit ratio + eviction, **tự nới cache** khi phát hiện thrashing (trong giới hạn ngân sách RAM).
- **Conflict vs Capacity thrashing.** Nếu cache **chưa đầy** mà vẫn miss nhiều → nghi **conflict** (hashing/associativity kém), không phải thiếu RAM → sửa **layout/hash**, đừng đổ tiền mua RAM oan.

---

## ⑩ KHUNG QUYẾT ĐỊNH STAFF ENGINEER

```
1. ĐO trước: hit ratio + eviction rate + MRC + working set (HLL). Có đúng thrashing không?
   └─ Eviction cao + hit thấp + bên trái knee → ĐÚNG. Đi tiếp.

2. Có LOCALITY không? (working set có lặp lại không, hay mỗi key dùng một lần?)
   ├─ KHÔNG có locality / cardinality nổ → GIẢM CARDINALITY key; nếu vẫn không → BỎ CACHE
   └─ CÓ locality → sang 3

3. Working set CHỈ hơi vượt sức chứa?
   ├─ Tăng RAM tới KNEE (rẻ & dứt điểm nếu chỉ thiếu chút)
   └─ Hoặc THU NHỎ working set: top-K + tier (L1 nhỏ hot, L2 lớn)

4. Do SCAN/batch cuốn bay?
   └─ Policy SCAN-RESISTANT (InnoDB midpoint/SLRU/ARC/SIEVE) hoặc BYPASS cache cho scan

5. Do POLLUTION (one-hit-wonder)?
   └─ ADMISSION CONTROL (W-TinyLFU / Caffeine)

6. Do NOISY NEIGHBOR (multi-tenant)?
   └─ POOL/QUOTA theo tenant

7. Do TTL quá ngắn?
   └─ Tăng TTL (cân với độ tươi — Bài 1)
```

> 🥇 **Mặc định thực dụng:** **đo MRC + working set trước**. Nếu thiếu chút RAM → nới tới knee. Nếu cardinality nổ → **sửa key** (đòn rẻ & mạnh nhất). Nếu scan phá → **policy scan-resistant**. Nếu pollution → **admission control (Caffeine)**. Nếu không có locality → **đừng cache**.

---

## ⑪ BẪY THƯỜNG GẶP / 🔎 AI HAY SAI

- 🔎 **Thấy hit ratio thấp → vô tư tăng RAM.** Nếu là **không-locality / cardinality nổ** thì thêm RAM **chẳng cứu gì** — phải sửa **key/locality** trước. Tăng RAM chỉ chữa thrashing **do capacity**.
- 🔎 **Nhồi chiều cao-cardinality vào key** (requestId, timestamp) → mỗi key dùng một lần → hit ~0 → "tưởng cache hỏng" nhưng thật ra **tự gây thrashing**.
- 🔎 **Nhầm thrashing với cold start** → ngồi chờ "cache ấm dần" trong khi nó **không bao giờ ấm** vì không đủ chỗ.
- 🔎 **Warm một working set không vừa** → thrash tức thì, tốn công vô ích.
- 🔎 **Để batch/scan dùng chung cache nóng** → mỗi lần job chạy là cuốn bay working set → thrashing chu kỳ.
- 🔎 **Đổ tiền RAM cho conflict-miss** (cache chưa đầy mà vẫn miss) → sai bệnh; phải sửa hashing/layout.
- 🔎 **Quên đo, đoán cảm tính** (đúng cảnh báo Bài 1) → "thêm Redis cho nhanh" nhưng thrash, p99 không giảm.

---

## ⑫ MENTAL MODEL + CÂU THẦN CHÚ

> **Thrashing = cache hoá treadmill: đuổi-rồi-nạp đúng những thứ vừa bỏ đi vì chúng không vừa chỗ → làm cật lực, hit gần 0.**
> **Gốc duy nhất: working set > sức chứa.** Bốn lối thoát: **(A) cache to hơn** (tới knee), **(B) working set nhỏ hơn** (top-K · giảm cardinality · tier · granularity thô), **(C) chống quét/pollution & cô lập** (admission · scan-resistant · pool), **(D) nếu không có locality → ĐỪNG cache.**
> **Phân biệt:** cold start *sẽ* ấm; **thrashing KHÔNG bao giờ ấm**. **ĐO MRC + working set trước khi đổ RAM.**

---

## ⑬ CÂU HỎI PHỎNG VẤN

- **(dễ)** Thrashing là gì, nguyên nhân gốc? → cache đuổi-nạp triền miên vì **working set > sức chứa**; hit ratio sụp.
- **(TB)** Giải thích "vòng lặp N+1" của LRU. → truy cập vòng N+1 key trên cache N → mỗi key vừa bị đuổi → miss 100%; thêm 1 slot → hit ~100% (cực nhạy).
- **(TB)** Phân biệt **thrashing** vs **cold start** vs **stampede**. → mãn-tính-không-tự-ấm / tạm-thời-sẽ-ấm / spike-một-key.
- **(TB)** Kể "3 C's" của cache miss. → Compulsory (cold) · **Capacity (= thrashing)** · Conflict (associativity).
- **(khó)** Hit ratio ~0 dù cache không nhỏ — vì sao? → **không có locality** hoặc **cardinality key nổ** (mỗi key dùng một lần) → sửa key, đừng tăng RAM.
- **(khó)** Làm sao biết tăng RAM bao nhiêu là đủ? → đo **Miss Ratio Curve**, nới tới **điểm gãy (knee)**, đừng quá; dùng reuse distance / HLL working set.
- **(thách đố)** "Thêm cache mà p99 không giảm" (challenge Bài 1) — chẩn đoán? → có thể **thrashing**: working set không vừa / không locality / cardinality nổ → **đo hit ratio + eviction + MRC**, rồi chọn A/B/C/D.

---

## ⑭ GLOSSARY MỞ RỘNG (đóng vào hoàn cảnh)

| Thuật ngữ | Nghĩa nhanh | 🎬 Hoàn cảnh |
|---|---|---|
| **Thrashing** | Đuổi-nạp triền miên, hit gần 0 | *"`evicted_keys` cao + hit ratio thấp → **thrashing**."* |
| **Working set** | Tập data thực sự dùng trong một khoảng | *"**Working set** > cache → thrash; đo bằng HLL."* |
| **Miss Ratio Curve (MRC)** | Đường miss theo dung lượng; có "knee" | *"Vận hành bên trái **knee** = vùng thrashing."* |
| **Knee / điểm gãy** | Chỗ working set vừa khít cache | *"Nới RAM tới **knee** là đủ, quá thì phí."* |
| **3 C's** | Compulsory · Capacity · Conflict | *"Miss **Capacity** cao = thrashing."* |
| **Capacity miss** | Miss do không đủ chỗ giữ working set | *"Đây chính là **thrashing**."* |
| **Conflict miss** | Miss do nhiều key map cùng set | *"Cache chưa đầy vẫn miss → **conflict**, sửa hashing."* |
| **Reuse distance / stack distance** | Khoảng cách giữa hai lần dùng cùng key | *"Reuse distance > cache size → chắc chắn miss."* |
| **Cardinality nổ** | Key gồm chiều cao-cardinality → mỗi key dùng 1 lần | *"Bỏ `requestId`/`timestamp` khỏi key."* |
| **Cache pollution** | Entry vô dụng choán chỗ hot set | *"One-hit-wonder đẩy hot key ra → thrash."* |
| **Admission control** | Chặn key không xứng vào cache | *"W-TinyLFU chống pollution → chống thrash."* |
| **Scan resistance** | Không bị một lần quét cuốn sạch | *"InnoDB midpoint giữ scan khỏi flush working set."* |
| **False sharing / cache-line ping-pong** | Hai core giằng một cache line (đa lõi) | *"Pad biến / `@Contended` để hết ping-pong."* |
| **Performance cliff** | Hit ratio sụp đột ngột qua một ngưỡng | *"Thiếu chút RAM rơi khỏi vách → over-provision nhẹ."* |
| **Noisy neighbor** | Một tenant đuổi cache của mọi người | *"Pool/quota theo tenant chống thrash lây."* |

---

> 🧭 **Một dòng để mang theo:** *Thrashing là cái treadmill của cache — nó đuổi rồi nạp lại đúng những thứ vừa bỏ đi, vì working set không vừa chỗ. Trước khi đổ RAM, hãy ĐO (MRC + working set + reuse distance) để biết bệnh là **capacity** (→ to hơn / top-K / tier), **scan/pollution** (→ scan-resistant / admission), **cardinality nổ** (→ sửa key), hay **không có locality** (→ đừng cache). Cold start rồi sẽ ấm; thrashing thì không bao giờ.*
