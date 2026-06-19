# 🎯 BÀI 1 — Vì sao Messaging: sync vs async & coupling

**Phủ:** K-WHY-001 → 008

> 💡 **Mục tiêu một câu:** Sau bài này, bạn trả lời được câu gốc nhất của cả Mục K — *"Thêm một **message broker** vào hệ thống để giải bài toán gì, và phải trả giá gì?"*

---

## ① Mục tiêu & vị trí trong mạch

Đây là bài **mở đầu Mục K**. Nó là cái móng. Nếu cái móng này lung lay, mọi bài sau sẽ nghiêng theo hai kiểu hỏng đối nghịch nhau:

- **Hỏng kiểu "nghiện queue":** không hiểu *vì sao* async, bạn sẽ nhét **queue** vào *mọi* chỗ cho "ngầu/scalable" → đây là **anti-pattern** (mục ② phần c sẽ mổ xẻ).
- **Hỏng kiểu "sợ async":** ngược lại, sợ độ phức tạp của async nên cố giữ *mọi thứ* **sync**, các service gọi nhau dây chuyền qua HTTP → khi một mắt xích chậm/chết, cả chuỗi sập theo (**cascade failure**, gặp lại ở Bài 9, 10).

> 🎯 **Chốt vị trí:** Bài này dạy bạn *quyết định*, không phải *công cụ*. Học xong bạn biết **khi nào nên async, khi nào tuyệt đối đừng**. Công cụ (broker nào) là chuyện của Bài 2; còn "async không miễn trừ bạn khỏi xử lý lỗi" là chuyện của Bài 9.

**Nối tới:**
- **Bài 2** — chọn **broker** nào (Kafka / RabbitMQ / SQS...).
- **Bài 6** — **dual-write problem** & **Outbox pattern** (bài này sẽ gài sẵn một cái bẫy cho bạn thấy).
- **Bài 9** — async *không* miễn trừ **resilience** (retry, timeout, DLQ).

---

## ② Giảng cơ bản → nâng cao

### (a) Trực giác trước đã

> 📞 **Ẩn dụ — Sync = gọi điện thoại.**
> Hai bên *phải cùng online*. Bạn nói xong thì **đứng im chờ** đầu kia trả lời mới làm tiếp được. Nếu họ bận, máy bận, hoặc rớt sóng → bạn kẹt cứng ở đó.

> 📮 **Ẩn dụ — Async qua broker = gửi thư qua bưu điện.**
> Bạn bỏ thư vào **hòm thư (broker)** rồi *đi làm việc khác ngay*. Người nhận lấy thư **khi nào họ rảnh**. Nếu họ đi vắng cả tuần, lá thư **vẫn nằm yên trong hòm**, không mất. Bưu điện chính là **broker** — nó **tách rời thời gian** giữa lúc *gửi* và lúc *nhận*.

Hai ẩn dụ này chứa gần hết tinh thần của Mục K. Toàn bộ phần dưới chỉ là *bóc tách* xem "bưu điện" giúp được những gì và lấy của bạn cái gì để đổi lại.

---

### (b) Cơ chế chi tiết

#### 📘 K-WHY-001 — 5 lớp bài toán mà message queue giải

Đây là **danh sách phải thuộc**. Mỗi khi định dùng queue, bạn phải chỉ tay vào *ít nhất một* trong 5 lớp này; nếu không chỉ được → đừng dùng (xem K-WHY-005).

**1. Decoupling (gỡ phụ thuộc).**
**Producer** (bên gửi) *không cần biết* **consumer** (bên nhận) là ai, ở đâu, có bao nhiêu cái. Nó chỉ cần biết tên **topic/queue**.
> *Ví dụ trực giác:* Đài phát thanh phát ở tần số 99.9 MHz. Đài *không cần biết* có 3 hay 3 triệu người đang nghe, ai đang nghe, ngồi đâu. Thêm một người nghe mới → đài không phải làm gì cả.
> Hệ quả: đổi/thêm consumer **không đụng** tới code producer.

