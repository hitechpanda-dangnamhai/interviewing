# 🎯 BÀI 6 — **SOLID** (3): **Dependency Inversion** & **DI** · SOLID trong thực chiến

**Phủ:** Q-SOLID-009 → Q-SOLID-010 → Q-SOLID-011 → Q-SOLID-012 → Q-SOLID-013 → Q-SOLID-014

> 💡 **Mental model mở đầu:**
> - **DIP** là chữ **"đắt giá" nhất** trong SOLID — nó là **nền của Clean / Hexagonal Architecture** (Bài 11) và của **DI container** trong Nest.
> - Bài này cũng **đóng phần SOLID bằng trade-off**: SOLID *có thể* là **over-engineering**. Học cách *không* áp SOLID máy móc cũng quan trọng như học SOLID.
> Hai ý lớn: **đảo chiều mũi tên phụ thuộc** + **biết liều lượng**.

---

## ① Mục tiêu & vị trí trong mạch

**DIP** là chữ **đắt giá nhất** — nền của **Clean / Hexagonal Architecture** (Bài 11) và **DI container** của Nest. Bài này cũng **đóng phần SOLID** bằng **trade-off**: SOLID có thể là **over-engineering**, và cách phản hồi review *"áp SOLID máy móc"*. Nối thẳng tới **judgment** (Bài 15).

```
   SRP/OCP (Bài 4) + LSP/ISP (Bài 5)
                    │
   Bài 6:  DIP — đảo chiều phụ thuộc  ──▶ nền Clean/Hexagonal (Bài 11)
           DI  — kỹ thuật cắm dependency ──▶ DI container (Nest, mục H-DI)
                    │
           TRADE-OFF: SOLID có thể over-engineer ──▶ judgment (Bài 15)
```

> 🎯 **Chốt vị trí:** đây là bài *kết* của khối SOLID. Nó vừa dạy chữ khó nhất (**DIP**), vừa dạy **sự kiềm chế** — biết khi nào *đừng* áp SOLID. Hai mặt của một người senior.

---

## ② Giảng cơ bản → nâng cao

### (a) DIP — Dependency Inversion Principle (hai vế — Q-SOLID-009)

> **Ẩn dụ — ổ cắm điện chuẩn quốc gia.** Nhà máy điện (low-level) và cái tivi (high-level) **không nối thẳng** vào nhau. Cả hai cùng tuân theo **chuẩn ổ cắm** (abstraction). Đổi nhà máy điện, tivi vẫn chạy; đổi tivi, nhà máy không quan tâm. Cái *chuẩn ổ cắm* mới là trung tâm, không phải cái nào trong hai.

**Hai vế của DIP:**

| Vế | Nội dung |
|---|---|
| Vế 1 | **Module cấp cao** (business/policy) **KHÔNG phụ thuộc** **module cấp thấp** (DB/HTTP/driver); **cả hai** phụ thuộc **abstraction** |
| Vế 2 | **Abstraction KHÔNG phụ thuộc chi tiết**; **chi tiết phụ thuộc abstraction** |

- **"high-level"** = **logic nghiệp vụ** — *cái vì sao app tồn tại* (vd: "đặt đơn hàng", "tính giá").
- **"low-level"** = **chi tiết kỹ thuật** — Postgres, SMTP, driver cụ thể.

**DIP đảo chiều mũi tên phụ thuộc:**

```
❌ KHÔNG DIP (mũi tên xuôi)            ✅ CÓ DIP (mũi tên ĐẢO)
   business ──▶ DB (Postgres)            business ──▶ interface ◀── DB-impl
   (business "biết" Postgres)            (cả hai trỏ vào interface ở giữa;
   đổi DB = sửa business                  business KHÔNG biết Postgres tồn tại)
```

> 💡 **Vì sao gọi là "inversion" (đảo)?** Vì theo lẽ thường, code "cao" gọi code "thấp" nên *mũi tên phụ thuộc xuôi từ cao xuống thấp*. DIP **bẻ ngược** lại: chi tiết thấp giờ *phụ thuộc vào* abstraction do tầng cao định nghĩa. Quyền lực đảo về tay business logic — nó **ra luật** (interface), hạ tầng phải *tuân theo*.

---

### (b) ⚠️ DIP ≠ DI (Q-SOLID-010) — bẫy phỏng vấn kinh điển

> 🚨 **Đây là bẫy phỏng vấn bị sập nhiều nhất.** Hai từ na ná nhau nhưng là **hai phạm trù khác hẳn**.

