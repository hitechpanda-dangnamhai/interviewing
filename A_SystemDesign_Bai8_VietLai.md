# 🎯 BÀI 8 — Serverless & edge computing

**Phủ:** A-SVL-001 → A-SVL-005

> 💡 **Tư tưởng cốt lõi của cả bài:** **Serverless (FaaS)** và **edge computing** là **một lựa chọn compute khác** với cụm container/k8s ở **Bài 3** — không phải "phiên bản tốt hơn". Việc của bạn là **chỉ huy quyết định "serverless hay container"**, dựa trên requirement, **không chạy theo hype**.

---

## ① Mục tiêu & vị trí trong mạch

Bài này **khép khối "Đánh đổi & ranh giới"**. Một **lựa chọn compute** khác với cụm **container/k8s** ở **Bài 3**.

> 🎯 **Mục tiêu:** hiểu **khi nào hợp / không hợp** và các **bẫy đặc thù** (**cold start**, **connection**, **stateless**) — để **chỉ huy quyết định**, không chạy theo hype.

---

## ② Giảng cơ bản → nâng cao

### (a) Trực giác

> **Ẩn dụ — taxi vs xe riêng:**
> **Serverless (FaaS)** như **thuê taxi theo cuốc**:
> - không có cuốc thì **không trả tiền** (**scale-to-zero**),
> - lên cuốc đầu hơi **chờ xe tới** (**cold start**),
> - **không hợp chở hàng đường dài liên tục** (tải đều cao thì tự nuôi xe — container — rẻ hơn).
>
> **Edge computing** như **đặt quầy phục vụ ngay cạnh khách** (gần user) cho việc **nhẹ**.

---

### (b) Cơ chế

#### 📘 **A-SVL-001 — Khi nào FaaS hợp**

| Hợp ✅ | Không hợp ❌ |
|---|---|
| tải **gai/không đều** | tải **đều cao** (cost cao hơn container) |
| **event-driven** | **latency cực nhạy** |
| muốn **ít vận hành** (**scale-to-zero**) | **long-running / stateful** |

> ⚠️ **Trade-off cố hữu:** **cold start** + **vendor lock-in** (bị khoá vào một nhà cung cấp).

#### 📘 **A-SVL-002 — Cold start**

> **Ví dụ trực giác:** Cuốc taxi đầu tiên trong ngày — tài xế phải nổ máy, làm nóng xe. Những cuốc sau (xe đang warm) thì đi ngay.

**Cold start** = lần gọi **đầu** / sau **idle** phải **khởi tạo runtime** ⇒ **tăng latency**.

Giảm bằng: **provisioned concurrency** / **keep-warm** / **runtime nhẹ**.

> ➕ **(verify)** — nay còn **SnapStart** (chụp **snapshot** môi trường đã init, giảm cold start ~90%, miễn phí ở runtime hỗ trợ) là lựa chọn **rẻ hơn provisioned concurrency**.

> 🔍 **Vì sao không phải lúc nào cold start cũng đáng lo?** Vấn đề chủ yếu với hệ **p99 latency-sensitive** (user đang chờ). Còn tải **async** (**S3 event**, **cron**, **batch**) thường **chịu được cold start** — chậm thêm vài trăm ms khi khởi tạo cũng chẳng ai thấy.

#### 📘 **A-SVL-003 — Stateless & ephemeral ⇒ hệ quả tới DB connection**

> **Ví dụ trực giác:** Mỗi cuốc taxi (invocation) là một người lạ tới gõ cửa DB đòi một đường dây riêng. 10 cuốc → 10 đường dây. **1000 cuốc đồng thời → 1000 đường dây → DB nghẹt thở.**

Mỗi **invocation** có thể mở **1 connection** ⇒ **bùng nổ connection** đập sập DB.

```
❌ Mỗi function tự mở pool:
spike traffic → 1000 invocation đồng thời
       ↓
mỗi cái mở connection riêng → 1000+ connection
       ↓
vượt max_connections của DB → ❌ DB từ chối / treo

✅ Qua connection proxy:
1000 invocation → đều trỏ vào RDS Proxy
       ↓
proxy dùng CHUNG một pool nhỏ (vd 80-90% DB limit)
       ↓
DB chỉ thấy số connection an toàn → ✅ sống
```

> 💡 **Giải:** dùng **connection proxy** (**verify**: **Amazon RDS Proxy** quản pool dùng chung cho Lambda) hoặc **external store**.
> 🚨 **Cấm:** **KHÔNG** để mỗi function tự mở **pool lớn** (**double pooling**). Thiết kế **khác hẳn** service thường trú.

#### 📘 **A-SVL-004 — Edge computing / CDN compute**

Chạy logic **gần user** (**auth nhẹ**, **A/B test**, **URL rewrite**, **personalization cache**) ⇒ **giảm round-trip tới origin**.

