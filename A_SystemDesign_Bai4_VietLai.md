# 🎯 BÀI 4 — Building blocks lõi

**Phủ:** A-BB-001 → A-BB-009

> 💡 **Tư tưởng cốt lõi của cả bài:** Đây là **"hộp lego"** của system design. Mọi **case study** sau (**Bài 9–15**) đều ráp lại từ chính những viên lego này. Học xong, khi gặp đề bạn **gọi tên đúng lego** thay vì **phát minh lại** từ đầu.

---

## ① Mục tiêu & vị trí trong mạch

Bài **"hộp lego"** của khối **Scaling primitives**. Nó tập hợp các kỹ thuật lõi dùng đi dùng lại: **CDN**, **object storage**, **consistent hashing**, **bloom filter**, **geo index**, **message queue**, **full-text search**, và **các tầng cache**.

> 🎯 **Vì sao học theo kiểu "gọi tên lego"?** Vì trong 45 phút phỏng vấn, bạn không có thời gian *nghĩ ra* giải pháp. Bạn cần *nhận diện* — "à, phần này là bài toán nearby → dùng **geohash**". Nhận diện nhanh = nửa trận đã thắng.

---

## ② Giảng cơ bản → nâng cao (gộp theo nhóm lego)

### 🧱 Nhóm phân phối nội dung & lưu trữ lớn

#### 📘 **A-BB-001 — CDN** (Content Delivery Network)

> **Ẩn dụ:** Thay vì ai cũng phải về *kho trung tâm* lấy hàng (origin), ta đặt **chi nhánh nhỏ ở từng khu phố** (edge gần user). Khách lấy hàng ở chi nhánh gần nhất → **nhanh** + **kho trung tâm đỡ tải**.

**CDN** cache **static / asset** gần user ⇒ **giảm latency** + **giảm tải origin**.

| Đặt LÊN CDN ✅ | KHÔNG đặt ❌ |
|---|---|
| ảnh, JS/CSS, video, **asset bất biến** | dữ liệu **cá nhân / động / nhạy cảm** |

> ⚠️ **Phần khó nhất:** cân nhắc **TTL** (Time To Live — bao lâu thì coi bản cache là cũ) + **invalidation** (làm mới khi nội dung đổi). **Cache invalidation là phần khó** *(➕ nhấn mạnh)*.

#### 📘 **A-BB-002 — Object storage vs database**

| Tiêu chí | **Object storage** (vd **S3**) | **Database** |
|---|---|---|
| Lưu gì | **file lớn** (blob): ảnh/video | **metadata + URL** |
| Đặc tính | rẻ / bền / scale | đắt, phình nếu nhồi blob |

> 🚨 **Cấm:** **KHÔNG nhồi blob** (ảnh/video) **vào DB**. DB chỉ giữ metadata + URL trỏ tới file. Phục vụ file qua **CDN** / **presigned URL**.
>
> 🔍 **Vì sao tệ nếu nhồi blob vào DB?** DB **phình to**, backup/replication chậm và đắt, và mọi truy vấn metadata bị "kéo" theo cục blob nặng. Tách ra: blob ở storage rẻ, DB nhẹ và nhanh.

> ⚠️ **Lưu ý (verify):** **verify tên dịch vụ** khi cần.

---

### 🧱 Nhóm phân phối key qua node

#### 📘 **A-BB-003 — Consistent hashing**

> **Ẩn dụ:** Hình dung một **mặt đồng hồ tròn** (vòng hash). Mỗi node ngồi ở một vị trí trên vòng. Mỗi key "đi theo chiều kim đồng hồ" tới node gần nhất. Thêm/bớt một node chỉ ảnh hưởng **cung nhỏ** quanh nó, không xáo trộn cả vòng.

**Consistent hashing** phân phối key qua node trên một **vòng hash**. Khi thêm/bớt node, chỉ **remap phần nhỏ key**.

So sánh với **hash % N** (modulo):

```
modulo:  node = hash(key) % N
N = 3 → 4:  thay đổi N → CÔNG THỨC đổi → GẦN NHƯ MỌI key đổi node
            ❌ cache miss bão hoà (cache rỗng đồng loạt → DB bị "dập")

consistent hashing:
N = 3 → 4:  chỉ ~1/4 key quanh node mới phải dời
            ✅ phần lớn key giữ nguyên node → cache vẫn ấm
```

> 🚨 **Điều tệ nhất với modulo:** đổi N (thêm/bớt một máy) khiến **gần như mọi key đổi node** ⇒ **cache miss bão hoà** ⇒ tất cả request đập thẳng xuống DB cùng lúc ⇒ DB sập. Một thao tác tưởng vô hại (thêm máy) lại gây sự cố.

> 💡 **Nền tảng cho:** cache cluster / shard cluster — **Redis Cluster**, **Cassandra**, **DynamoDB partitioning**.

