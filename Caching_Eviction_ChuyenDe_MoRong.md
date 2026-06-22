# 🗑️ EVICTION — Chuyên đề mở rộng (Caching · nối tiếp Bài 1)
> **Đuổi ai khỏi cache khi cache đầy?** · Keyword · Concept · Thuật toán · Case study · Staff-level
> (Đọc sau khi đã nắm: *locality · 3 thứ đánh đổi · multi-tier L1/L2 · khi nào KHÔNG cache*)

---

## ① ĐỊNH VỊ — Eviction nằm ở đâu trong bức tranh cache

Nhớ lại tam giác đánh đổi gốc của Bài 1: **tốc độ ↔ độ tươi (freshness) ↔ bộ nhớ (RAM)**.

- **TTL / Invalidate** là cách quản trị cạnh **độ tươi** (giữ cache không bị *stale*).
- **Eviction** là cách quản trị cạnh **bộ nhớ** — vì RAM **hữu hạn**, mà *working set* (tập dữ liệu thực sự được dùng) thì có thể **lớn hơn cache**.

> 🔑 **Một câu chốt:** Cache có giới hạn dung lượng. Khi đầy mà có entry mới muốn vào, bạn **buộc phải đuổi một entry cũ ra**. Việc chọn "đuổi ai" chính là **eviction**. Chọn sai → đuổi nhầm thứ sắp được hỏi lại → **miss tăng → hit ratio tụt**.

### ⚠️ 3 cách một entry RỜI khỏi cache — đừng lẫn lộn

Đây là phần **rất hay bị nói nhầm** kể cả trong phỏng vấn. Ba thứ này khác nhau **về nguyên nhân**:

| Cơ chế | Nguyên nhân | Quản trị cạnh nào | Câu mẫu |
|---|---|---|---|
| **Expiration (TTL)** | **Hết hạn theo thời gian** | Độ tươi | *"Entry này TTL 5s, quá 5s tự biến mất."* |
| **Invalidation** | **Nguồn thay đổi** → bản cũ sai | Độ tươi / đúng đắn | *"Vừa ghi DB → xoá key để khỏi phục vụ đồ stale."* |
| **Eviction** | **Cache đầy** → cần chỗ cho entry mới | Bộ nhớ | *"RAM hết, phải đuổi key ít dùng nhất ra."* |

🎬 *Phỏng vấn hỏi "key biến mất, vì sao?" → liệt kê đúng **3 nguyên nhân độc lập** này là ăn điểm. TTL ≠ eviction.*

---

## ② TRỰC GIÁC — Bài toán eviction thực chất là gì

Cache đầy + entry mới đến → phải chọn **một "nạn nhân" (victim)** để đuổi.

**Mục tiêu lý tưởng:** đuổi đúng cái **sẽ KHÔNG được hỏi lại trong tương lai gần nhất** — để mỗi byte RAM luôn giữ thứ "đáng giá" nhất.

> Nhưng bạn **không biết tương lai**. Nên eviction về bản chất là một **bài toán DỰ ĐOÁN**: dùng quá khứ (vừa dùng? dùng nhiều?) để đoán tương lai (sắp dùng lại?). Đây chính là chỗ **locality** của Bài 1 quay lại:
> - Đặt cược **temporal locality** → "thứ vừa dùng dễ dùng lại" → sinh ra **LRU**.
> - Đặt cược **tần suất** → "thứ hay được dùng sẽ còn được dùng" → sinh ra **LFU**.

**Ẩn dụ tờ giấy dán màn hình (nối Bài 1):** bàn của bạn chỉ dán được **5 tờ giấy**. Cần tra con số thứ 6 → phải **bóc một tờ cũ** đi. Bóc tờ nào? Tờ lâu rồi không liếc tới (LRU)? Hay tờ ít khi phải tra nhất (LFU)? **Đó chính là eviction policy.**

---

## ③ BELADY — Thước đo "trần lý tưởng" (phải biết để so sánh)

Có một thuật toán **tối ưu tuyệt đối** gọi là **Belady's OPT (MIN)**:

> **Đuổi entry mà còn LÂU NHẤT nữa mới được dùng lại.**

