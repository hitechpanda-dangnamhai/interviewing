# 🎯 BÀI 2 — **Composition > Inheritance** · **Fragile Base Class** · **Law of Demeter** · **Program-to-interface**

**Phủ:** Q-OOP-002 → Q-OOP-003 → Q-OOP-008 → Q-OOP-009

> 💡 **Mental model mở đầu:**
> Bài 1 cho bạn **la bàn** (coupling/cohesion). Bài này là **bốn cái cờ-lê** bạn rút ra dùng hằng ngày để vặn coupling xuống thấp:
> 1. Ghép thay vì kế thừa (**Composition > Inheritance**).
> 2. Hiểu vì sao kế thừa hay gãy ngầm (**Fragile Base Class**).
> 3. Chỉ nói chuyện với hàng xóm trực tiếp (**Law of Demeter**).
> 4. Phụ thuộc *bản hợp đồng*, không phụ thuộc *cái máy cụ thể* (**Program-to-interface**).
> Cả bốn đều là **tiền đề** để hiểu **GoF** lẫn **SOLID** sau này.

---

## ① Mục tiêu & vị trí trong mạch

**Bài 1** cho **la bàn** (**coupling** / **cohesion**) — tức là *biết thế nào là tốt*. Bài này dạy **bốn nguyên tắc giảm coupling** bạn dùng *hằng ngày* — tức là *làm sao đạt được cái tốt đó*.

```
   Bài 1: LA BÀN                     Bài 2: BỐN CÔNG CỤ
   (coupling/cohesion)      ───▶     • Composition > Inheritance
   "thế nào là tốt"                  • Fragile Base Class (cái bẫy)
                                     • Law of Demeter
                                     • Program-to-interface
                                              │
                                              ▼
                                   nền cho GoF + SOLID (Bài 4, 5...)
```

> 🎯 **Chốt vị trí:** Bốn nguyên tắc này **không độc lập** — tất cả đều quy về một câu của Bài 1: *kéo dây nối ra ngoài cho ít đi*. Khi học **GoF** và **SOLID**, bạn sẽ thấy chúng chỉ là cách *đóng gói lại* bốn nguyên tắc này.

---

## ② Giảng cơ bản → nâng cao

### (a) Favor composition over inheritance

> **Ẩn dụ — gắn cứng vs lắp ghép.**
> - **Inheritance** giống **hàn cứng** con vào cha: muốn đổi, phải cắt mối hàn (rủi ro vỡ cả hai).
> - **Composition** giống **lắp lego**: cần hành vi nào thì *cắm* mảnh đó vào, không cần thì *rút* ra — đổi runtime, không cần hàn xì.

| Tiêu chí | **Inheritance** (`B extends A`) | **Composition** (`B chứa & uỷ quyền`) |
|---|---|---|
| Quan hệ | **"is-a"** — B *là một* A | **"has-a"** — B *có một* thành phần |
| Coupling | **chặt**: B phụ thuộc cả **implementation** của A | **lỏng**: B chỉ dùng *hành vi* được phơi ra |
| Đổi lúc runtime | ❌ cố định lúc compile | ✅ ghép/đổi hành vi linh hoạt |
| Test từng mảnh | khó (dính cha) | ✅ mỗi mảnh test riêng được |
| Rủi ro | **Fragile Base Class** (xem mục b) | thấp |

> 💡 **Vì sao inheritance tạo coupling CHẶT?**
> Vì `B extends A` không chỉ thừa kế *giao diện* của A, mà thừa kế cả **implementation** (cách A làm bên trong). B vô tình "biết" và *dựa vào* chi tiết nội bộ của A. **Đổi A có thể làm vỡ B một cách NGẦM** — không có lỗi compile, chỉ sai khi chạy. Đó chính là **Fragile Base Class** ở mục tiếp theo.

