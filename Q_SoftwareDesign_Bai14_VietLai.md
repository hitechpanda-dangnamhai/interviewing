# 🎯 BÀI 14 — DDD (2): **Aggregate** · **Invariant** · **Transaction Boundary** · **Repository** · **Domain/App Service** · **Domain Event** · khi nào KHÔNG dùng DDD

**Phủ:** Q-DDD-004 → Q-DDD-005 → Q-DDD-006 → Q-DDD-007 → Q-DDD-008 → Q-DDD-009 → Q-DDD-010

> 💡 **Mental model mở đầu:**
> Bài 13 dạy *vẽ ranh giới & gọi tên* (strategic). Bài này dạy **building block để GIỮ tính nhất quán** bên trong domain phức tạp (tactical):
> - **Aggregate** = pháo đài có **một cổng** (root) bảo vệ **invariant**.
> - **một transaction ~ một aggregate**; muốn nhiều → gửi thư (**domain event** / **saga**).
> - Và **judgment**: khi nào DDD đầy đủ là **overhead**.

---

## ① Mục tiêu & vị trí trong mạch

Phần **tactical DDD** — **building block để giữ tính nhất quán** của domain phức tạp, và **judgment**: khi nào **DDD đầy đủ là overhead**.

```
   Bài 13: STRATEGIC (ranh giới + ngôn ngữ)
                    │
   Bài 14: TACTICAL (giữ nhất quán bên trong)  ← bài này
        aggregate · invariant · repository · domain event
        + KHI NÀO KHÔNG dùng DDD đầy đủ
                    │
   Bài 15: Tech Lead judgment
```

> 🎯 **Chốt vị trí:** đây là **đỉnh của mạch "mô hình hoá nghiệp vụ"**, chuẩn bị cho **Tech Lead judgment** (Bài 15).

---

## ② Giảng cơ bản → nâng cao

### (a) Aggregate & Aggregate Root (Q-DDD-004)

> **Ẩn dụ — pháo đài một cổng.** Một **Aggregate** là pháo đài; **Aggregate Root** là **cổng duy nhất** ra vào. Muốn sửa gì bên trong, *phải qua cổng* — lính gác (root) kiểm tra mọi thứ. Không ai trèo tường vào sửa lén.

| Khái niệm | Là gì |
|---|---|
| **Aggregate** | cụm object là một **consistency boundary** (một đơn vị nhất quán) |
| **Aggregate Root** | **cửa DUY NHẤT** truy cập/sửa bên trong |

> 🚨 **Quy tắc vàng — tham chiếu aggregate KHÁC bằng ID, KHÔNG nhúng object:**

```
✅ class Order {
     lines: OrderLine[];          // OrderLine NẰM TRONG aggregate Order
     customerId: string;          // tham chiếu Customer bằng ID (KHÔNG nhúng cả Customer)
   }
❌ class Order {
     customer: Customer;          // nhúng cả object Customer → load/sửa chéo, vỡ boundary
   }
```

> 💡 **Vì sao tham chiếu bằng ID?** Để **giữ boundary**, **tránh load/sửa chéo** (kéo cả Customer ra mỗi lần đụng Order), và **cho phép eventual consistency** (hai aggregate cập nhật độc lập).

---

### (b) Invariant (Q-DDD-005)

**Invariant** = **quy tắc nghiệp vụ LUÔN phải đúng** cho aggregate. Vd **"tổng item ≤ hạn mức đơn"**.

> 🎯 **Mọi thay đổi đi qua root** để root **kiểm tra/giữ nhất quán nội bộ**. Đây là **lý do encapsulation (Bài 1) quan trọng**: **không cho sửa thẳng vào trong aggregate** — nếu cho, ai đó sẽ đẩy aggregate vào trạng thái phá invariant mà root không kịp gác.

---

### (c) Aggregate boundary ≈ Transaction boundary (Q-DDD-006)

> 🎯 **Một aggregate = một đơn vị nhất quán giao dịch: một transaction nên sửa MỘT aggregate.**

> 🚨 **Khi thao tác phải đổi NHIỀU aggregate → ĐỪNG một transaction khổng lồ;** dùng **eventual consistency / domain event / saga**.

| Boundary | Hệ quả |
|---|---|
| **to** (gom nhiều thứ) | **contention / lock** nhiều → nghẽn |
| **nhỏ** | nhiều **cross-aggregate** → phải phối hợp qua event |

