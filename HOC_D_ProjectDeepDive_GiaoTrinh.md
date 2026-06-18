# 🎓 HỌC MỤC D — Deep-dive dự án cũ (Project Deep-Dive)
### Giáo trình ngược + Toàn bộ bài giảng (Bước B + Bước C của WORKFLOW 2)

> **Nguồn gốc:** dựng ngược từ `QB_D_projectdeepdive.md` (79 câu / 9 mục con).
> **Đặc thù mục D:** chất liệu là **dự án THẬT của bạn** → đây không phải bài lý thuyết engine, mà là **kỹ năng kể, biện minh và bảo vệ dự án dưới sức ép phỏng vấn**. Mỗi "bài học" dạy *cách trả lời một cụm câu vặn*, kèm khung tư duy + template + tiêu chí tự chấm (chính là dòng *"dò cái gì"* của QB).
> **Cách dùng tài liệu này:**
> - Đọc Bước B để thấy bản đồ 9 bài và ánh xạ Bài → ID câu hỏi.
> - Học từng bài theo thứ tự (cơ bản → nâng cao). Cuối mỗi bài có **Tự kiểm (recall)** — *che đáp án, tự trả lời trước, rồi mới dò* (active recall).
> - Trước khi luyện, chuẩn bị 2 thứ bắt buộc (QB yêu cầu): **sơ đồ C4 dự án bạn** (draw.io/Excalidraw) + **vài số liệu tải thật** (QPS, data size, latency p50/p95/p99).
> **Câu thần chú xuyên suốt:** *"Tôi không nhớ để gõ — tôi hiểu để chỉ huy và kiểm tra, phần còn lại tra cứu."*

---

## 📌 Bối cảnh 2026 — vì sao mục D ngày càng nặng (đã verify)

Tổng hợp các nguồn tuyển dụng kỹ sư 1–6/2026 xác nhận đúng giả định của QB:

- **AI sinh được kiến trúc "nghe rất hợp lý" trong vài giây**, nên interviewer dịch trọng tâm sang *kinh nghiệm production thật*: ở cấp senior/staff họ thường chọn **3–5 câu rồi khoan rất sâu** vào failure modes, trade-off và *"lần trước hỏng ở đâu"*, kỳ vọng có **sơ đồ + war story thật** chứ không phải lý thuyết. *(TechEon, Agentic AI System Design Interview Guide 2026; Adil Shamim, "Every AI Engineer Interview Question 2026 — from 100+ real interviews")*
- Vòng deep-dive ở senior có thể **chiếm trọn 1 giờ chỉ cho MỘT dự án**, trộn lẫn behavioral + technical; interviewer *đào sâu dần* và muốn đây là **đối thoại, không phải độc thoại**. Ví dụ Anthropic: **25 phút trình bày + 15–20 phút hỏi vặn**. *(alexeygrigorev, ai-engineering-field-guide — interview/project-deep-dive, Q4'25–Q1'26)*
- **Tín hiệu phân loại seniority quan trọng nhất:** cách bạn KỂ tiết lộ *bạn ra quyết định hay chỉ thực thi việc được giao, bạn chủ động phát hiện vấn đề hay chờ được bảo*. Ứng viên giỏi **đóng khung quanh impact** ("giảm 40% thời gian phản hồi"), không quanh tên công nghệ ("dùng LangChain + Pinecone"). *(alexeygrigorev field-guide)*
- Lời khuyên lặp lại khắp nguồn: **chuẩn bị sẵn 3–5 project deep-dive, thuộc từng quyết định & số liệu, luyện chịu follow-up probe**; mỗi câu trả lời nén ~**60–90 giây**, mỗi khái niệm gắn 1 ví dụ đã ship thật. *(Adil Shamim 2026; lockedin.ai Top-40 2026; Careery 2026)*

> ⚠️ Mọi chỗ chạm **tên/đời công nghệ, model, giá, số benchmark** trong câu trả lời CỦA BẠN vẫn phải *(verify tại nguồn chính thức)* trước khi khẳng định.

---
---

# 🔄 BƯỚC B — GIÁO TRÌNH NGƯỢC (dựng từ 79 câu)

**Nguyên tắc gom:** gom theo *cụm năng lực bị vặn*, sắp **dependency order** (cái nền trước) — KHÔNG theo tần suất. Bạn phải biết *kể & vẽ* trước, rồi mới *biện minh*, rồi *định lượng*, rồi *kể sự cố*, rồi *quy công*, cuối cùng là *phản tư* và *chịu khoan sâu / chính trực*.

### Mục lục 9 bài

| # | Bài học | Năng lực lõi | Phủ ID | Số câu |
|---|---|---|---|---|
| **1** | Khung kể & mở màn — STAR/CARL, front-load impact, tailor audience | Cấu trúc câu chuyện, giao tiếp | ARCH-001, ARCH-004, ARCH-007, COMM-003, COMM-004, COMM-006 | 6 |
| **2** | Vẽ & trình bày kiến trúc — C4, container, trace request | Trực quan hóa hệ thống | ARCH-002, ARCH-003, ARCH-005, ARCH-006, ARCH-010 | 5 |
| **3** | Biện minh "vì sao" mỗi lựa chọn — DB, mono/micro, queue, cache, sync/async | Justify decisions | ARCH-008, WHY-001, WHY-002, WHY-003, WHY-004, WHY-005, WHY-007, WHY-008, WHY-009 | 9 |
| **4** | Trade-off & quyết định dưới ràng buộc — alternatives, build-vs-buy, tech debt | Trade-off articulation | ARCH-009, WHY-010, TRADE-001…008 | 10 |
| **5** | Số liệu & quy mô thật — QPS, percentile latency, bottleneck, SLO, cost | Định lượng | SCALE-001…009 | 9 |
| **6** | Sự cố production, debug & on-call — incident, MTTR, postmortem, race condition | War story + phương pháp | PROB-001…009, WHY-006 | 10 |
| **7** | Đóng góp cá nhân — "I vs we", ownership, leadership kỹ thuật | Quy công trung thực | ROLE-001…009, COMM-005 | 10 |
| **8** | Hồi tố & cải tiến — làm lại đổi gì, Learnings, lãi kép, YAGNI | Phản tư | RETRO-001…008 | 8 |
| **9** | Chịu khoan sâu & chính trực — read-vs-built, "tôi không biết", chỉ huy & kiểm tra | Độ tin dưới sức ép | CRED-001…008, WHY-011, WHY-012, COMM-001, COMM-002 | 12 |

**Tổng phủ: 79/79 câu.** ✅

**Đồ thị phụ thuộc (học theo chiều mũi tên):**
```
Bài 1 (kể)  →  Bài 2 (vẽ)  →  Bài 3 (vì sao)  →  Bài 4 (trade-off)
                                                      │
                                                      ▼
   Bài 9 (chính trực) ← Bài 8 (hồi tố) ← Bài 7 (quy công) ← Bài 6 (sự cố) ← Bài 5 (số liệu)
```
Lý do thứ tự: Bài 1–2 cho bạn *khung & hình* để mọi câu sau bám vào. Bài 3–4 là *lõi biện minh* (đa số follow-up rơi vào đây). Bài 5 cấp *bằng chứng định lượng* để Bài 6–7 (sự cố, quy công) đáng tin. Bài 8–9 là *tầng senior+*: phản tư và giữ chính trực khi bị khoan tới đáy — chỉ vững khi 7 bài trước đã chắc.

---
---

# 🔄 BƯỚC C — TOÀN BỘ 9 BÀI GIẢNG (Hợp đồng đầu ra 10 mục)

> Ký hiệu: **📘** = bám đúng tiêu chí *"dò cái gì"* trong `QB_D` · **➕** = bổ sung/nâng cấp của giảng viên · **⬆️** = làm sâu hơn QB.
> Mục **⑤** trong mục D không phải "code" mà là **kịch bản mẫu (script) + artefact cần chuẩn bị** — vì sản phẩm ở đây là *câu trả lời*, không phải phần mềm.

---

## 📖 BÀI 1 — Khung kể & cách mở màn
*(Phủ: D-ARCH-001, D-ARCH-004, D-ARCH-007, D-COMM-003, D-COMM-004, D-COMM-006)*

### ① Mục tiêu & vị trí trong mạch
Đây là **bài nền của cả mục D**. Học xong bạn **mở màn một deep-dive không lạc**, **đóng khung quanh impact** (không quanh tech), **điều chỉnh độ chi tiết theo người nghe**, và **giữ mạch khi bị ngắt**. Mọi câu vặn ở Bài 2–9 đều bám vào bộ khung dựng ở đây. Không có khung → câu trả lời thành "liệt kê việc đã làm", interviewer không đọc ra seniority.

### ② Giảng: cơ bản → nâng cao

**(a) Trực giác.** Một deep-dive senior giống *dẫn tour một tòa nhà*, không phải *đọc bản vẽ từ móng lên*. Người nghe cần **bản đồ trước, rồi mới zoom**. Mở bằng "tòa nhà này để làm gì, ai dùng, vì sao đáng xây" → rồi chỉ 2–3 khu sẽ đi qua → rồi mới bước vào từng phòng khi được hỏi.

**(b) Cơ chế — 3 khung xương:**

- **Opener 3 lớp (📘 ARCH-001/004):**
  1. *1 câu bài toán + bối cảnh business* — "Hệ X xử lý thanh toán cho ~Y người dùng; vấn đề cũ là Z."
  2. *"Table of contents"* — "Em sẽ kể 3 trục: kiến trúc tổng thể, 2 quyết định khó, và 1 sự cố production."
  3. *Front-load impact* — đặt kết quả đo được lên TRƯỚC, không để cuối: "kết quả là giảm p99 từ 1.2s xuống 300ms."
- **STAR vs CARL (➕):**
  - **STAR** = Situation · Task · Action · Result — chuẩn cho *một tình huống ngắn*.
  - **CARL** = Context · Action · Result · **Learning** — tốt hơn cho *dự án dài*, vì lớp **Learning** là thứ làm nổi seniority (QB nhấn mạnh). Dùng CARL khi câu hỏi là "kể về dự án phức tạp nhất".
- **Tailor audience (📘 ARCH-007):** cùng một hệ, ba cách kể:
  | Người nghe | Nhấn vào | Bỏ bớt |
  |---|---|---|
  | Kỹ sư/TL | cơ chế, trade-off, failure mode | bối cảnh business dài |
  | PM | người dùng, timeline, phạm vi | chi tiết DB/queue |
  | VP/Exec | impact, cost, rủi ro | mọi tên công nghệ |

**(c) Mép giới hạn & sai lầm thường gặp:**
- ❌ Kể tuần tự *từ tầng dưới lên* ("đầu tiên em dựng DB...") → người nghe lạc trước khi tới điểm hay.
- ❌ Mở bằng tech stack ("dự án dùng Node + Postgres + Redis + Kafka...") → đó là *cái dùng*, không phải *vấn đề giải quyết*.
- ❌ Khi bị ngắt (ARCH/COMM-004) thì *bỏ luôn mạch cũ* → trả lời xong câu chen ngang là quên đang kể gì. Cách đúng: **trả lời thẳng câu hỏi (~30–60s) → 1 câu cầu nối "quay lại trục em đang kể..." → tiếp tục.**

> **🧭 Hiểu để chỉ huy & kiểm tra:** AI viết được một bản tóm tắt dự án "nghe trôi chảy", nhưng nó **không biết front-load impact NÀO là quan trọng với người nghe CỤ THỂ trước mặt**. Việc *chọn nhấn gì, bỏ gì theo audience* là phán đoán của con người — **kiểm tra**: nếu bản nháp AI mở bằng tech stack, đó là dấu hiệu sai khung, sửa lại thành bài-toán-trước.

### ③ ⚠️ Kiến thức cũ / lối kể bị thay thế

