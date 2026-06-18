# 📘 GIÁO TRÌNH HỌC — Mục C: Hiring & Team Building
### (Tech Lead Backend) — WORKFLOW 2: Bước B (dựng giáo trình ngược) + Bước C (giảng đủ 7 bài)

> Tài liệu này được sinh từ ngân hàng câu hỏi **`QB_C_hiring.md`** (82 câu, 7 mục con).
> Mỗi bài tuân **Hợp đồng đầu ra 10 mục**. Vì đây là domain *leadership / quy trình*, mục ⑤ "code" được thay bằng **template / rubric / checklist / worksheet** dùng được ngay — đó là "artifact tay nghề" tương đương cho mục C.
> Ký hiệu: **📘** = có ngụ ý trong roadmap/khung gốc · **➕** = bổ sung của giảng viên · **⚠️** = bẫy / điểm cũ cần thay · **🔗** = cross-link sang mục khác.
> Lưu ý đặc thù: mục C **không có "đáp án chuẩn" cố định**. Mốc ĐẠT (dòng *"dò cái gì"* trong QB) = bạn chạm đúng **tín hiệu tư duy của một TL thật** — có *framework* + *business framing* + *câu chuyện thật của chính bạn*. AI lo *khung*, bạn lo *phán đoán & ví dụ*.
>
> **Bối cảnh verify (tính 6/2026):** các con số ở đây là *rule-of-thumb*, không phải hằng số. Những chỗ đã search + cite được ghi nguồn ngay tại chỗ (Sackett 2022, DORA 2024, Fabric 2026, conventionalcomments.org…). Khi đi phỏng vấn thật, dùng số để *lập luận*, đừng đọc như chân lý.

---

# PHẦN I — BƯỚC B: GIÁO TRÌNH NGƯỢC TỪ BỘ ĐỀ

## Nguyên tắc gom cụm
Bộ đề có 7 mục con. Tôi gom thành **7 bài học** theo **dependency order = vòng đời một đội ngũ** (KHÔNG theo tần suất hỏi). Lý do thứ tự — mỗi bài là *điều kiện* để bài sau có nghĩa:

- Trước khi đánh giá ai, phải biết **đang tuyển vai gì, level nào, đóng thế nào** → **Bài 1 (Funnel)** là cái khung bao quanh tất cả.
- Có khung rồi mới nói được **cách đánh giá** cho chuẩn — đây là kỹ năng lõi & nặng nhất của TL khi hiring → **Bài 2 (Interviewing)**.
- Đánh giá xong, người vào → phải **đưa họ lên tốc độ** → **Bài 3 (Onboarding)**.
- Người đã vào việc → cần **chuẩn chất lượng hằng ngày** ở mức code → **Bài 4 (Code review)**.
- Chất lượng từng PR mở rộng thành **sức khỏe codebase theo thời gian** → **Bài 5 (Tech debt)**.
- Codebase ổn rồi → **phân phối công việc & nhân năng lực qua người** → **Bài 6 (Delegation)**.
- Tổng hợp tất cả thành **nuôi dưỡng con người & đội ngũ dài hạn** → **Bài 7 (Team building)** — capstone, đỉnh là ranh giới TL vs EM.

Trục xuyên suốt: **code-level → codebase-level → work-level → people-level** (tăng dần scope & độ trừu tượng).

## Mục lục bài học → ánh xạ ID câu hỏi

| Bài | Tên bài | Tag | ID câu hỏi (số câu) |
|---|---|---|---|
| **1** | Hiring funnel & chiến lược tuyển | C-FUN | C-FUN-001 → 008 (8) |
| **2** | Structured interviewing & đánh giá ứng viên | C-INT | C-INT-001 → 016 (16) |
| **3** | Onboarding & ramp-up | C-ONB | C-ONB-001 → 010 (10) |
| **4** | Code review — chuẩn & tooling cấp team | C-REV | C-REV-001 → 011 (11) |
| **5** | Tech debt management | C-DEBT | C-DEBT-001 → 013 (13) |
| **6** | Delegation & phân chia công việc | C-DEL | C-DEL-001 → 011 (11) |
| **7** | Team building, growth & health | C-TEAM | C-TEAM-001 → 013 (13) |
|  |  | **Tổng** | **82 câu** |

## Bản đồ cross-link (nhớ kỹ — đề hay xoáy vào chỗ nối)
- **Psychological safety** là *hub*: Bài 7 (C-TEAM-005) ↔ blameless postmortem ↔ buddy hạ rào hỏi (Bài 3) ↔ chống gatekeeping review (Bài 4). DORA State of DevOps 2024 xếp nó là **một trong những predictor mạnh nhất** của software delivery performance.
- **Bus factor**: Bài 6 (C-DEL-006) ↔ doc onboarding (C-ONB-007) ↔ review chia kiến thức (C-REV-001) ↔ justify headcount (C-TEAM-012).
- **Rubric / calibration**: Bài 2 (C-INT) ↔ leveling (C-FUN-004) — cùng một công cụ "biến vibe thành evidence".
- **Metric đừng để game**: Bài 4 (C-REV-011) ↔ team health (C-TEAM-006) ↔ DORA khuyến cáo **không bao giờ gắn metric vào lương cá nhân**.
- **"Hiểu để chỉ huy & kiểm tra"** (nguyên tắc chữ ký của project): đậm nhất ở **AI trong phỏng vấn** (C-INT-015) và **AI trong review** (C-REV-010).

---
---

# PHẦN II — BƯỚC C: GIẢNG 7 BÀI (HỢP ĐỒNG 10 MỤC)

# 🎓 BÀI 1 — HIRING FUNNEL & CHIẾN LƯỢC TUYỂN
*(phủ C-FUN-001 → 008)*

## ① Mục tiêu & vị trí trong mạch
- **Vị trí:** Bài nền của mục C. Funnel là *cái phễu* mà mọi bài sau (phỏng vấn, onboarding) chạy bên trong. Sai ở đây thì phỏng vấn giỏi mấy cũng tuyển nhầm vai.
- **Học xong làm được:** viết JD lọc đúng người; chọn kênh sourcing theo trade-off; thiết kế thứ tự vòng "rẻ trước, đắt sau"; quyết level đúng; close ứng viên tốt; và cân **bad hire vs slow hire** như một bài toán rủi ro, không cảm tính.
- **Kết nối:** leveling ở đây (C-FUN-004) là *đầu vào* cho rubric phỏng vấn ở Bài 2; "complementary skills" (C-FUN-007) nối sang team topology ở Bài 7.

## ② Giảng cơ bản → nâng cao

**(a) Trực giác**
Hình dung tuyển dụng là một **cái phễu**: nhiều người vào miệng phễu (sourcing), lọc dần qua từng vòng, một người ra đáy (offer + close). Nguyên tắc xương sống: **vòng càng đắt (tốn thời gian người giỏi) thì càng để sau**; lọc cái rẻ và cái dứt khoát trước. JD là *tấm biển* ở miệng phễu — viết sai biển thì hút nhầm người ngay từ đầu.

**(b) Cơ chế chi tiết**
- 📘 **JD (C-FUN-001):** tách *must-have* vs *nice-to-have* **thực tế**, mô tả **công việc & impact** thay vì liệt kê 20 công nghệ. JD kiểu "wishlist" (đòi 5 năm Kafka + Rust + K8s + ML…) lọc mất ứng viên tốt và khuếch đại bias (nghiên cứu quen thuộc: phụ nữ có xu hướng không apply nếu không khớp ~100% tiêu chí, nam apply ở ~60%).
- 📘 **Sourcing (C-FUN-002):** referral (chất lượng cao, nhanh, *nhưng* rủi ro đồng nhất → giảm đa dạng), inbound (thương hiệu mạnh mới nhiều), outbound LinkedIn (chủ động, tốn công), job board (rộng, nhiễu), nội bộ/promote (giữ người, ít rủi ro). Cân **tốc độ × đa dạng × chi phí**.
- 📘 **Screen (C-FUN-003):** phone/recruiter screen lọc *must-have cốt lõi* + **động cơ** + **mức lương/level kỳ vọng** *sớm* — để không phí 4 vòng rồi vỡ ở offer vì lệch lương.
- 📘 **Leveling (C-FUN-004):** chấm theo **rubric năng lực** — *scope* (việc lớn cỡ nào), *autonomy* (cần dắt hay tự chạy), *ảnh hưởng* (chỉ mình hay cả team) — **không theo số năm/title cũ**.
- 📘 **Closing (C-FUN-005):** bán đúng cơ hội (impact, growth, tech, đội), nói thật cả điểm yếu, tốc độ & cá nhân hóa, cho ứng viên nói chuyện với người sẽ làm cùng.
- ➕ **Potential vs experience (C-FUN-006)** và **complementary skills (C-FUN-007):** xây team đừng *clone bản thân*; phối generalist / infra-leaning / product-leaning để phủ rủi ro.

**(c) Mép giới hạn & sai lầm**
- ⚠️ JD = wishlist → phễu khô, toàn người "đủ giấy tờ" mà không hợp việc. *(C-FUN-001)*
- ⚠️ Chỉ tuyển qua referral → team đồng nhất, mù điểm yếu chung, đa dạng kém. *(C-FUN-002)*
- ⚠️ Để chuyện lương/level đến cuối phễu → vỡ offer, phí cả loop. *(C-FUN-003)*
- ⚠️ Mis-level (đẩy mid lên senior để "đỡ phải thuyết phục") → lương lệch, kỳ vọng sai, người rời trong 6–12 tháng. *(C-FUN-004)*
- ⚠️ Close bằng cách *chỉ đua lương* → người đến vì tiền, đi cũng vì tiền. *(C-FUN-005)*

## ③ ⚠️ Kiến thức cũ / bị thay thế

| Cái cũ (phổ biến) | Thay bằng (nên dùng) | Vì sao |
|---|---|---|
| JD liệt kê công nghệ "yêu cầu 5 năm X, Y, Z" | JD mô tả **outcome & must-have thực tế** | Wishlist lọc nhầm + khuếch đại bias |
| Level theo **số năm kinh nghiệm / title cũ** | Level theo **rubric scope·autonomy·impact** | Năm/title không đo được năng lực; gây mis-level |
| Đánh giá lương/level ở vòng cuối | Hỏi **expectation ở phone screen** | "Rẻ trước, đắt sau" — tránh vỡ offer |
| Close = thắng cuộc đua lương | Close = **bán đúng cơ hội + tốc độ + nói thật** | Người đến vì tiền sẽ đi vì tiền |
| "Tuyển cho nhanh kẻo team kiệt sức" hoặc "giữ bar cực cao bất chấp" | Cân **cost of bad hire vs cost of slow hire** có ý thức | Cả hai cực đều đắt; tối ưu *phễu/tốc độ*, không hạ chuẩn |