> 💡 **Cân theo invariant THẬT:** boundary nên đủ lớn để *gói trọn một invariant cần nhất quán tức thời*, không lớn hơn. Cái gì *không cần* nhất quán tức thời → tách ra, nối bằng event.

---

### (d) Repository trong DDD (Q-DDD-007)

> **Ví dụ trực giác:** repository như một **thủ thư** nói **ngôn ngữ của bạn** — bạn hỏi *"cho tôi các đơn hàng đang active của khách này"*, không phải *"chạy SELECT JOIN ba bảng"*.

**Repository (DDD)** khác **DAO/ORM** thường:

| Tiêu chí | **DAO/ORM thường** | **Repository (DDD)** |
|---|---|---|
| Trả về | **ORM entity** (leak persistence) | **domain object** (không leak ORM) |
| Hình dung | bảng/row | **collection-like** cho **aggregate root** |
| Interface theo | **khả năng DB** | **nhu cầu domain** (`findActiveOrdersFor(customer)`) |

> 🎯 **Repository "nói ngôn ngữ domain".** *(persistence/transaction/N+1 chi tiết ở **mục E**.)*

---

### (e) Domain Service vs Application Service (Q-DDD-008)

| Loại | Làm gì |
|---|---|
| **Domain Service** | **rule nghiệp vụ thuần**, **liên nhiều aggregate**, **không chạm hạ tầng** |
| **Application Service** | **điều phối use case**, quản **transaction**, **gọi ra ngoài (port)** |

> 🚨 **CẢNH BÁO — đặt side-effect đúng tầng:** **"Gửi email xác nhận sau khi đặt đơn"** = **side-effect hạ tầng** → thuộc **Application Service** (qua **port**), **KHÔNG nhét vào domain**. (Nối **Bài 11–12**.)

> 💡 **Vì sao email không được nằm trong domain?** Vì domain phải **thuần** (test được không cần mạng, không phụ thuộc nhà cung cấp mail). Nhét email vào `Order` → domain rò hạ tầng, hết test thuần.

---

### (f) Domain Event (Q-DDD-009)

> **Ẩn dụ:** aggregate không *gọi điện ra lệnh* cho aggregate khác, mà **dán thông báo** *"OrderPlaced — đơn đã đặt xong"* lên bảng tin; ai quan tâm thì tự phản ứng.

**Domain Event** = sự kiện **"đã xảy ra"** trong domain (**`OrderPlaced`**), phát ra cho phần khác **phản ứng async**.

```
   Order.place()  ──phát──▶  event "OrderPlaced"
                                  │ (async, qua bus)
                  ┌───────────────┼───────────────┐
                  ▼               ▼               ▼
            Inventory       Notification      Analytics
            (trừ kho)       (gửi mail)        (ghi nhận)
   → Order KHÔNG gọi trực tiếp 3 cái trên → loose coupling + eventual consistency
```

> 🔎 **Giảm coupling:** aggregate **không gọi trực tiếp** aggregate/context khác → **loose coupling + eventual consistency**. (Nối **Observer/PubSub — Bài 9**; cơ chế **wiring** ở **G/H**.)

---

### (g) ⚠️ Khi nào KHÔNG dùng DDD đầy đủ (Q-DDD-010) — judgment

**DDD tactical** có **chi phí học/triển khai cao**.

| | Áp DDD tactical đầy đủ |
|---|---|
| ❌ **Không đáng** | **CRUD đơn giản / ít rule / đội nhỏ** → overhead |
| ✅ **Đáng** | **nghiệp vụ phức tạp, nhiều invariant, ngôn ngữ domain giàu, thay đổi liên tục** |

> 🎯 **Phân biệt sống còn:** **strategic DDD** (**bounded context** — **luôn hữu ích**) vs **tactical DDD** (**aggregate/repo/VO** — **chọn lọc**). Bạn có thể vẽ bounded context đúng mà *không* dùng aggregate pattern, và đó là lựa chọn hợp lý cho service mỏng.

---

## ③ ⚠️ Cũ → Mới

