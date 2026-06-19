# 🎯 BÀI 1 — Hai thước đo gốc: **Coupling** & **Cohesion** · **Encapsulation** & **Abstraction**

**Phủ:** Q-OOP-001 → Q-OOP-004 → Q-OOP-005

> 💡 **Mental model mở đầu (đọc trước, ngẫm sau):**
> Mọi thứ trong thiết kế phần mềm rốt cuộc chỉ trả lời hai câu hỏi:
> 1. *"Đụng vào chỗ này thì lan ra bao nhiêu chỗ khác?"* → đó là **coupling**.
> 2. *"Những thứ nằm chung một chỗ có thật sự thuộc về nhau không?"* → đó là **cohesion**.
> Học xong bài này, bạn có **một cái la bàn** để chấm điểm bất kỳ thiết kế nào — kể cả code do AI sinh ra.

---

## ① Mục tiêu & vị trí trong mạch

Đây là **bài nền tảng nhất** của cả mục Q. Hãy hình dung toàn bộ kiến thức thiết kế phần mềm như một cái cây:

```
        loose coupling + high cohesion   ← GỐC RỄ (Bài 1, bài này)
                    │
        ┌───────────┼───────────┐
      SOLID       GoF        Kiến trúc
   (Bài 4...)  (patterns)   (microservice...)
```

> 🎯 **Chốt vị trí:** **SOLID**, các **design pattern** (**GoF**), mọi quyết định kiến trúc sau này **không phải mục tiêu** — chúng chỉ là **phương tiện** để đạt một mục tiêu duy nhất: **loose coupling + high cohesion**. Bài này đứng *trước* **SOLID** vì **SOLID** chính là cách *hệ thống hoá* hai thước đo gốc ở đây.

**Vì sao học hai thước đo này trước tất cả?**
Vì nếu bạn thuộc lòng 5 chữ cái SOLID nhưng không hiểu *tại sao* chúng tồn tại, bạn sẽ áp dụng máy móc và tạo ra thiết kế "đúng sách nhưng tệ". Ngược lại, khi đã có la bàn coupling/cohesion, bạn tự *suy ra* được SOLID, và quan trọng hơn: bạn biết **khi nào nên phá luật**.

---

## ② Giảng cơ bản → nâng cao

### (a) Trực giác trước

> **Ẩn dụ — Coupling là dây điện giữa các hộp.**
> **Coupling** (độ ghép) = mức một module **phụ thuộc/biết về** module khác. Tưởng tượng mỗi module là một cái hộp, mỗi phụ thuộc là một sợi **dây điện** nối ra ngoài. Càng nhiều dây, **đụng một hộp là giật cả mớ** — sửa một chỗ, vỡ mười chỗ.

> **Ẩn dụ — Cohesion là cái hộp dụng cụ.**
> **Cohesion** (độ cố kết) = mức các phần *bên trong* một module thật sự **thuộc về nhau**, cùng phục vụ một mục đích.
> - **High cohesion** = hộp toàn cờ-lê đúng bộ → mở ra là biết ngay dùng làm gì.
> - **Low cohesion** = hộp lẫn cờ-lê, bánh mì, hoá đơn tiền điện → mở ra hoang mang, chẳng biết hộp này "để làm gì".

**Mục tiêu cuối cùng — "loose coupling, high cohesion":**

| Khi đạt được | Hệ quả tốt | Vì sao |
|---|---|---|
| **loose coupling** (ít dây) | đổi một chỗ thì **ít lan** | ít module khác "biết" về bạn nên ít chỗ phải sửa theo |
| **high cohesion** (đúng bộ) | mỗi module **dễ hiểu / dễ test / dễ tái dùng** | nó làm đúng *một nhóm việc gắn kết*, không ôm đồm tạp nham |

> 🎯 **Chốt trực giác:** **loose coupling** giúp bạn *thay đổi an toàn*; **high cohesion** giúp bạn *hiểu và tin tưởng* từng mảnh. Một cái lo "lan ra ngoài", một cái lo "rối bên trong".

---

### (b) Cơ chế chi tiết — đào sâu "vì sao"

#### 🔍 Coupling có nhiều CẤP — và giảm coupling = leo lên thang abstraction

Không phải "có coupling" hay "không coupling", mà là coupling **chặt cỡ nào**. Đây là cái thang từ chặt nhất → lỏng nhất:

