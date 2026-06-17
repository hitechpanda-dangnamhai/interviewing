# 📗 Giáo trình mục G — Node.js Runtime Depth (WORKFLOW 2 · Bước B + C)

> Dựng **ngược** từ `QB_G_nodejs.md` (84 câu, 11 mục con). Mỗi bài bám đúng các câu nó phủ và tuân **Hợp đồng đầu ра 10 mục**.
> Bước D (phỏng vấn chấm live) làm riêng theo từng đợt — file này là phần **giảng**, không phải đáp án mẫu.
> *Câu thần chú: "Tôi không nhớ để gõ, tôi hiểu để chỉ huy — và tra cứu phần còn lại."*

## ⚙️ Bối cảnh phiên bản (verify 6/2026 — nguồn: nodejs.org, endoflife.date, AWS, nhà cung cấp)
- **Node 24 (Krypton)** = **Active LTS** → mặc định cho production mới (npm 11, V8 mới, TS type-stripping ổn định).
- **Node 22 (Jod)** = Maintenance LTS, EOL **4/2027**.
- **Node 20 (Iron)** đã **EOL 30/4/2026** → KHÔNG dùng production.
- **Node 26** = Current (ra 5/5/2026), lên LTS **10/2026**; mang V8 14.6 + Temporal API mặc định.
- Từ **Node 27 (10/2026)**: 1 major/năm (tháng 4), **mọi release đều thành LTS**, version theo năm. → *verify tại nodejs.org/en/about/previous-releases.*

---

# 🗺️ BƯỚC B — Giáo trình ngược từ bộ đề

Gom 84 câu theo **cụm khái niệm phụ thuộc** rồi sắp **cơ bản → nâng cao** (theo dependency, KHÔNG theo tần suất hỏi). 11 bài:

| Bài | Tên bài | Phủ ID câu | Số câu |
|---|---|---|---|
| **1** | Kiến trúc runtime & mô hình single-thread | G-RT-001, G-LU-001, G-LU-002, G-EL-001 | 4 |
| **2** | Event loop: các phase & cơ chế poll | G-EL-002, G-EL-005, G-EL-007, G-EL-009 | 4 |
| **3** | Thứ tự thực thi: micro/macrotask · nextTick · timers | G-EL-003, G-EL-004, G-EL-006, G-EL-008, G-TIM-001…007 | 11 |
| **4** | libuv thread pool (chiều sâu) | G-LU-003…008 | 6 |
| **5** | Async patterns: callback → Promise → async/await | G-ASY-001…009 | 9 |
| **6** | Block event loop & xử lý CPU-bound | G-BLK-001…006 | 6 |
| **7** | Scale đa core: worker_threads · cluster · child_process | G-WRK-001…007, G-CLU-001…007 | 14 |
| **8** | Streams & backpressure | G-STR-001…007 | 7 |
| **9** | Diagnostics, profiling, GC & event-loop observability | G-DIAG-001…009, G-EL-010, G-EL-011, G-RT-002 | 12 |
| **10** | Security hardening | G-SEC-001…009 | 9 |
| **11** | TL runtime depth: module system & version strategy | G-RT-003, G-RT-004 | 2 |

**Logic phụ thuộc:** Bài 1 (kiến trúc) là nền cho mọi bài. 2→3 là cốt lõi event loop (phải hiểu phase trước khi luận thứ tự). 4 (thread pool) nối tiếp 1. 5 (async) là tay nghề viết code. 6 (blocking) hệ quả trực tiếp của single-thread. 7 (scale) là lời giải cho 6. 8 (streams) độc lập nhưng cần hiểu backpressure ~ event loop. 9 (diagnostics) cần đủ vốn từ 1–8 mới debug được. 10 (security) tay nghề production. 11 đóng gói tư duy TL.

---

# 📚 BƯỚC C — Giảng từng bài

---

## 🎓 BÀI 1 — Kiến trúc runtime & mô hình single-thread
> Phủ: **G-RT-001, G-LU-001, G-LU-002, G-EL-001** · Phase 6 (App/Runtime) trong roadmap.

### ① Mục tiêu & vị trí trong mạch
Hết coi Node là "hộp đen". Sau bài: phân biệt rạch ròi **V8** (engine chạy JS + GC) với **libuv** (event loop + async I/O + thread pool), và phát biểu chính xác mô hình "single-thread" mà không nói lệch thành "đa luồng". Đây là nền cho TẤT CẢ bài sau — không nắm bài này thì event loop, thread pool, worker đều mơ hồ.

### ② Giảng cơ bản → nâng cao
**(a) Trực giác.** Hình dung Node như một **nhà hàng 1 đầu bếp giỏi (V8 chạy JS) + một đội phục vụ chạy việc nền (libuv)**. Đầu bếp chỉ làm 1 món tại 1 thời điểm (single-thread JS), nhưng không đứng chờ lò nướng: anh ta giao việc chờ (I/O, đọc file) cho đội phục vụ, làm món khác, khi việc nền xong thì có người báo lại để anh xử lý tiếp (callback).

**(b) Cơ chế.**
- **V8** (📘): biên dịch JIT + chạy JavaScript, quản lý **heap** và **garbage collection**. Một luồng JS duy nhất ("main thread") thực thi call stack của bạn.
- **libuv** (📘): thư viện C cung cấp (1) **event loop**, (2) **async I/O đa nền tảng** (bọc epoll/io_uring trên Linux, kqueue trên macOS, IOCP trên Windows), (3) **thread pool** (mặc định 4 thread) cho các tác vụ không có async OS-native, (4) abstraction cho fs/dns/crypto/zlib.
- **Single-thread nghĩa là gì cho đúng:** *code JavaScript của bạn* chạy trên 1 thread. Nhưng tổng thể runtime là **đa luồng dưới nắp** — libuv có thread pool + các thread nền lo I/O. Câu chuẩn: "JS single-thread, runtime đa luồng."

**(c) Mép giới hạn & sai lầm thường gặp.**
- Sai: "Node đa luồng nên nhanh" → nhầm. JS vẫn 1 luồng; sức mạnh đến từ **non-blocking I/O**, không phải song song hóa JS.
- Sai: "async = chạy trên nhiều thread" → phần lớn async I/O (network) KHÔNG dùng thread pool mà dùng cơ chế OS (bài 4).
- Sai: gộp V8 và libuv làm một. GC chậm là chuyện của V8; thread pool cạn là chuyện của libuv — debug khác nhau hoàn toàn.

### ③ ⚠️ Kiến thức cũ / dễ nhầm

| Quan niệm cũ/sai | Hiểu đúng hiện nay | Vì sao |
|---|---|---|
| "Node đa luồng" | JS single-thread, runtime đa luồng (libuv pool + I/O nền) | Tránh kỳ vọng sai về song song |
| "async I/O = thread pool" | Network I/O dùng epoll/io_uring/kqueue; chỉ fs/dns/crypto/zlib dùng pool | Quyết định cách scale & tune `UV_THREADPOOL_SIZE` |
| "V8 lo cả I/O" | V8 chỉ chạy JS + GC; I/O là libuv | Debug đúng tầng |

### ④ Áp dụng thực tế + So sánh bigtech
Hiểu kiến trúc này quyết định **mọi quyết định scale**: vì JS single-thread, 1 process Node chỉ dùng hết ~1 core → muốn dùng 16 core phải chạy 16 process (cluster/K8s pod) hoặc đẩy CPU-bound sang worker. Đây là lý do Netflix, PayPal, LinkedIn dùng Node cho tầng I/O-bound (API gateway, BFF) nơi non-blocking tỏa sáng, và tránh Node cho compute nặng. *(Pattern phổ biến, không phải con số tuyệt đối — kiến trúc nội bộ từng team khác nhau.)*

### ⑤ Code thực hành + cấu hình
```js
// file: runtime-proof.mjs — chứng minh "JS 1 thread, I/O nền"
// Chạy: node runtime-proof.mjs   (Node 24 LTS)
import { readFile } from 'node:fs/promises';

console.log('1. start (sync, main thread)');

readFile('./package.json', 'utf8')          // I/O đi xuống libuv (thread pool cho fs)
  .then(() => console.log('3. file read xong (callback quay lại main thread)'));

// vòng lặp đồng bộ này BLOCK main thread ~ vài trăm ms
let sum = 0;
for (let i = 0; i < 5e8; i++) sum += i;
console.log('2. xong vòng lặp đồng bộ — in TRƯỚC callback file dù file nhỏ');
// → chứng minh: I/O chạy nền, nhưng callback chỉ được xử lý khi main thread RẢNH
```
Cấu hình tối thiểu: `node:fs/promises` là API built-in, không cần `requirements`. Pin runtime trong `package.json`:
```json
{ "engines": { "node": ">=24 <25" } }
```

### ⑥ Keywords cần nhớ
- 🧠 **Cần HIỂU:** single-thread JS vs runtime đa luồng · vai trò V8 (JS+GC) vs libuv (loop+I/O+pool) · non-blocking I/O.
- 📌 **Cần THUỘC:** thread pool mặc định = 4 · libuv = C library · epoll/io_uring/kqueue/IOCP là cơ chế I/O OS.
- 🛠️ **Cần LÀM ĐƯỢC:** chứng minh main thread bị block bằng vòng lặp sync · pin Node version qua `engines`.

### ⑦ Mental model
**"V8 chạy JS (1 luồng) — libuv lo việc chờ (nhiều luồng). Code bạn block thì cả nhà hàng dừng."**

### ⑧ Câu hỏi phỏng vấn & thách đố
1. *(dễ)* Node có thật single-thread không? → JS có, runtime không (gợi: libuv pool + I/O nền).
2. *(TB)* V8 và libuv khác vai trò gì? → V8: chạy JS + GC; libuv: event loop + async I/O + thread pool.
3. *(khó)* Vì sao nói "Node giỏi I/O-bound, dở CPU-bound"? → non-blocking I/O cho phép 1 thread phục vụ ngàn kết nối; nhưng CPU nặng chiếm thread đó → treo tất cả.
4. **Thách đố:** "Tôi thêm `Promise` vào code, vậy nó có chạy trên thread khác để nhanh hơn không?" → **Bẫy.** Không. Promise chỉ là cơ chế lập lịch async trên CÙNG main thread; không tạo thread. Muốn dùng core khác cần worker_threads.

### ⑨ Bài tập + tiêu chí
Viết 1 file lặp tính tổng `1e9` ngay sau khi `setTimeout(()=>console.log('timer'),0)`. **Đạt khi:** giải thích được vì sao "timer" in *sau* khi vòng lặp xong, dù timeout = 0 (main thread bận → callback phải chờ).

### ⑩ Đọc thêm
nodejs.org/en/learn (Overview) · docs.libuv.org · "Don't Block the Event Loop" (nodejs.org guides).

> 🛂 *Kiểm tra AI:* AI hay viết "Node is multi-threaded so it scales" trong comment/README — **sửa thành** "JS is single-threaded; scale qua process/worker". Cú pháp code có thể đúng nhưng câu mô tả kiến trúc sai sẽ dẫn cả team tới quyết định scale sai.

---

## 🎓 BÀI 2 — Event loop: các phase & cơ chế poll
> Phủ: **G-EL-002, G-EL-005, G-EL-007, G-EL-009** · nối tiếp Bài 1.

