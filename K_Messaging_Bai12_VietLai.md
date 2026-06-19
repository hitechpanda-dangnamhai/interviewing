# 🎯 BÀI 12 — Distributed Tracing & Observability

**Phủ:** K-OBS-001 → 006

> 💡 **Mục tiêu một câu:** Đây là **bài chốt Mục K**. Khi một request đi xuyên nhiều service + qua broker (Bài 1–11), làm sao **nhìn xuyên toàn hệ để debug**? Đây cũng là **baseline #4 của Saga** (**observability** — Bài 7).

---

## ① Mục tiêu & vị trí trong mạch

Cả Mục K đã dạy bạn *xây* một hệ phân tán. Bài này dạy bạn *nhìn* nó khi nó hỏng.

> 🎯 **Chốt vị trí:** một request giờ đi qua nhiều service và qua broker — không còn một stack trace gọn gàng như monolith. Đây là **baseline #4 của Saga** (Bài 7). Cross-ref **O** (logging/metrics tổng quát).

---

## ② Giảng cơ bản → nâng cao

### (a) Vì sao khó hơn monolith 📘 K-OBS-001

> 🔍 **Ví dụ trực giác:** trong monolith, debug giống đọc *một cuốn nhật ký liền mạch*. Trong microservices, hành trình một request bị *xé thành mảnh* — mỗi service ghi nhật ký riêng, ở máy riêng. Bạn có 10 cuốn nhật ký rời, không biết trang nào nối trang nào.

1 request đi qua **nhiều service/queue** → **log rải rác** khắp nơi, **không thấy đường đi end-to-end**.

> 🎯 **Distributed tracing** gắn một **trace id** xuyên suốt + chia thành **span** mỗi chặng → **dựng lại timeline** + tìm chặng **chậm/lỗi**. Không có nó = *"mò kim đáy bể"*.

---

### (b) Trace / span / context propagation 📘 K-OBS-002

| Khái niệm | Nghĩa |
|---|---|
| **Trace** | toàn bộ **hành trình** của 1 request |
| **Span** | 1 **đơn vị công việc** (có **parent/child**, thời lượng, **tags**) — **trace = cây span** |
| **Context propagation** | truyền **trace/span id** qua **biên** |

```
Trace = cây span (vd checkout):
  [API request]                     ← root span
    ├─ [Order.create]               ← child span
    │    └─ [DB insert]
    ├─ [publish OrderPlaced→Kafka]
    └─ [Inventory.handle]           ← span NỐI QUA broker (cần propagation!)
         └─ [Payment.charge]  ← chậm 3s ❌  (nhìn cây là thấy ngay chặng nào chậm)
```

**Context propagation qua biên:**
- **HTTP** → **W3C Trace Context** header **`traceparent`** (chuẩn).
- **Message broker** → đính **trace context** vào **message header** (**Kafka headers / AMQP headers**) để nối chặng **async**.

> 🚨 **Thiếu propagation qua broker → trace ĐỨT ở queue** (mỗi bên thành trace riêng). **Đây là chỗ AI/dev hay quên: chỉ instrument HTTP mà bỏ broker.** (🔎 *verify W3C Trace Context*.)

```
❌ KHÔNG propagate qua Kafka:
  [trace A: API→Order→publish] ... đứt ...  [trace B: Inventory→Payment]
  → hai trace RỜI RẠC, không biết B là phần tiếp của A

✅ CÓ propagate (traceparent trong Kafka header):
  [trace A: API→Order→publish→Inventory→Payment]  → MỘT cây liền mạch
```

---

### (c) Correlation ID vs trace id 📘 K-OBS-003

> ⚖️ **Hai cái bổ trợ nhau, KHÔNG thay nhau:**

| | **Correlation/request id** | **Trace id** |
|---|---|---|
| Dùng cho | **logging** | **tracing** |
| Vai trò | gắn vào **mọi log** của cùng luồng nghiệp vụ → **grep** ra toàn hành trình trong log | dựng **cây span** trực quan |

> 💡 Cần **structured logging** (**JSON + field id**) để **truy vấn/aggregate** — *không* phải **log text thuần** (grep text khó aggregate). (cross-ref **O**.)

---

### (d) OpenTelemetry 📘 K-OBS-004

> 🔌 **Ẩn dụ — ổ cắm chuẩn:** trước đây *mỗi vendor một SDK* giống mỗi nước một loại ổ điện — đổi backend phải thay cả dây. **OTel** là *ổ cắm chuẩn quốc tế*: cắm một lần, đổi backend tuỳ ý.

**OTel** (**OpenTelemetry**) = chuẩn + **SDK trung lập** (**CNCF**) để sinh/truyền **trace–metric–log**, **export** tới **backend bất kỳ** (**Jaeger/Tempo/Zipkin/Datadog**...) → tránh **khoá vendor** (*"mỗi vendor một SDK"*).

