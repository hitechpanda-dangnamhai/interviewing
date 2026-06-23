# 🧪 QB_P — Ngân hàng câu hỏi: Coding / DSA

> **Mục P** trong roadmap Tech Lead Backend (🟡 ~35% · *biến động lớn*: FAANG/product lớn ~70% DSA nặng; product vừa/outsourcing nhẹ, thiên về **system-design coding**) · Sinh theo **WORKFLOW 2 — Bước A**.
> Đây là **mục LỚN** → bộ đề phủ HẾT 3 mục con của roadmap (**System-design coding (pseudo-code high-level)** · **DSA cơ bản: array/string/hashmap** · **Big-O**) + mở rộng đúng chiều TL backend Node/TS: **coding utilities JS/Node** (debounce, deep clone, Promise polyfill, async concurrency limit…) và **judgment cấp Tech Lead** (đánh giá DSA khi phỏng vấn, khi nào DSA *không* quan trọng, kiểm soát chất lượng code do AI sinh).
> **Chỉ câu hỏi — KHÔNG kèm đáp án.** Mỗi câu có dòng *"dò cái gì"* = tiêu chí ĐẠT, dùng để chấm **live** ở Bước D.
> **Tổng: 94 câu.**

> **Chống trùng (đã đọc `QB_A_systemdesign.md`, `QB_J_caching.md`, `QB_I_concurrency.md`, `QB_G_nodejs.md`, `QB_E_database.md`, `QB_Q_designpatterns.md`):**
> - **Rate limiter:** **A-RL** giữ *trade-off thuật toán + tầng phân tán* (token bucket vs fixed/sliding, lưu counter ở đâu, Redis, multi-instance). Ở **P-SDC** nhìn rate limiter ở **góc CODE/IMPLEMENT** (viết class token-bucket, toán refill, sliding-window-counter trong bộ nhớ, độ phức tạp). Khi đụng trade-off/phân tán → trỏ sang A-RL, KHÔNG lặp.
> - **LRU cache:** **J-EVICT** nhìn LRU ở *góc Redis eviction policy* (approximated LRU/LFU, `maxmemory-policy`, sampling). Ở **P-SDC** nhìn LRU ở *góc cấu trúc dữ liệu* — **tự code LRU đạt O(1)** (HashMap + doubly linked list / JS `Map`). Cross-ref J khi đụng eviction thật trong production.
> - **Consistent hashing / bloom filter:** **A-BB** giữ *khái niệm + dùng ở đâu trong thiết kế hệ thống* (virtual nodes, false positive). Ở **P** chỉ chạm ở *góc code/Big-O* và **trỏ sang A-BB** cho phần lý thuyết, không lặp.
> - **Promise.all/allSettled/race/any & await-in-loop:** **G-ASY** giữ *chọn combinator nào cho tình huống nào*. Ở **P-JS** nhìn ở *góc IMPLEMENT* — **tự viết polyfill `Promise.all`, viết async concurrency limiter (pLimit), retry-with-backoff**. Cross-ref G, không lặp câu "chọn combinator".
> - **EventEmitter:** **G** giữ *wiring runtime*. Ở **P-JS** là *bài coding* — **tự code class pub/sub `on/emit/off`**. Cross-ref G.
> - **Big-O của index / N+1 / query plan**: phần *database* nằm ở **E**. P-BIGO chỉ làm Big-O *thuật toán/cấu trúc dữ liệu thuần*.

## Cách đọc
- **ID:** `P-<tag>-<số>`. Tag mục con liệt kê ở mục lục dưới.
- **Độ khó:** ⭐ định nghĩa/bài easy → ⭐⭐⭐⭐⭐ phân biệt senior với TL thật (trade-off, tối ưu, "khi nào KHÔNG cần DSA", đánh giá người khác, kiểm soát code AI).
- **"Dò cái gì":** điều người trả lời PHẢI chạm tới mới tính ĐẠT (không phải đáp án — chỉ là mốc chấm live ở Bước D).
- Câu coding mong đợi **viết được code/pseudo-code + nêu Big-O**, không chỉ kể tên thuật toán.

## Bối cảnh "độ ổn định" của chủ đề (tính tại 6/2026)
- Kiến thức DSA & Big-O **rất ổn định** (Big-O, two pointers, sliding window, binary search, BFS/DFS, recursion/backtracking, DP — không đổi theo thời gian) → **ít cần search** hơn mục model/giá.
- Phần dễ đổi: **API/cú pháp JS/Node** dùng để minh họa (vd `structuredClone` đã là chuẩn built-in cho deep clone; `Array.prototype.flat`, `Map`/`Set` iteration order, `AbortController` cho cancel). → câu chạm API runtime gắn dấu *(verify)*.
- **Bối cảnh phỏng vấn 2025–2026** (đã search): xu hướng "**coding patterns**" (~28 patterns: two pointers, sliding window, fast/slow, DFS/BFS, backtracking, heap/top-K, intervals, monotonic stack, binary search on answer…) thay cho "cày 500 bài rời rạc" — nhận diện *pattern* quan trọng hơn nhớ lời giải. Với role TL Node/backend, **JS-flavored coding** (LRU, rate limiter, debounce, Promise utilities, async concurrency) và **system-design coding** xuất hiện nhiều hơn DSA hardcore.
- Câu chạm tên/cú pháp API runtime có dấu *(verify)*.

---

## Mục lục mục con (ID prefix → số câu)

| Tag | Mục con | Số câu |
|---|---|---|
| **P-BIGO** | Big-O / complexity: time–space, amortized, best/avg/worst, đọc complexity của code thật | 10 |
| **P-ARR** | Array / String / HashMap: frequency, prefix sum, in-place, dedup, anagram | 10 |
| **P-PTR** | Two pointers & sliding window | 8 |
| **P-STK** | Stack / Queue: valid parentheses, monotonic stack, queue-from-stacks | 6 |
| **P-SRCH** | Binary search & sorting (kể cả binary-search-on-answer) | 8 |
| **P-LL** | Linked list: reverse, cycle (fast/slow), merge, k-th from end | 6 |
| **P-TREE** | Trees & graphs: traversal, BFS/DFS, topological sort | 8 |
| **P-REC** | Recursion, backtracking, DP nhập môn, memoization (thuật toán) | 8 |
| **P-JS** | Coding utilities JS/Node: debounce/throttle, deep clone/equal, EventEmitter, memoize, curry, Promise polyfill, async pool, retry | 12 |
| **P-SDC** | System-design coding: code LRU, token-bucket, sliding-window-counter, ID gen, cursor pagination, dedup, top-K, KV-with-TTL | 10 |
| **P-TL** | Judgment cấp Tech Lead: đánh giá DSA khi phỏng vấn, khi nào DSA không quan trọng, communicate approach, tối ưu code thật, kiểm soát code AI | 8 |

