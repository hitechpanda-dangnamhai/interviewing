# 🧩 Mục Q — Software Design & Patterns — Giáo trình đầy đủ để học

> **WORKFLOW 2 — Bước B (dựng giáo trình ngược từ bộ đề) + Bước C (giảng trọn từng bài).**
> Nguồn: `QB_Q_designpatterns.md` (90 câu, 9 mục con). Mỗi bài bám đúng các câu nó phủ, theo **Hợp đồng đầu ra 10 mục**.
> Ngôn ngữ: tiếng Việt, giữ thuật ngữ Anh. Code: **TypeScript/Node** (khớp roadmap Tech Lead Backend).
> **Tính ổn định:** ~90% nội dung Q là kiến thức kinh điển ổn định (GoF 1994 · SOLID — R. C. Martin · *Clean Architecture* 2017 · *DDD* — Eric Evans 2003 · *Refactoring* — Fowler). Chỉ phần **tooling/cú pháp đổi nhanh** mới cần verify — đã verify ở dưới và đánh dấu rõ.

---

## 🔎 Ghi chú "đã verify" (tính tại 6/2026) — đọc một lần, dùng xuyên suốt

Hai điểm tooling mà bộ đề đánh dấu *(verify)*, đã kiểm chứng để khỏi lặp lại trong từng bài:

**1) TypeScript decorators — legacy vs TC39 Stage 3 (liên quan Q-GOFS-004, Q-OOP-010, Q-TL-004/007)**
- TS **5.0 (3/2023)** ship **TC39 Stage 3 decorators** làm mặc định, **không cần flag**. Tới TS **5.9** decorator metadata được đưa lên *stable*.
- **Stage 3 KHÔNG hỗ trợ** *parameter decorators* và `emitDecoratorMetadata`. Đây chính là lý do **NestJS vẫn dùng legacy** `experimentalDecorators: true` + `emitDecoratorMetadata: true` + `reflect-metadata`: DI của Nest cần parameter decorator + runtime type metadata. Một scaffold `nest new` mới tinh **vẫn** bật hai flag legacy này.
- **Hệ quả cho việc học:** khi ta dùng `@Injectable`, `@Get`… để minh hoạ, đó là **decorator legacy của Nest** — *khác* GoF Decorator pattern (xem Bài 8). Hai chế độ legacy/Stage-3 **không trộn lẫn** trong cùng một `tsconfig` (flag áp theo cả compilation, không theo từng file).
- *Nguồn:* TypeScript 5.0 release notes (typescriptlang.org), `tc39/proposal-decorators`, nestjs/nest issue #11414.

**2) Công cụ enforce kiến trúc trong CI (liên quan Q-TL-004, Q-ARCH-006)**
- Thật và đang dùng phổ biến 2026: **`eslint-plugin-boundaries`** (v6.x), **`dependency-cruiser`** (`depcruise`), ESLint `no-restricted-imports`, **Sheriff** (`@softarc/sheriff-core`), Nx module boundaries, và TS **project references**.
- Vai trò: biến "dependency rule" (domain không import infra) thành **lint chặn ở CI** thay vì dựa vào kỷ luật con người.
- *Nguồn:* npm `eslint-plugin-boundaries`, `dependency-cruiser` docs, các guide kiến trúc TS 2026.

> Quy ước trong tài liệu: 📘 = có/ngầm trong tinh thần bộ đề · ➕ = bổ sung nâng cao của giảng viên · ⚠️ = bẫy/điểm dễ sai · 🤖 = lưu ý "hiểu để chỉ huy & kiểm tra AI".

---

# 📚 BƯỚC B — Giáo trình ngược: từ 90 câu → 15 bài (cơ bản → nâng cao)

Bộ đề sắp theo *mục con*, nhưng để **học** thì phải theo *dependency order*: nền OOP (từ vựng) → SOLID (nguyên tắc) → GoF (mẫu cụ thể) → smell/refactoring (đặt tên vấn đề) → kiến trúc (nâng quy mô) → DDD (mô hình hoá nghiệp vụ) → judgment Tech Lead (tầng meta). Vì thế **Bài 1–3 là OOP** dù bộ đề đặt SOLID trước.

## Mục lục bài học → ID câu hỏi nó phủ

| Bài | Tên bài | Cụm | Câu hỏi phủ |
|---|---|---|---|
| **1** | Hai thước đo gốc: Coupling & Cohesion · Encapsulation & Abstraction | OOP nền | Q-OOP-001, 004, 005 |
| **2** | Composition > Inheritance · Fragile Base Class · Law of Demeter · Program-to-interface | OOP nền | Q-OOP-002, 003, 008, 009 |
| **3** | DRY / KISS / YAGNI · OOP vs Functional trong TS | OOP nền | Q-OOP-006, 007, 010 |
| **4** | SOLID (1): SRP & OCP | Nguyên tắc | Q-SOLID-001, 002, 003, 004, 005 |
| **5** | SOLID (2): LSP & ISP | Nguyên tắc | Q-SOLID-006, 007, 008 |
| **6** | SOLID (3): DIP & DI · SOLID trong thực chiến (trade-off) | Nguyên tắc | Q-SOLID-009, 010, 011, 012, 013, 014 |
| **7** | GoF Creational: Factory · Abstract Factory · Builder · Singleton · Prototype | Mẫu | Q-GOFC-001…008 |
| **8** | GoF Structural: Adapter · Facade · Proxy · Decorator · Composite · Bridge | Mẫu | Q-GOFS-001…008 |
| **9** | GoF Behavioral: Strategy · Observer · Command · Template Method · State · CoR | Mẫu | Q-GOFB-001…010 |
| **10** | Code Smell · Anti-pattern · Refactoring | Chẩn đoán | Q-SMELL-001…010 |
| **11** | Kiến trúc (1): Layered · Clean · Hexagonal · Onion · Dependency Rule · Screaming | Kiến trúc | Q-ARCH-001…006, 009 |
| **12** | Kiến trúc (2): Anemic model · Application/Domain service · CQRS · refactor big-ball-of-mud | Kiến trúc | Q-ARCH-007, 008, 010, 011, 012 |
| **13** | DDD (1): Strategic — Bounded Context · Ubiquitous Language · Entity vs Value Object | DDD | Q-DDD-001, 002, 003 |
| **14** | DDD (2): Aggregate · Invariant · Transaction boundary · Repository · Domain/App Service · Domain Event · khi nào KHÔNG dùng DDD | DDD | Q-DDD-004…010 |
| **15** | Judgment Tech Lead: chống over-engineering · enforce design · ADR · evolutionary architecture · kiểm soát AI-code | Meta | Q-TL-001…008 |

**Mạch phụ thuộc (vì sao thứ tự này):**
```
Bài 1–3 (OOP: coupling/cohesion là "la bàn")
        └─► Bài 4–6 (SOLID = cách hệ thống hoá coupling↓ + cohesion↑)
                └─► Bài 7–9 (GoF = "công thức" hiện thực SOLID cho từng tình huống)
                        └─► Bài 10 (smell = phát hiện khi VI PHẠM mấy nguyên tắc trên)
                                └─► Bài 11–12 (kiến trúc = nâng coupling/cohesion lên cấp module/hệ thống)
                                        └─► Bài 13–14 (DDD = mô hình hoá nghiệp vụ trong khung kiến trúc đó)
                                                └─► Bài 15 (judgment: khi nào DÙNG/ KHÔNG dùng tất cả những thứ trên)
```

> **Câu thần chú xuyên suốt:** *"Tôi không nhớ để gõ, tôi hiểu để chỉ huy — và tra cứu phần còn lại."* Cú pháp pattern thì AI/docs lo; việc **chọn pattern nào, khi nào KHÔNG dùng, và kiểm tra coupling/boundary** là của con người.

---

# 🟢 BÀI 1 — Hai thước đo gốc: Coupling & Cohesion · Encapsulation & Abstraction

> Phủ: **Q-OOP-001, Q-OOP-004, Q-OOP-005**

### ① Mục tiêu & vị trí trong mạch
Đây là bài **nền tảng nhất** của cả mục Q. Mọi nguyên tắc (SOLID), mọi pattern (GoF), mọi quyết định kiến trúc sau này đều chỉ là **phương tiện** để đạt một mục tiêu duy nhất: **loose coupling + high cohesion**. Học xong bài này bạn có "la bàn" để đánh giá *bất kỳ* thiết kế nào — kể cả code AI sinh ra. Bài này đứng trước SOLID vì SOLID chính là cách hệ thống hoá hai thước đo này.

### ② Giảng cơ bản → nâng cao

**(a) Trực giác.**
- **Coupling** (độ ghép) = mức module này *phụ thuộc/biết về* module kia. Ẩn dụ: dây điện nối giữa các hộp. Càng nhiều dây, đụng một hộp là giật cả mớ.
- **Cohesion** (độ cố kết) = mức các phần *bên trong* một module thật sự "thuộc về nhau", cùng phục vụ một mục đích. Ẩn dụ: một hộp dụng cụ — cohesion cao = hộp toàn cờ-lê đúng bộ; cohesion thấp = hộp lẫn cờ-lê, bánh mì, hoá đơn điện.
- Mục tiêu **"loose coupling, high cohesion"**: đổi một chỗ thì **ít lan** (coupling thấp) và mỗi module **dễ hiểu/tái dùng/test** vì nó làm đúng một nhóm việc gắn kết (cohesion cao).

**(b) Cơ chế chi tiết.**
- *Coupling* có nhiều cấp: phụ thuộc *kiểu cụ thể* (chặt nhất) → phụ thuộc *abstraction/interface* (lỏng hơn) → giao tiếp qua *message/event* (lỏng nhất). Giảm coupling = leo lên thang abstraction.
- *Cohesion* xét theo "lý do tồn tại chung": các method trong class có cùng phục vụ một trách nhiệm không? (Đây chính là cầu nối tới **SRP** ở Bài 4.)
- Hai thước đo **đối ngẫu nhưng không tự động đánh đổi**: thiết kế tốt kéo cohesion lên *và* coupling xuống cùng lúc. Thiết kế tệ thường **coupling cao + cohesion thấp** (God Object — Bài 10).

**(c) Encapsulation vs Abstraction** — cặp khái niệm hay bị gộp:
- **Encapsulation** (đóng gói) = *che giấu & bảo vệ* trạng thái nội bộ, chỉ cho sửa qua hành vi hợp lệ → **giữ invariant**. ⚠️ "Để field `private` rồi sinh `get`/`set` cho mọi field" **không** phải encapsulation — đó là phơi state qua cửa sau, mất khả năng bảo vệ bất biến (dẫn tới *anemic model*, Bài 12).
- **Abstraction** (trừu tượng hoá) = phơi ra **"cái gì"** (interface/khái niệm), giấu **"như thế nào"** (implementation). Một câu phân biệt: *abstraction quyết định cái gì lộ ra; encapsulation bảo vệ cái được giấu đi.* Hai mặt của cùng một đồng xu nhưng không đồng nhất.

➕ **Nâng cao:** Encapsulation thật được đo bằng câu hỏi *"có thể đặt object vào trạng thái không hợp lệ từ bên ngoài không?"*. Nếu có → encapsulation rò. Ví dụ `account.balance = -100` chạy được là encapsulation hỏng; phải buộc qua `account.withdraw(amount)` để kiểm tra số dư.

### ③ ⚠️ Kiến thức cũ / bị hiểu sai

| Cách hiểu cũ/sai phổ biến | Hiểu đúng hiện nay | Vì sao |
|---|---|---|
| "Encapsulation = để biến `private`" | Encapsulation = ẩn state + **bảo vệ invariant qua hành vi** | `private` + getter/setter đủ bộ ⇒ vẫn phơi state, mất bảo vệ |
| "Abstraction = encapsulation" | Abstraction phơi *cái gì*; encapsulation giấu *như thế nào* | Gộp hai khái niệm làm mờ mục tiêu thiết kế |
| "Coupling thấp luôn tốt, càng tách càng hay" | Tách *đúng chỗ*; tách quá ⇒ distributed/over-abstraction | Coupling=0 tuyệt đối là không thể & vô dụng; mục tiêu là *loose*, không phải *no* |

### ④ Áp dụng thực tế + So sánh bigtech
- **Use case:** Module thanh toán `payments/` chỉ nên phụ thuộc *abstraction* `PaymentGateway`, không phụ thuộc `StripeClient` cụ thể → đổi Stripe sang Adyen không đụng business logic (loose coupling). Bên trong module, gom đúng các thứ liên quan thanh toán (high cohesion), không nhét logic gửi mail vào đó.
- **Bigtech (pattern phổ biến, không tuyệt đối):** Google/Amazon tổ chức code/service quanh *bounded context* nghiệp vụ để cohesion cao; giao tiếp giữa service qua API/event để coupling lỏng. "Two-pizza team" của Amazon thực ra là cohesion ở cấp *tổ chức* — đội nhỏ sở hữu trọn một domain.

### ⑤ Code thực hành + cấu hình

```typescript
// file: account.ts  — minh hoạ encapsulation THẬT (bảo vệ invariant), không chỉ private
// TypeScript 5.x (cú pháp ổn định; không cần flag decorator cho ví dụ này)

export class Account {
  // #balance là true private (ECMAScript private field) — ngoài class KHÔNG truy cập được
  #balance: number;

  constructor(initial: number) {
    if (initial < 0) throw new Error('Initial balance must be >= 0'); // chặn trạng thái không hợp lệ ngay từ đầu
    this.#balance = initial;
  }

  // Hành vi hợp lệ duy nhất để thay đổi state — invariant "balance >= 0" được bảo vệ tại đây
  withdraw(amount: number): void {
    if (amount <= 0) throw new Error('Amount must be positive');
    if (amount > this.#balance) throw new Error('Insufficient funds'); // bảo vệ invariant
    this.#balance -= amount;
  }

  deposit(amount: number): void {
    if (amount <= 0) throw new Error('Amount must be positive');
    this.#balance += amount;
  }

  // Abstraction: lộ "cái gì" (số dư) ở dạng read-only, KHÔNG cho set tuỳ tiện từ ngoài
  get balance(): number {
    return this.#balance;
  }
}

// ✅ acc.withdraw(50)  -> hợp lệ, qua kiểm tra
// ❌ acc.balance = -100 -> lỗi compile (chỉ có getter) => encapsulation giữ được invariant
```
*Cấu hình:* không cần gì đặc biệt — `#field` là cú pháp JS chuẩn, chạy trên Node hiện đại. Không hardcode secret. So sánh phản ví dụ "anemic" (chỉ getter/setter mọi field) sẽ cho phép `acc.balance = -100` → hỏng.

### ⑥ Keywords cần nhớ
- 🧠 **Cần HIỂU:** coupling, cohesion, "loose coupling + high cohesion", abstraction vs encapsulation, invariant.
- 📌 **Cần THUỘC:** định nghĩa 1 câu của coupling/cohesion; câu phân biệt abstraction (phơi *cái gì*) vs encapsulation (giấu *như thế nào* + bảo vệ state).
- 🛠️ **Cần LÀM ĐƯỢC:** viết class bảo vệ invariant (không cho rơi vào trạng thái sai từ ngoài); nhìn một class nói được nó cohesion cao/thấp và coupling với ai.

### ⑦ Mental model
> **Coupling = số dây nối ra ngoài; Cohesion = độ "đúng bộ" bên trong. Mọi pattern/nguyên tắc sau này chỉ là cách kéo dây ra (coupling↓) và gom đúng bộ (cohesion↑).**

### ⑧ Câu hỏi phỏng vấn & thách đố
1. *(dễ)* Coupling và cohesion là gì? → *Gợi ý:* phụ thuộc-giữa-module vs thuộc-về-nhau-trong-module.
2. *(TB)* Vì sao mục tiêu là "loose coupling, high cohesion"? → đổi một chỗ ít lan; dễ hiểu/test/tái dùng.
3. *(TB)* `private` + getter/setter cho mọi field có phải encapsulation? → Không; phơi state, mất bảo vệ invariant.
4. *(khó)* Nói một câu phân biệt abstraction và encapsulation. → abstraction phơi *cái gì*, encapsulation giấu/bảo vệ *như thế nào*.
5. 🔥 **Thách đố:** "Tách module để giảm coupling — vậy tách càng nhiều càng tốt?" → *Bẫy:* over-decomposition tạo coupling phân tán + indirection vô ích; mục tiêu là *loose*, không phải *zero*. Tách theo *trục thay đổi thật*, không theo cảm tính.

### ⑨ Bài tập + tiêu chí tự chấm
- **BT1:** Cho class `ShoppingCart` có `items`, `total`, và để `cart.total = 999` set tuỳ ý từ ngoài. Refactor để `total` là *derived* và không set được trực tiếp. **Đạt khi:** không thể đặt cart vào trạng thái `total ≠ tổng item` từ bên ngoài.
- **BT2:** Lấy 1 class trong code thật của bạn, viết 2 câu: (i) nó cohesion cao/thấp vì sao; (ii) nó coupling chặt với ai và có thể đổi sang phụ thuộc abstraction không. **Đạt khi:** chỉ ra được ít nhất 1 phụ thuộc *kiểu cụ thể* nên đổi thành abstraction.

### ⑩ Đọc thêm
- *Clean Code* / *Clean Architecture* — Robert C. Martin (chương về cohesion/coupling).
- Bài viết kinh điển "Cohesion & Coupling" (Constantine/Yourdon — structured design).
- 🤖 **Hiểu để chỉ huy:** AI sinh class "chạy được" nhưng hay phơi state (getter/setter mọi field, public mutable field). Hãy kiểm tra: *"Object này có thể bị đặt vào trạng thái sai từ bên ngoài không?"* — nếu có, encapsulation hỏng dù cú pháp đúng.

---

# 🟢 BÀI 2 — Composition > Inheritance · Fragile Base Class · Law of Demeter · Program-to-interface

