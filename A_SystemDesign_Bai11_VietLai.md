# 🎯 BÀI 11 — Case study: News feed & fan-out

**Phủ:** A-FEED-001 → A-FEED-006

> 💡 **Tư tưởng cốt lõi của cả bài:** Câu hỏi của feed là *"khi A đăng bài, làm sao những người follow A thấy nó nhanh?"*. Có hai cách trái ngược — **đẩy (push)** vào hộp thư từng follower lúc đăng, hay **kéo (pull)** gom bài lúc follower mở app. Bài này dạy rõ nhất **vì sao một thiết kế "đúng" phải đổi khi gặp celebrity**.

---

## ① Mục tiêu & vị trí trong mạch

Case study **fan-out** kinh điển — ráp **Bài 2** (read:write), **Bài 5** (hot key/celebrity), **Bài 6** (eventual consistency), giao với **mục F** (pagination/cursor).

> 🎯 **Vì sao bài này quan trọng?** Vì nó là minh hoạ sống động nhất cho tinh thần "không có tốt nhất, chỉ có phù hợp nhất": một thiết kế hoàn hảo cho user thường **sụp đổ** ngay khi gặp một tài khoản triệu follower.

---

## ② Giảng cơ bản → nâng cao

### (a) Trực giác

> **Ẩn dụ — báo giao tận nhà vs ra sạp mua:**
> - **Push (đẩy):** lúc A đăng bài, **giao ngay** một bản vào **hộp thư từng follower**. Follower mở app là thấy liền (đọc nhanh), nhưng A tốn công giao triệu bản (ghi nặng).
> - **Pull (kéo):** follower **ra sạp gom** bài của những người mình follow lúc mở app. A khỏe (ghi nhẹ), nhưng follower phải đợi gom (đọc chậm).

> 🎯 **Chốt:** **Push = đọc nhanh/ghi nặng; Pull = ghi nhẹ/đọc chậm.**

---

### (b) Cơ chế

#### 📘 **A-FEED-001 — Fan-out-on-write vs on-read**

| Tiêu chí | **on-write (push)** | **on-read (pull)** |
|---|---|---|
| Lúc nào làm việc nặng | lúc **post** (ghi vào feed mọi follower) | lúc **đọc** (gom bài từ người follow) |
| Đọc | **cực nhanh** (feed **precomputed**) | **chậm** (gom + **sort** runtime) |
| Ghi | **nặng** + tốn **storage** (nhân bản) | **nhẹ** |

> 🎯 **Chọn theo:** **read:write ratio** & **phân bố follower**.

#### 📘 **A-FEED-002 — Celebrity problem** *(giao Bài 5)*

> **Ví dụ trực giác:** Một người có **10 triệu follower** đăng *một* bài. Theo push, hệ phải ghi **10 triệu feed** chỉ vì một thao tác đăng — như giao 10 triệu tờ báo cùng một lúc.

```
❌ on-write cho celebrity:
celeb (10M follower) đăng 1 post
       ↓
hệ phải ghi vào 10.000.000 feed  → ❌ BÃO GHI + độ trễ khổng lồ

✅ hybrid:
user thường  → on-write (push vào feed follower)
celebrity    → KHÔNG push; lưu vào timeline tác giả
follower đọc → MERGE: push-feed + pull các celeb đang follow
```

> 🎯 **Giải:** **hybrid** — **on-write cho user thường** + **pull cho celebrity**, **merge lúc đọc**. *(Đây là câu phân biệt senior.)*

#### 📘 **A-FEED-003 — Feed lưu ở đâu**

- **precomputed feed** trong **cache/Redis** theo user.
- chỉ lưu **ID post** rồi **hydrate** nội dung sau.
- **giới hạn độ dài feed** (vài trăm mục gần nhất).

> 🔍 **Vì sao lưu ID chứ không lưu nội dung?** Hai lý do: (1) **tiết kiệm RAM** — ID nhẹ hơn full post rất nhiều; (2) khi **nội dung post đổi** (sửa bài, đổi ảnh), bạn **không phải rewrite** lại triệu feed — chỉ cần sửa ở một chỗ, lúc hydrate sẽ lấy bản mới.

#### 📘 **A-FEED-004 — Ranking (không chỉ theo thời gian)**

Cần **feature/score** ⇒ **không thể chỉ sort timestamp**. Thường **tách ranking service**, tính **async/precompute**.

> 🎯 **Trade-off:** **realtime** vs **chất lượng xếp hạng** — tính score càng kỹ càng tốn thời gian, càng kém realtime.

#### 📘 **A-FEED-005 — Mức consistency**

Thấy post **trễ vài giây** thì **vô hại** (khác tiền/vé) ⇒ ưu **availability/latency** hơn **strong consistency**. **Eventual** là đủ.