---

## P-BIGO — Big-O & phân tích độ phức tạp

**P-BIGO-001** ⭐
"Big-O đo *cái gì*? Phân biệt time complexity với space complexity bằng một câu mỗi cái."
> *Dò cái gì:* Big-O = tốc độ tăng của chi phí theo kích thước input n (asymptotic upper bound), bỏ hằng số/bậc thấp; time = số phép tính, space = bộ nhớ phụ; KHÔNG phải 'thời gian chạy thực tế bằng giây'.

**P-BIGO-002** ⭐
"Xếp từ nhanh → chậm: O(1), O(n²), O(log n), O(n), O(n log n), O(2ⁿ)."
> *Dò cái gì:* O(1) < O(log n) < O(n) < O(n log n) < O(n²) < O(2ⁿ); không nhầm vị trí n log n và n².

**P-BIGO-003** ⭐⭐
"Một vòng for lồng trong một vòng for, mỗi vòng chạy tới n. Big-O? Nếu vòng trong chỉ chạy tới `i` (tam giác) thì sao?"
> *Dò cái gì:* cả hai là O(n²) (tam giác = n(n−1)/2 vẫn O(n²) vì bỏ hằng số); hiểu vì sao 'một nửa công việc' không đổi bậc.

**P-BIGO-004** ⭐⭐
"Vì sao tra cứu trong HashMap/`Map` thường gọi là O(1) còn tra trong array sorted bằng binary search là O(log n)? Đánh đổi gì?"
> *Dò cái gì:* hash O(1) trung bình (worst O(n) khi collision nhiều), tốn space; binary search O(log n) nhưng cần data đã sort + truy cập ngẫu nhiên; nêu được 'O(1) trung bình' chứ không tuyệt đối.

**P-BIGO-005** ⭐⭐⭐
"Đọc đoạn code: `for (const x of arr) { if (set.has(x)) ... ; set.add(x); }`. Time & space Big-O? Nếu thay `set` bằng `arr2.includes(x)` thì sao?"
> *Dò cái gì:* với Set: O(n) time / O(n) space; với `includes` (quét mảng): O(n²) time / O(1) space; nhận ra `includes`/`indexOf` trong vòng lặp là cái bẫy hay gặp.

**P-BIGO-006** ⭐⭐⭐
"Amortized O(1) nghĩa là gì? Ví dụ kinh điển (vd push vào dynamic array / `Array.push`)."
> *Dò cái gì:* trung bình theo dãy thao tác (không phải mỗi thao tác): array đôi khi resize tốn O(n) nhưng phân bổ ra nhiều push → trung bình O(1); phân biệt amortized với average-case.

**P-BIGO-007** ⭐⭐⭐
"Khác nhau giữa best / average / worst case? Cho ví dụ một thuật toán có ba mức rất khác nhau (gợi ý: quicksort)."
> *Dò cái gì:* quicksort best/avg O(n log n), worst O(n²) (pivot tệ / data đã sort); hiểu vì sao thường nói 'O(n log n) trung bình' và cách giảm worst (random pivot / median-of-three).

**P-BIGO-008** ⭐⭐⭐⭐
"Big-O của: đệ quy Fibonacci ngây thơ `fib(n-1)+fib(n-2)`? Sau khi memoize? Giải thích vì sao đổi bậc."
> *Dò cái gì:* ngây thơ O(2ⁿ) (cây đệ quy nhân đôi); memoize → O(n) time / O(n) space vì mỗi n tính đúng một lần; nối được 'overlapping subproblems' ↔ DP.

**P-BIGO-009** ⭐⭐⭐⭐
"Một hàm có 2 vòng lặp *nối tiếp* (mỗi cái O(n)) rồi gọi một sort. Tổng Big-O? Vì sao không cộng dồn thành O(3n) hay nhân?"
> *Dò cái gì:* O(n) + O(n) + O(n log n) → lấy bậc lớn nhất = O(n log n); hằng số (3) bỏ; phân biệt 'nối tiếp = cộng → lấy max bậc' vs 'lồng nhau = nhân'.

**P-BIGO-010** ⭐⭐⭐⭐⭐
"Ứng viên nói thuật toán của họ 'O(n)' nhưng dùng `arr.unshift()` trong vòng lặp, hoặc spread `[...acc, x]` mỗi vòng. Là người chấm, bạn bắt lỗi gì?"
> *Dò cái gì:* `unshift`/spread copy lại cả mảng → O(n) mỗi lần → tổng O(n²); chỉ ra cú pháp 'nhìn gọn' che giấu chi phí ẩn; TL phải đọc Big-O *thực tế của thao tác* chứ không tin lời khai.

---

## P-ARR — Array / String / HashMap

**P-ARR-001** ⭐⭐
"Two Sum: cho mảng số + target, trả về 2 index có tổng = target. Cách brute-force và cách O(n). Nêu Big-O từng cách."
> *Dò cái gì:* brute-force 2 vòng O(n²); cách O(n) dùng HashMap lưu (value→index), với mỗi x kiểm tra `target−x` đã thấy chưa; đánh đổi space O(n).

**P-ARR-002** ⭐⭐
"Đếm tần suất ký tự/phần tử bằng HashMap: viết khung code. Vì sao Map/object tốt hơn lồng vòng lặp?"
> *Dò cái gì:* duyệt 1 lần, `freq.set(x, (freq.get(x)||0)+1)`; O(n) thay vì O(n²); hiểu HashMap đổi 'tìm kiếm lặp lại' thành tra O(1).

**P-ARR-003** ⭐⭐
"Kiểm tra 2 chuỗi có phải anagram của nhau không. Cách dùng sort vs cách dùng frequency count — Big-O mỗi cách?"
> *Dò cái gì:* sort O(n log n); frequency map/array O(n) time; biết edge case độ dài khác nhau, Unicode/case; chọn frequency khi cần nhanh nhất.

**P-ARR-004** ⭐⭐⭐
"Prefix sum là gì? Dùng nó để trả lời 'tổng đoạn [i..j]' trong O(1) sau O(n) tiền xử lý — viết ý tưởng."
> *Dò cái gì:* `prefix[k]=sum(0..k-1)`; tổng đoạn = `prefix[j+1]−prefix[i]`; đổi nhiều truy vấn range từ O(n) mỗi câu → O(1); nhận ra đây là 'precompute' đánh đổi space.

**P-ARR-005** ⭐⭐⭐
"Subarray có tổng = k: vì sao prefix sum + HashMap cho O(n) thay vì O(n²)? (không cần code đầy đủ, nêu cơ chế)."
> *Dò cái gì:* lưu số lần xuất hiện của từng prefix sum; với prefix hiện tại S, đếm số prefix trước = S−k; biết xử lý prefix=0 ban đầu; hiểu vì sao tránh được vòng lồng.

