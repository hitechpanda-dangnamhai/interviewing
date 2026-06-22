# ⏳ CHUYÊN ĐỀ — EVENTUAL CONSISTENCY (Nhất quán cuối cùng) trong Caching
> Đào sâu một concept mở rộng của **Bài 1 — Vì sao cache · Các tầng cache · Khi nào KHÔNG cache**
> Mục tiêu: hiểu *bản chất*, biết *chọn mô hình nhất quán*, biết *bound độ stale* và *tránh các race* ở mức **staff engineer**.
> (✅ = đã có trong Bài 1 · ➕ = mở rộng thêm)

---

## ① Mục tiêu & vị trí trong mạch

Trong Bài 1, **Eventual consistency** xuất hiện ở **bảng concept mở rộng (B2)**:

> ➕ **Eventual consistency** — *Cuối cùng sẽ đồng nhất, nhưng có độ trễ* — *"Like count xài **eventual consistency** được, lệch vài giây không sao."*

Và nó đứng **đối cực** với một concept cốt lõi (B1):

> ✅ **Strong consistency** — *Đọc luôn ra giá trị mới nhất, tuyệt đối đúng* — *"Path tiền / tồn kho cần strong consistency → đừng cache."*

Chuyên đề này "mở hộp" cặp đối cực đó. Học xong bạn sẽ:

- Hiểu **eventual consistency là gì** và vì sao **mọi cache về bản chất là một cỗ máy eventual-consistent**.
- Nắm **phổ nhất quán** (spectrum): từ strong → bounded staleness → read-your-writes → causal → eventual.
- Hiểu **CAP / PACELC** đứng sau lựa chọn này, và vì sao Bài 1 nói *"đừng cache path tiền/tồn kho"*.
- Biết **các pattern staff-level** để *bound* độ stale và *tránh race*: TTL, invalidation đúng thứ tự, **leases**, **CDC**, **read-your-writes**, **versioning/CAS**, **CRDT/quorum**.

> 💡 **Một câu để gắn vào mạch Bài 1:** Khi bạn đặt một bản sao dữ liệu vào cache, bạn **đã chọn eventual consistency** rồi — dù có gọi tên nó hay không. "Stale" mà cả Phase Caching xoay quanh chính là **cửa sổ bất nhất (inconsistency window)** của eventual consistency. Vấn đề không phải *"có bất nhất không"* (có), mà *"cửa sổ đó rộng bao lâu và path này có chịu được không"*.

---

## ② Trực giác — Eventual consistency là gì?

**Eventual consistency (nhất quán cuối cùng) = NẾU ngừng cập nhật, thì SAU MỘT KHOẢNG THỜI GIAN, mọi bản sao sẽ hội tụ về cùng một giá trị mới nhất.**

Ba ý cần khắc:
- **"Cuối cùng" (eventually)** — có một **cửa sổ thời gian** mà các bản đọc *có thể* trả ra giá trị cũ.
- **"Hội tụ" (converge)** — đảm bảo *cuối cùng* sẽ đúng, không kẹt sai mãi.
- **Không hứa "ngay lập tức"** — đó là điểm khác strong consistency.

### Ẩn dụ tin đồn trong văn phòng

Sếp đổi giờ họp từ 3h sang 4h. Sếp báo cho vài người, họ báo tiếp người khác. Trong **vài phút**, một số người vẫn tưởng 3h (đọc giá trị cũ). Nhưng **cuối cùng** ai cũng biết là 4h — hệ **hội tụ**.

- **Tin đồn lan ra = quá trình hội tụ (replication / invalidation).**
- **Vài phút loạn thông tin = cửa sổ bất nhất (staleness window).**
- **Cuối cùng ai cũng biết 4h = eventual consistency.**

Nếu là **strong consistency**: không ai được phép biết giờ họp cho tới khi *tất cả* cùng được cập nhật đồng thời — chính xác nhưng **chậm và dễ kẹt** nếu một người vắng mặt (partition).

### Nối thẳng với "stale" của Bài 1

| Bài 1 nói | Tên hình thức |
|---|---|
| "Cache bị **stale**" | Đang ở trong **cửa sổ bất nhất** |
| "Đặt **TTL** để giới hạn thời gian bị stale" | **Bounded staleness** (giới hạn độ cũ) |
| "Mỗi lần ghi DB phải **invalidate**" | Cơ chế **rút ngắn cửa sổ hội tụ** |
| "**Source of truth** mới luôn đúng" | Strong consistency *ở nguồn*, eventual *ở bản sao* |

> 🔑 **Insight #1:** Cache **không phá vỡ** tính đúng đắn của hệ — nó **đổi mô hình nhất quán** từ strong (ở DB) sang eventual (ở bản sao). Câu hỏi thiết kế là: *"Path này chịu được mô hình eventual tới mức nào?"*

---

## ③ Phổ nhất quán (consistency spectrum)

Không chỉ có "strong" và "eventual" — có cả một dải ở giữa. Từ **mạnh nhất (chậm/đắt)** xuống **yếu nhất (nhanh/rẻ)**:

```
MẠNH ↑  Linearizable (strong)   → đọc LUÔN ra ghi mới nhất, như một bản duy nhất
        Sequential              → mọi node thấy CÙNG một thứ tự thao tác
        Causal                  → giữ quan hệ nhân-quả (A gây ra B thì ai cũng thấy A trước B)
        Read-your-writes        → bạn LUÔN thấy ghi của CHÍNH MÌNH
        Monotonic reads         → đã thấy giá trị mới thì không bao giờ thấy lại giá trị cũ
        Bounded staleness       → cũ tối đa X giây / X phiên bản (TTL là dạng này)
YẾU  ↓  Eventual                → cuối cùng hội tụ, không hứa thứ tự / thời điểm
```

> 🔑 **Insight #2:** "Eventual consistency" thường bị dùng như cái thùng rác cho "mọi thứ không strong". Nhưng staff engineer chọn **đúng nấc**: like-count thì eventual là đủ; *profile của chính user* thì cần **read-your-writes** (nếu không user đổi avatar xong F5 thấy avatar cũ → tưởng bug); feed thì cần **monotonic reads** (đừng để bài mới hiện rồi biến mất khi load lại). **Chọn nấc yếu nhất mà UX vẫn chấp nhận được** — đó là nghệ thuật.

---

## ④ Vì sao phải có eventual consistency? — CAP & PACELC

### CAP theorem (Bài 1 B2)

> Khi mạng **phân mảnh (partition)**, hệ phân tán chỉ được chọn **một trong hai**: **Consistency** (mọi đọc ra giá trị mới nhất) hoặc **Availability** (vẫn trả lời).

- Chọn **C** (CP): khi mất liên lạc giữa các bản, **từ chối trả lời** để khỏi trả đồ cũ → mất availability.
- Chọn **A** (AP): vẫn trả lời bằng bản *có thể cũ* → **eventual consistency**.

Cache gần như luôn nghiêng **AP**: bạn muốn cache **vẫn trả lời nhanh** kể cả khi nó chưa kịp đồng bộ với DB → chấp nhận stale.

### PACELC — bản mở rộng quan trọng hơn cho cache

> **P**artition: chọn **A** hay **C**. **E**lse (lúc bình thường, *không* partition): vẫn phải chọn **L**atency hay **C**onsistency.

PACELC bắt đúng tinh thần cache: **ngay cả khi mạng khoẻ mạnh**, bạn vẫn đánh đổi **độ trễ (latency) ↔ nhất quán (consistency)**. Đọc cache = chọn **L** (nhanh) và hi sinh **C** (có thể cũ). Đây chính là **ba thứ đánh đổi của Bài 1** (*tốc độ ↔ tươi ↔ RAM*) nhìn qua lăng kính lý thuyết phân tán.

> 🔑 **Insight #3:** Cache là một quyết định **PACELC = EL** (lúc bình thường, ưu tiên Latency hơn Consistency). Khi ai đó nói *"cứ thêm cache cho nhanh"*, họ đang âm thầm chọn EL cho **mọi path** — kể cả path tiền/tồn kho (nơi đáng lẽ phải chọn EC). Đó là cái sai mà Bài 1 cảnh báo.

---

## ⑤ Eventual consistency ĐƯỢC / KHÔNG được dùng ở đâu

Đây là phần nối thẳng với **"4 trường hợp không nên cache" (D2 Bài 1)**. Lật ngược lại: *cache được = path chịu được eventual consistency*.

| ✅ Eventual consistency OK | ❌ Cần strong consistency (đừng cache, hoặc đọc thẳng nguồn) |
|---|---|
| Like / view / follower count | **Số dư tiền** (rút quá số dư = thảm hoạ) |
| Feed, timeline, gợi ý | **Tồn kho lúc checkout** (bán đồ đã hết) |
| Search index, trending | **Hạn mức tín dụng** (tiêu quá hạn mức) |
| Profile công khai của *người khác* | **Quyền/role/permission** (cấp nhầm quyền) |
| Analytics, dashboard | **Ràng buộc duy nhất** (username/email trùng) |
| Số lượt comment | **Idempotency của thanh toán** (trừ tiền 2 lần) |

> 🔑 **Insight #4 — phép thử một câu:**
> **"Nếu user đọc trúng giá trị CŨ ở path này, hậu quả tệ nhất là gì?"**
> - *"Đếm like lệch vài giây"* → vô hại → **eventual OK, cache thoải mái.**
> - *"Bán món đã hết / trừ tiền sai"* → tiền & niềm tin → **cần strong, đừng cache** (hoặc tách phần đọc-được-cache khỏi phần quyết-định).

> 💡 **Mẹo tách path (từ ④ Bài 1):** Trang sản phẩm e-commerce: **hiển thị** "còn hàng" có thể eventual (cache TTL ngắn, lệch vài giây ok); nhưng **lúc bấm Đặt hàng** phải đọc/giữ chỗ tồn kho **strong** ở DB. Cùng một dữ liệu, **hai mô hình nhất quán cho hai mục đích**.

