# 🎯 BÀI 7 — Saga: orchestration vs choreography & compensation

**Phủ:** K-SAGA-001 → 009

> 💡 **Mục tiêu một câu:** **Saga** là cách đạt nhất quán **xuyên nhiều service** khi *không có transaction chung*. Học xong, bạn thiết kế được một **checkout saga** đúng — và quan trọng hơn, **gọi tên được những chỗ chết người** mà code happy-path che giấu.

---

## ① Mục tiêu & vị trí trong mạch

**Saga** giải bài toán: mỗi service có **DB riêng** (**database-per-service** — Bài 10) → **không có ACID transaction chung** để bọc cả luồng "trừ kho + tính tiền + xác nhận đơn".

> 🎯 **Saga đứng trên mọi thứ đã học:** nó dùng **Outbox (Bài 6)** + **idempotent consumer (Bài 5)** làm **baseline**. Nếu hai cái nền này yếu, saga của bạn sẽ trừ tiền hai lần và oversell mà không ai biết tại sao.

> 🔍 **Cross-ref H-CQRS** (Saga kiểu Nest/RxJS) — *đừng nhầm* với distributed saga (xem mục (e)).

---

## ② Giảng cơ bản → nâng cao

### (a) Vì sao Saga 📘 K-SAGA-001

> 🍽️ **Ẩn dụ — bữa tiệc nhiều bếp:** một đơn đặt tiệc cần *bếp lạnh* làm salad, *bếp nóng* làm món chính, *bếp bánh* làm tráng miệng. Không có **một** cái nồi chung để "nấu hết hoặc huỷ hết". Nếu bếp bánh cháy, bạn không thể "rút lại" salad đã làm — bạn phải **đem cho khác / bỏ đi** (hành động *ngược* trong một thế giới đã thay đổi).

Mỗi service có **DB riêng** → không có **ACID transaction** chung. **2PC** thì **khoá + kém chịu lỗi** (Bài 6). Vậy:

> 🎯 **Saga = chuỗi local transaction**, mỗi bước **commit riêng** ở service của nó; nếu một bước **fail** → chạy **compensating transaction** (giao dịch bù) **hoàn tác** các bước trước → đạt nhất quán **eventual** **không cần lock toàn cục**.

```
Saga happy-path:
  [reserve inventory] ✅ → [charge payment] ✅ → [confirm order] ✅
  mỗi bước = 1 local transaction commit riêng ở service của nó

Saga khi bước 2 fail:
  [reserve inventory] ✅ → [charge payment] ❌
        │                         │
        ▼ compensate NGƯỢC        │
  [release inventory] ◄───────────┘
  → quay về trạng thái nhất quán (eventual), KHÔNG cần lock toàn cục
```

---

### (b) Hai kiểu điều phối 📘 K-SAGA-002, 003

| Tiêu chí | **Orchestration** | **Choreography** |
|---|---|---|
| Điều phối | **1 orchestrator** trung tâm ra lệnh từng bước, lưu **state** | mỗi service **phát event**, service khác **phản ứng** |
| State | **tập trung**, dễ nhìn / dễ test | **phân tán**, khó theo dõi |
| Coupling | service **biết orchestrator** | **decouple cao** |
| Hợp với | saga **dài/phức tạp** (5+ bước, **branching**, conditional) | saga **ngắn** (2–4 bước), event flow rõ |
| Công cụ | **Temporal**, **Camunda 8/Zeebe**, **Orkes Conductor** | **broker thuần** (Kafka/Rabbit) |

> 📞 **Ẩn dụ:** **orchestration** = một *nhạc trưởng* chỉ huy từng nhạc công. **choreography** = các *vũ công* không có người chỉ huy, ai thấy bạn nhảy bước nào thì nhảy bước tiếp theo của mình.

> 🎯 **Chọn 📘 K-SAGA-003:** ⚠️ **đừng máy móc "đơn giản → choreography"**. Xét **cognitive load** + **visibility**: choreography khi saga **ngắn** & team quen **event-driven**; orchestration khi cần **visibility / branching / test**. **Hệ lớn thường dùng cả hai.** (🔎 *verify Temporal/Camunda*.)

