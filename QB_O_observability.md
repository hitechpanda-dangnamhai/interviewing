# 🔭 QB_O — Ngân hàng câu hỏi: Observability (Logging · Metrics · Tracing · SLO)

> **Mục O** trong roadmap Tech Lead Backend (🟢 ~25% — ít hỏi trực tiếp, nhưng lên TL/SRE-adjacent thì vặn sâu; và là "ngôn ngữ" để bàn vận hành trong system design) · Sinh theo **WORKFLOW 2 — Bước A**.
> Đây là **mục LỚN** → bộ đề phủ HẾT 5 mục con của roadmap (Structured logging · Metrics & alerting Prometheus/Grafana · Distributed tracing OpenTelemetry/Jaeger/Tempo · Log/error stack ELK vs Loki / Sentry · SLO/SLI/error budget) + lớp **TL-level** (build-vs-buy, chi phí observability, OTel làm chuẩn, observability-driven dev, on-call, blameless postmortem, privacy) mà phỏng vấn Tech Lead hay vặn.
> **Chỉ câu hỏi — KHÔNG kèm đáp án.** Mỗi câu có dòng *"dò cái gì"* = tiêu chí ĐẠT, dùng để chấm **live** ở Bước D.
> **Tổng: 85 câu.**

## 🚧 Chống trùng (đã đọc `QB_K_messaging.md`, `QB_M_security.md`, `QB_J_caching.md`, `QB_I_concurrency.md`, `QB_L_testing.md`, `QB_N_devops.md`, `QB_H_nestjs.md`, `QB_G_nodejs.md`, `QB_F_apidesign.md`)

Observability đụng nhiều mục — đặc biệt **K-OBS** đã giữ phần *nhập môn tracing cho microservice*. Quy ước phân ranh để **không lặp**:

- **`K-OBS` (Distributed tracing & observability cho microservice)** đã SỞ HỮU: *vì sao debug microservice khó* (K-OBS-001), **định nghĩa trace/span + context propagation qua HTTP & broker** (K-OBS-002), **correlation id vs trace id** (K-OBS-003), **OTel là gì + vendor-neutral, Jaeger/Zipkin/Tempo** (K-OBS-004), **head-based vs tail-based sampling (định nghĩa)** (K-OBS-005), **three pillars trả lời câu hỏi gì + thứ tự khoanh vùng** (K-OBS-006).
  → **O KHÔNG lặp các định nghĩa nền đó.** O đi **sâu hơn về cơ chế + vận hành từng pillar**: span kind/attributes/events/links/baggage/semantic conventions, OTel Collector, *consistency* của sampling & *đặt tail-sampling ở đâu*, critical path. Khi đụng định nghĩa nền → trỏ về K.
- **`G` (Node.js)** sở hữu *cơ chế Node*: `pino`/`winston` & tránh `console.log` đồng bộ hot path (G-318), **AsyncLocalStorage** giữ trace/request context qua async (G-281/282), event loop lag metric (G-78).
  → **O KHÔNG lặp wiring thư viện Node.** O làm phần *thiết kế/chiến lược logging* (field, level, cost, redaction); đụng lib Node → trỏ G.
- **`M` (Security)** sở hữu chiều sâu **PII/threat/compliance** và supply chain. → O chỉ lấy *góc observability* (đừng log secret, redaction đặt ở đâu, privacy của telemetry) rồi trỏ M cho threat model.
- **`F` (API Design)** giữ error response shape: trace id trong error (F-136/152), **business 4xx không alert / bug 5xx alert** (F-160), backpressure tổng quát (F-371). → O làm *thiết kế alert/severity*, cross-ref F.
- **`L` (Testing)** giữ **p95/p99 trong load-test threshold** (L-363) và metric *test-suite* (defect-escape/MTTR/change-failure — L-343/425). → O làm **SLI/SLO production** + percentile cho metric *vận hành*, cross-ref L.
- **`N` (DevOps)** giữ **health probe (liveness/readiness) ở góc primitive K8s** và "HPA cần metric". → O làm *synthetic/black-box monitoring* + agent thu log node-level, trỏ N cho cơ chế pod.
- **`K` (lớp khác)** giữ **consumer lag metric** (K-172) & **DLQ alert** (K-196) ở góc messaging. → O dùng làm *ví dụ* alert trên queue depth nhưng không lặp chi tiết broker.

---

## Cách đọc
- **ID:** `O-<tag>-<số>`. Tag mục con liệt kê ở mục lục dưới.
- **Độ khó:** ⭐ định nghĩa cơ bản → ⭐⭐⭐⭐⭐ phân biệt senior với TL thật (trade-off, vận hành production, chi phí, văn hoá).
- **"Dò cái gì":** điều người trả lời PHẢI chạm tới mới tính ĐẠT (không phải đáp án — chỉ là mốc chấm live ở Bước D).

## Bối cảnh phiên bản & landscape (tính tại 6/2026 — *verify lại khi học*)

> ⚠️ Phần này đổi nhanh → mọi câu chạm tên/đời phiên bản/feature đều nên kiểm chứng tại docs chính thức (opentelemetry.io, prometheus.io, grafana.com, docs.sentry.io) trước khi tin.

- **OpenTelemetry (OTel)** = chuẩn CNCF, tách *instrumentation* khỏi *backend*: API/SDK trong app + **OTLP** (giao thức) + **semantic conventions** (chuẩn tên field) + **Collector** (receivers → processors → exporters). *Traces* stable (từ ~2021); *metrics* data model stable; *logs* là **signal stable trong spec** (impl theo ngôn ngữ đang chín dần); **profiling** = signal thứ 4, RC ~đầu 2026, target GA ~giữa/cuối 2026. *(verify trạng thái từng signal/ngôn ngữ)*
- **W3C Trace Context** (`traceparent`) là chuẩn propagation; span = DAG, mang attributes/events/links.
- **Prometheus 3.0** (11/2024 — major đầu tiên kể từ 2.0/2017): ingest **OTLP metric trực tiếp** (`/api/v1/otlp/...`), **native histogram** (stable ~v3.8, vẫn opt-in khi scrape), UTF-8 label, Remote Write 2.0, UI mới. Prometheus **chỉ là metric store** → trace/log đi backend khác (thường route cả 3 qua OTel Collector). *(verify đời/feature flag)*
- **Grafana stack**: **Loki** (logs) · **Tempo** (traces) · **Mimir** (metrics long-term) · Grafana (visualize) · **Alertmanager** (routing alert). Đối thủ SaaS: **Datadog / New Relic / Honeycomb / Grafana Cloud**. *(verify giá — tính theo host/volume, đắt nhanh ở scale)*
- **Prometheus scale**: 1 node không HA / không long-term → **Thanos / Cortex / Mimir / VictoriaMetrics** cho global view + lưu dài. *(verify)*