- Đây là **giới hạn trên lý thuyết** — không thuật toán thực nào vượt qua được nó.
- Nhưng nó **cần biết trước tương lai** → **bất khả thi** trong thực tế.
- 👉 Công dụng: làm **thước chuẩn (benchmark)**. Mọi thuật toán thật (LRU, LFU…) đều chỉ là **xấp xỉ Belady** bằng cách đoán tương lai từ quá khứ. Khi research báo "thuật toán X đạt 95% hiệu năng của OPT" — nghĩa là gần chạm trần này.

🎬 *Khi ai đó khoe "policy của tôi tốt nhất", câu hỏi staff đặt ra là: **gần Belady cỡ nào, trên workload nào?***

---

## ④ CÁC THUẬT TOÁN KINH ĐIỂN (phải thuộc)

### B-1. Bảng tổng so sánh

| Policy | Đuổi ai | Đặt cược vào | Mạnh khi | Yếu / hỏng khi |
|---|---|---|---|---|
| **FIFO** | Vào sớm nhất (như xếp hàng) | Không gì cả (đơn giản) | Cần cực rẻ, đơn giản | Bỏ qua "đang hot"; dính **Belady's anomaly** |
| **LRU** | Lâu nhất chưa dùng | **Temporal locality** | Hầu hết workload web | **Bị scan quét sạch** (1 lần quét lớn xoá hết hot key) |
| **LFU** | Ít được dùng nhất | **Tần suất** | Vài key nóng ổn định dài hạn | **Cache pollution**: key cũ từng hot "ám" mãi không chịu đi |
| **MRU** | **Mới dùng nhất** | Mẫu "dùng xong là xong" | Quét tuần tự, bảng tạm | Hầu hết trường hợp web (ngược locality) |
| **Random** | Ngẫu nhiên | Không gì | **Đơn giản, rẻ, đủ tốt** ở quy mô lớn | Không khai thác locality tốt bằng LRU |
| **CLOCK / Second-chance** | LRU "giá rẻ" (xấp xỉ) | Temporal locality | Cần rẻ như FIFO + tốt gần LRU | Vẫn không scan-resistant |

### B-2. LRU — con ngựa thồ phổ biến nhất

- **Ý tưởng:** mỗi lần truy cập → đẩy entry lên "đầu" danh sách *mới-dùng-nhất*. Khi đầy → đuổi cái ở "đuôi" (*lâu nhất chưa đụng*).
- Khai thác **temporal locality** rất tốt → mặc định của đa số thư viện (`lru-cache` trong code Bài 1 chính là đây).
- ⚠️ **Tử huyệt — không chống quét (not scan-resistant):** một job analytics đọc **một lần** qua hàng triệu key sẽ **đẩy toàn bộ hot key ra khỏi LRU**, rồi chính những key quét-một-lần đó cũng vô dụng → **hit ratio sụp**. Đây là lý do các DB buffer pool (mục ⑦) **không xài LRU thuần**.

### B-3. LFU — và vì sao nó "ám" lâu

- **Ý tưởng:** đếm số lần mỗi entry được dùng; đầy thì đuổi cái **đếm thấp nhất**.
- Tốt khi có **nhúm key luôn nóng** trong thời gian dài.
- ⚠️ **Bệnh cache pollution / "stale frequency":** một key **hot hồi quá khứ** tích được counter rất cao, giờ **hết hot** nhưng counter vẫn lớn → **không bị đuổi**, choán chỗ key mới đang lên. ➜ LFU thực tế **phải có cơ chế lão hoá (aging/decay)** counter (xem Redis LFU ở mục ⑦).

### B-4. CLOCK (Second-Chance) — "LRU giá rẻ"

LRU thật phải cập nhật con trỏ **mỗi lần truy cập** (tốn). CLOCK xấp xỉ rẻ hơn:
- Xếp entry thành vòng tròn, mỗi entry có **1 bit "vừa dùng" (reference bit)**.
- Một "kim đồng hồ" quét vòng: gặp bit=1 → cho **cơ hội thứ hai**, gạt bit về 0, đi tiếp. Gặp bit=0 → **đuổi**.
- ➜ Gần bằng LRU nhưng **chỉ tốn 1 bit + thao tác O(1) phân bổ**. Đây là nền của PostgreSQL clock-sweep.

### 📌 Khái niệm phải nắm: SCAN RESISTANCE (chống quét)

