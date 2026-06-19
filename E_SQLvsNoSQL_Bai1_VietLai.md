# 🎯 BÀI 1 — SQL vs NoSQL & Chọn mô hình dữ liệu

**Phủ:** E-SQLNO-001 → 008

> 💡 **Mục E** là phần "chọn nền móng". Bài này là viên gạch đầu tiên: học xong, bạn cầm một bài toán bất kỳ là chọn được **loại database** phù hợp và **bảo vệ được lựa chọn bằng lý do kỹ thuật**, không phải bằng cảm tính hay trend.

---

## ① Mục tiêu & vị trí  `E-SQLNO-001`

Đây là **bài nền của cả mục E**. Mọi bài sau đều giả định rằng bạn đã chốt kiến trúc dữ liệu ngay tại đây — nghĩa là nếu bạn chọn sai ở Bài 1, các bài sau sẽ "xây nhà trên cát".

Học xong bài này, bạn phải làm được 3 việc:

1. Nhìn một bài toán → **liệt kê được access pattern** (dữ liệu được đọc/ghi theo kiểu nào).
2. Chọn được **loại DB** hợp lý cho bài toán đó.
3. **Bảo vệ lựa chọn bằng lý do kỹ thuật đo được**, không nói "vì NoSQL nhanh hơn".

> 🎯 **Chốt định vị:** Bài này không dạy bạn "DB nào tốt nhất". Nó dạy bạn cách *đặt câu hỏi đúng* trước khi chọn DB. Câu hỏi đúng quan trọng hơn câu trả lời.

---

## ② Giảng cơ bản → nâng cao

### (a) Trực giác — đừng hỏi câu hỏi sai

> ⚠️ Câu hỏi **"SQL hay NoSQL nhanh hơn?"** là **câu hỏi sai**. Nó giống như hỏi "xe máy hay xe tải nhanh hơn?" — vô nghĩa nếu chưa biết bạn cần chở gì, đi đường nào, xa bao nhiêu.

> **Ẩn dụ:** Chọn **database** như chọn **phương tiện di chuyển**. Không tồn tại "xe nhanh nhất" — chỉ có "xe hợp việc nhất". Chở 1 người đi làm → xe máy. Chở 5 tấn hàng → xe tải. Chở khách VIP → xe sang. Cùng một câu "nhanh", nhưng mỗi việc cần một loại xe.

Thay vì hỏi cái nhanh hơn, hãy hỏi **3 câu hỏi nền tảng** sau (đây là khung tư duy bạn dùng cả đời):

| # | Câu hỏi | Thuật ngữ | Ví dụ cụ thể |
|---|---------|-----------|--------------|
| 1 | Dữ liệu được **đọc/ghi theo kiểu nào**? | **access pattern** | "Luôn lấy 1 đơn hàng theo `order_id`" vs "Lọc sản phẩm theo nhiều bộ lọc động" |
| 2 | **Sai một xíu có chết người không**? | **consistency requirement** | Số dư ví tiền sai → chết người ❌. Lượt xem video lệch vài giây → không sao ✅ |
| 3 | Dữ liệu **nhiều quan hệ chéo** hay **khép kín**? | mô hình (relational vs **document**) | "User ↔ Order ↔ Product ↔ Review" cần **JOIN** vs "1 hồ sơ user gói gọn 1 chỗ" |

> 💡 **Mental model nhỏ:** **access pattern** quyết định *hình dạng* dữ liệu; **consistency requirement** quyết định *mức độ nghiêm ngặt* cần có; *mô hình quan hệ* quyết định *bạn có cần JOIN không*. Ba trục này, không phải tốc độ, mới là thứ chọn DB cho bạn.

**Vì sao 3 câu này quan trọng hơn tốc độ?** Vì tốc độ là *kết quả* của việc chọn đúng, không phải *tiêu chí* để chọn. Một DB "nhanh về lý thuyết" nhưng buộc bạn JOIN tay 5 lần ở tầng app sẽ chậm hơn nhiều so với một **RDBMS** làm JOIN đó trong một câu lệnh. Tốc độ thực tế **luôn tùy workload**.

---

### (b) Bốn họ NoSQL  `E-SQLNO-002`

> **Ví dụ trực giác:** Bốn họ **NoSQL** giống bốn loại tủ đựng đồ khác nhau. **Document** = tủ hồ sơ (mỗi ngăn 1 bộ giấy tờ hoàn chỉnh). **Key-value** = tủ khóa số (đưa số → lấy đúng ô đó). **Wide-column** = nhà kho theo dãy (ghi cực nhanh vào từng dãy đã đánh số). **Graph** = sơ đồ quan hệ họ hàng (đi từ người này lần ra người kia).