## ④ Áp dụng thực tế + So sánh bigtech
- **Use case:** bạn được giao build một backend squad 5 người từ con số 0. Đừng tuyển 5 senior giống nhau. Pattern phổ biến: 1–2 senior "anchor" (ownership vùng khó), 2–3 mid (gánh delivery), 1 junior (growth + rẻ + fresh eyes), phủ cả infra-leaning lẫn product-leaning.
- **Bigtech (pattern, có chỗ verify):** **Amazon** nổi tiếng với "raise the bar" — luôn hỏi *"liệu người này có giỏi hơn 50% người đang ở level đó không?"*. **Cost of a bad hire** được nhiều nguồn ước tính rất lớn (re-hire + sửa hậu quả + morale + có thể phải sa thải) → bigtech chấp nhận **false negative** (lỡ người tốt) hơn **false positive** (nhận nhầm). Đây là *triết lý*, mức độ tùy công ty — *verify khi cần con số cụ thể*.

## ⑤ Artifact thực hành — **Khung JD + Leveling rubric**

```text
# JD BACKEND (template lọc đúng người)
Role: <Senior Backend Engineer — Payments squad>
Impact (1 câu): Bạn sẽ sở hữu <X>, giảm <metric Y>, cho phép <business outcome Z>.

MUST-HAVE (tối đa 4–5, đều là "nếu thiếu thì fail"):
- [ ] Vận hành dịch vụ backend production (scale/độ tin cậy thật)
- [ ] Thành thạo 1 ngôn ngữ backend + SQL/DB design
- [ ] Có kinh nghiệm xử lý concurrency / data consistency  (cross-ref I/E)

NICE-TO-HAVE (không loại nếu thiếu):
- Kafka / event-driven · Kubernetes · từng làm fintech

# LEVELING RUBRIC (quyết mid vs senior — KHÔNG dùng số năm)
| Trục       | Mid                          | Senior                                  |
|------------|------------------------------|-----------------------------------------|
| Scope      | 1 feature / 1 service rõ ràng| 1 mảng / nhiều service, đề mơ hồ         |
| Autonomy   | cần định hướng, review dày   | tự chia nhỏ vấn đề, tự ra trade-off      |
| Impact     | hoàn thành phần mình         | nâng cả team (chuẩn, mentor, định hướng) |
```
> ⚠️ **AI viết JD hộ bạn rất nhanh — và rất dễ "wishlist hóa"** (nó nhồi mọi công nghệ liên quan). Hãy *kiểm*: cắt must-have xuống ≤5 dòng "nếu thiếu là fail". Đây là cú pháp đúng, phán đoán sai.

## ⑥ Keywords cần nhớ
- 🧠 **Cần HIỂU:** funnel "rẻ trước đắt sau", leveling theo scope·autonomy·impact, complementary skills, bad-hire vs slow-hire trade-off, referral diversity risk.
- 📌 **Cần THUỘC:** 3 trục leveling (scope, autonomy, impact); must-have vs nice-to-have; thứ tự phễu (source → screen → loop → offer → close).
- 🛠️ **Cần LÀM ĐƯỢC:** viết JD ≤5 must-have; quyết level một hồ sơ bằng rubric & bảo vệ được; soạn pitch close cá nhân hóa.

## ⑦ Mental model
> **"Phễu lọc rẻ trước, đắt sau; tuyển *outcome & level*, không tuyển *từ khóa & số năm*."**

## ⑧ Câu hỏi phỏng vấn & thách đố
1. *(dễ)* JD backend tốt gồm gì, lỗi hay gặp? → must-have thực tế vs wishlist. *(C-FUN-001)*
2. *(TB)* Phone screen nên lọc gì để không phí vòng sau? → must-have cốt lõi + động cơ + lương/level sớm. *(C-FUN-003)*
3. *(khó)* Quyết một ứng viên là mid hay senior, tránh mis-level? → rubric scope·autonomy·impact, calibrate, *không* số năm. *(C-FUN-004)*
4. *(khó)* Hire-for-potential vs experience — khi nào nghiêng cái nào? → vai cấp bách/đặc thù → experience; xây nền/đa dạng/ngân sách hạn → potential + kèm cặp. *(C-FUN-006)*
- 🎯 **Thách đố:** *"Team đang thiếu người trầm trọng, có một ứng viên 'tạm ổn' — tuyển hay chờ?"* → bẫy: nhận đại = bad hire (đắt hơn nhiều slow hire); đáp đúng: giữ bar nhưng **tối ưu tốc độ phễu** (sourcing, lịch phỏng vấn) thay vì hạ chuẩn; nêu rõ cost cả hai phía. *(C-FUN-008)*

## ⑨ Bài tập + tiêu chí tự chấm
- **BT1:** Viết JD cho 1 role backend bạn từng làm, **≤5 must-have**. *Đạt khi* mỗi must-have là "thiếu → fail" và không có dòng nào là công nghệ-cho-vui.
- **BT2:** Lấy 1 hồ sơ (thật/giả), quyết mid hay senior **chỉ bằng 3 trục rubric**, viết 3 câu lý do. *Đạt khi* lý do bám scope/autonomy/impact, không nhắc "số năm".

## ⑩ Đọc thêm
- *The Manager's Path* — Camille Fournier (chương hiring). · *An Elegant Puzzle* — Will Larson (org/headcount). · Amazon "Bar Raiser" (tra trang tuyển dụng Amazon — *verify*). · Lara Hogan — "What does sponsorship look like" cho complementary growth.

---

# 🎓 BÀI 2 — STRUCTURED INTERVIEWING & ĐÁNH GIÁ ỨNG VIÊN
*(phủ C-INT-001 → 016)* — **bài lõi & nặng nhất của mục C**

## ① Mục tiêu & vị trí trong mạch
- **Vị trí:** trái tim của hiring. Bài 1 cho biết *tuyển vai/level gì*; bài này là *cách đánh giá* cho chuẩn, công bằng, defensible.
- **Học xong làm được:** thiết kế hiring loop các vòng bù nhau; viết & dùng rubric/scorecard; tăng signal-giảm noise; chạy calibration & debrief; nhận diện & giảm bias; và xử lý format phỏng vấn thời **AI 2026**.
- **Kết nối:** rubric ở đây dùng chung ngôn ngữ leveling (Bài 1); behavioral signal (C-INT-009) là cửa ngõ sang mục B (STAR).

## ② Giảng cơ bản → nâng cao

**(a) Trực giác**
Phỏng vấn là một **thí nghiệm đo lường** trong điều kiện nhiễu cao (stress, may rủi, quen dạng bài). Mục tiêu: **tối đa signal (năng lực thật), tối thiểu noise**. *Structured* = mọi ứng viên đi qua **cùng bộ câu hỏi + cùng rubric** → so sánh được như nhau. *Unstructured* = chit-chat theo cảm tính → bạn đo cảm xúc của chính mình nhiều hơn đo ứng viên.

**(b) Cơ chế chi tiết**
- 📘 **Structured vs unstructured (C-INT-001):** structured = chuẩn hóa câu hỏi + rubric + chấm độc lập → **predictive validity cao hơn rõ rệt** và **ít bias hơn**. *(Theo Sackett et al. 2022 — meta-analysis ~500+ nghề: structured interview là predictor có validity TRUNG BÌNH cao nhất, ~gấp đôi unstructured và ~ít bias hơn ~1/3.)*
- 📘 **Hiring loop (C-INT-002):** các vòng **bù nhau, không trùng tín hiệu** — coding/pairing hoặc take-home (code thật) · system design có ràng buộc thật · behavioral/values · (tùy) project deep-dive. Mỗi vòng dò *một* competency.
- 📘 **Rubric/scorecard (C-INT-003):** bộ tiêu chí gắn job (problem-solving, coding, design, collaboration) + thang điểm + **behavioral anchor** ("điểm 4 nghĩa là gì cụ thể"). Biến *vibe* thành *evidence*.
- 📘 **Scorecard độc lập TRƯỚC debrief (C-INT-004):** mỗi interviewer nộp điểm + lý do **trước khi nghe người khác** → chặn anchoring/halo/post-hoc rationalization.
- 📘 **Signal vs noise (C-INT-005):** tăng signal = câu hỏi gắn việc thật, cho dùng IDE/tài liệu, follow-up "vì sao", tránh đố mẹo.
- 📘 **Take-home vs live vs whiteboard (C-INT-006):** whiteboard đo *stress* hơn skill; take-home công bằng hơn nhưng tốn giờ ứng viên (giữ nhỏ, vài giờ, có feedback); live pairing đo collab realtime.
- ➕ **Calibration session (C-INT-010):** nhiều interviewer chấm cùng 1 bài/ứng viên → so điểm lệch → thống nhất "good là gì" → chống **scorecard drift**; làm định kỳ.
- ➕ **Bias (C-INT-011):** affinity (cùng trường/giọng), halo, confirmation, recency. Giảm = structured + rubric anchored + scorecard độc lập + panel đa dạng + **audit drift theo data**.
- ➕ **Bar raiser (C-INT-012):** người trung lập *ngoài team* giữ "bar" nhất quán toàn công ty → chống hạ chuẩn vì áp lực headcount.
- ➕ **Debrief & quyết định (C-INT-013):** mỗi người vote hire/no-hire **dựa scorecard**; HM (hiring manager) ra quyết định cuối; panel chia rẽ → quay về *evidence theo rubric*, không "ai thích ứng viên hơn".

**(c) Mép giới hạn & sai lầm**
- ⚠️ Đố mẹo / câu hỏi xa công việc → đo *quen dạng bài*, không đo năng lực. *(C-INT-005)*
- ⚠️ Whiteboard thuật toán cho role backend vận hành → đo nhầm trục. *(C-INT-006)*
- ⚠️ Debrief mà người nói trước kéo cả phòng → vì không nộp scorecard độc lập. *(C-INT-004)*
- ⚠️ Hỏi DB/concurrency thành **trivia định nghĩa** thay vì dò *trade-off qua tình huống* (index, N+1, isolation level). *(C-INT-007)* 🔗 E/I.
- ⚠️ Chấm system design theo "vẽ trúng kiến trúc mẫu" thay vì *quy trình* clarify→estimate→high-level→trade-off. *(C-INT-008)* 🔗 A.

## ③ ⚠️ Kiến thức cũ / bị thay thế