**2. Async / giảm latency *cảm nhận*.**
Trả **response** cho user *ngay* sau khi *nhận* yêu cầu, đẩy phần xử lý nặng cho **worker** làm sau.
> *Ví dụ số:* Gửi 1 email qua **SMTP** mất ~**800ms**, render 1 video mất ~**30s**. Nếu bắt user chờ đủ → trang treo 30 giây. Nếu chỉ ghi nhận yêu cầu (~**20ms**) rồi đẩy việc cho worker → user thấy phản hồi *gần như tức thì*.
> ⚠️ Lưu ý chữ **"cảm nhận"**: tổng công việc *không* ít đi, chỉ là user không phải *đứng chờ* nó.

**3. Buffering / load-leveling (san tải).**
**Broker** hấp thụ **spike** (đỉnh tải đột ngột).
> *Ví dụ số (flash-sale):* **10.000 req/s** ập tới trong 1 phút, nhưng downstream chỉ xử lý nổi **1.000/s**. Không có queue → downstream *sập*. Có queue → nó giữ phần dư (9.000/s), downstream **rút theo nhịp của chính nó**.
> 🔍 **Ẩn dụ kỹ hơn:** broker đóng vai **shock absorber** (giảm xóc xe). Mặt đường xóc (tải tăng vọt) nhưng người ngồi trong xe (downstream) vẫn êm.

**4. Độ bền — durability.**
Message được **persist** (ghi xuống đĩa). Consumer *chết giữa chừng* → message **không mất**, được xử lý lại khi consumer sống dậy.
> *Điều tệ nhất nếu không có:* mất đơn hàng đã thanh toán vì service tính phí restart đúng lúc đang xử lý → tiền đã trừ, hàng không giao.

**5. Fan-out (rẽ nhánh).**
**1 event → N consumer độc lập.**
> *Ví dụ e-commerce:* một sự kiện `OrderPlaced` được **4 service** nghe cùng lúc: **kho** (trừ tồn), **email** (gửi xác nhận), **analytics** (đếm doanh thu), **audit** (ghi log tuân thủ). Thêm consumer thứ 5 → **không sửa** producer.

---

#### 📘 K-WHY-003 — Phân loại Coupling (đây là phần "vì sao" cốt lõi)

**Coupling** = mức độ *dính chặt* giữa hai thành phần. Càng dính, đổi cái này càng dễ vỡ cái kia. Có hai loại coupling mà async *gỡ được*, và một loại mới mà async *tạo ra* — không có bữa trưa miễn phí.

| Loại coupling | Nghĩa | Sync? | Async qua broker? |
|---|---|---|---|
| **Temporal coupling** (dính *thời gian*) | Hai bên **phải cùng online một lúc** | ❌ Có (dính chặt) | ✅ **Gỡ được** — consumer offline vẫn nhận sau |
| **Spatial coupling** (dính *không gian*) | Phải biết **địa chỉ** của nhau (IP/host/instance) | ❌ Có (dính chặt) | ✅ **Gỡ một phần** — chỉ cần biết tên *topic*, không cần IP |
| **Contract coupling** (dính *hợp đồng*) | Phải đồng ý về **schema của event** + phụ thuộc vào *chính broker* | (ít) | 🚨 **Tạo MỚI** — đây là cái giá bạn trả |

> ⚠️ **Khẩu quyết:** *Async không xoá coupling — nó **đổi loại coupling**.* Bạn trả lại **temporal + spatial coupling**, nhưng nhận về **coupling vào schema event và vào broker**. Đổi một bài toán dễ vỡ lúc-chạy lấy một bài toán cần kỷ luật-thiết-kế (versioning schema, vận hành broker).

🔎 *Vì sao điều này quan trọng?* Nhiều người tưởng "dùng queue = hết phụ thuộc". Sai. Ngày bạn đổi field trong `OrderPlaced` mà quên 1 trong 4 consumer → vỡ âm thầm. Đó chính là **contract coupling** đòi nợ.

---

#### 📘 K-WHY-004 — Command vs Event (phân biệt sống còn)

> 🧠 **Mental model:** **Command** là *"Hãy làm X"* (ra lệnh, có người chịu trách nhiệm). **Event** là *"X đã xảy ra rồi"* (thông báo, ai quan tâm thì nghe).

