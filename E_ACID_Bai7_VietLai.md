# 🎯 BÀI 7 — ACID & Transaction

**Phủ:** E-ACID-001 → 008

> 💡 Đây là **nền của tính đúng đắn (correctness)** — tách *hẳn* khỏi chuyện tốc độ. Bài 4–6 lo "nhanh"; bài này lo "đúng". Hiểu **transaction** để **không bao giờ làm hỏng dữ liệu tiền/tồn kho** — loại lỗi không được phép xảy ra dù chỉ một lần.

---

## ① Mục tiêu & vị trí

Bài này về **transaction** và **ACID** — bộ đảm bảo dữ liệu *đúng* bất kể có lỗi/crash.

> 🔎 Điểm cốt lõi: **correctness tách hẳn khỏi performance**. Một hệ thống nhanh mà trừ tiền sai thì vô giá trị. Bài này không bàn tốc độ — chỉ bàn *làm sao dữ liệu luôn đúng*.

Vị trí: là **nền trực tiếp cho Bài 8** (isolation / MVCC).

> 🎯 **Chốt định vị:** Khi bạn động tới *tiền* hoặc *tồn kho*, transaction không phải lựa chọn — nó là *bắt buộc*. Sai một transaction = mất tiền thật hoặc bán hàng không có.

---

## ② Giảng cơ bản → nâng cao

### (a) ACID  `E-ACID-001`

> **Ẩn dụ:** Chuyển tiền **A → B** giống *trao tay phong bì*. Hoặc tiền rời tay A **và** tới tay B (trọn vẹn), hoặc không gì xảy ra cả. Tuyệt đối không có cảnh "A mất tiền mà B chưa nhận".

Lấy ví dụ chuyển 100 từ A sang B để giải thích **4 chữ ACID**:

| Chữ | Tên | Nghĩa | Ví dụ chuyển tiền A→B |
|-----|-----|-------|------------------------|
| **A** | **Atomicity** | **all-or-nothing** | trừ A và cộng B **cùng xảy ra** hoặc **cùng không** |
| **C** | **Consistency** | sau tx, mọi **invariant/constraint** vẫn hợp lệ | **tổng tiền không tự sinh** ra hay biến mất |
| **I** | **Isolation** | tx chạy **như thể tuần tự** | (chi tiết **Bài 8**) |
| **D** | **Durability** | đã **commit** thì **không mất dù crash** | điện cúp ngay sau commit → tiền vẫn đã chuyển |

**Vì sao Atomicity quan trọng nhất với tiền?** Vì nếu thiếu nó, một crash giữa hai lệnh (đã trừ A, chưa cộng B) sẽ làm *bốc hơi* 100 đồng. Atomicity đảm bảo: nếu không hoàn tất, mọi thứ **rollback** về như chưa từng bắt đầu.

```text
Atomicity bảo vệ khi crash giữa chừng:
  T1: UPDATE account A: balance -= 100   ✅ (A đã bị trừ)
       ↓
  T2: 💥 CRASH trước khi cộng cho B
       ↓
  Không có atomicity: ❌ A mất 100, B không nhận → bốc hơi tiền
  CÓ atomicity:       ✅ rollback → A được hoàn lại → như chưa xảy ra
```

---

### (b) Bẫy thuật ngữ — ACID-C vs CAP-C  `E-ACID-002`

> 🚨 Đây là **câu hỏi phỏng vấn ưa thích** vì hai chữ "C" *trông giống nhau nhưng nghĩa hoàn toàn khác*.

| | **ACID-Consistency** | **CAP-Consistency** |
|---|----------------------|---------------------|
| Nghĩa | **ràng buộc/invariant luôn đúng** | **mọi node thấy data mới nhất** (**linearizability**) |
| Phạm vi | trong một DB / transaction | giữa nhiều node phân tán |
| Ví dụ | "tổng tiền không đổi sau chuyển khoản" | "đọc ở node nào cũng ra giá trị vừa ghi" |

