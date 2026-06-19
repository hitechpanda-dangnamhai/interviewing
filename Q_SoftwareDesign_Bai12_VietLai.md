# 🎯 BÀI 12 — Kiến trúc (2): **Anemic Model** · **Application/Domain Service** · **CQRS** · **Refactor Big-Ball-of-Mud**

**Phủ:** Q-ARCH-007 → Q-ARCH-008 → Q-ARCH-010 → Q-ARCH-011 → Q-ARCH-012

> 💡 **Mental model mở đầu:**
> Bài 11 dạy *kiến trúc lý tưởng* (đảo chiều vào trong). Bài này dạy **judgment kiến trúc** — *khi nào lý tưởng đó đáng, khi nào nó là over-engineering*:
> - **Anemic model** sai hay là lựa chọn?
> - Tách **read/write** (**CQRS**) lúc nào?
> - Refactor monolith **không big-bang** ra sao?
> Đây là bài về *sắc thái*, không phải *đúng/sai tuyệt đối*.

---

## ① Mục tiêu & vị trí trong mạch

Phần **nâng cao + judgment kiến trúc**: khi nào **full-clean** đáng, **anemic model** là sai hay là lựa chọn, tách **read/write** (**CQRS**), và **refactor monolith không big-bang**.

```
   Bài 11: kiến trúc lý tưởng (đảo chiều vào trong)
                    │  thêm JUDGMENT
                    ▼
   Bài 12: khi nào đáng / khi nào thừa  ← bài này
                    │
        nối DDD (Bài 13–14) + Tech Lead (Bài 15)
```

> 🎯 **Chốt vị trí:** bài này **nối DDD (Bài 13–14)** và **Tech Lead (Bài 15)**. Tinh thần xuyên suốt: **kiến trúc là quyết định có trade-off**, không phải "luôn dùng cái xịn nhất".

---

## ② Giảng cơ bản → nâng cao

### (a) Anemic Domain Model (Q-ARCH-007 — Fowler coi anti-pattern)

> **Ví dụ trực giác:** một con robot mà *không tự làm gì được* — mọi nút điều khiển nằm ở cái remote bên ngoài (service). Con robot chỉ là cái vỏ chứa pin (data).

**Anemic Domain Model** = **entity chỉ là túi getter/setter**, **mọi logic dồn vào service** → **mất encapsulation/OOP** (Bài 1). **DDD muốn hành vi + invariant nằm TRONG domain object.**

```
❌ ANEMIC                              ✅ RICH (DDD)
   class Account { balance; }            class Account {
   // logic ở ngoài:                       #balance;
   service.withdraw(acc, amt) {            withdraw(amt) {           // hành vi TRONG entity
     if (amt > acc.balance) throw ...       if (amt > this.#balance) throw ...
     acc.balance -= amt;                    this.#balance -= amt;   // invariant được bảo vệ
   }                                      } }
   → acc.balance = -100 vẫn set được     → không thể đặt trạng thái sai từ ngoài
```

> 🚨 **NHƯNG đây là trade-off CÓ CHỦ ĐÍCH, không phải luôn sai:** với **CRUD đơn giản** / đội quen **layered-service**, **anemic + service dày** chấp nhận được. Nhiều team **Nest** dùng **anemic entity + service** rất hiệu quả cho **domain mỏng**.

> 🎯 **Chốt:** anemic là *anti-pattern theo DDD*, nhưng *lựa chọn hợp lệ* khi domain mỏng. Đừng tụng "anemic luôn sai".

---

### (b) Application Service vs Domain Service vs Controller (Q-ARCH-010)

> **Ẩn dụ nhà hàng:** **Controller** = nhân viên *nhận order/bưng món* (I/O). **Application Service** = *bếp trưởng điều phối* (gọi món nào trước, đảm bảo cả bàn ra cùng lúc). **Domain Service** = *công thức nấu* (luật, không đụng tới chuyện bưng bê).

