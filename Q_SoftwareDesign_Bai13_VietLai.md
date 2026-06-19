# 🎯 BÀI 13 — DDD (1): Strategic — **Bounded Context** · **Ubiquitous Language** · **Entity vs Value Object**

**Phủ:** Q-DDD-001 → Q-DDD-002 → Q-DDD-003

> 💡 **Mental model mở đầu:**
> **DDD** không phải "thêm một đống pattern". Phần đáng giá nhất của nó là **strategic** — *trả lời câu hỏi "vẽ ranh giới ở đâu" và "gọi tên mọi thứ thế nào" TRƯỚC khi code*.
> - **Bounded Context** = chia hệ thống thành các "thế giới" có model riêng.
> - **Ubiquitous Language** = ngôn ngữ chung giữa dev và domain expert, *vào tận code*.
> Hai thứ này **hữu ích kể cả khi bạn không làm DDD đầy đủ**.

---

## ① Mục tiêu & vị trí trong mạch

**DDD** (**Eric Evans**, 2003) là cách **mô hình hoá nghiệp vụ phức tạp** trong khung kiến trúc **Bài 11–12**.

```
   Bài 11–12: khung kiến trúc (đảo chiều, tách domain/infra)
                    │  DDD lấp đầy phần DOMAIN
                    ▼
   Bài 13: STRATEGIC  ← bài này
        • Bounded Context · Ubiquitous Language (luôn hữu ích)
        • Entity vs Value Object (building block nền)
                    │
   Bài 14: TACTICAL (aggregate, repository, service, event) + khi nào KHÔNG dùng DDD
```

> 🎯 **Chốt vị trí:** bài này lo phần **strategic** (**Bounded Context, Ubiquitous Language**) — **luôn hữu ích kể cả khi không làm DDD đầy đủ** — và hai **building block** nền: **Entity vs Value Object**. **Bài 14** lo **tactical** + **khi nào KHÔNG dùng DDD**.

---

## ② Giảng cơ bản → nâng cao

### (a) Bounded Context (Q-DDD-001)

> **Ví dụ trực giác:** từ **"Customer"** mang nghĩa **khác nhau** tuỳ phòng ban:
> - Trong **Sales**: là *lead / đơn hàng tiềm năng*.
> - Trong **Billing**: là *tài khoản thanh toán*.
> - Trong **Support**: là *người sở hữu ticket*.
> Cùng một chữ, ba ý nghĩa. Ép cả ba dùng *một* class `Customer` → cái class đó phình to và mâu thuẫn.

**Bounded Context** = **ranh giới trong đó một model/ngôn ngữ NHẤT QUÁN**.

> 🚨 **CẢNH BÁO — "một model duy nhất cho toàn hệ thống" là kỳ vọng SAI:**

```
❌ MỘT MODEL TOÀN CỤC                  ✅ CHIA BOUNDED CONTEXT
   class Customer {                      [Sales]    Customer = lead
     // field cho Sales                  [Billing]  Customer = payment account
     // + field cho Billing              [Support]  Customer = ticket owner
     // + field cho Support              → mỗi context model GỌN, không mâu thuẫn
     ...phình to, mâu thuẫn...
   }
```

> 🎯 **Vì sao chia context lại tốt?** Vì mỗi context **model gọn**, **giảm phức tạp**, và mỗi đội hiểu rõ *Customer trong context của mình* nghĩa là gì. **Đây là strategic DDD — quyết định ranh giới TRƯỚC khi code.**

---

### (b) Ubiquitous Language (Q-DDD-002)

> **Ẩn dụ:** dev và chuyên gia nghiệp vụ nói **hai thứ tiếng** — dev nói "process data", expert nói "đặt đơn hàng". Nếu không có **từ điển chung**, mọi cuộc họp đều là dịch sai. **Ubiquitous Language** là cái từ điển chung đó, *và nó phải vào tận code*.

**Ubiquitous Language** = ngôn ngữ chung dùng **nhất quán** trong **cả trò chuyện và code** giữa **dev** và **domain expert**. Giải quyết vấn đề **dịch sai** giữa **nghiệp vụ ↔ kỹ thuật**.

> 💡 **Hệ quả cụ thể:** tên class/method **phản ánh đúng thuật ngữ domain**:

| ❌ Tên kỹ thuật mơ hồ | ✅ Tên theo Ubiquitous Language |
|---|---|
| `OrderManager.processData()` | `Order.place()` |
| `DataManager`, `Helper` | tên đúng nghiệp vụ |

> 🔎 **Ngôn ngữ này gắn với từng Bounded Context:** cùng "Customer" nhưng nghĩa **theo context**. Không có "ngôn ngữ chung toàn cục", chỉ có "ngôn ngữ chung *trong một context*".