| Cái cũ | Thay bằng (hiện nay) | Vì sao |
|---|---|---|
| Unstructured "nói chuyện cho hợp gu" | **Structured + rubric** | Sackett 2022: structured có validity cao nhất, ít bias hơn ~1/3 |
| "Cognitive ability / leetcode là predictor số 1" | Structured interview ≥ cognitive ability | Sackett 2022 *course-correct* số cũ của Schmidt-Hunter 1998 (lỗi range-restriction) |
| Whiteboard thuật toán cho mọi role | Code thật / take-home + live defense / pairing | Whiteboard đo stress; thực tế dev khác |
| Debrief "ai cũng nói rồi chốt" | **Scorecard độc lập trước → debrief theo evidence** | Chặn anchoring/halo |
| Cấm tuyệt đối AI trong vòng coding (2026) | Dò khả năng **chỉ huy & kiểm AI** hoặc thêm vòng no-AI cho fundamentals | Cấm khó kiểm & xa thực tế công việc |

> ⚠️ **Verify:** con số Sackett (2022) là validity *trung bình* đã hiệu chỉnh; nó *thấp hơn tuyệt đối* so với số huyền thoại cũ (vì sửa lỗi overcorrection range-restriction) **nhưng thứ hạng đảo lại có lợi cho structured interview**. Đừng đọc thuộc một con số "r = 0.x"; nhớ *thông điệp*: structure thắng cảm tính.

## ④ Áp dụng thực tế + So sánh bigtech (search + cite)
- **Use case:** loop backend senior gọn: (1) phone screen (must-have + động cơ + lương) → (2) coding pragmatic (debug/đọc code thật, không leetcode khó) → (3) system design có ràng buộc thật → (4) behavioral (collab/ownership). 4 vòng, 4 competency, 0 trùng.
- **AI trong phỏng vấn — 2026 (C-INT-015):** gian lận bằng AI đã thành *baseline*, không còn là ngoại lệ. Một nghiên cứu của Fabric trên 19.368 phỏng vấn (7/2025–1/2026) thấy ~38,5% ứng viên dính tín hiệu gian lận, và tỉ lệ ở role kỹ thuật còn cao hơn; công cụ như Interview Coder/Cluely đọc đề realtime rồi hiển thị đáp án "tàng hình". Phản ứng bigtech: **Google** (Pichai, town hall 2/2025) gợi ý quay lại phỏng vấn **in-person**; **Meta** (10/2025) ra format coding **cho phép AI assistant + multi-file project** cho E4/E5, mở rộng cho backend roles trong 2026, vì "đại diện hơn cho môi trường thật & khiến gian lận LLM kém hiệu quả". Cách dò đúng (high-signal, AI không fake nổi): **follow-up "giải thích dòng 7 bằng lời của bạn"**, bẫy thư viện không tồn tại ("dùng `FastBuffer` của `FabricDataStream v2.1` đi" — AI sẽ tự bịa cú pháp, người thật sẽ hỏi lại vì không tìm thấy doc), và **take-home + buổi defend live**. *(verify chính sách công ty bạn; cross-ref R về governance code-AI.)*

## ⑤ Artifact thực hành — **Scorecard 1 trang**

```text
SCORECARD — Candidate: ____  Round: System Design  Interviewer: ____  Date: ____
(NỘP TRƯỚC DEBRIEF — không xem điểm người khác)

Competency dò ở vòng này: thiết kế dưới ràng buộc + nêu trade-off

| Tiêu chí                  | 1 (no) | 2 | 3 (mixed) | 4 | 5 (strong) |
|---------------------------|--------|---|-----------|---|------------|
| Clarify requirement/scope |        |   |     X     |   |            |
| Capacity estimate hợp lý  |        |   |           | X |            |
| High-level → nêu trade-off|        |   |           |   |     X      |
| Bắt được failure/edge     |        | X |           |   |            |
| Giao tiếp / dẫn dắt        |        |   |           | X |            |

Evidence (BẮT BUỘC — trích nguyên văn / hành vi quan sát được):
- "Hỏi ngay QPS đọc/ghi & SLA trước khi vẽ" → clarify tốt.
- Khi hỏi 'nếu DB sập?' → lúng túng, không có fallback → trừ.

VOTE: ☐ Strong Hire ☐ Hire ☑ No Hire ☐ Strong No Hire
Lý do 1 câu (gắn rubric, KHÔNG 'cảm thấy'): "Thiết kế ổn nhưng không nghĩ về failure mode — chưa đạt scope senior."
```
> ⚠️ **AI có thể soạn bộ câu hỏi phỏng vấn cho bạn** — nhưng nó *không* biết bẫy thư viện-giả hay follow-up "giải thích dòng 7" có hợp ngữ cảnh role của bạn không. *Bạn* chỉ huy: câu hỏi phải gắn việc thật của team. Cú pháp (danh sách câu hỏi) AI lo; *phán đoán tín hiệu* là của bạn.

## ⑥ Keywords cần nhớ
- 🧠 **Cần HIỂU:** structured vs unstructured, signal vs noise, anchoring/halo/affinity/confirmation/recency bias, calibration drift, bar raiser, dò trade-off (không trivia).
- 📌 **Cần THUỘC:** 4 thành phần rubric (problem-solving, coding, design, collaboration); quy tắc *scorecard độc lập trước debrief*; cấu trúc loop "vòng bù nhau".
- 🛠️ **Cần LÀM ĐƯỢC:** viết scorecard có behavioral anchor; chạy 1 debrief dựa evidence; dựng 1 câu hỏi backend dò *trade-off*; thiết kế vòng coding chịu được AI.

## ⑦ Mental model
> **"Đo signal trong nhiễu: cùng câu hỏi + rubric + scorecard độc lập → biến *vibe* thành *evidence*. Năm 2026: dò *chỉ huy & kiểm AI*, không dò *gõ được code*."**

## ⑧ Câu hỏi phỏng vấn & thách đố
1. *(dễ)* Structured khác unstructured ở đâu, vì sao được khuyến nghị? *(C-INT-001)*
2. *(TB)* Thiết kế hiring loop backend mid/senior — mỗi vòng dò gì? *(C-INT-002)*
3. *(khó)* Vì sao scorecard phải nộp **độc lập trước** debrief? *(C-INT-004)*
4. *(khó)* Là người hỏi system design, anh chấm theo gì (không phải "vẽ trúng")? *(C-INT-008)*
5. *(rất khó)* Panel chia rẽ — ai quyết, xử lý sao? *(C-INT-013)*
- 🎯 **Thách đố 1:** *"Ứng viên cùng trường, nói chuyện rất hợp — bạn thấy mình muốn pass. Đó là signal hay noise?"* → affinity bias; quay về rubric/evidence. *(C-INT-011)*
- 🎯 **Thách đố 2:** *"2026, ứng viên vòng coding ngồi nhà — làm sao biết họ không nhờ AI?"* → đừng đua surveillance; chuyển sang *integrity layer* (follow-up giải thích, bẫy lib-giả, defend live), hoặc cho phép AI + đo *chỉ huy/kiểm*. *(C-INT-015)*

## ⑨ Bài tập + tiêu chí tự chấm
- **BT1:** Soạn scorecard cho 1 vòng (coding/design/behavioral) với **5 tiêu chí + behavioral anchor**. *Đạt khi* có cột Evidence và lý do vote gắn rubric.
- **BT2:** Viết 1 câu hỏi DB/concurrency **dò trade-off** + 2 follow-up "vì sao". *Đạt khi* không thể trả lời bằng một định nghĩa thuộc lòng.

## ⑩ Đọc thêm
- Sackett, Zhang, Berry, Lievens (2022) *Revisiting meta-analytic estimates of validity* (JAP) — nền khoa học. · Google re:Work — Structured interviewing & hiring. · Fabric / CoderPad *State of Tech Hiring 2025–2026* (AI cheating — *verify số*). · Amazon Bar Raiser (trang tuyển Amazon).

---

# 🎓 BÀI 3 — ONBOARDING & RAMP-UP
*(phủ C-ONB-001 → 010)*

## ① Mục tiêu & vị trí trong mạch
- **Vị trí:** nối hiring với vận hành. Tuyển đúng người mà onboard tệ → early attrition, phí cả công tuyển.
- **Học xong làm được:** dựng khung 30/60/90; đặt mục tiêu "first PR sớm"; phân vai buddy vs mentor; onboard *khác nhau* cho senior vs junior, remote vs onsite; *đo* ramp bằng metric, không cảm tính.
- **Kết nối:** doc onboarding (C-ONB-007) là vũ khí chống bus factor (Bài 6); buddy hạ rào hỏi → mầm psychological safety (Bài 7).

## ② Giảng cơ bản → nâng cao

**(a) Trực giác**
Onboarding là **rút ngắn thời gian từ "ngồi vào bàn" đến "tạo giá trị độc lập"** (time-to-productivity). Người mới giống một service vừa deploy: phải *chạy được end-to-end* sớm (smoke test) rồi mới scale. "First PR merged tuần đầu" chính là smoke test đó.

**(b) Cơ chế chi tiết**
- 📘 **30/60/90 (C-ONB-001):** 30 = học ngữ cảnh/setup + PR nhỏ đầu tiên; 60 = giao độc lập 1 feature vừa; 90 = ownership 1 mảng + hiểu domain. Mốc rõ để *đo*.
- 📘 **First PR sớm (C-ONB-002):** ép cả dev loop chạy thật (env→build→test→review→deploy), tạo win sớm, và **lộ ra chỗ tài liệu/quy trình hỏng**.
- 📘 **Buddy vs mentor (C-ONB-003):** buddy = người đi cùng tuần đầu trả lời câu "ngớ ngẩn", **tách khỏi đánh giá** → hạ rào hỏi; mentor = định hướng career dài hạn.
- 📘 **Senior vs junior (C-ONB-004):** senior cần *ngữ cảnh domain/quyết định + tự chủ*, tránh micromanage; junior cần *nền kỹ năng + task nhỏ tăng dần + review dày*.
- ➕ **Đo ramp (C-ONB-005):** time-to-first-PR, time-to-first-independent-feature, time-to-on-call-ready + feedback định tính. Dùng để **cải tiến onboarding**, KHÔNG để chấm điểm gây áp lực.
- ➕ **Remote/hybrid (C-ONB-008):** thiếu kênh hỏi tự nhiên → bù bằng buddy *chủ động ping*, async docs tốt, 1:1 dày tuần đầu, pairing có chủ đích.
- ➕ **Fresh eyes (C-ONB-010):** người mới đề xuất đổi convention tuần đầu → trân trọng góc nhìn mới (họ thấy chỗ team "mù vì quen") nhưng cân với context lịch sử; khuyến khích *quan sát + đề xuất có lập luận*.

