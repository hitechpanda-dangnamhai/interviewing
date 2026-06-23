# 🧪 QB_D — Ngân hàng câu hỏi: Deep-dive dự án cũ

> **Mục D** trong roadmap Tech Lead Backend (🔴 ~75%) · Sinh theo **WORKFLOW 2 — Bước A**.
> Đây là **mục LỚN** → bộ đề phủ HẾT 4 mục con của D (trình bày kiến trúc · khoan sâu "vì sao" · đóng góp cá nhân · hồi tố & cải tiến) + mở rộng các trục mà vòng deep-dive thật hay vặn (số liệu thật · sự cố production · thẩm tra độ tin · giao tiếp kỹ thuật).
> **Chỉ câu hỏi — KHÔNG kèm đáp án.** Mỗi câu có dòng *"dò cái gì"* = tiêu chí ĐẠT, dùng để chấm **live** ở Bước D.
> ⚠️ **Đặc thù mục D:** chất liệu là **dự án THẬT của bạn**, nên không có "đáp án mẫu" — câu hỏi đóng vai *lời vặn của interviewer*, còn *"dò cái gì"* là thứ một câu trả lời ĐẠT bắt buộc phải chạm tới (cấu trúc, sự thật, chiều sâu, độ trung thực). Chuẩn bị sẵn **sơ đồ kiến trúc** (C4 model — draw.io/Excalidraw) và **vài số liệu tải thật** (QPS, data size, latency) trước khi luyện.
> **Chống trùng:**
> - vs **A (System Design)** — A là "*thiết kế hệ X từ đầu (giả định/generic)*"; D là "*kể & biện minh hệ BẠN ĐÃ xây thật, số của bạn, quyết định của bạn, điều bạn sẽ sửa*". Chỗ nào đụng cơ chế thiết kế chung → ghi *(giao với A — đào sâu ở mục đó)*, không hỏi lại lý thuyết.
> - vs **B (Behavioral/Leadership)** *(chưa sinh)* — B là chuyện *xung đột/lãnh đạo/con người thuần*; D giữ ở **deep-dive kỹ thuật của dự án**. Chỗ chạm leadership → ghi *(giao với B)*.
> - vs **E/I/J/K/N/O** — khi câu khoan vào engine-level (index, lock, cache, queue, deploy, trace) → ghi *(giao với …)* và chấm ở góc "trong DỰ ÁN CỦA BẠN dùng nó ra sao", không hỏi lại cơ chế gốc.
> **Tổng: 79 câu.**

## Cách đọc
- **ID:** `D-<tag>-<số>`. Tag mục con liệt kê ở mục lục dưới.
- **Độ khó:** ⭐ định nghĩa/khởi động → ⭐⭐⭐⭐⭐ phân biệt senior với TL thật (chịu được vặn sâu, trung thực dưới sức ép).
- **"Dò cái gì":** điều người trả lời PHẢI chạm tới mới tính ĐẠT (không phải đáp án — chỉ là mốc chấm).

## Bối cảnh 2026 (vì sao mục D ngày càng nặng)
- Xu hướng phỏng vấn 2026: vì AI sinh được kiến trúc "nghe rất hợp lý", interviewer **đào sâu hơn vào kinh nghiệm production thật**, vặn tới **ranh giới giữa "thứ bạn ĐỌC" và "thứ bạn ĐÃ XÂY"** (vd: "anh nói dùng write-ahead log — đã tự implement chưa, hỏng ở đâu?"). *(theo các nguồn tuyển dụng/eng-leadership 5–6/2026 — kiểm chứng khi học.)*
- Ở senior+, lỗi kinh điển là nói **"chúng tôi"** thay vì **"tôi"** và **nhận vơ công**; ~80% lời kể nên là đóng góp cá nhân ("I"). Khung kể nên dùng **STAR**, hoặc **CARL** (Context–Action–Result–**Learnings**) cho dự án dài để làm nổi *bài học rút ra*.
- ⚠️ Mọi câu chạm **tên/đời công nghệ, model, giá, con số benchmark** đều cần *(verify)* tại nguồn chính thức trước khi khẳng định.

---

## Mục lục mục con (ID prefix → số câu)

