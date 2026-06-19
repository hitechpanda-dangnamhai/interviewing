# 🎯 BÀI 10 — **Code Smell** · **Anti-pattern** · **Refactoring**

**Phủ:** Q-SMELL-001 → Q-SMELL-002 → Q-SMELL-003 → Q-SMELL-004 → Q-SMELL-005 → Q-SMELL-006 → Q-SMELL-007 → Q-SMELL-008 → Q-SMELL-009 → Q-SMELL-010

> 💡 **Mental model mở đầu:**
> Bài này là **mặt sau của Bài 1–9**. Bài 1–9 dạy *làm sao cho đúng*; bài này dạy **nhận ra khi nó SAI** — khi vi phạm **coupling/cohesion/SOLID**, bạn **"ngửi" thấy smell**.
> Ba kỹ năng cốt lõi: **đặt tên vấn đề** (để review có vocab chung), **refactor an toàn** (đổi cấu trúc, giữ behavior), và **quyết định khi nào sửa** (smell vs deadline).

---

## ① Mục tiêu & vị trí trong mạch

Đây là **mặt sau của Bài 1–9**: khi vi phạm **coupling / cohesion / SOLID**, bạn **ngửi thấy smell**.

```
   Bài 1–9: làm cho ĐÚNG  ──────────┐
                                    ▼
   Bài 10: nhận ra khi SAI (smell), gọi đúng TÊN, refactor an toàn
                                    │
              ┌─────────────────────┴─────────────────────┐
   cầu nối kiến trúc (Bài 11–12)            judgment Tech Lead (Bài 15)
```

> 🎯 **Chốt vị trí:** bài này cho bạn **vocab chung khi review** (để nói "đây là God Object" thay vì "tôi thấy nó kỳ kỳ") + kỹ thuật **refactor an toàn** (đổi cấu trúc, **giữ behavior**). Là cầu nối tới **kiến trúc** (Bài 11–12) và **judgment Tech Lead** (Bài 15).

---

## ② Giảng cơ bản → nâng cao

### (a) Code smell là gì (Q-SMELL-001)

> **Ẩn dụ:** **mùi** thức ăn hơi lạ trong tủ lạnh. Chưa chắc đã hỏng (chưa phải bug), nhưng là **dấu hiệu** nên kiểm tra trước khi nó thành vấn đề thật.

**Code smell** = **dấu hiệu bề mặt** báo **vấn đề thiết kế tiềm ẩn** — **KHÔNG phải bug**, **không làm sai chức năng**.

> 💡 **Vì sao vẫn phải quan tâm dù không sai chức năng?** Vì smell **báo trước nợ kỹ thuật (technical debt)**: code vẫn chạy hôm nay, nhưng **khó bảo trì / mở rộng** nếu để lâu. Tệ nhất là bạn *quen mùi* rồi bỏ mặc, tới lúc cần sửa gấp thì cả vùng đã mục.

---

### (b) Catalog smell hay gặp

#### **God Object / God Class** (Q-SMELL-002)

> **Ví dụ trực giác:** một nhân viên *ôm hết việc* cả công ty — kế toán, IT, bảo vệ, pha cà phê. Người đó nghỉ một hôm là cả công ty tê liệt.

Một class **ôm quá nhiều trách nhiệm / biết quá nhiều** → vi phạm **SRP**, **cohesion thấp + coupling cao**; **mọi thay đổi đụng nó** → rủi ro. **Sửa: tách trách nhiệm.**

#### **Feature Envy** (Q-SMELL-003)

> **Ví dụ trực giác:** một method cứ "thèm" dữ liệu của nhà hàng xóm — toàn gọi `other.x`, `other.y`, `other.z` rồi tự tính. Nó nên *dọn sang nhà hàng xóm* mà ở.

**method quan tâm dữ liệu/method của class KHÁC hơn class của chính nó** → **method đặt sai chỗ**. **Refactor: move method** sang class **sở hữu dữ liệu**.

#### **Divergent Change vs Shotgun Surgery** (Q-SMELL-004) — ngược phương

> 🔎 **Hai smell NGƯỢC NHAU** — học cùng nhau để khỏi nhầm:

| Smell | Hiện tượng | Gốc bệnh | Cách trị |
|---|---|---|---|
| **Divergent Change** | **MỘT class đổi vì NHIỀU lý do** khác nhau | thiếu **SRP** | **TÁCH** ra |
| **Shotgun Surgery** | **MỘT thay đổi buộc sửa RẢI RÁC nhiều class** | **coupling cao** / thiếu gom | **GỘP** lại |

> 💡 **Cách nhớ:** Divergent = *một chỗ bị giật từ nhiều phía* → xé ra. Shotgun = *một phát đạn ghém bắn tứ tung* → gom lại. Hai smell, hai hướng refactor đối nghịch.