> 🚨 **Cấm:** **KHÔNG** đặt logic **nặng / stateful / data-heavy** ở **edge**.
> 🔍 **Vì sao?** **Edge** giới hạn **CPU / thời gian**, và **xa DB nguồn** — đặt logic data-heavy ở đó thì mỗi lần lại phải gọi ngược về region xa, mất hết lợi ích "gần user".

---

### (c) Khung quyết định 📘 **A-SVL-005**

> 🎯 *"Serverless hay container/k8s?"* — quyết theo các trục:

| Trục | Nghiêng **serverless** | Nghiêng **container** |
|---|---|---|
| **pattern tải** | gai / không đều | đều cao |
| **latency budget** | nới được | cực nhạy |
| **độ phức tạp vận hành** | muốn ít ops | chấp nhận quản |
| **cost ở tải dự kiến** | tải thấp/gai → rẻ | tải đều cao → rẻ hơn |
| **statefulness** | stateless | stateful/long-running |
| **vendor lock-in** | chấp nhận | muốn tránh |

> 🎯 **Chốt:** **Quyết theo requirement, không theo hype.**

> 💡 **Phân biệt 📘 vs ➕:** nguyên lý **FaaS / cold start / edge** là 📘. **SnapStart** và chi tiết **RDS Proxy** là ➕ (đã **verify** tại docs AWS — kiểm chứng **đời/region** khi triển khai).

---

## ③ ⚠️ Kiến thức cũ / hay bị làm sai

| Hay gặp (❌) | ✅ Nên làm | Vì sao |
|---|---|---|
| Mỗi **Lambda** tự mở **connection pool** tới DB | Dùng **RDS Proxy / external pool**; **NullPool** ở app | **Bùng nổ connection / double pooling** đập sập DB |
| *"Serverless luôn rẻ hơn"* | Tính **cost ở tải dự kiến**; tải đều cao ⇒ **container** rẻ hơn | **FaaS** rẻ ở tải **gai**, đắt ở tải **đều cao** liên tục |
| **Provisioned concurrency** cho mọi **cold start** | Cân nhắc **SnapStart** (miễn phí) / chịu cold start nếu **async** | Provisioned concurrency tốn **tiền cố định 24/7** |
| Đặt **business logic nặng** ở **edge** | Edge chỉ **logic nhẹ**; logic nặng ở **origin** | Edge giới hạn **CPU/time**, xa DB |

> 🔍 **Đào sâu — cost crossover:** Có một **điểm giao chi phí (cost crossover)**: dưới ngưỡng tải nào đó, FaaS rẻ hơn (trả theo lượt, không tải thì $0); trên ngưỡng đó, container rẻ hơn (máy chạy gần full-time, đơn giá mỗi request thấp hơn). *"Serverless luôn rẻ"* là sai vì nó bỏ qua điểm giao này.

---

## ④ Áp dụng thực tế + So sánh bigtech

- **FaaS hợp:** **webhook handler**, **ảnh thumbnail on-upload**, **cron/ETL**, **glue giữa service** — tải **gai**, **event-driven** (pattern phổ biến: **AWS Lambda**, **GCP Cloud Functions**, **Azure Functions**, **Cloudflare Workers**).
- **Edge:** **auth token check**, **A/B routing**, **geo-personalization**, **cache ở biên** (**Cloudflare Workers / CDN compute**). Bigtech đặt **logic nhẹ + cache ở edge**, **data + logic nặng ở region** — **verify** chi tiết sản phẩm/đời.

---

## ⑤ Code thực hành + cấu hình

**Lambda + DB qua RDS Proxy** đúng cách (Node.js): khai báo client ở **global scope** nhưng **validate trước khi dùng**; để **proxy lo pool**.

```javascript
// handler.js — Lambda; ĐÃ verify RDS Proxy là cách quản connection cho serverless (docs AWS)
// Quy tắc: KHÔNG mở pool lớn trong từng invocation; trỏ tới RDS Proxy endpoint.
const { Client } = require('pg');                 // pg client đơn (proxy lo pooling)
let client;                                       // GLOBAL SCOPE -> tái dùng giữa các invocation WARM

async function getClient() {
  if (client && client._connected) return client; // validate trước khi dùng (tránh stale conn)
  client = new Client({
    host: process.env.RDS_PROXY_ENDPOINT,         // <- RDS Proxy, KHÔNG nối thẳng DB
    user: process.env.DB_USER,
    database: process.env.DB_NAME,
    // dùng IAM auth token thay vì password tĩnh nếu có thể (an toàn hơn)
    ssl: { ca: process.env.RDS_CA }, password: process.env.DB_PASSWORD,
  });
  await client.connect();
  client._connected = true;
  return client;
}

exports.handler = async (event) => {
  const db = await getClient();                   // tái dùng client warm -> tránh mở connection mới mỗi lần
  const { rows } = await db.query('SELECT id, name FROM users WHERE id = $1', [event.userId]);
  return { statusCode: 200, body: JSON.stringify(rows[0] ?? null) };
};
```

