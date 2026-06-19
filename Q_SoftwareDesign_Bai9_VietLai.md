# 🎯 BÀI 9 — **GoF Behavioral**: **Strategy** · **Observer** · **Command** · **Template Method** · **State** · **Chain of Responsibility**

**Phủ:** Q-GOFB-001 → Q-GOFB-002 → Q-GOFB-003 → Q-GOFB-004 → Q-GOFB-005 → Q-GOFB-006 → Q-GOFB-007 → Q-GOFB-008 → Q-GOFB-009 → Q-GOFB-010

> 💡 **Mental model mở đầu:**
> **Behavioral pattern** = cách object **giao tiếp & phân chia trách nhiệm khi CHẠY** (runtime). Đây là nhóm bạn **gặp nhiều nhất ở backend**:
> - **middleware** = **Chain of Responsibility**
> - **event** = **Observer**
> - **state machine duyệt đơn** = **State**
> Có **2 cặp dễ nhầm** mà phỏng vấn viên rất thích hỏi: **State vs Strategy** và **Observer vs Pub/Sub**.

---

## ① Mục tiêu & vị trí trong mạch

**Behavioral** = cách object **giao tiếp & phân chia trách nhiệm khi chạy**. Đây là nhóm bạn **gặp nhiều nhất ở backend**.

```
   Bài 7 Creational → Bài 8 Structural → Bài 9 BEHAVIORAL (bài này)

   2 CẶP DỄ NHẦM (phỏng vấn hay hỏi):
        • State   vs Strategy
        • Observer vs Pub/Sub
                    │
        nối Bài 3 (Strategy bằng function) + mục G/H (EventEmitter, middleware)
```

> 🎯 **Chốt vị trí:** nhiều thứ backend bạn đã dùng *chính là* Behavioral pattern mà có thể chưa gọi tên: **middleware = CoR**, **event = Observer**, **state machine = State**. Bài này nối **Bài 3** (Strategy bằng function) và **mục G/H** (EventEmitter, middleware).

---

## ② Giảng cơ bản → nâng cao

### (a) Strategy (Q-GOFB-001, 002)

> **Ẩn dụ:** chọn *tuyến đường* đi làm — đường nhanh, đường tránh kẹt, đường đẹp. Cùng mục tiêu (tới chỗ làm), bạn **hoán đổi thuật toán đường đi** tuỳ hôm.

**Strategy** = đóng gói các **thuật toán hoán đổi được** sau **cùng một interface**, **chọn runtime**.

> 🔎 **Trong TS, truyền một hàm vào đã là Strategy bản gọn** (Bài 3).

**So `if/else` vs Strategy** (vd tính phí ship nhiều hãng):

| Tiêu chí | **if/else** | **Strategy** |
|---|---|---|
| Thêm hãng mới | **sửa** hàm cũ | **thêm class/hàm**, không sửa cũ (**OCP**) |
| Test | test cả khối if | **test từng strategy riêng** |
| Khi ít ca (2-3) | gọn, đủ | **thừa** class/indirection |

> 🎯 **Chốt:** **cân theo số biến thể** — nhiều biến thể & hay thêm → Strategy; chỉ 2-3 ca đơn giản → if/else đủ (đừng over-engineer).

---

### (b) Observer (Q-GOFB-003)

> **Ẩn dụ:** đăng ký nhận thông báo kênh YouTube. Kênh (**subject**) ra video mới → **notify** tất cả người đã **subscribe** (**observer**). Ai cũng hủy/đăng ký bất cứ lúc nào.

**Observer** = quan hệ **một-nhiều**: **subject** đổi state → **notify** các **observer** đã **subscribe**; observer **thêm/bớt runtime**.

> 💡 **Ví dụ:** UI update, **cache invalidation**, **domain event**.

---

### (c) ⚠️ Observer vs Pub/Sub (Q-GOFB-004) — khác cốt lõi ở coupling

> 🚨 **Bẫy phỏng vấn:** hai cái *nghe giống* nhau nhưng khác ở **mức coupling**.

```
OBSERVER (coupling vừa)              PUB/SUB (decouple mạnh)
   subject ──gọi trực tiếp──▶ observer    publisher ──▶ [BROKER] ──▶ subscriber
   (subject BIẾT observer)                (publisher KHÔNG biết subscriber là ai)
   cùng tiến trình                        thường async / cross-process
```

| Tiêu chí | **Observer** | **Pub/Sub** |
|---|---|---|
| subject/publisher có biết bên nhận? | **biết & gọi trực tiếp** | **không biết** (qua trung gian) |
| Trung gian | không | **broker / event channel** |
| Coupling | **vừa** | **decouple mạnh** |
| Tiến trình | thường cùng tiến trình | thường **async / cross-process** |