---

## ⑥ Ví dụ minh hoạ

### Ví dụ 1 — Like count (kinh điển, đúng câu của Bài 1)

User bấm like → tăng counter. Hiển thị "1.024 likes" có thể **cache + eventual**: nếu ai đó thấy 1.023 trong 2 giây rồi mới thành 1.024, **không ai chết**. Đổi lại bạn được phục vụ con số từ cache cực nhanh, không đập DB mỗi lần render.

### Ví dụ 2 — Read-your-writes: cái bẫy UX

User đổi avatar. Hệ cache avatar cũ với TTL 5 phút.
- **Không xử lý read-your-writes:** user F5 → thấy **avatar cũ** → *"Ơ sao chưa đổi? Bug à?"* → đổi lại lần nữa → vẫn cũ → mất niềm tin.
- **Xử lý đúng:** sau khi user *tự ghi*, đảm bảo lần đọc *của chính họ* thấy bản mới (invalidate ngay + route đọc về nguồn cho session đó, hoặc gắn version token).

> 🔑 Eventual consistency với **người khác** thì OK, nhưng với **chính tác giả của thay đổi** thì gần như luôn cần **read-your-writes**. Đây là sai sót phổ biến nhất khi áp eventual.

### Ví dụ 3 — DNS: hệ eventual-consistent kinh điển

Đổi bản ghi DNS (IP của domain). TTL ví dụ 3600s. Trong tối đa ~1 giờ, các resolver khắp thế giới vẫn trả **IP cũ** cho tới khi cache DNS hết hạn → rồi **hội tụ**. Đây là **bounded staleness** ở quy mô Internet, và là lý do người ta **hạ TTL DNS trước khi migrate**.

### Ví dụ 4 — Race kinh điển của cache-aside (PHẢI biết)

Pattern cache-aside (Bài 1 C4, chính là code Bài 1) có một **race tinh vi** làm cache **kẹt stale vĩnh viễn** tới khi TTL hết:

```
Luồng A (đọc):                         Luồng B (ghi):
1. GET cache(key) → MISS
2. đọc DB → nhận v1 (cũ)
                                        3. UPDATE DB → v2 (mới)
                                        4. DELETE cache(key)   ← invalidate
5. SET cache(key, v1)  ← GHI ĐÈ ĐỒ CŨ!
   → cache giờ = v1 (stale) cho tới khi TTL chết
```

> ⚠️ **Đây là lý do "invalidate on write" KHÔNG tự động cho bạn strong consistency.** Có một khe hở race. Cách sửa ở mục ⑧ (leases, CAS/versioning, delayed double-delete).

### Ví dụ 5 — Code: read-your-writes bằng version token

```js
// Mỗi entity có 'version' tăng dần ở DB. Cache lưu kèm version.
// Sau khi user GHI, client giữ 'minVersion' họ vừa tạo (vd qua cookie/header).
export async function getProfile(id, minVersion, loadFromDb) {
  const key = `profile:${id}`;
  const cached = await redis.get(key);

  if (cached !== null) {
    const val = JSON.parse(cached);
    if (val.version >= minVersion) return val;   // đủ mới → trả cache
    // cache cũ hơn ghi của chính user → BỎ QUA cache, đọc nguồn
  }
  const fresh = await loadFromDb(id);             // strong cho người vừa ghi
  if (fresh) await redis.set(key, JSON.stringify(fresh), "EX", 300);
  return fresh;
}
```

> Ý tưởng: **eventual cho số đông**, nhưng **read-your-writes cho người vừa ghi** nhờ so sánh `version`. Rẻ và hiệu quả.

---

## ⑦ Case study thực tế

> ⚠️ Tóm tắt theo tài liệu/bài báo công khai và pattern phổ biến. Verify số liệu cụ thể khi cần trích dẫn.

### CS1 — Amazon Dynamo (2007): eventual consistency có chủ đích cho giỏ hàng

Dynamo (bài báo nền tảng của NoSQL) chọn **AP**: luôn ghi được (high availability) kể cả khi partition, chấp nhận eventual. Khi có **conflict** (giỏ hàng được sửa ở hai bản khác nhau), Dynamo dùng **vector clock** để *phát hiện* xung đột và **để ứng dụng tự gộp** — ví dụ **hợp nhất hai giỏ hàng** (thà thừa món còn hơn mất món user đã thêm).

> 🎯 **Bài học:** Eventual consistency không có nghĩa "kệ nó sai". Phải có **chiến lược giải quyết xung đột** (conflict resolution) — merge, last-write-wins, hoặc CRDT. Chọn chiến lược theo *ngữ nghĩa nghiệp vụ* (giỏ hàng → merge, không LWW).

### CS2 — Facebook Memcached: leases để chặn stale-set & thundering herd (NSDI 2013)

