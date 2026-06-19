# 🎯 BÀI 5 — **SOLID** (2): **Liskov Substitution** & **Interface Segregation**

**Phủ:** Q-SOLID-006 → Q-SOLID-007 → Q-SOLID-008

> 💡 **Mental model mở đầu:**
> - **LSP** = *"bài kiểm tra inheritance có hợp lệ không"* — nó nối thẳng vào **Bài 2** (composition > inheritance): nếu kế thừa **rớt** bài kiểm tra LSP, hãy đổi sang **composition**.
> - **ISP** = *"SRP cho interface"* — kéo **cohesion** xuống tận mức **contract** (bản hợp đồng).
> Hai chữ này hay bị bỏ qua, nhưng chính chúng **lộ rõ ai thật sự hiểu OOP**.

---

## ① Mục tiêu & vị trí trong mạch

```
   Bài 2: composition > inheritance  ──────┐
                                           ▼
   Bài 5:  LSP = "TEST kế thừa có hợp lệ không?"
           rớt test → quay về composition (Bài 2)

           ISP = "SRP cho interface" (Bài 4 SRP áp lên contract)
                                           │
                                           ▼
              tiền đề CQRS (Bài 12)
```

> 🎯 **Chốt vị trí:** **LSP** không phải "thêm một luật kế thừa", mà là **cây thước đo** để biết một kế thừa có *thật sự đúng* hay chỉ "đúng cú pháp". **ISP** là **SRP** (Bài 4) áp lên *interface* thay vì *class*. Hai chữ này hay bị bỏ qua, nhưng phỏng vấn viên dùng chúng để phân biệt người *học thuộc SOLID* với người *hiểu SOLID*.

---

## ② Giảng cơ bản → nâng cao

### (a) LSP — Liskov Substitution Principle

> **Ẩn dụ — diễn viên đóng thế.** Một **diễn viên đóng thế** (subtype) phải thay được diễn viên chính (supertype) trong cảnh quay mà **khán giả không nhận ra sự khác biệt**. Nếu người đóng thế đột nhiên *không làm được* cảnh mà vai diễn yêu cầu → hỏng phim.

> 🎯 **Phát biểu:** **subtype** (kiểu con) phải **thay thế được supertype** (kiểu cha) mà **không phá hành vi / contract** mà **client** trông đợi.

**Ràng buộc hình thức (đáng nhớ):**

| Yếu tố | Quy tắc ở subtype | Trực giác |
|---|---|---|
| **precondition** (tiền điều kiện) | **không được mạnh hơn** | con không được *đòi hỏi khắt khe hơn* cha (vd cha nhận số bất kỳ, con chỉ nhận số dương → vỡ) |
| **postcondition** (hậu điều kiện) | **không được yếu hơn** | con không được *hứa ít hơn* cha (vd cha đảm bảo trả list đã sort, con trả list lộn xộn → vỡ) |
| **invariant** của base | **phải giữ** | điều luôn-đúng của cha, con không được phá |

> 💡 **Vì sao 3 ràng buộc này?** Vì **client** viết code dựa trên *lời hứa của cha*. Nếu con *đòi nhiều hơn* hoặc *hứa ít hơn*, thì code client (vốn tin theo cha) sẽ vỡ khi gặp con. LSP = "con phải giữ trọn lời hứa của cha, không thêm điều kiện, không bớt đảm bảo".

#### ⚠️ Hai ví dụ kinh điển vi phạm

**① `Square extends Rectangle`:**

```
Contract của Rectangle mà client TIN: "đặt width KHÔNG ảnh hưởng height"
                    │
   Square override: set width(w) { this._w = this._h = w; }  // buộc đổi cả height!
                    │
   Client làm:  rect.setWidth(5); rect.setHeight(4);
                expect(rect.area()).toBe(20);
                    │
                    ▼  nếu rect thực ra là Square → height bị ép = 5
❌ area() = 25 thay vì 20 → contract vỡ, client sai
```

