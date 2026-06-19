# 🎯 BÀI 3 — Idempotency (góc concurrency / distributed)

**Phủ:** I-IDEM-001 → I-IDEM-006 · *"An toàn khi lặp"* — **cầu nối từ single-DB sang phân tán.**

> 📌 **Phạm vi bài này.** Mục **I** nhìn **idempotency** từ góc **retry / đa luồng / distributed**. Cơ chế phía **HTTP** (`Idempotency-Key` header, **method semantics** theo **RFC 9110**, TTL/scope) thuộc **F-IDEM** — *không lặp lại ở đây*.

> 💡 **Cách đọc bài này.** Một câu cốt lõi: ***mạng sẽ giao trùng, nên consumer (bên xử lý) buộc phải "lặp-không-đổi".*** Nhưng có một cú twist chí mạng (I-IDEM-005): idempotency chỉ cứu bạn khi các bản trùng đến **tuần tự** — nếu chúng đến **song song**, bạn *vẫn dính race*. Nắm được cái twist đó là nắm được phần khó nhất của cả mục I.

---

## ① Mục tiêu & vị trí trong mạch

Học xong bài này bạn sẽ làm được 5 việc:

1. **Hiểu vì sao** **retry** + **at-least-once** (giao *ít nhất một lần*) **bắt buộc** consumer phải **idempotent**.
2. **Biết** rằng **"exactly-once delivery"** (giao *đúng một lần*) là **ảo tưởng**, và cách đạt **effectively-once** thay thế.
3. **Thiết kế dedup key atomic** (khóa chống trùng, claim nguyên tử).
4. **Phân biệt** thao tác **tự idempotent** (naturally idempotent) vs thao tác **phải làm cho idempotent**.
5. **Nắm điểm chí mạng:** *idempotency một mình chưa đủ* khi **2 retry chạy song song**.

> 🎯 **Vị trí trong mạch học.**
> - **Dùng lại vũ khí Bài 2:** bài này tận dụng chính **unique constraint / version** đã học ở Bài 2.
> - **Mở đường Bài 4 + mục K:** dẫn sang **distributed** (Bài 4) và **messaging** (mục K — Outbox).

**Ẩn dụ định vị:** Bài 1–2 lo chuyện trong *một* DB. Bài 3 bắt đầu bước ra thế giới **phân tán** — nơi *mạng không tin cậy* khiến mọi thông điệp có thể tới *nhiều lần*. Idempotency là *áo giáp tối thiểu* để sống sót ở thế giới đó.

---

## ② Giảng cơ bản → nâng cao

### (a) Trực giác

> **Ví dụ trực giác:** Bấm nút gọi **thang máy 5 lần** vẫn chỉ gọi **đúng 1 thang**. Đó là **idempotent** — *lặp lại bao nhiêu lần cũng không đổi kết quả*.

Đối chiếu hai kiểu thao tác bằng con số cụ thể:

| Thao tác | Lặp 2 lần | Idempotent? |
|---|---|---|
| `SET balance = 100` (gán tuyệt đối) | vẫn ra **100** | ✅ Có |
| `balance += 10` (cộng delta) | ra **+20** → **sai gấp đôi** | ❌ Không |

> 💡 **Idempotent** = *thực hiện lặp lại không làm đổi kết quả so với thực hiện một lần*. Chìa khóa nằm ở chỗ thao tác là **gán tuyệt đối** hay **cộng/tạo thêm**.

---

### (b) Cơ chế chi tiết

#### 🔍 Vì sao bắt buộc idempotent — **I-IDEM-001**

**Mạng và message queue SẼ giao trùng** — không phải "có thể", mà là *chắc chắn xảy ra ở quy mô đủ lớn*:
- **Timeout rồi retry:** client gửi request, mạng chậm, client tưởng hỏng → **gửi lại** (dù request đầu *đã* tới nơi).
- **Broker redeliver:** message broker giao message, chưa nhận được ack kịp → **giao lại**.

**Hậu quả nếu consumer KHÔNG idempotent:** tính tiền 2 lần, gửi mail 2 lần, ghi đơn 2 lần.

> 🎯 **Chốt:** **At-least-once** (giao *ít nhất một lần*) là **mặc định an toàn** của hầu hết queue — broker thà giao *thừa* còn hơn giao *thiếu*. Nên **consumer phải chịu được trùng**. Idempotency không phải "tính năng xịn", nó là *điều kiện sống còn*.