### ① Mục tiêu & vị trí
Biết **6 phase của event loop theo đúng thứ tự**, mỗi phase làm gì, và đặc biệt hiểu **poll phase** (nơi phần lớn code chạy, nơi loop quyết định block hay đi tiếp). Đây là điều kiện để Bài 3 luận được thứ tự output.

### ② Giảng cơ bản → nâng cao
**(a) Trực giác.** Event loop = **vòng tuần tra của 1 nhân viên** đi qua 6 trạm cố định, mỗi trạm xử lý 1 loại việc đang chờ, hết vòng quay lại đầu. "Một tick" = đi trọn 1 vòng qua các trạm.

**(b) Cơ chế — 6 phase (📘, theo libuv):**
1. **timers** — chạy callback của `setTimeout`/`setInterval` đã tới hạn.
2. **pending callbacks** — vài callback hệ thống bị hoãn (vd lỗi TCP).
3. **idle, prepare** — nội bộ libuv (bỏ qua ở mức app).
4. **poll** — **trạm quan trọng nhất:** lấy I/O event mới (đọc socket/file xong), chạy callback I/O; **block chờ I/O ở đây** nếu không có việc khác.
5. **check** — chạy callback `setImmediate`.
6. **close callbacks** — callback `close` (vd `socket.on('close')`).

→ Phần lớn code ứng dụng (HTTP handler, DB callback) chạy ở **poll phase**.

**(c) Poll phase sâu (G-EL-005):** khi vào poll, nếu poll queue **rỗng**:
- Có `setImmediate` đang chờ → **nhảy sang check** ngay.
- Không có setImmediate nhưng có **timer sắp tới hạn** → quay lại timers.
- Không có gì → **block tại poll chờ I/O** (đây là lúc Node "ngủ" tiết kiệm CPU, không phải spin-loop). 

**(d) Node vs browser event loop (G-EL-007):** trình duyệt có 1 task queue + microtask queue + render step; **không có** khái niệm phase poll/check/libuv, **không có** `process.nextTick`, dùng API browser (rAF, MessageChannel) thay vì libuv. Node có nhiều phase + `process.nextTick` + libuv I/O.

### ③ ⚠️ Cũ → Mới / dễ nhầm

| Hiểu sai phổ biến | Đúng | Vì sao |
|---|---|---|
| "Event loop chạy callback ngay khi I/O xong" | Callback xếp vào queue, chỉ chạy khi tới đúng phase | Giải thích vì sao có độ trễ dù I/O nhanh |
| "setImmediate chạy 'ngay'" | Chạy ở phase **check**, tức vòng sau poll | Đặt tên gây hiểu lầm |
| "Node event loop = browser event loop" | Khác phase + nextTick + libuv | Câu phỏng vấn bẫy |

### ④ Áp dụng thực tế + bigtech
Hiểu poll phase block-chờ-I/O giải thích vì sao 1 process Node phục vụ **hàng nghìn kết nối đồng thời** với ít RAM: nó không tạo thread/process mỗi kết nối (như mô hình thread-per-request của Java cũ/Apache prefork) mà ngủ ở poll, được OS đánh thức khi có data. Đây là nền của hiệu năng I/O trong các API gateway, BFF tại nhiều công ty dùng Node. *(pattern phổ biến.)*

### ⑤ Code thực hành
```js
// file: poll-vs-check.mjs — chứng minh setImmediate chạy ở 'check' sau 'poll'
import { readFile } from 'node:fs';

readFile(import.meta.filename, () => {     // callback này chạy trong POLL phase
  setTimeout(() => console.log('timeout'), 0);   // → timers (vòng SAU)
  setImmediate(() => console.log('immediate'));  // → check (NGAY sau poll, vòng NÀY)
  // KQ ổn định: 'immediate' trước 'timeout'
  // vì đang trong I/O callback (poll) → check tới trước khi vòng sau quay lại timers
});
```

### ⑥ Keywords
- 🧠 **HIỂU:** ý nghĩa poll phase (lấy I/O + block chờ) · 1 tick = 1 vòng qua phase · khác biệt Node vs browser.
- 📌 **THUỘC:** thứ tự 6 phase: timers → pending → idle/prepare → **poll** → check → close.
- 🛠️ **LÀM ĐƯỢC:** dự đoán immediate vs timeout *trong* I/O callback; giải thích "block tại poll".

### ⑦ Mental model
**"6 trạm tuần tra: timers → poll (việc chính, ngủ chờ I/O) → check (setImmediate). Hết vòng là 1 tick."**

### ⑧ Phỏng vấn & thách đố
1. Kể 6 phase đúng thứ tự. → timers/pending/idle-prepare/poll/check/close.
2. Phần lớn code chạy ở phase nào? → poll.
3. *(khó)* Khi nào poll block, khi nào nhảy check? → rỗng + có immediate → check; có timer tới hạn → timers; không gì → block chờ I/O.
4. **Thách đố:** "Vì sao Node CPU ~0% khi idle dù 'có vòng lặp'?" → Bẫy "busy-loop". Poll phase **block-chờ** ở tầng OS (epoll_wait), không spin → không tốn CPU.

### ⑨ Bài tập + tiêu chí
Đặt `setTimeout(0)` và `setImmediate` (a) ở top-level, (b) trong 1 `fs.readFile` callback; chạy 5 lần. **Đạt khi:** giải thích được (a) thứ tự *không ổn định*, (b) immediate luôn trước.

### ⑩ Đọc thêm
nodejs.org guide "The Node.js Event Loop, Timers, and process.nextTick" · docs.libuv.org/en/v1.x/design.html.

> 🛂 *Kiểm tra AI:* AI thường mô tả event loop kiểu browser (chỉ "task + microtask"), bỏ phase poll/check. Khi nó giải thích thứ tự `setImmediate` mà không nhắc "đang ở trong I/O callback hay không" → câu trả lời sẽ sai 50% trường hợp. Hỏi lại "ngữ cảnh nào?".

---

## 🎓 BÀI 3 — Thứ tự thực thi: micro/macrotask · nextTick · timers
> Phủ: **G-EL-003, G-EL-004, G-EL-006, G-EL-008, G-TIM-001…007** (11 câu — bài "xương sống" của mọi câu hỏi thứ tự).

### ① Mục tiêu & vị trí
Sau bài này bạn **luận được output của bất kỳ đoạn code trộn** `setTimeout` / `setImmediate` / `Promise.then` / `process.nextTick` mà không đoán mò — kỹ năng phân biệt senior với người học vẹt. Nối thẳng từ Bài 2 (phase).

### ② Giảng cơ bản → nâng cao
**(a) Hai loại task.**
- **Macrotask** (đi qua phase): `setTimeout`, `setInterval`, `setImmediate`, I/O callback. Mỗi vòng phase chạy **một nhóm** macrotask của phase đó.
- **Microtask** (chen giữa): `Promise.then/catch/finally`, `queueMicrotask`, và **đặc biệt** `process.nextTick` (kỹ thuật là "nextTick queue", không phải microtask chuẩn nhưng hành xử tương tự).

**(b) Luật vàng về thời điểm drain (G-EL-003, G-EL-004):** sau **mỗi** callback (hoặc giữa mỗi phase), Node **drain TRỌN** queue theo thứ tự ưu tiên:
> **`process.nextTick` queue (drain hết) → microtask/Promise queue (drain hết) → mới tiến phase macrotask kế.**

Nghĩa là nextTick ưu tiên hơn Promise; cả hai chạy *trước* mọi `setTimeout`/`setImmediate` còn chờ.

**(c) Các so sánh chốt:**
- `process.nextTick` vs `setImmediate` (G-TIM-001): nextTick chạy **ngay sau operation hiện tại, trước khi loop tiến phase**; setImmediate chạy ở **check phase vòng sau**. nextTick "sớm hơn nhiều".
- `setTimeout(0)` vs `setImmediate` (G-TIM-002, 003): **top-level → không xác định** (tùy thời điểm loop khởi động vs timer threshold 1ms); **trong I/O callback → setImmediate luôn trước** (vì poll→check tới trước khi vòng sau quay lại timers).
- `setTimeout(fn, N)` (G-TIM-004): N là **delay tối thiểu**, KHÔNG đảm bảo đúng N ms; loop bận/blocked sẽ trễ.
- `queueMicrotask` (G-TIM-006): đẩy vào microtask queue (ngang Promise, **sau** nextTick); chuẩn hơn dùng `Promise.resolve().then` chỉ để defer.

**(d) Mép giới hạn — STARVATION (G-EL-006, G-TIM-005):** vì nextTick/microtask **drain trọn trước khi tiến phase**, một đệ quy vô tận `process.nextTick(fn)` hoặc `Promise.then` lồng nhau sẽ **"đói" event loop** — timers và I/O không bao giờ tới lượt → server treo dù CPU không quá tải kiểu tính toán. Đây là bug production thật.

**(e) Anti-pattern timing (G-TIM-007):** dùng `setTimeout(…, 500)` để "chờ DB sẵn sàng/đợi state" là **đoán timing** → race condition. Sửa: dùng **event/Promise/readiness check thực sự** (vd `await pool.query('SELECT 1')`, event `'ready'`, health probe).

### ③ ⚠️ Cũ → Mới

| Cách cũ/sai | Nên dùng | Vì sao |
|---|---|---|
| `setTimeout(fn, 0)` để "defer tới sau" | `setImmediate` (defer 1 vòng) hoặc `queueMicrotask` (defer trong tick) | rõ ý định, ổn định hơn |
| `Promise.resolve().then(fn)` chỉ để defer | `queueMicrotask(fn)` | đúng ngữ nghĩa, không tạo Promise thừa |
| `setTimeout` chờ tài nguyên sẵn sàng | readiness check / event / retry-with-backoff | tránh race do đoán timing |
| Đệ quy `process.nextTick` cho việc dài | chia chunk bằng `setImmediate` (nhường loop) | nextTick starve I/O; setImmediate cho loop thở |

### ④ Áp dụng thực tế + bigtech
`process.nextTick` được dùng nội bộ để giữ **API async-consistent**: một hàm đôi khi sync đôi khi async sẽ gây bug Zalgo; defer bằng nextTick để callback "luôn async". Các thư viện lớn (Node core, Express middleware) dùng nguyên tắc này. Khi xử lý batch nặng mà muốn nhường loop cho request khác, pattern chuẩn là chia chunk + `setImmediate` (bài 6).

### ⑤ Code thực hành
```js
// file: ordering.mjs — câu kinh điển G-EL-008. Dự đoán TRƯỚC khi chạy.
import { readFile } from 'node:fs';

readFile(import.meta.filename, () => {           // ta đang ở POLL phase
  setTimeout(() => console.log('A timeout'), 0); // macrotask → timers (vòng sau)
  setImmediate(() => console.log('B immediate'));// macrotask → check (vòng này, sau poll)
  Promise.resolve().then(() => console.log('C promise')); // microtask
  process.nextTick(() => console.log('D nextTick'));       // nextTick (ưu tiên cao nhất)
});
// Thứ tự ĐÚNG: D → C → B → A
// Giải thích: hết callback readFile → drain nextTick(D) → drain microtask(C)
//            → tiến check phase (B immediate) → vòng sau tới timers (A timeout)
```
```js
// file: starve-demo.mjs — TÁI HIỆN starvation rồi sửa
function starve() { process.nextTick(starve); }   // ❌ I/O & timer KHÔNG BAO GIỜ chạy
// setTimeout(() => console.log('never?'), 100);   // sẽ bị "đói"
// FIX: dùng setImmediate để nhường loop mỗi vòng:
function polite(n) { if (n <= 0) return; setImmediate(() => polite(n - 1)); } // ✅
```

