# 🎯 BÀI 3 — Horizontal scaling & Load balancing

**Phủ:** A-LB-001 → A-LB-007

> 💡 **Tư tưởng cốt lõi của cả bài:** Cách mở rộng cơ bản nhất của mọi hệ phân tán là **thêm máy** (**horizontal scaling**) thay vì **mua máy to hơn** (**vertical scaling**). Nhưng để thêm máy được tự do, hệ phải **stateless** và phải có người điều phối — **load balancer (LB)**.

---

## ① Mục tiêu & vị trí trong mạch

Bài này **mở khối "Scaling primitives"** (các viên gạch nền của việc mở rộng).

> 🎯 **Vì sao đây là viên gạch nền tảng nhất?** Vì mọi **case study** ở **Bài 9–15** đều bắt đầu từ đúng một câu: *"đặt LB trước một cụm stateless service"*. Nắm chắc bài này, bạn có sẵn câu mở đầu cho mọi bài thiết kế.

> 🎯 **Mạch nối:**
> - → **Bài 5:** chính **LB** lại là một **bottleneck / SPOF** (điểm chết duy nhất) cần xử lý.
> - → **Bài 8:** **serverless** tự lo phần scaling này cho bạn.

---

## ② Giảng cơ bản → nâng cao

### (a) Trực giác

> **Ẩn dụ — căn bếp nhà hàng:**
> - **Vertical scaling** = thuê **một đầu bếp giỏi hơn**. Nhưng đầu bếp giỏi vẫn có **trần** (không ai nấu nhanh vô hạn), và nếu anh ta **ốm** thì cả bếp đóng cửa — đó là **SPOF (Single Point of Failure)**.
> - **Horizontal scaling** = thuê **thêm nhiều đầu bếp**. Mở rộng gần như vô hạn, và một người ốm thì bếp vẫn chạy.
>
> Nhưng nhiều đầu bếp thì cần **một người điều phối order** (= **load balancer**), và các đầu bếp **không được giữ ghi chú riêng trong túi** (= **stateless**) — nếu không, order của khách sẽ "lạc" khi chuyển sang đầu bếp khác.

---

### (b) Cơ chế

#### 📘 **A-LB-001 — Horizontal > vertical**

| Tiêu chí | **Vertical scaling** (máy to hơn) | **Horizontal scaling** (thêm máy) |
|---|---|---|
| Giới hạn | có **trần phần cứng** | mở rộng **gần vô hạn** |
| Chịu lỗi | **SPOF** — chết là sập | **chịu lỗi** — mất 1 node vẫn chạy |
| Nâng cấp | **downtime** khi nâng | thêm/bớt node nóng |
| Cái giá phải trả | (đơn giản) | đòi **stateless** + **LB** + phối hợp |

> 🔍 **Điều tệ nhất nếu chọn sai (cứ scale dọc mãi):** Bạn đụng trần phần cứng vào đúng lúc traffic tăng vọt, và lần nâng cấp cuối cùng ấy lại cần **downtime** — hệ sập ngay lúc đông khách nhất. Horizontal đổi sự phức tạp lấy khả năng *không bao giờ đụng trần*.

#### 📘 **A-LB-002 — Stateless service**

**Stateless** = instance **không giữ session state** trong bộ nhớ của chính nó. **Bất kỳ instance nào cũng phục vụ được request bất kỳ**. **State** (trạng thái phiên) bị **đẩy ra ngoài** — vào **Redis / DB**.

> 💡 **Vì sao stateless là *điều kiện* để scale ngang?** Vì có stateless thì bạn mới **thêm/bớt node tự do**. Nếu instance giữ riêng trạng thái của user A, thì khi node đó chết, phiên của A biến mất — và bạn không thể tùy ý chuyển A sang node khác.

> **Ví dụ trực giác:** Đầu bếp stateless = mọi công thức và đơn hàng đều dán trên bảng chung giữa bếp (Redis), không ai giấu trong đầu. Thay ca bất kỳ lúc nào cũng được.

