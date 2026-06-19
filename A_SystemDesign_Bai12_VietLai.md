# 🎯 BÀI 12 — Case study: Video/feed platform quy mô lớn

**Phủ:** A-VID-001 → A-VID-005

> 💡 **Tư tưởng cốt lõi của cả bài:** Phục vụ video cho hàng triệu người là **bài toán băng thông, không phải compute**. Xương sống: **đẩy nội dung ra rìa mạng (CDN)** + **chuẩn bị sẵn nhiều phiên bản** để client chọn theo mạng. Mọi thứ nặng đều phải **async**, mọi thứ phân phối đều **CDN-first**.

---

## ① Mục tiêu & vị trí trong mạch

Case study **media-heavy** — ráp **Bài 2** (bandwidth/storage khổng lồ), **Bài 4** (CDN + object storage + queue), **Bài 11** (feed/recommendation), **Bài 5** (counter hot row).

> 🎯 **Vì sao gọi là bài "đắt nhất"?** Vì hạ tầng video ngốn **storage** và **bandwidth** ở quy mô khổng lồ — mỗi quyết định sai (stream từ origin, transcode đồng bộ, đếm view trực tiếp) đều đổ thành tiền và sự cố rất nhanh. Mục tiêu: hiểu **pipeline upload→serve** và vì sao **mọi thứ phải async + CDN-first**.

---

## ② Giảng cơ bản → nâng cao

### (a) Trực giác

> **Ẩn dụ:** Một rạp chiếu phim trung tâm (origin) không thể phục vụ cả nước cùng lúc — đường tới rạp sẽ tắc nghẽn. Giải pháp: mở **rạp con khắp các khu phố** (**CDN**), và **chuẩn bị sẵn nhiều bản chất lượng** (4K, HD, SD) để khán giả mạng yếu vẫn xem được bản nhẹ.

> 🎯 **Chốt:** Không thể **stream mọi video từ một origin** (sập băng thông + latency cao toàn cầu). Đẩy ra **CDN** + chuẩn bị sẵn nhiều bản.

---

### (b) Cơ chế

#### 📘 **A-VID-001 — CDN + Adaptive Bitrate (ABR)**

Không stream từ **origin** mà qua **CDN** (gần user ⇒ giảm latency + giảm tải/băng thông gốc).

**ABR (HLS/DASH):** video được chia thành nhiều **độ phân giải/bitrate**, cắt thành **segment nhỏ**; **client tự chọn bitrate** theo băng thông hiện tại.

> 🔍 **Vì sao ABR quan trọng?** Vì mạng người dùng **thay đổi liên tục**. Khi mạng yếu, client **hạ chất lượng** (1080p → 480p) thay vì **buffer** (quay vòng "đang tải"). Người xem thà thấy hình hơi mờ còn hơn màn hình đứng yên.

```
ABR — pre-encode SẴN nhiều bitrate:
video gốc ──► [240p][480p][720p][1080p]  (mỗi bản cắt thành segment nhỏ)
                       ↓
client mạng tốt  → tự chọn 1080p
client mạng yếu  → tự chọn 480p (không buffer)
```

> ⚠️ **Lưu ý:** **Pre-encode sẵn** nhiều bitrate. (**verify** chi tiết hãng.)

#### 📘 **A-VID-002 — Pipeline upload → serve (vì sao async)**

```
upload → object storage → transcoding (nhiều bitrate/format) qua queue + worker → đẩy CDN → metadata "ready"
                                    ▲ NẶNG & LÂU (phút–giờ)
```

> 🚨 **Vì sao KHÔNG được chặn user?** **Transcoding** (chuyển mã sang nhiều bitrate/format) **nặng & lâu** — có thể vài **phút đến giờ**. Nếu bắt user chờ tới khi transcode xong, request **timeout** và trải nghiệm hỏng. Đúng: user thấy **"đang xử lý"** rồi nhận **thông báo khi xong**.

#### 📘 **A-VID-003 — "For You" vs follow-feed** *(giao Bài 11)*

| | **follow-feed** (Bài 11) | **"For You" (recommendation-driven)** |
|---|---|---|
| Nguồn bài | người user **follow** | **đề xuất** (không chỉ follow) |
| Kiến trúc thêm | fan-out + merge | **candidate generation + ranking + serving** riêng |
| Tính chất | mix precompute/realtime | nặng phần **ML serving** |

> 💡 (Pattern phổ biến.)

#### 📘 **A-VID-004 — View/like count tỷ lượt**

> 🚨 **KHÔNG `UPDATE` trực tiếp mỗi lượt** — ghi nóng vào **1 row** ⇒ **lock/bottleneck** *(đúng hot row của Bài 5)*.