**P-ARR-006** ⭐⭐⭐
"Xoá phần tử trùng *tại chỗ* (in-place) khỏi mảng đã sort, trả về độ dài mới. Vì sao không nên `splice` trong vòng lặp?"
> *Dò cái gì:* dùng two-pointer (write index) O(n)/O(1) space; `splice` trong loop là O(n) mỗi lần → O(n²) + làm hỏng index đang duyệt; nhận ra bẫy sửa mảng khi đang duyệt.

**P-ARR-007** ⭐⭐⭐
"Group anagrams: gom các từ là anagram của nhau. Key của HashMap nên là gì?"
> *Dò cái gì:* key = chuỗi đã sort ký tự (hoặc 26-count signature); gom vào map key→list; O(n·k log k) hoặc O(n·k); hiểu 'thiết kế key' là chìa khoá.

**P-ARR-008** ⭐⭐⭐⭐
"Tìm phần tử xuất hiện nhiều hơn n/2 lần (majority). Cách HashMap O(n) space và cách Boyer–Moore O(1) space — nêu ý tưởng Boyer–Moore."
> *Dò cái gì:* HashMap đếm rồi lọc; Boyer–Moore voting (candidate + count, +1/−1) đạt O(1) space; hiểu giả định 'majority tồn tại' và vì sao voting đúng.

**P-ARR-009** ⭐⭐⭐⭐
"Product of array except self (tích mọi phần tử trừ chính nó) *không dùng phép chia*, O(n). Vì sao cấm chia? Ý tưởng?"
> *Dò cái gì:* prefix product × suffix product; cấm chia vì có thể có số 0 (chia 0); 2 lượt duyệt O(n)/O(1) space phụ (ngoài output); nhận ra cái bẫy số 0.

**P-ARR-010** ⭐⭐⭐⭐⭐
"Trong JS, `for...of` vs `forEach` vs `for` index vs `reduce` — khi xử lý mảng lớn nóng (hot path) bạn chọn gì và vì sao? Bẫy performance/đúng đắn nào?"
> *Dò cái gì:* `forEach` không break/await được đúng cách; `reduce` tạo object mới mỗi vòng dễ thành O(n²); for/`for...of` linh hoạt break; ưu tiên đo (đừng micro-optimize mù); hiểu 'đúng trước, nhanh sau'.

---

## P-PTR — Two pointers & Sliding window

**P-PTR-001** ⭐⭐
"Two pointers là pattern gì? Cho một bài kinh điển nó giải gọn (vd 'pair có tổng = target trong mảng *đã sort*')."
> *Dò cái gì:* hai con trỏ đầu/cuối tiến vào giữa; tổng lớn quá → giảm phải, nhỏ quá → tăng trái; O(n)/O(1); điều kiện áp dụng: mảng đã sort hoặc cấu trúc cho phép.

**P-PTR-002** ⭐⭐
"Đảo ngược mảng/chuỗi tại chỗ bằng two pointers — viết vòng lặp. Big-O?"
> *Dò cái gì:* swap đầu–cuối, tiến vào giữa, dừng khi `l>=r`; O(n)/O(1); biết điều kiện dừng đúng (không swap hai lần).

**P-PTR-003** ⭐⭐⭐
"Sliding window: bài 'độ dài chuỗi con dài nhất không lặp ký tự'. Vì sao window co/giãn cho O(n) thay vì O(n²)?"
> *Dò cái gì:* mở rộng right, khi gặp ký tự trùng thì co left tới sau lần xuất hiện trước (dùng map last-seen); mỗi con trỏ đi tối đa n bước → O(n); hiểu 'mỗi phần tử vào/ra window 1 lần'.

**P-PTR-004** ⭐⭐⭐
"Phân biệt fixed-size window vs variable-size window. Cho mỗi loại một ví dụ bài."
> *Dò cái gì:* fixed: 'tổng/trung bình lớn nhất của k phần tử liên tiếp'; variable: 'subarray nhỏ nhất có tổng ≥ S'; biết fixed trượt đều, variable co/giãn theo điều kiện.

**P-PTR-005** ⭐⭐⭐
"Fast/slow pointer (Floyd) dùng làm gì ngoài linked list? (vd tìm phần tử lặp / điểm giữa)."
> *Dò cái gì:* phát hiện chu trình, tìm trung điểm (slow đi 1, fast đi 2), 'find the duplicate number' coi mảng như linked list ngầm; hiểu cơ chế gặp nhau.

**P-PTR-006** ⭐⭐⭐⭐
"3Sum: tìm bộ ba có tổng = 0, không trùng. Vì sao sort trước rồi two-pointer? Xử lý trùng thế nào để không ra bộ lặp?"
> *Dò cái gì:* sort O(n log n) + cố định i rồi two-pointer phần còn lại → O(n²); bỏ qua giá trị trùng ở cả i và l/r để tránh duplicate; hiểu vì sao O(n²) đã là tốt cho 3Sum.

**P-PTR-007** ⭐⭐⭐⭐
"Minimum window substring (cửa sổ nhỏ nhất chứa đủ ký tự cần). Cấu trúc theo dõi 'đã đủ chưa' là gì?"
> *Dò cái gì:* need-map + đếm 'formed/required'; mở right tới khi đủ, co left tới khi vừa thiếu, ghi min; O(n); hiểu cách dùng counter để biết khi nào window hợp lệ.

**P-PTR-008** ⭐⭐⭐⭐⭐
"Khi nào two-pointer/sliding window *không* áp dụng được dù bài 'trông giống'? (gợi ý: điều kiện monotonic)."
> *Dò cái gì:* cần tính chất đơn điệu — co/mở window phải làm điều kiện thay đổi một chiều dự đoán được; nếu có số âm (sliding window theo tổng có thể sai), phải đổi sang prefix sum + map; nhận diện được giả định ngầm.

---

## P-STK — Stack / Queue

**P-STK-001** ⭐⭐
"Valid parentheses (ngoặc đóng/mở khớp). Vì sao dùng stack? Big-O?"
> *Dò cái gì:* push khi gặp mở, pop + so khớp khi gặp đóng; cuối cùng stack rỗng = hợp lệ; O(n)/O(n); xử lý đóng-không-có-mở.

**P-STK-002** ⭐⭐
"Khác nhau stack (LIFO) vs queue (FIFO) — cho mỗi cái một use case backend thực tế."
> *Dò cái gì:* stack: undo, call stack, DFS, parse; queue: job/task queue, BFS, rate buffering; gọi đúng LIFO/FIFO và gắn use case.