| Tag | Mục con | Số câu |
|---|---|---|
| **D-ARCH** | Trình bày kiến trúc tổng thể (opener, C4, tailor audience) | 10 |
| **D-WHY** | Khoan sâu "vì sao" mỗi lựa chọn | 12 |
| **D-SCALE** | Số liệu & quy mô thật (QPS · data · latency · cost) | 9 |
| **D-ROLE** | Đóng góp cá nhân — "I vs we" · scope · ownership | 9 |
| **D-TRADE** | Trade-off, ràng buộc & quyết định | 8 |
| **D-PROB** | Sự cố production · debug · outage · on-call | 9 |
| **D-RETRO** | Hồi tố & cải tiến (làm lại sẽ đổi gì · Learnings) | 8 |
| **D-CRED** | Thẩm tra độ sâu thật (read-vs-built · trung thực) | 8 |
| **D-COMM** | Trình bày & "hiểu để chỉ huy & kiểm tra" | 6 |

---

## D-ARCH — Trình bày kiến trúc tổng thể

**D-ARCH-001** ⭐⭐
"Kể về hệ thống phức tạp nhất anh từng xây — anh sẽ mở đầu thế nào để người nghe không lạc?"
> *Dò cái gì:* mở bằng **1 câu nêu bài toán + bối cảnh business**, rồi dùng cách *"table of contents"* (nêu 2–3 trục sẽ kể: kiến trúc / quyết định / sự cố) trước khi lao vào chi tiết; **front-load impact**, không kể tuần tự từ tầng dưới lên.

**D-ARCH-002** ⭐⭐⭐
"Vẽ kiến trúc tổng thể dự án đó trong 2 phút (whiteboard) — gồm những thành phần nào?"
> *Dò cái gì:* vẽ được luồng client → gateway/LB → service(s) → DB/cache/queue → hệ ngoài; gọi đúng tên building block; biết **dừng ở mức container**, không sa lầy chi tiết class.

**D-ARCH-003** ⭐⭐⭐
"Dùng C4 model, anh mô tả hệ ở mức Context và Container ra sao?"
> *Dò cái gì:* phân biệt **Context** (hệ + actor + hệ ngoài) vs **Container** (app/DB/queue chạy độc lập); chọn đúng mức zoom theo người nghe; không nhảy thẳng vào code.

**D-ARCH-004** ⭐⭐
"Hệ này giải quyết bài toán kinh doanh gì? Vì sao nó cần tồn tại?"
> *Dò cái gì:* nối kiến trúc với **mục tiêu business / người dùng**, không kể tech thuần; cho thấy hiểu *why* của dự án chứ không chỉ *how*.

**D-ARCH-005** ⭐⭐⭐
"Trace giúp tôi một request điển hình: từ lúc vào tới khi trả kết quả, nó đi qua đâu?"
> *Dò cái gì:* đi end-to-end qua từng tầng (auth → validate → business → DB/cache → response); chỉ rõ chỗ **đồng bộ vs bất đồng bộ**; biết đâu là critical path.

**D-ARCH-006** ⭐⭐⭐⭐
"Điểm KHÓ nhất về mặt kỹ thuật của hệ này là gì, và vì sao nó khó?"
> *Dò cái gì:* chỉ ra **constraint thật** (consistency, scale, latency, legacy, thời gian thực…), không phải "code nhiều"; cho thấy chiều sâu nhận thức bài toán.

**D-ARCH-007** ⭐⭐⭐
"Nếu người nghe là VP/PM (không sâu kỹ thuật), anh kể lại hệ này khác đi thế nào?"
> *Dò cái gì:* **điều chỉnh độ chi tiết theo audience** — VP: impact/cost/rủi ro; PM: người dùng/timeline; quan sát engagement để tăng/giảm chi tiết.

**D-ARCH-008** ⭐⭐⭐
"Hệ chia thành mấy service/module? Anh chia ranh giới theo nguyên tắc gì?"
> *Dò cái gì:* chia theo **bounded context / domain / độ biến động**, không theo tầng kỹ thuật; biện minh được ranh giới *(giao với A — decomposition)*.

**D-ARCH-009** ⭐⭐⭐⭐
"Phần nào của hệ anh giữ 'đơn giản nhất có thể' và phần nào cố tình làm phức tạp? Vì sao?"
> *Dò cái gì:* nhận ra over-/under-engineering; **biện minh nơi đầu tư phức tạp là đáng giá** (và nơi cố tình giữ ngu ngơ cho dễ bảo trì).