#### 📘 **A-LB-003 — Sticky session thường nên tránh**

**Sticky session** = **ghim một user vào đúng một instance**. Nghe tiện, nhưng:

- ❌ **cản rebalance** (không phân bố lại tải được).
- ❌ **mất session khi node chết**.
- ❌ **lệch tải** (node "dính" nhiều user nặng thì quá tải).

> 🎯 **Thay bằng:** **external session store** (vd **Redis**). State ở ngoài → instance nào cũng phục vụ được → không cần ghim.

#### 📘 **A-LB-004 — L4 vs L7 LB**

> **Ví dụ trực giác:** **L4** như nhân viên bưu điện chỉ nhìn *địa chỉ trên phong bì* để chuyển; **L7** như thư ký *mở thư đọc nội dung* rồi mới quyết định chuyển cho phòng ban nào.

| Tiêu chí | **L4 LB** | **L7 LB** |
|---|---|---|
| Route theo | **IP / port** | nội dung **HTTP** (**path / header**) |
| "Hiểu" nội dung | không | có |
| Năng lực thêm | (thuần forward) | **TLS termination**, **retry**, sticky theo **cookie** |
| Tốc độ / chi phí | **nhanh**, rẻ | tốn hơn |
| Chọn khi | cần thuần **throughput** | cần **routing thông minh** / **TLS offload** |

#### 📘 **A-LB-005 — Thuật toán LB**

| Thuật toán | Cách chia | Hợp với |
|---|---|---|
| **round-robin** | luân phiên đều từng request | request **đồng đều**, đơn giản |
| **least-connections** | gửi tới node đang ít kết nối nhất | request **không đều thời lượng** |
| **hash theo key** | cùng key → cùng node | giữ **affinity / cache locality** (vd **shard theo user-id**) |

> 🔍 **Vì sao có hash theo key?** Để cùng một user/key luôn rơi vào cùng node → node đó đã có sẵn dữ liệu nóng trong cache (**cache locality**), không phải nạp lại.

#### 📘 **A-LB-006 — Health check & graceful drain**

- **Health check:** LB liên tục "hỏi thăm" từng node; node **unhealthy** thì LB **ngừng route** tới nó.
- **Graceful drain:** khi **scale-in** hoặc **rolling deploy**, phải **drain** (phục vụ nốt) các connection đang dở **trước khi** tắt node.

> 🚨 **Điều tệ nhất nếu quên drain:** Bạn tắt node ngay khi nó còn đang phục vụ → các request dở dang bị cắt giữa chừng → **5xx hàng loạt** đập vào mặt user, đúng lúc bạn tưởng "chỉ deploy nhẹ thôi".

```
Rolling deploy node cũ:
T1  LB đánh dấu node = "draining"  → ngừng gửi request MỚI tới node
       ↓
T2  node phục vụ NỐT các request đang dở   ✅ không ai bị cắt
       ↓
T3  hết request dở → mới tắt node           ✅ deploy sạch, 0 lỗi
```

---

### (c) Mép giới hạn 📘 **A-LB-007**

> 🚨 **Bản thân LB là một SPOF.** Nếu chỉ có một LB, nó chết là kéo sập toàn hệ — dù bạn có 100 node phía sau.

Làm tầng LB **highly available (HA)**: nhiều LB + một trong các cơ chế **DNS round-robin** / **anycast** / **floating IP** / **active-passive failover**.

> 🎯 **Nguyên tắc vàng:** **Một LB chết không được kéo sập toàn hệ.**

> ➕ **Thực tế cloud (cập nhật):** **managed LB** (**ALB / NLB**, **GCLB**…) đã **HA sẵn** ở tầng dưới. Nhưng nguyên lý *"đừng để 1 điểm chết kéo sập"* vẫn phải hiểu để **chỉ huy** thiết kế.