**P-STK-003** ⭐⭐⭐
"Trong JS, `array.push/pop` làm stack O(1), nhưng dùng `array.shift()` làm queue thì sao? Cách làm queue O(1)?"
> *Dò cái gì:* `shift` O(n) (dịch toàn bộ); queue hiệu quả dùng 2 stacks hoặc linked list / vòng tròn (deque) cho amortized O(1); nhận ra bẫy `shift` trong vòng lặp.

**P-STK-004** ⭐⭐⭐
"Implement queue bằng 2 stacks. Amortized Big-O của dequeue?"
> *Dò cái gì:* stack-in để enqueue, stack-out để dequeue; khi out rỗng đổ in→out; mỗi phần tử di chuyển tối đa 2 lần → amortized O(1); hiểu vì sao amortized chứ không O(1) mỗi op.

**P-STK-005** ⭐⭐⭐⭐
"Monotonic stack: bài 'next greater element' (phần tử lớn hơn kế tiếp). Vì sao O(n) dù 'nhìn như' lồng vòng?"
> *Dò cái gì:* giữ stack giảm dần, mỗi phần tử push/pop tối đa 1 lần → O(n); pop ra thì 'người vừa tới' chính là next greater; nhận diện pattern monotonic.

**P-STK-006** ⭐⭐⭐⭐⭐
"Largest rectangle in histogram (hoặc 'trapping rain water'). Vì sao monotonic stack thắng cách O(n²)? Nêu ý tưởng cốt lõi."
> *Dò cái gì:* dùng stack lưu index theo chiều cao tăng; khi gặp cột thấp hơn, pop và tính diện tích với biên trái/phải; O(n); thể hiện hiểu 'biên gần nhất nhỏ hơn' là thứ stack cung cấp.

---

## P-SRCH — Binary search & Sorting

**P-SRCH-001** ⭐⭐
"Binary search trên mảng đã sort: viết vòng lặp (lo/hi/mid). Bẫy off-by-one và overflow mid hay gặp?"
> *Dò cái gì:* `mid = lo + ((hi-lo)>>1)` (tránh overflow), điều kiện `lo<=hi`, cập nhật `lo=mid+1`/`hi=mid-1`; O(log n); nêu bẫy vòng lặp vô hạn khi cập nhật sai biên.

**P-SRCH-002** ⭐⭐
"Vì sao binary search yêu cầu data *đã sort*? Nếu data thay đổi liên tục thì chi phí giữ sort là gì?"
> *Dò cái gì:* cần tính đơn điệu để loại nửa; giữ sort khi insert nhiều tốn O(n) mỗi lần (hoặc dùng balanced tree/skip list); hiểu trade-off search nhanh vs insert chậm.

**P-SRCH-003** ⭐⭐⭐
"Tìm phần tử đầu tiên ≥ target (lower bound) khác gì tìm 'có tồn tại không'? Khi nào cần biến thể này?"
> *Dò cái gì:* binary search 'leftmost' giữ kết quả khả dĩ rồi tiếp tục thu hẹp trái; dùng cho insert position, range count, ceiling/floor; phân biệt với exact-match.

**P-SRCH-004** ⭐⭐⭐
"Search in rotated sorted array (mảng sort bị xoay). Vì sao vẫn O(log n)? Mấu chốt quyết định nửa nào?"
> *Dò cái gì:* mỗi bước xác định nửa nào đang 'sorted' bằng so sánh `nums[lo]`/`nums[mid]`, rồi kiểm tra target có nằm trong nửa sorted không; O(log n); xử lý phần tử trùng làm hỏng O(log n).

**P-SRCH-005** ⭐⭐⭐⭐
"Binary search on answer: bài 'capacity nhỏ nhất để chở hàng trong D ngày' hoặc 'tốc độ ăn chuối tối thiểu'. Ta binary search trên *cái gì*?"
> *Dò cái gì:* search trên không gian đáp án (capacity/speed) không phải trên mảng; cần hàm `feasible(x)` đơn điệu (x lớn hơn → dễ khả thi hơn); tìm biên feasible nhỏ nhất; nhận ra pattern.

**P-SRCH-006** ⭐⭐⭐
"Merge sort vs quick sort vs heap sort: Big-O time/space và tính ổn định (stable)? Khi nào chọn cái nào?"
> *Dò cái gì:* merge O(n log n) ổn định nhưng O(n) space; quick O(n log n) avg / O(n²) worst, in-place, không stable; heap O(n log n) in-place không stable; biết khi nào cần stable (sort theo nhiều khoá).

**P-SRCH-007** ⭐⭐⭐
"`Array.prototype.sort()` của JS mặc định so sánh thế nào, và bẫy kinh điển với số là gì?"
> *Dò cái gì:* mặc định so sánh **theo chuỗi (lexicographic)** → `[10,2,1].sort()` ra `[1,10,2]`; phải truyền comparator `(a,b)=>a-b`; biết sort mutate tại chỗ; *(verify: V8 sort nay stable theo spec ES2019)*.

**P-SRCH-008** ⭐⭐⭐⭐⭐
"Cho 1 tỷ số không vừa RAM, cần sort. Cách tiếp cận? (gợi ý: external sort) Big-O I/O quan trọng hơn Big-O CPU ở đâu?"
> *Dò cái gì:* external merge sort (chia chunk vừa RAM, sort từng chunk, k-way merge); chi phí thật là số lần đọc/ghi đĩa (I/O-bound), không chỉ phép so sánh; nối sang MapReduce/streaming; thể hiện tư duy 'data > RAM'.

---

## P-LL — Linked list

**P-LL-001** ⭐⭐
"Linked list khác array ở Big-O của: truy cập index k, chèn/xoá đầu, chèn/xoá giữa? Khi nào chọn linked list?"
> *Dò cái gì:* array O(1) random access nhưng chèn/xoá giữa O(n); linked list chèn/xoá đầu O(1) nhưng truy cập O(n); chọn LL khi nhiều chèn/xoá ở vị trí đã có con trỏ; ít dùng trong JS thực tế (cache locality kém).

**P-LL-002** ⭐⭐⭐
"Đảo ngược một singly linked list (iterative). Cần mấy con trỏ? Viết vòng lặp."
> *Dò cái gì:* prev/curr/next, mỗi bước `next=curr.next; curr.next=prev; prev=curr; curr=next`; trả về prev; O(n)/O(1); biết vì sao phải lưu `next` trước khi đổi.

**P-LL-003** ⭐⭐⭐
"Phát hiện chu trình (cycle) trong linked list. Fast/slow pointer hoạt động ra sao? Big-O space?"
> *Dò cái gì:* Floyd: slow +1, fast +2, gặp nhau = có cycle; O(n)/O(1) space (hơn cách dùng Set O(n) space); biết tìm cả 'điểm bắt đầu cycle' bằng reset một con trỏ về head.

