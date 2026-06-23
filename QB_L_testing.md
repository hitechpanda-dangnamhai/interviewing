# 🧪 QB_L — Ngân hàng câu hỏi: Testing & Quality

> **Mục L** trong roadmap Tech Lead Backend (🟡 ~40%) · Sinh theo **WORKFLOW 2 — Bước A**.
> Đây là **mục LỚN** → bộ đề phủ HẾT 6 mục con roadmap (test pyramid · unit/integration/e2e · TDD · mocking/test double · TestContainers/DB thật · đặt chuẩn chất lượng team) + mở rộng cấp TL (flaky test, coverage/mutation, load test, contract test, CI quality gate).
> **Chỉ câu hỏi — KHÔNG kèm đáp án.** Mỗi câu có dòng *"dò cái gì"* = tiêu chí ĐẠT, dùng để chấm **live** ở Bước D.
> **Tổng: 85 câu.**

## Chống trùng (đã đọc các QB sẵn có trong project)
- **`QB_H_nestjs.md` → nhóm `H-TST` (7 câu)** giữ phần **đặc thù NestJS**: `Test.createTestingModule()`, `overrideProvider/overrideGuard`, `createNestApplication` + Supertest *trong Nest*, Nest 12 đổi Jest→Vitest. → **L KHÔNG lặp cơ chế binding của Nest**; L bàn testing ở mức **kỷ luật/tool-agnostic** (khái niệm, chiến lược, runner generic). Khi câu L chạm Nest → cross-ref H.
- **`QB_F_apidesign.md`** đã có câu *"Contract testing (Pact) giải quyết vấn đề gì giữa consumer/producer"* (consumer-driven, CI gate). → **L chỉ giữ 3 câu** về **vị trí contract test trong pyramid + cơ chế gating (can-i-deploy/broker)**, cross-ref F, không lặp phần "giải quyết vấn đề gì".
- **`QB_Q_designpatterns.md`** sở hữu góc **DIP → testability** (vì sao interface giúp test, seam). → **L sở hữu *bảng phân loại test double* (mock/stub/spy/fake/dummy)** và cách *áp dụng* seam khi test, không lặp phần "vì sao thiết kế thế".

## Cách đọc
- **ID:** `L-<tag>-<số>`. Tag mục con liệt kê ở mục lục dưới.
- **Độ khó:** ⭐ định nghĩa cơ bản → ⭐⭐⭐⭐⭐ phân biệt senior với TL thật.
- **"Dò cái gì":** điều người trả lời PHẢI chạm tới mới tính ĐẠT (không phải đáp án — chỉ là mốc chấm).

## Bối cảnh phiên bản (tính tại 6/2026 — *verify lại tại docs khi học*)
- **Runner JS/TS 2026:** **Jest 30.x** (Meta) vẫn là default lâu đời, an toàn cho codebase lớn/legacy, React Native, Angular. **Vitest 4.x** (Vite-native, ESM/TS sẵn, watch nhanh hơn) là *khuyến nghị cho project mới*. Ngoài ra có **`node:test`** (native Node) và **`bun test`** (khi đã chạy Bun, suite chủ yếu backend TS). → Câu hỏi "chọn runner nào" không có đáp án tuyệt đối: cân theo stack/ESM/ecosystem.
- **HTTP e2e:** **Supertest** (gắn vào HTTP server in-process). **Integration DB thật:** **TestContainers**. **Load test:** **k6** (JS, hiện thuộc Grafana) / **Artillery**. **Contract test:** **Pact**. **Mutation test:** **Stryker** (StrykerJS).
- ⚠️ Mọi câu chạm version/tooling đổi nhanh đều có dấu *(verify)* — đối chiếu docs chính thức trước khi tin.

---

## Mục lục mục con (ID prefix → số câu)

| Tag | Mục con | Số câu |
|---|---|---|
| **L-PYR** | Test pyramid & chiến lược test (pyramid vs trophy, tỉ lệ, push-down) | 8 |
| **L-TYP** | Loại test & ranh giới (unit / integration / component / e2e) | 7 |
| **L-TDD** | TDD & BDD (red-green-refactor, khi nào hợp, Gherkin) | 7 |
| **L-DBL** | Test double taxonomy & chiến lược mocking (classicist vs mockist) | 10 |
| **L-RUN** | Runner & tooling generic (Jest/Vitest/node:test/Supertest) | 8 |
| **L-INT** | Integration test & TestContainers (góc kỷ luật, cross-ref H) | 7 |
| **L-FLK** | Flaky test, determinism, isolation, song song | 8 |
| **L-DAT** | Test data (fixture/factory/seed/cleanup) & snapshot test | 6 |
| **L-COV** | Coverage & mutation testing & metric chất lượng | 7 |
| **L-LOAD** | Load / performance test (k6 / Artillery, soak/spike/stress) | 6 |
| **L-CON** | Contract test — vị trí trong pyramid & CI gating (cross-ref F) | 3 |
| **L-TL** | Chuẩn chất lượng cấp Tech Lead (CI gate, DoD, review, legacy) | 8 |

---

## L-PYR — Test pyramid & chiến lược test

**L-PYR-001** ⭐⭐
"Test pyramid là gì? Mô tả 3 tầng và nguyên tắc cốt lõi của nó."
> *Dò cái gì:* nhiều unit (nhanh, rẻ, ổn định) ở đáy → ít integration ở giữa → rất ít e2e (chậm, brittle) ở đỉnh; nguyên tắc "push test xuống tầng thấp nhất có thể kiểm chứng được".