```
CHẶT NHẤT  ┌─────────────────────────────────────────────┐
   │       │ ① Phụ thuộc kiểu CỤ THỂ                      │
   │       │    A gọi thẳng `new StripeClient()`          │
   │       │    → A "biết" Stripe tồn tại, biết cả tên class│
   ▼       ├─────────────────────────────────────────────┤
           │ ② Phụ thuộc ABSTRACTION / interface          │
           │    A chỉ biết `PaymentGateway` (interface)    │
           │    → A không quan tâm bên dưới là Stripe hay Adyen│
           ├─────────────────────────────────────────────┤
   ▲       │ ③ Giao tiếp qua MESSAGE / EVENT               │
   │       │    A bắn event "PaymentRequested", ai xử lý kệ│
LỎNG NHẤT  │    → A thậm chí không biết B tồn tại          │
           └─────────────────────────────────────────────┘
```

> 💡 **Vì sao leo thang lại lỏng hơn?**
> Vì mỗi nấc thang **giảm lượng kiến thức** mà module A phải nắm về thế giới bên ngoài. Ở nấc ①, A biết *chính xác* class nào, đổi class là A vỡ. Ở nấc ③, A chỉ ném ra một thông điệp rồi quên — đổi cả hệ thống xử lý phía sau, A vẫn không hề hấn gì.
> **Giảm coupling = leo lên thang abstraction**, tức là cố tình "biết ít đi".

⚠️ **Nhưng leo cao có giá của nó:** mỗi nấc thêm một lớp **indirection** (gián tiếp). Event-driven cực lỏng nhưng khó debug (luồng không còn tuyến tính). Sẽ nói kỹ ở mục ③ và ⑧ — đây là cái bẫy "tách càng nhiều càng tốt".

#### 🔍 Cohesion xét theo "lý do tồn tại chung"

Câu hỏi để chấm cohesion của một class: **"Các method trong class này có cùng phục vụ MỘT trách nhiệm không?"**

> **Ví dụ trực giác — class `UserService` low cohesion:**
> ```
> class UserService {
>   createUser()        // ✅ liên quan user
>   resetPassword()     // ✅ liên quan user
>   sendMarketingEmail()// ❓ gửi mail... có thuộc về user không?
>   generatePdfReport() // ❌ xuất PDF — sao lại nằm đây???
>   connectToDatabase() // ❌ hạ tầng — lạc đề hoàn toàn
> }
> ```
> Mở class này ra, bạn không trả lời được "nó *để làm gì*" trong một câu → **cohesion thấp**.

> 🎯 **Đây chính là cầu nối tới SRP (Single Responsibility Principle) ở Bài 4.** SRP về bản chất chỉ là "hãy giữ cohesion cao": một class chỉ nên có *một lý do để thay đổi*. Giữ cohesion = thực thi SRP.

#### 🔍 Hai thước đo ĐỐI NGẪU nhưng KHÔNG tự động đánh đổi

Người mới hay tưởng "kéo coupling xuống thì phải hi sinh cohesion" — **sai**.

| Loại thiết kế | Coupling | Cohesion | Ví dụ điển hình |
|---|---|---|---|
| ✅ **Thiết kế tốt** | thấp | cao | module `payments/` gọn gàng, chỉ phụ thuộc interface |
| ❌ **Thiết kế tệ** | cao | thấp | **God Object** — ôm mọi thứ + dính tới mọi nơi (Bài 10) |

> 💡 **Vì sao chúng đi cùng chiều ở thiết kế tốt?**
> Khi bạn gom đúng những thứ thuộc về nhau (cohesion↑), tự nhiên các phần đó nói chuyện *với nhau bên trong* nhiều hơn là *thò ra ngoài* — nên số dây nối ra ngoài giảm (coupling↓). Cohesion cao **bịt bớt** nhu cầu coupling ra ngoài. Đó là lý do thiết kế tốt kéo cả hai về phía có lợi *cùng lúc*.

---

### (c) Encapsulation vs Abstraction — cặp khái niệm hay bị gộp

Đây là chỗ **AI và cả engineer lâu năm hay nhầm**. Phải tách bạch.