#### **Primitive Obsession & Data Clumps** (Q-SMELL-005)

> **Ví dụ trực giác:** dùng **`string`** cho email (ai cũng có thể truyền chuỗi rác `"abc"`), dùng `number` rời cho tiền (mất đơn vị tiền tệ). Và **lat/long/alt** lúc nào cũng đi cùng nhau thành bộ ba lẻ.

- **Primitive Obsession** = **lạm dụng kiểu nguyên thuỷ** thay **kiểu domain** (`string` cho email/tiền).
- **Data Clumps** = nhóm tham số **luôn đi cùng** (lat/long/alt).

> 🎯 **Sửa: tạo Value Object / object gói → encapsulate validation** (nối **DDD — Bài 13**). Gói lại để *không thể* tạo ra ở trạng thái sai.

#### **Long Method / Deep Nesting / section-comment** (Q-SMELL-006)

method **200 dòng**, nhiều cấp lồng + comment chia section ("// === phần tính thuế ===").

> 🎯 **Refactor:** **extract method** theo block/mức abstraction, **guard clause** giảm nesting, đặt **tên ý nghĩa**.
> 🚨 **Phải có test TRƯỚC + giữ behavior** (xem mục c).

---

### (c) Refactoring chặt (theo Fowler — Q-SMELL-007)

> 🎯 **Định nghĩa CHẶT:** **thay đổi cấu trúc nội bộ mà KHÔNG đổi hành vi quan sát được.**

```
   [code cũ, behavior X]  ──refactor──▶  [code mới gọn hơn, behavior VẪN X]
                              │
                   TEST là LƯỚI AN TOÀN
              chứng minh behavior trước = sau
```

> 🚨 **CẢNH BÁO — phân biệt sống còn:** **"refactor" (giữ behavior) ≠ "sửa kèm thêm tính năng / đổi behavior".**
> **Điều tệ nhất khi trộn hai việc:** lúc review không biết thay đổi nào là "dọn dẹp" và thay đổi nào "đổi logic" → khó review, và khi cần **rollback** thì gỡ một phần là vỡ phần kia. **Tách PR riêng.**

---

### (d) Anti-pattern vs code smell (Q-SMELL-008)

| | **Code smell** | **Anti-pattern** |
|---|---|---|
| Là gì | **dấu hiệu** (triệu chứng) | một **"giải pháp" phổ biến nhưng phản tác dụng** |

**Ví dụ anti-pattern backend:**

| Anti-pattern | Mô tả |
|---|---|
| **Golden Hammer** | cái gì cũng dùng **1 công cụ quen** ("tôi biết Mongo nên mọi thứ là Mongo") |
| **Lava Flow** | **code chết** không ai dám xoá (sợ vỡ thứ không rõ) |
| **Spaghetti** | luồng rối như mì, không lần được |
| **Magic Numbers/Strings** | số/chuỗi "thần kỳ" rải rác không tên (`if (status === 3)`) |
| **premature optimization** | tối ưu sớm khi chưa đo, chưa cần |

> 💡 **Vì sao anti-pattern nguy hiểm hơn smell?** Vì chúng **"có vẻ đúng"** — giải quyết được *trước mắt* nên dễ được chấp nhận, nhưng **tạo nợ về sau**. Smell là *triệu chứng*; anti-pattern là *thói quen sai được lặp lại*.

---

### (e) ⚠️ Lạm dụng design pattern cũng là smell (Q-SMELL-009)

> **Ví dụ trực giác:** dùng dao mổ trâu để gọt táo. Dao tốt, nhưng *sai chỗ* thành phiền.

**"Pattern đúng nhưng dùng sai chỗ":**

| Lạm dụng | Hậu quả |
|---|---|
| **Decorator chồng quá sâu** | chuỗi object **khó debug / overhead** |
| **Singleton** | tạo **global state** (Bài 7) |
| **Factory** cho thứ chỉ **`new` một lần** | indirection thừa |

> 🎯 **Chốt:** **pattern là công cụ CÓ CHI PHÍ, không phải huy chương.** Mỗi pattern thêm indirection — chỉ "trả phí" đó khi nó giải một vấn đề cụ thể.

---

### (f) Smell vs deadline — quyết định Tech Lead (Q-SMELL-010)

> **Tình huống:** thấy smell nhưng **deadline gấp**. Sửa hay không?

**Phân loại theo 3 trục:**

| Trục | Hỏi gì |
|---|---|
| **rủi ro lan** | smell này có lan ra nhiều chỗ không? |
| **tần suất đụng** | vùng code này có hay bị sửa không? |
| **độ rủi ro thay đổi** | sửa nó bây giờ có dễ vỡ không? |

