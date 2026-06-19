# 🎯 BÀI 13 — Case study: Chat real-time

**Phủ:** A-CHAT-001 → A-CHAT-005

> 💡 **Tư tưởng cốt lõi của cả bài:** Chat là case study đầu tiên về **stateful connection** — khác hẳn mọi bài **stateless** trước. Điểm mấu chốt: **connection là một loại state phải quản lý**, và scale nó khó hơn scale app server nhiều.

---

## ① Mục tiêu & vị trí trong mạch

Case study về **stateful connection** — ráp **Bài 3** (scale stateful khó hơn stateless), **Bài 4** (Redis pub/sub + registry), **Bài 11** (group fan-out), giao với **mục K** (ordering/at-least-once).

> 🎯 **Mục tiêu:** hiểu **vì sao connection là state phải quản lý** và **scale nó thế nào**. Ở Bài 3 ta học "đẩy state ra ngoài để instance stateless" — nhưng ở chat, bản thân **đường kết nối** là state không đẩy đi đâu được, nên phải có cách quản lý riêng.

---

## ② Giảng cơ bản → nâng cao

### (a) Trực giác

> **Ẩn dụ:** Request-response thường như **gửi thư rồi chờ thư trả lời**. Chat thì khác — server phải **chủ động gõ cửa** client bất cứ lúc nào có tin mới.
> - **Polling** (client hỏi liên tục *"có tin mới không?"*) như cứ vài giây lại ra hòm thư xem có thư chưa — **tốn công và trễ**.
> - **WebSocket** như lắp sẵn một **đường ống mở 2 chiều** giữa hai nhà — tin tới là chảy qua ngay. Nhưng đường ống này **dính vào một server cụ thể** (**stateful**), gây khó khi scale.

---

### (b) Cơ chế

#### 📘 **A-CHAT-001 — WebSocket vs polling**

| Tiêu chí | **Polling / long-poll** | **WebSocket** |
|---|---|---|
| Chiều | client hỏi, server trả | **2 chiều**, bền |
| Realtime | trễ (theo chu kỳ hỏi) | **push** tức thì |
| Overhead | cao (mở/đóng liên tục) | **ít hơn** |
| Cái giá | (đơn giản, stateless) | **connection stateful** ⇒ khó scale/LB, cần **connection registry** |

#### 📘 **A-CHAT-002 — Scale tầng connection (triệu WebSocket đồng thời)**

> **Ví dụ trực giác:** Một tổng đài triệu cuộc gọi không thể dồn vào một bàn trực. Cần nhiều bàn (**gateway**), và một **sổ địa chỉ** ghi *"khách X đang nối máy ở bàn nào"* để chuyển cuộc đúng chỗ.

- nhiều **gateway** giữ connection.
- **map user→server** (**registry / pub-sub** qua **Redis**).
- **route message** tới đúng server **đang giữ connection** của người nhận.

> 🎯 **Chốt:** **Connection là state phải quản lý** (khác **stateless app server** ở Bài 3 — nơi instance nào cũng phục vụ được request bất kỳ).

#### 📘 **A-CHAT-003 — Lưu & phân phối message (thứ tự + không mất)**

- **persist trước khi ack** (ghi DB rồi **mới** báo "đã gửi").
- **sequence/timestamp** cho **ordering** trong room.
- **offline** ⇒ lưu rồi push khi online.
- **at-least-once + dedup** (chấp nhận gửi lặp, khử trùng bằng **message-id**).

```
✅ PERSIST TRƯỚC ACK:
T1  nhận message
T2  ghi vào DB (persist)        ← bền vững trước
T3  rồi mới ack "đã gửi"        ✅ crash sau T2 vẫn còn tin

❌ ACK TRƯỚC PERSIST:
T1  nhận message
T2  ack "đã gửi" ngay
T3  CRASH trước khi ghi DB      ❌ tin BIẾN MẤT nhưng user tưởng đã gửi
```

> 🔍 **Vì sao at-least-once kèm dedup?** Trong hệ phân tán, đảm bảo "đúng một lần" rất khó. Dễ hơn là **giao ít nhất một lần** (at-least-once) — chấp nhận đôi khi giao lặp — rồi **khử trùng** bằng **message-id** ở phía nhận. Thà thừa rồi lọc, còn hơn thiếu mà mất tin.

#### 📘 **A-CHAT-004 — Presence (online/typing)**

> **Ví dụ trực giác:** Cái đèn "đang online" của mỗi người — nếu mỗi lần ai đổi trạng thái lại báo cho *tất cả* bạn bè, thì triệu người nhấp nháy = bão thông báo.

