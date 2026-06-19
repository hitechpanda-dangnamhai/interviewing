# 🎯 BÀI 9 — Resilience: circuit breaker, bulkhead, retry, timeout

**Phủ:** K-RESIL-001 → 009

> 💡 **Mục tiêu một câu:** **Async** đã gỡ **temporal coupling** (Bài 1), nhưng mọi **call sync còn lại** (consumer→downstream, service→service) vẫn cần được **bảo vệ**. Bài này dạy bộ công cụ chống **cascade failure** — để hệ thống **fail FAST** và **fail SOFT** thay vì sập dây chuyền.

---

## ① Mục tiêu & vị trí trong mạch

**Async** gỡ **temporal coupling** (Bài 1), nhưng *không* xoá hết call **sync**: consumer vẫn gọi downstream, service vẫn gọi service. Mỗi call sync qua mạng là một **điểm có thể kéo sập cả hệ**.

> 🎯 **Bộ công cụ của bài này (theo thứ tự lắp ráp):** **timeout → circuit breaker → bulkhead → retry (đúng cách) → fallback.**

> 🔍 **Cross-ref:** **I-IDEM** (retry cần **idempotent**), **J** (fallback **cache**), **G-STR** (**backpressure**).

---

## ② Giảng cơ bản → nâng cao

### (a) Cascade failure 📘 K-RESIL-001

> 🁢 **Ẩn dụ — domino:** một quân đổ, đè quân kế, lan hết hàng. Trong hệ phân tán: một downstream chậm → caller chờ → cạn tài nguyên → caller cũng treo → lan ngược cả chuỗi.

```
TIMELINE — cascade failure:
  T1  PaymentSvc bắt đầu chậm (latency 5s thay vì 50ms)
  T2  CheckoutSvc gọi sync → thread/connection BỊ GIỮ chờ 5s
  T3  request mới tới Checkout → lấy thread mới → cũng bị giữ
  T4  POOL CẠN (resource exhaustion) → Checkout không nhận request nào nữa
  T5  service gọi Checkout cũng treo theo... → lan toàn hệ ❌
  → latency NHỎ ở 1 service khuếch đại thành sự cố TOÀN hệ
```

> 🚨 **Đây là kẻ thù chính** của cả bài. Mọi công cụ phía dưới đều để chặn cái chuỗi domino này.

---

### (b) Circuit breaker 📘 K-RESIL-002

> ⚡ **Ẩn dụ — cầu dao điện:** khi chập điện, cầu dao **ngắt** để bảo vệ cả nhà, thay vì để dòng điện hỏng đốt cháy mọi thứ.

**3 trạng thái:**

| Trạng thái | Hành vi | Chuyển khi |
|---|---|---|
| **Closed** | cho qua, **đếm lỗi** | vượt ngưỡng → **Open** |
| **Open** | **fail-fast** ngay (không gọi downstream, trả **fallback**) trong **cooldown** | hết cooldown → **Half-open** |
| **Half-open** | thử **vài request** | OK → **Closed**; fail → **Open** tiếp |

```
        lỗi vượt ngưỡng              hết cooldown
  CLOSED ──────────────► OPEN ──────────────► HALF-OPEN
    ▲                     ▲                       │
    │  thử OK             │  thử fail             │ thử vài request
    └─────────────────────┴───────────────────────┘
```

> 🎯 **Mục tiêu:** **fail FAST thay vì fail SLOW.** Open cho downstream *thời gian hồi* + bảo vệ *tài nguyên caller* (không phí thread gọi cái đang chết).

---

### (c) Timeout là tiền đề của breaker 📘 K-RESIL-003

> ⚠️ **Vì sao không có timeout thì breaker vô dụng:**

```
KHÔNG timeout:
  request "chậm vô hạn" → KHÔNG BAO GIỜ tính là lỗi
  → breaker KHÔNG trip → thread vẫn bị giữ → cascade ❌

CÓ timeout (vd 2s):
  request > 2s → tính là LỖI → breaker đếm được → trip → fail-fast ✅
```

> 🎯 **Chốt:** **timeout biến "chậm" thành "lỗi đếm được".** ⇒ **timeout + breaker + retry là bộ ba đi cùng, không tách rời.**

