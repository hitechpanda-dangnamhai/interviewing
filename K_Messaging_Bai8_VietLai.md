# 🎯 BÀI 8 — Event Sourcing & CQRS

**Phủ:** K-ES-001 → 007

> 💡 **Mục tiêu một câu:** **ES & CQRS** là họ hàng *"event-driven"* nhưng ở **tầng lưu trữ trạng thái**, không phải tầng giao tiếp. Học xong, bạn phân biệt **ES** với *"audit log"*, hiểu **snapshot / projection / eventual**, và — quan trọng nhất — biết **khi nào KHÔNG dùng** (tránh over-engineering).

---

## ① Mục tiêu & vị trí trong mạch

Các bài trước (1–7) nói về **giao tiếp** giữa service (gửi/nhận message). Bài này đổi tầng: **ES & CQRS** ở **tầng lưu trữ trạng thái** — *cách bạn cất giữ sự thật của một aggregate*, không phải cách bạn gửi nó đi.

> 🎯 **Chốt vị trí:** điểm "đắt giá" nhất của bài không phải *cách dùng* mà là *khi nào đừng dùng*. Cross-ref **H-CQRS** (CQRS kiểu Nest).

---

## ② Giảng cơ bản → nâng cao

### (a) Event Sourcing 📘 K-ES-001, 002

> 🎬 **Ẩn dụ — cuốn phim vs tấm ảnh:** lưu **state hiện tại** giống chỉ giữ *một tấm ảnh chụp lúc này* (`UPDATE` đè lên ảnh cũ → mất quá khứ). **Event Sourcing** giống giữ **toàn bộ cuốn phim** — mọi khung hình *đã xảy ra* (event **bất biến**, **append-only**). Muốn biết "lúc này thế nào" → **tua phim (replay/fold)** tới khung cuối.

Thay vì lưu **state hiện tại** (`UPDATE` đè), **ES** lưu **chuỗi event bất biến** (**append-only**); **state hiện tại = replay/fold** các event.

| | **Được** | **Mất** |
|---|---|---|
| ES | **audit đầy đủ**; **time-travel** (state tại bất kỳ thời điểm); **rebuild view mới** từ lịch sử | **phức tạp**; query state hiện tại **khó** (phải **fold**); **eventual**; **schema event** tiến hoá khó; cần **snapshot** |

#### Khác "audit log thường" 📘 K-ES-002

> 🎯 **Điểm cốt lõi — "ai là source of truth":**

| | **Event Sourcing** | **Audit log thường** |
|---|---|---|
| Event là gì | **nguồn sự thật chính** (state *dựng từ* event) | chỉ là **phụ phẩm** bên cạnh bảng state |
| Bảng state | *suy ra* từ event | là **source of truth**, audit chỉ ghi kèm |

> 💡 **Vì sao phân biệt quan trọng:** nếu event chỉ là "log ghi kèm", bạn *có thể* sửa bảng state thẳng tay. Trong ES thì **không** — event là sự thật, sửa state mà không qua event = phá vỡ mô hình.

---

### (b) Snapshot 📘 K-ES-003

> 📸 **Ví dụ trực giác:** tua phim từ khung #0 mỗi lần *quá tốn*. **Snapshot** = chụp lại *một tấm ảnh tóm tắt* tại khung N; lần sau chỉ cần *tua từ N* trở đi.

**Replay từ event #0 mỗi lần quá tốn** → lưu **snapshot** (state tại **offset N**), chỉ replay event **sau N**.

> ⚖️ **Cần khi:** stream **dài** / đọc **thường xuyên**. **Trade-off: bộ nhớ ↔ tốc độ rebuild** (lưu nhiều snapshot tốn chỗ nhưng rebuild nhanh).

```
KHÔNG snapshot:                CÓ snapshot tại N=1000:
  replay event 0 → 1500          load snapshot(1000)
  = fold 1500 event 🐌           + replay event 1001 → 1500
                                 = fold 500 event ⚡
```

---

### (c) CQRS 📘 K-ES-004

> 🧠 **Mental model:** **CQRS** (**Command Query Responsibility Segregation**) = **tách write model khỏi read model**.

- **Write model** (**command**): tối ưu **ghi / nhất quán**.
- **Read model** (**query**): tối ưu **đọc**.

**ES + CQRS hay đi cùng** vì: ES ghi **event** (write) **khó query trực tiếp** → **project** ra **nhiều read model** phù hợp truy vấn.

> 🚨 **Nhưng đừng gộp làm một:** **CQRS không bắt buộc ES** và **ngược lại**. Bạn có thể làm CQRS thuần (read replica/materialized view) trên một hệ CRUD thường, hoàn toàn không có ES.

---

### (d) Projection eventual 📘 K-ES-005