Facebook gặp đúng **race ở Ví dụ 4** ở quy mô khổng lồ. Giải pháp: **lease**. Khi một client miss và định fill cache, server cấp cho nó một **token (lease)**. Mỗi lần dữ liệu bị invalidate, server **vô hiệu lease cũ**. Khi client quay lại SET với một lease đã hết hiệu lực → server **từ chối** → **không ghi đè đồ cũ**. Lease cũng giúp **gộp** nhiều miss cùng key (chống stampede).

> 🎯 **Bài học:** Để cache-aside *thật sự* an toàn ở quy mô lớn, cần cơ chế **"chỉ chấp nhận fill nếu vẫn hợp lệ"** — leases (Facebook), hoặc CAS/versioning (mục ⑧).

### CS3 — Cassandra / DynamoDB: nhất quán *điều chỉnh được* (tunable consistency)

Các hệ này cho bạn vặn núm **quorum**: với N bản sao, chọn **R** (số bản phải đọc) và **W** (số bản phải ghi).
- **R + W > N** → đọc *chắc chắn* thấy ghi mới nhất (strong-ish).
- **R + W ≤ N** → nhanh hơn nhưng **eventual**.

DynamoDB cho chọn *"eventually consistent read"* (rẻ, nhanh) hay *"strongly consistent read"* (đắt gấp ~2, chậm hơn) **ngay trên từng truy vấn**.

> 🎯 **Bài học:** Nhất quán **không phải bật/tắt nhị phân** — nó là **núm vặn theo từng request**. Đọc "đếm view" thì eventual; đọc "số dư trước khi chuyển tiền" thì strong. Cùng một store.

### CS4 — DNS & CDN purge: bounded staleness ở quy mô Internet

DNS (CS Ví dụ 3) và **CDN purge** đều là eventual: sau khi purge, các edge *dần dần* nhận bản mới. Cloudflare/Fastly cung cấp **"instant purge"** cố rút cửa sổ hội tụ xuống mili-giây, nhưng về bản chất vẫn là *lan truyền* → vẫn có cửa sổ.

> 🎯 **Bài học:** Cửa sổ hội tụ **không bao giờ = 0** trong hệ phân tán có bản sao. Bạn chỉ *rút ngắn* nó (TTL thấp, purge nhanh, pub/sub) chứ không *xoá* được — nên đừng đặt logic-cần-đúng-tuyệt-đối lên trên nó.

---

## ⑧ Giải pháp staff engineer — *bound* độ stale & *tránh race*

Sắp theo độ "trưởng thành". Thực tế thường kết hợp nhiều cái.

### G1. ✅ TTL = bounded staleness (núm đơn giản nhất) — Bài 1

TTL đặt **trần độ cũ**: "dữ liệu không bao giờ cũ quá X giây". Rẻ, không cần invalidation. Nhược điểm: trong khoảng X đó **vẫn stale**, và X càng nhỏ thì hit ratio càng giảm + nguồn càng bị đập. **TTL là đánh đổi freshness ↔ tải nguồn.**

### G2. ✅ Invalidate-on-write + thứ tự đúng

Mỗi lần ghi nguồn → xoá key cache (Bài 1: *"mỗi lần ghi DB phải invalidate key tương ứng"*). Nhưng **thứ tự quan trọng**:

| Thứ tự | Vấn đề |
|---|---|
| **Invalidate → rồi Write DB** | Khe hở: giữa hai bước, một đọc có thể nạp lại *đồ cũ* từ DB |
| **Write DB → rồi Invalidate** ✅ | An toàn hơn (thường khuyến nghị), nhưng vẫn dính race ở Ví dụ 4 nếu có đọc-miss xen vào |

→ Vì còn race, cần thêm G3/G4.

### G3. ➕ Versioning / CAS (compare-and-set) — sửa race Ví dụ 4

Lưu kèm **version/generation**. Khi SET cache, chỉ ghi nếu version **mới hơn** cái đang có (Bài 1 C4 đã nhắc **CAS**). Một fill mang version cũ (v1) sẽ **không ghi đè** được v2 → hết kẹt stale.

### G4. ➕ Leases (Facebook, CS2)

Cấp token khi miss; invalidate làm token hết hiệu lực; SET với token chết bị từ chối. Vừa chống **stale-set race** vừa chống **stampede**. Mạnh nhưng phức tạp — dùng khi quy mô lớn.

### G5. ➕ Delayed double-delete (mẹo thực dụng)

Khi ghi: **xoá cache → ghi DB → đợi một chút (vài trăm ms) → xoá cache LẦN NỮA.** Lần xoá thứ hai dọn sạch bản stale lỡ bị nạp trong khe race. Đơn giản, không cần đổi protocol, dù không hoàn hảo tuyệt đối.

### G6. ➕ Write-through để thu hẹp cửa sổ (Bài 1 C4)

Ghi **đồng thời** vào cache và DB → cache luôn tươi ngay sau ghi (cửa sổ ≈ 0 cho path đó). Đổi lại **ghi chậm hơn** và phức tạp hơn. Hợp với dữ liệu *đọc-ngay-sau-ghi*.