| Lối kể cũ (phổ biến) | Thay bằng (chuẩn 2026) | Vì sao |
|---|---|---|
| Kể tuần tự từ hạ tầng → tính năng | Front-load bài toán + impact, rồi "table of contents" | Interviewer chỉ có 30–60'; cần thấy điểm hay sớm |
| Mở màn bằng liệt kê tech stack | Mở bằng vấn đề business + kết quả đo được | "Dùng LangChain + Pinecone" là tín hiệu yếu; "giảm 40% latency" là tín hiệu mạnh *(field-guide)* |
| Một cách kể cho mọi người nghe | Tailor theo VP/PM/kỹ sư | Đọc sai audience = mất engagement |
| Độc thoại 10 phút liền mạch | Đối thoại: kể ngắn, mời hỏi, đào sâu theo dẫn dắt | Deep-dive senior là *dialogue, not monologue* *(field-guide)* |

### ④ Áp dụng thực tế + so sánh bigtech
- **Thực tế:** chuẩn bị **một elevator pitch 60 giây** cho mỗi dự án "tủ" (COMM-006) — bài toán + giải pháp + 1 con số impact. Đây là câu bạn bật ra khi interviewer nói "kể về dự án anh tự hào nhất".
- **Bigtech (verify):** ở Anthropic vòng deep-dive là **25' trình bày + 15–20' hỏi vặn** — tức bạn phải tự dựng cấu trúc kể trong 25', không ai dắt *(alexeygrigorev field-guide)*. Nhiều công ty (OpenAI, Apple, Discord, Anduril theo tổng hợp 2026) mở bằng "walk me through a project / your resume" — đúng kiểu cần opener 3 lớp. Khuyến nghị chung: **mỗi đáp ~60–90s, gắn ví dụ đã ship** *(lockedin.ai 2026)*.

### ⑤ Kịch bản mẫu (script) + artefact cần chuẩn bị
**Artefact bắt buộc:** 1 sơ đồ C4-Container (Bài 2) + 1 bảng "3 dòng impact" (số trước/sau).

**Template Opener (điền dự án của bạn):**
```
[Bài toán] "Hệ <tên> giải bài toán <X> cho <ai/quy mô>; trước đó <pain cũ>."
[Impact]   "Sau khi xây, <metric> cải thiện từ <A> → <B>."   ← front-load
[ToC]      "Em xin kể theo 3 trục: (1) kiến trúc tổng thể, (2) <quyết định khó nhất>,
            (3) <sự cố/đánh đổi đáng nhớ>. Anh muốn em đào sâu trục nào trước ạ?"
```
**Template "giữ mạch khi bị ngắt":**
```
"<trả lời thẳng câu vặn, 30–60s>. — Quay lại trục (2) em đang kể, thì..."
```
**Template "giải thích cho người không kỹ thuật trong 3 câu" (COMM-003):**
```
Câu 1: nó giúp AI gì cho người dùng (ẩn dụ đời thường).
Câu 2: phần khó là gì (1 hình ảnh, không thuật ngữ).
Câu 3: kết quả đo được.
```

### ⑥ Keywords cần nhớ (glossary)
- 🧠 **Cần HIỂU:** front-load impact · "table of contents" framing · dialogue-not-monologue · audience tailoring.
- 📌 **Cần THUỘC:** cấu trúc **STAR** (Situation-Task-Action-Result) · **CARL** (Context-Action-Result-Learning) · opener 3 lớp.
- 🛠️ **Cần LÀM ĐƯỢC:** đọc một dự án của bạn thành elevator pitch 60s · ứng biến đổi cách kể cho VP vs kỹ sư · trả lời câu chen ngang rồi quay về mạch.

### ⑦ Mental model
> **"Bản đồ trước, zoom sau; impact trước, cơ chế sau; đối thoại chứ không độc thoại."**

### ⑧ Câu hỏi phỏng vấn & thách đố (bám QB)
1. *(ARCH-001, ⭐⭐)* "Kể về hệ phức tạp nhất anh từng xây — mở đầu thế nào?" → **dò:** opener 3 lớp + front-load.
2. *(ARCH-004, ⭐⭐)* "Hệ này giải bài toán kinh doanh gì?" → **dò:** nối tech với mục tiêu business, không kể tech thuần.
3. *(ARCH-007, ⭐⭐⭐)* "Kể lại cho một VP nghe khác đi thế nào?" → **dò:** đổi nhấn sang impact/cost/rủi ro.
4. *(COMM-006, ⭐⭐⭐⭐)* "60 giây gây ấn tượng về dự án — anh nói gì?" → **dò:** nén bài toán+giải pháp+impact, front-load value.
5. *(COMM-004, ⭐⭐⭐⭐)* "Tôi ngắt giữa chừng hỏi sâu — anh giữ mạch ra sao?" → **dò:** trả lời thẳng rồi quay lại ToC, không lạc.
- 🎯 **Thách đố:** *"Anh vừa kể 2 phút toàn tên công nghệ mà tôi chưa nghe được vấn đề gì được giải. Kể lại trong 3 câu."* → **bẫy:** bạn đang lấy *cái dùng* làm *câu chuyện*. Đáp đúng = bài toán + giải pháp + 1 số impact, **không** một tên công nghệ nào.

### ⑨ Bài tập + tiêu chí tự chấm
- **BT1:** Viết opener 3 lớp cho dự án tủ của bạn (≤4 câu). **Đạt khi:** câu đầu là *bài toán* (không tech), có *1 con số impact*, có *ToC 2–3 trục*.
- **BT2:** Ghi âm bản pitch 60s, bấm giờ. **Đạt khi:** ≤75s, người nghe phi-kỹ-thuật tóm tắt lại đúng "hệ này để làm gì".

### ⑩ Đọc thêm / nguồn chuẩn
- alexeygrigorev — *ai-engineering-field-guide* → `interview/questions/03-project-deep-dive.md` (cấu trúc narrative, tín hiệu đánh giá).
- Khung STAR (gốc tuyển dụng hành vi) & CARL (biến thể nhấn *Learning*).

### 🧪 Tự kiểm (recall — che đáp án, tự nói trước)
1. Ba lớp của opener là gì? · 2. Khi nào dùng CARL thay STAR? · 3. Bị ngắt giữa chừng thì làm gì để không lạc?
<details><summary>Dò cái gì (mốc ĐẠT)</summary>

1. (i) bài toán+bối cảnh business · (ii) table-of-contents 2–3 trục · (iii) front-load impact.
2. Dự án **dài/phức tạp** — vì lớp **Learning** làm nổi seniority.
3. Trả lời thẳng câu vặn ~30–60s → câu cầu nối → quay lại trục đang kể (dialogue, không lạc).
</details>

---

## 📖 BÀI 2 — Vẽ & trình bày kiến trúc
*(Phủ: D-ARCH-002, D-ARCH-003, D-ARCH-005, D-ARCH-006, D-ARCH-010)*

### ① Mục tiêu & vị trí trong mạch
Sau khi có *khung kể* (Bài 1), bài này cho bạn **hình**: vẽ được kiến trúc ở **đúng mức zoom**, **trace một request end-to-end**, và **chọn một sơ đồ duy nhất** truyền tải cả hệ. Đây là artefact mà Bài 3–9 liên tục chỉ vào. Senior+ bị kỳ vọng *vẽ được trên whiteboard* và *mô tả rõ ràng* — nguồn 2026 ghi thẳng "expect architecture diagrams + war stories".

### ② Giảng: cơ bản → nâng cao

**(a) Trực giác.** Một sơ đồ tốt như *bản đồ tàu điện ngầm*: bỏ chi tiết không cần, giữ đúng các ga và tuyến. Sai lầm là vẽ *bản đồ địa hình* (mọi con hẻm) khi người ta chỉ cần biết đi từ A đến B.

**(b) Cơ chế — C4 model (📘 ARCH-003, ➕ làm sâu):** 4 mức zoom, từ xa đến gần:
| Mức | Trả lời câu hỏi | Vẽ gì |
|---|---|---|
| **C1 Context** | Hệ nằm ở đâu trong thế giới? | 1 hộp "hệ của tôi" + **actor** (user, admin) + **hệ ngoài** (payment gateway, email) |
| **C2 Container** | Hệ gồm những thứ *chạy độc lập* nào? | web app, API service, DB, cache, queue, worker — mỗi cái 1 hộp, có giao thức nối |
| **C3 Component** | Bên trong một container có module gì? | controller, service, repository... |
| **C4 Code** | Class/hàm cụ thể | hiếm khi vẽ trong phỏng vấn |

→ Trong deep-dive, **đa số thời gian bạn ở C1–C2**. C2 (Container) thường là *sơ đồ duy nhất đáng giữ* (ARCH-010) vì nó cho thấy thành phần + luồng + ranh giới triển khai cùng lúc.

- **Whiteboard 2 phút (📘 ARCH-002):** vẽ luồng `client → LB/gateway → service(s) → DB/cache/queue → hệ ngoài`, gọi đúng tên building block, **dừng ở mức container** — đừng sa vào tên class.
- **Trace một request (📘 ARCH-005):** đi end-to-end: `auth → validate → business logic → DB/cache → (async side-effects) → response`. Chỉ rõ chỗ **đồng bộ** (nằm trên critical path, người dùng phải chờ) vs **bất đồng bộ** (đẩy queue, trả response trước). Biết đâu là **critical path** — phần quyết định latency.

**(c) Mép giới hạn & sai lầm:**
- ❌ Nhảy thẳng C1 → C4 (code) khi bị hỏi "vẽ kiến trúc" → mất bức tranh lớn.
- ❌ Vẽ "đẹp" nhưng không gọi được tên giao thức/luồng dữ liệu → sơ đồ trang trí.
- **Điểm khó nhất kỹ thuật (📘 ARCH-006):** khi được hỏi, phải chỉ ra **constraint thật** (consistency mạnh + scale cao đồng thời; latency thời gian thực; legacy ràng buộc; lượng dữ liệu lớn), **không** trả lời "code nhiều/logic phức tạp". Đây là câu lộ chiều sâu nhận thức bài toán.

> **🧭 Hiểu để chỉ huy & kiểm tra:** AI vẽ được sơ đồ C4 chuẩn cú pháp. Nhưng **chọn mức zoom đúng cho câu hỏi đang được hỏi** (Context cho VP, Container cho TL) và **biết đâu là critical path thật trong hệ CỦA BẠN** là phán đoán người. Kiểm tra bản AI: nếu nó vẽ cả 4 mức cho câu "tổng quan 2 phút", đó là sai mức zoom.

### ③ ⚠️ Cũ → Mới

| Cũ | Mới | Vì sao |
|---|---|---|
| Sơ đồ "hộp và mũi tên" tùy hứng, không cấp độ | **C4 model** (Context/Container/Component/Code) | Có ngôn ngữ chung về *mức zoom*, chọn đúng cho audience |
| Vẽ mọi chi tiết trong 1 sơ đồ | Tách theo mức; giữ **C2 Container** làm sơ đồ chủ | Một sơ đồ quá tải = không ai đọc |
| Mô tả "request đi qua nhiều bước" chung chung | **Trace end-to-end + chỉ rõ sync vs async + critical path** | Lộ hiểu biết vận hành thật |

### ④ Áp dụng thực tế + bigtech
- **Thực tế:** chuẩn bị sẵn **1 sơ đồ C2** vẽ tay được trong 2 phút. Khi interviewer nói "vẽ cho tôi xem", bạn không dựng lại từ đầu.
- **Bigtech (verify):** nguồn 2026 nhất quán rằng ở senior/staff, interviewer **kỳ vọng sơ đồ trên whiteboard hoặc mô tả rõ + war story**, và *khoan sâu* sau đó *(TechEon 2026; field-guide)*. C4 model là chuẩn de-facto để giao tiếp kiến trúc (do Simon Brown đề xướng) — *(verify chi tiết tại c4model.com nếu cần trích)*.