---

## Mục lục mục con (ID prefix → số câu)

| Tag | Mục con | Số câu |
|---|---|---|
| **O-FND** | Nền tảng observability (vs monitoring, pillars, RED/USE, golden signals, cardinality, white/black-box) | 8 |
| **O-LOG** | Structured logging (level, field, redaction, correlation id, stdout/12-factor, sampling, cost, audit) | 12 |
| **O-LOGSTACK** | Log aggregation & error tracking (ELK/EFK, Loki, Sentry, agent placement, retention, backpressure, OTel logs) | 8 |
| **O-MET** | Metrics & Prometheus (metric types, histogram/percentile, pull vs push, cardinality, PromQL, recording rules, scale, exemplars, RED dashboard) | 14 |
| **O-ALERT** | Alerting & dashboards (symptom vs cause, alert fatigue, page vs ticket, Alertmanager, runbook, synthetic, đo chất lượng alert) | 10 |
| **O-TRACE** | Distributed tracing sâu (span kind/attributes/events/links, baggage, semantic conventions, instrumentation, Collector, tail sampling, consistency, critical path) | 12 |
| **O-SLO** | SLO/SLI/error budget (định nghĩa, SLI tốt, error budget, burn-rate, percentile SLO, MTTR/MTTD, toil, blameless postmortem) | 11 |
| **O-TL** | TL-level (build vs buy, OTel chuẩn, chi phí, observability-driven dev, on-call, definition-of-done, privacy, review AI/junior) | 10 |

---

## O-FND — Nền tảng observability

**O-FND-001** ⭐⭐
"Observability khác Monitoring ở đâu? Giải thích bằng 'known unknowns' vs 'unknown unknowns'."
> *Dò cái gì:* monitoring = theo dõi tập chỉ số/điều kiện **đã lường trước** (dashboard/alert cho known unknowns); observability = khả năng **đặt câu hỏi mới chưa lường trước** và trả lời từ telemetry để hiểu trạng thái nội tại (unknown unknowns) → cần dữ liệu đủ giàu/high-cardinality. Không chỉ là "có nhiều dashboard".

**O-FND-002** ⭐⭐
"Three pillars (logs/metrics/traces): điểm mạnh, điểm yếu và chi phí của mỗi loại?" *(cái nào trả lời câu hỏi gì + thứ tự khoanh vùng → K-OBS-006)*
> *Dò cái gì:* metrics = số tổng hợp, rẻ, low-cardinality, phát hiện "có bất thường" nhưng mất chi tiết; logs = sự kiện chi tiết giàu context nhưng đắt/khó aggregate; traces = đường đi 1 request qua service, định vị chặng chậm nhưng cần instrument + sampling. Hiểu trade-off **cost ↔ chi tiết**, không coi 3 cái thay thế nhau.

**O-FND-003** ⭐⭐⭐
"RED method vs USE method: mỗi cái đo gì, áp cho đối tượng nào?"
> *Dò cái gì:* **RED** (Rate, Errors, Duration) cho **request-driven service**; **USE** (Utilization, Saturation, Errors) cho **tài nguyên** (CPU/mem/disk/queue); chọn lăng kính theo việc đang theo dõi service hay resource.

**O-FND-004** ⭐⭐⭐
"'Four golden signals' của Google SRE là gì và vì sao đủ để bắt đầu theo dõi 1 service?"
> *Dò cái gì:* latency, traffic, errors, saturation; phủ trải nghiệm user + sức khỏe hệ ở mức tối thiểu; lưu ý latency phải **tách thành công vs thất bại** (lỗi nhanh không phải là "nhanh tốt").

**O-FND-005** ⭐⭐⭐
"Cardinality trong observability là gì, vì sao 'high cardinality' vừa quý vừa nguy hiểm?"
> *Dò cái gì:* cardinality = số tổ hợp giá trị nhãn/field khác nhau; cao = sức mạnh slice/dice điều tra unknown unknowns (hợp log/trace); nhưng với **metric** (mỗi tổ hợp label = 1 time series) cardinality cao (user_id, request_id...) → **nổ số series** → nổ RAM/chi phí TSDB. Phân biệt nơi cao là tài sản (log/trace) vs nơi là quả bom (label metric).

**O-FND-006** ⭐⭐⭐
"White-box vs black-box monitoring khác gì, dùng kết hợp thế nào?" *(health probe K8s → N)*
> *Dò cái gì:* white-box = nhìn nội tại (metric/log/trace app phát ra) → biết "vì sao"; black-box = thăm dò từ ngoài như user (synthetic/uptime) → biết "user có thấy hỏng không"; black-box bắt cả khi white-box chết hoặc lỗi mạng/DNS/TLS; dùng cả hai.

**O-FND-007** ⭐⭐⭐⭐
"Cùng một lỗi prod, vì sao chỉ có log đôi khi vẫn không debug nổi? Khi nào bắt buộc cần trace/metric?"
> *Dò cái gì:* log rời rạc không cho thấy **quan hệ thời gian/nhân quả xuyên service** và **phân bố** (p99); hệ phân tán/đồng thời cao → cần trace (đường đi) + metric (xu hướng/phân vị) để khoanh; nhận ra giới hạn của "thêm log nữa".

**O-FND-008** ⭐⭐⭐⭐
"MELT (Metrics/Events/Logs/Traces) + signal thứ 4 (profiling) và xu hướng hợp nhất qua OpenTelemetry — vì sao 'một chuẩn' lại quan trọng?" *(verify trạng thái signal)*
> *Dò cái gì:* tránh mỗi vendor một SDK/agent → OTel chuẩn hoá instrumentation + OTLP + semantic conventions để telemetry **tương quan** được & đổi backend không phải re-instrument; profiling đang thành signal thứ 4 (RC ~2026 — *verify*). Hiểu giá trị "tách instrumentation khỏi backend".

---

## O-LOG — Structured logging

**O-LOG-001** ⭐⭐
"Structured logging là gì, vì sao JSON-có-field thắng `console.log('user '+id+' failed')` ở production?" *(lib Node pino/winston → G-318)*
> *Dò cái gì:* log thành key-value/JSON có schema → máy **parse/filter/aggregate/correlate** ở scale; chuỗi tự do phải regex mong manh; production cần query "tất cả log user=123 level=error" → bắt buộc có field.