> ⚠️ **Đừng nhầm:** **ACID-C** nói về *quy tắc dữ liệu hợp lệ*; **CAP-C** nói về *các bản sao đồng bộ*. Lẫn lộn hai cái là cái bẫy kinh điển khi đi phỏng vấn.

---

### (c) Durability bằng WAL  `E-ACID-003`

> **Ẩn dụ:** Trước khi làm việc gì quan trọng, bạn **ghi vào nhật ký "tôi sắp làm X"** rồi mới làm. Nếu giữa chừng ngất xỉu, tỉnh dậy đọc nhật ký là biết phải làm tiếp/làm lại gì. **WAL** (**write-ahead log**) chính là cuốn nhật ký đó của DB.

**Durability** được đảm bảo bằng **WAL**: **ghi write-ahead log TRƯỚC** + **fsync** (ép xuống đĩa thật). **commit** = log đã *bền trên đĩa* → nếu **crash**, DB **replay log** để khôi phục.

```text
Durability qua WAL:
  T1: ghi thay đổi vào WAL (write-ahead log)
       ↓
  T2: fsync → ép WAL xuống đĩa vật lý (không chỉ nằm RAM)
       ↓
  T3: COMMIT trả về "thành công"   ✅ từ đây dữ liệu là BỀN
       ↓
  💥 nếu crash sau đó → khi khởi động lại, REPLAY WAL → khôi phục nguyên trạng
```

> 🔎 **Lưu ý sâu:** **WAL** cũng là **nền của replication** (xem **Bài 12**) — các bản sao đọc lại chính cuốn log này để đồng bộ.

---

### (d) Mép giới hạn

#### 🔍 Transaction nên NGẮN  `E-ACID-004`

> ⚠️ **long-running tx** (transaction kéo dài) là một trong những nguồn sự cố production khó chịu nhất.

Một tx dài sẽ:
- ❌ **giữ lock lâu** → **block tx khác** (chúng phải xếp hàng chờ).
- ❌ **giữ snapshot lâu** → gây **bloat** (**MVCC** không dọn được version cũ — xem **Bài 8/9**).
- ❌ **tăng deadlock** (càng giữ lock lâu càng dễ kẹt chéo).

> 🎯 **Khẩu quyết:** **Mở tx muộn, commit sớm.** Chỉ ôm vào transaction đúng phần *bắt buộc phải nguyên tử*.

#### 🔍 Savepoint  `E-ACID-005`

> **Ví dụ trực giác:** **SAVEPOINT** giống "checkpoint" trong game — lỡ bước sau hỏng thì quay về checkpoint, không phải chơi lại từ đầu.

**SAVEPOINT** cho phép **rollback một phần** trong một tx lớn.

> 🔎 **Lưu ý:** **Postgres** có **subtransaction** (qua savepoint) nhưng **không có "nested commit" độc lập** — savepoint không phải transaction con tự commit riêng được.

#### 🔍 Khi nào KHÔNG cần tx tường minh  `E-ACID-006`

> 💡 **Một câu lệnh đơn (single statement) đã atomic sẵn** nhờ **autocommit** — không cần `BEGIN/COMMIT`.

Chỉ cần **tx tường minh** khi **nhiều thao tác phải nguyên tử cùng nhau** (như chuyển tiền = 2 UPDATE). Bọc transaction quanh một câu lệnh đơn là thừa.

#### 🔍 Distributed tx  `E-ACID-007`

> **Ẩn dụ:** Phối hợp nhiều service như nhờ nhiều người ký một hợp đồng. **2PC** = bắt tất cả *cùng treo bút chờ nhau ký một lúc* (ai chậm thì cả nhóm kẹt). **Saga** = mỗi người ký phần mình, ai lỡ sai thì *làm động tác bù* để hủy.

Transaction qua **nhiều service/DB** → **tránh 2PC** (**two-phase commit** — **khóa nhiều** + **kém chịu lỗi**). Thay vào đó dùng **Saga** + **Outbox**, **chấp nhận eventual consistency** (liên hệ **mục K**).