**Timeline — at-least-once gây xử lý trùng:**

```
T1  Client → Server: POST /charge ($50)
T2  Server: tính tiền $50 ✅, gửi response...
       ↓  (mạng rớt response, client KHÔNG nhận được 2xx)
T3  Client: tưởng thất bại → RETRY: POST /charge ($50)
T4  Server (không idempotent): tính tiền $50 LẦN NỮA ❌ → khách bị trừ $100
```

---

#### 🔍 Exactly-once delivery là ảo tưởng — **I-IDEM-002**

> **Ví dụ trực giác — bài toán Two Generals:** Hai vị tướng ở hai ngọn đồi, muốn hẹn giờ tấn công, chỉ liên lạc qua *người đưa tin băng qua thung lũng địch* (có thể bị bắt). Tướng A gửi "tấn công lúc 5h". Làm sao A *chắc chắn* B đã nhận? A cần B gửi xác nhận. Nhưng làm sao B chắc A đã nhận *xác nhận*? Cần A xác nhận lại... → **vòng lặp vô tận, không bao giờ chắc chắn 100%.**

**Vì sao liên quan?** Bên gửi **không bao giờ chắc chắn tuyệt đối** rằng bên nhận đã nhận *đúng một lần* qua một **mạng không tin cậy**. Đó là lý do **exactly-once delivery** (giao đúng một lần, đảm bảo tuyệt đối) là **bất khả thi**.

**Vậy thực tế đạt gì?** **Effectively-once** (*hiệu quả như một lần*):

> 🎯 **Công thức vàng:** **effectively-once = at-least-once + consumer idempotent/dedup.**
>
> Tức: cứ để mạng giao *thừa* (at-least-once), nhưng consumer **lọc trùng** (dedup) → *kết quả cuối* y như chỉ xử lý một lần.

> 🔎 **Lưu ý sâu:** cái mà **Kafka** gọi là *"exactly-once"* thực chất cũng là **dedup + transaction nội bộ**, **không phá** định lý Two Generals. Đừng để cái tên marketing đánh lừa.

---

#### ✅ Dedup key atomic ở đâu — **I-IDEM-003**

**Ý tưởng:** mỗi thao tác mang một **dedup key** (khóa chống trùng, ví dụ `event_id`). Lưu key này bằng một thao tác **atomic** sao cho *chỉ một request claim được key, các bản trùng bị từ chối*.

**Cách đúng — atomic claim:**

```sql
INSERT ... ON CONFLICT DO NOTHING   -- hoặc unique constraint
```

→ Một request **thắng** (insert được), các bản trùng **bị từ chối** (conflict → không làm gì).

> 🚨 **KHÔNG dùng check-then-act:** *"`SELECT` xem có chưa → nếu chưa thì `INSERT`"* — vì đó lại đúng là **TOCTOU** (Bài 1)! Giữa `SELECT` và `INSERT` có khoảng trống, hai request cùng thấy "chưa có" → cùng INSERT.

**Timeline — vì sao SELECT-rồi-INSERT vẫn trùng:**

```
processed_events: (rỗng)

T1  webhook-A: SELECT WHERE event_id='evt_9' → 0 dòng ("chưa có")
T2  webhook-B: SELECT WHERE event_id='evt_9' → 0 dòng ("chưa có")  ❌ cùng thấy chưa có
       ↓
T3  webhook-A: INSERT 'evt_9' → ok, rồi fulfillOrder() ❌ lần 1
T4  webhook-B: INSERT 'evt_9' → ok (nếu không có unique constraint), rồi fulfillOrder() ❌ lần 2
→ 2 order trùng.  Đây là TOCTOU y như Bài 1.
```

> 🎯 **Chốt:** **Key theo business id** (ví dụ `event_id`, `order_id`), đặt **unique constraint**, và set **TTL hợp lý** (khóa không cần giữ mãi mãi).

---

### (c) Phân loại & bẫy chí mạng

#### 🔍 Naturally vs phải-làm-cho — **I-IDEM-004**

| Loại | Ví dụ | Cần thêm gì? |
|---|---|---|
| **Naturally idempotent** (tự idempotent) | **set tuyệt đối**: `PUT state=X`, `SET balance=100`, `DELETE id=5` | Không — *lặp sao cũng vậy* |
| **Phải làm cho idempotent** | **delta / create**: `+10`, `INSERT order` | Gắn **dedup key** hoặc **version / CAS** |

