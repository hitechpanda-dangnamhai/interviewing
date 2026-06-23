# 🤝 QB_C — Ngân hàng câu hỏi: Hiring & Team Building

> **Mục C** trong roadmap Tech Lead Backend (🟡 ~35–40% · "mục riêng cho TL") · Sinh theo **WORKFLOW 2 — Bước A**.
> Đây là **mục LỚN** → bộ đề phủ HẾT 5 mục con roadmap (phỏng vấn ứng viên backend · onboarding · code review standard & tooling · tech debt management · phân chia công việc & delegation) + mở rộng cấp TL (hiring funnel/leveling/closing, team growth, 1:1, psychological safety, team health, bus factor, TL vs EM).
> **Chỉ câu hỏi — KHÔNG kèm đáp án.** Mỗi câu có dòng *"dò cái gì"* = tiêu chí ĐẠT, dùng để chấm **live** ở Bước D.
> **Tổng: 82 câu.**

## Chống trùng (đã đọc các QB sẵn có trong project)
- **`QB_Q_designpatterns.md`** sở hữu góc *nội dung* review: `Q-099` (junior áp SOLID 1-1 interface), `Q-399` (thấy smell nhưng deadline → fix/log/bỏ), `Q-364` (smell báo trước nợ kỹ thuật), `Q-424` (ADR). → **C KHÔNG lặp "review một đoạn code cụ thể nên góp gì".** C sở hữu **quy trình & chuẩn review của team** (blocking vs nit, Conventional Comments, SLA, PR size, gatekeeping, metric) và **quản trị nợ ở cấp org** (thuyết phục business, ngân sách, đo/visualize, khi nào KHÔNG trả). Đụng "smell/ADR là gì" → trỏ Q.
- **`QB_L_testing.md`** sở hữu `L-TL` (DoD về test, chuẩn chất lượng *test code* trong review, quản trị flaky test là nợ). → **C chỉ bàn DoD/quality-gate ở góc "ngăn nợ mới"** (`C-DEBT-011`), cross-ref L cho phần test cụ thể; không lặp.
- **`QB_R_moderntrends.md`** sở hữu `R-AICODE` (review khi *phần lớn code do AI viết*, governance/supply-chain, năng suất AI, AI-assisted interview format). → **C chỉ giữ 2 câu chạm AI** ở góc *vận hành review/phỏng vấn của TL* (`C-REV-010`, `C-INT-015`), cross-ref R cho phần governance sâu.
- **`QB_N_devops.md`** chạm "onboard nhanh → mầm platform/IDP" (`N`) và `QB_R` chạm "onboarding chậm trigger IDP" (`R-317`). → **C sở hữu *quy trình onboarding con người* (30/60/90, buddy, ramp metric)**; khi đụng "khi nào cần IDP" → trỏ N/R.
- **Mục B (Behavioral/Leadership) không có file QB** (theo roadmap là STAR — kinh nghiệm thật). → C giữ các câu **"anh *thiết kế hệ thống/quy trình* gì"** (team-building), tránh dạng "kể về một lần…" (đó là đất của B/D).

## Cách đọc
- **ID:** `C-<tag>-<số>`. Tag mục con liệt kê ở mục lục dưới.
- **Độ khó:** ⭐ định nghĩa cơ bản → ⭐⭐⭐⭐⭐ phân biệt senior với TL thật.
- **"Dò cái gì":** điều người trả lời PHẢI chạm tới mới tính ĐẠT (không phải đáp án — chỉ là mốc chấm).

## Bối cảnh (tính tại 6/2026 — *verify lại khi học, nhất là benchmark & tên tool*)
- Đây là **mục mềm (leadership)**: tên framework & nguyên tắc ổn định, nhưng *số liệu/benchmark* (vd "structured interview ~2× predictive validity của unstructured", "PR <400 dòng", "review SLA 24–48h", "20% capacity cho nợ") là **rule-of-thumb tổng hợp từ thực hành**, không phải hằng số — dùng để lập luận, đừng đọc như con số tuyệt đối.
- **Tooling hay nhắc:** structured interview + **rubric/scorecard** (Greenhouse-style), **calibration session**, **Conventional Comments** (spec cộng đồng — *verify*), Google *eng-practices* "code health" standard, hotspot/complexity analysis (CodeScene-style — *verify*), CODEOWNERS, DORA metrics, **Team Topologies**, Fowler "technical debt quadrant", Tannenbaum-Schmidt/situational delegation.
- **Sách nền:** *The Manager's Path* (Camille Fournier), *An Elegant Puzzle* (Will Larson) — khung cho hiring/tech-debt/org.
- ⚠️ Chất liệu trả lời sâu nhất là **kinh nghiệm thật của bạn** — chuẩn bị STAR cho các câu phỏng vấn ứng viên/onboarding/delegation. AI lo *khung*, bạn lo *câu chuyện & phán đoán*.