> 💡 **Phân biệt 📘 vs ➕:** toàn bộ cơ chế là kiến thức ổn định (📘). Phần **managed LB** và *"external session store là mặc định hiện nay"* là ➕ cập nhật thực hành.

---

## ③ ⚠️ Kiến thức cũ / hay bị làm sai

| Cũ / hay gặp (❌) | ✅ Nên dùng nay | Vì sao |
|---|---|---|
| **Sticky session** để giữ login | **External session store** (Redis / JWT) | Sticky **cản scale** & **mất session** khi node chết |
| Lưu **state** trong RAM của app instance | Đẩy state ra **Redis / DB**; instance **stateless** | Mới **scale ngang** được |
| **Scale dọc** khi tải tăng | **Scale ngang + stateless** trước | Vertical có **trần** + **downtime** + **SPOF** |
| Một **LB** duy nhất | Cụm **LB** + **DNS / anycast / floating IP** | LB là **SPOF** |

---

## ④ Áp dụng thực tế + So sánh bigtech

Mọi web service production đều theo pattern:

```
DNS ──► (anycast) ──► LB ──► autoscaling group of stateless instances ──► shared state store
```

> ➕ **Pattern phổ biến:**
> - **Cloud LB managed**: **AWS ALB** (L7) / **NLB** (L4), **GCP Cloud Load Balancing**, **Cloudflare LB** — lo **health check** + **drain** + **multi-AZ**.
> - Trên **Kubernetes**, vai trò LB chia thành **Service** (L4) + **Ingress / Gateway** (L7); **session** đẩy ra **Redis**.

> ⚠️ **Lưu ý (verify):** Chi tiết sản phẩm có thể đổi — **verify tên / đời** khi cần.

---

## ⑤ Code thực hành + cấu hình

Minh hoạ **stateless + external session** (Express + Redis). Điểm cốt lõi: **state KHÔNG nằm trong process**.

```javascript
// app.js — chạy N instance sau LB; session ở Redis nên scale ngang tự do
// npm i express express-session connect-redis redis
const express = require('express');
const session = require('express-session');
const { RedisStore } = require('connect-redis');   // kiểm chứng API tại docs connect-redis (đã đổi qua các đời)
const { createClient } = require('redis');

const redisClient = createClient({ url: process.env.REDIS_URL }); // KHÔNG hardcode URL
redisClient.connect();

const app = express();
app.use(session({
  store: new RedisStore({ client: redisClient }),   // <- STATE RA NGOÀI → instance stateless
  secret: process.env.SESSION_SECRET,               // lấy từ env / secret manager, KHÔNG commit
  resave: false, saveUninitialized: false,
  cookie: { httpOnly: true, secure: true, maxAge: 3600_000 },
}));

// LB health check + graceful drain DỰA VÀO endpoint này:
app.get('/healthz', (_req, res) => res.sendStatus(200));

app.get('/me', (req, res) => {
  // đọc/ghi session Ở REDIS, không ở RAM của instance:
  req.session.views = (req.session.views || 0) + 1;
  // servedBy cho thấy request có thể được phục vụ bởi BẤT KỲ instance nào:
  res.json({ views: req.session.views, servedBy: process.env.HOSTNAME });
});
app.listen(3000);
```

```text
# requirements / cấu hình
- ĐẶT REDIS_URL, SESSION_SECRET qua biến môi trường / secret manager (KHÔNG commit).
- LB cấu hình health check trỏ /healthz; bật connection draining khi scale-in / deploy.
- Bật autoscaling theo CPU / QPS; mọi instance phải qua được test "tắt 1 node, request vẫn ok".
```

> 🔎 **Đọc kỹ vì sao:** `servedBy: process.env.HOSTNAME` in ra tên instance phục vụ. Nếu thiết kế đúng stateless, bạn gọi `/me` nhiều lần và thấy **HOSTNAME đổi** mà **số views vẫn liên tục** — bằng chứng state nằm ở Redis chứ không ở RAM instance.

---

## ⑥ Keywords cần nhớ

