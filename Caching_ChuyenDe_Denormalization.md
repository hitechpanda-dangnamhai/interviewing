# 🗂️ CHUYÊN ĐỀ — DENORMALIZATION (Phi chuẩn hoá) trong Database & Caching
> Đào sâu concept mở rộng cuối của **Bài 1 — Vì sao cache · Các tầng cache · Khi nào KHÔNG cache**
> Mục tiêu: hiểu denormalization là *"cache nằm trong database"*, biết *chọn normalize/denormalize*, và *quản lý drift* ở mức **staff engineer**.
> (✅ = đã có trong Bài 1 · ➕ = mở rộng thêm)

---

## ① Mục tiêu & vị trí trong mạch

Trong Bài 1, **Denormalization** xuất hiện ở **bảng concept mở rộng (B2)**:

> ➕ **Denormalization** — *Nhân bản dữ liệu để đọc nhanh, đánh đổi ghi phức tạp* — *"Denormalize cột thay vì cache khi đọc cực nhiều."*

Và nó gắn trực tiếp với hai chỗ khác của Bài 1:
- **Mục (g): Cache vs Precompute / Materialized View** — *"cache = lazy, MV = eager"*. Denormalization thuộc **phe eager** (tính sẵn).
- **D4 + cross-ref:** *"MV vs cột denormalized vs cache"* — ba anh em "có sẵn dữ liệu cho nhanh".

Chuyên đề **Working Set** (A7) cũng đã trỏ tới: *"denormalize/precompute để 1 entry phục vụ nhiều request → giảm số key cần nóng"*.

Chuyên đề này đưa nó ra sân khấu chính — và là **mảnh ghép khép cụm**, vì nó buộc lại cả 8 chuyên đề trước. Học xong bạn sẽ:

- Hiểu denormalization là **"cache trong database"**: eager precompute nướng vào schema, đổi đọc-nhanh lấy ghi-phức-tạp.
- Phân biệt rạch ròi **Cache vs Materialized View vs Denormalized column** (bảng so sánh Bài 1 D4).
- Nắm các **dạng** denormalization (cột nhân bản, cột dẫn xuất, aggregate, embedding, fan-out-on-write).
- Biết **quản lý drift** (đồng bộ vs bất đồng bộ, reconciliation) — cùng kỷ luật như cache stale.

> 💡 **Một câu để gắn vào mạch Bài 1:** Bài 1 dạy cache là *bản sao gần & nhanh, trả giá bằng rủi ro stale*. **Denormalization cũng y hệt — chỉ khác bản sao nằm NGAY TRONG database, được persist, và có thể cập nhật trong CÙNG transaction với nguồn.** Nó là "cache mà DBA kiểm soát", với cùng bài toán: giữ source of truth, quản lý lệch (drift), reconcile.

---

## ② Trực giác — Denormalization là gì? (và Normalization để đối chiếu)

### Normalization (chuẩn hoá) — nền để hiểu denormalization

**Normalization** (Codd) = tổ chức dữ liệu sao cho **mỗi sự thật được lưu ĐÚNG MỘT CHỖ**, loại bỏ trùng lặp. Các dạng chuẩn:

| Dạng chuẩn | Loại bỏ |
|---|---|
| **1NF** | Giá trị không nguyên tử / nhóm lặp |
| **2NF** | Phụ thuộc một phần vào khoá tổ hợp |
| **3NF** | Phụ thuộc bắc cầu (non-key → non-key) |
| **BCNF** | 3NF chặt hơn |

**Lợi của normalization:** không có **anomaly** (bất thường khi cập nhật/chèn/xoá), một source of truth, toàn vẹn dữ liệu. **Tối ưu cho GHI & đúng đắn.**

**Giá của normalization:** đọc phải **JOIN** nhiều bảng để ráp lại dữ liệu → **chậm ở quy mô lớn**.

### Denormalization — đi ngược lại một cách CÓ CHỦ ĐÍCH

**Denormalization = cố ý đưa TRÙNG LẶP trở lại** (nhân bản cột, lưu giá trị tính sẵn, nhúng dữ liệu liên quan) để **đọc nhanh** — đổi lại **ghi phức tạp** + **rủi ro lệch**. **Tối ưu cho ĐỌC.**

### Ẩn dụ danh bạ vs sổ tay cá nhân

- **Normalized:** chỉ có **một** danh bạ trung tâm. Cần số ai → tra danh bạ (JOIN). Số đổi → sửa **một** chỗ, ai cũng thấy đúng.
- **Denormalized:** mỗi người **chép sẵn** vài số hay gọi vào sổ tay riêng. Gọi nhanh (khỏi tra). Nhưng số đổi → phải **đi sửa mọi cuốn sổ** (ghi phức tạp), lỡ sót một cuốn → **lệch (drift)**.