---

## Mục lục mục con (ID prefix → số câu)

| Tag | Mục con | Số câu |
|---|---|---|
| **C-INT** | Phỏng vấn ứng viên backend (structured interview, rubric, signal, calibration, debrief) | 16 |
| **C-FUN** | Hiring funnel: JD · sourcing · screen · leveling · closing | 8 |
| **C-ONB** | Onboarding (30/60/90, buddy, ramp metric, senior vs junior) | 10 |
| **C-REV** | Code review — chuẩn & tooling cấp team (blocking/nit, SLA, gatekeeping) | 11 |
| **C-DEBT** | Tech debt management (phân loại, thuyết phục business, ngân sách, phòng ngừa) | 13 |
| **C-DEL** | Phân chia công việc & delegation (mức độ, bus factor, RACI, on-call) | 11 |
| **C-TEAM** | Team building, growth & health (ladder, 1:1, psych safety, retention, TL vs EM) | 13 |

---

## C-INT — Phỏng vấn ứng viên backend

**C-INT-001** ⭐⭐
"'Structured interview' khác 'unstructured' ở đâu? Vì sao structured được khuyến nghị?"
> *Dò cái gì:* structured = cùng bộ câu hỏi + rubric cho mọi ứng viên → predictive validity cao hơn rõ rệt (≈ gần gấp đôi) & giảm bias; unstructured = chit-chat theo cảm tính, dễ halo/affinity.

**C-INT-002** ⭐⭐⭐
"Thiết kế một hiring loop cho backend mid/senior gồm những vòng gì? Mỗi vòng dò cái gì?"
> *Dò cái gì:* các vòng *bù nhau* — coding/pairing hoặc take-home (code thật), system design có ràng buộc thật, behavioral/values, (tùy) deep-dive dự án; mỗi vòng một competency, không trùng tín hiệu.

**C-INT-003** ⭐⭐⭐
"Rubric/scorecard phỏng vấn là gì? Vì sao TL cần nó thay vì 'cảm nhận chung'?"
> *Dò cái gì:* bộ tiêu chí gắn job (problem-solving, coding, design, collaboration) + thang điểm + behavioral anchor; biến "vibe" thành evidence → giảm halo/groupthink, debrief nhanh & defensible.

**C-INT-004** ⭐⭐⭐⭐
"Vì sao mỗi interviewer phải nộp scorecard ĐỘC LẬP TRƯỚC buổi debrief?"
> *Dò cái gì:* chặn anchoring/halo/post-hoc rationalization — không bị người nói trước kéo theo; chỉ khi nộp độc lập mới so được điểm lệch & lộ bias.

**C-INT-005** ⭐⭐⭐
"'Signal vs noise' trong một buổi phỏng vấn nghĩa là gì? Làm sao tăng signal?"
> *Dò cái gì:* phân biệt năng lực thật vs nhiễu (stress, quen dạng bài, may rủi); tăng signal = câu hỏi gắn công việc thật, cho dùng tài liệu/IDE, follow-up "vì sao", tránh đố mẹo.

**C-INT-006** ⭐⭐⭐⭐
"Take-home vs live coding vs whiteboard: trade-off và chọn cái nào theo level?"
> *Dò cái gì:* whiteboard đo stress hơn skill; take-home công bằng hơn nhưng tốn thời gian ứng viên (giữ phạm vi nhỏ, vài giờ, có feedback); live pairing đo collab realtime; chọn theo role/level.

**C-INT-007** ⭐⭐⭐
"Phỏng vấn phần DB/concurrency của một backend candidate, anh dò gì để không thành hỏi trivia?"
> *Dò cái gì:* dò *hiểu trade-off* qua tình huống thật (index, N+1, transaction/isolation) + "vì sao", không phải định nghĩa thuộc lòng; cross-ref E/I.

**C-INT-008** ⭐⭐⭐⭐
"Với tư cách NGƯỜI HỎI system design, anh chấm ứng viên theo gì?"
> *Dò cái gì:* chấm quy trình (clarify→estimate→high-level→trade-off) & khả năng nêu đánh đổi + đặt câu hỏi làm rõ, không phải "vẽ trúng kiến trúc mẫu"; có rubric mức độ.

**C-INT-009** ⭐⭐⭐
"Vòng behavioral cho backend dò 'collaboration/ownership' bằng tín hiệu gì?"
> *Dò cái gì:* dùng STAR, hỏi tình huống thật (bất đồng kỹ thuật, bug nặng); dò fact-based disagreement, openness to feedback, ownership; tránh câu giả định mơ hồ.