→ quyết định: **fix-now** / **log nợ** / **bỏ qua có chủ đích**.

> 🎯 **Nguyên tắc:** **log nợ tường minh** (ticket/TODO **có chủ + lý do**), không sửa lén.
> 🚨 **Tránh "boy-scout" phá scope PR** — sửa lan man ngoài mục tiêu khiến PR phình to, khó review. **Cân chất lượng vs tốc độ giao.**

---

## ③ ⚠️ Cũ → Mới / Lưu ý

| Hiểu sai | Đúng | Vì sao |
|---|---|---|
| **"Smell = bug"** | **Smell = dấu hiệu thiết kế**, không sai chức năng | Quản **nợ kỹ thuật**, không phải sửa lỗi gấp |
| **Refactor tiện thể thêm feature** | **Refactor = giữ behavior**; tách PR feature riêng | Review/rollback rõ ràng |
| **Pattern càng nhiều càng "pro"** | **Pattern có chi phí**; lạm dụng = smell | **Indirection/overhead** vô ích |
| **Refactor không cần test** | **Test là điều kiện bắt buộc** (lưới an toàn) | Chứng minh **behavior không đổi** |

---

## ④ Áp dụng thực tế + So sánh bigtech

### Use case: refactor `OrderProcessor` 400 dòng

```
T1  Viết CHARACTERIZATION TEST (chụp lại behavior HIỆN TẠI)  ✅ lưới an toàn
T2  Extract method TỪNG BƯỚC
T3  Chạy test SAU MỖI BƯỚC  → xanh → behavior không đổi → đi tiếp
                            → đỏ  → hoàn tác bước vừa rồi
```

> 💡 **Vì sao viết test TRƯỚC khi refactor một mớ hỗn độn?** Vì bạn *chưa chắc hiểu* code 400 dòng đó làm gì. **Characterization test** "chụp ảnh" hành vi hiện tại (kể cả hành vi kỳ quặc) → sau đó bạn dọn dẹp mà *biết chắc* không làm đổi gì.

### Bigtech / ecosystem (pattern phổ biến — không tuyệt đối)

| Thực hành | Mô tả |
|---|---|
| **Google "readability review"** + **linters** | bắt smell tự động |
| **"tech debt register"** (sổ nợ kỹ thuật) | log nợ **có chủ đích** thay vì sửa bừa |

> 🔎 **Insight sâu:** bigtech **không cấm** nợ kỹ thuật — họ **ghi sổ** nó. Nợ kỹ thuật giống nợ tài chính: vay có kế hoạch (log tường minh) thì ổn; vay lén lút không ghi chép thì vỡ nợ. **"tech debt register"** biến nợ ẩn thành nợ *quản trị được*.

---

## ⑤ Code thực hành

```typescript
// refactor-long-method.ts — guard clause + extract method (GIỮ behavior)

// ❌ TRƯỚC: long method, deep nesting, section comments
function processOrderBad(o: any) {
  if (o) {
    if (o.items.length > 0) {
      if (o.paid) {
        // ... 50 dòng tính toán ...   ← lồng 3 cấp, khó đọc
      }
    }
  }
}

// ✅ SAU: guard clause (giảm nesting) + extract method (đặt tên theo ý nghĩa)
function processOrder(o: Order) {
  if (!o) throw new Error('Order required');            // guard clause: thoát sớm
  if (o.items.length === 0) throw new Error('Empty');   // guard clause
  if (!o.paid) throw new Error('Unpaid');               // → hết lồng, đọc tuyến tính

  const subtotal = calcSubtotal(o.items);               // extract method (tên rõ nghĩa)
  const tax = calcTax(subtotal);
  return subtotal + tax;
}
function calcSubtotal(items: Item[]) { return items.reduce((s, i) => s + i.price, 0); }
function calcTax(subtotal: number) { return subtotal * 0.1; }

// --- Primitive Obsession -> Value Object ---
// ❌ function pay(amount: number, currency: string) {}   // hai primitive luôn đi cặp = DATA CLUMP
class Money {                                             // ✅ VALUE OBJECT: gói + validate
  private constructor(readonly amount: number, readonly currency: string) {}
  static of(amount: number, currency: string) {
    if (amount < 0) throw new Error('amount >= 0');       // ENCAPSULATE validation: không tạo được Money âm
    return new Money(amount, currency);
  }
}
```

> 💡 **Đọc kỹ:** `guard clause` biến *"if lồng vào trong"* thành *"thoát sớm rồi đi tiếp"* → code đọc tuyến tính, hết deep nesting. `Money` gói `amount + currency` (vốn là Data Clump) lại + chặn trạng thái âm ngay tại `of()` — đúng tinh thần encapsulation (Bài 1).