> Một policy **scan-resistant** là policy **không bị một lần quét lớn (one-time scan) cuốn sạch** các hot key. LRU **không** có; ARC / LIRS / TinyLFU / SIEVE / InnoDB-midpoint-LRU **có**. Đây là tiêu chí phân loại quan trọng nhất khi chọn policy cho hệ thật.

### 🧨 Khái niệm phải nắm: BELADY'S ANOMALY

> Với **FIFO**, **cho cache TO HƠN lại có thể khiến MISS NHIỀU HƠN** — nghịch lý Belady. LRU và các "stack algorithm" **miễn nhiễm** (thêm RAM luôn ≥ tốt). ➜ Một lý do nữa để **không chọn FIFO thuần**.

---

## ⑤ THUẬT TOÁN HIỆN ĐẠI (state-of-the-art — phần nâng cao)

Vấn đề cốt lõi: **LRU mê recency nhưng mù frequency; LFU ngược lại.** Các thuật toán hiện đại cố **kết hợp cả hai** và **chống quét**.

| Thuật toán | Ý tưởng một dòng | Dùng ở đâu (thực tế) |
|---|---|---|
| **Segmented LRU (SLRU)** | Chia "probation" + "protected"; phải được hit **lần 2** mới lên khu bảo vệ → quét-một-lần không vào được | Memcached (HOT/WARM/COLD), nền của nhiều thiết kế |
| **2Q** | Hai hàng đợi: A1 (mới vào, FIFO) + Am (vào lần 2, LRU) — chống pollution | Một số DB / cache cũ |
| **LRU-K** | Đuổi theo lần truy cập **thứ K từ cuối** (K=2 phổ biến) → phân biệt "hot thật" vs "ngẫu nhiên" | Lý thuyết kinh điển, ảnh hưởng nhiều thiết kế |
| **LIRS** | Dùng **inter-reference recency**; rất scan-resistant | MySQL (biến thể), một số JVM cache |
| **ARC** (Adaptive Replacement Cache) | **Tự cân** recency↔frequency bằng 2 list thật (T1,T2) + 2 "ghost list" (B1,B2) học từ miss | **ZFS**, IBM storage (⚠️ vướng **bằng sáng chế** → Postgres né, không dùng) |
| **TinyLFU / W-TinyLFU** | **Admission control**: dùng *frequency sketch* (Count-Min Sketch) hỏi "key mới có **xứng đáng** chen vào không?" trước khi nhận | **Caffeine** (Java) — gần tối ưu, dùng rộng (Cassandra, Druid, Spring) |
| **S3-FIFO** (2023) | "Chỉ cần **3 hàng FIFO**" (Small/Main/Ghost) — đơn giản mà **vượt LRU** nhiều workload | Research CMU, đang lan vào sản phẩm |
| **SIEVE** (2024) | **Đơn giản hơn cả LRU**: 1 FIFO + 1 con trỏ "kim" + 1 bit visited; lazy promotion, scan-resistant | Đang nổi cho **web cache** |

### 📌 Khái niệm then chốt của thế hệ mới: ADMISSION CONTROL (kiểm soát NHẬN)

> LRU/LFU chỉ trả lời câu hỏi **"đuổi AI?"**. TinyLFU thêm câu hỏi đắt giá hơn: **"có nên NHẬN entry mới này không?"**
> Ý tưởng: trước khi cho key mới vào (và đuổi một victim), so sánh **tần suất ước lượng** của *key mới* với *victim sắp bị đuổi*. Nếu key mới còn ít được hỏi hơn cả victim → **từ chối nhận luôn**, giữ victim lại. ➜ Đây là chìa khoá giúp Caffeine **chống quét** và bám sát Belady.

**Vì sao W-TinyLFU thắng:** một cửa sổ **Window-LRU nhỏ (~1%)** đón "burst mới nổi" (recency), phần lớn còn lại là **SLRU** được bảo vệ bởi **bộ lọc admission TinyLFU** (frequency), cộng **cơ chế aging** (reset/halving sketch định kỳ) để key cũ-từng-hot phai dần. → Vừa recency vừa frequency vừa scan-resistant.

---

## ⑥ SIZE-AWARE & COST-AWARE EVICTION (cực quan trọng, hay bị bỏ qua)