> 🔑 **Insight #1:** Đây **đúng** là bài toán cache của Bài 1 (tờ giấy dán màn hình), chỉ đặt **bên trong database**. Denormalization, cache, materialized view đều là **"dữ liệu dẫn xuất, nhân bản, để đọc nhanh"** — và **đều chung một vấn đề: bản sao có thể lệch nguồn**.

---

## ③ Vì sao denormalize? — bài toán đọc & JOIN đắt

Lý do gốc: ở quy mô lớn, **đọc theo cách normalized trở nên đắt**:

1. **JOIN đắt** — nhất là many-to-many, hoặc dữ liệu **sharded/phân tán** (JOIN xuyên shard cực tốn). NoSQL thậm chí **không có JOIN**.
2. **Aggregate đắt** — tính `COUNT(*)`, `SUM` trên mỗi lần đọc (đếm comment, tổng đơn) → quét nhiều dòng.
3. **Workload đọc-nặng** — trả giá **một lần lúc ghi**, đọc **rẻ nhiều lần**. (Đúng tinh thần eager precompute.)

> 🔑 **Insight #2 (nối Bài 1 g — eager vs lazy):** Cache là **lazy** (tính khi có người hỏi, có thể miss). Denormalization là **eager** (tính sẵn lúc ghi, luôn có sẵn). Chọn denormalize khi: **đọc cực nhiều + cần luôn-sẵn + (có thể) cần đúng trong cùng transaction**. Chọn cache khi: **đọc thưa hơn + chấp nhận lần đầu miss + không muốn động schema**.

---

## ④ Cache vs Materialized View vs Denormalized column (Bài 1 D4, mở rộng)

Ba anh em "có sẵn dữ liệu". Phân biệt rạch ròi là dấu hiệu staff engineer:

| Tiêu chí | **Cache** (Redis) | **Materialized View** | **Denormalized column** |
|---|---|---|---|
| Nằm ở đâu | Store riêng (Redis) | Bảng tính sẵn trong DB | **Ngay trong bảng gốc** |
| Lazy/Eager | **Lazy** (nạp khi miss) | **Eager** (refresh định kỳ) | **Eager** (cập nhật lúc ghi) |
| Độ tươi | TTL/invalidate | Theo chu kỳ refresh | **Có thể strong** (cùng transaction với ghi) |
| Bền vững | Bay khi restart (nếu không persist) | Bền | **Bền (là một phần schema)** |
| Tính nhất quán | Eventual (cửa sổ stale) | Eventual (lag refresh) | **Strong nếu update đồng bộ**, eventual nếu async |
| Hợp nhất khi | Đọc thưa, chịu miss, không động schema | Aggregate đọc nhiều, refresh được | Đọc cực nhiều **một field**, cần trong transaction |
| Chi phí ghi | Thấp (invalidate) | Trung bình (refresh) | **Cao** (mỗi ghi cập nhật bản nhân) |

> 🔑 **Insight #3 (điểm tinh tế nhất):** Denormalization có thể **NHẤT QUÁN HƠN cache** — vì bản nhân bản được cập nhật trong **CÙNG transaction** với nguồn (không có cửa sổ eventual như cache TTL). Đổi lại **ghi đắt hơn + schema cứng hơn**. Khi bạn cần *"vừa đọc nhanh vừa đúng tuyệt đối"* (path mà Bài 1 bảo *"cần strong consistency"*), **denormalize đồng bộ** thường tốt hơn cache. Cache cho bạn tốc độ + eventual; denormalize đồng bộ cho tốc độ + strong (với giá ghi).

---

## ⑤ Các DẠNG denormalization (đừng nghĩ chỉ có "copy cột")

| Dạng | Mô tả | Ví dụ |
|---|---|---|
| **1. Nhân bản cột** | Chép cột từ bảng khác để khỏi JOIN | Lưu `author_name` trong `posts` cạnh `author_id` |
| **2. Cột dẫn xuất (derived)** | Lưu kết quả tính toán | `total_price` trong `orders`, `full_name` |
| **3. Aggregate tính sẵn** | Đếm/tổng cập nhật lúc ghi | `comment_count`, `like_count` trên `post` |
| **4. Embedding/nesting** | Nhúng dữ liệu liên quan vào một bản ghi | MongoDB nhúng comments vào document post |
| **5. Materialized view** | Bảng kết quả truy vấn tính sẵn | Báo cáo doanh thu theo ngày |
| **6. Star schema** | Fact + dimension denormalized cho OLAP | Data warehouse (Kimball) |
| **7. Fan-out-on-write** | Nhân bản dữ liệu vào nhiều đích lúc ghi | Timeline mạng xã hội (mục ⑦ CS1) |
| **8. Snapshot dữ liệu** | Chụp giá trị tại thời điểm (đúng theo nghiệp vụ) | `price_at_purchase` trong `order_items` |