> Phủ: **Q-OOP-002, Q-OOP-003, Q-OOP-008, Q-OOP-009**

### ① Mục tiêu & vị trí trong mạch
Bài 1 cho la bàn (coupling/cohesion). Bài này dạy **bốn nguyên tắc giảm coupling** mà bạn dùng *hằng ngày* và là tiền đề để hiểu cả GoF lẫn SOLID: ưu tiên composition, hiểu vì sao inheritance dễ tạo coupling chặt, Law of Demeter, và "program to an interface".

### ② Giảng cơ bản → nâng cao

**(a) Favor composition over inheritance.**
- *Inheritance* (`class B extends A`) tạo quan hệ **"is-a"** + coupling **chặt**: B phụ thuộc cả *implementation* của A. Đổi A có thể vỡ B ngầm (xem fragile base class).
- *Composition* ("has-a": B *chứa* và *uỷ quyền* cho thành phần) linh hoạt hơn: ghép/đổi hành vi **lúc runtime**, mỗi mảnh test riêng.
- Inheritance **vẫn đúng** khi quan hệ "is-a" *thật*, thoả **LSP** (Bài 5), và phân cấp **ổn định**. Đừng bài trừ — chỉ là *mặc định nên composition*.

**(b) Fragile Base Class problem (➕ nâng cao).**
Class con phụ thuộc *chi tiết implementation* của cha. Ví dụ kinh điển: `CountingList extends ArrayList`, override `add()` để đếm; nhưng `addAll()` của cha *gọi nội bộ* `add()` → tổng đếm sai gấp đôi. Bạn **không đụng** class con mà nó vẫn vỡ khi cha đổi cách `addAll` gọi `add`. Bài học: inheritance để lộ *white-box* nội bộ của cha → ưu tiên composition (black-box).

**(c) Law of Demeter ("principle of least knowledge").**
"Chỉ nói chuyện với bạn bè trực tiếp." `order.getCustomer().getAddress().getCity()` là **train wreck** `a.b().c().d()` — nó *lộ cấu trúc nội bộ* nhiều cấp, code gọi bị coupling với cả chuỗi. Đổi cấu trúc `Address` là vỡ mọi nơi reach sâu. Hướng sửa: **"Tell, Don't Ask"** — thêm method ở đúng tầng: `order.shippingCity()` (Order tự hỏi Customer/Address bên trong).

**(d) "Program to an interface, not an implementation".**
⚠️ Bẫy: nguyên tắc này **không** đồng nghĩa với từ khoá `interface`. Nó nói: *phụ thuộc vào contract/kiểu trừu tượng*, không phụ thuộc concrete type. Có thể hiện thực bằng abstract class, hoặc thậm chí *duck typing* trong JS/TS. Mục tiêu: tách client khỏi class cụ thể để thay được implementation.

### ③ ⚠️ Cũ → Mới

| Cách làm cũ/phản xạ | Nên dùng nay | Vì sao |
|---|---|---|
| Kế thừa sâu nhiều tầng để tái dùng code | **Composition + uỷ quyền** (mặc định) | Tránh fragile base class, coupling chặt, "is-a" giả |
| `a.b().c().d()` reach sâu vào chuỗi object | Thêm method ở đúng tầng ("Tell, Don't Ask") | Giảm coupling với cấu trúc nội bộ |
| Phụ thuộc concrete class trong code | Phụ thuộc abstraction/contract | Thay implementation/test không sửa client |

### ④ Áp dụng thực tế + So sánh bigtech
- **Use case:** thay vì `class JsonLogger extends Logger extends BaseWriter`, dùng composition: `new Service(logger)` với `logger` thoả interface `Logger`. Đổi từ console-logger sang file-logger chỉ là đổi thành phần inject.
- **Bigtech/ecosystem:** Go ngôn ngữ *cố tình bỏ inheritance*, chỉ có composition + interface ngầm — minh chứng "composition over inheritance" ở cấp thiết kế ngôn ngữ. React cũng khuyến nghị composition thay vì kế thừa component.

### ⑤ Code thực hành

```typescript
// composition-vs-inheritance.ts — TypeScript 5.x

// ❌ Inheritance: coupling chặt, fragile, chỉ "is-a"
// class JsonReportService extends HttpClient {}  // Service "là" HttpClient? Sai bản chất.

// ✅ Composition + program-to-interface
interface HttpClient {                      // abstraction (contract)
  get(url: string): Promise<unknown>;
}

class ReportService {
  // phụ thuộc abstraction, KHÔNG phụ thuộc class cụ thể -> loose coupling, dễ test (inject fake)
  constructor(private readonly http: HttpClient) {}

  async fetchReport(id: string) {
    return this.http.get(`/reports/${id}`);
  }
}

// --- Law of Demeter: "Tell, Don't Ask" ---
class Address { constructor(public readonly city: string) {} }
class Customer { constructor(private readonly address: Address) {}
  get city() { return this.address.city; }   // bọc lại, không bắt ngoài reach sâu
}
class Order {
  constructor(private readonly customer: Customer) {}
  // ✅ thay vì order.getCustomer().getAddress().getCity()
  shippingCity(): string { return this.customer.city; }
}
```
*Test seam nhờ program-to-interface:* trong unit test, truyền `{ get: async () => ({...}) }` làm `HttpClient` giả — không cần network thật.

### ⑥ Keywords
- 🧠 **HIỂU:** composition vs inheritance, fragile base class, Law of Demeter, "Tell Don't Ask", program-to-interface (≠ keyword `interface`).
- 📌 **THUỘC:** dấu hiệu train wreck `a.b().c().d()`; 3 điều kiện inheritance còn hợp lệ (is-a thật + LSP + ổn định).
- 🛠️ **LÀM ĐƯỢC:** refactor một kế thừa thành composition; viết method bọc để khử train wreck; inject abstraction để có test seam.

### ⑦ Mental model
> **Mặc định "has-a" (composition) thay vì "is-a" (inheritance); nói chuyện với hàng xóm trực tiếp (Demeter); phụ thuộc contract chứ không phụ thuộc class.**

### ⑧ Phỏng vấn & thách đố
1. *(TB)* "Favor composition over inheritance" nghĩa là gì, khi nào inheritance vẫn đúng? → is-a thật + LSP + ổn định.
2. *(TB)* Law of Demeter cấm gì? `order.getCustomer().getAddress().getCity()` sai sao? → reach sâu, coupling với cấu trúc nội bộ.
3. *(khó)* Fragile base class: cho một tình huống sửa cha làm vỡ con dù không đụng con. → override `add`, cha đổi `addAll` gọi `add` nội bộ.
4. *(khó)* "Program to interface" có nghĩa là phải dùng keyword `interface` không? → Không; nói về phụ thuộc abstraction/contract.
5. 🔥 **Thách đố:** "Composition luôn tốt hơn inheritance, đúng không?" → *Bẫy:* không tuyệt đối; với phân cấp ổn định + is-a thật + thoả LSP, inheritance gọn hơn. Nguyên tắc là *favor*, không phải *forbid*.

### ⑨ Bài tập + tiêu chí
- **BT1:** Có `class AdminUser extends User` chỉ để thêm quyền. Refactor sang composition (vd `User` + `Role`). **Đạt khi:** thêm loại quyền mới không cần thêm subclass.
- **BT2:** Tìm 1 train wreck trong code bạn, khử bằng method bọc. **Đạt khi:** client không còn gọi quá 1 cấp `.`.

### ⑩ Đọc thêm
- *Design Patterns* (GoF) — nguyên tắc mở đầu "favor object composition over class inheritance".
- *The Pragmatic Programmer* — Law of Demeter, "Tell, Don't Ask".
- 🤖 **Kiểm tra AI:** AI rất hay sinh kế thừa sâu để "tái dùng" và train wreck `a.b().c().d()`. Soát: kế thừa có phải is-a thật không? Có chuỗi `.` quá 1 cấp không?

---

# 🟢 BÀI 3 — DRY / KISS / YAGNI · OOP vs Functional trong TS

> Phủ: **Q-OOP-006, Q-OOP-007, Q-OOP-010**

### ① Mục tiêu & vị trí trong mạch
Ba chữ DRY/KISS/YAGNI là **bộ phanh chống over-engineering** — cực kỳ quan trọng cho phần judgment Tech Lead (Bài 15). Đồng thời bài này phá định kiến "thiết kế tốt = OOP thuần": trong TS/Node, first-class function thường thay được nhiều pattern GoF.

### ② Giảng cơ bản → nâng cao