---

### (c) Compensation — điểm khó nhất

#### Compensation ≠ rollback 📘 K-SAGA-004

> 🚨 **Đây là hiểu lầm nguy hiểm nhất của cả bài.**

| Tiêu chí | **Rollback** | **Compensation** |
|---|---|---|
| Phạm vi | trong **1 DB** | xuyên service, thế giới **đã thay đổi** |
| Bản chất | quay về **như chưa xảy ra** | **hành động nghiệp vụ ngược** |
| Sạch hoàn toàn? | ✅ có | ❌ có thể **không sạch** |
| Ví dụ | `ROLLBACK` transaction | **refund** ≠ "chưa từng charge" (còn phí, thời gian, lịch sử) |

> 💡 **Vì sao compensation khó:** khi bạn đã **trừ kho, gửi mail, charge tiền**, các side-effect đó *đã xảy ra thật*. **refund** một giao dịch **không** xoá được phí xử lý, thời gian đã trôi, dòng lịch sử đã ghi. Phải thiết kế **semantic compensation**, **thứ tự compensate ngược**, và **idempotent**.

---

#### Thiếu Isolation (chữ I của ACID) 📘 K-SAGA-005

> 🔍 **Ví dụ trực giác:** giữa lúc saga đang chạy (đã trừ kho nhưng *chưa* xác nhận đơn), một giao dịch *khác* nhìn vào kho và thấy *state nửa vời* — như nhìn trộm vào bếp khi món còn dang dở.

Giữa các bước, **state trung gian lộ ra** cho giao dịch khác → gây:
- **dirty read nghiệp vụ** (thấy đơn **PENDING**),
- **anomalies** (**lost update**, đọc dữ liệu *sẽ bị compensate*).

**Giảm thiểu:**
- **semantic lock** (trạng thái **PENDING / RESERVED**),
- **versioning**,
- **commutative updates** (cập nhật giao hoán được, không phụ thuộc thứ tự),
- **re-read**.

> 🎯 **Đây là điểm khó nhất của saga** — vì nó *âm thầm*: happy-path chạy ngon, lỗi chỉ lộ ra khi có tải đồng thời (oversell).

---

#### Compensation cũng fail 📘 K-SAGA-007

> ⚠️ **Câu hỏi ác:** chuyện gì xảy ra nếu chính cái **compensation** cũng lỗi?

- compensation phải **retry-able** + **idempotent**,
- lưu **state saga bền** (đang/đã compensate bước nào),
- fail mãi → **DLQ** + **alert** + **can thiệp thủ công**.

> 🚨 **Tuyệt đối tránh:** để hệ **kẹt ở trạng thái nửa vời *im lặng*** (đã trừ tiền, chưa nhả ghế, không ai biết).

---

### (d) Baseline bắt buộc cho MỌI saga 📘 K-SAGA-006

> 🎯 **Chọn pattern (orchestration/choreography) KHÔNG thay được bốn cái này:**

1. **Atomic tại biên mỗi bước:** ghi DB + publish event **nguyên tử** (**Outbox/CDC — Bài 6**).
2. **Idempotent consumer** (at-least-once — **Bài 5**).
3. **Compensation tường minh** + **DLQ**.
4. **Observability:** **correlation id** + **saga state** truy vấn được (**Bài 12**).

> 💡 Bốn cái này là *nền móng*; chọn orchestration hay choreography chỉ là *kiểu nhà* xây trên nền đó.

---

### (e) Đừng nhầm 2 mức 📘 K-SAGA-008

| | **@nestjs/cqrs Saga** | **Distributed Saga** |
|---|---|---|
| Phạm vi | **trong một process** | qua **nhiều service** |
| Cơ chế | phản ứng event bằng **RxJS** (điều phối command nội bộ) | **long-running**, cần **bền vững** |
| Công cụ | RxJS trong Nest | **Temporal/Camunda** hoặc **choreography qua broker** |

> ⚠️ Hai cái *cùng tên "Saga"* nhưng **khác hẳn tầng**. Nói "tôi dùng saga" mà không rõ tầng nào = nguồn của nhiều hiểu nhầm. (🔎 *verify*.)