> 🔑 **Insight #4 — phân biệt "tối ưu" với "đúng theo thiết kế":** Dạng **8 (snapshot)** không phải tối ưu mà là **yêu cầu nghiệp vụ**: giá lúc mua **phải** đông cứng, dù giá sản phẩm đổi sau này. Đây là denormalization mà bạn **luôn nên làm** — nó *không* drift vì nó **cố ý là bản chụp**, không phải bản sao của giá hiện tại. Đừng nhầm hai loại: snapshot (đúng theo thiết kế) vs duplicate-for-speed (cần quản lý drift).

---

## ⑥ Ví dụ minh hoạ

### Ví dụ 1 — Đếm comment: aggregate denormalized vs đếm mỗi lần

```sql
-- ❌ Normalized: đếm mỗi lần đọc post (quét bảng comments)
SELECT p.*, (SELECT COUNT(*) FROM comments c WHERE c.post_id = p.id) AS cnt
FROM posts p WHERE p.id = 42;     -- đọc nào cũng quét comments

-- ✅ Denormalized: cột comment_count cập nhật lúc ghi
SELECT * FROM posts WHERE id = 42;          -- đọc 1 dòng, không quét
-- khi thêm comment:
UPDATE posts SET comment_count = comment_count + 1 WHERE id = 42;
```

> Đọc rẻ hẳn. Nhưng `+1` này **không idempotent** (chuyên đề Idempotency!) — nếu insert comment và update count ở **hai transaction**, crash giữa chừng → lệch. Phải **bọc cùng transaction** hoặc dùng cơ chế idempotent.

### Ví dụ 2 — Cập nhật ĐỒNG BỘ (strong) vs BẤT ĐỒNG BỘ (eventual)

```sql
-- Đồng bộ: count luôn đúng, ghi chậm hơn chút (cùng transaction)
BEGIN;
  INSERT INTO comments(post_id, body) VALUES (42, '...');
  UPDATE posts SET comment_count = comment_count + 1 WHERE id = 42;
COMMIT;
```
- **Đồng bộ** (trên): `comment_count` **strong consistent**, không drift. Giá: mỗi ghi đụng `posts` (có thể là điểm contention nóng).
- **Bất đồng bộ:** insert comment xong, một **trigger/CDC/job nền** cập nhật count sau → ghi nhanh hơn nhưng **eventual** (count lệch vài giây) + cần **reconciliation**.

### Ví dụ 3 — Nhân bản cột để khỏi JOIN

```
posts(id, author_id, author_name, ...)   ← author_name nhân bản từ users
```
Hiển thị danh sách post + tên tác giả **không cần JOIN** `users`. Nhưng khi user **đổi tên** → phải đi cập nhật `author_name` ở **mọi post** của họ (ghi phức tạp, write amplification). Drift nếu sót.

### Ví dụ 4 — Giảm hot key bằng denormalize (nối Working Set A7)

Thay vì cache 5 mảnh dữ liệu rời (profile + count + badge + …) thành 5 hot key phải giữ nóng, **denormalize chúng vào một bản ghi `user_card`** → một lần đọc, một "key" → **giảm số key cần nằm trong working set**. Denormalization **thu nhỏ working set** cần cache.

---

## ⑦ Case study thực tế

> ⚠️ Tóm tắt theo tài liệu/bài báo công khai và pattern phổ biến. Verify số liệu cụ thể khi cần.

### CS1 — Twitter/Instagram timeline: fan-out-on-write (denormalize timeline) — *case kinh điển nhất*

Hai cách dựng "home timeline":
- **Fan-out-on-read (normalized):** lúc user mở app, **truy vấn tweet của tất cả người họ follow rồi trộn**. Ghi (đăng tweet) rẻ; **đọc đắt** (mỗi lần mở app là một truy vấn lớn).
- **Fan-out-on-write (denormalized):** lúc ai đó đăng tweet, **ghi sẵn tweet đó vào timeline của TỪNG follower**. Đọc cực rẻ (timeline đã dựng sẵn); **ghi đắt** — một người 100 triệu follower = 100 triệu lần ghi (**write amplification** khủng).

**Giải pháp lai (thực tế):** đa số user **fan-out-on-write**; **celebrity** (quá nhiều follower) **fan-out-on-read**, trộn lúc đọc. Cân giữa hai cực.

