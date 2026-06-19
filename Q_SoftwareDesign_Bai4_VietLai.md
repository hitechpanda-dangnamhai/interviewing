# 🎯 BÀI 4 — **SOLID** (1): **Single Responsibility** & **Open/Closed**

**Phủ:** Q-SOLID-001 → Q-SOLID-002 → Q-SOLID-003 → Q-SOLID-004 → Q-SOLID-005

> 💡 **Mental model mở đầu:**
> **SOLID** không phải 5 luật trời ơi cần học thuộc. Nó là cách **hệ thống hoá la bàn Bài 1**:
> - **SRP** + **ISP** → kéo **cohesion** LÊN (gom đúng bộ).
> - **OCP** + **DIP** → kéo **coupling** XUỐNG (ít dây nối).
> Bài này gói gọn **SRP + OCP** — *hai chữ bạn vi phạm nhiều nhất*. Nắm hai cái này là đã chặn được phần lớn code tệ.

---

## ① Mục tiêu & vị trí trong mạch

**SOLID** (do **Robert C. Martin** tổng hợp) chính là **la bàn Bài 1 được hệ thống hoá** thành 5 nguyên tắc cụ thể:

```
                  LA BÀN (Bài 1)
          ┌──────────────┴──────────────┐
     cohesion ↑                     coupling ↓
   ┌─────┴─────┐                 ┌─────┴─────┐
  SRP         ISP               OCP         DIP
 (Bài 4)    (Bài 5)           (Bài 4)     (Bài 6)
                  + LSP (Bài 5)
```

> 🎯 **Chốt vị trí:** Bài này lo **SRP + OCP**. **Bài 5** lo **LSP + ISP**. **Bài 6** lo **DIP + trade-off**. Học theo cặp để thấy mỗi cặp phục vụ một đầu của la bàn.

---

## ② Giảng cơ bản → nâng cao

**Năm chữ SOLID — gọi đúng tên** (đây là **Q-SOLID-001**):

| Chữ | Tên đầy đủ | Phục vụ |
|---|---|---|
| **S** | **SRP** — Single Responsibility Principle | cohesion ↑ |
| **O** | **OCP** — Open/Closed Principle | coupling ↓ |
| **L** | **LSP** — Liskov Substitution Principle | (Bài 5) |
| **I** | **ISP** — Interface Segregation Principle | (Bài 5) |
| **D** | **DIP** — Dependency Inversion Principle | (Bài 6) |

---

### (a) SRP — Single Responsibility Principle

> 🚨 **CẢNH BÁO — hiểu lầm số 1:** **SRP KHÔNG phải "một class làm một việc".**
> "Một việc" là cách hiểu **thô** và sai. Nó dẫn tới hoặc tách vụn máy móc, hoặc gộp nhầm.

> 🎯 **Phát biểu CHUẨN:** *"Một class chỉ nên có **MỘT lý do để thay đổi**"* — tức chỉ phục vụ **một actor / stakeholder** (một nhóm người có quyền yêu cầu thay đổi).
> **"Lý do thay đổi" ≠ "số lượng method".** Một class có 10 method mà tất cả cùng phục vụ một actor thì vẫn đúng SRP.

> **Ví dụ trực giác:** một nhân viên nhận lệnh từ *ba ông sếp khác nhau* (kế toán, marketing, IT) chắc chắn sẽ loạn — ba ông cùng giật một người, yêu cầu mâu thuẫn. Class cũng vậy: phục vụ nhiều actor = nhiều "ông chủ" cùng có quyền bắt nó đổi.

**Ví dụ vi phạm (Q-SOLID-003) — `UserService` ôm 3 lý do thay đổi:**

```
class UserService {
   validate()   ← luật NGHIỆP VỤ      → đổi khi business rule đổi   (actor: product)
   saveToDb()   ← PERSISTENCE          → đổi khi đổi DB              (actor: DBA/infra)
   sendEmail()  ← HẠ TẦNG mail         → đổi khi đổi mail provider   (actor: infra)
}
        │
        ▼  BA lý do thay đổi khác nhau dồn vào MỘT class
❌ Đổi mail provider → phải mở UserService → rủi ro vỡ logic validate (chẳng liên quan!)
```