> 🎯 **Nhưng đừng bài trừ inheritance.** Nó vẫn **đúng** khi thoả **đủ 3 điều kiện**:
> 1. Quan hệ **"is-a" thật** (không phải gượng ép).
> 2. Thoả **LSP** (Liskov Substitution Principle — Bài 5): con thay được cha mà không phá hành vi.
> 3. Phân cấp **ổn định** (ít thay đổi theo thời gian).
> Nguyên tắc là **favor** (ưu tiên), **không phải forbid** (cấm). Mặc định chọn composition; chỉ dùng inheritance khi *chứng minh được* cả 3 điều kiện.

---

### (b) ➕ Fragile Base Class problem (nâng cao)

> **Ví dụ trực giác:** bạn sửa nhà bếp của tầng trệt, nhưng tầng 2 (xây dựa lên tầng trệt) tự dưng nứt tường — dù bạn *không hề đụng* tầng 2. Đó là "gãy ngầm" của kế thừa.

**Bản chất:** class **con phụ thuộc chi tiết implementation của cha**. Khi cha đổi *cách làm bên trong* (dù giao diện không đổi), con vỡ.

**Ví dụ kinh điển — `CountingList`:**

```
T0  Bạn viết:  CountingList extends ArrayList
              override add()  → mỗi lần add, tăng biến đếm count
T1  Bạn gọi:   list.addAll([a, b, c])   // thêm 3 phần tử, kỳ vọng count += 3

    Nhưng addAll() của CHA (ArrayList) được hiện thực kiểu:
      addAll(items) { for (x of items) this.add(x); }   // gọi NỘI BỘ add()!
                                              │
                                              ▼
T2  Mỗi lần cha gọi this.add(x) → trúng add() ĐÃ OVERRIDE của con
    → count bị tăng MỘT LẦN NỮA
                                              │
                                              ▼
❌ Kết quả: count = 6 thay vì 3  → ĐẾM SAI GẤP ĐÔI
```

> 🚨 **Điều đáng sợ nhất:** bạn **không đụng** class con, code con vẫn đúng *trên giấy*. Nó vỡ chỉ vì **cha thay đổi cách `addAll` gọi `add` nội bộ** — một chi tiết lẽ ra là "chuyện riêng" của cha. Lỗi này **không có cảnh báo compile**, lặng lẽ trả số sai.

> 🎯 **Bài học:** **inheritance để lộ white-box** — con nhìn thấy & dựa vào *ruột* của cha. **Composition là black-box** — bạn chỉ gọi hành vi công khai, không phụ thuộc cách nó làm bên trong. Đây là lý do gốc rễ để **ưu tiên composition**.

| | **white-box** (inheritance) | **black-box** (composition) |
|---|---|---|
| Con/client thấy gì? | cả **implementation** nội bộ của cha | chỉ hành vi *công khai* được phơi |
| Cha đổi nội bộ → ? | có thể **vỡ ngầm** | an toàn (miễn hành vi công khai giữ nguyên) |

---

### (c) Law of Demeter ("principle of least knowledge")

> **Ẩn dụ:** bạn nhờ con đi mua đồ. Bạn nói *"con mua giúp bố ổ bánh mì"* (**Tell** — bảo nó làm) — chứ không thò tay vào **ví của con**, lấy tiền, đếm, rồi tự đi (**Ask + reach sâu**). Cái sau vừa thô lỗ vừa dễ hỏng.

**Law of Demeter** = **"principle of least knowledge"** (nguyên tắc biết-ít-nhất): *"chỉ nói chuyện với bạn bè trực tiếp"*.

🚨 **Dấu hiệu vi phạm — "train wreck"** (tàu trật bánh): chuỗi gọi nhiều cấp `a.b().c().d()`.

```
❌ order.getCustomer().getAddress().getCity()
        │            │            │
        │            │            └─ reach vào ruột Address
        │            └─ reach vào ruột Customer
        └─ lấy Customer ra khỏi Order
```

> 💡 **Vì sao train wreck xấu?** Vì code gọi giờ **coupling với CẢ CHUỖI** cấu trúc nội bộ: nó "biết" Order có Customer, Customer có Address, Address có city. **Điều tệ nhất:** chỉ cần đổi cấu trúc **Address** (vd gộp city vào một object `Location`), thì **MỌI nơi reach sâu tới `.getCity()` đều vỡ** cùng lúc. Một thay đổi nhỏ → vỡ rải rác khắp codebase.