| Vai | Làm gì | Không làm gì |
|---|---|---|
| **Controller** | **adapter I/O** (nhận request, trả response). **Mỏng** | không chứa logic |
| **Application/Use-case Service** | **điều phối luồng** (orchestrate), quản **transaction/unit-of-work**, **gọi ra ngoài (port)** | **không chứa rule lõi** |
| **Domain Service** | **rule nghiệp vụ liên nhiều aggregate** | **không chạm hạ tầng** |

> 🔎 **(verify:** phân định này là **quy ước cộng đồng**, **không tuyệt đối** — các team có thể vạch ranh giới hơi khác.)

---

### (c) ⚠️ CQRS (Q-ARCH-011)

> **Ẩn dụ:** một nhà hàng tách **quầy gọi món** (ghi — cần kiểm tra kỹ, đúng quy trình) khỏi **bảng menu trưng bày** (đọc — tối ưu cho khách xem nhanh, đẹp). Hai phía tối ưu cho hai mục tiêu khác nhau.

**CQRS** (**Command Query Responsibility Segregation**) = tách **model GHI** (giữ **invariant**) khỏi **model ĐỌC** (tối ưu **truy vấn/projection**).

| | Khi nào **đáng** | Khi nào **thừa** |
|---|---|---|
| CQRS | **read/write rất khác nhau** hoặc **khác tải** (đọc nhiều, ghi phức tạp) | **CRUD đơn giản** |

> 🚨 **CẢNH BÁO — đừng gộp hai khái niệm:** **CQRS KHÔNG bắt buộc đi kèm Event Sourcing.** CQRS chỉ là *tách đọc/ghi*; **Event Sourcing (ES)** là *lưu chuỗi sự kiện thay vì state* — hai thứ độc lập, có thể dùng riêng.

---

### (d) Full-clean cho microservice CRUD mỏng có đáng? (Q-ARCH-008)

**Hexagonal/clean** thêm **layer/mapping/interface**. Chi phí **ports/adapters/mapping** chỉ đáng khi **domain đủ phức tạp/biến động**.

| Độ phức tạp domain | Nên dùng |
|---|---|
| **CRUD mỏng** | layered đơn giản (full-clean = **over-engineering**) |
| **phức tạp / biến động** | full-clean (ports/adapters trả phí xứng đáng) |

> 🎯 **Quyết theo:** **độ phức tạp nghiệp vụ + vòng đời + đội ngũ**, **không giáo điều**.

---

### (e) Refactor big-ball-of-mud không big-bang (Q-ARCH-012)

> **Ẩn dụ:** sửa nhà đang ở — bạn **không đập sập cả nhà rồi xây lại** (lúc đó ở đâu?). Bạn sửa **từng phòng**, vẫn ở được suốt quá trình.

**Big-ball-of-mud** = monolith tầng trộn lẫn. **Đừng rewrite toàn bộ** (rủi ro cao). **Lộ trình incremental:**

```
T1  CHARACTERIZATION TEST  → chụp behavior hiện tại (lưới an toàn)
        │
T2  BỌC SEAM bằng interface tại ranh giới
        │
T3  TÁCH domain dần, đẩy I/O ra adapter
        │
T4  STRANGLER-FIG: viết tính năng MỚI theo kiến trúc mới,
        dần "siết" phần cũ cho tới khi thay hết
        ▼
✅ Hệ thống luôn chạy được trong suốt quá trình (không "tắt đèn" để rewrite)
```

> 💡 **Vì sao incremental thắng big-bang?** Vì **big-bang rewrite** = viết lại từ đầu trong khi bản cũ vẫn phải chạy, dễ *trễ hạn vô tận* và *bỏ sót hành vi ngầm* mà không ai nhớ. Incremental giữ hệ thống *luôn sống*, mỗi bước nhỏ có test bảo chứng.

---

## ③ ⚠️ Cũ → Mới / Lưu ý

