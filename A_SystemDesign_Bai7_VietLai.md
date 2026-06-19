# 🎯 BÀI 7 — Decomposition & service boundaries

**Phủ:** A-DEC-001 → A-DEC-004

> 💡 **Tư tưởng cốt lõi của cả bài:** Chia hệ thống thành service **không khó ở chỗ "chia ra"**, mà khó ở chỗ **chia theo đâu**. Chia đúng → mỗi service tự chủ, đổi một chỗ không kéo theo cả hệ. Chia sai → "đổi gì cũng phải họp cả công ty".

---

## ① Mục tiêu & vị trí trong mạch

Bài này **tiếp khối "Đánh đổi & ranh giới"**.

- **Bài 6** nói **khi nào** chia (**monolith vs micro**).
- **Bài 7** nói **chia theo đâu** và **dấu hiệu chia sai**.

> 🎯 **Mạch nối:** quan trọng cho các **case study lớn** — **checkout (Bài 15)**, **feed (Bài 11)** — khi phải đặt **ranh giới service**. Giao với **mục K** (messaging/event) khi **mỗi service có DB riêng**.

---

## ② Giảng cơ bản → nâng cao

### (a) Trực giác

> **Ẩn dụ — chia phòng ban công ty:**
> - Chia theo **chức năng nghiệp vụ** (kế toán, kho, bán hàng) → mỗi phòng **tự chủ**, lo trọn một việc.
> - Chia theo **tầng kỹ thuật** (phòng "viết SQL", phòng "viết API") → làm *bất cứ việc gì* cũng phải **họp cả 3 phòng**.

> 🎯 **Chốt:** **Ranh giới đúng = ít phải "họp"** (ít **gọi chéo**) khi có thay đổi.

---

### (b) Cơ chế

#### 📘 **A-DEC-001 — Chia theo business capability / volatility, KHÔNG theo layer kỹ thuật**

Chia theo **năng lực nghiệp vụ** hoặc theo **mức độ hay-thay-đổi (volatility)**, **không** theo tầng (**controller / service / db**).

> 💡 Tức là chia theo **bounded context (DDD)** — **cái gì thay đổi cùng nhau thì ở cùng một service**.

> 🔍 **Vì sao?** Để **giảm coupling khi thay đổi**: mỗi service **tự chủ một nghiệp vụ end-to-end**. Khi nghiệp vụ "đặt hàng" đổi, bạn chỉ sửa service Ordering — không phải đi sờ vào 3 service kỹ thuật khác nhau.

```
❌ Chia theo LAYER (đổi 1 nghiệp vụ → chạm cả 3):
  [API service] ── [Logic service] ── [DB service]
       ▲ đổi "đặt hàng" → phải sửa CẢ BA ▲

✅ Chia theo BOUNDED CONTEXT (đổi 1 nghiệp vụ → chạm 1):
  [Ordering]  [Catalog]  [Inventory]   ← mỗi cái có đủ API+logic+DB của mình
   ▲ đổi "đặt hàng" → chỉ sửa Ordering
```

#### 📘 **A-DEC-002 — Dấu hiệu boundary vẽ SAI**

- ❌ 2 service **luôn deploy/đổi cùng nhau**.
- ❌ **gọi chéo liên tục** (**chatty**).
- ❌ **transaction xuyên service** nhiều.
- ❌ **chia sẻ DB**.

> 🚨 Thấy các dấu hiệu này ⇒ **nên gộp lại** hoặc **vẽ lại ranh giới**. Chúng là tín hiệu bạn đã cắt ngang một bounded context — như cắt đôi một phòng ban rồi bắt hai nửa gọi điện cho nhau suốt ngày.

#### 📘 **A-DEC-003 — Database-per-service** *(giao với K)*

**Mỗi service một DB riêng.**

| Lợi ✅ | Kéo theo (cái giá) ❌ |
|---|---|
| **tự chủ** + **deploy độc lập** (đổi schema không ảnh hưởng service khác) | **mất join xuyên service** |
| | cần **đồng bộ dữ liệu** (**event / CDC**) |
| | **consistency xuyên service** phức tạp → phải dùng **Saga** thay vì **transaction** *(xem Bài 15)* |

> 🔍 **Vì sao mất join lại đau?** Vì trong một DB, JOIN hai bảng là một câu SQL. Khi tách DB, dữ liệu nằm ở hai nơi không JOIN trực tiếp được — bạn phải **đồng bộ** sang nhau qua **event/CDC**, và chấp nhận dữ liệu hội tụ **eventual** chứ không tức thì.