**C-INT-010** ⭐⭐⭐⭐
"Calibration session là gì? Vì sao phải làm định kỳ?"
> *Dò cái gì:* nhiều interviewer chấm cùng ứng viên/bài → so điểm lệch → thống nhất "good là gì"; chống *scorecard drift* khi team/role đổi → nên định kỳ (vd hằng quý/tháng).

**C-INT-011** ⭐⭐⭐⭐
"Những bias nào hay xuất hiện khi phỏng vấn, và cơ chế giảm chúng?"
> *Dò cái gì:* affinity (cùng trường/giọng), halo, confirmation, recency; giảm = structured + rubric anchored + scorecard độc lập + panel đa dạng + audit drift theo *data*, không chỉ "đào tạo nhận thức".

**C-INT-012** ⭐⭐⭐
"'Bar raiser' (người gác chuẩn tuyển dụng) là gì? Giải quyết vấn đề gì?"
> *Dò cái gì:* một người trung lập ngoài team chịu trách nhiệm giữ "bar" nhất quán toàn công ty → chống hạ chuẩn vì áp lực headcount hay thiên vị cục bộ team.

**C-INT-013** ⭐⭐⭐⭐⭐
"Debrief ra quyết định thế nào? Ai quyết, và xử lý ra sao khi panel chia rẽ?"
> *Dò cái gì:* mọi người vote hire/no-hire dựa scorecard, HM ra quyết định cuối; chia rẽ → quay về *evidence theo rubric*, không "ai thích ứng viên hơn"; tỉ lệ ra quyết định thấp = loop/rubric hỏng.

**C-INT-014** ⭐⭐⭐
"Candidate experience quan trọng thế nào, và TL ảnh hưởng tới nó ra sao?"
> *Dò cái gì:* phản hồi đúng hẹn & tôn trọng, quy trình rõ ràng/công bằng = employer brand kể cả với người trượt; phỏng vấn 2 chiều — ứng viên cũng đang đánh giá mình.

**C-INT-015** ⭐⭐⭐⭐
"2026: ứng viên dùng AI (Copilot/LLM) trong vòng coding — anh điều chỉnh format thế nào?"
> *Dò cái gì:* cấm cứng khó kiểm & xa thực tế → chuyển sang dò khả năng *chỉ huy & kiểm* AI (đọc/sửa code AI, bắt lỗi, giải thích trade-off) hoặc thêm vòng không-AI cho fundamentals; *verify chính sách công ty*; cross-ref R.

**C-INT-016** ⭐⭐⭐⭐
"Red flag / 'no-hire signal' nào anh coi là dứt khoát ở một backend candidate?"
> *Dò cái gì:* không nhận feedback, đổ lỗi, không giải thích "vì sao", coi nhẹ test/security, không clarify khi đề mơ hồ; biết phân biệt *thiếu kinh nghiệm* (đào tạo được) vs *thái độ* (khó sửa).

---

## C-FUN — Hiring funnel: JD · sourcing · screen · leveling · closing

**C-FUN-001** ⭐⭐
"Một job description backend tốt gồm gì? Lỗi thường gặp?"
> *Dò cái gì:* tách must-have vs nice-to-have *thực tế*, mô tả công việc & impact thay vì liệt kê 20 công nghệ; JD kiểu "wishlist" lọc mất ứng viên tốt và khuếch đại bias.

**C-FUN-002** ⭐⭐⭐
"Sourcing một backend senior: các kênh và đánh đổi?"
> *Dò cái gì:* referral (chất lượng cao nhưng rủi ro đồng nhất/đa dạng kém), inbound, outbound LinkedIn, job board, nội bộ/promote; cân tốc độ vs đa dạng vs chi phí.

**C-FUN-003** ⭐⭐⭐
"Phone/recruiter screen nên lọc gì để không phí các vòng sau?"
> *Dò cái gì:* lọc *must-have cốt lõi* + động cơ + mức lương/level kỳ vọng *sớm* để tránh mismatch ở cuối phễu; nguyên tắc "rẻ trước, đắt sau".

**C-FUN-004** ⭐⭐⭐⭐
"Leveling: làm sao quyết một ứng viên là mid hay senior, tránh mis-level?"
> *Dò cái gì:* chấm theo rubric năng lực (scope, autonomy, ảnh hưởng), không theo số năm/title cũ; mis-level → lương lệch, kỳ vọng sai, rời bỏ; calibrate cùng team.

**C-FUN-005** ⭐⭐⭐
"Là TL/HM, anh 'close' (chốt) một ứng viên giỏi đang có nhiều offer thế nào?"
> *Dò cái gì:* bán đúng cơ hội (impact, growth, tech, đội ngũ), nói thật cả điểm yếu, tốc độ & cá nhân hóa, kết nối với người sẽ làm cùng; không chỉ đua lương.