---

## ⑥ Keywords cần nhớ

> 🧠 **Cần HIỂU:**
> - **smell** (**≠ bug**)
> - **God Object**
> - **Feature Envy**
> - **Divergent Change vs Shotgun Surgery**
> - **Primitive Obsession / Data Clumps**
> - **refactoring (Fowler)**
> - **anti-pattern**
> - **pattern-abuse**

> 📌 **Cần THUỘC:**
> - Định nghĩa **refactoring** (giữ behavior + cần test).
> - **Divergent (tách) vs Shotgun (gộp)**.

> 🛠️ **Cần LÀM ĐƯỢC:**
> - **extract method + guard clause**.
> - Biến **Data Clump** thành **Value Object**.
> - **log nợ kỹ thuật** có chủ đích.

---

## ⑦ Mental model

> 🎯 **Khẩu quyết:**
> - **Smell = mùi báo nợ, không phải lửa cháy** (không phải bug).
> - **Refactor = đổi bên trong, giữ bên ngoài, có test làm lưới.**
> - **Pattern là công cụ có giá, không phải huy chương.**

```
   Thấy "kỳ kỳ"?     → gọi đúng TÊN smell (vocab review chung)
   Muốn dọn dẹp?     → viết test TRƯỚC, refactor, test SAU (giữ behavior)
   Muốn thêm pattern?→ hỏi "nó giải vấn đề gì?" không có → ĐỪNG
   Smell + deadline? → phân loại rủi ro → fix-now / log nợ / bỏ có chủ đích
```

---

## ⑧ Câu hỏi phỏng vấn & thách đố

**(TB)** *Code smell là gì, có phải bug không?*
→ **dấu hiệu thiết kế**, **không sai chức năng**.

**(TB)** *God Object nguy hiểm sao, vi phạm gì?*
→ **ôm nhiều trách nhiệm**; vi phạm **SRP / cohesion / coupling**.

**(khó)** *Divergent Change vs Shotgun Surgery?*
→ **một class đổi nhiều lý do (tách)** vs **một đổi sửa nhiều class (gộp)**.

**(khó)** *Định nghĩa chặt refactoring? Vì sao cần test? Có được đổi behavior?*
→ **đổi cấu trúc, giữ behavior**; **test = lưới**; **không** đổi behavior.

> 🧩 **🔥 Thách đố (Q-SMELL-009):** *"Dự án nên dùng nhiều design pattern để chứng tỏ chất lượng?"*
>
> **Bẫy:** **lạm dụng pattern = smell** (**overhead / indirection**); pattern phải **giải vấn đề cụ thể**. **Singleton/Decorator/Factory sai chỗ gây hại.** Trả lời "càng nhiều càng pro" là rơi bẫy — pattern là *công cụ có giá*, không phải *huy chương*.

---

## ⑨ Bài tập + tiêu chí tự chấm

**BT1.** Lấy **method dài nhất** trong code bạn, viết **test characterization** rồi **extract method + guard clause**.
→ ✅ **Đạt khi:** **test xanh trước/sau**, **không đổi behavior**, **nesting giảm**.

**BT2.** Tìm **1 Data Clump** (vd `lat, lng` luôn đi cùng), gói thành **Value Object** có validation.
→ ✅ **Đạt khi:** **không tạo được VO ở trạng thái sai**.

---

## ⑩ Đọc thêm

- **Martin Fowler** — **Refactoring** (2nd ed.) + **catalog smell** tại refactoring.com.
- **AntiPatterns** (Brown et al.) — **Golden Hammer**, **Lava Flow**…

---

## 🤖 Hiểu để chỉ huy (Kiểm tra AI)

> 🚨 **AI rất hay sinh:**
> 1. **long method**, **God service**, **magic number**, **primitive obsession** "vì chạy được".
> 2. Khi "refactor", có thể **đổi behavior lén** (vừa dọn vừa sửa logic).
>
> 🎯 **Ba câu soát mỗi lần nhận code/refactor từ AI:**
> - *"Có **test TRƯỚC** không?"* → không → bắt viết **characterization test** trước khi cho dọn.
> - *"**behavior** có đổi không, hay AI lén thêm/sửa logic?"* → so diff kỹ; tách phần đổi behavior ra.
> - *"**smell** nào đang xuất hiện?"* → gọi tên (God service? long method? primitive obsession?) → yêu cầu sửa đúng cái đó.
>
> AI tối ưu "chạy được", nên hay đẻ smell và đôi khi "tiện tay" đổi behavior khi được nhờ refactor. Vai trò chỉ huy của bạn: **giữ behavior bất biến** (test làm chứng) và **gọi đúng tên smell** để sửa có mục tiêu, không sửa lan man.
