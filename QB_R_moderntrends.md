# 🚀 QB_R — Ngân hàng câu hỏi: Modern Trends 2026 (backend / TL)

> **Mục R** trong roadmap Tech Lead Backend (🟢 ~15% — *ít hỏi trực tiếp*, giá trị chính là **bàn được khi bị hỏi** + cho thấy bạn cập nhật và có **gu đánh giá trend**) · Sinh theo **WORKFLOW 2 — Bước A**.
> Đây là **mục LỚN** → bộ đề phủ HẾT 4 mục con của roadmap (AI-assisted coding · Edge computing · Platform Engineering/IDP · Green/efficient coding) **+ mở rộng đúng các trend 2026 mà TL backend bị vặn**: AI-native backend (RAG/vector DB/MCP/agents), WebAssembly server-side, Serverless 2.0, FinOps/GreenOps, và lớp **gu đánh giá trend** (hype vs substance).
> **Chỉ câu hỏi — KHÔNG kèm đáp án.** Mỗi câu có dòng *"dò cái gì"* = tiêu chí ĐẠT, dùng để chấm **live** ở Bước D.
> **Tổng: 80 câu.**

## 🚧 Chống trùng (đã đọc `QB_A_systemdesign.md`, `QB_E_database.md`, `QB_F_apidesign.md`, `QB_G_nodejs.md`, `QB_H_nestjs.md`, `QB_I_concurrency.md`, `QB_J_caching.md`, `QB_K_messaging.md`, `QB_L_testing.md`, `QB_M_security.md`, `QB_N_devops.md`, `QB_O_observability.md`, `QB_Q_designpatterns.md`)

R là mục "tổng hợp xu hướng" nên **đụng nhiều mục**. Quy ước phân ranh để **không lặp**:

- **`A-SVL` (System Design — Serverless & edge computing, A-SVL-001→005)** đã sở hữu *trade-off lõi* serverless vs service truyền thống, "serverless hay container/k8s", state/DB-connection của serverless, và *đặt logic gì ở edge ở mức nguyên tắc*.
  → **R KHÔNG lặp trade-off lõi đó.** `R-EDGE` đào phần **trend 2026 cụ thể**: V8 isolates & sub-ms cold start, **Serverless 2.0 / Wasm@edge**, **edge data layer** (edge KV / edge DB), vendor lock-in edge, "K8s có thành legacy không". Khi đụng "serverless hợp lúc nào" → **trỏ A-SVL**.
- **`N-TL-006` (DevOps — Platform Engineering/IDP & golden path)** đã phủ *định nghĩa IDP + golden path + khác "team DevOps làm hộ"*. **N** cũng sở hữu K8s ops, cloud, CI/CD.
  → **R KHÔNG lặp định nghĩa IDP.** `R-PLAT` đào sâu hơn: **platform-as-product**, **build vs buy (Backstage vs Port/Cortex)**, bẫy adoption ~10%, software catalog làm system-of-record, FinOps/security shift-left ở provisioning, AI-augmented platform. Đụng cơ chế K8s/cloud → **trỏ N**.
- **`Q-431` (Design Patterns — AI coding assistant sinh code vi phạm *thiết kế*, kiểm soát chất lượng *thiết kế*)** đã sở hữu góc **design review** của code do AI sinh.
  → **R KHÔNG lặp góc design.** `R-AICODE` lấy góc **workflow & governance**: bảo mật/supply-chain của AI code, đo lường năng suất AI thật/giả, code review khi phần lớn do AI viết, định dạng **AI-assisted interview 2026**, chính sách dùng AI tool, atrophy kỹ năng. Đụng "code AI sai *thiết kế*" → **trỏ Q-431**.
- **`M` (Security)** sở hữu *threat model sâu*: supply chain SBOM/SLSA (M-SUPPLY), secrets (M-SECRET), prompt-injection ở góc app security.
  → `R-AICODE` / `R-AIARCH` chỉ **nêu** rủi ro (hallucinated package, data exfiltration qua agent-tool) rồi **trỏ M** cho chi tiết kiểm soát.
- **`E-427` (Database — serverless + Postgres "cháy" connection)** đã sở hữu bài toán connection pool ở serverless.
  → `R-EDGE` đụng "data gravity / round-trip tới central DB" thì **trỏ E** cho cơ chế pool/proxy, không lặp.
- **`O` (Observability) — mục riêng, CHƯA có file QB.** R chỉ **chạm** "eval/observability LLM khó hơn" và "debug ở edge khó hơn" rồi đánh dấu *adjacent → O*; không đào tracing/metrics sâu.
- **`J` (Caching)** giữ CDN ở góc *cache strategy*; **F** giữ rate-limit/pagination ở góc API. → R chỉ trỏ tới khi liên quan.

---

## Cách đọc
- **ID:** `R-<tag>-<số>`. Tag mục con liệt kê ở mục lục dưới.
- **Độ khó:** ⭐ định nghĩa cơ bản → ⭐⭐⭐⭐⭐ phân biệt senior với TL thật (gu đánh giá trend, trade-off production, governance, "khi nào KHÔNG").
- **"Dò cái gì":** điều người trả lời PHẢI chạm tới mới tính ĐẠT (không phải đáp án — chỉ là mốc chấm live ở Bước D).
- ⚠️ R là mục **đổi nhanh nhất** trong roadmap → gần như mọi câu chạm **tên/đời sản phẩm, version, số liệu** đều cần *(verify)* tại docs/nguồn chính thức khi học. Tinh thần: *bàn được pattern & trade-off ổn định*, còn *con số/đời thì tra cứu*.

## Bối cảnh landscape (tính tại ~6/2026 — *verify lại khi học*)

> ⚠️ Phần này là *ảnh chụp xu hướng*, không phải chân lý vĩnh viễn. Mọi con số/đời sản phẩm dưới đây **phải kiểm chứng** trước khi trích trong phỏng vấn.