### ⑥ Keywords
- 🧠 **HIỂU:** vì sao nextTick/microtask drain trọn trước phase → starvation · "delay là tối thiểu" · anti-pattern đoán timing.
- 📌 **THUỘC:** ưu tiên **nextTick > microtask(Promise/queueMicrotask) > macrotask** · trong I/O callback: immediate > timeout(0).
- 🛠️ **LÀM ĐƯỢC:** luận output đoạn trộn 4 loại · fix starvation bằng setImmediate · thay setTimeout-chờ bằng readiness check.

### ⑦ Mental model
**"Hết mỗi callback: dọn sạch nextTick → dọn sạch Promise → mới đi phase kế. nextTick lạm dụng = bỏ đói cả loop."**

### ⑧ Phỏng vấn & thách đố
1. nextTick vs setImmediate, cái nào trước? → nextTick (trước khi tiến phase) vs setImmediate (check, vòng sau).
2. `setTimeout(0)` vs `setImmediate` top-level — ổn định không? → **không**; trong I/O callback thì immediate luôn trước.
3. Phân loại 5 thứ vào micro/macro/nextTick. → Promise.then=micro; setTimeout/setImmediate/I/O=macro; nextTick=riêng (ưu tiên nhất).
4. **Thách đố 1:** "Đặt `setTimeout(fn, 100)` mà fn chạy sau ~3s — vì sao?" → loop bị block/bận; delay chỉ là tối thiểu.
5. **Thách đố 2 (trick):** "Code dùng `setTimeout(()=>start(), 2000)` chờ DB connect — review thế nào?" → race; thay bằng `await db.ready()` / retry. AI có thể viết đúng cú pháp setTimeout nhưng **kiến trúc sai**.

### ⑨ Bài tập + tiêu chí
Cho 1 đoạn trộn `nextTick, Promise.then, setTimeout(0), setImmediate` đặt trong I/O callback. **Đạt khi:** viết đúng thứ tự *trước khi chạy* và giải thích bằng "drain trọn → tiến phase".