**Presence tốn kém** vì cập nhật **liên tục** + **fan-out** tới nhiều người. Làm nhẹ:

- **heartbeat + TTL** trong Redis (không nhận heartbeat ⇒ coi **offline**).
- **throttle** cập nhật.
- chấp nhận **trễ/xấp xỉ** (presence sai vài giây vô hại).

#### 📘 **A-CHAT-005 — Group chat vs 1-1**

> 1 message group phải tới **N thành viên** (**fan-out** như feed). Nhóm **rất lớn** ⇒ cân nhắc **pull / giới hạn**; vẫn cần **ordering theo room**.

---

## ③ ⚠️ Cũ → mới / sai lầm hay gặp

| Hay gặp (❌) | ✅ Nên làm | Vì sao |
|---|---|---|
| **Polling** để lấy tin mới | **WebSocket** (push 2 chiều) | Polling trễ + tốn; **push** realtime hiệu quả hơn |
| Coi **WebSocket server** như **stateless** | **Connection registry** (user→server) | Connection **dính server cụ thể**, phải route đúng |
| **Ack trước khi persist** | **Persist trước rồi ack** | Ack sớm ⇒ **mất tin** nếu crash trước khi ghi |
| **Broadcast presence** mỗi thay đổi | **Heartbeat+TTL + throttle** | Presence fan-out liên tục ⇒ quá tải |

---

## ④ Áp dụng thực tế + So sánh bigtech

Pattern phổ biến cho chat (**WhatsApp/Slack/Messenger**):

- tầng **connection gateway stateful** tách khỏi tầng **business logic stateless**.
- **Redis pub/sub** (hoặc **message broker**) làm *"bảng tin trung gian"* để **server A gửi message tới user đang nối ở server B**.
- **message persist** ở store bền (**Cassandra-like** cho write-heavy + **ordering theo room**).
- **Presence** dùng **heartbeat+TTL**.

> ⚠️ (**verify** chi tiết hãng.)

---

## ⑤ Code thực hành + cấu hình

**Routing message** qua **connection registry + Redis pub/sub** (Node.js, minh hoạ):

```javascript
// chat_gateway.js — mỗi gateway giữ 1 phần WebSocket; Redis làm registry + pub/sub
// npm i ws ioredis
const WebSocket = require('ws');
const Redis = require('ioredis');
const pub = new Redis(process.env.REDIS_URL);
const sub = new Redis(process.env.REDIS_URL);
const GATEWAY_ID = process.env.HOSTNAME;
const local = new Map();                       // userId -> ws (connection LOCAL của gateway này)

const wss = new WebSocket.Server({ port: 8080 });
wss.on('connection', (ws, req) => {
  const userId = authenticate(req);            // xác thực
  local.set(userId, ws);
  pub.set(`conn:${userId}`, GATEWAY_ID, 'EX', 60);   // REGISTRY: user này đang ở gateway nào (TTL=heartbeat)
  ws.on('message', (raw) => routeMessage(JSON.parse(raw)));
  ws.on('pong', () => pub.expire(`conn:${userId}`, 60));  // heartbeat gia hạn presence (A-CHAT-004)
  ws.on('close', () => { local.delete(userId); pub.del(`conn:${userId}`); });
});

async function routeMessage(msg) {
  await persistMessage(msg);                   // PERSIST TRƯỚC khi ack (A-CHAT-003)
  for (const to of recipients(msg)) {          // 1-1 hoặc fan-out group (A-CHAT-005)
    const gw = await pub.get(`conn:${to}`);    // người nhận đang ở gateway nào?
    if (gw === GATEWAY_ID) deliverLocal(to, msg);          // cùng gateway -> đẩy thẳng
    else if (gw) pub.publish(`gw:${gw}`, JSON.stringify({ to, msg }));  // khác gateway -> qua pub/sub
    else queueOffline(to, msg);                // offline -> lưu, push khi online
  }
}
sub.subscribe(`gw:${GATEWAY_ID}`);             // nhận message gửi tới user nối ở gateway này
sub.on('message', (_ch, raw) => { const { to, msg } = JSON.parse(raw); deliverLocal(to, msg); });
```

```text
# Cấu hình & scale:
- Connection gateway tách khỏi logic; LB hỗ trợ WebSocket (sticky theo connection là chấp nhận được ở đây).
- Registry user->gateway trong Redis với TTL = chu kỳ heartbeat; hết hạn = offline.
- Message: persist trước ack; message-id để dedup (at-least-once); sequence/room cho ordering.
- Group lớn: cân nhắc pull cho thành viên ít hoạt động; throttle presence/typing.
```

