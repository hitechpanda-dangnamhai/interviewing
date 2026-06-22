# 🔁 CHUYÊN ĐỀ — IDEMPOTENCY (Tính bất biến khi lặp lại) trong Caching & Hệ phân tán
> Đào sâu một concept mở rộng của **Bài 1 — Vì sao cache · Các tầng cache · Khi nào KHÔNG cache**
> Mục tiêu: hiểu *idempotency là gì*, vì sao nó biến **at-least-once delivery** thành **effectively-once effects**, và cách *thiết kế thao tác/retry an toàn* ở mức **staff engineer**.
> (✅ = đã có trong Bài 1 · ➕ = mở rộng thêm)

---

## ① Mục tiêu & vị trí trong mạch

Trong Bài 1, **Idempotency** xuất hiện ở **bảng concept mở rộng (B2)**:

> ➕ **Idempotency** — *Làm lại nhiều lần kết quả vẫn như một* — *"Back-fill phải idempotent để retry an toàn."*

Và nó đã len lỏi khắp các chuyên đề trước như một **nhân vật phụ quan trọng**:
- **Eventual Consistency** (A6): *"trong hệ eventual + retry, mọi ghi/invalidate phải idempotent để retry an toàn"*.
- **CAP** (thách đố): khoá phân tán + **idempotency** để dù xử lý trùng cũng vô hại.
- **Cache Coherence** (A3): CDC consumer **idempotent** (xoá key hai lần vô hại).

Chuyên đề này đưa nó ra **sân khấu chính**. Học xong bạn sẽ:

- Hiểu **idempotency chính xác** (và phân biệt với "trả về cùng response").
- Nắm **cái khung trung tâm**: *delivery* chỉ có thể **at-least-once** → idempotency biến nó thành **effectively-once** *effects*.
- Thuộc **ngữ nghĩa idempotency của HTTP** (GET/PUT/DELETE idempotent, **POST thì không**).
- Biết **idempotency key pattern** (Stripe), **UPSERT**, **outbox**, và cách xử **race trên chính idempotency key**.

> 💡 **Một câu để gắn vào mạch Bài 1:** Cache sống nhờ **retry & lặp lại** — nhiều request cùng miss một key đều **back-fill**, một invalidate có thể được phát lại, một message có thể tới hai lần. Idempotency là **điều kiện để những lần lặp đó KHÔNG gây hại**. Không có nó, "thử lại cho chắc" trở thành "trừ tiền hai lần".

---

## ② Trực giác — Idempotency là gì?

### Định nghĩa

**Idempotency = thực hiện một thao tác MỘT LẦN hay NHIỀU LẦN đều cho cùng một KẾT QUẢ (cùng trạng thái cuối, cùng tác dụng phụ).**

Toán học: hàm idempotent thoả **f(f(x)) = f(x)**. Ví dụ `abs(abs(x)) = abs(x)`, nhân với 1, hợp một tập với chính nó.

### Phân biệt then chốt: idempotent là về TÁC DỤNG PHỤ, không (chỉ) về response

> 🔑 **Insight #1:** Idempotent **không** có nghĩa "lần nào cũng trả response y hệt". Nó có nghĩa **tác dụng phụ chỉ xảy ra MỘT lần** dù gọi nhiều lần. `DELETE /user/42` lần đầu xoá (200), lần hai trả 404 — *response khác nhau* nhưng **trạng thái cuối giống nhau** (user 42 đã biến mất) → **vẫn idempotent**. Cái ta quan tâm là *"gọi lại có gây thêm thay đổi không"*, không phải *"response có byte-by-byte giống không"*.

### Ẩn dụ công tắc đèn vs máy đếm

- **Công tắc "BẬT" (idempotent):** bấm "bật" một lần hay mười lần → đèn vẫn **bật**. Trạng thái cuối như nhau.
- **Nút "+1" của máy đếm (KHÔNG idempotent):** bấm một lần = 1, bấm mười lần = 10. Mỗi lần lặp **thêm tác dụng**.

> Thiết kế hệ phân tán = cố biến càng nhiều thao tác thành **"công tắc BẬT"** càng tốt; cái nào buộc phải là **"+1"** thì phải **bọc dedup** (mục ⑤).

---

## ③ Cái khung TRUNG TÂM — vì sao idempotency là bắt buộc

### Mạng không đáng tin → retry là CHẮC CHẮN

Trong hệ phân tán, một request có thể:
1. **Thất bại thật** (chưa tới server) → cần retry.
2. **Thành công nhưng response bị mất trên đường về** → client **tưởng** thất bại → **retry** → **server xử lý LẦN HAI**.

