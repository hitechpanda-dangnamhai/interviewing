# 🎯 BÀI 8 — Isolation · MVCC · Locking · Deadlock

**Phủ:** E-ISO-001 → 009

> 💡 Đây là phần **concurrency** — quyết định *điều gì xảy ra khi nhiều transaction đụng cùng một dữ liệu cùng lúc*. Bài 7 lo "một tx phải đúng"; bài này lo "nhiều tx chạy song song vẫn đúng". Đây là một trong những phần **phân biệt senior thật** với người chỉ biết viết query.

---

## ① Mục tiêu & vị trí

Bài này về **concurrency** (đồng thời): nhiều **transaction** chạy song song trên cùng dữ liệu thì sao?

Nó giới thiệu **MVCC** — kiến thức **bắt buộc** để hiểu **VACUUM/bloat** ở **Bài 9**.

> 🎯 **Chốt định vị:** Bug concurrency là loại bug *đáng sợ nhất*: nó không xuất hiện khi test một mình, chỉ nổ khi có *hai người dùng cùng lúc* — và thường nổ đúng lúc đông khách nhất (bán vé, flash sale).

---

## ② Giảng cơ bản → nâng cao

### (a) 4 isolation level chuẩn SQL  `E-ISO-001`

> **Ẩn dụ:** **isolation level** như độ dày bức tường giữa các phòng làm việc. Tường mỏng → nghe lỏm được nhau (nhanh nhưng dễ nhiễu). Tường dày → cách âm tuyệt đối (an toàn nhưng tốn kém).

Bốn mức, **chặt dần**:

```
Read Uncommitted  <  Read Committed  <  Repeatable Read  <  Serializable
    (lỏng nhất)                                              (chặt nhất)
```

> 💡 Càng chặt → càng ít **anomaly** nhưng càng tốn (giảm throughput, tăng abort/retry). Đây là một **đánh đổi**, không phải "càng chặt càng tốt".

---

### (b) 3 anomaly  `E-ISO-002`

> **Ví dụ trực giác:** Ba anomaly là ba kiểu "nhiễu" khi hai người đọc/ghi cùng lúc — như hai người cùng sửa một tài liệu Google Docs mà thấy phiên bản chỏi nhau.

| Anomaly | Là gì | Bị chặn từ mức |
|---------|-------|----------------|
| **Dirty read** | đọc data của tx **chưa commit** | **Read Committed** |
| **Non-repeatable read** | đọc **cùng row** 2 lần ra **khác nhau** (tx khác đã commit update) | **Repeatable Read** |
| **Phantom read** | đọc **cùng điều kiện** 2 lần ra **số dòng khác** | **Serializable** |

Timeline từng anomaly:

```text
DIRTY READ (đọc data chưa commit):
  T1 (tx A): UPDATE balance = 0    (CHƯA commit)
  T2 (tx B): SELECT balance → đọc 0   ❌ đọc bản chưa chắc chắn
  T3 (tx A): ROLLBACK              → số 0 đó CHƯA TỪNG có thật
       → B đã hành động trên dữ liệu "ma"

NON-REPEATABLE READ (cùng row, 2 lần khác nhau):
  T1 (tx B): SELECT price WHERE id=1 → 100
  T2 (tx A): UPDATE price = 200 WHERE id=1; COMMIT
  T3 (tx B): SELECT price WHERE id=1 → 200   ❌ cùng tx B mà 2 lần đọc khác nhau

PHANTOM READ (cùng điều kiện, số dòng khác):
  T1 (tx B): SELECT count(*) WHERE status='open' → 5
  T2 (tx A): INSERT 1 đơn status='open'; COMMIT
  T3 (tx B): SELECT count(*) WHERE status='open' → 6   ❌ "bóng ma" dòng mới xuất hiện
```

---

### (c) Postgres khác chuẩn  `E-ISO-003`

> 🔍 **Lưu ý sâu — đây là chỗ rất hay bị hỏi:**
> - **Postgres** **default = Read Committed**.
> - **"Repeatable Read" của Postgres** thực ra là **snapshot isolation** — **mạnh hơn mức tối thiểu chuẩn**, **chặn được phần lớn phantom** (chuẩn SQL chỉ yêu cầu RR chặn non-repeatable, không bắt buộc chặn phantom).