**D-ARCH-010** ⭐⭐⭐
"Nếu chỉ được giữ 1 sơ đồ để giải thích cả hệ, anh giữ sơ đồ nào?"
> *Dò cái gì:* chọn được **view truyền tải nhiều thông tin nhất** (thường là container / data-flow); hiểu sơ đồ là công cụ *giao tiếp*, không phải trang trí.

---

## D-WHY — Khoan sâu "vì sao" mỗi lựa chọn

**D-WHY-001** ⭐⭐⭐
"Vì sao chọn database đó (Postgres/Mongo/…) cho dự án này — không phải cái khác?"
> *Dò cái gì:* lý do theo **access pattern + yêu cầu nhất quán + năng lực team**, không "quen tay"; *(giao với E — SQL vs NoSQL)*.

**D-WHY-002** ⭐⭐⭐⭐
"Hệ là monolith hay microservices? Vì sao? Nếu chọn ngược lại thì sao?"
> *Dò cái gì:* biện minh theo **quy mô team / độ phức tạp / cách triển khai**; không chọn microservices vì "mốt"; ý thức được **chi phí vận hành** của phương án.

**D-WHY-003** ⭐⭐⭐
"Vì sao có message queue ở đây? Bỏ nó đi thì hỏng chỗ nào?"
> *Dò cái gì:* decoupling / buffering / retry / async; chỉ ra hệ quả nếu gọi đồng bộ trực tiếp (coupling, mất request lúc tải cao) *(giao với K)*.

**D-WHY-004** ⭐⭐⭐
"Anh cache cái gì, ở đâu, vì sao? Cache invalidation xử lý ra sao?"
> *Dò cái gì:* chọn **layer cache đúng** + chiến lược invalidation rõ ràng; nhận ra cache sai gây **stale data** *(giao với J)*.

**D-WHY-005** ⭐⭐⭐⭐
"Vì sao dùng framework/ngôn ngữ đó? Đây là quyết định của anh hay đã có sẵn?"
> *Dò cái gì:* phân biệt **ràng buộc kế thừa vs lựa chọn chủ động**; trung thực về context thay vì gán cho mình mọi quyết định.

**D-WHY-006** ⭐⭐⭐⭐⭐
"Anh nói hệ dùng eventual consistency — kể một bug NGƯỜI DÙNG nhìn thấy do nó gây ra, trong hệ anh OWN."
> *Dò cái gì:* có **câu chuyện thật** (không lý thuyết): triệu chứng + cách phát hiện + cách sửa; đây chính là *ranh giới đọc-vs-làm*.

**D-WHY-007** ⭐⭐⭐
"Vì sao đặt cái này đồng bộ còn cái kia bất đồng bộ?"
> *Dò cái gì:* phân biệt **critical path** (cần đồng bộ) vs **side-effect** (async được — email, log, analytics); hiểu đánh đổi latency vs reliability.

**D-WHY-008** ⭐⭐⭐⭐
"Anh dùng transaction / idempotency ở đâu trong hệ? Chỗ nào và vì sao?"
> *Dò cái gì:* chỉ đúng nơi cần **atomicity** / chống **double-processing**; gắn với endpoint/luồng cụ thể *(giao với E/I/K)*.

**D-WHY-009** ⭐⭐⭐
"Anh có tách read/write không? Có đọc từ replica không? Vì sao?"
> *Dò cái gì:* hiểu **read scaling** + rủi ro **replication lag** (đọc dữ liệu cũ); biện minh theo tỉ lệ đọc/ghi thật *(giao với E)*.

**D-WHY-010** ⭐⭐⭐⭐
"Quyết định kiến trúc nào trong hệ này anh tự tin nhất là ĐÚNG? Lấy gì chứng minh?"
> *Dò cái gì:* gắn quyết định với **dữ liệu/kết quả đo được** (latency giảm, sự cố giảm, cost giảm), không cảm tính.

**D-WHY-011** ⭐⭐⭐⭐⭐
"AI vẽ kiến trúc 'nghe rất hợp lý' rất nhanh. Chỉ cho tôi 1 chỗ anh đã TỰ implement và gặp trục trặc gì."
> *Dò cái gì:* chứng cứ **hands-on thật** (lock, retry, WAL, backpressure…); kể được *"went wrong"* + cách sửa — thứ một câu trả lời "đọc blog" không có.