Client **không phân biệt được** hai trường hợp. Nên nó **buộc phải retry** — và trường hợp 2 gây **double-execution**.

### Bậc thang "delivery semantics"

| Mức | Nghĩa | Khả thi? |
|---|---|---|
| **At-most-once** | Tối đa một lần (có thể mất, không bao giờ trùng) | Dễ, nhưng **mất việc** |
| **Exactly-once *delivery*** | Đúng một lần, không mất không trùng | ❌ **Bất khả thi** trên kênh không tin cậy (Two Generals) |
| **At-least-once** | Ít nhất một lần (không mất, **có thể trùng**) | ✅ Thực tế dùng cái này |

> 🔑 **Insight #2 (quan trọng nhất cả bài):** **Exactly-once *delivery* là HUYỀN THOẠI** — không tồn tại trên mạng không tin cậy. Thứ duy nhất thực tế là **at-least-once** (chấp nhận trùng). Vậy làm sao đạt "đúng một lần"?
>
> **Idempotency biến at-least-once *delivery* thành effectively-once *effects*.**
>
> Bạn *không* chặn được message tới hai lần; bạn *chặn* được **lần thứ hai gây tác dụng**. "Exactly-once" mà mọi người nói tới (kể cả Kafka) thực chất là **idempotency + dedup ở phía nhận**, không phải delivery thần kỳ.

### Hậu quả khi thiếu idempotency

Double-charge (trừ tiền 2 lần), double-order (2 đơn), double-email, tồn kho trừ 2 lần, +1 like thành +2. Toàn những bug **mất tiền & mất niềm tin** — đúng các path mà Bài 1 (D2) bảo *"cần strong consistency, đừng cache bừa"*.

---

## ④ Idempotency trong HTTP — kiến thức nền PHẢI thuộc

REST chuẩn hoá idempotency theo **method**. Đây là câu hỏi phỏng vấn kinh điển:

| Method | Safe? (không đổi state) | Idempotent? | Ghi chú |
|---|---|---|---|
| **GET / HEAD / OPTIONS / TRACE** | ✅ | ✅ | Chỉ đọc → lặp vô hại (đọc cache cũng vậy) |
| **PUT** | ❌ | ✅ | Gán **tuyệt đối** `x=5` → lặp vẫn `x=5` |
| **DELETE** | ❌ | ✅ | Xoá rồi xoá lại → vẫn "đã xoá" |
| **POST** | ❌ | ❌ | **Tạo mới mỗi lần** → double-submit = 2 resource |
| **PATCH** | ❌ | ⚠️ Tuỳ | `set field` thì idempotent; `increment` thì KHÔNG |

> 🔑 **Insight #3:** **Safe ⊂ Idempotent.** Mọi method safe đều idempotent (đọc không đổi gì), nhưng idempotent không nhất thiết safe (PUT/DELETE đổi state mà vẫn idempotent). Và **POST là thủ phạm double-order** — vì nó tạo mới mỗi lần. Đó là lý do form "Đặt hàng" cần **idempotency key** (mục ⑤), hoặc thiết kế lại thành PUT với ID do client sinh.

> 💡 **Mẹo thiết kế:** Khi được chọn, **ưu tiên ngữ nghĩa "gán tuyệt đối" (PUT/set)** hơn "tương đối" (POST/increment). "Đặt số dư = 100" idempotent; "cộng 30" thì không. Thiết kế API quanh trạng thái-mong-muốn (declarative) cho bạn idempotency gần như miễn phí (xem IaC ở mục ⑩).

---

## ⑤ Idempotency Key — pattern xương sống (biến POST/+1 thành an toàn)

Khi thao tác **bản chất không idempotent** (tạo đơn, trừ tiền, +1), ta **bọc** nó bằng một **khoá định danh duy nhất cho mỗi thao tác logic**:

### Cơ chế

1. **Client sinh một key duy nhất** (UUID) cho *một ý định* (vd "đơn hàng này"). Mọi lần **retry** dùng **cùng key**.
2. **Server**: tra key trong **dedup store**.
   - **Chưa thấy** → xử lý, **lưu `key → kết quả`**, trả kết quả.
   - **Đã thấy (retry)** → **KHÔNG xử lý lại**, trả thẳng kết quả đã lưu.

```
Lần 1: key=abc → chưa có → tạo đơn #1001, lưu abc→#1001 → trả #1001
Lần 2 (retry, response lần 1 bị mất): key=abc → ĐÃ CÓ → trả #1001 (KHÔNG tạo đơn mới) ✅
```

### Code minh hoạ (Redis SETNX để "giành" key nguyên tử)

