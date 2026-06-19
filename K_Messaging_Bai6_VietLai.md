# 🎯 BÀI 6 — Dual-write, Transactional Outbox & CDC

**Phủ:** K-OBX-001 → 008

> 💡 **Mục tiêu một câu:** Bài này **vá cái bẫy đã gài ở Bài 1** — dòng `insert()` rồi `publish()`. Đó là **dual-write problem**, và **Outbox / CDC** chính là lời giải chuẩn để *"publish event tin cậy"* mà không mất, không tạo event ma.

---

## ① Mục tiêu & vị trí trong mạch

Còn nhớ ở **Bài 1**, ta cố ý viết:
```javascript
await db.orders.insert(...);        // ghi DB
await broker.publish('OrderPlaced', ...);  // rồi publish — "code chạy được nhưng SAI kiến trúc"
```

Đó chính là **dual-write problem** mà bài này sẽ vá. **Outbox / CDC** là lời giải chuẩn.

> 🎯 **Chốt vị trí:** đây là **baseline bắt buộc** cho **Saga (Bài 7)**, và là câu *"cũ → hiện đại"* kinh điển (**2PC** vs **Saga + Outbox**). Cross-ref **K-DELIV-008** (**effectively-once** — Outbox là *mảnh "không mất"* trong chuỗi đó).

---

## ② Giảng cơ bản → nâng cao

### (a) Trực giác — dual-write 📘 K-OBX-001

> ✍️ **Ẩn dụ:** bạn cần *vừa* ghi vào **sổ cái** (DB) *vừa* bỏ một lá thư thông báo vào **hòm thư bưu điện** (broker). Nhưng hai hành động này **không có công tắc chung** — không có cách nào đảm bảo *"hoặc cả hai cùng xong, hoặc cả hai cùng không"*. Bạn ghi sổ xong, vừa định bỏ thư thì *ngã ra bất tỉnh* → sổ có, thư không.

**Dual-write** = bạn phải ghi **2 hệ thống** (**DB + broker**) nhưng chúng **không nằm trong một transaction**. Bốn kịch bản:

```
                       publish OK            publish FAIL
                   ┌────────────────────┬────────────────────┐
   DB commit OK    │ ✅ đúng             │ ❌ MẤT EVENT        │
                   │ (cả hai cùng xong) │ downstream KHÔNG    │
                   │                    │ biết order tồn tại  │
                   ├────────────────────┼────────────────────┤
   DB rollback     │ ❌ EVENT MA         │ ✅ nhất quán        │
                   │ downstream xử lý    │ ("chưa làm gì cả") │
                   │ order KHÔNG tồn tại │                    │
                   └────────────────────┴────────────────────┘
```

| Kịch bản | Kết quả | Vấn đề |
|---|---|---|
| DB commit **OK** + publish **OK** | ✅ | đúng |
| DB commit **OK** + publish **fail** | ❌ **mất event** | downstream không biết order tồn tại |
| DB **rollback** + publish **OK** | ❌ **event ma** (ghost event) | downstream xử lý order *không tồn tại* |
| Cả hai **fail** | ✅ | nhất quán ở trạng thái "chưa làm gì" |

> 🚨 **Vì sao `try/catch` KHÔNG cứu được?** Vì **không có atomicity** giữa 2 nguồn: **publish có thể thành công nhưng app crash *trước khi biết* kết quả**. Bạn không thể "undo" một message đã rời broker. → **Cần một pattern**, không phải xử lý lỗi khéo hơn.

---

### (b) Transactional Outbox 📘 K-OBX-002

> 🎯 **Ý tưởng chìa khoá:** **chuyển "2 hệ thống" thành "1 transaction DB".**

Ghi **business data** *và* ghi event vào **bảng `outbox`** trong **cùng một ACID transaction** của DB → **atomic**. Sau đó một tiến trình **bất đồng bộ** đọc bảng outbox rồi **publish** lên broker (đánh dấu *đã gửi*).

```
Transactional Outbox:
  ┌─────── 1 ACID transaction của DB ───────┐
  │  INSERT orders(...)          ✅          │
  │  INSERT outbox(OrderPlaced)  ✅          │  ← event sống/chết CÙNG order
  │  COMMIT                                  │
  └─────────────────────────────────────────┘
                     │
                     ▼ (bất đồng bộ, sau đó)
         publisher đọc outbox → publish broker → đánh dấu published
```