#### 📘 **A-BB-004 — Virtual nodes**

**Virtual nodes (vnodes)** = rải **mỗi physical node** thành **nhiều điểm** trên vòng hash.

> 🔍 **Vì sao cần?** Nếu mỗi máy chỉ chiếm 1 điểm, phân phối dễ **lệch** (máy nào "ôm" cung dài thì quá tải = **hot node**). Rải mỗi máy thành 100–200 điểm → phân phối **đều hơn**, **tránh hot node**, **rebalance mượt** khi node chết/thêm.

---

### 🧱 Nhóm cấu trúc dữ liệu thông minh

#### 📘 **A-BB-005 — Bloom filter**

> **Ẩn dụ:** Một người gác cửa trí nhớ kém. Hỏi "anh A đã vào chưa?" — nếu gác cửa nói **"chắc chắn chưa"** thì tin tuyệt đối; nếu nói **"hình như rồi"** thì *có thể nhầm*, phải vào sổ kiểm tra lại.

**Bloom filter** là cấu trúc **xác suất** trả về **"có thể có / chắc chắn không"**:

- ✅ có **false positive** (báo "có" nhưng thực ra không).
- ❌ **KHÔNG có false negative** (đã báo "không" thì chắc chắn không).

> 💡 **Dùng để lọc trước, né đụng DB/cache:**
> - chống **cache penetration** — key **không tồn tại** đập thẳng xuống DB liên tục.
> - **check tồn tại nhanh** — vd **username đã dùng chưa**, **URL đã crawl chưa**.

#### 📘 **A-BB-006 — Geo / nearby index**

> **Ẩn dụ:** Tìm quán ăn "gần tôi" — bạn không đo khoảng cách tới *từng* quán trong thành phố. Bạn chia bản đồ thành **ô lưới**, chỉ xét các ô **kế bên ô của mình**.

Dùng **geohash** / **quad-tree** / **S2** để biến **tọa độ 2D** thành **key truy vấn được**; query theo **ô lưới lân cận**.

> 🚨 **Cấm:** **KHÔNG quét toàn bảng** tính khoảng cách từng điểm — chậm khủng khiếp khi có hàng triệu điểm.

---

### 🧱 Nhóm tách rời & tìm kiếm

#### 📘 **A-BB-007 — Message queue**

*(giao với **K** — đào sâu ở **Bài về Messaging**)*

> **Ẩn dụ:** Quán cà phê đông khách — thu ngân **không** tự pha từng ly rồi mới nhận khách tiếp. Họ ghi order vào **hàng đợi** (queue), barista lấy ra pha dần. Thu ngân và barista **tách rời** nhịp độ.

**Message queue** thêm vào để **decouple producer/consumer**, **hấp thụ burst** (buffer), **xử lý async + retry**.

| Được | Đổi lại (cái giá) |
|---|---|
| decouple, buffer burst, async, retry | thêm **latency** + **eventual consistency** + **phức tạp vận hành** |

> 💡 **Tín hiệu nên thêm queue:** tác vụ **nặng/chậm không cần đồng bộ** — gửi mail, **transcode** video, **fan-out**.

#### 📘 **A-BB-008 — Full-text search**

> **Ẩn dụ:** Tìm một từ trong cả nghìn cuốn sách. Bạn **không** lật từng trang từng cuốn (đó là `LIKE '%x%'`). Bạn dùng **bảng tra cứu cuối sách** (inverted index) chỉ thẳng "từ này nằm ở sách X trang Y".

Cần tìm văn bản ⇒ thêm **Elasticsearch / OpenSearch** (**inverted index** + relevance + scale).

> 🚨 **Cấm:** **KHÔNG dùng `LIKE '%x%'`** — không dùng được **index** ⇒ **quét tuyến tính** chậm.

| Được | Đổi lại |
|---|---|
| tìm văn bản nhanh, có relevance | phải **đồng bộ dữ liệu** từ DB nguồn (qua **CDC / dual-write**) + vận hành thêm một hệ |

> ⚠️ **Lưu ý (verify):** **verify tên / đời** sản phẩm.

---

### 🧱 Nhóm tổng hợp

#### 📘⬆️ **A-BB-009 — Cache đặt ở nhiều tầng** ⭐

*(giao với **J** — **Bài về Caching**)*

```
client → CDN/edge → API gateway → app local (in-process) → distributed (Redis) → DB cache
  (gần user)                           (cùng máy app)        (chia sẻ giữa máy)
```

Mỗi tầng **giảm tải tầng dưới** nhưng **thêm bài toán invalidation / stale** (dữ liệu cũ).