| Hiểu sai / cũ | Đúng | Vì sao |
|---|---|---|
| **Nhúng object aggregate khác vào trong** | **Tham chiếu bằng ID** | Giữ boundary, cho **eventual consistency** |
| **Một transaction sửa nhiều aggregate** | **Một tx ~ một aggregate**; nhiều → **event/saga** | Tránh **lock lớn**, theo **consistency boundary** |
| **Repository = DAO trả ORM entity** | **Repo trả domain object**, nói **ngôn ngữ domain** | Không **leak persistence** vào domain |
| **Gửi email trong domain** | **Side-effect → Application Service** (qua **port**) | Domain **không chạm hạ tầng** |
| **"DDD cho mọi service"** | **Tactical chọn lọc**; **strategic luôn hữu ích** | DDD đầy đủ là **overhead** cho CRUD |

---

## ④ Áp dụng thực tế + So sánh bigtech

### Aggregate
**Order (root) + OrderLine (trong)**. **Invariant "tổng ≤ limit"** kiểm tại **`order.addLine()`**. Tham chiếu **`customerId` (ID)**, không nhúng Customer.

### Domain Event + Saga

```
   OrderPlaced
        ├──▶ service Inventory   trừ kho        (tx riêng)
        └──▶ service Notification gửi mail       (tx riêng)
   → async, MỖI aggregate MỘT tx → eventual consistency (không 1 tx khổng lồ)
```

### Bigtech / ecosystem (pattern phổ biến — không tuyệt đối)

| Trường hợp | Ghi chú |
|---|---|
| **event-driven microservices (Kafka)** | hiện thực **domain event** ở cấp hệ thống |
| **saga pattern** | điều phối **transaction phân tán** thay **2PC** (two-phase commit) |

> 🔎 **⚠️ Insight sâu:** **nhiều team chỉ dùng strategic DDD** (**bounded context = service boundary**) **mà bỏ tactical** cho service CRUD mỏng. Tức là họ lấy *phần rẻ-mà-giá-trị-cao* (vẽ ranh giới đúng) và bỏ *phần đắt-mà-chỉ-đáng-khi-phức-tạp* (aggregate/repo). Đó là judgment trưởng thành, không phải "làm DDD nửa vời".

---

## ⑤ Code thực hành

```typescript
// aggregate.ts — Aggregate Root bảo vệ invariant, tham chiếu bằng ID

class OrderLine {                                    // entity TRONG aggregate (không lộ ra ngoài root)
  constructor(readonly sku: string, readonly qty: number, readonly price: number) {}
  get subtotal() { return this.qty * this.price; }
}

class Order {                                         // AGGREGATE ROOT — cửa DUY NHẤT
  private lines: OrderLine[] = [];
  constructor(
    readonly id: string,
    private readonly customerId: string,              // tham chiếu aggregate khác bằng ID, KHÔNG nhúng Customer
    private readonly creditLimit: number,
  ) {}

  addLine(line: OrderLine) {                           // MỌI thay đổi QUA root
    const newTotal = this.total + line.subtotal;
    if (newTotal > this.creditLimit) throw new Error('Exceeds credit limit');  // INVARIANT kiểm tại root
    this.lines.push(line);
  }
  get total() { return this.lines.reduce((s, l) => s + l.subtotal, 0); }
}

// Repository nói NGÔN NGỮ DOMAIN (trả domain object, ẩn ORM)
interface OrderRepository {
  findActiveOrdersFor(customerId: string): Promise<Order[]>;   // theo NHU CẦU domain, không theo DB
  save(order: Order): Promise<void>;
}

// "Gửi email sau đặt đơn" => Application Service gọi port NotificationSender, KHÔNG nằm trong Order.
```

> 💡 **Đọc kỹ:** `lines` là `private` — không ai push thẳng vào từ ngoài, **bắt buộc qua `addLine()`** nơi root gác invariant `tổng ≤ creditLimit`. `customerId` là **ID** (không nhúng Customer). `findActiveOrdersFor` đặt tên **theo domain**, không phải `selectWhere(...)`. Đây là DDD tactical "đúng bài".

---

## ⑥ Keywords cần nhớ

> 🧠 **Cần HIỂU:**
> - **aggregate / root**
> - **invariant**
> - **aggregate ≈ transaction boundary**
> - **repository (ngôn ngữ domain)**
> - **domain vs app service**
> - **domain event**
> - **strategic vs tactical DDD**

> 📌 **Cần THUỘC:**
> - **"tham chiếu aggregate khác bằng ID"**.
> - **"một tx ~ một aggregate"**.
> - **"email = app service"**.

> 🛠️ **Cần LÀM ĐƯỢC:**
> - Dựng **aggregate root bảo vệ invariant**.
> - Viết **repo interface theo domain**.
> - Đặt **side-effect đúng tầng**.