**D-WHY-012** ⭐⭐⭐
"Có công nghệ nào trong hệ anh KHÔNG hiểu sâu, chỉ dùng theo người khác không? Trung thực."
> *Dò cái gì:* **tự nhận biên giới hiểu biết**; không giả vờ thông thạo; thể hiện self-awareness (chính nó tăng độ tin cho phần còn lại).

---

## D-SCALE — Số liệu & quy mô thật

**D-SCALE-001** ⭐⭐⭐
"Hệ này phục vụ bao nhiêu user/request? Cho con số thật."
> *Dò cái gì:* có **số cụ thể** (DAU/MAU, QPS/RPS, peak vs avg); không "rất nhiều"/"khá lớn".

**D-SCALE-002** ⭐⭐⭐⭐
"Peak QPS là bao nhiêu, và anh đo bằng cách nào?"
> *Dò cái gì:* biết **nguồn số** (metrics/APM/load test) chứ không bịa; phân biệt **peak vs trung bình** và biết peak xảy ra khi nào.

**D-SCALE-003** ⭐⭐⭐
"Dung lượng dữ liệu cỡ nào? Tăng bao nhiêu mỗi tháng?"
> *Dò cái gì:* biết **data size + growth rate**; nối số đó với quyết định archive/shard/retention *(giao với E)*.

**D-SCALE-004** ⭐⭐⭐⭐
"Latency p50/p95/p99 của endpoint chính là bao nhiêu? Vì sao p99 cao hơn nhiều?"
> *Dò cái gì:* dùng **percentile** (không chỉ trung bình); giải thích **tail latency** (GC, lock contention, cold cache, N+1, network) *(giao với O)*.

**D-SCALE-005** ⭐⭐⭐⭐
"Bottleneck THỰC SỰ của hệ nằm ở đâu? Làm sao anh biết?"
> *Dò cái gì:* xác định nghẽn bằng **đo** (profiling/trace/metrics), không đoán; *(giao với A — nhận diện bottleneck)*.

**D-SCALE-006** ⭐⭐⭐⭐⭐
"Nếu tải tăng 10×, chỗ nào VỠ trước? Anh đã chuẩn bị gì cho điều đó?"
> *Dò cái gì:* biết **điểm yếu mở rộng + headroom hiện tại**; có kế hoạch cụ thể (scale ngang, cache, shard, queue) thay vì "thì scale lên".

**D-SCALE-007** ⭐⭐⭐
"SLA/SLO của hệ là gì? Có đạt không?"
> *Dò cái gì:* phân biệt **SLA vs SLO**; biết target uptime/latency thật và mức đạt thực tế *(giao với O)*.

**D-SCALE-008** ⭐⭐⭐
"Chi phí vận hành hệ cỡ nào? Khoản nào tốn nhất?"
> *Dò cái gì:* **cost awareness** (compute/DB/egress/3rd-party); ý thức tối ưu chi phí là việc của TL, không chỉ của finance.

**D-SCALE-009** ⭐⭐⭐⭐
"Con số nào trong hệ làm anh BẤT NGỜ khi đo lần đầu?"
> *Dò cái gì:* kể được khoảng cách giữa **giả định và thực đo**; cho thấy **thói quen đo lường** thay vì tin trực giác.

---

## D-ROLE — Đóng góp cá nhân ("I vs we")

**D-ROLE-001** ⭐⭐
"Trong dự án này, phần nào do CHÍNH anh làm? Dùng 'tôi' chứ đừng 'chúng tôi'."
> *Dò cái gì:* tách rõ **đóng góp cá nhân** (~80% là "I"); **không nhận vơ** công của team; mô tả scope đúng.

**D-ROLE-002** ⭐⭐⭐
"Anh là người RA quyết định kiến trúc hay người thực thi quyết định của người khác?"
> *Dò cái gì:* trung thực về vai trò (**owner vs contributor**); không phóng đại; nếu là contributor thì kể đóng góp thật trong scope đó.

**D-ROLE-003** ⭐⭐⭐⭐
"Quyết định kỹ thuật quan trọng nhất mà anh TỰ đưa ra trong dự án là gì?"
> *Dò cái gì:* 1 quyết định **cụ thể + lý do + kết quả**; cho thấy quyền tự quyết thật, không phải "team quyết".