### ⑤ Kịch bản mẫu + artefact
**Artefact bắt buộc:** vẽ trước sơ đồ **C2 Container** dự án bạn (draw.io/Excalidraw), in ra hoặc thuộc để vẽ tay 2'.
**Script trace request:**
```
"Request POST /order vào: (1) gateway xác thực JWT [sync],
 (2) service validate payload [sync],
 (3) ghi DB trong 1 transaction [sync, critical path],
 (4) publish event 'order.created' lên queue [async] → worker gửi email/cập nhật analytics,
 (5) trả 201 cho client.  → Critical path = (1)(2)(3); (4) không chặn response."
```

### ⑥ Keywords
- 🧠 **Cần HIỂU:** mức zoom (Context↔Container↔Component↔Code) · critical path · sync vs async trên luồng.
- 📌 **Cần THUỘC:** 4 chữ C của C4 và mỗi mức trả lời câu hỏi gì · building blocks chuẩn (LB, gateway, service, cache, queue, worker).
- 🛠️ **Cần LÀM ĐƯỢC:** vẽ C2 dự án bạn trong 2' · trace 1 request end-to-end nói rõ sync/async · nêu điểm khó kỹ thuật thật của hệ.

### ⑦ Mental model
> **"C4 = bốn mức zoom; mặc định ở Container; luôn chỉ được critical path."**

### ⑧ Câu hỏi & thách đố
1. *(ARCH-002, ⭐⭐⭐)* "Vẽ kiến trúc trong 2 phút." → **dò:** luồng client→…→DB/cache/queue, dừng ở container.
2. *(ARCH-003, ⭐⭐⭐)* "Mô tả ở mức Context vs Container." → **dò:** phân biệt actor+hệ ngoài (Context) vs app/DB/queue chạy độc lập (Container).
3. *(ARCH-005, ⭐⭐⭐)* "Trace một request." → **dò:** end-to-end, chỉ sync/async, biết critical path.
4. *(ARCH-006, ⭐⭐⭐⭐)* "Điểm khó nhất kỹ thuật, vì sao khó?" → **dò:** constraint thật (consistency/scale/latency/legacy), không "code nhiều".
5. *(ARCH-010, ⭐⭐⭐)* "Chỉ giữ 1 sơ đồ thì giữ cái nào?" → **dò:** chọn view nhiều thông tin nhất (thường Container/data-flow), hiểu sơ đồ là công cụ giao tiếp.
- 🎯 **Thách đố:** *"Anh vừa vẽ 12 hộp. Cái nào nằm trên critical path của request chính?"* → **bẫy:** vẽ được nhưng không phân biệt được phần chặn người dùng vs phần phụ trợ. Đáp đúng = khoanh đúng 2–4 hộp sync trên đường người dùng phải chờ.

### ⑨ Bài tập + tiêu chí
- **BT1:** Vẽ C2 dự án bạn, gắn nhãn giao thức trên mỗi mũi tên. **Đạt khi:** ≤ ~8 hộp, mỗi mũi tên có giao thức (HTTP/gRPC/SQL/AMQP…).
- **BT2:** Viết trace 1 request, đánh dấu [sync]/[async] và khoanh critical path. **Đạt khi:** chỉ ra đúng phần quyết định latency.

### ⑩ Đọc thêm
- C4 model — `c4model.com` (Simon Brown). · Công cụ vẽ: draw.io, Excalidraw.

### 🧪 Tự kiểm (recall)
1. C2 (Container) khác C1 (Context) ở đâu? · 2. Critical path là gì, sao cần khoanh? · 3. Hỏi "vẽ tổng quan 2 phút" thì dừng ở mức nào?
<details><summary>Dò cái gì</summary>

1. Context = hệ + actor + hệ ngoài; Container = các đơn vị *chạy độc lập* (app/DB/queue) + giao thức nối.
2. Chuỗi bước **đồng bộ** người dùng phải chờ → quyết định latency; khoanh để biết tối ưu/đo ở đâu.
3. Mức **Container (C2)**; không sa vào component/code.
</details>

---

## 📖 BÀI 3 — Biện minh "vì sao" mỗi lựa chọn
*(Phủ: D-ARCH-008, D-WHY-001, 002, 003, 004, 005, 007, 008, 009)*

### ① Mục tiêu & vị trí trong mạch
Đây là **lõi của deep-dive** — hầu hết follow-up rơi vào "vì sao chọn X". Học xong bạn biện minh được **mọi lựa chọn lớn** theo *access pattern + ràng buộc + bối cảnh*, không phải "quen tay" hay "mốt". Bài này dùng *hình* của Bài 2 làm bàn đạp.

### ② Giảng: cơ bản → nâng cao

**(a) Trực giác.** Mỗi lựa chọn kỹ thuật là một *câu trả lời cho một ràng buộc*. Interviewer không kiểm tra bạn "biết Postgres" — họ kiểm tra bạn **biết VÌ SAO Postgres ở ĐÂY**. Công thức trả lời mọi câu "vì sao": **"Vì <yêu cầu thật của hệ> nên cần <thuộc tính>, mà <lựa chọn> cho thuộc tính đó tốt nhất trong <ràng buộc của tụi em>."**

**(b) Cơ chế — bộ "vì sao" theo từng quyết định (📘):**

- **Ranh giới service (ARCH-008):** chia theo **bounded context / domain / độ biến động**, KHÔNG theo tầng kỹ thuật (đừng tách "service-controller / service-DB"). *(cơ chế decomposition chi tiết → giao mục A.)*
- **DB choice (WHY-001):** lý do theo **access pattern** (đọc/ghi, quan hệ hay document, truy vấn ad-hoc?) + **yêu cầu nhất quán** + **năng lực team**. *(SQL vs NoSQL sâu → giao mục E.)*
- **Monolith vs microservices (WHY-002):** biện minh theo **quy mô team / độ phức tạp / cách deploy**; ý thức **chi phí vận hành** của micro (network, observability, dữ liệu phân tán). Không chọn micro vì "mốt". Phải trả lời được cả "nếu chọn ngược lại thì sao".
- **Message queue (WHY-003):** để **decouple / buffer / retry / async**; bỏ nó → coupling chặt, mất request lúc tải cao. *(cơ chế queue → giao mục K.)*
- **Cache (WHY-004):** chọn **đúng layer** (client / CDN / app / DB) + **chiến lược invalidation** rõ; cache sai → **stale data**. *(chi tiết → giao mục J.)*
- **Framework/ngôn ngữ (WHY-005):** phân biệt **ràng buộc kế thừa** (đã có sẵn) vs **lựa chọn chủ động** — trung thực, đừng nhận vơ mọi quyết định.
- **Sync vs async (WHY-007):** **critical path → sync**; **side-effect (email/log/analytics) → async**. Đánh đổi latency vs reliability.
- **Transaction / idempotency (WHY-008):** transaction ở nơi cần **atomicity**; idempotency ở nơi chống **double-processing** (retry, double-click). Gắn vào endpoint cụ thể. *(→ giao E/I/K.)*
- **Read/write split & replica (WHY-009):** đọc từ replica để **scale đọc**, nhưng coi chừng **replication lag** (đọc dữ liệu cũ); biện minh theo tỉ lệ đọc/ghi thật. *(→ giao E.)*

**(c) Mép giới hạn & sai lầm:**
- ❌ "Em chọn Mongo vì nó nhanh/hiện đại" → không nối với access pattern = trả lời rỗng.
- ❌ Nhận mọi quyết định là của mình trong khi thực ra kế thừa → mất điểm chính trực (Bài 7, 9 sẽ vặn lại).
- ❌ Đặt side-effect (gửi email) vào critical path đồng bộ → latency tăng, một service email chết kéo sập cả request.

> **🧭 Hiểu để chỉ huy & kiểm tra:** *(ví dụ kinh điển của roadmap)* — giao chatbot trả lời **tài liệu chính sách hay đổi**, AI/người thiếu kinh nghiệm chọn **fine-tune** (sai kiến trúc: dữ liệu đổi là phải train lại). Người **hiểu** sẽ **chỉ huy dùng RAG** ngay từ đầu. Cú pháp cả hai đều "chạy", nhưng *hiểu mới định hình đúng yêu cầu*. Trong deep-dive, đây đúng kiểu câu "vì sao chọn X" mà bạn phải bảo vệ.

### ③ ⚠️ Cũ → Mới (lối biện minh)

| Cũ | Mới | Vì sao |
|---|---|---|
| "Chọn vì quen / vì phổ biến / vì mới" | "Chọn vì access pattern + ràng buộc + năng lực team" | Lý do business/kỹ thuật mới chịu được vặn |
| Micro vì "mốt", mọi thứ thành service | Mặc định đơn giản; tách service khi có **lý do vận hành/team** thật | Micro có chi phí ẩn lớn |
| Side-effect chạy đồng bộ cho "chắc" | Critical path sync, side-effect async qua queue | Giảm latency + cô lập lỗi |
| Ép nhất quán mạnh mọi nơi | Chọn consistency theo từng phần (order vs feed) | Không có "một size vừa tất cả" |

### ④ Áp dụng thực tế + bigtech
- **Thực tế:** lập **"bảng quyết định"** — mỗi hàng: *quyết định · ràng buộc lúc đó · phương án loại · lý do*. Đây là vũ khí chống mọi câu "vì sao".
- **Bigtech (verify):** field-guide ghi rõ tín hiệu seniority là *bạn dri ve quyết định hay chỉ thực thi* — nên mỗi "vì sao" nên kèm "đó là quyết định của em vì…" *(field-guide)*. Pattern phổ biến (không khẳng định tuyệt đối): các hệ quy mô lớn mặc định **sync cho ghi giao dịch, async qua queue cho fan-out/side-effect**.

### ⑤ Kịch bản mẫu + artefact
**Artefact:** "Bảng quyết định" 5–7 hàng cho dự án bạn.
**Template trả lời "vì sao chọn X":**
```
"Yêu cầu của hệ là <truy vấn quan hệ phức tạp + nhất quán mạnh cho tiền>,
 nên em cần <ACID + join>. Postgres cho cái đó tốt và team đã vững.
 Em có cân nhắc <Mongo> nhưng loại vì <quan hệ nhiều, cần transaction đa bảng>.
 // các tên/đời/giá cụ thể: verify tại docs chính thức trước khi khẳng định"
```

### ⑥ Keywords
- 🧠 **Cần HIỂU:** access pattern · bounded context · decoupling/buffering · critical path vs side-effect · replication lag · idempotency.
- 📌 **Cần THUỘC:** công thức "vì <yêu cầu> nên cần <thuộc tính>, <lựa chọn> tốt nhất trong <ràng buộc>".
- 🛠️ **Cần LÀM ĐƯỢC:** biện minh DB/mono-micro/queue/cache/sync-async của hệ bạn + nêu phương án đã loại.

### ⑦ Mental model
> **"Mỗi lựa chọn = câu trả lời cho một ràng buộc; nói được ràng buộc + phương án đã loại."**

### ⑧ Câu hỏi & thách đố
1. *(WHY-001, ⭐⭐⭐)* "Vì sao chọn DB đó?" → **dò:** access pattern + nhất quán + team, không "quen tay".
2. *(WHY-002, ⭐⭐⭐⭐)* "Mono hay micro? Ngược lại thì sao?" → **dò:** theo team/độ phức tạp/deploy + ý thức chi phí vận hành.
3. *(WHY-003, ⭐⭐⭐)* "Vì sao có queue? Bỏ đi hỏng gì?" → **dò:** decouple/buffer/retry; bỏ đi → coupling + mất request lúc tải cao.
4. *(WHY-004, ⭐⭐⭐)* "Cache cái gì, ở đâu, invalidation ra sao?" → **dò:** đúng layer + chiến lược invalidation; rủi ro stale.
5. *(WHY-007, ⭐⭐⭐)* "Vì sao cái này sync, cái kia async?" → **dò:** critical path vs side-effect.
- 🎯 **Thách đố:** *(WHY-008/009)* "Anh nói đọc từ replica để scale. Vậy ngay sau khi user POST xong rồi GET lại, họ có thấy dữ liệu mình vừa ghi không?"* → **bẫy:** replication lag → có thể KHÔNG thấy (read-your-own-write). Đáp đúng = thừa nhận lag + cách xử lý (đọc từ primary cho luồng read-after-write, hoặc sticky/session consistency).