#### 📘 **A-FEED-006 — Pagination cuộn vô tận** *(giao F)*

Dùng **cursor** (timestamp/id) chứ **không offset**.

> **Ví dụ trực giác:** Bạn đang đọc trang sách bằng cách đếm "trang thứ 50" (**offset**). Ai đó **chèn thêm trang** vào đầu sách → "trang 50" giờ là nội dung khác → bạn đọc lại đoạn đã đọc hoặc nhảy mất đoạn. Nếu bạn đánh dấu *"đọc tiếp sau dòng chữ X cụ thể"* (**cursor**) thì dù sách bị chèn, bạn vẫn tiếp đúng chỗ.

```
❌ OFFSET khi có post mới chèn vào đầu:
T1  user xem trang 1 (post 100..91), rồi xin offset=10 (trang 2)
T2  có 3 post MỚI chèn vào đầu feed
T3  offset=10 giờ trỏ tới post 93..84  → ❌ post 91,90,89 BỊ XEM LẠI (nhảy/lặp)

✅ CURSOR:
T1  user xem tới post 91 → cursor = "id < 91"
T3  dù có post mới chèn, "id < 91" vẫn lấy đúng 90,89,88...  ✅ không lặp/nhảy
```

> 🔍 **Vì sao offset còn chậm dần?** Vì DB phải "đếm bỏ qua" N dòng đầu mỗi lần — list càng lớn, offset càng lớn, càng chậm. **Cursor** lọc thẳng `id < X` → dùng được index, ổn định và **hiệu quả realtime**.

---

## ③ ⚠️ Cũ → mới / sai lầm hay gặp

| Hay gặp (❌) | ✅ Nên làm | Vì sao |
|---|---|---|
| **On-write cho TẤT CẢ user** (kể cả celebrity) | **Hybrid**: on-write thường + pull celebrity | Celebrity ⇒ **bão ghi** triệu feed |
| Lưu **full nội dung post** trong mỗi feed | Lưu **ID post**, **hydrate** khi đọc | Đỡ RAM + nội dung sửa không phải **rewrite** feed |
| **Offset pagination** cho feed | **Cursor** (id/timestamp) | Offset chậm + **nhảy** khi có post mới |
| **Strong consistency** cho feed | **Eventual consistency** | Trễ vài giây vô hại; ưu latency/availability |

---

## ④ Áp dụng thực tế + So sánh bigtech

**Hybrid fan-out** là pattern phổ biến ở mạng xã hội lớn (**Twitter/X, Instagram**…): đa số user dùng **push**, tài khoản đông follower dùng **pull**, **merge lúc đọc**; **feed precomputed** ở store nhanh (**Redis-like**); **ranking** tách thành tầng **ML** riêng.

> ➕ **"For You" recommendation feed** (**Bài 12**) thêm tầng **candidate generation** + **ranking** trên nền **fan-out** này. (pattern phổ biến — **verify** nội bộ hãng.)

---

## ⑤ Code thực hành + cấu hình

**Hybrid fan-out** (Python, minh hoạ logic **push thường + pull celebrity + merge**):

```python
# feed.py — minh hoạ; redis cho feed store, queue cho fan-out async
CELEB_THRESHOLD = 100_000   # ngưỡng follower coi là "celebrity" (tinh chỉnh theo data)

def on_post(post_id, author_id, followers_count, redis, queue, db):
    if followers_count < CELEB_THRESHOLD:
        # FAN-OUT ON WRITE: đẩy post_id vào feed từng follower (async qua queue, KHÔNG inline)
        queue.publish("fanout", {"post_id": post_id, "author_id": author_id})
    else:
        # CELEBRITY: KHÔNG fan-out; chỉ lưu vào timeline tác giả, follower sẽ PULL lúc đọc
        db.append_author_timeline(author_id, post_id)

def fanout_worker(msg, redis, db):           # worker xử lý queue
    for follower in db.followers(msg["author_id"]):
        redis.lpush(f"feed:{follower}", msg["post_id"])    # lưu ID, KHÔNG lưu nội dung
        redis.ltrim(f"feed:{follower}", 0, 799)            # giới hạn ~800 mục gần nhất

def get_feed(user_id, cursor, redis, db):    # ĐỌC: merge push-feed + pull từ celebrity đang follow
    pushed = redis.lrange(f"feed:{user_id}", 0, 50)        # phần precomputed (user thường)
    celeb_posts = []
    for celeb in db.celebrities_followed_by(user_id):      # PULL celebrity timeline
        celeb_posts += db.author_timeline(celeb, after=cursor, limit=50)
    merged = sorted(set(pushed) | set(celeb_posts), reverse=True)[:50]  # MERGE + sort
    posts = db.hydrate(merged)               # hydrate nội dung từ ID (A-FEED-003)
    next_cursor = posts[-1].id if posts else None          # CURSOR pagination (A-FEED-006)
    return posts, next_cursor
```