> 💡 **Event chỉ tồn tại nếu DB commit** → **loại bỏ kịch bản 2 và 3** (mất event & event ma) ở trên. Đây là *chìa khoá*.

---

### (c) Đẩy outbox ra broker — 2 cách 📘 K-OBX-003, 004

| Tiêu chí | **Polling publisher** | **CDC (Change Data Capture)** |
|---|---|---|
| Cách làm | job **quét bảng outbox** theo chu kỳ, publish, đánh dấu **published** | đọc **transaction log** của DB (**WAL/binlog**) bằng **Debezium** |
| Hạ tầng | **đơn giản**, không thêm gì | thêm **Kafka Connect / Debezium Server** |
| Độ trễ | có **độ trễ** (chu kỳ poll) | **realtime** |
| Tải DB | thêm tải (SELECT lặp) | **ít tải hơn** |
| Bắt được ghi từ ngoài app? | ❌ chỉ thấy cái app ghi vào outbox | ✅ **mọi thay đổi**, kể cả update ngoài app |

> 🔎 *verify:* **Debezium 3.4.x** hỗ trợ **Kafka 4.x**; có **Debezium Server/Engine** chạy *không cần* Kafka Connect; **EOS** cho core connector từ **3.3**.

**Vì sao CDC > polling DB để đồng bộ 📘 K-OBX-004:**

> 📼 **Ví dụ trực giác:** **polling** = cứ mỗi 5 giây *mở tủ ra xem có gì mới không* (mệt, có thể bỏ sót cái vừa-thêm-vừa-xoá). **CDC** = *đọc thẳng cuốn nhật ký giao dịch* mà DB *bắt buộc* ghi cho mọi thay đổi → **không miss cái nào**.

- đọc **commit log** = **không miss** thay đổi nào (kể cả **update ngoài app**),
- **realtime**, **ít tải hơn** `SELECT ... WHERE updated_at > ?` lặp lại,
- còn dùng để đồng bộ **DB → search / cache**.

> 🎯 **Câu "cũ → mới":** *"polling DB để đồng bộ"* → *"**CDC / event-driven**"*.

---

### (d) Đầu nhận — Inbox 📘 K-OBX-005, 006

> ⚖️ **Outbox và Inbox là hai nửa đối xứng:**

**Outbox** đảm bảo **at-least-once publish** → consumer *vẫn* nhận **trùng**. **Inbox pattern**: consumer ghi **`eventId`** nhận được vào **bảng inbox** trong **cùng transaction** với việc xử lý → **dedup bền** + atomic *"đã xử lý"*.

| Pattern | Giải bài toán | Vị trí |
|---|---|---|
| **Outbox** | **"không mất"** event | phía **producer** |
| **Inbox / dedup** | **"không trùng"** | phía **consumer** |

> 🎯 **Chốt:** **Outbox + Inbox = hai nửa của effectively-once** (gặp lại K-DELIV-008).

---

### (e) Vận hành outbox 📘 K-OBX-008

| Vấn đề vận hành | Cách xử lý |
|---|---|
| Bảng outbox **phình** | **dọn** (archive/delete row đã **published**) |
| publish có thể **trùng** | consumer **dedup** (Inbox) |
| giữ **ordering** | publish theo **thứ tự ghi** (**sequence / created_at**) + **key partition** |
| theo dõi sức khoẻ | **giám sát backlog outbox** y như giám sát **consumer lag** (Bài 4) |

---

### (f) 2PC vs Saga+Outbox 📘 K-OBX-007 — câu kinh điển

> 🤝 **Ví dụ trực giác — 2PC giống cưới tập thể:** một người điều phối (**coordinator**) bắt *tất cả* cô dâu chú rể cùng nói "tôi đồng ý" *đúng một khoảnh khắc*. Nếu người điều phối ngất xỉu giữa chừng → tất cả **đứng hình chờ**, không ai dám đi.

