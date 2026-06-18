# 🧪 Mục L — Testing & Quality · Giáo trình đầy đủ (Bước B + C)

> **Tạo theo WORKFLOW 2** của project "Tech Lead Backend".
> - **Bước B** — dựng giáo trình *ngược* từ ngân hàng `QB_L_testing.md` (85 câu, 12 mục con) → gom thành **11 bài**, sắp **cơ bản → nâng cao** theo *dependency order*.
> - **Bước C** — giảng từng bài theo **Hợp đồng đầu ra 10 mục**, bám đúng các câu mỗi bài phủ.
> - **Phiên bản tooling đã verify (06/2026):** Jest **30.4.2** (đã chuyển về **OpenJS Foundation**, Meta bàn giao), Vitest **4.1.9** (v5 đang beta; cần Vite ≥6, Node ≥20), k6 **2.0** (Grafana, JS/TS native). ⚠️ Mọi version đổi nhanh → đối chiếu docs chính thức khi học.
> - Tài liệu này để **tự học & lưu trữ**. Mỗi bài có phần *"Tự kiểm tra (active recall)"* ở cuối — hãy **tự trả lời trước**, rồi mới xem gợi ý.

---

## 📍 BƯỚC B — Bản đồ giáo trình (dựng ngược từ bộ đề)

Nguyên tắc sắp xếp: **không** theo tần suất hỏi, mà theo **thứ tự phụ thuộc kiến thức** — học cái sau cần hiểu cái trước. Mocking phải hiểu trước khi bàn flaky; integration trước khi bàn TestContainers; mọi thứ trước khi tới chuẩn TL.

| Bài | Tên bài | Mục con phủ | Số câu | Vì sao đặt ở đây |
|---|---|---|---|---|
| **1** | Tư duy & chiến lược test: Pyramid, Trophy & ranh giới loại test | L-PYR, L-TYP | 15 | Mental model nền — định hình mọi quyết định test về sau |
| **2** | Test double & chiến lược mocking (mock/stub/spy/fake/dummy) | L-DBL | 10 | Kỹ năng lõi; hiểu sai mocking → hỏng cả unit lẫn flaky |
| **3** | TDD & BDD: red-green-refactor, outside-in, Gherkin | L-TDD | 7 | Kỷ luật viết test; cần hiểu test type + double trước |
| **4** | Test runner & tooling generic (Jest / Vitest / node:test / Supertest) | L-RUN | 8 | Công cụ tay nghề; chọn runner cần hiểu ESM + mock |
| **5** | Integration test & TestContainers (DB thật) | L-INT | 7 | Tầng giữa pyramid; cần hiểu ranh giới loại test |
| **6** | Test data: fixture, factory, seed/cleanup & snapshot | L-DAT | 6 | Nuôi dữ liệu cho integration; cần hiểu isolation |
| **7** | Flaky test, determinism & isolation | L-FLK | 8 | Tổng hợp: async + time + state + concurrency từ bài trước |
| **8** | Coverage & mutation testing & metric chất lượng | L-COV | 7 | Đo chất lượng suite — cần đã có suite để đo |
| **9** | Load / performance test (k6 / Artillery) | L-LOAD | 6 | Trục riêng (non-functional); đứng độc lập sau functional |
| **10** | Contract test: vị trí trong pyramid & CI gating | L-CON | 3 | Microservices-specific; cross-ref F (Pact) |
| **11** | Chuẩn chất lượng cấp Tech Lead (capstone) | L-TL | 8 | Tổng hợp toàn bộ → quyết sách team, CI gate, legacy |

**Tổng: 85/85 câu được phủ.** Ánh xạ chi tiết Bài → ID câu nằm ở đầu mỗi bài (mục ①).

> **Cách dùng để luyện phỏng vấn:** học hết một bài → mở `QB_L_testing.md`, tự trả lời đúng các ID bài đó phủ → đối chiếu dòng *"dò cái gì"* để tự chấm ĐẠT/CHƯA.

---

# 📗 BÀI 1 — Tư duy & chiến lược test: Pyramid, Trophy & ranh giới loại test

### ① Mục tiêu & vị trí trong mạch
- **Phase tương đương:** Phase 8 (Quality/LLMOps) ở roadmap GenAI, nhưng đây là nền **testing tổng quát** cho mọi backend.
- **Phủ câu:** `L-PYR-001 → 008`, `L-TYP-001 → 007` (15 câu).
- Học xong: anh **phân loại** được unit/integration/component/e2e theo ranh giới *thực dụng*, biết **đặt test ở tầng nào**, và đọc được khi nào pyramid bị lệch thành anti-pattern. Đây là khung tư duy mọi bài sau bám vào.

### ② Giảng cơ bản → nâng cao

**(a) Trực giác.** Hình dung suite test như một kim tự tháp:
- **Đáy — Unit (nhiều):** test một đơn vị logic cô lập khỏi I/O thật. Nhanh (mili-giây), rẻ, ổn định → viết hàng nghìn cái không sao.
- **Giữa — Integration (vừa):** ráp nhiều đơn vị + chạm ranh giới thật (DB, HTTP, queue). Chậm hơn, bắt được lỗi *ráp nối*.
- **Đỉnh — E2E (rất ít):** bắn request thật xuyên toàn hệ thống như user. Chậm, dễ vỡ (brittle), khó debug → chỉ giữ cho critical journey.

> **Nguyên tắc cốt lõi (push-down):** *đẩy mỗi test xuống tầng thấp nhất vẫn kiểm chứng được hành vi đó.* Cùng một niềm tin, tầng thấp cho nó với giá rẻ hơn nhiều. `[L-PYR-001, L-PYR-005]`

**(b) Cơ chế chi tiết.**

*Ranh giới loại test (đường ranh thực dụng):* `[L-TYP-001, L-TYP-003]`
- **Unit** = mọi out-of-process dependency đều bị thay bằng double. Dù gọi 5 class nhưng không chạm DB/mạng → vẫn là unit.
- **Integration** = chạm **dependency thật** qua ít nhất một ranh giới (DB/HTTP/queue/file).
- **E2E / API-level** = app đã wire đầy đủ (pipe, guard, DB), bắn request thật vào, assert status + body + **side-effect** (row trong DB, message phát ra). Backend không-UI thì "e2e" nghĩa là **API-level e2e**, không cần trình duyệt. `[L-TYP-004]`

*"Unit" co giãn* — đây là chỗ nhiều người tranh cãi: `[L-TYP-001, L-TYP-002]`
- **Solitary (mockist):** mock hết collaborator → cô lập tuyệt đối, nhưng test *bám implementation*, refactor nội bộ dễ làm đỏ.
- **Sociable (classicist):** để collaborator thật chạy cùng (chỉ mock ở ranh giới khó) → ít giòn hơn, gần hành vi thật hơn.

*Component test* — tầng "ẩn" giữa integration và e2e: test **một service trọn vẹn** qua public boundary, DB thật nhưng **mock các service ngoài ở mức mạng**. Cho microservices: nhanh & ổn định hơn e2e-xuyên-hệ-thống mà vẫn phủ wiring nội bộ. `[L-TYP-005]`

*Smoke vs regression:* smoke = vài kiểm tra critical-path chạy *sau deploy* để xác nhận "còn sống"; regression = phủ rộng tránh tái phát bug. Smoke đặt ở gate post-deploy/canary. `[L-TYP-006]`

**(c) Mép giới hạn & sai lầm thường gặp.**
- **Ice-cream cone** (đảo ngược pyramid: nhiều e2e, ít unit): triệu chứng = suite chạy hàng chục phút, hay đỏ vu vơ, dev mất niềm tin và *bỏ qua* test. Feedback loop chết. `[L-PYR-002]`
- **Phủ trùng nhiều tầng:** test cùng một logic ở cả unit lẫn e2e = lãng phí + brittle. Quy tắc: logic/nhánh/biên → unit; rủi ro ở ráp nối/SQL/serialization/transaction → integration. Không phủ trùng. `[L-TYP-007]`
- **Microservices:** "full e2e xuyên nhiều service" cực brittle & tốn hạ tầng → thay bằng **contract test** giữa service + integration trong từng service, chỉ giữ rất ít smoke e2e. `[L-PYR-007]` (chi tiết Bài 10).

> 📘 *Khái niệm pyramid/loại test là kiến thức ổn định.* ➕ *Phần "component test cho microservices", "sociable vs solitary", và push-down quantification là nâng cấp thực chiến ngoài định nghĩa SGK.*

### ③ ⚠️ Kiến thức cũ / bị thay thế

| Cái cũ (phổ biến / trong tài liệu) | Thay bằng (hiện nay) | Vì sao |
|---|---|---|
| "Cứ nhắm **70/20/10** unit/integration/e2e cho mọi service" | **Hình dạng theo bản chất rủi ro**: service logic-nặng → nghiêng unit; service glue/I/O-nặng → nghiêng integration (gần "testing trophy") | Tỉ lệ cứng bỏ qua việc rủi ro nằm ở đâu `[L-PYR-003, L-PYR-004]` |
| Pyramid là chân lý tuyệt đối | **Trophy** (Kent C. Dodds) cho app integration-nặng: integration là tầng ROI cao nhất | Test gần hành vi thật mà vẫn đủ nhanh, khi logic nằm ở ráp nối hơn thuật toán `[L-PYR-003]` |
| Nhiều e2e xuyên hệ thống cho microservices | **Contract test + component test**, e2e chỉ smoke | E2E-xuyên-service brittle/đắt `[L-PYR-007]` |
| Đo "chất lượng test" = coverage % | Nhìn thêm **tốc độ suite, flaky-rate, mutation score, defect-escape-rate** | Coverage đo *dòng chạy*, không đo *assertion có nghĩa* `[L-PYR-008]` |

### ④ Áp dụng thực tế + So sánh bigtech
- **Use case:** service `OrderService` (CRUD + tính phí ship phức tạp). Phân bổ ngân sách test: logic tính phí (nhiều nhánh/biên) → **unit dày**; phần CRUD ráp với Postgres → vài **integration** kiểm transaction/constraint; **1–2 smoke e2e** cho "đặt đơn → trừ kho → phát event". `[L-PYR-004]`
- **Bug chỉ e2e bắt được:** middleware order sai khiến auth chạy sau body-parser, hoặc CORS/serialization vỡ xuyên tầng — unit & mock không lộ. `[L-PYR-006]`
- **Bigtech (pattern phổ biến, không tuyệt đối):** Google công khai cổ vũ pyramid ("Just Say No to More End-to-End Tests"); Spotify/Kent Dodds đẩy "trophy"; các team microservices lớn dựa nặng vào **contract testing (Pact)** thay e2e. *(Chi tiết tooling đổi nhanh → đối chiếu khi cần.)*

### ⑤ Code thực hành + cấu hình
Minh họa "cùng hành vi, hai tầng khác nhau" bằng **Vitest 4** (API gần như giống Jest 30):

```ts
// src/shipping.ts — logic thuần, lý tưởng để UNIT
export function shippingFee(weightKg: number, distanceKm: number): number {
  if (weightKg <= 0 || distanceKm <= 0) throw new Error("invalid input");
  const base = 20_000;
  const perKg = weightKg * 5_000;
  const perKm = distanceKm > 50 ? distanceKm * 800 : distanceKm * 500; // biên 50km
  return base + perKg + perKm;
}
```

```ts
// src/shipping.unit.test.ts  — UNIT: nhanh, nhiều case biên, không I/O
import { describe, it, expect } from "vitest"; // Jest: bỏ import, dùng global
import { shippingFee } from "./shipping";

describe("shippingFee", () => {
  it("tính đúng dưới ngưỡng 50km", () => {
    expect(shippingFee(2, 10)).toBe(20_000 + 10_000 + 5_000);
  });
  it("đổi hệ số khi vượt 50km (test BIÊN)", () => {
    expect(shippingFee(1, 51)).toBe(20_000 + 5_000 + 51 * 800);
  });
  it("ném lỗi với input không hợp lệ", () => {
    expect(() => shippingFee(0, 10)).toThrow("invalid input");
  });
});
```

```ts
// src/order.e2e.test.ts — E2E API-level: app thật + DB thật, chỉ happy-path
import { describe, it, expect, beforeAll, afterAll } from "vitest";
import request from "supertest";              // bắn HTTP in-process, không cần listen cổng
import { buildApp } from "./app";             // app đã wire đủ middleware + repo thật
let app: ReturnType<typeof buildApp>;
beforeAll(() => { app = buildApp(); });
afterAll(async () => { /* đóng DB pool */ });

it("POST /orders tạo đơn và trừ kho", async () => {
  const res = await request(app).post("/orders").send({ sku: "A1", qty: 2 });
  expect(res.status).toBe(201);
  expect(res.body).toMatchObject({ status: "created" });
  // assert SIDE-EFFECT, không chỉ response:
  // const stock = await db.query(...); expect(stock).toBe(98);
});
```

**Cấu hình tối thiểu (`package.json` — pin version đã verify 06/2026):**
```json
{
  "devDependencies": {
    "vitest": "4.1.9",
    "supertest": "^7",
    "@types/supertest": "^6"
  },
  "scripts": { "test": "vitest run", "test:watch": "vitest" }
}
```
> `// kiểm chứng version mới nhất tại vitest.dev và npm trước khi dùng — v5 đang beta.`
> ⚠️ **Bẫy "hiểu để kiểm tra":** AI hay sinh test e2e cho *mọi* hành vi (vì nó "phủ nhiều nhất") → ice-cream cone. Người hiểu sẽ **đẩy logic tính phí xuống unit**, chỉ giữ e2e cho luồng xuyên hệ thống.

### ⑥ Keywords cần nhớ (glossary)
- 🧠 **Cần HIỂU:** test pyramid vs testing trophy, push-down, ice-cream cone anti-pattern, confidence-vs-speed trade-off, solitary vs sociable unit, component test, side-effect assertion.
- 📌 **Cần THUỘC:** 3 tầng pyramid; định nghĩa unit/integration/e2e theo *ranh giới out-of-process*; smoke ≠ regression.
- 🛠️ **Cần LÀM ĐƯỢC:** quyết định "test này nên ở tầng nào"; nhận ra suite bị lệch; viết 1 unit + 1 API-level e2e cho cùng feature mà không phủ trùng.

### ⑦ Mental model
> **"Đẩy test xuống thấp nhất vẫn kiểm được; lên cao chỉ khi rủi ro nằm ở ráp nối."** Hình dạng pyramid là *hệ quả của rủi ro*, không phải tỉ lệ cố định.