> 🔎 **Nối thực tế:** **EventEmitter** (in-proc) ≈ **Observer**; **message broker** (**Kafka/RabbitMQ**) ≈ **Pub/Sub**. (cơ chế cụ thể ở **G/H**.)

---

### (d) ⚠️ EventEmitter lạm dụng (Q-GOFB-005)

Node **EventEmitter** là **hiện thân của Observer**. Tiện, nhưng **rủi ro khi dùng cho luồng nghiệp vụ chính**:

| Rủi ro | Hậu quả |
|---|---|
| **control flow ẩn** | khó lần vết — không biết ai nghe, chạy theo thứ tự nào |
| **error event chưa handle** | → **crash** cả tiến trình |
| **quên `removeListener`** | → **memory leak** |
| **thứ tự / đồng bộ khó suy luận** | bug khó tái hiện |

> 🎯 **Nguyên tắc:** với **nghiệp vụ cần rõ ràng**, dùng **call trực tiếp / queue** thay vì event. Event hợp cho *thông báo phụ*, không cho *luồng nghiệp vụ chính*.

---

### (e) Command (Q-GOFB-006)

> **Ẩn dụ:** **phiếu gọi món** ở nhà hàng. Yêu cầu "1 phở tái" được **đóng gói thành tờ phiếu** — có thể chuyển cho bếp, xếp hàng, lưu lại, thậm chí hủy.

**Command** = đóng gói **một yêu cầu thành object** (có thể **truyền/lưu/hoãn**). Mở ra: **undo/redo**, **queue/lập lịch**, **logging/replay/audit**.

> 💡 **Ví dụ:** **job queue** (mỗi **job** là một Command).

---

### (f) Template Method (Q-GOFB-007)

> **Ẩn dụ:** công thức nấu ăn có **khung cố định** (sơ chế → nấu → trình bày), nhưng **vài bước** để người nấu **tự quyết** (nêm mặn/nhạt).

**Template Method** = lớp cha định **"khung" thuật toán**, để lớp con **override vài bước (hook)**. Dựa trên **inheritance** → **coupling base-class chặt**.

> 🎯 Khi cần linh hoạt hơn → cân nhắc **Strategy** (**composition** thay **kế thừa** — Bài 2).

---

### (g) ⚠️ State vs Strategy (Q-GOFB-008) — cấu trúc class giống, ý đồ khác

> 🚨 **Bẫy phỏng vấn kinh điển:** nhìn code **State** và **Strategy** *gần như giống hệt* (đều là object cầm một interface và uỷ quyền). Khác ở **ai điều khiển** và **ý đồ**.

| Tiêu chí | **State** | **Strategy** |
|---|---|---|
| Ai điều khiển việc chuyển? | **object TỰ chuyển** trạng thái | **client/bên ngoài** chọn thuật toán |
| Các "biến thể" có biết nhau? | **các state biết transition sang nhau** | **độc lập**, không tự đổi nhau |
| Ý đồ | **quản trạng thái** (hành vi đổi theo state) | **hoán thuật toán** |

> 💡 **Cách nhớ:** State = "tôi đang ở pha nào và tự đổi pha"; Strategy = "ai đó đưa tôi cách làm để dùng".

---

### (h) Chain of Responsibility (Q-GOFB-009)

> **Ẩn dụ:** quy trình duyệt hồ sơ qua nhiều bàn — bàn 1 xử lý hoặc đẩy sang bàn 2, bàn 2 xử lý hoặc đẩy tiếp...

**Chain of Responsibility (CoR)** = **chuỗi handler**; mỗi handler **xử lý hoặc chuyển tiếp**.

```
   req ──▶ mw1 ──next──▶ mw2 ──next──▶ handler
        (auth)       (rate-limit)    (xử lý chính)
   mỗi khâu: làm việc của mình rồi gọi next() để chuyển tiếp
```

> 🔎 **Middleware Express/Nest CHÍNH LÀ CoR.** Lợi: **thêm/bớt/đổi thứ tự** khâu xử lý **linh hoạt**.

---

### (i) State machine cho workflow (Q-GOFB-010)

> **Ví dụ trực giác:** đơn hàng đi qua các pha *tạo → chờ duyệt → duyệt/từ chối → đóng*. Mỗi pha **hành vi khác nhau** và **chỉ được chuyển sang một số pha hợp lệ**.