> 🎯 **Bài học:** Denormalization (fan-out-on-write) đổi **đọc rẻ** lấy **ghi đắt + write amplification**. Khi write amplification nổ (celebrity), phải **lai** với normalized. Đây là minh hoạ sống động nhất của đánh đổi đọc↔ghi.

### CS2 — DynamoDB single-table design: denormalize là BẮT BUỘC

NoSQL **không có JOIN**. Thiết kế DynamoDB (Rick Houlihan) bắt đầu từ **access patterns** rồi **denormalize/nhân bản** để mỗi pattern là một truy vấn key đơn. Trùng lặp dữ liệu là **chuẩn mực, không phải lỗi**.

> 🎯 **Bài học:** Ở NoSQL, denormalization **không tuỳ chọn** — nó là cách mô hình hoá. Trade-off chuyển sang **ứng dụng phải tự lo cập nhật mọi bản nhân** (vì DB không JOIN giúp).

### CS3 — Data warehouse: star schema cho analytics (Kimball)

OLTP (giao dịch) **normalized** cho ghi đúng. Nhưng **OLAP (phân tích)** dùng **star schema denormalized** (fact table + dimension table phẳng) để truy vấn tổng hợp nhanh, không JOIN sâu. Hai mô hình cho hai mục đích, đồng bộ bằng ETL/CDC.

> 🎯 **Bài học:** **Polyglot/đa mô hình:** giữ normalized làm nguồn, **denormalize ở nơi đọc-phân-tích**. Không phải "normalize hay denormalize" — mà **cả hai, ở đúng chỗ**.

### CS4 — Search index (Elasticsearch): document denormalized

Để search nhanh, dữ liệu được **flatten/denormalize thành document** trong index (đồng bộ từ DB nguồn qua CDC). Index là bản denormalized read-optimized; DB vẫn là source of truth.

> 🎯 **Bài học:** Search index = một dạng denormalized read model, giữ đồng bộ bằng CDC (nối Cache Coherence). Cùng kỷ luật: nguồn sự thật + bản dẫn xuất + cơ chế đồng bộ.

---

## ⑧ Giải pháp staff engineer — denormalize đúng cách

### Quy tắc gốc: GIỮ source of truth chuẩn hoá + bản denormalized là DẪN XUẤT

> 🔑 **Insight #5:** **Đừng để bản denormalized trở thành "sự thật".** Luôn có một **nguồn chuẩn hoá (authoritative)**; bản nhân bản/aggregate chỉ là **dẫn xuất có thể dựng lại**. Nhờ vậy khi drift, bạn **recompute từ nguồn** được. Mất nguyên tắc này → drift biến thành **hỏng dữ liệu vĩnh viễn** (không biết bản nào đúng).

### Chọn đồng bộ vs bất đồng bộ

| | **Đồng bộ (cùng transaction)** | **Bất đồng bộ (trigger/CDC/job)** |
|---|---|---|
| Nhất quán | **Strong** (không drift) | Eventual (có cửa sổ lệch) |
| Tốc độ ghi | Chậm hơn (ghi nhiều thứ atomic) | Nhanh hơn (ghi nguồn rồi đi) |
| Hợp khi | Field quan trọng, cần đúng ngay | Aggregate chịu được lệch ngắn |
| Cần thêm | — | **Reconciliation job** sửa drift |

### Bốn quy tắc vàng

1. **Đo trước (Amdahl!):** đọc có thật sự là nút thắt không? JOIN/aggregate này có thống trị p99 không? Nếu không → **đừng denormalize** (đừng tối ưu phần không-nút-thắt).
2. **Denormalize CÓ CHỌN LỌC:** chỉ những field/aggregate **đọc cực nhiều**. Denormalize tất cả → **write amplification nổ** + schema cứng.
3. **Aggregate update phải an toàn:** `+1` cùng transaction với nguồn, hoặc idempotent/CRDT (chuyên đề Idempotency) — đừng để counter drift.
4. **Luôn có reconciliation job:** denormalized data **là một cache** → sẽ drift → cần job định kỳ **recompute từ nguồn** để sửa lệch (như cache cần invalidate).

---

## ⑨ Giải pháp NÂNG CAO (staff / principal)

### A1. CQRS — denormalization ở tầng kiến trúc

**Command Query Responsibility Segregation:** tách **write model** (normalized, tối ưu ghi/đúng) khỏi **read model** (denormalized, tối ưu đọc), đồng bộ qua **event**. Read model có thể có **nhiều bản** hình dạng khác nhau cho từng pattern đọc. Đây là denormalization nâng lên cấp hệ thống.

### A2. CDC-driven denormalization (nối Cache Coherence A3)