**Sửa — tách theo lý-do-thay-đổi:**

```
✅ UserValidator   (chỉ luật nghiệp vụ)
   UserRepository  (chỉ persistence)
   EmailSender     (chỉ hạ tầng mail)
   UserService     (chỉ ĐIỀU PHỐI 3 cái trên — orchestrator)
```

> 💡 **Lợi cụ thể:** test/maintain **từng phần độc lập**; **đổi mail không đụng logic validate**. Mỗi class giờ chỉ vỡ vì *đúng một loại lý do*.

> 🔎 **Liên hệ:** **SRP chính là cohesion ở cấp class** (Bài 1) + là vũ khí **chống God Object** (class ôm tất cả — Bài 10). Đây cũng là **đối trọng của DRY sai** (Bài 3): DRY sai *gom* những thứ có lý do đổi khác nhau, SRP *cấm* đúng điều đó.

---

### (b) OCP — Open/Closed Principle

> **Ẩn dụ:** ổ điện trong nhà — bạn cắm thêm thiết bị mới (mở rộng) bằng cách **cắm vào ổ có sẵn**, **không cần đục tường nối lại dây điện** (sửa cái cũ đang chạy).

> 🎯 **Phát biểu:** *"**Mở** để **mở rộng**, **đóng** để **sửa đổi**"* — thêm hành vi mới **mà không sửa code cũ đã chạy ổn**. Đạt qua **abstraction / polymorphism** (interface, **Strategy**, **map handler**).

**Vì sao OCP quan trọng?**

| Cách thêm tính năng | Rủi ro | Vì sao |
|---|---|---|
| ❌ **Sửa code cũ** đang chạy | **regression** (làm hỏng thứ đang chạy) | code cũ đã được test/tin cậy; mở ra sửa là động vào vùng an toàn |
| ✅ **Thêm code mới** (class mới) | thấp | code cũ *không đổi một dòng* → không thể vỡ; chỉ cần test cái mới |

> 🚨 **Mùi vi phạm OCP (Q-SOLID-005) — "switch-on-type":**
> ```
> if (type === 'A') { ... }
> else if (type === 'B') { ... }
> else if (type === 'C') { ... }   ← mỗi loại MỚI lại phải MỞ hàm này ra sửa
> ```
> Đây là **"switch-on-type"** — dấu hiệu **thiếu abstraction**. **Điều tệ nhất:** mỗi lần thêm loại, bạn sửa đúng cái hàm mọi người đang dùng → rủi ro regression dồn vào một điểm.
> **Trị bằng:** **polymorphism / Strategy** hoặc **map handler** `{ A: handleA, B: handleB }` — thêm loại = thêm entry/class, không mở hàm điều phối.

> ➕ **Nâng cao — OCP KHÔNG có nghĩa "không bao giờ sửa code".**
> Nó nói: **thiết kế điểm mở rộng (extension point) tại trục THỰC SỰ biến động.** Đừng mở *mọi thứ* (đó là **over-engineering** — Bài 15). **Mở đúng chỗ hay đổi**, đóng phần ổn định.

---

## ③ ⚠️ Kiến thức cũ → Mới

| Cách cũ / hiểu sai | Nên dùng nay | Vì sao |
|---|---|---|
| **"SRP = một class một việc/một method"** | **SRP** = **một lý do để thay đổi** (một **actor**) | Tránh **tách máy móc** hoặc **gộp nhầm** |
| **`if/else if` theo type**, thêm loại là sửa hàm | **Strategy / polymorphism / map handler** | Đạt **OCP**, giảm rủi ro **regression** |
| **Sửa thẳng class cũ** mỗi lần thêm tính năng | **Thêm class mới** qua **extension point** | *"Đóng để sửa đổi, mở để mở rộng"* |