### G7. ➕ Read-your-writes / session consistency

Cho **người vừa ghi** thấy bản mới: route đọc của họ về nguồn trong một khoảng, hoặc dùng **version token** (Ví dụ 5), hoặc **sticky session**. Là "nâng cấp UX" rẻ nhất khi áp eventual.

### G8. ➕ Pub/sub invalidation cho L1 (Bài 1 (d) & B2 cache coherence)

Multi-tier: khi dữ liệu đổi → **phát pub/sub** cho mọi node tự xoá L1 → rút cửa sổ bất nhất giữa các node. (Đã có trong Bài 1.)

### G9. ➕ CDC-based invalidation (Change Data Capture) — chuẩn hiện đại

Thay vì tin app nhớ invalidate ở *mọi* chỗ ghi (dễ sót), **bắt thay đổi từ binlog/WAL của DB** (Debezium/Kafka) và phát sự kiện invalidate. Ưu điểm: **không bao giờ quên invalidate** (mọi ghi đều qua binlog), tách bạch logic. Đây là cách các hệ lớn giữ cache đồng bộ đáng tin cậy.

---

## ⑨ Giải pháp NÂNG CAO (staff / principal)

### A1. Tunable consistency theo từng request (CS3)

Không cứng "cache hay không" — cho **caller chọn**: đọc nhanh-eventual (mặc định) hoặc đọc strong (đi thẳng nguồn / quorum). Lộ qua API: `getProduct(id, { consistency: "strong" })`. Path checkout gọi strong; path hiển thị gọi eventual.

### A2. Conflict resolution đúng ngữ nghĩa (CS1)

Khi hai bản đổi khác nhau, đừng mặc định **LWW (last-write-wins)** mù quáng (LWW *mất ghi* và phụ thuộc đồng hồ). Chọn theo nghiệp vụ:
- **Counter** (like, view) → **CRDT counter** (cộng dồn, giao hoán, tự hội tụ, không mất đếm).
- **Set/giỏ hàng** → **merge** (union) hoặc CRDT set.
- **Field độc lập** → LWW per-field với **logical clock** (không dùng wall-clock thuần).

### A3. CRDT (Conflict-free Replicated Data Types) — hội tụ KHÔNG cần khoá

Cấu trúc dữ liệu *tự* hội tụ bất kể thứ tự đến (G-Counter, PN-Counter, OR-Set…). Cực hợp cho **multi-region eventual** (counter, presence, collaborative editing). Đổi lại: tốn metadata, không hợp mọi kiểu dữ liệu.

### A4. Bounded staleness như một SLA có thể đo

Đừng chỉ nói "eventual". **Cam kết & đo**: "p99 độ trễ hội tụ ≤ 2s", "cache cũ tối đa ≤ TTL". Theo dõi **replication lag / invalidation lag** như một metric. Khi lag vượt ngưỡng → cảnh báo (nó báo hiệu cache đang phục vụ đồ quá cũ).

### A5. Tách "đọc" (eventual) khỏi "quyết định" (strong) — CQRS-style

Mô hình mạnh nhất cho e-commerce/fintech: **đường đọc** phục vụ từ cache/replica eventual (nhanh, chịu tải); **đường quyết định** (đặt hàng, trừ tiền) đi qua **strong path** với khoá/giao dịch/idempotency. Hai đường, hai mô hình nhất quán — đúng tinh thần ④ Bài 1.

### A6. Idempotency để eventual không gây tác hại kép (Bài 1 B2)

Trong hệ eventual + retry, một thao tác có thể **chạy lại**. Mọi ghi/invalidate phải **idempotent** (làm lại kết quả không đổi) để retry an toàn — nếu không, eventual + retry = trừ tiền hai lần, gửi mail hai lần.

### A7. Monotonic reads để tránh "nhảy lùi thời gian"

Trong multi-replica, hai lần đọc liên tiếp có thể trúng hai replica lệch nhau → user thấy giá trị mới rồi **thấy lại giá trị cũ** (rất khó hiểu). Đảm bảo **monotonic reads** (gắn user vào một replica / mang theo version sàn) để "đã thấy mới thì không thấy cũ nữa".

---

## ⑩ Trường hợp MỞ RỘNG (eventual consistency ngoài cache app)

| Bối cảnh | Bản sao / cửa sổ bất nhất | Ghi chú |
|---|---|---|
| **DNS** | Resolver cache theo TTL | Bounded staleness toàn cầu; hạ TTL trước migrate |
| **CDN** | Edge cache, purge lan truyền | "Instant purge" rút cửa sổ nhưng không = 0 |
| **DB replica đọc (read replica)** | Replication lag | Đọc từ replica = eventual; cần read-your-writes nếu đọc sau ghi |
| **Multi-region DB** | Cross-region replication | Active-active cần conflict resolution (LWW/CRDT) |
| **Cache coherence CPU (MESI)** ✅ B2 | Cache L1/L2 mỗi core | **Ngược lại — strong!** Phần cứng giữ coherence chặt (dùng đối chiếu để thấy "eventual là lựa chọn của phần mềm khi coherence quá đắt") |
| **Git / hệ phân tán** | Mỗi clone một bản | Eventual: hội tụ khi push/pull/merge |
| **Microservices event-driven** | Mỗi service một bản chiếu (read model) | **Saga** thay transaction; eventual giữa các service |
| **Elasticsearch / search index** | Index trễ so với DB nguồn | "Near real-time"; refresh interval = cửa sổ bất nhất |
| **Collaborative editing (Google Docs)** | Bản local mỗi client | **CRDT/OT** để hội tụ không xung đột |