**C-FUN-006** ⭐⭐⭐⭐
"Hire-for-potential vs hire-for-experience: khi nào nghiêng cái nào?"
> *Dò cái gì:* vai trò cấp bách/đặc thù → experience; team đang xây nền/ngân sách hạn chế/muốn đa dạng & growth → potential + kèm cặp; nêu rủi ro mỗi hướng.

**C-FUN-007** ⭐⭐⭐
"'Hiring for complementary skills' khi build một team từ nhỏ nghĩa là gì?"
> *Dò cái gì:* không clone bản thân; phối thế mạnh bù nhau (generalist, infra-leaning, product-leaning) → tránh team toàn một kiểu, phủ được nhiều rủi ro.

**C-FUN-008** ⭐⭐⭐⭐
"Cost of a bad hire vs cost of a slow hire — anh cân thế nào?"
> *Dò cái gì:* bad hire rất đắt (re-hire, morale, sửa hậu quả, có thể sa thải) nhưng tuyển kéo dài làm kiệt team & lỡ cơ hội; giữ bar nhưng tối ưu phễu/tốc độ, không hạ chuẩn vì sốt ruột.

---

## C-ONB — Onboarding

**C-ONB-001** ⭐⭐
"Khung onboarding 30/60/90 ngày đặt mục tiêu gì ở mỗi mốc?"
> *Dò cái gì:* 30 = học ngữ cảnh/setup/PR nhỏ đầu tiên; 60 = đóng góp độc lập một feature vừa; 90 = ownership một mảng + hiểu domain; mốc rõ để *đo* ramp.

**C-ONB-002** ⭐⭐⭐
"Vì sao nên đặt mục tiêu 'first PR merged trong tuần đầu'?"
> *Dò cái gì:* kiểm tra toàn bộ dev loop sớm (env, build, test, review, deploy); tạo win sớm & niềm tin; lộ ra chỗ tài liệu/quy trình hỏng.

**C-ONB-003** ⭐⭐⭐
"Buddy system trong onboarding: vai trò gì, khác mentor dài hạn ở đâu?"
> *Dò cái gì:* buddy = người đi cùng tuần đầu trả lời câu "ngớ ngẩn", hạ rào cản hỏi, tách khỏi đánh giá; khác mentor career dài hạn (định hướng phát triển).

**C-ONB-004** ⭐⭐⭐
"Onboard một SENIOR khác onboard một JUNIOR thế nào?"
> *Dò cái gì:* senior cần ngữ cảnh domain/quyết định + quyền tự chủ & tránh micromanage; junior cần nền kỹ năng + task nhỏ tăng dần + review dày; không one-size-fits-all.

**C-ONB-005** ⭐⭐⭐
"Đo 'ramp-up' thành công bằng gì, không chỉ cảm tính?"
> *Dò cái gì:* time-to-first-PR, time-to-first-independent-feature, time-to-on-call-ready + feedback định tính; dùng để *cải tiến onboarding*, không phải để chấm điểm gây áp lực.

**C-ONB-006** ⭐⭐⭐⭐
"Onboarding kém gây hậu quả gì ở cấp team/org?"
> *Dò cái gì:* ramp chậm = tốn thời gian người kèm, early attrition (nghỉ sớm), giảm morale; onboarding chậm khi scale còn là trigger cần platform/IDP (trỏ N/R).

**C-ONB-007** ⭐⭐⭐
"Onboarding checklist/tài liệu nên gồm gì để không phụ thuộc một người?"
> *Dò cái gì:* setup env, kiến trúc tổng quan (C4/ADR), quy ước code/review/branch, runbook on-call, ai-làm-gì; giảm bus factor & câu hỏi lặp.

**C-ONB-008** ⭐⭐⭐⭐
"Remote/hybrid onboarding khó hơn ở đâu, và bù thế nào?"
> *Dò cái gì:* thiếu kênh hỏi tự nhiên/quan sát; bù bằng buddy *chủ động*, async docs tốt, 1:1 dày tuần đầu, pairing có chủ đích, checklist rõ.

**C-ONB-009** ⭐⭐⭐
"Người mới ramp chậm hơn kỳ vọng sau 60 ngày — anh xử lý thế nào?"
> *Dò cái gì:* tách nguyên nhân (onboarding kém vs skill gap vs fit) bằng feedback cụ thể & quan sát; vá đúng chỗ (mentor/cặp/điều chỉnh task), đặt mốc rõ; phân biệt với performance management.

**C-ONB-010** ⭐⭐⭐
"Người mới đề xuất đổi convention/tool ngay tuần đầu — anh phản hồi sao?"
> *Dò cái gì:* trân trọng 'fresh eyes' (họ thấy chỗ team đã mù quen) nhưng cân với lý do/context lịch sử; khuyến khích quan sát + đề xuất có lập luận, không bác bỏ ngay.

---

