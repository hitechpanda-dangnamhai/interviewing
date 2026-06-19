# 🎯 BÀI 11 — Kiến trúc (1): **Layered** · **Clean** · **Hexagonal** · **Onion** · **Dependency Rule** · **Screaming**

**Phủ:** Q-ARCH-001 → Q-ARCH-002 → Q-ARCH-003 → Q-ARCH-004 → Q-ARCH-005 → Q-ARCH-006 → Q-ARCH-009

> 💡 **Mental model mở đầu:**
> Bài này là **DIP (Bài 6) nâng lên cấp KIẾN TRÚC**. Cùng một ý — *đảo chiều mũi tên phụ thuộc* — nhưng giờ áp cho cả hệ thống: **tách domain khỏi infra**, mọi mũi tên **chĩa vào trong**.
> Hiểu xong, bạn **đọc được** vì sao `import { PrismaClient }` trong **entity** là **sai**, và vì sao **repo interface nằm ở domain**.

---

## ① Mục tiêu & vị trí trong mạch

Đây là **DIP (Bài 6) nâng lên cấp kiến trúc**: tách **domain** khỏi **infra**, **đảo chiều phụ thuộc vào trong**.

```
   Bài 6: DIP (đảo mũi tên ở cấp CLASS)
                    │  nâng lên cấp KIẾN TRÚC
                    ▼
   Bài 11: tách DOMAIN khỏi INFRA, mũi tên chĩa vào trong (bài này)
                    │
              Bài 12 (anemic, CQRS, refactor monolith)
              mục H-MOD (tổ chức module Nest cụ thể)
```

> 🎯 **Chốt vị trí:** hiểu xong bài này bạn đọc được vì sao `import { PrismaClient }` trong **entity** là **sai**, vì sao **repo interface nằm ở domain**. **Bài 12** lo phần nâng cao (**anemic, CQRS, refactor monolith**); cách tổ chức **module Nest** cụ thể ở **mục H-MOD**.

---

## ② Giảng cơ bản → nâng cao

### (a) Layered architecture (Q-ARCH-001)

> **Ẩn dụ:** toà nhà nhiều tầng — tầng trên đứng trên tầng dưới, không có chuyện tầng dưới "với lên" gọi tầng trên.

**Layered architecture** = tách trách nhiệm theo tầng **presentation / business / data**; **phụ thuộc đi một chiều xuống dưới**; tầng trên không bị tầng dưới gọi ngược.

> 💡 **Lợi:** đổi một tầng (vd đổi DB) **ít ảnh hưởng** tầng khác.

---

### (b) ⚠️ The Dependency Rule (Clean Architecture — Q-ARCH-002)

> **Ẩn dụ:** lõi Trái Đất (domain) là phần quý & ổn định nhất, được các lớp vỏ (infra) bao quanh bảo vệ. Vỏ phụ thuộc lõi, không phải ngược lại.

**The Dependency Rule** = **phụ thuộc chỉ được trỏ VÀO TRONG** — về phía **policy/domain**. **Domain/business KHÔNG phụ thuộc framework/DB/web.**

> 💡 **Vì sao bắt domain "sạch" như vậy?** Vì **domain là phần ỔN ĐỊNH & GIÁ TRỊ nhất** — logic nghiệp vụ ít đổi hơn nhiều so với "dùng Postgres hay Mongo", "REST hay GraphQL". Tách domain khỏi chi tiết → **dễ test**, **thay hạ tầng không đụng nghiệp vụ**.

> 🔎 **Đây là khác biệt CỐT LÕI** giữa hai kiểu:

```
LAYERED "NGÂY THƠ"                    CLEAN
   business ──▶ data (DB)               business ◀── data  (qua INTERFACE)
   (business "biết" DB)                 (data phụ thuộc abstraction của business)
   ❌ đổi DB → đụng business             ✅ đổi DB → business im
```

---

### (c) Hexagonal / Ports & Adapters (Q-ARCH-003)

> **Ẩn dụ:** app core là một thiết bị có nhiều **cổng cắm chuẩn** (port). Bất cứ thiết bị ngoài nào (DB, mail, HTTP) muốn nói chuyện đều phải làm một **đầu cắm vừa cổng** (adapter). Core không quan tâm đầu cắm là hãng nào.