> 🎯 **OTel tách instrumentation khỏi backend.** **Jaeger/Zipkin/Tempo** = backend **lưu/hiển thị** trace; **OTel** = lớp **sinh/truyền** trace. (🔎 *verify*.)

---

### (e) Sampling 📘 K-OBS-005

> 💰 **Vì sao cần:** **trace 100%** = chi phí **lưu/băng thông** lớn → phải **sample** (lấy mẫu).

| Kiểu | Quyết định khi | Được | Mất |
|---|---|---|---|
| **Head-based** | **ngay đầu** request | **rẻ, đơn giản** | có thể **bỏ lỡ trace lỗi** (vì chưa biết nó sẽ lỗi) |
| **Tail-based** | **sau khi thấy cả trace** | **giữ được trace lỗi/chậm** | **tốn buffer** (giữ span tới khi trace xong) |

> 🎯 **Trade-off: cost ↔ giữ được trace quan trọng.** (🔎 *verify*.)

---

### (f) Three pillars 📘 K-OBS-006

> 🧠 **Mental model — ba trụ, mỗi trụ trả lời một câu khác:**

| Pillar | Trả lời câu hỏi | Đặc điểm |
|---|---|---|
| **Metrics** | *"có gì bất thường, ở đâu"* (**rate/error/latency — RED**; hoặc **USE** cho tài nguyên) | **rẻ, aggregate, dùng để alert** |
| **Traces** | *"request đi đâu chậm/lỗi"* | khoanh **service/chặng** nào |
| **Logs** | *"chi tiết vì sao"* tại chặng đó | chi tiết nhất, đắt nhất |

```
QUY TRÌNH khi alert nổ (thứ tự không đổi):
  Metric phát hiện ──► Trace khoanh vùng ──► Log soi chi tiết
  "error rate vọt"     "chặng Payment chậm"   "timeout connect DB X"
```

> 🎯 **Hiểu vai trò từng pillar, KHÔNG lẫn lộn.** Dùng log để "phát hiện bất thường toàn hệ" là sai trụ (đắt + chậm); đó là việc của metrics.

---

## ③ ⚠️ Cũ → Mới

| Cái cũ | Nay | Vì sao |
|---|---|---|
| **Log text rải rác**, grep từng service | **Structured logging + correlation id + tracing** | Dựng được hành trình **end-to-end** |
| **Mỗi vendor một SDK** instrumentation | **OpenTelemetry** (trung lập) export tới backend bất kỳ | Tránh **vendor lock-in** (🔎 verify) |
| **Chỉ trace HTTP** | **Propagate context qua broker** (message header) | Trace **không đứt** ở queue |
| **Trace 100%** | **Sampling** (head/tail tuỳ nhu cầu) | Cân **chi phí ↔ giữ trace lỗi** |

---

## ④ Áp dụng thực tế + bigtech

**Use case:** request checkout đi **API → Order → (Kafka) → Inventory → (Kafka) → Payment**. **Inject `traceparent`** vào **HTTP** *và* **Kafka headers** → một **trace liền mạch** trong **Jaeger/Tempo** cho thấy chặng **Payment chậm 3s**.

```
TRACE LIỀN MẠCH trong Jaeger:
  API          ▕████▏ 20ms
  Order        ▕██▏ 10ms
  →Kafka       ▕▏
  Inventory    ▕███▏ 15ms
  →Kafka       ▕▏
  Payment      ▕████████████████████████▏ 3000ms ❌ ← thủ phạm hiện rõ
```

**Bigtech:** **OTel** là chuẩn **de-facto**, hầu hết hệ lớn instrument bằng OTel rồi export tới backend (**Tempo/Jaeger/Datadog**); **tail-based sampling** dùng khi cần **chắc chắn bắt trace lỗi**. (pattern; 🔎 *verify*.)

---

## ⑤ Code thực hành + cấu hình

**Propagate trace context qua Kafka** (ý niệm OTel JS — 🔎 *verify API theo đời SDK*):

```javascript
// producer: inject context hiện tại vào message headers (để trace KHÔNG đứt qua broker)
const { propagation, context, trace } = require('@opentelemetry/api'); // verify đời
const headers = {};
propagation.inject(context.active(), headers, {                 // bơm traceparent vào headers
  set: (carrier, k, v) => { carrier[k] = v; },
});
await producer.send({ topic: 'orders', messages: [{ key: order.id, value: JSON.stringify(evt), headers }] });
```

```javascript
// consumer: extract context từ headers -> span con NỐI ĐÚNG trace cha
await consumer.run({ eachMessage: async ({ message }) => {
  const parentCtx = propagation.extract(context.active(), message.headers, {
    get: (carrier, k) => carrier[k]?.toString(),
  });
  // startSpan với parentCtx -> span này thành CON của trace bắt đầu từ HTTP request gốc
  const span = trace.getTracer('inventory').startSpan('handle OrderPlaced', {}, parentCtx);
  try { await handle(JSON.parse(message.value)); }
  finally { span.end(); } // span này nối vào trace bắt đầu từ HTTP request gốc
}});
```