> **Ẩn dụ — chiếc máy ATM.**
> - **Abstraction** = cái màn hình và mấy nút bấm ("Rút tiền", "Kiểm tra số dư"). Nó cho bạn biết **làm được CÁI GÌ**, giấu đi chuyện bên trong tiền được đếm, sổ cái được ghi *NHƯ THẾ NÀO*.
> - **Encapsulation** = cái **vỏ thép khoá kín** quanh khay tiền. Nó **bảo vệ** ruột máy, không cho bạn thò tay vào tự cộng thêm số dư.

**Định nghĩa chuẩn:**

- **Encapsulation** (đóng gói) = **che giấu & bảo vệ trạng thái nội bộ**, chỉ cho phép thay đổi qua **hành vi hợp lệ** → mục đích là **giữ invariant** (bất biến — điều kiện luôn-phải-đúng của object).

  > 🚨 **CẢNH BÁO — hiểu lầm chết người:**
  > *"Để field `private` rồi sinh `get/set` cho mọi field"* **KHÔNG phải encapsulation**.
  > Đó là **phơi state qua cửa sau**: bạn khoá cửa trước (private) nhưng mở toang cửa sau (setter công khai). Kết quả là mất sạch khả năng bảo vệ **invariant** → đẻ ra **anemic model** (model thiếu máu — chỉ có data, không có hành vi bảo vệ, sẽ học ở Bài 12).

- **Abstraction** (trừu tượng hoá) = **phơi ra "cái gì"** (interface/khái niệm), **giấu "như thế nào"** (implementation).

> 🎯 **Một câu phân biệt — học thuộc câu này:**
> **Abstraction quyết định CÁI GÌ lộ ra; Encapsulation bảo vệ cái được GIẤU ĐI.**
> Hai mặt của cùng một đồng xu, nhưng **không đồng nhất**: một cái lo *giao diện*, một cái lo *bất khả xâm phạm*.

#### ➕ Nâng cao — phép thử encapsulation thật

> 🔎 **Encapsulation thật được đo bằng đúng một câu hỏi:**
> **"Từ BÊN NGOÀI, có thể đặt object vào trạng thái KHÔNG hợp lệ không?"**
> - Nếu **CÓ** → **encapsulation bị rò** (dù cú pháp `private` đầy đủ).
> - Nếu **KHÔNG** → encapsulation thật.

**Ví dụ cụ thể (tài khoản ngân hàng):**

```
❌ account.balance = -100   // chạy được  → encapsulation HỎNG (số dư âm là trạng thái sai)
✅ account.withdraw(amount) // buộc qua kiểm tra số dư → invariant "balance >= 0" được giữ
```

Số dư âm là một trạng thái **vô nghĩa về nghiệp vụ**. Nếu hệ thống cho phép tồn tại, sớm muộn nó sẽ rò ra báo cáo tài chính, ra API, ra mắt khách hàng. **Điều tệ nhất khi làm sai:** bug không nằm ở chỗ gán `-100`, mà nổ ở một nơi *cách đó 50 file*, lúc 2 giờ sáng. Encapsulation tốt = bắt lỗi *ngay tại nguồn*.

---

## ③ ⚠️ Kiến thức cũ / bị hiểu sai

| Cách hiểu cũ/sai phổ biến | Hiểu đúng hiện nay | Vì sao |
|---|---|---|
| **"Encapsulation = để biến `private`"** | **Encapsulation** = ẩn state **+ bảo vệ invariant** qua hành vi | `private` + đủ bộ getter/setter ⇒ vẫn **phơi state**, mất bảo vệ. Khoá cửa trước mà mở cửa sau thì cũng như không khoá |
| **"Abstraction = Encapsulation"** | **Abstraction** phơi *cái gì*; **Encapsulation** giấu *như thế nào* | Gộp hai khái niệm làm **mờ mục tiêu thiết kế** — bạn không còn biết mình đang thiết kế *giao diện* hay đang *bảo vệ ruột* |
| **"Coupling thấp luôn tốt, càng tách càng hay"** | **Tách đúng chỗ**; tách quá ⇒ **distributed / over-abstraction** | **coupling = 0** tuyệt đối là **không thể & vô dụng** (module không nói chuyện được với ai thì để làm gì?). Mục tiêu là **loose**, không phải **no** |

> ⚠️ **Lưu ý chữ "loose" vs "no":** Mục tiêu tiếng Anh là *loose* coupling — **lỏng**, không phải **không**. Một hệ thống mà các phần hoàn toàn không biết nhau thì không phải hệ thống, mà là một đống rời rạc. Bạn muốn các phần *biết về nhau ít nhất có thể để vẫn hợp tác được* — không hơn.

