# 🧪 QB_G — Ngân hàng câu hỏi: Node.js Runtime Depth

> **Mục G** trong roadmap Tech Lead Backend (🟡 ~60%) · Sinh theo **WORKFLOW 2 — Bước A**.
> Đây là **mục LỚN** → bộ đề phủ HẾT 10 mục con + thêm nhóm runtime/V8/GC để đủ chiều sâu TL.
> **Chỉ câu hỏi — KHÔNG kèm đáp án.** Mỗi câu có dòng *"dò cái gì"* = tiêu chí ĐẠT, dùng để chấm **live** ở Bước D.
> **Tổng: 84 câu.**

## Cách đọc
- **ID:** `G-<tag>-<số>`. Tag mục con liệt kê ở mục lục dưới.
- **Độ khó:** ⭐ định nghĩa cơ bản → ⭐⭐⭐⭐⭐ phân biệt senior với TL thật.
- **"Dò cái gì":** điều người trả lời PHẢI chạm tới mới tính ĐẠT (không phải đáp án — chỉ là mốc chấm).

## Bối cảnh phiên bản (tính tại 6/2026 — *verify lại khi học vì release cycle đang đổi*)
- **Node 24 (Krypton)** = Active LTS → mặc định production. **Node 22 (Jod)** = Maintenance LTS (đến 4/2027).
- **Node 20 (Iron)** đã **EOL 30/4/2026** → tránh dùng. **Node 26** = Current (5/2026), lên LTS 10/2026.
- Từ Node 27 (10/2026): mỗi năm 1 major (tháng 4), mọi release đều thành LTS — *kiểm chứng tại nodejs.org/en/about/previous-releases.*

---

## Mục lục mục con (ID prefix → số câu)

| Tag | Mục con | Số câu |
|---|---|---|
| **G-EL** | Event loop (phases, microtask/macrotask) | 11 |
| **G-TIM** | nextTick / setImmediate / setTimeout — thứ tự | 7 |
| **G-LU** | Single-thread vs thread pool (libuv) | 8 |
| **G-BLK** | Block event loop & xử lý CPU nặng | 6 |
| **G-WRK** | worker_threads vs offload (production) | 7 |
| **G-CLU** | cluster vs worker_threads vs child_process | 7 |
| **G-ASY** | Async patterns (callback/Promise/async-await) | 9 |
| **G-DIAG** | Diagnostics & profiling (memory leak, --inspect, clinic) | 9 |
| **G-SEC** | Node security hardening | 9 |
| **G-STR** | Streams & backpressure | 7 |
| **G-RT** | Runtime/V8/GC/module/versioning (chiều sâu TL) | 4 |

---

## G-EL — Event loop

**G-EL-001** ⭐⭐⭐
"Giải thích event loop trong Node.js." (câu kinh điển — mở màn gần như mọi vòng)
> *Dò cái gì:* hiểu event loop là vòng lặp single-thread điều phối callback non-blocking, phối hợp call stack + queue + libuv; KHÔNG nói lệch thành "đa luồng".

**G-EL-002** ⭐⭐⭐⭐
"Kể tên các phase của event loop theo đúng thứ tự và mỗi phase xử lý gì."
> *Dò cái gì:* gọi đúng chuỗi timers → pending callbacks → idle/prepare → poll → check → close callbacks, và hiểu phần lớn code chạy ở **poll**.

**G-EL-003** ⭐⭐⭐⭐
"Microtask queue chạy ở thời điểm nào so với các phase? nextTick queue và Promise queue khác nhau ra sao?"
> *Dò cái gì:* microtask (Promise) + nextTickQueue được drain **giữa mỗi phase / sau mỗi callback**, nextTick ưu tiên trước microtask.

**G-EL-004** ⭐⭐⭐
"Macrotask vs microtask — phân loại setTimeout, Promise.then, process.nextTick, I/O callback, setImmediate vào đâu?"
> *Dò cái gì:* phân loại đúng và hiểu microtask drain hết trước khi sang macrotask kế.

**G-EL-005** ⭐⭐⭐⭐
"Poll phase làm gì khi queue rỗng? Khi nào nó block, khi nào nhảy sang check?"
> *Dò cái gì:* poll chờ I/O; nếu có setImmediate đang chờ thì sang check, nếu có timer sắp tới hạn thì quay lại timers, không thì block chờ I/O.