> ⚠️ Đừng giả định "Repeatable Read ở mọi DB như nhau" — Postgres làm chặt hơn chuẩn.

---

### (d) MVCC — trái tim Postgres  `E-ISO-004`

> **Ẩn dụ:** **MVCC** (**Multi-Version Concurrency Control**) giống Google Docs giữ **lịch sử phiên bản**. Người đang đọc thấy *bản chụp tại thời điểm họ mở file*, người khác sửa thì tạo *phiên bản mới* — không ai phải chờ ai. Đọc và ghi diễn ra song song mà không giẫm chân nhau.

Cơ chế: mỗi **row có nhiều version**, đánh dấu bằng **`xmin`/`xmax`** (tx nào *tạo* / tx nào *xóa* version đó). **Reader** thấy **snapshot** phù hợp với *thời điểm tx bắt đầu* → **không cần read lock**.

> 🎯 **Khẩu quyết MVCC:** **"reader không block writer, writer không block reader."** Chỉ **writer–writer trên cùng một row** mới đụng nhau.

```text
MVCC: reader và writer KHÔNG chặn nhau:
  T1 (reader B): bắt đầu tx → chụp snapshot (thấy version cũ: price=100)
  T2 (writer A): UPDATE price=200 → TẠO version MỚI, không xóa version cũ ngay
       ↓
  T3 (reader B): vẫn đọc snapshot của mình → vẫn thấy 100   ✅ không phải chờ A
       → A và B chạy song song, không ai block ai
```

> 🔎 **Hệ quả quan trọng:** version cũ **tích tụ** lại (không xóa ngay) → cần **VACUUM** dọn dẹp (xem **Bài 9**). Đây chính là cây cầu nối sang bài sau.

---

### (e) Mép giới hạn

#### 🔍 Serializable (SSI)  `E-ISO-005`

> **Ví dụ trực giác:** **SSI** kiểu **optimistic** = "cứ làm đi, lát kiểm; nếu phát hiện hai tx tạo thành vòng phụ thuộc mâu thuẫn thì hủy một cái". Giống cho phép mọi người cùng đặt chỗ, cuối cùng nếu trùng thì hủy một đơn.

**Serializable** trong Postgres dùng **SSI** (Serializable Snapshot Isolation) — **optimistic**: **phát hiện vòng phụ thuộc** giữa các tx và **abort một cái** với lỗi **`serialization_failure`** → **ứng dụng phải retry**.

#### 🔍 SELECT ... FOR UPDATE  `E-ISO-006`

**`SELECT ... FOR UPDATE`** = **khóa các row được chọn** để cập nhật, **chặn tx khác sửa/khóa chúng** → dùng cho **read-modify-write**, **tránh lost update**.

#### 🔍 Lost update  `E-ISO-007`

> **Ẩn dụ:** Hai nhân viên cùng nhìn kho thấy "còn 10", cùng bán 1 cái, cùng ghi "còn 9". Thực tế bán 2 nhưng sổ ghi mất 1 lần trừ → **lost update**.

**lost update** = hai tx **đọc cùng giá trị rồi ghi đè nhau**.

```text
LOST UPDATE (hai request cùng trừ tồn kho):
  T1 (req 1): SELECT stock → 10
  T2 (req 2): SELECT stock → 10      (cả hai cùng thấy 10)
  T3 (req 1): UPDATE stock = 9       (10 - 1)
  T4 (req 2): UPDATE stock = 9       ❌ ghi đè! (đáng ra phải là 8)
       → bán 2 cái nhưng kho chỉ giảm 1 → lệch tồn kho
```

Ba cách chặn:

| Cách | Kiểu | Cách làm |
|------|------|----------|
| **FOR UPDATE** | **pessimistic** (bi quan) | khóa row trước khi sửa |
| **version / CAS** | **optimistic** (lạc quan) | so version, ghi chỉ khi version khớp; lệch → retry |
| **atomic update** | đơn giản nhất | `UPDATE ... SET stock = stock - 1` trong một câu |

#### 🔍 Deadlock  `E-ISO-008`