Mọi policy ở trên ngầm giả định **mọi entry "to" như nhau và "đắt" như nhau**. **Sai trong nhiều hệ thật.**

### 🧱 Size-aware — khi object KÍCH THƯỚC chênh lệch (CDN/HTTP)

CDN cache đủ thứ: ảnh 2KB lẫn video 200MB. Đuổi **1 object 200MB** có thể giải phóng chỗ cho **100.000 object nhỏ**.
- ➜ Không thể tính theo *số lượng entry*, phải tính theo **bytes** và theo **"giá trị/byte"**.
- **GDS (Greedy-Dual-Size)** và **GDSF (Greedy-Dual-Size-Frequency):** xếp hạng entry theo công thức gộp **tần suất / kích thước / chi phí lấy lại** → ưu tiên giữ thứ **nhỏ + hay dùng + đắt để fetch lại**.

### 💸 Cost-aware — khi entry ĐẮT để tính lại (nối Bài 1: "cost/request")

Bài 1 đã nói **cache LLM call vì cost/request rất cao**. Đây chính là eviction cost-aware:
- Một entry là **kết quả LLM tốn $0.05 + 2 giây** thì **đắt hơn nhiều** một entry chỉ là một phép cộng.
- ➜ Khi đầy, nên **ưu tiên giữ entry đắt-tính-lại** dù nó ít được hỏi hơn, vì *miss* nó tốn hơn nhiều. Policy thuần LRU/LFU **không biết điều này** → phải tự gắn trọng số chi phí.

> 🔑 **Staff insight:** "Đuổi cái ít dùng nhất" là sai khi entry **không đồng giá**. Phải đuổi cái có **giá trị kỳ vọng thấp nhất = (xác suất hỏi lại × chi phí miss) / kích thước**.

---

## ⑦ IMPLEMENTATION CHI TIẾT (staff level)

### 7.1 — LRU O(1): HashMap + Doubly Linked List

LRU "sách giáo khoa" = **bản đồ băm** (tra O(1)) + **danh sách liên kết đôi** (di chuyển/đuổi O(1)):

```javascript
// Lõi LRU O(1) — minh hoạ cơ chế (thực tế dùng lru-cache như Bài 1)
class LRU {
  constructor(max) { this.max = max; this.map = new Map(); } // Map giữ thứ tự chèn!
  get(k) {
    if (!this.map.has(k)) return undefined;     // MISS
    const v = this.map.get(k);
    this.map.delete(k); this.map.set(k, v);     // chạm vào → đẩy lên "mới nhất"
    return v;                                   // HIT
  }
  set(k, v) {
    if (this.map.has(k)) this.map.delete(k);
    else if (this.map.size >= this.max) {
      const oldest = this.map.keys().next().value; // phần tử đầu = lâu nhất chưa dùng
      this.map.delete(oldest);                     // 🗑️ EVICT
    }
    this.map.set(k, v);
  }
}
```

> 💡 Mẹo JS: `Map` **giữ thứ tự chèn**, nên `delete` rồi `set` lại = "đưa lên đầu". Trong ngôn ngữ khác bạn phải tự cài doubly-linked-list.

### 7.2 — Vì sao Redis KHÔNG dùng LRU thật mà dùng LRU "xấp xỉ"

LRU chính xác cần **con trỏ prev/next cho mỗi key** → **tốn RAM** và đụng nhiều cache-line. Với hàng trăm triệu key, chi phí này **không đáng**.

➜ **Redis dùng LRU xấp xỉ bằng lấy mẫu (sampling):** khi cần đuổi, **bốc ngẫu nhiên `maxmemory-samples` key** (mặc định **5**), đuổi cái "cũ nhất" trong mẫu đó. Tăng samples → gần LRU thật hơn nhưng tốn CPU hơn. **Triết lý: gần-đúng-mà-rẻ thắng đúng-tuyệt-đối-mà-đắt.**

**Các `maxmemory-policy` của Redis cần thuộc:**

| Policy | Hành vi |
|---|---|
| `noeviction` | **Không đuổi** — hết RAM thì **lệnh ghi báo lỗi** (⚠️ nguy hiểm nếu không lường) |
| `allkeys-lru` / `allkeys-lfu` / `allkeys-random` | Đuổi trên **mọi key** theo LRU / LFU / ngẫu nhiên |
| `volatile-lru` / `volatile-lfu` / `volatile-random` | Chỉ đuổi key **có đặt TTL** |
| `volatile-ttl` | Đuổi key **sắp hết hạn nhất** |