🧠 **Cần HIỂU:**
- Vì sao **stateless** là *điều kiện* scale ngang.
- **L4 vs L7** — khi nào dùng cái nào.
- Vì sao **sticky session** hại.
- **LB là SPOF**.

📌 **Cần THUỘC:**
- 3 thuật toán LB: **round-robin / least-connections / hash**.
- **health check + graceful drain**.
- Cách **HA tầng LB**: **DNS RR / anycast / floating IP**.

🛠️ **Cần LÀM ĐƯỢC:**
- Đặt **LB + cụm stateless + external session**.
- Cấu hình **health check**.
- Chứng minh **"tắt 1 node hệ vẫn chạy"**.

---

## ⑦ Mental model

> 🎯 **"Stateless + LB = scale ngang."**
> Đẩy **state** ra ngoài; đừng để node nào (**kể cả LB**) là điểm chết duy nhất.

---

## ⑧ Câu hỏi phỏng vấn & thách đố

**(dễ)** Vì sao ưu tiên **scale ngang** hơn dọc?
→ **vertical**: trần + SPOF + downtime; **horizontal**: gần vô hạn + chịu lỗi (cần **stateless + LB**).

**(TB)** **Stateless** nghĩa là gì, sao là điều kiện scale ngang?
→ không giữ **session state**; instance nào cũng phục vụ được; state ở **Redis / DB**.

**(TB)** **L4 vs L7 LB**?
→ **L4** IP/port nhanh; **L7** hiểu HTTP, route **path/header** + **TLS**, tốn hơn.

**(khó)** Vì sao tránh **sticky session**, thay bằng gì?
→ cản **rebalance** / mất session / lệch tải; dùng **external session store**.

**(khó)** **Graceful drain** là gì, vì sao cần khi deploy?
→ **drain** connection đang phục vụ trước khi tắt node ⇒ tránh **5xx** khi **rolling deploy / scale-in**.

> 🧩 **Thách đố (trick):** *"Em dùng round-robin nên tải chia đều rồi."* — sai chỗ nào?
>
> ```
> ❌ Niềm tin:  round-robin chia ĐỀU SỐ request
>
> Thực tế:
> node A:  [nhẹ][nhẹ][nhẹ]            ← 3 request nhẹ
> node B:  [NẶNG][NẶNG][NẶNG]         ← 3 request NẶNG → quá tải dù "cùng số"
>          (round-robin tính số, không tính công sức)
>
> ✅ Sửa:   workload không đều  → dùng LEAST-CONNECTIONS
> ```
> 🔎 **Vì sao là bẫy?** **round-robin** chia đều **số request** chứ **không** đều **công sức**. Request dài–ngắn khác nhau ⇒ node nhận toàn request nặng vẫn quá tải. Workload không đều → dùng **least-connections**.

---

## ⑨ Bài tập + tiêu chí tự chấm

**BT1.** Vẽ kiến trúc tối thiểu cho web app **5 instance**: chọn loại **LB**, **thuật toán**, nơi để **session**, **health check**.

> ✅ **Đạt khi:** **session ngoài process**, có **health check + drain**, giải thích chọn **L4 hay L7**.

**BT2.** Mô tả điều gì xảy ra khi **1 instance chết** giữa lúc đang phục vụ, và khi **LB chết**. Nêu cách phòng cả hai.

> ✅ **Đạt khi:** nhắc **external session** (mất instance không mất session) + **HA tầng LB**.

---

## ⑩ Đọc thêm

- **NGINX docs** — *HTTP Load Balancing*.
- **AWS** — *Elastic Load Balancing* (**ALB** vs **NLB**).
- **Kubernetes docs** — *Service / Ingress / Gateway API*.

---

> 🎯 **Khẩu quyết khép bài:** *Thêm máy, đừng mua máy to. Đẩy state ra ngoài để instance stateless. Drain trước khi tắt. Đừng để LB là điểm chết duy nhất.*