> 💡 **Kỹ năng cần luyện:** nhìn một thao tác và *biết ngay* nó thuộc loại nào. **Gán tuyệt đối → an toàn sẵn. Cộng thêm / tạo mới → phải gia cố.**

---

#### 🚨 Idempotent NHƯNG 2 retry chạy SONG SONG — **I-IDEM-005** (⭐⭐⭐⭐⭐)

> **Ví dụ trực giác:** "Bấm thang máy nhiều lần vẫn 1 thang" — đúng *khi bấm lần lượt*. Nhưng nếu **hai người ở hai tầng bấm GỌI ĐÚNG cùng một khoảnh khắc**, hệ thống *chưa kịp ghi nhận lần bấm đầu* thì lần bấm thứ hai đã tới → vẫn có thể sinh ra hành vi trùng.

**Đây là cái bẫy đắt giá nhất bài:** một handler **idempotent vẫn race** khi **2 retry chạy song song cùng key**.

**Vì sao?** Vì **idempotency chỉ nói**: *"lặp **tuần tự** thì không sao."* Nó **không nói gì** về lặp **song song**. Hai request cùng key chạy *đồng thời*, cả hai cùng thấy "chưa có" → cùng INSERT / cùng gây side-effect.

**Timeline — idempotent nhưng vẫn race song song:**

```
                  TUẦN TỰ (idempotent CỨU được)        SONG SONG (idempotent KHÔNG đủ)
                  ─────────────────────────────        ──────────────────────────────
 req-1: check "có chưa?" → chưa → làm ✅                req-1: check → chưa ─┐
 req-1: ghi "đã làm"                                    req-2: check → chưa ─┘ ❌ cùng thấy chưa
 req-2: check "có chưa?" → CÓ RỒI → bỏ qua ✅            req-1: làm ❌  req-2: làm ❌ → TRÙNG
```

> 🎯 **Chốt vàng:** **Idempotency ≠ chống song song.** Cần thêm **atomic claim** (**unique constraint**) hoặc **lock** để **serialize** (xếp các bản trùng thành tuần tự). → **idempotency + concurrency control phải đi cùng nhau.**

---

#### 🔍 Liên hệ optimistic lock & outbox — **I-IDEM-006**

| Cơ chế | Làm gì | Cùng mục tiêu |
|---|---|---|
| **version / CAS** (Bài 2) | Làm **write idempotent theo state** | An toàn khi lặp |
| **Outbox pattern + dedup** | Đảm bảo **publish event** không trùng / không mất: *ghi event vào **cùng transaction** với data, rồi relay (chuyển tiếp) sau* | An toàn khi lặp |

> 💡 **Outbox pattern** giải bài "vừa ghi DB vừa gửi message mà không bị lệch": thay vì gọi message broker trực tiếp (có thể ghi DB xong nhưng gửi message hỏng), ta *ghi "cần gửi" vào một bảng outbox **trong cùng tx** với data*, rồi một tiến trình relay đọc bảng đó gửi đi sau. → **cross-ref mục K (Outbox).**

> 📘 **vs** ➕ — đây gần như **toàn bộ là phần ➕** ở góc concurrency (*docs gốc không đào sâu*); phần **HTTP idempotency** thuộc **F**.

---

## ③ ⚠️ Kiến thức cũ / cách làm bị thay thế

| Cái cũ | Thay bằng (nên dùng) | Vì sao |
|---|---|---|
| Tin vào **"exactly-once delivery"** của queue | **at-least-once + consumer idempotent** (**effectively-once**) | **Exactly-once delivery** bất khả thi qua mạng (Two Generals) |
| Dedup bằng **`SELECT` rồi `INSERT` nếu chưa có** | **`INSERT ... ON CONFLICT DO NOTHING`** / **unique constraint** | **SELECT-then-INSERT** là **TOCTOU** |
| Coi mọi handler là idempotent vì *"đã có idempotency key"* | Thêm **atomic claim** cho key chạy song song | **Idempotency ≠ chống song song** |
| **Side-effect** (charge, email) chạy **trước** khi claim key | **Claim key atomic TRƯỚC**, side-effect sau (và idempotent hóa chính side-effect) | Tránh **double-charge** khi trùng |

---

## ④ Áp dụng thực tế + So sánh bigtech

**Use case — webhook thanh toán (Stripe-style).**
Provider gửi `payment.succeeded` và **retry đến khi nhận `2xx`** → bạn **nhận trùng**. Handler phải:

1. **Claim `event_id`** qua **unique constraint** *trước*,
2. rồi mới **xử lý**;
3. lần trùng thấy key **đã tồn tại** → trả **`200`** và **bỏ qua**.

**Timeline — webhook idempotent đúng cách:**

```
T1  provider → POST webhook (event_id='evt_42')
T2  handler: INSERT evt_42 ON CONFLICT DO NOTHING → claim ĐƯỢC (rowCount=1) → fulfillOrder() ✅
       ↓  (provider chưa nhận 2xx kịp → retry)
T3  provider → POST webhook (event_id='evt_42')  [lần 2]
T4  handler: INSERT evt_42 ON CONFLICT DO NOTHING → rowCount=0 (đã có) → trả 200, BỎ QUA ✅
→ side-effect chạy đúng 1 lần.
```

**Bigtech pattern:**
- Các **payment API** lớn yêu cầu client gửi **idempotency key** và họ **dedup ở server** (đúng kiểu **F-IDEM**).
- **Message platform** (**Kafka**) cung cấp *"exactly-once semantics"* **nội bộ** bằng **producer idempotent + transaction**, *nhưng phía consumer vẫn nên idempotent*.

> ⚠️ Đây là **pattern phổ biến**; **verify chi tiết API** trước khi phát biểu chắc nịch.

---

## ⑤ Code thực hành + cấu hình

**Bảng dedup + câu claim atomic:**

```sql
-- ✅ Atomic claim dedup key: 1 thắng, trùng bị từ chối — KHÔNG check-then-act
CREATE TABLE processed_events (
  event_id   TEXT PRIMARY KEY,        -- unique constraint = điểm dedup atomic
  created_at TIMESTAMPTZ DEFAULT now()
);

-- Trả về 1 dòng nếu claim thành công; 0 dòng nếu đã xử lý (trùng)
INSERT INTO processed_events (event_id)
VALUES ($1)
ON CONFLICT (event_id) DO NOTHING
RETURNING event_id;
```

**Webhook handler idempotent + atomic claim:**

```javascript
// ✅ Webhook handler idempotent + atomic claim (chống cả retry TUẦN TỰ lẫn SONG SONG)
async function handlePaymentWebhook(eventId: string, payload: Payload) {
  // 1) CLAIM key atomic TRƯỚC khi gây side-effect
  const claimed = await db.query(
    `INSERT INTO processed_events (event_id) VALUES ($1)
     ON CONFLICT (event_id) DO NOTHING RETURNING event_id`,
    [eventId],
  );
  if (claimed.rowCount === 0) {
    return { status: 200, note: 'đã xử lý — bỏ qua bản trùng' }; // idempotent: trả 200, không làm lại
  }
  // 2) Chỉ ĐẾN ĐÂY khi mình là người THẮNG claim → làm side-effect đúng 1 lần
  await fulfillOrder(payload);
  // (lý tưởng: gói claim + side-effect trong CÙNG transaction để all-or-nothing)
}

// ❌ ANTI-PATTERN: check-then-act (TOCTOU) — 2 webhook song song cùng lọt
// const seen = await db.query('SELECT 1 FROM processed_events WHERE event_id=$1', [eventId]);
// if (seen.rowCount === 0) { await db.query('INSERT ...'); await fulfillOrder(); }
```

> 🔧 **Cấu hình:**
> - Đặt **side-effect SAU claim**; lý tưởng **gói trong một transaction** (claim + ghi data) để **all-or-nothing** (được tất cả hoặc không gì).
> - Nếu side-effect là **gọi service ngoài** (không rollback được) → kết hợp **outbox** (ghi *"cần gửi"* vào **cùng tx**, relay sau) **thay vì gọi trực tiếp trong tx**.

---

## ⑥ Keywords cần nhớ (glossary)

**🧠 Cần HIỂU (nắm bản chất):**
- **idempotency** — lặp lại không đổi kết quả.
- **at-least-once vs exactly-once** — giao ít nhất một lần (an toàn, mặc định) vs giao đúng một lần (bất khả thi qua mạng).
- **effectively-once** — at-least-once + dedup = *hiệu quả như một lần*.
- **dedup** — lọc bản trùng.
- **"idempotent ≠ chống song song"** — chỉ cứu lặp tuần tự, không cứu lặp song song.
- **outbox** — ghi event vào cùng tx với data, relay sau.