---

## ③ ⚠️ Cũ → Mới

| Cái cũ | Nay | Vì sao |
|---|---|---|
| **2PC/XA** xuyên service | **Saga** (local tx + compensation) | 2PC blocking, kém chịu lỗi |
| Tự viết **orchestrator + cron + bảng state** | **Temporal / Camunda 8** (**durable execution**, retry/timer/visibility sẵn) | *"2026: workflow-orchestrated"* (🔎 verify) |
| **"Saga đơn giản thì choreography"** (máy móc) | Xét **visibility + cognitive load**; hệ lớn dùng **cả hai** | Tránh kết luận cứng |
| Coi **compensation như rollback** | **Semantic compensation** (idempotent, có thể không sạch) | Thế giới đã đổi |

---

## ④ Áp dụng thực tế + bigtech

### Checkout saga 📘 K-SAGA-009

**Luồng:** **reserve inventory** → **charge payment** → **confirm order**. Mỗi bước = **local tx + event/command**. **Fail payment** → **compensate release inventory**.

```
TIMELINE — checkout saga, fail ở charge:
  T1  reserve inventory  ✅ (kho: RESERVED, chưa trừ vĩnh viễn)
  T2  charge payment     ❌ thẻ bị từ chối
  T3  compensate: release inventory  ✅ (nhả RESERVED)
  T4  saga state = FAILED, alert
  → khách KHÔNG bị giữ kho oan, KHÔNG bị trừ tiền
```

> 🚨 **Ba chỗ dễ sai (phải thuộc):**
> - **(a)** không **idempotent** → **trừ tiền 2 lần** khi **redeliver**;
> - **(b)** lộ state **PENDING** → **oversell / đọc bẩn** (thiếu **semantic lock**);
> - **(c)** **publish trong tx code** thay vì **Outbox** → **mất event**.

**Bigtech:** **Temporal** nổi tiếng cho **durable saga** (compensation chạy *kể cả sau server crash*); **Camunda 8/Zeebe** cho saga **BPMN visual**; **Netflix Conductor/Orkes** cho **orchestration** quy mô lớn. (🔎 *verify*.)

---

## ⑤ Code thực hành + cấu hình

**Orchestration saga** (pseudo, kiểu Temporal — minh hoạ try/compensate):

```javascript
// checkoutSaga.js — orchestration: gọi activity, compensate NGƯỢC khi lỗi
// (Temporal đảm bảo workflow + compensation chạy lại đúng kể cả sau crash) // verify SDK
async function checkout(order) {
  const done = []; // ngăn xếp để compensate NGƯỢC (LIFO)
  try {
    await activities.reserveInventory(order); done.push('inventory');   // mỗi activity idempotent
    await activities.chargePayment(order);     done.push('payment');
    await activities.confirmOrder(order);
  } catch (err) {
    // compensate THEO THỨ TỰ NGƯỢC, mỗi compensation idempotent + retry-able
    if (done.includes('payment'))   await activities.refundPayment(order);    // ngược của charge
    if (done.includes('inventory')) await activities.releaseInventory(order); // ngược của reserve
    throw err; // ghi state FAILED, alert
  }
}
```

```javascript
// activity reserveInventory — semantic lock (RESERVED) tránh oversell + idempotent
async function reserveInventory({ orderId, sku, qty }) {
  // dedup theo orderId (Bài 5) + chuyển trạng thái sang RESERVED,
  // KHÔNG trừ "vĩnh viễn" cho tới confirm (semantic lock chống đọc bẩn)
  // UPDATE ... WHERE available >= qty  (commutative, re-check)  // verify SQL theo schema
}
```

> 🚨 **AI hay sai:** viết compensation như **`DELETE` bản ghi** (rollback) thay vì **hành động nghiệp vụ ngược** (**refund**, **releaseReservation**); và **quên** compensation phải **idempotent** (chạy lại khi retry).

