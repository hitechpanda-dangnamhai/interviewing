# 🔭 Mục O — Observability (Logging · Metrics · Tracing · SLO) · Giáo trình đầy đủ để học

> **Sinh theo WORKFLOW 2 — Bước B (dựng giáo trình ngược từ bộ đề) + Bước C (giảng từng bài theo Hợp đồng 10 mục).**
> Nguồn câu hỏi: `QB_O_observability.md` (**85 câu**, 8 mục con). Đây là **tài liệu học để lưu**, không phải phiên phỏng vấn — phỏng vấn live là Bước D, làm sau.
> **Bối cảnh kiến thức: tính tại 6/2026.** Mọi mục có *(verify)* là thứ đổi nhanh (đời OTel signal, đời Prometheus, tên/giá SaaS) → kiểm chứng tại docs chính thức khi thực hành.
> Quy ước nguồn: **📘** = có trong roadmap/QB gốc · **➕** = bổ sung của giảng viên · **⬆️** = làm sâu hơn.

---

## 🔎 Bối cảnh hiện hành đã verify (6/2026) — đọc trước

Những mốc này được dùng xuyên suốt tài liệu (đã search + verify tại nguồn chính thức opentelemetry.io / prometheus.io; vẫn nên tự kiểm chứng vì đổi nhanh). **Đây cũng là các câu "đính chính" so với mặc định cũ mà interviewer hay dò:**