### ⑧ Câu hỏi phỏng vấn & thách đố
1. *(dễ)* Mô tả 3 tầng pyramid và nguyên tắc cốt lõi. → **Gợi ý:** nhiều unit rẻ/nhanh/ổn → ít e2e chậm/giòn; push-down. `[L-PYR-001]`
2. *(TB)* Khi nào "trophy" hợp hơn pyramid? → app mà logic nằm ở ráp nối I/O/HTTP hơn thuật toán thuần. `[L-PYR-003]`
3. *(TB)* Phân biệt component test vs e2e cho microservices? → component cô lập 1 service (mock service ngoài, DB thật); e2e xuyên cả hệ thống. `[L-TYP-005]`
4. *(khó)* TL phân bổ ngân sách test cho CRUD-service vs compute-service ra sao? → dịch hình dạng theo rủi ro, không áp tỉ lệ cứng. `[L-PYR-004]`
5. *(khó)* Đường ranh phân loại unit vs integration của anh là gì? → có chạm dependency *thật* qua ranh giới không. `[L-TYP-003]`

**Thách đố/trick:**
- 🎯 *"Coverage 90% nhưng toàn e2e mock nông — con số này lừa anh chỗ nào?"* → coverage đo dòng chạy, không đo assertion có nghĩa; nhìn mutation score + flaky-rate + defect-escape. `[L-PYR-008]`
- 🎯 *"Một bug routing/CORS lọt prod dù unit xanh hết — tầng nào lẽ ra bắt được?"* → integration/e2e ở ranh giới wiring; bug ráp nối ngoài tầm unit/mock. `[L-PYR-006]`

### ⑨ Bài tập + tiêu chí tự chấm
1. Lấy một endpoint trong code của anh đang **chỉ** được e2e phủ → tách logic ra hàm thuần, viết unit cho nó, rút e2e về 1 happy-path. **Đạt khi:** số e2e giảm, unit phủ các nhánh biên, tổng thời gian suite giảm.
2. Vẽ pyramid hiện tại của một service anh đang làm (đếm thật số test mỗi tầng). **Đạt khi:** chỉ ra được 1 chỗ lệch và 1 hành động push-down cụ thể.

### ⑩ Đọc thêm / nguồn chuẩn
- Martin Fowler — "TestPyramid" & "UnitTest" (sociable vs solitary).
- Google Testing Blog — "Just Say No to More End-to-End Tests".
- Kent C. Dodds — "The Testing Trophy".

### 🧠 Tự kiểm tra (active recall) — *trả lời trước khi xem gợi ý*
1. Vì sao đẩy một hành vi từ e2e xuống unit lại "rẻ" hơn dù vẫn cùng niềm tin?
2. Một suite chạy 35 phút, hay đỏ vu vơ — đây là anti-pattern gì, sửa từ đâu?
3. Phân biệt "unit có mock DB" và "integration test" bằng MỘT câu.

<details><summary>Gợi ý chốt</summary>

(1) Tầng thấp không dựng hạ tầng/đợi I/O, chạy ms thay vì giây, ít điểm vỡ ngẫu nhiên. (2) Ice-cream cone → push-down: tách logic ra unit, bỏ e2e thừa, giữ smoke. (3) Có chạm dependency *thật* qua ranh giới out-of-process không: có → integration; mock hết → vẫn là unit.
</details>

---

# 📗 BÀI 2 — Test double & chiến lược mocking

### ① Mục tiêu & vị trí trong mạch
- **Phủ câu:** `L-DBL-001 → 010` (10 câu). Nối tiếp Bài 1 (solitary vs sociable) bằng *kỹ năng tay nghề*: tạo và dùng double cho đúng.
- Học xong: anh phân biệt được 5 loại double, biết **khi nào assert trạng thái vs assert tương tác**, tránh over-mocking, và nhận ra mock-assertion nào là code smell. Đây là kỹ năng quyết định một unit test "thật" hay "rỗng".

### ② Giảng cơ bản → nâng cao

**(a) Trực giác.** "Test double" = diễn viên đóng thế cho dependency thật, để anh kiểm soát môi trường test. Như đóng thế phim: có loại chỉ đứng cho đủ khung hình (dummy), có loại đọc thoại định sẵn (stub), có loại vừa diễn vừa ghi lại mọi cảnh quay (spy).

**(b) Taxonomy Meszaros — 5 loại** `[L-DBL-001]`:
| Loại | Làm gì | Dùng khi |
|---|---|---|
| **Dummy** | chỉ lấp tham số, không bao giờ được dùng | cần truyền đối số mà logic không đụng tới |
| **Stub** | trả **giá trị định sẵn** | điều khiển input gián tiếp ("giả sử repo trả user này") |
| **Spy** | stub + **ghi lại** cuộc gọi để xem sau | muốn kiểm "đã gọi" mà không đặt kỳ vọng cứng |
| **Mock** | đặt **kỳ vọng tương tác** trước & verify | tương tác *chính là* contract cần kiểm |
| **Fake** | implementation **thật-rút-gọn** (in-memory DB/repo) | muốn hành vi gần thật, ít brittle hơn stub từng call |

**Khác biệt then chốt stub vs mock** `[L-DBL-002]`:
- **Stub → state-based verification:** chạy xong, assert **kết quả/trạng thái**.
- **Mock → interaction-based verification:** assert **đã gọi đúng method, đúng args, đúng số lần**.
- Chọn sai → test brittle (mock bám implementation) hoặc rỗng (stub mà quên assert).

**(c) Mép giới hạn & sai lầm.**
- **Classicist (Detroit) vs Mockist (London)** `[L-DBL-003]`: classicist chỉ mock ở ranh giới khó (I/O), để collaborator thật → ít giòn; mockist mock mọi collaborator → refactor nội bộ làm đỏ test dù hành vi không đổi.
- **Over-mocking** `[L-DBL-004]`: mock quá nhiều → test chỉ chứng minh "mock khớp mock". Ví dụ kinh điển: mock repository trả data lý tưởng, nhưng query SQL thật sai → **unit xanh, prod vỡ**.
- **"Don't mock what you don't own"** `[L-DBL-005]`: mock thẳng SDK vendor → vendor đổi version, hành vi đổi, mock vẫn xanh. Đúng: **bọc adapter của riêng anh**, mock adapter đó, và integration test chạm thật ở ranh giới.
- **Fake vs mock** `[L-DBL-006]`: in-memory repo (fake) ít brittle hơn mock từng call; nhưng SQLite-thay-Postgres lệch dialect/feature → cân với TestContainers (Bài 5) khi cần DB thật.
- **Mock thời gian & randomness** `[L-DBL-007]`: fake timer/clock injectable + seed PRNG. Phụ thuộc `Date.now()`/`setTimeout` thật là nguồn flaky kinh điển (Bài 7).
- **`vi.mock`/`jest.mock` (module auto-mock) vs DI** `[L-DBL-008]`: module-mock can thiệp hệ thống module → ẩn, dễ rò; **DI-inject** tường minh, dễ đổi, cô lập sạch hơn. *(Cơ chế Nest-specific: xem `QB_H_nestjs` nhóm H-TST.)*

> 📘 *Taxonomy 5 loại là kiến thức ổn định.* ➕ *Bảng state-vs-interaction, "don't mock what you don't own", và phần code smell ở câu 010 là chiều sâu thực chiến.*

### ③ ⚠️ Kiến thức cũ / bị thay thế

| Cái cũ | Thay bằng (hiện nay) | Vì sao |
|---|---|---|
| Mock **mọi** collaborator (mockist mặc định) | Classicist: chỉ mock ở **ranh giới out-of-process**, để logic nội bộ chạy thật | Test bám implementation → đỏ giả khi refactor `[L-DBL-003]` |
| `expect(mock).toHaveBeenCalledTimes(3)` cho mọi test | **State-based** (assert kết quả); chỉ verify tương tác khi *tương tác là contract* | Đếm số lần gọi = code smell, bám cách làm nội bộ `[L-DBL-010]` |
| Mock trực tiếp SDK/HTTP client của vendor | **Bọc adapter mình sở hữu** rồi mock adapter; integration chạm thật | Vendor đổi → mock không đổi → xanh giả `[L-DBL-005]` |
| `jest.useFakeTimers()` rải rác, không inject clock | **Inject clock/random qua DI** + fake timer có chủ đích | Dễ control, ít rò trạng thái timer `[L-DBL-007]` |
| Trong Vitest cũ: `vi.spyOn` reset tùy tiện | **Vitest 4**: `restoreAllMocks` chỉ restore spy tạo bằng `vi.spyOn`, không đụng automock; `vi.fn().getMockName()` đổi | Hành vi mock được làm rõ ở v4 *(verify migration guide)* |

### ④ Áp dụng thực tế + So sánh bigtech
- **Use case:** `NotificationService.notify(user)` phải **gửi đúng 1 email**. Đây là ca *tương tác là contract* → **mock** `emailGateway.send` và verify gọi 1 lần với đúng địa chỉ. Ngược lại, `PriceCalculator` chỉ cần **stub** `taxRepo.getRate()` rồi assert kết quả số tiền. `[L-DBL-002, L-DBL-010]`
- **Bigtech pattern:** xu hướng chung là **classicist + bọc adapter ở ranh giới** (hexagonal/ports-adapters), giảm mock sâu; dùng **MSW/nock** record-replay cho HTTP thay vì mock client (Bài 7). *(Pattern phổ biến, không tuyệt đối.)*

### ⑤ Code thực hành + cấu hình
```ts
// stub (state-based) vs mock (interaction-based) — Vitest 4
import { describe, it, expect, vi } from "vitest";
import { PriceCalculator } from "./price";
import { NotificationService } from "./notify";

it("STUB: assert KẾT QUẢ (state-based)", () => {
  const taxRepo = { getRate: vi.fn().mockReturnValue(0.1) }; // stub trả giá trị định sẵn
  const calc = new PriceCalculator(taxRepo);
  expect(calc.total(100)).toBe(110);          // assert trạng thái, KHÔNG quan tâm gọi mấy lần
});

it("MOCK: assert TƯƠNG TÁC vì gửi email là contract", () => {
  const emailGateway = { send: vi.fn().mockResolvedValue(true) };
  const svc = new NotificationService(emailGateway);   // DI: inject double, không module-mock
  svc.notify({ email: "a@x.com" });
  expect(emailGateway.send).toHaveBeenCalledTimes(1);   // ĐÚNG: tương tác = contract
  expect(emailGateway.send).toHaveBeenCalledWith("a@x.com", expect.any(String));
});

it("inject clock thay vì Date.now() thật (determinism)", () => {
  const clock = { now: () => new Date("2026-06-18T00:00:00Z").getTime() }; // fake injectable
  const calc = new PriceCalculator({ getRate: () => 0 }, clock);
  // ... assert hành vi phụ thuộc thời gian một cách XÁC ĐỊNH
});
```
```ts
// FAKE (in-memory) thay vì mock từng call — ít brittle hơn
class InMemoryUserRepo {
  private rows = new Map<string, any>();
  async save(u: any) { this.rows.set(u.id, u); }
  async findById(id: string) { return this.rows.get(id) ?? null; }
}
// dùng fake này cho unit "sociable"; integration thật vẫn ở Bài 5 (TestContainers)
```
> ⚠️ **Bẫy "hiểu để kiểm tra":** AI viết test hay **mock-mọi-thứ** + assert `toHaveBeenCalled` → xanh nhưng rỗng. Người chỉ huy phải hỏi: *tương tác này có phải contract không?* Nếu không → đổi sang state-based.

### ⑥ Keywords cần nhớ
- 🧠 **Cần HIỂU:** state-based vs interaction-based; classicist vs mockist; over-mocking; "don't mock what you don't own"; vì sao fake ít giòn hơn mock.
- 📌 **Cần THUỘC:** 5 loại double (dummy/stub/spy/mock/fake) + định nghĩa; cú pháp `vi.fn().mockReturnValue/mockResolvedValue`, `toHaveBeenCalledWith`.
- 🛠️ **Cần LÀM ĐƯỢC:** chọn đúng loại double cho tình huống; bọc adapter để mock cái mình sở hữu; inject clock/random.

### ⑦ Mental model
> **"Stub điều khiển INPUT (assert kết quả); Mock kiểm OUTPUT-tương-tác (chỉ khi tương tác là contract). Mock cái mình sở hữu, để thật cái rủi ro nằm ở đó."**

### ⑧ Câu hỏi phỏng vấn & thách đố
1. *(dễ)* 5 loại double khác nhau gì? `[L-DBL-001]`
2. *(TB)* Stub vs mock khác nhau ở cái được assert? → state vs interaction. `[L-DBL-002]`
3. *(TB)* "Don't mock what you don't own" nghĩa là gì? `[L-DBL-005]`
4. *(khó)* Classicist vs mockist ảnh hưởng độ giòn khi refactor thế nào? `[L-DBL-003]`
5. *(khó)* Khi nào dùng fake thay mock, trade-off? `[L-DBL-006]`
6. *(khó)* Test nào **đáng mock** external call, test nào để **chạy thật**? → Ranh *"cô lập để nhanh/ổn định"* vs *"thật để tin"*: unit cô lập (mock I/O ngoài) cho tốc độ & xác định; nhưng rủi ro nằm ở chính ranh giới (SQL, serialize, contract HTTP) thì phải để **integration chạm thật** (Bài 5) — mock chỗ đó chỉ tạo niềm tin giả. `[L-DBL-009]`

**Thách đố/trick:**
- 🎯 *"`toHaveBeenCalledTimes(3)` — vì sao thường là code smell?"* → bám implementation; chỉ đáng khi tương tác là contract (gửi 1 email, publish 1 event). `[L-DBL-010]`
- 🎯 *"Unit xanh hết mà prod vỡ ở query — chẩn đoán?"* → over-mocking: mock repo trả data lý tưởng, query SQL thật sai; thiếu integration. `[L-DBL-004]`

### ⑨ Bài tập + tiêu chí tự chấm
1. Tìm trong code một test đang `toHaveBeenCalledTimes(n)` → xét xem tương tác đó có phải contract không; nếu không, đổi sang state-based. **Đạt khi:** test vẫn xanh khi anh refactor nội bộ (đổi cách gọi mà giữ hành vi).
2. Bọc một SDK vendor (vd HTTP client) bằng adapter của mình; mock adapter trong unit, để integration chạm thật. **Đạt khi:** unit không còn import trực tiếp SDK vendor.