| Họ | Mô hình | Truy cập tối ưu | DB tiêu biểu | Use case điển hình |
|----|---------|-----------------|--------------|--------------------|
| **Document** | **JSON** lồng nhau | đọc/ghi **1 aggregate** theo `id` | **MongoDB** | catalog sản phẩm, hồ sơ user |
| **Key-value** | key → value | `get`/`set` theo **key** | **Redis**, **DynamoDB** | **cache**, **session**, **counter** |
| **Wide-column** | **partition key** → cột | ghi cực lớn theo **partition** | **Cassandra**, **ScyllaDB** | **time-series**, feed, log |
| **Graph** | **node** + **edge** | duyệt quan hệ **nhiều bậc** | **Neo4j** | social, fraud, recommend |

Giải thích từng họ kèm "vì sao hợp việc đó":

- **Document** (**MongoDB**): lưu nguyên một **aggregate** (một cụm dữ liệu luôn-đi-cùng-nhau) dưới dạng **JSON** lồng. 💡 Hợp khi bạn *luôn đọc cả cụm theo `id`* — ví dụ trang chi tiết sản phẩm cần tên + giá + mô tả + thuộc tính cùng lúc.
- **Key-value** (**Redis**, **DynamoDB**): đơn giản nhất — đưa **key**, nhận **value**. 💡 Cực nhanh vì không phải nghĩ gì ngoài tra key. Hợp với **cache** (key = câu query, value = kết quả), **session** (key = session_id), **counter** (key = "views:video123").
- **Wide-column** (**Cassandra**, **ScyllaDB**): tối ưu cho **ghi cực lớn** phân tán theo **partition key**. 💡 Hợp với **time-series**/log vì dữ liệu chỉ ghi-thêm (append) khổng lồ, hiếm khi sửa.
- **Graph** (**Neo4j**): coi **quan hệ** (**edge**) là công dân hạng nhất. 💡 Hợp khi bạn cần đi *nhiều bậc* — "bạn của bạn của bạn", "tài khoản nào liên đới tài khoản gian lận này".

**Còn SQL/RDBMS thì sao?** **SQL/RDBMS** (**PostgreSQL**, **MySQL**) = **quan hệ** + **JOIN** + **ACID** mạnh + **schema** tường minh.

> 🎯 **RDBMS là "default an toàn".** Vì sao? Vì nó **không ép bạn phải đoán trước access pattern**. Bạn cứ chuẩn hóa dữ liệu, đến lúc cần query kiểu mới thì viết thêm câu **JOIN** là xong — không phải đập đi xây lại. NoSQL ngược lại: chọn sai cách tổ chức từ đầu thì sửa rất đau.

---

### (c) Mép giới hạn / sai lầm thường gặp

#### 🔍 "Schema-less" ≠ "khỏi thiết kế schema"  `E-SQLNO-005`

> ⚠️ Đây là hiểu lầm nguy hiểm nhất với người mới. **schema-less** **không** có nghĩa là "không cần schema". Nó có nghĩa là **schema bị dời lên tầng application** thay vì được DB ép buộc.

**Vì sao bỏ qua việc thiết kế schema lại tệ?** Điều tệ nhất xảy ra là **schema drift** — sau 6 tháng, cùng một collection có document ghi `price`, document khác ghi `price_cents`, document khác nữa ghi `cost`; có cái `userId` là số, cái khác là chuỗi. Hậu quả:

- ❌ Bạn phải **tự validate** mọi thứ ở tầng app (DB không còn là "lưới an toàn cuối cùng").
- ❌ Query **khó tối ưu** vì dữ liệu không đồng nhất, index không ăn đều.
- ❌ Bug âm thầm: code cũ đọc field cũ trả về `undefined`, không ai biết.

> 🎯 **Chốt:** **schema-less** = "tự do *hoãn* quyết định schema", không phải "tự do *bỏ*" nó. Cái giá của tự do là **trách nhiệm tự quản schema**.

#### 🔍 "NoSQL không có ACID" — đã lỗi thời  `E-SQLNO-004`

> ⚠️ Câu **"NoSQL không có ACID"** từng đúng nhưng nay đã **lỗi thời**. **MongoDB** có **multi-doc ACID** (transaction trên nhiều document) **từ phiên bản 4.0/4.2**.