| Khái niệm | Là gì |
|---|---|
| **Port** | **interface do DOMAIN định nghĩa** (cái domain *cần*) |
| **Adapter** | **implementation cụ thể** nối ra ngoài |
| **Driving / primary** | thứ **gọi VÀO** app (HTTP controller, CLI, test) |
| **Driven / secondary** | thứ **app gọi RA** (DB, mail, queue) |

```
   [Driving adapter]          APP CORE             [Driven adapter]
   HTTP controller ──gọi vào──▶ (domain) ──gọi ra──▶ DB / mail / queue
   CLI / test                  qua PORT             qua PORT
                    App core KHÔNG biết adapter nào đang cắm
```

---

### (d) Clean = Onion = Hexagonal? (Q-ARCH-004)

> 🎯 **Về bản chất là CÙNG MỘT tư tưởng:** **tách domain khỏi infra + đảo chiều phụ thuộc vào trong.** Khác **chỉ ở thuật ngữ / cách vẽ / nhấn mạnh.**

| Tên | "Đặc sản" cách gọi |
|---|---|
| **Clean Architecture** | nhấn **Dependency Rule** (vòng tròn đồng tâm) |
| **Onion Architecture** | nhấn các **lớp vỏ hành** quanh domain |
| **Hexagonal** | nhấn **ports & adapters** (hình lục giác nhiều cổng) |

> 💡 **Đừng tranh cãi tên** — hiểu **nguyên tắc chung**. Phỏng vấn viên hỏi "khác nhau gì" thường để xem bạn có *bị kẹt ở tên gọi* hay *thấy được cái chung*.

---

### (e) ⚠️ Vì sao repo interface ở domain, impl ở infra (Q-ARCH-005)

```
   DOMAIN định nghĩa: OrderRepository (PORT = abstraction)
        ▲
        │ implement (mũi tên phụ thuộc trỏ TỪ NGOÀI VÀO domain)
        │
   INFRA: PrismaOrderRepository (impl port)
```

> 💡 **Cơ chế đảo chiều:** **Domain SỞ HỮU port** (abstraction). **Infra implement port** → **mũi tên phụ thuộc trỏ từ ngoài vào domain** (**DIP ở cấp kiến trúc**). Domain **không import infra**, chỉ import **abstraction của chính nó**. Quyền "ra luật" thuộc về domain; infra phải *tuân theo*.

---

### (f) ⚠️ `import { PrismaClient }` trong entity = sai (Q-ARCH-006)

> 🚨 **CẢNH BÁO — lỗi kiến trúc kinh điển:**
> ```
> // domain/Order.ts
> import { PrismaClient } from '@prisma/client';   // ❌ DOMAIN rò rỉ phụ thuộc ORM/infra
> ```
> **Hậu quả:** **đảo ngược dependency rule**; **khó test / thay DB**; **business trộn persistence**. **Điều tệ nhất:** muốn unit test một quy tắc nghiệp vụ thuần (vd "total > 0") cũng phải kéo theo cả Prisma + kết nối DB.
> **Sửa:** định nghĩa **repository port ở domain** + **adapter Prisma ở tầng ngoài**.

---

### (g) Screaming Architecture (Q-ARCH-009 — Uncle Bob)

> **Ví dụ trực giác:** nhìn bản vẽ một toà nhà, bạn đoán ngay "đây là *bệnh viện*" hay "*thư viện*" — chứ không phải "xây bằng *gạch* hay *bê tông*". Thư mục code cũng nên *hét lên* nó *làm gì*.

**Screaming Architecture** = cấu trúc thư mục nên **"hét lên" domain/use case** (`orders/`, `billing/`) **chứ không phải framework/layer** (`controllers/`, `services/`, `models/`).

```
❌ HÉT FRAMEWORK              ✅ HÉT DOMAIN (Screaming)
   controllers/                 orders/
   services/                    billing/
   models/                      shipping/
   (đoán: "viết bằng MVC")      (đoán: "đây là hệ thống bán hàng")
```

> 💡 **Lợi:** nhìn cây thư mục **biết hệ thống LÀM GÌ**, không phải "viết **bằng gì**". Nối với **feature-module** (cross-ref **H-MOD**).

---