| Tiêu chí | **Command** | **Event** |
|---|---|---|
| Ngữ nghĩa | "Hãy làm X" (ra lệnh) | "X đã xảy ra" (thông báo) |
| Người nhận | **một owner cụ thể** | bất kỳ ai quan tâm (**fan-out**) |
| Cách đặt tên | mệnh lệnh — `ChargePayment` | quá khứ — `PaymentCharged` |
| Coupling | producer **biết** ai xử lý | producer **không quan tâm** ai xử lý |
| Mong đợi | có **side-effect**, có thể trả kết quả | không mong đợi gì cụ thể |

**Hệ quả thiết kế (vì sao phải phân biệt):**
- **Command** → coupling *cao hơn* (vì biết owner) nhưng **kiểm soát tốt**: rõ ai làm, dễ truy trách nhiệm.
- **Event** → **decouple cao nhất**, nhưng cái giá là *"không ai chịu trách nhiệm rõ ràng"* → logic nghiệp vụ bị **phân tán** ra nhiều nơi, khó lần.

> 🔍 **Quy tắc đặt tên là kim chỉ nam:** nếu bạn lỡ đặt một "event" bằng *động từ mệnh lệnh* (`SendEmail`), 90% là bạn đang ngụy trang một **command** thành event → producer thực ra *vẫn* phụ thuộc ngầm vào một consumer cụ thể. Đặt tên quá khứ (`UserRegistered`) ép bạn suy nghĩ đúng.

---

#### 📘 K-WHY-006 — Point-to-point vs Pub/Sub

Đây là *hai kiểu giao hàng* của broker. Nhận diện đúng nhu cầu thì chọn đúng kiểu.

| Tiêu chí | **Point-to-point (P2P)** | **Pub/Sub** |
|---|---|---|
| Mỗi message tới... | **đúng MỘT** consumer | **MỌI** subscriber |
| Tên dân dã | **work queue** (hàng đợi việc) | **broadcast** (phát quảng bá) |
| Dùng để | **chia tải** một loại job | **báo tin** cho nhiều bên |
| Ví dụ | 100 job render video chia cho 10 worker | `OrderPlaced` báo cho 4 phòng ban |

> 🎯 **Cách nhận diện nhanh:**
> - Nhu cầu là *"chia việc cho đỡ tải"* → **P2P**.
> - Nhu cầu là *"một chuyện xảy ra, nhiều bên cần biết"* → **pub/sub**.

```
P2P (work queue) — mỗi job tới ĐÚNG 1 worker:
                 ┌──► Worker A  (xử lý job #1)
  [Queue] ───────┼──► Worker B  (xử lý job #2)
                 └──► Worker C  (xử lý job #3)
   → chia tải, KHÔNG trùng

Pub/Sub — 1 event tới TẤT CẢ subscriber:
                 ┌──► Inventory  (nghe bản sao)
  OrderPlaced ───┼──► Email      (nghe bản sao)
                 ├──► Analytics  (nghe bản sao)
                 └──► Audit       (nghe bản sao)
   → fan-out, MỖI bên nhận 1 bản
```

---

### (c) Mép giới hạn & sai lầm

#### 📘 K-WHY-005 — "Cho mọi thứ qua queue" là anti-pattern

> 🚨 **Cấm:** đừng nhét async vào chỗ user cần **read-after-write tức thì**.

*Ví dụ trực giác:* user bấm "Đặt hàng" rồi muốn **xem ngay** đơn vừa tạo. Nếu việc tạo đơn bị đẩy async → user bấm xem mà *chưa thấy gì* (vì đơn còn nằm trong queue) → hoảng, bấm lại → tạo *hai* đơn.

**Cái giá của việc async-hoá *bừa*:**
- thêm **latency** (qua broker mất thêm chặng),
- thêm **hạ tầng** phải vận hành (broker, worker, monitoring),
- nhận thêm **eventual consistency** (mục dưới),
- **debug khổ** (luồng không còn tuyến tính, lỗi xảy ra ở chỗ khác lúc khác).

