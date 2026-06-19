# 🎯 BÀI 3 — **DRY** / **KISS** / **YAGNI** · **OOP vs Functional** trong TS

**Phủ:** Q-OOP-006 → Q-OOP-007 → Q-OOP-010

> 💡 **Mental model mở đầu:**
> Bài 1–2 dạy bạn *cách kéo coupling xuống*. Bài này dạy điều ngược lại nhưng quan trọng không kém: **biết khi nào ĐỪNG làm gì cả.**
> - **DRY / KISS / YAGNI** = **bộ phanh** chống **over-engineering** (vẽ vời quá mức).
> - **OOP vs Functional** = phá định kiến "thiết kế tốt = phải full class" — trong TS, **một hàm thường thay được cả cây class**.
> Nhiều bug kiến trúc không đến từ *làm thiếu*, mà từ *làm thừa*. Đây là bài về sự kiềm chế.

---

## ① Mục tiêu & vị trí trong mạch

Ba chữ **DRY / KISS / YAGNI** là **bộ phanh chống over-engineering** — cực kỳ quan trọng cho phần **judgment** (khả năng phán đoán) của **Tech Lead** (sẽ học sâu ở **Bài 15**).

Đồng thời bài này **phá một định kiến phổ biến**: *"thiết kế tốt = OOP thuần (toàn class)"*. Trong **TS / Node**, **first-class function** (hàm là công dân hạng nhất — truyền/trả về như giá trị) thường **thay được nhiều pattern GoF** mà gọn hơn nhiều.

```
   Bài 1–2: kéo coupling XUỐNG  (làm cho đúng)
                    │
   Bài 3:  bộ PHANH — biết khi nào ĐỪNG làm  (làm cho vừa)
                    │
                    ▼
        judgment của Tech Lead (Bài 15)
```

> 🎯 **Chốt vị trí:** kỹ năng đáng giá nhất của senior không phải "biết thêm pattern", mà là **biết khi nào KHÔNG dùng pattern**. Bài này là nền cho cái đó.

---

## ② Giảng cơ bản → nâng cao

### (a) Ba nguyên tắc — định nghĩa gốc

| Nguyên tắc | Viết đầy đủ | Nói gì | Ví dụ trực giác |
|---|---|---|---|
| **DRY** | **Don't Repeat Yourself** | mỗi mẩu **kiến thức/quy tắc** có **một nguồn chân lý duy nhất** (single source of truth) | thuế VAT 10% chỉ khai báo *một chỗ*, không rải khắp 20 file |
| **KISS** | **Keep It Simple** | chuộng giải pháp **đơn giản nhất đủ dùng** | cần cộng 2 số thì viết `a + b`, đừng dựng "CalculationEngine" |
| **YAGNI** | **You Aren't Gonna Need It** | **đừng xây trước** thứ chưa cần ("để sau biết đâu cần") | đừng thêm `multiCurrency` khi app mới chỉ bán ở VN |

---

### (b) ⚠️ DRY bị hiểu sai NẶNG nhất

> 🚨 **CẢNH BÁO — đây là hiểu lầm tai hại nhất trong cả bài:**
> **DRY KHÔNG phải "không có dòng code trùng nhau".**
> DRY là về **KIẾN THỨC**, **không** về **KÝ TỰ**.

> **Ví dụ trực giác:** hai người tình cờ cùng cao 1m70. Họ *giống nhau ở một con số*, nhưng đó là hai con người *khác nhau hoàn toàn* — gộp hồ sơ của họ làm một là vô lý. Code trùng ngẫu nhiên cũng vậy.

**Vì sao gộp code trùng ngẫu nhiên lại NGUY HIỂM?**

Hai đoạn code *trông giống nhau hôm nay* nhưng **thay đổi vì lý do khác nhau** thì **KHÔNG nên gộp**. Gộp lại tạo ra **coupling giả** (còn gọi **premature abstraction** / **false abstraction** — trừu tượng hoá non/sai):