## ③ ⚠️ Cũ → Mới

| Cũ / ngây thơ | Nay (clean) | Vì sao |
|---|---|---|
| **business import DB/ORM trực tiếp** | business định nghĩa **port**, infra implement | **Dependency Rule**: phụ thuộc trỏ **vào trong** |
| **Entity chứa `PrismaClient`/SQL** | Entity **thuần domain**; persistence ở **adapter** | Tách domain khỏi infra → **test/thay được** |
| **Thư mục theo layer** (`controllers/`, `models/`) | Thư mục theo **domain/use case** (**Screaming**) | Cấu trúc phản ánh hệ thống **làm gì** |
| **Tranh cãi Clean vs Onion vs Hexagonal** | **Hiểu cùng một tư tưởng** | Khác tên, chung nguyên tắc |

---

## ④ Áp dụng thực tế + So sánh bigtech

### Use case: port ở domain, adapter ở infra

```
   Controller (DRIVING adapter)
        │ gọi
        ▼
   PlaceOrderUseCase (business) ──phụ thuộc──▶ OrderRepository (PORT, ở domain)
                                                      ▲
                                                      │ cắm vào
                                          PrismaOrderRepository (DRIVEN adapter, ở infra)
   → Đổi sang TypeORM = thêm adapter mới, DOMAIN im
```

### Bigtech / ecosystem (pattern phổ biến — không tuyệt đối)

| Trường hợp | Cách làm |
|---|---|
| **Netflix / Uber** backend | tách **"core domain"** khỏi infra để **swap datastore/transport** |
| **Spring / NestJS** | hỗ trợ **DI** để hiện thực **port-adapter** |

> 🚨 **⚠️ Nhưng:** nhiều **startup cố tình dùng layered đơn giản** hơn khi **domain mỏng** (Bài 12 — trade-off). Clean Architecture *có chi phí* (nhiều file, nhiều abstraction) — chỉ "trả phí" khi domain đủ dày để xứng.

> 🔎 **Insight sâu:** Netflix/Uber tách core domain *không phải vì sách bảo thế*, mà vì họ **thật sự cần swap datastore/transport** ở quy mô lớn. Kiến trúc theo *nhu cầu thật*, không theo *thời trang* — đúng tinh thần "áp dụng khi đau thật" (Bài 3, Bài 6).

---

## ⑤ Code thực hành

```typescript
// hexagonal.ts — port ở domain, adapter ở infra

// ===== DOMAIN LAYER (KHÔNG import infra) =====
export interface Order { id: string; total: number; }

export interface OrderRepository {                 // PORT do DOMAIN định nghĩa (cái domain CẦN)
  findById(id: string): Promise<Order | null>;
  save(order: Order): Promise<void>;
}

export class PlaceOrderUseCase {                   // BUSINESS: phụ thuộc PORT, KHÔNG phụ thuộc DB
  constructor(private readonly repo: OrderRepository) {}
  async exec(order: Order) {
    if (order.total <= 0) throw new Error('Invalid total');  // DOMAIN RULE (thuần, test được không cần DB)
    await this.repo.save(order);                             // gọi qua PORT, kệ DB nào bên dưới
  }
}

// ===== INFRA LAYER (ADAPTER implement port; phụ thuộc trỏ VÀO domain) =====
// import { PrismaClient } from '@prisma/client';   // verify version ở docs chính thức
export class PrismaOrderRepository implements OrderRepository {
  // constructor(private prisma: PrismaClient) {}
  async findById(id: string) { /* prisma.order.findUnique... */ return null; }
  async save(order: Order) { /* prisma.order.create... */ }
}

// ❌ SAI: trong domain/Order.ts mà `import { PrismaClient }`
//    -> DOMAIN rò infra (vi phạm Dependency Rule)
```

> 💡 **Cấu hình (enforce):** dùng **`dependency-cruiser`** / **`eslint-plugin-boundaries`** (**đã verify**) để **chặn `domain/` import `infra/`** trong **CI** — biến **Dependency Rule** thành **lint** (xem **Bài 15**). Quy tắc kiến trúc *được máy gác* mới không bị xói mòn theo thời gian.

---

## ⑥ Keywords cần nhớ