**L-PYR-002** ⭐⭐⭐
"Vì sao KHÔNG nên đảo ngược pyramid thành 'ice-cream cone' (nhiều e2e, ít unit)? Triệu chứng của anti-pattern này?"
> *Dò cái gì:* e2e chậm + flaky + khó debug → CI dài, mất niềm tin, feedback loop kém; triệu chứng: suite chạy hàng chục phút, hay đỏ vu vơ, dev bỏ qua test.

**L-PYR-003** ⭐⭐⭐
"'Testing trophy' (Kent C. Dodds) khác test pyramid ở đâu? Khi nào triết lý 'integration-heavy' hợp lý hơn?"
> *Dò cái gì:* trophy nhấn integration là tầng cho ROI cao nhất (test gần hành vi thật mà vẫn đủ nhanh); hợp khi nhiều logic nằm ở ráp nối I/O/HTTP hơn là thuật toán thuần.

**L-PYR-004** ⭐⭐⭐⭐
"Là TL, anh phân bổ ngân sách test (thời gian + coverage) cho một service CRUD vs một service tính toán nghiệp vụ phức tạp khác nhau thế nào?"
> *Dò cái gì:* dịch hình dạng pyramid theo bản chất rủi ro: service logic-nặng → đầu tư unit; service ráp-nối-nặng (glue, I/O) → nghiêng integration; không áp một tỉ lệ cứng cho mọi service.

**L-PYR-005** ⭐⭐⭐
"'Pushing test down the pyramid' cụ thể là làm gì khi anh thấy một hành vi đang chỉ được phủ bởi e2e?"
> *Dò cái gì:* tách logic ra khỏi I/O để unit hoá; thay e2e bằng integration ở ranh giới module; giữ e2e chỉ cho happy-path xuyên hệ thống/critical journey.

**L-PYR-006** ⭐⭐⭐⭐
"Trade-off giữa 'độ tin cậy (confidence)' và 'tốc độ/chi phí' khi leo lên các tầng pyramid là gì? Cho ví dụ một bug chỉ e2e bắt được."
> *Dò cái gì:* càng lên cao càng giống prod (confidence ↑) nhưng chậm/đắt/brittle ↑; ví dụ: lỗi cấu hình routing, middleware order, CORS, serialization xuyên tầng mà unit/mock không lộ.

**L-PYR-007** ⭐⭐⭐
"Test pyramid áp cho microservices/distributed system có gì khác monolith? Vì sao 'full e2e xuyên nhiều service' bị hạn chế?"
> *Dò cái gì:* e2e xuyên nhiều service cực brittle & tốn hạ tầng; thay bằng contract test giữa service + integration trong từng service; chỉ giữ rất ít smoke e2e xuyên hệ thống.

**L-PYR-008** ⭐⭐⭐⭐⭐
"Coverage 90% nhưng pyramid lệch (toàn e2e mock nông) — vì sao con số coverage có thể 'lừa' anh? Anh đánh giá *chất lượng* test suite bằng gì ngoài tỉ lệ tầng?"
> *Dò cái gì:* coverage đo dòng chạy qua, không đo assertion có nghĩa; nhìn thêm tốc độ suite, tỉ lệ flaky, mutation score, defect-escape-rate, thời gian feedback.

---

## L-TYP — Loại test & ranh giới

**L-TYP-001** ⭐⭐
"Định nghĩa unit / integration / e2e test. 'Unit' của anh là một hàm hay một class/module — ranh giới đặt ở đâu?"
> *Dò cái gì:* unit = đơn vị cô lập khỏi dependency thật; integration = nhiều đơn vị/ranh giới I/O thật ráp lại; e2e = xuyên toàn hệ thống như user; thừa nhận "unit" co giãn (sociable vs solitary).

**L-TYP-002** ⭐⭐⭐
"Solitary unit test vs sociable unit test khác nhau gì? Mỗi loại kéo theo phong cách mocking nào?"
> *Dò cái gì:* solitary = mock hết collaborator (mockist); sociable = để collaborator thật chạy cùng (classicist); ảnh hưởng độ giòn khi refactor & độ tin cậy.

**L-TYP-003** ⭐⭐⭐
"Khi nào một bài là 'integration test' chứ không phải 'unit test có mock'? Đường ranh thực dụng anh dùng để phân loại?"
> *Dò cái gì:* integration chạm dependency *thật* qua ranh giới (DB/HTTP/queue/file); nếu mọi out-of-process đều bị mock → vẫn là unit dù gọi nhiều class.

**L-TYP-004** ⭐⭐⭐
"Backend không-UI thì 'e2e' nghĩa là gì? Test cái gì cho một service chỉ phơi REST/gRPC?"
> *Dò cái gì:* e2e/API-level = bắn request thật qua HTTP vào app đã wire đầy đủ (pipe/guard/DB), assert status/body/side-effect (DB row, message phát ra), không cần UI.

**L-TYP-005** ⭐⭐⭐⭐
"'Component test' (test một service/module qua public boundary với dependency ngoài bị mock ở mạng) nằm ở đâu giữa integration và e2e? Lợi gì cho microservices?"
> *Dò cái gì:* test một service trọn vẹn nhưng cô lập khỏi service khác (mock HTTP ngoài, DB thật); nhanh & ổn định hơn e2e xuyên hệ thống mà vẫn phủ wiring nội bộ.

**L-TYP-006** ⭐⭐⭐
"Smoke test / sanity test khác regression test thế nào? Vai trò của smoke trong pipeline deploy?"
> *Dò cái gì:* smoke = vài kiểm tra critical-path nhanh sau deploy để 'còn sống không'; regression = phủ rộng tránh tái phát bug; smoke chạy ở gate post-deploy/canary.

