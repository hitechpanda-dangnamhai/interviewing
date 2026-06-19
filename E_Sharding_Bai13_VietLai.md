# 🎯 BÀI 13 — Sharding & Partitioning

**Phủ:** E-SHRD-001 → 008

> 💡 Đây là **scale GHI** — **phương án cuối cùng** khi index/replica/cache đã hết đường. Sharding mạnh nhưng *đắt đỏ về độ phức tạp*: một khi đã shard, bạn sống chung với **cross-shard query**, **distributed transaction** và **hot shard** mãi mãi. Nên bài này cũng là bài về *khi nào KHÔNG nên shard*.

---

## ① Mục tiêu & vị trí

Bài này về **sharding** và **partitioning** — cách **scale ghi**.

> 🚨 **Sharding là phương án CUỐI CÙNG** — sau khi index/replica/cache đã hết đường. Vì sao? Vì nó tăng độ phức tạp *khổng lồ* và gần như không thể đảo ngược.

Cần **replication** (**Bài 12**) làm nền. Dẫn vào **Bài 14** (distributed SQL — tự động hóa phần lớn việc này).

> 🎯 **Chốt định vị:** Câu hỏi đúng không phải "shard thế nào" mà "đã thử hết mọi cách rẻ hơn chưa". Shard sai lúc sai thời điểm là một trong những sai lầm kiến trúc tốn kém nhất.

---

## ② Giảng cơ bản → nâng cao

### (a) Partition vs shard  `E-SHRD-001`

> **Ẩn dụ:** **partition** = chia một căn nhà thành nhiều phòng (vẫn một nhà, một địa chỉ). **shard** = chia ra *nhiều căn nhà* ở các địa chỉ khác nhau. Chia phòng thì dễ; chia ra nhiều nhà thì phải lo đi lại giữa chúng.

| Tiêu chí | **partition** | **shard** |
|----------|---------------|-----------|
| Chia gì | một bảng **trong cùng instance** | data qua **nhiều instance/node** |
| Tính chất | **logical** (vẫn một DB) | **vật lý** (nhiều DB) |
| Độ khó | ✅ **dễ** | ❌ **khó** |

> 💡 **Phân biệt cốt lõi:** **partition** vẫn nằm trong **một node**; **shard** trải ra **nhiều node**. Mọi cái khó của sharding đến từ chỗ "nhiều node" này.

---

### (b) Ba kiểu partition  `E-SHRD-002`

> **Ví dụ trực giác:** 3 kiểu chia như 3 cách sắp tủ hồ sơ — theo *khoảng thời gian*, theo *nhóm cố định*, hay *rải đều cho cân*.

| Kiểu | Chia theo | Ví dụ | Hợp khi |
|------|-----------|-------|---------|
| **range** | **khoảng** | theo **thời gian** (mỗi tháng 1 partition) | query theo dải thời gian |
| **list** | **tập giá trị** | theo **region** (VN/US/EU) | data nhóm rõ ràng |
| **hash** | hàm băm | **phân tán đều** | tránh lệch, không có trục tự nhiên |

> 🎯 **Chọn theo query + phân bố data** — không chọn bừa. Data theo thời gian → **range**; data theo nhóm → **list**; cần đều tay → **hash**.

---

### (c) Shard key  `E-SHRD-003`

> 🚨 **shard key là quyết định quan trọng nhất — và khó đảo ngược nhất — của sharding.**

Chọn sai **shard key** → **phân bố lệch** (**hot shard**), **không scale**, **cross-shard query đắt**.

> **Ví dụ trực giác:** Shard theo **ngày** → **shard hôm nay nóng rực** (mọi ghi mới đổ vào đó), **shard cũ nằm im** (không ai đụng). Bạn có nhiều node nhưng chỉ *một* node làm việc → vô nghĩa.

```text
HOT SHARD do shard key = ngày:
  shard[2026-06-19] ← TẤT CẢ ghi hôm nay dồn vào đây   🔥 quá tải
  shard[2026-06-18] ← nằm im                            😴
  shard[2026-06-17] ← nằm im                            😴
       ↓
  ❌ có 3 node nhưng chỉ 1 node gánh tải → không scale được gì
  ✅ sửa: shard theo hash(user_id) → ghi rải đều khắp các node
```

---

### (d) Mép giới hạn

#### 🚨 Cross-shard là ác mộng  `E-SHRD-004`

> **Ẩn dụ:** Query xuyên shard giống đi hỏi *từng căn nhà riêng lẻ* rồi tự gom kết quả (**scatter-gather**). Không có "JOIN tự nhiên" giữa các nhà; muốn giao dịch nguyên tử qua nhiều nhà thì cực kỳ rối (**distributed transaction**).