| Quan niệm | Sắc thái đúng | Vì sao |
|---|---|---|
| **"Anemic model luôn sai"** | **Anti-pattern theo DDD** nhưng là **trade-off hợp lệ** với domain mỏng | Tuỳ **độ phức tạp/đội** |
| **"CQRS = Event Sourcing"** | **CQRS chỉ tách read/write**; **ES độc lập** | Hai khái niệm khác nhau |
| **"Full clean cho mọi service"** | **Chỉ khi domain phức tạp/biến động** | Tránh **over-engineering CRUD** |
| **"Refactor = rewrite lại từ đầu"** | **Incremental**: **seam + strangler-fig + test** | **Big-bang rewrite** rủi ro cao |

---

## ④ Áp dụng thực tế + So sánh bigtech

### CQRS thật
**Trang sản phẩm đọc cực nhiều** (**read model** denormalized/cache) trong khi **đặt hàng ghi phức tạp** (**write model** giữ invariant) → **tách hai phía**.

### Strangler-fig
**Amazon/Shopify** tách monolith dần bằng cách **route tính năng mới sang service mới**, **giữ cũ chạy song song**.

### Bigtech / ecosystem (pattern phổ biến — không tuyệt đối)

| Trường hợp | Ghi chú |
|---|---|
| **Martin Fowler** đặt tên **"Strangler Fig"** | nhiều cuộc migration lớn (monolith→service) theo pattern này **thay vì rewrite** |

> 🔎 **Insight sâu — vì sao tên "Strangler Fig"?** Cây đa bóp cổ (strangler fig) mọc quấn quanh cây chủ, lớn dần, cho tới khi cây chủ mục đi thì cây đa đã thay thế hoàn toàn — *mà khu rừng chưa bao giờ thiếu một cái cây*. Đó đúng là tinh thần migration: cái mới lớn dần *quanh* cái cũ, không có khoảnh khắc "tắt hệ thống để chuyển".

---

## ⑤ Code thực hành

```typescript
// layering-roles.ts — phân vai Controller / App Service / Domain Service

// DOMAIN SERVICE: rule liên nhiều aggregate, KHÔNG chạm hạ tầng
class TransferPolicy {
  ensureCanTransfer(from: Account, to: Account, amount: number) {
    if (from.id === to.id) throw new Error('Same account');
    if (amount <= 0) throw new Error('Amount > 0');        // PURE business rule, không I/O
  }
}

// APPLICATION SERVICE: điều phối use case + transaction + gọi PORT ra ngoài
class TransferMoneyUseCase {
  constructor(
    private readonly accounts: AccountRepository,          // PORT (driven)
    private readonly policy: TransferPolicy,
    private readonly tx: UnitOfWork,                        // quản TRANSACTION
  ) {}
  async exec(fromId: string, toId: string, amount: number) {
    await this.tx.run(async () => {                        // mở transaction/unit-of-work
      const from = await this.accounts.get(fromId);
      const to   = await this.accounts.get(toId);
      this.policy.ensureCanTransfer(from, to, amount);     // gọi DOMAIN RULE
      from.withdraw(amount); to.deposit(amount);           // hành vi TRONG entity (CHỐNG anemic)
      await this.accounts.save(from); await this.accounts.save(to);
    });
  }
}

// CONTROLLER (adapter I/O): MỎNG — parse request, gọi use case, map response. (xem H-MOD)
```

> 💡 **Đọc kỹ phân vai:** `from.withdraw(amount)` là **hành vi nằm trong entity** (chống **anemic**); `TransferPolicy` là **rule liên 2 account** (Domain Service); `TransferMoneyUseCase` chỉ **điều phối + quản transaction** (App Service), **không chứa rule lõi**. Mỗi tầng đúng việc → cohesion cao, dễ test.

---

## ⑥ Keywords cần nhớ

> 🧠 **Cần HIỂU:**
> - **anemic model** (anti-pattern nhưng **trade-off**)
> - **Controller / App Service / Domain Service**
> - **CQRS** (**≠ ES**)
> - **full-clean trade-off**
> - **strangler-fig**

> 📌 **Cần THUỘC:**
> - Ranh giới **3 loại service**.
> - **CQRS không bắt buộc Event Sourcing**.

> 🛠️ **Cần LÀM ĐƯỢC:**
> - Đặt **rule vào đúng tầng**.
> - Vạch lộ trình **refactor monolith incremental**.