> 🎯 **Chọn tầng theo read pattern:**
>
> | Loại dữ liệu | Tầng cache hợp lý | Vì sao |
> |---|---|---|
> | data **toàn cục, ít đổi** | **CDN** | ai cũng đọc giống nhau, đặt gần user |
> | **per-user nóng** | **Redis** (distributed) | riêng từng user, chia sẻ giữa các máy app |
> | **cực nóng, cục bộ** | **in-process** (local) | nhanh nhất, chấp nhận **stale ngắn** |

> 💡 **Phân biệt 📘 vs ➕:** tất cả lego là kiến thức ổn định (📘). Ý *"invalidation là phần khó nhất của cache"* và *"CDC để đồng bộ search"* là ➕ nhấn mạnh.

---

## ③ ⚠️ Kiến thức cũ / hay bị làm sai

| Cũ / hay gặp (❌) | ✅ Nên dùng | Vì sao |
|---|---|---|
| **hash % N** để chia key qua cache/shard | **Consistent hashing + virtual nodes** | Đổi N ⇒ remap gần như toàn bộ key (**cache miss bão**) |
| Lưu ảnh/video trong cột **BLOB** của DB | **Object storage** + DB giữ **metadata/URL** | DB phình, đắt, khó scale; blob nên ở storage rẻ + **CDN** |
| **`LIKE '%keyword%'`** cho search | **Inverted index** (Elastic/OpenSearch) | `%x%` không xài **index** ⇒ chậm tuyến tính |
| **Cache mọi thứ "cho nhanh"** | Cache theo **read pattern** + có chiến lược **invalidation** | **Stale/invalidation** sai gây bug khó chịu hơn cả chậm |

> 🔍 **Đào sâu dòng cuối:** Vì sao "cache mọi thứ" lại nguy hiểm hơn cả chậm? Vì *chậm* thì user *thấy* và chấp nhận chờ; còn *stale* (trả dữ liệu cũ) thì user **không biết** mình đang xem sai — ví dụ thấy số dư cũ, đơn hàng đã hủy vẫn còn. Bug âm thầm khó truy hơn bug ồn ào.

---

## ④ Áp dụng thực tế + So sánh bigtech

- **Media-heavy** (ảnh/video): **object storage + CDN** là chuẩn ngành (pattern phổ biến) vì **cắt egress & latency**.
- **Cache/shard cluster**: **consistent hashing** là nền tảng của **Redis Cluster**, **Cassandra**, **DynamoDB** (**partition key + virtual nodes**) — **verify** chi tiết từng sản phẩm khi cần.
- **"Nearby"** (ride-hailing, food delivery): **geohash / S2** cho truy vấn lân cận; **bloom filter** hay gặp ở tầng **chống cache penetration** và **dedup crawler**.

---

## ⑤ Code thực hành + cấu hình

**Consistent hashing** tối giản để thấy *"thêm node remap ít key"*:

```python
# consistent_hash.py — chạy: python consistent_hash.py  (chỉ stdlib)
import hashlib, bisect

class ConsistentHash:
    def __init__(self, nodes=(), vnodes=100):   # vnodes = virtual nodes / physical node
        self.vnodes = vnodes
        self.ring = {}            # hash điểm -> node
        self.sorted_keys = []     # vòng hash đã sort (để tìm node theo chiều kim đồng hồ)
        for n in nodes: self.add(n)

    def _hash(self, key: str) -> int:
        return int(hashlib.md5(key.encode()).hexdigest(), 16)  # md5 chỉ để PHÂN PHỐI, KHÔNG bảo mật

    def add(self, node):
        for i in range(self.vnodes):                # rải 1 node thành nhiều điểm (virtual nodes)
            h = self._hash(f"{node}#{i}")
            self.ring[h] = node
            bisect.insort(self.sorted_keys, h)

    def get(self, key):           # key thuộc node nào?
        if not self.ring: return None
        h = self._hash(key)
        # đi theo chiều kim đồng hồ tới điểm node gần nhất:
        idx = bisect.bisect(self.sorted_keys, h) % len(self.sorted_keys)
        return self.ring[self.sorted_keys[idx]]

if __name__ == "__main__":
    ring = ConsistentHash(["A", "B", "C"], vnodes=200)
    sample = [f"key-{i}" for i in range(10_000)]
    before = {k: ring.get(k) for k in sample}      # map key->node TRƯỚC khi thêm node
    ring.add("D")                                  # thêm node thứ 4
    after = {k: ring.get(k) for k in sample}       # map key->node SAU khi thêm
    moved = sum(1 for k in sample if before[k] != after[k])   # đếm key phải dời
    print(f"Thêm 1 node (3->4): chỉ ~{moved/len(sample)*100:.1f}% key phải remap")
    # Kỳ vọng ~ 1/4 ~ 25% — so với modulo gần như 100%
```