```bash
# requirements/deps (verify đời mới nhất):
#   @opentelemetry/api, @opentelemetry/sdk-node, @opentelemetry/auto-instrumentations-node
#   Backend: Jaeger / Grafana Tempo. Log: structured JSON + correlation id.
```

> 🚨 **AI hay sai:** chỉ **auto-instrument HTTP** rồi tưởng *"có tracing"* — nhưng **trace đứt ở Kafka** vì không **inject/extract** qua message header. Và **log không gắn correlation/trace id** → không nối log với trace được.

---

## ⑥ Keywords

**🧠 Cần HIỂU:**
- *vì sao* tracing cần ở microservice.
- **trace/span/propagation** (HTTP & broker).
- **3 pillars** + thứ tự khoanh vùng.
- **OTel** tách **instrumentation/backend**.
- **sampling head vs tail**.

**📌 Cần THUỘC:**
- **`traceparent`** (**W3C**).
- **trace id** vs **correlation id**.
- **metrics → trace → log**.
- **head-based** (rẻ, lỡ lỗi) vs **tail-based** (giữ lỗi, tốn buffer).

**🛠️ Cần LÀM ĐƯỢC:**
- **propagate context** qua broker (**inject/extract**).
- **structured logging + correlation id**.
- chọn **sampling**.

---

## ⑦ Mental model

> 🧠 **"Trace id xuyên suốt + span mỗi chặng = bản đồ một request.**
> **Đừng để trace đứt ở queue (propagate qua header).**
> **Metric phát hiện → trace khoanh vùng → log soi chi tiết.**
> **OTel để khỏi khoá vendor."**

---

## ⑧ Phỏng vấn & thách đố

**(TB)** Vì sao debug request ở **microservices** khó hơn **monolith**? **Tracing** giải gì?
→ request rải qua nhiều service/queue, log rời rạc; tracing nối bằng trace id + span dựng timeline.

**(khó)** **Trace/span/context propagation** — context truyền qua **HTTP** và **broker** thế nào?
→ HTTP qua header `traceparent`; broker qua message header (Kafka/AMQP); thiếu broker → trace đứt.

**(TB)** **Correlation id** khác **trace id**? Vì sao cần **structured logging**?
→ correlation id cho log, trace id cho tracing, bổ trợ nhau; structured JSON để truy vấn/aggregate.

**(khó)** **OpenTelemetry** giải bài *"mỗi vendor một SDK"* thế nào? Quan hệ **Jaeger/Tempo**?
→ OTel chuẩn trung lập tách instrumentation khỏi backend; Jaeger/Tempo là backend lưu/hiển thị.

**(khó)** **Head-based** vs **tail-based** sampling đánh đổi gì?
→ head rẻ nhưng có thể lỡ trace lỗi; tail giữ trace lỗi nhưng tốn buffer.

> 🧩 **Thách đố:** *"Đã cài **OTel auto-instrumentation**, dashboard đẹp, nhưng mọi trace 'kết thúc' ở chỗ **publish Kafka** và service consumer có **trace riêng rời rạc**. Vì sao và sửa?"*
> → **không propagate trace context qua message header** → consumer tạo **trace mới** thay vì **span con**. **Sửa:** **inject `traceparent`** vào **Kafka headers** ở producer, **extract + startSpan** với **`parentCtx`** ở consumer.

---

## ⑨ Bài tập + tiêu chí

**Đề:** Thiết kế observability cho luồng **API → Order → Kafka → Inventory**: nêu **trace propagation**, **correlation id** trong log, **1 metric + 1 alert**, và **quy trình khoanh vùng** khi alert nổ.

> ✅ **Đạt khi:**
> - **inject/extract context** qua **Kafka header** (trace liền mạch).
> - log **JSON** có **trace/correlation id**.
> - metric **RED** (vd **error rate / p99 latency**) + **alert** ngưỡng.
> - quy trình **metric → trace → log**.
> - (tuỳ chọn) nêu **sampling**.

---

## ⑩ Đọc thêm

- **`opentelemetry.io`**; **`w3.org/TR/trace-context`**; **`grafana.com/oss/tempo`**; **`jaegertracing.io`**.
- **Distributed Systems Observability** (Cindy Sridharan). (🔎 *verify*.)

> 🎯 **Một câu mang về (chốt cả Mục K):** *Xây hệ phân tán mà không có **tracing + structured logging + metrics** là lái xe ban đêm tắt đèn. **Trace id xuyên suốt — đừng để nó đứt ở queue** — chính là ngọn đèn pha soi đường khi sự cố ập đến.*