---

## ④ Áp dụng thực tế + So sánh bigtech

### Use case: cổng thanh toán nhiều provider

```
❌ VI PHẠM OCP + SRP                        ✅ ĐẠT OCP + SRP
   if (provider === 'stripe')  {...}          interface PaymentProvider { pay(): ... }
   else if (provider === 'adyen') {...}                ▲           ▲
   else if (provider === 'paypal'){...}        StripeProvider  AdyenProvider
   (thêm provider = MỞ hàm này sửa)            (thêm provider = THÊM class,
                                                orchestrator KHÔNG đụng)
```

> 💡 Thêm provider = **thêm một class**, **không đụng orchestrator** → vừa **OCP** (không sửa cũ) vừa **SRP** (mỗi provider lo việc của mình).

### Bigtech / ecosystem (pattern phổ biến — không tuyệt đối)

| Hệ thống | OCP ở quy mô lớn |
|---|---|
| **VS Code extensions** | core "đóng", mở rộng qua **extension** |
| **webpack loaders** | thêm khả năng xử lý file mới = thêm **loader**, không sửa core |
| **Express middleware** | thêm hành vi request = thêm **middleware**, không sửa framework |

> 🔎 **Insight sâu:** **plugin architecture** chính là **OCP ở quy mô kiến trúc**. Core được *đóng* (ổn định, không ai sửa), còn *mở* qua **plugin / interface**. Cả một hệ sinh thái (hàng nghìn extension VS Code) xây trên ý tưởng OCP — đó là bằng chứng nó scale tới mức nào.

---

## ⑤ Code thực hành

```typescript
// ocp-strategy.ts — khử switch-on-type, đạt OCP + SRP

interface PriceRule {                          // ABSTRACTION = điểm mở rộng (OCP)
  appliesTo(type: string): boolean;
  price(base: number): number;
}

class RegularPrice implements PriceRule {
  appliesTo(t: string) { return t === 'regular'; }
  price(base: number) { return base; }
}

class VipPrice implements PriceRule {
  appliesTo(t: string) { return t === 'vip'; }
  price(base: number) { return base * 0.8; }   // thêm loại MỚI = thêm class, KHÔNG sửa calculatePrice
}

class PriceCalculator {                        // SRP: chỉ ĐIỀU PHỐI, không chứa từng luật giá
  constructor(private readonly rules: PriceRule[]) {}
  calculate(type: string, base: number): number {
    const rule = this.rules.find(r => r.appliesTo(type));   // tìm rule phù hợp (polymorphism)
    if (!rule) throw new Error(`No rule for ${type}`);
    return rule.price(base);
  }
}

// ✅ Thêm 'student' = thêm class StudentPrice + đăng ký vào mảng rules.
//    Code cũ (PriceCalculator, RegularPrice, VipPrice) KHÔNG đổi một dòng -> đúng OCP.
```

> 💡 **Đọc kỹ:** `PriceCalculator` **không biết** có những loại giá nào — nó chỉ biết *"có một danh sách rule, tìm cái khớp rồi gọi"*. Mọi loại giá là **class riêng** (SRP), thêm loại là **thêm class** (OCP). Hai nguyên tắc cùng đạt trong một thiết kế.

---

## ⑥ Keywords cần nhớ

> 🧠 **Cần HIỂU:**
> - **SRP** — một **lý do thay đổi** (một **actor**), không phải "một việc"
> - **OCP** — thiết kế **extension point** tại trục biến động
> - **switch-on-type** là **smell** (mùi code xấu) báo thiếu abstraction
> - **OCP ↔ giảm regression** — thêm code mới an toàn hơn sửa code cũ

> 📌 **Cần THUỘC:**
> - **5 chữ SOLID** đúng tên/nghĩa: **SRP / OCP / LSP / ISP / DIP**.
> - Phát biểu chuẩn **SRP** (một lý do đổi) và **OCP** (mở mở rộng, đóng sửa đổi).