Thay vì app nhớ cập nhật mọi bản nhân (dễ sót → drift), **bắt thay đổi từ binlog (CDC)** và **dựng read model/denormalized table** từ luồng sự kiện. Đảm bảo **không sót** đường ghi nào. Cùng kỹ thuật giữ cache coherent. Consumer phải **idempotent** (chuyên đề Idempotency).

### A3. Materialized view incremental refresh

MV refresh **toàn bộ** thì đắt. **Incremental/fast refresh** chỉ cập nhật phần đổi (dựa trên log thay đổi) → tươi hơn, rẻ hơn. Một số DB hỗ trợ `ON COMMIT` refresh (gần strong). Chọn chiến lược refresh = chọn điểm trên trục tươi↔chi phí.

### A4. Quản lý write amplification (bài học CS1)

Fan-out-on-write nhân ghi lên. Kiểm soát:
- **Lai normalized cho điểm nóng** (celebrity fan-out-on-read).
- **Batch/async fan-out** (đẩy vào queue, ghi dần — eventual).
- **Giới hạn fan-out** (chỉ denormalize cho follower active — nối Working Set: chỉ nhân cho phần "nóng").

### A5. Reconciliation & drift detection như một SLA

Denormalized data **sẽ** lệch (bug, crash, race). Cần:
- **Job recompute định kỳ** so denormalized vs nguồn, sửa lệch.
- **Drift metric**: đo % bản ghi lệch → cảnh báo khi vượt ngưỡng (như replication lag ở Cache Coherence).
- Coi denormalized = **cache bền** → áp đúng kỷ luật cache (source of truth + invalidate/reconcile).

### A6. Chọn GRANULARITY denormalize đúng

Không phải "denormalize hay không" mà "**denormalize cái gì, tới đâu**". Field đổi-thường-xuyên mà nhân bản rộng → ghi nổ; field đổi-thưa, đọc-nhiều → ứng viên vàng. Phân tích **tần suất đọc × tần suất ghi × độ rộng nhân bản** cho từng field trước khi quyết.

### A7. Polyglot persistence — denormalize ở đúng store

Giữ **OLTP normalized** (nguồn), **denormalize sang**: search index (ES), warehouse (star schema), cache (Redis), read replica hình dạng khác. Mỗi store một mục đích đọc; tất cả đồng bộ từ nguồn qua CDC/ETL. **Đừng ép một store làm mọi pattern đọc.**

---

## ⑩ Trường hợp MỞ RỘNG (denormalization ở mọi tầng)

| Bối cảnh | Denormalization thể hiện thế nào |
|---|---|
| **RDBMS (OLTP)** | Cột nhân bản/dẫn xuất, aggregate tính sẵn, MV |
| **NoSQL (DynamoDB/Mongo)** | Single-table design, embedding — denormalize **bắt buộc** |
| **Data warehouse (OLAP)** | Star/snowflake schema (Kimball) |
| **Search (Elasticsearch)** | Document denormalized, đồng bộ qua CDC |
| **Social feed** | Fan-out-on-write timeline (CS1) |
| **Caching (Bài 1)** | Cache = denormalization "lazy, ngoài DB, volatile" |
| **CQRS/event sourcing** | Read model denormalized từ event |
| **Graph DB** | Đôi khi nhân bản cạnh/đường đi hay duyệt |
| **GraphQL/API** | Response shape gộp sẵn nhiều nguồn (denormalize ở tầng API) |

> 🔑 **Insight #6:** Cache, materialized view, search index, read replica, CQRS read model, fan-out timeline — **tất cả là biến thể của cùng một ý**: *"giữ một bản dẫn xuất, nhân bản, được hình-dạng-lại để đọc nhanh; trả giá bằng ghi phức tạp + drift"*. Denormalization là **tên chung** của họ này khi nó nằm trong tầng dữ liệu bền. Học một lần, nhận ra ở mọi nơi.

---

## ⑪ Kiến thức MỞ RỘNG (kết nối cả cụm — 9 tài liệu)

- ✅ **Cache vs MV vs denormalized (B1 g, D4):** ba anh em dẫn-xuất-để-đọc-nhanh; chuyên đề này định vị rõ từng cái.
- ✅ **Eager vs lazy (B1 g):** denormalize = eager (tính sẵn lúc ghi); cache = lazy (tính khi miss).
- ➕ **Working Set (chuyên đề):** denormalize gộp nhiều mảnh → **giảm số key/working set** cần nóng (A7, Ví dụ 4).
- ➕ **Eventual Consistency (chuyên đề):** denormalized data drift = vấn đề stale; chọn **đồng bộ (strong)** hay **async (eventual)** để cập nhật.
- ➕ **Cache Coherence (chuyên đề):** giữ bản denormalized đồng bộ với nguồn = bài toán coherence; **CDC** là cơ chế (A2).
- ➕ **Idempotency (chuyên đề):** cập nhật aggregate (`+1`) **không idempotent** → cần cùng transaction / idempotent / CRDT để khỏi drift.
- ➕ **Amdahl (chuyên đề):** chỉ denormalize khi **đọc/JOIN là nút thắt thật** — đừng tối ưu phần không thống trị.
- ➕ **Normalization & anomaly (Codd):** lý do tồn tại normalization; denormalize là **đánh đổi có ý thức** khỏi nó.
- ➕ **CQRS / event sourcing / polyglot persistence / write amplification:** bộ khung kiến trúc quanh denormalization.

