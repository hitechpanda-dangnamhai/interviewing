# 🎯 BÀI 14 — Case study: Notification system đa kênh

**Phủ:** A-NOTI-001 → A-NOTI-004

> 💡 **Tư tưởng cốt lõi của cả bài:** Một hệ notification là **"hệ ống dẫn"** điển hình. Bí quyết là **tách policy** (*gửi cho ai, khi nào*) **khỏi delivery** (*đẩy ra provider thế nào*), và đảm bảo bốn điều: **không trùng, không mất, không spam, không chặn hệ chính**.

---

## ① Mục tiêu & vị trí trong mạch

Case study **queue + worker + reliability** — ráp **Bài 4** (queue), **Bài 5** (retry/DLQ/dedup), **Bài 11** (fan-out), giao với **mục K** (messaging).

> 🎯 **Mục tiêu:** hiểu cách **tách policy khỏi delivery** và đảm bảo **không trùng, không mất, không spam, không chặn hệ chính**.

---

## ② Giảng cơ bản → nâng cao

### (a) Trực giác

> **Ẩn dụ — bưu điện đa kênh:** Notification giống một bưu điện nhận yêu cầu *"gửi tin này cho user"*. Bưu điện phải: **chọn kênh** (email/push/SMS), **gọi nhà cung cấp** (**provider**), **thử lại** nếu hỏng, và **tôn trọng "đừng làm phiền"** (quiet hours) của khách.

> 🎯 **Chìa khóa:** **không gửi trùng, không mất, không spam, không chặn hệ chính.**

---

### (b) Cơ chế

#### 📘 **A-NOTI-001 — Kiến trúc tối thiểu**

```
ingestion → queue → per-channel worker / provider adapter → retry / DLQ
                         │
   tách riêng: template service (render) · user preference · observability (gửi/nhận/bounce)
```

> 🔍 **Vì sao mỗi kênh có adapter riêng?** Vì **API provider khác nhau** — gửi email (SMTP/SendGrid) khác hẳn gửi push (APNs/FCM) khác hẳn gửi SMS (Twilio…). **Provider adapter** giấu sự khác biệt đó sau một interface chung, để phần lõi không cần biết chi tiết từng vendor.

#### 📘 **A-NOTI-002 — Dedup + retry**

- **idempotency key / dedup store**: cùng notification **không gửi 2 lần**.
- provider lỗi ⇒ **exponential backoff** + **DLQ** (**dead-letter queue** cho cái thất bại mãi).
- **at-least-once** nên cần **dedup ở consumer**; theo dõi **delivery status**.

> 🔍 **Vì sao backoff chứ không retry ngay lập tức?** Nếu provider đang quá tải mà bạn **retry dồn dập**, bạn chỉ đổ thêm dầu vào lửa (đúng tinh thần chống cascading ở Bài 5). **Exponential backoff** (thử lại sau 1s, 2s, 4s, 8s…) cho provider thời gian hồi phục. Cái thất bại **mãi** thì dạt sang **DLQ** để xử lý riêng, không kẹt hàng đợi chính.

#### 📘 **A-NOTI-003 — Throttle + user preference (opt-out, quiet hours)**

Áp **TRƯỚC** khi đẩy provider — tôn trọng **preference** + chống **spam** + giới hạn **quota provider**.

```
ingestion → [POLICY: opt-out? quiet hours? rate limit?] → [DELIVERY: gọi provider]
                  ▲ quyết "CÓ gửi không"                      ▲ quyết "gửi THẾ NÀO"
```

> 🎯 **Chốt:** **Tách policy khỏi delivery** — quyết *"có gửi không"* tách khỏi *"gửi thế nào"*. Hai mối quan tâm khác nhau, đừng trộn vào một chỗ.

#### 📘 **A-NOTI-004 — Fan-out sự kiện triệu user** (vd "live started")

> 🚨 **KHÔNG gửi đồng bộ inline** trong request (sẽ **timeout + đập provider**).