**L-TYP-007** ⭐⭐⭐⭐
"Cùng một hành vi nghiệp vụ, khi nào anh chọn integration test thay vì viết thêm unit test (và ngược lại)? Tiêu chí quyết định?"
> *Dò cái gì:* logic/nhánh/biên → unit (nhanh, nhiều case); rủi ro nằm ở ráp nối/SQL/serialization/transaction → integration; tránh phủ trùng cùng thứ ở nhiều tầng (lãng phí + brittle).

---

## L-TDD — TDD & BDD

**L-TDD-001** ⭐⭐
"Vòng lặp TDD red-green-refactor gồm các bước nào? Vì sao 'viết test trước' lại quan trọng chứ không chỉ là thứ tự?"
> *Dò cái gì:* viết test đỏ → code tối thiểu cho xanh → refactor giữ xanh; test-first ép nghĩ về interface/spec & testability trước khi implement, tránh viết test 'vừa khít code sai'.

**L-TDD-002** ⭐⭐⭐
"Khi nào TDD hợp lý, khi nào KHÔNG? Cho ví dụ task mà TDD strict gây cản trở."
> *Dò cái gì:* hợp khi logic rõ/biên nhiều/critical; kém hợp khi đang exploratory/prototype, UI/integration nặng, hoặc chưa rõ thiết kế → spike trước rồi mới TDD.

**L-TDD-003** ⭐⭐⭐⭐
"TDD có cải thiện *thiết kế* code không, hay chỉ tạo lưới an toàn? Lập luận hai chiều."
> *Dò cái gì:* test-first đẩy về low coupling/seam rõ (vì code khó test = code coupling cao); nhưng TDD máy móc có thể đẻ over-mock/over-abstraction; design tốt vẫn cần chủ ý, không tự sinh.

**L-TDD-004** ⭐⭐⭐
"BDD khác TDD ở đâu (focus + ngôn ngữ)? Given-When-Then (Gherkin) dùng để làm gì và bẫy của nó?"
> *Dò cái gì:* BDD mô tả *hành vi từ góc người dùng*, ngôn ngữ tự nhiên cho cả non-dev; bẫy: tốn công maintain step-definition, dễ biến thành e2e brittle nếu lạm dụng.

**L-TDD-005** ⭐⭐⭐
"'Inside-out' (classicist) vs 'outside-in' (London/mockist) TDD khác nhau thế nào về điểm bắt đầu và cách dùng double?"
> *Dò cái gì:* outside-in bắt đầu từ ngoài (acceptance) đẩy vào trong, mock collaborator chưa có; inside-out xây từ domain core ra; ảnh hưởng lượng mock & thứ tự phát triển.

**L-TDD-006** ⭐⭐⭐⭐
"Người ta nói TDD làm 'chậm lúc đầu nhanh về sau'. Là TL, anh thuyết phục team/PM về ROI của test-first thế nào khi bị ép deadline?"
> *Dò cái gì:* nói bằng chi phí bug-late vs bug-early, regression an toàn, tốc độ thay đổi; không tuyệt-đối-hoá; có thể TDD chọn lọc cho phần rủi ro cao thay vì all-or-nothing.

**L-TDD-007** ⭐⭐⭐⭐⭐
"Bẫy: AI/dev viết test SAU khi có code, test luôn xanh, nhưng nó chỉ 'đóng băng bug hiện tại'. Anh phát hiện và sửa kiểu test này thế nào?"
> *Dò cái gì:* test viết-theo-code thường assert lại implementation sai mà không kiểm spec; dấu hiệu: không có case biên/đỏ, test đổi mỗi lần refactor; sửa: bắt nguồn từ requirement/đặc tả, thêm mutation test để lộ test rỗng. *(điểm "hiểu để kiểm tra AI".)*

---

## L-DBL — Test double taxonomy & chiến lược mocking

**L-DBL-001** ⭐⭐⭐
"Phân biệt 5 loại test double theo Meszaros: dummy, stub, spy, mock, fake. Mỗi loại dùng khi nào?"
> *Dò cái gì:* dummy = chỉ để lấp tham số; stub = trả giá trị định sẵn; spy = stub + ghi lại cuộc gọi; mock = đặt kỳ vọng tương tác và verify; fake = implementation thật-rút-gọn (vd in-memory DB).

**L-DBL-002** ⭐⭐⭐⭐
"Khác biệt then chốt giữa *stub* và *mock* về cái được assert là gì? Vì sao gọi đó là state-based vs interaction-based verification?"
> *Dò cái gì:* stub → assert *kết quả/trạng thái* sau khi chạy; mock → assert *tương tác* (đã gọi đúng method/đúng args/đúng số lần); chọn sai gây test brittle hoặc rỗng.

**L-DBL-003** ⭐⭐⭐⭐
"Classicist (Detroit) vs Mockist (London) school khác nhau thế nào về việc dùng mock? Hệ quả lên độ giòn khi refactor?"
> *Dò cái gì:* classicist chỉ mock ở ranh giới khó (I/O), để collaborator thật → ít giòn; mockist mock mọi collaborator → test bám implementation, refactor nội bộ dễ làm đỏ test dù hành vi không đổi.