> 🧠 **Cần HIỂU:**
> - **layered**
> - **Dependency Rule**
> - **port / adapter**
> - **driving vs driven**
> - **Clean = Onion = Hexagonal**
> - **repo port ở domain**
> - **Screaming Architecture**

> 📌 **Cần THUỘC:**
> - **"phụ thuộc trỏ vào trong"**.
> - Vì sao **entity không được import ORM**.

> 🛠️ **Cần LÀM ĐƯỢC:**
> - Đặt **port ở domain** + **adapter ở infra**.
> - Nhận diện **vi phạm dependency rule** trong code.

---

## ⑦ Mental model

> 🎯 **Khẩu quyết:**
> **Domain ở lõi, không biết thế giới bên ngoài. Mọi mũi tên phụ thuộc đều chĩa vào trong. Port = cái domain cần; Adapter = cái thế giới cắm vào. Thư mục phải "hét" domain, không "hét" framework.**

```
   domain/  ←── infra/   (mũi tên trỏ VÀO trong)
      │
      ├── định nghĩa PORT (cái cần)
      └── KHÔNG import gì từ ngoài
```

---

## ⑧ Câu hỏi phỏng vấn & thách đố

**(TB)** *Layered architecture + quy tắc phụ thuộc giữa tầng?*
→ **tách tầng**, **phụ thuộc một chiều xuống**.

**(khó)** *Dependency Rule phát biểu sao? Vì sao domain không phụ thuộc framework/DB?*
→ **trỏ vào trong**; domain **ổn định / test được**.

**(khó)** *Port vs adapter; driving vs driven?*
→ **interface domain / impl**; **gọi-vào / app-gọi-ra**.

**(khó)** *Vì sao repo interface ở domain mà impl ở infra?*
→ domain **sở hữu abstraction**, **đảo chiều (DIP)**.

> 🧩 **🔥 Thách đố (Q-ARCH-006):** *"Thấy `import { PrismaClient }` trong entity domain — sai chỗ nào?"*
>
> **Bẫy:** **domain rò infra**, **đảo ngược dependency rule**, **khó test / thay DB**. **Sửa:** **port + adapter**. Trả lời "không sao, chạy được mà" là rơi đúng cái bẫy "đúng cú pháp nhưng sai kiến trúc".

---

## ⑨ Bài tập + tiêu chí tự chấm

**BT1.** Tách một service đang gọi **ORM trực tiếp** thành **use case + port + adapter**.
→ ✅ **Đạt khi:** **domain/use case không import gì từ ORM**; test **inject in-memory repo**.

> 💡 *Gợi ý:* định nghĩa interface repo *trong domain*, để class Prisma/TypeORM implement nó *ở infra*; trong test cắm một `InMemoryRepo` thoả interface.

**BT2.** Đổi cấu trúc thư mục từ **layer** sang **Screaming** (theo domain).
→ ✅ **Đạt khi:** nhìn thư mục **đoán được nghiệp vụ**, không phải framework.

---

## ⑩ Đọc thêm

- **R. C. Martin** — **Clean Architecture**; **"Screaming Architecture"** (blog).
- **Alistair Cockburn** — **Hexagonal / Ports & Adapters**.
- **Jeffrey Palermo** — **Onion Architecture**.

---

## 🤖 Hiểu để chỉ huy (Kiểm tra AI)

> 🚨 **AI rất hay nhét ORM/HTTP thẳng vào `domain/entity`** "vì chạy được" → **vi phạm dependency rule**.
>
> 🎯 **Câu soát mỗi lần nhận code kiến trúc từ AI:**
> - *"**domain** có **import infra** không (Prisma/axios/express...)?"* → có → ép đổi sang **port + adapter**.
>
> 🔎 **Đây chính là chỗ "AI viết ĐÚNG CÚ PHÁP nhưng SAI KIẾN TRÚC".** Code biên dịch được, chạy được, test happy-path xanh — nhưng phụ thuộc đã trỏ sai chiều. Mắt người dễ bỏ sót, nên cần **guardrail lint** (`dependency-cruiser`/`eslint-plugin-boundaries`) gác ở **CI** (Bài 15). Vai trò chỉ huy của bạn: **không tin "chạy được" là "đúng kiến trúc"**, và **để máy gác Dependency Rule** thay vì trông chờ review thủ công.