Đẩy **event vào queue**, **worker scale ngang**, **batch theo kênh**, **throttle provider**.

---

## ③ ⚠️ Cũ → mới / sai lầm hay gặp

| Hay gặp (❌) | ✅ Nên làm | Vì sao |
|---|---|---|
| Gọi **provider đồng bộ** trong request | Đẩy **queue + worker async** | Provider chậm/lỗi sẽ **chặn/timeout** request chính |
| Không **dedup** (at-least-once) | **Idempotency key + dedup store** | Retry/redelivery ⇒ user nhận **trùng** |
| Bỏ qua **preference/quiet hours** | Áp **policy trước khi gửi** | Spam ⇒ user **opt-out** / provider phạt |
| **Retry vô hạn** khi provider hỏng | **Exponential backoff + DLQ** | Tránh **bão retry**; cô lập lỗi để xử riêng |

---

## ④ Áp dụng thực tế + So sánh bigtech

Mọi hệ noti (**push** của app, **email marketing**, **OTP**) đều theo pattern **queue→worker→provider adapter** với **retry/DLQ** + **preference** + **dedup** (pattern phổ biến).

> ➕ **Thực tế:**
> - **provider adapter** trừu tượng hoá nhiều vendor (vd nhiều **SMS gateway** để **failover**).
> - **rate limit** theo **quota** từng provider.
> - **pipeline observability** theo dõi **delivery/bounce/open-rate**.
> - **Fan-out lớn** dùng **batch + throttle** để không vượt **quota provider**.

---

## ⑤ Code thực hành + cấu hình

Pipeline noti: **enqueue → worker dedup + preference + retry/DLQ** (Python, minh hoạ):

```python
# notification.py — minh hoạ; queue + redis dedup + provider adapter
def notify(user_id, event, payload, queue):
    # idempotency key: cùng (user,event,payload-hash) chỉ tạo 1 noti (A-NOTI-002)
    idem = f"{user_id}:{event}:{hash(frozenset(payload.items()))}"
    queue.publish("notifications", {"user_id": user_id, "event": event,
                                    "payload": payload, "idem": idem})

def worker(msg, redis, prefs, providers, dlq):
    # 1) DEDUP: nếu idem đã xử lý -> bỏ qua (at-least-once nên PHẢI khử trùng)
    if not redis.set(f"noti:idem:{msg['idem']}", 1, nx=True, ex=86400):
        return
    pref = prefs.get(msg["user_id"])
    # 2) POLICY trước DELIVERY (A-NOTI-003): opt-out, quiet hours, throttle
    if pref.opted_out(msg["event"]):      return
    if pref.in_quiet_hours():             return queue_for_later(msg)
    if pref.over_rate_limit(msg["event"]): return   # chống spam + quota provider
    # 3) DELIVERY theo từng kênh, có retry/backoff + DLQ
    for channel in pref.channels(msg["event"]):      # email / push / sms
        try:
            providers[channel].send(msg["user_id"], render(channel, msg))  # adapter per-channel
        except ProviderError as e:
            if msg.get("attempt", 0) < 5:
                requeue_with_backoff(msg, channel)   # EXPONENTIAL BACKOFF
            else:
                dlq.put(msg, reason=str(e))          # DLQ sau khi hết lượt thử
```

```text
# Cấu hình:
- Ingestion -> queue (Kafka/SQS); worker scale ngang theo kênh.
- Dedup: idempotency key trong Redis (SET NX EX); xử lý at-least-once an toàn.
- Policy: preference service (opt-out/quiet hours/rate) áp TRƯỚC delivery.
- Retry: exponential backoff + jitter; DLQ cho lỗi vĩnh viễn; alert khi DLQ tăng.
- Fan-out lớn: batch theo kênh + throttle theo quota provider; KHÔNG gửi inline.
```

> 🔎 **Vì sao `SET NX EX` là cách dedup gọn?** `NX` = "chỉ set nếu key **chưa tồn tại**", `EX` = đặt **TTL**. Lần đầu gặp `idem` → set thành công → xử lý tiếp. Lần sau (do **redelivery**) → set thất bại vì key đã có → **bỏ qua**. Một thao tác Redis nguyên tử thay cho cả một vòng kiểm-tra-rồi-ghi dễ dính race.