**L-DBL-004** ⭐⭐⭐⭐
"'Over-mocking' là gì và vì sao nó nguy hiểm? Cho ví dụ test xanh nhưng prod vẫn vỡ."
> *Dò cái gì:* mock quá nhiều/sai → test chỉ chứng minh mock khớp mock, không khớp thật; ví dụ mock repository trả data lý tưởng nhưng query thật sai/SQL lỗi → unit xanh, integration/prod vỡ.

**L-DBL-005** ⭐⭐⭐
"'Don't mock what you don't own' nghĩa là gì? Vì sao mock trực tiếp thư viện/SDK bên thứ ba (vd HTTP client của vendor) rủi ro?"
> *Dò cái gì:* mock của mình dễ lệch hành vi thật của thư viện (đổi version → mock vẫn xanh); nên bọc adapter của riêng mình rồi mock adapter đó, và integration test chạm thật ở ranh giới.

**L-DBL-006** ⭐⭐⭐
"Khi nào nên dùng *fake* (vd in-memory DB / in-memory repository) thay vì mock? Trade-off?"
> *Dò cái gì:* fake giữ hành vi gần thật, ít brittle hơn mock từng call; nhưng in-memory (vd SQLite thay Postgres) dễ lệch dialect/feature → cân với TestContainers khi cần đúng DB thật.

**L-DBL-007** ⭐⭐⭐⭐
"Mock thời gian (`Date.now`/timers) và randomness thế nào để test xác định? Vì sao bắt buộc cho test ổn định?"
> *Dò cái gì:* fake timer/clock injectable + seed PRNG; tránh test phụ thuộc giờ thật/`setTimeout` thật → nguồn flaky kinh điển; inject clock/random qua dependency để control.

**L-DBL-008** ⭐⭐⭐
"`jest.mock`/`vi.mock` (module-level auto-mock) khác inject một double qua constructor/DI thế nào? Cái nào dễ bảo trì hơn và vì sao?"
> *Dò cái gì:* module-mock can thiệp hệ thống module (ẩn, dễ rò), DI-inject tường minh/dễ đổi; DI giúp test cô lập sạch hơn; auto-mock tiện nhưng dễ brittle/ngầm. *(cơ chế Nest-specific xem H-TST.)*

**L-DBL-009** ⭐⭐⭐⭐
"Test nào đáng mock external call, test nào nên để chạy thật? Đường ranh giữa 'cô lập để nhanh' và 'thật để tin'?"
> *Dò cái gì:* mock ở out-of-process khi muốn nhanh/cô lập lỗi; để thật khi rủi ro nằm chính ở ráp nối đó; phủ cả hai ở các tầng khác nhau theo pyramid, không phủ trùng.

**L-DBL-010** ⭐⭐⭐⭐⭐
"Bẫy: 'test này verify mock được gọi 3 lần' — vì sao loại assertion này thường là code smell? Khi nào interaction-verification mới thật sự đáng?"
> *Dò cái gì:* assert số lần gọi bám implementation → đổi cách làm nội bộ là đỏ dù hành vi đúng; chỉ verify tương tác khi *bản thân tương tác là contract* (vd 'phải gửi đúng 1 email', 'phải publish event'), còn lại ưu tiên state-based.

---

## L-RUN — Runner & tooling generic

**L-RUN-001** ⭐⭐
"Một test runner (Jest/Vitest) cung cấp những khối nào: runner, assertion, mocking, coverage? Vì sao gọi là 'framework' chứ không chỉ 'runner'?"
> *Dò cái gì:* tìm file + chạy + report; `expect` (assertion); `jest.fn/vi.fn` (mock); istanbul/v8 coverage; tích hợp sẵn nên là 'framework' all-in-one, khác runner thuần.

**L-RUN-002** ⭐⭐⭐
"Jest 30 vs Vitest 4 cho backend Node/TS 2026: khác biệt cốt lõi (ESM, TS, watch speed) và anh chọn cái nào cho project mới vs legacy? *(verify)*"
> *Dò cái gì:* Vitest = ESM/TS native, watch nhanh (Vite pipeline), default khuyến nghị cho project mới; Jest = chín, ecosystem rộng, an toàn cho suite lớn/RN/Angular; quyết theo stack chứ không 'cái nào tốt hơn tuyệt đối'. *(verify version.)*

**L-RUN-003** ⭐⭐⭐
"`describe/it/expect` gần như giống nhau giữa Jest và Vitest, nên 'chi phí migrate' thật ra nằm ở đâu? *(verify)*"
> *Dò cái gì:* API tương thích phần lớn; chi phí ở config/transform, mock API (`jest.*` → `vi.*`), custom serializer/reporter/preset, ESM differences; migrate được nhưng không free.

**L-RUN-004** ⭐⭐⭐
"`node:test` (native) và `bun test` là lựa chọn nào? Khi nào hợp lý hơn Jest/Vitest? *(verify)*"
> *Dò cái gì:* node:test = không thêm dependency, hợp lib/CLI thuần; bun test = nhanh khi đã chạy Bun, suite backend TS; đánh đổi: ecosystem/feature mỏng hơn Jest/Vitest. *(verify.)*

**L-RUN-005** ⭐⭐⭐
"ESM khiến cấu hình Jest hay vỡ vì sao? Vì sao Vitest đỡ đau hơn ở điểm này? *(verify)*"
> *Dò cái gì:* Jest gốc CommonJS, cần transform (babel/ts-jest) + cờ experimental cho ESM; Vitest dùng pipeline Vite → ESM/TS native, ít config; đây là động lực dịch chuyển chính.