> ⚠️ **Bẫy `volatile-*`:** nếu **không key nào có TTL**, Redis **không có gì để đuổi** → hành xử như `noeviction` → **lỗi ghi**. Rất hay sập production vì hiểu nhầm chỗ này.

**Redis LFU (từ 4.0):** không đếm thật (tốn) mà dùng **bộ đếm xác suất kiểu Morris 8-bit** — càng cao càng khó tăng (`lfu-log-factor`), và **lão hoá theo thời gian** (`lfu-decay-time`) để chữa bệnh "key cũ ám mãi" của LFU.

### 7.3 — Memcached: slab allocation + bệnh "slab calcification"

Memcached chia RAM thành **slab class** theo cỡ object (96B, 120B, …). LRU chạy **riêng từng slab class**.
- 🧨 **Slab calcification:** RAM đã trao cho slab class này thì **khó chuyển** sang class khác. Nếu pattern đổi (xưa toàn object nhỏ, nay toàn object lớn), class lớn **đói RAM và evict điên cuồng** dù tổng RAM còn trống ở class nhỏ. ➜ Memcached thêm **slab automover / rebalancer** và **Segmented LRU (HOT/WARM/COLD/TEMP)** để chống quét và phân bổ lại.

### 7.4 — CPU cache & DB buffer pool (eviction ở tầng thấp nhất — nối Bài 1 ⑥ "DB buffer pool")

- **CPU cache (L1/L2/L3 phần cứng):** không đủ transistor cho LRU thật → dùng **Pseudo-LRU (tree-PLRU)** hoặc **NRU** — xấp xỉ bằng vài bit.
- **PostgreSQL shared buffers:** dùng **clock-sweep** (biến thể CLOCK + usage_count), **không LRU**, để rẻ và chống quét; cũng vì ARC **vướng patent**.
- **MySQL InnoDB buffer pool:** **LRU chèn-giữa (midpoint insertion)** — key mới **không** vào thẳng đầu mà vào điểm 3/8 ("old sublist", `innodb_old_blocks_pct ≈ 37%`); phải bị đọc lại sau một khoảng mới được thăng lên "young". ➜ **Một job full-table-scan không thể đẩy hot page ra** = **scan-resistant** rất khéo.
- **Oracle:** dùng **touch-count** (LRU sửa đổi theo số lần chạm).

> 🎯 **Bài học xuyên suốt 7.2–7.4:** *Mọi hệ nghiêm túc đều KHÔNG dùng LRU thuần.* Hoặc **xấp xỉ cho rẻ** (Redis sampling, CLOCK, PLRU), hoặc **gia cố cho chống quét** (InnoDB midpoint, SLRU, ARC, TinyLFU).

---

## ⑧ CASE STUDY THỰC TẾ

**1. Redis — hành trình tiến hoá policy.** Ban đầu xấp xỉ thô; rồi cải tiến **sampled-LRU** (chọn cái cũ nhất trong N mẫu); rồi thêm hẳn **LFU xác suất có lão hoá** (4.0) để phục vụ workload "vài key siêu nóng". Tinh thần: **chọn cái gần-đúng-mà-rẻ**, để người vận hành tinh chỉnh qua `maxmemory-samples`.

**2. Facebook / Meta — Memcached, CacheLib, Segcache.** Meta vận hành Memcached ở quy mô khổng lồ, đối mặt **slab calcification** và **lease** (chống stampede). Sau đó họ rút lõi caching thành **CacheLib** (mã nguồn mở, OSDI 2020) — một engine dùng chung cho rất nhiều dịch vụ, với eviction lai (LRU + 2Q-like + TTL) và quản trị bộ nhớ tinh vi. Twitter ra **Segcache** (NSDI 2021) tổ chức cache theo **segment gom theo TTL** → đuổi/hết-hạn theo cả khối, giảm metadata.

**3. Caffeine (Java) — W-TinyLFU thành chuẩn de-facto.** Ben Manes chứng minh **admission control + frequency sketch** đạt hit ratio **gần Belady**, vượt Guava LRU rõ rệt. Nay là cache mặc định trong **Cassandra, Druid, Neo4j, Spring** … Đây là ví dụ điển hình "thuật toán hiện đại đánh bại LRU trong sản phẩm thật".