### ⑩ Đọc thêm
nodejs.org guide "Event Loop, Timers, and process.nextTick" · "The Node.js Event Loop" (phần Don't confuse nextTick & setImmediate).

> 🛂 *Kiểm tra AI:* khi AI giải bài "đoán output", nó hay quên ngữ cảnh phase (top-level vs trong I/O callback) → đảo thứ tự immediate/timeout. Luôn hỏi nó: "đoạn này đang ở phase nào?".

---

## 🎓 BÀI 4 — libuv thread pool (chiều sâu)
> Phủ: **G-LU-003…008** · nối Bài 1.

### ① Mục tiêu & vị trí
Biết **chính xác** thao tác nào dùng thread pool, thao tác nào không; tune `UV_THREADPOOL_SIZE` đúng cách và hiểu trade-off. Giải thích được hiện tượng "request thứ 5 chậm hẳn".

### ② Giảng cơ bản → nâng cao
**(a) Trực giác.** Thread pool = **4 nhân viên phụ** lo những việc mà OS không cho làm async kiểu native. Hết 4 người bận thì việc thứ 5 xếp hàng.

**(b) Ai dùng pool, ai không (G-LU-003, G-LU-006):**
- **DÙNG pool:** `fs.*` (file I/O), một số `crypto` (`pbkdf2`, `scrypt`, `randomBytes` lớn), `zlib` (nén), `dns.lookup` (qua `getaddrinfo`).
- **KHÔNG dùng pool:** **network socket** (TCP/HTTP) — dùng cơ chế OS native: **epoll** (Linux), **io_uring** (Linux mới), **kqueue** (macOS/BSD), **IOCP** (Windows). Vì OS có async network nhất quán nên libuv không cần mô phỏng bằng thread.
- **Lý do fs phải dùng pool:** không OS nào có async file I/O nhất quán/đáng tin → libuv mô phỏng bằng cách chạy syscall blocking trên thread nền.

**(c) Cấu hình (G-LU-004, G-LU-008):**
- Mặc định **4 thread**. Đổi bằng biến môi trường **`UV_THREADPOOL_SIZE`** (phải set TRƯỚC khi process khởi động/tạo pool; max 1024).
- Tăng khi: nhiều `fs`/`crypto`/`zlib` đồng thời và đang nghẽn pool.
- **Trade-off (G-LU-008):** tăng pool > số CPU core không làm CPU-bound (crypto) nhanh hơn — chỉ thêm **context switch**. Pool chủ yếu cho **đồng thời I/O/crypto**, không phải tăng tốc tính toán.

**(d) Bẫy DNS (G-LU-007):** `dns.lookup` (mặc định trong `http`/`net`) dùng `getaddrinfo` **trên thread pool** → nếu DNS chậm, nó **ăn slot pool** và làm chậm cả fs/crypto khác. Phòng: dùng `dns.resolve*` (không qua pool, hỏi thẳng DNS server) hoặc DNS cache, hoặc tăng pool.

### ③ ⚠️ Cũ → Mới / dễ nhầm

| Hiểu sai | Đúng | Vì sao |
|---|---|---|
| "Mọi async I/O đều dùng thread pool" | Chỉ fs/dns.lookup/crypto/zlib; network dùng OS native | Tune pool đúng chỗ |
| "Tăng UV_THREADPOOL_SIZE luôn nhanh hơn" | Quá số core → context switch, không nhanh hơn CPU-bound | Tránh tune mù |
| "DNS lookup không liên quan pool" | dns.lookup ăn pool → có thể nghẽn fs/crypto | Bug "chậm bí ẩn" |

### ④ Áp dụng thực tế + bigtech
Service hash mật khẩu (bcrypt/argon2/pbkdf2) hoặc nén ảnh nhiều đồng thời thường **nghẽn pool 4 thread** → p99 latency tăng vọt khi tải cao. Cách xử lý production: tăng `UV_THREADPOOL_SIZE` hợp lý (vd = số core), hoặc đẩy hashing sang **worker pool/service riêng** (bài 7). Các team chạy Node ở quy mô lớn thường set `UV_THREADPOOL_SIZE` rõ ràng trong manifest thay vì để mặc định 4. *(pattern phổ biến — verify theo workload.)*

### ⑤ Code thực hành
```js
// file: pool-saturation.mjs — TÁI HIỆN G-LU-005: lời gọi crypto thứ 5 chậm
// Chạy thử 2 lần: mặc định, rồi  UV_THREADPOOL_SIZE=8 node pool-saturation.mjs
import { pbkdf2 } from 'node:crypto';

const t0 = Date.now();
for (let i = 1; i <= 5; i++) {
  pbkdf2('pw', 'salt', 200000, 64, 'sha512', () => {
    console.log(`call ${i} xong sau ${Date.now() - t0}ms`);
    // pool=4: call 1-4 xong gần cùng lúc, call 5 xong TRỄ (chờ slot)
  });
}
```
Cấu hình: đặt biến TRƯỚC khi chạy (không đổi được sau khi pool đã tạo):
```bash
UV_THREADPOOL_SIZE=8 node pool-saturation.mjs   # tăng pool để 5 call song song
```

### ⑥ Keywords
- 🧠 **HIỂU:** vì sao fs dùng pool còn network thì không · pool ≠ tăng tốc CPU-bound · DNS lookup ăn pool.
- 📌 **THUỘC:** pool mặc định **4** · biến `UV_THREADPOOL_SIZE` · pool dùng cho **fs, crypto(pbkdf2/scrypt), zlib, dns.lookup** · network = epoll/io_uring/kqueue/IOCP.
- 🛠️ **LÀM ĐƯỢC:** tái hiện pool saturation · đổi pool size · thay dns.lookup→dns.resolve khi cần.

### ⑦ Mental model
**"4 nhân viên phụ lo fs/crypto/dns; network thì OS tự lo. Việc thứ 5 phải xếp hàng — đó là pool, không phải event loop."**

### ⑧ Phỏng vấn & thách đố
1. Pool mặc định mấy thread, đổi bằng gì? → 4, `UV_THREADPOOL_SIZE`.
2. fs vs HTTP request: cái nào dùng pool? → fs dùng pool; HTTP (network) dùng OS native.
3. *(khó)* Tăng pool lên 64 trên máy 4 core có nhanh hơn cho crypto không? → không cho CPU-bound; thêm context switch.
4. **Thách đố:** "App chậm bí ẩn khi DNS provider lag, dù không đụng file — vì sao?" → `dns.lookup` ăn slot pool → fs/crypto bị chậm theo. Fix: dns.resolve/cache.

### ⑨ Bài tập + tiêu chí
Chạy 8 `pbkdf2` song song với pool=4 rồi pool=8, đo thời gian. **Đạt khi:** giải thích đường cong "4 nhanh, phần dư xếp hàng" và vì sao pool=8 cải thiện.

### ⑩ Đọc thêm
docs.libuv.org "Thread pool work scheduling" · nodejs.org API `dns` (lookup vs resolve).

> 🛂 *Kiểm tra AI:* AI hay khuyên "tăng UV_THREADPOOL_SIZE để app nhanh hơn" một cách vô điều kiện. Kiểm tra: workload có phải I/O/crypto đồng thời không, máy mấy core — nếu CPU-bound thuần, tăng pool **không** giúp.

---

## 🎓 BÀI 5 — Async patterns: callback → Promise → async/await
> Phủ: **G-ASY-001…009** · tay nghề viết code hằng ngày.

### ① Mục tiêu & vị trí
Viết async code **đúng và an toàn**: chọn tuần tự vs song song đúng chỗ, xử lý lỗi không để "floating promise", dùng `Promise.all/allSettled/race/any` đúng tình huống, và biết `AsyncLocalStorage` cho context.

### ② Giảng cơ bản → nâng cao
**(a) Ba thế hệ, một mô hình (G-ASY-001, 002):** callback → Promise → async/await **cùng một mô hình async**, chỉ khác cú pháp. Callback lồng sâu = "callback hell" (khó đọc, lỗi rải rác). Promise giải bằng **chaining + `.catch` tập trung**. `async/await` là **đường cú pháp trên Promise** — viết async như sync.

**(b) Tuần tự vs song song (G-ASY-003) — bẫy phổ biến nhất:**
```js
// ❌ tuần tự không cần thiết (chậm n lần)
for (const id of ids) { results.push(await fetchUser(id)); }
// ✅ song song khi độc lập
const results = await Promise.all(ids.map(fetchUser));
```
Dùng **tuần tự** khi: cần thứ tự, bước sau phụ thuộc bước trước, hoặc cần **giới hạn concurrency** (vd `p-limit`) để không quá tải DB/API.

**(c) Combinators (G-ASY-004):**
- `Promise.all` — **fail-fast**: 1 reject → cả cụm reject (kết quả khác mất).
- `Promise.allSettled` — chờ **hết**, trả mảng `{status, value/reason}` (không fail-fast) → tốt khi muốn biết cái nào fail.
- `Promise.race` — kết quả **đầu tiên** settle (kể cả reject) → dùng timeout.
- `Promise.any` — **resolve** đầu tiên (bỏ qua reject, fail khi tất cả reject) → fallback nhiều nguồn.

**(d) Xử lý lỗi (G-ASY-005, 006, 007):**
- **Unhandled rejection:** từ Node ≥15 **crash process** mặc định. Cần `.catch`/`try-catch` + listener `process.on('unhandledRejection')` để log/graceful (không nuốt lỗi âm thầm).
- `try-catch` chỉ bắt lỗi khi **`await`**; KHÔNG bắt lỗi trong callback không-await hoặc Promise quên `await`/`return`.
- **Floating promise** (gọi async mà không await/catch): lỗi mất tăm, thứ tự sai, rejection không bắt → bug production. Phát hiện ở team: **lint rule `@typescript-eslint/no-floating-promises`**, code review.

**(e) Công cụ (G-ASY-008, 009):**
- Promisify API callback-style: `util.promisify(fn)` hoặc dùng bản `fs/promises`. Hiểu convention **error-first callback** `(err, data) => {}`.
- **`AsyncLocalStorage`**: giữ **context theo chuỗi async** (request id, trace id, user) mà không phải truyền tay qua mọi hàm → nền của logging/tracing per-request.

### ③ ⚠️ Cũ → Mới

| Cũ | Nên dùng | Vì sao |
|---|---|---|
| Callback lồng nhau (callback hell) | async/await + try-catch | đọc/handle lỗi gọn |
| `await` trong vòng for cho task độc lập | `Promise.all` / `allSettled` (+ concurrency limit) | nhanh n lần |
| Gọi async không await ("floating") | luôn `await`/`.catch` + lint no-floating-promises | tránh rejection mất tăm |
| Truyền `reqId` tay qua mọi hàm | `AsyncLocalStorage` | context tự theo chuỗi async |
| Nuốt lỗi bằng `.catch(()=>{})` | log + xử lý/rethrow; listener unhandledRejection | không che bug |

### ④ Áp dụng thực tế + bigtech
`AsyncLocalStorage` là cơ chế đằng sau **request-scoped logging/tracing** trong các framework (NestJS request context, OpenTelemetry context propagation). `Promise.allSettled` dùng cho **fan-out** gọi nhiều downstream rồi gộp kết quả một phần (vd trang tổng hợp nhiều microservice — service nào lỗi vẫn render phần còn lại). `Promise.race` dùng làm **timeout wrapper** quanh call mạng.

### ⑤ Code thực hành
```js
// file: async-toolkit.mjs (Node 24)
import { setTimeout as delay } from 'node:timers/promises';

// 1) timeout bằng race
function withTimeout(p, ms) {
  return Promise.race([
    p,
    delay(ms).then(() => { throw new Error('timeout'); }),
  ]);
}

// 2) song song có GIỚI HẠN concurrency (không quá tải downstream) — không cần lib
async function mapLimit(items, limit, fn) {
  const ret = []; const exec = [];
  for (const item of items) {
    const p = Promise.resolve().then(() => fn(item));
    ret.push(p);
    if (limit <= items.length) {
      const e = p.then(() => exec.splice(exec.indexOf(e), 1));
      exec.push(e);
      if (exec.length >= limit) await Promise.race(exec);
    }
  }
  return Promise.all(ret);
}

// 3) bắt rejection toàn cục (chỉ để LOG + graceful, không để "chữa cháy" thay catch)
process.on('unhandledRejection', (reason) => {
  console.error('UNHANDLED', reason);  // → log có cấu trúc; cân nhắc shutdown an toàn
});
```
```js
// file: context.mjs — AsyncLocalStorage giữ trace id per-request
import { AsyncLocalStorage } from 'node:async_hooks';
export const als = new AsyncLocalStorage();
export const log = (msg) => console.log(JSON.stringify({ reqId: als.getStore()?.reqId, msg }));
// middleware: als.run({ reqId: crypto.randomUUID() }, () => next())
```
Lint bắt floating promise (`.eslintrc`): bật `@typescript-eslint/no-floating-promises` (cần TS + type info). `// kiểm chứng cú pháp config tại typescript-eslint.io`.

### ⑥ Keywords
- 🧠 **HIỂU:** ba thế hệ = một mô hình · tuần tự vs song song · vì sao floating promise nguy hiểm · AsyncLocalStorage giải bài gì.
- 📌 **THUỘC:** all (fail-fast) / allSettled (chờ hết) / race (đầu tiên kể cả reject) / any (resolve đầu tiên) · unhandledRejection ≥ Node15 crash · error-first callback.
- 🛠️ **LÀM ĐƯỢC:** đổi await-in-loop → Promise.all có limit · viết timeout bằng race · promisify · bật lint no-floating-promises.

### ⑦ Mental model
**"async/await chỉ là cú pháp trên Promise. Độc lập thì all, cần biết ai fail thì allSettled, đừng để promise trôi nổi."**

### ⑧ Phỏng vấn & thách đố
1. callback vs Promise vs async/await? → cùng mô hình, khác cú pháp.
2. all vs allSettled vs race vs any — tình huống? → như mục (c).
3. *(khó)* try-catch có bắt mọi lỗi async không? → chỉ khi await; không bắt callback/floating.
4. **Thách đố 1:** "Vòng `for...await` gọi 100 API mất 50s, sửa thế nào mà không sập downstream?" → `mapLimit`/Promise.all có concurrency limit, không phải Promise.all thuần (100 đồng thời).
5. **Thách đố 2:** "Code chạy ổn local nhưng prod thỉnh thoảng crash không log — nghi gì?" → floating promise rejection; bật lint + unhandledRejection.

### ⑨ Bài tập + tiêu chí
(a) Viết `withTimeout`. (b) Đổi 1 vòng await tuần tự sang song song giới hạn 5. **Đạt khi:** test đúng + giải thích vì sao limit cần (bảo vệ downstream).

### ⑩ Đọc thêm
MDN Promise (all/allSettled/race/any) · nodejs.org API `async_hooks`/AsyncLocalStorage · typescript-eslint no-floating-promises.

> 🛂 *Kiểm tra AI:* AI mặc định viết `Promise.all(items.map(fn))` cho mọi danh sách — **bẫy** khi list lớn (vài nghìn) sẽ mở đồng thời quá nhiều kết nối, sập DB/API. Luôn hỏi "có cần giới hạn concurrency không?".

---

## 🎓 BÀI 6 — Block event loop & xử lý CPU-bound
> Phủ: **G-BLK-001…006** · hệ quả trực tiếp của single-thread (Bài 1) → đặt vấn đề cho Bài 7.

### ① Mục tiêu & vị trí
Nhận diện thứ gì **block event loop**, vì sao nó nguy hiểm đặc trưng cho Node, và **các bậc thang giải pháp** (từ chia chunk → worker → offload). Đây là chủ đề TL-level vì sai ở đây là sập cả service.

### ② Giảng cơ bản → nâng cao
**(a) Vì sao block ở Node nguy hiểm hơn (G-BLK-001):** vì JS single-thread, một tác vụ CPU nặng (loop lớn, regex tệ, JSON khổng lồ) **chiếm main thread** → **mọi request khác bị treo**, không chỉ request đang chạy. Mô hình thread-per-request (Java cũ) chỉ chậm 1 request; Node treo **tất cả**.

**(b) Các thủ phạm block thường gặp:**
- **Sync API trong hot path (G-BLK-004):** `readFileSync`, `crypto ...Sync`, `execSync` → block suốt thời gian I/O. OK lúc **startup**, KHÔNG OK trong request handler.
- **JSON lớn (G-BLK-003):** `JSON.parse/stringify` payload khổng lồ là **sync CPU-bound** → block. Xử lý: streaming JSON parser, giới hạn body size, hoặc offload.
- **ReDoS (G-BLK-006):** regex "catastrophic backtracking" chạy sync → block loop = **DoS** chỉ với 1 input độc hại. Phòng: regex an toàn (tránh nested quantifier `(a+)+`), validate độ dài input, timeout, dùng lib an toàn (RE2).

**(c) Bậc thang giải pháp CPU-bound (G-BLK-002, 005):**
1. **Chia chunk + nhường loop** (G-BLK-005): cắt vòng lặp dài thành phần nhỏ, mỗi phần `setImmediate`/`await` để loop xử lý request khác xen vào. **Giới hạn:** vẫn chiếm 1 core, chỉ giảm "treo", không tăng throughput tính toán.
2. **`worker_threads`** — chạy tính toán trên thread khác (Bài 7), không block main loop.
3. **`child_process`** — tiến trình con (kể cả ngôn ngữ khác).
4. **Offload sang service/queue** — đẩy job nặng sang microservice/worker queue (BullMQ/Redis, SQS) → scale độc lập, cô lập lỗi.

Chọn theo **độ nặng + nhu cầu scale + tần suất**: nhẹ/thỉnh thoảng → chunk; nặng/thường xuyên trong process → worker pool; rất nặng/cần scale riêng → offload.

### ③ ⚠️ Cũ → Mới

| Cũ/sai | Nên dùng | Vì sao |
|---|---|---|
| `readFileSync`/`*Sync` trong handler | bản async (`fs/promises`) | sync block cả service |
| `JSON.parse` payload khổng lồ vô tư | streaming parser + giới hạn body | tránh block + DoS |
| Regex tự chế phức tạp trên input user | regex an toàn / RE2 / validate length + timeout | chống ReDoS |
| Tính toán nặng ngay trong handler | chunk→worker→offload theo độ nặng | giữ loop thông |

### ④ Áp dụng thực tế + bigtech
Ví dụ kinh điển: endpoint export báo cáo build PDF/Excel nặng ngay trong request → 1 user export làm **toàn bộ API treo**. Cách production: đẩy sang **job queue** (BullMQ + worker process), trả `202 Accepted` + job id, client poll/nhận webhook. Hashing mật khẩu (bcrypt/argon2) là CPU-bound → dùng bản async (đi qua pool, Bài 4) và/hoặc worker khi tải cao. *(pattern phổ biến.)*

### ⑤ Code thực hành
```js
// file: chunk-yield.mjs — G-BLK-005: tính nặng mà KHÔNG treo cả service (chưa cần worker)
async function heavySum(n) {
  let sum = 0;
  for (let i = 0; i < n; i++) {
    sum += Math.sqrt(i);
    // mỗi 1e6 vòng: nhường event loop để request khác xen vào
    if (i % 1_000_000 === 0) await new Promise(r => setImmediate(r));
  }
  return sum;            // ⚠️ vẫn chiếm 1 core; chỉ hết "treo", không nhanh hơn
}
```
```js
// file: redos-safe.mjs — chặn ReDoS
// ❌ nguy hiểm: /^(\w+\s?)*$/  với input dài → backtracking bùng nổ
// ✅ validate độ dài trước + regex tuyến tính, hoặc RE2:
function safeMatch(input) {
  if (input.length > 256) throw new Error('input too long'); // chặn đầu vào độc
  return /^[\w ]+$/.test(input);                              // không nested quantifier
}
```

### ⑥ Keywords
- 🧠 **HIỂU:** vì sao block treo TẤT CẢ request · giới hạn của chia-chunk (vẫn 1 core) · ReDoS = DoS qua regex.
- 📌 **THUỘC:** sync API OK ở startup không OK ở hot path · JSON lớn = sync CPU-bound · bậc thang chunk→worker→child_process→offload.
- 🛠️ **LÀM ĐƯỢC:** refactor vòng nặng bằng setImmediate · viết regex an toàn + validate length · nhận diện *Sync trong code.

### ⑦ Mental model
**"Một main thread cho tất cả: ai block, cả nhà chờ. Nặng nhẹ quyết định leo bậc thang chunk → worker → offload."**

### ⑧ Phỏng vấn & thách đố
1. CPU nặng gây gì cho server Node? → block loop → treo mọi request.
2. Kể các cách xử lý CPU-bound + khi nào dùng. → chunk/worker/child_process/offload theo độ nặng.
3. *(khó)* Chia chunk + setImmediate có tăng throughput tính toán không? → không; vẫn 1 core, chỉ hết treo.
4. **Thách đố:** "Endpoint validate bằng regex, thỉnh thoảng 1 request làm CPU 100% mãi — vì sao, fix?" → ReDoS (catastrophic backtracking); fix: validate length + regex an toàn/RE2/timeout.

### ⑨ Bài tập + tiêu chí
Viết handler tính tổng `5e8`: bản (a) block thẳng, (b) chunk-yield. Bắn 2 request song song. **Đạt khi:** chứng minh bản (a) làm request 2 treo, bản (b) thì không (dù vẫn lâu).

### ⑩ Đọc thêm
nodejs.org guide "Don't Block the Event Loop" · OWASP ReDoS · BullMQ docs (job queue).

> 🛂 *Kiểm tra AI:* AI hay "giải" CPU-bound bằng `async/await` — **bẫy**: await KHÔNG làm tính toán sync hết block (không có I/O để chờ). Phải là chia-chunk-yield, worker, hoặc offload. Kiểm tra: có điểm `await` nào thực sự nhường loop không?

---

## 🎓 BÀI 7 — Scale đa core: worker_threads · cluster · child_process
> Phủ: **G-WRK-001…007, G-CLU-001…007** (14 câu) · lời giải cho Bài 6, chủ đề TL.

### ① Mục tiêu & vị trí
Phân biệt rạch ròi **3 cơ chế đa nhiệm** và chọn đúng: `worker_threads` (thread trong 1 process), `cluster` (nhiều process cùng app), `child_process` (tiến trình ngoài/script khác). Biết khi nào để **K8s/orchestrator** scale thay vì tự cluster.

### ② Giảng cơ bản → nâng cao
**(a) Ba cơ chế — bản chất (G-CLU-001, 006):**

| | worker_threads | cluster | child_process |
|---|---|---|---|
| Đơn vị | **thread** trong 1 process | nhiều **process** cùng app Node | **tiến trình ngoài** (kể cả non-Node) |
| Event loop/V8 | mỗi worker có **isolate + loop riêng** (G-WRK-003) | mỗi process loop riêng | process riêng |
| Chia sẻ memory | có (SharedArrayBuffer) / message | **không** (process tách biệt) | không (IPC qua pipe) |
| Hợp với | **CPU-bound** + cần chia sẻ state | **scale request** đa core | chạy lệnh/binary/script khác |

`fork()` = `child_process` chuyên cho script Node (có kênh IPC). `cluster` xây trên `child_process.fork` + chia sẻ cổng listen.

**(b) worker_threads sâu (G-WRK):**
- **Memory (G-WRK-002):** mặc định **không share heap** — dữ liệu **copy qua message** (structured clone). Muốn share thật → **SharedArrayBuffer**; giao tiếp qua **MessagePort/postMessage**.
- **Transfer vs clone (G-WRK-006):** clone copy (tốn với data lớn); **transfer** `ArrayBuffer` chuyển **quyền sở hữu** (zero-copy, nhưng bên gửi mất quyền truy cập).
- **Chi phí + pool (G-WRK-004):** tạo worker tốn (init V8 isolate + RAM ~vài MB). Đừng tạo mới mỗi request → dùng **worker pool** (vd **piscina**) để tái dùng + kiểm soát concurrency.
- **I/O-bound thì vô ích (G-WRK-005):** I/O đã non-blocking trên 1 loop; worker chỉ giúp **CPU-bound**. Dùng worker cho I/O chỉ thêm overhead message.
- **Recovery (G-WRK-007):** bắt `'error'`/`'exit'`, đặt **timeout**, **retry/replace** worker trong pool; không để main treo chờ worker chết.

**(c) cluster sâu (G-CLU):**
- **Cơ chế (G-CLU-002):** **primary** fork N worker, **chia sẻ cổng listen**, phân phối kết nối (OS round-robin hoặc primary điều phối); primary giám sát/health/restart.
- **Số worker (G-CLU-003):** thường ~**số CPU core** (hoặc core−1); cân nhắc **CPU limit của container** và RAM mỗi worker (đừng fork 16 worker trong pod giới hạn 2 vCPU).
- **State (G-CLU-005):** in-memory **không share** giữa worker → session/cache phải để **store ngoài (Redis)**. **Sticky session** = route 1 client về 1 worker (band-aid cho state cục bộ, không nên là kiến trúc chính).
- **K8s (G-CLU-004) — câu TL:** trong container/K8s thường **để orchestrator scale pod** (1 process/pod) để có observability/rolling deploy/độc lập lỗi tốt hơn. Cluster hợp khi muốn **dùng hết nhiều core trong 1 pod lớn**. Biết trade-off, không máy móc.

**(d) child_process an toàn (G-CLU-007):** `exec` dùng **shell** + buffer toàn bộ output → **command injection** (nếu nối input user) + tràn buffer. `spawn` **stream** output + tránh shell (truyền args mảng) → an toàn hơn. Quy tắc: **không bao giờ** nối input user vào `exec`.

### ③ ⚠️ Cũ → Mới

| Cũ/sai | Nên dùng | Vì sao |
|---|---|---|
| Tạo worker mới mỗi request | **worker pool** (piscina) | tạo worker tốn; pool tái dùng |
| Dùng worker cho I/O-bound | giữ trên main loop (non-blocking) | worker chỉ giúp CPU-bound |
| cluster để scale trong K8s mặc định | thường để K8s scale pod | observability/rolling tốt hơn |
| State in-memory trong cluster | store ngoài (Redis) + ít dùng sticky | worker không share heap |
| `exec(\`cmd ${userInput}\`)` | `spawn(cmd, [args])` không shell | chống command injection |

### ④ Áp dụng thực tế + bigtech
- **worker pool (piscina)**: xử lý ảnh/sharp, hashing argon2, parse/transform CPU nặng — giữ API loop thông.
- **cluster vs K8s**: nhiều team microservice chạy **1 process Node / pod** và để K8s HPA scale theo CPU/ELU (Bài 9); cluster còn dùng trong môi trường VM lớn hoặc dùng `pm2` quản lý cluster. *(pattern phổ biến — verify theo hạ tầng team.)*
- **child_process**: gọi `ffmpeg`/binary xử lý media là use case kinh điển — luôn `spawn` với args, không `exec` chuỗi.

### ⑤ Code thực hành
```js
// file: worker-pool.mjs — dùng piscina làm worker pool cho CPU-bound (Node 24)
// npm i piscina@^5
import Piscina from 'piscina';
import { fileURLToPath } from 'node:url';

const pool = new Piscina({
  filename: fileURLToPath(new URL('./cpu-task.mjs', import.meta.url)),
  maxThreads: 4,           // kiểm soát concurrency, tái dùng worker
});
export const compute = (data) => pool.run(data);   // trả Promise, không block main loop
```
```js
// file: cpu-task.mjs — chạy TRONG worker (có loop + isolate riêng)
export default function ({ n }) {
  let s = 0; for (let i = 0; i < n; i++) s += Math.sqrt(i); return s;
}
```
```js
// file: safe-exec.mjs — child_process AN TOÀN
import { spawn } from 'node:child_process';
function convert(input) {
  // ✅ spawn + args mảng, KHÔNG shell, KHÔNG nối chuỗi input user
  const p = spawn('ffmpeg', ['-i', input, 'out.mp4']); // input đã validate path
  p.on('error', (e) => console.error('spawn fail', e));
  return p; // stream output, không buffer toàn bộ
}
// ❌ TUYỆT ĐỐI tránh: exec(`ffmpeg -i ${input} out.mp4`) → command injection
```
`requirements`/deps (verify version mới nhất khi học): `piscina@^5` *(// kiểm chứng tại github.com/piscinajs/piscina)*.

### ⑥ Keywords
- 🧠 **HIỂU:** thread vs process vs tiến-trình-ngoài · worker chỉ cho CPU-bound · cluster không share state · K8s-scale vs cluster trade-off.
- 📌 **THUỘC:** worker = isolate+loop riêng · cluster fork qua primary, chia sẻ cổng · số worker ~ số core · sticky session = route về 1 worker · exec(shell) vs spawn(no-shell).
- 🛠️ **LÀM ĐƯỢC:** dựng worker pool (piscina) · cấu hình cluster ~ số core · thay exec→spawn · xử lý worker crash (retry/replace).

### ⑦ Mental model
**"CPU-bound chia state → worker_threads (pool). Scale request đa core → cluster, nhưng trong K8s để pod scale. Gọi binary ngoài → spawn, đừng exec input user."**

### ⑧ Phỏng vấn & thách đố
1. cluster vs worker_threads — khác & khi nào? → process vs thread; scale request vs CPU-bound chia state.
2. Số worker cluster đặt bao nhiêu? → ~số core, trừ container limit + RAM.
3. *(khó)* worker_threads có làm I/O nhanh hơn không? → không; I/O đã non-blocking.
4. *(khó)* Trong K8s nên cluster hay để pod scale? → thường pod scale; cluster khi muốn dùng hết core trong pod lớn.
5. **Thách đố 1:** "Tạo `new Worker()` mỗi request, tải cao thì RAM nổ + chậm — vì sao, fix?" → tạo worker tốn; dùng pool tái dùng.
6. **Thách đố 2 (security):** "`exec('git log ' + branch)` với branch từ user — rủi ro?" → command injection; dùng `spawn('git', ['log', branch])`.

### ⑨ Bài tập + tiêu chí
Đẩy hàm tính nặng (Bài 6) sang **piscina pool**; bắn 8 request song song. **Đạt khi:** API vẫn trả health-check nhanh trong lúc tính, và pool giới hạn đúng concurrency (không tạo worker vô hạn).

### ⑩ Đọc thêm
nodejs.org API `worker_threads`, `cluster`, `child_process` · piscina docs · pm2 cluster mode.

> 🛂 *Kiểm tra AI:* AI hay đề xuất `cluster` cho mọi nhu cầu "scale" kể cả khi app đã ở K8s (thừa, làm rối observability), và hay viết `new Worker` mỗi request (rò RAM). Kiểm tra: đã ở orchestrator chưa? có pool chưa? workload là CPU hay I/O?

---

## 🎓 BÀI 8 — Streams & backpressure
> Phủ: **G-STR-001…007** · độc lập nhưng cộng hưởng với hiểu biết về RAM/loop.

### ① Mục tiêu & vị trí
Xử lý dữ liệu **theo chunk** thay vì nạp hết vào RAM; hiểu **backpressure** (cơ chế chống tràn bộ nhớ) và dùng `pipeline()` đúng cách. Kỹ năng bắt buộc cho file lớn, proxy, upload/download.

### ② Giảng cơ bản → nâng cao
**(a) Stream là gì (G-STR-001):** xử lý dữ liệu **từng mảnh (chunk)** khi nó tới, không chờ/giữ toàn bộ trong RAM. **4 loại:** Readable (đọc), Writable (ghi), Duplex (cả hai, vd socket), Transform (Duplex biến đổi data, vd gzip).

**(b) Khi nào dùng (G-STR-002):** file lớn, proxy/relay, upload/download, transform on-the-fly (nén/mã hóa) → tránh ngốn RAM (nạp file 2GB vào buffer = OOM) và **giảm time-to-first-byte** (gửi chunk đầu ngay, không chờ đọc xong).

**(c) Backpressure — khái niệm cốt lõi (G-STR-003):** khi **writable chậm hơn readable** (vd ghi mạng chậm, đọc đĩa nhanh), dữ liệu dồn vào buffer → buffer **phình** → ngốn RAM → có thể crash. **Backpressure** = tín hiệu bảo nguồn (readable) **chậm lại/tạm dừng** cho tới khi đích tiêu thụ kịp. Bỏ qua nó = bom RAM chậm.

**(d) pipe vs pipeline (G-STR-004):** `src.pipe(dst)` **tự điều phối backpressure** NHƯNG **không tự cleanup khi lỗi** (stream lỗi có thể rò file descriptor/treo). **`stream.pipeline(src, ...transforms, dst, cb)`** (hoặc bản Promise) xử lý lỗi + **đóng đúng cách mọi stream** → **khuyến nghị**.

**(e) Mode & tuning:**
- **flowing vs paused (G-STR-005):** *paused* = bạn chủ động `read()`; *flowing* = data tự đẩy qua sự kiện `'data'`. Chuyển flowing bằng `pipe()`/`on('data')`/`resume()`, về paused bằng `pause()`.
- **highWaterMark (G-STR-006):** ngưỡng buffer kích hoạt backpressure. Lớn → throughput cao nhưng ngốn RAM; nhỏ → tiết kiệm RAM nhưng nhiều lần chuyển.
- **async iterator (G-STR-007):** `for await (const chunk of readable)` cho code **tuần tự dễ đọc + backpressure tự nhiên**. Pitfall: phải tự lo **lỗi/cleanup** (nên kết hợp pipeline) và **không xử lý song song** trong vòng (mỗi vòng chờ chunk trước).

### ③ ⚠️ Cũ → Mới

| Cũ/sai | Nên dùng | Vì sao |
|---|---|---|
| `fs.readFile` cả file lớn rồi xử lý | stream + pipeline | tránh OOM, giảm TTFB |
| `a.pipe(b)` cho luồng production | `stream.pipeline(a, b)` (Promise) | pipe không cleanup khi lỗi |
| Bỏ qua giá trị trả về `write()` | tôn trọng backpressure (`write()===false` → chờ `'drain'`) | tránh phình buffer |
| Tự quản `'data'`/`'end'`/`'error'` tay | `for await…of` + pipeline | gọn, ít rò lỗi |

### ④ Áp dụng thực tế + bigtech
Proxy/gateway stream body từ client → upstream (không buffer toàn bộ) là chuẩn để chịu file lớn với RAM thấp. Upload S3, transcode media, export CSV nhiều triệu dòng đều dùng stream + pipeline. Frameworks (NestJS `StreamableFile`, Express response stream) bọc lại cơ chế này.

### ⑤ Code thực hành
```js
// file: gzip-pipeline.mjs — nén file lớn an toàn, đúng backpressure + cleanup (Node 24)
import { pipeline } from 'node:stream/promises';   // bản Promise: tự xử lý lỗi + đóng stream
import { createReadStream, createWriteStream } from 'node:fs';
import { createGzip } from 'node:zlib';

await pipeline(
  createReadStream('big.log'),      // Readable
  createGzip(),                     // Transform (nén on-the-fly)
  createWriteStream('big.log.gz'),  // Writable
);                                  // ✅ backpressure tự lo; lỗi ở bất kỳ stage → reject + đóng hết
```
```js
// file: consume-async.mjs — đọc tuần tự bằng async iterator (backpressure tự nhiên)
import { createReadStream } from 'node:fs';
let lines = 0;
for await (const chunk of createReadStream('big.log', { highWaterMark: 64 * 1024 })) {
  lines += chunk.toString().split('\n').length;  // mỗi vòng CHỜ chunk trước → không phình RAM
}
// ⚠️ pitfall: lỗi giữa chừng nên bọc try/finally; muốn song song phải tự gom rồi Promise.all (cẩn thận RAM)
```

### ⑥ Keywords
- 🧠 **HIỂU:** stream = xử lý theo chunk · backpressure chống tràn buffer · vì sao pipeline > pipe.
- 📌 **THUỘC:** 4 loại (Readable/Writable/Duplex/Transform) · flowing vs paused · highWaterMark = ngưỡng buffer · `stream.pipeline`.
- 🛠️ **LÀM ĐƯỢC:** nối stream bằng pipeline · đọc bằng `for await…of` · chỉnh highWaterMark · tôn trọng `write()` trả false.

### ⑦ Mental model
**"Đừng nuốt cả file — nhấp từng ngụm. Đích chậm thì nguồn chờ (backpressure). Nối stream thì pipeline để lỗi tự dọn."**

### ⑧ Phỏng vấn & thách đố
1. Stream là gì, mấy loại? → xử lý theo chunk; Readable/Writable/Duplex/Transform.
2. Backpressure là gì, bỏ qua thì sao? → buffer phình → OOM/crash.
3. *(khó)* pipe vs pipeline — vì sao pipeline khuyến nghị? → pipeline xử lý lỗi + đóng stream đúng.
4. **Thách đố:** "Service nén file rò RAM rồi crash khi file lớn dù dùng `pipe` — vì sao?" → lỗi giữa chừng không cleanup + có thể bỏ qua backpressure; chuyển sang `stream.pipeline` (Promise).

### ⑨ Bài tập + tiêu chí
Viết endpoint download nén file 1GB. **Đạt khi:** RAM process giữ thấp (không tăng theo kích thước file) và lỗi giữa chừng đóng stream sạch (không rò fd) — chứng minh bằng pipeline.

### ⑩ Đọc thêm
nodejs.org API `stream` (Backpressure section) · "Stream backpressure" guide · `stream/promises`.

> 🛂 *Kiểm tra AI:* AI hay sinh code `readFile` toàn bộ rồi xử lý (chạy đúng với file test nhỏ, **OOM với file prod lớn**), hoặc dùng `a.pipe(b)` không xử lý lỗi. Kiểm tra: dữ liệu có thể lớn không? lỗi giữa luồng có đóng stream không?

---

## 🎓 BÀI 9 — Diagnostics, profiling, GC & event-loop observability
> Phủ: **G-DIAG-001…009, G-EL-010, G-EL-011, G-RT-002** (12 câu) · cần vốn Bài 1–8 mới debug được.

### ① Mục tiêu & vị trí
Có **quy trình điều tra** memory leak / latency / loop lag bằng **dữ liệu**, không đoán. Biết công cụ (inspect, clinic, heap snapshot, ELU) và đọc được flamegraph; hiểu GC gây spike. Đây là phần phân biệt TL thật.

### ② Giảng cơ bản → nâng cao
**(a) Đo event loop lag/ELU (G-EL-010, G-EL-011) — KHÓA của bài:**
- Triệu chứng "latency vọt dù CPU thấp" → **không** vội "tối ưu code". Đo **event loop delay** bằng `perf_hooks.monitorEventLoopDelay()` (histogram) và **ELU** (`performance.eventLoopUtilization()`).
- **ELU** = tỉ lệ thời gian loop **bận** / tổng thời gian. Gần **1** = loop **bão hòa** → tín hiệu scale **tốt hơn CPU%** (CPU có thể thấp khi đang chờ pool/lock nhưng loop vẫn nghẽn). Dùng ELU làm metric autoscale (HPA) thay/bổ sung CPU.

**(b) Phân biệt loại bottleneck bằng dữ liệu (G-DIAG-007):**
- **CPU cao + loop lag cao** → đang chạy **sync blocking** (vòng nặng, JSON lớn, regex). Dùng **CPU profile/flamegraph** tìm hàm tốn.
- **CPU thấp + loop lag cao** → đang **chờ** (thread pool cạn — Bài 4, lock, downstream chậm). Soi pool/`UV_THREADPOOL_SIZE`, downstream latency.

**(c) Memory leak — quy trình (G-DIAG-001, 003, 004):**
1. Theo dõi **RSS/heapUsed** theo thời gian (metric); leak = đường đi lên không trả về.
2. Lấy **heap snapshot** (qua `--inspect` + DevTools, hoặc `v8.writeHeapSnapshot()`), chụp **2–3 lần** cách quãng.
3. **So sánh snapshot** (Comparison view), tìm **retained objects** tăng dần.
4. Đọc **shallow size** (bộ nhớ của chính object) vs **retained size** (tổng giải phóng được nếu xóa object đó — quan trọng để biết "ai giữ ai").
5. **Nguyên nhân phổ biến:** listener không gỡ (`emitter.on` lặp), biến **global/closure** giữ tham chiếu, **cache không bound**, **timer** treo, **Map/Array** không dọn. KHÔNG chỉ "restart".

**(d) Công cụ (G-DIAG-002, 005, 006):**
- `node --inspect` + **Chrome DevTools** (breakpoint, CPU profile, heap snapshot); `--prof` (V8 profiler).
- **clinic.js**: *Doctor* (chẩn đoán tổng quát, gợi loại "bệnh"), *Flame* (CPU flamegraph), *Bubbleprof* (async). `0x` cũng tạo flamegraph. *(// verify gói/tên hiện hành tại clinicjs.org.)*
- **Flamegraph (G-DIAG-006):** trục ngang = **tần suất/thời gian** stack (bar **rộng** = tốn nhiều), chiều cao = **độ sâu call stack**. Tìm "bar rộng" = hot function.

**(e) GC gây spike (G-RT-002):** V8 GC **generational** — *young (new space)* dọn nhanh bằng **Scavenge**; *old space* dọn bằng **Major GC (Mark-Sweep-Compact)** chậm hơn, có **stop-the-world** ngắn → **latency spike** lúc GC. Tune `--max-old-space-size` (MB) cho heap lớn; giảm rác (object pool, tránh tạo object khổng lồ liên tục) giảm tần suất major GC.

**(f) Production hygiene (G-DIAG-008, 009):**
- **Logging có cấu trúc:** JSON log + level + **correlation/trace id** (gắn với AsyncLocalStorage — Bài 5). Dùng **pino** (nhanh, async) /winston; **tránh `console.log` đồng bộ ở hot path** (block + chậm).
- **Graceful shutdown (G-DIAG-009):** bắt **SIGTERM/SIGINT** → ngừng nhận request mới (`server.close()`) → **drain in-flight** → đóng DB/queue/pool → **timeout cưỡng bức** nếu treo. Bắt buộc cho **rolling deploy K8s** (pod nhận SIGTERM khi bị thay).

### ③ ⚠️ Cũ → Mới

| Cũ/sai | Nên dùng | Vì sao |
|---|---|---|
| Theo dõi chỉ **CPU%** để scale | thêm **ELU + event loop delay** | CPU thấp vẫn có thể nghẽn loop |
| "Leak thì cứ restart định kỳ" | heap snapshot + so sánh tìm retained | chữa gốc, không giấu bug |
| `console.log` khắp hot path | structured logger (pino) + trace id | console.log sync chậm/block |
| Kill pod thẳng (SIGKILL) | graceful shutdown SIGTERM + drain | tránh rớt request khi deploy |

### ④ Áp dụng thực tế + bigtech
- **ELU-based autoscaling:** export ELU làm custom metric → K8s HPA scale khi loop bão hòa (chuẩn hơn CPU cho I/O service). *(pattern phổ biến.)*
- **pino + trace id** là stack logging phổ biến cho Node ở quy mô lớn; gắn OpenTelemetry để trace phân tán.
- **Graceful shutdown** là yêu cầu cứng trong mọi Helm chart Node nghiêm túc (preStop hook + terminationGracePeriod).

### ⑤ Code thực hành
```js
// file: elu-monitor.mjs — đo loop lag + ELU để cảnh báo/scale (Node 24)
import { monitorEventLoopDelay, performance } from 'node:perf_hooks';

const h = monitorEventLoopDelay({ resolution: 20 });
h.enable();
let last = performance.eventLoopUtilization();

setInterval(() => {
  const elu = performance.eventLoopUtilization(last); last = performance.eventLoopUtilization();
  console.log(JSON.stringify({
    loopDelayP99ms: +(h.percentile(99) / 1e6).toFixed(1), // delay phân vị 99
    elu: +elu.utilization.toFixed(3),                      // ~1 => bão hòa => scale
  }));
  h.reset();
}, 1000).unref();
```
```js
// file: graceful.mjs — graceful shutdown đúng chuẩn cho K8s
function shutdown(server, deps) {
  let timer;
  const close = async (sig) => {
    console.log(`got ${sig}, draining...`);
    server.close(async () => {                 // 1) ngừng nhận request mới + drain in-flight
      await Promise.allSettled(deps.map(d => d.close())); // 2) đóng DB/queue/pool
      clearTimeout(timer); process.exit(0);
    });
    timer = setTimeout(() => process.exit(1), 10_000).unref(); // 3) cưỡng bức nếu treo
  };
  process.on('SIGTERM', () => close('SIGTERM'));
  process.on('SIGINT', () => close('SIGINT'));
}
```
Deps gợi ý (verify version khi học): `pino` cho logging. `// kiểm chứng API clinic tại clinicjs.org`.

### ⑥ Keywords
- 🧠 **HIỂU:** ELU vs CPU% · CPU+lag (blocking sync) vs lag-không-CPU (chờ pool/lock) · GC generational gây spike · vì sao graceful shutdown cần cho K8s.
- 📌 **THUỘC:** `monitorEventLoopDelay`, `eventLoopUtilization` · shallow vs retained size · clinic (Doctor/Flame/Bubbleprof) · `--max-old-space-size` · SIGTERM/SIGINT.
- 🛠️ **LÀM ĐƯỢC:** đo ELU/loop delay · lấy & so sánh heap snapshot · đọc flamegraph · viết graceful shutdown · setup pino + trace id.

### ⑦ Mental model
**"Đừng đoán — đo. ELU bão hòa = scale. CPU cao+lag = sync blocking; CPU thấp+lag = đang chờ. Leak thì so snapshot, deploy thì drain trước khi chết."**

### ⑧ Phỏng vấn & thách đố
1. Quy trình điều tra memory leak? → RSS/heap → snapshot → so sánh retained → tìm nguyên nhân (listener/cache/closure/timer).
2. shallow vs retained size? → của chính object vs tổng giải phóng được nếu xóa nó.
3. *(khó)* Latency vọt mà CPU thấp — đo gì, nghi gì? → ELU/loop delay; nghi chờ pool/lock/downstream, không phải tính toán.
4. *(khó)* Vì sao GC gây latency spike? → major GC (old space) stop-the-world ngắn.
5. **Thách đố:** "Rolling deploy K8s thỉnh thoảng rớt vài request 5xx — vì sao, fix?" → thiếu graceful shutdown: pod SIGTERM nhưng còn request in-flight bị cắt; thêm `server.close` + drain + preStop.

### ⑨ Bài tập + tiêu chí
Thêm `elu-monitor` + `graceful shutdown` vào 1 HTTP server. Bắn tải rồi gửi SIGTERM giữa chừng. **Đạt khi:** log ELU phản ánh tải, và shutdown drain hết request đang chạy (0 request bị cắt) trước khi exit.

### ⑩ Đọc thêm
nodejs.org Diagnostics guide · perf_hooks API (eventLoopUtilization) · clinicjs.org · pino docs · "Graceful shutdown in Node.js".

> 🛂 *Kiểm tra AI:* AI hay chẩn đoán "latency cao = tối ưu thuật toán" mà bỏ qua khả năng **chờ pool/GC/downstream**. Và hay quên graceful shutdown trong Dockerfile/handler. Kiểm tra: đã đo ELU/loop delay chưa? có bắt SIGTERM chưa?

---

## 🎓 BÀI 10 — Security hardening
> Phủ: **G-SEC-001…009** · tay nghề production, đụng nhiều mục khác (J caching/Redis, F API).

### ① Mục tiêu & vị trí
Hardening một Node/Express/Nest app **theo lớp** (defense in depth): headers, validate input, rate limit, CORS, secrets, hashing đúng, supply-chain, và Permission Model runtime. Nêu được **tên gói cụ thể**.

### ② Giảng cơ bản → nâng cao
**(a) Các lớp hardening (G-SEC-001):** không có "1 viên đạn bạc" — xếp lớp:
1. **Security headers** → `helmet`.
2. **Validate/sanitize input** → `zod`/`joi`/`class-validator`.
3. **Rate limiting** → `rate-limiter-flexible`/`express-rate-limit`.
4. **CORS chặt** (allowlist origin, không `*` với credentials).
5. **Secrets ngoài code** → env + secret manager.
6. **HTTPS/TLS** (thường ở ingress/LB).

**(b) helmet làm gì / KHÔNG làm (G-SEC-002):** **set security headers** (CSP, HSTS, X-Frame-Options, X-Content-Type-Options…). **KHÔNG** phải firewall/auth/validation — đừng tưởng "thêm helmet là an toàn". Nó là **1 lớp**.

**(c) Rate limiting đa instance (G-SEC-003):** per-user/IP bằng `rate-limiter-flexible`/`express-rate-limit`. **Khi scale ngang** (nhiều pod/worker), in-memory counter **không share** → mỗi instance đếm riêng = giới hạn sai. **Lưu state ở store dùng chung (Redis)**.

**(d) CSRF (G-SEC-004):** tấn công lợi dụng **cookie tự gửi** theo request. **CẦN** phòng khi auth bằng **cookie/session + form** (CSRF token, SameSite cookie). **THƯỜNG KHÔNG cần** khi API **stateless dùng Bearer token** trong header (trình duyệt không tự gắn header token). Hiểu *vì sao*, không bật/tắt máy móc.

**(e) Injection (G-SEC-005):**
- **SQL injection:** **parameterized query**/ORM, **không** nối string xây query. Validate kiểu.
- **NoSQL injection:** chặn **object operator injection** (vd `{$gt:''}` trộn vào filter Mongo) — ép kiểu input, không nhét object user thẳng vào query.

**(f) Lưu mật khẩu (G-SEC-007):** dùng **bcrypt/argon2** (chậm có chủ đích + **salt** tự động). KHÔNG dùng **SHA-256 thuần** (nhanh → brute-force dễ, không salt → rainbow table). **argon2** ưu tiên hiện đại (memory-hard). *(là CPU-bound → cân nhắc pool/worker, Bài 4&7.)*

**(g) Secrets (G-SEC-008):** không hardcode/không commit. `.env` (gitignore) cho dev; **secret manager** (Vault/AWS Secrets Manager/GCP Secret Manager) cho prod; **rotate** định kỳ; không log secrets.

**(h) Supply-chain npm (G-SEC-006):** giảm rủi ro deps độc/lỗ hổng: **`npm audit`**, **lockfile + `npm ci`** (cài đúng lock, không tự nâng), **Snyk/Dependabot** quét, **pin version**, soi **postinstall script** (vector tấn công), **tối thiểu hóa deps**.

**(i) Permission Model runtime (G-SEC-009):** Node có **`--permission`** giới hạn quyền **fs/child_process/worker** ở **cấp process** → phòng thủ chiều sâu cho code/deps không tin cậy (nếu 1 dep bị chiếm, nó vẫn không đọc được fs ngoài phạm vi cho phép). *(// verify trạng thái flag — đã ổn định ở Node LTS hiện hành chưa, tại nodejs.org/api/permissions.)*

### ③ ⚠️ Cũ → Mới

| Cũ/sai | Nên dùng | Vì sao |
|---|---|---|
| SHA-256/MD5 hash mật khẩu | **argon2** (hoặc bcrypt) + salt | hash nhanh = brute-force dễ |
| Nối string xây SQL | parameterized/ORM | chống SQLi |
| Nhét object user thẳng vào query Mongo | ép kiểu, chặn operator | chống NoSQLi |
| Rate limit in-memory khi đa instance | store Redis chung | đếm đúng khi scale |
| `npm install` tùy tiện trên CI | `npm ci` + lockfile + audit | build tái lập + bắt lỗ hổng |
| Hardcode key / commit .env | secret manager + gitignore + rotate | tránh lộ secret |

### ④ Áp dụng thực tế + bigtech
Stack hardening điển hình của API Node: `helmet` + `zod` validate + `rate-limiter-flexible` (Redis store) + CORS allowlist + argon2 + secrets qua Vault/Secrets Manager + Dependabot/Snyk + `npm ci` trong CI. Permission Model + container (non-root, read-only fs, drop capabilities) là lớp runtime. *(pattern phổ biến — verify gói/cờ mới nhất khi triển khai.)*

### ⑤ Code thực hành
```js
// file: harden.mjs — các lớp hardening cho Express (Node 24)
// npm i helmet express-rate-limit rate-limiter-flexible argon2 zod
import helmet from 'helmet';
import argon2 from 'argon2';
import { z } from 'zod';

app.use(helmet());                                  // lớp 1: security headers
app.use(express.json({ limit: '100kb' }));          // chặn body khổng lồ (Bài 6: JSON lớn)

const cors = { origin: ['https://app.example.com'], credentials: true }; // allowlist, không '*'

const Login = z.object({ email: z.string().email(), pw: z.string().min(8).max(128) });
app.post('/login', async (req, res) => {
  const { email, pw } = Login.parse(req.body);      // validate trước khi dùng
  const hash = await argon2.hash(pw);               // ✅ slow + salt tự động (KHÔNG SHA-256)
  // ... parameterized query, KHÔNG nối string:  db.query('SELECT * FROM u WHERE email=$1',[email])
});
```
```js
// file: ratelimit-redis.mjs — rate limit DÙNG CHUNG khi nhiều instance
// npm i rate-limiter-flexible ioredis
import { RateLimiterRedis } from 'rate-limiter-flexible';
import Redis from 'ioredis';
const limiter = new RateLimiterRedis({               // state ở Redis → đúng khi scale ngang
  storeClient: new Redis(process.env.REDIS_URL),     // secret qua env, không hardcode
  points: 100, duration: 60,                          // 100 req / 60s / key
});
// middleware: await limiter.consume(req.ip) → 429 nếu vượt
```
CI: `npm ci` (không `npm install`), `npm audit --audit-level=high` làm gate. Pin deps qua lockfile. `// verify version gói tại npmjs.com khi học`.

### ⑥ Keywords
- 🧠 **HIỂU:** defense in depth (nhiều lớp) · vì sao SHA thuần sai cho password · khi nào CSRF cần/không · rate limit phải share state khi scale.
- 📌 **THUỘC:** helmet=headers (≠ firewall/auth) · argon2/bcrypt + salt · parameterized query · `npm ci`+lockfile+audit · `--permission`.
- 🛠️ **LÀM ĐƯỢC:** lắp helmet+validate+rate-limit(Redis)+CORS · hash argon2 · viết parameterized query · setup Dependabot/`npm ci` gate.

### ⑦ Mental model
**"Không có viên đạn bạc — xếp lớp. Password thì argon2+salt, query thì tham số hóa, rate limit thì Redis, secret thì manager, deps thì `npm ci`+audit."**

### ⑧ Phỏng vấn & thách đố
1. Hardening gồm lớp gì, gói nào? → headers/validate/ratelimit/CORS/secrets/HTTPS + tên gói.
2. helmet KHÔNG làm gì? → không phải auth/firewall/validation.
3. Khi nào KHÔNG cần CSRF? → API stateless Bearer token header.
4. *(khó)* Vì sao không SHA-256 cho password? → nhanh + không salt → brute-force/rainbow.
5. **Thách đố 1:** "Rate limit hoạt động khi 1 instance, nhưng scale 5 pod thì user spam được gấp 5 — vì sao?" → in-memory counter per-instance; chuyển store Redis.
6. **Thách đố 2 (supply-chain):** "CI dùng `npm install` thỉnh thoảng kéo về version mới có lỗ hổng — fix quy trình?" → `npm ci` theo lockfile + `npm audit` gate + Dependabot.

### ⑨ Bài tập + tiêu chí
Hardening 1 endpoint login. **Đạt khi:** có helmet, validate (zod), rate limit qua Redis, argon2 hash, query tham số hóa, secret qua env, và CI dùng `npm ci`+audit. Giải thích mỗi lớp chặn tấn công nào.

### ⑩ Đọc thêm
OWASP Node.js Security Cheat Sheet · helmet/argon2/rate-limiter-flexible docs · nodejs.org Permissions API · GitHub Dependabot.

> 🛂 *Kiểm tra AI:* AI thường (1) hash password bằng `crypto.createHash('sha256')`, (2) rate limit in-memory, (3) bật CSRF cho API Bearer-token (thừa) hoặc tắt cho cookie-session (thiếu), (4) nối string SQL. Đây là 4 bẫy review thường gặp — kiểm tra từng cái.

---

## 🎓 BÀI 11 — TL runtime depth: module system & version strategy
> Phủ: **G-RT-003, G-RT-004** · đóng gói tư duy Tech Lead.

### ① Mục tiêu & vị trí
Hai quyết định cấp TL: (1) hiểu **CommonJS vs ESM** đủ để dẫn dắt migrate, (2) **chọn Node version cho production** đúng nguyên tắc — và giải thích được *vì sao không chọn bản mới nhất*.

### ② Giảng cơ bản → nâng cao
**(a) CommonJS (require) vs ESM (import) (G-RT-003):**
- **Loading:** CJS **đồng bộ** (`require` chạy ngay, có thể giữa file); ESM **bất đồng bộ + tĩnh** (phân tích import trước, hỗ trợ **top-level await**).
- **Khác biệt thực tế:** ESM không có `__dirname`/`__filename` sẵn (dùng `import.meta.url`/`import.meta.filename`); CJS không có top-level await; **interop** CJS↔ESM có gờ (named export từ CJS, `require()` ESM…).
- **Bật ESM:** `"type": "module"` trong package.json, hoặc đuôi `.mjs` (CJS = `.cjs`).
- **Điểm vấp migrate:** path/`__dirname`, dynamic `require`, deps chỉ-CJS hoặc chỉ-ESM, jest/ts-node config, conditional exports trong `package.json`.

**(b) Chọn Node version production 6/2026 (G-RT-004) — câu TL kinh điển:**
- Chọn **Active LTS = Node 24 (Krypton)**, KHÔNG chọn **Current = Node 26** (chưa ổn định, lên LTS 10/2026), KHÔNG dùng **Node 20** (đã EOL 30/4/2026 → hết vá bảo mật).
- **Vì sao không bản mới nhất:** Current là giai đoạn ổn định hóa cho library author thử nghiệm; production cần **ổn định + còn được vá bảo mật lâu** → chỉ chạy **Active/Maintenance LTS**.
- **Biết release cycle đang đổi:** từ **Node 27 (10/2026)**: 1 major/năm (tháng 4), **mọi release thành LTS**, version theo năm → kế hoạch nâng cấp đơn giản hơn nhưng phải cập nhật policy. *(verify nodejs.org.)*
- **Tay nghề kèm theo:** pin qua `engines` + `.nvmrc`; base image `node:24-alpine`; giữ image `node:22` để rollback; cập nhật SBOM/scan khi đổi baseline.

### ③ ⚠️ Cũ → Mới

| Cũ/sai | Nên dùng (6/2026) | Vì sao |
|---|---|---|
| Pin `node:20-*` cho prod mới | `node:24-*` (Active LTS) | Node 20 EOL 30/4/2026 |
| "Dùng bản mới nhất (26) cho mạnh" | Active LTS, không Current | Current chưa ổn định, vá ngắn |
| CJS-only mindset (`require` mọi nơi) | hiểu ESM + interop, dùng `import.meta` | hệ sinh thái đang dịch sang ESM |
| `__dirname` trong ESM | `import.meta.dirname`/`url` | ESM không có sẵn `__dirname` |

### ④ Áp dụng thực tế + bigtech
Chuẩn TL: khai báo **version policy rõ trong package.json `engines` + release notes**, chuẩn hóa base image lên Node 24, giữ Node 22 cho rollback tới khi migrate xong; chỉ raise minimum khi cần feature v24-only. *(theo hướng dẫn nâng cấp 2026 — verify khi áp dụng.)* Nhiều dự án mới chọn ESM từ đầu; codebase lớn cũ thường migrate dần qua dual-package/conditional exports.

### ⑤ Code thực hành
```js
// file: package.json — pin version + bật ESM
{
  "type": "module",
  "engines": { "node": ">=24 <25" },     // chỉ chạy Active LTS line
  "scripts": { "start": "node src/index.mjs" }
}
```
```js
// file: esm-paths.mjs — thay thế __dirname trong ESM
import { dirname } from 'node:path';
import { fileURLToPath } from 'node:url';
const __dirname = dirname(fileURLToPath(import.meta.url)); // hoặc Node mới: import.meta.dirname
```
```dockerfile
# Dockerfile — baseline Node 24 LTS, non-root cho security (Bài 10)
FROM node:24-alpine
WORKDIR /app
COPY package*.json ./
RUN npm ci --omit=dev          # npm ci theo lockfile (Bài 10 supply-chain)
COPY . .
USER node                       # không chạy root
CMD ["node", "src/index.mjs"]
```
`.nvmrc`: `24`. `// verify Node LTS hiện hành tại nodejs.org/en/about/previous-releases khi học`.

### ⑥ Keywords
- 🧠 **HIỂU:** CJS sync vs ESM async/tĩnh · vì sao prod chỉ dùng LTS không dùng Current · release cycle mới (Node 27+).
- 📌 **THUỘC:** `"type":"module"`/.mjs/.cjs · `import.meta.url`/`dirname` · Node 24=Active LTS, 22=Maintenance(EOL 4/2027), 20=EOL, 26=Current(LTS 10/2026).
- 🛠️ **LÀM ĐƯỢC:** pin qua engines+.nvmrc · thay __dirname trong ESM · viết Dockerfile node:24 non-root · lập version policy.

### ⑦ Mental model
**"Production ăn LTS, không ăn Current. Node 24 cho dự án mới 6/2026. ESM là tương lai — hiểu interop để dẫn migrate."**

### ⑧ Phỏng vấn & thách đố
1. CJS vs ESM khác gì, vấp gì khi migrate? → sync vs async/tĩnh; __dirname/top-level await/interop.
2. Chọn Node version cho prod mới 6/2026? → Node 24 (Active LTS); không 26 (Current), không 20 (EOL).
3. *(khó)* Vì sao không luôn dùng bản mới nhất? → Current chưa ổn định + vá ngắn; prod cần ổn định + LTS lâu.
4. **Thách đố:** "Team muốn nhảy thẳng Node 26 cho prod vì 'có Temporal API' — bạn phản biện thế nào?" → 26 là Current tới 10/2026, rủi ro ổn định + chu kỳ vá; chờ LTS hoặc cô lập feature; cân nhắc cost/benefit, không vì 1 API mà hi sinh ổn định.

### ⑨ Bài tập + tiêu chí
Setup repo mới: ESM + `engines` pin Node 24 + `.nvmrc` + Dockerfile `node:24-alpine` non-root + `npm ci`. **Đạt khi:** giải thích từng lựa chọn (vì sao 24, vì sao ESM, vì sao non-root) và biết khi nào rollback về 22.

### ⑩ Đọc thêm
nodejs.org "Previous Releases" + "Evolving the Release Schedule" · Node.js Modules: ECMAScript modules · endoflife.date/nodejs.

> 🛂 *Kiểm tra AI:* AI (do training cutoff) hay đề xuất Node version **lỗi thời** (vd "dùng Node 20 LTS") hoặc khuyên bản mới nhất mà không phân biệt Current vs LTS. LUÔN verify LTS hiện hành tại nodejs.org trước khi chốt version trong Dockerfile/CI.

---

# ✅ Sau Bước C — bản đồ ôn tập 1 trang

| Bài | Chốt 1 câu | Bẫy AI hay mắc |
|---|---|---|
| 1 Runtime | JS 1 thread, runtime đa luồng (V8≠libuv) | "Node multi-threaded so it scales" |
| 2 Phases | 6 trạm: …poll(việc chính)→check | mô tả kiểu browser, quên poll |
| 3 Ordering | nextTick→microtask→phase kế | quên ngữ cảnh phase top-level vs I/O |
| 4 Thread pool | 4 thread: fs/crypto/dns.lookup/zlib; network=OS | "tăng pool luôn nhanh hơn" |
| 5 Async | all/allSettled/race/any; đừng floating | Promise.all không giới hạn concurrency |
| 6 Blocking | ai block, cả nhà chờ; chunk→worker→offload | "await" fix CPU-bound (sai) |
| 7 Scale | thread(worker)/process(cluster)/ngoài(child) | cluster trong K8s + worker mỗi request |
| 8 Streams | nhấp từng ngụm; pipeline > pipe | readFile cả file lớn → OOM |
| 9 Diag | đo ELU; leak→so snapshot; drain khi deploy | "latency cao = tối ưu thuật toán" |
| 10 Security | xếp lớp; argon2/param-query/Redis-ratelimit | sha256 password, ratelimit in-memory |
| 11 TL version | prod ăn LTS (Node 24), không Current | đề xuất version lỗi thời |

## 📌 Tiếp theo (theo WORKFLOW 2)
- **Bước D — phỏng vấn chấm live:** mỗi đợt 10–15 câu trộn dễ→khó, **hỏi xong DỪNG** chờ bạn trả lời (recall trước, đáp án sau), rồi tôi dựng đáp án chuẩn + bẫy + trade-off bám đúng dòng "dò cái gì" để chấm ĐẠT/CHƯA.
- **Bước E — cổng hoàn thành:** ĐẠT toàn bộ 84 câu mới tính xong mục G; các đợt sau chèn 1–2 câu cũ (spaced repetition: hôm nay → mai → 3 ngày → 1 tuần).

> Nói **"bắt đầu Bước D"** (chọn đợt theo bài/độ khó nếu muốn) để vào phỏng vấn, hoặc **"giảng kỹ lại Bài X"** nếu cần đào sâu chỗ nào trước.