## C-REV — Code review: chuẩn & tooling cấp team

**C-REV-001** ⭐⭐
"Mục đích thật của code review là gì, ngoài 'bắt bug'?"
> *Dò cái gì:* cải thiện *code health* tổng thể, chia sẻ kiến thức/giảm bus factor, giữ nhất quán, dạy-học; KHÔNG phải cổng để chứng tỏ ai giỏi hơn.

**C-REV-002** ⭐⭐⭐
"Theo chuẩn 'code health' (Google eng-practices), khi nào reviewer nên approve?"
> *Dò cái gì:* approve khi thay đổi *chắc chắn cải thiện code health tổng thể*, kể cả chưa hoàn hảo; chỉ chặn cái *làm xấu đi*; không chặn để chạy theo "perfect".

**C-REV-003** ⭐⭐⭐
"Phân biệt comment 'blocking' vs 'nit'. Vì sao đánh dấu rõ lại quan trọng?"
> *Dò cái gì:* blocking = correctness/security/maintainability lớn/vi phạm policy; nit = style/đặt tên/polish — optional; prefix "nit:"/Conventional Comments để author biết cái nào *buộc* sửa, cái nào có thể bỏ qua.

**C-REV-004** ⭐⭐⭐⭐
"Quá nhiều nitpick gây hại gì? Là TL anh xử lý *gốc* thế nào?"
> *Dò cái gì:* nit nhiều = mất velocity, che vấn đề lớn, hại morale/đẩy tới gatekeeping; gốc = thiếu tooling/chuẩn → tự động hóa style bằng formatter/linter để nit không tới tay người.

**C-REV-005** ⭐⭐⭐
"Conventional Comments là gì, giải quyết vấn đề gì trong review?"
> *Dò cái gì:* chuẩn gắn *nhãn intent* vào comment (praise/nit/suggestion/issue/question + blocking/non-blocking) → author hiểu loại & mức độ feedback, giảm hiểu lầm/ego; *verify spec*.

**C-REV-006** ⭐⭐⭐
"Đặt SLA cho review thế nào, và vì sao cần?"
> *Dò cái gì:* cửa sổ phản hồi đầu (vd <24h, chấp nhận <48h) để PR không kẹt; backlog review phình = tín hiệu nghẽn; cân tải reviewer (roulette/CODEOWNERS).

**C-REV-007** ⭐⭐⭐⭐
"PR quá lớn hại review thế nào? Ngưỡng và cách kéo nhỏ?"
> *Dò cái gì:* PR lớn (>~400 dòng, nhất là >1000) → review hời hợt, miss bug, kẹt lâu; tách logic khỏi refactor, vertical slice, feature flag; theo dõi phân bố PR size.

**C-REV-008** ⭐⭐⭐⭐⭐
"'Gatekeeping' trong code review là gì? Triệu chứng và cách sửa?"
> *Dò cái gì:* dùng review như *quyền lực* (đòi mọi PR qua mình, chặn vì sở thích cá nhân ngoài convention, ép rewrite để khoe); sửa = chuẩn hóa rule + automate + chuyển sang teach + CODEOWNERS phân quyền.

**C-REV-009** ⭐⭐⭐⭐
"Reviewer và author bất đồng kéo dài trong một PR — quy trình giải quyết?"
> *Dò cái gì:* quay về principle/tests/standard đã viết; nếu vẫn lệch → call ngắn rồi *summarize quyết định vào PR*; tránh war comment; có người tie-break (CODEOWNER/TL).

**C-REV-010** ⭐⭐⭐
"Tách vai AUTOMATION vs HUMAN trong review (2026) thế nào?"
> *Dò cái gì:* linter/formatter/SAST/AI tool lo style/nit/bug rõ/security obvious; người tập trung kiến trúc, trade-off, logic, maintainability → giữ review high-signal; *verify tool*; governance code-AI sâu → trỏ R.

**C-REV-011** ⭐⭐⭐⭐
"Là TL, anh đo 'sức khỏe' của review process bằng metric gì? Cẩn trọng gì?"
> *Dò cái gì:* review latency/time-to-merge, PR size distribution, churn/rework, backlog; cảnh báo: tối ưu metric mù → review hời hợt; metric để *phát hiện nghẽn*, không để xếp hạng người.

---

## C-DEBT — Tech debt management

**C-DEBT-001** ⭐⭐
"Định nghĩa technical debt. Khác bug ở đâu?"
> *Dò cái gì:* chi phí ẩn của rework do chọn cách nhanh/dễ thay vì đúng; *không nhất thiết* sai chức năng (khác bug) nhưng làm *chậm & đắt* thay đổi tương lai.