**D-ROLE-004** ⭐⭐⭐
"Phần khó nhất anh đích thân giải quyết — kể chi tiết kỹ thuật."
> *Dò cái gì:* **chiều sâu hands-on** ở chính phần mình làm; chi tiết đủ mức không thể "học thuộc" mà phải đã làm thật.

**D-ROLE-005** ⭐⭐⭐⭐
"Anh dẫn dắt / ảnh hưởng team trong dự án này thế nào?"
> *Dò cái gì:* **leadership kỹ thuật** (đặt chuẩn, review, mentor, align hướng), không chỉ "viết code"; *(giao với B nếu là leadership thuần con người)*.

**D-ROLE-006** ⭐⭐⭐
"Có phần nào trong hệ anh KHÔNG đụng tới không? Trung thực về điều đó."
> *Dò cái gì:* **thành thật về ranh giới đóng góp**; nghịch lý là việc thừa nhận "tôi không làm phần X" làm tăng độ tin cho phần "tôi đã làm".

**D-ROLE-007** ⭐⭐⭐⭐
"Nếu tôi hỏi đồng nghiệp anh về vai trò anh trong dự án này, họ sẽ nói gì?"
> *Dò cái gì:* **tự đánh giá khớp thực tế**; không tô vẽ; cho thấy self-awareness và việc kể là phiên bản đồng đội xác nhận được.

**D-ROLE-008** ⭐⭐⭐
"Anh thuyết phục team/sếp theo một hướng kỹ thuật bằng cách nào?"
> *Dò cái gì:* dùng **dữ liệu / prototype / POC** để thuyết phục, không bằng chức danh hay giọng to; *(giao với B)*.

**D-ROLE-009** ⭐⭐⭐⭐
"Phần nào của dự án anh tự hào nhất, và phần nào anh thấy mình làm chưa tốt?"
> *Dò cái gì:* **cân bằng tự hào + tự phê**; dám nêu điểm yếu cá nhân cụ thể (không phải "điểm yếu là tôi quá cầu toàn").

---

## D-TRADE — Trade-off, ràng buộc & quyết định

**D-TRADE-001** ⭐⭐⭐⭐
"Quyết định kiến trúc lớn nhất — anh đã cân nhắc PHƯƠNG ÁN nào khác và vì sao loại?"
> *Dò cái gì:* nêu được **alternative thật + tiêu chí loại**; không "chỉ có một cách"; cho thấy đã đánh đổi có ý thức *(giao với A — trade-off articulation)*.

**D-TRADE-002** ⭐⭐⭐
"Ràng buộc lớn nhất khi làm hệ này (thời gian / người / tiền / legacy) định hình quyết định ra sao?"
> *Dò cái gì:* nối **constraint thực tế** với lựa chọn kỹ thuật; thể hiện ra quyết định trong môi trường có giới hạn, không phải phòng lab.

**D-TRADE-003** ⭐⭐⭐⭐
"Có chỗ nào anh chọn 'giải pháp TỆ HƠN về kỹ thuật' vì lý do thực tế không?"
> *Dò cái gì:* **pragmatism** — đánh đổi "đẹp" lấy "ship đúng hạn / đơn giản bảo trì"; biện minh được vì sao đó là lựa chọn đúng *trong bối cảnh*.

**D-TRADE-004** ⭐⭐⭐⭐
"Build vs buy: có thành phần nào anh tự viết mà lẽ ra nên dùng có sẵn (hoặc ngược lại)?"
> *Dò cái gì:* tư duy **build-vs-buy**; nhận ra **chi phí ẩn** của tự viết (bảo trì, edge case) vs khóa nhà cung cấp khi mua.

**D-TRADE-005** ⭐⭐⭐
"Hệ đánh đổi consistency vs availability ở đâu? Anh chọn bên nào và vì sao?"
> *Dò cái gì:* hiểu **CAP áp dụng cụ thể** vào từng phần (vd order=consistency, feed=availability); chọn có chủ đích *(giao với I)*.

**D-TRADE-006** ⭐⭐⭐⭐
"Một quyết định ĐÚNG-tại-thời-điểm-đó nhưng giờ thấy sai — kể lại bối cảnh."
> *Dò cái gì:* phân biệt **"sai theo hindsight" vs "sai khi quyết"**; công bằng với quá khứ thay vì tự đánh mình hoặc đổ lỗi.