#### 📘 **A-DEC-004 — Modular monolith**

**Ranh giới module rõ ràng trong một deploy** duy nhất.

> 🎯 **Vì sao là điểm khởi đầu tốt?** Vì nó cho bạn **lợi ích của boundary** (code tách bạch, ít coupling) mà **chưa phải gánh chi phí phân tán** (**network**, **distributed transaction**, **ops**). Tách ra **microservice sau** khi **seam** (đường nối) đã rõ và **có lý do thật**.

> 💡 **Phân biệt 📘 vs ➕:** toàn bộ cơ chế là kiến thức ổn định (📘). Liên hệ **DDD / bounded context** và **Conway's Law** (*cơ cấu hệ phản chiếu cơ cấu tổ chức*) là ➕ làm sâu.

---

## ③ ⚠️ Kiến thức cũ / hay bị làm sai

| Hay gặp (❌) | ✅ Nên làm | Vì sao |
|---|---|---|
| Chia service theo **tầng** (API service, DB service…) | Chia theo **bounded context / nghiệp vụ** | Theo tầng ⇒ đổi gì cũng phải sửa nhiều service |
| Nhiều service **share chung một DB** | **DB-per-service** + đồng bộ qua **event/CDC** | Share DB ⇒ **coupling ngầm**, mất tự chủ |
| **Distributed transaction / 2PC** xuyên service | **Saga + compensating action** | **2PC** mong manh & chậm trong hệ phân tán |
| Tách **micro** ngay từ đầu | **Modular monolith** trước, tách khi **seam** rõ | Tránh chi phí phân tán khi chưa cần |

> 🔍 **Đào sâu — vì sao 2PC mong manh?** **2PC (two-phase commit)** bắt mọi service "khoá" tài nguyên rồi chờ một **coordinator** ra lệnh commit. Nếu coordinator chết giữa chừng, các service treo trong trạng thái khoá — toàn hệ đứng. **Saga** né điều đó bằng cách làm từng bước rồi **bù trừ (compensating action)** nếu hỏng, thay vì khoá đồng loạt.

---

## ④ Áp dụng thực tế + So sánh bigtech

- **Conway's Law:** tổ chức nhiều **team độc lập** ⇒ hệ tự nhiên tách theo team — nên **ranh giới service thường khớp ranh giới team** (pattern phổ biến).
- **Strangler Fig pattern:** cách tách microservice từ monolith **dần dần** — bọc monolith bằng một **facade**, chuyển **từng capability** ra service mới rồi **"siết" dần**. An toàn hơn nhiều so với **"viết lại từ đầu"** (rewrite).

> **Ẩn dụ Strangler Fig:** giống cây sung bóp nghẹt — nó mọc bao quanh cây chủ, lớn dần, rồi cuối cùng thay thế hẳn. Bạn không chặt cây chủ ngay; bạn để cái mới lớn lên *bọc quanh* cái cũ cho tới khi cái cũ không còn cần nữa.

---

## ⑤ Code thực hành + cấu hình

Không phải bài code; dùng **bài tập vẽ ranh giới**. Ví dụ minh hoạ **ranh giới đúng** (**modular monolith** với module rõ):

```text
# Modular monolith — cấu trúc thư mục thể hiện bounded context
src/
  catalog/     # module Catalog: domain + service + repo, KHÔNG truy cập bảng của module khác
    domain/  api/  repo/
  ordering/    # module Ordering: tự chủ, gọi catalog QUA interface công khai, không qua DB
    domain/  api/  repo/
  inventory/   # module Inventory
    domain/  api/  repo/
  shared-kernel/   # type/contract dùng chung TỐI THIỂU

# Quy tắc: module chỉ gọi nhau qua public API (interface), không đụng bảng/đối tượng nội bộ nhau.
# Khi cần tách micro: mỗi module đã là một seam sẵn sàng -> bê ra thành service.
```

```text
# Heuristic kiểm tra boundary (tự hỏi):
- Hai service này có BAO GIỜ deploy riêng không? Nếu luôn cùng nhau -> gộp.
- Một thay đổi nghiệp vụ điển hình chạm mấy service? >2 thường -> boundary sai.
- Có transaction nào BẮT BUỘC xuyên 2 service không? Nhiều -> cân nhắc gộp/đổi ranh giới.
```