- **OpenTelemetry signal maturity:** *Traces* **stable** (production-ready mọi ngôn ngữ lớn). *Metrics* **stable** ở các ngôn ngữ chính. *Logs* là signal nền cuối cùng ổn định — spec ổn nhưng **SDK maturity còn lệch theo ngôn ngữ** (đang chín dần qua 2026). *Profiling* = **signal thứ 4**, đã vào **public Alpha (3/2026)** dựa trên eBPF agent (Elastic donate), **target GA ~Q3 2026** — *chưa production-ready*. → **Đính chính QB:** QB ghi "RC ~đầu 2026"; thực tế tại 3/2026 mới là **Alpha** (chưa Beta/RC). *(verify: opentelemetry.io/status, /blog profiles-alpha)*
- **W3C Trace Context** (`traceparent` header) là chuẩn propagation; mỗi trace = **DAG của span**; span mang attributes/events/links. **Semantic conventions** (chuẩn tên field `http.*`, `db.*`, `messaging.*`) tiến hoá chậm & có chủ đích. *(verify: opentelemetry.io)*
- **OTel Collector** status = **mixed** (component khác nhau ở mức maturity khác nhau) → đọc README từng component trước khi tin "stable". *(verify)*
- **Prometheus 3.0** ra **11/2024** (major đầu tiên kể từ 2.0/2017). Tại **5/2026 đã ở ~v3.12** (LTS v3.5.x). Điểm 3.x: **ingest OTLP metric trực tiếp** (`/api/v1/otlp/v1/metrics`, **off mặc định** vì Prometheus không có auth → phải opt-in), **native histogram stable từ v3.8** (vẫn **opt-in khi scrape** qua `scrape_native_histograms: true`), **UTF-8 label name** (cho `http.server.request.duration` kiểu OTel), **Remote Write 2.0** (nén tốt hơn, mang native histogram + exemplar + metadata), UI mới, range selector đổi sang **left-open right-closed**. → **Đính chính QB:** nay nói được đời cụ thể *~v3.12 (5/2026)* + *native histogram stable v3.8*. *(verify: prometheus.io/releases, /docs/specs/native_histograms)*
- **Prometheus vẫn CHỈ là metric store** → trace/log đi backend khác (**Tempo**/**Loki**). Pattern phổ biến: route cả 3 signal qua **OTel Collector** rồi fan-out; OTLP receiver của Prometheus **đồng bộ** (ghi WAL xong mới trả) → ở volume cao dễ backpressure, nên đẩy qua Collector (có buffer/retry/batch). *(verify)*
- **Prometheus scale**: 1 node = không HA, lưu ngắn hạn → long-term + global view dùng **Thanos / Cortex / Mimir / VictoriaMetrics**. *(verify)*
- **Grafana "LGTM" stack**: **L**oki (logs) · **G**rafana (visualize) · **T**empo (traces) · **M**imir (metrics long-term) · **Alertmanager** (route alert). Đối thủ SaaS: **Datadog / New Relic / Honeycomb / Grafana Cloud / Dynatrace**. *(verify giá — tính theo host/volume, đắt rất nhanh ở scale)*

> 🧭 **Câu thần chú của mục O:** *AI/junior thêm log–metric–trace **đúng cú pháp** nhưng dễ **sai chỗ**: metric gắn `user_id` (cardinality bom), alert trên CPU thay vì trên triệu chứng user, đo `average latency` thay vì histogram p99, log rò token/PII, SLO cho mọi endpoint thay vì user journey. Bắt những chỗ đó — và biết **khi nào telemetry là chi phí thật phải cắt** — là việc của Tech Lead.*

> ⚠️ **Trọng tâm theo roadmap:** mục O chỉ ~25% tần suất; độ sâu kỳ vọng ở TL backend là *"đủ để bàn vận hành/độ tin cậy trong system design + ra quyết định"*. Ưu tiên **đạt vững**: O-FND, O-LOG, O-MET, O-ALERT, O-SLO. Còn O-TRACE/O-LOGSTACK đạt mức *"giải thích + trade-off"* (nhiều phần nền tracing đã ở **K-OBS**); các câu ⭐⭐⭐⭐⭐ là điểm cộng phân biệt với TL/SRE thật.

> 🔗 **Cross-ref (không lặp):** định nghĩa nền trace/span/propagation/head-tail sampling/three-pillars → **K-OBS**; lib log Node `pino`/`winston` + `AsyncLocalStorage` → **G**; PII/threat/compliance → **M**; error shape có trace id + class lỗi 4xx/5xx → **F**; percentile load-test + metric test-suite → **L**; health probe pod + agent node-level + cost cloud → **N**.

---

# 🧱 BƯỚC B — Giáo trình ngược dựng từ 85 câu hỏi

## Nguyên tắc sắp xếp: **dependency order**, không theo tần suất hỏi

Chuỗi phụ thuộc khái niệm: hiểu **observability là gì & ba pillar** (cùng *cardinality* — khái niệm xuyên suốt) trước → xuống **logging** (pillar dễ nhất, ai cũng có) gồm *thiết kế nội dung* rồi *vận hành/cost* → **stack gom log & error tracking** (đưa log ra khỏi máy) → **metrics & Prometheus** (số tổng hợp — nền của alert) chia 3 lớp *cơ bản → PromQL/cardinality → scale & Prom 3.0* → **alerting & dashboard** (đặt **sau** metric vì alert *dựa trên* metric) → **distributed tracing sâu** (đặt sau cùng trong nhóm pillar vì nền micro đã học ở **K-OBS**, ở đây đào *cơ chế + vận hành*) chia 2 lớp → **SLO/SLI/error budget** (tổng hợp mọi pillar thành *mục tiêu & ngân sách*) → cuối là **lớp quyết định TL** (build-vs-buy, cost, văn hoá on-call, privacy, review AI). Đặt SLO **sau** alert có chủ đích: burn-rate alerting *dựa trên* SLI + metric đã học; đặt TL cuối vì nó *ghép* mọi thứ thành quyết định.

## Mục lục bài học → ID câu hỏi nó phủ

| Bài | Tên bài | Mục con | ID câu phủ | Số câu |
|---|---|---|---|---|
| **1** | Nền tảng observability (vs monitoring, pillars, RED/USE, golden signals, cardinality, white/black-box, MELT) | O-FND | O-FND-001 → 008 | 8 |
| **2** | Structured logging I — thiết kế & nội dung (JSON, level, redaction, correlation id, stdout, sampling) | O-LOG | O-LOG-001 → 006 | 6 |
| **3** | Structured logging II — vận hành & chi phí (hot path, canonical line, clock, cost, enrichment, audit) | O-LOG | O-LOG-007 → 012 | 6 |
| **4** | Log aggregation & error tracking (ELK/EFK, Loki, Sentry, agent placement, retention, backpressure, OTel logs) | O-LOGSTACK | O-LOGSTACK-001 → 008 | 8 |
| **5** | Metrics & Prometheus I — metric types, histogram/percentile, pull vs push, exposition | O-MET | O-MET-001 → 004 | 4 |
| **6** | Metrics & Prometheus II — cardinality, PromQL (rate/quantile), recording rules | O-MET | O-MET-005 → 008 | 4 |
| **7** | Metrics & Prometheus III — scale, Prometheus 3.0, exemplars, RED dashboard, business vs system, watermelon | O-MET | O-MET-009 → 014 | 6 |
| **8** | Alerting & dashboards (symptom vs cause, fatigue, page vs ticket, `for:`, Alertmanager, runbook, synthetic, đo chất lượng alert) | O-ALERT | O-ALERT-001 → 010 | 10 |
| **9** | Distributed tracing sâu I — span model, baggage, semantic conventions, instrumentation, Collector | O-TRACE | O-TRACE-001 → 006 | 6 |
| **10** | Distributed tracing sâu II — tail/consistent sampling, broken trace, cost/PII, waterfall, critical path | O-TRACE | O-TRACE-007 → 012 | 6 |
| **11** | SLO / SLI / Error budget (định nghĩa, SLI tốt, budget↔velocity, burn-rate, percentile, target, MTTR, toil, postmortem) | O-SLO | O-SLO-001 → 011 | 11 |
| **12** | TL-level cross-cutting (build vs buy, OTel chuẩn, cost, ODD, on-call, DoD, incident mù, golden path, privacy, review AI) | O-TL | O-TL-001 → 010 | 10 |

**Tổng: 12 bài / 85 câu.** Mỗi bài dưới đây theo **Hợp đồng đầu ra 10 mục**.

---

# 🎓 BƯỚC C — Các bài giảng

---

## Bài 1 — Nền tảng observability
*Phủ: O-FND-001 → 008*

### ① Mục tiêu & vị trí trong mạch
Đây là **nền của cả mục O**. Học xong bạn nói được *observability khác monitoring ở đâu*, *ba pillar trả lời câu hỏi gì với chi phí nào*, biết hai lăng kính **RED/USE** + **four golden signals** để bắt đầu theo dõi bất kỳ service nào, và — quan trọng nhất cho phần sau — hiểu **cardinality** (khái niệm chạy xuyên log/metric/trace). Mọi bài sau (logging, metrics, tracing, SLO) đều giả định bạn đã nắm khung tư duy này.

### ② Giảng cơ bản → nâng cao

**(a) Trực giác.** *Monitoring* là **canh những thứ bạn đã biết để mà lo** (dashboard/alert cho câu hỏi định sẵn — *known unknowns*: "CPU có cao không?"). *Observability* là **khả năng đặt câu hỏi MỚI mà bạn chưa kịp lường** rồi trả lời được từ dữ liệu hệ phát ra (*unknown unknowns*: "vì sao đúng nhóm user ở region X dùng feature Y mới bị chậm?") — mà **không phải deploy thêm code**. Observability không phải "có nhiều dashboard hơn" — nó là *độ giàu của dữ liệu đủ để truy vấn linh hoạt* (O-FND-001).

**(b) Cơ chế.**
- **Three pillars & trade-off cost↔chi tiết (O-FND-002):** *Metrics* = số tổng hợp theo thời gian, **rẻ, low-cardinality**, phát hiện "**có** bất thường" nhưng mất chi tiết (không biết *request nào*). *Logs* = sự kiện rời rạc, **giàu context** nhưng **đắt/khó aggregate**. *Traces* = đường đi **1 request** xuyên service, định vị *chặng nào chậm* nhưng cần instrument + sampling. **Không cái nào thay cái nào** — chúng trả lời câu hỏi khác nhau. Thứ tự khoanh vùng điển hình: *metric thấy bất thường → trace định vị service/chặng → log soi chi tiết dòng đó* (chi tiết → **K-OBS-006**).
- **RED vs USE (O-FND-003):** **RED** = **R**ate, **E**rrors, **D**uration → cho **request-driven service** (API, worker xử việc). **USE** = **U**tilization, **S**aturation, **E**rrors → cho **tài nguyên** (CPU, memory, disk, queue, connection pool). Chọn lăng kính theo *đang theo dõi service hay resource*. (RED là cái bạn alert; USE là cái bạn dùng để *chẩn đoán* khi RED đỏ.)
- **Four golden signals của Google SRE (O-FND-004):** **latency, traffic, errors, saturation** — bốn cái này phủ *trải nghiệm user + sức khoẻ hệ* ở mức tối thiểu để bắt đầu. Lưu ý tinh tế: **latency phải tách thành công vs thất bại** — một request lỗi trả về *rất nhanh* (fail fast) sẽ kéo "latency trung bình" xuống và **giấu** sự cố; đo p99 của *successful* requests riêng.
- **White-box vs black-box (O-FND-006):** white-box = nhìn **nội tại** (metric/log/trace app tự phát ra) → biết "**vì sao**". black-box = thăm dò **từ ngoài như user** (synthetic probe, uptime check) → biết "**user có thấy hỏng không**". Black-box bắt cả khi white-box *chết* hoặc lỗi tầng DNS/TLS/LB mà metric nội bộ không thấy. **Dùng cả hai** (probe → Bài 8).

**(c) Mép giới hạn & sai lầm thường gặp.**
- **Cardinality — quý hay bom tuỳ chỗ (O-FND-005):** cardinality = số *tổ hợp giá trị* nhãn/field khác nhau. Trong **log/trace**, high-cardinality (gắn `user_id`, `request_id`, `order_id`) là **tài sản** — cho phép slice/dice điều tra unknown unknowns. Trong **metric** thì **mỗi tổ hợp label = 1 time series** → gắn field cao-cardinality làm label → **nổ số series** → nổ RAM/disk/chi phí TSDB, có thể **sập Prometheus**. Cùng một field, *log thì để vào, metric thì cấm tiệt* — đây là phân biệt sống còn (lặp lại ở Bài 6).
- **Giới hạn của "thêm log nữa" (O-FND-007):** cùng một lỗi prod, *chỉ có log* đôi khi vẫn không debug nổi vì log **rời rạc**, không cho thấy **quan hệ thời gian/nhân quả xuyên service** và **phân bố** (1 request p99 xấu lẫn trong nghìn request bình thường). Hệ phân tán/đồng thời cao → cần **trace** (đường đi) + **metric** (phân vị/xu hướng). Nhận ra "thêm log không cứu được" là dấu hiệu cần pillar khác.
- **MELT + signal thứ 4 (O-FND-008, ➕ verify):** một số nơi gọi **MELT** (Metrics/Events/Logs/Traces). Xu hướng lớn: **OpenTelemetry hợp nhất** instrumentation → tránh "mỗi vendor một SDK/agent". **Profiling** đang thành signal thứ 4 (CPU/memory ở mức hàm) — *public Alpha 3/2026, target GA ~Q3 2026, chưa production* (verify). Giá trị cốt lõi của "một chuẩn": telemetry **correlate** được (cùng `trace_id`/attribute) & **đổi backend không phải re-instrument**.

### ③ ⚠️ Kiến thức cũ / bị thay thế

| Cái cũ (trong docs / phổ biến) | Thay bằng (hiện nay) | Vì sao |
|---|---|---|
| "Observability = nhiều dashboard / = monitoring" | Observability = **truy vấn được unknown unknowns** từ telemetry giàu | Dashboard chỉ trả lời câu đã biết; điều tra cái chưa lường mới là observability |
| Mỗi vendor một agent/SDK riêng | **OpenTelemetry** (instrument 1 lần, đổi backend tuỳ ý) *(verify signal status)* | Tránh lock-in ở chỗ đắt nhất (code đã gắn) + telemetry correlate được |
| "Three pillars" như khẩu hiệu cố định | **MELT** + **profiling** (signal thứ 4, Alpha 3/2026 — *verify*) | Bức tranh đang mở rộng; hiểu *trả lời câu hỏi gì*, đừng học vẹt con số "3" |
| Đo "latency trung bình" + alert "có lỗi" | Tách **latency thành công vs thất bại**, golden signals | Lỗi-nhanh giấu sự cố; average giấu đuôi (Bài 5) |

### ④ Áp dụng thực tế + So sánh bigtech
Khi thiết kế observability cho 1 service mới, *bắt đầu tối thiểu* bằng **golden signals / RED** (Rate–Errors–Duration của các endpoint chính) + **vài USE** cho tài nguyên nghẽn (pool DB, queue), rồi mới thêm trace cho luồng phức tạp. Kiến trúc tối thiểu: *app phát 3 signal → OTel Collector → metric→Prometheus, log→Loki, trace→Tempo → Grafana visualize*.

Pattern phổ biến (*không khẳng định tuyệt đối — verify từng nơi*): **Google SRE** phổ biến hoá golden signals + error budget; **Honeycomb** đẩy mạnh tư tưởng *high-cardinality wide events* để điều tra unknown unknowns; phần lớn tổ chức cloud-native 2026 chuẩn hoá **OTel** làm lớp instrumentation rồi gắn backend tuỳ chọn (Grafana LGTM tự host, hoặc Datadog/Honeycomb SaaS). Triết lý chung: *instrument theo chuẩn, đo triệu chứng user trước, giữ dữ liệu đủ giàu để hỏi câu chưa lường*.

### ⑤ Code thực hành + cấu hình
Không phải bài code-nặng — minh hoạ **RED cho một Express endpoint** bằng `prom-client` để thấy "đo gì là tối thiểu" (chi tiết Prometheus ở Bài 5–7):

```javascript
// red-min.js — RED tối thiểu cho 1 service (Node + Express + prom-client)
// npm i express prom-client@15   // verify đời tại npm/prometheus client docs
const express = require('express');
const client = require('prom-client');

const app = express();
client.collectDefaultMetrics(); // USE cơ bản: CPU/mem/event-loop của process

// Duration = histogram (KHÔNG phải average) — giữ phân bố để tính p95/p99 (Bài 5)
const httpDuration = new client.Histogram({
  name: 'http_request_duration_seconds',
  help: 'Latency theo route/method/status',
  // label BOUNDED — route TEMPLATE (/users/:id) chứ KHÔNG phải /users/123 (cardinality!)
  labelNames: ['method', 'route', 'status_code'],
  buckets: [0.01, 0.05, 0.1, 0.3, 0.5, 1, 2, 5], // phủ vùng quan tâm (Bài 6)
});

app.use((req, res, next) => {
  const end = httpDuration.startTimer();
  res.on('finish', () => {
    // route TEMPLATE, không phải URL thật → tránh nổ series
    end({ method: req.method, route: req.route?.path ?? 'unknown', status_code: res.statusCode });
  });
  next();
});

app.get('/users/:id', (req, res) => res.json({ id: req.params.id }));
app.get('/metrics', async (_req, res) => {
  res.set('Content-Type', client.register.contentType);
  res.end(await client.register.metrics()); // Rate + Errors suy ra từ counter status_code; Duration từ histogram
});
app.listen(8080);
```

> ⚠️ **Bẫy cardinality ngay từ đây:** đừng đặt label `route: req.url` (URL thật có id) hay `user_id` — đó là **bom series** (Bài 6, O-MET-005). Route phải là **template**.

### ⑥ Keywords cần nhớ (glossary)
- 🧠 **Cần HIỂU (giải thích bằng lời):** observability vs monitoring (unknown unknowns) · trade-off cost↔chi tiết của 3 pillar · cardinality là tài sản (log/trace) vs là bom (label metric) · white-box vs black-box · vì sao "thêm log" không phải lúc nào cũng cứu.
- 📌 **Cần THUỘC:** RED = Rate/Errors/Duration (service) · USE = Utilization/Saturation/Errors (resource) · 4 golden signals = latency/traffic/errors/saturation · MELT + profiling (signal thứ 4, Alpha 2026).
- 🛠️ **Cần LÀM ĐƯỢC:** với 1 service mới, liệt kê RED + vài USE nên đo ngay · chỉ ra một metric đang gắn label cao-cardinality và sửa thành bounded.

### ⑦ Mạch tư duy cần nhớ (mental model)
**"Monitoring canh điều đã biết; observability hỏi được điều chưa lường."** Và: *"Metric thấy CÓ bất thường → trace định vị Ở ĐÂU → log soi VÌ SAO."* Cardinality: *"log/trace để vào thì quý, label metric để vào thì nổ."*

### ⑧ Câu hỏi phỏng vấn & thách đố
1. *(dễ)* Observability khác monitoring ở đâu? → known vs unknown unknowns; observability = truy vấn linh hoạt từ telemetry giàu, không phải "nhiều dashboard".
2. *(TB)* Ba pillar mỗi cái mạnh/yếu gì? → metric rẻ-tổng hợp; log giàu-đắt; trace định vị-cần sampling; trade-off cost↔chi tiết.
3. *(TB)* RED vs USE áp cho gì? → RED cho service request-driven, USE cho resource.
4. *(khó)* Vì sao "latency trung bình" cộng cả request lỗi là sai? → fail-fast kéo average xuống, giấu sự cố; tách success/fail + p99.
5. *(khó)* Cardinality cao lúc nào quý lúc nào nguy? → quý ở log/trace (slice điều tra), nguy ở label metric (nổ series).

**Thách đố / trick:**
- *"Dashboard toàn xanh, CPU thấp, uptime 100% — nhưng user kêu lỗi. Vì sao 'quan sát tốt' mà vẫn mù?"* → đo nhầm *resource* (dễ-đo) thay vì *triệu chứng user* (SLI gắn trải nghiệm); "watermelon metric" (Bài 7). White-box khoẻ ≠ user gọi được (cần black-box).
- *"Tôi log mọi thứ rồi mà vẫn không tìm ra nguyên nhân lỗi xuyên 4 service — thiếu gì?"* → thiếu **trace** (quan hệ nhân quả/thời gian xuyên service) + **metric phân vị**; log rời rạc không dựng được timeline.

### ⑨ Bài tập thực hành + tiêu chí tự chấm
1. **Lập "observability spec" cho 1 service giả định** (vd checkout API): liệt kê RED, 2–3 USE, 2 black-box probe, và 1 câu unknown-unknown bạn muốn trả lời được.
   - *Đạt khi:* RED đúng cho service + USE đúng cho resource (không lẫn); probe là góc nhìn-ngoài; nêu được 1 câu hỏi *chưa-lường* cần dữ liệu high-cardinality (vd "lỗi tập trung ở app version nào?").
2. **Soi 1 metric "xấu"**: cho `orders_total{user_id=..., url=...}` — chỉ ra vì sao sai và sửa.
   - *Đạt khi:* nhận ra `user_id`/`url` là cardinality bom; sửa label thành bounded (`status`, `route_template`, `payment_method`); giải thích "mỗi tổ hợp = 1 series".

### ⑩ Đọc thêm / nguồn chuẩn
Google SRE Book/Workbook (chương Monitoring & SLO, "four golden signals"); opentelemetry.io (concepts/signals — traces/metrics/logs/profiles, status page); Honeycomb "observability" guides (wide events/high-cardinality). Định nghĩa nền trace/span → đối chiếu **K-OBS**.

> 🧪 **"Hiểu để chỉ huy & kiểm tra":** AI dựng dashboard thường nhồi *cái gì đo được thì đo* (CPU, mem, uptime) — cú pháp đúng, panel đẹp, nhưng **không gắn triệu chứng user**. Hãy kiểm: *dashboard này trả lời được "user có đang đau không" trong 10 giây không, hay chỉ khoe resource?*

---

## Bài 2 — Structured logging I: thiết kế & nội dung
*Phủ: O-LOG-001 → 006*

### ① Mục tiêu & vị trí trong mạch
Logging là pillar **dễ bắt đầu nhất** (service nào cũng log). Bài này dạy *log đúng cách để máy đọc được ở scale*: JSON có field, level hợp lý, **không bao giờ log secret/PII**, gắn **correlation/trace id** vào mọi dòng, ghi ra **stdout** (12-factor), và **sample** khi cần. Bài 3 sẽ lo *vận hành & chi phí*; Bài 4 lo *gom log ra khỏi máy*. Nắm bài này rồi mọi log sau mới *điều tra được*.

### ② Giảng cơ bản → nâng cao

**(a) Trực giác.** Log production không phải để *người đọc bằng mắt* — mà để **máy parse/filter/aggregate/correlate** ở scale. Vì vậy log phải là **dữ liệu có cấu trúc** (key-value/JSON), không phải câu văn. `console.log('user ' + id + ' failed')` là chuỗi tự do → muốn tìm "mọi log của user=123 level=error" phải regex mong manh; còn `{ "level":"error", "user_id":123, "event":"login_failed" }` thì query một phát ra (O-LOG-001). *(wiring lib Node `pino`/`winston` → **G-318**; ở đây ta lo chiến lược.)*

**(b) Cơ chế.**
- **Log levels (O-LOG-002):** level phản ánh **mức cần hành động / độ ồn**. `debug` cho dev điều tra; `info` cho **mốc nghiệp vụ** đáng ghi (order created); `warn` cho bất thường *tự hồi* (retry thành công); `error` cho hỏng *cần điều tra*; `fatal` cho sập-không-tiếp-tục-được. "Log mọi thứ ở info" là **sai** → noise nuốt tín hiệu + tốn tiền. Level phải **đổi được lúc runtime** (bật debug cho 1 service khi điều tra mà không redeploy).
- **Correlation/trace id xuyên async (O-LOG-004):** mọi dòng log của *cùng một request* phải mang cùng `request_id`/`trace_id` để **grep ra cả luồng** và **nối log↔trace**. Bí quyết: gắn id vào **logger context per-request** (middleware tạo child logger / `AsyncLocalStorage`) để mọi log con **tự kèm** — *không truyền tay qua từng hàm* (phân biệt 2 id → **K-OBS-003**; `AsyncLocalStorage` → **G-281**).
- **stdout / 12-factor (O-LOG-005):** app coi log là **event stream ghi ra stdout/stderr**, để *platform* (runtime/agent/Collector) lo thu gom/route/rotate. App **không tự ghi file + rotate** — nhất là trong container: file trong container **mất khi pod chết**, khó tập trung, và rotate trong app là *nhầm concern*. Tách app (sinh log) ↔ infra (vận chuyển log).

**(c) Mép giới hạn & sai lầm thường gặp.**
- **TUYỆT ĐỐI không log gì (O-LOG-003, ➕ verify threat → M):** password, token/API key, `Authorization` header, số thẻ/CVV, PII (SSN, định danh), secret. **Redaction đặt ở tầng logger** (serializer/redact-path của pino, hoặc xử lý ở Collector — Bài 3/9) + review. Lý do nặng: **log thường ít kiểm soát quyền hơn DB** (nhiều người xem được, đẩy lên SaaS, giữ lâu) → lộ secret trong log = **sự cố bảo mật**, không phải lỗi nhỏ.
- **Log sampling/throttling (O-LOG-006):** ở hot path / lỗi lặp triệu lần, log đầy đủ → **nổ chi phí + che tín hiệu**. Giải: sample/dedupe/rate-limit. **Rủi ro**: sample có thể bỏ đúng dòng cần lúc điều tra → nguyên tắc **giữ trọn error, sample info/debug**; cân *cost ↔ đủ chứng cứ*.

### ③ ⚠️ Kiến thức cũ / bị thay thế

| Cái cũ (trong docs / phổ biến) | Thay bằng (hiện nay) | Vì sao |
|---|---|---|
| `console.log('user '+id+' failed')` | **JSON structured log** có field (`{level,event,user_id,...}`) | Máy parse/filter/aggregate ở scale; chuỗi tự do = regex mong manh |
| App tự ghi file + tự rotate | Ghi **stdout** (12-factor), agent/platform lo phần sau | File trong container mất khi pod chết; rotate là việc infra |
| Truyền `requestId` thủ công qua mọi hàm | **`AsyncLocalStorage`/child logger** gắn context per-request | Tự kèm vào mọi dòng; không quên, không rối tham số *(G-281)* |
| "Log càng nhiều càng an toàn" / info mọi thứ | **Level kỷ luật + giữ error, sample info** | Noise nuốt tín hiệu + nổ chi phí; log là tiền thật (Bài 3) |
| Để PII/token lọt vào log, lọc sau | **Redact ở nguồn** (logger serializer) + review | Lộ secret = sự cố bảo mật; log ít kiểm soát quyền hơn DB *(M)* |

### ④ Áp dụng thực tế + So sánh bigtech
Use case: mọi backend production log JSON ra stdout, agent (Fluent Bit/Collector) gom → Loki/Elasticsearch; mỗi dòng có `trace_id` để từ một log nhảy thẳng sang trace (Bài 9). Kiến trúc tối thiểu: *app(JSON→stdout) → DaemonSet agent → log store → Grafana/Kibana*.

Pattern phổ biến (*verify từng nơi*): hầu hết tổ chức lớn **cấm log free-text** trong guideline, có **thư viện logger chuẩn nội bộ** tự gắn `service`, `env`, `trace_id` và **redact field nhạy cảm mặc định** (golden path → Bài 12). Triết lý: *log có schema, có id, không có secret*.

### ⑤ Code thực hành + cấu hình
Structured logging + correlation id qua `AsyncLocalStorage` (Node + pino) — *minh hoạ chiến lược; cơ chế lib → G*:

```javascript
// logger.js — pino structured + redaction + per-request context
// npm i pino@9 express   // verify đời tại getpino.io
const pino = require('pino');
const { AsyncLocalStorage } = require('node:async_hooks');

const als = new AsyncLocalStorage();

const base = pino({
  level: process.env.LOG_LEVEL ?? 'info',       // đổi runtime qua env (O-LOG-002)
  // REDACT ở nguồn: không để token/PII lọt ra log (O-LOG-003)
  redact: {
    paths: ['req.headers.authorization', 'password', 'card.number', '*.token'],
    censor: '[REDACTED]',
  },
  // mixin: gắn trace_id/request_id vào MỌI dòng nếu có context (O-LOG-004)
  mixin() {
    const store = als.getStore();
    return store ? { request_id: store.requestId, trace_id: store.traceId } : {};
  },
});

// middleware: mở context per-request → mọi log bên trong tự kèm id
function requestContext(req, res, next) {
  const requestId = req.headers['x-request-id'] ?? crypto.randomUUID();
  const traceId = req.headers['traceparent']?.split('-')[1]; // W3C trace context (Bài 9)
  als.run({ requestId, traceId }, () => next());
}

module.exports = { logger: base, requestContext };
```

```javascript
// app.js
const express = require('express');
const { logger, requestContext } = require('./logger');
const app = express();
app.use(requestContext);

app.post('/login', (req, res) => {
  // KHÔNG log req.body thô (có password) — log event + field an toàn
  logger.info({ event: 'login_attempt', user_id: req.body.userId });
  // ... nếu lỗi:
  logger.error({ event: 'login_failed', user_id: req.body.userId, reason: 'bad_password' });
  res.sendStatus(401);
});
app.listen(8080);
// Chạy container: app chỉ ghi stdout; Dockerfile KHÔNG redirect ra file (12-factor, O-LOG-005)
```

> ⚠️ **Bảo mật:** đừng `logger.info(req.body)` / `logger.info(user)` thẳng — dễ kéo theo password/PII. Redact theo path + chỉ log field bạn chủ động chọn. `// verify cú pháp redact theo đời pino tại docs.`

### ⑥ Keywords cần nhớ (glossary)
- 🧠 **Cần HIỂU:** vì sao JSON thắng free-text · vì sao stdout > tự ghi file (container) · vì sao id phải gắn qua context chứ không truyền tay · vì sao lộ secret trong log là sự cố bảo mật · trade-off của log sampling.
- 📌 **Cần THUỘC:** thang level trace/debug/info/warn/error/fatal · danh sách cấm-log (password/token/PII/card/secret/Authorization) · "giữ error, sample info" · `request_id`+`trace_id` trong mỗi dòng.
- 🛠️ **Cần LÀM ĐƯỢC:** cấu hình logger JSON + redact + child-logger per-request · chỉ ra dòng log đang rò PII và sửa.

### ⑦ Mạch tư duy cần nhớ (mental model)
**"Log = dữ liệu có schema cho MÁY, ra stdout, có id, KHÔNG có secret."**

### ⑧ Câu hỏi phỏng vấn & thách đố
1. *(dễ)* Vì sao structured (JSON) log thắng ở production? → máy parse/filter/aggregate/correlate; query field thay vì regex.
2. *(TB)* Tiêu chí gán log level? Vì sao "info mọi thứ" sai? → level theo mức cần-hành-động/độ-ồn; info-mọi-thứ = noise + chi phí.
3. *(TB)* Làm sao mọi dòng log của 1 request có cùng id qua async? → context per-request (AsyncLocalStorage/child logger), không truyền tay.
4. *(khó)* Vì sao app nên ghi stdout thay vì file trong container? → file mất khi pod chết; platform lo gom/rotate; tách concern.
5. *(khó)* Cái gì tuyệt đối không log, redact đặt ở đâu? → secret/token/PII/card; redact ở logger serializer + review; log ít kiểm soát quyền hơn DB.

**Thách đố / trick:**
- *"Tôi log `logger.info(user)` cho tiện debug — sao security bảo đây là incident?"* → object `user` kéo theo `passwordHash`/`email`/`token` → rò PII/secret vào log (nơi nhiều người xem, giữ lâu, đẩy SaaS). Fix: chỉ log field chủ động chọn + redact path.
- *"Lỗi lặp 1 triệu lần/phút, log đầy đủ làm sập pipeline — tắt log error đi?"* → KHÔNG tắt error (mất chứng cứ); **dedupe/rate-limit + sample** lỗi lặp (giữ mẫu + count), sample info; đừng đánh đổi "mất tín hiệu điều tra".

### ⑨ Bài tập thực hành + tiêu chí tự chấm
1. **Refactor một logger free-text** thành JSON + redact + correlation id.
   - *Đạt khi:* output JSON có `level/event/trace_id`; password/token bị `[REDACTED]`; mọi log trong 1 request mang cùng id mà không truyền tham số qua từng hàm.
2. **Thiết kế bảng level cho 1 service**: liệt kê 6 sự kiện thật → gán level + giải thích.
   - *Đạt khi:* mốc nghiệp vụ = info, bất-thường-tự-hồi = warn, hỏng-cần-điều-tra = error; không có cái nào "info cho chắc".

### ⑩ Đọc thêm / nguồn chuẩn
getpino.io (redact, child logger, transports); 12factor.net (mục XI Logs); OWASP Logging Cheat Sheet (cấm log gì). Cơ chế Node logger + `AsyncLocalStorage` → **G**; threat/PII → **M**.

> 🧪 **"Hiểu để chỉ huy & kiểm tra":** AI hay sinh `logger.info(JSON.stringify(req.body))` hoặc `console.log(user)` — chạy ngon, nhưng **rò PII/secret** và **đồng bộ chặn event loop** (Bài 3). Hãy kiểm: *log này có field nhạy cảm không? có gắn trace_id không? có ghi stdout không?*

---

## Bài 3 — Structured logging II: vận hành & chi phí
*Phủ: O-LOG-007 → 012*

### ① Mục tiêu & vị trí trong mạch
Bài 2 dạy *log đúng nội dung*. Bài này đi xuống **vận hành thật**: logging trên hot path **chặn event loop** (đặc biệt Node), gom thông tin thành **canonical/wide event** giàu field, đồng bộ **clock (UTC/NTP)** để dựng timeline, **chi phí logging ở scale** đến từ đâu và cắt thế nào, **enrich** field nên đặt ở app hay agent/Collector, và **audit log** khác application log ra sao. Đây là cầu sang Bài 4 (stack gom log).

### ② Giảng cơ bản → nâng cao

**(a) Trực giác.** Log không miễn phí: nó tốn **CPU lúc ghi** và **tiền lúc lưu/index**. Hai mặt trận: *(1)* ghi log không được làm chậm app, *(2)* giữ log không được làm vỡ ngân sách. Cả hai đều cần *kỷ luật*, không phải "log vô tận".

**(b) Cơ chế.**
- **Logging đồng bộ trên hot path (O-LOG-007, ➕ Node → G-318):** I/O log **đồng bộ** trên đường nóng → **chặn event loop** (Node single-thread) → tăng latency / giảm throughput cho *mọi* request. Giảm: dùng logger **async/transport tách luồng** (pino ghi nhanh, xử lý/ship ở worker/transport), tránh `JSON.stringify` object lớn trong vòng nóng, **đo** tác động (đừng phỏng đoán). Bẫy điển hình: thêm log debug "cho chắc" trong loop xử lý triệu item.
- **Wide events / canonical log line (O-LOG-008):** thay vì rải 10 dòng log rời cho 1 request, **gom thành 1 "canonical log line"** giàu field cuối request: `{status, latency_ms, user_id, route, db_calls, cache_hit, trace_id, ...}`. Đó là **1 event high-cardinality** truy vấn được như *mini-analytics* ("p99 của route X cho user tier premium?") → giảm noise, tăng khả năng điều tra unknown unknowns (tư tưởng Honeycomb).
- **Clock UTC + ISO-8601 + NTP (O-LOG-009):** để **so trục thời gian** giữa nhiều service/host, mọi timestamp nên **UTC + ISO-8601** và host **đồng bộ NTP**. Lệch clock → log/trace **lệch nhân quả** (cái "sau" hiện ra "trước") → dựng timeline sai, debug sai.

**(c) Mép giới hạn & sai lầm thường gặp.**
- **Chi phí logging ở scale (O-LOG-010):** phần lớn tiền nằm ở **index + storage** (và egress nếu sang SaaS), theo **volume × cardinality**. Đòn bẩy: giảm verbosity, **sample**, **retention theo level** (error giữ lâu, debug ngắn), **index có chọn lọc** (Loki chỉ index *label*, không full-text — Bài 4), **tier nóng/lạnh**, **drop field thừa ở Collector**. Hiểu *log là chi phí thật* — không "log cho chắc rồi tính sau".
- **Enrichment: app hay agent? (O-LOG-011, ➕ verify):** field **nghiệp vụ** (user_id, order_id, event) do **app** thêm (app mới biết). Field **môi trường** (pod, node, namespace, region, `service.version`) nên do **agent/Collector** tự gắn (vd `k8sattributesprocessor` của OTel) → đồng nhất, app không phải biết hạ tầng. Quan trọng: **cùng tên field giữa 3 signal** (semantic conventions) để correlate log↔metric↔trace.
- **Audit log ≠ application log (O-LOG-012, ➕ compliance → M):** audit là **bằng chứng bảo mật/pháp lý** → yêu cầu khác hẳn: **immutable/append-only**, **retention dài** theo compliance, nội dung chuẩn "**ai, làm gì, khi nào, lên tài nguyên nào**", **tách kênh & quyền** khỏi log debug. Đừng trộn audit vào log vận hành (debug log bị xoá/sample, audit thì không được).

### ③ ⚠️ Kiến thức cũ / bị thay thế

| Cái cũ (trong docs / phổ biến) | Thay bằng (hiện nay) | Vì sao |
|---|---|---|
| Log đồng bộ thẳng trong hot path | Logger **async/transport tách luồng** (pino) | I/O đồng bộ chặn event loop → tăng latency toàn cục *(G-318)* |
| Rải nhiều dòng log rời cho 1 request | **Canonical/wide event** 1 dòng giàu field | 1 event high-cardinality query được như analytics; ít noise |
| Timestamp local time mỗi nơi một kiểu | **UTC + ISO-8601 + NTP** | So trục thời gian xuyên service; tránh lệch nhân quả |
| "Log vô tận, tính chi phí sau" | Sample + retention theo level + index chọn lọc + tier | Index+storage là phần đắt theo volume×cardinality |
| App tự gắn cả field môi trường | App gắn field nghiệp vụ, **agent/Collector** gắn field hạ tầng | Đồng nhất + app không phụ thuộc hạ tầng *(k8sattributesprocessor)* |
| Audit lẫn trong application log | **Audit tách riêng**: immutable, retention dài, RBAC | Audit là bằng chứng pháp lý, không được xoá/sample *(M)* |

### ④ Áp dụng thực tế + So sánh bigtech
Use case: service ghi 1 canonical line/request ra stdout; Collector gắn `k8s.pod.name`/`service.version`, **drop** field thừa, **redact** PII còn sót, route error→giữ-lâu / debug→giữ-ngắn. Audit (login, đổi quyền, truy cập dữ liệu nhạy cảm) đi **kênh + store riêng** (append-only, WORM/immutable bucket).

Pattern phổ biến (*verify*): các tổ chức scale lớn đặt **"cost per signal" thành metric được review**, có ngân sách log theo team, dùng **Loki/tiering** để cắt chi phí index, và **chuẩn hoá field** qua OTel semantic conventions để 3 signal correlate. Triết lý: *log như tài sản có ngân sách, không phải bãi rác ghi-cho-chắc*.

### ⑤ Code thực hành + cấu hình
Async logging tránh chặn event loop + canonical line + drop/enrich ở Collector:

```javascript
// pino async transport (ship ở luồng khác, không chặn hot path) — verify đời pino@9
const pino = require('pino');
const logger = pino({
  level: 'info',
  transport: { target: 'pino/file', options: { destination: 1 } }, // 1 = stdout, xử lý ở worker thread
});

// CANONICAL LOG LINE: 1 dòng giàu field cuối mỗi request (O-LOG-008)
app.use((req, res, next) => {
  const start = process.hrtime.bigint();
  res.on('finish', () => {
    const latency_ms = Number(process.hrtime.bigint() - start) / 1e6;
    logger.info({                       // MỘT event/request, high-cardinality, query được
      event: 'http_request',
      method: req.method, route: req.route?.path ?? 'unknown',
      status: res.statusCode, latency_ms,
      user_id: req.user?.id, trace_id: req.traceId,
      db_calls: req.metrics?.dbCalls, cache_hit: req.metrics?.cacheHit,
    });
  });
  next();
});
```

```yaml
# otel-collector.yaml — gắn field hạ tầng + drop thừa + redact (O-LOG-011) — verify cú pháp processor theo đời
processors:
  k8sattributes:                # agent/Collector gắn pod/node/namespace (app không cần biết)
    extract:
      metadata: [k8s.pod.name, k8s.namespace.name, k8s.node.name]
  attributes/redact:            # phòng tuyến cuối: bỏ field nhạy cảm còn sót (O-LOG-003)
    actions:
      - { key: authorization, action: delete }
      - { key: password, action: delete }
  attributes/drop:              # drop field thừa để giảm cost index (O-LOG-010)
    actions:
      - { key: verbose_debug_blob, action: delete }
# pipeline logs: receivers(otlp/filelog) -> processors(k8sattributes, redact, drop, batch) -> exporters(loki/...)
```

> ⚠️ **Đo trước khi tin:** thêm log vào hot path → **benchmark** latency/throughput (đừng phỏng đoán). Audit log đi store **immutable** riêng, không qua pipeline debug có sample/drop.

### ⑥ Keywords cần nhớ (glossary)
- 🧠 **Cần HIỂU:** vì sao log đồng bộ chặn event loop · canonical/wide event vs rải dòng rời · vì sao lệch clock phá timeline · chi phí log đến từ index+storage · app-field vs infra-field · vì sao audit phải tách.
- 📌 **Cần THUỘC:** UTC + ISO-8601 + NTP · retention theo level (error lâu, debug ngắn) · `k8sattributesprocessor` enrich · audit = immutable/append-only + retention dài + "ai-làm-gì-khi-nào".
- 🛠️ **Cần LÀM ĐƯỢC:** cấu hình async logger + canonical line · cấu hình Collector drop/redact/enrich · phân biệt log nào là audit và tách kênh.

### ⑦ Mạch tư duy cần nhớ (mental model)
**"Log tốn CPU lúc ghi (đừng chặn event loop) và tốn tiền lúc lưu (đừng index vô tận) — và audit là bằng chứng, không phải debug log."**

### ⑧ Câu hỏi phỏng vấn & thách đố
1. *(dễ)* Vì sao log đồng bộ hại trên Node? → chặn event loop → latency toàn cục; dùng async transport.
2. *(TB)* Canonical/wide event là gì, lợi gì? → 1 dòng giàu field/request → query như analytics, ít noise, điều tra unknown unknowns.
3. *(TB)* Vì sao UTC+NTP quan trọng với log/trace? → so trục thời gian xuyên service; tránh lệch nhân quả.
4. *(khó)* Chi phí logging ở scale đến từ đâu, cắt thế nào? → index+storage theo volume×cardinality; sample/retention-theo-level/index-chọn-lọc/tier/drop-ở-Collector.
5. *(khó)* Audit log khác application log ở yêu cầu nào? → immutable/append-only, retention dài, "ai-làm-gì", tách kênh/quyền.

**Thách đố / trick:**
- *"Hai service log cùng sự kiện nhưng timeline trên dashboard ngược nhau — bug ở đâu?"* → **lệch clock** (thiếu NTP) hoặc timezone khác nhau → fix UTC + ISO-8601 + đồng bộ NTP; không phải "bug logic".
- *"Bill logging tháng này gấp 3 mà traffic không đổi — vì sao?"* → thường do **cardinality/verbosity tăng** (ai đó thêm field high-cardinality hoặc bật debug) làm nổ index; soi field/level mới, cắt + tier, đặt cost per signal làm metric.

### ⑨ Bài tập thực hành + tiêu chí tự chấm
1. **Đổi rải-dòng → canonical line** cho 1 endpoint + benchmark log đồng bộ vs async.
   - *Đạt khi:* 1 event/request đủ field điều tra; chỉ ra (bằng số đo) async giảm tác động latency so với đồng bộ.
2. **Thiết kế retention/enrichment policy** (bảng level→retention + field nào app/agent gắn) + tách audit.
   - *Đạt khi:* error giữ lâu hơn debug; field hạ tầng do Collector; audit có store immutable riêng + lý do.

### ⑩ Đọc thêm / nguồn chuẩn
getpino.io (async transports, performance); opentelemetry.io Collector (`k8sattributesprocessor`, `attributesprocessor`, `filelogreceiver`); Honeycomb "wide events". Cost log stack → Bài 4; compliance/audit → **M**.

> 🧪 **"Hiểu để chỉ huy & kiểm tra":** AI hay thêm `logger.info` rải khắp + `JSON.stringify` object lớn trong loop — *chạy đúng* nhưng **chặn event loop + nổ chi phí**. Hãy kiểm: *log này nằm trên hot path không? object có lớn không? đây có nên là 1 canonical line không?*

---

## Bài 4 — Log aggregation & error tracking
*Phủ: O-LOGSTACK-001 → 008*

### ① Mục tiêu & vị trí trong mạch
Log ghi ra stdout (Bài 2–3) rồi **đi đâu**? Bài này dạy *đưa log ra khỏi máy & biến thành thứ tra cứu được*: **ELK/EFK** vs **Loki** (hai mô hình index, hai bài toán chi phí), **Sentry** (error tracking — bài toán mà log/metric thuần làm dở), ranh giới **error tracking vs logging vs APM**, **đặt agent ở đâu** trong K8s, **retention/tiering**, **backpressure** khi pipeline tải, và **OTel logs** (bridge + correlate). Đây là pillar "log" hoàn chỉnh trước khi sang metric.

### ② Giảng cơ bản → nâng cao

**(a) Trực giác.** Một app ghi log ra stdout là vô dụng nếu bạn phải SSH vào 50 pod để đọc. Cần **pipeline**: *agent thu gom → store index → UI query*. Hai trường phái store: **full-text index** (mạnh-truy-vấn, đắt) vs **index-chỉ-label** (rẻ, query yếu hơn). Và có một lớp riêng — **error tracking** — không thay log mà *gom exception thành "issue"*.

**(b) Cơ chế.**
- **ELK/EFK (O-LOGSTACK-001):** **E**lasticsearch (lưu + **full-text index** + search), **L**ogstash *hoặc* **F**luentd/**Fluent Bit** (thu gom/transform/ship), **K**ibana (query/visualize). "EFK" = thay Logstash bằng Fluentd/Fluent Bit nhẹ hơn. Mô hình: *ingest → index (mạnh, đắt) → query linh hoạt*.
- **Loki khác ở mô hình index (O-LOGSTACK-002):** Loki **chỉ index *label*** (metadata: `service`, `level`, `pod`), log body lưu **nén theo stream, KHÔNG full-text index** → **rẻ hơn nhiều**, hợp volume lớn; đổi lại query full-text yếu hơn ES (lọc theo time/label rồi *quét* nội dung). Hệ quả: **thiết kế label Loki phải cẩn thận cardinality** (label cao-cardinality làm nổ stream — đúng bài học Bài 1). Chọn theo *nhu cầu search vs chi phí*.
- **Sentry / error tracking (O-LOGSTACK-003):** gom **exception** theo **issue** (grouping/dedup theo *fingerprint* của stacktrace) + stacktrace + breadcrumb + **release** + **user-impact** → 1 triệu lần cùng lỗi gộp thành **1 issue** + alert + **regression theo release** ("lỗi này xuất hiện từ v2.3.1"). Trả lời "**lỗi nào, ảnh hưởng ai, từ release nào**" — điều log thô/metric làm dở.
- **OTel logs (O-LOGSTACK-008, ➕ verify):** OTel **không bịa API log mới** mà **bridge** logging lib hiện có, **gắn trace context** vào log → log tự **correlate với trace/metric** qua cùng attribute, đẩy qua Collector/OTLP đồng nhất. Giá trị: 3 signal cùng ngữ cảnh + enrich tập trung.

**(c) Mép giới hạn & sai lầm thường gặp.**
- **Error tracking vs logging vs APM (O-LOGSTACK-004):** ba lớp **bổ trợ**, không thay nhau: *error tracking* (Sentry) = exception + ngữ cảnh release/user; *logging* = dòng sự kiện chi tiết; *APM/metrics* = hiệu năng/latency/throughput. APM thấy "**chậm ở đâu**", error tracking thấy "**vỡ vì exception nào**", log soi **chi tiết**. Lẫn lộn → dùng sai công cụ.
- **Đặt agent ở đâu trong K8s (O-LOGSTACK-005, ➕ pod → N):** **DaemonSet** (Fluent Bit) = **1 agent/node** đọc stdout *mọi pod* → **rẻ, ít overhead, chuẩn mặc định**. **Sidecar/pod** chỉ khi cần xử lý riêng hoặc log *không ra stdout* → tốn tài nguyên mỗi pod. Mặc định **node-level DaemonSet**.
- **Retention & tiering (O-LOGSTACK-006):** retention **theo giá trị/level** (error giữ lâu, debug ngắn); **nóng** (index, query nhanh, ít ngày) → **lạnh/archive** (S3, rẻ, query chậm); xoá/aggregate theo policy; gắn compliance khi cần (audit → **M**).
- **Backpressure (O-LOGSTACK-007, ➕ tổng quát → F-371):** đầu ra chậm/quá tải → agent **buffer đầy** → *drop log* / *chặn app* (nếu app block khi ghi) / *OOM*. Giải: **buffer có giới hạn + chính sách drop** + tách app khỏi pipeline (app ghi stdout, agent lo phần sau) → **log không kéo sập app**. Nguyên tắc vàng: **chấp nhận mất log thay vì mất service**.

### ③ ⚠️ Kiến thức cũ / bị thay thế

| Cái cũ (trong docs / phổ biến) | Thay bằng (hiện nay) | Vì sao |
|---|---|---|
| Mặc định Elasticsearch full-text cho mọi log | Cân nhắc **Loki** (index-chỉ-label) cho volume lớn | Rẻ hơn nhiều khi không cần full-text mạnh; thiết kế label cẩn thận |
| Logstash nặng làm agent | **Fluent Bit/Fluentd** (EFK) — nhẹ, node-level | Ít overhead, hợp DaemonSet/container |
| Sidecar logger mỗi pod | **DaemonSet** đọc stdout mọi pod (mặc định) | Rẻ, ít overhead; sidecar chỉ khi cần xử lý riêng *(N)* |
| "Tìm lỗi bằng grep log thô" | **Error tracking (Sentry)**: gom issue + release + impact | Dedup exception, thấy regression theo release, ai bị ảnh hưởng |
| App block khi ghi log, pipeline kéo sập app | **Buffer giới hạn + drop policy**, tách app↔pipeline | Mất log < mất service *(backpressure, F-371)* |
| Thu log riêng, không liên kết trace | **OTel logs** bridge + gắn trace context | 3 signal correlate qua cùng attribute *(verify maturity)* |

### ④ Áp dụng thực tế + So sánh bigtech
Kiến trúc điển hình K8s: *app→stdout → **Fluent Bit DaemonSet**/OTel Collector node → (Loki hoặc ES) + (Sentry cho exception) → Grafana/Kibana*. Trace context trong log để nhảy log↔trace (Bài 9). Retention: error 30–90 ngày nóng → archive S3; debug 1–7 ngày.

Pattern phổ biến (*verify*): nhiều tổ chức **Grafana LGTM** tự host (Loki rẻ ở scale) + **Sentry** cho error tracking app; nơi khác mua **Datadog/Honeycomb** (đỡ vận hành, đắt theo volume — build-vs-buy ở Bài 12). Triết lý: *gom tập trung, đừng SSH đọc log; tách error tracking khỏi log thô; đừng để log kéo sập app*.

### ⑤ Code thực hành + cấu hình
Fluent Bit DaemonSet đọc stdout + Sentry init + bridge OTel logs (rút gọn):

```yaml
# fluent-bit (DaemonSet, node-level) — đọc stdout mọi pod, buffer giới hạn (O-LOGSTACK-005/007)
# verify cú pháp theo đời Fluent Bit
[INPUT]
    Name              tail
    Path              /var/log/containers/*.log    # stdout mọi pod trên node
    multiline.parser  docker, cri
[FILTER]
    Name              kubernetes                   # enrich pod/namespace (Bài 3)
    Merge_Log         On
[OUTPUT]
    Name              loki
    Host              loki-gateway
    # buffer giới hạn + drop khi đầy → KHÔNG chặn/đổ sập (backpressure)
    storage.total_limit_size  5G
    Retry_Limit       3
```

```javascript
// Sentry: gom exception thành issue + release tracking (O-LOGSTACK-003)
// npm i @sentry/node   // verify đời tại docs.sentry.io
const Sentry = require('@sentry/node');
Sentry.init({
  dsn: process.env.SENTRY_DSN,
  release: process.env.APP_VERSION,    // → regression theo release
  tracesSampleRate: 0.1,               // sample trace (cost — Bài 10)
  beforeSend(event) {                  // redact PII trước khi gửi SaaS (privacy — Bài 12)
    if (event.request?.headers) delete event.request.headers.authorization;
    return event;
  },
});
// exception tự gom theo fingerprint; 1 triệu lần cùng lỗi → 1 issue
```

> ⚠️ **Cardinality lại trở lại:** với Loki, **label** mới là thứ index — đặt `user_id`/`request_id` làm **label** = nổ stream (để chúng trong *log body*, không phải label). Với Sentry, **redact PII** ở `beforeSend` (đẩy lên SaaS).

### ⑥ Keywords cần nhớ (glossary)
- 🧠 **Cần HIỂU:** ES full-text vs Loki index-chỉ-label (trade-off cost↔query) · vì sao Sentry gộp exception thành issue · 3 lớp error-tracking/logging/APM bổ trợ · vì sao DaemonSet là mặc định · backpressure → mất-log < mất-service · OTel logs bridge+correlate.
- 📌 **Cần THUỘC:** EFK = Elasticsearch + Fluentd/Fluent Bit + Kibana · Loki index *label* không full-text · DaemonSet node-level · retention nóng→lạnh(S3) · buffer giới hạn + drop policy.
- 🛠️ **Cần LÀM ĐƯỢC:** cấu hình DaemonSet agent → store · init error tracking có release + redact · thiết kế retention/tiering + chọn ES vs Loki theo nhu cầu.

### ⑦ Mạch tư duy cần nhớ (mental model)
**"Agent gom (DaemonSet) → store (full-text-đắt ES / label-rẻ Loki) → UI; error tracking gộp exception thành issue; và pipeline log KHÔNG được kéo sập app."**

### ⑧ Câu hỏi phỏng vấn & thách đố
1. *(dễ)* ELK/EFK gồm gì, mỗi phần vai trò? → ES(lưu+index+search), Fluentd/Fluent Bit(thu/ship), Kibana(query); EFK = agent nhẹ hơn.
2. *(TB)* Loki khác ES ở index thế nào, hệ quả? → Loki index chỉ-label, không full-text → rẻ, query full-text yếu hơn; thiết kế label cẩn thận.
3. *(TB)* Sentry giải bài toán gì log thuần làm dở? → gom exception theo issue/fingerprint + release + user-impact + regression.
4. *(khó)* Agent log đặt ở đâu trong K8s, trade-off? → DaemonSet node-level (mặc định, rẻ) vs sidecar (khi cần xử lý riêng, tốn).
5. *(khó)* Pipeline log quá tải xử lý sao? → buffer giới hạn + drop policy, tách app↔pipeline; chấp nhận mất log thay vì mất service.

**Thách đố / trick:**
- *"Chuyển sang Loki để tiết kiệm, nhưng query chậm kinh khủng và chi phí vẫn cao — sai gì?"* → đặt **label cao-cardinality** (`user_id`, `path` thật) → nổ stream + query phải quét nhiều; sửa: label bounded (`service`, `level`, `env`), để id trong body.
- *"App thỉnh thoảng OOM/đơ đúng lúc log dồn nhiều — vì sao?"* → app **block khi ghi** / agent buffer không giới hạn (backpressure) → app gánh hậu quả pipeline; tách app(ghi stdout) khỏi agent + buffer giới hạn + drop.

### ⑨ Bài tập thực hành + tiêu chí tự chấm
1. **So sánh ES vs Loki cho 2 use case** (cần full-text điều tra ad-hoc vs chỉ lọc theo service/level ở volume lớn) → chọn + lý do + thiết kế label Loki.
   - *Đạt khi:* chọn đúng theo nhu cầu search↔cost; label Loki bounded, không nhét id.
2. **Thiết kế pipeline log K8s** (agent placement + retention + backpressure) trên giấy.
   - *Đạt khi:* DaemonSet node-level; retention theo level + tier S3; buffer giới hạn + drop policy nêu rõ "mất log < mất service".

### ⑩ Đọc thêm / nguồn chuẩn
grafana.com/oss/loki (label best practices, cardinality); elastic.co (ELK); fluentbit.io (Kubernetes DaemonSet); docs.sentry.io (issues/grouping/releases); opentelemetry.io (logs bridge, `filelogreceiver`). Pod/agent cơ chế → **N**; PII redact → **M**.

> 🧪 **"Hiểu để chỉ huy & kiểm tra":** AI dựng Loki/ES config thường đặt nhiều **label** "cho dễ lọc" — *chạy được* nhưng **nổ cardinality stream + đắt + chậm**. Hãy kiểm: *label này có bounded không? id có đang bị nhét vào label thay vì body không?*

---

## Bài 5 — Metrics & Prometheus I: metric types, histogram/percentile, pull/push, exposition
*Phủ: O-MET-001 → 004*

### ① Mục tiêu & vị trí trong mạch
Metric là pillar **rẻ nhất & nền của alert** (Bài 8) + SLI (Bài 11). Bài này dạy *bốn loại metric Prometheus* (gán sai loại = vô nghĩa), vì sao **histogram thắng "latency trung bình"** (liên hệ p95/p99), vì sao Prometheus chọn **pull** và khi nào cần **push**, và cơ chế **exposition + metric `up`**. Bài 6 lo cardinality/PromQL, Bài 7 lo scale/Prom 3.0.

### ② Giảng cơ bản → nâng cao

**(a) Trực giác.** Metric = **số tổng hợp theo thời gian** (time series): rẻ vì *aggregate sẵn* (không giữ từng request), nhưng vì thế **mất chi tiết** (không biết request nào). Prometheus chủ động **đi hỏi** (scrape) `/metrics` của bạn theo chu kỳ — không phải bạn đẩy lên.

**(b) Cơ chế.**
- **4 loại metric (O-MET-001):** **counter** = chỉ **tăng** (số request/lỗi) → luôn dùng với `rate()`, không đọc thô; **gauge** = **lên-xuống** (memory, queue depth, in-flight requests); **histogram** = **phân bố** vào *bucket* (latency, size) → quantile (p95/p99) tính **phía server** từ bucket; **summary** = quantile tính **ở client** (cố định, không aggregate xuyên instance được). Gán sai loại (đếm bằng gauge, đo latency bằng counter) = **metric vô nghĩa**.
- **Histogram vs "latency trung bình" (O-MET-002, ➕ p95 vs avg → L-363):** **trung bình giấu đuôi** — vài request 5s bị nuốt trong nghìn request 50ms → average vẫn "đẹp" trong khi *một nhóm user đang khổ*. **Histogram giữ phân bố** → tính **p95/p99** phản ánh trải nghiệm xấu nhất. Gauge "latency hiện tại" = 1 điểm thời điểm, mất phân bố. **Luôn đo latency bằng histogram** (đây là sai lầm AI/junior hay mắc — Bài 12).
- **Pull vs push (O-MET-003):** Prometheus **pull** (tự scrape `/metrics`) → biết target **sống/chết** (metric `up`), kiểm soát tần suất, dễ **service discovery**. **Push** chỉ hợp **job ngắn/batch** (chạy xong là chết, không kịp bị scrape) → **Pushgateway** (lưu ý: **không** dùng cho liveness, dễ *stale*). Hiểu **pull là mặc định, push là ngoại lệ**.
- **Exposition + `up` (O-MET-004):** app/exporter mở endpoint `/metrics` (text format `metric{labels} value`); Prometheus scrape theo *target* (static config / service discovery) mỗi interval; metric `up{job,instance}` = 1/0 cho biết **target có scrape được không** → nền cho alert "service down" (Bài 8).

**(c) Mép giới hạn & sai lầm thường gặp.**
- **Counter thô vô nghĩa:** đọc giá trị counter trực tiếp không nói gì (nó chỉ tăng, reset khi restart) → phải `rate()/increase()` (Bài 6).
- **Summary không aggregate được xuyên instance:** quantile tính sẵn ở client → không cộng p99 của 5 pod thành p99 cụm. Cần aggregate xuyên instance → **dùng histogram** (server tính từ bucket).
- **Pushgateway lạm dụng:** dùng push cho service thường-trực → mất tín hiệu `up`, data stale; chỉ batch/cron mới push.

### ③ ⚠️ Kiến thức cũ / bị thay thế

| Cái cũ (trong docs / phổ biến) | Thay bằng (hiện nay) | Vì sao |
|---|---|---|
| Đo latency bằng "average latency" (gauge) | **Histogram** + `histogram_quantile` p95/p99 | Average giấu đuôi; histogram aggregate xuyên instance |
| `summary` cho latency cần aggregate | **`histogram`** (quantile tính server từ bucket) | Summary tính quantile ở client → không cộng xuyên pod |
| Đọc giá trị counter thô | `rate()`/`increase()` trên counter | Counter chỉ tăng + reset; cần tốc độ trên cửa sổ (Bài 6) |
| Push mọi metric lên Pushgateway | **Pull (scrape)** mặc định; push chỉ job ngắn/batch | Pull cho `up`/liveness; push thường-trực → stale, mất sống-chết |

### ④ Áp dụng thực tế + So sánh bigtech
Use case chuẩn: service expose `/metrics` (counter `http_requests_total{status}`, histogram `http_request_duration_seconds`), Prometheus scrape qua K8s service discovery; metric `up` → alert down; histogram → SLI latency (Bài 11). Job batch (migration, nightly report) → push Pushgateway.

Pattern phổ biến (*verify*): Prometheus + `prom-client`/micrometer/exporter là **de-facto** cho metric cloud-native; tổ chức scale lớn route metric qua **OTel Collector → Prometheus/Mimir** (Bài 7). Triết lý: *pull-based, histogram cho latency, label bounded*.

### ⑤ Code thực hành + cấu hình
4 loại metric + scrape config:

```javascript
// metrics.js — 4 loại metric đúng cách (prom-client@15 — verify đời)
const client = require('prom-client');

const reqTotal = new client.Counter({                 // COUNTER: chỉ tăng
  name: 'http_requests_total', help: 'Tổng request',
  labelNames: ['method', 'route', 'status'],           // BOUNDED (Bài 6)
});
const inFlight = new client.Gauge({                    // GAUGE: lên-xuống
  name: 'http_in_flight_requests', help: 'Request đang xử lý',
});
const duration = new client.Histogram({               // HISTOGRAM: phân bố → p95/p99 (KHÔNG dùng average!)
  name: 'http_request_duration_seconds', help: 'Latency',
  labelNames: ['method', 'route', 'status'],
  buckets: [0.01, 0.05, 0.1, 0.3, 0.5, 1, 2, 5],       // phủ vùng quan tâm (Bài 6)
});
// đo: inFlight.inc()/dec(); reqTotal.inc({...}); const end=duration.startTimer(); end({...});
```

```yaml
# prometheus.yml — pull/scrape + metric `up` (O-MET-003/004) — verify cú pháp theo đời v3.x
scrape_configs:
  - job_name: 'api'
    metrics_path: /metrics
    kubernetes_sd_configs: [{ role: pod }]   # service discovery → target động
    relabel_configs:                          # chỉ scrape pod có annotation
      - source_labels: [__meta_kubernetes_pod_annotation_prometheus_io_scrape]
        action: keep
        regex: "true"
# metric `up{job="api",instance=...}` tự sinh = 1/0 → alert "service down" (Bài 8)
```

> ⚠️ **Bẫy loại metric:** đừng dùng gauge để "đo latency trung bình" (mất phân bố) hay summary nếu cần p99 xuyên pod (không cộng được). Latency → **histogram**.

### ⑥ Keywords cần nhớ (glossary)
- 🧠 **Cần HIỂU:** vì sao metric rẻ nhưng mất chi tiết · vì sao histogram > average (giấu đuôi) · vì sao summary không aggregate xuyên instance · vì sao Prometheus chọn pull · vai trò `up`.
- 📌 **Cần THUỘC:** counter(chỉ tăng→rate) / gauge(lên-xuống) / histogram(bucket→quantile server) / summary(quantile client) · pull mặc định, push=Pushgateway cho batch · `/metrics` text format.
- 🛠️ **Cần LÀM ĐƯỢC:** chọn đúng loại metric cho 1 đại lượng · expose `/metrics` + cấu hình scrape · đọc `up` để biết down.

### ⑦ Mạch tư duy cần nhớ (mental model)
**"Counter→rate, gauge→giá trị hiện tại, histogram→phân bố(p99). Latency LUÔN là histogram. Prometheus tự đến hỏi (pull); `up` cho biết bạn còn sống."**

### ⑧ Câu hỏi phỏng vấn & thách đố
1. *(dễ)* 4 loại metric Prometheus, mỗi cái dùng gì? → counter/gauge/histogram/summary như trên.
2. *(TB)* Vì sao đo latency bằng histogram thay vì average? → average giấu đuôi; histogram giữ phân bố → p95/p99.
3. *(TB)* Vì sao Prometheus chọn pull? Khi nào push? → pull biết `up`/kiểm soát; push cho job ngắn/batch (Pushgateway).
4. *(khó)* Histogram vs summary khác nhau ở đâu, chọn cái nào? → histogram tính quantile server (aggregate xuyên pod); summary tính client (không cộng được) → ưu tiên histogram.
5. *(khó)* Metric `up` đến từ đâu, dùng làm gì? → Prometheus sinh khi scrape (1/0) → alert service down.

**Thách đố / trick:**
- *"p99 latency của tôi = trung bình p99 của 5 pod — đúng không?"* → **SAI** với summary (quantile của quantile vô nghĩa); với histogram thì cộng *bucket* rồi tính quantile mới đúng. Đây là lý do dùng histogram.
- *"Service vẫn chạy nhưng dashboard mất hết metric — bug app?"* → kiểm `up==0`: thường là **scrape fail** (endpoint `/metrics` lỗi/đổi port/network), không phải app chết; `up` tách "app sống" khỏi "scrape được".

### ⑨ Bài tập thực hành + tiêu chí tự chấm
1. **Gán loại metric cho 6 đại lượng** (số order, RAM, latency request, queue depth, in-flight, lỗi/giây) + lý do.
   - *Đạt khi:* counter cho đếm tích luỹ, gauge cho lên-xuống, histogram cho latency; không dùng gauge cho latency.
2. **Expose `/metrics` + scrape** một service nhỏ, gây fail scrape (đổi port) và quan sát `up`.
   - *Đạt khi:* giải thích `up=0` là scrape fail (không phải app chết) và alert nên dựa trên `up` + `rate(errors)`.

### ⑩ Đọc thêm / nguồn chuẩn
prometheus.io/docs (metric types, exposition format, jobs & instances, `up`); github.com/siimon/prom-client. p95 trong load-test → **L-363**.

> 🧪 **"Hiểu để chỉ huy & kiểm tra":** AI rất hay sinh `gauge` tên `avg_latency` hoặc `summary` cho latency — *cú pháp đúng* nhưng **mất phân bố / không aggregate xuyên pod**. Hãy kiểm: *latency có đang là histogram không? p99 có cộng được xuyên instance không?*

---

## Bài 6 — Metrics & Prometheus II: cardinality, PromQL, recording rules
*Phủ: O-MET-005 → 008*

### ① Mục tiêu & vị trí trong mạch
Bài 5 dạy *đo gì bằng loại nào*. Bài này dạy *vận hành Prometheus đúng*: **cardinality explosion** (lỗi sản xuất kinh điển có thể sập Prometheus), **PromQL trên counter** (`rate`/`irate`/`increase`), vì sao `histogram_quantile` chỉ **xấp xỉ** và bucket sai gây gì, và **recording rules** để precompute. Đây là phần "tay nghề" để alert/SLI (Bài 8, 11) chạy đúng & nhanh.

### ② Giảng cơ bản → nâng cao

**(a) Trực giác.** Mỗi **tổ hợp label khác nhau = một time series riêng** Prometheus phải lưu trong RAM. Thêm một label "vô hại" như `user_id` → bạn vừa tạo *hàng triệu* series → nổ RAM. Và vì counter chỉ tăng, bạn **không bao giờ đọc giá trị thô** — luôn hỏi "*tốc độ* tăng" qua PromQL.

**(b) Cơ chế.**
- **Cardinality explosion (O-MET-005):** mỗi tổ hợp label = 1 series; nhét label **cao-cardinality** (`user_id`, `request_id`, `email`, URL chứa id, `session_id`) → **bùng nổ series** → nổ RAM/disk/scrape, **có thể sập Prometheus**. Label phải **bounded** (số giá trị hữu hạn & nhỏ): `route` *template* (`/users/:id`), `status_code`, `method`, `region`. Đây là **lỗi sản xuất kinh điển** — và đúng bài học cardinality của Bài 1.
- **`rate` vs `irate` vs `increase` (O-MET-006, ➕ verify PromQL):** counter thô vô nghĩa (chỉ tăng, reset khi restart). `rate(c[5m])` = **tốc độ trung bình/giây** trên cửa sổ (mượt → hợp **alert/SLI**); `irate(c[5m])` = **tức thời** từ 2 điểm cuối (nhạy → hợp **đồ thị** zoom); `increase(c[5m])` = **tổng tăng** trong cửa sổ. Cả ba **tự xử lý counter reset**. (Lưu ý 3.x: range selector **left-open right-closed** — verify khi tính chính xác cửa sổ.)
- **`histogram_quantile` (O-MET-007, ➕ verify native histogram):** p99 tính bằng **nội suy *trong* bucket** → kết quả là **xấp xỉ**; độ chính xác phụ thuộc **ranh giới bucket**. Nếu p99 rơi vào bucket quá rộng (vd kẹt ở `+Inf`) → sai lệch lớn. Phải đặt bucket **phủ vùng quan tâm** (quanh ngưỡng SLO). **Native histogram (Prom 3, stable v3.8 — verify)** dùng bucket *exponential tự động* → giảm hẳn vấn đề chọn-bucket-tay (Bài 7).
- **Recording rules (O-MET-008):** **precompute** biểu thức tốn kém (vd `rate` của nhiều series, quantile) thành **series mới** → dashboard/alert query **nhanh hơn** (tính sẵn theo chu kỳ, không tính lại mỗi lần xem). Khác **alerting rules** (đánh giá điều kiện → bắn alert).

**(c) Mép giới hạn & sai lầm thường gặp.**
- **Label "tạm thời": "thêm `user_id` để debug rồi xoá sau"** → quên xoá → nổ series âm thầm; bill tăng (Bài 3) hoặc Prometheus OOM. Cardinality là **không thể đảo ngược rẻ** (series cũ vẫn nằm đó).
- **Bucket đặt bừa:** `buckets: [1, 10, 100]` cho latency ms → mọi request <1s rơi vào 1 bucket → p99 vô nghĩa. Bucket phải quanh vùng thực tế + ngưỡng SLO.
- **`irate` cho alert:** quá nhạy → flapping; alert/SLI dùng `rate` (mượt), đồ thị dùng `irate`.

### ③ ⚠️ Kiến thức cũ / bị thay thế

| Cái cũ (trong docs / phổ biến) | Thay bằng (hiện nay) | Vì sao |
|---|---|---|
| Gắn `user_id`/`request_id`/URL-thật làm label | Label **bounded**: route template, status, method, region | Mỗi tổ hợp = 1 series → cardinality explosion → sập Prometheus |
| Đọc/alert trên giá trị counter thô | `rate()`/`increase()` (alert/SLI dùng `rate`) | Counter chỉ tăng + reset; cần tốc độ trên cửa sổ |
| Bucket histogram đặt tuỳ ý | Bucket **phủ vùng quan tâm** quanh ngưỡng SLO; cân nhắc **native histogram** (Prom 3, v3.8 stable — *verify*) | `histogram_quantile` xấp xỉ theo bucket; native auto-bucket giảm sai số |
| Dashboard query biểu thức nặng trực tiếp | **Recording rules** precompute | Query nhanh hơn, giảm tải lúc xem/alert |

### ④ Áp dụng thực tế + So sánh bigtech
Use case: recording rule precompute `job:http_request_duration_seconds:p99` cho dashboard SLO; alert dựa trên `rate(http_requests_total{status=~"5.."}[5m])`. Bucket latency đặt quanh ngưỡng SLO (vd p99<300ms → bucket dày quanh 0.3).

Pattern phổ biến (*verify*): tổ chức scale lớn có **cardinality budget** + cảnh báo khi số series tăng đột biến; chuyển dần sang **native histogram** để khỏi đau đầu chọn bucket. Triết lý: *label bounded, rate cho alert, precompute cái nặng*.

### ⑤ Code thực hành + cấu hình
Cardinality đúng/sai + PromQL + recording rule:

```promql
# ❌ SAI: label cao-cardinality → nổ series
http_requests_total{user_id="12345", url="/users/12345/orders/9988"}
# ✅ ĐÚNG: label bounded
http_requests_total{method="GET", route="/users/:id/orders/:oid", status="200"}

# Counter → tốc độ lỗi 5xx/giây (alert/SLI dùng rate, KHÔNG đọc thô)  (O-MET-006)
rate(http_requests_total{status=~"5.."}[5m])

# p99 latency từ histogram bucket — XẤP XỈ theo bucket (O-MET-007)
histogram_quantile(0.99, sum by (le) (rate(http_request_duration_seconds_bucket[5m])))
```

```yaml
# rules.yml — recording rule (precompute) + alerting rule  (O-MET-008) — verify cú pháp đời v3.x
groups:
  - name: http-slo
    interval: 30s
    rules:
      # RECORDING: precompute p99/route → dashboard & alert query nhanh
      - record: route:http_p99_seconds
        expr: histogram_quantile(0.99, sum by (le, route) (rate(http_request_duration_seconds_bucket[5m])))
      # ALERTING: dựa trên series đã precompute
      - alert: HighP99Latency
        expr: route:http_p99_seconds > 0.3
        for: 5m                 # tránh flapping (Bài 8)
        labels: { severity: page }
        annotations: { runbook: "https://runbooks/internal/high-latency" }
```

> ⚠️ **Bom cardinality im lặng:** trước khi thêm 1 label, hỏi "*giá trị này có bị chặn (bounded) không?*". `user_id`/`id`/`url`/`email` → KHÔNG; chúng thuộc **log/trace** (Bài 2, 9), không phải label metric.

### ⑥ Keywords cần nhớ (glossary)
- 🧠 **Cần HIỂU:** vì sao mỗi tổ hợp label = 1 series (cardinality bom) · vì sao counter thô vô nghĩa · vì sao `histogram_quantile` chỉ xấp xỉ · recording vs alerting rule.
- 📌 **Cần THUỘC:** label phải bounded · `rate`(mượt/alert) vs `irate`(nhạy/đồ thị) vs `increase`(tổng) · bucket phủ vùng SLO · native histogram (Prom 3, v3.8) *(verify)*.
- 🛠️ **Cần LÀM ĐƯỢC:** chỉ ra & sửa label cao-cardinality · viết PromQL p99 + error rate · viết recording rule precompute.

### ⑦ Mạch tư duy cần nhớ (mental model)
**"Label phải BOUNDED (id/url/email → cấm). Counter → rate. p99 từ histogram là xấp xỉ theo bucket. Cái nặng → precompute (recording rule)."**

### ⑧ Câu hỏi phỏng vấn & thách đố
1. *(dễ)* Cardinality explosion là gì, ví dụ label sai? → mỗi tổ hợp=1 series; `user_id`/`request_id`/url-thật → nổ series.
2. *(TB)* `rate` vs `irate` vs `increase`? → trung bình/giây mượt vs tức thời nhạy vs tổng cửa sổ; alert dùng `rate`.
3. *(TB)* Vì sao không đọc counter thô? → chỉ tăng + reset khi restart; cần tốc độ → `rate`.
4. *(khó)* Vì sao p99 từ histogram là xấp xỉ, bucket sai gây gì? → nội suy trong bucket; bucket rộng/kẹt `+Inf` → sai lớn; native histogram giảm.
5. *(khó)* Recording rule để làm gì? → precompute biểu thức nặng thành series → dashboard/alert nhanh.

**Thách đố / trick:**
- *"Prometheus OOM sau khi team thêm 1 label tưởng vô hại — vì sao?"* → label cao-cardinality (vd `customer_email`) → triệu series; cardinality **không đảo ngược rẻ**. Audit label, đặt cardinality budget.
- *"p99 báo đúng 1.0s suốt dù latency thật biến thiên — bug?"* → p99 **kẹt trong 1 bucket rộng** (ranh giới sai, hoặc dồn vào `+Inf`) → giá trị "đứng"; sửa bucket phủ vùng thật hoặc dùng native histogram.

### ⑨ Bài tập thực hành + tiêu chí tự chấm
1. **Audit & sửa cardinality**: cho 5 metric có label, chỉ ra cái nào nổ series và sửa thành bounded.
   - *Đạt khi:* nhận diện `id/url/email/session` là bom; thay bằng template/status/method; giải thích "không đảo ngược rẻ".
2. **Viết bộ rule SLO**: recording (p99/route) + alert (error rate + p99) cho 1 service.
   - *Đạt khi:* dùng `rate` cho error, `histogram_quantile` đúng cú pháp `sum by (le)`, alert có `for:` + runbook annotation.

### ⑩ Đọc thêm / nguồn chuẩn
prometheus.io/docs (querying — `rate`/`irate`/`increase`, `histogram_quantile`, recording rules; cardinality/label best practices); prometheus.io/docs/specs/native_histograms. Native histogram đời/feature → Bài 7 (verify).

> 🧪 **"Hiểu để chỉ huy & kiểm tra":** AI viết PromQL thường thêm label tiện tay hoặc đọc counter thô — *query chạy* nhưng **nổ series / số sai**. Hãy kiểm: *label có bounded không? counter có bọc `rate()` không? bucket có phủ ngưỡng SLO không?*

---

## Bài 7 — Metrics & Prometheus III: scale, Prometheus 3.0, exemplars, RED dashboard, business metric
*Phủ: O-MET-009 → 014*

### ① Mục tiêu & vị trí trong mạch
Bài 5–6 lo *đo & query đúng*. Bài này lo *bức tranh lớn*: giới hạn **Prometheus single-node** & landscape scale, **những gì Prometheus 3.0 thêm** (và vì sao vẫn "không nhận trace/log"), **exemplars** (cầu metric↔trace), **RED dashboard** đọc lúc incident, **business vs system metric**, và bẫy **watermelon metric**. Đây là chốt pillar metric trước khi sang alert.

### ② Giảng cơ bản → nâng cao

**(a) Trực giác.** Prometheus là **một con dao thu thập metric tuyệt vời, nhưng chỉ là một node** — không tự HA, không lưu dài, không nhận trace/log. Để "thấy toàn cảnh nhiều cluster, giữ 1 năm" cần lớp khác. Và dù dashboard có đẹp đến đâu, nếu đo **nhầm thứ** (resource thay vì user) thì "toàn xanh" vẫn lừa bạn.

**(b) Cơ chế.**
- **Single-node limits & landscape (O-MET-009, ➕ verify):** 1 node Prometheus = giới hạn dung lượng, **lưu ngắn hạn**, **không HA tự thân**. Cần long-term + **global view** xuyên cluster → **Thanos / Cortex / Mimir / VictoriaMetrics** (remote-write/federation); HA = chạy **2 replica scrape song song**. Prometheus lo *thu thập*; **scale lưu trữ là lớp riêng**.
- **Prometheus 3.0 (O-MET-010, ➕ verify):** ra **11/2024** (major đầu kể từ 2017); tại 5/2026 ~**v3.12**. Thêm: **ingest OTLP metric trực tiếp** (`/api/v1/otlp/v1/metrics`, **off mặc định** vì không có auth), **native histogram stable v3.8** (vẫn opt-in scrape), **UTF-8 label** (cho tên kiểu OTel `http.server.request.duration`), **Remote Write 2.0**, UI mới, range selector **left-open right-closed**. **NHƯNG** Prometheus **vẫn chỉ là metric store** → trace→Tempo, log→Loki; pattern: route cả 3 qua **OTel Collector**. Đừng coi Prometheus là "all-in-one". *(OTLP receiver đồng bộ → volume cao nên đẩy qua Collector có buffer.)*
- **Exemplars (O-MET-011):** exemplar = **trace id đính kèm 1 điểm dữ liệu metric** (vd 1 request rơi vào bucket latency cao) → từ đồ thị p99 **click thẳng sang trace** của request chậm cụ thể. Đây là **cầu nối "thấy bất thường (metric) → soi nguyên nhân (trace)"** — hiện thực hoá luồng khoanh vùng của Bài 1.
- **RED dashboard (O-MET-012):** panel **Rate** (req/s), **Errors** (error rate/%), **Duration** (p50/p95/p99). Đọc lúc incident: *errors↑ hay latency↑? theo route/version nào? rate có spike (traffic) không?* → khoanh nhanh **trước khi** xuống trace/log. Dashboard phục vụ **điều tra**, không trang trí.

**(c) Mép giới hạn & sai lầm thường gặp.**
- **Business vs system metric (O-MET-013):** *system* (CPU/latency/error) nói **hệ chạy ổn**; *business* (order/giây, signup, **payment success rate**) nói **giá trị có chảy không**. Đôi khi hệ "**xanh**" nhưng business **rớt** (nút mua hỏng, payment provider lỗi) → chỉ system metric không thấy. TL phải đẩy team đo **cả hai**.
- **Watermelon metric (O-MET-014):** "**xanh ngoài, đỏ trong**" — dashboard toàn xanh (CPU thấp, uptime cao) nhưng user thực đang khổ (p99 tệ ở luồng quan trọng, lỗi ở checkout). Nguyên nhân: đo nhầm thứ **dễ-đo** (resource) thay vì thứ **user-cảm-nhận** (SLI gắn trải nghiệm — Bài 11). Phải đo *latency thành công theo route quan trọng*, không chỉ resource.

### ③ ⚠️ Kiến thức cũ / bị thay thế

| Cái cũ (trong docs / phổ biến) | Thay bằng (hiện nay) | Vì sao |
|---|---|---|
| 1 Prometheus lo mọi thứ, mãi mãi | **Thanos/Cortex/Mimir/VictoriaMetrics** (long-term + global), HA = 2 replica | Single-node: không HA, lưu ngắn; scale là lớp riêng *(verify)* |
| "Prometheus là all-in-one observability" | Prometheus = **metric store**; trace→Tempo, log→Loki, route qua **OTel Collector** | Prometheus 3.0 nhận OTLP **metric** nhưng vẫn không nhận trace/log |
| Chọn bucket histogram bằng tay mãi | **Native histogram** (stable v3.8, opt-in) — auto exponential bucket | Giảm sai số `histogram_quantile` + giảm cardinality *_bucket *(verify)* |
| Dashboard nhồi CPU/uptime, "toàn xanh = ổn" | **RED + SLI gắn user**; tránh **watermelon** | Resource xanh ≠ user vui; đo triệu chứng user-facing |
| Chỉ đo system metric | Thêm **business metric** (payment success, order/s) | Hệ "xanh" vẫn có thể mất doanh thu (nút mua hỏng) |

### ④ Áp dụng thực tế + So sánh bigtech
Kiến trúc scale: *app → OTel Collector → Prometheus (scrape/OTLP) → remote-write **Mimir/Thanos** (long-term, global) → Grafana*. Exemplar bật để nhảy p99→trace (Tempo). Dashboard: 1 trang RED + 1 panel business (payment success rate) trên cùng.

Pattern phổ biến (*verify*): tổ chức nhiều cluster dùng **Mimir/Thanos/VictoriaMetrics** cho global view; bật **exemplars** để correlate; **business metric** (checkout success, signup) được coi trọng ngang system. Triết lý: *Prometheus thu thập, lớp khác lưu/global; đo cả business; tránh watermelon*.

### ⑤ Code thực hành + cấu hình
OTLP ingest (Prom 3) + exemplar + remote-write long-term (rút gọn):

```yaml
# prometheus.yml (v3.x) — OTLP ingest (opt-in, off mặc định) + remote-write long-term
# verify cú pháp/đời tại prometheus.io
otlp:
  # nhận OTLP metric trực tiếp tại /api/v1/otlp/v1/metrics — chỉ bật khi có lớp bảo vệ trước (không auth!)
  promote_resource_attributes: [service.instance.id, service.name]
storage:
  tsdb: { out_of_order_time_window: 30m }   # cho OTLP/native histogram
remote_write:
  - url: http://mimir-gateway/api/v1/push   # long-term + global view (Mimir/Thanos/VM)
# native histogram: bật opt-in khi scrape
# scrape_configs[].scrape_native_histograms: true   (verify tên flag theo đời)
```

```javascript
// business metric + (gợi ý) exemplar — đo cả "giá trị có chảy không" (O-MET-013)
const paymentResult = new client.Counter({
  name: 'payments_total', help: 'Kết quả thanh toán',
  labelNames: ['status', 'provider'],   // status: success|failed|timeout — BOUNDED
});
// dashboard business: rate(payments_total{status="success"}) / rate(payments_total) → payment success rate
// exemplar: nhiều client lib cho phép đính trace_id vào observe() để p99→trace (verify hỗ trợ theo lib)
```

> ⚠️ **OTLP off mặc định có lý do:** Prometheus **không có auth** → bật OTLP receiver phải có proxy/network policy chắn trước, nếu không là lỗ hổng. Volume cao → đẩy qua **OTel Collector** (buffer/retry) thay vì SDK→Prometheus trực tiếp.

### ⑥ Keywords cần nhớ (glossary)
- 🧠 **Cần HIỂU:** vì sao single-node Prometheus không đủ ở scale · vì sao Prometheus vẫn không nhận trace/log · exemplar nối metric↔trace thế nào · đọc RED lúc incident · vì sao cần business metric · watermelon.
- 📌 **Cần THUỘC:** Thanos/Cortex/Mimir/VictoriaMetrics (long-term/global); HA=2 replica · Prom 3.0: OTLP ingest(off mặc định)/native histogram(v3.8)/UTF-8/RW2.0 *(verify ~v3.12)* · RED = Rate/Errors/Duration(p50/95/99).
- 🛠️ **Cần LÀM ĐƯỢC:** dựng RED dashboard + 1 business panel · cấu hình remote-write long-term · giải thích "toàn xanh nhưng user khổ" và sửa SLI.

### ⑦ Mạch tư duy cần nhớ (mental model)
**"Prometheus = thu thập metric (1 node); lưu-dài/global là lớp khác; trace/log đi backend khác. Đo cả BUSINESS, đừng để dashboard 'dưa hấu' ru ngủ."**

### ⑧ Câu hỏi phỏng vấn & thách đố
1. *(dễ)* Prometheus single-node giới hạn gì, giải pháp? → không HA/lưu ngắn → Thanos/Cortex/Mimir/VM; HA=2 replica.
2. *(TB)* Prometheus 3.0 thêm gì, vì sao vẫn không "all-in-one"? → OTLP metric/native histogram/UTF-8/RW2.0; vẫn metric-only, trace/log đi Tempo/Loki.
3. *(TB)* Exemplar là gì? → trace id đính metric point → p99→trace.
4. *(khó)* Watermelon metric là gì, vì sao nguy? → xanh-ngoài-đỏ-trong; đo resource dễ-đo thay vì SLI user → ru ngủ.
5. *(khó)* Business vs system metric, ví dụ hệ "xanh" mà hỏng? → system ổn nhưng payment success rớt (nút mua/provider lỗi).

**Thách đố / trick:**
- *"Mọi metric xanh, on-call ngủ ngon, nhưng doanh thu sáng nay = 0 — vì sao 'observability tốt' mà mù?"* → thiếu **business metric** + **watermelon**: đo system không đo *payment success/checkout*; provider thanh toán lỗi mà system vẫn xanh. Thêm SLI business.
- *"Bật OTLP receiver trên Prometheus rồi internet quét được — sao?"* → Prometheus **không có auth**; OTLP off mặc định chính vì vậy; phải chắn proxy/network policy trước khi bật.

### ⑨ Bài tập thực hành + tiêu chí tự chấm
1. **Thiết kế RED + business dashboard** cho checkout (panel + PromQL).
   - *Đạt khi:* có Rate/Errors/Duration(p50/95/99) + **payment success rate**; giải thích đọc gì trước lúc incident.
2. **Phác kiến trúc scale** (1 cluster → nhiều cluster, long-term 1 năm).
   - *Đạt khi:* HA 2 replica + remote-write Mimir/Thanos/VM cho global/long-term; nêu Prometheus vẫn metric-only.

### ⑩ Đọc thêm / nguồn chuẩn
prometheus.io (3.0 release notes, OTLP receiver, native histograms, federation/remote-write); thanos.io / grafana.com/oss/mimir / victoriametrics.com; grafana.com (exemplars). Đời ~v3.12 (5/2026) — *verify tại prometheus.io/releases*.

> 🧪 **"Hiểu để chỉ huy & kiểm tra":** AI dựng dashboard thường nhồi CPU/mem/uptime ("toàn xanh") — *đẹp* nhưng **watermelon**, không đo business/SLI user. Hãy kiểm: *dashboard này thấy được doanh thu/checkout đang rớt không, hay chỉ resource?*

---

## Bài 8 — Alerting & dashboards
*Phủ: O-ALERT-001 → 010*

### ① Mục tiêu & vị trí trong mạch
Metric (Bài 5–7) sinh ra để **alert đúng** và **dashboard hữu dụng**. Bài này dạy triết lý alert (**symptom > cause**), chống **alert fatigue**, phân tầng **page vs ticket**, vì sao `CPU>80%` là alert tồi, vai trò `for:`, **Alertmanager** (grouping/dedup/silence/inhibition/routing), dashboard cho on-call, **runbook**, **synthetic/black-box** alert, và **đo chất lượng hệ thống alert**. Đây là phần dùng-hàng-ngày của vận hành, và là cầu sang SLO/burn-rate (Bài 11).

### ② Giảng cơ bản → nâng cao

**(a) Trực giác.** Một alert tốt = "**có người cần làm gì đó NGAY không?**". Nếu câu trả lời là "không" thì **đừng page**. Mục tiêu không phải *nhiều alert* — mà là **mỗi page đều đáng dậy lúc 3h sáng**. Alert nên bắn theo **cái user cảm nhận** (symptom), không theo *nguyên nhân nội tại đoán trước* (cause).

**(b) Cơ chế.**
- **Symptom vs cause (O-ALERT-001):** ưu tiên **symptom-based** (user-facing: error rate, latency, vi phạm SLO) → **ít alert hơn**, đúng cái quan trọng, **bắt cả lối hỏng MỚI** chưa lường. Cause-based (CPU/mem/disk/connection cụ thể) dễ **nổ ồn & bỏ sót**. Quy tắc: **cause để chẩn đoán, symptom để page**.
- **Page vs ticket (O-ALERT-003, ➕ 4xx/5xx → F-160):** **page** (gọi dậy) = đang/sắp **vi phạm SLO**, cần xử lý ngay; **ticket/warning** = cần làm nhưng **không khẩn** (disk 70%, cảnh báo xu hướng). Tiêu chí duy nhất: "*cần con người hành động NGAY không?*". (Business 4xx của user-lỗi **không** page; bug 5xx có → **F-160**.)
- **`for:` duration (O-ALERT-005):** yêu cầu điều kiện đúng **liên tục N phút** mới bắn → **lọc spike thoáng qua** (giảm flapping/noise). Dài quá → phản ứng **chậm** với sự cố thật. Cân nhạy↔ồn theo mức nghiêm trọng.
- **Alertmanager (O-ALERT-006):** ngoài "bắn alert" còn: **grouping** (gom alert liên quan thành 1 thông báo), **dedup** (khử trùng lặp từ nhiều replica), **silence** (tắt tạm khi bảo trì), **inhibition** (alert cấp cao **chặn** alert hệ quả: *cluster down → im* mọi alert con của service trong đó), **routing** (theo team/severity → PagerDuty/Slack/email). Tránh **bão thông báo** trong incident.

**(c) Mép giới hạn & sai lầm thường gặp.**
- **Alert fatigue (O-ALERT-002, ➕ M-514):** quá nhiều alert / false positive / không-actionable → kỹ sư **phớt lờ** → **bỏ lỡ alert thật** (đây mới là chi phí thật của fatigue). Chữa: chỉ page khi **actionable + user-impacting**, gộp/dedup, ngưỡng theo **SLO/burn-rate** (Bài 11), mỗi alert có **runbook**, **định kỳ review & xoá** alert vô dụng.
- **`CPU>80%` là alert tồi (O-ALERT-004):** CPU cao **≠ user đau** (có thể đang chạy hiệu quả/đúng tải); là **cause** không **symptom**, dễ false positive/negative. Thay bằng alert trên **triệu chứng thật** (latency/error/saturation ảnh hưởng user); CPU làm tín hiệu **chẩn đoán/capacity**, không paging.
- **Dashboard "đẹp để demo" ≠ "tốt cho on-call" (O-ALERT-007):** on-call cần **ít panel, top-down** (SLO/golden signals trước → drill xuống), trả lời nhanh "**có hỏng không, ở đâu**". Demo dashboard nhiều panel màu mè → **nhiễu** lúc incident. Thiết kế theo **câu hỏi điều tra**, không "đo được gì nhét nấy".
- **Runbook (O-ALERT-008):** alert nổ 3h sáng → người trực cần ngay "**nghĩa là gì, kiểm tra gì, làm gì, escalate ai**". Thiếu runbook → MTTR cao + phụ thuộc 1 người. Tối thiểu: *triệu chứng, dashboard/trace liên quan, bước chẩn đoán, cách giảm thiểu*.
- **Synthetic/black-box alert (O-ALERT-009, ➕ pod probe → N):** probe định kỳ endpoint quan trọng **như user** → bắt cả khi hệ metric **chết** hoặc lỗi tầng **DNS/TLS/LB** mà metric nội bộ không thấy. "*Service nói nó khoẻ*" ≠ "*user gọi được*".
- **Đo chất lượng alert (O-ALERT-010):** theo dõi alert nào **dẫn tới hành động thật** vs bị ignore/ack-rồi-bỏ, **tỉ lệ false positive**, **MTTA** (thời gian ack), **số page/đêm**. Coi bộ alert là **sản phẩm cần tinh chỉnh** — cắt cái nhiễu, đừng để "đống alert mọc dại".

### ③ ⚠️ Kiến thức cũ / bị thay thế

| Cái cũ (trong docs / phổ biến) | Thay bằng (hiện nay) | Vì sao |
|---|---|---|
| Alert trên cause (CPU/mem/disk cụ thể) | **Symptom-based** (latency/error/SLO user-facing) | Ít alert, đúng cái quan trọng, bắt lối hỏng mới |
| `CPU > 80%` page | Latency/error/saturation **ảnh hưởng user**; CPU = chẩn đoán | CPU cao ≠ user đau; cause không page |
| Alert ngưỡng tĩnh, page mọi thứ | **Page chỉ khi actionable + user-impacting**; còn lại ticket | Chống alert fatigue → không bỏ lỡ alert thật |
| Bắn ngay khi vượt ngưỡng 1 lần | `for:` (đúng liên tục N phút) | Lọc spike, giảm flapping |
| Dashboard nhiều panel "đẹp demo" | On-call dashboard **ít panel, top-down** | Nhiễu lúc incident; cần khoanh nhanh |
| Alert không runbook | Mỗi **paging alert có runbook** | Giảm MTTR, bớt phụ thuộc 1 người |
| Chỉ alert nội bộ (white-box) | Thêm **synthetic/black-box** probe | Bắt khi metric chết / lỗi DNS/TLS/LB |

### ④ Áp dụng thực tế + So sánh bigtech
Setup điển hình: alert rule trên SLI (error rate / p99 / burn-rate — Bài 11) với `for:` → Alertmanager (group theo service, inhibition cluster-down, route severity=page→PagerDuty, severity=ticket→Jira/Slack) → mỗi alert kèm runbook URL. Synthetic check (Blackbox exporter / synthetic monitor) cho endpoint critical.

Pattern phổ biến (*verify*): Google SRE nhấn **symptom-based + page chỉ khi cần người hành động ngay**; nhiều tổ chức có **alert review định kỳ** (xoá alert noise), **MTTA/% actionable** là metric của đội platform. Triết lý: *page = "dậy đi, có việc"; mọi thứ khác là ticket; mỗi page có runbook*.

### ⑤ Code thực hành + cấu hình
Alert rule symptom-based + Alertmanager routing/inhibition:

```yaml
# alert-rules.yml — symptom-based + for: + severity + runbook  (O-ALERT-001/003/005/008)
groups:
  - name: slo-symptom
    rules:
      - alert: HighErrorRate                       # SYMPTOM (user-facing), KHÔNG phải CPU
        expr: |
          sum(rate(http_requests_total{status=~"5.."}[5m]))
            / sum(rate(http_requests_total[5m])) > 0.02
        for: 5m                                    # lọc spike (O-ALERT-005)
        labels: { severity: page }                 # page = cần hành động NGAY
        annotations:
          summary: "Error rate > 2% (user-facing)"
          runbook_url: "https://runbooks/internal/high-error-rate"   # O-ALERT-008
      - alert: DiskFillingUp
        expr: predict_linear(node_filesystem_avail_bytes[6h], 24*3600) < 0
        for: 30m
        labels: { severity: ticket }               # ticket = không khẩn (O-ALERT-003)
```

```yaml
# alertmanager.yml — grouping/dedup/silence/inhibition/routing  (O-ALERT-006)
route:
  group_by: ['service', 'alertname']               # gom alert liên quan
  receiver: slack-tickets
  routes:
    - matchers: ['severity="page"']
      receiver: pagerduty                            # route theo severity
inhibit_rules:                                       # cluster down → im alert con
  - source_matchers: ['alertname="ClusterDown"']
    target_matchers: ['severity="page"']
    equal: ['cluster']
receivers:
  - name: pagerduty
    pagerduty_configs: [{ routing_key: "${PD_KEY}" }]
  - name: slack-tickets
    slack_configs: [{ api_url: "${SLACK_WEBHOOK}", channel: "#alerts" }]
```

> ⚠️ **Đừng page CPU:** đặt `CPU>80%` thành **ticket/diagnostic**, không page. Page chỉ symptom user-facing + `for:` + runbook. `// verify cú pháp Alertmanager theo đời.`

### ⑥ Keywords cần nhớ (glossary)
- 🧠 **Cần HIỂU:** symptom vs cause (page vs chẩn đoán) · chi phí thật của alert fatigue (bỏ lỡ alert thật) · vì sao `CPU>80%` tồi · `for:` lọc spike · inhibition · dashboard on-call vs demo.
- 📌 **Cần THUỘC:** page = cần-hành-động-NGAY; còn lại ticket · Alertmanager: grouping/dedup/silence/inhibition/routing · mỗi paging alert có runbook · synthetic/black-box bổ sung · đo: %actionable, false-positive, MTTA, page/đêm.
- 🛠️ **Cần LÀM ĐƯỢC:** viết alert symptom-based + `for:` + runbook · cấu hình Alertmanager route/inhibit · review & cắt alert noise.

### ⑦ Mạch tư duy cần nhớ (mental model)
**"Page khi 'có người cần làm NGAY', dựa trên TRIỆU CHỨNG user (không phải CPU), có `for:` + runbook. Cause để chẩn đoán. Alert là sản phẩm phải tỉa, không để mọc dại."**

### ⑧ Câu hỏi phỏng vấn & thách đố
1. *(dễ)* Alert nên dựa symptom hay cause, vì sao? → symptom (user-facing): ít alert, đúng cái quan trọng, bắt lỗi mới.
2. *(TB)* Page vs ticket phân theo gì? → "cần hành động NGAY không?"; page=khẩn/vi-phạm-SLO, ticket=không khẩn.
3. *(TB)* `for:` để làm gì, trade-off? → đúng liên tục N phút → lọc spike; dài quá → phản ứng chậm.
4. *(khó)* Alertmanager lo gì ngoài bắn alert? → grouping/dedup/silence/inhibition/routing → tránh bão thông báo.
5. *(khó)* Vì sao `CPU>80%` thường tồi, thay sao? → cause không symptom; CPU cao ≠ user đau; alert trên latency/error/saturation.

**Thách đố / trick:**
- *"Đội tôi tắt thông báo Slack vì quá nhiều alert — giờ bỏ lỡ sự cố thật. Sai từ đâu?"* → **alert fatigue**: page cả cái không-actionable → người nhờn → bỏ lỡ alert thật. Chữa: chỉ page symptom+actionable, gộp/dedup, review & xoá noise, đo %actionable.
- *"Metric nội bộ xanh hết nhưng user không vào được web — alert không bắn?"* → hệ metric/scrape có thể đã chết, hoặc lỗi **DNS/TLS/LB** ngoài tầm white-box; thiếu **synthetic/black-box** probe gọi như user.

### ⑨ Bài tập thực hành + tiêu chí tự chấm
1. **Chuyển 3 alert cause→symptom** (vd `CPU>80%`, `mem>90%`, `connections>N`) + thêm `for:` + runbook stub.
   - *Đạt khi:* alert mới trên latency/error/saturation user-facing; cause giữ làm ticket/diagnostic; có `for:` + runbook.
2. **Cấu hình Alertmanager**: route severity, inhibition cluster-down, group theo service.
   - *Đạt khi:* page→PagerDuty/ticket→Slack; cluster-down inhibit alert con; giải thích chống bão thông báo.

### ⑩ Đọc thêm / nguồn chuẩn
Google SRE Book (Monitoring, "Alerting on SLOs", "My Philosophy on Alerting" của Rob Ewaschuk); prometheus.io/docs/alerting (rules, Alertmanager — grouping/inhibition/silences); prometheus blackbox_exporter. Burn-rate alert → Bài 11; 4xx/5xx alert class → **F-160**.

> 🧪 **"Hiểu để chỉ huy & kiểm tra":** AI sinh alert rule rất hay đặt **`CPU>80%` / `mem>90%`** và **không có `for:`/runbook** — *chạy* nhưng **noise + không actionable** → fatigue. Hãy kiểm: *cái này có cần người dậy NGAY không? có symptom user-facing không? có `for:` + runbook không?*

---

## Bài 9 — Distributed tracing sâu I: span model, baggage, semantic conventions, instrumentation, Collector
*Phủ: O-TRACE-001 → 006*

### ① Mục tiêu & vị trí trong mạch
Nền tracing (trace/span là gì, context propagation, head/tail sampling định nghĩa, three pillars) đã học ở **K-OBS**. Bài này **đào sâu cơ chế**: **span kind/attributes/events/links**, **baggage**, **semantic conventions**, **auto vs manual instrumentation**, và **OTel Collector** (receivers→processors→exporters, agent vs gateway). Bài 10 lo *sampling vận hành + đọc trace*. Nắm bài này để tracing thật-sự-hữu-dụng, không chỉ "có trace".

### ② Giảng cơ bản → nâng cao

**(a) Trực giác.** Một trace là **một DAG span** mô tả hành trình 1 request. Nhưng "có span" chưa đủ — backend phải biết **span này là caller hay callee**, **biên service/queue ở đâu**, **field tên là gì** để dựng đúng topology & cho dashboard tái dùng. Đó là việc của *kind*, *semantic conventions*, và một lớp trung gian (**Collector**) để xử lý telemetry trước khi tới backend.

**(b) Cơ chế.**
- **Span kind (O-TRACE-001, ➕ def → K-OBS-002):** `SERVER`/`CLIENT`/`PRODUCER`/`CONSUMER`/`INTERNAL` cho backend dựng đúng **quan hệ caller/callee** và **biên service/queue** (client→server qua HTTP; producer→consumer qua broker). Thiếu kind → **topology/timeline sai**.
- **attributes vs events vs links (O-TRACE-002):** **attributes** = key-value **mô tả span** (`http.method`, `http.route`, `http.status_code`); **events** = **mốc thời điểm trong span** (vd "cache miss", một exception kèm timestamp); **links** = nối span thuộc **trace KHÁC** (vd 1 batch consumer xử lý nhiều message gốc → fan-in, link tới trace của từng message). Dùng đúng loại → trace giàu ngữ cảnh, đọc được.
- **Semantic conventions (O-TRACE-004, ➕ verify):** chuẩn hoá **tên attribute** cho thao tác chung (`http.*`, `db.*`, `messaging.*`, `rpc.*`) → **mọi instrumentation phát cùng tên field** → backend phân tích/đối sánh nhất quán, **dashboard/alert tái dùng** giữa service. Mỗi service tự đặt tên khác → **không correlate được** (đây là lý do "một chuẩn" quan trọng — Bài 1).
- **OTel Collector (O-TRACE-006, ➕ verify):** một process trung gian **receivers → processors → exporters**: nhận OTLP, **xử lý** (batch, redact PII, sample, enrich `k8sattributes`, **tail-sampling** — Bài 10), rồi **export** tới backend bất kỳ → **app không gắn cứng backend**, gánh xử lý ra khỏi app. Hai mode: **agent** (mỗi node/sidecar, gom gần app) đẩy lên **gateway** (tập trung tail-sampling/route). *(status component = mixed → verify từng cái.)*

**(c) Mép giới hạn & sai lầm thường gặp.**
- **Baggage (O-TRACE-003):** baggage = key-value **nghiệp vụ** truyền **xuyên service cùng context** (vd `tenant_id`, `feature_flag`) để service sau đọc — **khác** trace/span id (vốn chỉ *định danh* trace). **Bẫy nặng**: nhét **dữ liệu nhạy cảm/lớn** vào baggage → **rò qua mạng** (baggage đi trong header xuyên mọi hop) + **phình header** → dùng **tiết kiệm**, không bỏ PII vào baggage.
- **Auto vs manual instrumentation (O-TRACE-005, ➕ verify ngôn ngữ):** **auto** (SDK/agent, zero-code) phủ nhanh framework/HTTP/DB/driver phổ biến → **visibility tức thì**, ít sửa code. **Manual** cần cho **span nghiệp vụ** (logic domain, mốc quan trọng) mà auto **không biết** (vd "bước duyệt rủi ro", "gọi LLM"). Thực tế **kết hợp cả hai**; chỉ auto → thiếu ngữ cảnh business, chỉ manual → tốn công + bỏ sót.

### ③ ⚠️ Kiến thức cũ / bị thay thế

| Cái cũ (trong docs / phổ biến) | Thay bằng (hiện nay) | Vì sao |
|---|---|---|
| Mỗi service tự đặt tên field trace | **Semantic conventions** (`http.*`/`db.*`/`messaging.*`) *(verify)* | Cùng tên → correlate + dashboard/alert tái dùng |
| App export thẳng tới 1 backend tracing | App→OTLP→**Collector**→backend bất kỳ | Đổi/đa backend không re-instrument; xử lý/redact/sample tập trung |
| Nhồi dữ liệu nghiệp vụ/PII vào baggage | Baggage **tiết kiệm**, không PII/không lớn | Baggage đi header mọi hop → rò + phình |
| Chỉ auto-instrument là đủ | **Auto + manual** (span nghiệp vụ) | Auto không biết logic domain quan trọng |
| Vendor SDK riêng cho tracing | **OpenTelemetry** (trace stable) | Tránh lock-in; chuẩn CNCF |

### ④ Áp dụng thực tế + So sánh bigtech
Kiến trúc: *app (auto-instrument HTTP/DB + manual span nghiệp vụ, OTLP) → **Collector agent** (node) → **Collector gateway** (tail-sampling/redact/route) → Tempo/Jaeger + (exemplar nối từ Prometheus)*. Mọi span theo semantic conventions để dashboard chung.

Pattern phổ biến (*verify*): tổ chức cloud-native chuẩn hoá **OTel SDK + Collector**, **auto cho hạ tầng + manual cho business span**, **redact PII ở Collector**; LLM apps thêm span theo convention LLM (token/model/latency) qua OpenLLMetry. Triết lý: *instrument theo chuẩn, xử lý tập trung ở Collector, span nghiệp vụ là tay làm*.

### ⑤ Code thực hành + cấu hình
OTel Node setup (auto + manual span) + Collector pipeline:

```javascript
// tracing.js — OTel Node: auto-instrument + manual business span  (O-TRACE-005)
// npm i @opentelemetry/sdk-node @opentelemetry/auto-instrumentations-node @opentelemetry/exporter-trace-otlp-http
// verify đời package tại opentelemetry.io
const { NodeSDK } = require('@opentelemetry/sdk-node');
const { getNodeAutoInstrumentations } = require('@opentelemetry/auto-instrumentations-node');
const { OTLPTraceExporter } = require('@opentelemetry/exporter-trace-otlp-http');

const sdk = new NodeSDK({
  traceExporter: new OTLPTraceExporter({ url: 'http://otel-collector:4318/v1/traces' }), // → Collector, KHÔNG gắn cứng backend
  instrumentations: [getNodeAutoInstrumentations()], // AUTO: HTTP/Express/pg/redis... visibility tức thì
});
sdk.start();

// MANUAL span cho logic nghiệp vụ auto không biết (O-TRACE-005)
const { trace, SpanStatusCode } = require('@opentelemetry/api');
const tracer = trace.getTracer('checkout');
async function riskCheck(order) {
  return tracer.startActiveSpan('risk.evaluate', async (span) => {
    span.setAttribute('order.amount', order.amount);     // attribute mô tả span (theo convention)
    try {
      const r = await evaluate(order);
      span.addEvent('risk.decided', { decision: r.decision }); // EVENT = mốc thời điểm
      return r;
    } catch (e) {
      span.recordException(e); span.setStatus({ code: SpanStatusCode.ERROR });
      throw e;
    } finally { span.end(); }
  });
}
```

```yaml
# otel-collector.yaml — receivers→processors→exporters; agent đẩy lên gateway  (O-TRACE-006)
receivers: { otlp: { protocols: { http: {}, grpc: {} } } }
processors:
  batch: {}
  k8sattributes: {}                  # enrich pod/namespace
  attributes/redact:                 # REDACT PII trước khi rời biên (Bài 10/12)
    actions: [{ key: http.url, action: delete }]   # URL có thể chứa token
exporters: { otlp: { endpoint: tempo-gateway:4317 } }
service:
  pipelines:
    traces: { receivers: [otlp], processors: [k8sattributes, attributes/redact, batch], exporters: [otlp] }
```

> ⚠️ **Baggage & PII:** đừng đưa `email`/`token` vào baggage (đi header mọi hop → rò). Redact attribute nhạy cảm (URL có token, body) **ở Collector** trước khi tới backend/SaaS.

### ⑥ Keywords cần nhớ (glossary)
- 🧠 **Cần HIỂU:** vì sao cần span kind (topology) · attributes(mô tả) vs events(mốc) vs links(trace khác) · baggage là gì + bẫy PII · vì sao semantic conventions cho correlate/tái dùng · Collector tách xử lý khỏi app.
- 📌 **Cần THUỘC:** kind SERVER/CLIENT/PRODUCER/CONSUMER/INTERNAL · `http.*`/`db.*`/`messaging.*` · Collector = receivers→processors→exporters; agent→gateway · auto + manual.
- 🛠️ **Cần LÀM ĐƯỢC:** setup OTel auto + thêm manual business span (attribute/event) · cấu hình Collector enrich/redact · tránh PII trong baggage.

### ⑦ Mạch tư duy cần nhớ (mental model)
**"Span có KIND (ai gọi ai) + attributes(mô tả)/events(mốc)/links(trace khác), đặt tên theo SEMANTIC CONVENTIONS; auto phủ hạ tầng + manual cho business; Collector xử lý/redact/route tập trung."**

### ⑧ Câu hỏi phỏng vấn & thách đố
1. *(dễ)* Span kind để làm gì? → dựng caller/callee + biên service/queue; thiếu → topology sai.
2. *(TB)* attributes vs events vs links? → mô tả span / mốc thời điểm / nối trace khác (fan-in).
3. *(TB)* Semantic conventions giải gì? → chuẩn tên field → correlate + dashboard/alert tái dùng.
4. *(khó)* Collector đặt giữa app↔backend để làm gì, agent vs gateway? → xử lý/redact/sample/enrich + đổi backend không re-instrument; agent gom gần app, gateway tập trung tail-sample/route.
5. *(khó)* Baggage là gì, bẫy? → key-value nghiệp vụ xuyên service; bẫy: PII/lớn → rò header + phình.

**Thách đố / trick:**
- *"Tôi nhét `user_email` vào baggage để service sau dùng — security báo rò dữ liệu xuyên service. Vì sao?"* → baggage đi trong **header mọi hop** (kể cả ra ngoài biên/đối tác) → email lộ; dùng id tham chiếu, không PII.
- *"Auto-instrument bật rồi mà trace không thấy 'bước duyệt rủi ro' tốn 2s — vì sao?"* → auto chỉ biết HTTP/DB/driver, **không biết logic domain**; phải thêm **manual span** cho bước nghiệp vụ đó.

### ⑨ Bài tập thực hành + tiêu chí tự chấm
1. **Thêm manual span** cho 1 bước nghiệp vụ (vd gọi payment provider) với attribute + event + xử lý exception.
   - *Đạt khi:* span có attribute mô tả (theo convention), event mốc quan trọng, set ERROR khi exception; auto vẫn phủ HTTP/DB.
2. **Cấu hình Collector** enrich k8s + redact 1 field nhạy cảm + route tới backend.
   - *Đạt khi:* pipeline receivers→processors(k8sattributes, redact, batch)→exporters; giải thích vì sao redact ở Collector (tập trung, trước khi rời biên).

### ⑩ Đọc thêm / nguồn chuẩn
opentelemetry.io/docs (traces — span kind/attributes/events/links, baggage; semantic conventions; Collector — receivers/processors/exporters, deployment agent vs gateway; auto-instrumentation theo ngôn ngữ). Nền trace/span/propagation → **K-OBS**.

> 🧪 **"Hiểu để chỉ huy & kiểm tra":** AI bật auto-instrument là xong → *có trace* nhưng **thiếu span nghiệp vụ** (không thấy bước business chậm) và dễ **để PII trong attribute/baggage**. Hãy kiểm: *bước domain quan trọng có manual span không? attribute/baggage có lộ PII không? tên field có theo convention không?*

---

## Bài 10 — Distributed tracing sâu II: sampling, broken trace, cost, waterfall, critical path
*Phủ: O-TRACE-007 → 012*

### ① Mục tiêu & vị trí trong mạch
Bài 9 dạy *dựng trace giàu*. Bài này lo *giữ trace nào & đọc trace ra sao*: **tail-based sampling** (giữ trace lỗi/chậm — đặt ở đâu, tốn gì), vì sao sampling phải **consistent** (parent-based), **trace bị đứt** ở async, **chi phí & PII** của tracing, đọc **waterfall** để tìm bottleneck fan-out, và **critical path** (tối ưu đúng chỗ). Đây là phần phân biệt "có tracing" với "tracing dùng được ở scale".

### ② Giảng cơ bản → nâng cao

**(a) Trực giác.** Trace 100% mọi request ở scale là **bất khả thi** (chi phí ingest/lưu) → phải **sampling**. Câu hỏi không phải "có sample không" mà "**giữ đúng trace đáng giá**" (lỗi/chậm) và "**giữ nguyên vẹn cả trace**" (đừng đứt khúc). Và khi đã có trace, đọc nó phải biết **chỗ nào thật sự kéo dài tổng thời gian** (critical path).

**(b) Cơ chế.**
- **Tail-based sampling (O-TRACE-007, ➕ head/tail def → K-OBS-005):** muốn "**giữ MỌI trace lỗi/chậm**" thì phải quyết **SAU khi trace hoàn tất** → cần **buffer toàn bộ span của 1 trace** tới khi đủ → đặt ở **gateway Collector** (mọi span cùng trace phải **route về cùng nơi** theo trace id) → tốn **RAM + độ trễ**, đổi lại giữ được trace bất thường. (Head-based quyết ngay đầu → rẻ nhưng *không biết trước cái nào sẽ lỗi*.)
- **Consistent / parent-based sampling (O-TRACE-008):** nếu mỗi service **tự quyết độc lập** → trace **đứt khúc** (service này giữ, service kia bỏ) → **vô dụng**. **Parent-based**: quyết định sample được **truyền trong trace context** (sampled flag trong `traceparent`), con **theo cha** → trace **nguyên vẹn hoặc bỏ trọn**.
- **Trace đứt ở async (O-TRACE-009, ➕ broker → K-OBS-002):** nguyên nhân thường gặp: **không propagate context qua message header** → consumer mở **trace mới**, mất liên kết. Fix: đính `traceparent` vào **message header** lúc publish, đọc lại lúc consume (parent/span link). (Lỗi kinh điển với queue/Kafka.)
- **Đọc waterfall & fan-out (O-TRACE-011, ➕ vì sao micro khó → K-OBS-001):** trong 1 request fan-out nhiều downstream, **waterfall** cho thấy chặng nào chiếm thời gian (DB chậm? downstream X? chờ lock?), **phân biệt** latency **tuần tự cộng dồn** vs **chờ song song**. Định vị **trước khi đoán**.

**(c) Mép giới hạn & sai lầm thường gặp.**
- **Chi phí & PII của tracing (O-TRACE-010):** instrument thêm CPU/latency **nhỏ**, nhưng **volume span lớn** → chi phí ingest/lưu → **cần sampling**. PII lọt vào **attribute** (URL có token, request body) → **rò** → phải **redact ở Collector** (Bài 9). "Trace 100%" không khả thi ở scale.
- **Critical path (O-TRACE-012):** = chuỗi span **quyết định tổng latency** (cái mà nhanh hơn thì request nhanh hơn). Span **song song không nằm trên critical path** dù chậm cũng **không kéo dài tổng** → tối ưu nó là **phí công**. Phải đọc trace để biết tối ưu chỗ nào **THẬT SỰ** giảm latency. (Sai lầm: thấy span X chậm là lao vào tối ưu, trong khi X chạy song song và không nằm trên đường tới hạn.)

### ③ ⚠️ Kiến thức cũ / bị thay thế

| Cái cũ (trong docs / phổ biến) | Thay bằng (hiện nay) | Vì sao |
|---|---|---|
| Trace 100% mọi request | **Sampling** (giữ lỗi/chậm) | Volume span lớn → bất khả thi ở scale |
| Mỗi service tự quyết sample | **Parent-based / consistent** sampling | Tránh trace đứt khúc → vô dụng |
| Head-based cho mọi nhu cầu | **Tail-based ở gateway** khi cần giữ lỗi/chậm | Head không biết trước cái nào sẽ lỗi |
| Để URL/body thô trong span attribute | **Redact ở Collector** (token/PII) | Span rò dữ liệu nhạy cảm xuyên backend/SaaS |
| Thấy span chậm là tối ưu | Tối ưu **critical path** | Span song song ngoài đường tới hạn → tối ưu vô ích |
| Consumer tự mở trace mới | **Propagate `traceparent` qua message header** | Nối trace xuyên async; tránh đứt |

### ④ Áp dụng thực tế + So sánh bigtech
Setup: **agent Collector** gom → **gateway Collector** làm **tail-sampling** (giữ 100% trace có error/latency>ngưỡng + sample N% còn lại) + redact; producer đính `traceparent` vào Kafka header; on-call đọc **waterfall** + nhìn **critical path** để tối ưu đúng chỗ.

Pattern phổ biến (*verify*): nhiều tổ chức dùng **tail-sampling ở gateway** để "luôn giữ trace lỗi/chậm" mà vẫn kiểm soát chi phí; **parent-based** mặc định để trace nguyên vẹn; redact PII ở Collector là chuẩn. Triết lý: *giữ trace đáng giá, nguyên vẹn, không rò PII; tối ưu critical path*.

### ⑤ Code thực hành + cấu hình
Tail-sampling ở gateway + propagate context qua Kafka:

```yaml
# gateway-collector.yaml — tail_sampling: giữ MỌI trace lỗi/chậm + sample phần còn lại  (O-TRACE-007)
# verify cú pháp processor theo đời tailsamplingprocessor
processors:
  tail_sampling:
    decision_wait: 10s            # buffer span tới khi trace gần hoàn tất → tốn RAM/độ trễ
    policies:
      - name: keep-errors
        type: status_code
        status_code: { status_codes: [ERROR] }   # luôn giữ trace lỗi
      - name: keep-slow
        type: latency
        latency: { threshold_ms: 1000 }           # luôn giữ trace chậm
      - name: sample-rest
        type: probabilistic
        probabilistic: { sampling_percentage: 5 } # phần còn lại sample 5%
# LƯU Ý: tail-sampling phải ở GATEWAY — mọi span cùng trace route về cùng nơi theo trace id (O-TRACE-007)
```

```javascript
// propagate context qua message header để trace KHÔNG đứt ở async  (O-TRACE-009)
const { propagation, context } = require('@opentelemetry/api');
// publish: tiêm traceparent vào header message
const headers = {};
propagation.inject(context.active(), headers);   // headers['traceparent'] = '00-<traceid>-<spanid>-01'
await producer.send({ topic, messages: [{ value, headers }] });
// consume: trích context từ header → span con nối đúng trace (không mở trace mới)
const parentCtx = propagation.extract(context.active(), msg.headers);
// tracer.startActiveSpan('consume', {}, parentCtx, span => { ... })
```

> ⚠️ **Đặt sai chỗ tail-sampling = hỏng:** nếu mỗi node tự tail-sample, một node có thể giữ vài span của trace còn node khác bỏ → trace **đứt**. Phải gom về **gateway** thấy đủ trace (route theo trace id). Và **redact PII** ở Collector trước khi sample/lưu.

### ⑥ Keywords cần nhớ (glossary)
- 🧠 **Cần HIỂU:** vì sao tail-sampling phải ở gateway (buffer cả trace) · vì sao sampling phải consistent (parent-based) · vì sao trace đứt ở async · vì sao "trace 100%" bất khả thi · critical path vs span song song.
- 📌 **Cần THUỘC:** tail (giữ lỗi/chậm, tốn RAM/độ trễ, gateway) vs head (rẻ, đầu) · parent-based = con theo cha · `traceparent` qua message header · redact PII ở Collector · critical path = chuỗi quyết định tổng latency.
- 🛠️ **Cần LÀM ĐƯỢC:** cấu hình tail-sampling giữ lỗi/chậm · propagate context qua broker · đọc waterfall & chỉ ra critical path.

### ⑦ Mạch tư duy cần nhớ (mental model)
**"Sample để sống được, nhưng GIỮ trace lỗi/chậm (tail ở gateway) và GIỮ NGUYÊN VẸN (parent-based). Nối trace qua header ở async. Tối ưu CRITICAL PATH, không phải span song song chậm."**

### ⑧ Câu hỏi phỏng vấn & thách đố
1. *(dễ)* Vì sao cần sampling, tail vs head khác gì? → volume lớn; tail giữ lỗi/chậm (sau khi xong), head rẻ (đầu).
2. *(TB)* Tail-sampling đặt ở đâu, tốn gì? → gateway (buffer cả trace, route theo trace id); tốn RAM/độ trễ.
3. *(TB)* Vì sao sampling phải consistent? → tránh trace đứt khúc; parent-based con theo cha.
4. *(khó)* Trace đứt ở queue — vì sao, fix? → không propagate context qua message header → consumer mở trace mới; đính `traceparent` vào header.
5. *(khó)* Critical path là gì, vì sao tối ưu span ngoài nó là phí? → chuỗi quyết định tổng latency; span song song ngoài đường tới hạn không kéo dài tổng.

**Thách đố / trick:**
- *"Tôi tối ưu cái span DB chậm nhất 500ms mà tổng latency không giảm — vì sao?"* → span đó **chạy song song**, **không nằm trên critical path**; đường tới hạn là chuỗi span khác cộng dồn. Đọc waterfall tìm critical path trước.
- *"Mỗi request lỗi tôi lại không tìm thấy trace của nó — sampling đúng chưa?"* → head-based/random bỏ mất trace lỗi; cần **tail-sampling giữ ERROR** ở gateway → luôn có trace của request lỗi.

### ⑨ Bài tập thực hành + tiêu chí tự chấm
1. **Cấu hình tail-sampling** "giữ 100% lỗi + 100% chậm >1s + 5% còn lại" ở gateway.
   - *Đạt khi:* policy đúng (status_code ERROR, latency, probabilistic); giải thích vì sao phải ở gateway (route theo trace id).
2. **Đọc 1 waterfall** (cho/giả lập) và chỉ ra critical path + chỗ tối ưu thật.
   - *Đạt khi:* phân biệt tuần-tự-cộng-dồn vs chờ-song-song; chỉ đúng span trên critical path để tối ưu.

### ⑩ Đọc thêm / nguồn chuẩn
opentelemetry.io (sampling — head/tail, parent-based; Collector `tailsamplingprocessor`; context propagation across messaging); Tempo/Jaeger docs (đọc waterfall). Head/tail định nghĩa nền → **K-OBS-005**; broker propagation → **K-OBS-002**.

> 🧪 **"Hiểu để chỉ huy & kiểm tra":** AI cấu hình sampling thường để **random/head 10%** → *chạy* nhưng **mất trace lỗi** và có thể **đứt khúc**. Hãy kiểm: *có giữ 100% trace lỗi/chậm không? tail-sampling có ở gateway không? parent-based để trace nguyên vẹn chưa? async có propagate context không?*

---

## Bài 11 — SLO / SLI / Error budget
*Phủ: O-SLO-001 → 011*

### ① Mục tiêu & vị trí trong mạch
Đây là phần **tổng hợp mọi pillar thành mục tiêu & ngân sách** — và là *ngôn ngữ ra quyết định* của TL (cân reliability ↔ tốc độ ship). Học xong bạn phân biệt **SLI/SLO/SLA**, định nghĩa **SLI tốt**, hiểu **error budget** nối reliability với velocity, vì sao **100% là sai**, **burn-rate alerting**, viết SLO latency đúng (percentile+ngưỡng+cửa sổ), chọn **target**, các chỉ số **MTTR/MTTD/...**, **toil**, và **blameless postmortem + error budget policy**. Burn-rate dựa trên metric/SLI đã học (Bài 5–8).

### ② Giảng cơ bản → nâng cao

**(a) Trực giác.** "Service phải nhanh và đáng tin" là **vô nghĩa** cho tới khi bạn **đo được** và **đặt mục tiêu**. SLO biến reliability thành **một con số có ngân sách**: bạn được phép lỗi *bao nhiêu*, và khi tiêu hết ngân sách thì **dừng ship feature, dồn vào reliability**. Nó chấm dứt cãi cảm tính "đủ ổn định chưa".

**(b) Cơ chế.**
- **SLI / SLO / SLA (O-SLO-001):** **SLI** = **số đo thực** (vd % request `<300ms` & thành công); **SLO** = **mục tiêu nội bộ** trên SLI (vd 99.9%); **SLA** = **cam kết với khách + hệ quả** (phạt/đền), thường **lỏng hơn** SLO. Đặt **SLO < SLA** để có biên an toàn.
- **SLI "tốt" (O-SLO-002):** đo **trải nghiệm user** = **good events / valid events** (request thành công & đủ nhanh / tổng request hợp lệ). "**Uptime của server**" là SLI **dở** vì server *up* vẫn có thể trả 500/chậm → không phản ánh user. Chọn SLI theo **điểm chạm user**.
- **Error budget nối reliability↔velocity (O-SLO-003):** budget = **100% − SLO** (99.9% → 0.1% được phép lỗi). **Còn budget** → team được **ship nhanh/mạo hiểm**; **hết budget** → **freeze feature**, dồn reliability. Biến reliability thành **ngân sách định lượng**, không cãi cảm tính.
- **Burn-rate alerting (O-SLO-005, ➕ verify công thức):** alert theo **tốc độ tiêu budget** (burn rate) thay vì ngưỡng tĩnh "error>1%". **Multi-window multi-burn-rate** (vd cửa sổ 1h **và** 5m): burn rate cao → **page gấp** (cháy nhanh); burn rate thấp-kéo-dài → **ticket** (rò rỉ chậm). Yêu cầu *cả* hai cửa sổ vượt → giảm false positive + phản ứng đúng tốc độ. Thay cho ngưỡng tĩnh vốn *hoặc ồn hoặc chậm*.

**(c) Mép giới hạn & sai lầm thường gặp.**
- **SLO latency phải đủ 3 yếu tố (O-SLO-006):** **percentile + ngưỡng + cửa sổ** (vd "**p99 < 300ms trong 30 ngày**"), không phải "nhanh". "Nhanh" **không đo được**; thiếu 1 yếu tố → SLO **không enforce được**.
- **100% là mục tiêu sai (O-SLO-004):** chi phí tăng **phi tuyến** gần 100%; user thường **không phân biệt** 99.99% vs 100% (đã có lỗi mạng/client phía họ); và **0 budget = không được ship/đổi gì**. Mục tiêu là "**đủ tin cậy cho user**", không tuyệt đối.
- **Chọn target (O-SLO-007):** dựa kỳ vọng user/đối thủ/chi phí + **phụ thuộc downstream** (SLO **không thể cao hơn** dependency của bạn); mỗi "số 9" thêm **tốn kém lớn**. Đặt theo nhu cầu thật, **đo được, giữ được**.
- **MTTD/MTTA/MTTR/MTBF (O-SLO-008):** MTT**D** (phát hiện), MTT**A** (ack), MTT**R** (khắc phục), MT**BF** (giữa 2 sự cố). Ở hệ phân tán, **giảm MTTD/MTTR** (phát hiện/phục hồi nhanh) thường **thực tế hơn** đuổi theo "không bao giờ hỏng" — và **observability tốt kéo MTTD/MTTR xuống**.
- **Toil (O-SLO-009):** việc **lặp tay, thủ công, không tạo giá trị lâu dài, scale tuyến tính theo tải** (restart tay, xử lý alert lặp). **Đo + đặt trần toil** → ép **automation**; alert/observability tốt giảm toil điều tra.
- **Blameless postmortem + error budget policy (O-SLO-010):** postmortem tập trung **hệ thống/quy trình** (vì sao môi trường *cho phép* lỗi) **không cá nhân** → người dám **báo cáo trung thực**. **Error budget policy** = quy ước **TRƯỚC** "hết budget thì làm gì" (freeze, ưu tiên reliability) → quyết định **khách quan, không chính trị**.
- **Ít SLO, gắn user journey (O-SLO-011):** quá nhiều SLO → **loãng, không ai hành động**; chọn **vài luồng giá trị nhất** (checkout, login, search). SLO là **công cụ ra quyết định**, không phải bộ sưu tập số.

### ③ ⚠️ Kiến thức cũ / bị thay thế

| Cái cũ (trong docs / phổ biến) | Thay bằng (hiện nay) | Vì sao |
|---|---|---|
| "Service phải nhanh & ổn định" (cảm tính) | **SLI/SLO + error budget** (định lượng) | Đo được + ra quyết định ship vs reliability |
| SLI = "server uptime" | SLI = **good/valid events** (trải nghiệm user) | Server up vẫn trả 500/chậm; uptime không phản ánh user |
| Mục tiêu 100% reliability | "**Đủ tin cậy cho user**" (vài số 9 đúng nhu cầu) | Chi phí phi tuyến; 0 budget = không ship được |
| Alert ngưỡng tĩnh "error>1%" | **Burn-rate multi-window** *(verify công thức)* | Bắt cả cháy nhanh (page) lẫn rò chậm (ticket), ít false positive |
| SLO "nhanh"/mơ hồ | **percentile + ngưỡng + cửa sổ** (p99<300ms/30 ngày) | Không đủ 3 yếu tố → không enforce được |
| Postmortem đổ lỗi cá nhân | **Blameless** + **error budget policy** định trước | Báo cáo trung thực + quyết định khách quan |
| SLO cho mọi endpoint/metric | **Ít SLO, gắn user journey** quan trọng | Quá nhiều → loãng, không ai hành động |

### ④ Áp dụng thực tế + So sánh bigtech
Setup: định nghĩa SLI (success & p99<300ms) cho **checkout/login/search**; SLO 99.9%/30 ngày; recording rule tính SLI; **burn-rate alert** (fast 1h + slow 6h) → page/ticket; error budget policy: hết budget → freeze feature. Postmortem blameless sau incident.

Pattern phổ biến (*verify*): Google SRE là nguồn gốc error budget + burn-rate + golden signals; nhiều tổ chức áp **error budget policy** chính thức (freeze khi cạn) và **blameless culture**. Triết lý (Google SRE Workbook): *SLO là hợp đồng giữa reliability và tốc độ; budget cho phép mạo hiểm có kiểm soát*.

### ⑤ Code thực hành + cấu hình
SLI recording rule + burn-rate multi-window alert (PromQL):

```yaml
# slo.yml — SLI + multi-window burn-rate (page nhanh / ticket chậm)  (O-SLO-005)
# verify công thức/cửa sổ theo Google SRE Workbook "Alerting on SLOs"
groups:
  - name: checkout-slo
    rules:
      # SLI = tỉ lệ request "tốt" (good/valid). Quy ước: status không 5xx & latency đạt = good.
      # Dùng recording rule cho nhiều cửa sổ để burn-rate query nhanh (Bài 6).
      - record: job:slo_errors:ratio_rate5m
        expr: |
          sum(rate(http_requests_total{job="checkout",status=~"5.."}[5m]))
            / sum(rate(http_requests_total{job="checkout"}[5m]))
      - record: job:slo_errors:ratio_rate1h
        expr: |
          sum(rate(http_requests_total{job="checkout",status=~"5.."}[1h]))
            / sum(rate(http_requests_total{job="checkout"}[1h]))
      # FAST BURN (page): tiêu budget rất nhanh. SLO 99.9% → budget 0.1% = 0.001.
      # Ngưỡng 14.4x ≈ tiêu hết 30 ngày budget trong ~2h (theo SRE Workbook).
      # Yêu cầu CẢ cửa sổ ngắn (5m) lẫn dài (1h) vượt → ít false positive.
      - alert: CheckoutErrorBudgetFastBurn
        expr: |
          job:slo_errors:ratio_rate1h  > 14.4 * 0.001
          and job:slo_errors:ratio_rate5m > 14.4 * 0.001
        for: 2m
        labels: { severity: page }
        annotations:
          summary: "Checkout đang đốt error budget rất nhanh"
          runbook_url: "https://runbooks/internal/checkout-burn"
      # SLOW BURN (ticket): rò rỉ chậm — cần xử lý nhưng không gọi dậy.
      - alert: CheckoutErrorBudgetSlowBurn
        expr: |
          job:slo_errors:ratio_rate1h > 3 * 0.001
        for: 1h
        labels: { severity: ticket }
        annotations: { summary: "Checkout rò rỉ budget chậm — xử lý trong giờ làm" }
```

> ⚠️ **Số ngưỡng burn-rate (14.4x, 6x, 3x...) là theo Google SRE Workbook cho từng cặp cửa sổ — `// kiểm chứng tại sre.google/workbook 'Alerting on SLOs'` trước khi áp.** SLI phải gắn **user journey** (checkout), không phải "uptime server". Latency-SLI: đếm request đạt ngưỡng (vd dùng bucket `le="0.3"` của histogram).

### ⑥ Keywords cần nhớ (glossary)
- 🧠 **Cần HIỂU:** SLI/SLO/SLA quan hệ · vì sao "server uptime" là SLI dở · error budget nối reliability↔velocity · vì sao 100% sai · burn-rate vs ngưỡng tĩnh · vì sao SLO phải ít & gắn user journey · blameless.
- 📌 **Cần THUỘC:** SLI=good/valid events · budget=100%−SLO · SLO latency = percentile+ngưỡng+cửa sổ · MTTD/MTTA/MTTR/MTBF · toil = lặp-tay-scale-tuyến-tính · error budget policy = quy ước trước.
- 🛠️ **Cần LÀM ĐƯỢC:** định nghĩa SLI cho 1 user journey · viết SLI recording rule + burn-rate alert · chạy postmortem blameless + áp budget policy.

### ⑦ Mạch tư duy cần nhớ (mental model)
**"SLI đo trải nghiệm user (good/valid); SLO là mục tiêu; budget = 100%−SLO nối reliability với tốc độ ship. Alert theo TỐC ĐỘ đốt budget (burn-rate), không phải ngưỡng tĩnh. 100% là sai. Ít SLO, gắn user journey."**

### ⑧ Câu hỏi phỏng vấn & thách đố
1. *(dễ)* Phân biệt SLI/SLO/SLA? → số đo / mục tiêu nội bộ / cam kết khách (lỏng hơn, có phạt); SLO<SLA.
2. *(TB)* SLI "tốt" định nghĩa thế nào, vì sao "uptime" dở? → good/valid events theo user; server up vẫn 500/chậm.
3. *(TB)* Error budget nối reliability với velocity ra sao? → budget=100%−SLO; còn → ship nhanh, hết → freeze.
4. *(khó)* Burn-rate khác alert ngưỡng tĩnh thế nào? → đo tốc độ đốt budget, multi-window bắt cả nhanh (page) lẫn chậm (ticket), ít false positive.
5. *(khó)* Vì sao 100% reliability là mục tiêu sai? → chi phí phi tuyến, user không phân biệt, 0 budget = không ship.

**Thách đố / trick:**
- *"Sếp muốn SLO 100% uptime cho mọi endpoint — tôi nên phản biện thế nào?"* → (1) 100% sai (chi phí phi tuyến, 0 budget không ship); (2) "uptime" không phải SLI user; (3) quá nhiều SLO → loãng. Đề xuất: vài SLO gắn user journey, target đo-được-giữ-được (vd 99.9%), error budget policy.
- *"SLO của tôi 99.99% nhưng dependency payment chỉ cam kết 99.9% — sao mãi không đạt?"* → SLO **không thể cao hơn downstream**; bạn bị chặn bởi dependency; hạ target hoặc thêm fallback/degradation, đừng đặt mục tiêu bất khả thi.

### ⑨ Bài tập thực hành + tiêu chí tự chấm
1. **Định nghĩa SLI/SLO cho checkout** (success + latency) + viết SLI recording rule.
   - *Đạt khi:* SLI = good/valid events gắn user journey; SLO có percentile+ngưỡng+cửa sổ; rule dùng histogram bucket cho latency-SLI.
2. **Viết burn-rate alert fast+slow** + soạn 1 error budget policy ngắn.
   - *Đạt khi:* fast→page/slow→ticket, multi-window; policy nêu rõ "hết budget thì freeze feature, ưu tiên reliability" (định trước, khách quan).

### ⑩ Đọc thêm / nguồn chuẩn
Google SRE Book + **SRE Workbook** (chương "Implementing SLOs", "Alerting on SLOs" — burn-rate multi-window; "Eliminating Toil"; "Postmortem Culture"); sre.google/workbook. Percentile load-test → **L-363**.

> 🧪 **"Hiểu để chỉ huy & kiểm tra":** AI viết SLO thường lấy **"uptime 99.9%"** hoặc alert **error rate tĩnh** — *trông hợp lý* nhưng **không gắn user + hoặc ồn hoặc chậm**. Hãy kiểm: *SLI có đo good/valid events theo user journey không? có dùng burn-rate multi-window không? target có đo-được-giữ-được (không 100%, không cao hơn downstream) không?*

---

## Bài 12 — TL-level cross-cutting
*Phủ: O-TL-001 → 010*

### ① Mục tiêu & vị trí trong mạch
Bài cuối **ghép mọi pillar thành quyết định của Tech Lead**: **build vs buy**, vì sao **OTel làm chuẩn instrumentation**, **kiểm soát chi phí observability**, **observability-driven development**, văn hoá **you-build-it-you-run-it / on-call**, đưa observability vào **definition of done & CI/CD**, biến **incident "mù"** thành cải thiện có mục tiêu, **golden path** chuẩn hoá across team, **privacy/compliance**, và **review observability do AI/junior thêm**. Đây là tầng "chỉ huy & kiểm tra" — chốt cả mục O.

### ② Giảng cơ bản → nâng cao

**(a) Trực giác.** Ở mức TL, observability không còn là "gắn thư viện" mà là **quyết định có đánh đổi**: tự dựng hay mua? cắt chi phí thế nào mà không mù? làm sao toàn org cùng một "ngôn ngữ" telemetry? đưa quan sát thành *yêu cầu của feature*, không phải chữa cháy. Và quan trọng: **AI viết đúng cú pháp nhưng dễ sai chỗ** — việc của bạn là *kiểm tra*.

**(b) Cơ chế.**
- **Build vs buy (O-TL-001, ➕ verify giá):** tự dựng (Prometheus+Grafana+Loki+Tempo+Jaeger) vs SaaS (Datadog/New Relic/Honeycomb/Grafana Cloud). Cân **engineer-time vận hành stack** (scale, lưu trữ, nâng cấp) **vs chi phí SaaS** (thường theo host/volume → **đắt rất nhanh ở scale**), **năng lực team**, **lock-in**, **data residency**. OSS rẻ license nhưng **tốn người**. Quyết theo **bối cảnh, không tín điều**.
- **OTel làm chuẩn instrumentation (O-TL-002, ➕ verify):** instrument **1 lần theo OTel** → đổi/đa backend **không re-instrument** → tránh **lock-in ở chỗ đắt nhất** (code đã gắn). Chuẩn CNCF + semantic conventions thống nhất → telemetry correlate. Tách "**thu thập**" khỏi "**lưu/hiển thị**" → tự do đổi backend.
- **Observability-driven development (O-TL-004):** coi telemetry là **yêu cầu của feature** — span/metric/log + **SLI quan trọng định nghĩa từ đầu**, kiểm tra "quan sát được" ở staging, mỗi service **ship kèm dashboard/alert**. Không "code xong mới nghĩ làm sao debug prod". Văn hoá: *bạn build nó, bạn quan sát nó*.
- **DoD + CI/CD (O-TL-006, ➕ gate → N-CICD):** feature **chưa có** log/metric/trace + SLI/alert thì **chưa "done"**; có thể **gate deploy** bằng "có dashboard/alert", smoke + SLO check sau deploy. Quan sát là **một phần chất lượng**, không tuỳ hứng.

**(c) Mép giới hạn & sai lầm thường gặp.**
- **Chi phí observability có thể vượt cả chi phí chạy app (O-TL-003):** đòn bẩy kiểm soát **mà không "mù"**: **cắt cardinality metric**, **sample trace/log thông minh (giữ lỗi)**, **retention/tiering** theo giá trị, **drop field thừa ở Collector**, đo "**cost per signal**", **review telemetry như review code**. Giữ đủ tín hiệu điều tra với chi phí kiểm soát — **KHÔNG tắt bừa rồi mù lúc incident**.
- **You build it, you run it / on-call (O-TL-005):** dev trực dịch vụ mình → **feedback chất lượng↔vận hành**, sửa tận gốc. **Rủi ro**: burnout, alert fatigue, thiếu kỹ năng ops. TL set up bền vững: **rotation hợp lý**, runbook, **alert tinh gọn (actionable)**, giảm toil, **không page rác**.
- **Incident "mù" → hành động hệ thống (O-TL-007):** sau incident không-đủ-telemetry, postmortem **xác định "tín hiệu nào thiếu"** khiến MTTD/MTTR cao → **bổ sung metric/trace/log/alert ĐÚNG chỗ mù đó** (không log bừa khắp nơi). Biến **mỗi incident thành cải thiện observability có mục tiêu**; tránh lặp.
- **Golden path / chuẩn hoá across team (O-TL-008, ➕ IDP → N-TL-006):** thư viện/instrumentation chuẩn + naming (semantic conventions) + **dashboard template** → telemetry **correlate** được giữa service, on-call **đọc được dashboard service lạ**, giảm reinvent; "**paved road**" qua platform/IDP; **vẫn cho lệch khi cần**. Tránh "mỗi team một kiểu".
- **Privacy/compliance (O-TL-009, ➕ threat/PII → M):** **redact PII ở nguồn/Collector**, kiểm soát ai xem log (**RBAC**), **retention theo luật** (GDPR xoá), **data residency** (telemetry rời biên/sang SaaS nước khác?), **audit tách kênh**. Observability **không được biến thành lỗ hổng rò dữ liệu**.
- **Review observability do AI/junior (O-TL-010):** **cú pháp đúng ≠ quan sát đúng**. Kiểm: metric có **cardinality bom** (label `user_id`)? đo **symptom user-facing** hay chỉ resource? latency dùng **histogram + percentile** hay average? log có rò **PII/secret**? alert có **actionable/runbook** hay sẽ thành noise? SLI có gắn **user journey**? — *Hiểu để chỉ huy & kiểm tra, không nghiệm thu theo "chạy không lỗi".*

### ③ ⚠️ Kiến thức cũ / bị thay thế

| Cái cũ (trong docs / phổ biến) | Thay bằng (hiện nay) | Vì sao |
|---|---|---|
| Vendor SDK/agent riêng cho mỗi backend | **OpenTelemetry** instrument 1 lần *(verify)* | Tránh lock-in ở code đã gắn; đổi backend không re-instrument |
| "Cứ log/trace tất, tính chi phí sau" | **Cost per signal + cardinality cut + sample giữ-lỗi + tiering** | Observability có thể đắt hơn cả app; cắt mà không mù |
| Observability = chữa cháy sau khi prod lỗi | **Observability-driven dev** (telemetry là requirement) | Ship kèm dashboard/alert/SLI; debug được từ đầu |
| Mỗi team một kiểu instrument/dashboard | **Golden path**: lib chuẩn + naming + template (IDP) | Correlate xuyên service; on-call đọc được dashboard lạ |
| Log/trace để PII tự nhiên, lo sau | **Redact ở nguồn/Collector + RBAC + retention luật + residency** | Observability không được thành lỗ hổng rò dữ liệu *(M)* |
| Nghiệm thu observability "chạy không lỗi" | **Review như code**: cardinality/symptom/histogram/PII/actionable | Cú pháp đúng ≠ quan sát đúng |

### ④ Áp dụng thực tế + So sánh bigtech
Quyết định TL điển hình: team nhỏ/sớm → **SaaS** (Datadog/Grafana Cloud, đỡ vận hành, chấp nhận chi phí) để dồn người vào sản phẩm; scale lớn/chi phí SaaS vượt ngưỡng → cân nhắc **tự host LGTM** + có đội platform. Mọi nơi: **OTel làm lớp instrumentation** để giữ tự do đổi backend. Golden path: thư viện telemetry nội bộ + dashboard template qua IDP (→ **N-TL-006**, **R**).

Pattern phổ biến (*verify*): nhiều tổ chức bắt đầu SaaS rồi "repatriate" một phần khi bill observability tăng phi mã; **OTel + Collector** là điểm chung để không bị khoá; **blameless + error budget policy** (Bài 11) + **golden path** là dấu hiệu org observability trưởng thành. Triết lý: *chuẩn hoá thu thập (OTel), linh hoạt backend, cắt chi phí có chủ đích, quan sát là chất lượng + là một phần của privacy*.

### ⑤ Code thực hành + cấu hình
Không phải bài code-nặng — minh hoạ **đòn bẩy cắt chi phí ở Collector** (cardinality/drop/sample) + **gate observability trong CI**:

```yaml
# cost-control ở Collector — cắt chi phí mà KHÔNG mù (O-TL-003)
processors:
  # 1) chặn metric cardinality bom trước khi vào TSDB (Bài 6)
  metricstransform/drop_high_card:
    transforms:
      - { include: ".*", match_type: regexp, action: update,
          operations: [{ action: delete_label_value, label: user_id }] }  # bỏ label cao-cardinality
  # 2) tail-sampling GIỮ lỗi/chậm, sample phần còn lại (Bài 10) — giữ tín hiệu điều tra
  # 3) drop field log thừa (Bài 3) + redact PII (Bài 9/privacy)
  attributes/cost:
    actions: [{ key: debug_blob, action: delete }, { key: authorization, action: delete }]
# verify tên processor theo đời Collector (status component = mixed)
```

```yaml
# .github/workflows — gate "observability là definition-of-done" (O-TL-006) — minh hoạ
jobs:
  obs-gate:
    steps:
      - run: ./scripts/check-dashboards.sh   # service có dashboard + alert chưa? thiếu → fail
      - run: ./scripts/check-slo.sh          # có định nghĩa SLI/SLO cho user journey chưa?
      - run: ./scripts/lint-metrics.sh       # cardinality lint: cấm label user_id/email/url-thật
```

> ⚠️ **Cắt mà không mù:** đừng "tắt trace cho rẻ" → mù lúc incident. Cắt theo thứ tự: **cardinality → drop field thừa → sample giữ-lỗi → tiering retention**, đo **cost per signal**. Redact PII là **bắt buộc** trước khi telemetry rời biên/sang SaaS (data residency — **M**).

### ⑥ Keywords cần nhớ (glossary)
- 🧠 **Cần HIỂU:** build-vs-buy đánh đổi gì · vì sao OTel chống lock-in ở chỗ đắt nhất · cắt chi phí mà không mù · observability-driven dev · you-build-it-you-run-it rủi ro · golden path · privacy của telemetry · vì sao "chạy không lỗi" không phải nghiệm thu.
- 📌 **Cần THUỘC:** đòn bẩy cost = cardinality/sample-giữ-lỗi/tiering/drop-field/cost-per-signal · DoD = chưa có telemetry+SLI+alert thì chưa done · redact-ở-nguồn + RBAC + retention-luật + data-residency.
- 🛠️ **Cần LÀM ĐƯỢC:** ra quyết định build-vs-buy có lập luận · thiết kế golden path tối thiểu · review 1 PR observability của junior/AI và chỉ ra lỗi quan-sát-sai.

### ⑦ Mạch tư duy cần nhớ (mental model)
**"Chuẩn hoá THU THẬP (OTel) → tự do đổi backend; cắt chi phí CÓ CHỦ ĐÍCH (không mù); observability là DEFINITION OF DONE + một phần PRIVACY; và cú pháp đúng ≠ quan sát đúng — TL kiểm tra, không nghiệm thu 'chạy được'."**

### ⑧ Câu hỏi phỏng vấn & thách đố
1. *(TB)* Build vs buy observability — quyết theo gì? → engineer-time vận hành vs chi phí SaaS (host/volume), năng lực team, lock-in, data residency; theo bối cảnh.
2. *(TB)* Vì sao OTel được khuyên làm chuẩn instrumentation? → instrument 1 lần, đổi backend không re-instrument; tránh lock-in ở code; semantic conventions correlate.
3. *(khó)* Chi phí observability vượt chi phí app — cắt sao mà không mù? → cardinality/sample-giữ-lỗi/tiering/drop-field/cost-per-signal; không tắt bừa.
4. *(khó)* Incident "mù" — TL rút ra hành động hệ thống gì? → tìm "tín hiệu nào thiếu" → bổ sung đúng chỗ mù (không log bừa); mỗi incident → cải thiện observability có mục tiêu.
5. *(khó)* Review observability AI/junior — kiểm gì? → cardinality bom, symptom vs resource, histogram vs average, PII/secret, alert actionable/runbook, SLI gắn user journey.

**Thách đố / trick:**
- *"AI thêm metric `request_duration` (gauge) + label `user_id` + alert `CPU>80%`, log `req.body`, SLO 'uptime 99.99%'. Code chạy, CI xanh. Duyệt không?"* → **KHÔNG**: gauge cho latency (mất phân bố → histogram), `user_id` = cardinality bom, alert cause-based noise, log rò PII, SLO sai (uptime không phải SLI user + có thể cao hơn downstream). *Đây chính là "chạy không lỗi ≠ quan sát đúng".*
- *"Bill Datadog tháng này = 2× chi phí chạy app — TL làm gì đầu tiên?"* → soi **cost per signal** + **cardinality** (label nổ) + **verbosity/sample** (ai bật debug/trace 100%?); cắt theo thứ tự giữ-lỗi; cân nhắc OTel để giữ tự do đổi backend; **không tắt bừa**.

### ⑨ Bài tập thực hành + tiêu chí tự chấm
1. **Viết quyết định build-vs-buy** 1 trang cho 1 bối cảnh (team 8 người, scale vừa, có compliance data residency).
   - *Đạt khi:* cân engineer-time vs SaaS cost + lock-in + residency + năng lực team; kết luận theo bối cảnh; đề xuất OTel làm lớp chung dù chọn gì.
2. **Code review 1 PR observability "xấu"** (gauge latency, label cao-cardinality, alert CPU, log PII, SLO uptime) — liệt kê lỗi + cách sửa.
   - *Đạt khi:* bắt đủ 5 lỗi quan-sát-sai + sửa (histogram, label bounded, symptom alert+runbook, redact, SLI user-journey).

### ⑩ Đọc thêm / nguồn chuẩn
opentelemetry.io (vendor-neutral, Collector, semantic conventions); Google SRE Workbook (toil, on-call, postmortem, error budget policy); CNCF observability whitepaper; tài liệu IDP/platform (golden path). Build-vs-buy giá → *verify trang pricing Datadog/New Relic/Grafana Cloud*. Privacy/PII → **M**; CI/CD gate → **N-CICD**; IDP → **N-TL-006 / R**.

> 🧪 **"Hiểu để chỉ huy & kiểm tra" (chốt mục O):** AI/junior thêm telemetry **đúng cú pháp** nhưng dễ **sai chỗ** — đó là lý do TL tồn tại. Quy tắc nghiệm thu: *cardinality bounded? symptom user-facing? histogram+percentile? không PII/secret? alert actionable+runbook? SLI gắn user journey? cắt chi phí mà không mù?* Nếu một trong số đó "không" → **chưa done**, dù CI xanh.

---

# ✅ Tổng kết & dùng tiếp (Bước D–E)

## Bản đồ "một câu chốt" mỗi bài
| Bài | Câu thần chú |
|---|---|
| 1 | Monitoring canh điều đã biết; observability hỏi được điều chưa lường. Cardinality: log/trace quý, label metric nổ. |
| 2 | Log = dữ liệu có schema cho MÁY, ra stdout, có id, KHÔNG có secret. |
| 3 | Log tốn CPU lúc ghi (đừng chặn event loop) + tốn tiền lúc lưu (đừng index vô tận); audit ≠ debug log. |
| 4 | Agent gom (DaemonSet) → store (ES đắt / Loki rẻ); error tracking gộp exception thành issue; pipeline log KHÔNG kéo sập app. |
| 5 | Counter→rate, gauge→hiện tại, histogram→p99. Latency LUÔN histogram. Prometheus pull; `up` báo còn sống. |
| 6 | Label phải BOUNDED. Counter→rate. p99 từ histogram là xấp xỉ theo bucket. Cái nặng → recording rule. |
| 7 | Prometheus = thu thập metric (1 node); lưu-dài/global là lớp khác; trace/log đi backend khác. Đo BUSINESS, tránh watermelon. |
| 8 | Page khi "cần làm NGAY" dựa TRIỆU CHỨNG user (không CPU), có `for:` + runbook. Alert là sản phẩm phải tỉa. |
| 9 | Span có KIND + attributes/events/links, đặt tên theo semantic conventions; auto + manual; Collector xử lý/redact/route. |
| 10 | Sample để sống, nhưng GIỮ lỗi/chậm (tail ở gateway) + NGUYÊN VẸN (parent-based). Tối ưu CRITICAL PATH. |
| 11 | SLI đo user (good/valid); budget=100%−SLO nối reliability↔ship; alert theo TỐC ĐỘ đốt budget; 100% là sai. |
| 12 | Chuẩn hoá thu thập (OTel) → tự do backend; cắt chi phí không mù; observability là DoD + privacy; cú pháp đúng ≠ quan sát đúng. |

## Lịch ôn ngắt quãng (spaced repetition) — gợi ý
- **Hôm nay:** Bài 1 (pillars/cardinality) + Bài 5–6 (metric types/histogram/cardinality) — phần "đạt vững".
- **Mai:** Bài 8 (alert symptom/page-vs-ticket) + Bài 11 (SLI/SLO/error budget/burn-rate).
- **3 ngày:** Bài 2–3 (logging) + Bài 4 (stack) + Bài 7 (Prom 3.0/business/watermelon).
- **1 tuần:** Bài 9–10 (tracing sâu) + Bài 12 (TL-level) — mức "giải thích + trade-off".

## Sang Bước D (phỏng vấn live)
Khi muốn luyện: yêu cầu *"phỏng vấn mục O, đợt 10–15 câu trộn dễ→khó"*. AI sẽ HỎI XONG **DỪNG**, chờ bạn trả lời (recall trước, đáp án sau), rồi dựng đáp án chuẩn live (đáp án lõi + bẫy + trade-off), bám đúng dòng *"dò cái gì"* của từng câu trong `QB_O_observability.md` để chấm ĐẠT/CHƯA, và chèn 1–2 câu cũ đã đạt để chống học vẹt. **"Học xong mục O" = ĐẠT toàn bộ 85 câu.**

> 🧭 **Câu thần chú xuyên suốt:** *"Tôi không nhớ để gõ — tôi hiểu để chỉ huy & kiểm tra AI, phần cú pháp/đời phiên bản thì tra cứu (opentelemetry.io, prometheus.io, sre.google/workbook)."*