```
❌ UPDATE views=views+1 mỗi lượt:
tỷ lượt xem → tỷ lần ghi vào CÙNG 1 row → ❌ hot row, lock, bottleneck

✅ gom batch / approximate counter:
mỗi lượt → INCR ở Redis (nhẹ)
mỗi ~10s → flush GỘP về DB (1 lần ghi thay vì N) → chấp nhận đếm xấp xỉ + trễ
```

> 💡 Dùng **gom batch / approximate counter / đẩy qua stream rồi aggregate**; chấp nhận **đếm xấp xỉ + trễ**.

#### 📘 **A-VID-005 — Thumbnail/preview**

Sinh **offline** lúc upload (nhiều size), lưu **object storage**, phục vụ qua **CDN**.

> 🚨 **KHÔNG** sinh **on-the-fly** mỗi request — lãng phí **compute** + chậm. Một thumbnail sinh một lần, phục vụ triệu lần.

> 💡 **Phân biệt 📘 vs ➕:** **pipeline, CDN, ABR, counter, thumbnail** đều 📘. Liên hệ **ML-serving** cho recommendation là ➕ (mức pattern).

---

## ③ ⚠️ Cũ → mới / sai lầm hay gặp

| Hay gặp (❌) | ✅ Nên làm | Vì sao |
|---|---|---|
| **Stream video thẳng từ origin** | **CDN + ABR (HLS/DASH)** | Origin **sập băng thông**; latency toàn cầu cao |
| **Transcode đồng bộ** trong request upload | **Async qua queue + worker** | Transcode lâu (phút–giờ); chặn user = hỏng UX |
| **`UPDATE views=views+1`** mỗi lượt xem | **Stream + aggregate / approximate counter** | **Hot row** ⇒ lock/bottleneck ở tỷ lượt |
| Sinh **thumbnail on-the-fly** | Sinh **offline** + lưu storage + CDN | Tiết kiệm compute, phục vụ nhanh |

---

## ④ Áp dụng thực tế + So sánh bigtech

Đây là xương sống của mọi nền tảng video (**YouTube/Netflix/TikTok** — pattern phổ biến):

- **object storage** cho **master file**,
- **transcoding farm** (**queue + worker**) sinh **ABR ladder**,
- **CDN** phân phối **segment**,
- **recommendation** tách thành hệ **ML** riêng.

> ➕ **Netflix** nổi tiếng với **CDN đặt sâu trong mạng ISP** (**Open Connect**) — **verify** chi tiết khi trích. **View counter** dùng **stream processing** (**Kafka/Flink-like**) aggregate thay vì ghi DB trực tiếp.

---

## ⑤ Code thực hành + cấu hình

**Upload → enqueue transcoding (async)** + **approximate view counter**:

```python
# video_pipeline.py — minh hoạ; object storage + queue + redis counter
def handle_upload(file, user_id, storage, queue, db):
    master_key = storage.put_master(file)                  # 1) lưu master vào object storage
    video_id = db.create_video(user_id, master_key, status="processing")
    # 2) enqueue transcoding cho từng bitrate — ASYNC, không chặn response upload
    for profile in ["240p", "480p", "720p", "1080p"]:
        queue.publish("transcode", {"video_id": video_id, "master_key": master_key, "profile": profile})
    return {"video_id": video_id, "status": "processing"}   # trả NGAY; user nhận noti khi "ready"

def transcode_worker(msg, storage, cdn, db):               # worker (scale ngang)
    out_key = transcode(msg["master_key"], msg["profile"]) # nặng/lâu -> chạy NỀN
    storage.put(out_key); cdn.push(out_key)                # 3) đẩy CDN
    if db.all_profiles_done(msg["video_id"]):
        db.set_status(msg["video_id"], "ready")            # 4) cập nhật metadata khi xong

# View count xấp xỉ: tăng ở Redis, flush batch về DB định kỳ (A-VID-004)
def record_view(video_id, redis):
    redis.incr(f"views:{video_id}")                        # ghi nóng vào REDIS, KHÔNG vào DB row
def flush_views(redis, db):                                # cron mỗi ~10s: gom batch -> DB
    for key in redis.scan_iter("views:*"):
        vid = key.split(":")[1]; cnt = int(redis.getset(key, 0))
        if cnt: db.increment_views(vid, cnt)               # 1 lần ghi GỘP thay vì N lần
```