| Tiêu chí | **2PC (distributed transaction)** | **Saga + Outbox/CDC** |
|---|---|---|
| Cơ chế | **coordinator khoá** tất cả participant, **prepare → commit** | chuỗi **local tx + event**, **eventual** |
| Khi coordinator chết | participant **kẹt (blocking)** | **không có coordinator** khoá toàn cục |
| Availability/throughput | **giảm** (khoá, đồng bộ) | **cao hơn** (không khoá toàn cục) |
| Microservices | **ít dùng** | **chuẩn hiện đại** |

> 🎯 **Vì sao 2PC ít dùng:** **blocking** + **kém chịu lỗi** + **giảm availability**. Microservices thiên về **eventual** qua **Saga + Outbox/CDC**.

---

## ③ ⚠️ Cũ → Mới

| Cái cũ | Nay | Vì sao |
|---|---|---|
| **`insert(); publish();`** trong cùng hàm | **Transactional Outbox** (ghi outbox cùng tx) | Xoá **dual-write** (mất/ma event) |
| **Polling DB** để đồng bộ service/search/cache | **CDC (Debezium)** đọc **WAL/binlog** | Realtime, không miss, ít tải (🔎 verify) |
| **2PC / XA** cho consistency xuyên service | **Saga + Outbox** (eventual) | 2PC blocking, kém chịu lỗi |
| **Chỉ Outbox là đủ** | **Outbox** (producer) + **Inbox/dedup** (consumer) | Outbox = không mất; cần thêm để không trùng |

---

## ④ Áp dụng thực tế + bigtech

**Use case:**
1. `placeOrder` ghi **`orders`** + ghi **`outbox(OrderPlaced)`** trong **1 tx**;
2. **Debezium** đọc **WAL** bảng outbox → publish lên **Kafka topic `orders`**;
3. **InventorySvc** consume + **dedup** qua **Inbox**.

```
placeOrder ──1 tx──► [orders + outbox]
                          │ WAL
                          ▼
                     Debezium (CDC) ──► Kafka topic "orders"
                          │
                          ▼
                  InventorySvc ──dedup(Inbox)──► trừ kho (1 lần duy nhất ✅)
```

**Bigtech:** **Outbox + Debezium CDC** là pattern phổ biến để *"publish event tin cậy"* mà **không cần 2PC**; nhiều hệ dùng **Debezium** đồng bộ **DB → Elasticsearch/cache** realtime. (pattern; 🔎 *verify Debezium docs*.)

---

## ⑤ Code thực hành + cấu hình

**Outbox atomic (Postgres):**

```javascript
// placeOrder.js — ghi order + outbox trong CÙNG transaction (atomic)
async function placeOrder(input) {
  const client = await pool.connect();
  try {
    await client.query('BEGIN');

    const { rows } = await client.query(
      'INSERT INTO orders(id, status, payload) VALUES($1,$2,$3) RETURNING *',
      [input.id, 'PENDING', input]
    );

    await client.query(
      `INSERT INTO outbox(id, aggregate_type, aggregate_id, type, payload)
       VALUES($1,'order',$2,'OrderPlaced',$3)`,
      [crypto.randomUUID(), input.id, JSON.stringify(rows[0])]
    ); // ⭐ event sống/chết CÙNG order -> KHÔNG bao giờ "DB ok mà mất event"

    await client.query('COMMIT'); // orders + outbox commit CÙNG LÚC (atomic)
    return rows[0];
  } catch (e) {
    await client.query('ROLLBACK'); // cả hai cùng huỷ -> không có event ma
    throw e;
  } finally {
    client.release();
  }
}
// ⚠️ KHÔNG publish ở đây. Debezium (CDC) hoặc polling job sẽ đẩy outbox -> broker.
```

```sql
CREATE TABLE outbox (
  id UUID PRIMARY KEY,
  aggregate_type TEXT, aggregate_id TEXT, type TEXT,
  payload JSONB, created_at TIMESTAMPTZ DEFAULT now(),
  published BOOLEAN DEFAULT false        -- cho polling; với CDC có thể bỏ
);
```

```json
// Debezium connector (Outbox Event Router) — verify tên/đời connector
{
  "name": "outbox-connector",
  "config": {
    "connector.class": "io.debezium.connector.postgresql.PostgresConnector",
    "table.include.list": "public.outbox",
    "transforms": "outbox",
    "transforms.outbox.type": "io.debezium.transforms.outbox.EventRouter"
  }
}
```