> 🎯 **Hướng sửa — "Tell, Don't Ask":** thêm method ở **đúng tầng** để mỗi object tự hỏi hàng xóm trực tiếp của nó.
> ```
> ✅ order.shippingCity()
>    → Order tự hỏi Customer, Customer tự hỏi Address — BÊN TRONG.
>    → Client chỉ nói chuyện với Order (bạn bè trực tiếp), không reach sâu.
> ```
> Giờ đổi cấu trúc Address chỉ phải sửa **một chỗ** (bên trong Customer/Address), client không hề hấn.

---

### (d) "Program to an interface, not an implementation"

> **Ẩn dụ — ổ cắm điện.** Cái quạt cắm vào **ổ điện tiêu chuẩn** (contract), không hàn thẳng vào nhà máy điện cụ thể. Đổi nhà máy (thuỷ điện → điện mặt trời), cái quạt vẫn chạy vì nó chỉ dựa vào *chuẩn ổ cắm*, không dựa vào *cái máy*.

**Nguyên tắc:** phụ thuộc vào **contract / kiểu trừu tượng**, **không** phụ thuộc **concrete type** (class cụ thể). Mục tiêu: **tách client khỏi class cụ thể** để **thay được implementation**.

> 🚨 **CẢNH BÁO — bẫy ngôn từ:** *"Program to an interface"* **KHÔNG đồng nghĩa với từ khoá `interface`.**
> Nó nói về **phụ thuộc abstraction/contract**, và có thể hiện thực bằng:
> - một `interface` (TS/Java), **hoặc**
> - một **abstract class**, **hoặc**
> - thậm chí **duck typing** trong JS/TS (chỉ cần "trông giống vịt, kêu như vịt" — có đúng method là dùng được).
>
> Đừng tưởng "tôi đã dùng từ khoá `interface` nên tôi đã program-to-interface". Câu hỏi thật là: *"client của tôi có đang phụ thuộc vào một class cụ thể nào không?"*

---

## ③ ⚠️ Kiến thức cũ → Mới

| Cách làm cũ/phản xạ | Nên dùng nay | Vì sao |
|---|---|---|
| **Kế thừa sâu nhiều tầng** để tái dùng code | **Composition + uỷ quyền** (mặc định) | Tránh **fragile base class**, **coupling chặt**, **"is-a" giả** |
| `a.b().c().d()` **reach sâu** vào chuỗi object | Thêm method ở **đúng tầng** (**"Tell, Don't Ask"**) | Giảm **coupling** với cấu trúc nội bộ; đổi 1 chỗ không vỡ rải rác |
| Phụ thuộc **concrete class** trong code | Phụ thuộc **abstraction / contract** | Thay **implementation** / test không cần sửa **client** |

---

## ④ Áp dụng thực tế + So sánh bigtech

### Use case: chuỗi logger

```
❌ KẾ THỪA SÂU                          ✅ COMPOSITION + program-to-interface
   JsonLogger extends Logger              interface Logger { log(msg): void }
            extends BaseWriter                       ▲           ▲
   (3 tầng hàn cứng, đổi 1 tầng vỡ cả)    ConsoleLogger      FileLogger
                                          new Service(logger)  ← inject mảnh cần
                                          Đổi console→file = ĐỔI THÀNH PHẦN inject,
                                          không sửa Service
```

> 💡 Đổi từ console-logger sang file-logger chỉ là **đổi thành phần inject** — `Service` không biết và không cần biết. Đó là sức mạnh của **composition + program-to-interface** cộng lại.

### Bigtech / ecosystem (pattern phổ biến — không tuyệt đối)

| Hệ sinh thái | Bằng chứng "composition > inheritance" |
|---|---|
| **Go** | Ngôn ngữ **cố tình bỏ inheritance**, chỉ có **composition + interface ngầm** → minh chứng ở cấp *thiết kế ngôn ngữ* |
| **React** | Tài liệu chính thức **khuyến nghị composition** thay vì kế thừa component |