**G-EL-006** ⭐⭐⭐⭐⭐
"Vì sao một Promise.then() lồng vô hạn (hoặc đệ quy nextTick) có thể làm 'đói' (starve) event loop?"
> *Dò cái gì:* microtask/nextTick drain TRỌN trước khi event loop tiến phase tiếp → đệ quy vô tận chặn timers/I/O mãi mãi.

**G-EL-007** ⭐⭐⭐
"Event loop của Node khác event loop của trình duyệt ở điểm nào?"
> *Dò cái gì:* khác về phase (Node có check/poll/libuv), có process.nextTick, dùng libuv thay vì các API browser.

**G-EL-008** ⭐⭐⭐⭐
"Cho đoạn code trộn setTimeout(0), setImmediate, Promise.resolve().then, process.nextTick trong một I/O callback — output theo thứ tự nào và vì sao?"
> *Dò cái gì:* suy luận thứ tự dựa trên phase hiện tại + microtask drain, không đoán mò.

**G-EL-009** ⭐⭐⭐
"'Tick' của event loop nghĩa là gì? Một tick gồm những gì?"
> *Dò cái gì:* một vòng đi qua các phase; phân biệt với process.nextTick.

**G-EL-010** ⭐⭐⭐⭐⭐
"Một endpoint của bạn thỉnh thoảng latency tăng vọt dù CPU thấp. Bạn nghi event loop lag — kiểm chứng và định lượng bằng cách nào?"
> *Dò cái gì:* đo **event loop lag/delay** (perf_hooks `monitorEventLoopDelay`, metric ELU/event loop utilization), không chỉ nói "tối ưu code".

**G-EL-011** ⭐⭐⭐
"Event loop utilization (ELU) là gì và dùng nó để quyết định scale như thế nào?"
> *Dò cái gì:* ELU = tỉ lệ thời gian loop bận; gần 1 nghĩa là bão hòa → scale ngang/offload, một tín hiệu tốt hơn CPU%.

---

## G-TIM — nextTick / setImmediate / setTimeout

**G-TIM-001** ⭐⭐⭐
"process.nextTick vs setImmediate khác nhau thế nào? Cái nào chạy trước?"
> *Dò cái gì:* nextTick chạy ngay sau operation hiện tại (trước khi loop tiến phase); setImmediate chạy ở phase check vòng sau.

**G-TIM-002** ⭐⭐⭐
"setTimeout(fn, 0) vs setImmediate(fn) — cái nào chạy trước? Câu trả lời có luôn ổn định không?"
> *Dò cái gì:* ở top-level thứ tự **không xác định** (tùy timing); nhưng trong một I/O callback thì setImmediate luôn trước setTimeout(0).

**G-TIM-003** ⭐⭐⭐⭐
"Vì sao trong một I/O callback, setImmediate luôn chạy trước setTimeout(0)?"
> *Dò cái gì:* sau poll là check (setImmediate) trước khi vòng sau quay lại timers.

**G-TIM-004** ⭐⭐⭐⭐
"setTimeout có đảm bảo chạy đúng sau N ms không? Vì sao delay thực tế có thể lớn hơn?"
> *Dò cái gì:* delay là **tối thiểu**, không đảm bảo; loop bận/blocked sẽ trễ.

**G-TIM-005** ⭐⭐⭐⭐
"process.nextTick dùng đúng vào việc gì, và lạm dụng nó gây hại gì?"
> *Dò cái gì:* dùng để defer ngay/đảm bảo API async-consistent; lạm dụng/đệ quy gây starve I/O.

**G-TIM-006** ⭐⭐⭐
"queueMicrotask() là gì, khác process.nextTick và Promise.then ở đâu?"
> *Dò cái gì:* queueMicrotask đẩy vào microtask queue (sau nextTick, ngang Promise); chuẩn hơn dùng Promise.resolve().then chỉ để defer.

**G-TIM-007** ⭐⭐⭐⭐⭐
"Bạn thấy code dùng setTimeout để 'chờ DB sẵn sàng/đợi state'. Vì sao đây là anti-pattern và sửa thế nào?"
> *Dò cái gì:* nhận ra race do đoán timing; thay bằng event/Promise/readiness check thực sự.

---

## G-LU — Single-thread vs thread pool (libuv)