**(c) Mép giới hạn & sai lầm**
- ⚠️ Đổ nguyên 2 tuần "đọc tài liệu" rồi mới cho đụng code → mất momentum, không phát hiện lỗi quy trình. *(C-ONB-002)*
- ⚠️ Onboard senior như junior (giao task vụn, review từng dòng) → bóp nghẹt, senior nản. *(C-ONB-004)*
- ⚠️ Ramp chậm sau 60 ngày mà *đoán* nguyên nhân → phải **tách**: onboarding kém? skill gap? fit? *(C-ONB-009)* 🔗 phân biệt với performance management (Bài 7).

## ③ ⚠️ Kiến thức cũ / bị thay thế

| Cái cũ | Thay bằng | Vì sao |
|---|---|---|
| "Đọc hết docs/wiki rồi mới code" | **First PR merged trong tuần đầu** | Smoke-test cả dev loop, win sớm, lộ doc hỏng |
| Onboarding = một buổi orientation | **Khung 30/60/90 có mốc đo được** | Orientation 1 ngày không tạo ramp |
| Một quy trình cho mọi người | **Senior ≠ junior, remote ≠ onsite** | Nhu cầu khác nhau; one-size hại cả hai đầu |
| "Cảm thấy bạn ấy ổn rồi" | **Ramp metric** (time-to-PR/feature/on-call) | Đo để *cải tiến onboarding*, không để ép người |
| Onboarding nằm trong đầu 1 người cũ | **Checklist + docs** (env, C4/ADR, runbook) | Giảm bus factor & câu hỏi lặp |

## ④ Áp dụng thực tế + So sánh bigtech
- **Use case:** ngày 1 — laptop + access + buddy + một "good first issue" đã gắn nhãn. Tuần 1 — PR nhỏ merged. Tuần 2–4 — pairing + đọc ADR mảng mình sẽ sở hữu.
- **Bigtech (pattern):** nhiều công ty lớn dùng **"starter project"** (task có thật, độ khó vừa, scope rõ) để người mới chạm toàn bộ pipeline trong tuần đầu. Khi onboarding chậm lúc *scale* → đó là một trigger kinh điển để đầu tư **platform/IDP (Internal Developer Platform)** nhằm tự động hóa setup. 🔗 N/R.

## ⑤ Artifact thực hành — **Checklist 30/60/90 + buddy**

```text
ONBOARDING — <Tên> — start <ngày> — buddy: <ai> — mentor: <ai>

TUẦN 1 (mục tiêu: first PR merged)
[ ] Access: repo, CI/CD, DB read-only, on-call tool, chat
[ ] Setup env theo README — GHI LẠI mọi chỗ vướng (đó là bug doc)
[ ] Good-first-issue đã gán → PR nhỏ → merged
[ ] Buddy ping chủ động 1 lần/ngày (remote: bắt buộc)

NGÀY 30 (ngữ cảnh)
[ ] Đọc C4/ADR mảng sẽ sở hữu · vẽ lại sơ đồ bằng lời mình (Feynman)
[ ] Hiểu dev loop & quy ước review/branch · shadow 1 phiên on-call

NGÀY 60 (độc lập)
[ ] Ship 1 feature vừa, ít cần dắt
[ ] 1:1 review: ramp đúng kỳ vọng? nếu chậm → tách nguyên nhân

NGÀY 90 (ownership)
[ ] Sở hữu 1 mảng nhỏ · on-call-ready · review được PR người khác

RAMP METRIC (để cải tiến onboarding, KHÔNG chấm điểm):
time-to-first-PR = __ ngày · time-to-first-feature = __ · time-to-on-call = __
```

## ⑥ Keywords cần nhớ
- 🧠 **Cần HIỂU:** time-to-productivity, smoke-test dev loop, buddy ≠ mentor, fresh eyes, early attrition.
- 📌 **Cần THUỘC:** mục tiêu 30/60/90; 3 ramp metric (time-to-first-PR/feature/on-call); buddy tách khỏi đánh giá.
- 🛠️ **Cần LÀM ĐƯỢC:** dựng checklist 30/60/90; chọn good-first-issue; chẩn đoán ramp chậm (onboarding vs skill vs fit).

## ⑦ Mental model
> **"Người mới = service vừa deploy: cho chạy end-to-end (first PR) sớm, rồi mới scale. Đo ramp để sửa *quy trình*, không để ép *người*."**

## ⑧ Câu hỏi phỏng vấn & thách đố
1. *(dễ)* 30/60/90 đặt mục tiêu gì mỗi mốc? *(C-ONB-001)*
2. *(TB)* Vì sao "first PR tuần đầu"? *(C-ONB-002)*
3. *(TB)* Buddy khác mentor ở đâu? *(C-ONB-003)*
4. *(khó)* Onboard senior khác junior thế nào? *(C-ONB-004)*
- 🎯 **Thách đố:** *"Người mới ramp chậm sau 60 ngày — kết luận họ yếu?"* → bẫy: đổ ngay cho skill. Đáp đúng: **tách nguyên nhân** (onboarding kém / skill gap / fit) bằng feedback cụ thể + quan sát, vá đúng chỗ, đặt mốc — *trước khi* nghĩ tới performance management. *(C-ONB-009)*

## ⑨ Bài tập + tiêu chí tự chấm
- **BT1:** Viết checklist tuần-1 cho role của bạn, đảm bảo **first PR khả thi trong 5 ngày**. *Đạt khi* có good-first-issue cụ thể + buddy được chỉ định.
- **BT2:** Liệt kê 3 ramp metric và nói rõ *dùng để làm gì*. *Đạt khi* nêu được "để cải tiến onboarding, không chấm điểm".

## ⑩ Đọc thêm
- *The Manager's Path* (onboarding/mentoring). · Google re:Work — onboarding checklist. · Team Topologies — cognitive load & onboarding khi scale. · 🔗 mục N/R cho IDP/platform.

---

# 🎓 BÀI 4 — CODE REVIEW: CHUẨN & TOOLING CẤP TEAM
*(phủ C-REV-001 → 011)*

## ① Mục tiêu & vị trí trong mạch
- **Vị trí:** chuẩn chất lượng *hằng ngày* ở mức code — nơi văn hóa team hiện ra rõ nhất.
- **Học xong làm được:** định nghĩa mục đích review; phân blocking vs nit; đặt SLA & ngưỡng PR size; chống gatekeeping; dùng Conventional Comments; tách automation vs human; đo "sức khỏe" review mà không biến nó thành thước đo người.
- **Kết nối:** review chia kiến thức → chống bus factor (Bài 6); review là *quality gate* ngăn nợ mới (Bài 5); chống gatekeeping ↔ psychological safety (Bài 7).
- ⚠️ **Ranh giới:** "một đoạn code *cụ thể* nên góp gì" thuộc mục Q (design patterns). Ở đây ta bàn **quy trình & chuẩn review của team**.

## ② Giảng cơ bản → nâng cao

**(a) Trực giác**
Review **không phải cái cổng để chứng tỏ ai giỏi hơn**. Nó là cơ chế *cải thiện code health tổng thể* + *chia sẻ kiến thức* (giảm bus factor) + *giữ nhất quán* + *dạy-học*. Tư duy gốc (Google eng-practices): reviewer **approve khi thay đổi *chắc chắn làm code health tốt lên***, kể cả chưa hoàn hảo — chỉ chặn cái *làm xấu đi*, không chặn để chạy theo "perfect".

**(b) Cơ chế chi tiết**
- 📘 **Mục đích (C-REV-001):** health + knowledge sharing + consistency + mentoring; *không* phải gatekeeping.
- 📘 **Khi nào approve (C-REV-002):** "improves overall code health" ⇒ approve; đừng để cái tốt thành kẻ thù của cái hoàn hảo.
- 📘 **Blocking vs nit (C-REV-003):** *blocking* = correctness/security/maintainability lớn/vi phạm policy; *nit* = style/đặt tên/polish (optional). **Đánh dấu rõ** (prefix `nit:`) để author biết cái nào *buộc* sửa.
- 📘 **Conventional Comments (C-REV-005):** chuẩn cộng đồng (conventionalcomments.org), format `<label> [decorations]: <subject>`. Label lõi: **praise, nitpick, suggestion, issue, todo, question, thought, chore**; decorations: **(blocking) / (non-blocking) / (if-minor)**. Ví dụ: `suggestion (non-blocking): cân nhắc đổi tên uc → userCount`. → author hiểu *loại* và *mức độ* feedback, giảm hiểu lầm/ego. *(verify spec tại site.)*
- 📘 **SLA (C-REV-006):** cửa sổ phản hồi đầu (vd <24h, chấp nhận <48h) để PR không kẹt; backlog review phình = tín hiệu nghẽn; cân tải reviewer (CODEOWNERS / round-robin).
- 📘 **PR size (C-REV-007):** PR lớn (>~400 dòng, nhất là >1000) → review hời hợt, miss bug, kẹt lâu. Kéo nhỏ = tách logic khỏi refactor, vertical slice, feature flag.
- ➕ **Gatekeeping (C-REV-008):** dùng review như *quyền lực* (đòi mọi PR qua mình, chặn vì sở thích cá nhân ngoài convention, ép rewrite để khoe). Sửa = chuẩn hóa rule + **automate** + chuyển sang *teach* + CODEOWNERS phân quyền.
- ➕ **Bất đồng kéo dài (C-REV-009):** quay về principle/tests/standard *đã viết*; vẫn lệch → call ngắn rồi **summarize quyết định vào PR**; có người tie-break (CODEOWNER/TL).
- ➕ **Automation vs human (C-REV-010):** linter/formatter/SAST/AI tool lo style/nit/bug rõ/security hiển nhiên; **người** tập trung kiến trúc, trade-off, logic, maintainability → giữ review **high-signal**. *(verify tool; governance code-AI sâu → R.)*
- ➕ **Metric (C-REV-011):** review latency/time-to-merge, PR size distribution, rework/churn, backlog. ⚠️ tối ưu metric mù → review hời hợt; metric để *phát hiện nghẽn*, **không để xếp hạng người**.

**(c) Mép giới hạn & sai lầm**
- ⚠️ Nit tràn lan → mất velocity (~20–40% theo các nguồn thực hành), che vấn đề lớn, hại morale, đẩy tới gatekeeping. **Gốc**: thiếu tooling → **automate style bằng formatter/linter để nit không tới tay người**. *(C-REV-004)*
- ⚠️ "Changes requested" mà không nói cái nào blocking → author đoán mò, vòng review kéo dài.
- ⚠️ Đo time-to-merge rồi *ép* → người approve cho nhanh, review thành hình thức. *(C-REV-011)*

