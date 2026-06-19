# 🎯 BÀI 10 — Service Decomposition & bounded context

**Phủ:** K-DECOMP-001 → 009

> 💡 **Mục tiêu một câu:** Đây là **gốc rễ** sinh ra mọi bài trên — chính việc **tách service** (**database-per-service**) tạo ra nhu cầu **Saga / Outbox / event**. Học xong, bạn tách service đúng cách, tránh được hai cái bẫy đối nghịch: **distributed monolith** và **nano-service**.

---

## ① Mục tiêu & vị trí trong mạch

Vì sao bài "tách service" lại đặt *cuối* chứ không *đầu*? Có chủ đích: để khi bàn *"tách sao cho đúng"*, bạn **đã biết** *"tách xong đồng bộ dữ liệu kiểu gì"* (Saga, Outbox, event, replication).

> 🎯 **Chốt vị trí:** chính **database-per-service** là *nguyên nhân* khiến ta không còn `JOIN`/transaction chung → phải dùng tất cả cơ chế đã học. Học xong bài này bạn tránh được **distributed monolith** và **nano-service**. Cross-ref **Q (DDD)**, **A (system design)**.

---

## ② Giảng cơ bản → nâng cao

### (a) Microservice thật sự là gì 📘 K-DECOMP-001

> 🚨 **Hiểu lầm phổ biến nhất:** microservice **KHÔNG** phải *"service nhỏ"* hay *"dùng Docker"*.

**Microservice** = service **tự trị** quanh một **business capability / bounded context**, **deploy độc lập**, **sở hữu dữ liệu riêng** (**no shared DB**), giao tiếp qua **API/event**, có thể khác **stack**.

> 🎯 **Trọng tâm: ranh giới + autonomy, KHÔNG phải kích thước/công cụ.**

> 💡 **Ví dụ trực giác:** hai cửa hàng *nhượng quyền* (franchise) tự trị — mỗi cái tự quản kho, tự quyết giờ mở cửa, tự thuê nhân viên. Chúng *không* dùng chung một cái kho hay một bảng lương. Đó là "autonomy", không phải "nhỏ".

---

### (b) Tách theo gì 📘 K-DECOMP-002, 003

Tách theo **business capability / bounded context (DDD)** + theo **volatility / đội sở hữu**, sao cho mỗi service **đổi độc lập**.

> 🚨 **Tách theo tầng kỹ thuật (UI / logic / DB) là SAI** → mọi feature đụng nhiều service (coupling cao, deploy lệ thuộc) = **distributed monolith**.

```
❌ TÁCH THEO TẦNG (sai):
  feature "thêm coupon" phải sửa:
    UI-service + Logic-service + DB-service  → 3 service đổi cùng lúc, deploy lệ thuộc

✅ TÁCH THEO CAPABILITY (đúng):
  feature "thêm coupon" chỉ sửa:
    Promotion-service  → 1 service đổi độc lập, deploy riêng
```

> 💡 **Bounded context** = phạm vi một **mô hình ngôn ngữ nhất quán**. Cùng từ **"Customer"** *nghĩa khác* ở **Sales** (lead tiềm năng) vs **Billing** (tài khoản có hoá đơn). Biên service trùng bounded context → **ít coupling ngữ nghĩa**, mỗi đội sở hữu mô hình của mình.

---

### (c) Database-per-service 📘 K-DECOMP-004

> 🚨 **Vì sao shared DB là anti-pattern:**
> - **coupling schema** (đổi bảng → vỡ nhiều service),
> - **không deploy độc lập**,
> - **khoá chung** (lock tranh nhau).

**Cái giá khi tách DB:** mất **JOIN / transaction xuyên service** → cần **API composition / Saga / event** để đồng bộ + **chấp nhận eventual**.

> 🎯 **Hiểu cả lợi và giá:** lợi = **autonomy**; giá = mất JOIN/transaction chung, phải đồng bộ thủ công qua event. *Không có bữa trưa miễn phí* (gặp lại tinh thần Bài 1).

---

### (d) Hai anti-pattern granularity

> ⚖️ **Granularity (độ mịn) là một trục có hai đầu đều xấu.** Quá thô và quá mịn đều hỏng theo cách riêng.

#### Distributed monolith 📘 K-DECOMP-005

> 🚨 **Tệ hơn cả monolith.**

Nhiều service nhưng **coupling chặt** (deploy phải **đồng bộ**, **sync call dây chuyền**, **shared DB/lib**, đổi 1 phải đổi nhiều) → gánh **chi phí phân tán** (latency/phức tạp/failure) mà **KHÔNG** được lợi **autonomy**.

> 🔍 **Dấu hiệu nhận biết:** *"không deploy 1 service mà không deploy service khác"*.

#### Nano-service 📘 K-DECOMP-007

> ⚠️ Quá **mịn** → 1 thao tác nghiệp vụ gọi **nhiều service** (**chatty**, **latency cộng dồn**, **transaction phân tán khắp nơi**, vận hành nặng).