### ⑨ Bài tập + tiêu chí
- **BT1:** Điền "bảng quyết định" 5 hàng. **Đạt khi:** mỗi hàng có *phương án đã loại + lý do gắn ràng buộc thật*.
- **BT2:** Chọn 1 endpoint, liệt kê chỗ nào sync/async/transaction/idempotency và vì sao. **Đạt khi:** transaction nằm đúng chỗ cần atomicity, side-effect nằm async.

### ⑩ Đọc thêm
- Mục **E** (DB), **J** (Caching), **K** (Messaging), **I** (Concurrency) trong project — cơ chế engine phía sau từng "vì sao".

### 🧪 Tự kiểm (recall)
1. Công thức trả lời "vì sao chọn X"? · 2. Cái gì nên async, cái gì phải sync? · 3. Đọc replica có rủi ro gì?
<details><summary>Dò cái gì</summary>

1. "Vì <yêu cầu thật> nên cần <thuộc tính>, <lựa chọn> đáp ứng tốt nhất trong <ràng buộc>" + nêu phương án đã loại.
2. Sync = critical path (người dùng phải chờ, ghi giao dịch); async = side-effect (email/log/analytics) qua queue.
3. **Replication lag** → đọc dữ liệu cũ; xử lý read-after-write bằng đọc primary/sticky.
</details>

---

## 📖 BÀI 4 — Trade-off & quyết định dưới ràng buộc
*(Phủ: D-ARCH-009, D-WHY-010, D-TRADE-001…008)*

### ① Mục tiêu & vị trí trong mạch
Bài 3 dạy *biện minh một lựa chọn*; bài này lên tầng cao hơn — **so sánh phương án, nêu cái đã loại, và biện minh quyết định "tệ hơn về kỹ thuật nhưng đúng trong bối cảnh"**. Đây là nơi interviewer phân biệt người *làm theo công thức* với người *ra quyết định*.

### ② Giảng: cơ bản → nâng cao

**(a) Trực giác.** Không có quyết định kiến trúc "đúng tuyệt đối" — chỉ có *đánh đổi phù hợp với ràng buộc*. Câu trả lời senior luôn có hình "em chọn A **đánh đổi** B, vì lúc đó <ràng buộc> quan trọng hơn".

**(b) Cơ chế — các trục trade-off (📘):**
- **Alternatives + tiêu chí loại (TRADE-001):** luôn nêu được **ít nhất 1 phương án khác** đã cân nhắc + **vì sao loại**. "Chỉ có một cách" = red flag. *(articulate trade-off → giao A.)*
- **Ràng buộc định hình quyết định (TRADE-002):** nối **thời gian/người/tiền/legacy** với lựa chọn — ra quyết định *trong môi trường có giới hạn*, không phải phòng lab.
- **Pragmatism (TRADE-003):** dám chọn "giải pháp xấu hơn về kỹ thuật" để **ship đúng hạn / dễ bảo trì**, và biện minh được vì sao đúng *trong bối cảnh*.
- **Build vs buy (TRADE-004):** nhận **chi phí ẩn** của tự viết (bảo trì, edge case) vs **khóa nhà cung cấp** khi mua.
- **CAP áp dụng cụ thể (TRADE-005):** chọn consistency vs availability **theo từng phần** — vd `order = consistency`, `feed = availability`. *(→ giao I.)*
- **Sai theo hindsight vs sai khi quyết (TRADE-006):** công bằng với quá khứ — quyết định *đúng tại thời điểm đó* với thông tin có lúc đó không phải là "ngu".
- **Tech debt có chủ đích (TRADE-007):** cố ý tạo nợ để kịp deadline là OK **nếu có kế hoạch trả** (ticket, lịch refactor). *(→ giao C/Q.)*
- **Ưu tiên dưới sức ép (TRADE-008):** cắt 50% scope mà vẫn giữ lõi giá trị — biết đâu là **lõi** vs **phụ**.
- **Đơn giản vs phức tạp có chủ đích (ARCH-009):** nhận ra over/under-engineering; biện minh nơi đầu tư phức tạp là *đáng giá*, nơi cố tình giữ đơn giản cho dễ bảo trì.
- **Quyết định tự tin nhất + bằng chứng (WHY-010):** gắn với **dữ liệu đo được** (latency↓, sự cố↓, cost↓), không cảm tính → cầu nối sang Bài 5.

**(c) Mép giới hạn & sai lầm:**
- ❌ "Em không đánh đổi gì, giải pháp của em tối ưu mọi mặt" → bất khả tín.
- ❌ Tự đánh mình vì quyết định cũ ("hồi đó em ngu") → thiếu khung *sai-theo-hindsight*.
- ❌ Nợ kỹ thuật "vô tình" không ai theo dõi → khác hẳn nợ *có chủ đích + có kế hoạch trả*.

> **🧭 Hiểu để chỉ huy & kiểm tra:** AI thường đề xuất giải pháp "đúng sách" (microservices, event sourcing…) mà **bỏ qua ràng buộc team/thời gian** của bạn. Người hiểu sẽ **chỉ huy chọn phương án "xấu hơn" nhưng ship được** — và **kiểm tra** đề xuất AI bằng câu: "giải pháp này tốn bao nhiêu chi phí vận hành mà team 3 người của tôi gánh được không?".

### ③ ⚠️ Cũ → Mới

| Cũ | Mới | Vì sao |
|---|---|---|
| "Giải pháp của em là tốt nhất" | "Em chọn A, đánh đổi B, vì ràng buộc C" | Trung thực về trade-off = tín hiệu senior |
| Chỉ kể phương án đã chọn | Nêu **alternative + tiêu chí loại** | Cho thấy đã cân nhắc có ý thức |
| Né tech debt / giả vờ code luôn sạch | Thừa nhận **nợ có chủ đích + kế hoạch trả** | Thực tế, có kiểm soát |
| Tự trách quyết định cũ | Phân biệt **sai-khi-quyết vs sai-theo-hindsight** | Công bằng + phản tư đúng |

### ④ Áp dụng thực tế + bigtech
- **Thực tế:** với mỗi quyết định lớn, chuẩn bị bộ ba **(phương án đã loại · ràng buộc · bằng chứng kết quả)**.
- **Bigtech (verify):** nguồn 2026 nhấn interviewer senior **khoan vào trade-off và "what went wrong last time"**, kỳ vọng *honest acknowledgment of what we still don't know* *(TechEon 2026)* — tức nói thẳng trade-off được điểm, giấu thì mất.

### ⑤ Kịch bản mẫu + artefact
**Artefact:** danh sách 3 "quyết định lớn" + bảng trade-off mỗi cái.
**Template:**
```
"Quyết định: <X>. Phương án khác: <Y, Z>. Em loại Y vì <…>, Z vì <…>.
 Ràng buộc chi phối: <deadline/3 người/legacy>.
 Bằng chứng đúng: <metric đo được>.  // số liệu: verify từ APM/log thật, đừng bịa"
```

### ⑥ Keywords
- 🧠 **Cần HIỂU:** trade-off articulation · build-vs-buy + chi phí ẩn · CAP theo phần · sai-theo-hindsight · over/under-engineering.
- 📌 **Cần THUỘC:** "chọn A đánh đổi B vì ràng buộc C" · định nghĩa tech debt có chủ đích.
- 🛠️ **Cần LÀM ĐƯỢC:** với 1 quyết định nêu alternative + tiêu chí loại + bằng chứng · chỉ ra 1 nợ kỹ thuật có chủ đích + cách trả.

### ⑦ Mental model
> **"Không có lựa chọn đúng — chỉ có đánh đổi phù hợp ràng buộc; luôn nói được cái đã loại."**

### ⑧ Câu hỏi & thách đố
1. *(TRADE-001, ⭐⭐⭐⭐)* "Quyết định lớn nhất — cân nhắc phương án nào khác, vì sao loại?" → **dò:** alternative thật + tiêu chí loại.
2. *(TRADE-003, ⭐⭐⭐⭐)* "Có chỗ nào chọn giải pháp tệ hơn vì lý do thực tế?" → **dò:** pragmatism + biện minh trong bối cảnh.
3. *(TRADE-005, ⭐⭐⭐)* "Đánh đổi consistency vs availability ở đâu?" → **dò:** CAP áp dụng từng phần, chọn có chủ đích.
4. *(TRADE-007, ⭐⭐⭐)* "Tech debt cố ý nào? Quản lý sau đó ra sao?" → **dò:** nợ có chủ đích + kế hoạch trả.
5. *(WHY-010, ⭐⭐⭐⭐)* "Quyết định tự tin nhất là đúng? Lấy gì chứng minh?" → **dò:** gắn dữ liệu đo được.
- 🎯 **Thách đố:** *(TRADE-008, ⭐⭐⭐⭐⭐)* "Cắt 50% scope mà vẫn giữ giá trị cốt lõi — anh cắt gì?"* → **bẫy:** trả lời "không cắt được, mọi phần đều quan trọng". Đáp đúng = phân biệt **lõi tạo giá trị** vs **phụ trợ**, dám hi sinh phụ.

### ⑨ Bài tập + tiêu chí
- **BT1:** Bảng trade-off cho 2 quyết định lớn. **Đạt khi:** mỗi cái có alternative + ràng buộc + (nếu có) số liệu.
- **BT2:** Viết 1 đoạn "quyết định đúng-lúc-đó nhưng giờ sai" theo khung hindsight. **Đạt khi:** không tự trách, nêu rõ thông tin lúc đó vs bây giờ.

### ⑩ Đọc thêm
- Mục **A** (System Design — trade-off chung), **I** (CAP), **C/Q** (tech debt, design) trong project.

### 🧪 Tự kiểm (recall)
1. Tín hiệu red flag khi nói về quyết định? · 2. "Sai-theo-hindsight" khác "sai-khi-quyết" thế nào? · 3. Tech debt có chủ đích cần đi kèm gì?
<details><summary>Dò cái gì</summary>

1. "Không đánh đổi gì / chỉ có một cách / giải pháp em tối ưu mọi mặt".
2. Hindsight = giờ mới biết là sai; sai-khi-quyết = lẽ ra biết sai với thông tin lúc đó. Công bằng với quá khứ.
3. **Kế hoạch trả** (ticket/lịch refactor) — không phải nợ vô tình bị bỏ quên.
</details>

---

## 📖 BÀI 5 — Số liệu & quy mô thật
*(Phủ: D-SCALE-001…009)*

### ① Mục tiêu & vị trí trong mạch
Đây là **bằng chứng** biến mọi câu chuyện ở Bài 1–4 từ "nghe hợp lý" thành "đã làm thật". Học xong bạn nói được **con số cụ thể** (QPS, data size, latency percentile, cost, SLO), biết **nguồn số**, và nhận diện **bottleneck bằng đo, không đoán**. Field-guide nhấn: ứng viên giỏi đóng khung quanh *con số impact*.

### ② Giảng: cơ bản → nâng cao

**(a) Trực giác.** "Rất nhiều user" là vô nghĩa; "~5k DAU, peak 800 RPS lúc 8h tối" là tín hiệu bạn *đã vận hành thật*. Số là dấu vân tay của kinh nghiệm.