**D-TRADE-007** ⭐⭐⭐
"Có 'tech debt' nào anh CỐ Ý tạo ra để kịp deadline không? Anh quản lý nó sau đó thế nào?"
> *Dò cái gì:* **nợ kỹ thuật có chủ đích + kế hoạch trả** (ticket, lịch refactor); không giả vờ "code của tôi luôn sạch" *(giao với C/Q)*.

**D-TRADE-008** ⭐⭐⭐⭐⭐
"Nếu phải cắt 50% scope của hệ mà vẫn giữ giá trị cốt lõi, anh cắt gì?"
> *Dò cái gì:* biết đâu là **lõi tạo giá trị** vs phụ; tư duy **ưu tiên dưới sức ép**, dấu hiệu của TL.

---

## D-PROB — Sự cố production, debug & on-call

**D-PROB-001** ⭐⭐⭐⭐
"Sự cố production TỆ NHẤT của hệ này — chuyện gì đã xảy ra?"
> *Dò cái gì:* kể incident thật: **triệu chứng + impact + timeline**; không né, không đổ hết cho người khác.

**D-PROB-002** ⭐⭐⭐⭐⭐
"Bug khó nhất anh từng debug trong hệ — anh khoanh vùng nguyên nhân thế nào?"
> *Dò cái gì:* **phương pháp debug có hệ thống** (repro → bisect → đọc log/metric/trace → giả thuyết → kiểm chứng), không "may mắn tìm ra".

**D-PROB-003** ⭐⭐⭐⭐
"Lần outage đó anh khôi phục bằng cách nào? Mất bao lâu (MTTR)?"
> *Dò cái gì:* có **quy trình recover** (rollback / failover / hotfix / feature flag) + ý thức về **MTTR**; *(giao với N/O)*.

**D-PROB-004** ⭐⭐⭐⭐
"Sau sự cố, anh làm gì để nó KHÔNG lặp lại?"
> *Dò cái gì:* **postmortem blameless + action item cụ thể** (alert mới, test hồi quy, guardrail); cho thấy học từ lỗi, không chỉ vá nóng.

**D-PROB-005** ⭐⭐⭐
"Có lần nào dữ liệu bị sai/hỏng không? Anh phát hiện và xử lý ra sao?"
> *Dò cái gì:* **data integrity issue thật**; backfill/reconcile; ý thức về tính đúng của dữ liệu (thường nguy hiểm hơn downtime).

**D-PROB-006** ⭐⭐⭐⭐
"Lỗi nào 'chạy ổn ở dev/staging nhưng VỠ ở production'? Vì sao khác?"
> *Dò cái gì:* nhận ra **khác biệt môi trường** (tải thật, data thật, concurrency, config, secret); *(giao với L/N)*.

**D-PROB-007** ⭐⭐⭐⭐⭐
"Một lỗi do RACE CONDITION / concurrency trong hệ — kể lại."
> *Dò cái gì:* hiểu **concurrency bug thật** (lost update, double-spend, double-charge); cách phát hiện + fix (lock / idempotency / version) *(giao với I)*.

**D-PROB-008** ⭐⭐⭐
"On-call cho hệ này thế nào? Alert nào hay kêu nhất và anh xử lý ra sao?"
> *Dò cái gì:* kinh nghiệm **vận hành thật**; ý thức giảm **alert noise / fatigue**; runbook *(giao với O)*.

**D-PROB-009** ⭐⭐⭐⭐
"Có lần nào anh phải quyết định nhanh khi 'đang cháy nhà', thiếu thông tin? Anh quyết sao?"
> *Dò cái gì:* **ra quyết định dưới bất định** + chấp nhận rủi ro có kiểm soát (mitigate trước, root-cause sau); *(giao với B)*.

---

## D-RETRO — Hồi tố & cải tiến

**D-RETRO-001** ⭐⭐⭐
"Làm lại hệ này từ đầu, anh đổi gì lớn nhất? Vì sao?"
> *Dò cái gì:* 1–2 thay đổi **cụ thể + lý do từ trải nghiệm thật**; tránh "không đổi gì cả" (red flag thiếu phản tư).

**D-RETRO-002** ⭐⭐⭐⭐
"Bài học lớn nhất (Learnings) từ dự án này — và nó đã giúp gì cho dự án SAU?"
> *Dò cái gì:* rút **bài học chuyển giao được** (CARL); chứng minh đã **áp dụng lại** lần sau (dấu hiệu wisdom của senior+).