> 🔍 **Dấu hiệu:** nhiều service **luôn đổi cùng nhau** / **luôn gọi nhau đồng bộ** → nên **gộp lại**.

> 🎯 **Chốt:** **granularity là trade-off**, KHÔNG *"càng nhỏ càng tốt"*.

| | **Distributed monolith** | **Nano-service** |
|---|---|---|
| Lỗi | quá **thô/coupling chặt** | quá **mịn** |
| Triệu chứng | deploy lệ thuộc, sync-chain | chatty, latency cộng dồn |
| Dấu hiệu | "không deploy độc lập được" | "service luôn đổi/gọi cùng nhau" |
| Sửa | tách thật / bỏ shared DB-lib | **gộp lại** |

---

### (e) Khi service A cần dữ liệu của B 📘 K-DECOMP-006, 008

| Cách | Bản chất | Được | Mất |
|---|---|---|---|
| **Sync call B (API composition)** | gọi **runtime** | đơn giản, **fresh** | A phụ thuộc B **sống**, **+latency**, **cascade** |
| **Data replication qua event** | A giữ **bản sao local** từ event của B | **decouple**, nhanh, chịu lỗi | **eventual**, trùng dữ liệu, phức tạp |
| **CQRS read model** | **read model** riêng xuyên service | query nhanh | thêm hạ tầng, **eventual** |

> 🎯 **Chọn theo:** **độ tươi cần thiết** + **tần suất** + **độ chịu lỗi**. **Không có default.**

> 💡 **Ví dụ cụ thể:** **Order** cần tên & giá sản phẩm. Nếu cần *giá tại thời điểm đặt* → **replication** (giữ snapshot, giá không đổi theo Catalog sau này, lại không cascade). Nếu cần *giá realtime* → có thể phải **sync call**.

---

### (f) Di trú monolith → microservices 📘 K-DECOMP-009

> 🌳 **Ẩn dụ — Strangler Fig (cây đa bóp nghẹt):** cây đa mọc *quấn quanh* cây chủ, lớn dần, tới khi cây chủ tiêu biến mà tán lá vẫn liền mạch — không có khoảnh khắc "chặt phăng cây cũ".

**Strangler Fig:** **bọc monolith**, dần tách từng **capability** ra service mới + **route traffic** dần (qua **gateway/proxy**), giữ monolith chạy **song song** tới khi **rút hết**.

> 🚨 **Vì sao không big-bang rewrite:** rủi ro cao, lâu, dễ thất bại (mất tính năng, **không có đường lùi**). (🔎 *verify tên pattern*.)

```
Strangler Fig — tách dần, luôn có đường lùi:
  [Gateway] ──► Monolith (100% traffic)
  [Gateway] ──► OrderSvc (mới)  + Monolith (phần còn lại)   ← route dần
  [Gateway] ──► OrderSvc + PaymentSvc + ... (Monolith teo dần)
  [Gateway] ──► tất cả service mới (Monolith = 0)            ← rút hết
```

---

## ③ ⚠️ Cũ → Mới

| Cái cũ | Nay | Vì sao |
|---|---|---|
| **"Microservice = service nhỏ/Docker"** | **Tự trị + DB riêng + deploy độc lập** quanh **bounded context** | Trọng tâm là **ranh giới** |
| **Tách theo tầng kỹ thuật** (UI/logic/DB) | Tách theo **business capability/DDD** | Tránh **distributed monolith** |
| **Shared database** cho tiện JOIN | **Database-per-service** + composition/Saga/event | **Autonomy** + deploy độc lập |
| **"Càng nhiều service càng tốt"** | **Granularity là trade-off**; **gộp** khi nano | Tránh **chatty / transaction phân tán** |
| **Big-bang rewrite** monolith | **Strangler Fig** từng phần | Giảm rủi ro (🔎 verify) |

---

## ④ Áp dụng thực tế + bigtech

**Use case:** e-commerce tách **Catalog, Order, Inventory, Payment, Shipping** theo **bounded context**, mỗi cái **DB riêng**.

> 💡 **Order cần thông tin sản phẩm** → giữ **bản sao tối thiểu** (**id, tên, giá tại thời điểm đặt**) qua event **`ProductUpdated`** *thay vì* gọi **Catalog** mỗi lần.
> *Vì sao?* giá tại thời điểm đặt là *sự thật lịch sử của đơn* — không nên đổi khi Catalog đổi giá; lại tránh **cascade** khi Catalog chết.

**Bigtech:** **Amazon/Netflix** nổi tiếng **database-per-service** + tách theo **capability**; **Strangler Fig** được dùng rộng để thoát monolith dần. (pattern; 🔎 *verify*.)

---

## ⑤ Code thực hành + cấu hình

> 💡 **Không phải bài code — là bài thiết kế.** Minh hoạ **API composition** vs **replication**:

```javascript
// (A) API composition: Order gọi Catalog runtime (fresh nhưng coupling + latency + cascade)
async function getOrderView(orderId) {
  const order = await orderRepo.find(orderId);
  const product = await catalogClient.get(order.productId); // phụ thuộc Catalog SỐNG -> bọc resilience (Bài 9)!
  return { ...order, productName: product.name };
}
```