**G-LU-001** ⭐⭐⭐
"Node có thật sự single-thread không?"
> *Dò cái gì:* JS chạy single-thread, nhưng libuv có thread pool + I/O nền → tổng thể đa luồng dưới nắp.

**G-LU-002** ⭐⭐⭐
"libuv là gì và đảm nhận những gì trong Node?"
> *Dò cái gì:* C library lo event loop, async I/O đa nền tảng, thread pool, abstraction cho fs/dns/crypto.

**G-LU-003** ⭐⭐⭐⭐
"Những operation nào dùng thread pool của libuv, những cái nào KHÔNG?"
> *Dò cái gì:* dùng pool: fs, một số crypto (pbkdf2…), dns.lookup, zlib; network socket dùng cơ chế OS (epoll/io_uring/kqueue) chứ không phải pool.

**G-LU-004** ⭐⭐⭐⭐
"Thread pool mặc định bao nhiêu thread? Đổi bằng gì và đổi để làm gì?"
> *Dò cái gì:* mặc định 4, đổi qua `UV_THREADPOOL_SIZE`; tăng khi nhiều fs/crypto đồng thời, hiểu trade-off CPU.

**G-LU-005** ⭐⭐⭐⭐⭐
"Demo crypto.pbkdf2 chạy song song 4 lời gọi rồi tới lần thứ 5 chậm hẳn — giải thích."
> *Dò cái gì:* pool 4 thread bão hòa → request thứ 5 phải xếp hàng; minh họa cạnh tranh thread pool.

**G-LU-006** ⭐⭐⭐⭐
"Vì sao file I/O 'async' đôi khi vẫn dùng thread pool còn network I/O thì không?"
> *Dò cái gì:* OS không có async fs nhất quán → libuv mô phỏng bằng pool; network có epoll/io_uring/kqueue native.

**G-LU-007** ⭐⭐⭐
"DNS lookup trong Node có thể bất ngờ block thread pool — vì sao và phòng thế nào?"
> *Dò cái gì:* dns.lookup dùng getaddrinfo trên pool (có thể chậm); cân nhắc dns.resolve hoặc DNS cache.

**G-LU-008** ⭐⭐⭐⭐
"Nếu tăng UV_THREADPOOL_SIZE quá cao thì sao?"
> *Dò cái gì:* hiểu nhiều thread hơn số core → context switch, không nhanh hơn cho CPU-bound; pool chủ yếu giúp I/O/crypto đồng thời.

---

## G-BLK — Block event loop & CPU nặng

**G-BLK-001** ⭐⭐⭐
"Một tác vụ CPU nặng (loop tính toán lớn) gây gì cho server Node, và vì sao nguy hiểm hơn ở mô hình khác?"
> *Dò cái gì:* block event loop → mọi request khác bị treo (vì single-thread JS), không chỉ request đang chạy.

**G-BLK-002** ⭐⭐⭐⭐
"Liệt kê các cách xử lý CPU-bound work trong Node và khi nào chọn cái nào."
> *Dò cái gì:* worker_threads, child_process, chia nhỏ + setImmediate, offload sang service/queue; cân nhắc theo độ nặng & nhu cầu scale.

**G-BLK-003** ⭐⭐⭐⭐
"JSON.parse/JSON.stringify một payload khổng lồ có block event loop không? Xử lý sao?"
> *Dò cái gì:* có, là sync CPU-bound; dùng streaming parser hoặc giới hạn payload/offload.

**G-BLK-004** ⭐⭐⭐
"Vì sao nên tránh hàm *Sync (readFileSync, …) trong request handler?"
> *Dò cái gì:* chúng block event loop suốt thời gian I/O → giảm throughput; OK ở startup, không OK trong hot path.

**G-BLK-005** ⭐⭐⭐⭐⭐
"Cho một vòng lặp tính toán 5 giây trong handler — bạn refactor để không block mà chưa cần worker. Cách?"
> *Dò cái gì:* chia chunk + yield bằng setImmediate/async, hoặc nhận ra giới hạn của cách này (vẫn chiếm 1 core).

**G-BLK-006** ⭐⭐⭐⭐
"Regex 'catastrophic backtracking' (ReDoS) liên quan gì tới việc block event loop?"
> *Dò cái gì:* regex tệ chạy sync → block loop = DoS; cần regex an toàn/timeout/validate input.