> 🔎 **Insight sâu:** khi cả một ngôn ngữ (Go) *bỏ hẳn* tính năng kế thừa mà vẫn xây được hệ thống lớn, đó là bằng chứng mạnh rằng inheritance **không phải thứ thiết yếu** — composition + interface là đủ, và thường tốt hơn.

---

## ⑤ Code thực hành

```typescript
// composition-vs-inheritance.ts — TypeScript 5.x

// ❌ Inheritance: coupling chặt, fragile, chỉ đúng khi "is-a" THẬT
// class JsonReportService extends HttpClient {}  // Service "LÀ" HttpClient? Sai bản chất "is-a".

// ✅ Composition + program-to-interface
interface HttpClient {                      // ABSTRACTION (contract) — không phải class cụ thể
  get(url: string): Promise<unknown>;
}

class ReportService {
  // Phụ thuộc ABSTRACTION, KHÔNG phụ thuộc class cụ thể
  // -> loose coupling, và dễ test (inject fake thay vì gọi network thật)
  constructor(private readonly http: HttpClient) {}

  async fetchReport(id: string) {
    return this.http.get(`/reports/${id}`);   // dùng contract, kệ bên dưới là fetch/axios/fake
  }
}

// --- Law of Demeter: "Tell, Don't Ask" ---
class Address { constructor(public readonly city: string) {} }

class Customer {
  constructor(private readonly address: Address) {}
  // Bọc lại: KHÔNG bắt bên ngoài reach sâu vào address.city
  get city() { return this.address.city; }
}

class Order {
  constructor(private readonly customer: Customer) {}
  // ✅ Thay vì train wreck: order.getCustomer().getAddress().getCity()
  // Order tự hỏi Customer BÊN TRONG -> client chỉ nói chuyện với Order (bạn bè trực tiếp)
  shippingCity(): string { return this.customer.city; }
}
```

> 💡 **Test seam nhờ program-to-interface:** trong unit test, truyền một object giả `{ get: async () => ({...}) }` làm **HttpClient** — **không cần network thật**. Đây là phần thưởng cụ thể của việc phụ thuộc contract: bạn *cắm* được một implementation giả vào lúc test.

---

## ⑥ Keywords cần nhớ

> 🧠 **Cần HIỂU:**
> - **composition vs inheritance** — "has-a" (ghép) vs "is-a" (kế thừa); mặc định ưu tiên composition
> - **fragile base class** — con vỡ ngầm khi cha đổi implementation nội bộ
> - **Law of Demeter** — "principle of least knowledge", chỉ nói chuyện với bạn bè trực tiếp
> - **"Tell, Don't Ask"** — bảo object làm việc, đừng moi ruột nó ra rồi tự làm
> - **program-to-interface** — phụ thuộc abstraction/contract (**≠ từ khoá `interface`**)

> 📌 **Cần THUỘC:**
> - Dấu hiệu **train wreck**: `a.b().c().d()` (chuỗi `.` quá 1 cấp).
> - **3 điều kiện inheritance còn hợp lệ**: **is-a thật** + **LSP** (Bài 5) + **ổn định**.

> 🛠️ **Cần LÀM ĐƯỢC:**
> - Refactor một **kế thừa** thành **composition**.
> - Viết **method bọc** để khử **train wreck**.
> - **Inject abstraction** để tạo **test seam**.

---

## ⑦ Mental model

> 🎯 **Khẩu quyết:**
> - Mặc định **"has-a"** (composition) thay vì **"is-a"** (inheritance).
> - Nói chuyện với **hàng xóm trực tiếp** (Demeter), đừng reach xuyên nhiều cấp.
> - Phụ thuộc **contract**, đừng phụ thuộc **class cụ thể**.