| | **2PC** | **Saga + Outbox** |
|---|---------|-------------------|
| Cách hoạt động | mọi node khóa rồi commit đồng thời | mỗi bước commit cục bộ + bước bù khi lỗi |
| Khóa | ❌ giữ lock chờ tất cả | ✅ lock cục bộ ngắn |
| Chịu lỗi | ❌ một node chết → kẹt | ✅ bền hơn với lỗi từng phần |
| Nhất quán | mạnh tức thì | **eventual consistency** |

#### 🔍 Anti-pattern: gọi HTTP giữa tx  `E-ACID-008`

> 🚨 **CẤM:** mở tx rồi **gọi API ngoài** ở giữa → **giữ lock + connection chờ I/O mạng** → **timeout**, **bloat**.

```text
ANTI-PATTERN — gọi HTTP trong transaction:
  T1: BEGIN
  T2: UPDATE ... (đã giữ lock dòng)
       ↓
  T3: ❌ gọi payment gateway qua HTTP (chờ 2 giây mạng)
       → suốt 2s này: lock vẫn giữ + connection DB vẫn bận
       → tx khác chờ dài → có thể timeout → bloat tích lũy
  T4: COMMIT
  ✅ Sửa: tách side-effect RA NGOÀI tx (ghi outbox trong tx, gửi SAU commit)
```

> 🎯 **Chốt:** **Tách side-effect ra ngoài tx** — ghi **outbox** trong tx, để một worker gửi đi *sau commit*.

---

## ③ ⚠️ Cũ → Mới (kiến thức bị thay thế)

| Quan niệm CŨ ❌ | Hiện nay ✅ | Vì sao |
|----------------|------------|--------|
| **2PC** cho **distributed tx** | **Saga** + **Outbox**, **eventual consistency** | 2PC **khóa** + **kém chịu lỗi** ở microservices |
| Tx "to cho chắc", bọc cả request | Tx **ngắn**, **mở muộn commit sớm** | **long tx** gây **lock** + **bloat** + **deadlock** |
| Gọi **API ngoài trong tx** | **Outbox** / **sau commit** | Giữ **lock** chờ mạng = nguy hiểm |

---

## ④ Thực tế + bigtech

> 🔎 **Pattern ngành (verify):**
> - **Hệ tài chính:** chuyển tiền trong **một tx DB** (cùng DB) khi có thể; **cross-service** dùng **Saga** + **outbox** + **idempotency key** (khóa chống xử lý trùng).
> - **Outbox pattern:** **ghi event vào bảng cùng tx với dữ liệu**, rồi **một worker đọc bảng đó phát đi** → **không mất event**, **không cần 2PC**.

> 💡 Vì sao outbox thắng? Vì nó biến "ghi DB" và "phát event" thành *một thao tác nguyên tử* (cùng tx) — loại bỏ kẽ hở mất event mà không cần khóa phân tán.

---

## ⑤ Code + cấu hình

```sql
-- Chuyển tiền nguyên tử trong MỘT transaction
BEGIN;
  UPDATE account SET balance_cents = balance_cents - 100 WHERE id = 'A';  -- trừ A
  UPDATE account SET balance_cents = balance_cents + 100 WHERE id = 'B';  -- cộng B
  -- 🚨 KHÔNG gọi HTTP/email ở đây (anti-pattern E-ACID-008). Ghi outbox thay thế:
  INSERT INTO outbox(topic, payload) VALUES ('transfer.done', '{"from":"A","to":"B"}');
COMMIT;
-- → worker RIÊNG đọc outbox → gửi event → đánh dấu đã gửi (idempotent: gửi lại không sao)

-- Savepoint: rollback MỘT PHẦN trong tx lớn
BEGIN;
  INSERT INTO log(msg) VALUES ('step1');
  SAVEPOINT sp1;                         -- đặt checkpoint
  INSERT INTO log(msg) VALUES ('maybe-fail');
  ROLLBACK TO sp1;                       -- hủy 'maybe-fail', GIỮ lại step1
COMMIT;
```