---

## G-WRK — worker_threads vs offload (production)

**G-WRK-001** ⭐⭐⭐⭐
"Khi nào dùng worker_threads, khi nào offload sang một service/queue riêng?"
> *Dò cái gì:* worker_threads cho CPU-bound trong tiến trình; offload khi cần scale độc lập, cô lập lỗi, ngôn ngữ khác, hoặc workload rất nặng.

**G-WRK-002** ⭐⭐⭐⭐
"worker_threads chia sẻ memory với main thread thế nào? SharedArrayBuffer/MessagePort dùng làm gì?"
> *Dò cái gì:* mặc định không share heap (copy qua message), trừ SharedArrayBuffer; giao tiếp qua MessagePort/postMessage.

**G-WRK-003** ⭐⭐⭐
"Mỗi worker thread có event loop và V8 isolate riêng không?"
> *Dò cái gì:* có — mỗi worker là isolate + event loop riêng, không phải chỉ một thread chia sẻ runtime.

**G-WRK-004** ⭐⭐⭐⭐
"Chi phí tạo worker thread là gì? Vì sao nên dùng worker pool thay vì tạo mới mỗi request?"
> *Dò cái gì:* tạo worker tốn (V8 init, RAM); pool (vd piscina) tái dùng, kiểm soát concurrency.

**G-WRK-005** ⭐⭐⭐⭐⭐
"worker_threads có giúp một workload I/O-bound nhanh hơn không? Vì sao thường là không?"
> *Dò cái gì:* I/O đã non-blocking trên 1 loop; worker chỉ giúp CPU-bound; lạm dụng cho I/O thêm overhead.

**G-WRK-006** ⭐⭐⭐
"Truyền dữ liệu lớn giữa main và worker — transferable vs clone khác nhau gì?"
> *Dò cái gì:* clone (structured clone) copy tốn; transfer (ArrayBuffer) chuyển sở hữu, zero-copy.

**G-WRK-007** ⭐⭐⭐⭐
"Một worker crash giữa chừng — bạn thiết kế xử lý lỗi/recovery thế nào?"
> *Dò cái gì:* bắt 'error'/'exit', timeout, retry/replace trong pool, không để main treo chờ.

---

## G-CLU — cluster vs worker_threads vs child_process

**G-CLU-001** ⭐⭐⭐⭐
"cluster và worker_threads khác nhau ở đâu, dùng khi nào?"
> *Dò cái gì:* cluster = nhiều **process** (tận dụng nhiều core, mỗi process 1 event loop); worker_threads = nhiều **thread** trong 1 process (chia sẻ memory được). Cluster cho scale request, worker cho CPU-bound chia sẻ state.

**G-CLU-002** ⭐⭐⭐
"cluster mode hoạt động ra sao? Vai trò của master/primary và worker?"
> *Dò cái gì:* primary fork worker, chia sẻ cổng listen, phân phối kết nối; primary giám sát/health.

**G-CLU-003** ⭐⭐⭐⭐
"Đặt số worker cluster bằng bao nhiêu? Có nên = số CPU core không, vì sao?"
> *Dò cái gì:* thường ~số core (hoặc core−1); cân nhắc container CPU limit, memory mỗi worker.

**G-CLU-004** ⭐⭐⭐⭐⭐
"Chạy cluster trong Kubernetes/container có còn hợp lý không, hay để orchestrator scale?"
> *Dò cái gì:* TL-level — thường để K8s scale pod (1 process/pod) cho observability/rolling; cluster hợp khi muốn dùng hết core trong 1 pod lớn. Biết trade-off.

**G-CLU-005** ⭐⭐⭐⭐
"Session/state trong app cluster gặp vấn đề gì? Sticky session là gì?"
> *Dò cái gì:* state in-memory không share giữa worker → cần store ngoài (Redis); sticky session route 1 client về 1 worker (band-aid).

**G-CLU-006** ⭐⭐⭐
"child_process (spawn/exec/fork) khác worker_threads/cluster thế nào?"
> *Dò cái gì:* child_process chạy tiến trình ngoài/script khác (kể cả non-Node), IPC qua pipe; fork là child_process Node chuyên dụng.

**G-CLU-007** ⭐⭐⭐⭐
"exec vs spawn khác nhau gì, vì sao exec một lệnh với input người dùng là rủi ro?"
> *Dò cái gì:* exec dùng shell + buffer toàn output (command injection, tràn buffer); spawn stream + tránh shell an toàn hơn.