> **Ẩn dụ:** Hai người qua cửa hẹp, mỗi người nhường nửa và *cùng chờ người kia đi trước* → kẹt vĩnh viễn. Đó là **deadlock**: hai tx **khóa chéo theo thứ tự ngược nhau**.

```text
DEADLOCK (khóa chéo ngược thứ tự):
  T1 (tx A): khóa row A ✅ ... rồi xin khóa row B (B đang bị tx B giữ) → chờ
  T2 (tx B): khóa row B ✅ ... rồi xin khóa row A (A đang bị tx A giữ) → chờ
       ↓
  ❌ A chờ B, B chờ A → kẹt vòng tròn
  ✅ DB tự DETECT → abort một "nạn nhân", cái còn lại đi tiếp
```

> 🎯 **Phòng deadlock:** **khóa theo thứ tự nhất quán** (luôn A trước B) · **tx ngắn** · **thu hẹp phạm vi lock**.

---

## ③ ⚠️ Cũ → Mới (kiến thức bị thay thế)

| Quan niệm CŨ ❌ | Hiện nay ✅ | Vì sao |
|----------------|------------|--------|
| "Đọc-ghi luôn **block** nhau" | **MVCC**: **reader không block writer** | Postgres giữ **nhiều version** |
| Nâng **Serializable** "cho an toàn" toàn cục | Giải đúng **điểm nóng** ở **RC** + **locking/atomic** | Serializable tốn **abort/retry**, **giảm throughput** `E-ISO-009` |
| **SELECT rồi UPDATE** (đọc-rồi-ghi) | **`UPDATE ... SET x = x - 1`** atomic / **FOR UPDATE** | Tránh **lost update** |

---

## ④ Thực tế + bigtech

> 🔎 **Pattern ngành (verify):**
> - **Điểm nóng** (trừ tồn kho, ví điểm) thường giải bằng **atomic update** hoặc **FOR UPDATE cục bộ** thay vì nâng isolation toàn DB.
> - Hệ cần **đảm bảo mạnh** (đặt vé, ngân hàng) dùng **Serializable** có **retry loop**.

> 🧩 **TL cân nhắc** `E-ISO-009`: chi phí **abort/retry** + **contention** (tranh chấp) — **KHÔNG "Serializable cho lành"**. Nâng isolation toàn cục là đem búa tạ đập ruồi: tốn throughput cho mọi query chỉ để xử một điểm nóng.

---

## ⑤ Code + cấu hình

```sql
-- LOST UPDATE (XẤU): đọc rồi ghi → hai request ghi đè nhau
SELECT stock FROM product WHERE id = 1;        -- ❌ cả hai cùng đọc 10
UPDATE product SET stock = 9 WHERE id = 1;      -- cả hai cùng ghi 9 (mất 1 lần trừ)

-- Cách 1 — atomic update (đơn giản nhất cho counter)
-- stock - 1 tính NGAY trong DB + điều kiện stock > 0 chống bán âm
UPDATE product SET stock = stock - 1 WHERE id = 1 AND stock > 0;

-- Cách 2 — pessimistic lock (FOR UPDATE khóa row)
BEGIN;
  SELECT stock FROM product WHERE id = 1 FOR UPDATE;  -- khóa row, tx khác phải chờ
  UPDATE product SET stock = stock - 1 WHERE id = 1;
COMMIT;

-- Cách 3 — optimistic (version/CAS): ghi chỉ khi version còn khớp
-- nếu affected rows = 0 → ai đó đã sửa trước → app phải RETRY
UPDATE product SET stock = stock - 1, version = version + 1
WHERE id = 1 AND version = 42;

-- Serializable: phải bắt serialization_failure và RETRY ở tầng app
SET TRANSACTION ISOLATION LEVEL SERIALIZABLE;
```

> 🚨 **AI hay sai ở đây:** AI hay sinh pattern **SELECT-rồi-UPDATE** (kinh điển gây **lost update**), hoặc đặt **`SET ISOLATION LEVEL SERIALIZABLE`** mà **quên retry loop** (gặp `serialization_failure` là vỡ), hoặc nâng Serializable toàn cục "cho lành" làm tụt throughput.