> 🚨 **AI hay sai ở đây:** AI hay **gọi API/email/payment ngay trong transaction** (giữ lock chờ mạng), hoặc **bọc transaction quanh một câu lệnh đơn** (thừa), hoặc dùng **2PC** cho cross-service thay vì **Saga + Outbox**.

---

## ⑥ Keywords

> 🧠 **HIỂU** (nắm bản chất):
> **ACID-C vs CAP-C** · vì sao **long tx** nguy hiểm · **WAL**/**redo** · **Saga**/**Outbox** vs **2PC**.

> 📌 **THUỘC** (nhớ chính xác):
> **A/C/I/D** · **single statement = autocommit** · **commit = WAL bền**.

> 🛠️ **LÀM ĐƯỢC** (kỹ năng tay):
> Viết **tx chuyển tiền** đúng · áp dụng **outbox** thay vì gọi API trong tx.

---

## ⑦ Mental model

> 🎯 **NGUYÊN TẮC VÀNG CỦA BÀI:**
> *"**Transaction = đơn vị all-or-nothing, phải NGẮN**. Đừng **nhốt I/O mạng trong tx**. Cross-service thì **Saga + Outbox**, đừng **2PC**."*

Cách dùng: trước khi `BEGIN`, hỏi *"đúng những thao tác nào phải nguyên tử cùng nhau?"* — chỉ ôm đúng chúng. Bất kỳ lời gọi mạng/email nào → đẩy ra **outbox**, gửi sau commit.

---

## ⑧ Phỏng vấn & thách đố

| Mức | Câu hỏi | Trả lời cốt lõi |
|-----|---------|-----------------|
| **(dễ)** | **ACID** + ví dụ chuyển tiền | 4 chữ + **all-or-nothing** |
| **(TB)** | **ACID-C** khác **CAP-C**? | **invariant** vs **node thấy data mới** |
| **(TB)** | **Durability** đảm bảo bằng gì? | **WAL** + **fsync** |
| **(khá)** | Vì sao **long tx** nguy hiểm? | **lock** + **bloat** + **deadlock** |
| **(khá)** | **Distributed tx** — tránh 2PC, dùng gì? | **Saga** + **Outbox** |

> 🧩 **Thách đố tình huống:**
> *Gọi HTTP giữa tx — vì sao là anti-pattern?*
>
> Trả lời chuẩn: vì **giữ lock/connection chờ mạng** → block tx khác, timeout, bloat.
> ⚠️ **Bẫy phải vạch ra:** **side-effect (email/API/payment) phải ra ngoài tx** — dùng outbox / gửi sau commit.

---

## ⑨ Bài tập + tiêu chí tự chấm

### BT1 — Tx trừ tồn kho + tạo đơn, không gọi email trong tx

> ✅ **Tiêu chí Đạt:**
> - **2 thao tác (trừ tồn kho + tạo đơn) trong CÙNG một tx**.
> - Email đẩy ra **outbox**, gửi sau commit (không gọi trong tx).
> - ❌ Trượt nếu gọi gửi email ngay trong transaction.

### BT2 — Service mở tx, gọi payment gateway (2s), rồi commit

Chỉ ra vấn đề + sửa.

> ✅ **Tiêu chí Đạt:**
> - Nhận ra **giữ lock suốt 2 giây** chờ payment gateway → block + nguy cơ timeout/bloat.
> - Sửa: **tách call ra ngoài tx** / dùng **Saga** + **idempotency key**.
> - ❌ Trượt nếu chỉ "tăng timeout" mà không tách call ra ngoài tx.

---

## ⑩ Đọc thêm

- **PostgreSQL — Transactions & WAL** — `postgresql.org/docs/current/tutorial-transactions.html`
- **microservices.io** — **Saga** & **Transactional Outbox** patterns

> 🎯 **Một câu mang về:** Transaction là cái ôm "all-or-nothing" — nhưng cái ôm đó phải *ngắn* và *không bao giờ ôm theo một cuộc gọi mạng*. Tiền và tồn kho không tha thứ cho sai sót.