> 🔑 **Insight #5:** Đối chiếu hay nhất là **cache coherence của CPU (MESI)** — phần cứng chọn **strong** vì để hai core đọc lệch nhau là thảm hoạ, và nó *đủ tiền* (khoảng cách ngắn) để giữ chặt. Hệ phân tán phần mềm thì khoảng cách xa, partition có thật, giữ strong quá đắt/chậm → **chọn eventual**. ⇒ Eventual không phải "lười", nó là **phản ứng hợp lý với chi phí của khoảng cách**.

---

## ⑪ Kiến thức MỞ RỘNG (kết nối)

- ✅ **Strong consistency (B1):** đầu kia của phổ. Cặp strong↔eventual là trục chính của mọi quyết định cache.
- ➕ **CAP / PACELC:** lý do eventual tồn tại. PACELC (EL) mô tả đúng cache lúc bình thường: ưu tiên latency hơn consistency.
- ➕ **Cache coherence (B2):** giữ *nhiều bản L1* đồng nhất — chính là *cố ép eventual về gần strong* bằng pub/sub.
- ➕ **Idempotency (B2):** điều kiện để eventual + retry không gây hại kép.
- ➕ **CRDT / vector clock / LWW:** bộ công cụ *giải quyết xung đột* khi các bản hội tụ.
- ➕ **Quorum (R/W/N):** núm vặn nhất quán định lượng (CS3).
- ➕ **Write patterns (C4 Bài 1):** write-through (cửa sổ ≈0, ghi chậm) vs cache-aside (lazy, có race) vs write-back (nhanh, rủi ro mất data) — mỗi cái một profile nhất quán.
- ➕ **Linearizability vs Serializability:** linearizability = nhất quán *theo thời gian thực* cho một object; serializability = nhất quán *giao dịch*. Strong consistency hay nói tới linearizability.

---

## ⑫ Anti-patterns — AI & người mới hay sai

| ❌ Sai lầm | Vì sao nguy hiểm | ✅ Đúng |
|---|---|---|
| Cache path **tiền / tồn kho / quyền** | Đọc đồ cũ = quyết định sai (bán đồ hết, cấp nhầm quyền) | **Strong** cho path quyết định; tách phần public ra cache |
| Tưởng **invalidate-on-write = strong** | Vẫn dính **stale-set race** (Ví dụ 4) | Thêm **versioning/CAS / leases / double-delete** |
| Quên **read-your-writes** | User đổi data xong thấy đồ cũ → tưởng bug | Version token / route-to-source cho người vừa ghi |
| Mặc định **LWW** cho mọi xung đột | LWW **mất ghi**, phụ thuộc đồng hồ | Conflict resolution theo nghiệp vụ (merge/CRDT) |
| Tin invalidate rải rác trong app là đủ | **Dễ sót** một đường ghi → cache stale lén lút | **CDC** từ binlog → không bao giờ quên |
| Hai lần đọc nhảy mới→cũ | Vi phạm **monotonic reads**, user hoang mang | Gắn replica / version sàn |
| Gọi mọi thứ là "eventual" rồi thôi | Bỏ qua các nấc giữa (read-your-writes, bounded) | Chọn **nấc yếu nhất mà UX chịu được** |
| Không đo **độ trễ hội tụ** | Cache có thể phục vụ đồ rất cũ mà không ai biết | Theo dõi **replication/invalidation lag** như SLA |

> 🔎 **AI hay sai (nối Bài 1 & cuối Bài 1):** AI mặc định *"thêm cache = nhanh hơn"* và bê Redis vào **mọi path**, ngầm chọn **eventual cho cả tiền/tồn kho**. Luôn ép hỏi: *"Path này chịu được đọc đồ cũ tới mức nào? Hậu quả tệ nhất khi stale là gì?"* — nếu là tiền/quyền → **không eventual**.

---

## ⑬ Khung quyết định / Playbook

### D-EC-1 — Chọn mô hình nhất quán cho một path
```
1. Hậu quả tệ nhất khi đọc đồ CŨ là gì?
   → vô hại (đếm/feed)         → EVENTUAL (cache thoải mái + TTL)
   → user tự thấy sai          → + READ-YOUR-WRITES
   → tiền/tồn kho/quyền/unique → STRONG (đừng cache path quyết định)
2. Người vừa ghi có cần thấy ngay không? → có → read-your-writes.
3. Hai lần đọc nhảy mới→cũ có chấp nhận? → không → monotonic reads.
4. Có nhiều bản ghi đồng thời (multi-region)? → cần conflict resolution (merge/CRDT), tránh LWW mù.
```