## ③ ⚠️ Kiến thức cũ / bị thay thế

| Cái cũ | Thay bằng | Vì sao |
|---|---|---|
| Review để "bắt lỗi & chứng tỏ" | **Cải thiện code health + share knowledge** | Gatekeeping giết velocity & morale |
| Chặn PR tới khi "hoàn hảo" | Approve khi **code health tốt lên** | Perfect là kẻ thù của good |
| Comment trộn lẫn không phân loại | **Conventional Comments** (label + blocking/non-blocking) | Author biết cái nào *buộc* sửa |
| Reviewer góp ý style/format thủ công | **Linter/formatter tự động** | Nit không nên tốn não người |
| PR khổng lồ "review một thể" | **PR nhỏ <~400 dòng, vertical slice** | PR lớn → review hời hợt, miss bug |
| Metric review để xếp hạng người | Metric để **phát hiện nghẽn** | Gắn người → game metric, hỏng review |

## ④ Áp dụng thực tế + So sánh bigtech
- **Use case:** team đặt 3 luật ghi vào CONTRIBUTING.md — (1) PR <400 dòng; (2) phản hồi đầu <24h; (3) chỉ `issue (blocking)` mới chặn merge, mọi style do CI lo. Kết quả điển hình: time-to-merge giảm, bug lọt giảm, comment war giảm.
- **Bigtech (pattern):** **Google eng-practices** ("code health") là chuẩn được trích dẫn rộng — reviewer approve khi *cải thiện tổng thể*. Văn hóa nhiều nơi: **automate hết cái máy làm được** (format/lint/SAST), để người review *thiết kế & logic*. **DORA State of DevOps 2024** nhấn: code review nhanh & PR nhỏ tương quan với delivery tốt; và **psychological safety** là predictor mạnh — gatekeeping phá chính cái này.

## ⑤ Artifact thực hành — **Review standard + cheat-sheet Conventional Comments**

```text
# TEAM REVIEW STANDARD (dán vào CONTRIBUTING.md)
1. PR size: mục tiêu <400 dòng diff. Tách logic ≠ refactor. PR >1000 → yêu cầu chẻ.
2. SLA: phản hồi đầu tiên < 24h (chấp nhận <48h). Backlog >N = báo TL.
3. Chỉ comment 'issue (blocking)' mới chặn merge. Mọi style do CI lo (đừng comment tay).
4. Bất đồng >2 vòng → call 15' → ghi quyết định vào PR. Tie-break: CODEOWNER.
5. Approve khi PR LÀM CODE HEALTH TỐT LÊN (không cần hoàn hảo).

# CONVENTIONAL COMMENTS — cheat sheet (conventionalcomments.org)
format: <label> (decoration): <subject>
praise:        Khen — ghi nhận chỗ làm tốt (nên dùng!)
nitpick (non-blocking): cosmetic, được phép bỏ qua
suggestion (non-blocking): đề xuất cải thiện, optional
issue (blocking): vấn đề PHẢI sửa trước merge (kèm 'suggestion' cách sửa)
question (non-blocking): cần làm rõ, chưa chắc là vấn đề
todo / chore: việc nhỏ cần làm trước khi accept
```
> ⚠️ **AI reviewer (2026) viết được hàng tá comment đúng cú pháp** — nhưng dễ ngập **nit** và bỏ sót **trade-off kiến trúc**. *Bạn* chỉ huy: để AI/linter lo style & bug rõ, dành mắt người cho thiết kế, maintainability, "vì sao". Kiểm AI: nó *không* biết PR này nằm trên critical path hay vùng sắp deprecate.

## ⑥ Keywords cần nhớ
- 🧠 **Cần HIỂU:** code health, blocking vs nit, gatekeeping (triệu chứng & gốc), automation vs human split, metric-gaming.
- 📌 **Cần THUỘC:** Conventional Comments labels + (blocking)/(non-blocking)/(if-minor); ngưỡng PR ~400 dòng; SLA <24–48h.
- 🛠️ **Cần LÀM ĐƯỢC:** viết review standard cho team; label comment đúng; setup CODEOWNERS + linter để dẹp nit.

## ⑦ Mental model
> **"Review để code *khỏe lên* và kiến thức *lan ra* — không phải cái cổng quyền lực. Máy lo style/nit, người lo trade-off; metric để dò nghẽn, không xếp hạng người."**

## ⑧ Câu hỏi phỏng vấn & thách đố
1. *(dễ)* Mục đích thật của review ngoài "bắt bug"? *(C-REV-001)*
2. *(TB)* Phân blocking vs nit — vì sao đánh dấu rõ lại quan trọng? *(C-REV-003)*
3. *(khó)* Quá nhiều nitpick hại gì, anh xử lý *gốc* thế nào? *(C-REV-004)*
4. *(rất khó)* Gatekeeping là gì — triệu chứng & cách sửa? *(C-REV-008)*
5. *(khó)* Đo sức khỏe review bằng metric nào, cẩn trọng gì? *(C-REV-011)*
- 🎯 **Thách đố:** *"PR kẹt 5 ngày vì reviewer đòi đổi tên 12 biến cho 'đẹp' — bạn làm gì?"* → đây là gatekeeping + nit-overload; gốc là *thiếu automation* + *thiếu luật blocking/non-blocking*. Sửa hệ thống, không cãi tay đôi. *(C-REV-004/008)*

## ⑨ Bài tập + tiêu chí tự chấm
- **BT1:** Viết review standard 5 dòng cho team bạn. *Đạt khi* có ngưỡng PR size + SLA + luật "chỉ blocking mới chặn".
- **BT2:** Lấy 5 comment review cũ của bạn, gán nhãn Conventional Comments. *Đạt khi* phân biệt rõ cái nào (blocking) cái nào (non-blocking).

## ⑩ Đọc thêm
- Google *eng-practices* — How to do a code review / The Standard of Code Review. · conventionalcomments.org (spec). · DORA State of DevOps 2024. · 🔗 Q (nội dung review cụ thể), R (governance code-AI).

---

# 🎓 BÀI 5 — TECH DEBT MANAGEMENT
*(phủ C-DEBT-001 → 013)*

## ① Mục tiêu & vị trí trong mạch
- **Vị trí:** từ chất lượng *từng PR* (Bài 4) lên **sức khỏe codebase theo thời gian**. Đây là chỗ TL phải *nói chuyện với business*.
- **Học xong làm được:** định nghĩa & phân loại nợ (Fowler quadrant); ưu tiên theo business impact; **thuyết phục stakeholder phi kỹ thuật**; chọn chiến lược ngân sách; biết khi nào *cố ý nhận nợ* và khi nào *KHÔNG trả*; chọn rewrite vs incremental; **ngăn nợ mới**; tránh anti-pattern loại một TL.
- **Kết nối:** prevention (C-DEBT-011) = quality gate ↔ review (Bài 4) + test (🔗 L) + ADR (🔗 Q).
- ⚠️ **Ranh giới:** "smell/ADR *là gì*" thuộc mục Q. Ở đây là **quản trị nợ ở cấp org** (business framing, ngân sách, đo, khi nào không trả).

## ② Giảng cơ bản → nâng cao

**(a) Trực giác**
Ẩn dụ gốc (Ward Cunningham): nợ kỹ thuật như **vay tiền** — vay để ship nhanh là hợp lý *nếu* bạn **trả lãi** (refactor) đúng lúc; quên trả → lãi mẹ đẻ lãi con, mỗi thay đổi sau đều *chậm & đắt hơn*. Khác **bug**: nợ *không nhất thiết sai chức năng* — code chạy đúng nhưng *khó & chậm* để đổi.

**(b) Cơ chế chi tiết**
- 📘 **Định nghĩa (C-DEBT-001):** chi phí ẩn của rework do chọn cách nhanh/dễ thay vì đúng; làm chậm & đắt thay đổi tương lai (khác bug = sai chức năng).
- 📘 **Fowler quadrant (C-DEBT-002):** 2 trục — *deliberate vs inadvertent* × *prudent vs reckless*. "Prudent-deliberate" (cố ý ship nhanh có ý thức, có kế hoạch trả) **chấp nhận được**; "reckless" (thiếu kỷ luật/kiến thức) cần **ngăn**. *(verify khái niệm.)*
- 📘 **Phân loại & ưu tiên (C-DEBT-003):** bucket theo impact — *must-fix-now* (chặn tiến độ/security) → *fix-this-quarter* (chậm dev, chưa nguy) → *observe/monitor*. Ưu tiên theo **business impact**, không coi mọi nợ bằng nhau.
- 📘 **Thuyết phục business (C-DEBT-004):** **dịch rủi ro kỹ thuật sang ngôn ngữ business** — nợ làm *chậm ship feature*, *tăng chi phí bảo trì*, *tăng bug/incident*. Dùng số/ví dụ. **KHÔNG nói "code xấu".**
- ➕ **Chiến lược ngân sách (C-DEBT-005):** % cố định mỗi sprint (vd ~20%) · "boy scout rule" (để code sạch hơn lúc chạm vào) · tech-debt sprint định kỳ. Là *framework*, không ad-hoc lúc đã vỡ.
- ➕ **Đo/visualize (C-DEBT-006):** code complexity, bug/incident rate theo module, lead time/churn, **hotspot analysis** (CodeScene-style — *verify*). Track trong backlog.
- ➕ **Khi nào CỐ Ý nhận nợ (C-DEBT-007):** validate market/deadline thật, vùng sắp bỏ, MVP/prototype — *điều kiện*: cố ý + **log** + có kế hoạch trả.
- ➕ **Khi nào KHÔNG trả (C-DEBT-008):** code ổn định ít đổi / sắp deprecate / ROI thấp; nợ nằm *ngoài* đường thay đổi sắp tới. Tránh "refactor for the sake of it".
- ➕ **Rewrite vs incremental (C-DEBT-009):** rewrite rủi ro cao (mất kiến thức ẩn, dừng feature, hay trễ); ưu tiên **strangler-fig** (bọc & thay dần); rewrite chỉ khi nền tảng *thật sự* chặn & có ngân sách/cam kết.
- ➕ **Prevention (C-DEBT-011):** DoD/quality gate (test, review, lint) + ADR/design review nhẹ + architectural guardrail tự động. **Ngăn rẻ hơn trả.** 🔗 L/Q.