**O-LOG-002** ⭐⭐
"Log levels (trace/debug/info/warn/error/fatal): tiêu chí gán level, và vì sao 'log mọi thứ ở info' là sai?"
> *Dò cái gì:* level phản ánh **mức cần hành động/độ ồn**; debug cho dev, info cho mốc nghiệp vụ, warn cho bất thường tự hồi, error cho hỏng cần điều tra; "info mọi thứ" → noise nuốt tín hiệu + tốn chi phí; level phải **đổi runtime** được.

**O-LOG-003** ⭐⭐⭐
"Cái gì TUYỆT ĐỐI không được log? Cơ chế redaction đặt ở đâu?" *(PII/threat → M)*
> *Dò cái gì:* password/token/PII/số thẻ/secret/Authorization header; redaction/masking ở tầng logger (serializer/redact path) + review; lộ secret trong log = **sự cố bảo mật** (log thường ít kiểm soát quyền hơn DB).

**O-LOG-004** ⭐⭐⭐
"Correlation/request id và trace id nên xuất hiện trong MỌI dòng log của một request thế nào (qua async)?" *(phân biệt 2 id → K-OBS-003; AsyncLocalStorage → G-281)*
> *Dò cái gì:* gắn id vào **logger context per-request** (AsyncLocalStorage/middleware) để mọi log con tự kèm → grep ra cả luồng + nối log↔trace; không truyền tay từng hàm.

**O-LOG-005** ⭐⭐⭐
"Vì sao log nên ghi ra **stdout** (12-factor) thay vì tự quản file/rotate trong app, nhất là trong container?"
> *Dò cái gì:* app coi log là **event stream ra stdout**, để platform (runtime/agent) thu gom/route/rotate; tự ghi file trong container → mất khi pod chết, khó tập trung; tách concern app ↔ infra.

**O-LOG-006** ⭐⭐⭐
"Log sampling/throttling: khi nào cần và rủi ro?"
> *Dò cái gì:* volume cao (hot path, lỗi lặp) → sample/dedupe/rate-limit để khỏi nổ chi phí & che tín hiệu; rủi ro: sample mất đúng dòng cần lúc điều tra → ưu tiên **giữ error**, sample info; cân cost ↔ đủ chứng cứ.

**O-LOG-007** ⭐⭐⭐⭐
"Logging đồng bộ trên hot path hại gì (đặc biệt Node single-thread)? Cách giảm?" *(cross-ref G-318)*
> *Dò cái gì:* I/O log đồng bộ **chặn event loop** → tăng latency/giảm throughput; dùng async/transport tách luồng (pino), tránh `JSON.stringify` object lớn trong vòng nóng; phải **đo** tác động, không phỏng đoán.

**O-LOG-008** ⭐⭐⭐
"'Wide structured events' / canonical log line (một dòng log giàu field cho mỗi request) — ý tưởng và lợi ích so với rải nhiều dòng rời?"
> *Dò cái gì:* gom 1 **canonical log line** giàu field/request (status, latency, user, route, ids...) → 1 event high-cardinality truy vấn được như mini-analytics → giảm noise, tăng khả năng điều tra unknown unknowns.

**O-LOG-009** ⭐⭐⭐
"Vì sao timestamp nên là UTC + ISO-8601 và đồng bộ clock (NTP) quan trọng với log/trace?"
> *Dò cái gì:* so trục thời gian giữa service/host cần chuẩn chung; lệch clock → log/trace lệch nhân quả, khó dựng timeline; UTC tránh nhập nhằng timezone.

**O-LOG-010** ⭐⭐⭐⭐
"Chi phí logging ở scale đến từ đâu (ingest/index/storage/egress) và các đòn bẩy giảm?"
> *Dò cái gì:* phần lớn chi phí ở **index + lưu trữ** theo volume/cardinality; đòn bẩy: giảm verbosity, sample, retention theo level, index có chọn lọc (Loki: index label, không full-text), tier nóng/lạnh, drop field thừa ở Collector. Hiểu log = chi phí thật, không "log vô tận".

**O-LOG-011** ⭐⭐⭐
"Log enrichment đặt ở đâu (app vs agent/Collector)? Cho ví dụ field nên do hạ tầng thêm." *(verify)*
> *Dò cái gì:* field nghiệp vụ do app; field môi trường (pod/node/namespace/region/version) nên do **agent/Collector** tự gắn (vd `k8sattributesprocessor`) → đồng nhất, app không phải biết; đảm bảo **cùng tên field giữa 3 signal** để correlate.

**O-LOG-012** ⭐⭐⭐⭐
"Audit log khác application log thế nào về yêu cầu (bất biến, retention, ai-làm-gì)?" *(compliance/threat → M)*
> *Dò cái gì:* audit = bằng chứng bảo mật/pháp lý → **immutable/append-only**, retention dài theo compliance, nội dung "ai, làm gì, khi nào, lên tài nguyên nào", tách kênh/quyền khỏi log debug; không lẫn với log vận hành.

---

## O-LOGSTACK — Log aggregation & error tracking