> 🛠️ **Cần LÀM ĐƯỢC:**
> - Tách một **God class** theo **lý-do-thay-đổi**.
> - Khử **`if/else if` type** bằng **Strategy / map**.

---

## ⑦ Mental model

> 🎯 **Khẩu quyết:**
> - **SRP:** mỗi class chỉ có **một "ông chủ"** để phục vụ (một lý do đổi).
> - **OCP:** thêm hành vi bằng **thêm class**, **không cưa lại code đang chạy**.

```
   Class này có MẤY ông chủ?  >1  → tách (SRP)
   Thêm loại = phải MỞ hàm cũ? có → thêm abstraction/Strategy (OCP)
```

---

## ⑧ Câu hỏi phỏng vấn & thách đố

**(dễ)** *SOLID là 5 chữ gì?*
→ **SRP / OCP / LSP / ISP / DIP**.

**(TB)** *SRP thật ra nói về gì? "một class một việc" đúng không?*
→ **Một lý do thay đổi** (một **actor**); "một việc" là **hiểu thô**.

**(TB)** *`UserService` validate + lưu DB + gửi mail vi phạm gì, sửa sao?*
→ Vi phạm **SRP**; **tách collaborator** (Validator / Repository / EmailSender), Service chỉ điều phối.

**(khó)** *OCP đạt nhờ cơ chế OOP nào?*
→ **abstraction / polymorphism**.

> 🧩 **🔥 Thách đố:** *"Để đạt OCP, tạo interface cho MỌI thứ luôn cho chắc?"*
>
> **Bẫy:** **OCP chỉ mở tại trục THỰC SỰ biến động.** Mở mọi thứ = **over-engineering** — đẻ ra **interface 1-1 vô nghĩa** (một interface chỉ có đúng một implementation, chẳng để làm gì — Bài 6/15). Trả lời "interface mọi thứ" là rơi bẫy.

---

## ⑨ Bài tập + tiêu chí tự chấm

**BT1.** Refactor `NotificationService.send(type)` có `if(type==='sms')…else if(type==='email')…` sang **Strategy**.
→ ✅ **Đạt khi:** thêm kênh `'push'` **không sửa** `send`.

> 💡 *Gợi ý:* y hệt ví dụ `PriceRule` ở mục ⑤ — mỗi kênh thành một class implement `NotificationChannel`, `send` chỉ tìm & gọi.

**BT2.** Tách một class trong code bạn có **≥2 lý do thay đổi**.
→ ✅ **Đạt khi:** mỗi class mới **chỉ vỡ vì một loại lý do**.

> 💡 *Gợi ý:* liệt kê "ai có quyền bắt class này đổi" (product? infra? DBA?). Mỗi actor → một class.

---

## ⑩ Đọc thêm

- **R. C. Martin** — **Clean Architecture**, **Agile Software Development** (chương **SOLID**).
- Bài gốc **"The Single Responsibility Principle"** (blog.cleancoder.com) — định nghĩa **"actor"**.

---

## 🤖 Hiểu để chỉ huy (Kiểm tra AI)

> 🚨 **AI rất hay sinh:**
> 1. **God service** — một class ôm validate + DB + mail + log... "vì nó chạy".
> 2. **switch-on-type** — `if (type === ...)` dài dằng dặc, thêm loại là sửa hàm.
>
> 🎯 **Hai câu soát mỗi lần nhận code từ AI:**
> - *"Class này có **mấy lý do để đổi**?"* → >1 → ép **tách theo actor** (SRP).
> - *"Có **`if type`** nào nên thành **polymorphism** không?"* → có → ép đổi sang **Strategy / map handler** (OCP).
>
> AI tối ưu cho "chạy được ngay", nên mặc định gom hết vào một chỗ và rẽ nhánh bằng `if`. Vai trò chỉ huy của bạn: thấy God class thì *tách theo ông chủ*, thấy switch-on-type thì *mở extension point đúng chỗ* — và **không mở dư** (nhớ thách đố ở mục ⑧).