**query/JOIN/transaction xuyên shard** → **scatter-gather** nhiều node, **không JOIN tự nhiên**, **distributed tx phức tạp**.

> 🎯 **Chốt:** Phải **thiết kế shard key để NÉ cross-shard** — gom data hay-truy-vấn-cùng-nhau vào cùng một shard (ví dụ shard theo `tenant_id` để mọi data của một tenant nằm chung).

#### 🔍 Consistent hashing  `E-SHRD-005`

> **Ví dụ trực giác:** Với **hash % N**, khi đổi số node N → **gần như mọi key phải tính lại** (rehash) → di chuyển dữ liệu khổng lồ. **consistent hashing** sắp key/node trên một "vòng tròn" → thêm/bớt node chỉ ảnh hưởng *một phần nhỏ* key kề bên.

**consistent hashing** = **giảm tối thiểu số key phải di chuyển khi thêm/bớt node** (so với **hash % N** phải **rehash gần như toàn bộ**).

```text
hash % N khi N đổi 3 → 4:
  key 10: 10 % 3 = 1  →  10 % 4 = 2   ❌ phải move
  key 11: 11 % 3 = 2  →  11 % 4 = 3   ❌ phải move
  → gần như MỌI key đổi chỗ → di chuyển dữ liệu khổng lồ

consistent hashing khi thêm 1 node:
  → chỉ các key NẰM KỀ node mới phải move, phần còn lại y nguyên   ✅
```

#### 🔍 Declarative partitioning Postgres  `E-SHRD-007`

Ngoài tốc độ query (**partition pruning** — chỉ quét partition liên quan), còn cho:
- **DROP/DETACH partition** để **xóa data cũ tức thì** (vs **DELETE** nặng + **bloat** — Bài 9),
- **VACUUM/bảo trì theo từng partition**.

```text
Xóa data cũ: DELETE vs DROP PARTITION
  DELETE FROM orders WHERE created_at < '2025-01-01':
    → quét + xóa hàng triệu dòng → tạo dead tuple khổng lồ → bloat   ❌ nặng
  DROP TABLE orders_2024 (cả partition):
    → bỏ nguyên một mảnh, gần như TỨC THÌ, không bloat   ✅
```

---

## ③ ⚠️ Cũ → Mới (kiến thức bị thay thế)