> 🎯 **Nguyên tắc vàng:** *Default phải là **sync**. **Async** là một quyết định **có chủ đích**, phải chỉ được ra ít nhất 1 trong 5 lớp K-WHY-001.*

---

#### 📘 K-WHY-007 — Async đổi *mô hình nhất quán*

Async-hoá *một bước* sẽ kéo phần đó từ **strong consistency** → **eventual consistency**.

*Ví dụ cụ thể:* bạn async-hoá bước "gửi email khi đặt hàng". Bây giờ user thấy màn hình *"Đặt hàng thành công!"* **trước khi** email thực sự được gửi đi. Có một khoảng thời gian mà *sự thật của hệ thống* và *cái user thấy* **chưa khớp**.

```
TIMELINE — async-hoá "gửi email":
 T1  User bấm Đặt hàng
 T2  API ghi Order (status = PENDING)        ✅ DB đã có đơn
 T3  API publish OrderPlaced → broker         ✅ "đã ghi nhận"
 T4  API trả 201 "Đặt hàng OK"  ───────────►  user THẤY thành công
        ▲ ở đây email VẪN CHƯA gửi ❌
 T5  EmailSvc rảnh, lấy message, gửi SMTP
 T6  SMTP lỗi → retry... (user không hề biết)
 T7  Gửi xong → email_verified / mail_sent = true ✅ giờ mới thật khớp
```

> 🚨 **Sai lầm chết người:** coi *"publish xong = đã xong"*. Không! **Publish chỉ là "đã ghi nhận"**; trạng thái *thật* vẫn là **PENDING** cho tới khi consumer xử lý xong.

**Vì sao điều này bắt bạn thiết kế thêm:**
- phải có **trạng thái trung gian** (`PENDING` / `PROCESSING` / `CONFIRMED`) để phản ánh sự thật,
- phải có **retry** khi consumer lỗi,
- phải có **cơ chế báo lỗi muộn** (nếu sau N lần vẫn fail thì báo user/ops thế nào).

---

## ③ ⚠️ Kiến thức cũ / bị thay thế

| Cái cũ (phổ biến) | Thay bằng (hiện nay) | Vì sao |
|---|---|---|
| **"Microservice = phải gọi nhau qua HTTP REST"** | **Sync** chỉ cho path *cần phản hồi ngay*; **event-driven** cho phần còn lại | REST-dây-chuyền = **distributed monolith** + **cascade failure** (Bài 9, 10) |
| **"Queue chỉ để chạy nhanh hơn"** | Queue = **decouple + buffer + durability + fan-out** (tốc độ chỉ là **1 trong 5**) | Hiểu sai mục đích → đặt sai chỗ (K-WHY-005) |
| **"Publish event = việc đã hoàn tất"** | Publish = *"đã ghi nhận"*; trạng thái thật vẫn **PENDING** tới khi consumer xong | **Eventual consistency** (K-WHY-007) |

---

## ④ Áp dụng thực tế + So sánh bigtech

**Use case thật — đặt hàng e-commerce.** Tách rạch ròi cái gì sync, cái gì async:

| Bước | Sync hay Async? | Vì sao |
|---|---|---|
| Tạo **Order** trong DB | ✅ **Sync** | User cần `orderId` *ngay* để xem đơn (read-after-write) |
| Trừ **kho** (inventory) | **Async** | Không cần chặn user; có thể bù trừ sau |
| Tính **phí / charge** | **Async** | Xử lý nặng, cần retry riêng |
| Gửi **mail xác nhận** | **Async** | SMTP chậm (~800ms), không nên bắt user chờ |
| Cập nhật **analytics** | **Async** | Hoàn toàn không ảnh hưởng trải nghiệm đặt hàng |

**Kiến trúc tối thiểu:**

```
                                   ┌──► InventorySvc  (trừ kho)
  API ──► tạo Order (DB) ──► publish OrderPlaced ──┼──► EmailSvc      (gửi mail)
        (sync, trả 201)        (pub/sub, fan-out)  └──► AnalyticsSvc  (đếm doanh thu)
                                                  → 3 consumer ĐỘC LẬP, thêm bớt không sửa API
```

