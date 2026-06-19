# 🎯 BÀI 1 — Quy trình & tư duy làm system design

**Phủ:** A-PROC-001 → A-PROC-007

> 💡 **Tư tưởng cốt lõi của cả bài:** Trong vòng system design, bạn **không** được chấm vì "biết nhiều công nghệ", mà vì **cách bạn suy nghĩ có tổ chức**. Một người bình tĩnh chạy đúng quy trình sẽ thắng một người giỏi hơn nhưng làm lộn xộn.

---

## ① Mục tiêu & vị trí trong mạch

Đây là **bài mở màn khối "Nền tảng tư duy"**. Học xong, bạn cầm trong tay một **khung 6 bước** chạy được gọn trong **45 phút phỏng vấn**, và hiểu được một điều quan trọng: **quy trình được chấm điểm độc lập với kiến thức**.

> 🎯 **Vì sao "quy trình tách rời kiến thức" lại quan trọng?**
> Vì interviewer không thể nhét cả Google vào 45 phút. Họ không kỳ vọng bạn nhớ chính xác mọi con số hay mọi sản phẩm. Cái họ *thật sự* quan sát được trong 45 phút là **bạn tiếp cận một bài toán mơ hồ, lớn, áp lực thời gian như thế nào**. Đó là thứ phản ánh đúng nhất công việc thật của một kỹ sư senior.

**Vị trí trong mạch học:** Mọi **case study** ở **Bài 9–15** thực chất chỉ là **Bài 1 áp vào một đề cụ thể**.

> **Ẩn dụ:** Bài 1 là *khuôn bánh*. Bài 9–15 là *các vị nhân khác nhau* đổ vào cùng khuôn đó. Nắm chắc khuôn ở đây thì 7 case study sau chỉ còn là việc **"điền nội dung vào khung"** — bạn không bao giờ phải bắt đầu từ con số 0 nữa.

---

## ② Giảng cơ bản → nâng cao

### (a) Trực giác

> **Ẩn dụ:** Một bài system design giống như bạn được giao **"xây một thành phố trong 45 phút"**.

- **Người yếu** lao ngay vào vẽ từng tòa nhà (vẽ từng **box**: "đây là service A, đây là DB B…"). Họ xây nhà trước cả khi biết thành phố này cho bao nhiêu dân.
- **Người giỏi** hỏi trước: *Thành phố này cho bao nhiêu dân? Ở vùng nào? Ưu tiên giao thông hay nhà ở?* — rồi mới quy hoạch.

> 💡 **Chốt trực giác:** Câu hỏi đặt ra lúc đầu (**clarify**) sẽ định hình **toàn bộ bản vẽ**. Bạn không thể vẽ đúng nếu chưa biết mình đang vẽ cho ai và với ràng buộc nào.

---

### (b) Cơ chế — khung 6 bước 📘 **A-PROC-001**

Đây là xương sống cần **thuộc lòng**. Sáu bước đi theo đúng thứ tự, mỗi bước "khóa" lại nền móng cho bước sau:

```
  ① Clarify ──► ② Estimate ──► ③ High-level ──► ④ API & Data ──► ⑤ Scale/Deep-dive ──► ⑥ Trade-off
   (hỏi)         (đo)            (vẽ khối lớn)     (neo hợp đồng)    (đào sâu 1-2 phần)     (tự soi nghẽn)
```

#### Bước 1 — **Clarify requirements** (làm rõ yêu cầu)

Chia yêu cầu thành **hai loại**:

| Loại | Nghĩa | Ví dụ cụ thể (đề "thiết kế Twitter") |
|---|---|---|
| **functional** | Hệ **làm gì** | post tweet, follow user, đọc timeline, redirect link rút gọn… |
| **non-functional** | Hệ phải **chạy tốt đến mức nào** | scale, latency, availability, consistency, durability |