---

## G-ASY — Async patterns

**G-ASY-001** ⭐⭐
"Callback / Promise / async-await — quan hệ và khác biệt?"
> *Dò cái gì:* async-await là cú pháp trên Promise; Promise giải callback hell; hiểu cả ba là cùng một mô hình async.

**G-ASY-002** ⭐⭐⭐
"'Callback hell' là gì và Promise giải quyết nó ra sao?"
> *Dò cái gì:* nesting sâu khó đọc/khó handle lỗi; Promise chaining + .catch tập trung.

**G-ASY-003** ⭐⭐⭐⭐
"Bẫy thường gặp: dùng await trong vòng for tuần tự khi đáng lẽ chạy song song. Khi nào sequential, khi nào Promise.all?"
> *Dò cái gì:* hiểu await-in-loop tuần tự (chậm) vs Promise.all song song; biết khi cần thứ tự/giới hạn concurrency.

**G-ASY-004** ⭐⭐⭐⭐
"Promise.all vs allSettled vs race vs any — chọn cái nào cho tình huống nào?"
> *Dò cái gì:* all fail-fast; allSettled chờ hết kể cả lỗi; race kết quả đầu tiên (kể cả reject); any kết quả resolve đầu tiên.

**G-ASY-005** ⭐⭐⭐⭐
"Unhandled promise rejection trong Node hiện hành gây gì? Xử lý ở tầng app thế nào?"
> *Dò cái gì:* mặc định crash process (Node ≥15); cần .catch/try-catch + listener unhandledRejection để log/graceful.

**G-ASY-006** ⭐⭐⭐
"try-catch có bắt được lỗi async không? Trường hợp nào nó KHÔNG bắt?"
> *Dò cái gì:* bắt khi await; KHÔNG bắt lỗi trong callback không await hoặc Promise quên await/return.

**G-ASY-007** ⭐⭐⭐⭐⭐
"Vì sao 'floating promise' (gọi async không await/không catch) nguy hiểm? Phát hiện thế nào ở team?"
> *Dò cái gì:* lỗi mất tăm + thứ tự sai + rejection không bắt; dùng lint rule (no-floating-promises), code review.

**G-ASY-008** ⭐⭐⭐
"Promisify một API callback-style (vd fs cũ) như thế nào? Có sẵn công cụ gì?"
> *Dò cái gì:* util.promisify hoặc fs/promises; hiểu convention error-first callback.

**G-ASY-009** ⭐⭐⭐⭐
"AsyncLocalStorage giải quyết bài toán gì (vd request context/trace id qua async)?"
> *Dò cái gì:* giữ context theo chuỗi async không phải truyền tay; ứng dụng logging/tracing per-request.

---

## G-DIAG — Diagnostics & profiling

**G-DIAG-001** ⭐⭐⭐⭐
"Một service Node rò rỉ memory trong production — quy trình điều tra của bạn?"
> *Dò cái gì:* nêu được luồng: theo dõi RSS/heap, lấy heap snapshot, so sánh snapshot, tìm retained objects/closure/listener; không chỉ nói "restart".

**G-DIAG-002** ⭐⭐⭐
"Bạn debug/profiling một app Node bằng những công cụ nào?"
> *Dò cái gì:* `node --inspect` + Chrome DevTools, clinic.js, 0x (flamegraph), heap snapshot, --prof; biết ít nhất vài cái và mục đích.

**G-DIAG-003** ⭐⭐⭐⭐
"Heap snapshot cho biết gì? 'Retained size' vs 'shallow size' khác nhau ra sao?"
> *Dò cái gì:* snapshot = ảnh chụp heap; shallow = bộ nhớ của chính object, retained = tổng giải phóng được nếu xóa object đó.

**G-DIAG-004** ⭐⭐⭐⭐
"Nguyên nhân memory leak phổ biến trong Node là gì?"
> *Dò cái gì:* listener không gỡ, biến global/closure giữ tham chiếu, cache không bound, timer treo, Map không dọn.

**G-DIAG-005** ⭐⭐⭐
"clinic.js gồm những công cụ con nào và mỗi cái dùng để chẩn đoán gì?"
> *Dò cái gì:* Doctor (chẩn đoán tổng quát/loại bệnh), Flame (CPU/flamegraph), Bubbleprof (async). *(verify gói hiện hành.)*