> ⏱️ **Ví dụ trực giác:** bạn vừa nạp tiền, nhưng màn hình số dư *chưa nhảy ngay* — vì **read model** được cập nhật **bất đồng bộ**, trễ một nhịp so với **write**.

**Read model** cập nhật **bất đồng bộ** → **trễ** so với write → user có thể **không thấy ngay** thay đổi vừa ghi.

```
TIMELINE — projection trễ:
  T1  user nạp 100k → write model ghi event Deposited  ✅
  T2  user xem số dư → read model CHƯA cập nhật → vẫn hiện số cũ ❌
  T3  projection chạy xong → read model = số mới  ✅
  → khoảng T2 chính là "độ trễ projection" phải thiết kế cho nó
```

**Xử lý:**
- **chấp nhận** + UI **optimistic** (hiện luôn kết quả mong đợi),
- **đọc-từ-write-model** cho path cần **fresh**,
- hiển thị **"đang xử lý"**.

> 🎯 **Phải thiết kế cho độ trễ projection** — đừng giả định read model luôn khớp write tức thì.

---

### (e) Schema evolution trong ES 📘 K-ES-006

> ⚠️ **Chi phí dài hạn ít ai lường:** **event đã ghi là bất biến**, **không sửa được**.

Cần:
- **versioning event** + **upcaster** (chuyển **event v1 → v2** khi *đọc*),
- hoặc **weak schema** (bỏ qua field thiếu).

> 🚨 **Không bao giờ rewrite lịch sử tuỳ tiện.** Event là sự thật đã xảy ra — sửa nó = nói dối về quá khứ.

---

### (f) Khi nào KHÔNG dùng ES 📘 K-ES-007

> 🚨 **Cảnh báo over-engineering — đây là phần đắt giá nhất bài.**

**CRUD đơn giản**, không cần **audit/time-travel**, team chưa quen → **ES thêm phức tạp lớn** (**eventual, versioning, rebuild**) **không tương xứng lợi ích**.

> 🎯 **Chỉ dùng khi domain *thật sự* cần lịch sử/audit/temporal:** **finance, ledger, kế toán, đặt chỗ**. Nhận diện **cost/benefit**.

| Domain | ES đáng? |
|---|---|
| **sổ cái / ledger / payment** | ✅ cần audit tuyệt đối, lịch sử số dư |
| **đặt chỗ / booking** | ✅ cần temporal |
| **CRUD hồ sơ user / profile** | ❌ over-engineering |

---

## ③ ⚠️ Cũ → Mới

| Cái cũ | Nay | Vì sao |
|---|---|---|
| **"ES = audit table cho mọi app"** | ES **chỉ khi** cần lịch sử/audit/time-travel là *yêu cầu* | Tránh **over-engineering** |
| **Replay từ #0 mỗi lần** | **Snapshot** + replay phần đuôi | Tốc độ **rebuild** |
| **Sửa/migrate event cũ tại chỗ** | **Versioning + upcaster** (event bất biến) | Không **rewrite lịch sử** |
| **Gộp ES = CQRS** | Tách: **CQRS không cần ES** & ngược lại | Hai khái niệm **độc lập** |

---

## ④ Áp dụng thực tế + bigtech

| Use case | Quyết định | Vì sao |
|---|---|---|
| **sổ cái tài khoản** | ✅ **ES** | mọi giao dịch là event bất biến; **số dư = fold**; cần **audit tuyệt đối** |
| **CRUD hồ sơ user/profile** | ❌ **không ES** | không cần lịch sử, ES chỉ thêm gánh |
| **hệ đọc nặng** | **CQRS không-ES** | tách **read replica / materialized view** query nhanh, write vẫn **CRUD thường** |

**Bigtech:** **ledger/payment systems** dùng **ES**; nhiều hệ dùng **CQRS** (read model riêng cho search/feed) mà **không ES**. (pattern; 🔎 *verify khi cần ví dụ cụ thể*.)

---

## ⑤ Code thực hành + cấu hình

**Replay + snapshot** (pseudo):

```javascript
// rebuildState — fold event; dùng snapshot để khỏi replay từ đầu
function rebuildAccount(accountId) {
  const snap = store.getLatestSnapshot(accountId);          // {state, version} hoặc null
  let state = snap ? snap.state : { balance: 0, version: 0 };
  const events = store.getEventsAfter(accountId, state.version); // CHỈ event sau snapshot (tua nhanh)
  for (const e of events) state = apply(state, e);          // fold: apply thuần, deterministic
  if (events.length > SNAPSHOT_EVERY) store.saveSnapshot(accountId, state); // tạo snapshot mới
  return state;
}

function apply(s, e) {                                       // PHẢI thuần + xử lý version event
  switch (e.type) {
    case 'Deposited':  return { ...s, balance: s.balance + e.amount, version: e.version };
    case 'Withdrawn':  return { ...s, balance: s.balance - e.amount, version: e.version };
    // case 'DepositedV2': ... // hoặc upcast v1->v2 TRƯỚC khi vào đây
    default: return s; // weak schema: bỏ qua event lạ (forward-compat)
  }
}
```