---

## ⑫ Anti-patterns — AI & người mới hay sai

| ❌ Sai lầm | Vì sao nguy hiểm | ✅ Đúng |
|---|---|---|
| Denormalize **trước khi đo** đọc có là nút thắt | Tối ưu phần không thống trị (Amdahl) | Đo p99/JOIN cost trước; denormalize nếu thật sự nút thắt |
| **Denormalize tất cả** | Write amplification nổ, schema cứng | Chọn lọc field **đọc-nhiều, ghi-thưa** |
| Bản denormalized thành **sự thật** | Drift → hỏng vĩnh viễn, không biết bản nào đúng | Giữ **source of truth chuẩn hoá**; bản nhân là dẫn xuất |
| **Không reconciliation job** | Drift tích tụ âm thầm (như cache không invalidate) | Job recompute định kỳ + drift metric |
| Update aggregate **khác transaction với nguồn** | Counter drift khi crash/race | Cùng transaction / idempotent / CRDT |
| Lẫn **snapshot** với **duplicate-for-speed** | Cập nhật nhầm bản chụp giá-lúc-mua | Snapshot cố ý đông cứng; duplicate mới cần đồng bộ |
| Denormalize trong workload **ghi-nặng** | Ghi đã là nút thắt → làm tệ hơn | Cân nhắc kỹ; có khi giữ normalized + cache đọc |
| Fan-out-on-write cho **celebrity** | Write amplification khủng | **Lai** fan-out-on-read cho điểm nóng (CS1) |

> 🔎 **AI hay sai (nối Bài 1):** AI hay đề xuất denormalize "cho nhanh" mà **quên rằng mỗi bản nhân là một nghĩa vụ đồng bộ** — rồi đẻ ra drift. Luôn ép hỏi: *"Đọc này có phải nút thắt không? Field này đổi thường xuyên cỡ nào? Khi nguồn đổi, ai cập nhật bản nhân, đồng bộ hay async, và có job sửa drift chưa?"*

---

## ⑬ Khung quyết định / Playbook denormalization

### D-DENORM-1 — Có nên denormalize không?
```
1. Đọc (JOIN/aggregate này) có phải NÚT THẮT thật không? (Amdahl — đo p99)
   → Không → ĐỪNG. Giải bằng index/query/cache trước.
2. Tỉ lệ đọc:ghi? → đọc ≫ ghi → ứng viên tốt; ghi-nặng → cân nhắc lại.
3. Field đổi THƯA hay THƯỜNG XUYÊN? → thưa → tốt; thường xuyên + nhân rộng → ghi nổ.
4. Cần STRONG (giá-lúc-mua) hay chịu EVENTUAL (đếm like)?
   → strong → denormalize ĐỒNG BỘ (cùng transaction).
   → eventual → async (trigger/CDC/job) + reconciliation.
```

### D-DENORM-2 — Triển khai an toàn
```
□ Giữ source of truth CHUẨN HOÁ (bản denormalized là dẫn xuất, dựng lại được).
□ Cập nhật bản nhân: đồng bộ (cùng transaction) hay async (CDC) — chọn rõ.
□ Aggregate update: cùng transaction / idempotent / CRDT (đừng drift).
□ CÓ reconciliation job recompute từ nguồn + drift metric/cảnh báo.
□ Đo write amplification: 1 ghi logic → mấy ghi vật lý? Chấp nhận được?
□ Celebrity/điểm nóng → cân nhắc lai normalized.
```

### D-DENORM-3 — Chọn giữa Cache / MV / Denormalized (Bài 1 D4)
```
Đọc thưa, không động schema, chịu miss        → CACHE (lazy, ngoài DB).
Aggregate đọc nhiều, refresh chu kỳ chấp nhận → MATERIALIZED VIEW.
Một field đọc cực nhiều, cần trong transaction → DENORMALIZED COLUMN (eager, strong).
(Thường KẾT HỢP: denormalized column + cache đọc lên trên.)
```

---

## ⑭ Mental model (câu thần chú gốc)