### ⑩ Đọc thêm / nguồn chuẩn
- Gerard Meszaros — *xUnit Test Patterns* (taxonomy double).
- Martin Fowler — "Mocks Aren't Stubs", "TestDouble".
- Vitest docs — Mocking; Vitest 4 migration guide (đổi hành vi spy/restore).

### 🧠 Tự kiểm tra (active recall)
1. Một test "assert kết quả trả về" dùng loại double nào, gọi là verification gì?
2. Vì sao mock thẳng SDK của vendor nguy hiểm? Cách đúng?
3. Khi nào `toHaveBeenCalledWith` là *đúng đắn* chứ không phải code smell?

<details><summary>Gợi ý chốt</summary>

(1) Stub → state-based verification. (2) Vendor đổi version → hành vi đổi mà mock vẫn xanh → xanh giả; bọc adapter mình sở hữu rồi mock adapter, integration chạm thật. (3) Khi bản thân tương tác là contract: "phải gửi đúng 1 email", "phải publish đúng event" — lúc đó tương tác *là* hành vi cần kiểm.
</details>

---

# 📗 BÀI 3 — TDD & BDD: red-green-refactor, outside-in, Gherkin

### ① Mục tiêu & vị trí trong mạch
- **Phủ câu:** `L-TDD-001 → 007` (7 câu). Cần đã hiểu loại test (Bài 1) + double (Bài 2) vì TDD quyết định *thứ tự* dựng chúng.
- Học xong: anh chạy được vòng red-green-refactor, biết khi nào TDD hợp/không, phân biệt outside-in vs inside-out, dùng BDD/Gherkin đúng chỗ, và **phát hiện test viết-sau-code chỉ "đóng băng bug"**.

### ② Giảng cơ bản → nâng cao

**(a) Trực giác.** TDD = viết *bài kiểm tra* trước khi viết *bài làm*. Không phải vì kỷ luật hình thức, mà vì **test-first ép anh nghĩ về interface/spec & testability trước khi implement** → tránh viết test "vừa khít code sai". `[L-TDD-001]`

**(b) Vòng lặp red-green-refactor** `[L-TDD-001]`:
1. **Red:** viết test mô tả hành vi mong muốn → chạy → **đỏ** (chưa có code).
2. **Green:** viết code **tối thiểu** cho test xanh (chấp nhận xấu).
3. **Refactor:** dọn code giữ test xanh — lưới an toàn cho phép refactor mạnh dạn.

**Outside-in (London/mockist) vs Inside-out (Detroit/classicist)** `[L-TDD-005]`:
- **Outside-in:** bắt đầu từ ngoài (acceptance/endpoint) đẩy vào trong, **mock collaborator chưa tồn tại** → định hình interface từ nhu cầu. Nhiều mock.
- **Inside-out:** xây từ **domain core** ra ngoài, ít mock, ráp dần. Khác nhau ở *điểm bắt đầu* và *lượng double*.

**BDD vs TDD** `[L-TDD-004]`:
- TDD nói về *đơn vị code* (góc dev). **BDD** mô tả **hành vi từ góc người dùng**, ngôn ngữ tự nhiên (Given-When-Then / Gherkin) cho cả non-dev đọc.
- **Bẫy Gherkin:** tốn công maintain step-definition; dễ biến thành **e2e brittle** nếu lạm dụng cho mọi thứ.

**(c) Mép giới hạn & sai lầm.**
- **Khi nào TDD KHÔNG hợp** `[L-TDD-002]`: exploratory/prototype, UI/integration nặng, hoặc **chưa rõ thiết kế** → nên **spike** (thử nghiệm vứt đi) trước rồi mới TDD phần đã rõ.
- **TDD có cải thiện thiết kế không?** `[L-TDD-003]` — hai chiều: test-first đẩy về **low coupling/seam rõ** (code khó test = coupling cao); NHƯNG TDD máy móc đẻ **over-mock/over-abstraction**. Design tốt vẫn cần *chủ ý*, không tự sinh.
- **ROI "chậm đầu nhanh sau"** `[L-TDD-006]`: thuyết phục bằng chi phí *bug-late vs bug-early* + regression an toàn; không tuyệt-đối-hoá → TDD **chọn lọc** cho phần rủi ro cao thay vì all-or-nothing.

> ➕ *Toàn bài là chiều "kỷ luật/thiết kế" — docs gốc thường chỉ dạy cú pháp test, không dạy thứ tự & ROI.*

### ③ ⚠️ Kiến thức cũ / bị thay thế

| Cái cũ / hiểu lầm | Hiện nay | Vì sao |
|---|---|---|
| "TDD = viết test cho mọi dòng, all-or-nothing" | **TDD chọn lọc**: test-first cho logic rủi ro cao; spike cho phần chưa rõ | Tránh giáo điều giết tốc độ `[L-TDD-002, L-TDD-006]` |
| Viết test **sau** code rồi tự hào "xanh hết" | **Test-first** (hoặc ít nhất test xuất phát từ **spec**, không từ implementation) | Test-after thường "đóng băng bug hiện tại" `[L-TDD-007]` |
| BDD/Gherkin cho **mọi** test | Gherkin chỉ cho **acceptance/hành vi nghiệp vụ** quan trọng | Lạm dụng → step-def brittle, đắt maintain `[L-TDD-004]` |
| Outside-in mock-mọi-thứ là "chuẩn TDD" | Cân classicist/mockist theo bài; giảm mock sâu | Mockist quá tay → test bám implementation `[L-TDD-005]` |

### ④ Áp dụng thực tế + So sánh bigtech
- **Use case:** thêm rule "đơn > 5 triệu được giảm 3%". TDD: viết test đỏ cho mốc 5tr + biên (4.999.999 và 5.000.000) **trước**, rồi code tối thiểu → buộc nghĩ ngay về biên thay vì phát hiện bug sau. `[L-TDD-001]`
- **Bigtech pattern:** nhiều team **không** TDD-strict toàn bộ; họ TDD cho **domain logic** và để integration/e2e viết-sau. BDD/Cucumber phổ biến ở team có **BA/QA** cùng đọc spec. *(Pattern phổ biến, không tuyệt đối.)*

### ⑤ Code thực hành + cấu hình
```ts
// RED trước — Vitest 4. Code discount() CHƯA tồn tại → test đỏ
import { describe, it, expect } from "vitest";
import { discount } from "./discount"; // sẽ tạo sau

describe("discount theo TDD", () => {
  it("RED: đơn đúng 5tr được giảm 3%", () => {
    expect(discount(5_000_000)).toBe(4_850_000);
  });
  it("RED: biên ngay dưới 5tr KHÔNG giảm", () => {
    expect(discount(4_999_999)).toBe(4_999_999);
  });
});
```
```ts
// GREEN — code TỐI THIỂU cho xanh
export function discount(amount: number): number {
  return amount >= 5_000_000 ? Math.round(amount * 0.97) : amount;
}
// REFACTOR — tách hằng số, giữ test xanh:
const THRESHOLD = 5_000_000, RATE = 0.03;
export function discount2(a: number) {
  return a >= THRESHOLD ? Math.round(a * (1 - RATE)) : a;
}
```
```gherkin
# BDD/Gherkin — chỉ cho hành vi nghiệp vụ quan trọng (Cucumber/Jest-Cucumber)
Feature: Giảm giá đơn lớn
  Scenario: Đơn vượt ngưỡng được giảm
    Given giỏ hàng trị giá 6000000 đồng
    When khách thanh toán
    Then tổng tiền phải là 5820000 đồng
```
> ⚠️ **Bẫy "hiểu để kiểm tra":** bảo AI "viết test cho file này" → nó đọc code, viết test **khớp y hệt code hiện tại** (kể cả bug). Dấu hiệu: không có case biên/đỏ, test đổi mỗi lần refactor. Sửa: test xuất phát từ **requirement**, thêm **mutation test** (Bài 8) để lộ test rỗng. `[L-TDD-007]`

### ⑥ Keywords cần nhớ
- 🧠 **Cần HIỂU:** vì sao test-first ép nghĩ về spec/testability; outside-in vs inside-out; TDD-cải-thiện-design hai chiều; bẫy test-after.
- 📌 **Cần THUỘC:** 3 bước red-green-refactor; cấu trúc Given-When-Then.
- 🛠️ **Cần LÀM ĐƯỢC:** chạy một vòng TDD thật; quyết định spike-hay-TDD; viết 1 Gherkin scenario đúng phạm vi.

### ⑦ Mental model
> **"Đỏ → xanh tối thiểu → dọn. Test-first để định hình spec, không phải để đạt coverage. Test phải bắt nguồn từ yêu cầu, không từ code đã viết."**

### ⑧ Câu hỏi phỏng vấn & thách đố
1. *(dễ)* 3 bước TDD và vì sao test-first quan trọng? `[L-TDD-001]`
2. *(TB)* Khi nào KHÔNG nên TDD strict? `[L-TDD-002]`
3. *(TB)* BDD khác TDD ở focus & ngôn ngữ? Bẫy Gherkin? `[L-TDD-004]`
4. *(khó)* Outside-in vs inside-out khác nhau ở điểm bắt đầu & lượng mock? `[L-TDD-005]`
5. *(khó)* TDD có thật sự cải thiện thiết kế? Lập luận hai chiều. `[L-TDD-003]`

**Thách đố/trick:**
- 🎯 *"AI/dev viết test SAU code, luôn xanh — vì sao nguy hiểm, phát hiện thế nào?"* → chỉ đóng băng bug; không case biên/đỏ; sửa bằng spec-first + mutation. `[L-TDD-007]`
- 🎯 *"PM ép bỏ TDD vì chậm — thuyết phục thế nào?"* → bug-early rẻ hơn bug-late; TDD chọn lọc phần rủi ro cao. `[L-TDD-006]`

### ⑨ Bài tập + tiêu chí tự chấm
1. Làm một feature nhỏ **hoàn toàn bằng TDD** (viết test đỏ trước cho cả happy + 2 biên). **Đạt khi:** có commit đỏ trước commit code; mỗi biên có test riêng.
2. Lấy một test "viết-sau" trong repo → kiểm xem nó có bắt được bug không bằng cách *cố tình* sửa code sai; nếu test vẫn xanh → test rỗng, vá lại. **Đạt khi:** test đỏ khi anh phá hành vi.

### ⑩ Đọc thêm / nguồn chuẩn
- Kent Beck — *Test-Driven Development: By Example*.
- Freeman & Pryce — *Growing Object-Oriented Software, Guided by Tests* (outside-in).
- Dan North — "Introducing BDD"; Cucumber docs (Gherkin).

### 🧠 Tự kiểm tra (active recall)
1. Vì sao "viết test trước" không chỉ là thứ tự mà còn định hình thiết kế?
2. Cho một task mà TDD strict gây cản trở, và cách xử lý.
3. Làm sao phát hiện một test chỉ "đóng băng bug hiện tại"?

<details><summary>Gợi ý chốt</summary>

(1) Test-first buộc nghĩ về interface/testability/spec trước khi implement → đẩy về low coupling. (2) Prototype/exploratory/chưa rõ thiết kế → spike trước, TDD phần đã rõ sau. (3) Không có case đỏ/biên, test đổi mỗi lần refactor; thử phá hành vi xem test có đỏ không; thêm mutation test.
</details>

---

# 📗 BÀI 4 — Test runner & tooling generic (Jest / Vitest / node:test / Supertest)

### ① Mục tiêu & vị trí trong mạch
- **Phủ câu:** `L-RUN-001 → 008` (8 câu). Sau khi đã hiểu *cái gì cần test* (Bài 1–3), bài này là *công cụ chạy test*.
- Học xong: anh biết một "framework test" gồm gì, chọn được runner cho project mới vs legacy theo **stack/ESM**, hiểu chi phí migrate, dùng Supertest, và cô lập test song song.

### ② Giảng cơ bản → nâng cao

**(a) Trực giác.** Jest/Vitest **không chỉ là runner** — chúng là *framework all-in-one*: tìm file + chạy + report (runner), `expect` (assertion), `vi.fn`/`jest.fn` (mocking), v8/istanbul (coverage). `[L-RUN-001]`

**(b) Bối cảnh 2026 (đã verify):** `[L-RUN-002, L-RUN-004, L-RUN-005]`
| Runner | Bản chất | Hợp khi |
|---|---|---|
| **Jest 30.4.2** | chín, ecosystem rộng; gốc CommonJS, ESM cần `--experimental-vm-modules` + transform (ts-jest/babel) | suite lớn/legacy, **React Native (Expo/RN CLI)**, Angular monorepo |
| **Vitest 4.1.9** | Vite-native, **ESM/TS native**, watch rất nhanh; cần Vite ≥6, Node ≥20 | **project mới**, codebase TS/ESM hiện đại |
| **`node:test`** | native Node, **không thêm dependency** | lib/CLI thuần, muốn tối giản |
| **`bun test`** | nhanh khi đã chạy Bun | suite backend TS chạy trên Bun |

> ⚠️ Jest đã được **Meta bàn giao về OpenJS Foundation** (vẫn release đều). Không có "cái nào tốt hơn tuyệt đối" — **quyết theo stack**. `// verify version tại jestjs.io / vitest.dev`

**Migrate Jest ⇄ Vitest** `[L-RUN-003]`: `describe/it/expect` tương thích phần lớn → chi phí **không** ở test body mà ở **config/transform, mock API (`jest.*` → `vi.*`), custom serializer/reporter/preset, khác biệt ESM**. Migrate được nhưng không free.

**Supertest** `[L-RUN-006]`: bắn HTTP request vào instance app **in-process** (truyền `app` vào supertest), assert status/header/body; **không bind cổng OS** → nhanh, song song được. *(Dựng app trong NestJS: xem H-TST.)*

**(c) Mép giới hạn & sai lầm.**
- **ESM khiến Jest hay vỡ** `[L-RUN-005]`: gốc CJS → cần transform + cờ experimental; Vitest đi pipeline Vite nên ESM/TS native, ít config — đây là **động lực dịch chuyển chính**.
- **Test song song mặc định (mỗi file một worker)** `[L-RUN-007]`: xung đột **state/port/DB row** giữa worker → flaky. Cô lập bằng: **schema/DB riêng mỗi worker**, container riêng, **transaction-rollback per test**, tránh global mutable.
- **`--shard`** `[L-RUN-008]`: chia suite lớn qua nhiều máy CI để giảm wall-time; bẫy: **cân tải không đều**, test phụ thuộc thứ tự/global vỡ khi tách, phải **gộp coverage** từ nhiều shard. `// verify cờ tại docs runner`