> 🔎 *"Hình vuông LÀ hình chữ nhật"* về **toán học** đúng, nhưng **behaves-not-like** (cư xử không giống) trong **code** — vì hành vi setter khác nhau.

**② `Ostrich extends Bird` với `fly()`:**

```
Bird có fly(). Client viết:  birds.forEach(b => b.fly());
                    │
   Ostrich (đà điểu) override fly() { throw new Error('cannot fly'); }
                    │
                    ▼
❌ Gặp con đà điểu → throw → client vỡ. Kế thừa "đúng cú pháp" nhưng PHÁ LSP.
```

> 🎯 **Bài học (Q-SOLID-007):** **quan hệ phân loại đời thực ("is-a") ≠ quan hệ thay thế hành vi ("behaves-like-a").** **LSP là TEST** cho việc **inheritance có hợp lệ không**. Nếu **"is-a" KHÔNG kéo theo "behaves-like-a"** → **ưu tiên composition** (Bài 2). Đà điểu *là* chim về sinh học, nhưng *không cư xử như* một `Bird-biết-bay` trong code.

---

### (b) ISP — Interface Segregation Principle

> **Ẩn dụ:** đừng bắt một người **ký hợp đồng** cho cả những việc họ **không bao giờ làm**. Bắt robot ký vào dòng "phải ăn trưa" là vô lý.

> 🎯 **Phát biểu:** **client không nên bị buộc phụ thuộc vào method nó KHÔNG dùng.** **"Fat interface"** (interface phình to, nhồi nhét) là một **mùi** (smell).

**Ví dụ (Q-SOLID-008):**

```
❌ interface Worker { work(); eat(); }
        │
   class RobotWorker implements Worker {
     work() { ... }
     eat()  { /* rỗng?? robot đâu có ăn */ }   ← bị ÉP implement method vô nghĩa
   }
        │
        ▼  sửa:
✅ interface Workable { work(); }
   interface Eatable  { eat();  }
   → client nào cần gì IMPLEMENT nấy. Robot chỉ Workable, người thì cả hai.
```

> 🔎 **ISP là SRP áp cho interface:** **gom đúng nhóm hành vi gắn kết, không nhồi.** Một interface cũng nên có *một lý do tồn tại* — đúng tinh thần SRP (Bài 4), chỉ khác là áp lên *contract* thay vì *class*.

> 🚨 **Vì sao fat interface tệ?** Vì nó **lan coupling**: mọi client của interface "biết về" *tất cả* method, kể cả thứ chúng không dùng. Đổi một method (kể cả method robot chẳng dùng) cũng có thể buộc robot phải đụng vào. Và nó đẻ ra **method rỗng / `throw NotSupported`** — dấu hiệu thiết kế sai.

---

## ③ ⚠️ Kiến thức cũ → Mới

| Cách cũ / sai | Nên dùng nay | Vì sao |
|---|---|---|
| **Kế thừa theo "is-a" đời thường** (Square is-a Rectangle) | Kiểm bằng **LSP** (**behaves-like-a?**), không đạt thì **composition** | **"is-a" ≠ thay thế hành vi** |
| Một **"fat interface"** cho mọi client | **Tách interface nhỏ** theo nhu cầu (**ISP**) | Tránh **implement method rỗng / ném exception** |
| **Override rồi `throw NotSupported`** | Thiết kế lại phân cấp / **tách interface** | **`throw` trong override = mùi LSP** |

---

## ④ Áp dụng thực tế + So sánh bigtech

### Use case ISP: tách Repository

```
❌ FAT INTERFACE                          ✅ TÁCH THEO NHU CẦU (ISP)
   interface Repository {                   interface ReadRepository  { find(); query(); }
     create(); read(); update();            interface WriteRepository { create(); update(); delete(); }
     delete(); bulkInsert();
     stream(); cacheWarm();  ...             → service ĐỌC chỉ phụ thuộc ReadRepository
   }                                           (không dính write/bulk/stream nó chẳng dùng)
```