---

## ⑥ Keywords

> 🧠 **HIỂU** (nắm bản chất):
> **MVCC** (**xmin**/**xmax**, **snapshot**) · **3 anomaly** · **SSI optimistic** · **lost update** · **deadlock detection**.

> 📌 **THUỘC** (nhớ chính xác):
> **thứ tự 4 level** · **Postgres default = Read Committed** · **RR của Postgres = snapshot isolation**.

> 🛠️ **LÀM ĐƯỢC** (kỹ năng tay):
> Chặn **lost update** bằng **3 cách** · viết **retry loop** cho **Serializable** · phòng **deadlock** bằng **thứ tự lock**.

---

## ⑦ Mental model

> 🎯 **NGUYÊN TẮC VÀNG CỦA BÀI:**
> *"**MVCC = reader đọc bản snapshot, không block writer**. **Điểm nóng** thì **atomic/FOR UPDATE cục bộ**; đừng **nâng Serializable toàn cục cho lành**."*

Cách dùng: gặp tranh chấp ghi, đừng vội nâng isolation. Hỏi *"điểm nóng là dòng/bảng nào?"* rồi giải đúng chỗ đó bằng **atomic update** hoặc **FOR UPDATE**, giữ phần còn lại ở **Read Committed**.

---

## ⑧ Phỏng vấn & thách đố

| Mức | Câu hỏi | Trả lời cốt lõi |
|-----|---------|-----------------|
| **(dễ)** | 4 **isolation level** theo độ chặt? | **RU < RC < RR < Serializable** |
| **(TB)** | **3 anomaly** + level chặn? | **dirty** / **non-repeatable** / **phantom** |
| **(khá)** | Default Postgres + RR Postgres khác chuẩn? | **RC**; **RR = snapshot**, mạnh hơn |
| **(khó)** | **MVCC**: vì sao reader không block writer? | **snapshot** theo **version** |
| **(khó)** | **Lost update** + cách chặn? | **FOR UPDATE** / **CAS** / **atomic update** |

> 🧩 **Thách đố tình huống (TL):**
> *Nâng Serializable "cho an toàn" có miễn phí không?*
>
> Trả lời chuẩn: KHÔNG — tốn **abort/retry**, **throughput giảm**; nên **giải điểm nóng cục bộ**.
> ⚠️ **Bẫy phải vạch ra:** **Serializable cần retry loop** — nếu không bắt `serialization_failure` để retry thì app sẽ vỡ.

---

## ⑨ Bài tập + tiêu chí tự chấm

### BT1 — Hai request đồng thời trừ 1 vé cuối

Viết SQL đảm bảo **không bán âm**.

> ✅ **Tiêu chí Đạt:**
> - **atomic** `UPDATE ... WHERE seats > 0` **hoặc** **FOR UPDATE**.
> - Giải thích vì sao **SELECT-rồi-UPDATE sai** (gây **lost update**, hai request cùng đọc rồi ghi đè).
> - ❌ Trượt nếu dùng SELECT rồi UPDATE rời mà tưởng an toàn.

### BT2 — Deadlock do thứ tự khóa ngược

Hai tx update bảng **A rồi B** vs **B rồi A** → **deadlock**. Hãy sửa.

> ✅ **Tiêu chí Đạt:**
> - **Thống nhất thứ tự khóa** (luôn **A trước B**) ở mọi tx.
> - Giải thích vì sao thứ tự nhất quán phá được vòng chờ.
> - ❌ Trượt nếu chỉ "thêm retry" mà không sửa thứ tự khóa.

---

## ⑩ Đọc thêm

- **PostgreSQL — Transaction Isolation & MVCC** — `postgresql.org/docs/current/transaction-iso.html`, `.../mvcc.html`
- **Kleppmann, *DDIA* Ch.7** (Transactions)

> 🎯 **Một câu mang về:** Concurrency không sửa bằng cách "siết chặt tất cả". Hiểu **MVCC**, khoanh đúng **điểm nóng**, dùng **atomic/FOR UPDATE** tại chỗ — đó là tư duy của senior, không phải nâng **Serializable** toàn cục rồi cầu may.