```
   Cần hành vi X?  →  CẮM mảnh X vào (composition), đừng HÀN bằng kế thừa
   Cần dữ liệu sâu? →  BẢO object trả lời (Tell), đừng reach a.b().c().d()
   Cần gọi service? →  cầm CONTRACT, đừng ôm CLASS cụ thể
```

---

## ⑧ Câu hỏi phỏng vấn & thách đố

**(TB)** *"Favor composition over inheritance" nghĩa là gì, khi nào inheritance vẫn đúng?*
→ Mặc định ghép (**composition**) vì lỏng & linh hoạt; **inheritance** đúng khi **is-a thật + LSP + ổn định**.

**(TB)** *Law of Demeter cấm gì? `order.getCustomer().getAddress().getCity()` sai sao?*
→ Cấm **reach sâu**; sai vì **coupling với cấu trúc nội bộ** nhiều cấp → đổi cấu trúc là vỡ rải rác.

**(khó)** *Fragile base class: cho một tình huống sửa cha làm vỡ con dù không đụng con.*
→ Con **override `add()`** để đếm; cha đổi **`addAll()` gọi `add()` nội bộ** → đếm sai gấp đôi.

**(khó)** *"Program to interface" có nghĩa là phải dùng từ khoá `interface` không?*
→ **Không.** Nói về **phụ thuộc abstraction/contract** — có thể là abstract class hoặc **duck typing**.

> 🧩 **🔥 Thách đố:** *"Composition luôn tốt hơn inheritance, đúng không?"*
>
> **Bẫy:** **không tuyệt đối.** Với **phân cấp ổn định + is-a thật + thoả LSP**, inheritance gọn hơn (ít boilerplate uỷ quyền). Nguyên tắc là **favor**, **không phải forbid**. Trả lời "luôn luôn" là rơi bẫy — câu đúng là *"mặc định composition, nhưng inheritance hợp lệ khi chứng minh được 3 điều kiện"*.

---

## ⑨ Bài tập + tiêu chí tự chấm

**BT1.** Có class `AdminUser extends User` chỉ để **thêm quyền**.
→ **Yêu cầu:** Refactor sang **composition** (vd: `User` + `Role`).
→ ✅ **Đạt khi:** thêm một loại quyền mới **không cần thêm subclass** (chỉ thêm/đổi `Role`).

> 💡 *Gợi ý:* `AdminUser` là **"is-a" giả** — admin không phải *một loại người dùng khác về bản chất*, mà là *người dùng có thêm quyền*. "có thêm quyền" = **"has-a Role"** → dùng composition.

**BT2.** Tìm **1 train wreck** trong code của bạn, khử bằng **method bọc**.
→ ✅ **Đạt khi:** client **không còn gọi quá 1 cấp `.`** (hết `a.b().c().d()`).

---

## ⑩ Đọc thêm

- **Design Patterns (GoF)** — nguyên tắc mở đầu cả cuốn: *"favor object composition over class inheritance"*.
- **The Pragmatic Programmer** — **Law of Demeter**, **"Tell, Don't Ask"**.

---

## 🤖 Hiểu để chỉ huy (Kiểm tra AI)

> 🚨 **AI rất hay sinh:**
> 1. **Kế thừa sâu** nhiều tầng để "tái dùng code" — kiểu `A extends B extends C`.
> 2. **Train wreck** `a.b().c().d()` reach sâu vào ruột object.
>
> 🎯 **Hai câu soát bạn PHẢI hỏi mỗi lần nhận code OOP từ AI:**
> - *"Kế thừa này có phải **is-a thật** không, hay chỉ là tái dùng code trá hình?"* → nếu chỉ để tái dùng → ép đổi sang **composition**.
> - *"Có chuỗi `.` nào quá **1 cấp** không?"* → nếu có → ép thêm **method bọc** ở đúng tầng (**Tell, Don't Ask**).
>
> Bạn cầm bốn nguyên tắc này như bốn cái lưới lọc. Code AY "chạy được" rất dễ qua mắt — nhưng coupling chặt và train wreck chỉ lộ ra khi *bạn chủ động hỏi*. Đó là khác biệt giữa nghiệm thu và chỉ huy.