---

### (d) Bulkhead 📘 K-RESIL-004

> 🚢 **Ẩn dụ — vách ngăn tàu thuỷ:** thân tàu chia thành nhiều khoang kín. Một khoang thủng → nước chỉ ngập khoang đó, tàu **không chìm**.

**Bulkhead** = chia tài nguyên (**thread/connection pool**) thành **ngăn riêng cho từng downstream**. 1 downstream hỏng chỉ **cạn ngăn của nó**, không nuốt hết tài nguyên chung (**isolation**).

| | **Bulkhead** | **Circuit breaker** |
|---|---|---|
| Làm gì | **cô lập tài nguyên** (pool riêng) | **chặn call** hỏng (fail-fast) |
| Ví dụ | pool riêng cho payment vs cho search | ngắt call tới payment khi nó lỗi |

> 🎯 **Bổ trợ nhau, dùng cùng:** bulkhead giữ cho lỗi *không lan*, breaker giúp *không phí call* vào cái đang chết.

---

### (e) Retry đúng cách 📘 K-RESIL-005

> 🚨 **Retry "ngây thơ" (retry ngay, mọi lỗi, đồng loạt) khi downstream đang ngộp → retry storm làm nó sập hẳn (cộng hưởng).**

```
RETRY STORM — vì sao retry ngây thơ giết downstream:
  downstream đang ngộp (xử lý chậm)
    1000 client cùng fail → 1000 retry NGAY LẬP TỨC, ĐỒNG LOẠT
    → downstream nhận GẤP ĐÔI tải → sập hẳn ❌
```

**Đúng cách:**
- **exponential backoff** (giãn cách tăng dần: 200ms → 400ms → 800ms),
- **jitter** (thêm nhiễu ngẫu nhiên → **phá đồng bộ** giữa các client, tránh dồn cùng lúc),
- chỉ retry lỗi **transient / idempotent**,
- **retry budget** (giới hạn *tỷ lệ* retry tổng).

> ⚠️ **Retry không idempotent = nhân đôi side-effect** (charge 2 lần). Cross-ref **Bài 5, I-IDEM**.

---

### (f) Fallback & graceful degradation 📘 K-RESIL-007

> 🛟 **Ví dụ trực giác:** khi bếp chính hỏng, nhà hàng *không đóng cửa* — họ phục vụ **menu rút gọn**. Giữ trải nghiệm lõi, bỏ phần phụ.

Khi downstream chết, trả gì thay **lỗi cứng**? → **cached/stale**, **giá trị mặc định**, **kết quả một phần**, hoặc **xếp hàng xử lý sau**.

> 🎯 **"Fail soft":** degrade tính năng *phụ* để giữ tính năng *lõi*.
> *Ví dụ:* **gợi ý cá nhân hoá** chết → trả **danh sách phổ biến** thay vì **lỗi 500**.

---

### (g) Anti-pattern breaker 📘 K-RESIL-006

| Anti-pattern | Hậu quả |
|---|---|
| breaker **quá chặt** | **trip oan** (ngắt cả khi downstream khoẻ) |
| breaker **quá lỏng** | **lỗi lọt** (không bảo vệ kịp) |
| **không fallback** | vẫn **lỗi tới user** |
| bọc call **in-process** (nhẹ) | **thừa** — breaker chỉ cho call **remote / latency cao** |
| **thiếu visibility** | **silent failure** (hỏng âm thầm) |

> 💡 **Hiểu cả "khi nào KHÔNG dùng":** breaker không phải bọc mọi hàm — chỉ bọc call ra ngoài có độ trễ cao.

---

### (h) Backpressure 📘 K-RESIL-008

> 💧 **Ví dụ trực giác:** vòi nước (upstream) chảy nhanh hơn ống thoát (downstream) → bồn tràn (buffer phình → ngốn RAM/crash hoặc **lag vô hạn**).

**Xử lý:**
- **pull-based** (consumer **tự điều nhịp** — gặp lại Bài 2),
- **prefetch / concurrency limit**,
- **load shedding** (**drop / 429**),
- **queue có giới hạn**,
- **scale consumer**.