**D-RETRO-003** ⭐⭐⭐
"Quyết định nào về sau chứng minh là TỐT hơn anh nghĩ ban đầu?"
> *Dò cái gì:* nhận diện **quyết định 'lãi kép'** (đầu tư observability/test/abstraction/đặt tên đúng); hiểu giá trị tích lũy theo thời gian.

**D-RETRO-004** ⭐⭐⭐⭐
"Phần nào của hệ trở thành 'gánh nặng' theo thời gian? Vì sao nó thoái hóa?"
> *Dò cái gì:* hiểu vì sao kiến trúc/code **xuống cấp** (coupling tăng, thiếu test, scope creep, người rời đi) *(giao với Q)*.

**D-RETRO-005** ⭐⭐⭐
"Nếu yêu cầu/tải tăng gấp đôi, kiến trúc cũ còn đáp ứng được không? Cần sửa gì?"
> *Dò cái gì:* đánh giá **khả năng tiến hóa** của thiết kế; biết điểm bắt buộc phải refactor vs điểm còn dư địa.

**D-RETRO-006** ⭐⭐⭐⭐
"Có công nghệ mới nào ra đời SAU dự án mà giờ anh sẽ dùng thay không?"
> *Dò cái gì:* **cập nhật xu hướng + biện minh thay thế** (lợi ích thật, không "mới = tốt"); *(verify tên/đời công nghệ cụ thể)* *(giao với R)*.

**D-RETRO-007** ⭐⭐⭐
"Điều gì anh ƯỚC mình đã đo/log ngay từ đầu nhưng đã không làm?"
> *Dò cái gì:* ý thức **observability-first**; học từ **điểm mù dữ liệu** đã từng cản trở debug.

**D-RETRO-008** ⭐⭐⭐⭐
"Nhìn lại, scope ban đầu có đúng không? Có gì over-built hay under-built?"
> *Dò cái gì:* tự phê về **YAGNI vs thiếu chuẩn bị**; cân bằng giữa "vẽ vời quá sớm" và "thiếu nền để mở rộng".

---

## D-CRED — Thẩm tra độ sâu thật (read-vs-built)

**D-CRED-001** ⭐⭐⭐⭐⭐
"Anh nói dùng [pattern X]. Anh đã TỰ tay implement nó, hay đọc/dùng thư viện có sẵn?"
> *Dò cái gì:* **ranh giới đọc-vs-làm**; trung thực; nếu tự làm thì có chi tiết hands-on chứng minh, nếu dùng lib thì nói thẳng.

**D-CRED-002** ⭐⭐⭐⭐
"Giải thích cơ chế bên trong của [thành phần anh vừa nhắc] — sâu hết mức anh hiểu."
> *Dò cái gì:* **chiều sâu thật phía sau buzzword**; biết **dừng đúng chỗ "tôi không chắc"** thay vì bịa tiếp.

**D-CRED-003** ⭐⭐⭐
"Có thuật ngữ nào trong CV / câu trả lời mà anh không tự tin giải thích kỹ không?"
> *Dò cái gì:* **tự nhận biên giới**, không **resume-padding**; thành thật về thứ chỉ "nghe qua".

**D-CRED-004** ⭐⭐⭐⭐
"Tôi sẽ đào 'tại sao' liên tục cho tới khi anh nói 'tôi không biết'. Anh sẵn sàng chứ?"
> *Dò cái gì:* phản ứng khi **chạm đáy hiểu biết**: nói thẳng "tới đây tôi không chắc" + **cách mình sẽ tra cứu/kiểm chứng**, không chống chế.

**D-CRED-005** ⭐⭐⭐⭐
"Con số anh vừa đưa (QPS/latency) — anh nhớ CHÍNH XÁC hay đang ước? Trung thực."
> *Dò cái gì:* phân biệt **số đo thật vs ước lượng**; **gắn nhãn đúng** ("khoảng ~", "tôi nhớ là…"); giữ uy tín thay vì khẳng định chắc nịch số bịa.

**D-CRED-006** ⭐⭐⭐⭐⭐
"Nếu tôi mở codebase này ngay bây giờ, anh chỉ ra được đúng những chỗ anh đã viết chứ?"
> *Dò cái gì:* **chứng cứ quyền sở hữu code thật**; không nhận phần người khác; mô tả khớp được với cấu trúc thực.