**State machine** mô hình workflow bằng cách **gom hành vi + transition hợp lệ theo state** → tránh **`if(status===…)` rải khắp**; **chặn transition sai**, dễ thêm state.

```
   created ──▶ pending ──┬──▶ approved ──▶ closed
                         └──▶ rejected ──▶ closed
   ❌ created ──▶ closed   (KHÔNG có trong bảng → ném lỗi "Illegal transition")
```

> 🚨 **Nhưng đừng over-engineer:** khi **đơn giản** (2 trạng thái tuyến tính), **enum + guard** cũng đủ. State machine đáng khi nhiều state + nhiều transition cần kiểm soát.

---

## ③ ⚠️ Phân biệt / Cũ → Mới

| Hay nhầm / cũ | Đúng | Vì sao |
|---|---|---|
| **State = Strategy** (vì class giống) | **State tự-chuyển + biết transition; Strategy do ngoài chọn, độc lập** | Khác **ai điều khiển + ý đồ** |
| **Observer = Pub/Sub** | **Observer gọi trực tiếp (in-proc); Pub/Sub qua broker (decouple mạnh)** | Khác mức **coupling/async** |
| **Dùng EventEmitter cho mọi luồng nghiệp vụ** | **Event cho thông báo phụ; nghiệp vụ chính → call/queue** | **Control flow ẩn, leak, error nuốt** |
| **`if(status===…)` rải khắp** cho workflow | **State machine** (transition tường minh) | **Chặn chuyển sai, dễ mở rộng** |

---

## ④ Áp dụng thực tế + So sánh bigtech

| Pattern | Hiện thân thực tế |
|---|---|
| **CoR** | mọi framework web (**Express/Koa/Nest**) = **pipeline middleware**: Auth → rate-limit → validate → handler |
| **Command** | **BullMQ/Sidekiq job** = Command (**serialize, retry, schedule**) |
| **State machine** | **order/subscription lifecycle**; thư viện **XState (TS)** mô hình state machine tường minh |

> 🔎 **Insight sâu (bigtech):** **event-driven** (**Observer/PubSub**) là **xương sống của microservices**; **Stripe webhooks** = **pub/sub cross-system** — Stripe (publisher) bắn event, hệ thống của bạn (subscriber) nghe, Stripe **không cần biết** bạn là ai. Đó là decouple mạnh ở quy mô liên-hệ-thống.

---

## ⑤ Code thực hành

```typescript
// behavioral.ts — TypeScript 5.x

// --- Strategy bằng FUNCTION (idiom TS) ---
type ShipFee = (weightKg: number) => number;
const carriers: Record<string, ShipFee> = {
  ghn:  (w) => w * 12000,
  ghtk: (w) => w * 10000 + 5000,       // thêm hãng = thêm ENTRY (OCP), KHÔNG sửa quote()
};
const quote = (carrier: string, w: number) =>
  (carriers[carrier] ?? (() => { throw new Error('?'); }))(w);

// --- Chain of Responsibility (middleware-style) ---
type Ctx = { user?: string; allowed?: boolean };
type Handler = (ctx: Ctx, next: () => void) => void;
const auth:   Handler = (ctx, next) => { if (!ctx.user) throw new Error('401'); next(); };  // xử lý rồi next()
const verify: Handler = (ctx, next) => { ctx.allowed = ctx.user === 'admin'; next(); };
function run(handlers: Handler[], ctx: Ctx) {
  let i = 0;
  const next = () => { if (i < handlers.length) handlers[i++](ctx, next); };  // chuyển tiếp dây chuyền
  next();                              // thêm/bớt/đổi THỨ TỰ handler tự do
}

// --- State machine cho duyệt đơn (transition TƯỜNG MINH) ---
type OrderState = 'created' | 'pending' | 'approved' | 'rejected' | 'closed';
const transitions: Record<OrderState, OrderState[]> = {
  created:  ['pending'],
  pending:  ['approved', 'rejected'],
  approved: ['closed'],
  rejected: ['closed'],
  closed:   [],                        // trạng thái cuối, không chuyển đi đâu nữa
};
function transition(from: OrderState, to: OrderState): OrderState {
  if (!transitions[from].includes(to))                                       // tra BẢNG transition hợp lệ
    throw new Error(`Illegal ${from} -> ${to}`);                             // ❌ CHẶN chuyển sai ngay
  return to;
}
```

> 💡 **Đọc kỹ state machine:** mọi transition hợp lệ nằm gọn trong **một bảng** `transitions`. Thêm state mới = **sửa bảng**, không phải săn lùng `if(status===…)` rải khắp codebase. Và chuyển sai (vd `created → closed`) bị **ném lỗi ngay**, không lọt vào trạng thái vô nghĩa.