### ③ ⚠️ Kiến thức cũ / bị thay thế

| Cái cũ | Hiện nay | Vì sao |
|---|---|---|
| Mặc định **Jest** cho mọi project TS mới | **Vitest** cho project mới (ESM/TS native, watch nhanh); Jest cho legacy/RN/Angular | ESM của Jest còn experimental `[L-RUN-002, L-RUN-005]` |
| "Jest là của Meta" | Jest đã về **OpenJS Foundation** (Meta bàn giao), vẫn release đều | Quản trị thay đổi *(verify)* |
| App phải `listen()` cổng để e2e | **Supertest** chạy app in-process, không bind cổng | Nhanh + song song + không đụng cổng OS `[L-RUN-006]` |
| Chạy tuần tự cho an toàn | **Song song + cô lập per-worker** (schema/container/transaction) | Tốc độ; cô lập đúng mới hết flaky `[L-RUN-007]` |
| Một máy CI chạy cả suite | **`--shard`** qua nhiều runner + gộp coverage | Cắt wall-time `[L-RUN-008]` |

### ④ Áp dụng thực tế + So sánh bigtech
- **Use case:** project NestJS + TS mới → cân nhắc Vitest (Nest gần đây hỗ trợ Vitest preset, *verify tại docs Nest*). Codebase RN cũ → giữ Jest.
- **Bigtech pattern:** Meta-internal vẫn Jest (tích hợp Babel/Hermes/Metro sâu); nhiều project OSS mới (vd Astro) đã chuyển Vitest, cắt CI time đáng kể. *(Pattern phổ biến, verify.)*

### ⑤ Code thực hành + cấu hình
```ts
// vitest.config.ts — cô lập song song + coverage (Vitest 4)
import { defineConfig } from "vitest/config";
export default defineConfig({
  test: {
    globals: true,                 // dùng describe/it/expect không cần import
    environment: "node",
    pool: "forks",                 // mỗi file một process — cô lập tốt cho backend
    coverage: { provider: "v8", reporter: ["text", "lcov"], thresholds: { lines: 80, branches: 75 } },
    // tags (v4.1): gắn nhãn nhóm test chậm để cấu hình riêng
  },
});
```
```ts
// Supertest in-process — không listen cổng
import request from "supertest";
import { buildApp } from "./app";
const app = buildApp();
it("GET /health 200", async () => {
  const res = await request(app).get("/health");
  expect(res.status).toBe(200);
});
```
```jsonc
// package.json (verify version trước khi pin)
{ "devDependencies": { "vitest": "4.1.9", "@vitest/coverage-v8": "4.1.9", "supertest": "^7" },
  "scripts": { "test": "vitest run", "test:shard": "vitest run --shard=$SHARD/$TOTAL" } }
```
> ⚠️ **Bẫy "hiểu để kiểm tra":** AI hay sinh config Jest cho project ESM rồi báo lỗi `Cannot use import statement outside a module`. Người hiểu biết đây là **vấn đề ESM/transform**, sửa bằng cấu hình đúng hoặc chuyển Vitest — không "thêm bừa babel preset" cho qua.

### ⑥ Keywords cần nhớ
- 🧠 **Cần HIỂU:** runner = framework all-in-one; vì sao ESM làm Jest đau; chi phí migrate nằm ở đâu; vì sao song song gây flaky.
- 📌 **Cần THUỘC:** Jest 30 vs Vitest 4 khác biệt cốt lõi; Supertest chạy in-process; `--shard` làm gì.
- 🛠️ **Cần LÀM ĐƯỢC:** dựng `vitest.config.ts`; viết Supertest test; cô lập per-worker.

### ⑦ Mental model
> **"Chọn runner theo STACK, không theo hype. Vitest cho mới/ESM, Jest cho legacy/RN. Song song nhanh nhưng phải cô lập tài nguyên hoặc trả giá bằng flaky."**

### ⑧ Câu hỏi phỏng vấn & thách đố
1. *(dễ)* Một test framework cung cấp những khối nào? `[L-RUN-001]`
2. *(TB)* Jest 30 vs Vitest 4 — chọn cái nào cho mới vs legacy, vì sao? `[L-RUN-002]`
3. *(TB)* Supertest vì sao không cần app listen cổng? `[L-RUN-006]`
4. *(TB)* Chi phí migrate Jest→Vitest thật ra nằm ở đâu? `[L-RUN-003]`
5. *(khó)* Test song song gây vấn đề gì với DB/port, cô lập thế nào? `[L-RUN-007]`

**Thách đố/trick:**
- 🎯 *"Vì sao cấu hình ESM của Jest hay vỡ còn Vitest đỡ đau?"* → Jest gốc CJS cần transform + cờ experimental; Vitest pipeline Vite native. `[L-RUN-005]`
- 🎯 *"`--shard` giúp gì và bẫy?"* → cắt wall-time; cân tải lệch + order-dependency + gộp coverage. `[L-RUN-008]`

### ⑨ Bài tập + tiêu chí tự chấm
1. Dựng một project nhỏ chạy cả unit + 1 Supertest e2e với Vitest 4 + coverage v8. **Đạt khi:** `vitest run --coverage` ra report, test song song không flaky.
2. Cấu hình cô lập DB per-worker (schema theo `process.env.VITEST_WORKER_ID` hoặc tương đương). **Đạt khi:** chạy `--pool=forks` nhiều worker không đụng nhau.

### ⑩ Đọc thêm / nguồn chuẩn
- vitest.dev (Getting Started, Migration, CLI `--shard`), jestjs.io (v30 docs).
- Supertest README; Node.js `node:test` docs.

### 🧠 Tự kiểm tra (active recall)
1. Cùng một test body, vì sao migrate Jest→Vitest vẫn tốn công?
2. Vì sao chạy test song song có thể làm chúng đỏ ngẫu nhiên, và bạn cô lập bằng gì?
3. Project TS mới, ESM-first — chọn runner nào và một lý do chính.

<details><summary>Gợi ý chốt</summary>

(1) Body tương thích, nhưng config/transform, `jest.*`→`vi.*`, serializer/reporter/preset, khác biệt ESM phải sửa. (2) Worker dùng chung DB/port/global → xung đột state; cô lập bằng schema/DB riêng, container riêng, transaction-rollback, tránh global mutable. (3) Vitest — ESM/TS native, watch nhanh, ít config.
</details>

---

# 📗 BÀI 5 — Integration test & TestContainers (DB thật)

### ① Mục tiêu & vị trí trong mạch
- **Phủ câu:** `L-INT-001 → 007` (7 câu). Là tầng giữa pyramid (Bài 1); cần hiểu ranh giới loại test + double (Bài 2) để biết *cái gì để thật, cái gì mock*.
- Học xong: anh biết integration với DB thật bắt được gì mà mock không thấy, vận hành TestContainers, cô lập state giữa test, chạy migration đúng prod, và debug "xanh local đỏ CI".

### ② Giảng cơ bản → nâng cao

**(a) Trực giác.** Mock repo giả định "query đúng". Integration test với **DB thật** kiểm chính cái giả định đó — bắt lỗi **SQL/migration/constraint/dialect/transaction/index** mà mock & in-memory không lộ. SQLite-thay-Postgres lệch feature (vd JSONB, array, `ON CONFLICT`). `[L-INT-001]`

**(b) TestContainers** `[L-INT-002]`: spin **Postgres/Redis/Kafka thật trong Docker** theo vòng đời test rồi **destroy**. "Ephemeral + isolated" quan trọng vì: mỗi run **sạch**, không lệ thuộc DB dùng chung → **reproducible**, không "state rò" giữa lần chạy/CI.

**Cô lập state giữa test cùng một DB** `[L-INT-003]`:
| Cách | Tốc độ | Độ cô lập | Bẫy |
|---|---|---|---|
| **Transaction-rollback** per test | nhanh nhất | tốt | **vỡ khi code tự commit** / đa connection |
| **Truncate** bảng | chậm hơn | sạch | phải biết thứ tự FK |
| **Schema/DB mới** mỗi test/worker | chậm nhất | tốt nhất | tốn tài nguyên |

→ Chọn theo cân bằng *tốc độ vs cô lập*.

**Migration trong integration** `[L-INT-004]`: chạy **chính file migration prod** lên container test → bắt lỗi migration thật. `synchronize`/auto-DDL của ORM dựng schema theo entity, **bỏ qua bug trong script migration** → che lỗi sẽ nổ ở prod.

**(c) Mép giới hạn & sai lầm.**
- **Chi phí CI** `[L-INT-005]`: TestContainers làm CI chậm/đắt → tái dùng/giữ ấm container, song song hợp lý, **chỉ integration phần rủi ro** (query/transaction), giữ logic ở unit. Đừng biến mọi test thành integration nặng.
- **Own vs không-own** `[L-INT-006]`: flow chạm Postgres + Redis + HTTP vendor → **DB/cache của mình = container thật** (rủi ro ở đó); **HTTP vendor "không own" = mock/stub** ở ranh giới (hoặc contract test, Bài 10).
- **Xanh local đỏ CI** `[L-INT-007]`: nghi **readiness** (chờ container "ready" chứ không chỉ "started"), tài nguyên CI ít hơn, **race khi song song**, port/clock. Sửa: **wait-strategy** đúng, tăng timeout có chủ đích, cô lập state per worker.

> ➕ *TestContainers + chiến lược cô lập + readiness là phần thực chiến docs gốc thường bỏ qua. Cross-ref `QB_H_nestjs` H-TST-004 cho cách dựng trong Nest.*

### ③ ⚠️ Kiến thức cũ / bị thay thế

| Cái cũ | Hiện nay | Vì sao |
|---|---|---|
| **SQLite in-memory** thay Postgres cho "integration" | **TestContainers** chạy Postgres thật | SQLite lệch dialect/feature → che bug `[L-INT-001]` |
| Dùng **DB dùng chung** (staging) cho test | Container **ephemeral per run** | State rò, không reproducible `[L-INT-002]` |
| `synchronize: true` dựng schema test | Chạy **migration prod thật** lên container | Bỏ qua bug migration `[L-INT-004]` |
| Xoá data bằng `DELETE *` thủ công | **Transaction-rollback / truncate / DB riêng** theo cân nhắc | Cô lập đúng + nhanh `[L-INT-003]` |

### ④ Áp dụng thực tế + So sánh bigtech
- **Use case:** repo `OrderRepo.createWithStockDecrement()` dùng transaction + `SELECT ... FOR UPDATE`. Mock không bao giờ bắt được deadlock/lock thật → **integration với Postgres container** là bắt buộc.
- **Bigtech pattern:** TestContainers (Java/Go/Node) là chuẩn de-facto cho integration DB thật trong CI; nhiều team giữ container "ấm" (reuse) để cắt thời gian. *(Pattern phổ biến, verify.)*

### ⑤ Code thực hành + cấu hình
```ts
// integration với Postgres thật qua @testcontainers/postgresql + Vitest 4
import { describe, it, expect, beforeAll, afterAll, beforeEach } from "vitest";
import { PostgreSqlContainer, StartedPostgreSqlContainer } from "@testcontainers/postgresql";
import { Client } from "pg";

let container: StartedPostgreSqlContainer;
let db: Client;

beforeAll(async () => {
  container = await new PostgreSqlContainer("postgres:16").start(); // ephemeral
  db = new Client({ connectionString: container.getConnectionUri() });
  await db.connect();
  await runMigrations(db);          // CHẠY MIGRATION PROD THẬT, không synchronize
}, 60_000);                          // timeout rộng cho lần kéo image đầu

afterAll(async () => { await db.end(); await container.stop(); });

beforeEach(async () => { await db.query("BEGIN"); });   // cô lập bằng transaction...
// afterEach: await db.query("ROLLBACK")  -> nhanh, sạch (lưu ý: vỡ nếu code tự COMMIT)

it("createWithStockDecrement trừ kho đúng dưới transaction", async () => {
  await db.query("INSERT INTO stock(sku, qty) VALUES ('A1', 100)");
  // ... gọi repo thật, assert side-effect:
  const { rows } = await db.query("SELECT qty FROM stock WHERE sku='A1'");
  expect(rows[0].qty).toBe(98);
});
```
```jsonc
// package.json (verify version)
{ "devDependencies": { "@testcontainers/postgresql": "^11", "pg": "^8", "vitest": "4.1.9" } }
// Yêu cầu: Docker chạy trên máy dev/CI. // verify image tag postgres:16 còn hỗ trợ
```
> ⚠️ **Bẫy "hiểu để kiểm tra":** AI hay viết integration test dùng `synchronize: true` cho nhanh → schema test khác prod, migration script không bao giờ được kiểm. Người hiểu **chạy đúng migration prod** lên container.

### ⑥ Keywords cần nhớ
- 🧠 **Cần HIỂU:** vì sao DB thật bắt lỗi mock không thấy; ephemeral+isolated; own vs không-own; readiness vs started.
- 📌 **Cần THUỘC:** 3 cách cô lập state + trade-off; vì sao chạy migration prod thay vì synchronize.
- 🛠️ **Cần LÀM ĐƯỢC:** dựng TestContainers Postgres; cấu hình wait-strategy; cô lập per-test.

### ⑦ Mental model
> **"Mock kiểm logic; integration kiểm GIAO KÈO với hạ tầng thật. Cái mình sở hữu → container thật; cái không own → mock ở ranh giới. Migration phải chạy như prod."**

### ⑧ Câu hỏi phỏng vấn & thách đố
1. *(TB)* Integration DB thật bắt gì mà mock/in-memory không thấy? `[L-INT-001]`
2. *(TB)* TestContainers vận hành thế nào, vì sao ephemeral quan trọng? `[L-INT-002]`
3. *(khó)* Cô lập state: truncate vs rollback vs DB mới — trade-off? `[L-INT-003]`
4. *(TB)* Vì sao "synchronize schema từ ORM" che lỗi migration? `[L-INT-004]`
5. *(khó)* Flow chạm Postgres + Redis + HTTP vendor — cái nào thật, cái nào mock? `[L-INT-006]`