**O-LOGSTACK-001** ⭐⭐⭐
"ELK/EFK stack gồm gì, mỗi thành phần vai trò gì?"
> *Dò cái gì:* Elasticsearch (lưu + index + search), Logstash/Fluentd/**Fluent Bit** (thu gom/transform/ship), Kibana (truy vấn/visualize); "EFK" = thay Logstash bằng Fluentd/Fluent Bit nhẹ hơn. Hiểu pipeline **ingest → index → query**.

**O-LOGSTACK-002** ⭐⭐⭐⭐
"Grafana Loki khác Elasticsearch ở mô hình index thế nào, và hệ quả chi phí/độ mạnh truy vấn?"
> *Dò cái gì:* Loki **chỉ index label** (metadata), lưu log nén theo stream, **không full-text index** → rẻ hơn nhiều, hợp volume lớn; đổi lại query full-text yếu hơn ES (lọc theo time/label rồi quét). Chọn theo nhu cầu search vs chi phí; thiết kế label cẩn thận (cardinality).

**O-LOGSTACK-003** ⭐⭐⭐
"Sentry (error tracking) giải bài toán gì mà log/metric thuần không làm tốt?"
> *Dò cái gì:* gom **exception** theo *issue* (grouping/dedup theo fingerprint) + stacktrace/breadcrumb/release/user-impact → nghìn lần cùng lỗi thành 1 issue + alert + **regression theo release**; tập trung "lỗi nào, ảnh hưởng ai, từ release nào", không duyệt log thô.

**O-LOGSTACK-004** ⭐⭐⭐
"Error tracking vs logging vs APM: ranh giới và vì sao thường dùng chung?"
> *Dò cái gì:* error tracking (Sentry) = exception + ngữ cảnh release/user; logging = dòng sự kiện chi tiết; APM/metrics = hiệu năng/latency/throughput; bổ trợ — APM thấy "chậm ở đâu", error tracking thấy "vỡ vì exception nào", log soi chi tiết.

**O-LOGSTACK-005** ⭐⭐⭐⭐
"Đặt agent thu log ở đâu trong K8s: sidecar vs DaemonSet node-level? Trade-off?" *(cơ chế pod → N)*
> *Dò cái gì:* **DaemonSet** (Fluent Bit) 1 agent/node đọc stdout mọi pod → rẻ, ít overhead, chuẩn; **sidecar/pod** khi cần xử lý riêng / log không ra stdout → tốn tài nguyên mỗi pod; mặc định node-level.

**O-LOGSTACK-006** ⭐⭐⭐
"Retention & tiering log: thiết kế thế nào để vừa điều tra được vừa không vỡ chi phí?"
> *Dò cái gì:* retention theo giá trị/level (error giữ lâu, debug ngắn), **nóng** (index, query nhanh, ít ngày) → **lạnh/archive** (S3, rẻ, query chậm); xoá/aggregate theo policy; gắn compliance khi cần (audit → M).

**O-LOGSTACK-007** ⭐⭐⭐⭐
"Backpressure khi log pipeline quá tải / đầu ra chậm: rủi ro và cách xử lý?" *(backpressure tổng quát → F-371)*
> *Dò cái gì:* agent buffer đầy → drop log / chặn app (nếu app block khi ghi) / OOM; cần **buffer có giới hạn + chính sách drop** + tách app khỏi pipeline (app ghi stdout, agent lo phần sau) → log không kéo sập app; **chấp nhận mất log thay vì mất service**.

**O-LOGSTACK-008** ⭐⭐⭐
"OpenTelemetry với logs khác cách thu log truyền thống thế nào (bridge legacy + correlate)?" *(verify trạng thái logs signal)*
> *Dò cái gì:* OTel **không bịa API log mới** mà **bridge** logging lib hiện có, gắn trace context → log tự correlate với trace/metric qua cùng attribute, đẩy qua Collector/OTLP đồng nhất. Giá trị: 3 signal cùng ngữ cảnh, enrich tập trung.

---

## O-MET — Metrics & Prometheus

**O-MET-001** ⭐⭐
"4 loại metric Prometheus (counter, gauge, histogram, summary): mỗi loại dùng cho gì?"
> *Dò cái gì:* counter = chỉ tăng (số request/lỗi) → dùng `rate()`; gauge = lên xuống (mem, queue depth, in-flight); histogram = phân bố vào bucket (latency) → quantile tính phía server; summary = quantile tính ở client. Gán sai loại = metric vô nghĩa.

**O-MET-002** ⭐⭐⭐
"Vì sao đo latency bằng **histogram** thay vì gauge 'latency trung bình'? Liên hệ p95/p99." *(p95 vs avg trong load test → L-363)*
> *Dò cái gì:* trung bình **giấu đuôi** (vài request 5s bị nuốt); histogram giữ phân bố → tính p95/p99 phản ánh trải nghiệm xấu nhất; gauge latency = 1 điểm thời điểm, mất phân bố.

**O-MET-003** ⭐⭐⭐
"Pull (scrape) vs push: vì sao Prometheus chọn pull, và khi nào vẫn cần push (Pushgateway)?"
> *Dò cái gì:* pull = Prometheus tự scrape `/metrics` → biết target sống/chết (`up`), kiểm soát tần suất, dễ service discovery; push hợp **job ngắn/batch** không kịp bị scrape → Pushgateway (lưu ý: không dùng cho liveness, dễ stale). Hiểu pull là mặc định, push là ngoại lệ.

**O-MET-004** ⭐⭐⭐
"Prometheus exposition: app expose metric thế nào, scrape qua đâu, vai trò metric `up`?"
> *Dò cái gì:* app/exporter mở endpoint `/metrics` (text format); Prometheus scrape theo target (static/SD) mỗi interval; metric `up` cho biết target có scrape được không → nền cho alert "service down".

**O-MET-005** ⭐⭐⭐⭐
"'Cardinality explosion' trên Prometheus xảy ra thế nào và vì sao chí mạng? Cho ví dụ label sai."
> *Dò cái gì:* mỗi tổ hợp label = 1 time series; nhét label cao-cardinality (user_id, request_id, email, URL có id) → **bùng nổ series** → nổ RAM/disk/scrape, có thể sập Prometheus; label phải **bounded** (route template, status_code, method). Lỗi sản xuất kinh điển.

**O-MET-006** ⭐⭐⭐
"`rate()` vs `irate()` vs `increase()` trên counter khác gì, vì sao không lấy giá trị counter thô?" *(verify cú pháp PromQL)*
> *Dò cái gì:* counter thô vô nghĩa (chỉ tăng, reset khi restart); `rate()` = tốc độ TB/giây trên cửa sổ (mượt, hợp alert), `irate()` = tức thời 2 điểm cuối (nhạy, đồ thị), `increase()` = tổng tăng trong cửa sổ; đều xử lý counter reset.

**O-MET-007** ⭐⭐⭐⭐
"`histogram_quantile()` tính p99 từ bucket — vì sao kết quả là **xấp xỉ** và bucket boundary đặt sai gây gì?" *(verify native histogram)*
> *Dò cái gì:* quantile **nội suy trong bucket** → độ chính xác phụ thuộc ranh giới; nếu p99 rơi vào bucket quá rộng (vd `+Inf`) → sai lệch lớn; cần bucket phủ vùng quan tâm; native histogram (Prom 3) giảm vấn đề này — *verify*.

**O-MET-008** ⭐⭐⭐
"Recording rules để làm gì, khác alerting rules?"
> *Dò cái gì:* recording rule **precompute** biểu thức tốn kém thành series mới (dashboard/alert query nhanh hơn); alerting rule đánh giá điều kiện → bắn alert; tách tính toán nặng khỏi lúc query.

**O-MET-009** ⭐⭐⭐⭐
"Prometheus single-node có giới hạn gì (HA, long-term storage, scale ngang)? Giải pháp landscape?" *(verify tên/đời)*
> *Dò cái gì:* 1 node = giới hạn dung lượng/giữ ngắn hạn, không HA tự thân; cần long-term + global view → **Thanos / Cortex / Mimir / VictoriaMetrics**; remote-write/federation; HA = 2 replica scrape song song. Prometheus lo *thu thập*, scale lưu trữ là lớp khác.

**O-MET-010** ⭐⭐⭐⭐
"Prometheus 3.0 thêm gì đáng chú ý (OTLP ingestion, native histogram, UTF-8...) và vì sao Prometheus vẫn 'không nhận trace/log'?" *(verify đời/feature)*
> *Dò cái gì:* Prom 3.0 (11/2024) ingest **OTLP metric trực tiếp**, native histogram (stable ~v3.8, vẫn opt-in scrape), UTF-8 label, Remote Write 2.0; nhưng Prometheus **chỉ là metric store** → trace/log đi Tempo/Loki, thường route cả 3 qua OTel Collector. Đừng coi Prometheus là "all-in-one".

**O-MET-011** ⭐⭐⭐
"Exemplars là gì, nối metric ↔ trace thế nào?"
> *Dò cái gì:* exemplar = mẫu **trace id đính kèm điểm dữ liệu metric** (vd 1 request trong bucket latency cao) → từ đồ thị p99 click thẳng sang trace của request chậm cụ thể; cầu nối "thấy bất thường (metric) → soi nguyên nhân (trace)".

**O-MET-012** ⭐⭐⭐⭐
"RED dashboard cho 1 service gồm panel nào, và đọc thế nào khi incident?"
> *Dò cái gì:* Rate (req/s), Errors (error rate/%), Duration (p50/p95/p99); khi nổ: errors↑ hay latency↑? theo route/version nào? rate có spike (traffic)? → khoanh nhanh **trước khi** xuống trace/log. Dashboard phục vụ điều tra, không trang trí.

**O-MET-013** ⭐⭐⭐
"Metric 'business' vs 'system': vì sao TL nên đẩy team đo cả hai? Ví dụ."
> *Dò cái gì:* system (CPU/latency/error) nói hệ chạy ổn; **business** (đơn/giây, signup, payment success rate) nói *giá trị* có chảy không — đôi khi hệ "xanh" nhưng business rớt (nút mua hỏng); cả hai mới thấy tác động thật.

**O-MET-014** ⭐⭐⭐⭐
"'Watermelon metric' (xanh ngoài, đỏ trong) là gì, vì sao đo sai dễ ru ngủ?"
> *Dò cái gì:* dashboard toàn xanh (CPU thấp, uptime cao) nhưng user thực đang khổ (p99 tệ, lỗi ở luồng quan trọng) → đo nhầm thứ **dễ-đo** thay vì thứ **user-cảm-nhận**; phải đo SLI gắn trải nghiệm (latency thành công theo route quan trọng), không chỉ resource.

---

## O-ALERT — Alerting & dashboards

**O-ALERT-001** ⭐⭐⭐
"Alert nên dựa trên **symptom** (user thấy) hay **cause** (nguyên nhân nội tại)? Vì sao?"
> *Dò cái gì:* ưu tiên **symptom-based** (user-facing: error rate/latency/SLO) → ít alert hơn, đúng cái quan trọng, bắt cả lối hỏng mới; cause-based dễ nổ ồn & bỏ sót; cause để **chẩn đoán**, symptom để **page**.

**O-ALERT-002** ⭐⭐⭐⭐
"Alert fatigue: nguyên nhân, hậu quả, và cách thiết kế alert để không bị 'nhờn'?" *(security gate fatigue → M-514)*
> *Dò cái gì:* quá nhiều alert/false positive/không-actionable → kỹ sư phớt lờ → **bỏ lỡ alert thật** (chi phí thật của fatigue); chữa: chỉ page khi actionable + user-impacting, gộp/dedup, ngưỡng theo SLO/burn-rate, mỗi alert có runbook, định kỳ review & xoá alert vô dụng.

**O-ALERT-003** ⭐⭐⭐
"Phân tầng alert: cái gì **page** (gọi dậy 3h sáng) vs cái gì chỉ **ticket/warning**?" *(business 4xx vs 5xx → F-160)*
> *Dò cái gì:* page = đang/sắp vi phạm SLO, cần xử lý ngay; ticket = cần làm nhưng không khẩn (disk 70%, cảnh báo xu hướng); tiêu chí: "có cần con người hành động **NGAY** không?" Nếu không → đừng page.

**O-ALERT-004** ⭐⭐⭐⭐
"Vì sao alert 'CPU > 80%' thường là alert tồi? Đặt lại thế nào?"
> *Dò cái gì:* CPU cao ≠ user đau (có thể đang chạy hiệu quả); là **cause** không **symptom**, dễ false positive/negative; thay bằng alert trên triệu chứng thật ảnh hưởng user (latency/error/saturation), CPU làm tín hiệu **chẩn đoán/capacity** chứ không paging.

**O-ALERT-005** ⭐⭐⭐
"`for:` (duration) trong alert rule để làm gì? Trade-off ngắn/dài?"
> *Dò cái gì:* yêu cầu điều kiện đúng **liên tục N phút** mới bắn → lọc spike thoáng qua (giảm flapping/noise); dài quá → phản ứng chậm với sự cố thật; cân nhạy ↔ ồn theo mức nghiêm trọng.

**O-ALERT-006** ⭐⭐⭐⭐
"Alertmanager lo gì ngoài việc 'bắn alert' (grouping, dedup, silence, inhibition, routing)?"
> *Dò cái gì:* gom alert liên quan thành 1 thông báo (grouping), khử trùng lặp từ nhiều replica (dedup), tắt tạm khi bảo trì (silence), **inhibition** (alert cấp cao chặn alert hệ quả: cluster down → im alert con), route theo team/severity tới kênh (PagerDuty/Slack). Tránh bão thông báo.

**O-ALERT-007** ⭐⭐⭐
"Dashboard 'tốt' cho on-call khác dashboard 'đẹp để demo' thế nào?"
> *Dò cái gì:* on-call cần **ít panel, top-down** (SLO/golden signals trước → drill xuống), trả lời nhanh "có hỏng không, ở đâu"; demo dashboard nhiều panel màu mè gây nhiễu lúc incident; thiết kế theo **câu hỏi điều tra**, không theo "đo được gì nhét nấy".

**O-ALERT-008** ⭐⭐⭐⭐
"Runbook gắn với alert: vì sao mỗi paging alert nên có, và nội dung tối thiểu?"
> *Dò cái gì:* alert nổ 3h sáng → người trực cần biết ngay "nghĩa là gì, kiểm tra gì, làm gì, escalate ai"; thiếu runbook → MTTR cao, phụ thuộc 1 người; tối thiểu: triệu chứng, dashboard/trace liên quan, bước chẩn đoán, cách giảm thiểu.

**O-ALERT-009** ⭐⭐⭐
"Synthetic/black-box alert (probe từ ngoài) bổ sung gì cho alert nội bộ?" *(liveness/readiness pod → N)*
> *Dò cái gì:* probe định kỳ endpoint quan trọng **như user** → bắt cả khi hệ metric chết hoặc lỗi tầng DNS/TLS/LB mà metric nội bộ không thấy; "service nói nó khoẻ" ≠ "user gọi được".

**O-ALERT-010** ⭐⭐⭐⭐
"Đo 'chất lượng hệ thống alert' bằng tín hiệu gì (precision/recall của alert, % actionable, MTTA)?"
> *Dò cái gì:* theo dõi alert nào dẫn tới hành động thật vs bị ignore/ack-rồi-bỏ, tỉ lệ false positive, thời gian ack (MTTA), số page/đêm; coi bộ alert là **sản phẩm cần tinh chỉnh**, cắt cái nhiễu — không để "đống alert mọc dại".

---

## O-TRACE — Distributed tracing (sâu)

**O-TRACE-001** ⭐⭐⭐
"Span 'kind' (server/client/producer/consumer/internal) khác nhau để làm gì?" *(def trace/span nền → K-OBS-002)*
> *Dò cái gì:* span kind cho backend dựng đúng **quan hệ caller/callee** và biên service/queue (client→server, producer→consumer qua broker); thiếu kind → topology/timeline sai.

**O-TRACE-002** ⭐⭐⭐⭐
"Span **attributes** vs span **events** vs span **links**: mỗi cái cho gì?"
> *Dò cái gì:* attributes = key-value mô tả span (http.method, route, status); events = mốc thời điểm trong span (vd "cache miss", exception); links = nối span thuộc **trace khác** (vd 1 batch xử lý nhiều message gốc — fan-in). Dùng đúng loại → trace giàu ngữ cảnh.

**O-TRACE-003** ⭐⭐⭐⭐
"Baggage trong OpenTelemetry là gì, khác trace context thế nào, và bẫy khi dùng?"
> *Dò cái gì:* baggage = key-value **nghiệp vụ** truyền xuyên service cùng context (vd tenant_id) để service sau đọc; khác trace/span id (định danh trace); **bẫy**: nhét dữ liệu nhạy cảm/lớn → rò qua mạng + phình header; dùng tiết kiệm.

**O-TRACE-004** ⭐⭐⭐
"Semantic conventions của OTel giải quyết vấn đề gì?" *(verify tên convention)*
> *Dò cái gì:* chuẩn hoá **tên attribute** cho thao tác chung (`http.*`, `db.*`, `messaging.*`) → mọi instrumentation phát cùng tên field → backend phân tích/đối sánh nhất quán, dashboard/alert tái dùng; mỗi service đặt tên khác → không correlate được.

**O-TRACE-005** ⭐⭐⭐⭐
"Auto-instrumentation (zero-code) vs manual instrumentation: được/mất, khi nào phải manual?" *(verify ngôn ngữ hỗ trợ)*
> *Dò cái gì:* auto (SDK/agent) phủ nhanh framework/HTTP/DB/driver phổ biến, ít sửa code → visibility tức thì; **manual** cần cho span **nghiệp vụ** (logic domain, mốc quan trọng) mà auto không biết; thực tế kết hợp cả hai.

**O-TRACE-006** ⭐⭐⭐⭐
"OpenTelemetry Collector (receivers → processors → exporters) đặt giữa app và backend để làm gì? Agent vs gateway mode?" *(verify)*
> *Dò cái gì:* Collector nhận OTLP, **xử lý** (batch, redact PII, sample, enrich k8sattributes, tail-sampling), export tới backend bất kỳ → app không gắn cứng backend, gánh xử lý ra khỏi app; **agent** (mỗi node/sidecar) gom rồi đẩy lên **gateway** (tập trung tail-sampling/route).

**O-TRACE-007** ⭐⭐⭐⭐⭐
"Tail-based sampling muốn 'giữ mọi trace lỗi/chậm' thì phải đặt ở đâu và tốn gì?" *(head vs tail cơ bản → K-OBS-005)*
> *Dò cái gì:* phải **buffer toàn bộ span của 1 trace** tới khi hoàn tất mới quyết → cần **gateway Collector** thấy đủ trace (mọi span cùng trace phải route về cùng nơi theo trace id), tốn RAM/độ trễ; đổi lại giữ được trace bất thường.

**O-TRACE-008** ⭐⭐⭐⭐
"Sampling phải **consistent** (cùng 1 trace mọi service cùng quyết) — vì sao, và parent-based sampling giải thế nào?"
> *Dò cái gì:* mỗi service tự quyết độc lập → trace **đứt khúc** (service này giữ, service kia bỏ) → vô dụng; **parent-based**: quyết định sample được truyền trong trace context, con theo cha → trace nguyên vẹn hoặc bỏ trọn.

**O-TRACE-009** ⭐⭐⭐
"Trace bị 'đứt' ở hàng đợi/async — nguyên nhân thường gặp và cách nối?" *(propagation qua broker → K-OBS-002)*
> *Dò cái gì:* không **propagate context qua message header** → consumer mở trace mới, mất liên kết; phải đính `traceparent` vào message lúc publish, đọc lại lúc consume (span link/parent).

**O-TRACE-010** ⭐⭐⭐⭐
"Chi phí & rủi ro hiệu năng của tracing (overhead instrument, volume span, PII trong attribute)?"
> *Dò cái gì:* instrument thêm CPU/latency nhỏ nhưng **volume span lớn** → chi phí ingest/lưu → cần sampling; PII lọt vào attribute (URL có token, body) → rò → phải **redact ở Collector**; "trace 100%" không khả thi ở scale.

**O-TRACE-011** ⭐⭐⭐⭐
"Trace giúp tìm bottleneck thế nào trong 1 request fan-out nhiều downstream? Đọc waterfall ra sao?" *(vì sao micro khó debug → K-OBS-001)*
> *Dò cái gì:* span con song song/tuần tự → nhìn waterfall thấy chặng nào chiếm thời gian (DB chậm? downstream X? chờ lock?), phân biệt latency **tuần tự cộng dồn** vs **chờ song song**; định vị trước khi đoán.

**O-TRACE-012** ⭐⭐⭐⭐⭐
"'Critical path' của 1 trace là gì, và vì sao tối ưu span KHÔNG nằm trên critical path là phí công?"
> *Dò cái gì:* critical path = chuỗi span quyết định **tổng latency** (cái mà nhanh hơn thì request nhanh hơn); span song song không nằm trên đường tới hạn dù chậm cũng không kéo dài tổng → phải tối ưu **đúng** đường tới hạn; đọc trace để biết tối ưu chỗ nào *thật sự* giảm latency.

---

## O-SLO — SLO / SLI / Error budget

**O-SLO-001** ⭐⭐⭐
"Phân biệt SLI, SLO, SLA. Quan hệ và ví dụ."
> *Dò cái gì:* **SLI** = số đo thực (vd % request <300ms & thành công); **SLO** = mục tiêu nội bộ trên SLI (vd 99.9%); **SLA** = cam kết với khách + hệ quả (phạt/đền), thường lỏng hơn SLO. Đặt SLO < SLA để có biên an toàn.

**O-SLO-002** ⭐⭐⭐⭐
"SLI 'tốt' định nghĩa thế nào (good events / valid events)? Vì sao 'uptime của server' thường là SLI dở?"
> *Dò cái gì:* SLI nên đo **trải nghiệm user** = tỉ lệ sự kiện tốt / tổng sự kiện hợp lệ (request thành công & đủ nhanh / tổng request); "server up" không phản ánh user (server up vẫn trả 500/chậm); chọn SLI theo **điểm chạm user**.

**O-SLO-003** ⭐⭐⭐⭐
"Error budget là gì và nó **kết nối reliability với tốc độ ship** ra sao?"
> *Dò cái gì:* budget = 100% − SLO (99.9% → 0.1% được phép lỗi); còn budget → team được ship nhanh/mạo hiểm; hết budget → freeze feature, dồn vào reliability; biến reliability thành **ngân sách định lượng** thay vì cãi cảm tính.

**O-SLO-004** ⭐⭐⭐⭐
"'100% reliability' là mục tiêu sai — vì sao?"
> *Dò cái gì:* chi phí tăng phi tuyến gần 100%, user thường không phân biệt 99.99% vs 100% (đã có lỗi mạng/client), và 0 budget = không được ship/đổi gì; mục tiêu là **"đủ tin cậy cho user"**, không tuyệt đối.

**O-SLO-005** ⭐⭐⭐⭐⭐
"Burn-rate alerting (multi-window, multi-burn-rate) khác alert ngưỡng tĩnh thế nào và giải gì?" *(verify công thức)*
> *Dò cái gì:* alert theo **tốc độ tiêu budget** (burn rate) → bắt cả sự cố cháy nhanh (page gấp) lẫn rò rỉ chậm (ticket); multi-window (vd 1h & 5m) giảm false positive + phản ứng đúng tốc độ; thay cho "error>1%" tĩnh vốn hoặc ồn hoặc chậm.

**O-SLO-006** ⭐⭐⭐
"Vì sao SLO latency phải nói rõ percentile + ngưỡng + cửa sổ (vd 'p99 < 300ms trong 30 ngày') chứ không 'nhanh'?"
> *Dò cái gì:* "nhanh" không đo được; cần **percentile** (p99 phản ánh đuôi user xấu), **ngưỡng** cụ thể, **cửa sổ** thời gian để tính tuân thủ + budget; thiếu 1 yếu tố → SLO vô nghĩa/không enforce được.

**O-SLO-007** ⭐⭐⭐⭐
"Chọn target SLO (99.9 vs 99.99) dựa trên gì? Vì sao không 'càng cao càng tốt'?"
> *Dò cái gì:* dựa trên kỳ vọng user/đối thủ/chi phí + **phụ thuộc downstream** (SLO không thể cao hơn dependency); mỗi 'số 9' thêm tốn kém lớn; đặt theo nhu cầu thật, đo được, giữ được.

**O-SLO-008** ⭐⭐⭐
"MTTR, MTTD, MTBF, MTTA: mỗi cái đo gì và cải thiện cái nào thường giá trị nhất?"
> *Dò cái gì:* MTTD (phát hiện), MTTA (ack), MTTR (khắc phục), MTBF (giữa 2 sự cố); ở hệ phân tán, **giảm MTTD/MTTR** (phát hiện/phục hồi nhanh) thường thực tế hơn theo đuổi "không bao giờ hỏng" — observability tốt kéo MTTD/MTTR xuống.

**O-SLO-009** ⭐⭐⭐⭐
"Toil (theo SRE) là gì và vì sao đo/giảm toil liên quan tới observability & automation?"
> *Dò cái gì:* toil = việc lặp tay, thủ công, không tạo giá trị lâu dài, scale tuyến tính theo tải (restart tay, xử lý alert lặp); đo + đặt trần toil → ép automation; alert/observability tốt giảm toil điều tra.

**O-SLO-010** ⭐⭐⭐⭐⭐
"Blameless postmortem & 'error budget policy': là TL anh vận hành thế nào để học từ sự cố mà không đổ lỗi?"
> *Dò cái gì:* postmortem tập trung **hệ thống/quy trình** (vì sao môi trường cho phép lỗi) không cá nhân → người dám báo cáo trung thực; error budget policy = quy ước **trước** "hết budget thì làm gì" (freeze, ưu tiên reliability) → quyết định khách quan, không chính trị.

**O-SLO-011** ⭐⭐⭐⭐
"Vì sao SLO nên **ít** và gắn 'user journey quan trọng' thay vì SLO cho mọi endpoint/metric?"
> *Dò cái gì:* quá nhiều SLO → loãng, không ai hành động; chọn vài luồng giá trị nhất (checkout, login, search) đặt SLO có ý nghĩa kinh doanh; SLO là **công cụ ra quyết định**, không phải bộ sưu tập số.

---

## O-TL — TL-level / cross-cutting

**O-TL-001** ⭐⭐⭐⭐
"Build vs buy observability (tự dựng Prometheus+Grafana+Loki+Jaeger vs Datadog/New Relic/Honeycomb): TL quyết theo gì?" *(verify giá)*
> *Dò cái gì:* cân **engineer-time vận hành stack** (scale, lưu trữ, nâng cấp) vs chi phí SaaS (thường theo host/volume → đắt nhanh ở scale), năng lực team, lock-in, data residency; OSS rẻ license nhưng tốn người; quyết theo bối cảnh, không tín điều.

**O-TL-002** ⭐⭐⭐⭐
"Vì sao OpenTelemetry được khuyên làm 'lớp instrumentation chuẩn' dù chọn backend nào? Lợi ích chiến lược?" *(verify trạng thái signal)*
> *Dò cái gì:* instrument 1 lần theo OTel → đổi/đa backend **không re-instrument** → tránh lock-in ở chỗ đắt nhất (code đã gắn); chuẩn CNCF + semantic conventions thống nhất; tách "thu thập" khỏi "lưu/hiển thị".

**O-TL-003** ⭐⭐⭐⭐⭐
"Chi phí observability có thể vượt cả chi phí chạy app — đòn bẩy kiểm soát mà không 'mù'?"
> *Dò cái gì:* cắt cardinality metric, sample trace/log **thông minh** (giữ lỗi), retention/tiering theo giá trị, drop field thừa ở Collector, đo "cost per signal", review telemetry như review code; giữ đủ tín hiệu điều tra với chi phí kiểm soát — **không tắt bừa rồi mù lúc incident**.

**O-TL-004** ⭐⭐⭐⭐
"'Observability-driven development': đưa observability vào lúc thiết kế thay vì chữa cháy — nghĩa là gì trong thực hành?"
> *Dò cái gì:* coi telemetry là **yêu cầu của feature** (span/metric/log + SLI quan trọng định nghĩa từ đầu), kiểm tra được quan sát ở staging, mỗi service ship kèm dashboard/alert; không "code xong mới nghĩ làm sao debug prod". Văn hoá: bạn build nó, bạn quan sát nó.

**O-TL-005** ⭐⭐⭐⭐
"'You build it, you run it' / on-call cho dev: lợi ích, rủi ro, và TL set up thế nào cho bền vững?"
> *Dò cái gì:* dev trực dịch vụ mình → feedback chất lượng ↔ vận hành, sửa tận gốc; **rủi ro**: burnout, alert fatigue, thiếu kỹ năng ops; cần rotation hợp lý, runbook, alert tinh gọn (actionable), giảm toil, không page rác.

**O-TL-006** ⭐⭐⭐⭐
"Đặt observability vào 'definition of done' và CI/CD thế nào?" *(vị trí gate → N-CICD; metric test suite → L)*
> *Dò cái gì:* feature chưa có log/metric/trace + SLI/alert thì **chưa 'done'**; có thể gate deploy bằng "có dashboard/alert", smoke + SLO check sau deploy; quan sát là một phần **chất lượng**, không tuỳ hứng.

**O-TL-007** ⭐⭐⭐⭐⭐
"Một incident 'mù' (không đủ telemetry để biết vì sao): là TL anh rút ra hành động hệ thống gì sau đó?"
> *Dò cái gì:* postmortem → xác định **"tín hiệu nào thiếu"** khiến MTTD/MTTR cao → bổ sung metric/trace/log/alert **đúng chỗ mù đó** (không log bừa khắp nơi); biến mỗi incident thành cải thiện observability có mục tiêu; tránh lặp.

**O-TL-008** ⭐⭐⭐⭐
"Chuẩn hoá observability across nhiều team/service (golden path): vì sao cần và tránh 'mỗi team một kiểu'?" *(IDP → N-TL-006, R)*
> *Dò cái gì:* thư viện/instrumentation chuẩn + naming (semantic conventions) + dashboard template → telemetry correlate được giữa service, on-call đọc được dashboard service lạ, giảm reinvent; "paved road" qua platform/IDP; vẫn cho lệch khi cần.

**O-TL-009** ⭐⭐⭐⭐
"Privacy/compliance trong observability (PII trong log/trace, data residency, retention): TL phải lo gì?" *(threat/PII → M)*
> *Dò cái gì:* redact PII ở nguồn/Collector, kiểm soát ai xem log (RBAC), retention theo luật (GDPR xoá), data residency (telemetry rời biên/sang SaaS nước khác?), audit tách kênh; observability **không được biến thành lỗ hổng rò dữ liệu**.

**O-TL-010** ⭐⭐⭐⭐⭐
"'AI có thể tự thêm log/metric đúng cú pháp nhưng sai chỗ' — TL kiểm tra gì khi review observability do AI/junior thêm?"
> *Dò cái gì:* cú pháp đúng ≠ quan sát đúng — kiểm: metric có **cardinality bom** (label user_id)? đo **symptom user-facing** hay chỉ resource? latency dùng **histogram + percentile** hay average? log có rò **PII/secret**? alert có **actionable**/runbook hay sẽ thành noise? SLI có gắn **user journey**? *Hiểu để chỉ huy & kiểm tra, không nghiệm thu theo "chạy không lỗi".*