> 🗂️ **Denormalization = "cache nằm trong database":** bản dẫn xuất, nhân bản, hình-dạng-lại để đọc nhanh — trả giá bằng **ghi phức tạp + drift**.
>
> Khác cache ở ba điểm: **bền** (part of schema), **có thể strong** (cùng transaction với nguồn), **ghi đắt hơn**.
>
> Bốn điều phải nhớ:
> 1. **Đo trước (Amdahl)** — chỉ denormalize khi đọc/JOIN là nút thắt thật.
> 2. **Giữ source of truth chuẩn hoá** — bản nhân chỉ là dẫn xuất, dựng lại được.
> 3. **Aggregate update an toàn** (cùng transaction / idempotent) — đừng để counter drift.
> 4. **Luôn có reconciliation** — denormalized data là cache, sẽ lệch, phải recompute sửa.

---

## ⑮ Câu hỏi phỏng vấn & thách đố

- **(dễ)** Denormalization là gì? → Cố ý đưa trùng lặp/dữ liệu tính sẵn vào schema để đọc nhanh, đổi lại ghi phức tạp + rủi ro drift.
- **(dễ)** Normalize tối ưu gì, denormalize tối ưu gì? → Normalize: ghi & đúng đắn (một sự thật một chỗ). Denormalize: đọc (tránh JOIN/aggregate).
- **(TB)** Cache vs MV vs denormalized column khác nhau ra sao? → Cache (lazy, ngoài DB, volatile, eventual); MV (eager, trong DB, refresh chu kỳ); denormalized column (eager, trong bảng gốc, có thể strong cùng transaction).
- **(TB)** Khi nào denormalize thay vì cache? → Đọc cực nhiều + cần luôn-sẵn + (có thể) cần đúng trong transaction; hoặc NoSQL không JOIN.
- **(bẫy)** Denormalize có làm dữ liệu kém nhất quán hơn cache không? → Không nhất thiết — denormalize **đồng bộ** có thể **strong** (cùng transaction), trong khi cache là eventual.
- **(TB)** Fan-out-on-write vs fan-out-on-read? → Write: ghi sẵn vào timeline mỗi follower (đọc rẻ, ghi đắt/amplification); read: dựng lúc đọc (ghi rẻ, đọc đắt); lai cho celebrity.
- **(khó)** Denormalized count bị lệch dần — chẩn đoán & sửa? → Update count khác transaction với nguồn / race / non-idempotent → drift; sửa: cùng transaction hoặc idempotent/CRDT + reconciliation job recompute.
- **(khó)** Vì sao denormalization và caching cùng một họ vấn đề? → Cả hai là bản dẫn xuất nhân bản để đọc nhanh → cùng rủi ro stale/drift → cùng cần source of truth + cơ chế đồng bộ + reconcile.

> 🧩 **Thách đố:** *"Trang feed chậm vì mỗi post phải JOIN users (lấy tên/avatar) + đếm comments + đếm likes. Đề xuất giải pháp đầy đủ và đánh đổi."*
> → (1) **Đo trước (Amdahl):** xác nhận JOIN+aggregate thống trị p99, không phải render/network. (2) **Denormalize:** nhân `author_name`/`author_avatar` vào `posts`; thêm cột aggregate `comment_count`/`like_count` cập nhật **cùng transaction** lúc thêm comment/like (strong, idempotent qua transaction). (3) **Đổi tên/avatar user** → cập nhật bản nhân ở mọi post (async qua CDC nếu nhiều, chịu eventual) **+ reconciliation job**. (4) **Lên trên** đặt **cache đọc** cho feed (lazy) nếu vẫn cần. (5) Đánh đổi: ghi đắt hơn + write amplification khi user đổi tên + nghĩa vụ chống drift — chấp nhận vì feed đọc ≫ ghi. *Giữ `users` chuẩn hoá làm source of truth.*

---

## ⑯ Bài tập + tiêu chí tự chấm

**BT1.** Cho schema chuẩn hoá `orders / order_items / products`. Chỉ ra field nào nên denormalize cho trang "lịch sử đơn", field nào là **snapshot** (không drift), field nào là **duplicate** (cần đồng bộ).
> ✅ **Đạt khi:** `price_at_purchase`, `product_name_at_purchase` = snapshot (đông cứng, đúng theo thiết kế); nếu nhân `current_stock` thì là duplicate cần đồng bộ; phân biệt rõ hai loại.

**BT2.** Thiết kế cập nhật `like_count` không drift. Nêu phương án đồng bộ và bất đồng bộ + cơ chế chống lệch.
> ✅ **Đạt khi:** đồng bộ = cùng transaction với insert like; async = CDC/job + reconciliation; xử lý non-idempotent của `+1`.

