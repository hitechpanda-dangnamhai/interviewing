# 🎯 BÀI 2 — Chuẩn hóa & Quan hệ (Normalization)

**Phủ:** E-NORM-001 → 006

> 💡 Bài 1 dạy bạn *chọn* được **SQL**. Bài này dạy bạn *thiết kế bảng cho đúng* sau khi đã chọn SQL — tức là biết cách sắp xếp dữ liệu sao cho **không trùng lặp, không mâu thuẫn**, trước khi học cách cố ý phá luật (**denormalize**) ở phần sau.

---

## ① Mục tiêu & vị trí

Bài này nằm ngay sau **Bài 1**: khi bạn đã chốt dùng **SQL**, bước tiếp theo là thiết kế bảng "đúng" — và chỉ *sau khi hiểu thế nào là đúng*, bạn mới được phép biết **khi nào cố ý phá luật** (**denormalize**).

Vị trí trong mạch giáo trình:
- Là **nền cho Bài 3** (data modeling thực chiến) — vì modeling tốt bắt đầu từ schema chuẩn.
- Là **nền cho Bài 4** (index) — vì index đặt trên một **schema tốt** mới phát huy; index trên schema bẩn chỉ là vá víu.

> 🎯 **Chốt định vị:** Học **Normalization** không phải để "đạt điểm 3NF". Học để hiểu *vì sao* trùng lặp dữ liệu là nguồn gốc của bug, rồi *chủ động* quyết định khi nào chấp nhận trùng lặp để đổi lấy tốc độ.

---

## ② Giảng cơ bản → nâng cao

### (a) Trực giác — Normalization là gì và để làm gì

> **Ẩn dụ:** Hãy tưởng tượng bạn lưu địa chỉ của một khách hàng bằng cách **chép tay nó vào 50 tờ hóa đơn**. Hôm khách đổi nhà, bạn phải lục đúng 50 tờ để sửa. Sót một tờ → hệ thống mâu thuẫn: tờ này địa chỉ mới, tờ kia địa chỉ cũ. **Normalization** = thay vì chép 50 lần, ta lưu địa chỉ **một chỗ duy nhất**, 50 hóa đơn chỉ "trỏ" tới chỗ đó.

**Normalization** = loại bỏ trùng lặp để mỗi dữ liệu chỉ có **một nguồn sự thật** (**single source of truth**).

**Vì sao điều này quan trọng đến vậy?** Vì trùng lặp sinh ra **anomaly** — các bất thường xảy ra khi insert/update/delete:

| Loại **anomaly** | Nghĩa | Ví dụ cụ thể (lưu địa chỉ lặp 50 chỗ) |
|------------------|-------|----------------------------------------|
| **update anomaly** | Sửa 1 sự thật phải sửa nhiều chỗ → dễ sót → lệch | Đổi địa chỉ khách, sửa 49/50 chỗ → 1 chỗ còn địa chỉ cũ ❌ |
| **insert anomaly** | Không thêm được dữ liệu nếu thiếu dữ liệu kèm | Không lưu nổi khách mới nếu chưa có đơn hàng nào |
| **delete anomaly** | Xóa cái này vô tình mất luôn cái khác | Xóa đơn hàng cuối → mất luôn thông tin khách |

> 🚨 **Dấu hiệu thiết kế sai:** *Đổi địa chỉ một khách hàng mà phải sửa ở 50 chỗ.* Nếu thấy mình phải "sửa nhiều nơi cho một sự thật", schema đang vi phạm **single source of truth**.

---

### (b) Ba dạng chuẩn cốt lõi  `E-NORM-001`

> 🔎 **Học theo NGHĨA, không học vẹt.** Đừng cố thuộc định nghĩa hàn lâm. Hãy hiểu mỗi dạng chuẩn *cấm điều gì* và *vì sao*.