**C-DEBT-002** ⭐⭐⭐
"Fowler 'technical debt quadrant' (deliberate/inadvertent × prudent/reckless) dùng để làm gì?"
> *Dò cái gì:* phân loại *nguồn gốc* nợ → 'prudent-deliberate' (cố ý ship nhanh có ý thức, trả sau) chấp nhận được; 'reckless' do thiếu kỷ luật/kiến thức cần ngăn; *verify khái niệm*.

**C-DEBT-003** ⭐⭐⭐
"Phân loại & ưu tiên nợ kỹ thuật thế nào (bucket)?"
> *Dò cái gì:* chia theo impact — must-fix-now (chặn tiến độ/security) → fix-this-quarter (chậm dev, chưa nguy) → observe/monitor; ưu tiên theo *business impact*, không coi mọi nợ bằng nhau.

**C-DEBT-004** ⭐⭐⭐⭐
"Thuyết phục một stakeholder phi kỹ thuật chi thời gian trả nợ — anh làm thế nào?"
> *Dò cái gì:* *dịch rủi ro kỹ thuật sang ngôn ngữ business* — nợ làm *chậm ship feature*, *tăng chi phí bảo trì*, *tăng bug/incident*; dùng số/ví dụ; KHÔNG nói "code xấu".

**C-DEBT-005** ⭐⭐⭐⭐
"Các chiến lược cấp 'ngân sách' để trả nợ (20% / boy scout / debt sprint)?"
> *Dò cái gì:* dành % cố định mỗi sprint (vd ~20%), 'boy scout rule' (để code sạch hơn lúc thấy), hoặc tech-debt sprint định kỳ; là *framework*, không ad-hoc khi đã vỡ.

**C-DEBT-006** ⭐⭐⭐⭐
"Đo/visualize nợ bằng gì để quyết định data-informed?"
> *Dò cái gì:* code complexity, bug/incident rate theo module, lead time/churn, hotspot analysis (CodeScene-style); track trong backlog; *verify tool*.

**C-DEBT-007** ⭐⭐⭐⭐
"Khi nào CỐ Ý nhận thêm nợ là quyết định đúng?"
> *Dò cái gì:* ship nhanh để validate market/deadline thật, vùng code sắp bỏ, MVP/prototype; điều kiện: *cố ý + ghi nhận (log) + có kế hoạch trả*; phân biệt prudent vs reckless.

**C-DEBT-008** ⭐⭐⭐⭐⭐
"Khi nào KHÔNG nên trả nợ?"
> *Dò cái gì:* code ổn định ít đổi / sắp deprecate / ROI thấp; 'refactor for the sake of it'; chi phí trả > lợi ích; nợ nằm *ngoài* đường thay đổi sắp tới.

**C-DEBT-009** ⭐⭐⭐⭐
"Big rewrite vs incremental refactor (strangler) — chọn thế nào?"
> *Dò cái gì:* rewrite rủi ro cao (mất kiến thức ẩn, dừng feature, hay trễ); ưu tiên incremental/strangler-fig bọc dần; rewrite chỉ khi nền tảng thật sự chặn & có ngân sách/cam kết.

**C-DEBT-010** ⭐⭐⭐
"Track nợ ở đâu để nó không 'trôi' mãi không ai đụng?"
> *Dò cái gì:* backlog/issue có nhãn + gắn module/impact, review định kỳ; ADR ghi quyết định *nhận* nợ; gắn vào planning; tránh "log rồi quên".

**C-DEBT-011** ⭐⭐⭐⭐
"Ngăn nợ MỚI sinh ra (prevention), không chỉ trả nợ cũ — bằng gì?"
> *Dò cái gì:* DoD/quality gate (test, review, lint), ADR/design review nhẹ, architectural guardrail tự động; không để 'nhanh ẩu' thành mặc định; ngăn rẻ hơn trả; cross-ref L/Q.

**C-DEBT-012** ⭐⭐⭐⭐⭐
"Anti-pattern nào khi nói về tech debt khiến interviewer loại một TL ngay?"
> *Dò cái gì:* chỉ đụng nợ khi *vỡ* (reactive), coi mọi nợ critical như nhau (không nuance), *đổ lỗi product team* (thiếu ownership/collab); ngược lại = proactive + framework + business framing.

**C-DEBT-013** ⭐⭐⭐⭐
"Team đòi 'dừng feature một quý để refactor toàn bộ' — anh phản hồi sao?"
> *Dò cái gì:* cảnh giác big-bang; đòi business case + phạm vi hẹp + incremental + đo impact; cân với cam kết sản phẩm; tránh cả 'không bao giờ trả' lẫn 'dừng hết để trả'.

---

## C-DEL — Phân chia công việc & delegation

**C-DEL-001** ⭐⭐
"Vì sao TL phải delegate thay vì ôm hết việc khó?"
> *Dò cái gì:* ôm việc = bottleneck + bus factor + team không lớn lên; delegate tạo *leverage*, phát triển người, scale bản thân; vai TL là *nhân* năng lực team, không phải IC giỏi nhất.