**P-LL-004** ⭐⭐⭐
"Tìm node thứ k-từ-cuối trong một lượt duyệt (one pass). Kỹ thuật?"
> *Dò cái gì:* two-pointer cách nhau k bước, dịch cùng nhau tới khi đầu chạm cuối; O(n)/O(1); xử lý edge case k > độ dài.

**P-LL-005** ⭐⭐⭐⭐
"Merge 2 sorted linked list. Đệ quy vs lặp — đánh đổi space? Tổng quát hoá merge k list thì sao?"
> *Dò cái gì:* lặp dùng dummy head O(1) space; đệ quy tốn O(n) stack; merge k list dùng min-heap O(N log k) hoặc divide-and-conquer; cross-ref P-SDC top-K/heap.

**P-LL-006** ⭐⭐⭐⭐⭐
"Vì sao bài LL như 'reverse/cycle' hay được hỏi dù production hiếm dùng linked list thuần? Người chấm thật ra đang dò *kỹ năng* gì?"
> *Dò cái gì:* dò khả năng quản lý con trỏ/tham chiếu cẩn thận, xử lý edge (null, 1 phần tử), không leak/đứt list; là proxy cho 'cẩn thận với pointer/reference & state' — tư duy TL nhìn xuyên qua bài tập.

---

## P-TREE — Trees & Graphs

**P-TREE-001** ⭐⭐
"Phân biệt BFS và DFS: chiến lược, cấu trúc dữ liệu kèm theo (queue vs stack/đệ quy), khi nào dùng cái nào?"
> *Dò cái gì:* BFS dùng queue, đi theo tầng → đường ngắn nhất (unweighted); DFS dùng stack/đệ quy, đi sâu → tìm đường/tồn tại, ít memory hơn trên cây sâu-hẹp; gọi đúng cặp queue/stack.

**P-TREE-002** ⭐⭐
"Duyệt cây nhị phân: in-order / pre-order / post-order khác nhau ở thứ tự *thăm root*. In-order của BST cho ra gì?"
> *Dò cái gì:* pre = root trước, in = root giữa, post = root sau; in-order của BST → dãy **sorted tăng dần**; biết ứng dụng (post-order để xoá/giải phóng, pre-order để copy).

**P-TREE-003** ⭐⭐⭐
"Tính độ sâu (max depth) của cây nhị phân. Đệ quy vs lặp (BFS theo tầng) — Big-O & space?"
> *Dò cái gì:* đệ quy `1+max(left,right)` O(n) time, O(h) stack; BFS đếm tầng O(n) time, O(width) space; biết worst-case stack = O(n) khi cây lệch.

**P-TREE-004** ⭐⭐⭐
"Level-order traversal (in cây theo từng tầng). Làm sao biết một tầng kết thúc ở đâu khi dùng queue?"
> *Dò cái gì:* ghi `queue.length` đầu mỗi tầng, lặp đúng số đó rồi sang tầng mới; O(n); nhận ra mẹo 'snapshot size' để gom theo tầng.

**P-TREE-005** ⭐⭐⭐⭐
"Biểu diễn graph: adjacency list vs adjacency matrix — Big-O space và 'có cạnh u-v không' của mỗi cái? Khi nào chọn matrix?"
> *Dò cái gì:* list O(V+E) space, kiểm tra cạnh O(deg); matrix O(V²) space, kiểm tra cạnh O(1); chọn matrix khi đồ thị dày/V nhỏ + cần check cạnh nhiều; list cho đồ thị thưa.

**P-TREE-006** ⭐⭐⭐⭐
"Phát hiện chu trình trong đồ thị *có hướng*. Vì sao 'visited' đơn thuần không đủ, cần thêm trạng thái gì?"
> *Dò cái gì:* cần 3 màu / tập 'đang trong stack đệ quy' (gray) phân biệt 'đã xong' (black); gặp lại node gray = cycle; undirected thì khác (dùng parent); hiểu vì sao directed cần recursion-stack set.

**P-TREE-007** ⭐⭐⭐⭐
"Topological sort: dùng để làm gì (vd build dependency, task scheduling)? Hai cách (Kahn's BFS vs DFS) khác nhau ra sao?"
> *Dò cái gì:* sắp xếp DAG sao cho mọi cạnh u→v thì u trước v; Kahn dùng in-degree + queue, phát hiện cycle (còn node chưa lấy = cycle); DFS post-order đảo; gắn use case backend (migration order, job DAG).

**P-TREE-008** ⭐⭐⭐⭐⭐
"Number of islands / connected components: BFS/DFS/Union-Find đều giải. Khi nào Union-Find vượt trội? Big-O của Union-Find?"
> *Dò cái gì:* Union-Find tốt khi cạnh đến *dần* (dynamic connectivity, gộp nhóm online); với path compression + union by rank ≈ O(α(n)) gần hằng số; BFS/DFS tốt cho lưới tĩnh; thể hiện hiểu use case động vs tĩnh.

---

## P-REC — Recursion, Backtracking & DP nhập môn

**P-REC-001** ⭐⭐
"Mọi hàm đệ quy cần *cái gì* để không lặp vô hạn? Recursion và call stack liên quan thế nào tới `RangeError: Maximum call stack`?"
> *Dò cái gì:* base case + tiến về base case mỗi lần gọi; mỗi lần gọi đẩy frame lên call stack, sâu quá → stack overflow; biết đổi sang lặp/tail-call (lưu ý JS engine phần lớn không tối ưu tail-call).

**P-REC-002** ⭐⭐⭐
"Backtracking là gì? Khung chung (choose → explore → un-choose). Cho một bài (permutations / subsets / combination sum)."
> *Dò cái gì:* thử một lựa chọn, đệ quy, rồi *hoàn tác* để thử nhánh khác; cây quyết định + cắt nhánh (pruning); biết Big-O thường mũ/giai thừa và vì sao chấp nhận được (không gian nghiệm lớn).

**P-REC-003** ⭐⭐⭐
"Sinh tất cả tập con (subsets) của một mảng. Hai cách: backtracking vs bit-mask. Big-O?"
> *Dò cái gì:* 2ⁿ tập con; backtracking chọn/không-chọn mỗi phần tử; bit-mask duyệt 0..2ⁿ−1 mỗi bit = lấy/không; O(n·2ⁿ); hiểu vì sao không thể nhanh hơn 2ⁿ.

**P-REC-004** ⭐⭐⭐⭐
"DP là gì so với đệ quy thường? Hai dấu hiệu cho biết bài 'có thể DP' (overlapping subproblems + optimal substructure) — giải thích mỗi cái."
> *Dò cái gì:* DP = đệ quy + nhớ kết quả con (memo) hoặc dựng bảng bottom-up; overlapping = cùng subproblem tính lại nhiều lần; optimal substructure = nghiệm tối ưu ghép từ nghiệm con tối ưu; nhận diện được khi nào KHÔNG phải DP (greedy đủ).