> **Ví dụ trực giác:** 3 dạng chuẩn giống 3 lớp dọn nhà. **1NF**: không nhét nhiều món vào một ngăn. **2NF**: mỗi món để đúng tủ của nó, không để nhờ. **3NF**: không để món A "ăn theo" vị trí của món B.

| Dạng chuẩn | Cấm điều gì | Giải thích + ví dụ |
|------------|-------------|--------------------|
| **1NF** | ô không **atomic** | Mỗi ô là **một giá trị nguyên tử**. ❌ Nhồi `"đỏ,xanh,vàng"` vào 1 cột. ✅ Tách thành nhiều dòng |
| **2NF** (đã 1NF) | **partial dependency** | Cột **không được phụ thuộc vào *một phần* của khóa kép**. Nếu PK là `(order_id, product_id)` mà `customer_name` chỉ phụ thuộc `order_id` → vi phạm |
| **3NF** (đã 2NF) | **transitive dependency** | Cột **không phụ thuộc gián tiếp qua một cột non-key khác**. Ví dụ `city` phụ thuộc `zipcode`, mà `zipcode` không phải PK → `city` nên tách ra |

**Vì sao thứ tự 1→2→3?** Vì mỗi mức là điều kiện của mức sau: phải **atomic** (1NF) rồi mới xét được phụ thuộc vào khóa (2NF), rồi mới xét được phụ thuộc gián tiếp (3NF). Cứ leo thang từng bậc.

> 💡 **Cách nhớ nhanh:** **1NF = atomic** (không list trong ô) · **2NF = bỏ partial dependency** (không phụ thuộc nửa khóa) · **3NF = bỏ transitive dependency** (không phụ thuộc bắc cầu qua non-key).

---

### (c) Denormalize có chủ đích  `E-NORM-002`

> **Ẩn dụ:** **Normalization** là cất tiền trong két (an toàn, một chỗ). **Denormalize** là để sẵn ít tiền lẻ trong ví (lấy nhanh, nhưng phải nhớ nạp lại ví cho khớp két). Tiện hơn nhưng *bạn gánh trách nhiệm giữ ví và két không lệch*.

**Khi nào denormalize?** Khi **đọc nhiều** + **JOIN đắt** → ta cố ý **nhân bản dữ liệu** để đọc nhanh, khỏi JOIN lại mỗi lần.

**Cái giá phải trả** (luôn có giá):
- ❌ **Ghi phức tạp hơn** — sửa một chỗ phải nhớ sửa các bản sao.
- ❌ **Nguy cơ data lệch** — bản sao quên cập nhật → mâu thuẫn.
- ❌ **Phải đồng bộ** — gánh nặng nằm trên vai bạn, không phải DB.

#### 🔍 Giữ data nhân bản không lệch  `E-NORM-003`

Khi đã nhân bản, có **3 cách** giữ đồng bộ:

| Cách | Cách làm | Đánh đổi |
|------|----------|----------|
| **Cùng transaction** | Cập nhật bản gốc + bản sao trong **cùng một transaction** | ✅ Không bao giờ lệch · ❌ Ghi nặng hơn |
| **Trigger** | DB tự động cập nhật bản sao khi gốc đổi | ✅ Tự động · ❌ Logic ẩn, khó debug |
| **Eventual + job đồng bộ** | Chấp nhận lệch tạm thời, job định kỳ dọn cho khớp | ✅ Ghi nhẹ · ❌ Có cửa sổ **stale** (lệch tạm) |

> ⚠️ Dù chọn cách nào, **phải Ý THỨC được rủi ro**. Denormalize mà "quên" đồng bộ là nguồn của những bug data lệch khó truy nhất.

Timeline minh họa data lệch khi quên đồng bộ:

```text
DENORMALIZE quên đồng bộ (order_item đổi, total_cents không đổi):
  T1: cập nhật order_item → thêm 1 sản phẩm 50.000đ   ✅
  T2: ❌ QUÊN cập nhật cột total_cents (bản nhân bản)
       ↓
  T3: trang đọc total_cents → vẫn hiện tổng CŨ        ❌ stale
       → "tổng tiền" hiển thị ≠ tổng thật của các item → khách thấy sai
```

#### 🔍 Lưu dữ liệu tính sẵn  `E-NORM-004`

Khi cần "kết quả tính sẵn" (như tổng doanh thu ngày), có 3 lựa chọn — **chọn theo độ tươi (freshness) bạn cần**:

| Cách | Độ tươi | Chi phí ghi | Khi nào dùng |
|------|---------|-------------|--------------|
| **materialized view** | Hơi cũ (refresh định kỳ) | Thấp (tính lúc refresh) | Báo cáo, dashboard — chấp nhận trễ phút/giờ |
| **cột denormalized** | **Realtime** | Cao (cập nhật mỗi lần ghi) | Cần số liệu *luôn đúng tức thì* |
| **cache** | Chấp nhận **stale** (theo **TTL**) | Rất thấp | Đọc cực nhiều, lệch chút không sao |

> 🎯 **Chốt:** Không có lựa chọn "tốt nhất" — chỉ có lựa chọn *hợp với độ tươi bạn cần*. Cần đúng-tức-thì → **cột denormalized**. Báo cáo cuối ngày → **materialized view**. Đọc triệu lần, lệch vài giây OK → **cache** với **TTL**.

---

## ③ ⚠️ Cũ → Mới (kiến thức bị thay thế)

| Quan niệm CŨ ❌ | Hiện nay ✅ | Vì sao |
|----------------|------------|--------|
| "Phải chuẩn **3NF** mới đúng" | **3NF** là default tốt; **denormalize** có chủ đích vì performance là **hợp lệ** | Phán xét theo **access pattern** + **đo**, không theo "điểm thi 3NF" `E-NORM-006` |
| **Denormalize** bừa cho nhanh | Chỉ **denormalize** khi **đo được JOIN là bottleneck** | Tránh **nợ kỹ thuật** data lệch |

> 💡 Hai dòng này là hai thái cực sai đối nghịch: một bên *thờ phụng 3NF* (giáo điều), một bên *phá 3NF bừa bãi* (ẩu). Tư duy đúng nằm ở giữa: **3NF làm mặc định, lệch khỏi nó phải có số đo chứng minh**.

---

## ④ Thực tế + bigtech  `E-NORM-005`

### OLTP chuẩn hóa, OLAP denormalize

> **Ví dụ trực giác:** **OLTP** là quầy thu ngân (ghi từng giao dịch nhanh, chính xác từng đồng). **OLAP** là phòng kế toán phân tích cuối tháng (đọc gộp khổng lồ để ra báo cáo). Hai mục đích khác nhau → hai cách tổ chức dữ liệu khác nhau.

| Tiêu chí | **OLTP** (hệ giao dịch) | **OLAP** / warehouse (kho phân tích) |
|----------|-------------------------|--------------------------------------|
| Tối ưu cho | **ghi** + **nhất quán** | **đọc** + **aggregate** (tổng hợp) |
| Cách tổ chức | **normalize** (chuẩn hóa) | **denormalize** — **star schema** / **snowflake schema** |
| Cấu trúc | nhiều bảng quan hệ | **fact table** + **dimension** (ít JOIN) |
| DB điển hình | **Postgres**, **MySQL** | **Snowflake**, **BigQuery**, **Redshift** |

**Vì sao OLTP normalize còn OLAP denormalize?** Vì mục tiêu ngược nhau: **OLTP** cần ghi nhanh và không bao giờ lệch (chuẩn hóa giúp điều đó); **OLAP** cần đọc gộp hàng triệu dòng cực nhanh, JOIN nhiều bảng lúc đó là gánh nặng → người ta **denormalize** sẵn thành **star schema** (một **fact table** ở giữa + các **dimension** xung quanh).