```text
# Cấu hình:
- Fan-out: luôn async qua queue; KHÔNG fan-out đồng bộ trong request post.
- Feed store: Redis list/sorted-set theo user, lưu ID, ltrim giới hạn độ dài.
- Pagination: cursor (id/timestamp giảm dần), KHÔNG offset.
- Ranking (nếu có): tách ranking service, precompute score, merge khi serve.
- Consistency: eventual (chấp nhận trễ vài giây).
```

> 🔎 **Đọc kỹ `get_feed`:** đây là trái tim của **hybrid** — nó **không** chỉ đọc push-feed. Nó **merge** phần precomputed (push, user thường) với phần **pull** từ các celebrity đang follow. Bỏ bước pull này thì bài của celebrity sẽ **biến mất** khỏi feed (xem thách đố mục ⑧).

---

## ⑥ Keywords cần nhớ

🧠 **Cần HIỂU:**
- **fan-out on-write vs on-read** đánh đổi.
- **celebrity problem** & **hybrid**.
- vì sao lưu **ID** không lưu nội dung.
- vì sao **cursor** không **offset**.
- vì sao **eventual** đủ.

📌 **Cần THUỘC:**
- **push = đọc nhanh/ghi nặng, pull = ghi nhẹ/đọc chậm**.
- **hybrid merge lúc đọc**.
- **cursor pagination**.

🛠️ **Cần LÀM ĐƯỢC:**
- thiết kế **feed hybrid**.
- xử **celebrity**.
- chọn **cursor**.
- tách **ranking**.

---

## ⑦ Mental model

> 🎯 **"Push cho dân thường, pull cho ngôi sao, merge lúc đọc."**
> Lưu **ID + hydrate**; **cursor không offset**; **eventual** là đủ.

---

## ⑧ Câu hỏi phỏng vấn & thách đố

**(TB)** **Fan-out on-write vs on-read** khác gì?
→ **write**: đẩy lúc post, đọc nhanh/ghi nặng; **read**: gom lúc đọc, ghi nhẹ/đọc chậm.

**(khó)** **Celebrity problem** phá on-write sao, fix?
→ 1 post ghi triệu feed ⇒ **bão ghi**; **hybrid** push-thường + pull-celebrity, **merge**.

**(TB)** **Feed** lưu thế nào để đọc nhanh?
→ **precomputed cache** theo user, lưu **ID post**, **hydrate** sau, **giới hạn độ dài**.

**(TB)** Vì sao **eventual consistency** chấp nhận được?
→ trễ vài giây vô hại; ưu **latency/availability**.

**(khó)** Vì sao **cursor** chứ không **offset** cho cuộn vô tận?
→ offset chậm + nhảy khi có post mới; **cursor** ổn định/hiệu quả.

> 🧩 **Thách đố (trick):** *"Hybrid: user thường push, celebrity pull. Vậy một bài của celebrity, follower có thấy trong feed precomputed không?"*
>
> ```
> celeb đăng → KHÔNG fan-out → bài KHÔNG có trong push-feed
>        ↓
> ❌ Nếu chỉ đọc push-feed  → follower THIẾU post của celeb → bug feed
>        ↓
> ✅ Phải PULL celeb timeline + MERGE lúc đọc → mới đủ
> ```
> 🔎 **Vì sao là bẫy?** Vì bài celebrity **không nằm trong push-feed** (do không fan-out). Phải **pull + merge** lúc đọc. Nếu chỉ đọc push-feed, follower sẽ **thiếu post của celebrity** ⇒ **bug feed**.

---

## ⑨ Bài tập + tiêu chí tự chấm

**BT1.** Thiết kế feed cho **200M user**, phân bố follower lệch (đa số <1k, một số >10M). Quyết **push/pull/hybrid** + **ngưỡng**.

> ✅ **Đạt khi:** dùng **hybrid**, giải thích **ngưỡng celebrity**, mô tả **merge lúc đọc**.

**BT2.** Cho feed dùng **offset pagination** bị **"nhảy item"** khi có post mới. Giải thích nguyên nhân & sửa.

> ✅ **Đạt khi:** chỉ ra **offset dịch khi insert** + chuyển sang **cursor**.

---

## ⑩ Đọc thêm

- **System Design Interview** (Alex Xu) — *"Design a News Feed"*.
- **system-design-primer** — *feed*.
- Bài blog kỹ thuật về **timeline/fan-out** (**verify** nguồn/đời).

---

> 🎯 **Khẩu quyết khép bài:** *Push cho dân thường, pull cho ngôi sao, merge lúc đọc. Lưu ID rồi hydrate. Cursor chứ không offset. Feed trễ vài giây là chuyện thường — eventual là đủ.*