---

## ⑥ Keywords cần nhớ

🧠 **Cần HIỂU:**
- vì sao **async qua queue**.
- vì sao **at-least-once cần dedup**.
- **tách policy khỏi delivery**.
- vì sao **fan-out lớn phải batch+throttle**.

📌 **Cần THUỘC:**
- **ingestion→queue→worker→provider**.
- **idempotency key**.
- **backoff + DLQ**.
- **quiet hours / opt-out / rate limit**.

🛠️ **Cần LÀM ĐƯỢC:**
- thiết kế **pipeline noti reliable**.
- cài **dedup**.
- xử **fan-out triệu user**.

---

## ⑦ Mental model

> 🎯 **"Queue ở giữa, policy trước delivery, dedup chống trùng, DLQ giữ lỗi."**
> **Không bao giờ gọi provider đồng bộ trong request.**

---

## ⑧ Câu hỏi phỏng vấn & thách đố

**(TB)** Kiến trúc tối thiểu noti đa kênh?
→ **ingestion→queue→per-channel worker/adapter→retry/DLQ** + **template** + **preference** + **observability**.

**(khó)** Đảm bảo **không trùng** + **retry** khi provider lỗi?
→ **idempotency/dedup store**; **backoff+DLQ**; at-least-once nên **dedup**.

**(TB)** **Throttle + preference** đặt ở đâu?
→ **trước khi đẩy provider**; **tách policy khỏi delivery**.

**(khó)** Fan-out **"live started"** triệu user chịu tải sao?
→ **event→queue**, **worker scale ngang**, **batch+throttle**, không inline.

**(TB)** Vì sao không gọi **provider đồng bộ**?
→ provider chậm/lỗi **chặn request** + **đập quota**.

> 🧩 **Thách đố (trick):** Hệ retry khi provider timeout, nhưng user phàn nàn nhận **3 email giống nhau**. Nguyên nhân?
>
> ```
> ❌ Retry mà KHÔNG dedup:
> T1  worker gọi provider → provider GỬI THÀNH CÔNG
> T2  nhưng phản hồi về bị TIMEOUT (mạng chậm)
> T3  worker tưởng hỏng → RETRY → provider gửi LẠI
>     (lặp vài lần → user nhận 3 email)
>
> ✅ Fix: idempotency key ở provider (nếu hỗ trợ) + dedup store + theo dõi delivery status
> ```
> 🔎 **Vì sao là bẫy?** Provider **có thể đã gửi thành công** rồi *mới* timeout phản hồi ⇒ worker tưởng hỏng và **retry gửi lại**. Cần **idempotency key** ở provider (nếu hỗ trợ) + **dedup store** + theo dõi **delivery status** để không gửi lại cái đã gửi.

---

## ⑨ Bài tập + tiêu chí tự chấm

**BT1.** Thiết kế hệ noti cho event **"đơn hàng đã giao"** gửi **email+push**, tôn trọng **quiet hours**.

> ✅ **Đạt khi:** **queue+worker**, **dedup**, **policy trước delivery**, **retry+DLQ**.

**BT2.** Xử **"live started"** cho streamer **5M follower** mà không đập provider.

> ✅ **Đạt khi:** **event→queue**, **worker scale ngang**, **batch theo kênh**, **throttle quota**.

---

## ⑩ Đọc thêm

- **System Design Interview Vol 2** (Alex Xu) — *"Design a Notification System"*.
- **AWS** — *SQS DLQ*; *idempotency patterns*.

---

> 🎯 **Khẩu quyết khép bài:** *Queue ở giữa để không chặn hệ chính. Policy (có gửi không) tách khỏi delivery (gửi thế nào). Dedup vì at-least-once. Backoff + DLQ thay vì retry vô hạn. Fan-out lớn thì batch + throttle.*