**(b) Cơ chế — các loại số phải nắm (📘):**
- **Tải (SCALE-001/002):** DAU/MAU, **QPS/RPS**, phân biệt **peak vs trung bình** + biết peak xảy ra khi nào; biết **nguồn số** (metrics/APM/load test) — không bịa.
- **Dữ liệu (SCALE-003):** **data size + growth rate**; nối với quyết định archive/shard/retention. *(→ giao E.)*
- **Latency percentile (SCALE-004):** dùng **p50/p95/p99**, KHÔNG chỉ trung bình. Hiểu **tail latency** (p99 cao do GC, lock contention, cold cache, N+1 query, network). *(→ giao O.)*
  > Vì sao percentile? Trung bình giấu đuôi: 1% request chậm 5s vẫn cho "trung bình đẹp", nhưng đó là 1% user khốn khổ. p99 mới phản ánh trải nghiệm tệ nhất phổ biến.
- **Bottleneck (SCALE-005):** xác định bằng **profiling/trace/metrics**, không đoán. *(→ giao A.)*
- **Headroom & 10× (SCALE-006):** biết **chỗ vỡ trước** khi tải ×10 + kế hoạch cụ thể (scale ngang, cache, shard, queue), không "thì scale lên".
- **SLA vs SLO (SCALE-007):** **SLA** = cam kết với khách (có hậu quả pháp lý/tiền); **SLO** = mục tiêu nội bộ; biết target uptime/latency thật + mức đạt. *(→ giao O.)*
- **Cost (SCALE-008):** **cost awareness** (compute/DB/egress/3rd-party), khoản nào tốn nhất — tối ưu chi phí là việc của TL.
- **Số gây bất ngờ (SCALE-009):** kể được khoảng cách **giả định vs thực đo** → chứng minh *thói quen đo lường*.

**(c) Mép giới hạn & sai lầm:**
- ❌ Báo "trung bình 200ms" mà không có p95/p99 → giấu tail.
- ❌ Đưa số tròn trịa quá đẹp, không nói nguồn → nghi bịa (Bài 9 sẽ vặn "anh nhớ chính xác hay đang ước?").
- ❌ "Nếu tải tăng thì scale lên" → không có headroom plan cụ thể.

> **🧭 Hiểu để chỉ huy & kiểm tra:** AI hay đề xuất tối ưu *trung bình*. Người hiểu **kiểm tra theo p99** — vì cú pháp "giảm latency" có thể đúng mà vẫn để 1% user chịu 3s. Biết hỏi "p99 là bao nhiêu?" là tư duy kiểm tra của TL.

### ③ ⚠️ Cũ → Mới

| Cũ | Mới | Vì sao |
|---|---|---|
| "Rất nhiều / khá lớn" | Con số cụ thể + nguồn (APM/load test) | Số = bằng chứng kinh nghiệm thật |
| Chỉ báo latency trung bình | **p50/p95/p99** + giải thích tail | Trung bình giấu trải nghiệm tệ |
| "Tải tăng thì scale lên" | Chỉ rõ **chỗ vỡ trước + plan ×10** | Cho thấy đã nghĩ về headroom |
| Lẫn SLA và SLO | Phân biệt cam kết-khách vs mục tiêu-nội-bộ | Khác nhau về hậu quả |

### ④ Áp dụng thực tế + bigtech
- **Thực tế:** chuẩn bị **"bảng số liệu 1 trang"**: tải (avg/peak), data size + growth, p50/p95/p99 endpoint chính, SLO + mức đạt, cost/tháng + khoản tốn nhất. Gắn nhãn *(đo / ước)* cho từng số.
- **Bigtech (verify):** nguồn 2026 lặp lại "know every decision and **metric**", "frame around impact (reduced response time 40%)" *(Adil Shamim 2026; field-guide)*. Pattern phổ biến: theo dõi p99 + error rate + cost/request là bộ ba vận hành cơ bản.

### ⑤ Kịch bản mẫu + artefact
**Artefact bắt buộc:** bảng số liệu 1 trang (đính kèm sơ đồ C2 của Bài 2).
```
Tải:      ~<DAU> DAU, avg <X> RPS, peak <Y> RPS (lúc <khi nào>)   [nguồn: APM]
Data:     <size> hiện tại, +<growth>/tháng
Latency:  p50 <a>ms / p95 <b>ms / p99 <c>ms (endpoint chính)      [nguồn: trace]
SLO:      uptime <%>, p95 < <ms> — đạt <bao nhiêu %>
Cost:     ~$<n>/tháng, tốn nhất: <khoản>
*Số đo thật vs ước: gắn nhãn "~" cho ước lượng.
```

### ⑥ Keywords
- 🧠 **Cần HIỂU:** percentile & tail latency · peak vs avg · headroom · SLA vs SLO · cost awareness.
- 📌 **Cần THUỘC:** ý nghĩa p50/p95/p99 · nguyên nhân p99 cao (GC, lock, cold cache, N+1, network).
- 🛠️ **Cần LÀM ĐƯỢC:** đọc số hệ bạn có nhãn nguồn · chỉ bottleneck đã đo · vẽ plan tải ×10.

### ⑦ Mental model
> **"Số có nguồn, latency theo percentile, scale theo headroom — không đoán, không tròn trịa giả."**

### ⑧ Câu hỏi & thách đố
1. *(SCALE-001, ⭐⭐⭐)* "Bao nhiêu user/request? Số thật." → **dò:** DAU/MAU, QPS, peak vs avg.
2. *(SCALE-004, ⭐⭐⭐⭐)* "p50/p95/p99 endpoint chính? Vì sao p99 cao hơn nhiều?" → **dò:** dùng percentile + giải thích tail.
3. *(SCALE-005, ⭐⭐⭐⭐)* "Bottleneck thật ở đâu? Biết bằng cách nào?" → **dò:** xác định bằng đo (profiling/trace).
4. *(SCALE-007, ⭐⭐⭐)* "SLA/SLO là gì? Đạt không?" → **dò:** phân biệt + số thật.
5. *(SCALE-008, ⭐⭐⭐)* "Chi phí vận hành? Khoản nào tốn nhất?" → **dò:** cost awareness.
- 🎯 **Thách đố:** *(SCALE-006, ⭐⭐⭐⭐⭐)* "Tải ×10 thì chỗ nào vỡ TRƯỚC?"* → **bẫy:** "thì scale server lên". Đáp đúng = chỉ **một** điểm nghẽn cụ thể (vd connection pool DB / single writer / cache miss storm) + cách đã/sẽ chuẩn bị.

### ⑨ Bài tập + tiêu chí
- **BT1:** Hoàn thành bảng số liệu 1 trang. **Đạt khi:** có percentile (không chỉ avg), mỗi số có nhãn nguồn hoặc "~".
- **BT2:** Viết 3 câu giải thích vì sao p99 endpoint chính cao. **Đạt khi:** nêu nguyên nhân cụ thể đã quan sát (không lý thuyết suông).

### ⑩ Đọc thêm
- Mục **O** (Observability — percentile, SLO, tracing), **E** (sharding/retention), **A** (bottleneck) trong project.

### 🧪 Tự kiểm (recall)
1. Vì sao báo p99 thay vì trung bình? · 2. SLA khác SLO ra sao? · 3. Trả lời "tải ×10 vỡ ở đâu" thế nào cho đạt?
<details><summary>Dò cái gì</summary>

1. Trung bình giấu **tail**; p99 phản ánh 1% user tệ nhất → đúng trải nghiệm thật.
2. SLA = cam kết với khách (có hậu quả); SLO = mục tiêu nội bộ.
3. Chỉ **một điểm nghẽn cụ thể** vỡ trước + plan (cache/shard/queue/scale ngang), không "thì scale lên".
</details>

---

## 📖 BÀI 6 — Sự cố production, debug & on-call
*(Phủ: D-PROB-001…009, D-WHY-006)*

### ① Mục tiêu & vị trí trong mạch
Đây là **war story** — thứ AI/blog không có. Học xong bạn kể được **incident thật** (triệu chứng → impact → timeline → recover → phòng tái diễn), trình bày **phương pháp debug có hệ thống**, và một **bug do eventual consistency / race condition** mà người dùng nhìn thấy. Cần số liệu Bài 5 để incident đáng tin.

### ② Giảng: cơ bản → nâng cao

**(a) Trực giác.** Interviewer không tìm "hệ của anh chưa bao giờ sập" (bất khả tín) — họ tìm *bạn phản ứng thế nào khi cháy nhà*. Một incident kể tốt cho thấy **bình tĩnh, phương pháp, học hỏi**.

**(b) Cơ chế — khung kể incident (📘):**
- **Sự cố tệ nhất (PROB-001):** triệu chứng + **impact (ai/bao nhiêu bị ảnh hưởng)** + timeline; không né, không đổ hết lỗi.
- **Debug có hệ thống (PROB-002):** `repro → bisect → đọc log/metric/trace → giả thuyết → kiểm chứng` — KHÔNG "may mắn tìm ra".
- **Recover & MTTR (PROB-003):** quy trình **rollback / failover / hotfix / feature flag** + ý thức **MTTR** (mean time to recovery). *(→ giao N/O.)*
- **Phòng tái diễn (PROB-004):** **postmortem blameless + action item cụ thể** (alert mới, test hồi quy, guardrail) — học từ lỗi, không chỉ vá nóng.
- **Data integrity (PROB-005):** dữ liệu sai/hỏng thường **nguy hiểm hơn downtime**; backfill/reconcile.
- **Dev/staging ổn nhưng prod vỡ (PROB-006):** khác biệt **tải thật / data thật / concurrency / config / secret**. *(→ giao L/N.)*
- **Race condition (PROB-007):** concurrency bug thật (lost update, double-spend, double-charge); fix bằng **lock / idempotency / version (optimistic)**. *(→ giao I.)*
- **On-call (PROB-008):** alert nào kêu nhất, giảm **noise/fatigue**, có **runbook**. *(→ giao O.)*
- **Quyết định khi cháy nhà (PROB-009):** ra quyết định **dưới bất định** — *mitigate trước, root-cause sau*; rủi ro có kiểm soát. *(→ giao B.)*
- **Eventual consistency user-visible bug (WHY-006, ⭐⭐⭐⭐⭐):** câu chuyện THẬT trong hệ bạn OWN — vd: user cập nhật hồ sơ rồi reload thấy dữ liệu cũ (đọc replica chưa kịp đồng bộ); phát hiện qua report của user + log; sửa bằng đọc primary cho luồng đó. Đây chính là *ranh giới đọc-vs-làm*.

**(c) Mép giới hạn & sai lầm:**
- ❌ "Hệ em chưa từng có sự cố" → đỏ cờ ngay.
- ❌ Đổ toàn bộ cho người khác / hạ tầng → thiếu ownership.
- ❌ Kể "tìm ra lỗi nhờ may mắn" → thiếu phương pháp.
- ❌ Postmortem chỉ trách người (blameful) → văn hóa xấu, không học được.

> **🧭 Hiểu để chỉ huy & kiểm tra:** *(ví dụ kinh điển roadmap)* — AI viết code phân loại để `temperature=0.7`, **chạy không lỗi nhưng cùng input ra khác nhau**. Chỉ người hiểu **decoding** mới bắt được và sửa `temperature=0`. *Cú pháp đúng, hành vi sai* — đúng kiểu bug "chạy ổn ở dev nhưng sai ở prod" (PROB-006) mà chỉ con người hiểu mới bắt.

### ③ ⚠️ Cũ → Mới (cách xử lý/kể sự cố)

| Cũ | Mới | Vì sao |
|---|---|---|
| Giấu sự cố / "chưa bao giờ sập" | Kể incident thật + cách phản ứng | Bất khả tín mới là red flag |
| Debug "thử cho tới khi hết lỗi" | Phương pháp: repro→bisect→log/trace→giả thuyết→kiểm chứng | Lộ tư duy có hệ thống |
| Postmortem đổ lỗi cá nhân | **Blameless postmortem + action item** | Học từ lỗi, không lặp |
| Fix race bằng "thêm sleep/retry bừa" | **lock / idempotency / optimistic version** | Đúng gốc concurrency |