### D-EC-2 — Làm cache-aside an toàn (chống stale-set race)
```
□ Ghi: WRITE DB trước → INVALIDATE cache sau (không ngược lại).
□ SET cache chỉ khi version MỚI HƠN (CAS/versioning).  ← chặn Ví dụ 4
□ Quy mô lớn: dùng LEASES (Facebook) thay vì tự xử.
□ Thực dụng: DELAYED DOUBLE-DELETE để dọn bản lỡ stale.
□ Invalidate đáng tin: dùng CDC (binlog) thay vì rải rác trong app.
```

### D-EC-3 — Bound & đo độ stale
```
□ Đặt TTL = trần độ cũ chấp nhận được (bounded staleness).
□ Đo replication/invalidation lag → cảnh báo khi vượt ngưỡng.
□ Đặt SLA hội tụ (vd p99 ≤ 2s) thay vì nói chung chung "eventual".
□ pub/sub invalidate cho L1 multi-tier để rút cửa sổ giữa các node.
```

---

## ⑭ Mental model (câu thần chú gốc)

> ⏳ **Cache = chọn eventual consistency.** "Stale" của Bài 1 chính là **cửa sổ bất nhất** của nó.
> Câu hỏi không phải *"có bất nhất không"* (có), mà *"cửa sổ rộng bao lâu, và path này chịu được tới đâu"*.
>
> **Hậu quả khi đọc đồ cũ vô hại → eventual (cache thoải mái).**
> **Hậu quả là tiền / tồn kho / quyền → strong (đừng cache path quyết định).**
> **Người vừa ghi → luôn cho read-your-writes.**
>
> "Invalidate on write" **KHÔNG** tự cho bạn strong — còn **stale-set race**; vá bằng **versioning/CAS, leases, hoặc CDC**. Và đừng chỉ nói "eventual" — hãy **bound nó (TTL) và đo nó (lag)**.

---

## ⑮ Câu hỏi phỏng vấn & thách đố

- **(dễ)** Eventual consistency là gì? → Nếu ngừng cập nhật, sau một khoảng thời gian mọi bản sao hội tụ về giá trị mới nhất; có cửa sổ bất nhất ở giữa.
- **(dễ)** Vì sao cache liên quan tới eventual consistency? → Cache là một bản sao → đọc cache có thể ra đồ cũ → đúng định nghĩa eventual; "stale" = cửa sổ bất nhất.
- **(TB)** CAP vs PACELC khác gì? → CAP: lúc partition chọn A hay C. PACELC: *cả lúc bình thường* vẫn chọn Latency hay Consistency — bắt đúng đánh đổi của cache.
- **(TB)** Cho 3 path nên dùng eventual và 3 path cần strong. → eventual: like/feed/analytics; strong: số dư/tồn kho-checkout/permission.
- **(TB)** Read-your-writes là gì, vì sao quan trọng với cache? → Người vừa ghi luôn thấy bản mới; nếu không, user đổi data xong thấy đồ cũ → tưởng bug.
- **(khó)** Mô tả race của cache-aside và cách sửa. → đọc-miss nạp v1 cũ *sau khi* ghi đã invalidate v2 → cache kẹt v1; sửa bằng CAS/versioning, leases, hoặc delayed double-delete.
- **(khó)** Xung đột khi hai bản cùng đổi — LWW có ổn không? → LWW *mất ghi* + phụ thuộc đồng hồ; chọn theo nghiệp vụ: counter→CRDT, giỏ hàng→merge.
- **(khó)** "Tunable consistency" nghĩa là gì? → vặn R/W/N (quorum) hoặc chọn strong/eventual *từng request*; cùng store phục vụ cả hai nhu cầu.

> 🧩 **Thách đố:** *"User báo: đổi tên hiển thị xong, chỗ này thấy tên mới, chỗ kia vẫn tên cũ, refresh lại thì lúc mới lúc cũ. Chẩn đoán?"*
> → Vi phạm **read-your-writes** (chưa invalidate/route cho chính họ) **+ thiếu monotonic reads** (đọc trúng các replica/cache-node lệch nhau). Sửa: invalidate ngay khi ghi (lý tưởng qua CDC) + version token để người vừa ghi luôn thấy ≥ version mình tạo + gắn session vào một nguồn để không nhảy mới↔cũ.

---

## ⑯ Bài tập + tiêu chí tự chấm

**BT1.** Cho 6 loại dữ liệu (like count, số dư ví, tồn kho trang sản phẩm, tồn kho lúc checkout, avatar của chính user, trending hashtag). Gán mô hình nhất quán phù hợp + lý do.
> ✅ **Đạt khi:** like/trending→eventual; số dư & checkout→strong (đừng cache quyết định); tồn kho hiển thị→eventual TTL ngắn; avatar-của-chính-user→read-your-writes.