**Vì sao đừng nói tuyệt đối?** Vì hệ sinh thái đã tiến hóa. Cách nói **chính xác** là: **"ACID tùy DB"** — không phải tùy nhãn "SQL/NoSQL". 

> 🔎 **Lưu ý sâu:** **multi-doc ACID** trong **MongoDB** *có* tồn tại nhưng **kèm chi phí và giới hạn** (hiệu năng, ràng buộc về cấu hình). Nó là công cụ "dùng khi cần", không phải "bật mặc định cho vui". Nếu app của bạn *thường xuyên* cần transaction nhiều bản ghi, đó là tín hiệu mạnh rằng có thể **RDBMS** mới đúng nhà.

#### 🔍 Embed vs Reference (trong MongoDB)  `E-SQLNO-003`

Đây là quyết định mô hình hóa quan trọng nhất khi dùng **MongoDB**.

> **Ẩn dụ:** **embed** = đóng tất cả vào *một thùng* (mở thùng là thấy hết). **reference** = để trong *nhiều thùng* và dán *nhãn chỉ đường* sang thùng khác (phải đi lấy thêm).

| Tiêu chí | **embed** (nhúng) | **reference** (tham chiếu) |
|----------|-------------------|----------------------------|
| Khi nào dùng | dữ liệu **đọc-cùng** + **bounded** (có giới hạn số lượng) | dữ liệu **unbounded** / **chia sẻ** / **cập nhật độc lập** |
| Tốc độ đọc | ✅ Nhanh (1 lần đọc ra hết) | ❌ Chậm hơn (phải "join tay" — đọc nhiều lần) |
| Rủi ro | ❌ Document **phình to** (giới hạn **16MB/doc**) | ✅ Document gọn, không phình |
| Ví dụ | user + vài địa chỉ giao hàng (bounded) | user + hàng nghìn comment (unbounded) |

**Vì sao bounded thì embed, unbounded thì reference?** Vì **MongoDB** giới hạn **16MB mỗi document**. Nếu bạn **embed** một danh sách *không có trần* (như comment của user nổi tiếng), document sẽ lớn dần đến vỡ giới hạn — và mỗi lần đọc user bạn lôi về cả MB comment không cần. **reference** tách chúng ra: đọc user nhẹ tênh, cần comment thì query riêng.

So sánh luồng đọc bằng timeline:

```text
EMBED — đọc 1 user kèm địa chỉ (bounded):
  T1: client → findOne({_id: 1})
  T2: DB trả về { name, addresses:[...] }   ✅ 1 round-trip, có hết
      → nhanh vì mọi thứ đã nằm trong 1 document

REFERENCE — đọc 1 user kèm comment (unbounded):
  T1: client → findOne(users, {_id: 1})        → ra user
  T2: client → find(comments, {userId: 1})     → ra comment
      → 2 round-trip = "join tay" ở tầng app
      ✅ đổi lại: document user không bao giờ phình vỡ 16MB
```

> 🎯 **Khẩu quyết:** *"Bounded + đọc-cùng → **embed**. Unbounded / chia sẻ / sửa-riêng → **reference**."*

#### 🔍 Wide-column là "query-first"  `E-SQLNO-006`

> **Ví dụ trực giác:** **RDBMS** là "lưu data cho gọn rồi tính sau". **Wide-column** ngược lại: "biết trước sẽ hỏi gì → thiết kế bảng *vừa khít* câu hỏi đó". Giống như sắp sẵn quầy bánh theo đúng đơn khách hay gọi, thay vì để nguyên liệu rồi chế biến lúc khách tới.

**query-first** nghĩa là: bạn thiết kế bảng **quanh truy vấn đã biết trước**, **chấp nhận duplicate** dữ liệu, **không chuẩn hóa** (denormalize).