**Thách đố/trick:**
- 🎯 *"Integration xanh local đỏ ngẫu nhiên CI — debug từ đâu?"* → readiness/wait-strategy, tài nguyên CI, race song song, port/clock. `[L-INT-007]`
- 🎯 *"TestContainers làm CI chậm — TL cân bằng sao?"* → reuse/giữ ấm, song song, chỉ integration phần rủi ro, giữ logic ở unit. `[L-INT-005]`

### ⑨ Bài tập + tiêu chí tự chấm
1. Viết integration test cho một repo dùng transaction + constraint, chạy trên Postgres container. **Đạt khi:** test đỏ nếu constraint/migration sai, xanh khi đúng.
2. Đổi chiến lược cô lập từ truncate sang transaction-rollback và đo thời gian. **Đạt khi:** giải thích được khi nào rollback *không* dùng được (code tự commit).

### ⑩ Đọc thêm / nguồn chuẩn
- testcontainers.com (Node module, wait strategies, reuse).
- Docs ORM của anh (TypeORM/Prisma/Drizzle) — migration vs synchronize.

### 🧠 Tự kiểm tra (active recall)
1. Một bug `ON CONFLICT`/JSONB chỉ nổ trên Postgres — vì sao SQLite-test cho qua?
2. Khi nào transaction-rollback per test KHÔNG dùng được để cô lập?
3. "Xanh local, đỏ CI" — nghi ngờ đầu tiên của bạn là gì?

<details><summary>Gợi ý chốt</summary>

(1) SQLite lệch dialect/feature so với Postgres → query hợp lệ trên SQLite nhưng sai/khác trên Postgres. (2) Khi code dưới test tự `COMMIT` hoặc mở connection riêng → rollback ngoài không bao trùm. (3) Readiness: container "started" nhưng chưa "ready" (chưa accept connection) + tài nguyên CI ít hơn → thêm wait-strategy, tăng timeout.
</details>

---

# 📗 BÀI 6 — Test data: fixture, factory, seed/cleanup & snapshot

### ① Mục tiêu & vị trí trong mạch
- **Phủ câu:** `L-DAT-001 → 006` (6 câu). Nuôi dữ liệu cho integration (Bài 5); cần hiểu isolation để thấy vì sao seed/cleanup phải xác định.
- Học xong: anh tránh được fixture khổng lồ, dùng factory/builder, làm seed→cleanup an toàn, dùng snapshot đúng chỗ, và xử lý dữ liệu nhạy cảm + thời gian/timezone.

### ② Giảng cơ bản → nâng cao

**(a) Fixture & anti-pattern** `[L-DAT-001]`: fixture = data dựng sẵn cho test. **Fixture khổng lồ dùng chung mọi test** là anti-pattern: test **ngầm phụ thuộc nhau**, khó biết test cần gì. → Dùng **data tối thiểu, cục bộ** cho từng test.

**(b) Factory / builder (Object Mother)** `[L-DAT-002]`: tạo entity hợp lệ với **override field cần thiết** → test rõ ý định ("chỉ field X quan trọng"), giảm trùng lặp, dễ bảo trì khi schema đổi. Hơn hẳn hardcode object trong từng test.

**Seed → test → cleanup** `[L-DAT-003]`: thứ tự phải **xác định**. Rủi ro lớn nhất: **cleanup không chạy khi test fail** → state rò sang test sau → flaky/order-dependency. → cleanup phải chạy **kể cả khi fail** (`afterEach` / transaction rollback / DB ephemeral). Đừng dựa vào dữ liệu còn sót.

**(c) Snapshot test** `[L-DAT-004]`: lưu output chuẩn rồi so lần sau. Hợp cho **output ổn định/serialization**. **Bẫy "snapshot rot / approve mù":** dev update snapshot vô tội vạ khi đỏ → snapshot mất ý nghĩa kiểm chứng (chỉ còn xác nhận "output = output").

**Mép giới hạn:**
- **PII / dữ liệu nhạy cảm** `[L-DAT-005]`: dùng **synthetic/anonymized** (faker), **không** copy prod dump chứa PII; rủi ro lộ dữ liệu thật trong fixture/CI log. *(cross-ref M security/PII.)*
- **Thời gian/timezone** `[L-DAT-006]`: test "báo cáo cuối tháng" vỡ khi đổi múi giờ/ngày chạy → **inject clock / freeze time**, test theo **input cố định** không phải `now`, cố định timezone trong test env. Tránh `new Date()` trực tiếp trong logic.

> ➕ *Factory/builder, snapshot-rot, PII trong test là phần kỷ luật dữ liệu test ít được dạy bài bản.*

### ③ ⚠️ Kiến thức cũ / bị thay thế

| Cái cũ | Hiện nay | Vì sao |
|---|---|---|
| Một **fixture lớn** seed sẵn cho cả suite | **Factory/builder** tạo data tối thiểu, cục bộ | Tránh phụ thuộc ngầm giữa test `[L-DAT-001, L-DAT-002]` |
| Cleanup thủ công cuối test | **afterEach/rollback/DB ephemeral** chạy cả khi fail | Fail giữa chừng → rò state `[L-DAT-003]` |
| Snapshot mọi output + `-u` khi đỏ | Snapshot cho output **ổn định**; review snapshot như code | Approve mù giết giá trị kiểm chứng `[L-DAT-004]` |
| Copy **prod dump** vào test cho "thật" | **Synthetic/anonymized** (faker) | Lộ PII, vi phạm bảo mật `[L-DAT-005]` |
| `new Date()` trong logic + test | **Inject clock / freeze time**, input cố định | Test phụ thuộc giờ thật → flaky `[L-DAT-006]` |

### ④ Áp dụng thực tế + So sánh bigtech
- **Use case:** test "user VIP được freeship" → factory `aUser({ tier: "VIP" })` thay vì hardcode object 20 field; chỉ field `tier` quan trọng, phần còn lại default hợp lệ.
- **Bigtech pattern:** factory libraries (fishery/factory-bot-style), faker cho synthetic data, snapshot dùng dè dặt (chủ yếu cho serializer/DTO ổn định). *(Pattern phổ biến.)*

### ⑤ Code thực hành + cấu hình
```ts
// factory/builder — Vitest 4 + faker
import { faker } from "@faker-js/faker";
type User = { id: string; email: string; tier: "FREE" | "VIP"; createdAt: Date };

export function aUser(overrides: Partial<User> = {}): User {  // Object Mother
  return {
    id: faker.string.uuid(),
    email: faker.internet.email(),       // SYNTHETIC, không phải PII thật
    tier: "FREE",
    createdAt: new Date("2026-01-01T00:00:00Z"), // cố định để xác định
    ...overrides,                        // chỉ override field test quan tâm
  };
}

it("VIP được freeship", () => {
  const u = aUser({ tier: "VIP" });      // ý định rõ: chỉ tier quan trọng
  expect(isFreeship(u)).toBe(true);
});
```
```ts
// cleanup chạy CẢ KHI FAIL + freeze time
import { afterEach, beforeEach, vi } from "vitest";
beforeEach(() => { vi.useFakeTimers(); vi.setSystemTime(new Date("2026-06-30T23:59:00Z")); });
afterEach(async () => {
  vi.useRealTimers();
  await db.query("ROLLBACK").catch(() => {}); // cleanup an toàn dù test đỏ
});
```
> ⚠️ **Bẫy "hiểu để kiểm tra":** AI hay sinh test snapshot rồi "approve" mọi thay đổi → snapshot chỉ xác nhận output bằng chính nó. Người hiểu **chỉ snapshot output ổn định** và **đọc kỹ diff** trước khi update.

### ⑥ Keywords cần nhớ
- 🧠 **Cần HIỂU:** vì sao fixture lớn xấu; snapshot rot; vì sao cleanup-khi-fail quan trọng; PII trong test.
- 📌 **Cần THUỘC:** factory/builder pattern; `setSystemTime`/freeze; afterEach cleanup.
- 🛠️ **Cần LÀM ĐƯỢC:** viết factory với override; làm test thời gian deterministic; snapshot đúng phạm vi.

### ⑦ Mental model
> **"Data test: tối thiểu + cục bộ + xác định. Factory dựng đúng cái cần; cleanup phải chạy cả khi đỏ; đừng bê PII thật; đừng để `now` quyết định kết quả."**

### ⑧ Câu hỏi phỏng vấn & thách đố
1. *(dễ)* Fixture là gì, vì sao fixture chung khổng lồ là anti-pattern? `[L-DAT-001]`
2. *(TB)* Factory/builder giải quyết gì so với hardcode? `[L-DAT-002]`
3. *(TB)* Vì sao seed→test→cleanup phải xác định? Rủi ro cleanup không chạy khi fail? `[L-DAT-003]`
4. *(TB)* Snapshot hợp cho gì, bẫy approve mù? `[L-DAT-004]`
5. *(khó)* Test cần PII — sinh data thế nào để an toàn? `[L-DAT-005]`

**Thách đố/trick:**
- 🎯 *"Test 'báo cáo cuối tháng' đỏ khi chạy ngày khác — thiết kế deterministic thế nào?"* → inject clock/freeze, input cố định, cố định timezone. `[L-DAT-006]`

### ⑨ Bài tập + tiêu chí tự chấm
1. Thay 3 test đang hardcode object lớn bằng factory với override. **Đạt khi:** mỗi test chỉ nêu field nó quan tâm, phần còn lại default.
2. Làm một test phụ thuộc thời gian thành deterministic bằng freeze time. **Đạt khi:** chạy ở bất kỳ ngày/timezone nào vẫn xanh ổn định.

### ⑩ Đọc thêm / nguồn chuẩn
- faker-js docs; fishery (factory cho TS).
- Vitest "Fake Timers"; Martin Fowler — "ObjectMother".

### 🧠 Tự kiểm tra (active recall)
1. Vì sao factory tốt hơn fixture lớn dùng chung?
2. Điều gì xảy ra nếu cleanup không chạy khi một test fail giữa chừng?
3. Một dấu hiệu cho thấy snapshot test đã "rot"?

<details><summary>Gợi ý chốt</summary>

(1) Factory tạo data tối thiểu, cục bộ, rõ ý định, tránh phụ thuộc ngầm; fixture lớn làm test ràng buộc nhau. (2) State rò sang test sau → flaky/order-dependency; phải cleanup ở afterEach/rollback/DB ephemeral. (3) Dev update snapshot mỗi lần đỏ mà không đọc diff → snapshot chỉ còn xác nhận output bằng chính nó.
</details>

---

# 📗 BÀI 7 — Flaky test, determinism & isolation

### ① Mục tiêu & vị trí trong mạch
- **Phủ câu:** `L-FLK-001 → 008` (8 câu). Bài *tổng hợp*: gom async (Bài 4), time/random (Bài 2), state/cleanup (Bài 5–6), concurrency (Bài 4) thành một chủ đề — vì sao test "chập chờn" và cách trị.
- Học xong: anh định nghĩa được flaky, liệt kê nguồn, dùng randomize-order để phát hiện, sửa async sai, và **đặt quy trình quản trị flaky cấp team**.

### ② Giảng cơ bản → nâng cao

**(a) Flaky là gì** `[L-FLK-001]`: test pass/fail **không ổn định** dù code không đổi. **Tệ hơn test luôn đỏ** vì: làm **mất niềm tin** vào suite → dev bỏ qua đỏ → **che bug thật**. Test đỏ-luôn ít ra trung thực và bị sửa ngay.

**(b) Nguồn flakiness phổ biến** `[L-FLK-002]`: thời gian/timer thật, randomness, **race/concurrency**, **thứ tự test phụ thuộc**, **state chia sẻ** (DB/global) không cô lập, phụ thuộc service ngoài/mạng, **async chưa await**.

**Order-dependent test** `[L-FLK-003]`: sinh do **shared mutable state/leak** giữa test (global, DB row, mock không reset). **Randomize test order** là công cụ phát hiện tốt vì nó **làm lộ** phụ thuộc ẩn thay vì giấu sau thứ tự cố định.

**Async sai gây flaky** `[L-FLK-004]`: quên `await`/return promise → test **kết thúc trước khi assert chạy** → **false green**. Fix: await đúng, dùng API async của runner, tránh assert trong callback không được chờ.

**(c) Chiến lược xử lý.**
- **Retry vs quarantine vs fix gốc** `[L-FLK-005]`: retry/quarantine để **không chặn pipeline tạm thời** + **phải log/track để fix gốc**. **Retry-mọi-thứ là bẫy** → che bug thật & nuôi flakiness. Cần policy + ownership.
- **Test pollution / shared state** `[L-FLK-006]`: cô lập bằng reset mock / clear DB / đóng kết nối / container per worker. **"Không có state chia sẻ" là kim chỉ nam** — gốc của order-dependency & flaky.
- **Phụ thuộc network/service ngoài** `[L-FLK-007]`: thay bằng **mock server/stub** ở ranh giới (determinism); **record-replay** (nock/MSW) cho HTTP; **contract test** (Bài 10) để vẫn bắt breaking thật mà không gọi mạng mỗi lần.

> ➕ *Toàn bài là kỷ luật chống flaky + quản trị cấp team — phần "TL governance flaky" (008) là chiều lãnh đạo.*

### ③ ⚠️ Kiến thức cũ / bị thay thế

| Cái cũ | Hiện nay | Vì sao |
|---|---|---|
| Flaky? **retry cho qua** | Retry **tạm thời + track + fix gốc**; quarantine có hạn định | Retry-mọi-thứ che bug, nuôi flaky `[L-FLK-005]` |
| Cố định thứ tự test để "ổn định" | **Randomize order** để *phát hiện* phụ thuộc | Thứ tự cố định giấu bug `[L-FLK-003]` |
| Gọi service ngoài thật trong test cho "thật" | **Mock server / record-replay / contract test** | Mạng = nguồn flaky `[L-FLK-007]` |
| `setTimeout` thật + `Date.now()` trong test | **Fake timers + inject clock** (Bài 2/6) | Phụ thuộc giờ thật flaky `[L-FLK-002]` |
| Bỏ mặc flaky như "chuyện nhỏ" | **Đo flaky-rate + owner + SLA dọn** | Bỏ mặc → team tắt CI gate → mất lưới an toàn `[L-FLK-008]` |

### ④ Áp dụng thực tế + So sánh bigtech
- **Use case:** test gọi API tỉ giá thật → đôi khi timeout → đỏ ngẫu nhiên. Sửa: **MSW** chặn HTTP, trả response cố định; thêm 1 contract test để vẫn bắt breaking schema vendor.
- **Bigtech pattern:** Google/Meta có hệ thống **phát hiện flaky tự động** (chạy lại nhiều lần, đo tỉ lệ), **quarantine** test flaky khỏi gate + dashboard owner. *(Pattern phổ biến, không tuyệt đối.)*