- **AI-assisted interview đã thành xu hướng thật:** Meta pilot vòng coding "human-led, AI-assisted" trong CoderPad (10/2025), Google đưa "code comprehension" + chấm **AI fluency** (prompt / validate output / debug) vào loop 2026, Canva yêu cầu dùng AI tool khi phỏng vấn. Trọng tâm dịch từ "viết thuật toán" sang **"chỉ huy & kiểm tra AI"**. *(verify từng công ty)*
- **MCP (Model Context Protocol)** nổi lên 2024→2026 như **lớp chuẩn hoá kết nối tool/data cho agent**; đi kèm tranh luận MCP vs CLI/API trực tiếp (chi phí token, governance). *(verify trạng thái chuẩn)*
- **Vector "table stakes":** nhiều DB quan hệ/streaming đã có vector native (pgvector trong Postgres, TiDB...) → "luôn cần một vector DB riêng" không còn đúng mặc định. *(verify)*
- **WebAssembly ngoài browser** trưởng thành: **WASI** cho Wasm truy cập file/net/time; **Component Model** cho gọi chéo ngôn ngữ; cold start ms→µs vs container; Akamai mua Fermyon (Wasm serverless). Lưu ý **WASI 0.2** là mặc định, **0.3 (async) đang tới**, **Wasm 3.0** mới. *(verify đời WASI/Wasm)*
- **Serverless 2.0 / edge:** Cloudflare Workers (V8 isolates), Fastly Compute, Vercel Edge — cold start sub-ms; tranh luận "K8s thành legacy?". *(verify)*
- **Platform Engineering:** Gartner dự báo ~80% tổ chức lớn có platform team (mốc 2026/2027 tuỳ báo cáo); **Backstage** vẫn phổ biến nhất nhưng tốn vận hành → đội <200 eng nghiêng về SaaS (**Port/Cortex/OpsLevel**); xu hướng **platform-as-product**, AI-augmented, FinOps/security embedded. *(verify số liệu)*
- **GreenOps/FinOps hợp nhất:** "cost ≈ carbon"; ARM/**Graviton** double-win; carbon-aware scheduling; "carbon budget trong CI/CD" đang manh nha. *(verify)*

---

## Mục lục mục con (ID prefix → số câu)

| Tag | Mục con | Số câu |
|---|---|---|
| **R-AICODE** | AI-assisted coding & dev workflow (Copilot/Cursor/agent, governance, security, đo năng suất, AI-interview format) | 14 |
| **R-AIARCH** | AI-native backend (LLM-in-app, RAG, vector DB, MCP, agents, multi-agent, eval, "khi nào KHÔNG dùng LLM") | 16 |
| **R-EDGE** | Edge computing & Serverless 2.0 (isolates, edge data, K8s-legacy?, lock-in, data gravity) | 12 |
| **R-WASM** | WebAssembly server-side (WASI, Component Model, vs container, sandbox, plugin runtime, hype-check) | 10 |
| **R-PLAT** | Platform Engineering / IDP (platform-as-product, build vs buy, adoption trap, catalog, AI-augmented) | 12 |
| **R-GREEN** | Green / FinOps / efficient coding (cost-at-decision, GreenOps, Graviton, carbon-aware, rightsizing) | 8 |
| **R-TL** | Gu đánh giá trend (hype vs substance, adopt timing, reversibility, risk governance AI, "X is dead?") | 8 |

---

## R-AICODE — AI-assisted coding & dev workflow

**R-AICODE-001** ⭐⭐⭐
"Team bắt đầu dùng Copilot/Cursor đại trà. Là TL, anh giữ chất lượng & kiến trúc khỏi *trôi dạt* bằng những lớp gì?" *(góc design vi phạm → Q-431)*
> *Dò cái gì:* AI tăng tốc *sản lượng* nhưng không tăng *phán đoán* → cần guardrail: review bắt buộc (người chịu trách nhiệm là người commit, không phải AI), CI gate (lint/test/SAST), chuẩn hoá pattern qua template/skeleton, PR nhỏ dễ soát, định nghĩa "AI viết được phần nào / cấm phần nào". Trỏ Q-431 cho phần kiểm soát *thiết kế*.

**R-AICODE-002** ⭐⭐⭐
"AI sinh code 'chạy được nhưng sai' — phân loại các kiểu sai mà TL bắt buộc phải bắt được?"
> *Dò cái gì:* (1) bug tinh vi/logic mép (off-by-one, edge case), (2) lỗ hổng bảo mật (SQLi, secret hardcode, dep lỗi thời), (3) hành vi sai dù cú pháp đúng (vd `temperature` cho task cần ổn định, race), (4) API/thư viện **bịa** (hallucinated), (5) license/IP của snippet. Ý chính: cú pháp đúng ≠ đúng.

**R-AICODE-003** ⭐⭐⭐⭐
"Rủi ro bảo mật *đặc thù* của code do AI sinh, và anh chặn ở đâu trong pipeline?" *(threat model sâu → M-SUPPLY)*
> *Dò cái gì:* hallucinated package / **slopsquatting** (kẻ xấu đăng ký tên package mà AI hay bịa), copy code dính lỗ hổng/secret, license nhiễm; chặn = SAST/secret-scan + SCA/dependency pinning theo digest + allowlist registry + review người. Trỏ M cho SBOM/supply-chain.

**R-AICODE-004** ⭐⭐⭐
"Khi phần lớn code do AI viết, code review thay đổi thế nào? Cái gì review *nhiều hơn*, cái gì *ít đi*?"
> *Dò cái gì:* dịch trọng tâm từ "cú pháp/style" (AI/linter lo) sang **ý định, ranh giới, bảo mật, hợp kiến trúc, test có ý nghĩa không**; PR nhỏ, mô tả "tại sao"; **ownership vẫn ở con người**; cảnh giác review-fatigue khi volume tăng (rubber-stamp).

**R-AICODE-005** ⭐⭐⭐⭐
"Sếp nói 'AI giúp dev nhanh hơn vì số PR/dòng code tăng'. Anh phản biện & đề xuất đo lường đúng thế nào?"
> *Dò cái gì:* lines/PR là **vanity metric**, có thể tăng do code phình/rework; đo bằng **DORA** (lead time, deploy freq, change-fail rate, MTTR) + **rework/churn rate**, defect-escape, review burden, dev satisfaction; coi chừng Goodhart (đo gì sẽ bị tối ưu méo).

**R-AICODE-006** ⭐⭐⭐
"Phân biệt autocomplete (Copilot), AI-IDE (Cursor), và *autonomous coding agent*. Mỗi loại hợp việc gì?"
> *Dò cái gì:* autocomplete = gợi ý từng dòng/đoạn (in-flow, người lái); AI-IDE = chat + sửa nhiều file theo ngữ cảnh repo; agent = nhận task, tự lặp đọc-sửa-chạy-test (cần sandbox + review chặt). Mức tự chủ tăng → cần guardrail/blast-radius control tăng.

**R-AICODE-007** ⭐⭐⭐⭐
"Lạm dụng AI gây 'teo kỹ năng' junior / 'vibe coding'. Là TL anh bảo vệ năng lực học của team thế nào?"
> *Dò cái gì:* nhận diện rủi ro junior không hiểu code mình ship; biện pháp: bắt giải thích được code (kèm khi review), giai đoạn học nền không dựa AI, pairing, AI để *tăng tốc cái đã hiểu* chứ không thay *việc hiểu*; câu thần chú "hiểu để chỉ huy & kiểm tra".

**R-AICODE-008** ⭐⭐⭐
"Vòng phỏng vấn 'AI-assisted coding' 2026 (Meta/Google) thực chất *test cái gì* khác trước?"
> *Dò cái gì:* không còn đo thuộc thuật toán; đo **AI fluency** = prompt rõ, **validate/critique output của AI**, debug, đọc-hiểu codebase có sẵn (code comprehension), giao tiếp quyết định; "human-led, AI-assisted". *(verify từng công ty)*

**R-AICODE-009** ⭐⭐⭐⭐
"Ứng viên giao hết cho AI rồi *không giải thích nổi* code. Tín hiệu gì? Anh sẽ probe thế nào?"
> *Dò cái gì:* dấu hiệu thiếu phán đoán/ownership — nguy hiểm hơn cả không biết, vì sẽ ship cái mình không hiểu; probe: "vì sao chọn cách này?", "edge case nào hỏng?", "nếu input X?", yêu cầu review chính output AI vừa cho. Đây cũng là cách **anh kiểm tra cấp dưới** ở thực tế.

**R-AICODE-010** ⭐⭐⭐
"Ngoài viết code, AI nên hỗ trợ chỗ nào trong SDLC — và nguy hiểm tiềm ẩn ở mỗi chỗ?"
> *Dò cái gì:* test gen, migration/refactor lớn, mô tả PR, phân tích log/incident, sinh tài liệu; nguy hiểm tương ứng: test 'xanh giả', refactor đổi hành vi âm thầm, tóm tắt incident sai dẫn dắt; nguyên tắc: AI làm nháp, người duyệt mốc quan trọng.

**R-AICODE-011** ⭐⭐⭐⭐⭐
"Soạn *policy* dùng AI coding tool cho codebase nhạy cảm/được quản lý (fintech/health). Anh quản những trục nào?"
> *Dò cái gì:* (1) **data leakage** — code/secret có bị gửi lên model? chọn tier no-training/self-host; (2) IP/license của code sinh ra; (3) repo nào được/không được dùng; (4) tool nào approved; (5) log/audit & trách nhiệm; (6) gate bảo mật bắt buộc. Cân bằng năng suất ↔ rủi ro tuân thủ.

**R-AICODE-012** ⭐⭐⭐
"'AI viết luôn cả test' — vì sao điều này rủi ro cho việc *bắt bug thật*?" *(bản chất test → L)*
> *Dò cái gì:* AI dễ viết test *khẳng định lại hành vi hiện tại* (kể cả khi hành vi đó sai) → test xanh nhưng không bảo vệ gì; thiếu test cho ý định/đặc tả; người phải định nghĩa *cái đúng kỳ vọng* trước, AI điền cơ học sau. Trỏ L cho ý nghĩa test.

**R-AICODE-013** ⭐⭐⭐⭐
"Có ý kiến: 'spec/prompt là artifact mới, quan trọng hơn cú pháp'. Anh hiểu thế nào & nó đổi cách làm việc ra sao?"
> *Dò cái gì:* khi AI lo cú pháp, **giá trị dịch lên tầng đặc tả**: mô tả yêu cầu/ràng buộc/tiêu chí chấp nhận chính xác để AI làm đúng → prompt giống viết spec/test; kỹ năng đắt là *diễn đạt vấn đề + nghiệm thu*, không phải gõ.

**R-AICODE-014** ⭐⭐⭐
"Khi nào *nhanh hơn/an toàn hơn* nếu KHÔNG dùng AI cho một task?"
> *Dò cái gì:* task nhỏ/đã thuộc lòng (prompt+review tốn hơn tự gõ), vùng cực nhạy cảm cần kiểm soát từng dòng, domain bí truyền AI không có ngữ cảnh, khi chi phí review > tiết kiệm; nhận diện chi phí ẩn của "delegate rồi phải kiểm".

---

## R-AIARCH — AI-native backend

**R-AIARCH-001** ⭐⭐
"Vì sao 'thêm tính năng LLM' nay là việc của *backend* chứ không chỉ team ML? Backend sở hữu phần gì?"
> *Dò cái gì:* model là commodity gọi qua API; backend lo **mọi thứ quanh nó**: data/retrieval, orchestration, tool/function calling, cache, rate-limit/cost, fallback/timeout, security (injection, PII), eval/observability — đúng bài toán hệ thống phân tán có thêm thành phần phi tất định.

**R-AIARCH-002** ⭐⭐⭐
"Chatbot trả lời tài liệu chính sách *hay đổi*: chọn RAG / fine-tune / prompt? Vì sao?" *(câu kinh điển 'chỉ huy đúng kiến trúc')*
> *Dò cái gì:* dữ liệu hay đổi + cần dẫn nguồn → **RAG** (cập nhật doc không cần train lại, giảm hallucination, có citation); fine-tune hợp *đổi hành vi/style/format ổn định*, không hợp *nhồi kiến thức biến động*; prompt cho yêu cầu nhẹ. Bẫy: chọn fine-tune cho data đổi = sai kiến trúc.

**R-AIARCH-003** ⭐⭐⭐
"Vector database làm gì? Năm 2026 có *luôn* cần một cái *riêng*?" *(connection serverless → E)*
> *Dò cái gì:* lưu embedding + similarity search (ANN) cho semantic retrieval; nhưng **pgvector** (Postgres) / DB tích hợp vector đã đủ cho nhiều ca → tránh thêm hệ thống khi chưa cần; chọn DB chuyên (Pinecone/Qdrant/Weaviate/Milvus) khi scale/throughput/feature (hybrid, multi-tenant) đòi hỏi. *(verify)*

**R-AIARCH-004** ⭐⭐⭐⭐
"Vẽ pipeline một hệ RAG production từ đầu đến cuối, và chỉ chỗ nó *hay vỡ*."
> *Dò cái gì:* ingest → chunk → embed → index (vector store) → (query rewrite) → retrieve → **rerank** → đưa context vào prompt → generate → trả lời + citation; chỗ vỡ: chunking sai kích cỡ, retrieval trượt (cần hybrid BM25+dense), thiếu rerank, context tràn, không grounding, **không có eval**. "RAG sai dù tài liệu có thông tin đúng → debug từ retrieval trước, không phải model".

**R-AIARCH-005** ⭐⭐⭐
"MCP (Model Context Protocol) là gì và nó chuẩn hoá vấn đề gì cho backend?"
> *Dò cái gì:* lớp giao thức chuẩn để agent/model kết nối **tool & data source** mà không cần code tích hợp riêng từng cái (tương tự "USB cho tool của AI"); tách business logic khỏi model cụ thể; server MCP phơi bày tool/resource theo interface chuẩn. *(verify trạng thái chuẩn)*

**R-AIARCH-006** ⭐⭐⭐⭐
"MCP vs gọi API/CLI trực tiếp cho agent truy cập tool — đánh đổi gì? Khi nào MCP là *thừa*?"
> *Dò cái gì:* MCP mạnh ở **auth/OAuth chuẩn hoá, khả năng cắm-rút, governance, tái dùng**; nhược: **chi phí token/context lớn** để load schema, thêm tầng vận hành; pipeline production hẹp/đơn giản ưu tiên token-efficiency → API/CLI trực tiếp có khi gọn hơn. Quyết theo nhu cầu, không theo hype. *(verify)*

**R-AIARCH-007** ⭐⭐⭐
"Agent = vòng *think–act–observe*. Tính năng *agentic* cần thêm hạ tầng gì so với một LLM call stateless?"
> *Dò cái gì:* tool layer (function calling/MCP), **memory/state** (ngắn & dài hạn), orchestration/control flow (LangGraph...), eval + tracing, guardrail + giới hạn quyền tool; quản state là phần khó nhất, khác hẳn 1 lời gọi không trạng thái.

**R-AIARCH-008** ⭐⭐⭐⭐
"Multi-agent orchestration (orchestrator + sub-agent chuyên biệt): khi nào đáng dùng, và failure mode?"
> *Dò cái gì:* đáng khi task phức tạp tách được thành chuyên trách chạy song song với context riêng (vd planner → DB/API/test/security sub-agent); failure: chi phí/độ trễ nhân lên, lỗi lan truyền, khó debug, phối hợp agent-to-agent chưa chuẩn hoá; bắt đầu đơn giản, chỉ thêm khi *cái cụ thể* vỡ.

**R-AIARCH-009** ⭐⭐⭐⭐
"Vì sao 'memory' của agent không chỉ là vector similarity? Footgun phổ biến khi làm memory?"
> *Dò cái gì:* memory cần *kiến trúc* (tóm tắt, ưu tiên, quên, phiên/đa phiên), không chỉ nhét rồi top-k; footgun: ghi memory **đồng bộ chặn response** (thêm latency người dùng cảm thấy) → nên async; thiếu rerank nên lấy đúng candidate sai thứ tự. *(verify công cụ)*

**R-AIARCH-010** ⭐⭐⭐⭐⭐
"Đánh giá một tính năng LLM/agent *không có ground-truth* thế nào?" *(observability sâu → O, adjacent)*
> *Dò cái gì:* golden/eval set + tiêu chí; **LLM-as-judge** (kèm cảnh giác bias/đắt); offline eval + online (A/B, feedback); metric: hallucination/faithfulness, context recall, latency, cost/req; **regression gate** trước mỗi đổi prompt/model. Trỏ O cho tracing/observability.

**R-AIARCH-011** ⭐⭐⭐⭐
"Thành phần phi tất định trong backend (LLM): làm sao có độ tin cậy đủ để production?"
> *Dò cái gì:* `temperature` thấp cho task cần ổn định; structured output (tool/function calling) thay regex; retry + timeout + **fallback** (model rẻ/heuristic); **idempotency** cho action; guardrail input/output; coi LLM như dependency có thể sai → degrade duyên dáng. (liên hệ "cùng prompt sao ra khác" = decoding).

**R-AIARCH-012** ⭐⭐⭐⭐
"Kiểm soát *chi phí & latency* khi gọi LLM ở scale?"
> *Dò cái gì:* **semantic/exact cache**, **model routing** (rẻ cho phần dễ, mạnh cho phần khó), batching, giảm context/token, prompt gọn, streaming cho cảm nhận latency, token budget + hạn mức theo tenant; đo cost/req như một SLO.

**R-AIARCH-013** ⭐⭐⭐
"Agent có tool + truy cập data của anh: prompt injection / data exfiltration đe doạ thế nào?" *(threat model → M)*
> *Dò cái gì:* nội dung không tin cậy (doc/web/email) có thể *chèn chỉ thị* khiến agent gọi tool xấu / rò dữ liệu; phòng: tách quyền tool, least-privilege, người duyệt action ghi, lọc/đánh dấu nội dung untrusted, không nhét secret vào context. Trỏ M cho chi tiết.

**R-AIARCH-014** ⭐⭐⭐⭐⭐
"Khi nào KHÔNG nên dùng LLM cho một bài toán backend?"
> *Dò cái gì:* cần **tất định/đúng tuyệt đối** (tính tiền, pháp lý), cần auditability/giải thích, latency/cost không chịu nổi, có giải pháp rule/ML cổ điển rẻ-chắc hơn, dữ liệu quá nhạy cảm; dùng LLM vì 'hợp việc', không vì 'đang hot'.

**R-AIARCH-015** ⭐⭐⭐
"'Model là commodity, *data* mới là moat' — câu này định hình cách anh kiến trúc *data layer* cho agent thế nào?"
> *Dò cái gì:* đầu tư vào độ tươi/chất lượng/khả-truy-cập của dữ liệu (CDC giữ view cập nhật, embedding luôn fresh, phơi qua giao thức chuẩn cho agent), thay vì chạy theo model mới nhất; agent chỉ tốt bằng data nó với tới. (liên hệ CDC/Outbox ở K).

**R-AIARCH-016** ⭐⭐⭐⭐
"Agent được phép gọi *tool ghi* (đặt hàng, refund, gửi mail): thiết kế cho an toàn & idempotent thế nào?" *(idempotency → I/F)*
> *Dò cái gì:* dry-run/preview + **human-in-the-loop approval** cho action rủi ro, idempotency key, giới hạn **blast radius** (quota, allowlist tool, môi trường sandbox), audit log, khả năng rollback/compensate (liên hệ Saga ở K). Tự chủ cao → kiểm soát phải tăng tương ứng.

---

## R-EDGE — Edge computing & Serverless 2.0

**R-EDGE-001** ⭐⭐⭐
"Edge compute là gì, và *logic nào* nên ở edge vs origin?" *(trade-off lõi serverless/edge → A-SVL-003)*
> *Dò cái gì:* chạy code ở PoP gần user → cắt round-trip; hợp: auth/JWT nhẹ, A/B, redirect/rewrite, personalization cache, rate-limit cạnh; KHÔNG: logic nặng/stateful, giao dịch cần central DB, data-heavy. Trỏ A-SVL cho nguyên tắc đặt logic.

**R-EDGE-002** ⭐⭐⭐
"V8 isolates vs container vs FaaS truyền thống — vì sao cold start ở edge xuống sub-ms?"
> *Dò cái gì:* isolate = nhiều "ngăn" cô lập trong **một** V8 process (như Chrome tab), không boot container/VM riêng → khởi động ~ms, mật độ cao; FaaS container boot 50–500ms; đánh đổi: ngăn ngôn ngữ/runtime API so với container đầy đủ. *(verify số)*

**R-EDGE-003** ⭐⭐⭐⭐
"'Serverless 2.0 + Wasm@edge sẽ khai tử Kubernetes' — đánh giá luận điểm này như một TL."
> *Dò cái gì:* nhận ra đây là **bài hype phổ biến**; đúng cho workload stateless, sự kiện, geo-distributed, cold-start nhạy; SAI khi cần stateful phức tạp, đa service nội bộ, control hạ tầng, workload nặng/đặc thù; K8s & edge **đồng tồn tại**, chọn theo bài toán. Trỏ A-SVL / N. *(verify)*

**R-EDGE-004** ⭐⭐⭐⭐
"State & data ở edge khó ở đâu, và có những pattern data-at-edge nào?"
> *Dò cái gì:* edge phân tán toàn cầu → consistency & data gravity là vấn đề; pattern: edge KV/cache (đọc nhanh, eventual), **edge DB** (Turso/libSQL, Cloudflare D1, replica gần), ghi về region trung tâm; chấp nhận eventual cho đọc, giữ ghi nhất quán ở core. *(verify sản phẩm)*

**R-EDGE-005** ⭐⭐⭐
"Edge runtime khác Node ở đâu? Vì sao app Nest/Express của anh có thể *không 'just run'* ở edge?"
> *Dò cái gì:* runtime edge (Workers/Vercel Edge) dựa Web API, **thiếu nhiều Node API** (fs, net thô, một số lib native), giới hạn CPU/time/bộ nhớ per-request; cần code tương thích (fetch, Web Streams), không phụ thuộc filesystem/long-lived connection.

**R-EDGE-006** ⭐⭐⭐⭐
"Phục vụ logic personalized/auth gần user mà KHÔNG gây bug đúng-sai: cân latency vs consistency thế nào?"
> *Dò cái gì:* đọc/verify token + cache personalization ở edge (nhanh, chấp nhận hơi cũ), nhưng **nguồn sự thật về quyền/giao dịch ở core**; tránh quyết định bảo mật chỉ dựa data edge có thể stale; phân tách rõ "fast path đọc" vs "correct path ghi".

**R-EDGE-007** ⭐⭐⭐
"Cloudflare Workers / Fastly Compute / Vercel Edge — mỗi cái nhắm class workload nào?" *(verify đặc tính)*
> *Dò cái gì:* hiểu *đại ý*: Workers (JS/Wasm isolates, KV/D1/Durable Objects), Fastly Compute (Wasm-first, hiệu năng), Vercel Edge (gắn frontend/Next middleware); không cần thuộc feature-matrix — biết "đây là edge-functions + storage kèm" và **verify chi tiết tại docs**.

**R-EDGE-008** ⭐⭐⭐⭐
"Một request cần Postgres trung tâm: đẩy logic ra edge giúp hay hại?" *(connection/pool → E-427)*
> *Dò cái gì:* nếu mọi việc vẫn phải gọi DB ở 1 region → edge **thêm** một chặng, có khi *chậm hơn*; data gravity kéo compute về gần data; edge chỉ lợi khi việc xử lý được *gần user* (cache/độc lập DB). Connection từ nhiều edge → bài toán pool/proxy (trỏ E).

**R-EDGE-009** ⭐⭐⭐
"Cost model edge/serverless (per-request) vs always-on — khi nào mỗi cái thắng?"
> *Dò cái gì:* per-request thắng cho tải **spiky/thấp/không đều** (không trả lúc rảnh, scale-to-zero); always-on/reserved thắng cho tải **cao & ổn định liên tục** (per-request đắt khi volume lớn) + tránh cold start; tính theo profile tải thực, không theo nhãn 'rẻ'.

**R-EDGE-010** ⭐⭐⭐⭐⭐
"Thiết kế: read path nhanh toàn cầu + write path nhất quán mạnh. Anh chẻ edge vs core ra sao?"
> *Dò cái gì:* read = replica/cache/edge gần user (eventual ok), write = về region core có nhất quán mạnh + single source of truth; xử lý read-your-write (route user vừa ghi về primary/sticky), invalidation; nêu rõ đánh đổi consistency theo từng path. *(consistency → I)*

**R-EDGE-011** ⭐⭐⭐
"Vendor lock-in ở edge: code edge của anh *portable* tới đâu, và hedge thế nào?"
> *Dò cái gì:* mỗi nền có API/storage riêng (KV, Durable Objects, D1...) → khó port; hedge: tách business logic khỏi API nền tảng (adapter), bám chuẩn Web/Wasm/WASI khi được, chấp nhận lock-in có chủ đích nếu giá trị đủ lớn — nhưng *biết mình đang đổi gì*.

**R-EDGE-012** ⭐⭐⭐
"Observability/debug ở edge vì sao khó hơn một region duy nhất?" *(tracing sâu → O, adjacent)*
> *Dò cái gì:* hàng trăm PoP toàn cầu, ngắn-sống, log phân tán, khó reproduce theo địa lý; cần tracing tập trung + sampling + correlation id; nhận ra "khó quan sát" là một *chi phí thật* của edge. Trỏ O.

---

## R-WASM — WebAssembly server-side

**R-WASM-001** ⭐⭐
"WebAssembly là gì, và vì sao 2026 nó là chuyện *server/edge* chứ không chỉ browser?"
> *Dò cái gì:* định dạng bytecode portable, sandboxed, chạy gần tốc độ native, khởi động µs–ms; ngoài browser nay chạy ở edge/serverless/plugin/embedded nhờ runtime + WASI; "viết một lần, chạy x86/ARM/RISC-V". *(verify)*

**R-WASM-002** ⭐⭐⭐
"WASI thay đổi điều gì khiến Wasm server-side *dùng được thật*?"
> *Dò cái gì:* trước WASI, Wasm module ~ hàm thuần không I/O; **WASI** = interface chuẩn để xin host quyền file/network/env/time → module thành chương trình thật; lưu ý đời WASI (0.2 mặc định, 0.3 async đang tới). *(verify)*

**R-WASM-003** ⭐⭐⭐
"Wasm module vs container: so cold start, kích thước, cô lập — đánh đổi thật là gì?"
> *Dò cái gì:* Wasm: cold start µs–ms, ~MB, sandbox capability-based, mật độ cao; container: boot ms–s, chục–trăm MB, có user-space đầy đủ + tương thích hệ sinh thái rộng; Wasm thắng density/startup, container thắng độ chín/compat. *(verify số)*

**R-WASM-004** ⭐⭐⭐⭐
"Mô hình bảo mật/sandbox của Wasm ('capability-based, no container escape') — vì sao mạnh hơn cho nhiều ca, và caveat?"
> *Dò cái gì:* Wasm mặc định **không có quyền gì** trừ thứ host cấp tường minh (deny-by-default) → bề mặt tấn công nhỏ, không 'thoát container' kiểu chia sẻ kernel; caveat: phụ thuộc host/WASI bind đúng, lỗ hổng runtime, side-channel; mạnh cho *chạy code không tin cậy* (multi-tenant, plugin).

**R-WASM-005** ⭐⭐⭐
"Khi nào Wasm hợp trong backend, khi nào không?"
> *Dò cái gì:* hợp: plugin/extension sandboxed, chạy code khách hàng (multi-tenant), edge function, đoạn CPU-nặng (crypto/codec/parse/inference); không hợp: app CRUD thường (toolchain/lib chưa chín, syscall ngoài WASI thiếu), khi container đã đủ tốt. Đừng dùng vì hype.

**R-WASM-006** ⭐⭐⭐⭐
"Component Model + gọi chéo ngôn ngữ cho phép gì, và vì sao quan trọng?"
> *Dò cái gì:* định nghĩa interface để các module Wasm (Rust, Go, JS...) **ghép lại & gọi nhau** qua interface chuẩn, không cần cùng ngôn ngữ; mở đường tái dùng component đa ngôn ngữ + hệ sinh thái plugin; là bước đưa Wasm từ 'hàm' lên 'thành phần phần mềm'. *(verify trạng thái)*

**R-WASM-007** ⭐⭐⭐
"Lỗ hổng *tooling* nào anh cảnh báo team trước khi đặt cược vào Wasm?" *(verify)*
> *Dò cái gì:* debug xuyên biên ngôn ngữ còn vụng, một số thư viện chưa compile sạch sang Wasm (đụng syscall ngoài WASI), ecosystem/đời WASI còn dịch chuyển; → pilot nhỏ trước, chọn Rust + một edge function làm bước đầu, **verify lib support**.

**R-WASM-008** ⭐⭐⭐⭐
"Wasm như *plugin runtime* trong app (Envoy/Istio/extension DB): vì sao plugin sandboxed thắng nạp .so native?"
> *Dò cái gì:* plugin native (.so) chạy cùng quyền với host → crash/độc hại = sập cả app; Wasm plugin **cô lập + capability**, nạp/gỡ động, đa ngôn ngữ, an toàn hơn cho code bên thứ ba; đổi lại overhead biên + giới hạn API.

**R-WASM-009** ⭐⭐⭐
"Wasm vs JS *ở edge* — cái gì đẩy anh sang Wasm?"
> *Dò cái gì:* JS edge nhanh & tiện cho I/O/glue; sang Wasm khi cần **CPU-nặng & hiệu năng đoán trước**: crypto, xử lý ảnh, parse nặng, inference; viết bằng Rust/C++ compile ra .wasm nhỏ, chạy cạnh JS. *(verify)*

**R-WASM-010** ⭐⭐⭐⭐⭐
"Là TL, anh đánh giá Wasm đã *production-ready cho ca của mình* (2026) hay chỉ hype — bằng tiêu chí gì?"
> *Dò cái gì:* khung đánh giá: use case có thật khớp (plugin/edge/CPU-nặng) không?, lib/toolchain đã hỗ trợ?, đội có Rust/Go skill?, đo benchmark thật vs container?, rủi ro WASI version & reversibility?; kết luận theo dữ liệu, không theo bài blog "Wasm rules 2026".

---

## R-PLAT — Platform Engineering / IDP

**R-PLAT-001** ⭐⭐
"Internal Developer Platform (IDP) giải quyết vấn đề gì *ở scale*?" *(định nghĩa + golden path → N-TL-006, không lặp)*
> *Dò cái gì:* khi nhiều team/nhiều service, mọi dev tự làm chủ mọi tầng = quá tải nhận thức + phân mảnh; IDP cung cấp **self-service + abstraction + golden path** để ship độc lập mà không phải thành chuyên gia K8s. (Định nghĩa cốt lõi ở N — ở R bàn sâu hơn dưới.)

**R-PLAT-002** ⭐⭐⭐
"'Platform engineering chỉ là DevOps đổi tên' — phản biện chính xác (nó vá *failure mode* nào)?"
> *Dò cái gì:* DevOps ('you build it you run it') tốt ở đội nhỏ; ở scale, văn hoá-không-đủ → mỗi team tooling/pipeline khác nhau, kiến thức phân mảnh, lặp & lệ thuộc ẩn; platform eng là **đáp ứng có cấu trúc** (sản phẩm nội bộ), không phải rebrand.

**R-PLAT-003** ⭐⭐⭐
"Golden path / paved road là gì, vì sao là *default-không-mandate*?"
> *Dò cái gì:* con đường mặc định tốt phủ ~80% ca (template, pipeline chuẩn, deploy chuẩn) để dev đi nhanh; vẫn cho **lệch khi có lý do** → giữ autonomy; mục tiêu *enable*, nếu ép cứng sẽ thành bottleneck/ticket-queue mới.

**R-PLAT-004** ⭐⭐⭐⭐
"'Platform as a product' — coi dev là *khách hàng* đổi cách làm platform thế nào?"
> *Dò cái gì:* có product management, đo **adoption/NPS/satisfaction**, roadmap theo nhu cầu dev (không mandate top-down); cạnh tranh với 'tự build' của team → buộc tập trung vào giá trị; platform tồi = 'expensive distraction'. *(verify số liệu)*

**R-PLAT-005** ⭐⭐⭐⭐
"Build vs buy: self-host **Backstage** vs SaaS (**Port/Cortex/OpsLevel**) — chi phí thật & khi nào chọn gì?" *(verify)*
> *Dò cái gì:* Backstage free nhưng TCO lớn (setup/auth/plugin React, nâng cấp, maintain → time-to-value 6–18 tháng, adoption hay thấp); đội <~200 eng nghiêng SaaS (nhanh, ít gánh vận hành); chọn theo *có platform team chuyên trách + nhu cầu tuỳ biến sâu* hay không. *(verify số/tên)*

**R-PLAT-006** ⭐⭐⭐⭐⭐
"Platform dựng xong nhưng adoption ~10% / 'portal mà không phải platform'. Vì sao thất bại & anh sửa thế nào?"
> *Dò cái gì:* team burn-out vào maintain portal, không giao thứ dev thật sự muốn; dựng UI dễ, giữ relevant khó; sửa: treat-as-product, đo adoption, golden path giải quyết pain thật, không ép dùng; system-of-record có giá trị tái dùng (incident/cost) mới kéo adoption.

**R-PLAT-007** ⭐⭐⭐
"Software catalog / system-of-record (ai sở hữu service, lifecycle, metadata) — vì sao là nền của mọi thứ khác?"
> *Dò cái gì:* khi dữ liệu ownership/lifecycle/môi trường/compliance đáng tin → tái dùng cho **incident management, cost attribution, KPI, security posture, operational review**; thiếu catalog = mỗi việc tự đi tìm 'ai chịu trách nhiệm'.

**R-PLAT-008** ⭐⭐⭐
"Khi nào team *quá nhỏ* để cần IDP? Trigger để bắt đầu?"
> *Dò cái gì:* đội nhỏ + ít service → giao tiếp trực tiếp & quy ước phi chính thức đủ, IDP là over-engineering; trigger: nhiều team/hàng chục–trăm service/đa cloud, tooling phân mảnh, onboarding chậm, autonomy gây inconsistency → lúc đó IDP đáng đầu tư.

**R-PLAT-009** ⭐⭐⭐⭐
"Metric nào chứng minh platform *đang thật sự hiệu quả* (không phải vanity)?"
> *Dò cái gì:* time-to-first-deploy của dev mới, lead time/deploy freq (**DORA**), cognitive load (khảo sát), % service trên golden path, adoption/NPS, giảm ticket tới platform team; cảnh giác Gartner-gap: có platform team ≠ có productivity gain.

**R-PLAT-010** ⭐⭐⭐⭐
"Platform 2026 'AI-augmented': AI assistant + guardrail/gate cho tốc độ do AI tạo ra — liên hệ thế nào?"
> *Dò cái gì:* AI coding đẩy tốc độ ship lên → cần IDP đặt **paved road + guardrail/gate** để tốc độ không thành hỗn loạn; platform tích hợp AI assistant (scaffold, doc, đổi diện rộng); platform engineering & AI 'merge' — speed cần governance. *(verify)*

**R-PLAT-011** ⭐⭐⭐
"FinOps & security 'embedded ở provisioning time' (shift-left) — vì sao đặt ở tầng platform?"
> *Dò cái gì:* hiện cost/carbon & policy bảo mật **trước khi deploy** (tại lúc tạo resource), không phải sau hoá đơn; platform là chỗ tự nhiên để nhúng default an toàn/đúng giá mà dev vẫn tự-service → governance không thành gatekeeper. (liên hệ R-GREEN.)

**R-PLAT-012** ⭐⭐⭐⭐
"Platform team thành *bottleneck* thay vì *enabler*: anti-pattern tổ chức & cách cấu trúc đội?"
> *Dò cái gì:* anti-pattern: platform thành 'ticket queue', mandate cứng, xa nhu cầu dev; cấu trúc: nhỏ-gọn, product mindset, đo adoption, golden path optional, lắng nghe khách hàng-nội-bộ; mục tiêu giảm cognitive load + tăng throughput toàn cục, đổi chút tự do cục bộ.

---

## R-GREEN — Green / FinOps / efficient coding

**R-GREEN-001** ⭐⭐
"FinOps là gì và vì sao cloud cost là trách nhiệm *kỹ thuật* chứ không chỉ tài chính?"
> *Dò cái gì:* FinOps = văn hoá+thực hành đưa **accountability chi phí** vào engineering; kỹ sư quyết kiến trúc/instance/scaling → quyết cost; finance một mình không tối ưu được; mục tiêu: gắn chi tiêu với giá trị, ai tạo cost người đó thấy.

**R-GREEN-002** ⭐⭐⭐
"'Cost visibility tại *thời điểm quyết định*, không phải sau hoá đơn' — đổi workflow thế nào?"
> *Dò cái gì:* surface cost ngay khi provision/PR (estimate trước deploy) → dev điều chỉnh sớm thay vì giật mình cuối tháng; dashboard tĩnh ít tác dụng; tích hợp vào IDP/CI (liên hệ R-PLAT-011).

**R-GREEN-003** ⭐⭐⭐
"GreenOps / sustainable software: vì sao 'lãng phí tiền ≈ lãng phí năng lượng'?"
> *Dò cái gì:* trong cloud, tài nguyên thừa = điện thừa = carbon thừa → tối ưu cost & carbon **đi cùng hướng** (rightsize, tắt môi trường non-prod ngoài giờ, autoscale); sustainability thành KPI vận hành cạnh cost/reliability, không phải ESG-PR.

**R-GREEN-004** ⭐⭐⭐
"Instance ARM/Graviton — 'double win' là gì và caveat khi migrate?"
> *Dò cái gì:* thường **giá-hiệu năng tốt hơn + năng lượng/chu kỳ thấp hơn** x86; caveat: cần build/đa-kiến-trúc ARM, kiểm thư viện/dependency native tương thích, một số workload chưa tối ưu ARM → benchmark trước. *(verify)*

**R-GREEN-005** ⭐⭐⭐
"Carbon-aware scheduling: dời batch/training theo vùng/giờ ít carbon — áp dụng ở đâu, *không* ở đâu?"
> *Dò cái gì:* hợp **batch/training trễ-được** (dời tới lúc/vùng grid sạch hơn, không phạm SLA); KHÔNG hợp request latency-nhạy/realtime; cần data cường độ carbon theo vùng-giờ; lợi ích directional là đủ để đổi hành vi.

**R-GREEN-006** ⭐⭐⭐⭐
"'Default instance size là kẻ thù của hiệu quả' — anh tìm & sửa over-provisioning thế nào?"
> *Dò cái gì:* profiling thật (CPU/mem/utilization) thay vì size mặc định; rightsize theo dữ liệu, autoscale/scale-to-zero cho spiky, dời workload spiky khỏi infra always-on; tool FinOps đưa khuyến nghị + tự động hoá remediation (không để backlog).

**R-GREEN-007** ⭐⭐⭐⭐
"'Carbon budget trong CI/CD' / cost-gate — khả thi hay 'theater'? Anh implement directional thế nào?"
> *Dò cái gì:* khả thi ở mức *directional*: ước lượng cost/carbon delta của thay đổi, cảnh báo/fail khi vượt ngưỡng (như test gate); không cần chính xác tuyệt đối, đủ để đổi hành vi; rủi ro thành theater nếu không gắn quyết định thật. *(verify công cụ)*

**R-GREEN-008** ⭐⭐⭐
"AI workload làm cost/năng lượng phình ở đâu, và backend TL nắm *đòn bẩy* nào?" *(ties R-AIARCH-012)*
> *Dò cái gì:* inference/embedding/token tốn compute+điện; đòn bẩy: chọn model đúng kích cỡ (không 'frontier' cho việc dễ), **cache**, batching, model routing, token budget, quantize/edge-inference khi hợp; coi cost/carbon của AI như SLO.

---

## R-TL — Gu đánh giá trend (TL judgment)

**R-TL-001** ⭐⭐⭐
"Một bài viết tuyên bố 'X đã chết, dùng Y'. Khung của anh để tách *hype* khỏi *substance*?"
> *Dò cái gì:* hỏi: vấn đề thật Y giải là gì? ai đang dùng production & ở quy mô nào? trade-off & 'khi nào KHÔNG'? độ chín toolchain/lib? reversibility? nguồn có động cơ bán hàng không?; trend ổn định (pattern) vs dễ đổi (tên/đời/số) — tin pattern, verify số.

**R-TL-002** ⭐⭐⭐⭐
"Khi nào *adopt* một trend mới vs *chờ*? Tiêu chí?"
> *Dò cái gì:* Gartner hype-cycle (đỉnh ảo vs plateau giá trị), **reversibility/blast radius**, mức trưởng thành & ecosystem, team readiness/skill, chi phí migration vs lợi ích, có pilot nhỏ đo được không; adopt sớm cho lợi thế lớn + rủi ro kiểm soát được, chờ khi đắt-không-hoàn-lại.

**R-TL-003** ⭐⭐⭐⭐
"Một engineer hào hứng: 'mình nên viết lại bằng Rust / chuyển Wasm / all-agentic'. Anh phản hồi thế nào?"
> *Dò cái gì:* không dập nhiệt huyết nhưng đòi **vấn đề cụ thể** đang giải; đề xuất pilot nhỏ/spike đo dữ liệu; cân nhắc chi phí rewrite vs lợi ích, rủi ro, skill, reversibility; quyết theo bằng chứng, ghi nhận để thử có kiểm soát — lãnh đạo bằng câu hỏi, không bằng 'không'.

**R-TL-004** ⭐⭐⭐
"Anh giữ bản thân & team *cập nhật* mà không chạy theo mọi thứ shiny thế nào?"
> *Dò cái gì:* nguồn tín hiệu chất lượng (radar công nghệ, báo cáo DORA/CNCF, người làm thật), thử nghiệm có chủ đích (spike/pilot, 20% time), phân biệt 'cần biết để bàn' vs 'cần dùng', chia sẻ nội bộ; kỷ luật chọn lọc thay vì FOMO.

**R-TL-005** ⭐⭐⭐⭐
"Quyết định *đảo được* vs *không đảo được* (one-way / two-way door) áp vào việc theo trend thế nào?"
> *Dò cái gì:* two-way door (đảo rẻ) → thử nhanh, học bằng làm; one-way door (đổi schema/dữ liệu/lock-in sâu/ngừng dịch vụ) → cẩn trọng, cần dữ liệu & buy-in; nhiều quyết 'trend' là two-way nếu bọc adapter → cho phép thử mà không cược cả hệ thống.

**R-TL-006** ⭐⭐⭐⭐⭐
"Risk governance cho AI *tự chủ* chạy trong production (agent ra hành động): cần controls gì *trước khi* ship?"
> *Dò cái gì:* least-privilege tool + allowlist, **human-in-the-loop** cho action rủi ro, blast-radius limit/quota, dry-run + rollback/compensate, audit log đầy đủ, eval/guardrail chống injection, kill-switch, trách nhiệm pháp lý rõ; tự chủ tăng → control tăng tương ứng. (liên hệ R-AIARCH-016, M.)

**R-TL-007** ⭐⭐⭐
"'Kubernetes có đang thành legacy không?' — trả lời cân bằng kiểu TL."
> *Dò cái gì:* không vơ đũa; K8s vẫn là chuẩn cho orchestration nhiều service/scale/đa team; với app nhỏ/stateless/spiky → serverless/edge/Wasm thường hợp hơn (đỡ gánh vận hành); 'legacy' là sai khung — đúng khung là **chọn theo bài toán**, nhiều paradigm đồng tồn tại. Trỏ A-SVL / N. *(verify)*

**R-TL-008** ⭐⭐⭐⭐
"Một trend anh thấy *đang bị thổi phồng* lúc này, và lý do? (không có đáp án 'đúng' — chấm phán đoán)"
> *Dò cái gì:* chọn một trend + lập luận có cấu trúc: phân biệt phần giá trị thật vs marketing, 'khi nào KHÔNG', bằng chứng/đối chứng, không cực đoan (không 'vô dụng' lẫn 'cứu rỗi'); chấm **chất lượng lý luận & cân bằng**, không phải chọn đúng/sai.

---

## Ghi chú dùng bộ đề (cho Bước B–E)

- **Dựng giáo trình ngược (Bước B):** R không có dependency chặt như N, nhưng nên dạy theo cụm *từ gần backend nhất → trừu tượng hơn*: **R-AICODE** (đang dùng hằng ngày) → **R-AIARCH** (xây tính năng AI) → **R-EDGE** + **R-WASM** (paradigm hạ tầng mới) → **R-PLAT** + **R-GREEN** (vận hành/tổ chức ở scale) → **R-TL** (gu đánh giá, học cuối vì nó *tổng hợp* phán đoán). Có thể học R-TL xen kẽ như "lăng kính" áp lên từng cụm.
- **Trọng tâm theo roadmap:** R chỉ ~15% tần suất và kỳ vọng ở TL là *"bàn được + cho thấy mình cập nhật + có gu"*, KHÔNG phải implement sâu. → ưu tiên đạt vững **R-AICODE, R-AIARCH (RAG/agent/eval), R-PLAT, R-TL**; **R-EDGE/R-WASM/R-GREEN** đạt mức *"giải thích + trade-off + khi nào KHÔNG"*; các câu ⭐⭐⭐⭐⭐ là điểm cộng "senior vs TL".
- **Khi chấm live (Bước D):** R là mục **đổi nhanh nhất** → mọi câu chạm **tên/đời sản phẩm, version, con số** (WASI 0.x, model/giá, % Gartner, đặc tính Workers/Fastly/Vercel, đời Backstage/Port) → yêu cầu người học nói rõ *"cái này cần verify ở nguồn chính thức"* thay vì khẳng định số; đúng tinh thần **"hiểu để chỉ huy & kiểm tra, phần còn lại tra cứu"**. Mốc ĐẠT nằm ở **pattern & trade-off**, không ở việc nhớ con số.
- **Spaced repetition (Bước E):** trộn câu cũ đã đạt ở các đợt sau (hôm nay → mai → 3 ngày → 1 tuần). R đặc biệt hợp để **xen vào** khi học các mục khác: ví dụ học A (System Design) thì rút R-EDGE/R-AIARCH ra hỏi, học N (DevOps) thì rút R-PLAT/R-GREEN.