**BT3.** So sánh fan-out-on-write vs fan-out-on-read cho timeline; khi nào lai và vì sao.
> ✅ **Đạt khi:** nêu đánh đổi đọc/ghi + write amplification; lai khi có celebrity (fan-out-on-write nổ).

**BT4.** Giải thích vì sao "denormalized data là một cache" và liệt kê kỷ luật phải áp dụng.
> ✅ **Đạt khi:** bản dẫn xuất → có thể drift → cần source of truth + cơ chế đồng bộ (sync/CDC) + reconciliation + drift metric.

---

## ⑰ Glossary (đóng vào hoàn cảnh)

| Thuật ngữ | Nghĩa nhanh | 🎬 Hoàn cảnh sử dụng |
|---|---|---|
| **Denormalization** | Nhân bản/tính sẵn trong schema để đọc nhanh | *"Denormalize cột thay vì cache khi đọc cực nhiều."* |
| **Normalization / normal forms** | Mỗi sự thật một chỗ (1NF–BCNF) | *"Giữ users chuẩn hoá làm source of truth."* |
| **Update/insert/delete anomaly** | Bất thường do trùng lặp | *"Denormalize đẻ ra anomaly nếu không quản lý."* |
| **Derived / precomputed column** | Cột lưu kết quả tính sẵn | *"comment_count cập nhật lúc ghi."* |
| **Aggregate denormalization** | Đếm/tổng tính sẵn | *"like_count để khỏi COUNT mỗi lần đọc."* |
| **Embedding** | Nhúng dữ liệu liên quan vào một bản ghi | *"Mongo nhúng comments vào post."* |
| **Snapshot** | Chụp giá trị tại thời điểm (đúng theo nghiệp vụ) | *"price_at_purchase đông cứng, không drift."* |
| **Fan-out-on-write** | Nhân dữ liệu vào nhiều đích lúc ghi | *"Ghi tweet vào timeline mỗi follower."* |
| **Write amplification** | 1 ghi logic → nhiều ghi vật lý | *"Celebrity fan-out → amplification khủng."* |
| **Drift** | Bản denormalized lệch nguồn | *"Đổi tên user → author_name drift nếu sót."* |
| **Reconciliation job** | Recompute từ nguồn để sửa lệch | *"Job đêm recompute count sửa drift."* |
| **CQRS** | Tách write model / read model | *"Read model denormalized đồng bộ qua event."* |
| **Materialized view** | Bảng kết quả truy vấn tính sẵn | *"MV cho báo cáo đọc nhiều."* |
| **Star schema** | Fact + dimension denormalized (OLAP) | *"Warehouse dùng star schema cho analytics."* |
| **Polyglot persistence** | Nhiều store, mỗi cái một mục đích đọc | *"Normalized OLTP + denormalized ES/warehouse."* |

---

## ⑱ Đọc thêm

- **E. F. Codd — normalization & normal forms** (gốc lý thuyết quan hệ).
- **Martin Kleppmann, *Designing Data-Intensive Applications*** — derived data, materialized views, fan-out timeline (chương 1 & 11), CDC.
- **Ralph Kimball, *The Data Warehouse Toolkit*** — star schema, dimensional modeling.
- **Rick Houlihan — "DynamoDB Single-Table Design" (AWS re:Invent talks)** — denormalize theo access pattern.
- **Greg Young / Udi Dahan — CQRS & Event Sourcing**.
- **Twitter Engineering — timeline architecture** (fan-out-on-write/read, hybrid).
- **Bài 1 mục (g) & D4** — Cache vs MV vs denormalized (cross-ref nội bộ cụm).

---

> 🔗 **Nối lại cả cụm (9 tài liệu — khép vòng):** Bài 1 dựng khung *tốc độ ↔ tươi ↔ RAM* và đặt denormalization cạnh cache & MV trong họ "dữ liệu dẫn xuất để đọc nhanh". Chuyên đề này cho thấy **denormalization là cache đặt bên trong database** — nên nó **thừa hưởng TOÀN BỘ** bài học của cụm: phải **đo trước** (Amdahl), giữ **source of truth** và quản lý **drift/stale** (Eventual Consistency, Cache Coherence), cập nhật aggregate **idempotent** (Idempotency), và **thu nhỏ working set** khi gộp dữ liệu (Working Set). Câu chốt của cả cụm: ***mọi bản sao bạn tạo ra để đọc nhanh — dù là cache (lazy, ngoài DB), MV (eager, trong DB), hay denormalized column (eager, trong bảng) — đều là một lời hứa giữ-cho-đúng. Tốc độ luôn được mua bằng độ phức tạp của việc giữ lời hứa đó.***