**G-DIAG-006** ⭐⭐⭐⭐
"Flamegraph đọc thế nào? Nó giúp tìm vấn đề gì?"
> *Dò cái gì:* trục ngang = thời gian/tần suất stack, chiều cao = độ sâu call; tìm hàm chiếm CPU (wide bars).

**G-DIAG-007** ⭐⭐⭐⭐⭐
"Phân biệt CPU-bound bottleneck với event-loop-blocking bằng dữ liệu nào?"
> *Dò cái gì:* dùng event loop delay/ELU + CPU profile; CPU cao + loop lag = blocking sync; CPU thấp + loop lag = chờ pool/lock.

**G-DIAG-008** ⭐⭐⭐
"Logging production nên có cấu trúc thế nào, dùng thư viện nào trong Node?"
> *Dò cái gì:* structured JSON log, level, correlation/trace id; pino/winston; tránh console.log đồng bộ ở hot path.

**G-DIAG-009** ⭐⭐⭐⭐
"Graceful shutdown một Node service đúng cách gồm các bước nào?"
> *Dò cái gì:* bắt SIGTERM/SIGINT, ngừng nhận request mới, drain in-flight, đóng DB/queue, timeout cưỡng bức; quan trọng cho rolling deploy K8s.

---

## G-SEC — Node security hardening

**G-SEC-001** ⭐⭐⭐
"Hardening một Express/Nest app gồm những lớp gì? Kể công cụ cụ thể."
> *Dò cái gì:* helmet (headers), validate/sanitize input, rate limit, CORS chặt, secrets ngoài code, HTTPS; nêu được tên gói.

**G-SEC-002** ⭐⭐⭐
"helmet làm gì? Nó KHÔNG làm gì (đừng nhầm)?"
> *Dò cái gì:* set security headers (CSP, HSTS, X-Frame…); KHÔNG phải tường lửa/auth/validation toàn diện.

**G-SEC-003** ⭐⭐⭐⭐
"Rate limiting per-user/IP trong Node — dùng gì, lưu state ở đâu khi nhiều instance?"
> *Dò cái gì:* rate-limiter-flexible/express-rate-limit; store dùng chung (Redis) khi scale ngang, không in-memory.

**G-SEC-004** ⭐⭐⭐
"CSRF là gì, khi nào cần phòng và khi nào KHÔNG cần?"
> *Dò cái gì:* tấn công lợi dụng cookie tự gửi; cần khi auth bằng cookie + form; API token Bearer/stateless thường không cần như session-cookie.

**G-SEC-005** ⭐⭐⭐⭐
"Phòng SQL injection và NoSQL injection trong Node backend bằng cách nào?"
> *Dò cái gì:* parameterized query/ORM, validate type, tránh build query bằng string nối; NoSQL: chặn object operator injection.

**G-SEC-006** ⭐⭐⭐⭐⭐
"Supply-chain attack qua npm: làm sao giảm rủi ro dependency độc hại/lỗ hổng?"
> *Dò cái gì:* `npm audit`, lockfile + `npm ci`, Snyk/Dependabot, pin version, kiểm tra postinstall script, tối thiểu hóa deps.

**G-SEC-007** ⭐⭐⭐⭐
"Lưu mật khẩu đúng cách trong Node — thuật toán nào, vì sao không dùng SHA-256 thuần?"
> *Dò cái gì:* bcrypt/argon2 (slow + salt), không hash nhanh không salt; argon2 ưu tiên hiện đại.

**G-SEC-008** ⭐⭐⭐
"Quản lý secrets thế nào để không hardcode/không lộ qua git?"
> *Dò cái gì:* env/.env (gitignore), secret manager (Vault/AWS Secrets Manager), không commit, rotate.

**G-SEC-009** ⭐⭐⭐⭐
"Node có Permission Model (--permission) — nó giải quyết bài toán gì ở mức runtime?"
> *Dò cái gì:* giới hạn quyền truy cập fs/child_process/worker ở cấp process; phòng thủ chiều sâu cho code/deps không tin cậy. *(verify trạng thái flag ở Node LTS hiện hành.)*

---

## G-STR — Streams & backpressure