### ④ Áp dụng thực tế + bigtech
- **Thực tế:** chuẩn bị **2 incident story** theo khung *Triệu chứng → Khoanh vùng → Root cause → Fix → MTTR → Action item*.
- **Bigtech (verify):** senior/staff bị khoan vào **failure modes + "what went wrong last time"**, kỳ vọng **war story thật** *(TechEon 2026)*; behavioral 2026 hỏi thẳng "describe a time you reduced hallucinations/cost in production" và "a safety-first decision" *(Adil Shamim 2026)*. Pattern phổ biến: blameless postmortem (phổ biến hóa bởi Google SRE) + runbook + feature flag để rollback nhanh.

### ⑤ Kịch bản mẫu + artefact
**Artefact:** 2 incident card theo khung dưới.
```
INCIDENT: <tên>
Triệu chứng:  <user thấy gì / alert gì> — Impact: <ai, bao nhiêu, bao lâu>
Khoanh vùng:  repro → bisect → log/metric/trace cho thấy <…>
Root cause:   <nguyên nhân gốc>
Fix tức thời: <rollback/flag/hotfix> — MTTR ~<phút>
Phòng tái diễn (action item): <alert mới / test hồi quy / guardrail>
```

### ⑥ Keywords
- 🧠 **Cần HIỂU:** MTTR · blameless postmortem · tail vs systemic failure · read-your-own-write bug · alert fatigue.
- 📌 **Cần THUỘC:** trình tự debug (repro→bisect→trace→giả thuyết→kiểm chứng) · 3 cách fix race (lock/idempotency/version).
- 🛠️ **Cần LÀM ĐƯỢC:** kể 1 incident đủ khung · kể 1 bug eventual-consistency user-visible thật · mô tả quy trình recover + MTTR.

### ⑦ Mental model
> **"Không ai hỏi 'có sập không' — họ hỏi 'cháy nhà anh làm gì': mitigate trước, root-cause sau, postmortem blameless."**

### ⑧ Câu hỏi & thách đố
1. *(PROB-001, ⭐⭐⭐⭐)* "Sự cố tệ nhất — chuyện gì xảy ra?" → **dò:** triệu chứng + impact + timeline, không né.
2. *(PROB-003, ⭐⭐⭐⭐)* "Khôi phục thế nào? MTTR bao lâu?" → **dò:** rollback/failover/flag + ý thức MTTR.
3. *(PROB-004, ⭐⭐⭐)* "Làm gì để không lặp lại?" → **dò:** blameless postmortem + action item cụ thể.
4. *(PROB-008, ⭐⭐⭐)* "On-call ra sao? Alert nào hay kêu?" → **dò:** giảm noise/fatigue + runbook.
5. *(WHY-006, ⭐⭐⭐⭐⭐)* "Một bug NGƯỜI DÙNG thấy do eventual consistency, trong hệ anh OWN." → **dò:** câu chuyện thật: triệu chứng+phát hiện+sửa.
- 🎯 **Thách đố:** *(PROB-002 / PROB-007, ⭐⭐⭐⭐⭐)* "Kể bug khó nhất do RACE CONDITION — anh khoanh vùng thế nào?"* → **bẫy:** kể chung chung "có lỗi concurrency, em thêm lock". Đáp đúng = triệu chứng cụ thể (vd double-charge), cách repro/quan sát (log trùng, version conflict), fix đúng gốc (idempotency key / optimistic lock) + xác nhận hết.

### ⑨ Bài tập + tiêu chí
- **BT1:** Viết 2 incident card đủ khung. **Đạt khi:** mỗi card có *root cause + action item phòng tái diễn* (không dừng ở "fix xong").
- **BT2:** Viết lại 1 bug eventual-consistency/race của bạn. **Đạt khi:** có triệu chứng người-dùng-thấy + cách phát hiện + fix đúng gốc.

### ⑩ Đọc thêm
- Mục **I** (Concurrency — race, lock, idempotency), **O** (alert, MTTR), **N** (deploy/rollback) trong project. · Google SRE Book — chương Postmortem (*verify*).

### 🧪 Tự kiểm (recall)
1. Trình tự debug có hệ thống? · 2. "Blameless postmortem" nghĩa là gì + đi kèm gì? · 3. Ba cách fix race condition?
<details><summary>Dò cái gì</summary>

1. repro → bisect → đọc log/metric/trace → giả thuyết → kiểm chứng.
2. Không trách cá nhân, tập trung hệ thống/quy trình; **đi kèm action item cụ thể** (alert/test/guardrail).
3. Lock (pessimistic) · idempotency key · optimistic version.
</details>

---

## 📖 BÀI 7 — Đóng góp cá nhân ("I vs we"), ownership
*(Phủ: D-ROLE-001…009, D-COMM-005)*

### ① Mục tiêu & vị trí trong mạch
Mọi câu chuyện ở Bài 1–6 vô nghĩa nếu interviewer không biết **phần nào là CỦA BẠN**. Đây là tín hiệu seniority *quan trọng nhất* (field-guide: narrative tiết lộ bạn *drive hay execute*). Học xong bạn dùng "tôi" đúng chỗ, **không nhận vơ**, mô tả scope chính xác, và quy công công bằng.

### ② Giảng: cơ bản → nâng cao

**(a) Trực giác.** "Chúng tôi xây hệ X" cho biết *team làm gì*, không cho biết *bạn làm gì*. Interviewer phải nghe được **"tôi quyết / tôi xây / tôi sửa"** thì mới chấm được bạn ở mức nào. QB nêu chuẩn: **~80% lời kể nên là "I"**.

**(b) Cơ chế (📘):**
- **I vs we (ROLE-001):** tách rõ phần *chính bạn* làm; dùng "tôi"; mô tả scope đúng.
- **Owner vs contributor (ROLE-002):** trung thực bạn **ra quyết định kiến trúc** hay **thực thi quyết định người khác**; nếu là contributor, kể đóng góp thật trong scope đó — không phóng đại.
- **Quyết định tự đưa ra (ROLE-003):** 1 quyết định **cụ thể + lý do + kết quả** → chứng minh quyền tự quyết thật.
- **Phần khó tự giải (ROLE-004):** chiều sâu hands-on ở chính phần mình — chi tiết đủ mức *không thể học thuộc, phải đã làm*.
- **Leadership kỹ thuật (ROLE-005):** đặt chuẩn, review, mentor, align hướng — không chỉ "viết code". *(leadership con người thuần → giao B.)*
- **Phần KHÔNG đụng (ROLE-006):** nghịch lý quan trọng — **thừa nhận "tôi không làm phần X" làm TĂNG độ tin** cho phần "tôi đã làm".
- **Đồng nghiệp sẽ nói gì (ROLE-007):** tự đánh giá khớp thực tế, không tô vẽ — kể phiên bản đồng đội xác nhận được.
- **Thuyết phục bằng gì (ROLE-008):** **dữ liệu / prototype / POC**, không bằng chức danh hay giọng to. *(→ giao B.)*
- **Tự hào & tự phê (ROLE-009):** cân bằng — dám nêu điểm yếu *cụ thể* (không "điểm yếu là quá cầu toàn").
- **Quy công công bằng (COMM-005):** khi đồng đội không đồng ý cách bạn kể về đóng góp chung → chỉnh để **vừa đúng vừa công bằng**, vẫn nêu rõ phần mình. *(→ giao B.)*

**(c) Mép giới hạn & sai lầm:**
- ❌ "Chúng tôi" suốt → không phân biệt được bạn với team.
- ❌ Nhận vơ quyết định kiến trúc của người khác → vỡ khi bị khoan sâu (Bài 9).
- ❌ Điểm yếu giả vờ ("quá cầu toàn", "làm việc quá chăm") → ai cũng biết là né.
- ❌ Phóng đại từ contributor thành owner → mất chính trực.

> **🧭 Hiểu để chỉ huy & kiểm tra:** đây chính là chỗ phân biệt **CHỈ HUY** (bạn *định hình* quyết định kiến trúc) vs **THỰC THI** (bạn *làm theo*). Kể đúng vai trò là tự-kiểm-tra trung thực; nói quá là lỗi sẽ bị Bài 9 phơi bày.

### ③ ⚠️ Cũ → Mới (cách quy công)

| Cũ | Mới | Vì sao |
|---|---|---|
| "Chúng tôi xây / chúng tôi quyết" | "Tôi xây phần…, tôi quyết…" (~80% "I") | Interviewer chấm *bạn*, không phải team |
| Nhận hết để nghe oách | Owner hay contributor — nói thẳng | Bị khoan sâu sẽ lộ |
| Giấu phần không làm | **Thừa nhận phần không đụng** | Nghịch lý: tăng độ tin phần đã làm |
| Điểm yếu sáo rỗng | Điểm yếu *cụ thể, thật* | Tự phê thật = dấu hiệu trưởng thành |

### ④ Áp dụng thực tế + bigtech
- **Thực tế:** với mỗi dự án, vẽ **"bản đồ ownership"**: phần *tôi quyết & xây* / phần *tôi đóng góp* / phần *tôi không đụng*. Chỉ kể sâu phần đầu.
- **Bigtech (verify):** field-guide nói thẳng — narrative tiết lộ *bạn drive decisions hay execute tasks*, và mức ownership quyết định **fit cho seniority** *(field-guide)*. "Tell me about a project you're most proud of, and what role you played" là câu phổ biến 2026 *(Adil Shamim 2026)*.

### ⑤ Kịch bản mẫu + artefact
**Artefact:** bản đồ ownership 3 ô (quyết&xây / đóng góp / không đụng).
**Template:**
```
"Trong hệ này, phần em OWN là <X> — em ra quyết định <quyết> và tự implement <phần khó>.
 Em đóng góp vào <Y> cùng <ai>. Phần <Z> do <người khác> làm, em không đụng tới.
 Quyết định em tự tin nhất: <quyết> → kết quả <metric>."
```

### ⑥ Keywords
- 🧠 **Cần HIỂU:** I-vs-we · owner vs contributor · ownership như tín hiệu seniority · nghịch lý "thừa nhận tăng độ tin".
- 📌 **Cần THUỘC:** chuẩn ~80% "I" · thuyết phục bằng data/POC, không bằng chức danh.
- 🛠️ **Cần LÀM ĐƯỢC:** vẽ bản đồ ownership · kể 1 quyết định tự đưa ra (lý do+kết quả) · nêu 1 điểm yếu cụ thể thật.

### ⑦ Mental model
> **"Dùng 'tôi' cho phần của tôi, 'không đụng' cho phần không phải — trung thực về scope chính là tín hiệu seniority."**

### ⑧ Câu hỏi & thách đố
1. *(ROLE-001, ⭐⭐)* "Phần nào CHÍNH anh làm? Dùng 'tôi'." → **dò:** tách đóng góp cá nhân, không nhận vơ.
2. *(ROLE-002, ⭐⭐⭐)* "Anh ra quyết định hay thực thi?" → **dò:** trung thực owner vs contributor.
3. *(ROLE-003, ⭐⭐⭐⭐)* "Quyết định kỹ thuật quan trọng nhất anh TỰ đưa ra?" → **dò:** cụ thể + lý do + kết quả.
4. *(ROLE-006, ⭐⭐⭐)* "Phần nào anh KHÔNG đụng tới?" → **dò:** thành thật ranh giới → tăng độ tin.
5. *(ROLE-009, ⭐⭐⭐⭐)* "Phần tự hào nhất & làm chưa tốt?" → **dò:** cân bằng tự hào + tự phê cụ thể.
- 🎯 **Thách đố:** *(ROLE-007, ⭐⭐⭐⭐)* "Nếu tôi hỏi đồng nghiệp anh về vai trò anh, họ sẽ nói gì?"* → **bẫy:** mô tả vai trò *to hơn* phiên bản đồng đội xác nhận. Đáp đúng = phiên bản **khớp thực tế**, có thể kiểm chứng.