**(c) Mép giới hạn & sai lầm**
- ⚠️ Chỉ đụng nợ khi *vỡ* (reactive). *(C-DEBT-012)*
- ⚠️ Coi mọi nợ critical như nhau (không nuance). *(C-DEBT-012)*
- ⚠️ *Đổ lỗi product team* "ép deadline nên code xấu" (thiếu ownership/collab). *(C-DEBT-012)*
- ⚠️ Big-bang rewrite "cho sạch" — kinh điển trễ & mất kiến thức. *(C-DEBT-009)*

## ③ ⚠️ Kiến thức cũ / bị thay thế

| Cái cũ | Thay bằng | Vì sao |
|---|---|---|
| "Code xấu, cho tôi 1 quý refactor" | **Business framing**: chậm ship X%, tăng incident, đắt bảo trì | Stakeholder không mua "đẹp/xấu" |
| Trả nợ ad-hoc khi đã vỡ (reactive) | **Framework ngân sách**: ~20% / boy-scout / debt-sprint | Proactive rẻ & ít rủi ro hơn |
| Big rewrite "viết lại cho sạch" | **Strangler-fig incremental** | Rewrite hay trễ & mất kiến thức ẩn |
| "Mọi nợ đều phải trả" | **Phân bucket; có nợ KHÔNG nên trả** | ROI thấp/ngoài đường thay đổi → phí công |
| Chỉ trả nợ cũ | **Ngăn nợ mới** (DoD/quality gate/ADR) | Phòng rẻ hơn chữa |
| Trực giác "module này nát" | **Đo**: complexity, incident/module, hotspot | Quyết định data-informed, defensible |

## ④ Áp dụng thực tế + So sánh bigtech
- **Use case (business framing mẫu):** *"Module thanh toán có incident rate gấp 3 phần còn lại và mỗi feature mới ở đó tốn gấp đôi thời gian. Nếu dành 3 sprint x 20% để tách nó ra, ta giảm thời gian giao feature thanh toán ~30% và bớt rủi ro sự cố mùa cao điểm."* — toàn ngôn ngữ business + số, không một chữ "xấu".
- **Bigtech (pattern):** nhiều org dùng **hotspot/complexity analysis** (kiểu CodeScene) để *visualize* nợ theo module + churn → quyết định data-informed. **Strangler-fig** (Martin Fowler) là pattern chuẩn để thay hệ thống lớn dần. *(verify tên tool/giá khi cần.)*

## ⑤ Artifact thực hành — **Tech-debt register + ADR nhận nợ**

```text
# TECH DEBT REGISTER (1 dòng / khoản nợ, để trong backlog có nhãn 'debt')
| ID | Module     | Mô tả nợ            | Business impact         | Bucket        | Owner |
|----|------------|---------------------|-------------------------|---------------|-------|
| D1 | payments   | không có idempotency| double-charge risk + ↑incident | MUST-FIX-NOW | An   |
| D2 | reporting  | query N+1           | dashboard chậm, chưa nguy| THIS-QUARTER  | Bình |
| D3 | legacy-sms | coupling cao        | sắp deprecate Q4         | DON'T-PAY     | —    |

# ADR — CỐ Ý NHẬN NỢ (prudent-deliberate)
Title: Ship MVP loyalty không cache, chấp nhận latency cao
Context: deadline ra mắt thật 30/6; loyalty là feature thử nghiệm thị trường.
Decision: bỏ qua caching layer giờ; log nợ D4.
Consequences: latency ~300ms (chấp nhận được cho MVP).
Pay-back plan: nếu retention >X% sau 1 tháng → thêm cache (ticket D4, owner: An).
```
> ⚠️ **AI gợi ý refactor rất hăng** — nó hay khuyên "viết lại cho sạch" mọi smell nó thấy. *Bạn* kiểm: nợ này có *nằm trên đường thay đổi sắp tới* không? ROI? Có khi câu trả lời đúng là **KHÔNG trả** (C-DEBT-008). Cú pháp refactor AI lo; *quyết định trả/không* là phán đoán business của bạn.

## ⑥ Keywords cần nhớ
- 🧠 **Cần HIỂU:** debt ≠ bug, lãi suất nợ, Fowler quadrant, business framing, prudent vs reckless, strangler-fig, prevention.
- 📌 **Cần THUỘC:** 3 bucket (must-fix-now / this-quarter / observe); chiến lược ngân sách (~20% / boy-scout / debt-sprint); điều kiện "cố ý nhận nợ" = cố ý + log + plan trả.
- 🛠️ **Cần LÀM ĐƯỢC:** viết business case trả nợ (có số, 0 chữ "xấu"); lập debt register; viết ADR nhận nợ; quyết rewrite vs incremental.

## ⑦ Mental model
> **"Nợ là khoản vay — vay có ý thức & trả đúng lúc thì ổn. Nói với business bằng *chậm/đắt/rủi ro*, không bằng *đẹp/xấu*. Có nợ đáng trả, có nợ nên kệ."**

## ⑧ Câu hỏi phỏng vấn & thách đố
1. *(dễ)* Định nghĩa tech debt, khác bug ở đâu? *(C-DEBT-001)*
2. *(TB)* Phân loại & ưu tiên nợ thế nào? *(C-DEBT-003)*
3. *(khó)* Thuyết phục stakeholder phi kỹ thuật chi thời gian trả nợ? *(C-DEBT-004)*
4. *(khó)* Khi nào *cố ý* nhận thêm nợ là đúng? *(C-DEBT-007)*
5. *(rất khó)* Khi nào **KHÔNG** nên trả nợ? *(C-DEBT-008)*
- 🎯 **Thách đố 1:** *"Team đòi dừng feature một quý để refactor toàn bộ — bạn nói gì?"* → cảnh giác big-bang; đòi business case + phạm vi hẹp + incremental + đo impact; tránh cả "không bao giờ trả" lẫn "dừng hết để trả". *(C-DEBT-013)*
- 🎯 **Thách đố 2 (loại TL ngay):** *"Tech debt là lỗi của product vì cứ ép deadline."* → anti-pattern: reactive + không nuance + đổ lỗi/thiếu ownership. Đáp đúng = proactive + framework + business framing + collab. *(C-DEBT-012)*

## ⑨ Bài tập + tiêu chí tự chấm
- **BT1:** Viết 1 business case trả nợ cho module bạn biết, **có ≥2 số & 0 chữ "xấu"**. *Đạt khi* stakeholder phi kỹ thuật hiểu được rủi ro/lợi ích.
- **BT2:** Lập debt register 5 dòng + đánh bucket. *Đạt khi* có ít nhất 1 dòng "DON'T-PAY" có lý do ROI.

## ⑩ Đọc thêm
- Martin Fowler — *TechnicalDebt* & *TechnicalDebtQuadrant* & *StranglerFigApplication* (martinfowler.com). · Ward Cunningham — debt metaphor (video gốc). · CodeScene — behavioral code/hotspot (*verify*). · 🔗 L (test/DoD), Q (ADR/smell).

---

# 🎓 BÀI 6 — DELEGATION & PHÂN CHIA CÔNG VIỆC
*(phủ C-DEL-001 → 011)*

## ① Mục tiêu & vị trí trong mạch
- **Vị trí:** chuyển TL từ "IC giỏi nhất" sang "người *nhân* năng lực team". Đây là bước đổi tư duy lớn nhất khi lên TL.
- **Học xong làm được:** phân biệt delegate/abdicate/micromanage; điều chỉnh mức delegation theo skill×will×risk; giao việc *vừa xong vừa phát triển người*; phá bus factor; dùng RACI/DRI; chia on-call công bằng; ưu tiên khi "cái gì cũng urgent"; cân "tự code vs tạo leverage".
- **Kết nối:** bus factor (C-DEL-006) ↔ doc onboarding (Bài 3) + review chia kiến thức (Bài 4); stretch assignment (C-DEL-004) ↔ growth plan (Bài 7).

## ② Giảng cơ bản → nâng cao

**(a) Trực giác**
TL ôm hết việc khó = **bottleneck + bus factor + team không lớn lên**. Delegation tạo **leverage**: cùng số giờ của bạn nhưng *nhân* qua người khác. Vai TL không phải "code nhiều nhất" mà "làm cả team giỏi hơn".

**(b) Cơ chế chi tiết**
- 📘 **Vì sao delegate (C-DEL-001):** chống bottleneck/bus factor, phát triển người, scale bản thân.
- 📘 **Delegate vs abdicate vs micromanage (C-DEL-002):** *micromanage* = giao rồi soi từng bước (giết autonomy); *abdicate* = quăng việc bỏ mặc (không support/đo); **delegate đúng** = giao *outcome + context + mức tự chủ + checkpoint*, **vẫn chịu trách nhiệm cuối**.
- 📘 **Mức delegation (C-DEL-003):** điều chỉnh theo **skill × will** của người **× rủi ro task**. Người mới/việc rủi ro cao → dắt sát; người giỏi/việc rõ → giao outcome & buông. *(Tannenbaum–Schmidt / situational leadership.)*
- 📘 **Stretch (C-DEL-004):** giao việc hơi quá tầm + safety net (pairing/review/mentor); cân "việc cần xong" với "cơ hội học".
- 📘 **Phân theo level (C-DEL-005):** junior → task rõ ràng; senior → mơ hồ/ownership. Tránh một người ôm hết phần khó (bus factor) hoặc junior chìm.
- 📘 **Bus factor / hero culture (C-DEL-006):** một người biết mọi thứ = rủi ro nghỉ/ốm/burnout + nghẽn. Phá = pairing, rotation, doc, review chia kiến thức; **đo & chủ động phân tán**.
- ➕ **RACI / DRI (C-DEL-007):** làm rõ Responsible/Accountable/Consulted/Informed (hoặc 1 DRI) → hết "tưởng người kia làm", nhất là task cross-team.
- ➕ **Can thiệp khi đi sai (C-DEL-008):** đặt **checkpoint/milestone** thay vì soi liên tục; can thiệp khi chạm *ngưỡng rủi ro đã thỏa thuận*; coaching (hỏi-hướng-dẫn) **trước** khi giành tay lái.
- ➕ **On-call (C-DEL-009):** rotation rõ, không dồn 1–2 người; gắn với *giảm toil* & blameless; tách on-call vs feature trong capacity.
- ➕ **Ưu tiên khi "cái gì cũng urgent" (C-DEL-010):** hỏi context/business objective/effort trước; framework (impact vs effort, hạn *thật* vs *giả*); **đẩy ngược câu hỏi**, không nhận hết.
- ➕ **TL tự code bao nhiêu (C-DEL-011):** TL vẫn code (tooling/automation/refactor/vùng rủi ro) nhưng **không nằm trên critical path** mọi feature. Quá hands-on → bottleneck; quá hands-off → mất context/credibility.