**Bigtech (pattern phổ biến — *không* khẳng định tuyệt đối, vì hạ tầng đổi nhanh):** các hệ quy mô lớn (thương mại điện tử, ride-hailing, fintech) thường chạy một **"spine" event-driven** (hay trên **Kafka** hoặc broker tương đương) để phục vụ **fan-out + analytics + CDC**, đồng thời **giữ sync** cho những call cần độ tươi. Netflix/Uber nổi tiếng với kiến trúc event-driven nội bộ.

> 🔍 **verify khi cần:** nếu bạn cần *con số/tên hệ thống cụ thể* để trích dẫn, hãy kiểm chứng lại từ nguồn chính thức — ở đây chỉ nêu *pattern* mức nguyên lý.

---

## ⑤ Code thực hành + cấu hình

Minh hoạ tinh thần *"trả response ngay, xử lý sau"* bằng pseudo-Node (chưa cần broker thật — Bài 2 mới chọn broker):

```javascript
// orderController.js — KHÔNG chờ email/inventory đồng bộ
//
// ❌ AI hay viết: gọi sync emailService.send() NGAY trong handler
//    → user phải chờ trọn vòng SMTP (~800ms) mới thấy phản hồi.
// ✅ Đúng kiến trúc: ghi DB → publish event → trả 201/202 NGAY.

async function placeOrder(req, res) {
  // (1) Ghi đơn với TRẠNG THÁI TRUNG GIAN — sự thật lúc này là "chưa xong"
  const order = await db.orders.insert({ ...req.body, status: 'PENDING' });

  // (2) Phát event fan-out cho mọi bên quan tâm (kho/mail/analytics)
  //     ⚠️ Bài 6: publish NGAY SAU insert ở đây CHƯA an toàn — đó là dual-write!
  await broker.publish('OrderPlaced', { orderId: order.id, ...order });

  // (3) Trả về NGAY: user thấy PENDING, KHÔNG đứng chờ email
  res.status(201).json({ orderId: order.id, status: 'PENDING' });
}
```

> 🚨 **Bẫy đã gài sẵn (cố ý):** dòng `broker.publish(...)` *ngay sau* `db.orders.insert(...)` chính là **dual-write problem** — ghi vào *hai* hệ thống (DB + broker) mà *không* có gì đảm bảo cả hai cùng thành công. Nếu app chết *giữa* hai dòng → DB có đơn nhưng event *không* phát ra (hoặc ngược lại). **Bài 6** sẽ vá bằng **Outbox pattern**. Để đây để bạn thấy tận mắt: *"code chạy được nhưng sai kiến trúc"*.

**Cấu hình ở bài này:** chưa cần broker. Chỉ cần thống nhất *quy ước trạng thái* trong schema đơn:

```
PENDING ──► CONFIRMED ──► ... (các trạng thái sau)
   ▲ đây là "trạng thái trung gian" bắt buộc khi đã async-hoá
```

---

## ⑥ Keywords cần nhớ

**🧠 Cần HIỂU (nắm bản chất):**
- **decoupling** — gỡ phụ thuộc producer↔consumer.
- **temporal coupling** vs **spatial coupling** — dính thời gian vs dính địa chỉ.
- **load-leveling / buffering** — broker làm shock absorber cho spike.
- **fan-out** — 1 event → N consumer độc lập.
- **eventual consistency** — sự thật và cái-user-thấy khớp *sau cùng*, không tức thì.
- **"khi nào KHÔNG dùng queue"** — read-after-write tức thì, không chỉ ra được 1 trong 5 lớp.

**📌 Cần THUỘC (đọc là bật ra ngay):**
- **5 lớp bài toán MQ:** decoupling, async, buffering, durability, fan-out.
- **command** (mệnh lệnh, 1 owner, tên hiện tại) vs **event** (quá khứ, fan-out).
- **P2P** (chia việc) vs **pub/sub** (báo nhiều bên).