```
VÌ SAO compensate phải theo thứ tự NGƯỢC (LIFO):
  Làm:        reserve → charge → (confirm fail)
  Compensate: refund(charge) TRƯỚC → release(reserve) SAU
  ❌ Nếu release kho trước khi refund: có thể bán lại kho cho người khác
     trong khi tiền vẫn đang giữ → trạng thái mâu thuẫn
```

---

## ⑥ Keywords

**🧠 Cần HIỂU:**
- **saga = local tx + compensation eventual**.
- **orchestration** vs **choreography**.
- **compensation ≠ rollback**.
- **thiếu Isolation → semantic lock**.
- **baseline saga**.

**📌 Cần THUỘC:**
- **2PC** bị né.
- **orchestrator** (**Temporal/Camunda**) vs **event-driven**.
- **compensate thứ tự ngược** + **idempotent**.
- **4 baseline**.

**🛠️ Cần LÀM ĐƯỢC:**
- thiết kế **checkout saga** + **compensation** + gọi tên **3 chỗ sai**.
- phân biệt **Nest cqrs saga** vs **distributed saga**.

---

## ⑦ Mental model

> 🧠 **"Saga = nhiều local transaction nối bằng event, lỗi thì làm hành động nghiệp vụ NGƯỢC** (không phải rollback).
> **Baseline = Outbox + idempotent + compensation + observability.**
> **Orchestration để nhìn rõ, choreography để decouple."**

---

## ⑧ Phỏng vấn & thách đố

**(khó)** Vì sao **không ACID xuyên service**? **Saga** giải gì?
→ database-per-service → không có tx chung; saga dùng chuỗi local tx + compensation đạt eventual.

**(khó)** **Orchestration** vs **choreography** — cơ chế & khi nào chọn?
→ orchestrator trung tâm (visibility/branching) vs event phản ứng (decouple/ngắn); hệ lớn dùng cả hai.

**(rất khó)** **Compensation** khác **rollback** chỗ nào, vì sao khó đúng?
→ rollback trong 1 DB về như chưa xảy ra; compensation là hành động nghiệp vụ ngược trong thế giới đã đổi, không sạch hoàn toàn.

**(rất khó)** Saga **thiếu Isolation** — hệ quả & giảm thiểu?
→ dirty read/anomalies; giảm bằng semantic lock, versioning, commutative updates, re-read.

**(khó)** **Baseline** bắt buộc cho mọi saga?
→ atomic biên (Outbox) + idempotent consumer + compensation+DLQ + observability (correlation id).

> 🧩 **Thách đố:** *"Checkout saga của junior chạy **happy-path** ngon nhưng production thỉnh thoảng **trừ tiền 2 lần** và **oversell**. Hai nguyên nhân gốc?"*
> → **(1)** consumer/activity **không idempotent** dưới **at-least-once** → charge/trừ kho **lặp** khi **redeliver**; **(2)** thiếu **semantic lock** (**RESERVED**) → **đọc bẩn** state **PENDING** gây **oversell**. *(Có thể thêm: publish trong tx → mất event.)*

---

## ⑨ Bài tập + tiêu chí

**Đề:** Thiết kế saga **"đặt vé"**: **giữ ghế → thanh toán → xuất vé**. Liệt kê bước, **compensation** mỗi bước, **semantic lock**, **2 chỗ idempotency** bắt buộc.

> ✅ **Đạt khi:**
> - **compensation** đúng **nghiệp vụ ngược** (**nhả ghế**, **refund**) + **idempotent** + **thứ tự ngược**.
> - ghế ở trạng thái **HELD** (**semantic lock**, có **TTL**).
> - **idempotency** ở **thanh toán** + **xuất vé**.
> - nêu **Outbox** cho event.

---

## ⑩ Đọc thêm

- **`microservices.io`**: **Saga pattern**.
- **Temporal docs** (**durable execution**, saga).
- **Camunda 8 docs**. (🔎 *verify*.)

> 🎯 **Một câu mang về:** *Saga không phải "transaction phân tán phiên bản nhẹ" — nó là chấp nhận thế giới đã thay đổi và thiết kế **hành động ngược** cho từng bước, trên nền **Outbox + idempotency**.*