**(c) Mép giới hạn & sai lầm**
- ⚠️ Giao xong "soi từng commit" = micromanage trá hình. *(C-DEL-002)*
- ⚠️ Giao rồi biến mất, không checkpoint = abdicate → fail rồi mới biết. *(C-DEL-002/008)*
- ⚠️ Một mức delegation cho mọi người (cùng buông hết hoặc cùng dắt sát). *(C-DEL-003)*
- ⚠️ Để "ticket khó nào cũng về một người giỏi" = nuôi bus factor=1. *(C-DEL-006)*

## ③ ⚠️ Kiến thức cũ / bị thay thế

| Cái cũ | Thay bằng | Vì sao |
|---|---|---|
| TL = người code giỏi nhất, ôm việc khó | TL = **nhân năng lực qua delegation** | Ôm việc = bottleneck + bus factor |
| Giao việc = giao *task* + theo dõi sát | Giao **outcome + context + mức tự chủ + checkpoint** | Tránh micromanage & abdicate |
| Cùng một kiểu giao cho mọi người | **Skill × will × risk** điều chỉnh mức | Người & việc khác nhau |
| "Để bạn A lo, bạn ấy rành nhất" | **Rotation + pairing + doc** phá bus factor | Hero culture = rủi ro + burnout |
| Ai cũng "urgent" → làm hết | **Hỏi context + ưu tiên impact/effort + đẩy ngược** | Không có ưu tiên = không có chiến lược |

## ④ Áp dụng thực tế + So sánh bigtech
- **Use case:** vùng code "chỉ Minh hiểu" (bus factor=1). Kế hoạch phá trong 1 quý: (1) Minh pair với 1 người mỗi lần đụng vùng đó; (2) viết runbook/ADR; (3) rotate 2 ticket vùng đó sang người khác có Minh review; (4) đo: số người có thể on-call vùng đó tăng từ 1 → 3.
- **Bigtech (pattern):** **DRI** (Directly Responsible Individual — phổ biến từ Apple) cho mỗi đầu việc cross-team; **on-call rotation** + **blameless postmortem** (Google SRE) là chuẩn để chia tải vận hành bền vững. **Delegation poker** (Management 3.0) là công cụ làm rõ "mức tự chủ" với team.

## ⑤ Artifact thực hành — **Delegation brief + RACI**

```text
# DELEGATION BRIEF (dùng khi giao 1 việc quan trọng)
Việc (OUTCOME, không phải task): "Giảm p95 latency API checkout xuống <200ms"
Context (vì sao quan trọng): mùa sale, mỗi 100ms latency ~ X% drop conversion.
Mức tự chủ (chọn 1):
  [ ] làm theo cách tôi chỉ   [ ] đề xuất rồi tôi duyệt   [X] tự quyết, báo kết quả
Ngưỡng phải hỏi tôi (risk threshold): đổi schema DB / đụng payment provider.
Checkpoint: sync 15' giữa tuần · demo cuối sprint.
Safety net: pair với senior 2 buổi đầu · tôi review thiết kế trước khi build.

# RACI cho task cross-team "Migrate auth service"
| Hoạt động        | Backend | Platform | Security | PM |
|------------------|---------|----------|----------|----|
| Thiết kế migration|   A/R   |    C     |    C     | I  |
| Rollout          |   R     |   A/R    |    C     | I  |
| Threat review    |   C     |    I     |   A/R    | I  |
(A=Accountable chỉ 1 người/dòng · R=làm · C=hỏi ý · I=báo tin)
```
> ⚠️ **"Hiểu để chỉ huy":** AI/junior có thể *làm* được task khi bạn giao rõ outcome + context + ngưỡng rủi ro. Sai lầm hay gặp là giao *task mơ hồ* rồi bực vì kết quả lệch — gốc là **bạn chưa định hình yêu cầu**, không phải họ kém.

## ⑥ Keywords cần nhớ
- 🧠 **Cần HIỂU:** leverage, delegate vs abdicate vs micromanage, skill×will×risk, stretch + safety net, bus factor, coaching trước khi giành lái.
- 📌 **Cần THUỘC:** RACI (1 Accountable/dòng); 3 mức tự chủ; cấu trúc delegation brief (outcome+context+autonomy+checkpoint+threshold).
- 🛠️ **Cần LÀM ĐƯỢC:** viết delegation brief; lập RACI cross-team; vạch kế hoạch phá bus factor có metric; ưu tiên backlog "ai cũng urgent".

## ⑦ Mental model
> **"Đừng làm việc của team — làm cho team làm được việc. Giao *outcome + ngưỡng rủi ro*, không giao *từng bước*; chịu trách nhiệm cuối nhưng buông tay lái."**

## ⑧ Câu hỏi phỏng vấn & thách đố
1. *(dễ)* Vì sao TL phải delegate thay vì ôm việc khó? *(C-DEL-001)*
2. *(TB)* Phân biệt delegate / abdicate / micromanage. *(C-DEL-002)*
3. *(khó)* Mức delegation điều chỉnh theo gì? *(C-DEL-003)*
4. *(rất khó)* Bus factor=1 / hero culture — rủi ro & cách phá? *(C-DEL-006)*
5. *(khó)* Là TL, anh cân tự code vs tạo leverage qua người thế nào? *(C-DEL-011)*
- 🎯 **Thách đố:** *"Đã giao việc, nhưng bạn thấy người làm sắp đi sai hướng — can thiệp ngay?"* → bẫy: nhảy vào giành lái = micromanage/abdicate-flip. Đáp đúng: đã có checkpoint/ngưỡng từ đầu; can thiệp khi *chạm ngưỡng*; coaching (hỏi-hướng) trước; để họ học từ sai lầm nhỏ. *(C-DEL-008)*

## ⑨ Bài tập + tiêu chí tự chấm
- **BT1:** Viết delegation brief cho 1 việc bạn từng tự ôm. *Đạt khi* có outcome (không phải task) + ngưỡng rủi ro + checkpoint.
- **BT2:** Chọn 1 vùng bus-factor cao, viết kế hoạch phá có **metric đo được**. *Đạt khi* nêu được "số người làm được vùng đó: từ X → Y".

## ⑩ Đọc thêm
- *The Manager's Path* (delegation). · *An Elegant Puzzle* (sizing/load). · Google SRE Book — on-call & blameless. · Management 3.0 — Delegation Poker. · Tannenbaum & Schmidt — continuum of leadership.

---

# 🎓 BÀI 7 — TEAM BUILDING, GROWTH & HEALTH (capstone)
*(phủ C-TEAM-001 → 013)*

## ① Mục tiêu & vị trí trong mạch
- **Vị trí:** đỉnh của mục C — tổng hợp tất cả thành **nuôi dưỡng con người & đội ngũ dài hạn**. Đỉnh-của-đỉnh là **ranh giới TL vs EM**.
- **Học xong làm được:** dùng career ladder; chạy 1:1 thật; dựng growth plan; phát hiện & xử lý underperformer sớm/công bằng; xây psychological safety; đo team health đa chiều; giữ chân; cho/nhận feedback; quyết topology/headcount/transfer; xây standards không top-down; và nói rõ TL vs EM.
- **Kết nối:** psych safety là *điều kiện* cho blameless (Bài 5), chống gatekeeping (Bài 4), buddy hỏi thoải mái (Bài 3). Đây là nơi mọi bài trước "khép vòng".

## ② Giảng cơ bản → nâng cao

**(a) Trực giác**
Velocity là *kết quả*, không phải *nguyên nhân*. Nguyên nhân là **con người khỏe & an toàn**. Một team có **psychological safety** (dám nói "tôi không biết", dám báo lỗi, dám phản biện sếp) sẽ raise risk sớm, học nhanh, không giấu lỗi. DORA State of DevOps 2024 xếp psych safety là **một trong những predictor mạnh nhất** của software delivery performance — nó không phải "mềm", nó là *đòn bẩy hiệu suất*.

**(b) Cơ chế chi tiết**
- 📘 **Career ladder (C-TEAM-001):** định nghĩa kỳ vọng theo level (scope/autonomy/impact) → công bằng, rõ đường thăng tiến, làm chuẩn cho feedback/promotion/leveling.
- 📘 **1:1 (C-TEAM-002):** **không phải** status update; là kênh build trust, gỡ blocker, career/growth, feedback 2 chiều, bắt sớm tín hiệu nghỉ. **Người làm sở hữu agenda.**
- 📘 **Growth plan (C-TEAM-003):** gắn ladder → xác định gap cụ thể (scope/ambiguity/leadership) → stretch + mentor + feedback đo được + mốc; làm *cùng* họ, không hứa suông.
- 📘 **Underperformer (C-TEAM-004):** **tách nguyên nhân** (skill/will/fit/ngữ cảnh/onboarding) bằng feedback cụ thể & evidence *sớm*; plan có mốc, documented; phân biệt với PIP chính thức (loop HR); kéo dài hại cả team.
- 📘 **Psychological safety (C-TEAM-005):** an toàn để sai/hỏi/phản biện mà không bị trừng phạt → blameless postmortem, dám raise risk sớm.
- ➕ **Team health (C-TEAM-006):** kết hợp *delivery* (lead time, deploy freq — DORA) + *con người* (eNPS, retention, on-call load, burnout signal). **Velocity đơn lẻ dễ bị game.** 🔗 O/N.
- ➕ **Retention (C-TEAM-007):** growth/impact/autonomy/mastery + tech tốt + manager tốt; lương là *điều kiện cần không đủ*; bắt tín hiệu sớm ở 1:1.
- ➕ **Feedback (C-TEAM-008):** cụ thể-kịp thời-*về hành vi* (không nhân cách), riêng tư khi tiêu cực, gắn impact (SBI: Situation-Behavior-Impact); văn hóa feedback 2 chiều *thường xuyên*, không dồn tới review cuối năm.
- ➕ **Topology / khi nào tách team (C-TEAM-009):** team quá lớn → communication overhead (~n²); "two-pizza"/Team Topologies (stream-aligned, ownership rõ); tách khi cognitive load/đường biên service quá tải. 🔗 A.
- ➕ **Internal transfer (C-TEAM-010):** ưu tiên lợi ích *người & org* dài hạn hơn giữ cục bộ; minh bạch, không "giam" người; có kế hoạch backfill.
- ➕ **Standards không top-down (C-TEAM-011):** đồng thuận qua thảo luận/RFC/ADR, *viết ra & tự động hóa* (lint/CI), review định kỳ; ownership tập thể tăng tuân thủ.
- ➕ **Headcount (C-TEAM-012):** gắn với load/roadmap/khả năng giao hàng & rủi ro bus factor; dữ liệu (backlog, on-call, lead time) + cân nhắc tự động hóa/build-vs-buy trước.
- ➕ **TL vs EM (C-TEAM-013):** **TL** thiên technical direction/architecture/chất lượng (có thể không quản nhân sự chính thức); **EM** sở hữu performance/career/comp/headcount; nhiều nơi *chồng lấn* — quan trọng là **nói rõ ai làm gì**.