> 📘 **A-PROC-002 — Phải chốt non-functional TRƯỚC khi vẽ.**
> **Vì sao?** Vì **non-functional định hình kiến trúc**. Cùng một tính năng "chuyển tiền", nếu yêu cầu **strong consistency** (tiền không được sai dù một xu) thì thiết kế khác *hẳn* so với khi chấp nhận **eventual consistency** (cho phép lệch tạm thời rồi hội tụ sau).
>
> 🔍 **Điều tệ nhất nếu làm sai:** Bạn vẽ xong cả kiến trúc dựa trên giả định "eventual là được", đến phút 30 interviewer nói "à mà đây là hệ thống ngân hàng" → toàn bộ bản vẽ phải đập đi. Đó là lý do **non-functional luôn phải chốt đầu tiên**.

Năm trục **non-functional** cần nhớ:

- **scale** — bao nhiêu người dùng, bao nhiêu request mỗi giây?
- **latency** — phản hồi phải nhanh cỡ nào (đặc biệt **p99**)?
- **availability** — được phép "chết" bao lâu mỗi năm?
- **consistency** — đọc ra có cần luôn thấy dữ liệu mới nhất không?
- **durability** — mất dữ liệu có chấp nhận được không?

#### Bước 2 — **Estimate** (ước lượng back-of-envelope)

Tính nhẩm trên giấy: từ **DAU** (Daily Active Users) suy ra **QPS** (Queries Per Second), rồi **storage**, **bandwidth**. *(Chi tiết cách tính ở **Bài 2**.)*

> ⚠️ **Nguyên tắc vàng:** Chỉ ước lượng **con số nào dẫn tới một quyết định**. Đừng tính cho "đủ bài". *(Xem thêm A-CAP-006 — vì sao estimation trang trí là lãng phí.)*
>
> **Ví dụ:** Tính ra "peak QPS = 50.000" là *có ích* vì nó nói cho bạn biết một DB đơn không gánh nổi → cần sharding/cache. Nhưng tính "trung bình mỗi user có 3.2 avatar" thì… để làm gì?

#### Bước 3 — **High-level design** (vẽ khối lớn)

Vẽ luồng dữ liệu xương sống:

```
client ──► LB ──► service ──► cache ──► DB
                      │
                      └──► queue (xử lý bất đồng bộ)
```

Quan trọng: **đặt tên cho luồng dữ liệu**, đừng chỉ vẽ box rời rạc. Nói thành tiếng: "request đọc đi qua cache trước, miss thì xuống DB…".

#### Bước 4 — **API contract & data model** 📘 **A-PROC-006**

Làm **sau** high-level, **trước** scale. Định nghĩa vài **endpoint** chính + **schema** cốt lõi.

> 💡 **Vì sao bước này quan trọng dù nhiều người bỏ qua?** Vì nó **"neo"** thiết kế xuống mặt đất. Khi bạn viết ra `POST /tweet { text, userId }` và bảng `tweets(id, user_id, text, created_at)`, cuộc nói chuyện không còn **chung chung lơ lửng** nữa — mọi tranh luận sau đó đều có chỗ bám.
>
> **Ví dụ:** Thay vì nói mơ hồ "service lưu tweet vào DB", bạn neo lại:
> ```
> POST /tweets        { userId, text }        -> 201 { tweetId }
> GET  /users/:id/feed?cursor=...             -> 200 { tweets[], nextCursor }
> ```
> Giờ khi bàn tới scale, ta biết chính xác *cái gì* cần scale.

#### Bước 5 — **Scale & deep-dive**

Chọn **1–2 phần** để đào sâu (ví dụ: **DB scaling**, **cache**, **hot key**…). **Không** rải mỏng khắp nơi.

#### Bước 6 — **Bottleneck & trade-off**

**Tự** chỉ ra điểm nghẽn của chính thiết kế mình vừa vẽ, và nêu **đánh đổi** của mỗi lựa chọn.

> 🎯 **Đây là bước phân biệt junior với senior.** Junior chờ bị hỏi "thế nó nghẽn ở đâu?". Senior **tự** nói: "Chỗ này sẽ nghẽn ở fan-out khi user có 10 triệu follower, đánh đổi là…".

---

#### ⏱️ Phân bổ thời gian ~45 phút 📘 **A-PROC-004**