### ⑤ Code thực hành + cấu hình
```ts
// async đúng cách — tránh false green
import { it, expect, vi } from "vitest";

// ❌ SAI: quên await → test xanh giả (assert có thể chưa chạy)
it.skip("BAD", () => { doAsync().then(r => expect(r).toBe(1)); });

// ✅ ĐÚNG: await/return promise
it("GOOD", async () => { const r = await doAsync(); expect(r).toBe(1); });

// ✅ MSW: chặn HTTP ngoài → deterministic
import { setupServer } from "msw/node";
import { http, HttpResponse } from "msw";
const server = setupServer(
  http.get("https://fx.vendor.com/rate", () => HttpResponse.json({ usdVnd: 25_000 })),
);
beforeAll(() => server.listen({ onUnhandledRequest: "error" })); // chặn gọi mạng ngoài ý muốn
afterEach(() => server.resetHandlers());
afterAll(() => server.close());
```
```jsonc
// vitest.config.ts — bật randomize order để LỘ order-dependency
{ "test": { "sequence": { "shuffle": true } } } // verify cú pháp tại vitest.dev
```
> ⚠️ **Bẫy "hiểu để kiểm tra":** AI sửa flaky bằng cách **thêm `retry`** hoặc **tăng `sleep`** cho qua → che bug. Người hiểu truy **nguồn determinism** (await/clock/state/network) và sửa gốc.

### ⑥ Keywords cần nhớ
- 🧠 **Cần HIỂU:** vì sao flaky tệ hơn đỏ-luôn; false green do async; vì sao randomize-order phát hiện tốt; vì sao retry-mọi-thứ là bẫy.
- 📌 **Cần THUỘC:** danh sách nguồn flakiness; 3 hướng xử lý (retry/quarantine/fix gốc).
- 🛠️ **Cần LÀM ĐƯỢC:** await đúng; dùng MSW/fake timer; bật shuffle; lập policy flaky.

### ⑦ Mental model
> **"Flaky = thiếu determinism. Xóa nguồn ngẫu nhiên (time, random, network, state chia sẻ, async chưa chờ), đừng dán băng bằng retry. Không state chia sẻ = không order-dependency."**

### ⑧ Câu hỏi phỏng vấn & thách đố
1. *(dễ)* Flaky là gì, vì sao tệ hơn đỏ-luôn? `[L-FLK-001]`
2. *(TB)* Liệt kê nguồn flakiness backend. `[L-FLK-002]`
3. *(TB)* Async sai gây false green thế nào? `[L-FLK-004]`
4. *(khó)* Order-dependent do đâu, vì sao randomize-order phát hiện tốt? `[L-FLK-003]`
5. *(khó)* Retry vs quarantine vs fix gốc — khi nào dùng gì? `[L-FLK-005]`

**Thách đố/trick:**
- 🎯 *"Vì sao retry-mọi-thứ là cái bẫy?"* → che bug thật, nuôi flakiness; cần track + fix gốc. `[L-FLK-005]`
- 🎯 *"TL đặt quy trình quản trị flaky thế nào?"* → đo flaky-rate, quarantine có hạn, gán owner, SLA dọn; bỏ mặc → mất lưới an toàn. `[L-FLK-008]`

### ⑨ Bài tập + tiêu chí tự chấm
1. Bật `sequence.shuffle` trong suite của anh, chạy 5 lần. **Đạt khi:** phát hiện (hoặc chứng minh không có) test order-dependent, và sửa cái nào lộ ra.
2. Thay một test gọi HTTP thật bằng MSW. **Đạt khi:** test xanh ổn định offline + thêm 1 contract test giữ phần bắt breaking.

### ⑩ Đọc thêm / nguồn chuẩn
- Google Testing Blog — "Flaky Tests".
- MSW docs (mock HTTP); Vitest "Test Sequence"/"Retry".

### 🧠 Tự kiểm tra (active recall)
1. Vì sao một test flaky nguy hiểm hơn một test luôn đỏ?
2. Bạn dùng cơ chế nào để *phát hiện* test phụ thuộc thứ tự?
3. Kể 1 cách async sai làm test "xanh giả".

<details><summary>Gợi ý chốt</summary>

(1) Flaky bào mòn niềm tin → dev bỏ qua đỏ → che bug thật; đỏ-luôn trung thực, bị sửa ngay. (2) Randomize/shuffle test order — làm lộ shared state thay vì giấu sau thứ tự cố định. (3) Quên `await`/không return promise → test kết thúc trước khi assertion trong `.then` chạy.
</details>

---

# 📗 BÀI 8 — Coverage & mutation testing & metric chất lượng

### ① Mục tiêu & vị trí trong mạch
- **Phủ câu:** `L-COV-001 → 007` (7 câu). Cần đã có suite (Bài 1–7) để *đo* nó. Đây là chỗ phân biệt "có test" với "test có hiệu lực".
- Học xong: anh phân biệt line/branch/function coverage, hiểu vì sao 100% coverage không = đã test, dùng mutation testing để đo **chất lượng assertion**, đặt gate đúng (patch coverage), và chọn metric outcome.

### ② Giảng cơ bản → nâng cao

**(a) Các loại coverage** `[L-COV-001]`:
- **Line:** dòng nào *được chạy qua*.
- **Branch:** mỗi nhánh true/false *được phủ* — **thông tin hơn line** vì line cao vẫn bỏ lọt nhánh (vd `else` chưa test).
- **Function:** hàm nào được gọi.

**(b) Vì sao coverage "lừa"** `[L-COV-002]`: coverage đo **dòng được chạy**, **không** đo assertion có nghĩa. Test gọi hàm rồi **không assert gì** vẫn tính phủ. → Coverage là **cận dưới** ("chỗ này chưa chạy = chắc chắn chưa test"), **không** là đảm bảo chất lượng.

```ts
// 100% line coverage nhưng test RỖNG:
it("rỗng nhưng phủ 100%", () => { shippingFee(2, 10); }); // chạy qua, KHÔNG assert
```

**Mutation testing** `[L-COV-003]`: gài lỗi nhỏ vào code (đổi `>`→`>=`, xoá dòng, đổi `+`→`-`) tạo **mutant**, rồi xem test có **"giết"** mutant không (test đỏ = giết được). Đo **chất lượng assertion** chứ không chỉ dòng chạy. **Mutant sống** = test rỗng/yếu chỗ đó. **Mutation score** = % mutant bị giết. *(Stryker — verify.)*

**(c) Mép giới hạn & metric.**
- **Coverage threshold làm gate có mặt trái** `[L-COV-004]`: ép "90% line" → dev viết **test phủ-cho-đủ-số không assert** (gaming). → Kết hợp **branch + mutation + review** thay vì một con số line tuyệt đối.
- **Patch coverage vs repo coverage** `[L-COV-005]`: gate theo coverage **code mới đụng trong PR** (patch) tránh bắt cả repo legacy lên chuẩn ngay → nâng dần chất lượng theo từng PR. Thực dụng cho legacy.
- **Metric sức khỏe suite** `[L-COV-006]`: tốc độ → feedback loop; flaky-rate → niềm tin; **defect-escape-rate** (bug lọt prod) → hiệu lực thật; mutation → chất lượng assertion. Coverage chỉ là **một mảnh**.

> ➕ *Mutation testing + patch coverage + metric outcome là phần "đo chất lượng test" docs gốc gần như không chạm.*

### ③ ⚠️ Kiến thức cũ / bị thay thế

| Cái cũ | Hiện nay | Vì sao |
|---|---|---|
| **Line coverage %** là thước đo chất lượng | **Branch + mutation score + outcome metrics** | Line đo dòng chạy, không đo assertion `[L-COV-001, L-COV-002]` |
| Gate cứng "**90% line** toàn repo" | **Patch coverage** + branch + mutation + review | Ép số → test rác/gaming; legacy không lên chuẩn ngay được `[L-COV-004, L-COV-005]` |
| "Coverage cao = ít bug" | Theo **defect-escape-rate / mutation** | Coverage là proxy đầu vào, outcome mới đo kết quả `[L-COV-006, L-COV-007]` |

### ④ Áp dụng thực tế + So sánh bigtech
- **Use case:** service thanh toán coverage 95% nhưng prod vẫn bug → chạy **Stryker**, phát hiện mutation score thấp ở module tính phí (nhiều mutant sống) → test assert nông. Bổ sung assert + case biên.
- **Bigtech pattern:** dùng **patch/diff coverage** (Codecov/Coveralls) làm gate PR thay vì repo coverage; mutation testing chạy **định kỳ/nightly** (đắt) cho module critical. *(Pattern phổ biến, verify.)*

### ⑤ Code thực hành + cấu hình
```jsonc
// vitest.config.ts — branch threshold (không chỉ line)
{ "test": { "coverage": {
  "provider": "v8",
  "thresholds": { "lines": 80, "branches": 80, "functions": 80 }, // branch quan trọng
  "reporter": ["text", "lcov"]
}}}
```
```jsonc
// stryker.config.json — mutation testing (verify version Stryker)
{
  "packageManager": "npm",
  "testRunner": "vitest",
  "coverageAnalysis": "perTest",
  "mutate": ["src/**/*.ts", "!src/**/*.test.ts"],
  "thresholds": { "high": 80, "low": 60, "break": 50 } // CI fail nếu score < break
}
// chạy: npx stryker run   // verify tại stryker-mutator.io
```
> ⚠️ **Bẫy "hiểu để kiểm tra":** AI sinh test để **"đạt 90% coverage"** → nhiều test gọi-hàm-không-assert. Người hiểu chạy **mutation test**: nếu code đổi mà test vẫn xanh → test rỗng, dù coverage đẹp.

### ⑥ Keywords cần nhớ
- 🧠 **Cần HIỂU:** vì sao branch > line; vì sao coverage không = chất lượng; mutation score đo gì; gaming khi ép số.
- 📌 **Cần THUỘC:** 3 loại coverage; định nghĩa mutant/mutation score; patch coverage.
- 🛠️ **Cần LÀM ĐƯỢC:** đặt branch threshold; chạy Stryker; đọc mutant sống → bổ sung assert.

### ⑦ Mental model
> **"Coverage nói 'đã chạy', mutation nói 'đã KIỂM'. Đừng tôn thờ % line — đo branch + mutation + defect-escape. Gate patch coverage cho legacy, đừng ép cả repo."**

### ⑧ Câu hỏi phỏng vấn & thách đố
1. *(dễ)* Line vs branch vs function coverage; vì sao branch thông tin hơn? `[L-COV-001]`
2. *(TB)* Vì sao 100% coverage ≠ đã test đúng? `[L-COV-002]`
3. *(khó)* Mutation testing đo gì mà coverage không đo? `[L-COV-003]`
4. *(TB)* Patch coverage vs repo coverage — vì sao patch thực dụng cho legacy? `[L-COV-005]`
5. *(TB)* Ngoài coverage, metric nào đo sức khỏe suite? `[L-COV-006]`

**Thách đố/trick:**
- 🎯 *"Service coverage 95% mà prod lắm bug — chẩn đoán từ đâu?"* → assert nông/mock nông, thiếu integration/contract, thiếu case biên, mutation score thấp, đo sai thứ. `[L-COV-007]`
- 🎯 *"Ép 90% line có mặt trái gì?"* → test phủ-cho-đủ-số không assert (gaming). `[L-COV-004]`

### ⑨ Bài tập + tiêu chí tự chấm
1. Bật branch threshold + chạy coverage cho 1 module. **Đạt khi:** tìm được 1 nhánh `else`/error chưa test mà line coverage giấu.
2. Chạy Stryker trên module logic. **Đạt khi:** tìm ≥1 mutant sống và bổ sung assert để giết nó.

### ⑩ Đọc thêm / nguồn chuẩn
- stryker-mutator.io (StrykerJS).
- Vitest Coverage; Codecov "patch vs project coverage".
- DORA metrics (change-failure-rate, MTTR) — bối cảnh outcome.

### 🧠 Tự kiểm tra (active recall)
1. Một test "gọi hàm, không assert" — coverage tính nó thế nào, vấn đề ở đâu?
2. Mutation score thấp nói lên điều gì về test của bạn?
3. Vì sao gate **patch** coverage hợp lý hơn gate repo coverage cho codebase legacy?

<details><summary>Gợi ý chốt</summary>

(1) Vẫn tính phủ (dòng được chạy) → coverage đẹp giả; nó đo dòng chạy không đo assertion có nghĩa. (2) Nhiều mutant sống → assertion nông/rỗng, test không thật sự kiểm hành vi. (3) Gate code mới đụng → nâng chất lượng dần theo PR, không block vì nợ cũ của cả repo.
</details>

---

# 📗 BÀI 9 — Load / performance test (k6 / Artillery)

### ① Mục tiêu & vị trí trong mạch
- **Phủ câu:** `L-LOAD-001 → 006` (6 câu). Trục **non-functional** riêng, đứng độc lập sau testing chức năng. Cần hiểu CI gate (sẽ nối Bài 11).
- Học xong: anh phân biệt load/stress/spike/soak, dùng k6/Artillery, đặt threshold theo percentile, hiểu closed vs open model, đọc điểm bão hoà, và tích hợp perf test vào pipeline mà không làm CI chậm.

### ② Giảng cơ bản → nâng cao

**(a) 4 loại perf test** `[L-LOAD-001]`:
| Loại | Trả lời câu hỏi |
|---|---|
| **Load** | tải kỳ vọng → có đạt SLO không? |
| **Stress** | đẩy tới gãy → điểm vỡ ở đâu, suy thoái thế nào? |
| **Spike** | tăng đột ngột → auto-scale/queue chịu được không? |
| **Soak (endurance)** | chạy lâu → memory leak / degradation dần? |

**(b) Công cụ** `[L-LOAD-002]`: **k6** (Grafana, JS/TS native, code-first, CLI, exit non-zero khi threshold fail → CI gate dễ) / **Artillery** (YAML config). Đo latency/throughput/error-rate. **Chạy load từ máy local cho số sai** vì máy local nghẽn CPU/network của chính nó & quá gần server → không phản ánh tải thật phân tán; cần env giống prod (hoặc load zone phân tán).