---

## ④ Áp dụng thực tế + So sánh bigtech

### Use case: Module thanh toán `payments/`

```
                  ❌ COUPLING CHẶT                       ✅ COUPLING LỎNG
   ┌────────────────────────────┐        ┌────────────────────────────────────┐
   │ business logic             │        │ business logic                     │
   │      │ phụ thuộc thẳng      │        │      │ phụ thuộc INTERFACE         │
   │      ▼                      │        │      ▼                            │
   │  StripeClient (cụ thể)      │        │  PaymentGateway (abstraction)      │
   └────────────────────────────┘        │      ▲          ▲                   │
   Đổi Stripe→Adyen = sửa business        │  StripeImpl   AdyenImpl            │
   logic, vỡ test, vỡ mọi nơi gọi tới     └────────────────────────────────────┘
                                          Đổi Stripe→Adyen = thêm 1 file impl,
                                          business logic KHÔNG đụng tới
```

> 💡 **Vì sao cái phải (lỏng) thắng?** Vì **lý do thay đổi của "đổi nhà cung cấp thanh toán"** bị **cô lập** vào đúng một file `AdyenImpl`. Business logic không có lý do gì để biết tên "Stripe" → nó **miễn nhiễm** với việc đổi vendor. Đây là **loose coupling** trong thực chiến.

Đồng thời, *bên trong* module `payments/`, ta gom **đúng** những thứ liên quan thanh toán (tính phí, gọi gateway, ghi giao dịch) → **high cohesion**. **KHÔNG** nhét logic gửi mail marketing vào đây (đó là việc của module khác).

### So sánh bigtech (pattern phổ biến — không tuyệt đối)

| Khía cạnh | Cách bigtech làm | Ánh xạ về thước đo gốc |
|---|---|---|
| **Tổ chức code/service** | xoay quanh **bounded context** nghiệp vụ (giỏ hàng, thanh toán, kho...) | mỗi context gom đúng việc của mình → **cohesion cao** |
| **Giao tiếp giữa service** | qua **API / event**, không gọi thẳng vào ruột nhau | **coupling lỏng** ở cấp hệ thống |
| **Tổ chức đội** | **"two-pizza team"** của Amazon (đội nhỏ đủ ăn 2 pizza) | thực chất là **cohesion ở cấp tổ chức** — một đội nhỏ **sở hữu trọn một domain** |

> 🔎 **Insight sâu:** *"Two-pizza team"* nghe như chuyện quản lý nhân sự, nhưng nó là **coupling/cohesion áp lên con người**. Đội nhỏ + sở hữu trọn một domain = ít phải họp với đội khác (coupling thấp) + ai cũng hiểu sâu domain của mình (cohesion cao). **Conway's Law:** cấu trúc tổ chức sẽ in dấu lên cấu trúc hệ thống — nên người ta thiết kế *đội* để có được *kiến trúc* mong muốn.

---

## ⑤ Code thực hành + cấu hình

> Ngôn ngữ theo bản gốc: **TypeScript 5.x** (cú pháp ổn định; ví dụ này không cần flag decorator).

```typescript
// file: account.ts  — minh hoạ ENCAPSULATION THẬT (bảo vệ invariant), không chỉ là private

export class Account {
  // #balance là TRUE PRIVATE (ECMAScript private field).
  // Dấu # khác hẳn từ khoá `private` của TS: ngoài class KHÔNG cách nào truy cập được,
  // kể cả lúc runtime → đây là rào chắn thật, không phải "private chỉ kiểm tra lúc compile".
  #balance: number;

  constructor(initial: number) {
    // ✅ Chặn trạng thái KHÔNG hợp lệ NGAY TỪ ĐẦU — object chưa kịp ra đời ở trạng thái sai.
    if (initial < 0) throw new Error('Initial balance must be >= 0');
    this.#balance = initial;
  }

  // Hành vi HỢP LỆ duy nhất để giảm số dư.
  // Invariant "balance >= 0" được BẢO VỆ NGAY TẠI ĐÂY — không nơi nào khác lọt qua được.
  withdraw(amount: number): void {
    if (amount <= 0) throw new Error('Amount must be positive');           // chặn input rác
    if (amount > this.#balance) throw new Error('Insufficient funds');     // ✅ bảo vệ invariant
    this.#balance -= amount;
  }

  deposit(amount: number): void {
    if (amount <= 0) throw new Error('Amount must be positive');
    this.#balance += amount;
  }

  // ABSTRACTION: lộ "CÁI GÌ" (số dư) ở dạng READ-ONLY.
  // Chỉ có getter, KHÔNG có setter → bên ngoài đọc được nhưng KHÔNG set tuỳ tiện.
  get balance(): number {
    return this.#balance;
  }
}

// ✅ acc.withdraw(50)   -> hợp lệ, qua kiểm tra số dư
// ❌ acc.balance = -100 -> LỖI COMPILE (chỉ có getter) => encapsulation giữ được invariant
```