**🛠️ Cần LÀM ĐƯỢC:**
- Nhìn một flow → quyết **sync** hay **async** + **nêu lý do** (chỉ vào 1 trong 5 lớp).
- Thiết kế **trạng thái trung gian** (`PENDING`...) khi async-hoá một bước.

---

## ⑦ Mental model

> 🧠 **"Sync = gọi điện** (cùng online, đứng chờ máy); **async = gửi thư** (broker giữ giùm).
> **Queue KHÔNG phải để nhanh** — để **decouple / buffer / bền / fan-out**.
> **Async-hoá = đổi strong lấy eventual** → bắt buộc phải có **trạng thái PENDING**."

---

## ⑧ Câu hỏi phỏng vấn & thách đố

**(dễ)** Kể ≥3 bài toán **message queue** giải.
→ Gợi ý: **decouple, async, buffer, durability, fan-out**.

**(TB)** **Sync vs async** đánh đổi gì, chọn theo tiêu chí nào?
→ Hỏi: *Cần phản hồi tức thì không? Chịu được eventual không? Có spike tải không?*

**(khó)** **Temporal vs spatial coupling** — async gỡ loại nào, *tạo* coupling gì mới?
→ Gỡ **temporal** + một phần **spatial**; **tạo mới** coupling vào **schema event** + vào **broker** (contract coupling).

**(khó)** **Command vs event** khác gì về **coupling** và *ai sở hữu logic*?
→ Command: coupling cao hơn (biết owner), logic tập trung, dễ truy trách nhiệm. Event: decouple cao nhất nhưng logic *phân tán*, "không ai chịu trách nhiệm rõ ràng".

**(rất khó)** Async-hoá *"gửi email khi đặt hàng"* — mô hình nhất quán đổi thế nào, user thấy gì?
→ **strong → eventual**; user thấy **PENDING / "Đặt hàng OK"** *trước khi* email thật sự gửi; cần **retry** + **báo lỗi muộn**.

> 🧩 **Thách đố:** *"Sếp bảo: cứ cho **tất cả** call nội bộ qua **Kafka** cho **'scalable'**. Bạn phản biện sao?"*
> → **Đáp:** async thêm **latency** + **eventual consistency** + **hạ tầng** vận hành + **debug khó**. Call cần **read-after-write** / phản hồi ngay phải **giữ sync**. Queue chỉ dành cho chỗ chỉ ra được lý do (**decouple/buffer/fan-out/bền**). *"Scalable"* tự nó **không phải** một lý do — phải chỉ vào 1 trong 5 lớp K-WHY-001.

---

## ⑨ Bài tập + tiêu chí tự chấm

**Đề:** Cho luồng *"đăng ký user → gửi email verify → tặng coupon chào mừng"*.
Hãy: (1) vẽ phần nào **sync**, phần nào **async**; (2) đặt tên **event** (thì *quá khứ*); (3) chỉ ra **trạng thái trung gian**.

> ✅ **Đạt khi:**
> - Tạo user là **sync** (cần `userId` *ngay*).
> - Gửi **email verify** + tặng **coupon** là **async** qua event **`UserRegistered`**.
> - Có **trạng thái trung gian** rõ ràng, ví dụ `email_verified = false`.
> - Nêu được **rủi ro** nếu coi *"publish = đã gửi"* (email có thể chưa gửi xong / fail mà user đã thấy "đăng ký thành công").

*Tự kiểm thêm:* nếu bạn đặt tên là `SendVerifyEmail` (mệnh lệnh) thay vì `UserRegistered` (quá khứ) → bạn đang viết **command** trá hình, producer lại dính vào một consumer cụ thể (xem K-WHY-004).

---

## ⑩ Đọc thêm

- **Enterprise Integration Patterns** (Hohpe & Woolf) — kinh điển về **messaging patterns**.
- **Microsoft Learn** / **AWS Prescriptive Guidance**: *"messaging patterns"*, *"choreography vs orchestration"* (khái niệm nền, ổn định).

> 🎯 **Một câu mang về:** *Đừng hỏi "có nên dùng queue không"; hãy hỏi "bước này dính lý do nào trong 5 lớp — và mình có chịu nổi eventual consistency cho nó không".*