**L-RUN-006** ⭐⭐⭐
"Supertest dùng để làm gì trong API test? Vì sao nó không cần app phải listen cổng thật?"
> *Dò cái gì:* bắn HTTP request vào instance app in-process (truyền server vào supertest), assert status/header/body; không bind port → nhanh, song song được, không đụng cổng OS. *(dựng app trong Nest xem H-TST.)*

**L-RUN-007** ⭐⭐⭐⭐
"Test song song mặc định (mỗi file một worker) gây vấn đề gì với tài nguyên chia sẻ (DB, port, file)? Anh cô lập thế nào?"
> *Dò cái gì:* xung đột state/port/row giữa worker → flaky; cô lập bằng schema/DB riêng mỗi worker, container riêng, transaction-rollback per test, tránh global mutable.

**L-RUN-008** ⭐⭐⭐⭐
"`--shard` (chia suite qua nhiều CI runner) giải quyết gì và bẫy khi dùng? *(verify)*"
> *Dò cái gì:* chia suite lớn cho nhiều máy CI để giảm wall-time; bẫy: cân tải không đều, test phụ thuộc thứ tự/global vỡ khi tách, gộp coverage từ nhiều shard. *(verify cờ.)*

---

## L-INT — Integration test & TestContainers

**L-INT-001** ⭐⭐⭐
"Integration test với DB thật giải quyết hạn chế gì mà mock/in-memory không thấy? (góc kỷ luật, không nói cơ chế Nest)"
> *Dò cái gì:* bắt lỗi SQL/migration/constraint/dialect/transaction/index thật; mock giả định query đúng, in-memory (SQLite) lệch Postgres feature; cross-ref H-TST-004 cho cách dựng trong Nest.

**L-INT-002** ⭐⭐⭐⭐
"TestContainers vận hành thế nào (container ephemeral) và vì sao 'ephemeral + isolated' quan trọng cho integration test? *(verify)*"
> *Dò cái gì:* spin Postgres/Redis/Kafka thật trong Docker theo vòng đời test rồi destroy; mỗi run sạch, không lệ thuộc DB dùng chung → reproducible, không 'state rò' giữa lần chạy/CI.

**L-INT-003** ⭐⭐⭐⭐
"Cô lập state giữa các test integration cùng một DB: anh chọn truncate, transaction-rollback, hay DB/schema mới mỗi test? Trade-off mỗi cách?"
> *Dò cái gì:* rollback nhanh nhưng vỡ khi code tự commit/đa connection; truncate sạch nhưng chậm hơn; schema/DB riêng cô lập tốt nhất nhưng tốn; chọn theo tốc độ vs độ cô lập.

**L-INT-004** ⭐⭐⭐
"Migration nên chạy thế nào trong integration test để khớp prod? Vì sao 'sync schema từ ORM' có thể che lỗi migration?"
> *Dò cái gì:* chạy chính file migration prod lên container test → bắt lỗi migration thật; `synchronize`/auto-DDL dựng schema theo entity, bỏ qua bug trong migration script.

**L-INT-005** ⭐⭐⭐⭐
"TestContainers làm CI chậm/đắt hơn. Là TL anh cân bằng 'phủ DB thật' với 'thời gian CI' ra sao?"
> *Dò cái gì:* tái dùng container/giữ ấm, song song hợp lý, chỉ integration phần rủi ro (query/transaction), giữ logic ở unit; tránh biến mọi test thành integration nặng.

**L-INT-006** ⭐⭐⭐⭐
"Test một flow chạm cả Postgres + Redis + một HTTP service ngoài: cái nào dùng thật (container), cái nào mock, vì sao?"
> *Dò cái gì:* DB/cache của mình → container thật (rủi ro nằm ở đó); HTTP của vendor 'không own' → mock/stub ở ranh giới (hoặc contract test); cân theo own/không-own + tốc độ.

**L-INT-007** ⭐⭐⭐⭐⭐
"Integration test xanh ở local nhưng đỏ ngẫu nhiên trên CI. Anh debug nguồn gốc (Docker resource, thứ tự, port, thời gian khởi động container) từ đâu?"
> *Dò cái gì:* nghi readiness (chờ container 'ready' chứ không chỉ 'started'), tài nguyên CI ít hơn, race khi song song, port/clock; thêm wait-strategy, tăng timeout có chủ đích, cô lập state per worker.

---

## L-FLK — Flaky test, determinism, isolation

**L-FLK-001** ⭐⭐
"Flaky test là gì? Vì sao một test pass/fail không ổn định lại *tệ hơn* một test luôn đỏ?"
> *Dò cái gì:* kết quả non-deterministic không đổi code; flaky làm mất niềm tin vào suite, dev bỏ qua đỏ → che bug thật; đỏ-luôn ít ra trung thực và bị sửa ngay.

**L-FLK-002** ⭐⭐⭐
"Liệt kê các nguồn flakiness phổ biến nhất trong backend test."
> *Dò cái gì:* thời gian/timer thật, randomness, race/concurrency, thứ tự test phụ thuộc, state chia sẻ (DB/global) không cô lập, phụ thuộc service ngoài/mạng, async chưa await.

**L-FLK-003** ⭐⭐⭐⭐
"Test phụ thuộc thứ tự chạy (order-dependent) sinh ra do đâu? Vì sao 'randomize test order' lại là công cụ phát hiện tốt?"
> *Dò cái gì:* shared mutable state/leak giữa test (global, DB row, mock không reset); chạy ngẫu nhiên thứ tự làm lộ phụ thuộc ẩn thay vì giấu nó sau thứ tự cố định.