| | **DIP** | **DI (Dependency Injection)** |
|---|---|---|
| Bản chất | **nguyên tắc** (thiết kế) | **kỹ thuật** (cách làm) |
| Nói gì | phụ thuộc **abstraction** | **cung cấp dependency từ NGOÀI vào** (constructor / setter / container) |

> 🚨 **CẢNH BÁO — "có DI chưa chắc có DIP":**
> ```
> constructor(private ses: SesEmailSender) { }   // inject concrete class
> ```
> Đây **vẫn là DI** (bạn inject từ ngoài vào), **nhưng KHÔNG đạt DIP** — vì bạn vẫn **phụ thuộc chi tiết** (class cụ thể `SesEmailSender`).
> **Đạt DIP cần inject qua interface / abstraction:**
> ```
> constructor(private notifier: NotificationSender) { }   // ✅ DI + DIP
> ```

> 🔎 *(Cơ chế **container** Nest — **provider / scope / useFactory** — thuộc **mục H-DI**; ở đây ta hiểu **vì sao** thiết kế như thế, chưa đi vào cú pháp container.)*

---

### (c) Vì sao DIP → dễ test (Q-SOLID-011)

```
   business phụ thuộc ABSTRACTION (NotificationSender)
                    │
   Trong test: thay implementation THẬT (SES) bằng TEST DOUBLE
               (mock / stub / fake)
                    │
                    ▼
✅ Tách business khỏi I/O thật (DB / network) → unit test NHANH & XÁC ĐỊNH
```

> 💡 **Vì sao quan trọng?** Test gọi SES/DB thật thì **chậm** (mạng), **không xác định** (mạng lỗi → test đỏ oan), và **tốn tiền/rủi ro** (gửi mail thật). Nhờ DIP, bạn *cắm một fake vô hại* vào → test chạy trong mili-giây, kết quả luôn nhất quán. **Test seam** (khe để chèn test double) là phần thưởng trực tiếp của DIP.

---

### (d) SOLID ↔ coupling/cohesion (Q-SOLID-013)

| Nhóm chữ | Kéo gì | Cơ chế |
|---|---|---|
| **SRP + ISP** | **cohesion ↑** | gom đúng trách nhiệm, interface gọn |
| **OCP + DIP** | **coupling ↓** | phụ thuộc abstraction, không sửa lan |

> 🎯 **Chốt:** mục tiêu cuối vẫn là **loose coupling + high cohesion** (Bài 1). SOLID chỉ là 5 con đường cụ thể dẫn về đúng cái đích của la bàn. Quay lại đúng nơi xuất phát.

---

### (e) ⚠️⚠️ SOLID có thể là over-engineering (Q-SOLID-012 & 014) — phần Tech Lead

> **Ví dụ trực giác:** SOLID là **thuốc** — đúng liều thì khỏi bệnh, **quá liều thì ngộ độc**. Áp SOLID vào một script 30 dòng dùng một lần là "uống cả vỉ thuốc cho chắc".