---

## Ghi chú dùng bộ đề (cho Bước B–E)

- **Dựng giáo trình ngược (Bước B):** gom cụm theo *dependency*: **O-FND (nền/pillars/cardinality) → O-LOG → O-LOGSTACK → O-MET → O-ALERT → O-TRACE → O-SLO → O-TL**. Học khái niệm pillar + cardinality trước; logging/metrics trước alert (alert dựa trên metric); tracing sau (đã hiểu micro qua mục K); SLO/error-budget tổng hợp; TL-level chốt.
- **Trọng tâm theo roadmap:** mục O chỉ ~25% và độ sâu kỳ vọng ở TL backend là *"đủ để bàn vận hành/độ tin cậy trong system design + ra quyết định"* — ưu tiên **O-FND, O-LOG, O-MET, O-ALERT, O-SLO** (đạt vững), còn O-TRACE/O-LOGSTACK đạt mức *"giải thích + trade-off"* (nhiều phần nền tracing đã ở K-OBS), các câu ⭐⭐⭐⭐⭐ là điểm cộng phân biệt với TL/SRE thật.
- **Cross-ref khi chấm:** câu chạm tracing nền (trace/span/propagation/head-tail sampling/three pillars) → đối chiếu **K-OBS**; lib log Node + AsyncLocalStorage → **G**; PII/compliance → **M**; alert theo class lỗi + backpressure → **F**; percentile load-test + metric test-suite → **L**; health probe + agent pod + cost cloud → **N**.
- **Khi chấm live (Bước D):** mọi câu chạm **đời/feature phiên bản** (OTel signal maturity & profiling, Prometheus 3.0/native histogram, công thức burn-rate, tên SaaS/giá, semantic convention) → yêu cầu người học nói rõ *"cái này cần verify ở docs"* thay vì khẳng định số/đời — đúng tinh thần *"hiểu để chỉ huy & kiểm tra, phần còn lại tra cứu"*.