> 🔎 **Vì sao ở đây sticky session lại chấp nhận được, dù Bài 3 bảo "tránh"?** Vì một **WebSocket** *bản chất* là một đường ống gắn cứng vào một gateway — không thể "đẩy state ra ngoài" như session HTTP. Cái ta đẩy ra ngoài (vào Redis) là **bản đồ user→gateway**, chứ không phải bản thân connection. Sticky ở tầng connection là **đặc thù không tránh được** của bài toán stateful này.

---

## ⑥ Keywords cần nhớ

🧠 **Cần HIỂU:**
- vì sao **WebSocket** hơn **polling**.
- vì sao **connection là state khó scale**.
- vì sao **persist trước ack**.
- vì sao **presence tốn kém**.

📌 **Cần THUỘC:**
- **connection registry** (user→server).
- **at-least-once + dedup**.
- **sequence/ordering theo room**.
- **heartbeat+TTL** cho presence.

🛠️ **Cần LÀM ĐƯỢC:**
- route message **xuyên gateway** qua **pub/sub**.
- thiết kế **presence nhẹ**.
- xử **group fan-out**.

---

## ⑦ Mental model

> 🎯 **"Đường ống mở sẵn, dính server — nên cần sổ địa chỉ (registry) để gửi đúng chỗ."**
> **Persist trước ack; presence bằng heartbeat+TTL; group = fan-out.**

---

## ⑧ Câu hỏi phỏng vấn & thách đố

**(TB)** Vì sao chat dùng **WebSocket** thay **polling**, đổi gì?
→ **2 chiều/push/ít overhead**; nhưng **stateful**, khó scale, cần **registry**.

**(khó)** Triệu **WebSocket** đồng thời scale sao?
→ nhiều **gateway** + **registry user→server** (**Redis pub/sub**) + **route đúng server**.

**(TB)** Đảm bảo **thứ tự + không mất tin**?
→ **persist trước ack**, **sequence/timestamp theo room**, offline lưu+push, **at-least-once+dedup**.

**(khó)** **Presence** tốn kém vì sao, làm nhẹ sao?
→ cập nhật liên tục + **fan-out**; **heartbeat+TTL**, **throttle**, chấp nhận **xấp xỉ**.

**(TB)** **Group chat** khác **1-1** ở đâu?
→ 1 message tới N (**fan-out**); nhóm lớn cân nhắc **pull**; vẫn **ordering theo room**.

> 🧩 **Thách đố (trick):** Server A nhận message gửi cho user X, nhưng X đang nối ở server B. A gửi thẳng cho X được không?
>
> ```
> ❌ Quên connection stateful:
> A nhận msg cho X  →  A KHÔNG giữ socket của X (X nối ở B)  →  gửi thẳng = thất bại
>
> ✅ Đúng:
> A tra REGISTRY → "X ở gateway B"
>        ↓
> A publish qua pub/sub tới B → B đẩy xuống socket của X  ✅
> ```
> 🔎 **Vì sao là bẫy?** **A không giữ socket của X**. Phải **tra registry** (X ở server B) rồi **route qua pub/sub** tới B; B mới đẩy xuống socket của X. Đây chính là lý do cần **registry + pub/sub**.

---

## ⑨ Bài tập + tiêu chí tự chấm

**BT1.** Thiết kế chat **10M concurrent connection**: bố trí **gateway**, **registry**, **message store**, đảm bảo **ordering + không mất tin**.

> ✅ **Đạt khi:** tách **gateway/logic**, **registry user→server**, **persist-trước-ack**, **dedup**, **ordering theo room**.

**BT2.** Thiết kế **presence** cho **10M user** không làm sập hệ.

> ✅ **Đạt khi:** **heartbeat+TTL Redis**, **throttle**, chấp nhận trễ; **không broadcast mỗi thay đổi**.

---

## ⑩ Đọc thêm

- **System Design Interview Vol 2** (Alex Xu) — *"Design a Chat System"*.
- **WebSocket RFC 6455**.
- **Redis pub/sub docs**.

---

> 🎯 **Khẩu quyết khép bài:** *Connection là state — cần registry để gửi đúng gateway. Persist trước, ack sau. At-least-once + dedup để không mất tin. Presence bằng heartbeat+TTL, đừng broadcast mỗi thay đổi.*