### ⑨ Bài tập + tiêu chí
- **BT1:** Bản đồ ownership cho dự án tủ. **Đạt khi:** có đủ 3 ô, ô "không đụng" không bị bỏ trống vì ngại.
- **BT2:** Viết lại 1 đoạn kể đang dùng "chúng tôi" → đổi sang "tôi" cho phần của bạn. **Đạt khi:** ~80% câu là "I" mà vẫn công bằng với team.

### ⑩ Đọc thêm
- Mục **B** (Behavioral/Leadership — quy công, thuyết phục, xung đột) trong project.

### 🧪 Tự kiểm (recall)
1. Vì sao "chúng tôi" là vấn đề? · 2. Vì sao thừa nhận "phần tôi không làm" lại tăng độ tin? · 3. Thuyết phục team bằng gì là đúng?
<details><summary>Dò cái gì</summary>

1. Không cho biết *bạn* làm gì → interviewer không chấm được seniority của bạn.
2. Cho thấy bạn trung thực về scope → phần "tôi đã làm" trở nên đáng tin hơn.
3. **Dữ liệu / prototype / POC**, không bằng chức danh hay giọng to.
</details>

---

## 📖 BÀI 8 — Hồi tố & cải tiến (Learnings)
*(Phủ: D-RETRO-001…008)*

### ① Mục tiêu & vị trí trong mạch
Tầng **phản tư** — phân biệt người *làm xong rồi quên* với người *rút bài học chuyển giao được*. Học xong bạn trả lời "làm lại đổi gì", "bài học lớn nhất", "phần nào thoái hóa", "tải ×2 còn chịu được không" — bằng *trải nghiệm thật*, không lý thuyết. Lớp **Learning** của CARL (Bài 1) sống ở đây.

### ② Giảng: cơ bản → nâng cao

**(a) Trực giác.** "Làm lại em không đổi gì cả" là **red flag thiếu phản tư**. Senior luôn có 1–2 thứ muốn làm khác — và quan trọng hơn, **đã áp dụng bài học đó vào dự án sau**.

**(b) Cơ chế (📘):**
- **Làm lại đổi gì (RETRO-001):** 1–2 thay đổi **cụ thể + lý do từ trải nghiệm thật**; tránh "không đổi gì".
- **Bài học lớn nhất (RETRO-002):** rút **Learning chuyển giao được** + chứng minh **đã áp dụng lại** ở dự án sau (dấu hiệu wisdom senior+).
- **Quyết định "lãi kép" (RETRO-003):** nhận ra đầu tư về sau hóa ra rất đáng (observability / test / abstraction đúng / đặt tên đúng) — hiểu giá trị tích lũy theo thời gian.
- **Phần thoái hóa (RETRO-004):** hiểu vì sao code/kiến trúc **xuống cấp** (coupling tăng, thiếu test, scope creep, người rời đi). *(→ giao Q.)*
- **Tải ×2 (RETRO-005):** đánh giá **khả năng tiến hóa** — điểm bắt buộc refactor vs điểm còn dư địa.
- **Công nghệ mới sau dự án (RETRO-006):** cập nhật xu hướng + **biện minh thay thế** (lợi ích thật, không "mới = tốt"). *(verify tên/đời công nghệ; → giao R.)*
- **Ước đã đo/log từ đầu (RETRO-007):** ý thức **observability-first**; học từ điểm mù dữ liệu từng cản debug.
- **Scope ban đầu (RETRO-008):** tự phê **YAGNI vs thiếu chuẩn bị** — cân bằng "vẽ vời quá sớm" và "thiếu nền mở rộng".

**(c) Mép giới hạn & sai lầm:**
- ❌ "Không đổi gì" / "mọi thứ đều ổn" → thiếu phản tư.
- ❌ Bài học chung chung ("em học được teamwork") → không chuyển giao được.
- ❌ "Mới = tốt" khi nói công nghệ thay thế → thiếu biện minh.
- ❌ Quá phê phán bản thân ("toàn bộ thiết kế là rác") → mất khung *đúng-lúc-đó* (Bài 4).

> **🧭 Hiểu để chỉ huy & kiểm tra:** RETRO-006 hay dụ bạn khen công nghệ mới. Người hiểu **kiểm tra**: "công nghệ X mới giải đúng vấn đề tôi từng gặp, hay chỉ đang hot?" — và *(verify)* tên/đời/giá trước khi khẳng định trong phỏng vấn.

### ③ ⚠️ Cũ → Mới (cách hồi tố)

| Cũ | Mới | Vì sao |
|---|---|---|
| "Làm lại không đổi gì" | Nêu 1–2 thay đổi cụ thể + lý do thật | Cho thấy phản tư |
| Bài học chung chung | **Learning chuyển giao được + đã dùng lại** | Dấu hiệu wisdom senior+ |
| "Công nghệ mới nên dùng" | Biện minh lợi ích thật + *(verify)* | "Mới = tốt" là tư duy lười |
| Thêm log/metric khi đã cháy | **Observability-first** từ đầu | Điểm mù dữ liệu giết thời gian debug |

### ④ Áp dụng thực tế + bigtech
- **Thực tế:** chuẩn bị **"3 Learnings"** — mỗi cái: *điều rút ra · vì sao · đã áp dụng ở đâu sau đó*.
- **Bigtech (verify):** "what you'd change" là phần cố định của deep-dive ("present a proud project: design decisions, trade-offs, what broke, and **what you'd change**") *(Adil Shamim 2026; Careery 2026)*. Field-guide nhấn *recent & greenfield* dễ kể hơn — chọn dự án còn nhớ rõ để hồi tố sắc.

### ⑤ Kịch bản mẫu + artefact
**Artefact:** danh sách "3 Learnings" + 1 đoạn "làm lại sẽ đổi gì".
```
LEARNING #1: <điều rút ra> — vì <trải nghiệm thật> — đã áp dụng ở <dự án/việc sau>.
LÀM LẠI:     "Em sẽ đổi <X> vì <lý do từ vận hành thật>. Giữ nguyên <Y> vì nó hóa ra lãi kép (<bằng chứng>)."
```

### ⑥ Keywords
- 🧠 **Cần HIỂU:** Learnings chuyển giao · quyết định lãi kép · architecture decay · YAGNI · observability-first.
- 📌 **Cần THUỘC:** khung CARL nhấn **Learning** · phân biệt over-built vs under-built.
- 🛠️ **Cần LÀM ĐƯỢC:** nêu 1–2 thay đổi cụ thể nếu làm lại · 1 Learning đã áp dụng lại · 1 phần đã thoái hóa + vì sao.

### ⑦ Mental model
> **"Làm xong là phải rút Learning; bài học thật là cái đã dùng lại ở dự án sau."**

### ⑧ Câu hỏi & thách đố
1. *(RETRO-001, ⭐⭐⭐)* "Làm lại đổi gì lớn nhất?" → **dò:** 1–2 thay đổi cụ thể + lý do thật.
2. *(RETRO-002, ⭐⭐⭐⭐)* "Bài học lớn nhất, giúp gì cho dự án sau?" → **dò:** Learning chuyển giao + đã áp dụng lại.
3. *(RETRO-004, ⭐⭐⭐⭐)* "Phần nào thành gánh nặng theo thời gian? Vì sao?" → **dò:** hiểu nguyên nhân decay.
4. *(RETRO-005, ⭐⭐⭐)* "Tải ×2 còn chịu được không? Sửa gì?" → **dò:** đánh giá khả năng tiến hóa.
5. *(RETRO-007, ⭐⭐⭐)* "Ước đã đo/log gì từ đầu?" → **dò:** observability-first.
- 🎯 **Thách đố:** *(RETRO-003, ⭐⭐⭐)* "Quyết định nào về sau chứng minh tốt HƠN anh nghĩ ban đầu?"* → **bẫy:** chỉ kể quyết định "đúng ngay". Đáp đúng = một đầu tư *lúc đó tưởng tốn công vô ích* (test/observability/abstraction) mà sau **lãi kép** — hiểu giá trị tích lũy.

### ⑨ Bài tập + tiêu chí
- **BT1:** Viết "3 Learnings". **Đạt khi:** mỗi Learning có *đã áp dụng lại ở đâu* (không treo lơ lửng).
- **BT2:** Viết "làm lại đổi gì + giữ gì". **Đạt khi:** có cả thay đổi lẫn phần lãi-kép giữ nguyên (không cực đoan "đổi hết/giữ hết").

### ⑩ Đọc thêm
- Mục **Q** (Software Design — decay, abstraction), **R** (Modern Trends — công nghệ mới) trong project.

### 🧪 Tự kiểm (recall)
1. Vì sao "không đổi gì cả" là red flag? · 2. Thế nào là bài học "chuyển giao được"? · 3. Quyết định "lãi kép" là gì?
<details><summary>Dò cái gì</summary>

1. Thiếu phản tư — senior luôn có 1–2 thứ muốn làm khác.
2. Learning trừu tượng đủ để **áp dụng vào dự án khác**, và bạn *đã dùng lại* nó.
3. Đầu tư lúc đầu tưởng tốn (test/observability/abstraction/đặt tên) nhưng giá trị **tích lũy lớn** về sau.
</details>

---

## 📖 BÀI 9 — Chịu khoan sâu & chính trực dưới sức ép
*(Phủ: D-CRED-001…008, D-WHY-011, D-WHY-012, D-COMM-001, D-COMM-002)*

### ① Mục tiêu & vị trí trong mạch
Bài **khó nhất, tầng senior+** — phân biệt người *đã XÂY* với người *đã ĐỌC*. Học xong bạn: trung thực về **ranh giới đọc-vs-làm**, biết **dừng đúng chỗ "tôi không chắc"** thay vì bịa, gắn nhãn **đo vs ước** cho số liệu, và thể hiện tư duy **"hiểu để chỉ huy & kiểm tra"**. Mọi bài trước chỉ vững nếu bài này giữ được chính trực.

### ② Giảng: cơ bản → nâng cao

**(a) Trực giác.** Interviewer 2026 sẽ **khoan 'tại sao' liên tục cho tới khi bạn nói 'tôi không biết'**. Điều họ test không phải *bạn biết hết* — mà *bạn trung thực ở biên giới hiểu biết*. Nói thẳng "tới đây tôi không chắc, em sẽ tra X" **tăng** độ tin cho mọi thứ trước đó. Bịa một câu → đổ sập toàn bộ.

**(b) Cơ chế (📘):**
- **Read-vs-built (CRED-001, WHY-011):** tự tay implement hay dùng lib? Nếu tự làm → có chi tiết hands-on (*went wrong* ở đâu, sửa sao). Nếu dùng lib → **nói thẳng**. Đây là ranh giới AI/blog không vượt được.
- **Chiều sâu sau buzzword (CRED-002):** giải thích cơ chế bên trong thành phần vừa nhắc — **sâu hết mức bạn hiểu**, rồi **dừng đúng chỗ** "tôi không chắc".
- **Tự nhận biên giới (CRED-003, WHY-012):** thuật ngữ nào trong CV/câu trả lời bạn *không tự tin giải thích kỹ* → nói trước, không resume-padding.
- **Phản ứng khi chạm đáy (CRED-004):** khi bị khoan tới "tôi không biết" → nói thẳng + **cách sẽ tra cứu/kiểm chứng**, không chống chế.
- **Số đo vs ước (CRED-005):** gắn nhãn đúng ("khoảng ~", "tôi nhớ là…"); giữ uy tín thay vì khẳng định số bịa.
- **Quyền sở hữu code (CRED-006):** nếu mở codebase ngay, bạn chỉ đúng phần mình viết được chứ? Không nhận phần người khác.
- **Khác bài blog (CRED-007):** điều khiến câu trả lời bạn khác AI/blog = **chi tiết production thật** (số liệu, sự cố cụ thể, đánh đổi đã thực gặp).
- **Tự hiệu chỉnh (CRED-008):** phần nào bạn đã **tô hồng/đơn giản hóa** khi kể → kể lại cho đúng; phân biệt "kể gọn" (ổn) vs "nói quá" (red flag).
- **Chỉ huy vs kiểm tra (COMM-001):** trong dự án, bạn **CHỈ HUY** (định hình quyết định) ở đâu, **KIỂM TRA** (bắt lỗi máy/người) ở đâu — ví dụ cụ thể cho từng loại.
- **AI đúng-cú-pháp-sai-kiến-trúc (COMM-002):** chỉ ra 1 chỗ trong hệ mà **chỉ người HIỂU mới bắt được lỗi** (vd: fine-tune cho dữ liệu hay đổi thay vì RAG; `temperature` cao cho task cần ổn định; side-effect đặt vào critical path).