**D-CRED-007** ⭐⭐⭐
"AI giờ vẽ được kiến trúc nghe rất xịn. Điều gì khiến câu trả lời của anh KHÁC một bài blog?"
> *Dò cái gì:* **chi tiết production thật** (số liệu, sự cố cụ thể, đánh đổi đã thực sự gặp) — thứ AI/blog tổng quát không có.

**D-CRED-008** ⭐⭐⭐⭐
"Phần nào trong dự án anh đã tô hồng / đơn giản hóa khi kể? Giờ kể lại cho đúng."
> *Dò cái gì:* **tự hiệu chỉnh**; phân biệt "kể cho gọn" (ổn) vs "nói quá" (red flag); chính trực dưới sức ép.

---

## D-COMM — Trình bày & "hiểu để chỉ huy & kiểm tra"

**D-COMM-001** ⭐⭐⭐
"Theo câu thần chú của roadmap — 'hiểu để chỉ huy, không thuộc để gõ' — trong dự án này anh CHỈ HUY (định hình quyết định) ở đâu, KIỂM TRA (bắt lỗi máy/người) ở đâu?"
> *Dò cái gì:* phân biệt **vai trò ra-quyết-định** vs **kiểm-soát-chất-lượng**; cho ví dụ cụ thể cho từng loại trong chính dự án.

**D-COMM-002** ⭐⭐⭐⭐
"Giả sử AI viết phần này 'đúng cú pháp nhưng sai kiến trúc'. Trong hệ của anh, chỗ nào chỉ người HIỂU mới bắt được lỗi đó?"
> *Dò cái gì:* chỉ ra 1 chỗ **đúng-cú-pháp-sai-hành-vi** (vd chọn fine-tune cho dữ liệu hay đổi thay vì RAG; để `temperature` cao cho task cần ổn định; đặt side-effect vào critical path); cho thấy **tư duy kiểm tra của TL**.

**D-COMM-003** ⭐⭐⭐
"Giải thích hệ này cho một người KHÔNG kỹ thuật trong 3 câu."
> *Dò cái gì:* **trừu tượng hóa đúng mức + ẩn dụ**; "hiểu sâu mới nói đơn giản được" — dấu hiệu nắm bản chất.

**D-COMM-004** ⭐⭐⭐⭐
"Khi interviewer ngắt giữa chừng để hỏi sâu, anh giữ mạch kể thế nào?"
> *Dò cái gì:* **trả lời thẳng câu hỏi rồi quay lại "table of contents"**; không lạc đề; đọc tín hiệu người nghe để tăng/giảm chi tiết.

**D-COMM-005** ⭐⭐⭐
"Một đồng đội không đồng ý cách anh kể về đóng góp CHUNG — anh chỉnh cách kể thế nào để vừa đúng vừa công bằng?"
> *Dò cái gì:* **chính trực khi quy công** + tôn trọng đóng góp người khác; vẫn nêu rõ phần mình *(giao với B)*.

**D-COMM-006** ⭐⭐⭐⭐
"Nếu chỉ có 60 giây để gây ấn tượng về dự án này, anh nói gì?"
> *Dò cái gì:* nén được **bài toán + giải pháp + impact đo được** vào một *elevator pitch*; **front-load value**, không lan man kỹ thuật.

---

## Tự kiểm (trước khi sang Bước B)
- [ ] Đã đọc các `QB_*.md` khác để tránh trùng? *(D không lặp cơ chế engine của E/I/J/K/N/O — chỉ hỏi "trong dự án CỦA BẠN dùng nó ra sao"; không lặp thiết kế-generic của A.)*
- [ ] Mỗi câu có **ID + độ khó + dòng "dò cái gì"**, **không** kèm đáp án? ✅
- [ ] Phủ đủ 4 mục con roadmap + các trục deep-dive thật (số liệu, sự cố, độ tin, giao tiếp)? ✅ (79 câu)
- [ ] Câu chạm tên/đời công nghệ đã gắn *(verify)*? ✅ (D-RETRO-006, bối cảnh 2026)

> **Bước A xong.** Hãy **lưu file này vào project** (để các mục sau dò trùng). Khi bạn sẵn sàng, mình chạy **Bước B** — dựng giáo trình ngược từ bộ đề (gom câu thành bài học, sắp cơ bản → nâng cao, ánh xạ Bài → ID câu).