| Quan niệm CŨ ❌ | Hiện nay ✅ | Vì sao |
|----------------|------------|--------|
| "Tải cao = **shard ngay**" | **Shard là phương án cuối** `E-SHRD-006` | Shard tăng **độ phức tạp khổng lồ** |
| **hash % N** chia shard | **Consistent hashing** | Đổi N không phải **rehash toàn bộ** |
| **DELETE** data cũ hằng tháng | **Range partition** + **DROP PARTITION** | Xóa **tức thì**, không **bloat** |
| Tự **shard tay** đầu tiên | Cân nhắc **Citus**/**NewSQL** trước | Đỡ tự gánh **distributed tx** |

---

## ④ Thực tế + bigtech

### Lộ trình TL cho bảng `orders` ~2 tỷ dòng, tải tăng  `E-SHRD-008`

> 🧩 **Lộ trình có thứ tự** (đo trước, leo thang dần — **E-SHRD-006**):

```text
1. ĐO bottleneck trước (đừng đoán)
        ↓
2. index/partition + read replica + cache    ← thử hết cách rẻ
        ↓
3. cân nhắc Citus (Postgres extension shard) HOẶC NewSQL
        ↓
4. shard ở tầng app  ← CHỈ khi thật sự cần (cửa cuối)
```

> ⭐ **Trước khi shard hãy thử HẾT:** tuning **index/query**, **vertical scale** (nâng cấu hình máy), **read replica** (Bài 12), **caching**, **partitioning**.

> 🔎 **Bigtech (verify):** **Vitess** (cho **MySQL**), **Citus** (cho **Postgres**) **đẩy việc shard xuống tầng hạ tầng** thay vì code app — bạn đỡ phải tự viết logic phân tán.

---

## ⑤ Code + cấu hình

```sql
-- Range partition theo tháng (Postgres declarative partitioning)
CREATE TABLE orders (id bigint, created_at timestamptz NOT NULL, ...)
  PARTITION BY RANGE (created_at);

CREATE TABLE orders_2026_06 PARTITION OF orders
  FOR VALUES FROM ('2026-06-01') TO ('2026-07-01');   -- partition tháng 6/2026

-- Xóa data cũ TỨC THÌ (vs DELETE nặng + bloat)
ALTER TABLE orders DETACH PARTITION orders_2025_06;   -- tách ra
-- hoặc: DROP TABLE orders_2025_06;                    -- bỏ hẳn

-- Partition pruning: query WHERE created_at trong 1 tháng → chỉ quét 1 partition
EXPLAIN SELECT * FROM orders
WHERE created_at >= '2026-06-10' AND created_at < '2026-06-11';
```

> 🚨 **AI hay sai ở đây:** AI hay khuyên **shard ngay khi tải cao** (bỏ qua index/replica/cache/partition rẻ hơn), hoặc chọn **shard key theo thời gian** (gây **hot shard**), hoặc dùng **DELETE** xóa data cũ thay vì **DROP PARTITION** (tạo bloat khổng lồ).

---

## ⑥ Keywords

> 🧠 **HIỂU** (nắm bản chất):
> **partition** (1 node) vs **shard** (N node) · **hot shard** · **cross-shard scatter-gather** · **consistent hashing** · **partition pruning**.

> 📌 **THUỘC** (nhớ chính xác):
> **range/list/hash partition** · **shard là phương án cuối** · **DROP PARTITION ≫ DELETE**.

> 🛠️ **LÀM ĐƯỢC** (kỹ năng tay):
> Vạch **lộ trình scale có thứ tự** · chọn **shard key tránh hotspot** · dùng **range partition** cho data theo thời gian.

---

## ⑦ Mental model

> 🎯 **NGUYÊN TẮC VÀNG CỦA BÀI:**
> *"**Partition chia trong 1 nhà; shard chia ra nhiều nhà**. **Shard là cửa cuối** — thử **index/replica/cache/partition** trước. **Shard key sai = hot shard + cross-shard địa ngục**."*

Cách dùng: nghe đề xuất "shard đi", hỏi ngay *"đã thử index/replica/cache/partition chưa? đã đo bottleneck chưa?"*. Nếu phải shard, dồn toàn lực chọn **shard key** né cross-shard và tránh hot shard.

---

## ⑧ Phỏng vấn & thách đố

| Mức | Câu hỏi | Trả lời cốt lõi |
|-----|---------|-----------------|
| **(dễ)** | **Partition** vs **shard**? | cùng instance vs nhiều instance |
| **(TB)** | 3 kiểu partition hợp gì? | **range**/**list**/**hash** |
| **(khá)** | **Shard key** sai gây gì? Ví dụ hot shard | lệch/**scatter-gather**; shard theo ngày |
| **(khá)** | Vì sao **cross-shard JOIN/tx** là ác mộng? | không JOIN tự nhiên + **distributed tx** |
| **(khá)** | **Consistent hashing** giải gì? | giảm key phải move khi đổi node |
| **(TB)** | **Partition** Postgres giúp gì ngoài tốc độ? | **DROP/DETACH** + bảo trì theo partition |

> 🧩 **Thách đố tình huống (TL):**
> *2 tỷ dòng, tải tăng — lộ trình & mốc shard?*
>
> Trả lời chuẩn: **đo** → **index/partition/replica/cache** → **Citus/NewSQL** → **shard cuối cùng**.

---

## ⑨ Bài tập + tiêu chí tự chấm

### BT1 — Bảng `events` 3 tỷ dòng, chỉ query 30 ngày gần nhất, xóa data >1 năm

Thiết kế.

> ✅ **Tiêu chí Đạt:**
> - **range partition theo thời gian** + **DROP partition cũ** + **partition pruning**.
> - Giải thích vì sao DROP partition tốt hơn DELETE (tức thì, không bloat).
> - ❌ Trượt nếu dùng một bảng phẳng + DELETE định kỳ.

### BT2 — Đề xuất shard theo `created_at` cho hệ ghi lớn

Chỉ ra rủi ro + thay bằng gì.

> ✅ **Tiêu chí Đạt:**
> - Rủi ro: **hot shard hôm nay** (mọi ghi dồn một shard).
> - Thay bằng **shard theo `hash(tenant_id)`/`user_id`** để phân bố đều.
> - ❌ Trượt nếu không nhận ra hot shard của shard-theo-thời-gian.

---

## ⑩ Đọc thêm

- **PostgreSQL — Table Partitioning** — `postgresql.org/docs/current/ddl-partitioning.html`
- **Citus docs** (`citusdata.com`); **Vitess docs**

> 🎯 **Một câu mang về:** Sharding không phải huy chương "hệ thống lớn" — nó là khoản nợ phức tạp trả mãi không hết. Trước khi shard, hãy thử *mọi thứ* rẻ hơn; khi buộc phải shard, hãy dồn hết trí lực vào **shard key**.