---

### (c) Entity vs Value Object (Q-DDD-003) — phân biệt nền tảng

> **Ví dụ trực giác:**
> - **Bạn** là một **Entity**: đổi tên, đổi địa chỉ, đổi tóc — vẫn là *chính bạn* (có **identity**). Người ta nhận ra bạn qua *bạn là ai*, không qua thuộc tính.
> - **Tờ 100k** là một **Value Object**: hai tờ 100k *hoàn toàn thay thế nhau*, không ai quan tâm "tờ nào". Giá trị mới là tất cả.

| Tiêu chí | **Entity** | **Value Object** |
|---|---|---|
| Định danh | **identity riêng** (ID) | **bằng giá trị** |
| Qua thời gian | **tồn tại** dù thuộc tính đổi | **bất biến** (sửa = tạo mới) |
| Ví dụ | `User#42` (đổi tên/email vẫn là user đó) | `Money(100, 'USD')`, `Address`, `DateRange` |
| So sánh | **bằng ID** | **bằng giá trị** (hai `Money(100,'USD')` là bằng nhau) |

> 🎯 **Tiêu chí phân biệt — hỏi đúng MỘT câu:** **"có cần identity không?"** Cần → **Entity**; không → **Value Object**.

> 🔎 **Nối Bài 10:** biến **Primitive Obsession / Data Clumps** thành **Value Object** — đây chính là building block để làm việc đó.

---

## ③ ⚠️ Cũ → Mới

| Hiểu sai | Đúng | Vì sao |
|---|---|---|
| **"Một model chung cho cả hệ thống"** | **Chia Bounded Context**, model nhất quán **trong ranh giới** | Cùng từ **khác nghĩa** giữa context |
| **Tên kỹ thuật mơ hồ** (`DataManager`, `Helper`) | Tên theo **Ubiquitous Language** domain | Giảm **dịch sai** dev↔expert |
| **"Cứ là object thì như nhau"** | Phân **Entity (identity)** vs **Value Object (giá trị, immutable)** | Quyết cách **so sánh/lưu/sửa** |

---

## ④ Áp dụng thực tế + So sánh bigtech

### Use case: e-commerce chia context

```
   [Catalog]   model "Product" = thông tin trưng bày
   [Ordering]  model "Order"   = giỏ + đặt hàng
   [Billing]   model riêng     = hoá đơn/thanh toán
   [Shipping]  model riêng     = vận đơn
        │
   các context giao tiếp qua ID / event (Bài 14), KHÔNG share model
```

### Value Object thật

| VO | Giải quyết gì |
|---|---|
| **Money** | tránh lỗi **cộng tiền khác currency** |
| **EmailAddress** | **validate một chỗ** (không rải `if(email.includes('@'))` khắp nơi) |

### Bigtech / ecosystem (pattern phổ biến — không tuyệt đối)

| Thực hành | Ghi chú |
|---|---|
| **microservices** | thường ánh xạ **1 service ≈ 1 bounded context** |
| **Context Map** (DDD) | công cụ **vạch ranh giới TRƯỚC khi chia service** |

> 🔎 **Insight sâu:** **strategic DDD hữu ích kể cả khi không làm tactical đầy đủ.** Bạn có thể *không* dùng aggregate/repository pattern (Bài 14), nhưng việc *vẽ Bounded Context đúng* và *thống nhất Ubiquitous Language* đã cứu bạn khỏi phần lớn rắc rối — vì sai ranh giới context là loại sai *đắt nhất để sửa* (phải xé service ra).

---

## ⑤ Code thực hành

```typescript
// ddd-blocks.ts — Entity vs Value Object

// VALUE OBJECT: immutable, so sánh bằng GIÁ TRỊ, validate khi tạo
class Money {
  private constructor(readonly amount: number, readonly currency: string) {}
  static of(amount: number, currency: string): Money {
    if (amount < 0) throw new Error('amount >= 0');                  // validate ngay khi tạo
    return new Money(amount, currency);
  }
  add(other: Money): Money {
    if (other.currency !== this.currency) throw new Error('Currency mismatch');  // INVARIANT: cùng currency
    return Money.of(this.amount + other.amount, this.currency);     // TẠO MỚI, KHÔNG sửa (immutable)
  }
  equals(o: Money) { return this.amount === o.amount && this.currency === o.currency; }  // bằng theo GIÁ TRỊ
}

// ENTITY: có IDENTITY, so sánh bằng ID, thuộc tính đổi vẫn là CÙNG entity
class User {
  constructor(readonly id: string, private name: string) {}
  rename(name: string) { this.name = name; }                        // đổi thuộc tính, VẪN User#id
  equals(o: User) { return this.id === o.id; }                      // bằng theo IDENTITY (ID)
}
```