---

## ⑦ Mental model

> 🎯 **Khẩu quyết:**
> **Controller mỏng (I/O) · App Service điều phối + transaction · Domain Service = rule liên aggregate · Entity giữ hành vi (chống anemic).**
> Refactor monolith: **bóp cổ dần (strangler), đừng đập đi xây lại.**

```
   Request → Controller (mỏng) → App Service (điều phối+tx) → Domain rule + Entity
   Read nhiều ≠ Write phức tạp?  → cân nhắc CQRS (KHÔNG kèm ES bắt buộc)
   Domain mỏng?                  → layered đơn giản, anemic OK (đừng full-clean)
   Monolith rối?                 → test + seam + strangler-fig (KHÔNG rewrite)
```

---

## ⑧ Câu hỏi phỏng vấn & thách đố

**(khó)** *Anemic Domain Model là gì, vì sao Fowler coi anti-pattern?*
→ **entity rỗng, logic ở service**; **mất encapsulation**.

**(khó)** *App Service vs Domain Service vs Controller?*
→ **điều phối+tx / rule liên aggregate / adapter I/O**.

**(rất khó)** *CQRS giải gì? Khi nào tách read/write hợp lý, khi nào thừa?*
→ **tách model**; khi **read/write khác nhau/tải**; **CRUD thì thừa**.

**(rất khó)** *Full clean cho CRUD microservice mỏng — đáng không, quyết theo gì?*
→ **thường over-engineering**; theo **độ phức tạp/vòng đời/đội**.

> 🧩 **🔥 Thách đố (Q-ARCH-012):** *"Monolith big-ball-of-mud — đề xuất rewrite toàn bộ?"*
>
> **Bẫy:** **big-bang rewrite rủi ro cực cao**; **incremental**: **characterization test + seam + strangler-fig** theo module. Trả lời "rewrite cho sạch" là rơi bẫy kinh điển — nhiều dự án rewrite chết vì cái cũ chưa hiểu hết đã đập.

---

## ⑨ Bài tập + tiêu chí tự chấm

**BT1.** Lấy **1 entity anemic + service dày**, chuyển **1-2 rule vào entity** (chống anemic).
→ ✅ **Đạt khi:** **entity không rơi vào trạng thái sai**; **service mỏng hơn**.

> 💡 *Gợi ý:* tìm chỗ service đọc field của entity ra rồi tự kiểm tra/sửa → đó là rule nên *dọn vào* entity (giống Feature Envy ở Bài 10).

**BT2.** Phác **lộ trình strangler-fig** cho 1 module monolith bạn biết.
→ ✅ **Đạt khi:** có bước **test-an-toàn + seam + thứ tự siết**, **không rewrite một phát**.

---

## ⑩ Đọc thêm

- **Fowler** — **"AnemicDomainModel"**, **"StranglerFigApplication"**.
- **Greg Young / Udi Dahan** — **CQRS**.

---

## 🤖 Hiểu để chỉ huy (Kiểm tra AI)

> 🚨 **AI rất hay:**
> 1. **Sinh anemic** — dồn logic vào service, entity rỗng.
> 2. **Gợi ý CQRS/Event Sourcing "cho xịn"** lên CRUD mỏng.
> 3. **Đề xuất rewrite** khi gặp monolith rối.
>
> 🎯 **Hai câu soát mỗi lần nhận đề xuất kiến trúc từ AI:**
> - *"**Domain** có cần phức tạp đến thế không?"* → không → cắt CQRS/ES/full-clean thừa (chống over-engineering).
> - *"**Rule** đặt **đúng tầng** chưa?"* → logic lõi nằm ở service mà entity rỗng → kéo rule về entity/domain service.
>
> AI mặc định "thích" trông-có-vẻ-enterprise: gợi ý CQRS, Event Sourcing, rewrite — vì chúng *nghe oách*. Vai trò chỉ huy của bạn: **cân độ phức tạp thật của domain**, **đặt rule đúng tầng**, và **không bao giờ chọn big-bang rewrite** khi incremental khả thi.