**P-REC-005** ⭐⭐⭐⭐
"Top-down (memoization) vs bottom-up (tabulation): đánh đổi? Khi nào bottom-up giúp giảm space (rolling array)?"
> *Dò cái gì:* top-down dễ viết từ đệ quy, tốn call stack, chỉ tính state cần; bottom-up tránh stack, dễ tối ưu space giữ vài hàng cuối (vd Fibonacci/knapsack 1D); biết đánh đổi rõ ràng.

**P-REC-006** ⭐⭐⭐⭐
"Coin change (số xu ít nhất cho tổng X) hoặc climbing stairs. Định nghĩa state & transition. Vì sao greedy 'tham xu lớn nhất' có thể sai?"
> *Dò cái gì:* state = dp[amount] = min xu; transition dp[a]=min(dp[a−coin]+1); greedy sai với mệnh giá không 'canonical' (vd coins=[1,3,4], X=6); thể hiện hiểu giới hạn greedy.

**P-REC-007** ⭐⭐⭐⭐⭐
"Word break / edit distance: vì sao 2D DP? Cách điền bảng và đọc đáp án. Big-O time/space?"
> *Dò cái gì:* state 2 chiều (i,j) = so khớp tiền tố; transition theo match/insert/delete/replace; O(m·n) time/space, tối ưu xuống O(min(m,n)) space; thể hiện dựng được recurrence, không chỉ kể tên.

**P-REC-008** ⭐⭐⭐⭐⭐
"Trong phỏng vấn backend role này (~35% DSA, biến động), bạn có nên 'cày' DP khó (LIS, matrix DP) không? Lý luận phân bổ thời gian ôn."
> *Dò cái gì:* lý luận theo *xác suất gặp × chi phí ôn*: DP khó hiếm với product/outsourcing; ưu tiên array/hashmap/two-pointer/sliding-window + system-design coding + JS utilities; FAANG thì khác; thể hiện ra quyết định ôn tập như một TL, không cày mù.

---

## P-JS — Coding utilities JS / Node (góc IMPLEMENT)

> *Phần này dò khả năng viết utility production-grade — closure, `this`, async, edge case. Cross-ref **G-ASY** (chọn Promise combinator) — ở đây là TỰ VIẾT.*

**P-JS-001** ⭐⭐⭐
"Implement `debounce(fn, delay)`. Giải thích closure giữ `timer` thế nào. Khác `throttle` ở đâu?"
> *Dò cái gì:* closure giữ biến `timer`, mỗi lần gọi `clearTimeout` rồi đặt lại → chỉ chạy sau khi 'ngừng gọi' delay; throttle = chạy tối đa 1 lần mỗi khoảng; nêu use case (search input vs scroll); xử lý `this`/args (`apply`).

**P-JS-002** ⭐⭐⭐
"Implement `throttle(fn, interval)` (leading edge). Bẫy thường gặp về lần gọi cuối (trailing) là gì?"
> *Dò cái gì:* dùng timestamp/flag để chặn trong interval; bẫy: bỏ mất lần gọi cuối nếu chỉ leading → cân nhắc trailing; hiểu khác biệt hành vi 2 biến thể.

**P-JS-003** ⭐⭐⭐
"Deep clone một object lồng nhau. Vì sao `JSON.parse(JSON.stringify(x))` không đủ? `structuredClone` giải quyết gì?"
> *Dò cái gì:* JSON mất `undefined`/`function`/`Date`/`Map`/`Set`/circular/`BigInt`; `structuredClone` (built-in) xử lý nhiều kiểu + circular nhưng không copy function/DOM; biết khi nào tự viết recursive clone; *(verify support).*

**P-JS-004** ⭐⭐⭐⭐
"Implement `deepClone` tự viết xử lý được circular reference. Cấu trúc nào tránh vòng lặp vô hạn?"
> *Dò cái gì:* đệ quy theo kiểu (array/object/Date/Map/Set), dùng `WeakMap` (seen) để map node-đã-clone → tránh loop & giữ shared reference; nhận ra circular là cái bẫy chính.

**P-JS-005** ⭐⭐⭐
"Implement `deepEqual(a, b)`. Các edge case dễ sai (NaN, +0/−0, Date, thứ tự key, kiểu khác nhau)?"
> *Dò cái gì:* so kiểu, so primitive (lưu ý `NaN!==NaN` → cần `Object.is` cho một số ca), đệ quy object/array, so số lượng key; nêu được vài edge thay vì chỉ `===`.

**P-JS-006** ⭐⭐⭐⭐
"Implement một EventEmitter nhỏ: `on/emit/off/once`. `once` khác `on` thế nào về quản lý listener?"
> *Dò cái gì:* map event→Set/array listener; `emit` gọi tuần tự; `off` gỡ đúng tham chiếu; `once` wrap listener tự gỡ sau lần đầu; cross-ref **G** (EventEmitter runtime). Xử lý lỗi trong listener không phá vòng lặp.

**P-JS-007** ⭐⭐⭐
"Implement `memoize(fn)` cho hàm thuần. Key cache hình thành từ args thế nào, và rò rỉ bộ nhớ là rủi ro gì?"
> *Dò cái gì:* Map key = serialize args (giới hạn: object/thứ tự arg); chỉ đúng cho pure function; cache không bao giờ xoá → memory leak, cân nhắc LRU/size-limit (cross-ref P-SDC LRU); hiểu giả định 'pure'.

**P-JS-008** ⭐⭐⭐⭐
"Implement `Promise.all` polyfill (không dùng `Promise.all`). Xử lý: giữ thứ tự kết quả, reject ngay khi 1 cái fail, mảng rỗng."
> *Dò cái gì:* đếm resolved, ghi kết quả theo index (giữ thứ tự dù xong không theo thứ tự), reject ngay khi một promise reject, resolve `[]` khi input rỗng; cross-ref **G-ASY**.

**P-JS-009** ⭐⭐⭐⭐⭐
"Implement async concurrency limiter (`pLimit`/pool): chạy N tác vụ async nhưng tối đa K cùng lúc. Vì sao `Promise.all([...])` thẳng tay là nguy hiểm với API có rate limit?"
> *Dò cái gì:* `Promise.all` bắn hết cùng lúc → vượt rate limit/quá tải DB/connection pool cạn; limiter dùng hàng đợi + đếm 'đang chạy', khi 1 cái xong kéo cái tiếp; nối sang backpressure; đây là câu rất 'backend TL'.

**P-JS-010** ⭐⭐⭐⭐
"Implement `retry(fn, {retries, backoff})` cho call mạng. Vì sao cần exponential backoff + jitter? Lỗi nào KHÔNG nên retry?"
> *Dò cái gì:* lặp gọi, chờ tăng dần (2^k) + jitter để tránh thundering herd; chỉ retry lỗi *transient/idempotent* (timeout, 503), KHÔNG retry 4xx/validation/non-idempotent mù; cross-ref idempotency ở **I**.