**(a) Ba nguyên tắc.**
- **DRY** (Don't Repeat Yourself): mỗi *mẩu kiến thức/quy tắc* có **một nguồn chân lý duy nhất**.
- **KISS** (Keep It Simple): chuộng giải pháp đơn giản nhất *đủ dùng*.
- **YAGNI** (You Aren't Gonna Need It): **đừng** xây trước thứ chưa cần ("để sau này biết đâu cần").

**(b) ⚠️ DRY bị hiểu sai nặng nhất.** DRY **KHÔNG** phải "không có dòng code trùng nhau". Nó về **kiến thức**, không về **ký tự**. Hai đoạn code *trông giống nhau* nhưng **thay đổi vì lý do khác nhau** thì **KHÔNG nên gộp** — gộp lại tạo *coupling giả* (premature/false abstraction): mai một bên cần đổi, bạn phải thêm `if/flag` vào abstraction chung làm nó méo mó. Câu thần chú: *"hai thứ giống nhau hôm nay không đảm bảo cùng đổi ngày mai"*. (Đây là tiền đề của smell "false abstraction" — Bài 10, và đối trọng với SRP — Bài 4.)

**(c) DRY quá đà — ví dụ.** Có 2 hàm validate: `validateUserAge` và `validateProductYear`, tình cờ cùng là "số trong [0, 150]". Gộp thành `validateRange(0,150)` rồi sau product cho phép tới 9999 → bạn nhét tham số/biến thể vào hàm chung. Đáng lẽ để riêng vì **lý do thay đổi khác nhau** (luật tuổi vs luật năm). KISS+YAGNI bảo: cứ để 2 hàm tới khi xuất hiện nhu cầu thật.

**(d) OOP vs Functional trong TS/Node (➕).** Nhiều pattern GoF "co lại" nhờ first-class function/closure/module:
- **Strategy** = truyền một *hàm* vào (không cần cây class).
- **Command** = một *closure* bắt context.
- **Singleton** = một *module* (Node cache module → export instance là singleton sẵn — xem Bài 7).
- **Decorator (hành vi)** = hàm bọc hàm (higher-order function).
Judgment: chọn paradigm theo bài toán; không ép OOP-thuần để "cho giống sách". *(verify minh hoạ cú pháp khi học — TS đổi nhanh ở mảng decorator/type.)*

### ③ ⚠️ Cũ → Mới

| Hiểu sai | Hiểu đúng | Vì sao |
|---|---|---|
| "DRY = xoá mọi dòng code trùng" | DRY = một nguồn chân lý cho mỗi *quy tắc* | Gộp code giống-ngẫu-nhiên ⇒ coupling giả |
| "Thiết kế tốt ⇒ phải full OOP/class" | Chọn paradigm theo bài toán; FP thay nhiều GoF trong TS | First-class function làm pattern gọn hơn |
| "Cứ trừu tượng hoá sẵn cho tương lai" | YAGNI: chỉ trừu tượng khi *đau thật* | Speculative generality ⇒ phức tạp thừa |

### ④ Áp dụng thực tế + So sánh bigtech
- **Use case:** thay vì `class SortStrategy` + 3 subclass, dùng `sort(items, compareFn)` truyền hàm so sánh. Gọn, dễ test, đúng tinh thần TS.
- **Bigtech/ecosystem:** API chuẩn JS (`Array.sort(compareFn)`, `Array.map(fn)`) chính là Strategy bằng hàm — minh chứng FP-style là *idiom* của ngôn ngữ, không phải "thiếu thiết kế".

### ⑤ Code thực hành

```typescript
// strategy-as-function.ts — GoF Strategy thu gọn bằng first-class function
type ShippingFee = (weightKg: number) => number;

const fees: Record<string, ShippingFee> = {     // "factory" = object map, không cần class hierarchy
  standard: (w) => w * 1.0,
  express:  (w) => w * 2.5 + 3,
};

function quote(method: string, weightKg: number): number {
  const fn = fees[method];
  if (!fn) throw new Error(`Unknown method: ${method}`);
  return fn(weightKg);                            // chọn "strategy" runtime, thêm method = thêm 1 dòng (gần OCP)
}

// ⚠️ Phản ví dụ DRY quá đà — KHÔNG nên gộp 2 hàm chỉ vì trông giống:
// const validateRange = (min:number,max:number) => (n:number)=> n>=min && n<=max;
//   -> nếu luật tuổi và luật năm rồi sẽ rẽ nhánh khác nhau, hãy để RIÊNG (2 lý do thay đổi).
```

### ⑥ Keywords
- 🧠 **HIỂU:** DRY (về kiến thức), KISS, YAGNI, premature/false abstraction, FP thay GoF.
- 📌 **THUỘC:** phát biểu đúng của DRY; "2 thứ giống nhau + lý do đổi khác nhau ⇒ không gộp".
- 🛠️ **LÀM ĐƯỢC:** nhận ra một abstraction gộp sai và tách lại; viết Strategy/Command bằng function/closure.

### ⑦ Mental model
> **DRY là về kiến thức, không về ký tự. Đơn giản trước (KISS), chỉ làm khi cần (YAGNI), và trong TS — một hàm thường thay được cả cây class.**

### ⑧ Phỏng vấn & thách đố
1. *(TB)* DRY/KISS/YAGNI mỗi cái cảnh báo gì? → lặp kiến thức / phức tạp / làm trước cái chưa cần.
2. *(khó)* Phát biểu đúng của DRY? → một nguồn chân lý cho mỗi quy tắc, không phải "không trùng ký tự".
3. *(khó)* Trong TS, "truyền một hàm vào" đã là Strategy chưa? → Rồi — bản gọn của Strategy.
4. 🔥 **Thách đố:** "Thấy 2 đoạn code y hệt — gộp ngay chứ?" → *Bẫy:* phải hỏi *chúng có cùng lý do thay đổi không?* Nếu không, gộp = coupling giả, sau này khổ. Trùng *ngẫu nhiên* ≠ trùng *kiến thức*.

### ⑨ Bài tập + tiêu chí
- **BT1:** Tìm trong code bạn 1 abstraction "gộp cho DRY" mà giờ đầy `if/flag` cho 2 ca khác nhau. Tách lại. **Đạt khi:** mỗi nhánh là một hàm/class độc lập, không còn cờ rẽ nhánh chung.
- **BT2:** Viết lại một `switch` xử lý theo loại bằng object map of functions. **Đạt khi:** thêm loại mới = thêm 1 entry, không sửa hàm điều phối.

### ⑩ Đọc thêm
- *The Pragmatic Programmer* — DRY (định nghĩa gốc về "knowledge").
- Sandi Metz — "The Wrong Abstraction" (vì sao duplication đôi khi rẻ hơn abstraction sai).
- 🤖 **Kiểm tra AI:** AI hay "DRY hoá" hai đoạn giống nhau thành một hàm tham-số-hoá, tạo coupling giả; và hay over-engineer (class cho thứ chỉ cần 1 hàm). Soát theo YAGNI/KISS: *cái này có nhu cầu thật chưa?*

---

# 🟡 BÀI 4 — SOLID (1): Single Responsibility & Open/Closed

> Phủ: **Q-SOLID-001, 002, 003, 004, 005**

### ① Mục tiêu & vị trí trong mạch
SOLID (Robert C. Martin) là **cách hệ thống hoá** la bàn Bài 1: SRP/ISP kéo *cohesion lên*, OCP/DIP kéo *coupling xuống*. Bài này gói gọn SRP + OCP — hai chữ bạn vi phạm nhiều nhất. Bài 5 lo LSP+ISP, Bài 6 lo DIP + trade-off.

### ② Giảng cơ bản → nâng cao

**Năm chữ (gọi đúng tên — Q-SOLID-001):**
- **S**RP — Single Responsibility · **O**CP — Open/Closed · **L**SP — Liskov Substitution · **I**SP — Interface Segregation · **D**IP — Dependency Inversion.

**(a) SRP — Single Responsibility Principle.**
- ⚠️ **Không phải** "một class làm một việc". Phát biểu chuẩn: *"một class chỉ nên có **MỘT lý do để thay đổi**"* — tức chỉ phục vụ **một actor/stakeholder**. "Lý do thay đổi" ≠ "số lượng method".
- Ví dụ vi phạm (Q-SOLID-003): `UserService` vừa **validate** (luật nghiệp vụ), vừa **lưu DB** (persistence), vừa **gửi email** (hạ tầng). Ba *lý do thay đổi* khác nhau (đổi rule, đổi DB, đổi mail provider) → tách thành `UserValidator`, `UserRepository`, `EmailSender`; `UserService` *điều phối*. Lợi: test/maintain từng phần độc lập, đổi mail không đụng logic validate.
- SRP chính là **cohesion ở cấp class** + chống God Object.

**(b) OCP — Open/Closed Principle.**
- *"Mở để mở rộng, đóng để sửa đổi"*: thêm hành vi mới **không sửa code cũ đã chạy ổn**. Đạt qua **abstraction/polymorphism** (interface, Strategy, map handler).
- Vì sao quan trọng: sửa code cũ = rủi ro **regression** lên thứ đang chạy. Thêm code mới (class mới) an toàn hơn.
- ⚠️ **Mùi vi phạm OCP** (Q-SOLID-005): chuỗi `if (type==='A') … else if (type==='B') …` mà *mỗi loại mới phải sửa hàm*. Đây là "switch-on-type" — báo **thiếu abstraction**. Trị bằng **polymorphism/Strategy** hoặc **map handler** `{ A: handleA, B: handleB }`.

➕ **Nâng cao:** OCP không có nghĩa "không bao giờ sửa code". Nó nói: thiết kế *điểm mở rộng* (extension point) tại trục *thực sự biến động*. Đừng mở mọi thứ (over-engineering — Bài 15). Mở đúng chỗ hay đổi.

### ③ ⚠️ Cũ → Mới

| Cách cũ / hiểu sai | Nên dùng nay | Vì sao |
|---|---|---|
| "SRP = một class một việc/một method" | SRP = **một lý do để thay đổi** (một actor) | Tránh tách máy móc hoặc gộp nhầm |
| `if/else if` theo `type`, thêm loại là sửa hàm | **Strategy / polymorphism / map handler** | Đạt OCP, giảm rủi ro regression |
| Sửa thẳng class cũ mỗi lần thêm tính năng | Thêm class mới qua extension point | "Đóng để sửa đổi, mở để mở rộng" |

### ④ Áp dụng thực tế + So sánh bigtech
- **Use case:** cổng thanh toán hỗ trợ nhiều provider. Thay vì `if(provider==='stripe')…`, định nghĩa interface `PaymentProvider` + mỗi provider một class; thêm provider = thêm class, không đụng orchestrator (OCP + SRP).
- **Bigtech/ecosystem:** plugin architecture (VS Code extensions, webpack loaders, Express middleware) là OCP ở quy mô lớn — core "đóng", mở rộng qua plugin/interface.

### ⑤ Code thực hành

```typescript
// ocp-strategy.ts — khử switch-on-type, đạt OCP + SRP

interface PriceRule {                         // abstraction = điểm mở rộng (OCP)
  appliesTo(type: string): boolean;
  price(base: number): number;
}

class RegularPrice implements PriceRule {
  appliesTo(t: string) { return t === 'regular'; }
  price(base: number) { return base; }
}
class VipPrice implements PriceRule {
  appliesTo(t: string) { return t === 'vip'; }
  price(base: number) { return base * 0.8; }    // thêm loại mới = thêm class, KHÔNG sửa calculatePrice
}

class PriceCalculator {                        // SRP: chỉ điều phối, không chứa từng luật
  constructor(private readonly rules: PriceRule[]) {}
  calculate(type: string, base: number): number {
    const rule = this.rules.find(r => r.appliesTo(type));
    if (!rule) throw new Error(`No rule for ${type}`);
    return rule.price(base);
  }
}
// ✅ thêm 'student' = thêm StudentPrice + đăng ký, code cũ không đổi
```

### ⑥ Keywords
- 🧠 **HIỂU:** SRP (lý do thay đổi), OCP (extension point), switch-on-type là smell, OCP↔giảm regression.
- 📌 **THUỘC:** 5 chữ SOLID đúng tên/nghĩa; phát biểu chuẩn SRP và OCP.
- 🛠️ **LÀM ĐƯỢC:** tách một God class theo lý-do-thay-đổi; khử `if/else if type` bằng Strategy/map.

### ⑦ Mental model
> **SRP: mỗi class chỉ có một "ông chủ" để phục vụ (một lý do đổi). OCP: thêm hành vi bằng thêm class, không cưa lại code đang chạy.**

### ⑧ Phỏng vấn & thách đố
1. *(dễ)* SOLID là 5 chữ gì? → SRP/OCP/LSP/ISP/DIP.
2. *(TB)* SRP thật ra nói về gì? "một class một việc" đúng không? → một *lý do thay đổi*; "một việc" là hiểu thô.
3. *(TB)* `UserService` validate+lưu DB+gửi mail vi phạm gì, sửa sao? → SRP; tách collaborator.
4. *(khó)* OCP đạt nhờ cơ chế OOP nào? → abstraction/polymorphism.
5. 🔥 **Thách đố:** "Để đạt OCP, tạo interface cho *mọi* thứ luôn cho chắc?" → *Bẫy:* OCP chỉ mở tại trục *thực sự biến động*; mở mọi thứ = over-engineering (interface 1-1 vô nghĩa — Bài 6/15).

### ⑨ Bài tập + tiêu chí
- **BT1:** Refactor `NotificationService.send(type)` có `if(type==='sms')…else if(type==='email')…` sang Strategy. **Đạt khi:** thêm kênh 'push' không sửa `send`.
- **BT2:** Tách một class trong code bạn có ≥2 lý do thay đổi. **Đạt khi:** mỗi class mới chỉ vỡ vì *một* loại lý do.

### ⑩ Đọc thêm
- R. C. Martin — *Clean Architecture*, *Agile Software Development* (chương SOLID).
- Bài gốc "The Single Responsibility Principle" (blog.cleancoder.com) — định nghĩa "actor".
- 🤖 **Kiểm tra AI:** AI rất hay sinh God service + switch-on-type "vì nó chạy". Soát: class này có mấy lý do để đổi? Có `if type` nào nên thành polymorphism không?

---

# 🟡 BÀI 5 — SOLID (2): Liskov Substitution & Interface Segregation

> Phủ: **Q-SOLID-006, 007, 008**

### ① Mục tiêu & vị trí trong mạch
LSP là "bài kiểm tra inheritance có hợp lệ không" — nối thẳng Bài 2 (composition > inheritance). ISP là "SRP cho interface" — kéo cohesion ở mức contract. Hai chữ này hay bị bỏ qua nhưng lộ rõ ai *hiểu* OOP.

### ② Giảng cơ bản → nâng cao

**(a) LSP — Liskov Substitution Principle.**
- Phát biểu: *subtype phải **thay thế được** supertype mà **không phá hành vi/contract** mà client trông đợi.*
- Ràng buộc hình thức (đáng nhớ): **tiền điều kiện (precondition) không được mạnh hơn**, **hậu điều kiện (postcondition) không được yếu hơn** ở subtype; invariant của base phải giữ.
- ⚠️ Ví dụ kinh điển vi phạm:
  - **Square extends Rectangle:** set `width` của Square buộc đổi `height` → code dựa trên contract "đặt width không ảnh hưởng height" vỡ. "Hình vuông *là* hình chữ nhật" về toán nhưng *behaves-not-like* trong code.
  - **Ostrich extends Bird** với `fly()`: đà điểu `fly()` ném exception → client gọi `bird.fly()` vỡ. Kế thừa "đúng cú pháp" nhưng phá LSP.
- **Bài học (Q-SOLID-007):** quan hệ phân loại đời thực ("is-a") **≠** quan hệ thay thế *hành vi*. LSP là *test* cho việc inheritance có hợp lệ không; "is-a" không kéo theo "behaves-like-a" thì **ưu tiên composition**.

**(b) ISP — Interface Segregation Principle.**
- *Client không nên bị buộc phụ thuộc vào method nó không dùng.* "Fat interface" là mùi.
- Ví dụ (Q-SOLID-008): `interface Worker { work(); eat(); }`; `RobotWorker` phải implement `eat()` rỗng → sai. Tách thành `Workable` và `Eatable`; client nào cần gì implement nấy.
- ISP là **SRP áp cho interface**: gom đúng nhóm hành vi gắn kết, không nhồi.

### ③ ⚠️ Cũ → Mới

| Cách cũ / sai | Nên dùng nay | Vì sao |
|---|---|---|
| Kế thừa theo "is-a" đời thường (Square is-a Rectangle) | Kiểm bằng **LSP** (behaves-like-a?), không thì composition | "is-a" ≠ thay thế hành vi |
| Một "fat interface" cho mọi client | Tách interface nhỏ theo nhu cầu (ISP) | Tránh implement method rỗng/ném exception |
| Override rồi `throw NotSupported` | Thiết kế lại phân cấp / tách interface | `throw` trong override = mùi LSP |

### ④ Áp dụng thực tế + So sánh bigtech
- **Use case ISP:** thay vì `interface Repository` khổng lồ (CRUD + bulk + stream + cache), tách `ReadRepository` / `WriteRepository`; service đọc chỉ phụ thuộc phần đọc (cũng là tiền đề CQRS — Bài 12).
- **Bigtech/ecosystem:** Go khuyến khích **interface nhỏ** ("the bigger the interface, the weaker the abstraction" — Rob Pike). `io.Reader`/`io.Writer` mỗi cái 1 method là ISP cực đoan mà hiệu quả.

### ⑤ Code thực hành

```typescript
// lsp-isp.ts — TypeScript 5.x

// --- ISP: tách fat interface ---
interface Workable { work(): void; }
interface Eatable  { eat(): void; }

class HumanWorker implements Workable, Eatable {
  work() {/* ... */}
  eat()  {/* ... */}
}
class RobotWorker implements Workable {   // ✅ không bị ép eat() rỗng
  work() {/* ... */}
}

// --- LSP: tránh "is-a" giả ---
// ❌ class Square extends Rectangle { set width(w){ this._w=this._h=w; } }  // phá contract
// ✅ Tách abstraction theo HÀNH VI, không theo phân loại toán học:
interface Shape { area(): number; }
class Rectangle implements Shape { constructor(private w:number, private h:number){} area(){return this.w*this.h;} }
class Square    implements Shape { constructor(private s:number){} area(){return this.s*this.s;} }
// cả hai "là Shape" theo nghĩa behaves-like-a (tính được area), không kế thừa nhau.
```

### ⑥ Keywords
- 🧠 **HIỂU:** LSP (thay thế không phá contract; pre không mạnh hơn / post không yếu hơn), ISP (client không phụ thuộc thứ không dùng), "is-a" ≠ "behaves-like-a".
- 📌 **THUỘC:** ví dụ Square/Rectangle, Ostrich/Bird; phát biểu ISP.
- 🛠️ **LÀM ĐƯỢC:** phát hiện override `throw NotSupported`; tách fat interface.

### ⑦ Mental model
> **LSP: con phải "đóng thế" cha mà khán giả không nhận ra. ISP: đừng bắt ai ký hợp đồng cho việc họ không làm.**

### ⑧ Phỏng vấn & thách đố
1. *(khó)* LSP nói gì? Cho ví dụ kế thừa "đúng cú pháp" nhưng phá LSP. → Square/Rectangle, Ostrich/Bird.
2. *(khó)* Vì sao "chim cánh cụt *là* chim" lại có thể vi phạm LSP? → phân loại ≠ thay thế hành vi.
3. *(TB)* `Worker{work,eat}` bắt robot `eat()` rỗng — sai gì, sửa sao? → ISP; tách `Workable`/`Eatable`.
4. 🔥 **Thách đố:** "Tôi override method và `throw new Error('not supported')` — vẫn compile mà?" → *Bẫy:* compile được nhưng **phá LSP** (subtype không thay thế được supertype). Dấu hiệu phân cấp sai — tách interface/đổi composition.

### ⑨ Bài tập + tiêu chí
- **BT1:** Tìm 1 chỗ override ném "not supported" trong code/lib bạn dùng, đề xuất tách interface. **Đạt khi:** không subtype nào phải ném-vì-không-hỗ-trợ.
- **BT2:** Tách 1 fat interface (≥5 method phục vụ ≥2 nhóm client) thành các interface nhỏ. **Đạt khi:** mỗi client chỉ phụ thuộc method nó gọi.

### ⑩ Đọc thêm
- Barbara Liskov — "Behavioral subtyping" (nguồn gốc LSP).
- R. C. Martin — SOLID papers; Rob Pike — Go proverbs (interface nhỏ).
- 🤖 **Kiểm tra AI:** AI hay tạo phân cấp kế thừa theo "is-a" đời thường (vi phạm LSP) và interface to nhồi nhét. Soát: subtype có thay thế được supertype ở *mọi* chỗ dùng không?

---

# 🟡 BÀI 6 — SOLID (3): Dependency Inversion & DI · SOLID trong thực chiến

> Phủ: **Q-SOLID-009, 010, 011, 012, 013, 014**

### ① Mục tiêu & vị trí trong mạch
DIP là chữ "đắt giá" nhất — nền của Clean/Hexagonal (Bài 11) và DI container của Nest. Bài này cũng đóng phần SOLID bằng **trade-off**: SOLID có thể là over-engineering, và cách phản hồi review "áp SOLID máy móc". Nối thẳng tới judgment Bài 15.

### ② Giảng cơ bản → nâng cao

**(a) DIP — Dependency Inversion Principle (hai vế — Q-SOLID-009):**
1. Module **cấp cao** (business/policy) **không phụ thuộc** module **cấp thấp** (DB/HTTP/driver); **cả hai phụ thuộc abstraction**.
2. **Abstraction không phụ thuộc chi tiết; chi tiết phụ thuộc abstraction.**
- "High-level" = logic nghiệp vụ (cái *vì sao* app tồn tại); "low-level" = chi tiết kỹ thuật (Postgres, SMTP). DIP **đảo chiều** mũi tên phụ thuộc: thay vì business → DB, ta để business → `interface` ← DB-impl.

**(b) ⚠️ DIP ≠ DI (Q-SOLID-010) — bẫy phỏng vấn kinh điển.**
- **DIP** là *nguyên tắc* (phụ thuộc abstraction).
- **DI** (Dependency Injection) là *kỹ thuật* cung cấp dependency từ ngoài vào (constructor/setter/container).
- **Có DI chưa chắc có DIP:** nếu bạn `inject` một **concrete class** (vd inject `StripeClient` thẳng) → vẫn là DI nhưng **không đạt DIP** (vẫn phụ thuộc chi tiết). Đạt DIP cần inject **qua interface/abstraction**. *(Cơ chế container Nest — provider/scope/`useFactory` — thuộc mục H-DI; ở đây hiểu *vì sao thiết kế thế*.)*

**(c) Vì sao DIP → dễ test (Q-SOLID-011):** phụ thuộc abstraction ⇒ trong test thay implementation thật bằng **test double** (mock/stub/fake) → tách business khỏi I/O thật (DB/network) → unit test **nhanh & xác định**.

**(d) SOLID ↔ coupling/cohesion (Q-SOLID-013):** **SRP + ISP** kéo **cohesion ↑** (gom đúng trách nhiệm, interface gọn); **OCP + DIP** kéo **coupling ↓** (phụ thuộc abstraction, không sửa lan). Mục tiêu cuối vẫn là **loose coupling + high cohesion** (Bài 1).

**(e) ⚠️⚠️ SOLID có thể là over-engineering (Q-SOLID-012 & 014) — phần Tech Lead.**
- Dự án nhỏ/script/throwaway, hoặc **abstraction sớm khi chưa rõ trục biến thiên** → thêm class/interface/indirection **vô ích**. Nguyên tắc: *"áp dụng khi đau thật"*, không giáo điều.
- **Review case (Q-SOLID-014):** junior tạo **interface 1-1 cho mọi class** (mỗi class một interface chỉ-một-implementation) "để SOLID". Đây là **over-abstraction (YAGNI)**: interface chỉ có giá trị khi có **≥2 implementation** *hoặc* cần **đảo chiều phụ thuộc/test seam thật**. Cách phản hồi: **dẫn dắt bằng câu hỏi** ("interface này có implementation thứ 2 nào sắp tới? có cần seam để test không?"), không chỉ chê — giúp họ tự thấy.

### ③ ⚠️ Cũ → Mới

| Cách cũ / hiểu sai | Nên dùng nay | Vì sao |
|---|---|---|
| Business `import` thẳng `PgClient`/`StripeClient` | Business phụ thuộc **port/interface**, infra implement | Đảo chiều phụ thuộc (DIP) → test/thay được |
| "Inject là đã DIP" | DI ≠ DIP; phải inject **qua abstraction** | Inject concrete vẫn là phụ thuộc chi tiết |
| Interface 1-1 cho mọi class "cho SOLID" | Interface khi có ≥2 impl hoặc cần seam thật | Tránh over-abstraction (YAGNI) |

### ④ Áp dụng thực tế + So sánh bigtech
- **Use case:** `OrderService` (high-level) phụ thuộc `OrderRepository` (port). `PostgresOrderRepository` (low-level) implement port. Đổi sang Mongo = thêm adapter, **không đụng** `OrderService`. Test = inject `InMemoryOrderRepository`.
- **Bigtech/ecosystem:** Spring (Java), NestJS, Angular đều xây quanh DI container để hiện thực DIP ở quy mô lớn. AWS Lambda + interface cho phép swap implementation theo môi trường (local/cloud).

### ⑤ Code thực hành

```typescript
// dip-vs-di.ts — phân biệt DI và DIP

// 1) Abstraction do high-level định nghĩa (port)
export interface NotificationSender {
  send(to: string, msg: string): Promise<void>;
}

// 2) High-level module phụ thuộc abstraction (DIP) + nhận qua constructor (DI)
export class OrderService {
  constructor(private readonly notifier: NotificationSender) {} // DI + DIP (interface)
  async placeOrder(userEmail: string) {
    // ... business logic ...
    await this.notifier.send(userEmail, 'Order placed');         // không biết Email/SMS/SES cụ thể
  }
}

// 3) Low-level chi tiết implement port (chi tiết phụ thuộc abstraction)
export class SesEmailSender implements NotificationSender {
  async send(to: string, msg: string) { /* gọi AWS SES — verify SDK ở docs chính thức */ }
}

// ⚠️ Phản ví dụ: constructor(private ses: SesEmailSender) -> vẫn là DI nhưng KHÔNG đạt DIP.

// 4) Test seam nhờ DIP:
// const fake: NotificationSender = { send: async () => {} };
// new OrderService(fake).placeOrder('a@b.com');  // unit test, không gọi SES thật
```
*Cấu hình:* wiring cụ thể (chọn `SesEmailSender` ở production, `FakeSender` ở test) do **DI container** lo — xem mục H-DI cho cách Nest `useClass/useFactory`. Không hardcode key; SES credential qua biến môi trường/secret manager.

### ⑥ Keywords
- 🧠 **HIỂU:** DIP (2 vế, đảo chiều phụ thuộc), DIP≠DI, vì sao DIP→testable, SOLID↔coupling/cohesion, SOLID có thể over-engineer.
- 📌 **THUỘC:** phát biểu 2 vế DIP; "inject concrete = DI nhưng không DIP".
- 🛠️ **LÀM ĐƯỢC:** đảo một phụ thuộc concrete thành port+adapter; phản hồi review "interface 1-1" mang tính dẫn dắt.

### ⑦ Mental model
> **DIP: đảo mũi tên — business định nghĩa interface, hạ tầng cắm vào. DI chỉ là cách "đưa đồ vào tận tay". Và: SOLID là thuốc — đúng liều thì khỏi, quá liều thì ngộ độc (over-engineering).**

### ⑧ Phỏng vấn & thách đố
1. *(khó)* DIP phát biểu 2 vế nào? high/low-level là gì? → business vs chi tiết; cả hai phụ thuộc abstraction.
2. *(khó)* DIP khác DI thế nào? Có DI là có DIP không? → nguyên tắc vs kỹ thuật; inject concrete vẫn không DIP.
3. *(TB)* Vì sao lập trình theo interface dễ test hơn? → thay impl thật bằng test double.
4. *(rất khó)* SOLID bao giờ là over-engineering? Khi nào cố tình không áp triệt để? → script/nhỏ; abstraction sớm khi chưa rõ trục biến thiên.
5. 🔥 **Thách đố (review):** junior tạo interface 1-1 cho mọi class "cho SOLID" — bạn nói gì? → *Bẫy:* khen tinh thần nhưng chỉ ra over-abstraction; interface đáng khi có ≥2 impl/seam test; dẫn dắt bằng câu hỏi, không chỉ chê.

### ⑨ Bài tập + tiêu chí
- **BT1:** Lấy 1 service đang `import` thẳng client DB/HTTP. Tách port + adapter, inject qua interface. **Đạt khi:** viết được unit test cho service mà không chạm I/O thật.
- **BT2:** Soát repo bạn tìm 1 interface 1-1 vô nghĩa, quyết định giữ hay bỏ kèm lý do. **Đạt khi:** lý do dựa trên "có impl thứ 2 / seam test không", không phải "cho SOLID".

### ⑩ Đọc thêm
- R. C. Martin — "The Dependency Inversion Principle" + *Clean Architecture* (DIP ở cấp kiến trúc → Bài 11).
- NestJS docs — *Custom providers* (cơ chế DI; cross-ref H-DI).
- 🤖 **Kiểm tra AI:** AI hay inject concrete class (DI mà không DIP) và/hoặc đẻ interface 1-1 thừa. Soát: phụ thuộc có trỏ vào *abstraction* không? Interface này có lý do tồn tại (impl thứ 2 / test seam) không?

---

# 🟠 BÀI 7 — GoF Creational: Factory · Abstract Factory · Builder · Singleton · Prototype

> Phủ: **Q-GOFC-001…008**

### ① Mục tiêu & vị trí trong mạch
GoF (Gang of Four, 1994) chia **23 pattern** làm 3 nhóm. Bài này lo **Creational** — *cách tạo object* để tách "quyết định tạo cái gì" khỏi "nơi dùng". Hiểu xong, bạn thấy nhiều pattern Java co lại trong TS nhờ first-class function/module (nối Bài 3). Đây là nền cho Structural (Bài 8) và Behavioral (Bài 9).

### ② Giảng cơ bản → nâng cao

**(a) Ba nhóm GoF (Q-GOFC-001):**
- **Creational** — *tạo* object (Factory Method, Abstract Factory, Builder, Singleton, Prototype). Đại diện: **Factory**.
- **Structural** — *ghép/cấu trúc* object (Adapter, Decorator, Facade…). Đại diện: **Adapter**.
- **Behavioral** — *giao tiếp & phân chia trách nhiệm* (Strategy, Observer…). Đại diện: **Strategy**.

**(b) Factory Method (Q-GOFC-002).** Tách *quyết định "tạo concrete class nào"* khỏi nơi dùng, ẩn logic khởi tạo/lựa chọn. ⚠️ `new X()` trực tiếp **lộ concrete type** → không decouple → **không phải factory**. Factory trả về *abstraction*, người gọi không biết class cụ thể.

**(c) Factory kiểu TS (Q-GOFC-003).** Thực tế trong Node/TS, "factory" thường chỉ là **object map** `{ pdf: makePdf, csv: makeCsv }` thay vì cây class + switch.
- *Được:* gọn, thêm loại ≈ thêm entry (gần OCP), tận dụng first-class function.
- *Mất:* ràng buộc kiểu lỏng hơn, ít "khung" cho biến thể phức tạp/đa sản phẩm.
- Chọn theo độ phức tạp: map cho đơn giản, hierarchy khi cần cấu trúc.

**(d) Abstract Factory (Q-GOFC-004).** Factory Method tạo **một** sản phẩm; Abstract Factory tạo **cả họ sản phẩm liên quan, nhất quán cùng "gia đình"** (vd bộ UI theo theme: `Button`+`Checkbox` cùng Dark/Light; hoặc bộ driver cùng vendor). Cần khi phải đảm bảo các object **đi cùng nhau đúng bộ**.

**(e) Builder (Q-GOFC-005).** Tạo object phức tạp **từng bước**, tránh *telescoping constructor* (`new X(a,b,c,d,e,f)`), đặt tên rõ từng phần, bất biến sau `build()`. ⚠️ Trong TS, **options object** `new X({a,b,c})` thường đã đủ → Builder chỉ đáng khi có **thứ tự/validation phức tạp giữa các bước** hoặc nhiều biến thể build.

**(f) Singleton — vì sao bị coi anti-pattern (Q-GOFC-006 & 007).**
- Singleton = **global state ẩn** → khó test (state rò giữa test), **coupling ngầm** (ai cũng `Singleton.getInstance()`), vấn đề concurrency/lifecycle/khởi tạo thứ tự.
- ⚠️ **Trong Node, "module là singleton sẵn":** module được **cache** sau lần `require/import` đầu → `export const db = new Pool()` đã là một instance dùng chung toàn app. Không cần tự cài Singleton thủ công.
- **Connection pool "chỉ một" (Q-GOFC-007):** ưu tiên **cung cấp qua DI** (một instance do container quản lý) thay vì Singleton cổ điển — vẫn **test/override/khởi tạo có thứ tự** được. Nhớ: *"một instance" ≠ "phải dùng Singleton pattern"*. *(cross-ref H-DI.)*

**(g) Prototype (Q-GOFC-008).** Tạo object mới bằng **clone** một mẫu thay vì khởi tạo tốn kém/không biết concrete type. JS vốn prototype-based; có `Object.create`, `structuredClone`. ⚠️ Cảnh giác **shallow vs deep clone** (nested object/refs).

### ③ ⚠️ Cũ → Mới

| Cũ | Nay (TS/Node) | Vì sao |
|---|---|---|
| `getInstance()` Singleton thủ công | **Module export** hoặc **DI single-scope** | Module đã cache; DI vẫn test/override được |
| Telescoping constructor `new X(a,b,c,d,e)` | **Options object** (hoặc Builder khi phức tạp) | Rõ tên, tránh nhầm thứ tự tham số |
| `switch(type){ new A()… }` rải rác | **Object map factory** `{a:makeA}` | Gần OCP, gọn, first-class function |
| Deep copy bằng `JSON.parse(JSON.stringify())` | `structuredClone()` (Node ≥17) | Giữ Date/Map/Set, không mất kiểu *(verify Node version)* |

### ④ Áp dụng thực tế + So sánh bigtech
- **Use case Factory:** `createExporter(format)` trả về `Exporter` (PDF/CSV/XLSX) qua map → controller không biết class cụ thể.
- **Use case Singleton-via-DI:** DB pool, Redis client, HTTP client tái dùng — Nest provider scope `DEFAULT` (singleton) quản lý.
- **Bigtech/ecosystem:** driver DB (pg, mongodb) export *factory/client* tái dùng; SDK đám mây (AWS SDK v3) dùng client tạo một lần inject khắp nơi — đúng tinh thần "một instance qua DI", không Singleton thủ công.

### ⑤ Code thực hành

```typescript
// creational.ts — TypeScript 5.x

// --- Factory bằng object map (idiom TS) ---
interface Exporter { export(rows: object[]): string; }
class CsvExporter implements Exporter { export(r){ return '/* csv */'; } }
class JsonExporter implements Exporter { export(r){ return JSON.stringify(r); } }

const exporters: Record<string, () => Exporter> = {
  csv:  () => new CsvExporter(),
  json: () => new JsonExporter(),    // thêm 'xlsx' = thêm 1 dòng, không sửa createExporter
};
function createExporter(format: string): Exporter {
  const make = exporters[format];
  if (!make) throw new Error(`Unsupported format: ${format}`);
  return make();                     // trả ABSTRACTION (Exporter), người gọi không biết concrete
}

// --- "Singleton" đúng cách trong Node: module export (đã được cache) ---
// db.ts
// import { Pool } from 'pg';                 // verify ở docs chính thức
// export const pool = new Pool({ /* config từ ENV, KHÔNG hardcode */ });
// các file khác: import { pool } from './db'  -> cùng một instance toàn app

// --- Builder chỉ khi thật phức tạp; đa số dùng options object ---
class Pizza { private constructor(readonly opts: {size:string; toppings:string[]}) {} 
  static builder(){ const t:string[]=[]; let size='M';
    return { size:(s:string)=>{size=s; return this as any;},
             add:(x:string)=>{t.push(x); return this as any;},
             build:()=> new (Pizza as any)({size, toppings:t}) }; }
}
```
*Cấu hình:* config DB/SDK qua `process.env` + `requirements`-equivalent là `package.json` (pin version, vd `"pg": "^8.x"` — verify bản mới khi học). Không commit credential.

### ⑥ Keywords
- 🧠 **HIỂU:** 3 nhóm GoF, Factory Method (tách quyết định tạo), Abstract Factory (họ sản phẩm), Builder vs options object, vì sao Singleton là anti-pattern, "module = singleton", Prototype/clone.
- 📌 **THUỘC:** `new X()` ≠ factory; "một instance ≠ phải Singleton"; shallow vs deep clone.
- 🛠️ **LÀM ĐƯỢC:** viết object-map factory; cung cấp một-instance qua DI thay Singleton; clone an toàn.

### ⑦ Mental model
> **Creational = giấu chuyện "đẻ" object. Trong TS: factory thường là một map hàm; Singleton thường là một module; Builder chỉ khi options object không đủ.**

### ⑧ Phỏng vấn & thách đố
1. *(TB)* 3 nhóm GoF giải quyết gì, ví dụ mỗi nhóm? → tạo/ghép/giao tiếp; Factory/Adapter/Strategy.
2. *(TB)* Vì sao `new X()` không phải factory? → lộ concrete, không decouple.
3. *(khó)* Abstract Factory khác Factory Method? → một sản phẩm vs cả họ nhất quán.
4. *(khó)* Vì sao Singleton bị coi anti-pattern? "module là singleton sẵn" nghĩa gì? → global state ẩn; module được cache.
5. 🔥 **Thách đố:** "Cần đúng *một* connection pool toàn app — Singleton chứ gì nữa?" → *Bẫy:* "một instance" không bắt buộc Singleton pattern; dùng **DI single-scope** để vẫn test/override/khởi tạo có thứ tự. Singleton thủ công tạo hidden global dependency.

### ⑨ Bài tập + tiêu chí
- **BT1:** Đổi một `switch(type){...new...}` thành object-map factory. **Đạt khi:** thêm loại mới không sửa hàm tạo.
- **BT2:** Tìm một `getInstance()` Singleton trong code, chuyển sang module export hoặc DI. **Đạt khi:** test có thể inject fake thay vì dùng global thật.

### ⑩ Đọc thêm
- *Design Patterns* (GoF) — phần Creational. *Refactoring Guru* — minh hoạ TS.
- Node.js docs — module caching; MDN — `structuredClone`.
- 🤖 **Kiểm tra AI:** AI thích đẻ Singleton `getInstance()` và Builder cồng kềnh cho thứ chỉ cần options object. Soát: có biến thành global state ẩn không? Builder này có thật sự cần không?

---

# 🟠 BÀI 8 — GoF Structural: Adapter · Facade · Proxy · Decorator · Composite · Bridge

> Phủ: **Q-GOFS-001…008**

### ① Mục tiêu & vị trí trong mạch
Structural = *ghép object thành cấu trúc lớn hơn* mà vẫn linh hoạt. Bài này có bẫy phỏng vấn kinh điển: **Adapter vs Facade vs Proxy** (cùng "bọc" nhưng khác mục đích) và **GoF Decorator ≠ TS/Nest decorator** (verify ở đầu tài liệu). Nối Bài 6 (DIP/port-adapter) và chuẩn bị cho kiến trúc (Bài 11).

### ② Giảng cơ bản → nâng cao

**(a) Adapter (Q-GOFS-001).** Chuyển *interface* của một class sang interface client mong đợi. Use case backend: bọc **SDK bên thứ ba/legacy API** về interface domain của bạn → nối ý với **Anti-Corruption Layer** (DDD, Bài 13/14).

**(b) ⚠️ Adapter vs Facade vs Proxy (Q-GOFS-002) — cùng "wrap", khác *ý đồ*:**
- **Adapter:** đổi interface cho **tương thích** (A → B). "Phích cắm chuyển đổi".
- **Facade:** **đơn giản hoá** một subsystem phức tạp thành một mặt gọn (nhiều thứ → một cửa). "Lễ tân khách sạn".
- **Proxy:** **cùng interface** với đối tượng thật, nhưng **kiểm soát truy cập/thêm hành vi** (lazy/cache/quyền). "Người đại diện".

**(c) Decorator pattern (Q-GOFS-003).** Bọc object **cùng interface**, **thêm trách nhiệm lúc runtime**, có thể **chồng nhiều lớp**. Hợp **OCP** (mở rộng không sửa class gốc). Ví dụ: `LoggingService(CachingService(RealService))` — logging/caching/auth wrap quanh một service.

**(d) ⚠️⚠️ GoF Decorator ≠ TypeScript/Nest decorator (Q-GOFS-004) — bẫy cực phổ biến:**
- **GoF Decorator** = *pattern* wrap object thêm hành vi (runtime, composition).
- **TS/Nest decorator** (`@Injectable`, `@Get`) = *cú pháp annotation* gắn **metadata**/biến đổi declaration **lúc define class**.
- **Trùng tên nhưng KHÁC nhau.** *(Đã verify đầu tài liệu: Nest dùng legacy `experimentalDecorators` + `reflect-metadata`; TC39 Stage 3 là chuẩn mới mặc định từ TS 5.0 nhưng không có parameter decorator nên Nest chưa chuyển.)*

**(e) Proxy — 3 biến thể (Q-GOFS-005):**
- **Virtual:** lazy init — chỉ tạo object đắt khi thật sự cần (vd lazy-load entity nặng).
- **Protection:** kiểm tra **quyền** trước khi cho gọi (authz wrapper).
- **Remote:** đại diện object ở **xa** (RPC stub/client). JS có cả `Proxy` built-in để intercept truy cập.

**(f) Facade vs service layer / God object (Q-GOFS-006).** Facade = mặt gọn cho subsystem; rủi ro phình thành **God object** khi nó **ôm cả business logic**. Ranh giới: Facade **điều phối** (gọi các thành phần), **không chứa** rule nghiệp vụ lõi.

**(g) Composite (Q-GOFS-007).** Xử lý cấu trúc **cây** (object chứa con cùng kiểu) bằng cách cho **leaf** và **composite** chung một interface → client xử lý **đồng nhất** (đệ quy). Ví dụ ngoài UI: **filesystem** (folder/file), org chart, BOM (bill of materials), cây quyền.

**(h) Bridge (Q-GOFS-008).** Tách **abstraction** khỏi **implementation** để **cả hai biến thiên độc lập**. ⚠️ Nhận diện: có **2 trục biến thiên độc lập** (vd `Report` {PDF,HTML} × `Renderer` {Screen,Printer}). Nếu kế thừa thẳng → **m×n class** (combinatorial explosion). Bridge nối 2 phân cấp bằng **composition** → **m+n class**. Khác Adapter: **Bridge thiết kế *trước*** (chủ động tách 2 trục), **Adapter chữa *sau*** (làm 2 thứ có sẵn tương thích).

### ③ ⚠️ Cũ → Mới / Phân biệt

| Hay nhầm | Phân biệt đúng | Vì sao |
|---|---|---|
| Adapter = Facade = Proxy ("đều bọc") | Adapter đổi interface · Facade đơn giản hoá · Proxy kiểm soát truy cập | Khác *mục đích*, không khác "có bọc hay không" |
| GoF Decorator = `@Decorator` của TS/Nest | Pattern wrap object ≠ annotation gắn metadata | Trùng tên, khác bản chất |
| Kế thừa m×n cho 2 trục biến thiên | **Bridge** (composition, m+n) | Tránh class explosion |

### ④ Áp dụng thực tế + So sánh bigtech
- **Adapter:** bọc Stripe/Twilio SDK về `PaymentGateway`/`SmsSender` domain → đổi vendor không lan (ACL).
- **Decorator (pattern):** middleware/interceptor wrap handler thêm logging/auth/caching — Nest `Interceptor` về *ý tưởng* là decorator-pattern (dù khai báo bằng `@`).
- **Bigtech/ecosystem:** ORM repository wrap driver (Adapter); CDN/cache layer trước origin (Proxy); facade SDK gom nhiều micro-API thành một client.

### ⑤ Code thực hành

```typescript
// structural.ts — TypeScript 5.x

// --- Adapter: bọc SDK lạ về interface domain ---
interface SmsSender { send(to: string, text: string): Promise<void>; }       // interface domain
// SDK bên thứ ba có signature khác:
class TwilioSdk { async messages_create(p: {To:string; Body:string}) {/*...*/} }

class TwilioSmsAdapter implements SmsSender {
  constructor(private sdk: TwilioSdk) {}
  async send(to: string, text: string) {
    await this.sdk.messages_create({ To: to, Body: text });                   // chuyển interface
  }
}

// --- Decorator pattern: chồng hành vi runtime, cùng interface ---
interface DataService { read(id: string): Promise<string>; }
class RealDataService implements DataService { async read(id){ return `data:${id}`; } }

class CachingDecorator implements DataService {                               // cùng interface
  private cache = new Map<string,string>();
  constructor(private inner: DataService) {}
  async read(id: string) {
    if (this.cache.has(id)) return this.cache.get(id)!;                       // thêm caching, không sửa Real
    const v = await this.inner.read(id); this.cache.set(id, v); return v;
  }
}
// const svc = new CachingDecorator(new LoggingDecorator(new RealDataService())); // chồng lớp (OCP)

// --- Composite: cây file ---
interface Node { size(): number; }
class FileLeaf implements Node { constructor(private bytes:number){} size(){ return this.bytes; } }
class Folder implements Node {
  private children: Node[] = [];
  add(n: Node){ this.children.push(n); return this; }
  size(){ return this.children.reduce((s,c)=> s + c.size(), 0); }            // đệ quy đồng nhất leaf+composite
}
```

### ⑥ Keywords
- 🧠 **HIỂU:** Adapter/Facade/Proxy (khác *ý đồ*), Decorator pattern + OCP, GoF Decorator ≠ TS decorator, Composite (cây), Bridge (2 trục, m+n vs m×n).
- 📌 **THUỘC:** 3 biến thể Proxy; Bridge-trước vs Adapter-sau.
- 🛠️ **LÀM ĐƯỢC:** viết Adapter bọc SDK; chồng Decorator caching/logging; dựng Composite cây.

### ⑦ Mental model
> **Adapter đổi phích cắm · Facade làm lễ tân · Proxy làm người gác cổng · Decorator dán thêm lớp · Composite gộp cây thành một · Bridge tách hai trục để khỏi nổ số class.**

### ⑧ Phỏng vấn & thách đố
1. *(dễ)* Adapter làm gì? Use case backend? → đổi interface; bọc SDK/legacy.
2. *(TB)* Adapter vs Facade vs Proxy khác *mục đích* sao? → tương thích / đơn giản hoá / kiểm soát truy cập.
3. *(khó)* Phân biệt GoF Decorator với TS/Nest decorator. → wrap object vs annotation metadata.
4. *(rất khó)* Bridge: cho tình huống không dùng Bridge thì class nổ tổ hợp. → Report×Renderer = m×n.
5. 🔥 **Thách đố:** "Facade cho gọn — nhét luôn business logic vào cho tiện?" → *Bẫy:* Facade chỉ *điều phối*; nhét logic → thành God object (Bài 10), cohesion sụp.

### ⑨ Bài tập + tiêu chí
- **BT1:** Bọc một SDK bên thứ ba (mail/sms/payment) bằng Adapter về interface domain. **Đạt khi:** đổi vendor chỉ cần adapter mới, business không đổi.
- **BT2:** Thêm caching cho một service bằng Decorator (không sửa service gốc). **Đạt khi:** bật/tắt caching = bọc/không bọc, code Real giữ nguyên.

### ⑩ Đọc thêm
- *Design Patterns* (GoF) — Structural. *Refactoring Guru*.
- TypeScript Handbook — Decorators; `tc39/proposal-decorators` (để phân biệt với GoF Decorator).
- 🤖 **Kiểm tra AI:** AI hay lẫn "decorator" (sinh `@SomeDecorator` khi bạn muốn pattern wrap, hoặc ngược lại) và hay biến Facade thành God object. Soát đúng *ý đồ* và ranh giới điều-phối-vs-chứa-logic.

---

# 🟠 BÀI 9 — GoF Behavioral: Strategy · Observer · Command · Template Method · State · Chain of Responsibility

> Phủ: **Q-GOFB-001…010**

### ① Mục tiêu & vị trí trong mạch
Behavioral = *cách object giao tiếp & phân chia trách nhiệm khi chạy*. Đây là nhóm bạn gặp nhiều nhất ở backend (middleware = CoR, event = Observer, state machine duyệt đơn = State). Có 2 cặp dễ nhầm: **State vs Strategy** và **Observer vs Pub/Sub**. Nối Bài 3 (Strategy bằng function) và mục G/H (EventEmitter, middleware).

### ② Giảng cơ bản → nâng cao

**(a) Strategy (Q-GOFB-001, 002).** Đóng gói các **thuật toán hoán đổi được** sau cùng một interface, chọn **runtime**. ⚠️ Trong TS, **truyền một hàm vào đã là Strategy** bản gọn. So `if/else` vs Strategy (vd tính phí ship nhiều hãng): Strategy → thêm hãng = thêm class/hàm, **không sửa code cũ** (OCP), test từng strategy riêng; *mất* nếu chỉ 2-3 ca đơn giản (nhiều class/indirection thừa). Cân theo **số biến thể**.

**(b) Observer (Q-GOFB-003).** Quan hệ **một-nhiều**: subject đổi state → **notify** các observer đã subscribe; observer thêm/bớt runtime. Ví dụ: UI update, cache invalidation, domain event.

**(c) ⚠️ Observer vs Pub/Sub (Q-GOFB-004) — khác cốt lõi ở *coupling*:**
- **Observer:** subject **biết & gọi trực tiếp** observer (cùng tiến trình, coupling *vừa*).
- **Pub/Sub:** qua **broker/event channel trung gian**; publisher **không biết** subscriber (decouple *mạnh*, thường **async/cross-process**).
- Nối: `EventEmitter` (in-proc) ≈ Observer; **message broker** (Kafka/RabbitMQ) ≈ Pub/Sub. *(cơ chế cụ thể ở G/H.)*

**(d) ⚠️ EventEmitter lạm dụng (Q-GOFB-005).** Node `EventEmitter` là hiện thân Observer. Rủi ro khi dùng cho **luồng nghiệp vụ**: **control flow ẩn** (khó lần vết), **`error` event chưa handle → crash**, **memory leak** do quên `removeListener`, thứ tự/đồng bộ khó suy luận. → Với nghiệp vụ cần rõ ràng, dùng **call trực tiếp/queue** thay vì event.

**(e) Command (Q-GOFB-006).** Đóng gói **một yêu cầu thành object** (có thể truyền/lưu/hoãn). Mở ra: **undo/redo**, **queue/lập lịch**, **logging/replay/audit**. Ví dụ: job queue (mỗi job là một Command).

**(f) Template Method (Q-GOFB-007).** Lớp cha định **"khung" thuật toán**, để lớp con **override vài bước (hook)**. Dựa trên **inheritance** → coupling base-class **chặt**. Khi cần linh hoạt hơn → cân nhắc **Strategy** (composition thay kế thừa).

**(g) ⚠️ State vs Strategy (Q-GOFB-008) — cấu trúc class giống, ý đồ khác:**
- **State:** object **tự chuyển trạng thái**, hành vi đổi theo state; các state **biết transition** sang nhau.
- **Strategy:** **client/bên ngoài chọn** thuật toán; các strategy **độc lập, không tự đổi** nhau.
- Khác ở **ai điều khiển việc chuyển** và **ý đồ** (quản trạng thái vs hoán thuật toán).

**(h) Chain of Responsibility (Q-GOFB-009).** Chuỗi handler; mỗi handler **xử lý hoặc chuyển tiếp**. ⚠️ **Middleware Express/Nest chính là CoR** (`req → mw1 → mw2 → handler`). Lợi: thêm/bớt/đổi thứ tự khâu xử lý linh hoạt.

**(i) State machine cho workflow (Q-GOFB-010).** Workflow duyệt đơn nhiều bước (tạo → chờ duyệt → duyệt/từ chối → đóng) hành vi khác nhau mỗi bước → mô hình bằng **State pattern/state machine**: gom hành vi + **transition hợp lệ** theo state, tránh `if(status===…)` rải khắp; **chặn transition sai**, dễ thêm state. ⚠️ Nhưng khi đơn giản, **enum + guard** cũng đủ — đừng over-engineer.

### ③ ⚠️ Phân biệt / Cũ → Mới

| Hay nhầm / cũ | Đúng | Vì sao |
|---|---|---|
| State = Strategy (vì class giống) | State tự-chuyển + biết transition; Strategy do ngoài chọn, độc lập | Khác *ai điều khiển* + ý đồ |
| Observer = Pub/Sub | Observer gọi trực tiếp (in-proc); Pub/Sub qua broker (decouple mạnh) | Khác mức coupling/async |
| Dùng EventEmitter cho mọi luồng nghiệp vụ | Event cho thông báo phụ; nghiệp vụ chính → call/queue | Control flow ẩn, leak, error nuốt |
| `if(status===…)` rải khắp cho workflow | **State machine** (transition tường minh) | Chặn chuyển sai, dễ mở rộng |

### ④ Áp dụng thực tế + So sánh bigtech
- **CoR:** mọi framework web (Express/Koa/Nest) = pipeline middleware. Auth → rate-limit → validate → handler.
- **Command:** BullMQ/Sidekiq job = Command (serialize, retry, schedule).
- **State machine:** order/subscription lifecycle; thư viện như XState (TS) mô hình state machine tường minh.
- **Bigtech/ecosystem:** event-driven (Observer/PubSub) là xương sống microservices; Stripe webhooks = pub/sub cross-system.

### ⑤ Code thực hành

```typescript
// behavioral.ts — TypeScript 5.x

// --- Strategy bằng function (idiom TS) ---
type ShipFee = (weightKg: number) => number;
const carriers: Record<string, ShipFee> = {
  ghn: (w) => w * 12000,
  ghtk:(w) => w * 10000 + 5000,        // thêm hãng = thêm entry (OCP), không sửa quote()
};
const quote = (carrier: string, w: number) => (carriers[carrier] ?? (() => { throw new Error('?'); }))(w);

// --- Chain of Responsibility (middleware-style) ---
type Ctx = { user?: string; allowed?: boolean };
type Handler = (ctx: Ctx, next: () => void) => void;
const auth: Handler   = (ctx, next) => { if (!ctx.user) throw new Error('401'); next(); };
const verify: Handler = (ctx, next) => { ctx.allowed = ctx.user === 'admin'; next(); };
function run(handlers: Handler[], ctx: Ctx) {
  let i = 0; const next = () => { if (i < handlers.length) handlers[i++](ctx, next); };
  next();                              // thêm/bớt/đổi thứ tự handler tự do
}

// --- State machine cho duyệt đơn (transition tường minh) ---
type OrderState = 'created' | 'pending' | 'approved' | 'rejected' | 'closed';
const transitions: Record<OrderState, OrderState[]> = {
  created:  ['pending'],
  pending:  ['approved', 'rejected'],
  approved: ['closed'],
  rejected: ['closed'],
  closed:   [],
};
function transition(from: OrderState, to: OrderState): OrderState {
  if (!transitions[from].includes(to)) throw new Error(`Illegal ${from} -> ${to}`); // chặn chuyển sai
  return to;
}
```

### ⑥ Keywords
- 🧠 **HIỂU:** Strategy (hoán thuật toán/function), Observer vs Pub/Sub (coupling), EventEmitter risks, Command (undo/queue), Template Method (inheritance), State vs Strategy, CoR=middleware, state machine cho workflow.
- 📌 **THUỘC:** điểm khác State/Strategy (ai chuyển); điểm khác Observer/PubSub (broker?).
- 🛠️ **LÀM ĐƯỢC:** Strategy bằng map/function; CoR pipeline; state machine với transition guard.

### ⑦ Mental model
> **Strategy = hoán thuật toán (ngoài chọn) · State = máy trạng thái (tự chuyển) · Observer/PubSub = thông báo (trực tiếp vs qua broker) · Command = đóng gói hành động · CoR = dây chuyền middleware.**

### ⑧ Phỏng vấn & thách đố
1. *(TB)* Strategy giải quyết gì? Truyền hàm vào đã là Strategy chưa? → hoán thuật toán; rồi.
2. *(khó)* Observer vs Pub/Sub khác coupling ở đâu? → gọi trực tiếp vs qua broker.
3. *(khó)* State khác Strategy thế nào dù class giống? → ai điều khiển chuyển + ý đồ.
4. *(khó)* CoR liên hệ middleware Express/Nest ra sao? → pipeline handler chuyển tiếp.
5. 🔥 **Thách đố (Q-GOFB-010):** "Workflow 5 bước — cứ `if(status===...)` cho nhanh?" → *Bẫy:* `if` rải khắp khó chặn transition sai & khó mở rộng; **state machine** gom transition hợp lệ. Nhưng nếu chỉ 2 trạng thái tuyến tính thì enum+guard đủ — đừng over-engineer.

### ⑨ Bài tập + tiêu chí
- **BT1:** Mô hình lifecycle một entity (vd subscription) bằng state machine với bảng transition. **Đạt khi:** mọi chuyển sai bị ném lỗi, thêm state mới chỉ sửa bảng.
- **BT2:** Viết pipeline CoR cho request (auth → validate → handle). **Đạt khi:** thêm 1 khâu = thêm 1 handler, không sửa các handler khác.

### ⑩ Đọc thêm
- *Design Patterns* (GoF) — Behavioral. XState docs (state machine TS).
- Node.js docs — `events`/`EventEmitter` (memory leak warning, error event).
- 🤖 **Kiểm tra AI:** AI hay rải `if(status===...)` thay vì state machine, và lạm dụng EventEmitter cho luồng nghiệp vụ (control flow ẩn). Soát: transition có được kiểm soát không? Event có nuốt error/leak listener không?

---

# 🔵 BÀI 10 — Code Smell · Anti-pattern · Refactoring

> Phủ: **Q-SMELL-001…010**

### ① Mục tiêu & vị trí trong mạch
Đây là **mặt sau của Bài 1–9**: khi vi phạm coupling/cohesion/SOLID, bạn ngửi thấy *smell*. Bài này dạy **đặt tên vấn đề** (vocab chung khi review) + **refactor an toàn** (đổi cấu trúc, giữ behavior). Là cầu nối tới kiến trúc (Bài 11–12) và judgment Tech Lead (Bài 15).

### ② Giảng cơ bản → nâng cao

**(a) Code smell là gì (Q-SMELL-001).** *Dấu hiệu bề mặt báo vấn đề thiết kế tiềm ẩn* — **KHÔNG phải bug**, không làm sai chức năng. Vẫn phải quan tâm vì nó báo trước **nợ kỹ thuật**: khó bảo trì/mở rộng nếu để lâu.

**(b) Catalog smell hay gặp:**
- **God Object/God Class (Q-SMELL-002):** một class ôm quá nhiều trách nhiệm/biết quá nhiều → vi phạm **SRP**, cohesion thấp + coupling cao; mọi thay đổi đụng nó → rủi ro. Sửa: tách trách nhiệm.
- **Feature Envy (Q-SMELL-003):** method quan tâm dữ liệu/method của class **khác** hơn class của chính nó → method đặt **sai chỗ**. Refactor: **move method** sang class sở hữu dữ liệu.
- **Divergent Change vs Shotgun Surgery (Q-SMELL-004)** — ngược phương:
  - *Divergent Change:* **một class** đổi vì **nhiều lý do** khác nhau → thiếu SRP → **nên tách**.
  - *Shotgun Surgery:* **một thay đổi** buộc sửa **rải rác nhiều class** → coupling cao/thiếu gom → **nên gộp lại**.
- **Primitive Obsession & Data Clumps (Q-SMELL-005):** lạm dụng kiểu nguyên thuỷ thay kiểu domain (string cho email/tiền); nhóm tham số luôn đi cùng (lat/long/alt). Sửa: tạo **Value Object/object gói** → encapsulate validation (nối DDD — Bài 13).
- **Long Method / Deep Nesting / section-comment (Q-SMELL-006):** method 200 dòng nhiều cấp lồng + comment chia section. Refactor: **extract method** theo block/mức abstraction, **guard clause** giảm nesting, đặt tên ý nghĩa. ⚠️ **Phải có test trước + giữ behavior**.

**(c) Refactoring chặt (theo Fowler — Q-SMELL-007).** *Thay đổi **cấu trúc nội bộ** mà **KHÔNG đổi hành vi quan sát được**.* Test là **lưới an toàn** chứng minh behavior không đổi. ⚠️ Phân biệt: "refactor" (giữ behavior) **≠** "sửa kèm thêm tính năng/đổi behavior". Trộn hai việc làm review/rollback khó.

**(d) Anti-pattern vs code smell (Q-SMELL-008).** **Anti-pattern** = một "giải pháp" **phổ biến nhưng phản tác dụng** (vs smell là *dấu hiệu*). Ví dụ backend: **Golden Hammer** (cái gì cũng dùng 1 công cụ quen), **Lava Flow** (code chết không ai dám xoá), **Spaghetti**, **Magic Numbers/Strings**, **premature optimization**. Chúng "có vẻ đúng" vì giải quyết được trước mắt nhưng tạo nợ về sau.

**(e) ⚠️ Lạm dụng *design pattern* cũng là smell (Q-SMELL-009).** "Pattern đúng nhưng dùng sai chỗ": Decorator chồng quá sâu → chuỗi object khó debug/overhead; Singleton tạo global state; Factory cho thứ chỉ `new` một lần. **Pattern là công cụ có chi phí, không phải huy chương.**

**(f) Smell vs deadline — quyết định Tech Lead (Q-SMELL-010).** Thấy smell nhưng deadline gấp: phân loại theo **rủi ro lan / tần suất đụng / độ rủi ro thay đổi** → **fix-now / log nợ / bỏ qua** *có chủ đích*. Log nợ **tường minh** (ticket/TODO có chủ + lý do). ⚠️ Tránh "boy-scout" phá scope PR (sửa lan man ngoài mục tiêu). Cân **chất lượng vs tốc độ giao**.

### ③ ⚠️ Cũ → Mới / Lưu ý

| Hiểu sai | Đúng | Vì sao |
|---|---|---|
| "Smell = bug" | Smell = dấu hiệu thiết kế, không sai chức năng | Quản nợ kỹ thuật, không phải sửa lỗi gấp |
| Refactor tiện thể thêm feature | Refactor = giữ behavior; tách PR feature riêng | Review/rollback rõ ràng |
| Pattern càng nhiều càng "pro" | Pattern có chi phí; lạm dụng = smell | Indirection/overhead vô ích |
| Refactor không cần test | Test là điều kiện bắt buộc (lưới an toàn) | Chứng minh behavior không đổi |

### ④ Áp dụng thực tế + So sánh bigtech
- **Use case:** trước khi refactor một `OrderProcessor` 400 dòng, viết **characterization test** (chụp lại behavior hiện tại) → extract method từng bước → chạy test sau mỗi bước.
- **Bigtech/ecosystem:** Google "readability review" + linters bắt smell tự động; "tech debt register" (sổ nợ kỹ thuật) là thực hành phổ biến để log nợ có chủ đích thay vì sửa bừa.

### ⑤ Code thực hành

```typescript
// refactor-long-method.ts — guard clause + extract method (giữ behavior)

// ❌ Trước: long method, deep nesting, section comments
function processOrderBad(o: any) {
  if (o) {
    if (o.items.length > 0) {
      if (o.paid) {
        // ... 50 dòng tính toán ...
      }
    }
  }
}

// ✅ Sau: guard clause (giảm nesting) + extract method (đặt tên theo ý nghĩa)
function processOrder(o: Order) {
  if (!o) throw new Error('Order required');           // guard clause
  if (o.items.length === 0) throw new Error('Empty');  // guard clause
  if (!o.paid) throw new Error('Unpaid');

  const subtotal = calcSubtotal(o.items);              // extract method
  const tax = calcTax(subtotal);
  return subtotal + tax;
}
function calcSubtotal(items: Item[]) { return items.reduce((s,i)=> s + i.price, 0); }
function calcTax(subtotal: number) { return subtotal * 0.1; }

// --- Primitive Obsession -> Value Object ---
// ❌ function pay(amount: number, currency: string) {}   // hai primitive luôn đi cặp = Data Clump
class Money {                                            // ✅ Value Object: gói + validate
  private constructor(readonly amount: number, readonly currency: string) {}
  static of(amount: number, currency: string) {
    if (amount < 0) throw new Error('amount >= 0');      // encapsulate validation
    return new Money(amount, currency);
  }
}
```

### ⑥ Keywords
- 🧠 **HIỂU:** smell (≠ bug), God Object, Feature Envy, Divergent Change vs Shotgun Surgery, Primitive Obsession/Data Clumps, refactoring (Fowler), anti-pattern, pattern-abuse.
- 📌 **THUỘC:** định nghĩa refactoring (giữ behavior + cần test); Divergent (tách) vs Shotgun (gộp).
- 🛠️ **LÀM ĐƯỢC:** extract method + guard clause; biến Data Clump thành Value Object; log nợ kỹ thuật có chủ đích.

### ⑦ Mental model
> **Smell = mùi báo nợ, không phải lửa cháy. Refactor = đổi *bên trong*, giữ *bên ngoài*, có test làm lưới. Pattern là công cụ có giá, không phải huy chương.**

### ⑧ Phỏng vấn & thách đố
1. *(TB)* Code smell là gì, có phải bug không? → dấu hiệu thiết kế, không sai chức năng.
2. *(TB)* God Object nguy hiểm sao, vi phạm gì? → ôm nhiều trách nhiệm, SRP/cohesion/coupling.
3. *(khó)* Divergent Change vs Shotgun Surgery? → một class đổi nhiều lý do (tách) vs một đổi sửa nhiều class (gộp).
4. *(khó)* Định nghĩa chặt refactoring? Vì sao cần test? Có được đổi behavior? → đổi cấu trúc, giữ behavior; test=lưới; không đổi behavior.
5. 🔥 **Thách đố (Q-SMELL-009):** "Dự án nên dùng nhiều design pattern để chứng tỏ chất lượng?" → *Bẫy:* lạm dụng pattern = smell (overhead/indirection); pattern phải *giải vấn đề cụ thể*. Singleton/Decorator/ Factory sai chỗ gây hại.

### ⑨ Bài tập + tiêu chí
- **BT1:** Lấy method dài nhất trong code bạn, viết test characterization rồi extract method + guard clause. **Đạt khi:** test xanh trước/sau, không đổi behavior, nesting giảm.
- **BT2:** Tìm 1 Data Clump (vd `lat, lng` luôn đi cùng), gói thành Value Object có validation. **Đạt khi:** không tạo được VO ở trạng thái sai.

### ⑩ Đọc thêm
- Martin Fowler — *Refactoring* (2nd ed.) + catalog smell tại refactoring.com.
- *AntiPatterns* (Brown et al.) — Golden Hammer, Lava Flow…
- 🤖 **Kiểm tra AI:** AI thường sinh long method, God service, magic number, primitive obsession "vì chạy được". Và khi "refactor" có thể **đổi behavior** lén. Soát: có test trước không? behavior có đổi không? smell nào xuất hiện?

---

# 🟣 BÀI 11 — Kiến trúc (1): Layered · Clean · Hexagonal · Onion · Dependency Rule · Screaming

> Phủ: **Q-ARCH-001…006, Q-ARCH-009**

### ① Mục tiêu & vị trí trong mạch
Đây là **DIP (Bài 6) nâng lên cấp kiến trúc**: tách domain khỏi infra, đảo chiều phụ thuộc vào trong. Hiểu xong bạn đọc được vì sao `import { PrismaClient }` trong entity là sai, vì sao repo interface nằm ở domain. Bài 12 lo phần nâng cao (anemic, CQRS, refactor monolith); cách tổ chức module Nest cụ thể ở mục H-MOD.

### ② Giảng cơ bản → nâng cao

**(a) Layered architecture (Q-ARCH-001).** Tách trách nhiệm theo tầng **presentation / business / data**; phụ thuộc đi **một chiều xuống dưới**; tầng trên không bị tầng dưới gọi ngược. Lợi: đổi một tầng (vd đổi DB) ít ảnh hưởng tầng khác.

**(b) ⚠️ The Dependency Rule (Clean Architecture — Q-ARCH-002).** *Phụ thuộc chỉ được trỏ **vào trong** — về phía policy/domain.* Domain/business **KHÔNG** phụ thuộc framework/DB/web. Vì sao: domain là phần **ổn định & giá trị nhất**; tách khỏi chi tiết → dễ test, thay hạ tầng không đụng nghiệp vụ. Đây là **khác biệt cốt lõi** giữa layered "ngây thơ" (business → data) và clean (business ← data qua interface).

**(c) Hexagonal / Ports & Adapters (Q-ARCH-003).**
- **Port** = *interface do domain định nghĩa* (cái domain *cần*).
- **Adapter** = *implementation cụ thể* nối ra ngoài.
- **Driving/primary** = thứ **gọi vào** app (HTTP controller, CLI, test).
- **Driven/secondary** = thứ **app gọi ra** (DB, mail, queue).
- App core **không biết** adapter nào đang cắm.

**(d) Clean = Onion = Hexagonal? (Q-ARCH-004).** Về bản chất là **cùng một tư tưởng**: tách domain khỏi infra + đảo chiều phụ thuộc vào trong. Khác chỉ ở **thuật ngữ/cách vẽ/nhấn mạnh**. → Đừng tranh cãi tên, hiểu **nguyên tắc chung**.

**(e) ⚠️ Vì sao repo interface ở domain, impl ở infra (Q-ARCH-005).** Domain **sở hữu port** (abstraction). Infra **implement** port → mũi tên phụ thuộc trỏ **từ ngoài vào domain** (DIP ở cấp kiến trúc). Domain **không import** infra, chỉ import abstraction của chính nó. Đây chính là **cơ chế đảo chiều**.

**(f) ⚠️ `import { PrismaClient }` trong entity = sai (Q-ARCH-006).** Domain **rò rỉ phụ thuộc ORM/infra** → **đảo ngược** dependency rule; khó test/thay DB, business trộn persistence. Sửa: định nghĩa **repository port** ở domain + **adapter Prisma** ở tầng ngoài.

**(g) Screaming Architecture (Q-ARCH-009 — Uncle Bob).** Cấu trúc thư mục nên **"hét lên" domain/use case** (`orders/`, `billing/`) chứ **không** phải framework/layer (`controllers/`, `services/`, `models/`). Lợi: nhìn cây thư mục biết hệ thống **làm gì**, không phải "viết bằng gì". Nối với feature-module (cross-ref H-MOD).

### ③ ⚠️ Cũ → Mới

| Cũ / ngây thơ | Nay (clean) | Vì sao |
|---|---|---|
| business `import` DB/ORM trực tiếp | business định nghĩa **port**, infra implement | Dependency Rule: phụ thuộc trỏ vào trong |
| Entity chứa `PrismaClient`/SQL | Entity thuần domain; persistence ở adapter | Tách domain khỏi infra → test/thay được |
| Thư mục theo layer (`controllers/`, `models/`) | Thư mục theo **domain/use case** (Screaming) | Cấu trúc phản ánh *hệ thống làm gì* |
| Tranh cãi Clean vs Onion vs Hexagonal | Hiểu **cùng một tư tưởng** | Khác tên, chung nguyên tắc |

### ④ Áp dụng thực tế + So sánh bigtech
- **Use case:** `domain/order/OrderRepository.ts` (port) ← `infra/persistence/PrismaOrderRepository.ts` (adapter). Controller (driving adapter) gọi use case; use case gọi port; Prisma adapter cắm vào port. Đổi sang TypeORM = thêm adapter, domain im.
- **Bigtech/ecosystem:** Netflix/Uber backend tách "core domain" khỏi infra để swap datastore/transport. Spring/NestJS hỗ trợ DI để hiện thực port-adapter. ⚠️ Nhiều startup *cố tình* dùng layered đơn giản hơn khi domain mỏng (Bài 12 — trade-off).

### ⑤ Code thực hành

```typescript
// hexagonal.ts — port ở domain, adapter ở infra

// ===== domain layer (KHÔNG import infra) =====
export interface Order { id: string; total: number; }
export interface OrderRepository {                 // PORT do domain định nghĩa
  findById(id: string): Promise<Order | null>;
  save(order: Order): Promise<void>;
}
export class PlaceOrderUseCase {                    // business: phụ thuộc PORT, không phụ thuộc DB
  constructor(private readonly repo: OrderRepository) {}
  async exec(order: Order) {
    if (order.total <= 0) throw new Error('Invalid total'); // domain rule
    await this.repo.save(order);
  }
}

// ===== infra layer (ADAPTER implement port; phụ thuộc trỏ VÀO domain) =====
// import { PrismaClient } from '@prisma/client';   // verify version ở docs chính thức
export class PrismaOrderRepository implements OrderRepository {
  // constructor(private prisma: PrismaClient) {}
  async findById(id: string) { /* prisma.order.findUnique... */ return null; }
  async save(order: Order) { /* prisma.order.create... */ }
}
// ❌ Sai: trong domain/Order.ts mà `import { PrismaClient }` -> domain rò infra (vi phạm Dependency Rule)
```
*Cấu hình (enforce):* dùng `dependency-cruiser`/`eslint-plugin-boundaries` (đã verify) để **chặn** `domain/` import `infra/` trong CI — biến Dependency Rule thành lint (xem Bài 15).

### ⑥ Keywords
- 🧠 **HIỂU:** layered, Dependency Rule, port/adapter, driving vs driven, Clean=Onion=Hexagonal, repo port ở domain, Screaming Architecture.
- 📌 **THUỘC:** "phụ thuộc trỏ vào trong"; vì sao entity không được import ORM.
- 🛠️ **LÀM ĐƯỢC:** đặt port ở domain + adapter ở infra; nhận diện vi phạm dependency rule trong code.

### ⑦ Mental model
> **Domain ở lõi, không biết thế giới bên ngoài. Mọi mũi tên phụ thuộc đều chĩa *vào trong*. Port = cái domain cần; Adapter = cái thế giới cắm vào. Thư mục phải "hét" domain, không "hét" framework.**

### ⑧ Phỏng vấn & thách đố
1. *(TB)* Layered architecture + quy tắc phụ thuộc giữa tầng? → tách tầng, phụ thuộc một chiều xuống.
2. *(khó)* Dependency Rule phát biểu sao? Vì sao domain không phụ thuộc framework/DB? → trỏ vào trong; domain ổn định/test được.
3. *(khó)* Port vs adapter; driving vs driven? → interface domain / impl; gọi-vào / app-gọi-ra.
4. *(khó)* Vì sao repo interface ở domain mà impl ở infra? → domain sở hữu abstraction, đảo chiều (DIP).
5. 🔥 **Thách đố (Q-ARCH-006):** "Thấy `import { PrismaClient }` trong entity domain — sai chỗ nào?" → *Bẫy:* domain rò infra, đảo ngược dependency rule, khó test/thay DB. Sửa: port + adapter.

### ⑨ Bài tập + tiêu chí
- **BT1:** Tách một service đang gọi ORM trực tiếp thành use case + port + adapter. **Đạt khi:** domain/use case không import gì từ ORM; test inject in-memory repo.
- **BT2:** Đổi cấu trúc thư mục từ layer sang Screaming (theo domain). **Đạt khi:** nhìn thư mục đoán được nghiệp vụ, không phải framework.

### ⑩ Đọc thêm
- R. C. Martin — *Clean Architecture*; "Screaming Architecture" (blog). Alistair Cockburn — Hexagonal/Ports & Adapters. Jeffrey Palermo — Onion Architecture.
- 🤖 **Kiểm tra AI:** AI hay nhét ORM/HTTP thẳng vào domain/entity (vì "chạy được") — vi phạm dependency rule. Soát: domain có import infra không? Đây chính là chỗ "AI viết đúng cú pháp nhưng sai kiến trúc" → cần guardrail lint (Bài 15).

---

# 🟣 BÀI 12 — Kiến trúc (2): Anemic Model · Application/Domain Service · CQRS · Refactor Big-Ball-of-Mud

> Phủ: **Q-ARCH-007, 008, 010, 011, 012**

### ① Mục tiêu & vị trí trong mạch
Phần nâng cao + **judgment kiến trúc**: khi nào full-clean đáng, anemic model là sai hay là lựa chọn, tách read/write (CQRS), và **refactor monolith không big-bang**. Nối DDD (Bài 13–14) và Tech Lead (Bài 15).

### ② Giảng cơ bản → nâng cao

**(a) Anemic Domain Model (Q-ARCH-007 — Fowler coi anti-pattern).** Entity chỉ là **túi getter/setter**, **mọi logic dồn vào service** → mất encapsulation/OOP (Bài 1). DDD muốn **hành vi + invariant nằm trong domain object**. ⚠️ Nhưng đây là **trade-off có chủ đích**: với CRUD đơn giản/đội quen layered-service, anemic + service dày **chấp nhận được** — không phải *luôn* sai. (Nhiều team Nest dùng anemic entity + service rất hiệu quả cho domain mỏng.)

**(b) Application Service vs Domain Service vs Controller (Q-ARCH-010).**
- **Controller** = adapter I/O (nhận request, trả response). Mỏng.
- **Application/Use-case Service** = **điều phối luồng** (orchestrate), quản **transaction/unit-of-work**, gọi ra ngoài (port). **Không chứa rule lõi**.
- **Domain Service** = **rule nghiệp vụ** liên **nhiều aggregate**, **không chạm hạ tầng**.
- *(verify: phân định này là quy ước cộng đồng, không tuyệt đối.)*

**(c) ⚠️ CQRS (Q-ARCH-011).** Tách **model ghi** (giữ invariant) khỏi **model đọc** (tối ưu truy vấn/projection). Đáng khi read/write **rất khác nhau** hoặc **khác tải** (đọc nhiều, ghi phức tạp). **Thừa** khi CRUD đơn giản. ⚠️ CQRS **không bắt buộc** đi kèm **Event Sourcing** — đừng gộp hai khái niệm.

**(d) Full-clean cho microservice CRUD mỏng có đáng? (Q-ARCH-008).** Hexagonal/clean thêm **layer/mapping/interface**. Chi phí ports/adapters/mapping **chỉ đáng** khi domain đủ **phức tạp/biến động**. CRUD mỏng → **over-engineering**. Quyết theo **độ phức tạp nghiệp vụ + vòng đời + đội ngũ**, không giáo điều.

**(e) Refactor big-ball-of-mud không big-bang (Q-ARCH-012).** Monolith tầng trộn lẫn → **đừng rewrite toàn bộ** (rủi ro cao). Lộ trình **incremental**:
1. **Characterization test** làm lưới an toàn (chụp behavior hiện tại).
2. **Bọc seam** bằng interface tại ranh giới.
3. Tách **domain dần**, đẩy **I/O ra adapter**.
4. **Strangler-fig**: viết tính năng mới theo kiến trúc mới, dần "siết" phần cũ cho tới khi thay hết.

### ③ ⚠️ Cũ → Mới / Lưu ý

| Quan niệm | Sắc thái đúng | Vì sao |
|---|---|---|
| "Anemic model luôn sai" | Anti-pattern theo DDD nhưng là **trade-off** hợp lệ với domain mỏng | Tuỳ độ phức tạp/đội |
| "CQRS = Event Sourcing" | CQRS chỉ tách read/write; ES độc lập | Hai khái niệm khác nhau |
| "Full clean cho mọi service" | Chỉ khi domain phức tạp/biến động | Tránh over-engineering CRUD |
| "Refactor = rewrite lại từ đầu" | Incremental: seam + strangler-fig + test | Big-bang rewrite rủi ro cao |

### ④ Áp dụng thực tế + So sánh bigtech
- **CQRS thật:** trang sản phẩm đọc cực nhiều (read model denormalized/cache) trong khi đặt hàng ghi phức tạp (write model giữ invariant) → tách hai phía.
- **Strangler-fig:** Amazon/Shopify tách monolith dần bằng cách route tính năng mới sang service mới, giữ cũ chạy song song.
- **Bigtech/ecosystem:** Martin Fowler đặt tên "Strangler Fig"; nhiều cuộc migration lớn (monolith→service) theo pattern này thay vì rewrite.

### ⑤ Code thực hành

```typescript
// layering-roles.ts — phân vai Controller / App Service / Domain Service

// Domain Service: rule liên nhiều aggregate, KHÔNG chạm hạ tầng
class TransferPolicy {
  ensureCanTransfer(from: Account, to: Account, amount: number) {
    if (from.id === to.id) throw new Error('Same account');
    if (amount <= 0) throw new Error('Amount > 0');       // pure business rule
  }
}

// Application Service: điều phối use case + transaction + gọi PORT ra ngoài
class TransferMoneyUseCase {
  constructor(
    private readonly accounts: AccountRepository,         // port (driven)
    private readonly policy: TransferPolicy,
    private readonly tx: UnitOfWork,                      // quản transaction
  ) {}
  async exec(fromId: string, toId: string, amount: number) {
    await this.tx.run(async () => {
      const from = await this.accounts.get(fromId);
      const to   = await this.accounts.get(toId);
      this.policy.ensureCanTransfer(from, to, amount);    // gọi domain rule
      from.withdraw(amount); to.deposit(amount);          // hành vi trong entity (chống anemic)
      await this.accounts.save(from); await this.accounts.save(to);
    });
  }
}
// Controller (adapter I/O): mỏng — parse request, gọi use case, map response. (xem H-MOD)
```

### ⑥ Keywords
- 🧠 **HIỂU:** anemic model (anti-pattern nhưng trade-off), Controller/App Service/Domain Service, CQRS (≠ ES), full-clean trade-off, strangler-fig.
- 📌 **THUỘC:** ranh giới 3 loại service; CQRS không bắt buộc Event Sourcing.
- 🛠️ **LÀM ĐƯỢC:** đặt rule vào đúng tầng; vạch lộ trình refactor monolith incremental.

### ⑦ Mental model
> **Controller mỏng (I/O) · App Service điều phối + transaction · Domain Service = rule liên aggregate · Entity giữ hành vi (chống anemic). Refactor monolith: bóp cổ dần (strangler), đừng đập đi xây lại.**

### ⑧ Phỏng vấn & thách đố
1. *(khó)* Anemic Domain Model là gì, vì sao Fowler coi anti-pattern? → entity rỗng, logic ở service; mất encapsulation.
2. *(khó)* App Service vs Domain Service vs Controller? → điều phối+tx / rule liên aggregate / adapter I/O.
3. *(rất khó)* CQRS giải gì? Khi nào tách read/write hợp lý, khi nào thừa? → tách model; khi read/write khác nhau/tải; CRUD thì thừa.
4. *(rất khó)* Full clean cho CRUD microservice mỏng — đáng không, quyết theo gì? → thường over-engineering; theo độ phức tạp/vòng đời/đội.
5. 🔥 **Thách đố (Q-ARCH-012):** "Monolith big-ball-of-mud — đề xuất rewrite toàn bộ?" → *Bẫy:* big-bang rewrite rủi ro cực cao; incremental: characterization test + seam + strangler-fig theo module.

### ⑨ Bài tập + tiêu chí
- **BT1:** Lấy 1 entity anemic + service dày, chuyển 1-2 rule vào entity (chống anemic). **Đạt khi:** entity không rơi vào trạng thái sai; service mỏng hơn.
- **BT2:** Phác lộ trình strangler-fig cho 1 module monolith bạn biết. **Đạt khi:** có bước test-an-toàn + seam + thứ tự siết, không rewrite một phát.

### ⑩ Đọc thêm
- Fowler — "AnemicDomainModel", "StranglerFigApplication". Greg Young / Udi Dahan — CQRS.
- 🤖 **Kiểm tra AI:** AI hay sinh anemic (logic dồn vào service), gợi ý CQRS/Event Sourcing "cho xịn" lên CRUD mỏng, hoặc đề xuất rewrite. Soát: domain có cần phức tạp thế? Rule đặt đúng tầng chưa?

---

# 🔴 BÀI 13 — DDD (1): Strategic — Bounded Context · Ubiquitous Language · Entity vs Value Object

> Phủ: **Q-DDD-001, 002, 003**

### ① Mục tiêu & vị trí trong mạch
DDD (Eric Evans, 2003) là cách **mô hình hoá nghiệp vụ phức tạp** trong khung kiến trúc Bài 11–12. Bài này lo phần **strategic** (Bounded Context, Ubiquitous Language) — phần **luôn hữu ích** kể cả khi không làm DDD đầy đủ — và hai building block nền: **Entity vs Value Object**. Bài 14 lo tactical (aggregate, repository, service, event) + khi nào KHÔNG dùng DDD.

### ② Giảng cơ bản → nâng cao

**(a) Bounded Context (Q-DDD-001).** *Ranh giới trong đó một model/ngôn ngữ **nhất quán**.* ⚠️ "Một model duy nhất cho toàn hệ thống" là **kỳ vọng sai**: cùng từ **"Customer"** mang **nghĩa khác** giữa context (trong **Sales** là lead/đơn hàng; trong **Billing** là tài khoản thanh toán; trong **Support** là ticket owner). Một model toàn cục → **phình + mâu thuẫn**. Chia context → mỗi context model gọn, giảm phức tạp. Đây là **strategic DDD** — quyết định ranh giới trước khi code.

**(b) Ubiquitous Language (Q-DDD-002).** *Ngôn ngữ chung* dùng **nhất quán trong cả trò chuyện *và* code** giữa dev và domain expert. Giải quyết vấn đề **dịch sai** giữa nghiệp vụ ↔ kỹ thuật. Hệ quả: **tên class/method phản ánh đúng thuật ngữ domain** (`Order.place()`, không phải `OrderManager.processData()`). Ngôn ngữ này **gắn với từng bounded context** (cùng "Customer" nhưng nghĩa theo context).

**(c) Entity vs Value Object (Q-DDD-003) — phân biệt nền tảng:**
- **Entity:** có **identity** riêng, **tồn tại qua thời gian** dù thuộc tính đổi. Ví dụ `User#42` (đổi tên/email vẫn là user đó). So sánh bằng **ID**.
- **Value Object:** định danh bằng **giá trị**, **bất biến** — thay vì sửa thì **tạo mới**. Ví dụ `Money(100, 'USD')`, `Address`, `DateRange`. So sánh bằng **giá trị** (hai `Money(100,'USD')` là bằng nhau).
- **Tiêu chí phân biệt:** *"có cần identity không?"* Cần → Entity; không → Value Object.
- (Nối Bài 10: biến Primitive Obsession/Data Clumps thành Value Object.)

### ③ ⚠️ Cũ → Mới

| Hiểu sai | Đúng | Vì sao |
|---|---|---|
| "Một model chung cho cả hệ thống" | Chia **Bounded Context**, model nhất quán *trong* ranh giới | Cùng từ khác nghĩa giữa context |
| Tên kỹ thuật mơ hồ (`DataManager`, `Helper`) | Tên theo **Ubiquitous Language** domain | Giảm dịch sai dev↔expert |
| "Cứ là object thì như nhau" | Phân **Entity (identity)** vs **Value Object (giá trị, immutable)** | Quyết cách so sánh/lưu/sửa |

### ④ Áp dụng thực tế + So sánh bigtech
- **Use case:** e-commerce tách context `Catalog` / `Ordering` / `Billing` / `Shipping` — mỗi context có model "Product"/"Order" riêng, giao tiếp qua ID/event (Bài 14).
- **Value Object thật:** `Money` (tránh lỗi cộng tiền khác currency), `EmailAddress` (validate một chỗ).
- **Bigtech/ecosystem:** microservices thường **ánh xạ 1 service ≈ 1 bounded context**; Context Map (DDD) là công cụ vạch ranh giới trước khi chia service. (Strategic DDD hữu ích kể cả khi không làm tactical đầy đủ.)

### ⑤ Code thực hành

```typescript
// ddd-blocks.ts — Entity vs Value Object

// Value Object: immutable, so sánh bằng giá trị, validate khi tạo
class Money {
  private constructor(readonly amount: number, readonly currency: string) {}
  static of(amount: number, currency: string): Money {
    if (amount < 0) throw new Error('amount >= 0');
    return new Money(amount, currency);
  }
  add(other: Money): Money {
    if (other.currency !== this.currency) throw new Error('Currency mismatch'); // invariant
    return Money.of(this.amount + other.amount, this.currency);                  // tạo mới, không sửa
  }
  equals(o: Money) { return this.amount === o.amount && this.currency === o.currency; } // bằng theo giá trị
}

// Entity: có identity, so sánh bằng ID, thuộc tính đổi vẫn là cùng entity
class User {
  constructor(readonly id: string, private name: string) {}
  rename(name: string) { this.name = name; }                  // đổi thuộc tính, vẫn User#id
  equals(o: User) { return this.id === o.id; }                // bằng theo identity
}
```

### ⑥ Keywords
- 🧠 **HIỂU:** Bounded Context, Ubiquitous Language, Entity (identity) vs Value Object (giá trị, immutable), strategic DDD.
- 📌 **THUỘC:** tiêu chí Entity/VO ("cần identity không?"); "một model toàn cục là sai".
- 🛠️ **LÀM ĐƯỢC:** dựng Value Object immutable có validation; đặt tên theo Ubiquitous Language; vạch context.

### ⑦ Mental model
> **Mỗi Bounded Context là một "thế giới" có ngôn ngữ riêng. Entity = "ai" (có identity, sống qua thời gian); Value Object = "cái gì/bao nhiêu" (bất biến, bằng nhau theo giá trị).**

### ⑧ Phỏng vấn & thách đố
1. *(khó)* Bounded Context là gì? Vì sao "một model duy nhất" là sai? → ranh giới model nhất quán; cùng từ khác nghĩa.
2. *(TB)* Ubiquitous Language giải quyết gì? → ngôn ngữ chung dev↔expert, vào cả code.
3. *(khó)* Entity vs Value Object, tiêu chí phân biệt? → identity vs giá trị; "cần identity không?".
4. 🔥 **Thách đố:** "`Money` cứ để là `number` cho gọn?" → *Bẫy:* primitive obsession → cộng nhầm currency, validate rải rác. VO `Money` gói invariant một chỗ.

### ⑨ Bài tập + tiêu chí
- **BT1:** Vạch 2-3 bounded context cho một domain bạn biết (vd shop), nêu 1 từ mang nghĩa khác giữa các context. **Đạt khi:** chỉ ra được xung đột nghĩa nếu gộp model.
- **BT2:** Biến một cụm primitive (vd `email: string`) thành Value Object có validate. **Đạt khi:** không tạo được VO sai định dạng.

### ⑩ Đọc thêm
- Eric Evans — *Domain-Driven Design* (2003); Vaughn Vernon — *Implementing DDD*.
- 🤖 **Kiểm tra AI:** AI hay gộp model toàn cục + dùng primitive thay VO + đặt tên kỹ thuật mơ hồ. Soát: tên có theo ngôn ngữ domain không? Có VO nào nên gói không?

---

# 🔴 BÀI 14 — DDD (2): Aggregate · Invariant · Transaction Boundary · Repository · Domain/App Service · Domain Event · khi nào KHÔNG dùng DDD

> Phủ: **Q-DDD-004, 005, 006, 007, 008, 009, 010**

### ① Mục tiêu & vị trí trong mạch
Phần **tactical DDD** — building block để giữ **tính nhất quán** của domain phức tạp, và **judgment**: khi nào DDD đầy đủ là overhead. Đây là đỉnh của mạch "mô hình hoá nghiệp vụ", chuẩn bị cho Tech Lead judgment (Bài 15).

### ② Giảng cơ bản → nâng cao

**(a) Aggregate & Aggregate Root (Q-DDD-004).** **Aggregate** = cụm object là **một consistency boundary** (một đơn vị nhất quán). **Root** = **cửa duy nhất** truy cập/sửa bên trong. ⚠️ Quy tắc **tham chiếu aggregate khác bằng ID, không nhúng object**: giữ boundary, tránh load/sửa chéo, cho phép **eventual consistency**. (Vd `Order` chứa `OrderLine` bên trong; tham chiếu `Customer` bằng `customerId`, không nhúng cả `Customer`.)

**(b) Invariant (Q-DDD-005).** *Quy tắc nghiệp vụ **luôn phải đúng** cho aggregate.* Vd "tổng item ≤ hạn mức đơn". **Mọi thay đổi đi qua root** để root kiểm tra/giữ nhất quán nội bộ → đây là lý do encapsulation (Bài 1) quan trọng: không cho sửa thẳng vào trong aggregate.

**(c) Aggregate boundary ≈ Transaction boundary (Q-DDD-006).** Một aggregate = một **đơn vị nhất quán giao dịch**: một transaction nên sửa **một** aggregate. ⚠️ Khi thao tác phải đổi **nhiều** aggregate → **đừng** một transaction khổng lồ; dùng **eventual consistency / domain event / saga**. (Boundary to → contention/lock nhiều; boundary nhỏ → nhiều cross-aggregate. Cân theo invariant thật.)

**(d) Repository trong DDD (Q-DDD-007).** Khác DAO/ORM thường: repository là **collection-like cho aggregate root**, **ẩn persistence**, **trả về domain object** (không leak ORM entity). Interface theo **nhu cầu domain** (`findActiveOrdersFor(customer)`) chứ không theo *khả năng DB*. → "nói ngôn ngữ domain". *(persistence/transaction/N+1 chi tiết ở mục E.)*

**(e) Domain Service vs Application Service (Q-DDD-008).**
- **Domain Service** = rule nghiệp vụ **thuần**, liên nhiều aggregate, **không chạm hạ tầng**.
- **Application Service** = **điều phối use case**, quản **transaction**, **gọi ra ngoài** (port).
- ⚠️ "Gửi email xác nhận sau khi đặt đơn" = **side-effect hạ tầng** → **Application Service** (qua port), **KHÔNG** nhét vào domain. (Nối Bài 11–12.)

**(f) Domain Event (Q-DDD-009).** Sự kiện **"đã xảy ra"** trong domain (`OrderPlaced`), **phát ra** cho phần khác **phản ứng async**. Giảm coupling: aggregate **không gọi trực tiếp** aggregate/context khác → **loose coupling + eventual consistency**. (Nối Observer/PubSub — Bài 9; cơ chế wiring ở G/H.)

**(g) ⚠️ Khi nào KHÔNG dùng DDD đầy đủ (Q-DDD-010) — judgment.** DDD tactical có **chi phí học/triển khai cao**. **Khuyên không** áp đầy đủ khi: **CRUD đơn giản / ít rule / đội nhỏ** → overhead. **Đáng** khi: nghiệp vụ phức tạp, **nhiều invariant**, ngôn ngữ domain giàu, **thay đổi liên tục**. ⚠️ Phân biệt: **strategic DDD** (bounded context — **luôn hữu ích**) vs **tactical DDD** (aggregate/repo/VO — **chọn lọc**).

### ③ ⚠️ Cũ → Mới

| Hiểu sai / cũ | Đúng | Vì sao |
|---|---|---|
| Nhúng object aggregate khác vào trong | Tham chiếu bằng **ID** | Giữ boundary, cho eventual consistency |
| Một transaction sửa nhiều aggregate | Một tx ~ một aggregate; nhiều → event/saga | Tránh lock lớn, theo consistency boundary |
| Repository = DAO trả ORM entity | Repo trả **domain object**, nói ngôn ngữ domain | Không leak persistence vào domain |
| Gửi email trong domain | Side-effect → **Application Service** (qua port) | Domain không chạm hạ tầng |
| "DDD cho mọi service" | Tactical chọn lọc; strategic luôn hữu ích | DDD đầy đủ là overhead cho CRUD |

### ④ Áp dụng thực tế + So sánh bigtech
- **Aggregate:** `Order` (root) + `OrderLine` (trong). Invariant "tổng ≤ limit" kiểm tại `order.addLine()`. Tham chiếu `customerId` (ID), không nhúng `Customer`.
- **Domain Event + Saga:** `OrderPlaced` → service Inventory trừ kho, service Notification gửi mail — async, mỗi aggregate một tx (eventual consistency).
- **Bigtech/ecosystem:** event-driven microservices (Kafka) hiện thực domain event ở cấp hệ thống; saga pattern điều phối transaction phân tán thay 2PC. ⚠️ Nhiều team *chỉ* dùng strategic DDD (bounded context = service boundary) mà bỏ tactical cho service CRUD mỏng.

### ⑤ Code thực hành

```typescript
// aggregate.ts — Aggregate Root bảo vệ invariant, tham chiếu bằng ID

class OrderLine {                                   // entity trong aggregate (không lộ ra ngoài root)
  constructor(readonly sku: string, readonly qty: number, readonly price: number) {}
  get subtotal() { return this.qty * this.price; }
}

class Order {                                        // AGGREGATE ROOT — cửa duy nhất
  private lines: OrderLine[] = [];
  constructor(
    readonly id: string,
    private readonly customerId: string,             // tham chiếu aggregate khác bằng ID, KHÔNG nhúng Customer
    private readonly creditLimit: number,
  ) {}

  addLine(line: OrderLine) {                          // mọi thay đổi qua root
    const newTotal = this.total + line.subtotal;
    if (newTotal > this.creditLimit) throw new Error('Exceeds credit limit'); // INVARIANT
    this.lines.push(line);
  }
  get total() { return this.lines.reduce((s, l) => s + l.subtotal, 0); }
}

// Repository nói ngôn ngữ domain (trả domain object, ẩn ORM)
interface OrderRepository {
  findActiveOrdersFor(customerId: string): Promise<Order[]>;   // theo nhu cầu domain, không theo DB
  save(order: Order): Promise<void>;
}
// "Gửi email sau đặt đơn" => Application Service gọi port NotificationSender, KHÔNG nằm trong Order.
```

### ⑥ Keywords
- 🧠 **HIỂU:** aggregate/root, invariant, aggregate≈transaction boundary, repository (ngôn ngữ domain), domain vs app service, domain event, strategic vs tactical DDD.
- 📌 **THUỘC:** "tham chiếu aggregate khác bằng ID"; "một tx ~ một aggregate"; "email = app service".
- 🛠️ **LÀM ĐƯỢC:** dựng aggregate root bảo vệ invariant; viết repo interface theo domain; đặt side-effect đúng tầng.

### ⑦ Mental model
> **Aggregate = pháo đài có một cổng (root) bảo vệ invariant; ra ngoài thì gọi tên (ID), không khiêng cả lâu đài. Một transaction một pháo đài; muốn nhiều thì gửi thư (domain event). Strategic DDD luôn dùng; tactical chỉ khi domain đủ giàu.**

### ⑧ Phỏng vấn & thách đố
1. *(khó)* Aggregate/Aggregate Root là gì? Vì sao tham chiếu aggregate khác bằng ID? → consistency boundary; root là cửa; giữ boundary/eventual consistency.
2. *(khó)* Invariant là gì? Vì sao root bảo vệ? → rule luôn đúng; mọi thay đổi qua root.
3. *(khó)* Aggregate boundary ≈ transaction boundary — vì sao? Đổi nhiều aggregate thì sao? → một đơn vị nhất quán; dùng event/saga.
4. *(khó)* Repository DDD khác DAO/ORM? → trả domain object, nói ngôn ngữ domain, ẩn persistence.
5. 🔥 **Thách đố (Q-DDD-010):** "Service nội bộ CRUD đơn giản — áp full DDD tactical cho chuẩn?" → *Bẫy:* overhead lớn; CRUD/ít rule/đội nhỏ → không đáng. Strategic (bounded context) thì vẫn dùng; tactical thì chọn lọc theo độ phức tạp/invariant.

### ⑨ Bài tập + tiêu chí
- **BT1:** Dựng aggregate `Cart`/`Order` bảo vệ một invariant (vd tổng ≤ limit), tham chiếu user bằng ID. **Đạt khi:** không thể vi phạm invariant từ ngoài root.
- **BT2:** Viết 1 repository interface theo nhu cầu domain (không lộ ORM). **Đạt khi:** method đặt tên domain (`findOverdueInvoices`), trả domain object.

### ⑩ Đọc thêm
- Eric Evans — *DDD*; Vaughn Vernon — *Implementing DDD* (aggregate design rules). Fowler — Saga/eventual consistency.
- 🤖 **Kiểm tra AI:** AI hay nhúng object thay vì ID, nhét side-effect (email) vào domain, để repo trả thẳng ORM entity, và đề xuất full-DDD cho CRUD. Soát: invariant có được root bảo vệ? side-effect đúng tầng? DDD có *cần* ở đây không?

---

# ⚫ BÀI 15 — Judgment Tech Lead: chống over-engineering · enforce design · ADR · evolutionary architecture · kiểm soát AI-code

> Phủ: **Q-TL-001…008**

### ① Mục tiêu & vị trí trong mạch
Tầng **meta** trên toàn bộ mục Q: biết *khi nào DÙNG / KHÔNG dùng* tất cả những thứ Bài 1–14, *enforce* nguyên tắc trong team, và *kiểm soát chất lượng thiết kế của code AI*. Đây là phần phân biệt **senior** với **Tech Lead thật**.

### ② Giảng cơ bản → nâng cao

**(a) Khi nào KHÔNG dùng một design pattern (Q-TL-001 — nguyên tắc chung).** Khi pattern thêm **indirection/abstraction** mà **chưa có biến thiên/đau thật** (YAGNI); khi **đội chưa quen** → giảm dễ đọc. Chọn pattern **để giải vấn đề cụ thể**, không vì "cho xịn". Ưu tiên **đơn giản (KISS)**.

**(b) Over- vs under-engineering (Q-TL-002).**
- *Over-engineering — dấu hiệu:* nhiều layer/**interface 1-1**/abstraction **không có implementation thứ 2**/flexibility cho tương lai **chưa đến**.
- *Under-engineering — dấu hiệu:* copy-paste, coupling chặt, **không có seam để test**.
- Tìm **điểm cân** theo **bối cảnh** (độ phức tạp/vòng đời/đội).

**(c) Dẫn dắt thảo luận thiết kế (Q-TL-003).** Dev giỏi muốn áp **DDD + hexagonal + CQRS** cho **CRUD nội bộ nhỏ**: **dẫn dắt bằng câu hỏi giá trị/chi phí**, gắn với **độ phức tạp thực + vòng đời + đội**. Không **chặn cứng** cũng không **chiều**. Có thể đề "**bắt đầu đơn giản, để seam để tiến hoá**". Ra quyết định **có lý do**.

**(d) ⚠️ Enforce nguyên tắc thiết kế không thành bottleneck review (Q-TL-004).** **Công cụ hoá** thay vì phụ thuộc con người:
- **Architecture lint/boundary** trong CI: `dependency-cruiser`, `eslint-plugin-boundaries`, ESLint `no-restricted-imports`, Sheriff, Nx boundaries (đã verify) — chặn `domain/` import `infra/` tự động.
- **ADR + template**, **design review nhẹ**, **pairing/mentoring**.
- → Biến nguyên tắc thành **guardrail tự động**, giảm phụ thuộc người gác cổng. *(verify tên/cấu hình tooling khi học.)*

**(e) ADR — Architecture Decision Record (Q-TL-005).** Tài liệu **ngắn** ghi *quyết định + bối cảnh + các lựa chọn + hệ quả*. Giúp nhớ **"vì sao"**, onboard nhanh, tránh tranh luận lặp. ADR tối thiểu gồm: **Context · Decision · Alternatives · Consequences/Status**.

**(f) Evolutionary architecture / design for change (Q-TL-006).** Thiết kế dễ đổi **mà không đoán trước mọi tương lai** (chống **speculative generality**): tạo **seam/abstraction tại điểm *thực sự* biến động**, **để hở** chỗ chưa rõ thay vì khoá sớm. **YAGNI + refactor liên tục** khi biến thiên xuất hiện. Phân biệt "**để dễ đổi**" (seam tại điểm đau thật) với "**đoán mò tương lai**" (abstraction cho nhu cầu chưa tồn tại).

**(g) ⚠️🤖 Kiểm soát chất lượng *thiết kế* của AI-code (Q-TL-007).** AI (Copilot/Cursor) sinh code **"chạy được" nhưng vi phạm thiết kế** (nhét logic vào controller, import DB vào domain, lặp pattern sai). TL kiểm soát:
- AI lo **cú pháp**; **người chỉ huy & kiểm tra thiết kế**.
- **Guardrail:** architecture lint/boundary test, **PR checklist**, **review tập trung vào boundary/coupling** (không sa vào style vặt), conventions trong **prompt/repo** (cho AI biết quy ước kiến trúc).
- Câu thần chú: *"hiểu để chỉ huy, không thuộc để gõ"*. *(R-trend; verify công cụ.)*

**(h) ⚠️ Principle vs Tool/Pattern/Framework (Q-TL-008) — vì sao TL lý luận ở tầng nguyên tắc.** Pattern/framework là **phương tiện**, **đổi theo thời gian**; **nguyên tắc** (SOLID, coupling/cohesion) **ổn định** và là **cơ sở "vì sao"** chọn phương tiện. TL phải **biện minh quyết định bằng trade-off** (coupling/cohesion/đổi-được), **không** "vì framework bảo thế". Khi phỏng vấn: nhận ra ứng viên **thuộc-pattern** (đọc tên vanh vách) vs **hiểu-nguyên-tắc** (giải thích *vì sao* và *khi nào không*).

### ③ ⚠️ Cũ → Mới

| Cũ / phản xạ | Tech Lead nên | Vì sao |
|---|---|---|
| Áp pattern "cho xịn"/cho đủ bộ | Pattern *để giải vấn đề cụ thể*; mặc định KISS/YAGNI | Tránh over-engineering |
| Enforce thiết kế bằng review thủ công | **Architecture lint/boundary test trong CI** | Guardrail tự động, không bottleneck |
| Quyết định kiến trúc "trong đầu", không ghi | **ADR** (context/decision/alternatives/consequences) | Nhớ vì sao, tránh tranh luận lặp |
| Abstraction sẵn cho mọi tương lai | Seam tại điểm *biến động thật*; để hở phần chưa rõ | Chống speculative generality |
| Tin code AI vì "chạy được" | Review boundary/coupling + guardrail lint | AI đúng cú pháp ≠ đúng thiết kế |

### ④ Áp dụng thực tế + So sánh bigtech
- **Enforce:** thêm `depcruise --validate` + rule boundaries vào GitHub Actions; PR vi phạm `domain → infra` bị **fail CI** tự động.
- **ADR:** thư mục `docs/adr/0001-use-hexagonal.md` theo template Michael Nygard — chuẩn phổ biến nhiều công ty.
- **Bigtech/ecosystem:** Google readability + large-scale lint; Spotify/ThoughtWorks phổ biến ADR; "fitness functions" (*Building Evolutionary Architecture* — ThoughtWorks) = test tự động cho thuộc tính kiến trúc. AI-code guardrails đang là chủ đề nóng 2025–2026 (verify công cụ).

### ⑤ Code thực hành (enforce kiến trúc trong CI)

```javascript
// .dependency-cruiser.cjs — chặn domain import infra (đã verify công cụ tồn tại)
module.exports = {
  forbidden: [
    {
      name: 'domain-no-infra',
      comment: 'Domain layer KHÔNG được phụ thuộc infra (Dependency Rule)',
      severity: 'error',
      from: { path: '^src/domain' },
      to:   { path: '^src/(infra|adapters)' },   // vi phạm -> fail CI
    },
    { name: 'no-circular', severity: 'error', from: {}, to: { circular: true } },
  ],
};
// CI:  npx depcruise --config .dependency-cruiser.cjs src
```
```markdown
<!-- ADR tối thiểu: docs/adr/0007-hexagonal-for-order-service.md -->
# 7. Dùng Hexagonal cho Order Service
## Status: Accepted
## Context: domain đặt hàng nhiều invariant, dự kiến thay datastore.
## Decision: tách domain/port/adapter; enforce bằng dependency-cruiser.
## Alternatives: layered đơn giản (loại vì domain phức tạp/biến động).
## Consequences: thêm mapping/interface; bù lại test/thay hạ tầng dễ.
```

### ⑥ Keywords
- 🧠 **HIỂU:** khi nào KHÔNG dùng pattern, over/under-engineering, dẫn dắt thảo luận, enforce bằng tooling, ADR, evolutionary architecture/speculative generality, AI-code guardrails, principle vs tool.
- 📌 **THUỘC:** 4 phần ADR (Context/Decision/Alternatives/Consequences); dấu hiệu over-engineering (interface 1-1, abstraction không impl thứ 2).
- 🛠️ **LÀM ĐƯỢC:** cấu hình một rule boundary lint; viết một ADR; phản hồi "áp full DDD cho CRUD" mang tính dẫn dắt.

### ⑦ Mental model
> **Nguyên tắc là la bàn (ổn định), pattern/framework là phương tiện (thay được). TL biện minh bằng trade-off, enforce bằng guardrail tự động chứ không bằng người gác cổng, và ghi lại "vì sao" (ADR). Với AI: hiểu để chỉ huy & kiểm tra, không thuộc để gõ.**

### ⑧ Phỏng vấn & thách đố
1. *(rất khó)* Khi nào KHÔNG nên dùng một design pattern (nguyên tắc chung)? → khi thêm indirection mà chưa có biến thiên/đau thật; đội chưa quen; KISS/YAGNI.
2. *(khó)* Dấu hiệu over-engineering cụ thể? → interface 1-1, abstraction không impl thứ 2, flexibility cho tương lai chưa đến.
3. *(khó)* ADR là gì, gồm gì? → context/decision/alternatives/consequences; nhớ "vì sao".
4. *(rất khó)* Enforce dependency rule trong team đông không thành bottleneck — cách? → công cụ hoá (boundary lint trong CI) + ADR + mentoring.
5. *(rất khó)* Vì sao TL phải lý luận ở tầng *nguyên tắc* thay vì *framework*? → nguyên tắc ổn định và là cơ sở "vì sao"; framework đổi theo thời gian.
6. 🔥 **Thách đố (Q-TL-007):** "AI sinh code chạy được — merge luôn cho nhanh?" → *Bẫy:* "chạy được" ≠ đúng thiết kế; AI hay nhét logic vào controller/import DB vào domain. Cần guardrail lint + review boundary/coupling. *Người chỉ huy & kiểm tra, AI lo cú pháp.*

### ⑨ Bài tập + tiêu chí
- **BT1:** Thêm 1 rule `dependency-cruiser`/`eslint-plugin-boundaries` chặn `domain → infra` vào repo bạn + chạy trong CI. **Đạt khi:** một PR cố tình vi phạm bị fail tự động.
- **BT2:** Viết 1 ADR cho một quyết định kiến trúc gần đây (đủ 4 phần). **Đạt khi:** người mới đọc hiểu *vì sao* chọn, đã cân nhắc gì.

### ⑩ Đọc thêm
- Michael Nygard — "Documenting Architecture Decisions" (ADR). Neal Ford et al. — *Building Evolutionary Architecture* (fitness functions). R. C. Martin — SOLID/Clean.
- Công cụ (đã verify): `dependency-cruiser`, `eslint-plugin-boundaries`, Sheriff, Nx module boundaries.
- 🤖 **Hiểu để chỉ huy:** cả mục Q hội tụ ở đây — AI viết *cú pháp* pattern rất nhanh, nhưng *chọn pattern nào, khi nào KHÔNG dùng, và giữ boundary/coupling* là việc của bạn. Cặp ví dụ kinh điển: AI để `temperature=0.7` cho phân loại (cú pháp đúng, hành vi sai) · AI chọn fine-tune cho dữ liệu hay đổi đáng lẽ dùng RAG (hiểu định hình yêu cầu). Tương tự ở Q: AI import ORM vào domain (cú pháp đúng, kiến trúc sai) — bạn phải bắt được.

---

# ✅ TỔNG KẾT & THEO DÕI (WORKFLOW 2 — Bước D/E)

## Bản đồ hội tụ một trang
```
Coupling↓ + Cohesion↑  (Bài 1)  ──── la bàn cho tất cả
   │
   ├─ Nguyên tắc OOP nền (Bài 2–3): composition>inheritance · Demeter · DRY/KISS/YAGNI
   ├─ SOLID (Bài 4–6): SRP/ISP↑cohesion · OCP/DIP↓coupling · DIP≠DI · SOLID có thể over-engineer
   ├─ GoF (Bài 7–9): Creational(tạo) · Structural(ghép) · Behavioral(giao tiếp) — nhiều cái co lại nhờ function/module trong TS
   ├─ Smell/Refactoring (Bài 10): phát hiện vi phạm + sửa an toàn (giữ behavior, có test)
   ├─ Kiến trúc (Bài 11–12): Dependency Rule · port/adapter · anemic/CQRS · refactor incremental
   ├─ DDD (Bài 13–14): bounded context + ubiquitous language (strategic, luôn dùng) · aggregate/invariant/repo/event (tactical, chọn lọc)
   └─ Judgment TL (Bài 15): khi nào KHÔNG dùng · enforce bằng tooling · ADR · evolutionary · kiểm soát AI-code
```

## TRẠNG THÁI HỌC TẬP — mục Q (cập nhật khi bạn phỏng vấn ở Bước D)
```
TIẾN ĐỘ HỌC — Q (Software Design & Patterns) — [ngày bạn học]
- Tổng câu trong bộ đề: 90
- Đã sinh giáo trình: 15 bài (phủ 100% 90 câu)  ✅ Bước B + C xong
- Đã ĐẠT (qua phỏng vấn Bước D): [chưa bắt đầu — cập nhật dần]
- Đang học: [bài ...]
- Điểm yếu cần ôn: [...]
- Bài tiếp theo đề xuất: Bài 1 → ... → Bài 15 (theo thứ tự dependency)
```

## Bước D — Phỏng vấn theo đợt (chấm LIVE)
Khi bạn sẵn sàng, nói **"phỏng vấn Bài X"** hoặc **"test tôi mục Q"** → tôi ra **10–15 câu trộn dễ→khó** (bám đúng dòng *"dò cái gì"* trong `QB_Q_designpatterns.md`), **hỏi xong DỪNG** chờ bạn trả lời, rồi mới dựng đáp án chuẩn + chấm ĐẠT/CHƯA + vá lỗ hổng. (Recall trước, đáp án sau.)

## Bước E — Cổng hoàn thành + ôn ngắt quãng
- "Xong mục Q" = ĐẠT toàn bộ 90 câu qua phỏng vấn.
- **Lịch spaced repetition đề xuất:** hôm nay (Bài 1–6) → mai (Bài 7–10) → 3 ngày (Bài 11–14) → 1 tuần (toàn bộ + Bài 15). Mỗi đợt sau tôi chèn 1–2 câu CŨ đã đạt để chống học vẹt.
- Câu CHƯA đạt → vá đúng chỗ yếu, hỏi lại biến thể; chưa tính xong tới khi đạt.

> **Câu thần chú:** *"Tôi không nhớ để gõ, tôi hiểu để chỉ huy — và tra cứu phần còn lại."*

---
*Hết tài liệu mục Q. Nguồn ổn định: GoF 1994 · SOLID (R.C. Martin) · Clean Architecture 2017 · DDD (Evans 2003) · Refactoring (Fowler). Tooling đã verify 6/2026: TС39 Stage-3 vs legacy decorators (Nest dùng legacy), dependency-cruiser/eslint-plugin-boundaries. Verify lại version cú pháp khi thực hành.*