**Khi nào SOLID thành gánh nặng?**
Dự án **nhỏ / script / throwaway** (dùng một lần), hoặc **abstraction sớm** khi **chưa rõ trục biến thiên** (chưa biết cái gì sẽ thay đổi) → thêm class/interface/**indirection** vô ích.

> 🎯 **Nguyên tắc vàng:** *"áp dụng khi đau THẬT"*, **không giáo điều**.

**Review case (Q-SOLID-014):**

```
   Junior tạo INTERFACE 1-1 cho MỌI class
   (mỗi class một interface chỉ-có-một-implementation) "để cho SOLID"
                    │
                    ▼
❌ Đây là OVER-ABSTRACTION (vi phạm YAGNI):
   interface chỉ có GIÁ TRỊ khi:
     • có ≥2 implementation, HOẶC
     • cần đảo chiều phụ thuộc / test seam THẬT
```

> 💡 **Cách phản hồi review (quan trọng — kỹ năng Tech Lead):** **dẫn dắt bằng câu hỏi**, không chỉ chê:
> - *"Interface này có **implementation thứ 2** nào sắp tới không?"*
> - *"Có cần **seam** để **test** không?"*
> Khen tinh thần cẩn thận, rồi để họ **tự thấy** chỗ thừa. Chê thẳng làm junior phòng thủ; hỏi đúng câu giúp họ tự suy ra.

---

## ③ ⚠️ Kiến thức cũ → Mới

| Cách cũ / hiểu sai | Nên dùng nay | Vì sao |
|---|---|---|
| **Business import thẳng `PgClient`/`StripeClient`** | Business phụ thuộc **port/interface**, infra implement | **Đảo chiều phụ thuộc (DIP)** → test/thay được |
| **"Inject là đã DIP"** | **DI ≠ DIP**; phải inject qua **abstraction** | Inject **concrete** vẫn là **phụ thuộc chi tiết** |
| **Interface 1-1** cho mọi class "cho SOLID" | Interface khi có **≥2 impl** hoặc cần **seam** thật | Tránh **over-abstraction (YAGNI)** |

---

## ④ Áp dụng thực tế + So sánh bigtech

### Use case: OrderService + port/adapter

```
   OrderService (HIGH-LEVEL)
        │ phụ thuộc
        ▼
   OrderRepository (PORT = interface)
        ▲                    ▲
        │ implement          │ implement
   PostgresOrderRepository   InMemoryOrderRepository
   (LOW-LEVEL, production)   (cho TEST)
```

> 💡 Đổi sang Mongo = **thêm một adapter** (`MongoOrderRepository`), **không đụng `OrderService`**. Test = inject **`InMemoryOrderRepository`**. Đây là **port & adapter** — chính là DIP ở dạng thực chiến (sẽ thành kiến trúc đầy đủ ở **Bài 11**).

### Bigtech / ecosystem (pattern phổ biến — không tuyệt đối)

| Hệ sinh thái | Cách hiện thực DIP ở quy mô lớn |
|---|---|
| **Spring (Java)**, **NestJS**, **Angular** | xây quanh **DI container** để hiện thực **DIP** ở quy mô lớn |
| **AWS Lambda + interface** | **swap implementation** theo môi trường (local/cloud) |

> 🔎 **Insight sâu:** cả một thế hệ framework backend (Spring → Nest) đặt **DI container** làm *trái tim*. Vì sao? Vì khi app lớn lên, việc **"ai cắm cái gì vào ai"** trở nên phức tạp — container tự động hoá phần wiring đó, để business logic chỉ cần *khai báo "tôi cần một `NotificationSender`"* mà không cần biết ai cung cấp.

---

## ⑤ Code thực hành

```typescript
// dip-vs-di.ts — phân biệt DI và DIP

// 1) ABSTRACTION do HIGH-LEVEL định nghĩa (đây là "port")
export interface NotificationSender {
  send(to: string, msg: string): Promise<void>;
}

// 2) HIGH-LEVEL module phụ thuộc ABSTRACTION (DIP) + nhận qua constructor (DI)
export class OrderService {
  constructor(private readonly notifier: NotificationSender) {}   // DI + DIP (qua interface)
  async placeOrder(userEmail: string) {
    // ... business logic ...
    await this.notifier.send(userEmail, 'Order placed');          // KHÔNG biết Email/SMS/SES cụ thể
  }
}

// 3) LOW-LEVEL chi tiết IMPLEMENT port (chi tiết phụ thuộc abstraction)
export class SesEmailSender implements NotificationSender {
  async send(to: string, msg: string) { /* gọi AWS SES — verify SDK ở docs chính thức */ }
}

// ⚠️ PHẢN VÍ DỤ: constructor(private ses: SesEmailSender)
//    -> VẪN là DI (inject từ ngoài) nhưng KHÔNG đạt DIP (phụ thuộc concrete class).

// 4) TEST SEAM nhờ DIP:
// const fake: NotificationSender = { send: async () => {} };
// new OrderService(fake).placeOrder('a@b.com');   // unit test, KHÔNG gọi SES thật
```

> 💡 **Cấu hình:** **wiring** cụ thể (chọn `SesEmailSender` ở **production**, `FakeSender` ở **test**) do **DI container** lo — xem **mục H-DI** cho cách Nest **`useClass` / `useFactory`**. **Không hardcode key**; **SES credential** qua **biến môi trường / secret manager**.

---

## ⑥ Keywords cần nhớ

> 🧠 **Cần HIỂU:**
> - **DIP** (2 vế, **đảo chiều phụ thuộc**)
> - **DIP ≠ DI** — nguyên tắc vs kỹ thuật
> - vì sao **DIP → testable** (cắm **test double**)
> - **SOLID ↔ coupling/cohesion**
> - **SOLID có thể over-engineer**

> 📌 **Cần THUỘC:**
> - Phát biểu **2 vế DIP**.
> - Câu: **"inject concrete = DI nhưng KHÔNG DIP"**.

> 🛠️ **Cần LÀM ĐƯỢC:**
> - Đảo một phụ thuộc **concrete** thành **port + adapter**.
> - Phản hồi review **"interface 1-1"** mang tính **dẫn dắt** (hỏi, không chê).

---

## ⑦ Mental model

> 🎯 **Khẩu quyết:**
> - **DIP:** **đảo mũi tên** — business **định nghĩa interface**, hạ tầng **cắm vào**.
> - **DI** chỉ là cách **"đưa đồ vào tận tay"**.
> - Và: **SOLID là thuốc — đúng liều thì khỏi, quá liều thì ngộ độc** (over-engineering).

```
   Business "biết" tên DB/SDK cụ thể?  có → đảo mũi tên (DIP): tạo port
   Inject nhưng inject CONCRETE?        → chưa DIP, đổi sang inject INTERFACE
   Tạo interface mà chỉ 1 impl?         → hỏi "có impl#2 / seam test?" không → BỎ