> **Ví dụ trực giác:** Coi 45 phút như một cái bánh. Nếu bạn ăn hết 30 phút cho phần khai vị (clarify + vẽ box), bạn sẽ **không còn chỗ** cho món chính (deep-dive) — và món chính mới là thứ ghi điểm.

```
0'         5'         10'                  20'                            40'        45'
├──────────┼──────────┼────────────────────┼──────────────────────────────┼──────────┤
│ Clarify  │ Estimate │ High-level + API    │ Deep-dive (1–2 phần)          │ Trade-off│
│  ~5'     │  ~5'     │      ~10'           │         ~15–20'               │   ~5'    │
└──────────┴──────────┴────────────────────┴──────────────────────────────┴──────────┘
                                            ▲ phần ăn điểm nhất            ▲ đừng quên!
```

> ⚠️ **Nguyên tắc:** **Không sa lầy** vào một phần. **Quản lý thời gian là một kỹ năng được chấm**, không phải chuyện phụ. Một thiết kế "đủ tốt mọi mặt" thắng một thiết kế "hoàn hảo phần đầu, bỏ trống phần cuối".

---

### (c) Mép giới hạn & sai lầm 📘 **A-PROC-007**

> 🚨 **Câu hỏi đắt giá:** *Vì sao có người biết rất nhiều vẫn trượt vòng này?* Câu trả lời gần như luôn là: **quy trình kém**, không phải thiếu kiến thức.

Các lý do trượt phổ biến:

- ❌ **Nhảy vào chi tiết quá sớm** — vẽ box trước khi hiểu đề.
- ❌ **Không hỏi requirement** — đoán mò scope.
- ❌ **Im lặng** — không **think out loud**, interviewer không thấy được bạn nghĩ gì.
- ❌ **Không nêu trade-off** — trình bày như "chỉ có một đáp án đúng".
- ❌ **Over-engineer** — vẽ thừa thành phần không ai cần *(xem **Bài 6**)*.

> ➕ **A-PROC-005 — "Drive the interview" (cầm lái cuộc phỏng vấn).**
> Chủ động: **đề xuất scope**, **nói giả định thành tiếng**, **tự chọn phần deep-dive**, **tự chỉ bottleneck**.
>
> 🔍 **Vì sao ngồi chờ là điểm trừ?** Vì system design ở công ty thật **không ai cầm tay chỉ việc**. Khi bạn ngồi chờ interviewer hỏi mới nói, bạn đang phát tín hiệu **thiếu ownership** — đúng thứ mà một senior *không được phép* thiếu.

> 💡 **Phân biệt 📘 vs ➕:**
> - 📘 (chuẩn phổ biến): khung 6 bước và phân bổ thời gian.
> - ➕ (nhấn mạnh của giảng viên): ý **"drive the interview"** và **"think out loud như tín hiệu senior"**.

---

## ③ ⚠️ Kiến thức cũ / hay bị làm sai

| Cách làm hay gặp (❌ sai) | ✅ Nên làm | Vì sao |
|---|---|---|
| Vẽ **box** ngay khi nghe đề | **Clarify** functional + non-functional trước | Không có **non-functional** thì không biết thiết kế cho **scale** nào |
| Liệt kê mọi công nghệ mình biết | Chỉ đưa thành phần mà **requirement** đòi hỏi | Khoe kiến thức = **over-engineer** = điểm trừ *(A-TO-007)* |
| Ước lượng mọi con số cho "đủ bài" | Chỉ ước lượng con số **đổi quyết định** | **Estimation** trang trí lãng phí thời gian *(A-CAP-006)* |
| Im lặng suy nghĩ rồi mới nói kết quả | **Think out loud** suốt quá trình | Interviewer chấm **quá trình tư duy**, không chỉ kết quả |

---

## ④ Áp dụng thực tế + So sánh bigtech

Khung 6 bước **không chỉ dùng cho phỏng vấn** — nó chính là cách viết **design doc / RFC** thật ở công ty:

```
Context & Requirements   ◄── (≈ bước 1: clarify)
        ↓
Goals / Non-goals        ◄── (chốt scope, nói rõ cái gì NẰM NGOÀI)
        ↓
Proposed design          ◄── (≈ bước 3–5: high-level + deep-dive)
        ↓
Alternatives considered  ┐
        ↓                ├── (≈ bước 6: trade-off)
Trade-offs               ┘
        ↓
Rollout
```

> 💡 Phần **"Alternatives considered"** và **"Trade-offs"** trong RFC chính là **bước 6** ở trên — chỉ đổi tên, không đổi bản chất.

> ➕ **So sánh bigtech:** Các template lớn đều bắt buộc nêu **non-goals** và **alternatives**:
>
> | Công ty / template | Đặc trưng | Khớp với tinh thần Bài 1 |
> |---|---|---|
> | **Google Design Doc** | Bắt buộc mục Goals/Non-goals, Alternatives | "clarify scope + nêu trade-off" |
> | **Amazon "Working Backwards" / PRFAQ** | Viết từ thông cáo báo chí ngược về thiết kế | ép làm rõ requirement & người dùng trước |
> | **RFC nội bộ** (nhiều công ty) | Mục Trade-offs là bắt buộc | đúng bước 6 |
>
> 🎯 **Chốt:** Cùng một bộ não — quy trình bạn luyện cho phỏng vấn cũng là quy trình bạn dùng để viết doc thật. Học một lần, dùng cả sự nghiệp.

---

## ⑤ Code thực hành + cấu hình

Bài này là **quy trình**, không phải code. Thay vào đó, đây là **checklist clarify dán sẵn** — đọc to khi bắt đầu *mọi* bài design (và mở đầu *mọi* design doc):

```text
# CLARIFY CHECKLIST (đọc to khi bắt đầu)

FUNCTIONAL
- [ ] Tính năng cốt lõi (must-have) vs nice-to-have?
- [ ] Ai dùng? Luồng chính end-to-end là gì?

NON-FUNCTIONAL  (⚠️ CHỐT TRƯỚC KHI VẼ — vì nó định hình kiến trúc)
- [ ] Scale: DAU? QPS avg/peak? read:write ratio?
- [ ] Latency budget (p99)? real-time hay async được?
- [ ] Consistency cần: strong hay eventual?   (tiền/tồn kho => strong)
- [ ] Availability target? Durability (mất data có chấp nhận không)?
- [ ] Đặc thù data: kích thước item? tăng trưởng/năm?

SCOPE
- [ ] Cái gì NẰM NGOÀI phạm vi (non-goals)?
```

> 🔎 **Mẹo dùng:** Dòng đáng giá nhất là dòng cuối — **non-goals**. Nói rõ "tôi sẽ *không* làm phần X trong scope này" cho thấy bạn **drive the interview** và quản lý phạm vi chủ động, thay vì để đề trôi lan man.

---

## ⑥ Keywords cần nhớ

🧠 **Cần HIỂU:**
- **functional vs non-functional** — hệ làm gì vs hệ chạy tốt đến đâu.
- **"non-functional định hình kiến trúc"** — câu thần chú của cả bài.
- **drive the interview** — chủ động cầm lái, không ngồi chờ.
- **think out loud** — nói suy nghĩ thành tiếng để được chấm quá trình.
- **YAGNI / over-engineering** — "You Aren't Gonna Need It"; đừng vẽ thừa.

📌 **Cần THUỘC:**
- **6 bước:** clarify → estimate → high-level → API/data → scale → trade-off.
- **5 trục non-functional:** **scale**, **latency**, **availability**, **consistency**, **durability**.

🛠️ **Cần LÀM ĐƯỢC:**
- Chạy trơn **clarify checklist**.
- Phân bổ **thời gian 45'** (5/5/10/20/5).
- **Tự chỉ ra bottleneck** của thiết kế mình vừa vẽ.

---

## ⑦ Mental model

> 🎯 **"Clarify → Estimate → Sketch → Contract → Scale → Trade-off."**
> **Hỏi trước khi vẽ; non-functional vẽ ra kiến trúc.**