```
T0  Bạn thấy 2 đoạn giống hệt → gộp thành 1 abstraction chung  ✅ (tưởng là)
                    │
T1  Một bên cần đổi (vd luật riêng) → abstraction chung KHÔNG cover
                    │
T2  Bạn nhét if/flag vào abstraction chung để "chiều" cả 2 ca
                    │
T3  Lặp lại T1–T2 vài lần → abstraction MÉO MÓ, đầy cờ rẽ nhánh
                    │
                    ▼
❌ Giờ sửa ca A vô tình vỡ ca B (vì chung một hàm) → coupling giả phản đòn
```

> 🎯 **Câu thần chú khắc cốt:**
> **"Hai thứ giống nhau hôm nay không đảm bảo cùng đổi ngày mai."**
> Trùng **ngẫu nhiên** (coincidental) ≠ trùng **kiến thức** (knowledge). Chỉ gộp khi chúng *cùng một quy tắc*, *cùng lý do thay đổi*.

> 🔎 **Liên hệ:** đây là tiền đề của smell **"false abstraction"** (Bài 10), và là **đối trọng với SRP** (Bài 4 — một class một lý do thay đổi). DRY sai sẽ *gom* những thứ có *lý do thay đổi khác nhau* — đúng cái SRP cấm.

---

### (c) DRY quá đà — ví dụ cụ thể

```
Có 2 hàm validate, TÌNH CỜ cùng logic "số trong [0, 150]":
   validateUserAge(n)     → n trong [0, 150]   (luật TUỔI)
   validateProductYear(n) → n trong [0, 150]   (luật NĂM, tạm thời)
                    │
   "DRY!" → gộp thành validateRange(0, 150)
                    │
T+1 tháng:  product cho phép năm tới 9999
                    │
                    ▼
❌ Phải nhét tham số/biến thể vào validateRange → hàm chung phình to,
   trong khi đáng lẽ để RIÊNG vì "luật tuổi" và "luật năm" là HAI quy tắc khác nhau
```

> 💡 **KISS + YAGNI bảo gì ở đây?**
> *"Cứ để 2 hàm riêng cho tới khi xuất hiện nhu cầu THẬT để gộp."* Hai hàm trùng vài dòng **rẻ hơn nhiều** so với một abstraction sai mà bạn phải gỡ về sau. **Duplication đôi khi rẻ hơn abstraction sai** (xem Sandi Metz, mục ⑩).

---

### (d) ➕ OOP vs Functional trong TS/Node (nâng cao)

> **Ví dụ trực giác:** thay vì xây cả một **dây chuyền nhà máy** (cây class) chỉ để "chọn cách làm", trong TS bạn thường chỉ cần **đưa cho nó một công thức** (một hàm). Gọn hơn rất nhiều.

Nhiều **pattern GoF** "**co lại**" nhờ **first-class function** / **closure** (hàm bắt giữ biến môi trường xung quanh) / **module**:

| Pattern GoF (OOP-thuần) | Bản thu gọn trong TS | Cơ chế |
|---|---|---|
| **Strategy** | **truyền một hàm vào** | không cần cây class, chỉ cần `fn` |
| **Command** | **một closure bắt context** | closure giữ sẵn dữ liệu cần thực thi |
| **Singleton** | **một module** | Node **cache module** → `export` instance là **singleton sẵn** (xem **Bài 7**) |
| **Decorator** (hành vi) | **hàm bọc hàm** (**higher-order function**) | hàm nhận hàm, trả về hàm "đã bọc thêm" |

> 🎯 **Judgment (phán đoán):** chọn **paradigm** (OOP hay FP) **theo bài toán**, **KHÔNG** ép OOP-thuần "cho giống sách". Đôi khi class là đúng (state phức tạp, nhiều invariant); đôi khi một hàm là đủ và tốt hơn.

> ⚠️ **Lưu ý "verify":** khi học, **verify lại minh hoạ cú pháp** — **TS đổi nhanh** ở mảng **decorator / type**. Đừng tin cú pháp cũ trong tài liệu cũ; kiểm tra với phiên bản TS bạn đang dùng.

---

## ③ ⚠️ Kiến thức cũ → Mới