> 💡 Service chỉ-đọc giờ **chỉ phụ thuộc phần đọc** — đổi logic write không thể ảnh hưởng nó. Đây cũng là **tiền đề của CQRS** (Command Query Responsibility Segregation — tách đường đọc và đường ghi, **Bài 12**).

### Bigtech / ecosystem (pattern phổ biến — không tuyệt đối)

| Hệ sinh thái | Bằng chứng ISP |
|---|---|
| **Go** | Khuyến khích **interface nhỏ** — câu của **Rob Pike**: *"the bigger the interface, the weaker the abstraction"* (interface càng to, abstraction càng yếu) |
| **`io.Reader` / `io.Writer`** | mỗi cái **đúng 1 method** — ISP "cực đoan" mà cực kỳ hiệu quả & tái dùng khắp nơi |

> 🔎 **Insight sâu:** vì sao interface **nhỏ** lại mạnh? Vì interface càng nhỏ thì càng **dễ thoả mãn** → càng **nhiều kiểu** có thể implement → càng **tái dùng** rộng. `io.Reader` (1 method `Read`) khớp với file, network, buffer, gzip... mọi thứ "đọc được" — đó là sức mạnh của ISP đẩy tới cùng.

---

## ⑤ Code thực hành

```typescript
// lsp-isp.ts — TypeScript 5.x

// --- ISP: tách FAT INTERFACE ---
interface Workable { work(): void; }
interface Eatable  { eat(): void; }

class HumanWorker implements Workable, Eatable {  // người: cần cả hai
  work() {/* ... */}
  eat()  {/* ... */}
}

class RobotWorker implements Workable {           // ✅ robot chỉ Workable -> KHÔNG bị ép eat() rỗng
  work() {/* ... */}
}

// --- LSP: tránh "is-a" GIẢ ---
// ❌ class Square extends Rectangle { set width(w){ this._w = this._h = w; } }
//    -> phá contract "đặt width không đổi height" mà client của Rectangle tin tưởng

// ✅ Tách abstraction theo HÀNH VI (behaves-like-a), KHÔNG theo phân loại toán học:
interface Shape { area(): number; }

class Rectangle implements Shape {
  constructor(private w: number, private h: number) {}
  area() { return this.w * this.h; }
}

class Square implements Shape {
  constructor(private s: number) {}
  area() { return this.s * this.s; }
}
// Cả hai "LÀ Shape" theo nghĩa behaves-like-a (đều TÍNH ĐƯỢC area),
// và KHÔNG kế thừa nhau -> không có chuyện con phá contract của cha.
```

> 💡 **Đọc kỹ:** mấu chốt là `Rectangle` và `Square` **không còn quan hệ cha-con**. Chúng chỉ cùng *thoả một contract hành vi* (`Shape`: tính được `area`). Bỏ kế thừa = bỏ luôn cơ hội phá LSP.

---

## ⑥ Keywords cần nhớ

> 🧠 **Cần HIỂU:**
> - **LSP** — thay thế không phá **contract**; **precondition** không mạnh hơn / **postcondition** không yếu hơn / **invariant** giữ nguyên
> - **ISP** — **client** không phụ thuộc thứ nó không dùng
> - **"is-a" ≠ "behaves-like-a"** — phân loại đời thực không bằng thay thế hành vi

> 📌 **Cần THUỘC:**
> - Ví dụ **Square / Rectangle**, **Ostrich / Bird**.
> - Phát biểu **ISP** (client không phụ thuộc method không dùng).

> 🛠️ **Cần LÀM ĐƯỢC:**
> - Phát hiện **override `throw NotSupported`**.
> - **Tách fat interface** thành các interface nhỏ.

---

## ⑦ Mental model

> 🎯 **Khẩu quyết:**
> - **LSP:** con phải **"đóng thế"** cha mà **khán giả không nhận ra**.
> - **ISP:** đừng bắt ai **ký hợp đồng** cho việc họ **không làm**.