```bash
# .env (KHÔNG hardcode): DATABASE_URL, KAFKA_BROKERS, CONNECT_URL
# Postgres cần wal_level=logical + replication slot cho Debezium // verify
```

> 🎯 **Điểm vàng:** **không có dòng `publish` nào** trong `placeOrder`. Việc publish được *tách hẳn* sang CDC/polling — đó chính là cách bạn *xoá sổ* dual-write.

---

## ⑥ Keywords

**🧠 Cần HIỂU:**
- **dual-write** 4 kịch bản.
- **Outbox** = **1 transaction DB**.
- **CDC** đọc **WAL**.
- **Inbox** đối xứng.
- *vì sao* **2PC** bị né.

**📌 Cần THUỘC:**
- **Outbox** (ghi business + event cùng tx).
- **polling** vs **CDC**.
- **Debezium** đọc **WAL/binlog**.
- **Inbox dedup**.
- *"polling → CDC"*, *"2PC → Saga + Outbox"*.

**🛠️ Cần LÀM ĐƯỢC:**
- viết **`placeOrder` atomic** với outbox.
- cấu hình **Debezium EventRouter** (🔎 verify).
- **giám sát outbox backlog**.

---

## ⑦ Mental model

> 🧠 **"Đừng ghi 2 nơi. Ghi business + event vào outbox trong MỘT transaction DB, để CDC (Debezium) đẩy đi.**
> **Outbox = không mất, Inbox/dedup = không trùng.**
> **2PC chết, Saga + Outbox sống."**

---

## ⑧ Phỏng vấn & thách đố

**(khó)** **Dual-write** là gì, vì sao **`try/catch` không cứu**?
→ ghi 2 hệ thống không atomic; publish có thể thành công nhưng app crash trước khi biết → không undo được message đã rời broker.

**(rất khó)** **Outbox** đảm bảo **atomicity** thế nào?
→ ghi business + event vào outbox trong cùng 1 ACID transaction; event chỉ tồn tại nếu DB commit.

**(khó)** **Polling** vs **CDC** — đánh đổi? Vì sao **CDC** tốt hơn polling DB?
→ polling đơn giản nhưng trễ + tải DB; CDC realtime, không miss, ít tải, bắt cả ghi ngoài app.

**(khó)** **Inbox** bổ trợ **Outbox** ra sao?
→ Outbox lo "không mất" (producer), Inbox lo "không trùng" (consumer) = hai nửa effectively-once.

**(rất khó)** **2PC** vs **Saga + Outbox** — vì sao 2PC ít dùng ở microservices?
→ 2PC blocking + kém chịu lỗi + giảm availability; microservices chọn eventual qua Saga + Outbox.

> 🧩 **Thách đố:** *"Dev nói: 'tôi publish event trong cùng `try` với DB commit, có rollback rồi, an toàn'. Phản biện."*
> → **publish lên broker KHÔNG nằm trong DB transaction**. Nếu **DB commit xong rồi app crash trước publish** → **mất event**; nếu **publish xong rồi DB rollback** → **event ma**. **Rollback DB không undo được message đã rời broker.** Phải dùng **Outbox**.

---

## ⑨ Bài tập + tiêu chí

**Đề:** Chuyển `placeOrder` (đang `insert + publish`) sang **Outbox** + chọn **polling** hoặc **CDC**, nêu cách **dedup** phía consumer.

> ✅ **Đạt khi:**
> - **business + outbox** cùng **1 tx**.
> - mô tả đẩy ra broker (**polling job** đánh dấu `published` / **Debezium EventRouter**).
> - consumer **dedup theo eventId** (**Inbox**).
> - nêu cách **dọn outbox** + **giám sát backlog**.

---

## ⑩ Đọc thêm

- **`microservices.io`**: **Transactional Outbox**, **Polling Publisher**, **Transaction Log Tailing**.
- **`debezium.io`** (**Outbox Event Router**, **CDC**). (🔎 *verify đời*.)

> 🎯 **Một câu mang về:** *Mỗi khi bạn định viết `insert()` rồi `publish()` cạnh nhau, hãy dừng lại — đó là dual-write. Ghi event vào **outbox** cùng transaction, rồi để **CDC** lo phần phát đi.*