> 🚨 **AI hay sai (hai lỗi kinh điển):**
> 1. dùng **ES** cho **CRUD bình thường** *"cho hiện đại"* → gánh **eventual + versioning + rebuild** vô ích.
> 2. viết **`apply` không deterministic** (gọi **`Date.now()` / `random`**) → **replay ra state khác nhau** mỗi lần.

```
VÌ SAO apply PHẢI deterministic:
  apply gọi Date.now() bên trong:
    replay lần 1 (hôm nay)   → state có timestamp A
    replay lần 2 (mai)       → state có timestamp B  ❌ KHÁC NHAU!
  → cùng chuỗi event PHẢI luôn cho cùng state; mọi giá trị "biến đổi"
    phải nằm SẴN trong event, không sinh ra lúc apply
```

---

## ⑥ Keywords

**🧠 Cần HIỂU:**
- **event là source of truth** (vs **audit log**).
- **fold/replay**; **snapshot**.
- **CQRS** tách read/write.
- **projection eventual**.
- **khi nào KHÔNG dùng**.

**📌 Cần THUỘC:**
- **ES ≠ audit log**.
- **snapshot** = state tại **offset N**.
- **CQRS ⟂ ES** (độc lập).
- **upcaster** cho **schema evolution**.

**🛠️ Cần LÀM ĐƯỢC:**
- viết **`apply` thuần / deterministic** + **snapshot**.
- quyết **ES có đáng không** cho một domain.

---

## ⑦ Mental model

> 🧠 **"ES: lưu chuyện đã xảy ra (event bất biến), state = tua lại.**
> **Snapshot để tua nhanh. CQRS tách đọc/ghi, KHÔNG bắt buộc ES.**
> **Đừng dùng ES cho CRUD — chỉ khi lịch sử/audit là yêu cầu thật."**

---

## ⑧ Phỏng vấn & thách đố

**(khó)** **ES** là gì, đánh đổi gì?
→ lưu chuỗi event bất biến, state = fold; được audit/time-travel/rebuild, mất phức tạp/eventual/query khó.

**(TB)** **ES** khác **audit log** ở điểm cốt lõi nào?
→ ai là source of truth: ES event là sự thật chính; audit log chỉ là phụ phẩm.

**(TB)** **Snapshot** để làm gì, khi nào cần?
→ tránh replay từ #0; cần khi stream dài / đọc thường xuyên.

**(khó)** **CQRS** là gì, vì sao hay đi với **ES**? CQRS có cần ES không?
→ tách read/write; ES khó query nên project ra read model; CQRS không cần ES.

**(rất khó)** **Schema event** tiến hoá xử lý sao khi event bất biến?
→ versioning + upcaster (v1→v2 khi đọc) hoặc weak schema; không rewrite lịch sử.

> 🧩 **Thách đố:** *"Team định **ES hoá toàn bộ** app quản lý nhân sự 'để audit'. Bạn cản hay ủng hộ?"*
> → **Cản (phần lớn).** HR phần lớn là **CRUD**; nếu **audit** là yêu cầu, **audit log / temporal table** thường **đủ**, **rẻ hơn nhiều** ES (không gánh **eventual/versioning/rebuild**). Chỉ **ES các aggregate thật sự cần time-travel/replay**. Tránh **over-engineering toàn cục**.

---

## ⑨ Bài tập + tiêu chí

**Đề:** Cho **Wallet** (nạp/rút). Quyết: **ES hay CRUD**? Nếu ES, định nghĩa **3 event** + hàm **`apply`** + chính sách **snapshot**.

> ✅ **Đạt khi:**
> - lập luận **ES hợp lý** cho ví/tiền (**audit**, lịch sử số dư).
> - event **quá khứ** (**Deposited/Withdrawn**).
> - **`apply` thuần deterministic**.
> - **snapshot** mỗi **N event**.
> - nhắc **upcaster** khi đổi schema.

---

## ⑩ Đọc thêm

- **`martinfowler.com`**: **Event Sourcing**, **CQRS**.
- **`microservices.io`**.
- **EventStoreDB docs**. (🔎 *verify*.)

> 🎯 **Một câu mang về:** *ES cho bạn cả cuốn phim quá khứ — nhưng cuốn phim đó tốn chỗ và khó dựng. Chỉ quay phim khi bạn **thật sự** cần xem lại quá khứ; còn lại, một tấm ảnh (CRUD) là đủ.*