**4. CDN (Akamai/Cloudflare) — size-aware là bắt buộc.** Vì object chênh lệch từ KB tới GB và "giá fetch lại từ origin" khác nhau, CDN dùng các biến thể **GDSF / cost-size-aware**, không phải LRU thuần — đuổi sao cho **bytes giải phóng** và **giá trị giữ lại** tối ưu.

**5. Làn sóng research mới (CMU group).** **S3-FIFO** (SOSP 2023) và **SIEVE** (NSDI 2024) chỉ ra: trên các **web cache workload** thật, **các cấu trúc cực đơn giản dựa trên FIFO** lại **vượt LRU** cả về hit ratio lẫn throughput/đa luồng — vì FIFO ít tranh chấp khoá hơn LRU. Bài học staff: **"phức tạp hơn" không tự động "tốt hơn"; phải đo trên workload thật.**

---

## ⑨ EVICTION TRONG MULTI-TIER (nối thẳng Bài 1 mục d: L1 + L2)

Bài 1 dạy multi-tier L1 (local) + L2 (Redis). Eviction ở đây có thêm vấn đề:

- **L1 và L2 evict ĐỘC LẬP nhau.** L1 mỗi node một bản, dung lượng nhỏ → **evict rất nhanh**; L2 lớn hơn → giữ lâu hơn. Một key bị **L1 đuổi nhưng L2 còn** là chuyện thường (và tốt: L2 đỡ cho miss).
- **Inclusive vs Exclusive:** L1 có **buộc phải là tập con** của L2 không? Nếu inclusive, khi L2 đuổi một key thì **cũng nên báo L1 đuổi** (nếu không L1 giữ thứ L2 đã bỏ → khó suy luận về độ tươi). Thường người ta chọn lỏng + **TTL L1 ngắn** (như code Bài 1: L1 TTL 5s) để tự lành.
- 🌩️ **Eviction storm / thundering herd (nối "stampede" ở bộ học thuộc):** nếu **một loạt key bị evict (hoặc TTL hết) cùng lúc**, request đồng loạt **miss và đập xuống nguồn** → có thể sập DB. Hai liều thuốc (đã có trong bộ học thuộc của bạn): **jitter TTL** (cộng nhiễu ngẫu nhiên để key không hết hạn cùng lúc) + **singleflight/request coalescing** (gộp nhiều miss cùng key thành 1 lần đi nguồn).

---

## ⑩ VẬN HÀNH & METRIC (staff phải nhìn)

| Metric / dấu hiệu | Ý nghĩa | Hành động |
|---|---|---|
| **`evicted_keys` / eviction rate** | Bao nhiêu key bị đuổi/giây | Tăng vọt = cache **quá nhỏ so với working set** |
| **Hit ratio tụt + eviction cao đồng thời** | Dấu hiệu **THRASHING** (Bài 1 ➕): đuổi rồi nạp lại liên tục | Tăng RAM / siết working set / dùng admission control |
| **`noeviction` + lỗi ghi** | Hết RAM, Redis từ chối ghi | Đặt `maxmemory-policy` hợp lý + cảnh báo dung lượng |
| **Tỉ lệ key có TTL** (với `volatile-*`) | Nếu ~0% → không có gì để đuổi → kẹt | Đổi sang `allkeys-*` hoặc đảm bảo set TTL |

### 📏 Định cỡ cache (sizing) — "RAM bao nhiêu là đủ?"

- Mục tiêu: cache đủ chứa **working set**, **không cần** chứa toàn bộ data (Bài 1 ➕).
- Công cụ staff: **Miss Ratio Curve (MRC)** — vẽ "miss ratio theo dung lượng cache". MRC cho thấy **điểm gãy (knee)**: vượt điểm đó, thêm RAM **gần như không giảm miss** nữa → đừng đốt tiền RAM vô ích.
- Ước lượng MRC rẻ trên log production bằng **SHARDS** / lấy mẫu thay vì mô phỏng toàn bộ.

> Đây chính là tinh thần **"đo trước khi thêm tầng"** của Bài 1, áp vào eviction: **đo MRC trước khi quyết định cấp thêm RAM hay đổi policy.**