**L-FLK-004** ⭐⭐⭐
"Async/await sai cách gây flaky thế nào? Cho ví dụ test xanh giả vì assertion chạy trước khi promise resolve."
> *Dò cái gì:* quên `await`/return promise → test kết thúc trước khi assert chạy → false green; fix: await đúng, dùng API async của runner, tránh assert trong callback không được chờ.

**L-FLK-005** ⭐⭐⭐⭐
"Anh xử lý flaky test 'tạm thời' bằng auto-retry vs quarantine vs fix gốc — chiến lược nào khi nào, và vì sao retry-mọi-thứ là cái bẫy?"
> *Dò cái gì:* retry/quarantine để không chặn pipeline *tạm thời* + phải log/track để fix gốc; retry-mọi-thứ che bug thật & nuôi flakiness; cần policy + ownership để dọn.

**L-FLK-006** ⭐⭐⭐⭐
"Test pollution / shared state giữa các test: cô lập bằng gì (reset mock, beforeEach, transaction, container per worker)? Vì sao 'không có state chia sẻ' là kim chỉ nam?"
> *Dò cái gì:* mỗi test bắt đầu từ trạng thái sạch xác định; reset mock/clear DB/đóng kết nối; state chia sẻ là gốc của order-dependency & flaky.

**L-FLK-007** ⭐⭐⭐⭐
"Test phụ thuộc network/service ngoài thật là nguồn flaky. Anh thay thế bằng gì để vẫn tin (mock server, contract test, record-replay)?"
> *Dò cái gì:* mock server/stub ở ranh giới cho determinism; record-replay (vd nock/MSW) cho HTTP; contract test để vẫn bắt breaking thật mà không gọi mạng mỗi lần.

**L-FLK-008** ⭐⭐⭐⭐⭐
"Là TL, anh đặt 'quy trình quản trị flaky test' cho team thế nào (detect, ưu tiên, ai sửa, ngưỡng chấp nhận)? Vì sao bỏ mặc flaky là nợ kỹ thuật chết người?"
> *Dò cái gì:* đo flaky-rate, dashboard/quarantine có hạn định, gán owner, SLA dọn; bỏ mặc → mất niềm tin toàn suite → team tắt CI gate → mất luôn lưới an toàn.

---

## L-DAT — Test data, fixtures, factories & snapshot

**L-DAT-001** ⭐⭐
"Fixture là gì? Vì sao 'fixture khổng lồ dùng chung mọi test' lại là anti-pattern?"
> *Dò cái gì:* data dựng sẵn cho test; fixture chung lớn → test ngầm phụ thuộc nhau & khó hiểu test cần gì; nên data tối thiểu, cục bộ cho từng test.

**L-DAT-002** ⭐⭐⭐
"Test data factory / builder pattern (vd Object Mother) giải quyết gì so với hardcode object trong từng test?"
> *Dò cái gì:* tạo entity hợp lệ với override field cần thiết → test rõ ý định ('chỉ field X quan trọng'), giảm trùng lặp, dễ bảo trì khi schema đổi.

**L-DAT-003** ⭐⭐⭐
"Seeding & cleanup giữa các integration test: vì sao thứ tự seed→test→cleanup phải xác định? Rủi ro nếu cleanup không chạy khi test fail?"
> *Dò cái gì:* state rò sang test sau → flaky/order-dependency; cleanup phải chạy kể cả khi fail (afterEach/transaction rollback/DB ephemeral); không dựa vào dữ liệu còn sót.

**L-DAT-004** ⭐⭐⭐
"Snapshot test là gì? Nó hợp cho cái gì và bẫy 'snapshot rot / approve mù' là gì?"
> *Dò cái gì:* lưu output chuẩn rồi so lần sau; hợp cho output ổn định/serialization; bẫy: dev update snapshot vô tội vạ khi đỏ → snapshot mất ý nghĩa kiểm chứng.

**L-DAT-005** ⭐⭐⭐⭐
"Test cần PII/dữ liệu nhạy cảm: anh sinh/ẩn danh data thế nào để không bê dữ liệu prod thật vào test? Rủi ro bảo mật?"
> *Dò cái gì:* synthetic/anonymized data (faker), không copy prod dump chứa PII; rủi ro lộ dữ liệu thật trong fixture/CI log; cross-ref M (security/PII).

**L-DAT-006** ⭐⭐⭐⭐
"Test phụ thuộc thời gian/timezone (vd 'báo cáo cuối tháng') dễ vỡ khi đổi múi giờ/ngày chạy. Anh thiết kế test deterministic cho logic thời gian thế nào?"
> *Dò cái gì:* inject clock/freeze time, test theo input cố định không phải 'now', cố định timezone trong test env; tránh `new Date()` trực tiếp trong logic.

---

## L-COV — Coverage & mutation testing & metric chất lượng

**L-COV-001** ⭐⭐
"Phân biệt line coverage / branch coverage / function coverage. Vì sao branch coverage thông tin hơn line?"
> *Dò cái gì:* line = dòng chạy qua; branch = mỗi nhánh true/false được phủ; line cao vẫn bỏ lọt nhánh (vd else chưa test) → branch sát hơn với 'đã test logic'.

**L-COV-002** ⭐⭐⭐
"Vì sao '100% coverage' KHÔNG đồng nghĩa 'code đã được test đúng'? Cho ví dụ test chạy qua dòng nhưng không assert gì."
> *Dò cái gì:* coverage đo dòng *được chạy*, không đo assertion có nghĩa; test gọi hàm rồi không assert vẫn tính phủ; coverage là cận dưới, không phải đảm bảo chất lượng.