```js
// Idempotency key do client gửi qua header. Dedup store = Redis.
export async function createOrder(idempotencyKey, payload, db) {
  const lockKey = `idem:${idempotencyKey}`;

  // 1) Giành key nguyên tử: SET NX = chỉ set nếu CHƯA tồn tại
  const won = await redis.set(lockKey, "IN_PROGRESS", "NX", "EX", 86400); // giữ 24h
  if (won === null) {
    // Đã có → là retry hoặc request song song trùng key
    const saved = await redis.get(lockKey);
    if (saved && saved !== "IN_PROGRESS") return JSON.parse(saved); // trả kết quả cũ
    throw new Error("DUPLICATE_IN_PROGRESS"); // đang xử lý → client chờ/thử lại sau
  }

  // 2) Lần đầu → xử lý thật (lý tưởng: trong cùng transaction DB)
  const order = await db.insertOrder(payload);

  // 3) Lưu kết quả cho các retry sau
  await redis.set(lockKey, JSON.stringify(order), "EX", 86400);
  return order;
}
```

> 🔑 **Insight #4 — cạm bẫy race trên CHÍNH idempotency key:** Hai request **song song** cùng key (double-click, retry sớm) có thể **cùng vượt qua bước kiểm tra** nếu kiểm tra không nguyên tử → **cùng xử lý** → mất tác dụng. Phải **giành key nguyên tử**: `SET NX` (Redis), hoặc **unique index** trên cột `idempotency_key` ở DB (để DB *từ chối* bản trùng), hoặc advisory lock. Trạng thái `IN_PROGRESS` xử lý ca "đang chạy thì retry tới".

### Vòng đời key & trạng thái