**C-DEL-002** ⭐⭐⭐
"Phân biệt delegate vs abdicate vs micromanage."
> *Dò cái gì:* micromanage = giao rồi soi từng bước (giết autonomy); abdicate = quăng việc bỏ mặc (không support/đo); delegate đúng = giao *outcome + context + mức tự chủ + checkpoint*, vẫn chịu trách nhiệm cuối.

**C-DEL-003** ⭐⭐⭐⭐
"'Mức độ delegation' nên điều chỉnh theo gì?"
> *Dò cái gì:* theo *skill × will* của người + độ rủi ro task; người mới/việc rủi ro cao → hướng dẫn sát; người giỏi/việc rõ → giao outcome & buông; không một mức cho mọi người.

**C-DEL-004** ⭐⭐⭐
"Giao task thế nào để vừa xong việc vừa *phát triển* người (stretch)?"
> *Dò cái gì:* stretch assignment hơi quá tầm + safety net (pairing/review/mentor); cân 'việc cần xong' với 'cơ hội học'; không chỉ giao việc dễ/nhàm.

**C-DEL-005** ⭐⭐⭐⭐
"Phân task theo level/strength trong một team backend ra sao?"
> *Dò cái gì:* ánh xạ độ phức tạp/scope/ambiguity với level (junior → task rõ ràng, senior → mơ hồ/ownership); tránh để một người ôm hết phần khó (bus factor) hoặc junior chìm; cân load.

**C-DEL-006** ⭐⭐⭐⭐⭐
"'Bus factor = 1' / hero culture: rủi ro gì và phá thế nào?"
> *Dò cái gì:* một người biết mọi thứ = rủi ro nghỉ/ốm/burnout & nghẽn; phá bằng pairing, rotation, doc, review chia kiến thức; không để mọi ticket khó về một người; *đo & chủ động phân tán*.

**C-DEL-007** ⭐⭐⭐
"RACI (hoặc single DRI) giúp gì khi chia việc, nhất là cross-team?"
> *Dò cái gì:* làm rõ ai Responsible/Accountable/Consulted/Informed (hoặc một DRI) → hết 'tưởng người kia làm', giảm trùng/lọt việc ở task liên team.

**C-DEL-008** ⭐⭐⭐⭐
"Đã delegate nhưng người làm sắp đi sai hướng — can thiệp lúc nào, thế nào?"
> *Dò cái gì:* đặt checkpoint/milestone thay vì soi liên tục; can thiệp khi chạm *ngưỡng rủi ro đã thỏa thuận*; coaching (hỏi-hướng-dẫn) trước khi giành tay lái; để họ học từ sai lầm nhỏ.

**C-DEL-009** ⭐⭐⭐⭐
"Phân việc on-call/vận hành thế nào cho công bằng & bền vững?"
> *Dò cái gì:* rotation rõ, không dồn 1–2 người; handoff/follow-the-sun; gắn với *giảm toil* & blameless; chống burnout; phân biệt việc on-call vs feature trong capacity.

**C-DEL-010** ⭐⭐⭐
"Khi 'cái gì cũng urgent', anh ưu tiên & xếp lịch task ra sao?"
> *Dò cái gì:* hỏi context/business objective/effort-cost trước khi xếp; framework ưu tiên (impact vs effort, hạn *thật* vs *giả*); đẩy ngược câu hỏi để làm rõ, không nhận hết.

**C-DEL-011** ⭐⭐⭐⭐
"Là TL, anh cân 'tự code bao nhiêu' vs 'tạo leverage qua người' thế nào?"
> *Dò cái gì:* TL vẫn code (tooling/automation/refactor/vùng rủi ro) nhưng *không nằm trên critical path* mọi feature; cân theo nhu cầu team & giai đoạn; quá hands-on → bottleneck, quá hands-off → mất context/credibility.

---

## C-TEAM — Team building, growth & health

**C-TEAM-001** ⭐⭐⭐
"Career ladder / growth framework dùng để làm gì?"
> *Dò cái gì:* định nghĩa kỳ vọng theo level (scope/autonomy/impact) → công bằng, rõ đường thăng tiến, làm chuẩn cho feedback/promotion/leveling; thiếu nó = thiên vị & mơ hồ.

**C-TEAM-002** ⭐⭐⭐
"1:1 (one-on-one) để làm gì? Khác status update thế nào?"
> *Dò cái gì:* không phải báo cáo tiến độ; là kênh build trust, gỡ blocker, career/growth, feedback 2 chiều, bắt sớm vấn đề/tín hiệu nghỉ; *người làm sở hữu agenda*.