**Phép thử nhanh trên code này:**

```
Câu hỏi: "Từ ngoài có đặt được balance âm không?"
   acc.#balance = -100   → ❌ SyntaxError, # field bất khả xâm phạm
   acc.balance  = -100   → ❌ chỉ có getter, không gán được
   acc.withdraw(99999)   → ❌ throw 'Insufficient funds'
=> KHÔNG có đường nào tạo trạng thái sai  ✅ encapsulation THẬT
```

**Cấu hình:** không cần gì đặc biệt — `#field` là cú pháp **JS chuẩn**, chạy trên **Node hiện đại**. Không hardcode secret. *(Nếu chạy TS với target rất cũ, hãy verify rằng `#private` được giữ nguyên thay vì hạ cấp về WeakMap — vẫn đúng về mặt ngữ nghĩa, chỉ khác cách biên dịch.)*

> 🚨 **Phản ví dụ "anemic" (cần tránh):** nếu thay `#balance` private + getter bằng `public balance: number` (hoặc đủ bộ getter/setter), thì `acc.balance = -100` **chạy ngon lành** → **encapsulation hỏng**, dẫn thẳng tới **anemic model** (Bài 12). Cú pháp đúng *không* đồng nghĩa với thiết kế đúng.

---

## ⑥ Keywords cần nhớ

> 🧠 **Cần HIỂU** (nắm bản chất, giải thích được):
> - **coupling** — độ một module phụ thuộc/biết về module khác (số "dây" nối ra ngoài)
> - **cohesion** — độ các phần bên trong module thật sự thuộc về nhau (độ "đúng bộ")
> - **"loose coupling + high cohesion"** — mục tiêu tối thượng mọi nguyên tắc/pattern hướng tới
> - **abstraction vs encapsulation** — phơi *cái gì* vs giấu & bảo vệ *như thế nào*
> - **invariant** — điều kiện luôn-phải-đúng của object (vd: `balance >= 0`)

> 📌 **Cần THUỘC** (nói trôi chảy, không vấp):
> - Định nghĩa **1 câu** của **coupling** / **cohesion**.
> - Câu phân biệt: **abstraction** = *phơi cái gì*; **encapsulation** = *giấu như thế nào + bảo vệ state*.

> 🛠️ **Cần LÀM ĐƯỢC** (kỹ năng thực chiến):
> - Viết một class **bảo vệ invariant** (không cho rơi vào trạng thái sai từ bên ngoài).
> - Nhìn một class là nói được nó **cohesion** cao/thấp và đang **coupling** chặt với ai.

---

## ⑦ Mental model

> 🎯 **Khẩu quyết để khắc vào đầu:**
> **Coupling = số dây nối ra ngoài. Cohesion = độ "đúng bộ" bên trong.**
> Mọi **pattern** / **nguyên tắc** học sau này chỉ là cách **kéo dây ra** (coupling↓) và **gom đúng bộ** (cohesion↑). Không có ngoại lệ.

```
        ┌──────────────┐  dây ít  ┌──────────────┐
        │   MODULE A   │─────────▶│   MODULE B   │
        │ (toàn cờ-lê  │          │ (toàn tua-vít│
        │  đúng bộ)    │          │  đúng bộ)    │
        └──────────────┘          └──────────────┘
          cohesion CAO              cohesion CAO
              └────── coupling LỎNG (ít dây) ──────┘
                        ✅ THIẾT KẾ TỐT
```

---

## ⑧ Câu hỏi phỏng vấn & thách đố

**(dễ)** *Coupling và cohesion là gì?*
→ Gợi ý: **coupling** = phụ thuộc-**giữa**-module; **cohesion** = thuộc-về-nhau-**trong**-module.