**L-COV-003** ⭐⭐⭐⭐
"Mutation testing (vd Stryker) đo cái gì mà coverage không đo? 'Mutation score' nói lên điều gì? *(verify)*"
> *Dò cái gì:* gài lỗi nhỏ (đổi `>`→`>=`, xoá dòng) rồi xem test có 'giết' mutant không; đo *chất lượng assertion* chứ không chỉ dòng chạy; mutant sống = test rỗng/yếu chỗ đó. *(verify Stryker.)*

**L-COV-004** ⭐⭐⭐⭐
"Đặt coverage threshold làm CI gate có mặt trái gì? Vì sao 'ép 90% line' có thể đẻ test rác?"
> *Dò cái gì:* dev viết test phủ-cho-đủ-số không có assertion nghĩa; số ép cứng tạo gaming; nên kết hợp branch + mutation + review thay vì một con số line tuyệt đối.

**L-COV-005** ⭐⭐⭐
"'Coverage trên diff/PR mới' (patch coverage) khác 'coverage toàn repo' thế nào, và vì sao patch coverage thực dụng hơn cho legacy?"
> *Dò cái gì:* gate theo coverage *code mới đụng* tránh bắt cả repo legacy lên chuẩn ngay; nâng dần chất lượng theo từng PR thay vì block vì nợ cũ.

**L-COV-006** ⭐⭐⭐⭐
"Ngoài coverage, anh dùng metric nào để đánh giá *sức khỏe* của test suite (tốc độ, flaky-rate, defect-escape, mutation score)? Mỗi cái nói lên gì?"
> *Dò cái gì:* tốc độ → feedback loop; flaky-rate → niềm tin; defect-escape (bug lọt prod) → hiệu lực thật; mutation → chất lượng assertion; coverage chỉ là một mảnh.

**L-COV-007** ⭐⭐⭐⭐⭐
"Bẫy phỏng vấn: 'Service quan trọng coverage 95% mà prod vẫn lắm bug.' Anh chẩn đoán nguyên nhân từ đâu?"
> *Dò cái gì:* test phủ dòng nhưng assert nông/mock nông; thiếu integration/contract (bug nằm ở ráp nối/biên ngoài tầm unit); thiếu case biên/lỗi; mutation score thấp; đo sai thứ.

---

## L-LOAD — Load / performance test

**L-LOAD-001** ⭐⭐⭐
"Phân biệt load test / stress test / spike test / soak (endurance) test. Mỗi loại trả lời câu hỏi gì? *(verify công cụ)*"
> *Dò cái gì:* load = tải kỳ vọng (đạt SLO không); stress = đẩy tới gãy (điểm vỡ/độ suy thoái); spike = tăng đột ngột (auto-scale/queue chịu được?); soak = chạy lâu (memory leak/degradation).

**L-LOAD-002** ⭐⭐⭐
"k6 và Artillery dùng để làm gì trong perf test backend? Vì sao chạy load từ một máy local thường cho số liệu sai lệch? *(verify)*"
> *Dò cái gì:* script kịch bản tải + đo latency/throughput/error-rate; máy local bị nghẽn CPU/network của chính nó & gần server → không phản ánh tải thật phân tán; cần môi trường giống prod.

**L-LOAD-003** ⭐⭐⭐⭐
"Đặt 'threshold/SLO assertion' trong load test (vd p95 latency < 300ms, error-rate < 1%) để CI fail tự động — vì sao đo p95/p99 thay vì trung bình?"
> *Dò cái gì:* trung bình giấu đuôi chậm; p95/p99 phản ánh trải nghiệm xấu nhất của phần lớn user; gate theo percentile + error-rate để bắt suy thoái performance trước khi release.

**L-LOAD-004** ⭐⭐⭐⭐
"Phân biệt closed-model (cố định VU) vs open-model (cố định arrival rate) trong load test. Vì sao chọn sai model cho ra kết luận sai về capacity? *(verify)*"
> *Dò cái gì:* closed = N user lặp request (throughput tự co khi server chậm → che nghẽn); open = request tới theo rate bất kể server (mô phỏng traffic thật, lộ queue/đổ vỡ); chọn theo bản chất traffic.

**L-LOAD-005** ⭐⭐⭐⭐
"Đọc kết quả load test: throughput tăng nhưng p99 latency vọt và error-rate lên — anh suy ra điều gì về điểm bão hoà của hệ thống?"
> *Dò cái gì:* đã qua 'knee'/điểm bão hoà; thêm tải không thêm throughput mà chỉ tăng queue/latency/timeout; cần xác định bottleneck (DB pool, CPU, lock) trước khi scale mù.

**L-LOAD-006** ⭐⭐⭐⭐⭐
"Là TL anh tích hợp performance test vào pipeline thế nào để bắt *regression* hiệu năng mà không làm CI quá chậm/đắt?"
> *Dò cái gì:* smoke-load nhẹ mỗi PR (gate threshold), full load định kỳ/pre-release ở env riêng; baseline + so sánh để bắt regression; tách khỏi unit pipeline để không chậm feedback.

---

## L-CON — Contract test: vị trí & gating (cross-ref F)

> Phần "Pact giải quyết vấn đề gì" đã ở **`QB_F_apidesign.md`**. L chỉ giữ vị-trí-trong-pyramid & cơ chế gating.

**L-CON-001** ⭐⭐⭐⭐
"Contract test nằm ở đâu trong test pyramid của microservices? Vì sao nó *thay thế* được phần lớn e2e xuyên service?"
> *Dò cái gì:* tầng giữa, mỗi service test độc lập với 'contract' của láng giềng → bắt breaking integration mà không cần dựng cả hệ thống chạy cùng lúc (vốn brittle/đắt). *(cross-ref F.)*