**G-STR-001** ⭐⭐⭐
"Stream trong Node là gì, có mấy loại?"
> *Dò cái gì:* xử lý dữ liệu theo chunk thay vì load hết; Readable/Writable/Duplex/Transform.

**G-STR-002** ⭐⭐⭐⭐
"Khi nào dùng stream thay vì đọc cả file/response vào memory? Cho ví dụ."
> *Dò cái gì:* file lớn, proxy/upload, transform on-the-fly → tránh ngốn RAM & tăng độ trễ first byte.

**G-STR-003** ⭐⭐⭐⭐
"Backpressure là gì? Điều gì xảy ra nếu bỏ qua nó?"
> *Dò cái gì:* khi writable chậm hơn readable → buffer phình → ngốn RAM/crash; backpressure báo nguồn chậm lại.

**G-STR-004** ⭐⭐⭐⭐
"pipe()/pipeline() xử lý backpressure và lỗi thế nào? Vì sao pipeline được khuyến nghị?"
> *Dò cái gì:* pipe tự điều phối backpressure nhưng không tự cleanup khi lỗi; stream.pipeline xử lý lỗi + đóng đúng cách.

**G-STR-005** ⭐⭐⭐
"flowing mode vs paused mode của Readable khác nhau gì?"
> *Dò cái gì:* paused = đọc chủ động bằng read(); flowing = data đẩy qua 'data' event; chuyển mode bằng pipe/on('data')/pause/resume.

**G-STR-006** ⭐⭐⭐⭐
"highWaterMark là gì và chỉnh nó ảnh hưởng gì?"
> *Dò cái gì:* ngưỡng buffer kích hoạt backpressure; lớn → throughput cao/ngốn RAM, nhỏ → ngược lại.

**G-STR-007** ⭐⭐⭐⭐⭐
"Async iterator (`for await...of`) trên stream có lợi gì so với event 'data'? Có pitfall nào?"
> *Dò cái gì:* code tuần tự dễ đọc + backpressure tự nhiên; pitfall: xử lý lỗi/cleanup, không xử lý song song trong vòng.

---

## G-RT — Runtime / V8 / GC / module / versioning (chiều sâu TL)

**G-RT-001** ⭐⭐⭐
"V8 và libuv đóng vai trò gì trong Node? Phân biệt rạch ròi."
> *Dò cái gì:* V8 = engine thực thi JS + GC; libuv = event loop + async I/O + thread pool. Không lẫn lộn hai cái.

**G-RT-002** ⭐⭐⭐⭐
"Garbage collection của V8 hoạt động theo mô hình nào (generational)? Vì sao GC có thể gây latency spike?"
> *Dò cái gì:* young/old generation, scavenge nhanh + major GC chậm hơn (có thể stop-the-world ngắn) → spike; biết --max-old-space-size.

**G-RT-003** ⭐⭐⭐⭐
"CommonJS (require) vs ESM (import) trong Node hiện nay khác nhau gì và gây vướng gì khi migrate?"
> *Dò cái gì:* sync vs async loading, __dirname/top-level await, interop, "type": "module"; biết điểm vấp thực tế.

**G-RT-004** ⭐⭐⭐⭐⭐
"Là TL, bạn chọn Node version cho dự án mới production 6/2026 thế nào? Vì sao không chọn version mới nhất?"
> *Dò cái gì:* chọn **Active LTS (Node 24)** chứ không phải Current (26); tránh EOL (20); biết release cycle đang đổi (Node 27+ annual). Lý do: ổn định + còn được vá bảo mật.

---

## ✅ Sau Bước A
Đây là **84 câu** phủ HẾT 10 mục con của G + nhóm runtime/V8/GC. Tất cả viết lại bằng lời mình, không chép danh sách nguồn; mỗi câu có mốc chấm riêng.

**Việc của bạn:** lưu file này vào project (kéo vào Project files). Sau đó tôi sẽ:
- **Bước B** — dựng giáo trình *ngược* từ bộ đề (gom câu → bài học, sắp cơ bản → nâng cao, ánh xạ Bài → ID câu).
- **Bước C** — giảng từng bài theo Hợp đồng 10 mục.
- **Bước D** — phỏng vấn theo đợt, chấm **live**.

> *Câu thần chú:* **"Tôi không nhớ để gõ, tôi hiểu để chỉ huy — và tra cứu phần còn lại."**