| Hiểu sai | Hiểu đúng | Vì sao |
|---|---|---|
| **"DRY = xoá mọi dòng code trùng"** | **DRY** = **một nguồn chân lý** cho mỗi **quy tắc** | Gộp code **giống-ngẫu-nhiên** ⇒ **coupling giả** |
| **"Thiết kế tốt ⇒ phải full OOP/class"** | Chọn **paradigm** theo bài toán; **FP thay nhiều GoF** trong TS | **first-class function** làm pattern gọn hơn |
| **"Cứ trừu tượng hoá sẵn cho tương lai"** | **YAGNI**: chỉ trừu tượng khi **đau thật** | **speculative generality** (tổng quát hoá đầu cơ) ⇒ phức tạp thừa |

---

## ④ Áp dụng thực tế + So sánh bigtech

### Use case: chọn cách sort

```
❌ OOP-THUẦN (thừa)                     ✅ FP-STYLE (idiom của TS)
   abstract class SortStrategy            sort(items, compareFn)
   class AscStrategy extends ...          → truyền HÀM so sánh vào
   class DescStrategy extends ...         → gọn, dễ test, không cây class
   class ByPriceStrategy extends ...
   (4 class cho 1 việc nhỏ)
```

> 💡 Truyền `compareFn` chính là **Strategy bằng hàm**. Gọn, dễ test (chỉ cần test cái hàm), đúng tinh thần TS.

### Bigtech / ecosystem (pattern phổ biến — không tuyệt đối)

| Bằng chứng | Ý nghĩa |
|---|---|
| **`Array.sort(compareFn)`**, **`Array.map(fn)`** trong JS chuẩn | chính là **Strategy bằng hàm** — nằm sẵn trong API ngôn ngữ |

> 🔎 **Insight sâu:** khi **API chuẩn của chính ngôn ngữ** dùng FP-style (truyền hàm), thì đó là **idiom** (cách viết tự nhiên) của ngôn ngữ, **không phải "thiếu thiết kế"**. Viết `[3,1,2].sort((a,b)=>a-b)` không ai chê là "thiếu OOP" — vì nó *đúng chất* ngôn ngữ.

---

## ⑤ Code thực hành

```typescript
// strategy-as-function.ts — GoF Strategy thu gọn bằng first-class function

type ShippingFee = (weightKg: number) => number;

// "factory" = OBJECT MAP, không cần class hierarchy
// Mỗi "strategy" chỉ là một hàm nằm trong map
const fees: Record<string, ShippingFee> = {
  standard: (w) => w * 1.0,
  express:  (w) => w * 2.5 + 3,
};

function quote(method: string, weightKg: number): number {
  const fn = fees[method];                         // chọn "strategy" lúc RUNTIME theo key
  if (!fn) throw new Error(`Unknown method: ${method}`);
  return fn(weightKg);                             // thêm method mới = thêm 1 ENTRY vào map (gần với OCP)
}

// ⚠️ PHẢN VÍ DỤ — DRY quá đà, KHÔNG nên gộp 2 hàm chỉ vì TRÔNG giống:
// const validateRange = (min: number, max: number) => (n: number) => n >= min && n <= max;
//   -> Nếu "luật tuổi" và "luật năm" rồi sẽ rẽ nhánh KHÁC nhau,
//      hãy để RIÊNG (2 lý do thay đổi khác nhau) — đừng gộp tạo coupling giả.
```

> 💡 **Đọc kỹ dòng `fees`:** thêm phương thức ship mới (vd `economy`) chỉ là **thêm 1 entry**, **không sửa** hàm điều phối `quote`. Đây là tinh thần **OCP** (Open/Closed) đạt được *bằng hàm*, không cần cây class.

---

## ⑥ Keywords cần nhớ

> 🧠 **Cần HIỂU:**
> - **DRY** (về **kiến thức**, không về ký tự) — một nguồn chân lý cho mỗi quy tắc
> - **KISS** — đơn giản nhất đủ dùng
> - **YAGNI** — đừng làm trước thứ chưa cần
> - **premature / false abstraction** — gộp sai tạo coupling giả
> - **FP thay GoF** — first-class function thay nhiều pattern

> 📌 **Cần THUỘC:**
> - Phát biểu **đúng** của **DRY**: *một nguồn chân lý cho mỗi quy tắc* (không phải "không trùng ký tự").
> - Câu: **"2 thứ giống nhau + lý do đổi khác nhau ⇒ KHÔNG gộp."**

> 🛠️ **Cần LÀM ĐƯỢC:**
> - Nhận ra một **abstraction gộp sai** và **tách lại**.
> - Viết **Strategy / Command** bằng **function / closure**.