**(c) Mép giới hạn & sai lầm:**
- ❌ Bịa tiếp khi không biết → interviewer thường biết câu trả lời, bịa = trượt.
- ❌ Khẳng định số "chắc nịch" mà thực ra đang ước → vỡ khi bị hỏi nguồn.
- ❌ Resume-padding (liệt kê công nghệ chỉ "nghe qua") → bị khoan là lộ.
- ❌ Nhận phần code/quyết định của người khác → CRED-006 phơi bày.

> **🧭 Hiểu để chỉ huy & kiểm tra (đỉnh của mục D):** cả Bài 9 là hiện thân của câu thần chú. **CHỈ HUY** = bạn định hình yêu cầu đúng (chọn RAG cho dữ liệu hay đổi, không fine-tune). **KIỂM TRA** = bạn bắt lỗi *đúng-cú-pháp-sai-hành-vi* mà máy/người tạo ra (temperature=0.7 cho task cần xác định, side-effect trong critical path). Hai ví dụ này chính là câu COMM-002 — chuẩn bị sẵn 1 chỗ như vậy trong hệ của bạn.

### ③ ⚠️ Cũ → Mới (độ tin dưới sức ép)

| Cũ | Mới | Vì sao |
|---|---|---|
| Bịa tiếp khi bị khoan | "Tới đây tôi không chắc, em sẽ tra <X>" | Trung thực ở biên giới = tăng độ tin |
| Khẳng định mọi số chắc nịch | Gắn nhãn **đo vs ước** ("~", "tôi nhớ là") | Một số bịa bị bắt → sập cả phần còn lại |
| Resume-padding mọi buzzword | Tự nhận thứ chỉ "nghe qua" | Self-awareness chính nó tăng tín nhiệm |
| Nhận vơ quyết định/code | Khớp đúng phần mình | Mở codebase ra là lộ |
| Trả lời như blog tổng quát | Chi tiết production thật (số, sự cố, trade-off đã gặp) | Đó là thứ AI/blog không có |

### ④ Áp dụng thực tế + bigtech
- **Thực tế:** lập **"bản đồ độ tin"** trước phỏng vấn — với mỗi thuật ngữ/thành phần trong CV: *tự làm / dùng lib / chỉ đọc* + *mức tự tin giải thích (1–5)*. Khi bị khoan, bạn biết khi nào nói "tôi không chắc".
- **Bigtech (verify):** đây đúng xu hướng cốt lõi 2026 — interviewer khoan **failure modes + "what went wrong last time"** và quý **honest acknowledgment of what we still don't know** *(TechEon 2026)*; field-guide: "can you go multiple levels deep when probed?" là tiêu chí chính, và *impact-framing* > *technology-name-dropping* *(field-guide)*.

### ⑤ Kịch bản mẫu + artefact
**Artefact:** bản đồ độ tin (term · tự làm/lib/đọc · tự tin 1–5).
**Template "chạm đáy hiểu biết" (CRED-004):**
```
"Phần này em hiểu tới mức <…>. Sâu hơn nữa — vd <chi tiết X> — em không chắc;
 em sẽ kiểm chứng tại <docs chính thức / đọc source> trước khi khẳng định."
```
**Template "đo vs ước" (CRED-005):**
```
"p99 em NHỚ là ~300ms — số này em ước theo trí nhớ, để chắc em sẽ tra lại dashboard.
 Còn QPS peak ~800 thì em chắc vì vừa nhìn tuần trước."
```

### ⑥ Keywords
- 🧠 **Cần HIỂU:** read-vs-built · "dừng đúng chỗ tôi không chắc" · đo vs ước · chỉ huy vs kiểm tra · đúng-cú-pháp-sai-hành-vi.
- 📌 **Cần THUỘC:** câu mẫu thừa nhận biên giới hiểu biết · cách gắn nhãn số liệu.
- 🛠️ **Cần LÀM ĐƯỢC:** trả lời chuỗi "tại sao" tới khi nói "tôi không biết" một cách bình tĩnh · chỉ 1 chỗ AI đúng-cú-pháp-sai-kiến-trúc trong hệ bạn · phân biệt CHỈ HUY/KIỂM TRA bằng ví dụ thật.

### ⑦ Mental model
> **"Không ai test bạn biết hết — họ test bạn trung thực ở biên giới. 'Tôi không chắc, sẽ tra X' tăng tín nhiệm; bịa một câu là sập tất."**

### ⑧ Câu hỏi & thách đố
1. *(CRED-001 / WHY-011, ⭐⭐⭐⭐⭐)* "Pattern X — anh TỰ implement hay dùng lib?" → **dò:** ranh giới đọc-vs-làm; nếu tự làm có chi tiết *went wrong*.
2. *(CRED-002, ⭐⭐⭐⭐)* "Giải thích cơ chế bên trong, sâu hết mức." → **dò:** chiều sâu thật + biết dừng đúng chỗ.
3. *(CRED-005, ⭐⭐⭐⭐)* "Số QPS/latency đó — nhớ chính xác hay ước?" → **dò:** gắn nhãn đo vs ước.
4. *(COMM-001, ⭐⭐⭐)* "Trong dự án, anh CHỈ HUY ở đâu, KIỂM TRA ở đâu?" → **dò:** phân biệt ra-quyết-định vs kiểm-soát-chất-lượng, ví dụ cụ thể.
5. *(COMM-002, ⭐⭐⭐⭐)* "AI viết phần này đúng-cú-pháp-sai-kiến-trúc — chỗ nào chỉ người HIỂU mới bắt?" → **dò:** 1 ví dụ thật (RAG-vs-fine-tune / temperature / side-effect trong critical path).
- 🎯 **Thách đố:** *(CRED-004 / CRED-008, ⭐⭐⭐⭐⭐)* "Tôi sẽ hỏi 'tại sao' liên tục tới khi anh nói 'tôi không biết'. Sẵn sàng chứ? … Và phần nào nãy giờ anh đã tô hồng?"* → **bẫy:** cố chống đỡ tới cùng / không dám rút lại lời nói quá. Đáp đúng = bình tĩnh tới "tôi không chắc + cách tra", và **chủ động hiệu chỉnh** phần đã đơn giản hóa quá.

### ⑨ Bài tập + tiêu chí
- **BT1:** Lập bản đồ độ tin cho 8 thuật ngữ trong CV. **Đạt khi:** có ít nhất 1–2 mục tự đánh dấu "chỉ đọc / tự tin ≤3" (dám thừa nhận biên giới).
- **BT2:** Nhờ người khác khoan "tại sao" 5 lần liên tiếp vào 1 quyết định. **Đạt khi:** bạn tới được "tôi không chắc, sẽ tra X" mà không bịa.

### ⑩ Đọc thêm
- Mục **A, E, I, J, K, R** trong project — để *thật sự hiểu sâu* phần bạn đánh dấu "tự tin thấp", thu hẹp ranh giới đọc-vs-làm.

### 🧪 Tự kiểm (recall)
1. Khi bị khoan tới chỗ không biết, làm gì? · 2. Vì sao gắn nhãn "đo vs ước" lại quan trọng? · 3. CHỈ HUY khác KIỂM TRA thế nào (cho 1 ví dụ mỗi loại)?
<details><summary>Dò cái gì</summary>

1. Nói thẳng "tới đây tôi không chắc" + **cách sẽ tra cứu/kiểm chứng**; không bịa, không chống chế.
2. Một số bịa bị bắt → sập độ tin của cả phần còn lại; gắn nhãn giữ uy tín.
3. CHỈ HUY = định hình quyết định (chọn RAG cho dữ liệu hay đổi); KIỂM TRA = bắt lỗi đúng-cú-pháp-sai-hành-vi (sửa temperature=0.7→0 cho task cần ổn định).
</details>

---
---

# ✅ Cổng hoàn thành mục D (WORKFLOW 2 — Bước E)

**"Học xong mục D" = ĐẠT toàn bộ 79 câu** qua các đợt phỏng vấn live (Bước D). Tài liệu này là *nền để ôn*; phần chấm ĐẠT/CHƯA diễn ra khi bạn luyện hỏi-đáp.

### Lộ trình ôn ngắt quãng (spaced repetition)
| Mốc | Việc |
|---|---|
| **Hôm nay** | Đọc Bài 1–3, làm BT từng bài, chuẩn bị **sơ đồ C2** + **bảng số liệu 1 trang** |
| **Mai** | Đọc Bài 4–6; tự test recall Bài 1–3 (che đáp án) |
| **+3 ngày** | Đọc Bài 7–9; nhờ người khoan "tại sao" vào 1 quyết định (Bài 9) |
| **+1 tuần** | Phỏng vấn thử trọn vẹn: opener (Bài 1) → vẽ (Bài 2) → bị vặn ngẫu nhiên Bài 3–9 |

### Checklist sẵn sàng trước phỏng vấn thật
- [ ] Có **sơ đồ C4 (Context + Container)** vẽ tay được trong 2 phút.
- [ ] Có **bảng số liệu 1 trang** (tải/data/latency p50-p95-p99/SLO/cost), gắn nhãn *đo/ước*.
- [ ] Có **2 incident card** đủ khung (root cause + action item).
- [ ] Có **bản đồ ownership** (quyết&xây / đóng góp / không đụng).
- [ ] Có **3 Learnings** đã-áp-dụng-lại + đoạn "làm lại đổi gì".
- [ ] Có **bản đồ độ tin** + 1 ví dụ "AI đúng-cú-pháp-sai-kiến-trúc" trong hệ của bạn.
- [ ] Mọi tên/đời công nghệ, model, giá, benchmark định nói → đã *(verify)* nguồn chính thức.

---

## 📚 Nguồn đã tham khảo (verify-able)
- **alexeygrigorev — ai-engineering-field-guide** (Q4'25–Q1'26): `interview/questions/03-project-deep-dive.md` — cấu trúc narrative, tín hiệu seniority, impact-framing, format Anthropic 25'+15-20'.
- **TechEon — *The Complete Agentic AI System Design Interview Guide 2026*** (Medium, 1/2026): senior khoan 3–5 câu sâu vào failure modes + war stories + diagrams.
- **Adil Shamim — *Every AI Engineer Interview Question 2026 (from 100+ real interviews)*** (Medium, 5/2026): "prepare 3–5 project deep dives, know every decision and metric, practice follow-up probes".
- **lockedin.ai — *AI Engineer Interview Prep: Top 40 2026***; **Careery — *AI Engineer Interview Questions 2026***: nén đáp ~60–90s, gắn ví dụ đã ship; review own projects deeply.
- **C4 model** — c4model.com (Simon Brown). · **Google SRE Book** — chương Postmortem (blameless).
> ⚠️ Các nguồn trên là blog/tổng hợp tuyển dụng — coi là *pattern phổ biến*, không phải chân lý tuyệt đối. Số liệu/đời công nghệ luôn *verify* tại docs chính thức.

---

*Tài liệu sinh theo instruction "Giảng viên GenAI Applications (Pro) — v3", WORKFLOW 2 Bước B+C, cho mục D — Deep-dive dự án cũ. Phần phỏng vấn chấm-live (Bước D) và cổng ĐẠT/CHƯA thực hiện khi luyện hỏi-đáp.*