> 🔎 **Pattern bigtech (verify — có thể đã đổi):** Bigtech thường **tách hẳn** **OLTP** (**Postgres**/**MySQL**) và **OLAP** (**Snowflake**/**BigQuery**/**Redshift**), nối với nhau qua **pipeline ETL/ELT**. Lý do: hai workload chọi nhau, để chung sẽ làm chậm cả hai.

---

## ⑤ Code + cấu hình

### Chuẩn hóa: tách `order_item` ra khỏi `order`

```sql
-- Chuẩn hóa (1NF): KHÔNG nhồi danh sách sản phẩm vào 1 cột của order
-- → mỗi sản phẩm trong đơn là 1 dòng riêng ở bảng order_item
CREATE TABLE "order" (
  id          bigint GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
  customer_id bigint NOT NULL REFERENCES customer(id),   -- trỏ tới customer (single source of truth)
  created_at  timestamptz NOT NULL DEFAULT now()
);

CREATE TABLE order_item (
  order_id   bigint NOT NULL REFERENCES "order"(id),
  product_id bigint NOT NULL REFERENCES product(id),
  qty        int    NOT NULL CHECK (qty > 0),            -- số lượng phải dương
  PRIMARY KEY (order_id, product_id)                     -- khóa kép: 1 sản phẩm/đơn chỉ 1 dòng
);
```

### Denormalize có chủ đích: lưu sẵn `total_cents`

```sql
-- Denormalize: lưu sẵn tổng tiền đơn để KHỎI phải SUM(order_item) mỗi lần đọc
ALTER TABLE "order" ADD COLUMN total_cents bigint NOT NULL DEFAULT 0;
-- ⚠️ LỜI HỨA: phải cập nhật total_cents CÙNG transaction khi order_item đổi,
--    nếu không → total_cents lệch khỏi tổng thật (xem timeline mục ②c)
```

### Materialized view cho báo cáo (chấp nhận hơi cũ)

```sql
-- Materialized view: tính sẵn doanh thu theo ngày cho dashboard
-- (refresh định kỳ → dữ liệu hơi cũ nhưng đọc cực nhanh)
CREATE MATERIALIZED VIEW daily_revenue AS
SELECT date_trunc('day', created_at) AS d, SUM(total_cents) AS rev
FROM "order"
GROUP BY 1;

-- CONCURRENTLY = refresh mà không khóa đọc (cần có unique index trên view)
REFRESH MATERIALIZED VIEW CONCURRENTLY daily_revenue;
```

> 🚨 **AI hay sai ở đây:** AI thường **denormalize mà quên cơ chế đồng bộ** — thêm cột `total_cents` nhưng không cập nhật nó cùng transaction; hoặc nhồi `product_list` dạng chuỗi vào một cột (phá **1NF**). Code chạy được nhưng để lại bom hẹn giờ **data lệch**.

---

## ⑥ Keywords

> 🧠 **HIỂU** (nắm bản chất):
> **anomaly** (**insert**/**update**/**delete** anomaly) · **partial dependency** vs **transitive dependency** · **single source of truth** · **trade-off đọc/ghi của denormalize**.

> 📌 **THUỘC** (nhớ chính xác):
> **1NF = atomic** · **2NF = bỏ partial dependency** · **3NF = bỏ transitive dependency** · **star schema** (**fact table** + **dimension**).

> 🛠️ **LÀM ĐƯỢC** (kỹ năng tay):
> Chuẩn hóa một bảng "bẩn" về **3NF**; quyết định **denormalize** một chỗ + nêu rõ cách **giữ đồng bộ**.

---

## ⑦ Mental model

> 🎯 **NGUYÊN TẮC VÀNG CỦA BÀI:**
> *"**Chuẩn hóa để đúng, denormalize để nhanh** — và mỗi lần **denormalize** là một lời hứa **phải tự giữ đồng bộ**."*

Cách dùng: mỗi khi định thêm một bản dữ liệu nhân bản, hãy tự hỏi ngay *"Ai sẽ giữ nó khớp với bản gốc? Bằng cách nào? Nếu lệch thì ai phát hiện?"* Trả lời được mới làm.

---

## ⑧ Phỏng vấn & thách đố

| Mức | Câu hỏi | Trả lời cốt lõi |
|-----|---------|-----------------|
| **(dễ)** | **1NF**/**2NF**/**3NF** nghĩa thực tế? | **atomic** / bỏ **partial dependency** / bỏ **transitive dependency** |
| **(TB)** | Khi nào **denormalize**? Trade-off? | đọc nhiều / **JOIN** đắt; trả giá **ghi phức tạp** + **lệch** |
| **(khá)** | **materialized view** vs **cột denorm** vs **cache**? | theo độ tươi: định kỳ / **realtime** tốn ghi / **TTL** **stale** |
| **(khó)** | Vì sao **OLTP** normalize còn **OLAP** denormalize? | ghi + nhất quán vs đọc + **aggregate** |

> 🧩 **Thách đố tình huống:**
> *Reviewer chê schema của bạn "chưa chuẩn **3NF**" — khi nào đó là vấn đề thật, khi nào là giáo điều?*
>
> Trả lời chuẩn:
> - **Denormalize có chủ đích vì performance là HỢP LỆ** — không phải lỗi.
> - Phán xét phải theo **access pattern** + **số đo**, không theo "điểm 3NF".
> - ⚠️ **Bẫy phải vạch ra:** **"3NF" không phải mục tiêu tự thân**. Nó là công cụ chống **anomaly**, không phải huy chương để khoe. Nếu đã **đo** được JOIN là bottleneck và có cơ chế đồng bộ → việc lệch 3NF là quyết định kỹ thuật đúng.

---

## ⑨ Bài tập + tiêu chí tự chấm

### BT1 — Sửa bảng "bẩn" về chuẩn

Cho bảng `orders(id, customer_name, customer_email, product_list, ...)`. Hãy chỉ ra **vi phạm normal form nào** và tách lại cho đúng.

> ✅ **Tiêu chí Đạt:**
> - Nhận ra **`product_list` vi phạm 1NF** (nhồi list vào 1 ô) → tách bảng **`order_item`**.
> - Nhận ra **`customer_*` nên tách bảng `customer`** (chống **update anomaly** khi khách đổi thông tin).
> - ❌ Trượt nếu chỉ "thấy xấu" mà không gọi tên được normal form bị vi phạm.

### BT2 — "Tổng đơn của user" cho hàng triệu request/ngày

Trang chủ hiển thị **"tổng đơn của user"**, chịu hàng triệu request/ngày. Chọn **materialized view** / **cột denorm** / **cache** + lý do.

> ✅ **Tiêu chí Đạt:**
> - Lý do phải bám **độ tươi cần** + **tần suất ghi**.
> - Ví dụ hợp lệ: tổng đơn ít đổi + cần realtime nhẹ → **cột denorm**; hoặc đọc khổng lồ + chấp nhận lệch giây → **cache TTL**.
> - ❌ Trượt nếu chọn đại không gắn với độ tươi/tần suất ghi.

---

## ⑩ Đọc thêm

- **PostgreSQL — Materialized Views** — `postgresql.org/docs/current/rules-materializedviews.html`
- **Kleppmann, *DDIA* Ch.3** (storage & retrieval) — hiểu sâu vì sao tổ chức dữ liệu ảnh hưởng đọc/ghi

> 🎯 **Một câu mang về:** **3NF** không phải đích đến, nó là điểm xuất phát. Bạn chuẩn hóa để *hiểu rõ dữ liệu*, rồi *có chủ đích* lệch khỏi nó khi — và chỉ khi — số đo bảo bạn nên làm thế.