Một idempotency key nên là **máy trạng thái**: `IN_PROGRESS → COMPLETED` (lưu kết quả) hoặc `FAILED`. Và phải có **thời hạn lưu (retention)**: đủ dài để bao trùm mọi retry hợp lý (Stripe ~24h), nhưng không vô hạn (tốn bộ nhớ). Quá ngắn → retry sau khi key hết hạn = **xử lý lại** (Insight #2 thất bại).

---

## ⑥ Ví dụ minh hoạ (gắn caching)

### Ví dụ 1 — Back-fill idempotent (đúng câu Bài 1 B2)

Nhiều request cùng miss `product:42` → **tất cả** đọc DB rồi **back-fill** vào cache. Vì "ghi `product:42 = <value>`" là **gán tuyệt đối** (PUT-style), back-fill **idempotent tự nhiên**: 5 request cùng ghi cùng giá trị → cache vẫn đúng. (Còn để khỏi 5 lần đập DB thì cần **singleflight** — Cold Start doc — nhưng *tính đúng đắn* đã được idempotency bảo đảm.)

### Ví dụ 2 — Invalidate idempotent

`DELETE cache key` lặp lại vô hại (key đã mất vẫn "đã mất"). Nên CDC pipeline (Cache Coherence doc) có thể **phát invalidate at-least-once** mà không sợ — consumer xoá hai lần chẳng sao. Idempotency là *thứ làm at-least-once messaging dùng được*.

### Ví dụ 3 — KHÔNG idempotent: cache một counter qua write-back

"Tăng view +1" **không** idempotent. Nếu retry write hoặc message trùng → **+2**. Sửa: hoặc dùng **counter dedup theo event-id** (mỗi view có id, đếm distinct id), hoặc **CRDT counter** (mục ⑨), hoặc chấp nhận lệch (vì như Bài 1 nói, view-count chịu được eventual).

### Ví dụ 4 — Double-submit đặt hàng (POST)

User bấm "Đặt hàng" hai lần (mạng lag). Hai POST → **hai đơn**. Sửa: nút sinh **idempotency key** lúc mở form; cả hai submit mang **cùng key** → server tạo **một** đơn (mục ⑤).

---

## ⑦ Case study thực tế

> ⚠️ Tóm tắt theo tài liệu/bài báo công khai và pattern phổ biến. Verify số liệu cụ thể khi cần trích dẫn.

### CS1 — Stripe: Idempotency-Key cho thanh toán (chuẩn mực ngành)

Stripe cho client gửi header **`Idempotency-Key`** (UUID) khi tạo charge. Server lưu key + kết quả (khoảng **24 giờ**); mọi **retry cùng key** trả lại **kết quả gốc**, **không trừ tiền lần nữa**. Request song song cùng key bị khoá để tránh race.

> 🎯 **Bài học:** Trên path tiền, **idempotency key là bắt buộc, không phải tuỳ chọn**. Đây là cách biến API thanh toán (POST, không idempotent) thành an toàn với retry.

### CS2 — Kafka: "exactly-once" thực chất là idempotency + transaction

Kafka **idempotent producer** gắn **Producer ID + sequence number** cho mỗi message; broker **dedupe** các message trùng do producer retry → **không ghi trùng**. Cộng với **transactions** (read_committed) cho **exactly-once semantics (EOS)** *trong Kafka*.

> 🎯 **Bài học:** Cái gọi là "exactly-once" của Kafka **củng cố Insight #2**: nó **không** phải delivery thần kỳ — là **dedup bằng sequence number** (idempotency ở tầng broker) + transaction. Đầu cuối tới sink ngoài vẫn cần **sink idempotent**.

### CS3 — SQS & message queue: at-least-once → consumer phải idempotent

**SQS standard** giao **at-least-once** (có thể trùng) → **consumer buộc phải idempotent**. **SQS FIFO** cho "exactly-once processing" trong **cửa sổ dedup ~5 phút** qua **MessageDeduplicationId** (lại là idempotency key).

> 🎯 **Bài học:** Mọi hàng đợi at-least-once (SQS, RabbitMQ, hầu hết) **đẩy trách nhiệm idempotency sang consumer**. Thiết kế consumer = thiết kế dedup.

### CS4 — Database UPSERT: idempotent write tích hợp sẵn

`INSERT ... ON CONFLICT DO UPDATE` (Postgres) / `ON DUPLICATE KEY UPDATE` (MySQL) / `MERGE`: ghi **theo khoá**, chạy lại cho **cùng trạng thái cuối** → idempotent. Kết hợp **unique index trên idempotency_key** để DB *tự* từ chối bản trùng (giải Insight #4 race).

### CS5 — TCP: idempotency xây sẵn trong giao thức

TCP gắn **sequence number** cho mỗi byte; segment **truyền lại** (do mất ack) bị **nhận diện trùng và bỏ** ở đầu nhận. ⇒ truyền lại at-least-once nhưng **effect exactly-once**. Idempotency không phải khái niệm mới — nó là **nền của mạng đáng tin xây trên mạng không tin**.

---

## ⑧ Giải pháp staff engineer — làm thao tác/retry an toàn

### Bậc thang chọn cách (rẻ → công phu)

1. **Thao tác vốn idempotent?** → ưu tiên thiết kế **gán tuyệt đối** (PUT/set/UPSERT) thay vì +1/append/INSERT. *Miễn phí.*
2. **Đọc?** → tự nhiên idempotent (kể cả back-fill cache). Không cần làm gì.
3. **Không tránh được non-idempotent?** → **idempotency key + dedup store** (mục ⑤).
4. **DB sẵn có?** → **unique constraint** trên `idempotency_key` để DB chặn trùng (nguyên tử, đáng tin nhất).
5. **Hàng đợi?** → **consumer idempotent** (dedup table theo message-id) + thao tác idempotent.

### Bốn quy tắc vàng

- **Giành key NGUYÊN TỬ** (SETNX / unique index) — đừng "check rồi set" hai bước (Insight #4).
- **Bọc xử-lý + ghi-dedup trong CÙNG transaction** nếu có thể → tránh "xử lý xong, crash trước khi ghi dedup → retry chạy lại".
- **Đặt retention key đủ dài** để bao retry, có TTL để khỏi phình.
- **Retry phải có backoff + jitter** (đừng retry bão hoà → retry storm; nối Cold Start). Và chỉ retry cái **idempotent** — retry mù cái non-idempotent là tự bắn chân.

### Cái khó nhất: tác dụng phụ NGOÀI database

Idempotency key bảo vệ ghi DB của *bạn*. Nhưng **gửi email / gọi API bên thứ ba / bắn webhook** thì sao? Gửi email hai lần thì DB dedup không cứu. Cách xử:
- **Đẩy side-effect qua một bảng/outbox idempotent** (mục ⑨ A2), đánh dấu "đã gửi" theo key.
- Hoặc dùng **nhà cung cấp hỗ trợ idempotency** (nhiều email/payment API có sẵn).
- Hoặc **chấp nhận at-least-once** ở chỗ vô hại (một email trùng hiếm khi khác một sự cố trừ tiền hai lần — *định lượng rủi ro*, như tư duy CAP).

---

## ⑨ Giải pháp NÂNG CAO (staff / principal)

### A1. Effectively-once đầu cuối = idempotent operation + dedup, KHÔNG phải exactly-once delivery

Đóng đinh tư duy: thiết kế **mọi mắt xích** (producer, broker, consumer, sink) theo **at-least-once + dedup**, không đi tìm "delivery đúng một lần". Đây là cách các pipeline nghiêm túc đạt "exactly once" *trên thực tế*.

### A2. Outbox pattern — giải "dual write" (ghi DB + phát event nguyên tử)

Vấn đề: ghi DB rồi phát Kafka là **hai thao tác** — crash giữa chừng → lệch. Giải: ghi **dữ liệu nghiệp vụ + bản ghi event vào bảng `outbox` trong CÙNG transaction**; một relay đọc outbox phát event (at-least-once); **consumer idempotent**. ⇒ không mất, không lệch, dù có trùng (vì idempotent).

### A3. Saga + compensating action, mỗi bước idempotent

Giao dịch phân tán dài (đặt vé + trừ tiền + cấp ghế) dùng **saga**: mỗi bước **idempotent** để retry an toàn; mỗi **bù trừ (compensation)** cũng **idempotent** (hoàn tiền hai lần không được phép → key hoá). Idempotency là *điều kiện sống còn* của saga.

### A4. Idempotency + fencing token (nối CAP/Coherence)

Khi có cả khoá phân tán: một worker **stale** (bị partition) có thể retry muộn. **Fencing token** (CAP doc) chặn token cũ ở storage; **idempotency key** chặn lặp tác dụng. Hai lớp cùng nhau cho an toàn dù split-brain.

### A5. Idempotency cho thao tác KHÔNG tất định

"Tạo resource với ID ngẫu nhiên" — retry sẽ sinh ID khác → không idempotent về kết quả. Giải: **chốt kết quả ở lần đầu và phát lại** (lưu `key → resource_id đã sinh`), hoặc cho **client cung cấp ID** (đẩy về PUT). Idempotency với phi-tất-định = *capture-and-replay* kết quả.

### A6. Liên hệ CRDT: idempotent + giao hoán + kết hợp (nối Eventual Consistency)

CRDT hội tụ được vì **merge** của chúng **idempotent + commutative + associative**: nhận cùng update nhiều lần / sai thứ tự vẫn ra cùng trạng thái. Idempotency là **một trong ba chân** của "Strong Eventual Consistency". Đây là idempotency ở tầng *cấu trúc dữ liệu*, không chỉ tầng request.

### A7. Bảo vệ chống retry storm (nối Cold Start)

Retry **không kiểm soát** = khuếch đại tải lúc sự cố → death spiral. Dùng **exponential backoff + jitter + giới hạn số lần + circuit breaker**. Idempotency làm retry *an toàn về tính đúng*, nhưng vẫn cần *kiểm soát về tải*.

---

## ⑩ Trường hợp MỞ RỘNG (idempotency ở mọi tầng)

| Bối cảnh | Idempotency thể hiện thế nào |
|---|---|
| **HTTP/REST API** | Method semantics (GET/PUT/DELETE idempotent, POST không) + Idempotency-Key header |
| **Message queue / event-driven** | At-least-once → **consumer idempotent** (dedup theo message-id) |
| **Payment / fintech** | Idempotency key bắt buộc (Stripe) — chống double-charge |
| **Database** | **UPSERT/MERGE**, unique constraint trên key |
| **Caching (Bài 1)** | **Back-fill** & **invalidate** idempotent → retry/at-least-once an toàn |
| **Infrastructure as Code** (Terraform/Ansible) | **Khai báo trạng-thái-mong-muốn** → apply lặp lại = cùng kết quả (idempotent theo thiết kế) |
| **Kubernetes** | **Reconciliation loop**: liên tục đưa thực-tế về desired-state → idempotent |
| **TCP / transport** | Sequence number dedupe gói truyền lại |
| **CRDT** | Merge idempotent + commutative + associative |
| **Webhook** | Gửi at-least-once → receiver dedup theo event-id |

> 🔑 **Insight #5:** Mảng **declarative** (IaC, K8s, PUT, UPSERT) **idempotent theo thiết kế** vì chúng mô tả *"trạng thái phải thế nào"* chứ không *"làm thêm thao tác gì"*. Mảng **imperative** (POST, +1, INSERT, "chạy script này") **không** idempotent và phải bọc dedup. ⇒ **chọn declarative khi có thể** là cách rẻ nhất để có idempotency.

---

## ⑪ Kiến thức MỞ RỘNG (kết nối cả cụm)

- ✅ **Back-fill idempotent (B2 Bài 1):** ca dùng gốc; ghi-gán-tuyệt-đối nên an toàn tự nhiên.
- ➕ **Eventual Consistency (chuyên đề):** idempotency là *điều kiện để retry trong hệ eventual không gây hại kép*; CRDT cần idempotent merge.
- ➕ **CAP (chuyên đề):** lúc partition client retry nhiều → idempotency + fencing giữ đúng đắn.
- ➕ **Cache Coherence (chuyên đề):** invalidate idempotent → phát at-least-once qua pub/sub/CDC mà không sợ.
- ➕ **Cold Start (chuyên đề):** retry storm — idempotency lo *đúng đắn*, backoff/circuit breaker lo *tải*.
- ➕ **At-least/at-most/exactly-once:** bậc thang delivery; exactly-once-delivery bất khả thi → effectively-once-effects qua idempotency.
- ➕ **Two Generals / FLP:** lý do nền tảng vì sao không có thoả thuận/delivery chắc chắn trên kênh không tin cậy → buộc dùng idempotency.
- ➕ **Outbox / Saga / dual-write:** các pattern messaging mà idempotency là trụ.
- ➕ **Declarative vs imperative:** declarative ⇒ idempotent gần như miễn phí.

---

## ⑫ Anti-patterns — AI & người mới hay sai

| ❌ Sai lầm | Vì sao nguy hiểm | ✅ Đúng |
|---|---|---|
| Tin có **exactly-once delivery** | Không tồn tại trên mạng không tin cậy | At-least-once + **idempotent dedup** = effectively-once |
| Dùng **POST** cho thao tác sẽ bị retry | Double-submit = double-order/charge | Idempotency key, hoặc PUT với ID client-sinh |
| Idempotency key nhưng **check-rồi-set hai bước** | Race: hai request song song cùng xử lý | **Giành key nguyên tử** (SETNX / unique index) |
| **Increment/append** trong pipeline at-least-once | Message trùng → +2, nhân đôi | Dedup theo event-id / CRDT / set tuyệt đối |
| Key **không có expiry** hoặc **quá ngắn** | Phình bộ nhớ / retry sau hết hạn lại xử lý | Retention đủ bao retry + có TTL |
| Quên **side-effect ngoài DB** (email, API) | DB dedup không cứu email gửi 2 lần | Outbox idempotent / nhà cung cấp hỗ trợ idem |
| Chỉ idempotent ở **happy path** | Retry lúc đang xử lý (`IN_PROGRESS`) chạy trùng | Máy trạng thái key (in-progress/done/failed) |
| **Retry mù** cái non-idempotent | Khuếch đại lỗi + tải | Chỉ retry cái idempotent; backoff+jitter; circuit breaker |
| Xử lý + ghi-dedup ở **hai transaction** | Crash giữa chừng → mất dedup → chạy lại | Bọc cùng transaction / outbox |

> 🔎 **AI hay sai (nối Bài 1):** AI hay viết handler "tạo đơn / trừ tiền" **không kèm idempotency**, và **retry mù** khi lỗi — chính là công thức double-charge. Luôn ép hỏi: *"Thao tác này có idempotent không? Nếu client retry (vì response mất), nó chạy mấy lần? Race hai request song song cùng key xử ra sao?"*

---

## ⑬ Khung quyết định / Playbook idempotency

### D-IDEM-1 — Một thao tác có cần "làm idempotent" không?
```
1. Chỉ ĐỌC? → idempotent tự nhiên (kể cả back-fill cache). Xong.
2. GHI gán-tuyệt-đối (PUT/set/UPSERT/DELETE)? → idempotent tự nhiên. Xong.
3. GHI tương-đối (POST/INSERT/+1/append/charge/email)? → PHẢI bọc:
     □ Idempotency key (client sinh, dùng lại khi retry).
     □ Giành key NGUYÊN TỬ (SETNX / unique index).
     □ Máy trạng thái: IN_PROGRESS → COMPLETED(kết quả) / FAILED.
     □ Bọc xử-lý + ghi-dedup cùng transaction.
     □ Retention key đủ dài + TTL.
     □ Side-effect ngoài DB? → outbox idempotent.
```

### D-IDEM-2 — Thiết kế retry an toàn
```
□ Chỉ retry thao tác IDEMPOTENT (hoặc đã bọc key).
□ Exponential backoff + JITTER (đừng retry storm — Cold Start).
□ Giới hạn số lần + circuit breaker.
□ Phân biệt lỗi nên-retry (timeout/5xx) vs không (4xx).
```

### D-IDEM-3 — Pipeline messaging effectively-once
```
□ Giả định at-least-once (sẽ có trùng) — KHÔNG tin exactly-once delivery.
□ Producer: idempotent (sequence/key) nếu có.
□ Consumer: dedup theo message-id + thao tác idempotent.
□ Dual-write (DB + event)? → outbox trong cùng transaction.
```

---

## ⑭ Mental model (câu thần chú gốc)

> 🔁 **Idempotent = làm một lần hay nhiều lần, tác dụng vẫn như một** (công tắc BẬT, không phải nút +1).
>
> **Exactly-once *delivery* là huyền thoại** — mạng sẽ làm message tới hai lần. Thứ bạn đạt được là **effectively-once *effects*** nhờ **idempotency + dedup**.
>
> Bốn câu phải thuộc:
> 1. **Ưu tiên gán-tuyệt-đối** (PUT/UPSERT/declarative) → idempotent gần như miễn phí.
> 2. **Không tránh được → idempotency key**, giành key **nguyên tử** (SETNX/unique index).
> 3. **Side-effect ngoài DB** (email/API) cần dedup riêng (outbox).
> 4. **Retry chỉ an toàn nếu idempotent** — và vẫn cần backoff/jitter để khỏi storm.

---

## ⑮ Câu hỏi phỏng vấn & thách đố

- **(dễ)** Idempotency là gì? → Làm một thao tác nhiều lần cho cùng kết quả/trạng thái cuối như làm một lần.
- **(bẫy)** Idempotent có nghĩa "response luôn giống nhau"? → Không. Về **tác dụng phụ chỉ xảy ra một lần**; DELETE lần 2 trả 404 nhưng trạng thái cuối vẫn idempotent.
- **(dễ)** Method HTTP nào idempotent? → GET/HEAD/OPTIONS/TRACE (safe) + PUT + DELETE. **POST không**, PATCH tuỳ.
- **(TB)** Vì sao cần idempotency dù đã có retry? → Vì response có thể mất → client retry → server chạy lần hai; không có idempotency = double effect.
- **(bẫy)** Có exactly-once delivery không? → Không trên mạng không tin cậy. Đạt **effectively-once effects** bằng at-least-once + idempotency/dedup. "Exactly-once" của Kafka = idempotent producer + transaction.
- **(TB)** Idempotency key hoạt động ra sao, race ở đâu? → Client gửi UUID, server dedup `key→result`; race khi hai request song song cùng key → giành key nguyên tử (SETNX/unique index) + trạng thái IN_PROGRESS.
- **(khó)** Back-fill cache có idempotent không, vì sao? → Có — ghi gán-tuyệt-đối cùng giá trị; nhiều request back-fill vẫn đúng (singleflight chỉ để giảm tải, không phải để đúng).
- **(khó)** Side-effect ngoài DB (gửi email) làm sao idempotent? → Dedup ở tầng email (đánh dấu đã-gửi theo key/outbox) hoặc nhà cung cấp hỗ trợ idem; hoặc chấp nhận rủi ro nếu vô hại.

> 🧩 **Thách đố:** *"Hệ trừ tiền ví qua một message queue at-least-once. Thỉnh thoảng user bị trừ hai lần. Phân tích & sửa đầy đủ."*
> → Queue at-least-once → message trừ-tiền **giao trùng** → consumer **chạy hai lần** (vì "trừ $X" non-idempotent). Sửa: (1) **idempotency key = transaction-id**, consumer **dedup nguyên tử** (unique index trên `txn_id`, hoặc SETNX) trước khi trừ; (2) **bọc dedup + trừ tiền trong CÙNG DB transaction** (tránh crash-giữa-chừng); (3) thiết kế trừ-tiền thành **ghi "đã áp dụng txn_id"** (gán tuyệt đối) thay vì "-X" thuần; (4) retry có **backoff/jitter**; (5) nếu có gọi cổng thanh toán ngoài → dùng **idempotency key của cổng** (Stripe-style). Đóng đinh: *không* đi tìm "exactly-once delivery" — đạt **effectively-once** bằng dedup + idempotent.

---

## ⑯ Bài tập + tiêu chí tự chấm

**BT1.** Phân loại 8 thao tác (GET số dư, POST tạo đơn, PUT đặt avatar, DELETE bài viết, "+1 like", UPSERT tồn kho, gửi email xác nhận, INSERT log) theo idempotent / không, và nêu cách làm idempotent cái nào chưa.
> ✅ **Đạt khi:** GET/PUT/DELETE/UPSERT→idempotent; POST/+1/email/INSERT→không, kèm cách bọc (key/dedup/event-id/outbox).

**BT2.** Viết handler `chargeWallet` an toàn với retry: chỉ ra điểm giành-key nguyên tử, transaction, và trạng thái IN_PROGRESS.
> ✅ **Đạt khi:** SETNX/unique-index trên txn-id; trừ tiền + ghi dedup cùng transaction; xử ca đang-xử-lý.

**BT3.** Giải thích vì sao "exactly-once delivery" bất khả thi nhưng Kafka quảng cáo "exactly-once". Hai cái này hoà giải ra sao?
> ✅ **Đạt khi:** delivery không thể đúng-một-lần (Two Generals); Kafka đạt effectively-once bằng idempotent producer (seq number dedup) + transaction; đầu cuối cần sink idempotent.

**BT4.** Cho pipeline "DB write + publish event". Mô tả dual-write problem và cách outbox + consumer idempotent giải quyết.
> ✅ **Đạt khi:** chỉ ra crash giữa hai ghi gây lệch; outbox ghi event cùng transaction; relay phát at-least-once; consumer dedup idempotent.

---

## ⑰ Glossary (đóng vào hoàn cảnh)

| Thuật ngữ | Nghĩa nhanh | 🎬 Hoàn cảnh sử dụng |
|---|---|---|
| **Idempotency** | Làm nhiều lần = làm một lần (về tác dụng) | *"Back-fill phải idempotent để retry an toàn."* |
| **At-least-once** | Giao ít nhất một lần (có thể trùng) | *"Queue at-least-once → consumer phải idempotent."* |
| **Exactly-once (delivery)** | Đúng một lần — **bất khả thi** trên mạng | *"Đừng đi tìm exactly-once delivery."* |
| **Effectively-once (effects)** | Tác dụng đúng một lần nhờ dedup | *"Đạt effectively-once bằng idempotency + dedup."* |
| **Idempotency key** | UUID định danh một ý định, dùng lại khi retry | *"Stripe: gửi Idempotency-Key chống double-charge."* |
| **Dedup store** | Nơi lưu `key → kết quả` để chặn lặp | *"Redis/DB làm dedup store cho idempotency key."* |
| **SETNX / unique index** | Giành key nguyên tử chống race | *"SETNX để hai request song song không cùng xử lý."* |
| **UPSERT / MERGE** | Ghi theo khoá, lặp ra cùng trạng thái | *"INSERT ON CONFLICT = ghi idempotent."* |
| **Safe method** | Không đổi state (GET…) | *"Safe ⊂ idempotent."* |
| **Outbox pattern** | Ghi event cùng transaction nghiệp vụ | *"Outbox giải dual-write, consumer idempotent."* |
| **Saga / compensation** | Giao dịch dài, mỗi bước idempotent | *"Bù trừ cũng phải idempotent (key hoá)."* |
| **Two Generals** | Bất khả thi thoả thuận trên kênh không tin | *"Vì Two Generals nên cần idempotency."* |
| **Retry storm** | Retry không kiểm soát khuếch đại tải | *"Backoff + jitter chống retry storm."* |
| **Declarative (desired state)** | Mô tả trạng-thái → idempotent theo thiết kế | *"Terraform/K8s apply lặp lại = cùng kết quả."* |

---

## ⑱ Đọc thêm

- **RFC 9110 (HTTP Semantics)** — định nghĩa safe & idempotent methods.
- **Stripe API docs — "Idempotent Requests"** — pattern idempotency key chuẩn mực.
- **Kafka docs — "Idempotent Producer" & "Exactly-Once Semantics"** — seq number dedup + transaction.
- **AWS — SQS at-least-once & FIFO deduplication** — idempotent consumer thực chiến.
- **"Transactional Outbox" & "Saga" patterns (microservices.io — Chris Richardson)**.
- **Two Generals' Problem** & **Nakamoto/FLP** — vì sao delivery chắc chắn bất khả thi.
- **Martin Kleppmann, *Designing Data-Intensive Applications*** — chương "The Trouble with Distributed Systems" & exactly-once.
- **Pat Helland, "Idempotence Is Not a Medical Condition"** — bài viết kinh điển, dễ đọc.

---

> 🔗 **Nối lại cả cụm (7 tài liệu):** Bài 1 dựng khung *tốc độ ↔ tươi ↔ RAM* và nhắc *"back-fill phải idempotent để retry an toàn"*. Idempotency là **chất keo làm mọi cơ chế kia dùng được trong thực tế không tin cậy**: nó khiến **retry** (Eventual Consistency), **invalidate at-least-once** (Cache Coherence), **xử lý lúc partition** (CAP), và **back-fill khi cache lạnh** (Cold Start) đều **an toàn dù lặp lại**. Câu chốt: ***bạn không ngăn được thế giới gọi lại thao tác của mình — bạn chỉ thiết kế để lần gọi thứ hai không gây thêm hại. Đó là idempotency, và nó là điều kiện để mọi hệ "thử lại cho chắc" không biến thành "làm hại hai lần".***