```javascript
// (B) Replication qua event: Order giữ snapshot product tại thời điểm đặt (decouple, eventual)
//   - subscribe 'ProductUpdated' -> upsert vào bảng product_replica của Order (idempotent - Bài 5)
//   - getOrderView đọc LOCAL, KHÔNG gọi Catalog -> không cascade, nhưng dữ liệu có thể trễ
```

> 🚨 **AI hay sai:** mặc định **API composition** (gọi nhau sync) **khắp nơi** → **distributed monolith + cascade**; hoặc **shared DB** "cho nhanh" → **mất autonomy**. **Tech Lead phải hỏi: *độ tươi cần bao nhiêu?* trước khi chọn.**

```
CÂU HỎI QUYẾT ĐỊNH trước khi chọn share-data:
  "Dữ liệu này cần tươi tới mức nào?"
    ├─ phải realtime tuyệt đối  → cân nhắc sync (chấp nhận coupling+cascade, bọc resilience)
    └─ chấp nhận trễ vài giây   → replication qua event (decouple, không cascade) ✅ thường tốt hơn
```

---

## ⑥ Keywords

**🧠 Cần HIỂU:**
- **microservice = autonomy + bounded context + DB riêng**.
- **distributed monolith**; **nano-service**.
- **sync vs replication**.
- **Strangler Fig**.

**📌 Cần THUỘC:**
- tách theo **capability** không theo **tầng**.
- **shared DB là anti-pattern**.
- **3 cách share data**.
- dấu hiệu **distributed monolith** (*"không deploy độc lập"*).

**🛠️ Cần LÀM ĐƯỢC:**
- vẽ **ranh giới service** theo **bounded context**.
- chọn cách **share data** theo **độ tươi**.
- nhận diện **over/under-decomposition**.

---

## ⑦ Mental model

> 🧠 **"Service = bounded context tự trị, DB riêng, deploy độc lập.**
> **Tách theo nghiệp vụ, KHÔNG theo tầng.**
> **Quá chặt = distributed monolith (tệ hơn monolith); quá mịn = nano (chatty).**
> **Thoát monolith bằng Strangler, không big-bang."**

---

## ⑧ Phỏng vấn & thách đố

**(dễ)** Điều gì *thật sự* định nghĩa **microservice**?
→ autonomy + bounded context + DB riêng + deploy độc lập; không phải kích thước/Docker.

**(khó)** Tách theo tiêu chí gì? Vì sao tách theo **tầng kỹ thuật** sai?
→ theo business capability/DDD; tách theo tầng → mọi feature đụng nhiều service = distributed monolith.

**(rất khó)** **Shared DB** là anti-pattern thế nào, cái giá khi tách DB?
→ coupling schema/khoá chung/không deploy độc lập; giá tách = mất JOIN/transaction → cần composition/Saga/event + eventual.

**(rất khó)** **Distributed monolith** — dấu hiệu, vì sao tệ hơn monolith?
→ "không deploy độc lập được"; gánh chi phí phân tán mà không được autonomy.

**(khó)** A cần dữ liệu B — **sync vs event**, chọn sao?
→ theo độ tươi + tần suất + chịu lỗi; không có default.

> 🧩 **Thách đố:** *"Hệ có **30 service** nhưng mỗi lần release phải **deploy đồng loạt**, và một thao tác checkout gọi **12 service đồng bộ**. Chẩn đoán?"*
> → **distributed monolith** (deploy lệ thuộc) + **nano/chatty** (12 sync call). **Gốc:** tách sai (theo tầng / quá mịn / shared lib-DB). **Sửa:** **gộp theo bounded context**, thay **sync-chain** bằng **event/replication**, làm **deploy độc lập** được.

---

## ⑨ Bài tập + tiêu chí

**Đề:** Cho monolith **"đặt đồ ăn"** (user, nhà hàng, món, đơn, thanh toán, giao hàng). Đề xuất **ranh giới service** + chọn **share-data** cho *"Đơn cần tên món & giá"*.

> ✅ **Đạt khi:**
> - **ranh giới** theo **bounded context** (không theo tầng).
> - **DB riêng** mỗi service.
> - **"Đơn"** giữ **snapshot** món/giá tại thời điểm đặt qua **event** (giải thích *vì sao không gọi sync mỗi lần*).
> - nêu **Strangler** nếu tách dần từ monolith.

---

## ⑩ Đọc thêm

- **Building Microservices** (Sam Newman); **Monolith to Microservices** (**Strangler**).
- **DDD** (Eric Evans) cho **bounded context**.
- **`microservices.io`**. (🔎 *verify*.)

> 🎯 **Một câu mang về:** *Microservice không phải chuyện "nhỏ" hay "Docker" — mà là vẽ đúng **ranh giới nghiệp vụ** để mỗi đội **deploy độc lập**. Vẽ sai ranh giới, bạn lãnh đủ chi phí phân tán mà chẳng được tự do.*