**BT2.** Vẽ timeline race của cache-aside (Ví dụ 4) và chỉ ra **chính xác** bước nào CAS/versioning chặn được, vì sao.
> ✅ **Đạt khi:** chỉ đúng bước SET v1 bị từ chối vì version < v2 hiện có.

**BT3.** Thiết kế đường đọc + đường đặt hàng cho trang sản phẩm sao cho hiển thị nhanh (eventual) nhưng không bao giờ bán quá tồn kho.
> ✅ **Đạt khi:** đọc hiển thị từ cache eventual (TTL ngắn); đặt hàng đi strong path (transaction/giữ chỗ ở DB) với idempotency; tách rõ hai đường.

**BT4.** Giải thích vì sao "invalidate khi ghi" không đủ để có strong consistency, và liệt kê 3 cách vá.
> ✅ **Đạt khi:** mô tả stale-set race + nêu CAS/versioning, leases, delayed double-delete (hoặc CDC).

---

## ⑰ Glossary (đóng vào hoàn cảnh)

| Thuật ngữ | Nghĩa nhanh | 🎬 Hoàn cảnh sử dụng |
|---|---|---|
| **Eventual consistency** | Cuối cùng các bản hội tụ về giá trị mới | *"Like count eventual là đủ, lệch vài giây ok."* |
| **Strong consistency** | Đọc luôn ra ghi mới nhất | *"Số dư ví cần strong, đừng cache."* |
| **Staleness window** | Khoảng thời gian bản đọc có thể cũ | *"Hạ TTL để thu hẹp staleness window."* |
| **Bounded staleness** | Cũ tối đa X giây/X version | *"TTL 5s = bounded staleness 5 giây."* |
| **Read-your-writes** | Người vừa ghi luôn thấy bản mới | *"Đổi avatar xong phải thấy ngay → read-your-writes."* |
| **Monotonic reads** | Đã thấy mới thì không thấy lại cũ | *"Gắn session vào 1 nguồn để khỏi nhảy mới↔cũ."* |
| **CAP / PACELC** | Khung đánh đổi C-A / L-C | *"Cache là PACELC=EL: ưu tiên latency."* |
| **Quorum (R/W/N)** | Núm vặn nhất quán định lượng | *"R+W>N → đọc thấy ghi mới nhất."* |
| **Stale-set race** | Đọc-miss ghi đè cache bằng đồ cũ | *"Cache-aside dính stale-set race, vá bằng CAS."* |
| **Lease** | Token chỉ cho fill hợp lệ mới được ghi | *"Facebook dùng lease chặn stale-set + stampede."* |
| **Delayed double-delete** | Xoá cache 2 lần (sau ghi + sau trễ) | *"Double-delete dọn bản lỡ stale trong khe race."* |
| **CDC** | Bắt thay đổi từ binlog để invalidate | *"CDC để không bao giờ quên invalidate."* |
| **LWW** | Last-write-wins khi xung đột | *"LWW mất ghi — đừng dùng cho giỏ hàng."* |
| **CRDT** | Cấu trúc tự hội tụ không cần khoá | *"Counter đa vùng → CRDT, không mất đếm."* |
| **Replication / invalidation lag** | Độ trễ lan truyền thay đổi | *"Cảnh báo khi lag > 2s → cache đang quá cũ."* |

---

## ⑱ Đọc thêm

- **DeCandia et al., "Dynamo: Amazon's Highly Available Key-value Store" (SOSP 2007)** — eventual + vector clock + merge.
- **Brewer, "CAP Twelve Years Later"** + **Abadi, "Consistency Tradeoffs… PACELC"** — khung lý thuyết.
- **Scaling Memcached at Facebook (NSDI 2013)** — **leases**, read-after-write ở quy mô lớn.
- **Werner Vogels, "Eventually Consistent" (ACM Queue)** — bài viết nền tảng dễ đọc.
- **Shapiro et al., "Conflict-free Replicated Data Types"** — CRDT.
- **Martin Kleppmann, *Designing Data-Intensive Applications*** — chương Consistency & Replication (bản đồ toàn cảnh).
- **AWS DynamoDB docs** — *eventually vs strongly consistent reads*; **Azure Cosmos DB** — 5 mức nhất quán (gồm *bounded staleness*).

---

> 🔗 **Nối lại cả cụm:** Bài 1 dựng khung *tốc độ ↔ tươi ↔ RAM*; **eventual consistency là tên hình thức của trục "tươi"** đó. Nó nối với **Cold Start** (lúc cache lạnh, cửa sổ bất nhất = ∞ vì chưa có gì để hội tụ) và **Working Set** (chỉ phần nóng mới đáng chịu cửa sổ bất nhất để đổi lấy tốc độ). Câu chốt của cả ba: ***cache cho bạn tốc độ bằng cách mượn trước sự nhất quán — hãy biết bạn đang mượn ở path nào, bao lâu, và trả lại bằng cách nào.***