**(TB)** *Vì sao mục tiêu là "loose coupling, high cohesion"?*
→ Đổi một chỗ thì **ít lan** (coupling thấp); mỗi mảnh **dễ hiểu / dễ test / dễ tái dùng** (cohesion cao).

**(TB)** *`private` + getter/setter cho mọi field có phải encapsulation không?*
→ **Không.** Đó là **phơi state qua cửa sau**, mất bảo vệ **invariant**. Encapsulation thật phải *không cho* đặt object vào trạng thái sai từ ngoài.

**(khó)** *Nói một câu phân biệt abstraction và encapsulation.*
→ **Abstraction phơi CÁI GÌ; encapsulation giấu & bảo vệ NHƯ THẾ NÀO.**

> 🧩 **🔥 Thách đố:** *"Tách module để giảm coupling — vậy tách càng nhiều càng tốt?"*
>
> **Bẫy:** **over-decomposition** (tách quá đà) tạo ra **coupling phân tán** + **indirection vô ích**. Khi bạn xé một thứ gắn kết thành 10 mảnh tí hon, chúng vẫn phải nói chuyện với nhau — nhưng giờ phải nói *qua mạng/qua nhiều lớp gián tiếp*, vừa khó debug vừa chậm. Bạn không *giảm* coupling, bạn chỉ **đẩy nó đi nơi khác và làm nó khó thấy hơn**.
>
> 🎯 **Trả lời đúng:** Mục tiêu là **loose**, không phải **zero**. Tách theo **trục thay đổi thật** (những thứ *thay đổi cùng nhau* nên *ở cùng nhau*), **không tách theo cảm tính**.

---

## ⑨ Bài tập + tiêu chí tự chấm

**BT1.** Cho class `ShoppingCart` có `items`, `total`, và đang cho phép `cart.total = 999` set tuỳ ý từ bên ngoài.
→ **Yêu cầu:** Refactor để `total` là **derived** (suy ra từ `items`) và **không set được trực tiếp**.
→ ✅ **Đạt khi:** không thể đặt cart vào trạng thái `total ≠ tổng item` từ bên ngoài (thử `cart.total = 999` phải bất khả thi).

> 💡 *Gợi ý hướng làm:* biến `total` thành getter tính trên `#items`, và chỉ cho thêm/bớt item qua method `addItem()` / `removeItem()`. Đây chính là **encapsulation + invariant "total luôn = tổng item"**.

**BT2.** Lấy **1 class trong code thật** của bạn, viết đúng **2 câu**:
- (i) Nó **cohesion** cao/thấp — **vì sao**.
- (ii) Nó **coupling** chặt với ai — và **có thể đổi sang phụ thuộc abstraction không**.
→ ✅ **Đạt khi:** chỉ ra được **ít nhất 1 phụ thuộc kiểu cụ thể** nên đổi thành **abstraction** (vd: đang `new XxxClient()` thẳng → nên nhận qua interface).

---

## ⑩ Đọc thêm

- **Clean Code / Clean Architecture** — *Robert C. Martin* (chương về **cohesion/coupling**).
- Bài viết kinh điển **"Cohesion & Coupling"** — *Constantine / Yourdon* (**structured design** — nguồn gốc lịch sử của hai thước đo này).

---

## 🤖 Hiểu để chỉ huy (AI hay sai ở đây)

> 🚨 **AI sinh class "chạy được" nhưng RẤT HAY phơi state:** đủ bộ getter/setter cho mọi field, hoặc `public mutable field`. Cú pháp đúng → bạn dễ tưởng là ổn.
>
> 🎯 **Câu lệnh kiểm tra bạn PHẢI hỏi mỗi lần nhận code OOP từ AI:**
> **"Object này có thể bị đặt vào trạng thái SAI từ bên ngoài không?"**
> - Nếu **CÓ** → **encapsulation hỏng**, *dù cú pháp đúng* → bắt AI thêm method bảo vệ **invariant**, bỏ setter thừa.
> - Nếu **KHÔNG** → ok.
>
> Bạn không cần viết từng dòng — bạn cần **cầm la bàn coupling/cohesion** và **chỉ huy**: chỉ ra chỗ phơi state, chỗ phụ thuộc class cụ thể nên đổi sang abstraction. Đó là khác biệt giữa "người dùng AI" và "người chỉ huy AI".