```text
# Cấu hình:
- Object storage cho master + variants; CDN phục vụ segment (HLS/DASH manifest + .ts/.m4s).
- Transcoding: queue + autoscaling worker farm; idempotent theo (video_id, profile).
- Counter: Redis incr + flush batch; chấp nhận trễ/xấp xỉ.
- Thumbnail: job offline lúc upload, lưu storage, phục vụ CDN.
```

> 🔎 **Vì sao worker phải idempotent theo (video_id, profile)?** Vì queue có thể giao **trùng** một message (at-least-once). Nếu transcode "720p của video X" chạy hai lần mà không idempotent, bạn tốn compute gấp đôi hoặc ghi đè lẫn lộn. Khoá theo `(video_id, profile)` đảm bảo chạy lại cũng cho cùng kết quả — *(chủ đề này đào sâu ở mục messaging)*.

---

## ⑥ Keywords cần nhớ

🧠 **Cần HIỂU:**
- vì sao **CDN-first**.
- **ABR** giải bài băng thông client.
- vì sao **pipeline phải async**.
- vì sao **counter không ghi DB trực tiếp**.

📌 **Cần THUỘC:**
- pipeline **upload→transcode→CDN→ready**.
- **HLS/DASH ABR**.
- **approximate counter**.
- **thumbnail offline**.

🛠️ **Cần LÀM ĐƯỢC:**
- thiết kế **pipeline video**.
- tách **counter** khỏi **hot row**.
- bố trí **CDN + object storage**.

---

## ⑦ Mental model

> 🎯 **"Master ở object storage, biến thể ở CDN, nặng thì async, đếm thì xấp xỉ."**
> Bài toán video là **băng thông + pipeline**, không phải **compute đồng bộ**.

---

## ⑧ Câu hỏi phỏng vấn & thách đố

**(TB)** Vì sao không **stream từ origin**, **ABR** là gì?
→ **CDN** gần user giảm latency/băng thông; **ABR** chia nhiều bitrate, client chọn theo mạng.

**(khó)** **Pipeline upload→serve** gồm gì, vì sao async?
→ **storage→transcode(queue+worker)→CDN→ready**; transcode nặng/lâu nên không chặn user.

**(khó)** **View count** tỷ lượt — vì sao không **UPDATE trực tiếp**?
→ **hot row** lock; **stream/aggregate/approximate counter**.

**(TB)** **"For You"** khác **follow-feed** ở kiến trúc nào?
→ recommendation: **candidate gen + ranking + ML serving** riêng.

**(TB)** **Thumbnail** quy mô lớn sinh/serve sao?
→ **offline** lúc upload, **object storage**, **CDN**.

> 🧩 **Thách đố (trick):** *"Upload xong em trả response sau khi transcode để chắc chắn video sẵn sàng."* Vấn đề?
>
> ```
> ❌ Đồng bộ hoá việc nặng:
> upload → CHỜ transcode 1080p (vài phút) → mới trả response
>        ↓
> request TIMEOUT, user chờ vô lý
>
> ✅ trả NGAY status "processing" → transcode nền → báo "ready" qua notification (Bài 14)
> ```
> 🔎 **Vì sao là bẫy?** Đồng bộ hoá việc nặng: **transcode 1080p** có thể mất **nhiều phút** ⇒ **request timeout**, user chờ vô lý. Đúng: trả ngay status **"processing"**, transcode **nền**, báo **"ready"** qua **notification** *(Bài 14)*.

---

## ⑨ Bài tập + tiêu chí tự chấm

**BT1.** Ước lượng **storage + bandwidth** cho nền tảng **1M video mới/ngày**, mỗi video 100MB master × 4 bitrate, 10M view/ngày @ 5MB/view.

> ✅ **Đạt khi:** dùng **Bài 2**, tách **storage (master+variant)** và **egress (view)**, kết luận **CDN bắt buộc**.

**BT2.** Thiết kế **counter view** chịu tỷ lượt/ngày không làm nóng DB.

> ✅ **Đạt khi:** **Redis incr + flush batch** (hoặc stream aggregate), chấp nhận **xấp xỉ/trễ**, **không ghi 1 row/lượt**.

---

## ⑩ Đọc thêm

- **HLS / MPEG-DASH specs** (ABR).
- **Netflix Tech Blog** — *Open Connect / encoding*.
- **System Design Interview Vol 2** (Alex Xu) — *YouTube/video*.

> ⚠️ **Lưu ý (verify):** **verify đời/hãng** khi trích số liệu.

---

> 🎯 **Khẩu quyết khép bài:** *Video = băng thông, không phải compute. CDN-first + ABR. Việc nặng (transcode) chạy nền qua queue. Đếm view bằng Redis + flush batch, chấp nhận xấp xỉ. Thumbnail sinh offline.*