**Threshold theo percentile** `[L-LOAD-003]`: đặt SLO assertion (vd `p95 < 300ms`, `error-rate < 1%`) để CI fail tự động. **Đo p95/p99 thay vì trung bình** vì trung bình **giấu đuôi chậm**; p95/p99 phản ánh trải nghiệm xấu nhất của phần lớn user.

**(c) Mép giới hạn & đọc kết quả.**
- **Closed vs open model** `[L-LOAD-004]`: **closed** = cố định VU (N user lặp request) → throughput **tự co khi server chậm** → *che* nghẽn. **open** = cố định arrival rate (request tới bất kể server) → mô phỏng traffic thật, **lộ queue/đổ vỡ**. Chọn sai model → kết luận sai về capacity.
- **Đọc điểm bão hoà** `[L-LOAD-005]`: throughput tăng nhưng **p99 vọt + error-rate lên** → đã qua **"knee"/điểm bão hoà**; thêm tải không thêm throughput mà chỉ tăng queue/latency/timeout. Tìm **bottleneck** (DB pool, CPU, lock) trước khi scale mù.
- **Tích hợp pipeline** `[L-LOAD-006]`: **smoke-load nhẹ mỗi PR** (gate threshold) + **full load định kỳ/pre-release** ở env riêng; giữ **baseline** + so sánh để bắt regression; **tách khỏi unit pipeline** để không chậm feedback.

> ➕ *Closed/open model + đọc knee + perf-in-CI là phần senior/TL ít tài liệu cơ bản dạy.*

### ③ ⚠️ Kiến thức cũ / bị thay thế

| Cái cũ | Hiện nay | Vì sao |
|---|---|---|
| **JMeter** GUI + XML | **k6** (JS/TS, code-first, CI-friendly) / Artillery (YAML) | Review được trong Git, nhẹ, tích hợp CI `[L-LOAD-002]` |
| Đo **latency trung bình** | **p95/p99** + error-rate | Trung bình giấu đuôi chậm `[L-LOAD-003]` |
| Chạy load từ **laptop** | Env giống prod / **load zone phân tán** | Local nghẽn tài nguyên → số sai `[L-LOAD-002]` |
| Mặc định **closed-model** (cố định VU) | Chọn **open-model** khi mô phỏng traffic thật | Closed che nghẽn, kết luận sai capacity `[L-LOAD-004]` |
| Perf test thủ công cuối dự án | **Smoke-load mỗi PR + full pre-release**, có baseline | Bắt regression sớm, không chậm CI `[L-LOAD-006]` |

### ④ Áp dụng thực tế + So sánh bigtech
- **Use case:** trước Black Friday, chạy **spike test** mô phỏng traffic tăng 10x trong 30s xem auto-scale + queue chịu được; **soak test** 8 giờ tìm memory leak trong worker.
- **Bigtech pattern:** k6 (Grafana Cloud k6 cho distributed) phổ biến cho team DevOps/microservices; tích hợp threshold vào pipeline + dashboard Grafana. *(Pattern phổ biến, verify.)*

### ⑤ Code thực hành + cấu hình
```js
// k6 script — load test với threshold percentile (k6 2.0, JS/TS)
import http from "k6/http";
import { check, sleep } from "k6";

export const options = {
  scenarios: {
    // OPEN model: cố định arrival rate (mô phỏng traffic thật, lộ nghẽn)
    constant_rate: {
      executor: "constant-arrival-rate",
      rate: 100, timeUnit: "1s", duration: "2m",
      preAllocatedVUs: 50, maxVUs: 200,
    },
  },
  thresholds: {                       // CI fail tự động nếu vi phạm
    http_req_duration: ["p(95)<300", "p(99)<800"],  // percentile, KHÔNG dùng avg
    http_req_failed: ["rate<0.01"],                 // error-rate < 1%
  },
};

export default function () {
  const res = http.get("https://staging.api.example.com/orders");
  check(res, { "status 200": (r) => r.status === 200 });
  sleep(1);
}
// chạy: k6 run script.js   // exit code != 0 khi threshold fail -> pipeline đỏ
```
> ⚠️ **Bẫy "hiểu để kiểm tra":** AI hay viết load test **closed-model** (cố định VU) và báo "p95 ổn" — nhưng throughput tự co khi server chậm nên **che nghẽn**. Người hiểu chọn **open-model** khi cần biết capacity thật, và **đọc p99 + error-rate** chứ không chỉ avg.

### ⑥ Keywords cần nhớ
- 🧠 **Cần HIỂU:** closed vs open model; vì sao p95/p99 > avg; điểm knee/bão hoà; vì sao local cho số sai.
- 📌 **Cần THUỘC:** 4 loại perf test; threshold + exit-code làm CI gate.
- 🛠️ **Cần LÀM ĐƯỢC:** viết k6 script với threshold percentile; đọc kết quả tìm bottleneck; gài smoke-load vào CI.

### ⑦ Mental model
> **"Load = đạt SLO?, Stress = gãy ở đâu?, Spike = scale kịp?, Soak = rò không? Đo ĐUÔI (p95/p99) không đo trung bình. Open-model lộ nghẽn; smoke mỗi PR, full pre-release."**

### ⑧ Câu hỏi phỏng vấn & thách đố
1. *(TB)* Phân biệt load/stress/spike/soak — mỗi loại trả lời gì? `[L-LOAD-001]`
2. *(TB)* Vì sao chạy load từ local cho số sai? `[L-LOAD-002]`
3. *(khó)* Vì sao đo p95/p99 thay vì trung bình? `[L-LOAD-003]`
4. *(khó)* Closed vs open model — chọn sai dẫn tới kết luận sai gì? `[L-LOAD-004]`
5. *(khó)* TL tích hợp perf test vào CI thế nào để bắt regression mà không chậm? `[L-LOAD-006]`

**Thách đố/trick:**
- 🎯 *"Throughput tăng nhưng p99 vọt + error-rate lên — suy ra gì?"* → qua điểm bão hoà; tìm bottleneck (DB pool/CPU/lock) trước khi scale. `[L-LOAD-005]`

### ⑨ Bài tập + tiêu chí tự chấm
1. Viết 1 k6 script open-model có threshold p95/p99 + error-rate, chạy vào staging. **Đạt khi:** pipeline đỏ khi vi phạm threshold, xanh khi đạt.
2. Chạy stress test tăng dần tải đến khi error-rate lên. **Đạt khi:** xác định được điểm knee và 1 bottleneck nghi ngờ.

### ⑩ Đọc thêm / nguồn chuẩn
- grafana.com/docs/k6 (test types, thresholds, scenarios, open vs closed model).
- Artillery docs.

### 🧠 Tự kiểm tra (active recall)
1. Vì sao báo cáo p95/p99 quan trọng hơn latency trung bình?
2. Closed-model có thể *che* vấn đề gì của hệ thống?
3. Cách tích hợp perf test vào CI mà không làm dev bỏ qua pipeline?

<details><summary>Gợi ý chốt</summary>

(1) Trung bình giấu đuôi chậm; p95/p99 phản ánh trải nghiệm xấu nhất của phần lớn user. (2) Throughput tự co khi server chậm → che nghẽn/điểm bão hoà thật. (3) Smoke-load nhẹ + threshold mỗi PR, full load định kỳ/pre-release ở env riêng, có baseline so sánh, tách khỏi unit pipeline.
</details>

---

# 📗 BÀI 10 — Contract test: vị trí trong pyramid & CI gating

### ① Mục tiêu & vị trí trong mạch
- **Phủ câu:** `L-CON-001 → 003` (3 câu). Microservices-specific; nối Bài 1 (pyramid microservices) + Bài 7 (thay e2e xuyên service). **Cross-ref `QB_F_apidesign`** (Pact giải quyết vấn đề gì — không lặp ở đây).
- Học xong: anh biết contract test nằm đâu trong pyramid, vì sao nó thay phần lớn e2e xuyên service, cơ chế gating (broker/can-i-deploy), và **hạn chế** của nó.

### ② Giảng cơ bản → nâng cao

**(a) Vị trí trong pyramid** `[L-CON-001]`: contract test ở **tầng giữa**. Mỗi service test **độc lập** với "contract" của láng giềng → bắt **breaking integration** mà **không cần dựng cả hệ thống chạy cùng lúc** (vốn brittle/đắt). Vì thế nó **thay thế được phần lớn e2e xuyên service**.

**(b) Cơ chế consumer-driven + gating** `[L-CON-002]`:
1. **Consumer** (bên gọi) định nghĩa expectation → **publish contract lên broker** (Pact Broker).
2. **Producer** (bên cung cấp) chạy verify: "mình có thoả contract của mọi consumer đang chạy không?".
3. **`can-i-deploy`** chặn producer release nếu nó **phá contract** của consumer đang ở môi trường đích.
→ Gate này **ngăn breaking lan ra prod** mà không cần e2e xuyên hệ thống. *(Pact — verify.)*

**(c) Hạn chế / bẫy** `[L-CON-003]`: contract test chỉ kiểm **hình dạng tương tác** hai bên đồng thuận (schema, field, status). **KHÔNG** kiểm logic nghiệp vụ end-to-end hay hành vi thật của DB/side-effect. → Vẫn cần **ít integration + smoke e2e** bổ sung. Không thay e2e **hoàn toàn**.

> ➕ *Phần "vị trí pyramid + gating + hạn chế" là góc TL; cơ chế Pact cơ bản đã ở mục F.*

### ③ ⚠️ Kiến thức cũ / bị thay thế

| Cái cũ | Hiện nay | Vì sao |
|---|---|---|
| **E2E xuyên nhiều service** để bắt breaking integration | **Contract test** mỗi service + ít smoke e2e | E2E-xuyên-service brittle/đắt, chậm `[L-CON-001]` |
| Deploy producer rồi chờ xem consumer có vỡ | **`can-i-deploy`** gate trước deploy (broker) | Ngăn breaking *trước khi* lan prod `[L-CON-002]` |
| Tin contract test phủ hết | Bổ sung **integration + smoke e2e** cho logic/side-effect | Contract chỉ kiểm hình dạng tương tác `[L-CON-003]` |

### ④ Áp dụng thực tế + So sánh bigtech
- **Use case:** `web-bff` (consumer) gọi `order-service` (producer). Consumer publish contract "GET /orders/:id trả {id, status, total}". Producer CI verify + `can-i-deploy` → nếu producer định đổi `total`→`amount` mà chưa version, gate chặn deploy.
- **Bigtech pattern:** consumer-driven contract (Pact + Pact Broker / PactFlow) phổ biến ở tổ chức nhiều team/nhiều service; gắn `can-i-deploy` vào pipeline. *(Pattern phổ biến, verify.)*

### ⑤ Code thực hành + cấu hình
```ts
// Consumer side (Pact) — định nghĩa expectation rồi publish (verify API tại docs Pact JS)
// pseudo-minimal, đối chiếu @pact-foundation/pact phiên bản hiện tại
import { PactV4, MatchersV3 } from "@pact-foundation/pact"; // verify package + API
const provider = new PactV4({ consumer: "web-bff", provider: "order-service" });

it("contract: GET /orders/:id", () => provider
  .addInteraction()
  .uponReceiving("a request for an order")
  .withRequest("GET", "/orders/42")
  .willRespondWith(200, (b) => b.jsonBody({
    id: MatchersV3.integer(42),
    status: MatchersV3.string("created"),
    total: MatchersV3.decimal(120000),   // hình dạng, không phải logic
  }))
  .executeTest(async (mock) => {
    // gọi client thật vào mock.url, assert nó parse đúng
  }));
// CI: publish pact lên broker; producer verify; `pact-broker can-i-deploy` gate deploy
```
```bash
# Gate ở pipeline (verify cú pháp CLI Pact Broker)
pact-broker can-i-deploy --pacticipant order-service --version $GIT_SHA --to-environment production
```
> ⚠️ **Bẫy "hiểu để kiểm tra":** AI/dev dễ tưởng "có contract test là khỏi e2e". Người hiểu biết contract chỉ chốt **hình dạng tương tác**; logic nghiệp vụ + side-effect vẫn cần integration + 1 ít smoke e2e.

### ⑥ Keywords cần nhớ
- 🧠 **Cần HIỂU:** vì sao contract thay phần lớn e2e xuyên service; consumer-driven; vì sao verify *trước* deploy; hạn chế.
- 📌 **Cần THUỘC:** vị trí tầng giữa pyramid; broker + `can-i-deploy`.
- 🛠️ **Cần LÀM ĐƯỢC:** viết 1 contract consumer-side; gài `can-i-deploy` vào pipeline.

### ⑦ Mental model
> **"Contract test = chốt GIAO KÈO hình dạng giữa hai service, kiểm độc lập, gate bằng can-i-deploy. Nó thay e2e-xuyên-service nhưng KHÔNG thay integration + smoke cho logic/side-effect."**

### ⑧ Câu hỏi phỏng vấn & thách đố
1. *(khó)* Contract test nằm đâu trong pyramid microservices, vì sao thay phần lớn e2e? `[L-CON-001]`
2. *(rất khó)* Consumer-driven contract chạy như CI gate thế nào (broker, can-i-deploy)? Vì sao producer phải verify trước deploy? `[L-CON-002]`
3. *(khó)* Hạn chế của contract test, vì sao không thay integration/e2e hoàn toàn? `[L-CON-003]`

**Thách đố/trick:**
- 🎯 *"Có contract test rồi, bỏ hết e2e được chưa?"* → Không: contract chỉ kiểm hình dạng tương tác; cần ít integration + smoke e2e cho logic/side-effect. `[L-CON-003]`

### ⑨ Bài tập + tiêu chí tự chấm
1. Viết 1 consumer contract cho một endpoint nội bộ, publish (broker local nếu có). **Đạt khi:** producer verify đỏ khi anh cố đổi schema response.
2. Mô tả pipeline gating với `can-i-deploy`. **Đạt khi:** chỉ ra điểm nào trong CI chặn producer phá contract.

### ⑩ Đọc thêm / nguồn chuẩn
- docs.pact.io (consumer-driven, Pact Broker, can-i-deploy).
- `QB_F_apidesign.md` (Pact giải quyết vấn đề gì) — cross-ref.

### 🧠 Tự kiểm tra (active recall)
1. Vì sao contract test "rẻ" hơn e2e xuyên nhiều service mà vẫn bắt breaking integration?
2. `can-i-deploy` ngăn điều gì, dựa trên dữ liệu gì?
3. Một loại bug mà contract test KHÔNG bắt được?