---

## ⑥ Keywords cần nhớ

> 🧠 **Cần HIỂU:**
> - **Strategy** (hoán thuật toán/function)
> - **Observer vs Pub/Sub** (coupling)
> - **EventEmitter risks**
> - **Command** (undo/queue)
> - **Template Method** (inheritance)
> - **State vs Strategy**
> - **CoR = middleware**
> - **state machine cho workflow**

> 📌 **Cần THUỘC:**
> - Điểm khác **State/Strategy** (**ai chuyển**).
> - Điểm khác **Observer/PubSub** (**broker?**).

> 🛠️ **Cần LÀM ĐƯỢC:**
> - **Strategy** bằng map/function.
> - **CoR pipeline**.
> - **State machine** với **transition guard**.

---

## ⑦ Mental model

> 🎯 **Khẩu quyết (một câu mỗi pattern):**
> **Strategy** = hoán thuật toán (ngoài chọn) · **State** = máy trạng thái (tự chuyển) · **Observer/PubSub** = thông báo (trực tiếp vs qua broker) · **Command** = đóng gói hành động · **CoR** = dây chuyền middleware.

---

## ⑧ Câu hỏi phỏng vấn & thách đố

**(TB)** *Strategy giải quyết gì? Truyền hàm vào đã là Strategy chưa?*
→ **hoán thuật toán**; **rồi**.

**(khó)** *Observer vs Pub/Sub khác coupling ở đâu?*
→ **gọi trực tiếp** vs **qua broker**.

**(khó)** *State khác Strategy thế nào dù class giống?*
→ **ai điều khiển chuyển + ý đồ** (tự chuyển/quản state vs ngoài chọn/hoán thuật toán).

**(khó)** *CoR liên hệ middleware Express/Nest ra sao?*
→ **pipeline handler chuyển tiếp** (`next()`).

> 🧩 **🔥 Thách đố (Q-GOFB-010):** *"Workflow 5 bước — cứ `if(status===...)` cho nhanh?"*
>
> **Bẫy:** **`if` rải khắp** khó **chặn transition sai** & khó mở rộng; **state machine** gom **transition hợp lệ** vào một chỗ. **Nhưng** nếu chỉ **2 trạng thái tuyến tính** thì **enum + guard** đủ — **đừng over-engineer**. Trả lời cực đoan (luôn if / luôn state machine) đều rơi bẫy — cân theo độ phức tạp.

---

## ⑨ Bài tập + tiêu chí tự chấm

**BT1.** Mô hình **lifecycle một entity** (vd **subscription**) bằng **state machine** với **bảng transition**.
→ ✅ **Đạt khi:** mọi chuyển sai **bị ném lỗi**, thêm state mới **chỉ sửa bảng**.

> 💡 *Gợi ý:* y hệt ví dụ `transitions` ở mục ⑤ — liệt kê các state, rồi với mỗi state ghi danh sách state hợp lệ kế tiếp.

**BT2.** Viết **pipeline CoR** cho request (**auth → validate → handle**).
→ ✅ **Đạt khi:** thêm 1 khâu = **thêm 1 handler**, **không sửa các handler khác**.

---

## ⑩ Đọc thêm

- **Design Patterns (GoF)** — **Behavioral**. **XState docs** (state machine TS).
- **Node.js docs** — **events/EventEmitter** (**memory leak warning**, **error event**).

---

## 🤖 Hiểu để chỉ huy (Kiểm tra AI)

> 🚨 **AI rất hay:**
> 1. **Rải `if(status===...)`** thay vì **state machine** → khó chặn transition sai.
> 2. **Lạm dụng EventEmitter** cho luồng nghiệp vụ → **control flow ẩn**.
>
> 🎯 **Hai câu soát mỗi lần nhận code từ AI:**
> - *"**Transition** có được **kiểm soát** không, hay đang `if(status===...)` rải khắp?"* → rải → ép gom vào **state machine** (transition tường minh).
> - *"**Event** này có **nuốt error / leak listener** không, và đây là *nghiệp vụ chính* hay *thông báo phụ*?"* → nghiệp vụ chính qua event → ép đổi sang **call/queue**.
>
> AI thích event vì "linh hoạt", thích `if` vì "nhanh", nhưng cả hai làm luồng nghiệp vụ **mờ đi**. Vai trò chỉ huy của bạn: bắt **transition tường minh** và **không để nghiệp vụ chính chạy ngầm qua event**.