**L-CON-002** ⭐⭐⭐⭐⭐
"Consumer-driven contract chạy như một CI gate thế nào (broker, 'can-i-deploy')? Vì sao producer phải verify contract *trước khi* deploy? *(verify Pact)*"
> *Dò cái gì:* consumer publish expectation lên broker → producer verify mình thoả → 'can-i-deploy' chặn producer release nếu phá contract của consumer đang chạy; gate ngăn breaking lan ra prod. *(verify.)*

**L-CON-003** ⭐⭐⭐⭐
"Hạn chế/bẫy của contract test là gì? Vì sao nó KHÔNG thay được integration/e2e hoàn toàn?"
> *Dò cái gì:* chỉ kiểm *hình dạng tương tác* hai bên đồng thuận, không kiểm logic nghiệp vụ end-to-end hay hành vi thật của DB/side-effect; vẫn cần ít integration + smoke e2e bổ sung.

---

## L-TL — Chuẩn chất lượng cấp Tech Lead

**L-TL-001** ⭐⭐⭐
"Là TL, anh đặt 'definition of done' về test cho team thế nào (loại test bắt buộc theo loại thay đổi)?"
> *Dò cái gì:* gắn yêu cầu test theo rủi ro: logic mới → unit; endpoint mới → integration/API; bug fix → regression test tái hiện bug; không 'một-cỡ-cho-tất-cả'.

**L-TL-002** ⭐⭐⭐⭐
"Anh thiết kế CI quality gate gồm những tầng nào (lint, type-check, unit, integration, coverage/mutation, e2e/smoke)? Cái gì chặn merge vs cái gì chỉ cảnh báo?"
> *Dò cái gì:* fast gate (lint/type/unit) chặn merge; integration/e2e nặng có thể chạy stage sau/nightly; cân thời gian feedback vs độ phủ; gate cứng cái rẻ-và-quyết-định.

**L-TL-003** ⭐⭐⭐⭐
"Đối phó test suite chạy 40 phút khiến team bỏ qua CI: anh ưu tiên hành động nào để cắt thời gian mà không giảm độ tin cậy?"
> *Dò cái gì:* song song/shard, đẩy test xuống tầng thấp (bớt e2e thừa), tách smoke vs full, cache/parallel container, xoá test trùng/chậm vô ích; đo trước khi tối ưu.

**L-TL-004** ⭐⭐⭐⭐
"Codebase legacy gần như không test. Anh đưa test vào thế nào *mà không* dừng feature 3 tháng để 'viết test cho tất cả'?"
> *Dò cái gì:* characterization test làm lưới an toàn cho phần đụng tới, gate coverage trên diff mới, bọc seam dần (cross-ref Q legacy), ưu tiên module rủi ro cao; tăng dần không big-bang.

**L-TL-005** ⭐⭐⭐⭐
"Test bản thân cũng là code phải bảo trì. Anh đặt chuẩn chất lượng cho *test code* (DRY vừa đủ, đặt tên, một assert-ý-niệm) thế nào trong code review?"
> *Dò cái gì:* test phải đọc như spec (tên rõ hành vi), tránh logic phức tạp trong test, một test một hành vi, tránh over-DRY giấu ý định; review test như review code thật.

**L-TL-006** ⭐⭐⭐⭐⭐
"PM ép cắt test để kịp deadline. Anh thương lượng thế nào để vẫn ship mà không tích nợ chất lượng mất kiểm soát?"
> *Dò cái gì:* nói bằng rủi ro/ chi phí bug-prod, đề xuất cắt có chọn lọc (giữ critical-path test), ghi nợ tường minh + kế hoạch trả; không 'cắt im lặng'; cân bối cảnh thay vì giáo điều.

**L-TL-007** ⭐⭐⭐⭐
"Anh đo 'hiệu quả thật' của chiến lược test của team bằng gì ngoài coverage (defect-escape-rate, MTTR, change-failure-rate)? Vì sao những metric outcome này quan trọng hơn?"
> *Dò cái gì:* defect-escape (bug lọt prod), change-failure-rate, MTTR phản ánh test có *thật sự* bắt lỗi & cho deploy an toàn; coverage là proxy đầu vào, outcome mới đo kết quả.

**L-TL-008** ⭐⭐⭐⭐⭐
"AI sinh test hàng loạt (vd 'viết test cho file này'). Là TL anh kiểm *chất lượng* test do AI viết thế nào — đâu là chỗ AI đúng cú pháp nhưng sai bản chất?"
> *Dò cái gì:* AI hay viết test bám implementation/assert nông/mock-mọi-thứ/snapshot-approve mù → xanh nhưng rỗng; kiểm bằng mutation test, đọc assertion có nghĩa không, có case biên/lỗi không; người chỉ huy spec, AI lo cú pháp. *(điểm "hiểu để chỉ huy & kiểm tra".)*

---

> **Đã sinh xong bộ đề mục L (85 câu, chỉ câu hỏi).** Bước tiếp theo theo WORKFLOW 2:
> - **Lưu file này vào project** (cùng các `QB_*.md` khác) để lần sau AI đọc & chống trùng.
> - **Bước B:** dựng giáo trình ngược từ bộ đề (gom cụm → bài học, cơ bản → nâng cao).
> - **Bước C/D:** giảng từng bài (Hợp đồng 10 mục) rồi phỏng vấn theo đợt, chấm live theo dòng *"dò cái gì"*.