<details><summary>Gợi ý chốt</summary>

(1) Mỗi service test độc lập với contract của láng giềng, không cần dựng cả hệ thống chạy cùng lúc. (2) Ngăn producer deploy bản phá contract của consumer đang chạy ở môi trường đích; dựa trên contract đã publish lên broker. (3) Bug logic nghiệp vụ end-to-end / side-effect DB — contract chỉ kiểm hình dạng tương tác.
</details>

---

# 📗 BÀI 11 — Chuẩn chất lượng cấp Tech Lead (capstone)

### ① Mục tiêu & vị trí trong mạch
- **Phủ câu:** `L-TL-001 → 008` (8 câu). **Capstone** — tổng hợp toàn bộ Bài 1–10 thành **quyết sách team**: DoD, CI gate, cắt thời gian suite, đưa test vào legacy, chất lượng test-code, thương lượng với PM, metric outcome, và kiểm test do AI viết.
- Học xong: anh hành xử như TL — không chỉ viết test mà **đặt chuẩn** và **đo hiệu lực** cho cả team.

### ② Giảng cơ bản → nâng cao

**(a) Definition of Done về test** `[L-TL-001]`: gắn yêu cầu test **theo rủi ro/loại thay đổi**, không "một-cỡ-cho-tất-cả":
- Logic mới → **unit**.
- Endpoint mới → **integration/API**.
- Bug fix → **regression test tái hiện bug** (đỏ trước khi fix, xanh sau).

**(b) CI quality gate nhiều tầng** `[L-TL-002]`:
- **Fast gate (chặn merge):** lint → type-check → unit. Rẻ + quyết định nhanh.
- **Stage sau / nightly (có thể chỉ cảnh báo hoặc gate riêng):** integration, e2e/smoke nặng, mutation.
- Nguyên tắc: **gate cứng cái rẻ-và-quyết-định**; cân thời gian feedback vs độ phủ.

**Cắt thời gian suite 40 phút** `[L-TL-003]`: **đo trước khi tối ưu** → song song/shard (Bài 4), **đẩy test xuống tầng thấp** (bớt e2e thừa), tách smoke vs full, cache/parallel container (Bài 5), xoá test trùng/chậm vô ích.

**(c) Vận hành & con người.**
- **Legacy gần như không test** `[L-TL-004]`: **characterization test** làm lưới an toàn cho phần đụng tới, gate **coverage trên diff mới** (Bài 8), bọc seam dần (*cross-ref Q legacy*), ưu tiên module rủi ro cao. **Tăng dần, không big-bang** (không dừng feature 3 tháng).
- **Chất lượng test-code** `[L-TL-005]`: test là code phải bảo trì → đọc **như spec** (tên rõ hành vi), tránh logic phức tạp trong test, **một test một hành vi**, tránh **over-DRY** giấu ý định; **review test như review code thật**.
- **PM ép cắt test vì deadline** `[L-TL-006]`: nói bằng **rủi ro/chi phí bug-prod**, đề xuất **cắt có chọn lọc** (giữ critical-path test), **ghi nợ tường minh + kế hoạch trả**; không "cắt im lặng". Cân bối cảnh, không giáo điều.
- **Đo hiệu quả thật** `[L-TL-007]`: ngoài coverage → **defect-escape-rate** (bug lọt prod), **change-failure-rate**, **MTTR**. Đây là **outcome** đo kết quả; coverage chỉ là proxy đầu vào.
- **Kiểm test do AI viết** `[L-TL-008]`: AI hay viết test **bám implementation / assert nông / mock-mọi-thứ / snapshot-approve mù** → xanh nhưng rỗng. Kiểm bằng: **mutation test** (Bài 8), đọc **assertion có nghĩa không**, có **case biên/lỗi** không. **Người chỉ huy spec, AI lo cú pháp.**

> ➕ *Toàn bài là chiều lãnh đạo kỹ thuật — gần như không có trong tài liệu testing cơ bản.*

### ③ ⚠️ Kiến thức cũ / bị thay thế

| Cái cũ | Hiện nay | Vì sao |
|---|---|---|
| DoD "có test" chung chung | **DoD theo loại thay đổi** (logic→unit, endpoint→integration, bug→regression) | Một-cỡ không khớp rủi ro `[L-TL-001]` |
| Một gate chạy **tất cả** test chặn merge | **Tiered gate**: fast chặn merge, nặng ở stage sau/nightly | Cân feedback vs phủ `[L-TL-002]` |
| Legacy: "dừng lại viết test cho tất cả" | **Characterization + diff-coverage gate**, tăng dần | Không thể dừng feature 3 tháng `[L-TL-004]` |
| Đánh giá test bằng **coverage** | **defect-escape, change-failure-rate, MTTR** (DORA) | Outcome đo kết quả, coverage chỉ proxy `[L-TL-007]` |
| Tin test AI vì "xanh hết" | **Mutation + đọc assertion + case biên** | AI đúng cú pháp, sai bản chất `[L-TL-008]` |

### ④ Áp dụng thực tế + So sánh bigtech
- **Use case:** team mới nhận service legacy 5% coverage, prod hay cháy. TL: (1) đặt diff-coverage gate 80% cho PR mới; (2) viết characterization test quanh module thanh toán trước khi sửa; (3) đo defect-escape hàng sprint; (4) review test-code trong PR như code thật.
- **Bigtech pattern:** Google "Beyoncé Rule" (nếu anh quan tâm, hãy có test); DORA metrics (change-failure-rate, MTTR) là chuẩn đo hiệu năng delivery; tiered CI gate phổ biến. *(Pattern phổ biến, verify.)*

### ⑤ Code thực hành + cấu hình
```yaml
# .github/workflows/ci.yml — tiered quality gate (verify cú pháp action版本)
name: ci
on: [pull_request]
jobs:
  fast-gate:                       # CHẶN MERGE — rẻ & quyết định
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - run: npm ci
      - run: npm run lint
      - run: npm run typecheck
      - run: npm run test -- --coverage   # unit + branch threshold (Bài 8)
      # diff coverage gate cho PR (Codecov/Coveralls) — patch coverage, không repo
  integration:                     # stage sau — nặng hơn
    needs: fast-gate
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - run: npm ci
      - run: npm run test:integration   # TestContainers (Bài 5)
  # nightly job riêng: mutation (Stryker) + full load (k6) — không chặn mỗi PR
```
```ts
// characterization test cho legacy — "đóng băng hành vi HIỆN TẠI" làm lưới an toàn TRƯỚC khi refactor
it("characterize: legacy calcInvoice giữ nguyên output cho input đã biết", () => {
  // KHÔNG khẳng định đúng/sai — chỉ chốt hành vi hiện tại để refactor an toàn
  expect(calcInvoiceLegacy(sampleInput)).toEqual(knownCurrentOutput);
});
```
> ⚠️ **Bẫy "hiểu để kiểm tra" (đỉnh điểm):** bảo AI "viết test cho cả file" → 30 test xanh, coverage 95%, nhưng mock-mọi-thứ + assert nông. TL **không** merge vì tin màu xanh; chạy mutation, đọc xem có case biên/lỗi, hỏi "tương tác này có phải contract không". *Đây là điểm cốt lõi: người hiểu để CHỈ HUY & KIỂM TRA, AI lo cú pháp.*

### ⑥ Keywords cần nhớ
- 🧠 **Cần HIỂU:** DoD theo rủi ro; tiered gate; characterization test; outcome metrics vs proxy; vì sao test AI dễ rỗng.
- 📌 **Cần THUỘC:** loại test theo loại thay đổi; defect-escape / change-failure-rate / MTTR; fast-gate gồm gì.
- 🛠️ **Cần LÀM ĐƯỢC:** thiết kế CI gate tiered; đưa test vào legacy không big-bang; review & kiểm test AI bằng mutation.

### ⑦ Mental model
> **"TL không viết nhiều test hơn — TL đặt CHUẨN đúng chỗ và ĐO hiệu lực. Gate cứng cái rẻ; legacy tăng dần; đo outcome (bug lọt prod) không đo proxy (coverage); màu xanh của test AI phải bị nghi ngờ cho tới khi mutation chứng minh."**

### ⑧ Câu hỏi phỏng vấn & thách đố
1. *(TB)* TL đặt DoD về test theo loại thay đổi thế nào? `[L-TL-001]`
2. *(khó)* Thiết kế CI quality gate gồm tầng nào, cái gì chặn merge vs cảnh báo? `[L-TL-002]`
3. *(khó)* Suite 40 phút khiến team bỏ CI — ưu tiên hành động nào? `[L-TL-003]`
4. *(khó)* Đưa test vào legacy mà không dừng feature 3 tháng? `[L-TL-004]`
5. *(rất khó)* Đo hiệu quả thật của chiến lược test ngoài coverage bằng gì? `[L-TL-007]`

**Thách đố/trick:**
- 🎯 *"PM ép cắt test kịp deadline — thương lượng sao?"* → nói bằng rủi ro bug-prod, cắt có chọn lọc giữ critical-path, ghi nợ + kế hoạch trả, không cắt im lặng. `[L-TL-006]`
- 🎯 *"AI viết 30 test xanh, coverage 95% — kiểm chất lượng thế nào?"* → mutation test, đọc assertion có nghĩa, case biên/lỗi; xanh ≠ có hiệu lực. `[L-TL-008]`

### ⑨ Bài tập + tiêu chí tự chấm
1. Viết DoD-test 1 trang cho team + thiết kế tiered CI gate. **Đạt khi:** phân biệt rõ fast-gate (chặn merge) vs stage sau, gắn loại test theo loại thay đổi.
2. Lấy 1 PR test do AI sinh → chạy mutation + tìm test rỗng + vá. **Đạt khi:** chỉ ra ≥1 test xanh nhưng không bắt được bug (mutant sống) và sửa.

### ⑩ Đọc thêm / nguồn chuẩn
- DORA / *Accelerate* (Forsgren, Humble, Kim) — change-failure-rate, MTTR.
- Michael Feathers — *Working Effectively with Legacy Code* (characterization test, seam).
- Google — *Software Engineering at Google* (testing culture, Beyoncé Rule).

### 🧠 Tự kiểm tra (active recall)
1. Cái gì nên ở fast-gate (chặn merge) và cái gì để stage sau/nightly?
2. Đưa test vào legacy: hai bước đầu tiên của bạn?
3. Vì sao "coverage 90%" không đủ để TL tin chiến lược test đang hiệu quả?

<details><summary>Gợi ý chốt</summary>

(1) Fast-gate: lint + type-check + unit (rẻ, quyết định nhanh) chặn merge; integration/e2e nặng + mutation + full load để stage sau/nightly. (2) Characterization test quanh phần sắp đụng + gate diff-coverage cho PR mới; ưu tiên module rủi ro cao, tăng dần. (3) Coverage là proxy đầu vào (đo dòng chạy); hiệu lực thật đo bằng defect-escape-rate, change-failure-rate, MTTR — outcome mới phản ánh test có *thật sự* bắt lỗi & cho deploy an toàn.
</details>

---

## 🗺️ Bản đồ ôn tập 1 trang + lịch spaced repetition

### Câu thần chú
> *"Tôi không nhớ để gõ, tôi hiểu để chỉ huy — và tra cứu phần còn lại."*

### Chốt 11 mental model (đọc lướt mỗi lần ôn)
1. **Pyramid:** đẩy test xuống thấp nhất vẫn kiểm được; hình dạng = hệ quả của rủi ro.
2. **Double:** stub điều khiển input (state-based); mock kiểm tương tác (chỉ khi tương tác là contract).
3. **TDD:** đỏ → xanh tối thiểu → dọn; test từ yêu cầu, không từ code đã viết.
4. **Runner:** chọn theo stack (Vitest mới/ESM, Jest legacy/RN); song song phải cô lập.
5. **Integration:** mock kiểm logic, integration kiểm giao kèo hạ tầng; migration chạy như prod.
6. **Test data:** tối thiểu + cục bộ + xác định; cleanup chạy cả khi đỏ.
7. **Flaky:** thiếu determinism; xóa nguồn ngẫu nhiên, đừng dán băng bằng retry.
8. **Coverage/mutation:** coverage nói "đã chạy", mutation nói "đã kiểm".
9. **Load:** đo đuôi (p95/p99); open-model lộ nghẽn; smoke mỗi PR.
10. **Contract:** chốt hình dạng tương tác + can-i-deploy; không thay integration/e2e hoàn toàn.
11. **TL:** đặt chuẩn đúng chỗ + đo outcome (bug lọt prod), không tôn thờ coverage; nghi ngờ màu xanh của test AI.

### Lịch ôn ngắt quãng (đánh dấu khi hoàn thành)
| Mốc | Việc cần làm |
|---|---|
| **Hôm nay** | Đọc 11 mental model; tự trả lời "Tự kiểm tra" của Bài 1–3 |
| **Mai** | Mở `QB_L_testing.md`, tự trả lời L-PYR + L-DBL + L-TDD; chấm theo dòng "dò cái gì" |
| **+3 ngày** | L-RUN + L-INT + L-DAT + L-FLK; trộn 2 câu cũ từ Bài 1–3 |
| **+1 tuần** | L-COV + L-LOAD + L-CON + L-TL; làm 1 đợt "phỏng vấn" 12 câu trộn dễ→khó toàn mục |

### Trạng thái học tập (tự cập nhật)
```
TIẾN ĐỘ HỌC — Mục L (Testing & Quality)
- Phase: Quality/LLMOps (testing nền backend)
- Tài liệu giáo trình: ĐÃ DỰNG (11 bài, phủ 85/85 câu)
- Đã ĐẠT (tự chấm qua QB_L): [ ... điền ID khi luyện ... ]
- Đang học: [bài ...] — trạng thái: (đang đọc / chờ recall / cần vá: ...)
- Điểm yếu cần ôn lại: [ ... ]
- Bài tiếp theo đề xuất sau khi đạt hết L: Mục kế trong roadmap (vd M — Security)
```

> **Cách "chốt ĐẠT" cả mục L (WORKFLOW 2 — Bước E):** mở `QB_L_testing.md`, làm từng đợt 10–15 câu, tự trả lời TRƯỚC rồi đối chiếu dòng *"dò cái gì"*. Câu nào trả lời thiếu mốc đó → quay lại đúng Bài phủ nó, vá, hỏi lại biến thể. Đạt hết 85 câu = xong mục L.