**Vì sao chấp nhận duplicate?** Vì trong **wide-column** (**Cassandra**/**ScyllaDB**), JOIN gần như không có. Muốn 3 kiểu truy vấn → bạn tạo 3 bảng lưu cùng dữ liệu theo 3 cách sắp xếp. Tốn dung lượng, nhưng đổi lại mỗi truy vấn chỉ đọc đúng 1 partition → cực nhanh và scale phẳng.

> 🔎 **Wide-column thắng Postgres khi nào?** Khi đồng thời có: **ghi cực lớn** + **multi-region** (nhiều vùng địa lý) + **query theo partition key**. Thiếu một trong ba, thường **Postgres** vẫn là lựa chọn đơn giản hơn và đủ tốt.

---

## ③ ⚠️ Cũ → Mới (kiến thức bị thay thế)

> 🚨 Bốn niềm tin dưới đây từng được dạy như chân lý, nay đã sai hoặc thiếu. Thuộc bảng này giúp bạn không bị "tư duy 2015" dẫn đi lạc.

| Quan niệm CŨ ❌ | Hiện nay ✅ | Vì sao |
|----------------|------------|--------|
| "**NoSQL** nhanh hơn **SQL**" | Chọn theo **access pattern** + **consistency** + mô hình | Tốc độ **luôn tùy workload**, không có hằng số |
| "**NoSQL** không có **ACID**" | **Tùy DB**; **Mongo 4.0+** có **multi-doc ACID** | Hệ sinh thái đã tiến hóa |
| "**schema-less** = khỏi thiết kế" | **schema** dời lên app, vẫn phải quản | Tránh **schema drift** |
| "Scale = đổi sang **NoSQL**" | **Postgres** + **JSONB**/partition/replica thường đủ | Đổi DB **rất tốn** — công sức, rủi ro, tiền |

> 💡 Điểm chung của 4 dòng: tư duy cũ tuyệt-đối-hóa ("luôn", "không bao giờ"); tư duy mới *điều kiện hóa* ("tùy", "khi nào"). Kỹ sư giỏi nói "tùy", rồi nêu rõ tùy theo cái gì.

---

## ④ Thực tế + bigtech  `E-SQLNO-007`

### Polyglot persistence — "đa ngôn ngữ lưu trữ"

> **Ẩn dụ:** **polyglot persistence** giống một căn bếp chuyên nghiệp: dao thái thịt, dao gọt hoa quả, kéo cắt cánh gà — *mỗi việc một dụng cụ*. Không ai dùng một con dao cho mọi món.

Một hệ thống thật thường dùng **nhiều store song song**, mỗi cái cho đúng việc nó giỏi:

| Store | Vai trò | Vì sao chọn nó |
|-------|---------|----------------|
| **PostgreSQL** | **record** + **tiền** (nguồn sự thật) | **ACID** mạnh, không được sai số dư |
| **Redis** | **cache** | đọc cực nhanh, giảm tải cho Postgres |
| **Elasticsearch** | **search** | tìm full-text, lọc đa tiêu chí giỏi |
| **Cassandra** (khi cần) | event stream | **ghi cực lớn** kiểu append |

> ⚠️ **Trade-off của polyglot persistence:** **vận hành phức tạp hơn** (nhiều hệ thống phải nuôi, giám sát, backup) + **phải đồng bộ giữa các store** (ghi vào nhiều nơi — thường gọi là **dual-write** — dễ lệch nhau).

**Vì sao "đồng bộ giữa các store" là cái bẫy lớn?** Vì không có **ACID** trùm lên *nhiều hệ thống khác nhau*. Ghi Postgres xong ghi Redis là **2 thao tác tách rời** — nếu cái thứ hai lỗi, hai store lệch nhau:

```text
DUAL-WRITE bị lệch (Postgres + Redis):
  T1: ghi PostgreSQL  → price = 200  ✅ thành công
  T2: ghi Redis cache → price = 200  ❌ network lỗi, KHÔNG ghi được
       ↓
  T3: client đọc từ Redis → vẫn thấy price = 150 (giá CŨ)   ❌ stale
       → DB nói 200, cache nói 150 → hai nguồn "sự thật" mâu thuẫn
```

> 🎯 **Chốt:** Dùng **polyglot persistence** là đánh đổi *sức mạnh chuyên biệt* lấy *gánh nặng đồng bộ + vận hành*. Chỉ thêm một store mới khi lợi ích vượt rõ cái giá này.

### Bigtech nói gì?

> 🔎 **Lưu ý (pattern report công khai — có thể đã đổi, cần verify):**
> - Nhiều sản phẩm khổng lồ vẫn chạy **lõi quan hệ cho OLTP** (giao dịch trực tuyến). **RDBMS** chưa hề "hết thời".
> - **Discord** từng đi hành trình **MongoDB → Cassandra → ScyllaDB** cho message store (chỗ ghi tin nhắn cực lớn).
> - **Redis** gần như là lựa chọn **mặc định làm cache** ở khắp nơi.

💡 Bài học rút ra: ngay cả bigtech cũng *đổi store theo bài toán* và *theo thời gian*, chứ không trung thành một công nghệ. **OLTP** lõi → quan hệ; ghi khổng lồ → wide-column; cache → Redis.

---

## ⑤ Code + cấu hình

### A. Thuần quan hệ — khi thuộc tính **ỔN ĐỊNH**

```sql
-- PostgreSQL 18.4 (verify tại postgresql.org)
-- Dùng kiểu thuần quan hệ khi tập thuộc tính ÍT thay đổi
CREATE TABLE product (
  id          bigint GENERATED ALWAYS AS IDENTITY PRIMARY KEY,  -- khóa tự sinh
  sku         text        NOT NULL UNIQUE,                       -- mã SKU duy nhất
  name        text        NOT NULL,
  price_cents bigint      NOT NULL CHECK (price_cents >= 0),     -- ⚠️ tiền lưu bằng SỐ NGUYÊN cents, KHÔNG float
  created_at  timestamptz NOT NULL DEFAULT now()                 -- timestamptz (có timezone), KHÔNG dùng timestamp trần
);
```

> 🚨 **Hai cái sai kinh điển AI hay sinh ra:**
> - **price** kiểu **float** → sai số nhị phân làm lệch tiền (0.1 + 0.2 ≠ 0.3). Luôn dùng **`price_cents` bigint** (lưu đơn vị nhỏ nhất, ví dụ 250000 = 2.500đ).
> - **timestamp** trần (không timezone) → loạn múi giờ. Luôn dùng **`timestamptz`**.

### B. JSONB — khi thuộc tính **LINH HOẠT**

```sql
-- Khi thuộc tính sản phẩm hay đổi (mỗi loại hàng có field khác nhau)
ALTER TABLE product ADD COLUMN attributes jsonb NOT NULL DEFAULT '{}';   -- JSONB linh hoạt
CREATE INDEX idx_product_attrs ON product USING gin (attributes);        -- GIN index để query bên trong JSONB nhanh
SELECT id, name FROM product WHERE attributes @> '{"color":"red"}';      -- @> = "chứa" key/value này
```

> 💡 **Đây chính là lý do "Scale = đổi sang NoSQL" thường sai:** **Postgres** + **JSONB** + **GIN index** cho bạn *sự linh hoạt kiểu document* mà **vẫn giữ ACID + JOIN** của quan hệ. Bạn được cả hai thế giới mà không phải đổi DB.

### C. MongoDB — embed (bounded) vs reference (unbounded)

```javascript
// EMBED: địa chỉ là bounded (mỗi user vài cái) → nhúng thẳng vào user
db.users.insertOne({ _id:1, name:"An", addresses:[{city:"HCMC"}] });   // embed: đọc 1 phát ra hết

// REFERENCE: order là unbounded + cập nhật độc lập → để bảng riêng, chỉ giữ userId
db.orders.insertOne({ _id:100, userId:1, total_cents:250000 });        // reference: trỏ về user qua userId
```

### Cấu hình — kỷ luật bắt buộc

- ⚠️ **KHÔNG hardcode connection string** trong code → đưa vào **`.env`** và **không commit** `.env` (tránh lộ mật khẩu DB lên Git).
- ⚠️ **Pin version** thư viện (ví dụ `psycopg[binary]==3.x`) → tránh "máy tôi chạy được, máy bạn lỗi" do version lệch.

> 🚨 **AI hay sai ở đây:** AI thường sinh schema dùng **`text` cho mọi thứ** + **timestamp** trần + **float cho tiền**. Code đó **chạy được nhưng sai kiến trúc** — và sai kiến trúc DB là loại nợ kỹ thuật đắt nhất để trả về sau.

---

## ⑥ Keywords

> 🧠 **HIỂU** (nắm bản chất, giải thích được):
> **access pattern** (đọc/ghi theo kiểu gì) · **consistency requirement** (sai có chết người không) · **polyglot persistence** (mỗi việc một store) · **schema drift** (dữ liệu lệch chuẩn theo thời gian) · **embed/reference** (nhúng vs tham chiếu) · **query-first** (thiết kế bảng quanh câu hỏi).

> 📌 **THUỘC** (nhớ chính xác, đọc là bật ra):
> **4 họ NoSQL** (**Document**/**Key-value**/**Wide-column**/**Graph**) + DB tiêu biểu mỗi họ · **Mongo multi-doc ACID từ 4.0/4.2** · **Postgres = default OLTP**.

> 🛠️ **LÀM ĐƯỢC** (kỹ năng tay):
> Từ 1 yêu cầu → liệt kê **access pattern** → chọn DB + nêu lý do · chọn **embed**/**reference** cho một quan hệ trong **Mongo** + giải thích bounded/unbounded.

---

## ⑦ Mental model

> 🎯 **NGUYÊN TẮC VÀNG CỦA BÀI:**
> *"Chọn DB theo **access pattern** + **nhất quán (consistency)** + **quan hệ**, **KHÔNG** theo tốc độ. **Postgres là mặc định**; lệch khỏi nó phải có **lý do đo được**."*

Cách dùng mental model này trong đời thực: mỗi khi có người (hoặc chính bạn) định chọn một DB "lạ", hãy hỏi ngược: *"Postgres không làm được việc này à? Đo được con số nào chứng minh nó không đủ?"* Nếu không trả lời được, mặc định quay về **Postgres**.

---

## ⑧ Phỏng vấn & thách đố

| Mức | Câu hỏi | Trả lời cốt lõi |
|-----|---------|-----------------|
| **(dễ)** | Chọn **SQL/NoSQL** dựa trên gì? | **access pattern** + **consistency** + mô hình quan hệ |
| **(TB)** | Kể **4 họ NoSQL** + use case + DB | → bảng ở mục ②(b) |
| **(khá)** | **embed** vs **reference**? | **bounded** → **embed**; **unbounded** → **reference** |
| **(khó)** | "**NoSQL** không **ACID**" — chính xác không? | Không. **Tùy DB**; **Mongo 4.0+** có, kèm chi phí/giới hạn |

> 🧩 **Thách đố tình huống (rất hay bị hỏi):**
> *Team đòi đổi sang **MongoDB** "để scale dễ" — bạn hỏi/kiểm gì trước khi gật đầu?*
>
> Trả lời chuẩn — kiểm 4 thứ:
> 1. **access pattern thật** là gì (chứ không phải tưởng tượng)?
> 2. App có cần **transaction** / **JOIN** nhiều không? (Nếu có → Mongo không phải lợi thế)
> 3. **Chi phí migration** (đổi DB rất tốn) đã tính chưa?
> 4. **Postgres đã thực sự đủ chưa** (đã thử **JSONB**/partition/replica)?
>
> ⚠️ **Bẫy phải vạch ra:** **Mongo cũng phải chọn shard key đúng mới scale được**. "Đổi sang Mongo" không tự động cho bạn scale — chọn sai **shard key** còn scale tệ hơn Postgres.

---

## ⑨ Bài tập + tiêu chí tự chấm

### BT1 — Gán store cho hệ e-commerce

Cho 3 thành phần, hãy **gán loại store** + **nêu lý do bám access pattern**:
- (a) đơn hàng + thanh toán
- (b) giỏ hàng tạm
- (c) gợi ý "mua cùng"

> ✅ **Tiêu chí Đạt:**
> - (a) → **RDBMS** (**ACID**): tiền + đơn hàng tuyệt đối không được sai.
> - (b) → **Redis** (**session**): tạm thời, cần nhanh, mất cũng không chết người.
> - (c) → **Graph** hoặc **precompute**: bản chất là duyệt quan hệ "ai mua gì cùng gì".
> - ❌ Trượt nếu lý do là "vì nhanh hơn" mà không bám **access pattern**.

### BT2 — User ↔ Comment trong MongoDB

Một user có thể có **hàng nghìn comment**. Trong **MongoDB**, bạn chọn **embed** hay **reference**?

> ✅ **Tiêu chí Đạt:**
> - Chọn **reference**, vì comment là **unbounded** (không có trần).
> - Nêu được **rủi ro phình document** nếu embed → có thể vỡ giới hạn **16MB/doc**.
> - ❌ Trượt nếu chọn embed mà không cảnh báo rủi ro phình.

---

## ⑩ Đọc thêm

- **PostgreSQL JSON types** — `postgresql.org/docs/current/datatype-json.html` (hiểu sâu **JSONB** + **GIN index**)
- **MongoDB Transactions & Data Modeling** — `mongodb.com/docs/manual/core/transactions/` (về **multi-doc ACID** + **embed/reference**)
- **Kleppmann, *Designing Data-Intensive Applications*, Ch.2–3** — kinh thánh về mô hình hóa dữ liệu

> 🎯 **Một câu mang về:** Đừng bao giờ trả lời "DB nào tốt hơn". Hãy trả lời "với **access pattern** này, **consistency** này, mô hình này — đây là DB hợp nhất, và đây là con số chứng minh."