**(c) Mép giới hạn & sai lầm**
- ⚠️ Biến 1:1 thành status update → mất kênh trust & tín hiệu nghỉ sớm. *(C-TEAM-002)*
- ⚠️ Đo team chỉ bằng velocity → bị game, bỏ sót burnout. *(C-TEAM-006)*
- ⚠️ Né underperformer cho "đỡ căng" → cả team gánh & mất công bằng. *(C-TEAM-004)*
- ⚠️ Feedback đánh vào *nhân cách* ("anh cẩu thả") thay vì *hành vi* ("PR này thiếu test, gây incident X"). *(C-TEAM-008)*
- ⚠️ Áp standards top-down cứng → phản kháng, không tuân. *(C-TEAM-011)*

## ③ ⚠️ Kiến thức cũ / bị thay thế

| Cái cũ | Thay bằng | Vì sao |
|---|---|---|
| 1:1 = báo cáo tiến độ | 1:1 = **trust/growth/blocker, người làm sở hữu agenda** | Status update đã có ở standup |
| Đo team bằng **velocity** | **DORA delivery + tín hiệu con người (eNPS/retention/burnout)** | Velocity đơn lẻ bị game; bỏ sót người |
| Feedback dồn vào review cuối năm | **Feedback liên tục, SBI, về hành vi** | Kịp thời + cụ thể mới đổi được hành vi |
| "Psych safety là chuyện mềm" | Predictor **mạnh nhất** của delivery (DORA 2024) | Là đòn bẩy hiệu suất, không phải xa xỉ |
| Standards áp top-down | **RFC/ADR đồng thuận + tự động hóa** | Ownership tập thể → tuân thủ thật |
| Giữ người giỏi bằng tăng lương | Growth/autonomy/mastery/manager tốt | Lương cần-không-đủ; người đi vì sếp/nhàm |
| Promote theo "thâm niên/cảm tình" | **Career ladder** (scope/autonomy/impact) | Công bằng & defensible |

## ④ Áp dụng thực tế + So sánh bigtech (search + cite)
- **Use case:** dashboard team health hằng quý: DORA 4 keys (deploy freq, lead time, change failure rate, failed-deploy recovery) + on-call load + retention + 1 câu eNPS. Đọc *cả cụm*, không cherry-pick velocity.
- **Bigtech (cite):** **DORA State of DevOps 2024** xác nhận lại psychological safety + ranh giới team rõ ràng + autonomy là những yếu tố dự báo hiệu suất mạnh; 2024 còn bổ sung **metric thứ 5 (rework rate)** cho stability và nhấn mạnh **không gắn metric vào lương cá nhân** (sẽ bị game). **Team Topologies** (Skelton & Pais) là khung chuẩn cho stream-aligned teams & cognitive load — cơ sở để quyết khi nào tách team. Google "Project Aristotle" cũng đặt psychological safety là yếu tố #1 của team hiệu quả.

## ⑤ Artifact thực hành — **1:1 template + growth plan + SBI feedback**

```text
# 1:1 (người làm SỞ HỮU agenda; TL chỉ thêm mục cuối)
1. Bạn muốn nói gì hôm nay? (họ dẫn)
2. Có blocker nào tôi gỡ được không?
3. Career/growth: tuần này có gì tiến tới mục tiêu senior?
4. Feedback cho tôi/team? (2 chiều)
[TL thêm] 1 lời ghi nhận cụ thể + 1 feedback SBI nếu có.
→ Lắng nghe tín hiệu: mệt mỏi? nhàm? nhắc đến offer ngoài?

# GROWTH PLAN (mid → senior)
Gap (theo ladder): chưa dẫn dắt ở task mơ hồ; chưa mentor.
Hành động đo được: (1) own 1 dự án scope rộng Q3 + tôi mentor;
  (2) lead 1 design review; (3) onboard 1 người mới.
Mốc & review: check 1:1 mỗi 2 tuần; đánh giá lại cuối Q3.

# FEEDBACK SBI (về HÀNH VI, không nhân cách)
Situation: "PR thanh toán hôm thứ 3..."
Behavior:  "...merge mà chưa có test cho path refund..."
Impact:    "...gây incident 2 giờ + mất niềm tin của on-call."
→ Hỏi: "Bạn thấy lần sau làm khác được chỗ nào?" (cùng tìm cách)
```
> ⚠️ **"Hiểu để chỉ huy & kiểm tra" — ví dụ capstone:** AI dựng được dashboard "team health" đầy số đẹp. Nhưng nó *không* biết velocity đang **tăng giả** vì team chẻ task để game metric, hay một senior đang burnout dù chỉ số xanh. Chỉ người *hiểu* mới đọc được *cả cụm* tín hiệu (DORA + con người) và bắt được khi metric bị game. **Số đúng, kết luận sai** — và đó là việc của bạn.

## ⑥ Keywords cần nhớ
- 🧠 **Cần HIỂU:** psychological safety, team health đa chiều, velocity-gaming, skill/will/fit khi underperform, communication overhead n², TL vs EM.
- 📌 **Cần THUỘC:** DORA 4(+1) keys; SBI (Situation-Behavior-Impact); "1:1 = người làm sở hữu agenda"; ladder = scope/autonomy/impact; "two-pizza" / stream-aligned.
- 🛠️ **Cần LÀM ĐƯỢC:** chạy 1:1 thật; viết growth plan & SBI feedback; dựng dashboard team health; justify headcount bằng data; chẩn đoán underperformer theo nguyên nhân.

## ⑦ Mental model
> **"Velocity là *kết quả*; con người an toàn & khỏe là *nguyên nhân*. Đo *cả cụm* (delivery + người), feedback về *hành vi*, và biết rõ ranh giới TL ↔ EM."**

## ⑧ Câu hỏi phỏng vấn & thách đố
1. *(TB)* 1:1 để làm gì, khác status update? *(C-TEAM-002)*
2. *(khó)* Psychological safety là gì, vì sao nó quyết định chất lượng team? *(C-TEAM-005)*
3. *(khó)* Đo team health bằng gì, không chỉ velocity? *(C-TEAM-006)*
4. *(rất khó)* Phát hiện & xử lý underperformer sớm/công bằng thế nào? *(C-TEAM-004)*
5. *(rất khó)* Phân biệt TL vs EM — ranh giới trách nhiệm con người? *(C-TEAM-013)*
- 🎯 **Thách đố 1:** *"Sếp bảo: tăng velocity 30% quý sau."* → bẫy: nhận và ép. Đáp đúng: velocity là output dễ game; hỏi *outcome business* thật, đo *cả cụm* DORA + người, không gắn metric vào lương. *(C-TEAM-006)*
- 🎯 **Thách đố 2:** *"Một engineer giỏi muốn transfer sang team khác — bạn giữ?"* → bẫy: giam người vì lợi ích cục bộ. Đáp đúng: ưu tiên người & org dài hạn, minh bạch, backfill. *(C-TEAM-010)*

## ⑨ Bài tập + tiêu chí tự chấm
- **BT1:** Viết 1 feedback tiêu cực thật theo **SBI**. *Đạt khi* nói về *hành vi + impact*, 0 chữ đánh vào nhân cách, kết bằng câu hỏi cùng tìm cách.
- **BT2:** Thiết kế dashboard team health 5 chỉ số (≥2 DORA + ≥2 con người). *Đạt khi* giải thích được mỗi chỉ số dễ bị game ra sao và vì sao đọc cả cụm.

## ⑩ Đọc thêm
- DORA *State of DevOps 2024* (dora.dev). · Google re:Work — *Project Aristotle* (psychological safety). · *Team Topologies* — Skelton & Pais. · *The Manager's Path* (1:1, ladder, underperformer). · Amy Edmondson — *The Fearless Organization*. · SBI model — Center for Creative Leadership.

---
---

# ✅ KẾT — Tổng kết mục C & bước tiếp

**Một câu xuyên suốt mục C:**
> *"Hiring & Team Building là chuỗi hệ thống con người: phễu lọc đúng → đánh giá có evidence → onboard nhanh → review để code khỏe & kiến thức lan → quản nợ bằng business framing → delegate để nhân năng lực → nuôi team an toàn & khỏe. Mọi thứ đều quy về: *biến cảm tính thành hệ thống đo được, và giữ con người an toàn để hệ thống chạy.*"*

**Trục "hiểu để chỉ huy & kiểm tra" trong mục C (nhớ cho phỏng vấn 2026):**
- Phỏng vấn: dò *chỉ huy & kiểm AI*, không dò gõ code (Bài 2).
- Review: máy lo nit/style, người lo trade-off (Bài 4).
- Delegation: giao *outcome + ngưỡng*, người/AI làm phần còn lại (Bài 6).
- Team health: AI dựng dashboard, *bạn* bắt khi số bị game (Bài 7).

**Theo WORKFLOW 2 — sau Bước B & C là:**
- **Bước D — Phỏng vấn theo đợt (chấm LIVE):** gõ *"phỏng vấn tôi mục C"* → tôi hỏi 10–15 câu trộn dễ→khó từ 82 ID, **dừng chờ bạn trả lời**, rồi dựng đáp án chuẩn + chấm ĐẠT/CHƯA theo dòng *"dò cái gì"*.
- **Bước E — Cổng hoàn thành + spaced repetition:** xong mục C = ĐẠT toàn bộ 82 câu; các đợt sau chèn 1–2 câu cũ (hôm nay → mai → 3 ngày → 1 tuần).

> ⚠️ **Chất liệu mạnh nhất là kinh nghiệm thật của bạn.** Trước Bước D, hãy chuẩn bị sẵn vài câu chuyện STAR cho: một lần *phỏng vấn/tuyển* (tốt hoặc hỏng), một lần *thuyết phục business trả nợ*, một lần *delegate* (hoặc ôm việc rồi hối hận), một lần *xử lý underperformer*. AI lo khung — bạn lo câu chuyện.

*(Tài liệu sinh 6/2026. Các con số benchmark là rule-of-thumb; verify lại tên tool & chính sách công ty khi dùng thật.)*