> 💡 **Broker (queue) vốn là một buffer có kiểm soát.** (cross-ref **G-STR**.)

---

### (i) Async ≠ viên đạn bạc 📘 K-RESIL-009

> 🚨 **Khẩu quyết:** **queue gỡ temporal coupling, NHƯNG không gỡ những thứ sau:**

| Vẫn cần | Vì sao |
|---|---|
| consumer gọi downstream **sync** | vẫn cần **breaker** |
| **DLQ / poison** | cần xử lý (Bài 5) |
| **lag** | cần **giám sát** |
| **idempotency** | vẫn **bắt buộc** |
| **broker** cũng có thể **sập** | cần HA (Bài 4) |

> 🎯 **Resilience vẫn cần ở từng consumer + biên ra ngoài.** Đừng tưởng "có Kafka là an toàn".

---

## ③ ⚠️ Cũ → Mới

| Cái cũ | Nay | Vì sao |
|---|---|---|
| **Retry ngay, mọi lỗi** | **Backoff + jitter + retry-budget**, chỉ transient/idempotent | Tránh **retry storm** |
| **Call remote không timeout** | **Timeout** luôn đặt (tiền đề breaker) | *"Chậm vô hạn"* = treo thread |
| **Tin "có queue là hết lo resilience"** | Vẫn cần **breaker/DLQ/idempotent/giám sát lag** | Queue chỉ gỡ **temporal coupling** |
| **Resilience nhúng trong code mỗi service** | Có thể đẩy lên **service mesh** (Bài 11) — cẩn thận **retry amplification** | Đồng bộ policy nhưng dễ **chồng retry** |

---

## ④ Áp dụng thực tế + bigtech

**Use case:** **Checkout** gọi **PaymentGateway** (3rd-party). Lắp đủ bộ:
- **timeout 2s**,
- **breaker** (mở khi **>50% lỗi / 10s**),
- **bulkhead** (**pool riêng** cho payment),
- **retry 2 lần** **backoff+jitter** *chỉ cho lỗi mạng* (**idempotency-key** để không **charge lặp**),
- **fallback** *"thanh toán đang xử lý, sẽ xác nhận sau"*.

**Bigtech:** **Netflix** sinh ra **Hystrix** (nay bảo trì hạn chế) → khái niệm **breaker/bulkhead** phổ cập; **resilience4j** (JVM), **opossum** (Node) là thư viện phổ biến; nhiều hệ đẩy **retry/breaker** xuống **mesh**. (🔎 *verify thư viện/đời*.)

---

## ⑤ Code thực hành + cấu hình

**Circuit breaker Node với `opossum` + timeout + fallback:**

```javascript
// paymentClient.js
const CircuitBreaker = require('opossum'); // pin "opossum@x.y" // verify đời
async function callPayment(req) { /* gọi HTTP tới payment gateway */ }

const breaker = new CircuitBreaker(callPayment, {
  timeout: 2000,                 // (c) timeout: biến 'chậm' thành lỗi -> breaker trip được
  errorThresholdPercentage: 50,  // mở khi >50% lỗi
  resetTimeout: 10000,           // cooldown trước khi half-open
  // volumeThreshold: 10,        // tối thiểu request mới tính tỉ lệ (tránh trip oan)
});
breaker.fallback(() => ({ status: 'PENDING', note: 'thanh toán đang xử lý' })); // (f) fail soft

async function pay(req) {
  return breaker.fire(req); // Open -> fail-fast -> trả fallback NGAY, không gọi downstream
}
```

```javascript
// retry backoff + jitter, CHỈ cho lỗi transient + idempotency-key (không charge lặp)
async function withRetry(fn, { tries = 3, base = 200 } = {}) {
  for (let i = 0; i < tries; i++) {
    try { return await fn(); }
    catch (e) {
      if (!isTransient(e) || i === tries - 1) throw e;      // lỗi VĨNH VIỄN -> KHÔNG retry (phí tài nguyên)
      const delay = base * 2 ** i + Math.random() * base;   // exponential (2^i) + jitter (random)
      await new Promise(r => setTimeout(r, delay));
    }
  }
}
```