> 💡 Cách ghi nhớ nhanh 6 bước: *Hỏi → Đo → Phác → Neo → Phình → Soi.*
> (Hỏi yêu cầu, Đo con số, Phác khối lớn, Neo API/schema, Phình ra để scale, Soi điểm nghẽn.)

---

## ⑧ Câu hỏi phỏng vấn & thách đố

**(dễ)** Kể 6 bước tiếp cận một bài system design theo thứ tự.
→ **Gợi ý:** clarify → estimate → high-level → API/data model → scale/deep-dive → bottleneck/trade-off.

**(TB)** Functional vs non-functional khác gì, vì sao chốt non-functional trước?
→ **functional** = làm gì; **non-functional** = scale/latency/availability/consistency/durability; nó **định hình kiến trúc** (strong consistency ⇒ thiết kế khác hẳn).

**(khó)** "Drive the interview" nghĩa là gì?
→ chủ động **scope**, nêu **giả định**, tự chọn **deep-dive** & chỉ **bottleneck** — thể hiện **ownership**.

**(khó)** Phỏng vấn 45', bạn chia thời gian sao?
→ **5/5/10/20/5**; không sa lầy; chừa thời gian cho **trade-off**.

**(rất khó)** Vì sao có người biết nhiều vẫn trượt vòng này?
→ **quy trình kém**: nhảy chi tiết sớm, không clarify, không trade-off, im lặng, over-engineer.

> 🧩 **Thách đố (trick):** Interviewer ném đề rồi **im lặng** chờ. Bạn làm gì đầu tiên?
>
> ```
> T0  Interviewer:  ném đề "thiết kế Twitter" → rồi IM LẶNG (đang test)
>        ↓
> T1  ❌ BẪY:       lao vào vẽ box ngay      → mất điểm (không drive được)
>        ↓
> T1  ✅ ĐÚNG:      đặt câu hỏi clarify       → "Cho mình hỏi: scale cỡ nào? read-heavy?"
>                   + nói giả định thành tiếng → "Mình giả định ~100M DAU, read:write ~100:1"
> ```
> 🔎 **Vì sao đây là bẫy?** Sự im lặng *chính là* bài test — nó kiểm tra xem bạn có **drive the interview** được không, hay chỉ biết phản ứng khi bị hỏi.

---

## ⑨ Bài tập + tiêu chí tự chấm

**BT1.** Lấy đề **"thiết kế Twitter"**. Trong **5 phút**, viết ra:
- 3 **functional** must-have,
- 5 **non-functional** (kèm con số giả định),
- 2 **non-goals**.

> ✅ **Đạt khi:** có đủ **5 trục non-functional** và **mỗi cái có con số/giả định cụ thể**, không bỏ trống.
>
> **Ví dụ mẫu một dòng đạt:** "latency: timeline load p99 < 200ms" (✅ có trục + có con số) — chứ không phải "latency: phải nhanh" (❌ trống số).

**BT2.** Vẽ **timeline 45 phút** cho đề đó, ghi rõ phút nào làm gì.

> ✅ **Đạt khi:** có **≥1 deep-dive** được cấp **≥10'**, và có **một ô riêng cho trade-off** ở cuối.

---

## ⑩ Đọc thêm

- **System Design Interview** (Alex Xu) — chương *"A Framework for System Design Interviews"*.
- **Google Eng Practices** — *Design Docs*.
- **AWS** — *Working Backwards / PRFAQ*.
- **github.com/donnemartin/system-design-primer** — mục *"How to approach a system design interview question"*.

> ⚠️ **Lưu ý (verify):** Tên chương/mục và template công ty có thể đổi theo thời gian — nên **verify** lại nguồn gốc khi tra cứu, đừng trích dẫn nguyên văn từ trí nhớ.

---

> 🎯 **Khẩu quyết khép bài:** *Hỏi trước khi vẽ. Non-functional vẽ ra kiến trúc. Nói thành tiếng. Tự soi điểm nghẽn.* — Bốn câu này đi cùng bạn qua cả 7 case study sau.