---

## ⑪ KHUNG QUYẾT ĐỊNH STAFF ENGINEER — chọn policy thế nào

```
1. Object ĐỒNG CỠ & ĐỒNG GIÁ không?
   ├─ KHÔNG (CDN, kích thước/chi phí lệch lớn)  → SIZE/COST-AWARE (GDSF, cost-weighted)
   └─ CÓ → sang bước 2

2. Workload có bị QUÉT (scan/analytics/batch) không?
   ├─ CÓ  → cần SCAN-RESISTANT: W-TinyLFU (Caffeine) / SLRU / ARC / InnoDB-midpoint / SIEVE
   └─ KHÔNG → sang bước 3

3. Đặc tính truy cập?
   ├─ Vài key NÓNG ỔN ĐỊNH dài hạn      → LFU (NHỚ bật aging/decay)
   ├─ "Vừa-dùng-dễ-dùng-lại" là chính   → LRU (xấp xỉ là đủ)
   └─ Quy mô cực lớn, cần rẻ/đa luồng    → Random / CLOCK / S3-FIFO

4. Dù chọn gì:
   ├─ KHÔNG cài LRU "thật" tốn pointer ở quy mô lớn → xấp xỉ (sampling/CLOCK)
   ├─ ĐO eviction rate + hit ratio + MRC, đừng tối ưu hit ratio mù quáng (Bài 1)
   └─ Bọc chống stampede: jitter TTL + singleflight
```

> 🥇 **Mặc định an toàn 2025:** nếu phân vân và đang ở tầng app (Java) → **Caffeine (W-TinyLFU)**. Ở L2 → **Redis `allkeys-lru`** (hoặc `allkeys-lfu` nếu có hot key rõ rệt), `maxmemory-samples` 5–10. **Đừng bao giờ để mặc định `noeviction` mà không cảnh báo dung lượng.**

---

## ⑫ BẪY THƯỜNG GẶP / 🔎 AI HAY SAI

- 🔎 **Lẫn TTL với eviction.** "Set TTL rồi khỏi lo đầy RAM" — **sai**: TTL kiểm soát *độ tươi*, eviction kiểm soát *dung lượng*. Key chưa hết TTL **vẫn bị evict** nếu cache đầy.
- 🔎 **Mặc định LRU cho mọi thứ.** AI hay bê LRU vào cả path **analytics/scan** → bị quét cuốn sạch. Hỏi: *"workload này có quét không?"*
- 🔎 **Tin LFU mà quên aging.** Không có decay → key cũ-từng-hot **ám vĩnh viễn**.
- 🔎 **Quên size/cost.** Đối xử object 200MB như object 2KB; coi kết quả LLM đắt ngang phép cộng.
- 🔎 **Để `volatile-*` mà không key nào có TTL** → Redis kẹt như `noeviction`.
- 🔎 **Tối ưu hit ratio mù quáng** (đúng cảnh báo Bài 1): hit ratio đẹp nhờ giữ thứ rẻ/ít gọi, nhưng **không bảo vệ được DB QPS/p99**.

---

## ⑬ MENTAL MODEL + CÂU THẦN CHÚ

> **Eviction = trả lời "đuổi AI khi cache đầy", bằng cách ĐOÁN tương lai từ quá khứ.**
> **LRU** đoán bằng *recency*, **LFU** đoán bằng *frequency*, **thế hệ mới** (TinyLFU/ARC/SIEVE) **gộp cả hai + chống quét**.
> **Trần lý tưởng là Belady. Hệ thật luôn XẤP XỈ Belady cho rẻ, và GIA CỐ để chống quét.**
> **Khi object không đồng giá → đuổi theo (xác suất hỏi lại × chi phí miss) / kích thước, không chỉ "ít dùng nhất".**

---

## ⑭ CÂU HỎI PHỎNG VẤN