> 💡 **Đọc kỹ điểm khác biệt:** `Money.add()` **trả về object mới** (immutable — không đụng object cũ), so sánh **bằng amount+currency**. `User.rename()` **sửa tại chỗ** (vẫn cùng identity), so sánh **bằng id**. Cùng là "object" nhưng *bản chất khác nhau* → cách lưu/sửa/so sánh khác nhau.

---

## ⑥ Keywords cần nhớ

> 🧠 **Cần HIỂU:**
> - **Bounded Context**
> - **Ubiquitous Language**
> - **Entity (identity)** vs **Value Object (giá trị, immutable)**
> - **strategic DDD**

> 📌 **Cần THUỘC:**
> - Tiêu chí **Entity/VO** (**"cần identity không?"**).
> - **"một model toàn cục là sai"**.

> 🛠️ **Cần LÀM ĐƯỢC:**
> - Dựng **Value Object immutable** có validation.
> - Đặt tên theo **Ubiquitous Language**.
> - **Vạch context**.

---

## ⑦ Mental model

> 🎯 **Khẩu quyết:**
> Mỗi **Bounded Context** là một **"thế giới" có ngôn ngữ riêng**. **Entity = "ai"** (có identity, sống qua thời gian); **Value Object = "cái gì/bao nhiêu"** (bất biến, bằng nhau theo giá trị).

```
   Gặp một khái niệm?    → nó thuộc CONTEXT nào? (nghĩa đổi theo context)
   Đặt tên class/method? → dùng từ của DOMAIN EXPERT (Order.place, không processData)
   Object này:           → cần biết "là cái nào"? CÓ → Entity · KHÔNG → Value Object
```

---

## ⑧ Câu hỏi phỏng vấn & thách đố

**(khó)** *Bounded Context là gì? Vì sao "một model duy nhất" là sai?*
→ **ranh giới model nhất quán**; **cùng từ khác nghĩa** giữa context.

**(TB)** *Ubiquitous Language giải quyết gì?*
→ **ngôn ngữ chung dev↔expert**, **vào cả code**.

**(khó)** *Entity vs Value Object, tiêu chí phân biệt?*
→ **identity vs giá trị**; **"cần identity không?"**.

> 🧩 **🔥 Thách đố:** *"Money cứ để là `number` cho gọn?"*
>
> **Bẫy:** **primitive obsession** (Bài 10) → **cộng nhầm currency** (cộng 100 USD + 100 JPY ra 200 vô nghĩa), **validate rải rác**. **VO Money gói invariant một chỗ** (chặn cộng khác currency ngay tại `add()`). Trả lời "để number cho gọn" là rơi bẫy.

---

## ⑨ Bài tập + tiêu chí tự chấm

**BT1.** Vạch **2-3 bounded context** cho một domain bạn biết (vd shop), nêu **1 từ mang nghĩa khác** giữa các context.
→ ✅ **Đạt khi:** chỉ ra được **xung đột nghĩa** nếu gộp model.

> 💡 *Gợi ý:* thử từ "Product" — trong Catalog là *trang trưng bày*, trong Inventory là *số lượng tồn kho*, trong Ordering là *dòng trong giỏ*. Gộp lại → một class ôm cả ba, mâu thuẫn.

**BT2.** Biến một cụm primitive (vd `email: string`) thành **Value Object** có validate.
→ ✅ **Đạt khi:** **không tạo được VO sai định dạng**.

---

## ⑩ Đọc thêm

- **Eric Evans** — **Domain-Driven Design** (2003).
- **Vaughn Vernon** — **Implementing DDD**.

---

## 🤖 Hiểu để chỉ huy (Kiểm tra AI)

> 🚨 **AI rất hay:**
> 1. **Gộp model toàn cục** — một `Customer`/`User` ôm field của mọi context.
> 2. **Dùng primitive thay VO** — `string` cho email, `number` cho tiền.
> 3. **Đặt tên kỹ thuật mơ hồ** — `DataManager`, `Helper`, `processData`.
>
> 🎯 **Hai câu soát mỗi lần nhận model từ AI:**
> - *"Tên có theo **ngôn ngữ domain** không, hay là `Manager/Helper/processData`?"* → mơ hồ → ép đặt theo **Ubiquitous Language**.
> - *"Có **VO** nào nên gói không (tiền, email, khoảng thời gian)?"* → có → ép biến **primitive** thành **Value Object** có validate.
>
> AI không biết *nghiệp vụ thật* của bạn nên hay gom mọi thứ vào một model "cho tiện" và đặt tên kỹ thuật. Vai trò chỉ huy của bạn: **mang ranh giới context và ngôn ngữ domain vào** — đó là thứ AI *không thể tự suy ra* từ code.