---

## ⑦ Mental model

> 🎯 **Khẩu quyết:**
> **DRY là về kiến thức, không về ký tự.** Đơn giản trước (**KISS**), chỉ làm khi cần (**YAGNI**), và trong TS — **một hàm thường thay được cả cây class**.

```
   Thấy code trùng?     → hỏi "CÙNG kiến thức không?" trùng ngẫu nhiên thì ĐỪNG gộp
   Định trừu tượng hoá? → hỏi "đau THẬT chưa?" chưa đau thì ĐỪNG (YAGNI)
   Định dựng cây class? → hỏi "một hàm có đủ không?" đủ thì DÙNG HÀM (KISS)
```

---

## ⑧ Câu hỏi phỏng vấn & thách đố

**(TB)** *DRY / KISS / YAGNI mỗi cái cảnh báo gì?*
→ lặp **kiến thức** / **phức tạp** thừa / làm **trước** cái chưa cần.

**(khó)** *Phát biểu đúng của DRY?*
→ **Một nguồn chân lý cho mỗi quy tắc** — **không phải** "không trùng ký tự".

**(khó)** *Trong TS, "truyền một hàm vào" đã là Strategy chưa?*
→ **Rồi** — đó là **bản gọn của Strategy**.

> 🧩 **🔥 Thách đố:** *"Thấy 2 đoạn code y hệt — gộp ngay chứ?"*
>
> **Bẫy:** phải hỏi **"chúng có CÙNG lý do thay đổi không?"** Nếu **không** → gộp = **coupling giả**, sau này khổ (đầy if/flag). **Trùng ngẫu nhiên ≠ trùng kiến thức.** Trả lời "gộp ngay" là rơi bẫy.

---

## ⑨ Bài tập + tiêu chí tự chấm

**BT1.** Tìm trong code bạn **1 abstraction "gộp cho DRY"** mà giờ **đầy if/flag** cho 2 ca khác nhau. **Tách lại.**
→ ✅ **Đạt khi:** mỗi nhánh là một **hàm/class độc lập**, **không còn cờ rẽ nhánh chung**.

> 💡 *Gợi ý:* dấu hiệu của false abstraction là tham số kiểu `isAdmin`, `mode`, `type` được truyền vào *chỉ để* hàm chung rẽ nhánh `if`. Tách mỗi ca ra một hàm riêng, xoá cờ.

**BT2.** Viết lại một **`switch` xử lý theo loại** bằng **object map of functions**.
→ ✅ **Đạt khi:** thêm loại mới = **thêm 1 entry**, **không sửa** hàm điều phối.

> 💡 *Gợi ý:* y hệt ví dụ `fees` ở mục ⑤ — biến mỗi `case` thành một entry `{ key: fn }`.

---

## ⑩ Đọc thêm

- **The Pragmatic Programmer** — **DRY** (định nghĩa gốc, nhấn vào từ **"knowledge"**).
- **Sandi Metz** — **"The Wrong Abstraction"** (vì sao **duplication đôi khi rẻ hơn abstraction sai**).

---

## 🤖 Hiểu để chỉ huy (Kiểm tra AI)

> 🚨 **AI rất hay mắc HAI lỗi ngược chiều nhau:**
> 1. **"DRY hoá"** hai đoạn *giống ngẫu nhiên* thành một **hàm tham-số-hoá** → tạo **coupling giả**.
> 2. **Over-engineer**: dựng cả **class** cho thứ chỉ cần **1 hàm**.
>
> 🎯 **Câu soát theo YAGNI / KISS mỗi lần nhận code từ AI:**
> - *"Hai đoạn AI vừa gộp có **cùng lý do thay đổi** không?"* → không → bắt **tách lại**.
> - *"Cái abstraction/class này có **nhu cầu THẬT** chưa, hay chỉ 'cho có'?"* → chưa → bắt **đơn giản hoá** về một hàm.
>
> AI mặc định "thích" trông-có-vẻ-kỹ-thuật: nhiều class, nhiều abstraction. Vai trò chỉ huy của bạn là **cắt bớt** — vì code dễ bảo trì nhất thường là code *vừa đủ*, không phải code *nhiều pattern nhất*.