```
   Subtype gặp chỗ dùng supertype → có "đóng thế" trơn tru không?
        không (throw / đổi hành vi) → phá LSP → composition
   Interface bắt client implement method nó không dùng?
        có → tách nhỏ (ISP)
```

---

## ⑧ Câu hỏi phỏng vấn & thách đố

**(khó)** *LSP nói gì? Cho ví dụ kế thừa "đúng cú pháp" nhưng phá LSP.*
→ Subtype thay thế supertype không phá contract; ví dụ **Square/Rectangle**, **Ostrich/Bird**.

**(khó)** *Vì sao "chim cánh cụt là chim" lại có thể vi phạm LSP?*
→ **Phân loại ≠ thay thế hành vi**: nó *là* chim nhưng *không bay được*, nên không "đóng thế" được `Bird-biết-bay`.

**(TB)** *`Worker{work,eat}` bắt robot `eat()` rỗng — sai gì, sửa sao?*
→ Vi phạm **ISP**; **tách `Workable` / `Eatable`**.

> 🧩 **🔥 Thách đố:** *"Tôi override method và `throw new Error('not supported')` — vẫn compile mà?"*
>
> **Bẫy:** **compile được nhưng PHÁ LSP** (subtype không thay thế được supertype). Đó là **dấu hiệu phân cấp sai** — phải **tách interface** hoặc **đổi sang composition**. "Compile được" không có nghĩa là "thiết kế đúng" (đúng như tinh thần Bài 2 về Fragile Base Class).

---

## ⑨ Bài tập + tiêu chí tự chấm

**BT1.** Tìm **1 chỗ override ném "not supported"** trong code/lib bạn dùng, đề xuất **tách interface**.
→ ✅ **Đạt khi:** **không subtype nào phải ném-vì-không-hỗ-trợ**.

> 💡 *Gợi ý:* mỗi `throw NotSupported` là một "method thừa" với subtype đó → kéo method đó ra một interface riêng mà chỉ subtype *thật sự hỗ trợ* mới implement.

**BT2.** Tách **1 fat interface** (≥5 method phục vụ ≥2 nhóm client) thành các interface nhỏ.
→ ✅ **Đạt khi:** mỗi client **chỉ phụ thuộc method nó gọi**.

> 💡 *Gợi ý:* nhóm method theo "ai gọi cái gì" (như tách Read/Write ở mục ④), mỗi nhóm client → một interface.

---

## ⑩ Đọc thêm

- **Barbara Liskov** — **"Behavioral subtyping"** (nguồn gốc **LSP**).
- **R. C. Martin** — **SOLID papers**; **Rob Pike** — **Go proverbs** (interface nhỏ).

---

## 🤖 Hiểu để chỉ huy (Kiểm tra AI)

> 🚨 **AI rất hay sinh:**
> 1. **Phân cấp kế thừa theo "is-a" đời thường** (Square extends Rectangle, Ostrich extends Bird) → **vi phạm LSP**.
> 2. **Interface to nhồi nhét** (**fat interface**), buộc client implement method thừa.
>
> 🎯 **Câu soát mỗi lần nhận code OOP từ AI:**
> - *"**Subtype** có thay thế được **supertype** ở **MỌI chỗ dùng** không?"* → không (có `throw`, đổi hành vi) → ép **bỏ kế thừa**, dùng **composition** hoặc **tách interface theo hành vi**.
> - *"Có **client** nào bị ép phụ thuộc method nó **không dùng** không?"* → có → ép **tách fat interface** (ISP).
>
> AI dựng phân cấp theo *trực giác đời thường* ("đà điểu là chim → extends Bird"), nhưng code chạy theo **hành vi**, không theo *phân loại sinh học*. Vai trò chỉ huy của bạn: luôn hỏi **"behaves-like-a chứ không chỉ is-a"**.