**📌 Cần THUỘC (nhớ nguyên văn):**
- `INSERT ... ON CONFLICT DO NOTHING RETURNING;`
- *"claim key TRƯỚC, side-effect SAU"*.
- *"naturally-idempotent = absolute set; delta/create = cần dedup"*.

**🛠️ Cần LÀM ĐƯỢC (kỹ năng tay):**
- Viết **webhook handler idempotent**.
- Thiết kế **dedup key atomic**.
- **Nhận diện** thao tác **cần thêm cơ chế**.

---

## ⑦ Mạch tư duy cần nhớ (mental model)

> 🎯 **Khẩu quyết 1 — lặp tuần tự vs song song:**
> *"**Mạng SẼ giao trùng → consumer phải lặp-không-đổi.** Nhưng idempotent chỉ cứu lặp **TUẦN TỰ**; lặp **SONG SONG** cần thêm **atomic claim**."*

> 🎯 **Khẩu quyết 2 — exactly-once là ảo tưởng:**
> *"**Exactly-once delivery KHÔNG tồn tại** — chỉ có **at-least-once + dedup = effectively-once**."*

---

## ⑧ Câu hỏi phỏng vấn & thách đố

**(TB)** Vì sao **retry + at-least-once** bắt buộc consumer **idempotent**?
→ *Gợi ý:* mạng/broker giao trùng → không idempotent thì charge/email/đơn nhân đôi.

**(khó)** **"Exactly-once delivery"** có thật không? Đạt **effectively-once** bằng gì?
→ *Gợi ý:* không (Two Generals); **at-least-once + consumer idempotent/dedup**.

**(khó)** Thiết kế **dedup key** chống trùng dưới **concurrency** — lưu ở đâu cho **atomic**?
→ *Gợi ý:* **unique constraint / `INSERT ON CONFLICT`**, *không* SELECT-then-INSERT.

**(khó)** Phân biệt **naturally idempotent** vs **phải-làm-cho-idempotent**, cho ví dụ.
→ *Gợi ý:* set tuyệt đối (PUT/SET/DELETE) vs delta/create (+10 / INSERT) cần dedup.

> 🧩 **Thách đố / trick (⭐⭐⭐⭐⭐) — AI hay sai ở đây:**
> *"Handler của tôi đã idempotent rồi, sao 2 webhook đến cùng lúc vẫn tạo 2 order?"*
> → **idempotency không chống song song**; thiếu **atomic claim** → cả hai cùng lọt; phải dùng **unique constraint / ON CONFLICT** để **serialize**.
>
> *"AI viết dedup bằng `SELECT ... if not exists then INSERT` — ổn chứ?"*
> → ❌ **không** — đó là **TOCTOU**; phải **atomic `INSERT ON CONFLICT`**. Lại đúng kiểu **AI viết đúng cú pháp nhưng sai hành vi**.

---

## ⑨ Bài tập thực hành + tiêu chí tự chấm

**BT1 — Webhook idempotent với `processed_events`.**
> ✅ **Đạt khi:** gửi **cùng `event_id` 5 lần** (kể cả **2 lần đồng thời**) → side-effect chạy **đúng 1 lần**, 4 lần còn lại trả **`200` "đã xử lý"**.

**BT2 — Phân loại 6 thao tác.**
Phân loại: `PUT name`, `+10 điểm`, `DELETE`, `INSERT comment`, `SET status='done'`, `gửi email` → thành **tự-idempotent** vs **phải-thêm-cơ-chế**.
> ✅ **Đạt khi:** **phân đúng** + **nêu cơ chế cần thêm** cho nhóm sau.

---

## ⑩ Đọc thêm / nguồn chuẩn

- **RFC 9110 §9.2** (*idempotent methods*) — cho góc **HTTP (F)**.
- **Stripe docs** — *Idempotent Requests* (mô hình tham chiếu).
- **Two Generals' Problem** — cơ sở của *"exactly-once bất khả thi"*.
- **microservices.io** — *Transactional Outbox / Idempotent Consumer*.

---

> 🔚 **Hết Bài 3.** → **Bước 3 — Chốt ghi nhớ.**
>
> 🎯 **Điểm nóng để recall:** *"idempotent nhưng vẫn race khi song song"*. Gợi ý 3 câu tự hỏi: *(1) Vì sao at-least-once buộc consumer idempotent? (2) Exactly-once delivery có thật không, thay bằng gì? (3) Handler đã idempotent mà 2 webhook song song vẫn tạo 2 order — thiếu gì và sửa thế nào?*