---

## ⑦ Mental model

> 🎯 **Khẩu quyết:**
> **Aggregate = pháo đài có một cổng (root) bảo vệ invariant**; ra ngoài thì **gọi tên (ID)**, không **khiêng cả lâu đài**. **Một transaction một pháo đài**; muốn nhiều thì **gửi thư (domain event)**. **Strategic DDD luôn dùng; tactical chỉ khi domain đủ giàu.**

```
   Sửa trong aggregate?     → QUA root (gác invariant), không sửa lén
   Trỏ tới aggregate khác?  → bằng ID, không nhúng object
   Đổi nhiều aggregate?     → mỗi cái 1 tx + domain event (đừng 1 tx khổng lồ)
   Side-effect (email)?     → App Service qua port, KHÔNG vào domain
   Service CRUD mỏng?       → strategic OK, bỏ tactical (đừng overhead)
```

---

## ⑧ Câu hỏi phỏng vấn & thách đố

**(khó)** *Aggregate/Aggregate Root là gì? Vì sao tham chiếu aggregate khác bằng ID?*
→ **consistency boundary**; **root là cửa**; giữ **boundary/eventual consistency**.

**(khó)** *Invariant là gì? Vì sao root bảo vệ?*
→ **rule luôn đúng**; **mọi thay đổi qua root**.

**(khó)** *Aggregate boundary ≈ transaction boundary — vì sao? Đổi nhiều aggregate thì sao?*
→ **một đơn vị nhất quán**; dùng **event/saga**.

**(khó)** *Repository DDD khác DAO/ORM?*
→ **trả domain object**, **nói ngôn ngữ domain**, **ẩn persistence**.

> 🧩 **🔥 Thách đố (Q-DDD-010):** *"Service nội bộ CRUD đơn giản — áp full DDD tactical cho chuẩn?"*
>
> **Bẫy:** **overhead lớn**; **CRUD/ít rule/đội nhỏ → không đáng**. **Strategic (bounded context) thì vẫn dùng; tactical thì chọn lọc** theo độ phức tạp/invariant. Trả lời "full DDD cho chuẩn" là rơi bẫy over-engineering.

---

## ⑨ Bài tập + tiêu chí tự chấm

**BT1.** Dựng aggregate **Cart/Order** bảo vệ **một invariant** (vd **tổng ≤ limit**), tham chiếu user bằng **ID**.
→ ✅ **Đạt khi:** **không thể vi phạm invariant từ ngoài root**.

> 💡 *Gợi ý:* để mảng item là `private`, chỉ cho thêm/bớt qua method có kiểm tra invariant — y như `Order.addLine()` ở mục ⑤.

**BT2.** Viết **1 repository interface** theo **nhu cầu domain** (không lộ ORM).
→ ✅ **Đạt khi:** method đặt tên **domain** (`findOverdueInvoices`), **trả domain object**.

---

## ⑩ Đọc thêm

- **Eric Evans** — **DDD**; **Vaughn Vernon** — **Implementing DDD** (**aggregate design rules**).
- **Fowler** — **Saga / eventual consistency**.

---

## 🤖 Hiểu để chỉ huy (Kiểm tra AI)

> 🚨 **AI rất hay:**
> 1. **Nhúng object thay vì ID** (`order.customer = customer` thay vì `customerId`).
> 2. **Nhét side-effect (email) vào domain**.
> 3. **Để repo trả thẳng ORM entity**.
> 4. **Đề xuất full-DDD cho CRUD**.
>
> 🎯 **Ba câu soát mỗi lần nhận code domain từ AI:**
> - *"**Invariant** có được **root** bảo vệ không, hay sửa lén được từ ngoài?"* → sửa lén được → ép qua root + `private`.
> - *"**Side-effect** (email/HTTP) có **đúng tầng** không?"* → trong domain → kéo ra Application Service qua port.
> - *"**DDD** có cần ở đây không, hay CRUD mỏng?"* → mỏng → cắt tactical, giữ strategic.
>
> AI thuộc lý thuyết DDD nhưng hay áp *máy móc* (full tactical) hoặc *cẩu thả* (nhúng object, email trong domain). Vai trò chỉ huy của bạn: **đảm bảo root gác invariant**, **side-effect đúng tầng**, và **cân tactical theo độ phức tạp thật** — không làm DDD vì "cho chuẩn".