**P-JS-011** ⭐⭐⭐
"Implement `flatten` một mảng lồng độ sâu bất kỳ (không dùng `Array.prototype.flat(Infinity)`). Đệ quy vs stack — đánh đổi?"
> *Dò cái gì:* đệ quy đơn giản nhưng sâu quá → stack overflow; cách iterative dùng stack tránh overflow; O(n); biết `flat(Infinity)` là cách built-in *(verify)* nhưng đề thường cấm để dò tư duy.

**P-JS-012** ⭐⭐⭐⭐⭐
"`curry(fn)` và `compose(...fns)` — implement và giải thích. Trong code backend thật, khi nào dùng FP utilities này *thật sự* giúp, khi nào chỉ làm code khó đọc?"
> *Dò cái gì:* curry gom args dần tới đủ arity; compose nối hàm phải→trái; thể hiện viết được + *judgment*: hữu ích cho pipeline/middleware nhỏ, lạm dụng → khó debug stack trace; TL cân nhắc readability của team.

---

## P-SDC — System-design coding (pseudo-code high-level)

> *Phần "phác thảo code/luồng cho thành phần X" — đề thường gặp ở product/outsourcing. Mong đợi viết được class/luồng + Big-O + nêu giới hạn. Cross-ref **A** (system-design tầng kiến trúc), **J** (cache eviction Redis).*

**P-SDC-001** ⭐⭐⭐⭐
"Implement LRU cache đạt **O(1)** cho cả `get` và `put`. Cấu trúc dữ liệu? (HashMap + doubly linked list — hoặc tận dụng JS `Map`)."
> *Dò cái gì:* Map(key→node) + DLL giữ thứ tự dùng gần đây; `get` move node lên đầu, `put` thêm/cập nhật + evict tail khi vượt capacity; **vì sao DLL**: xoá/chèn O(1) với con trỏ; biết mẹo JS `Map` giữ insertion order (delete+set để 'làm mới'); cross-ref **J-EVICT** (Redis LRU approximated khác bản 'chuẩn O(1)' này).

**P-SDC-002** ⭐⭐⭐⭐
"Implement (code) một **token bucket** rate limiter trong bộ nhớ: refill theo thời gian, cho phép burst. Công thức tính token khi một request tới?"
> *Dò cái gì:* lưu `tokens`, `lastRefill`; khi request: `tokens = min(capacity, tokens + (now−last)*rate)`, nếu `tokens>=1` cho qua & trừ; lazy-refill (không cần timer chạy nền); cross-ref **A-RL** cho trade-off & bản phân tán (lưu ở Redis, atomic).

**P-SDC-003** ⭐⭐⭐⭐
"Implement (code) **sliding window counter** rate limiter. Vì sao nó dung hoà giữa fixed window (burst ở biên) và sliding log (tốn memory)?"
> *Dò cái gì:* nội suy từ count window trước + window hiện tại theo tỷ lệ thời gian; O(1) memory mỗi key; chính xác hơn fixed, rẻ hơn log; cross-ref **A-RL** (so sánh các thuật toán) — ở đây là phần *code công thức nội suy*.

**P-SDC-004** ⭐⭐⭐⭐
"Implement key-value store in-memory **có TTL** (set với expiry, get tự loại key hết hạn). Lazy expiration vs active sweep — đánh đổi?"
> *Dò cái gì:* lưu `{value, expireAt}`; lazy = kiểm tra khi `get` (đơn giản, nhưng key 'chết' chiếm RAM); active = job quét/timer xoá (tốn CPU, giải phóng sớm); Redis dùng kết hợp; cross-ref **J**.

**P-SDC-005** ⭐⭐⭐⭐
"Implement **top-K** phần tử hay gặp nhất trong stream lớn. Vì sao min-heap kích thước K? Big-O so với sort toàn bộ?"
> *Dò cái gì:* đếm tần suất (Map) rồi giữ min-heap size K (đẩy ra phần tử nhỏ nhất) → O(n log k) thay vì O(n log n) sort hết; với stream khổng lồ → count-min sketch (xấp xỉ); nhận ra heap cho 'top-K' là pattern.

**P-SDC-006** ⭐⭐⭐⭐
"Thiết kế (code khung) **cursor-based pagination** thay cho offset. Vì sao offset lớn chậm và lệch khi data thay đổi? Cursor mã hoá gì?"
> *Dò cái gì:* offset `LIMIT n OFFSET m` quét bỏ m hàng (chậm khi m lớn) + bỏ/lặp item khi insert/delete giữa chừng; cursor = giá trị khoá sort cuối (id/timestamp) → `WHERE key > cursor LIMIT n`; mã hoá cursor (base64) + cần khoá sort ổn định, unique; cross-ref **F** (API) nếu đụng contract.

**P-SDC-007** ⭐⭐⭐⭐
"Thiết kế hàm sinh **ID phân tán** (unique, gần sort được theo thời gian) không cần central DB. Các thành phần (timestamp + machine + sequence — kiểu Snowflake)?"
> *Dò cái gì:* ghép timestamp (bit cao → sortable theo thời gian) + node/worker id + counter trong cùng ms; tránh collision đa instance không cần round-trip DB; nêu rủi ro clock skew/clock going backwards; so với UUIDv4 (random, không sort được) và UUIDv7 *(verify)*.

**P-SDC-008** ⭐⭐⭐⭐⭐
"Cho 2 file mỗi cái hàng triệu user id, cần tìm **giao/khác nhau** mà không nạp hết vào RAM. Cách tiếp cận coding?"
> *Dò cái gì:* nếu vừa RAM → Set/HashMap O(n); nếu không → external sort rồi merge tuyến tính, hoặc hash-partition theo `id % K` rồi xử lý từng partition, hoặc bloom filter (xấp xỉ, cross-ref **A-BB**); thể hiện tư duy 'data > RAM' + đánh đổi chính xác.

**P-SDC-009** ⭐⭐⭐⭐
"Merge **k sorted streams/list** (vd k file log đã sort theo thời gian) thành một dòng sorted. Cấu trúc? Big-O?"
> *Dò cái gì:* min-heap chứa head mỗi stream, pop nhỏ nhất + đẩy phần tử kế của stream đó; O(N log k); xử lý stream cạn; ứng dụng thật: merge log/time-series, k-way merge của external sort (cross-ref P-SRCH-008).