```text
# Lưu ý cấu hình cho lego thật:
- CDN: đặt TTL hợp lý + cơ chế purge/invalidation khi asset đổi (versioned URL: app.v3.js).
- Object storage: dùng presigned URL cho upload/download trực tiếp, KHÔNG proxy blob qua app.
- Search: dựng pipeline đồng bộ (CDC/Debezium hoặc outbox) DB -> index; KHÔNG dual-write tay dễ lệch.
```

> 🔎 **Đọc kỹ vì sao "versioned URL":** Thay vì đi **purge** cache CDN mỗi lần đổi file (chậm, hay sót), bạn đổi *tên file* — `app.v3.js` → CDN coi đó là asset mới hoàn toàn, không cần invalidation. Né được "phần khó nhất của cache".

---

## ⑥ Keywords cần nhớ

🧠 **Cần HIỂU:**
- Vì sao **modulo hashing** tệ khi đổi N.
- **Bloom filter** *"false positive nhưng không false negative"*.
- Vì sao **blob** ra **object storage**.
- **Cache nhiều tầng** đánh đổi **stale**.

📌 **Cần THUỘC:**
- **Consistent hashing + virtual nodes**.
- **geohash / quad-tree / S2**.
- **inverted index**.
- **presigned URL**.
- các tầng cache (**client / CDN / gateway / local / distributed / DB**).

🛠️ **Cần LÀM ĐƯỢC:**
- Chọn **đúng lego** cho một yêu cầu.
- Cài **consistent hashing**.
- Thiết kế **pipeline đồng bộ DB → search**.

---

## ⑦ Mental model

> 🎯 **"Đừng phát minh lại lego."**
> **Blob → object storage + CDN; chia key → consistent hashing; tồn tại? → bloom; nearby → geohash; text → inverted index; nặng/async → queue.**

---

## ⑧ Câu hỏi phỏng vấn & thách đố

**(dễ)** **CDN** cache cái gì, KHÔNG cache cái gì?
→ **static/asset** gần user; **không** cache data **cá nhân/động/nhạy cảm**.

**(TB)** **Consistent hashing** giải quyết gì mà **modulo** không?
→ thêm/bớt node chỉ **remap phần nhỏ key** thay vì gần toàn bộ.

**(TB)** **Virtual nodes** để làm gì?
→ **phân phối đều** + tránh **hot node** + **rebalance mượt**.

**(khó)** **Bloom filter** dùng ở đâu, đặc tính gì?
→ lọc trước chống **cache penetration / dedup**; **false positive** có, **false negative** không.

**(khó)** Vì sao không **`LIKE '%x%'`**, thêm gì, trả giá gì?
→ không xài **index** ⇒ chậm; thêm **inverted index**; phải **đồng bộ data** + vận hành thêm hệ.

> 🧩 **Thách đố (trick):** Bloom filter báo **"có"** username `john` ⇒ bạn từ chối đăng ký luôn?
>
> ```
> bloom nói "CÓ"  →  ❌ TỪ CHỐI NGAY?      (sai — "có" = "CÓ THỂ có")
>                  →  ✅ đi xác nhận ở DB    (vì có false positive)
>
> bloom nói "KHÔNG" → ✅ tin tuyệt đối       (không có false negative → chắc chắn chưa dùng)
> ```
> 🔎 **Vì sao là bẫy?** **false positive** ⇒ "có thể có" **không** đồng nghĩa "chắc chắn có". Phải **xác nhận ở DB** khi bloom nói "có"; chỉ tin **tuyệt đối** khi bloom nói **"chắc chắn không"**.

---

## ⑨ Bài tập + tiêu chí tự chấm

**BT1.** Chạy `consistent_hash.py`, đổi **vnodes** từ 1 → 200, quan sát **độ lệch phân phối** & **% remap**.

> ✅ **Đạt khi:** giải thích được vì sao **vnodes lớn ⇒ phân phối đều hơn**.

**BT2.** Cho yêu cầu *"upload + phục vụ ảnh + tìm theo caption + đề xuất ảnh gần vị trí"*. Liệt kê **lego** cho từng phần.

> ✅ **Đạt khi:** ảnh → **object storage + CDN**; caption → **inverted index**; gần vị trí → **geohash**; (và metadata → **DB**).

---

## ⑩ Đọc thêm

- **Karger et al.** — *Consistent Hashing and Random Trees* (paper gốc).
- **AWS S3 docs** (**presigned URL**); **Elastic / OpenSearch docs** (**inverted index**); **Uber Eng blog** — **H3** (geo indexing).

> ⚠️ **Lưu ý (verify):** **verify đời / sản phẩm** khi trích số liệu.

---

> 🎯 **Khẩu quyết khép bài:** *Mỗi bài toán có một viên lego sẵn. Đừng phát minh lại — gọi đúng tên: object storage, consistent hashing, bloom, geohash, inverted index, queue.*