- **(dễ)** Phân biệt **TTL · invalidation · eviction**. → 3 nguyên nhân rời cache: hết-hạn / nguồn-đổi / đầy-RAM.
- **(TB)** LRU vs LFU khác gì, mỗi cái hỏng khi nào? → LRU mê recency, **bị quét cuốn sạch**; LFU mê frequency, **key cũ ám mãi nếu không aging**.
- **(TB)** Vì sao Redis dùng LRU **xấp xỉ** chứ không LRU thật? → pointer/metadata tốn RAM ở quy mô lớn → **sampling N key, đuổi cũ nhất trong mẫu**; rẻ-mà-đủ-tốt.
- **(khó)** "Scan-resistant" là gì, kể 3 cách đạt được? → không bị one-time-scan cuốn; **InnoDB midpoint insertion**, **SLRU/2Q (phải hit lần 2)**, **admission control TinyLFU / ARC / LIRS**.
- **(khó)** Cache CDN nên đuổi theo gì? → **size/cost-aware (GDSF)**: giữ thứ **nhỏ + hay dùng + đắt fetch lại**, không theo số lượng entry.
- **(khó)** `maxmemory-policy=volatile-lru` nhưng app **không set TTL** cho key → chuyện gì xảy ra? → không có ứng viên để đuổi → Redis **báo lỗi ghi như noeviction**.
- **(thách đố)** Thêm RAM cho cache mà hit ratio gần như không tăng — vì sao? → đã **qua điểm gãy của Miss Ratio Curve** (working set nằm gọn rồi); hoặc bottleneck **không ở dung lượng** mà ở **policy sai** (đang bị quét) → **đo MRC trước khi đốt RAM**.

---

## ⑮ GLOSSARY MỞ RỘNG (đóng vào hoàn cảnh)

| Thuật ngữ | Nghĩa nhanh | 🎬 Hoàn cảnh |
|---|---|---|
| **Eviction** | Đuổi entry khi cache đầy | *"`evicted_keys` tăng vọt → cache quá nhỏ."* |
| **Victim** | Entry bị chọn để đuổi | *"LRU chọn **victim** ở đuôi danh sách."* |
| **Belady / OPT (MIN)** | Policy tối ưu lý thuyết (cần biết tương lai) | *"Caffeine đạt ~95% **Belady** trên workload này."* |
| **Belady's anomaly** | FIFO: thêm RAM lại tăng miss | *"Đừng dùng FIFO thuần — dính **anomaly**."* |
| **Scan resistance** | Không bị một lần quét cuốn sạch | *"Analytics job đừng làm sập cache → cần **scan-resistant**."* |
| **Admission control** | Quyết định có **NHẬN** key mới không | *"TinyLFU hỏi 'key mới có xứng chen vào?' trước khi đuổi victim."* |
| **Cache pollution** | Cache đầy thứ vô dụng (vd key cũ-từng-hot) | *"LFU thiếu aging → **pollution**."* |
| **Thrashing** | Đuổi rồi nạp lại liên tục, hit ratio tụt | *"Working set > cache → **thrashing**."* |
| **CLOCK / second-chance** | Xấp xỉ LRU bằng 1 bit + kim quay | *"Postgres clock-sweep là họ **CLOCK**."* |
| **SLRU** | LRU phân khu, phải hit lần 2 mới được bảo vệ | *"Memcached HOT/WARM/COLD là **SLRU**."* |
| **ARC** | Tự cân recency↔frequency, có ghost list | *"ZFS dùng **ARC**; Postgres né vì patent."* |
| **W-TinyLFU** | Window-LRU + admission frequency-sketch | *"Caffeine = **W-TinyLFU**, gần tối ưu."* |
| **GDSF** | Đuổi theo tần suất / kích thước / chi phí | *"CDN dùng **GDSF** vì object chênh cỡ lớn."* |
| **Miss Ratio Curve (MRC)** | Đường miss theo dung lượng cache | *"Đo **MRC** để biết cấp thêm RAM có đáng không."* |
| **Slab calcification** | RAM kẹt trong slab class cũ (Memcached) | *"Pattern đổi → class lớn evict điên dù còn RAM ở class nhỏ."* |
| **`maxmemory-policy`** | Chính sách evict của Redis | *"Set `allkeys-lru`, đừng để mặc định `noeviction`."* |

---

> 🧭 **Một dòng để mang theo:** *TTL lo độ tươi — Eviction lo dung lượng. Và "đuổi cái ít dùng nhất" chỉ là điểm khởi đầu: hệ thật phải chống quét, biết aging, và đo lường (xác suất hỏi lại × chi phí) / kích thước — chứ không tối ưu hit ratio một cách mù quáng.*