> 🔎 **Vì sao "chỉ gọi nhau qua public API" lại quan trọng?** Vì đó là thứ biến module thành **seam sẵn sàng để tách**. Nếu module Ordering lén đọc thẳng bảng của Catalog, ngày bạn muốn tách Catalog ra microservice riêng, sợi dây ngầm đó sẽ đứt — và bạn phát hiện ra coupling quá muộn. Gọi qua interface = ranh giới "giả lập" sẵn của microservice tương lai.

---

## ⑥ Keywords cần nhớ

🧠 **Cần HIỂU:**
- chia theo **business capability / volatility** vs theo **layer**.
- vì sao **share DB** là **anti-pattern**.
- vì sao **Saga** thay **2PC**.
- **modular monolith**.

📌 **Cần THUỘC:**
- dấu hiệu **boundary sai** (**chatty / deploy cùng nhau / txn xuyên service / share DB**).
- **DB-per-service** đánh đổi.
- **Strangler Fig**.
- **Conway's Law**.

🛠️ **Cần LÀM ĐƯỢC:**
- vẽ **ranh giới** cho một domain.
- chỉ ra **boundary sai**.
- thiết kế **modular monolith** có **seam**.

---

## ⑦ Mental model

> 🎯 **"Chia theo cái thay đổi cùng nhau, không theo tầng kỹ thuật."**
> Bắt đầu **modular monolith**; tách **micro** khi **seam** rõ + có lý do.

---

## ⑧ Câu hỏi phỏng vấn & thách đố

**(TB)** Chia service theo gì, vì sao không theo **layer**?
→ theo **bounded context / nghiệp vụ**; theo layer ⇒ **coupling** cao khi đổi.

**(TB)** Dấu hiệu một **boundary** bị chia sai?
→ **chatty**, **deploy cùng nhau**, **txn xuyên service**, **share DB**.

**(khó)** **DB-per-service** lợi/hại gì?
→ **tự chủ + deploy độc lập**; nhưng **mất join** + cần **event/CDC** + **consistency** phức tạp.

**(TB)** **Modular monolith** là gì, vì sao là điểm khởi đầu tốt?
→ **boundary rõ trong 1 deploy**; lợi ích tách mà **chưa gánh chi phí phân tán**.

**(khó)** Tách **micro** từ monolith an toàn thế nào?
→ **Strangler Fig**: bọc **facade**, chuyển từng **capability**, **siết dần**.

> 🧩 **Thách đố (trick):** Hai microservice **"Order"** và **"OrderItem"** luôn được sửa và deploy cùng nhau. Có gì sai?
>
> ```
> ❌ Chia quá nhỏ theo ENTITY:   [Order] ──chatty──► [OrderItem]
>        ↓ luôn sửa cùng nhau, luôn deploy cùng nhau
> Chẩn đoán: cùng một bounded context "Ordering" bị cắt đôi
>        ↓
> ✅ GỘP lại thành 1 service "Ordering"
>    (tách ra chỉ tạo chatty calls + distributed transaction vô ích)
> ```
> 🔎 **Vì sao là bẫy?** Đây là chia **theo entity** thay vì **theo capability** — **boundary sai**. Order và OrderItem thuộc **cùng một bounded context "Ordering"**, nên **gộp lại**. Tách ra chỉ đẻ ra **chatty calls** + **distributed transaction** vô ích.

---

## ⑨ Bài tập + tiêu chí tự chấm

**BT1.** Cho domain e-commerce (**catalog, cart, order, payment, inventory, shipping, notification**). Nhóm thành **4–6 service** và giải thích ranh giới.

> ✅ **Đạt khi:** chia theo **capability**, chỉ ra cặp nào nên **gộp/tách** và vì sao, nêu chỗ cần **event** giữa service.

**BT2.** Chỉ ra **3 dấu hiệu** cho biết bạn đã chia service sai trong một thiết kế giả định.

> ✅ **Đạt khi:** dùng đúng các tín hiệu (**chatty / deploy cùng / txn xuyên / share DB**).

---

## ⑩ Đọc thêm

- **Sam Newman** — *Building Microservices* (chương boundaries) & *Monolith to Microservices* (**Strangler Fig**).
- **Eric Evans** — *Domain-Driven Design* (**Bounded Context**).

---

> 🎯 **Khẩu quyết khép bài:** *Chia theo nghiệp vụ, không theo tầng. Cái gì đổi cùng nhau thì ở cùng nhau. Modular monolith trước, tách micro khi seam rõ. Chatty + share DB = boundary sai.*