```text
# Cấu hình serverless (checklist):
- DB: dùng RDS Proxy (MaxConnectionsPercent ~80-90%); ở app dùng NullPool/không pool nội bộ.
- Cold start: ưu tiên SnapStart (nếu runtime hỗ trợ) trước khi mua provisioned concurrency.
- Secrets: dùng secret manager + IAM auth, KHÔNG hardcode password.
- Edge: chỉ deploy logic nhẹ (auth check/rewrite/AB); giữ data-heavy ở origin.
- // kiểm chứng tên dịch vụ/đời/region tại docs AWS — có thể đổi.
```

> 🔎 **Vì sao đặt `client` ở global scope?** Vì runtime Lambda **tái dùng môi trường giữa các invocation "warm"**. Biến global sống sót qua nhiều lần gọi → connection được **tái dùng** thay vì mở mới mỗi lượt. Nhưng vẫn phải **validate** (`_connected`) vì một connection cũ có thể đã **stale** (bị đóng phía DB sau idle).

---

## ⑥ Keywords cần nhớ

🧠 **Cần HIỂU:**
- khi nào **FaaS** hợp/không.
- vì sao **serverless** gây **bùng nổ connection**.
- vì sao **edge** chỉ cho **logic nhẹ**.
- **cost crossover** (gai vs đều).

📌 **Cần THUỘC:**
- **cold start** + cách giảm (**provisioned concurrency / SnapStart / keep-warm**).
- **RDS Proxy** cho connection.
- **vendor lock-in**.
- tiêu chí **"serverless vs container"**.

🛠️ **Cần LÀM ĐƯỢC:**
- quyết **serverless hay container** theo requirement.
- cấu hình **Lambda + RDS Proxy** đúng.
- chọn logic đặt ở **edge**.

---

## ⑦ Mental model

> 🎯 **"FaaS = trả theo cuốc, hợp tải gai."**
> **Stateless ⇒ connection là bài toán (proxy); edge chỉ cho logic nhẹ; quyết theo requirement không theo hype.**

---

## ⑧ Câu hỏi phỏng vấn & thách đố

**(TB)** Khi nào **serverless** hợp, khi nào không?
→ hợp tải **gai/event-driven/ít ops**; không hợp tải **đều cao/latency nhạy/long-running/stateful**.

**(TB)** **Cold start** là gì, giảm sao?
→ init runtime lần đầu/sau idle ⇒ ↑latency; **provisioned concurrency / SnapStart / keep-warm / runtime nhẹ**.

**(khó)** **Serverless** ảnh hưởng **DB connection** thế nào?
→ mỗi **invocation** 1 connection ⇒ **bùng nổ**; dùng **RDS Proxy / external pool**.

**(TB)** **Edge computing** đặt logic gì, KHÔNG đặt gì?
→ **auth nhẹ/AB/rewrite/personalization cache**; không **logic nặng/stateful/data-heavy**.

**(khó)** *"Serverless hay k8s?"* quyết theo gì?
→ **pattern tải, latency, ops, cost ở tải dự kiến, statefulness, lock-in**.

> 🧩 **Thách đố (trick):** Lambda của bạn chạy tốt khi test nhưng **sập DB khi traffic spike**. Vì sao?
>
> ```
> ❌ Quên ephemeral connection:
> test (1 invocation)   → 1 connection → OK
>        ↓
> spike (hàng nghìn invocation đồng thời)
>        ↓
> mỗi cái mở connection → vượt max_connections của DB → ❌ DB từ chối / treo
>
> ✅ Fix: RDS Proxy + reserved concurrency (giới hạn số invocation) + NullPool
> ```
> 🔎 **Vì sao là bẫy?** Khi **test** chỉ 1 invocation nên không thấy vấn đề; **spike** mới lộ ra **ephemeral connection** — hàng nghìn invocation đồng thời vượt **max_connections** của DB.

---

## ⑨ Bài tập + tiêu chí tự chấm

**BT1.** Cho 3 workload (**webhook ingest** gai; **API user-facing** tải đều 5k QPS; **cron ETL** đêm). Quyết **serverless hay container** cho từng cái + lý do.

> ✅ **Đạt khi:** webhook → **FaaS**, API tải đều → **container** (cost+latency), ETL → **FaaS** (async chịu cold start).

**BT2.** Liệt kê **3 bẫy production** của serverless và cách phòng.

> ✅ **Đạt khi:** nêu **cold start** (**SnapStart**), **connection blow-up** (**RDS Proxy**), **vendor lock-in** (**abstraction/đa cloud** cân nhắc).

---

## ⑩ Đọc thêm

- **AWS docs** — *Using AWS Lambda with Amazon RDS & RDS Proxy connections*; *Lambda SnapStart*.
- **AWS Well-Architected** — *Serverless Application Lens*. **Cloudflare** — *Workers* (edge compute).

> ⚠️ **Lưu ý (verify):** **verify đời/region**.

---

> 🎯 **Khẩu quyết khép bài:** *FaaS trả theo cuốc — hợp tải gai, đắt khi tải đều. Stateless nên connection phải đi qua proxy. Edge chỉ cho việc nhẹ gần user. Quyết theo requirement, không theo hype.*