**P-SDC-010** ⭐⭐⭐⭐⭐
"Phác thảo (pseudo-code + luồng) một **in-memory job scheduler** chạy task theo delay/lịch, tối đa K worker đồng thời. Những thành phần lõi & rủi ro?"
> *Dò cái gì:* hàng đợi ưu tiên theo `runAt` (min-heap), vòng lặp lấy task tới hạn, pool giới hạn concurrency (cross-ref P-JS-009), retry/backoff khi fail (cross-ref P-JS-010); rủi ro: mất task khi process chết (cần persistence → BullMQ/Redis, cross-ref **J/K**), single point; thể hiện biết 'in-memory đẹp nhưng không bền'.

---

## P-TL — Judgment cấp Tech Lead

**P-TL-001** ⭐⭐⭐
"Phỏng vấn một backend dev bằng bài coding: bạn ưu tiên *quan sát* điều gì hơn 'ra đáp án đúng'? (communication, edge case, complexity, test)."
> *Dò cái gì:* cách phân tích bài & nói thành tiếng (think-aloud), hỏi làm rõ yêu cầu, nêu Big-O, xử lý edge case, tự test; coi 'ra đúng nhanh' chỉ là một phần; thể hiện rubric đánh giá, không nhị phân pass/fail.

**P-TL-002** ⭐⭐⭐
"Một câu phỏng vấn coding 'tốt' khác câu 'gotcha/đố mẹo' ở đâu? Vì sao gotcha hại cho việc tuyển?"
> *Dò cái gì:* câu tốt = nhiều mức độ, phản ánh công việc thật, cho phép trao đổi; gotcha = phụ thuộc 'biết trước mẹo', tạo nhiễu tín hiệu + thiên vị người luyện đề; TL chọn câu đo *cách suy nghĩ* chứ không đo trí nhớ.

**P-TL-003** ⭐⭐⭐⭐
"Khi nào tối ưu Big-O của một đoạn code là *lãng phí*? Cho ví dụ 'O(n²) nhưng n≤100 và chạy 1 lần/ngày' so với 'hot path mỗi request'."
> *Dò cái gì:* tối ưu theo *ngữ cảnh thực thi* (tần suất × kích thước n thực tế), không tối ưu mù; readability/maintainability có giá; đo trước (profile) rồi tối ưu chỗ nóng; câu thần chú 'premature optimization'.

**P-TL-004** ⭐⭐⭐⭐
"Trong code review, một PR dùng thuật toán O(n²) ở chỗ có thể O(n). Bạn quyết định approve/đề nghị sửa dựa trên *yếu tố* nào?"
> *Dò cái gì:* hỏi n thực tế & tần suất, có nằm trên đường nóng không, độ phức tạp thêm khi tối ưu, có dữ liệu/benchmark không; ra quyết định có lý do thay vì 'O(n²) là cấm'; ghi nhận trade-off rõ ràng.

**P-TL-005** ⭐⭐⭐⭐
"Phân biệt 'DSA để *qua phỏng vấn*' với 'DSA *dùng thật* trong backend Node'. Kỹ năng nào của P chuyển hoá thành công việc hằng ngày?"
> *Dò cái gì:* dùng thật: chọn đúng cấu trúc dữ liệu (Map/Set vs array), nhận ra O(n²) ẩn (includes-in-loop), batch/dedup, hiểu cost của thao tác; ít khi tự code red-black tree; thể hiện cái nào 'transfer' và cái nào chỉ là proxy phỏng vấn.

**P-TL-006** ⭐⭐⭐⭐⭐
"AI (Copilot/Cursor) sinh một hàm 'chạy đúng' nhưng `O(n²)` (vd `includes` trong vòng lặp) hoặc dùng `temperature`-kiểu lỗi logic ẩn. Là TL bạn *kiểm soát* chất lượng thuật toán do AI sinh thế nào?"
> *Dò cái gì:* AI lo *cú pháp*, người *chỉ huy & kiểm tra* thuật toán/độ phức tạp; guardrail: review tập trung Big-O & data structure, benchmark/load test, lint pattern xấu, prompt nêu rõ ràng buộc complexity; câu thần chú 'hiểu để chỉ huy, không thuộc để gõ'. *(R-trend, verify công cụ.)*

**P-TL-007** ⭐⭐⭐⭐⭐
"Khi nào bạn *cố tình KHÔNG* tự code thuật toán mà dùng thư viện/DB/hệ có sẵn (vd dedup ở DB, top-K bằng Redis ZSET, sort bằng engine)? Trade-off?"
> *Dò cái gì:* 'đừng viết lại cái đã có & được tối ưu/được test'; đẩy việc nặng xuống tầng phù hợp (DB index, Redis ZSET cho top-K/leaderboard — cross-ref **J**); trade-off: phụ thuộc, vận hành, nhưng đổi lấy đúng đắn & ít bug; TL biết *không* code mới là quyết định.

**P-TL-008** ⭐⭐⭐⭐⭐
"Role TL backend Node/outsourcing này DSA chỉ ~35% và biến động. Bạn xây *chiến lược ôn coding* thế nào để tối ưu kỳ vọng đậu, thay vì cày LeetCode mù?"
> *Dò cái gì:* phân tích theo loại công ty (FAANG nặng DSA vs product/outsourcing thiên system-design coding + JS utilities); ưu tiên patterns hay gặp (array/hashmap/two-pointer/sliding-window/BFS-DFS) + P-JS + P-SDC; luyện *communicate approach*; đặt mức 'đủ tốt' theo target; thể hiện ra quyết định ôn như một TL phân bổ nguồn lực.

---

## Tự kiểm (đếm câu)

| Tag | Số câu |
|---|---|
| P-BIGO | 10 |
| P-ARR | 10 |
| P-PTR | 8 |
| P-STK | 6 |
| P-SRCH | 8 |
| P-LL | 6 |
| P-TREE | 8 |
| P-REC | 8 |
| P-JS | 12 |
| P-SDC | 10 |
| P-TL | 8 |
| **Tổng** | **94** |

---

## ✅ Sau khi LƯU file này vào project — bước tiếp theo

1. **LƯU** `QB_P_coding.md` vào **project knowledge** (để các lần sinh đề mục khác đọc chống trùng, và để Bước C/D dùng).
2. Nói **"Bước B cho mục P"** → tôi dựng **giáo trình ngược** từ 94 câu này (gom cụm khái niệm → bài học, sắp cơ bản → nâng cao, ánh xạ Bài → ID câu).
3. Hoặc **"phỏng vấn mục P đợt 1"** → tôi bốc 10–15 câu trộn dễ→khó, **hỏi xong DỪNG** chờ bạn trả lời, rồi chấm **live** (đáp án lõi + bẫy + trade-off) theo dòng *"dò cái gì"*.

> *Lưu ý độ ổn định:* nội dung DSA/Big-O của mục P rất ổn định — phần cần **verify khi học** chỉ là **cú pháp/API runtime JS-Node** (vd `structuredClone`, `Array.flat`, hành vi `sort` stable, UUIDv7) đã gắn dấu *(verify)* trong các câu liên quan.