> 🚨 **AI hay sai (ba lỗi kinh điển):**
> 1. bọc **breaker** quanh hàm **in-process** (vô nghĩa — không có latency mạng để bảo vệ);
> 2. **retry mọi exception** kể cả lỗi nghiệp vụ **4xx** (không bao giờ thành công + tốn tài nguyên);
> 3. retry **POST charge** **không idempotency-key** → **charge nhiều lần**.

---

## ⑥ Keywords

**🧠 Cần HIỂU:**
- **cascade failure**; **3 trạng thái breaker**.
- **timeout là tiền đề**; **bulkhead vs breaker**.
- **retry storm**; **fallback/degradation**.
- **backpressure**; **queue không miễn resilience**.

**📌 Cần THUỘC:**
- **closed / open / half-open**.
- **backoff + jitter + budget**.
- **retry chỉ transient + idempotent**.
- **bulkhead = pool riêng**.

**🛠️ Cần LÀM ĐƯỢC:**
- cấu hình **breaker + timeout + fallback** (**opossum**).
- viết **retry backoff+jitter** an toàn.
- nhận diện **anti-pattern breaker**.

---

## ⑦ Mental model

> 🧠 **"Timeout → breaker → bulkhead → retry(backoff+jitter, chỉ idempotent) → fallback.**
> **Một service chậm có thể kéo sập cả hệ; resilience là để fail FAST và fail SOFT.**
> **Queue KHÔNG cứu bạn khỏi việc này."**

---

## ⑧ Phỏng vấn & thách đố

**(TB)** **Cascade failure** xảy ra thế nào?
→ downstream chậm → thread caller bị giữ → cạn pool → caller treo → lan toàn chuỗi.

**(TB)** **3 trạng thái circuit breaker**?
→ closed (đếm lỗi) / open (fail-fast + cooldown) / half-open (thử vài request).

**(khó)** Vì sao **timeout là tiền đề** của breaker?
→ không timeout thì "chậm vô hạn" không tính là lỗi → breaker không trip → thread vẫn bị giữ.

**(khó)** **Bulkhead** khác **breaker**, vì sao dùng cùng?
→ bulkhead cô lập tài nguyên (pool riêng), breaker chặn call hỏng; bổ trợ nhau.

**(khó)** **Retry ngây thơ** hại gì, **backoff+jitter+budget** cứu sao?
→ retry storm dìm downstream; backoff giãn cách, jitter phá đồng bộ, budget giới hạn tỷ lệ.

> 🧩 **Thách đố:** *"Sự cố: downstream chậm 5s, toàn hệ sập dù chỉ 1 service lỗi. Trace cho thấy **hàng nghìn retry**. Hai nguyên nhân gốc và cách chặn?"*
> → **(1)** **không timeout** → thread bị giữ → **cạn pool** → cascade; **(2)** **retry không backoff/budget** → **retry storm** dìm downstream. **Chặn:** đặt **timeout** + **breaker** (fail-fast) + **bulkhead** (cô lập pool) + **retry backoff+jitter+budget**.

---

## ⑨ Bài tập + tiêu chí

**Đề:** Bọc một call tới **`recommendationService`** (không lõi) với resilience đầy đủ; khi nó chết, **trang vẫn render**.

> ✅ **Đạt khi:**
> - **timeout** + **breaker** + **fallback** trả danh sách **"phổ biến"** (**graceful degradation**).
> - **retry** chỉ lỗi **transient**.
> - nêu *vì sao* **KHÔNG retry** cho lỗi **4xx**.
> - (tuỳ chọn) **bulkhead** pool riêng.

---

## ⑩ Đọc thêm

- **Release It!** (Michael Nygard) — **cascade/breaker/bulkhead** kinh điển.
- **resilience4j docs**; **opossum** (npm). (🔎 *verify*.)

> 🎯 **Một câu mang về:** *Một service chậm là chuyện thường; một hệ sập vì service chậm là lỗi thiết kế. **Timeout + breaker + bulkhead + retry tử tế + fallback** là cách bạn biến "chậm" thành "giảm nhẹ" thay vì "thảm hoạ".*