```

---

## ⑧ Câu hỏi phỏng vấn & thách đố

**(khó)** *DIP phát biểu 2 vế nào? high/low-level là gì?*
→ **business vs chi tiết**; **cả hai phụ thuộc abstraction**; abstraction không phụ thuộc chi tiết.

**(khó)** *DIP khác DI thế nào? Có DI là có DIP không?*
→ **nguyên tắc vs kỹ thuật**; **inject concrete vẫn KHÔNG DIP**.

**(TB)** *Vì sao lập trình theo interface dễ test hơn?*
→ **thay impl thật bằng test double** (mock/stub/fake) → tách khỏi I/O thật.

**(rất khó)** *SOLID bao giờ là over-engineering? Khi nào cố tình không áp triệt để?*
→ **script/nhỏ/throwaway**; **abstraction sớm** khi **chưa rõ trục biến thiên**.

> 🧩 **🔥 Thách đố (review):** *junior tạo **interface 1-1** cho mọi class "cho SOLID" — bạn nói gì?*
>
> **Bẫy:** **khen tinh thần** nhưng **chỉ ra over-abstraction**; interface đáng có khi có **≥2 impl** hoặc **seam test**. **Dẫn dắt bằng câu hỏi, không chỉ chê.** Trả lời "chê thẳng nó sai" là rơi bẫy về kỹ năng mentor — đúng là phải hỏi để họ *tự thấy*.

---

## ⑨ Bài tập + tiêu chí tự chấm

**BT1.** Lấy **1 service đang import thẳng client DB/HTTP**. **Tách port + adapter**, inject qua **interface**.
→ ✅ **Đạt khi:** viết được **unit test** cho service mà **không chạm I/O thật**.

> 💡 *Gợi ý:* định nghĩa interface ở phía service (high-level định luật), để client cụ thể implement nó; trong test inject một object fake thoả interface.

**BT2.** Soát repo bạn tìm **1 interface 1-1 vô nghĩa**, quyết định **giữ hay bỏ** kèm **lý do**.
→ ✅ **Đạt khi:** lý do dựa trên **"có impl thứ 2 / seam test không"**, **không phải "cho SOLID"**.

---

## ⑩ Đọc thêm

- **R. C. Martin** — **"The Dependency Inversion Principle"** + **Clean Architecture** (DIP ở cấp kiến trúc → **Bài 11**).
- **NestJS docs** — **Custom providers** (cơ chế **DI**; cross-ref **H-DI**).

---

## 🤖 Hiểu để chỉ huy (Kiểm tra AI)

> 🚨 **AI rất hay mắc HAI lỗi ngược chiều:**
> 1. **Inject concrete class** (DI mà **không DIP**) — `constructor(private ses: SesEmailSender)`.
> 2. **Đẻ interface 1-1 thừa** — tạo abstraction cho thứ chỉ có một implementation.
>
> 🎯 **Hai câu soát mỗi lần nhận code từ AI:**
> - *"Phụ thuộc có trỏ vào **abstraction** không, hay vào **concrete class**?"* → concrete → ép đổi sang **port/interface** (DIP).
> - *"Interface này có **lý do tồn tại** (impl thứ 2 / test seam) không?"* → không → ép **bỏ bớt** (YAGNI).
>
> AI hiểu DIP/DI nửa vời: nó "biết phải inject" nhưng hay inject sai cái (concrete), và "biết SOLID hay" nên hay over-abstract. Vai trò chỉ huy của bạn cân **cả hai phía**: vừa **bắt đảo chiều phụ thuộc khi cần**, vừa **cắt abstraction thừa khi không cần** — đúng tinh thần "thuốc đúng liều".