**C-TEAM-003** ⭐⭐⭐⭐
"Dựng growth plan cho một engineer muốn lên senior thế nào?"
> *Dò cái gì:* gắn ladder → xác định gap cụ thể (scope/ambiguity/leadership) → giao stretch + mentor + feedback đo được + mốc; phối hợp *với* họ, không hứa suông.

**C-TEAM-004** ⭐⭐⭐⭐⭐
"Phát hiện & xử lý underperformer thế nào cho sớm và công bằng?"
> *Dò cái gì:* tách nguyên nhân (skill/will/fit/ngữ cảnh/onboarding) bằng feedback cụ thể & evidence *sớm*; plan có mốc, documented; phân biệt với PIP chính thức (loop HR); né tránh kéo dài hại cả team.

**C-TEAM-005** ⭐⭐⭐⭐
"'Psychological safety' là gì? Vì sao nó quyết định chất lượng team?"
> *Dò cái gì:* an toàn để nói 'tôi không biết' / báo lỗi / phản biện sếp mà không sợ trừng phạt → blameless postmortem, dám raise risk sớm; thiếu nó = giấu lỗi, im lặng nguy hiểm.

**C-TEAM-006** ⭐⭐⭐
"Đo 'team health' bằng gì, không chỉ velocity?"
> *Dò cái gì:* kết hợp delivery (lead time, deploy freq — DORA) + con người (eNPS, retention, on-call load, burnout signal); velocity đơn lẻ dễ bị *game*; cross-ref O/N.

**C-TEAM-007** ⭐⭐⭐⭐
"Giữ chân (retention) một engineer giỏi bằng gì, ngoài lương?"
> *Dò cái gì:* growth/impact/autonomy/mastery, tech tốt, manager tốt (1:1, ghi nhận), tránh burnout & việc nhàm; bắt tín hiệu sớm ở 1:1; lương là *điều kiện cần không đủ*.

**C-TEAM-008** ⭐⭐⭐
"Nguyên tắc cho/nhận feedback hiệu quả trong team?"
> *Dò cái gì:* cụ thể-kịp thời-*về hành vi* (không nhân cách), riêng-tư khi tiêu cực, gắn impact (SBI-ish); văn hóa feedback 2 chiều thường xuyên thay vì dồn tới review cuối năm.

**C-TEAM-009** ⭐⭐⭐⭐
"Quy mô/topology team ảnh hưởng gì? Khi nào nên tách team?"
> *Dò cái gì:* team quá lớn → communication overhead (n²), coordination chậm; 'two-pizza' / Team Topologies (stream-aligned, ownership rõ); tách khi cognitive load/đường biên service quá tải; cross-ref A.

**C-TEAM-010** ⭐⭐⭐⭐
"Một engineer hợp team khác hơn — anh xử lý chuyện internal transfer sao?"
> *Dò cái gì:* ưu tiên lợi ích *người & org* dài hạn hơn giữ cục bộ; bàn với manager/đối tác liên quan trước; minh bạch, không 'giam' người; giữ quan hệ & có kế hoạch backfill.

**C-TEAM-011** ⭐⭐⭐
"Xây 'team standards' (convention, DoD, ways of working) mà không áp đặt top-down?"
> *Dò cái gì:* đồng thuận qua thảo luận/RFC/ADR, *viết ra & tự động hóa* (lint/CI), review định kỳ; ownership tập thể tăng tuân thủ; top-down cứng gây phản kháng.

**C-TEAM-012** ⭐⭐⭐⭐
"Justify nhu cầu tuyển thêm (headcount plan) thế nào cho thuyết phục?"
> *Dò cái gì:* gắn với load/roadmap/khả năng giao hàng & rủi ro bus factor, không 'team đông cho oách'; dữ liệu (backlog, on-call, lead time) + cân nhắc tự động hóa/build-vs-buy trước.

**C-TEAM-013** ⭐⭐⭐⭐⭐
"Phân biệt vai Tech Lead vs Engineering Manager — ranh giới trách nhiệm con người?"
> *Dò cái gì:* TL thiên technical direction/architecture/chất lượng (có thể không quản nhân sự chính thức); EM sở hữu performance/career/comp/headcount; nhiều nơi *chồng lấn* (recruitment, perf eval, mentoring có thể thuộc TL) → quan trọng là *nói rõ ai làm gì*.

---

> **Hết bộ đề C (82 câu).** Bước tiếp theo của WORKFLOW 2:
> **Bước B** — dựng giáo trình ngược (gom câu → cụm khái niệm → bài học, sắp cơ bản → nâng cao).
> **Bước D** — phỏng vấn theo đợt 10–15 câu, chấm **live** (đáp án dựng ngay lúc đó, bám dòng *"dò cái gì"*).
> Cứ bảo *"Bước B mục C"* hoặc *"phỏng vấn tôi mục C"* để tiếp.
