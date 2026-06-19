# 🎯 BÀI 16 — Phán Đoán Tech Lead: Hiểu Để Chỉ Huy & Kiểm Tra

**Phủ:** A-RT-001, A-RT-002 + soi lại toàn bộ Bài 1→15

> 💡 **Tư tưởng cốt lõi của cả bài:** Đây **không phải** bài "thêm kiến thức" mà là bài **"đổi vai"**: từ **người trả lời đúng** thành **người phán đoán đúng** — biết **cái gì sai trước khi nó sai**, biết **"đúng trên giấy" khác "sống ở production"**.

> 🧭 **Vị trí:** Khối V — **Lăng kính tổng hợp**.

---

## ① Mục tiêu & vị trí trong mạch

Mười lăm bài trước cho bạn **vũ khí**. Bài này dạy **khi nào rút vũ khí nào** và **làm sao biết người khác đang chọn sai**.

> 🎯 Một **Tech Lead (TL)** không cần gõ lại từng dòng — họ cần:
> 1. **phản biện** một lựa chọn kiến trúc bằng **câu hỏi đúng**, và
> 2. **dự đoán cái gì sẽ vỡ ở production** dù thiết kế *"nhìn đẹp"*.

> 🎯 **Khẩu quyết kết tinh của cả mục A:**
> *"Tôi không nhớ để gõ, tôi hiểu để chỉ huy — và tra cứu phần còn lại."*

---

## ② Giảng: hai năng lực phán đoán cốt lõi

### 📘 Năng lực 1 — Phản biện lựa chọn công nghệ (**A-RT-001**)

> **Tình huống mẫu:** Có người đề xuất **fine-tune** một **LLM** để làm chatbot trả lời theo **tài liệu nội bộ hay thay đổi**. Nghe *"AI xịn"*, nhưng TL phải thấy ngay đây là **lựa chọn sai công cụ**.

**Vì sao sai:**

- **Tài liệu hay đổi** ⇒ **fine-tune** là "nướng" kiến thức vào **trọng số model**; đổi tài liệu là phải **train lại** → tốn tiền, tốn thời gian, chậm cập nhật.
- Fine-tune dễ **hallucinate** dữ kiện và **khó truy nguồn** (không chỉ ra "câu trả lời lấy từ trang nào").
- Không kiểm soát được **phiên bản tri thức**; rủi ro **nói sai thông tin nội bộ**.

**Lựa chọn đúng: RAG (Retrieval-Augmented Generation) / lookup.**

> **Ẩn dụ:** Fine-tune như bắt học sinh **học thuộc lòng** cả cuốn sách — đổi sách thì phải học lại từ đầu. **RAG** như cho học sinh **mở sách tra** lúc làm bài — đổi sách chỉ cần thay sách trên kệ, và em ấy luôn **trích được trang nào**.

| Tiêu chí | **Fine-tune** | **RAG / lookup** |
|---|---|---|
| Tri thức nằm ở đâu | "nướng" vào **trọng số** | để ngoài (**vector store / search**) |
| Đổi tài liệu | phải **train lại** | chỉ **cập nhật index** |
| Truy nguồn | khó | **trích nguồn** được |
| Hallucinate | dễ hơn | giảm |
| Hợp khi | đổi **phong cách/định dạng/giọng** hoặc **kỹ năng ổn định** | **tri thức động** |

> 🎯 **Chốt:** **Fine-tune chỉ hợp khi cần đổi phong cách/định dạng/giọng hoặc kỹ năng ổn định**, KHÔNG phải để nhồi tri thức hay-đổi.

**Khung phản biện tổng quát** (áp cho MỌI đề xuất công nghệ):

1. **Bản chất bài toán** là gì? (tri thức hay-đổi? hay phong cách cố định?)
2. Công cụ này **khớp bản chất** đó không? (fine-tune hợp "kỹ năng/giọng", RAG hợp "tri thức động")
3. **Chi phí thay đổi & vận hành**? (train lại tốn gì? cập nhật ra sao?)
4. Có **cách đơn giản hơn** đạt 80% kết quả không? (đừng dùng **búa tạ đập đinh ghim**)
5. **Đo bằng gì** để biết đúng/sai? (không có **metric** thì không phải kỹ thuật, là niềm tin)

---

### 📘 Năng lực 2 — Dự đoán cái "đúng trên giấy nhưng vỡ ở production" (**A-RT-002**)

> **Ví dụ trực giác:** Bản vẽ nhà trên giấy giả định **đất phẳng, không mưa bão**. **Production** là đất thật có **động đất, lũ lụt**. TL giỏi là **kỹ sư kết cấu** nhìn bản vẽ đẹp và chỉ ra **chỗ sẽ nứt**.

Thiết kế trên **whiteboard** giả định **tải đều, mạng hoàn hảo, dữ liệu phân bố đẹp**. Production thì ngược lại. Danh mục **"tử huyệt"** hay gặp — **checklist vàng**:

| Tử huyệt production | "Trên giấy" trông sao | Thực tế vỡ thế nào | Truy về bài |
|---|---|---|---|
| **Hot key / partition lệch** | "Sharding chia đều tải" | 1 key/1 shard nóng (celeb, sản phẩm flash sale) ôm phần lớn traffic | Bài 11, 15, 5 |
| **Cold start** | "Serverless tự co giãn, rẻ" | Burst → loạt cold start → **p99 latency** vọt; pool DB cạn | Bài 8 |
| **Cache stampede** | "Có cache là nhanh" | Key hết hạn đồng loạt → ngàn request cùng đập DB | Bài 4, 12 |
| **Connection pool cạn** | "Mỗi instance nối DB là xong" | Scale ngang × pool/instance > **max_connections** DB → DB từ chối | Bài 8, 5 |
| **Network partition** | "Các node nói chuyện với nhau" | Mạng đứt → **split-brain** / phải chọn **C hay A (CAP)** | Bài 6 |
| **Thundering herd / retry storm** | "Lỗi thì retry" | Lỗi lan → mọi client retry đồng loạt → **tự DDoS chính mình** | Bài 14, 3 |
| **Chi phí ở tải thật** | "Kiến trúc này chạy được" | Đúng nhưng hóa đơn gấp 10 lần; serverless/**egress/cross-AZ** ăn tiền | Bài 8, 2 |
| **Thiếu observability** | "Chạy là biết" | Vỡ mà không có **metrics/log/trace** → mù, không rollback nổi | Toàn mục |
| **Thiếu rollback / migration nguy hiểm** | "Deploy là xong" | **Schema change** khóa bảng giờ cao điểm; không có đường lùi | Bài 4, 5 |
| **Clock skew / thời gian** | "Lấy timestamp là xong" | Đồng hồ lệch giữa node → ID trùng, **ordering** sai | Bài 4, 13 |

---

### ⬆️ Tư duy "soi lại 15 bài qua lăng kính TL"

Mỗi bài trước giờ trở thành **một câu hỏi kiểm tra** bạn ném vào bất kỳ thiết kế nào:

- **(Bài 1)** Đã làm rõ **scope/giả định/SLA** chưa, hay nhảy vào vẽ box ngay?
- **(Bài 2)** Con số **tải/throughput/dung lượng** có ước lượng không, hay *"chắc ổn"*?
- **(Bài 3,4,5)** **Điểm nghẽn** nằm đâu? **LB, cache, queue** đặt đúng chỗ chưa? Cái gì là **single bottleneck**?
- **(Bài 6)** Chỗ nào cần **strong**, chỗ nào **eventual**? Có nhầm lẫn không?
- **(Bài 7,8)** Cắt service theo **nghiệp vụ** hay theo **tầng kỹ thuật**? **Serverless** có hợp **workload** không?
- **(Bài 9→15)** Mỗi case study có đúng **pattern lõi** (**atomic, idempotent, fan-out, dedup**…) không?

> 🎯 **Mental shift:** người mới hỏi *"thiết kế này làm thế nào?"*; **Tech Lead** hỏi *"thiết kế này **sai ở đâu, đắt ở đâu, vỡ khi nào, và đo bằng gì**?"*.

---

## ③ ⚠️ Cũ → mới / sai lầm hay gặp

| Sai lầm tư duy (❌) | Vì sao nguy hiểm | ✅ Tư duy TL đúng |
|---|---|---|
| Chọn công nghệ vì *"hot/xịn"* (fine-tune mọi thứ) | Sai công cụ cho **bản chất** bài toán | Khớp công cụ với bản chất (**RAG** cho tri thức động) |
| Tin *"thiết kế đẹp = chạy tốt"* | Production có **hot key, cold start, partition** | Luôn hỏi *"vỡ ở đâu khi tải thật/mạng đứt?"* |
| Bỏ qua **chi phí** | Kiến trúc đúng nhưng **phá sản vì hóa đơn** | Đưa **cost** thành tiêu chí thiết kế |
| Không có **observability/rollback** | Vỡ mà mù, không lùi được | Coi **metrics/log/trace/rollback** là bắt buộc |
| *"Retry là xử lý lỗi"* | **Retry storm** tự DDoS | **Backoff + jitter + circuit breaker + DLQ** |
| Phán đoán không kèm **metric** | Tranh luận theo cảm tính | Mọi quyết định gắn **số đo** |

---

## ④ Áp dụng thực tế + so sánh bigtech (pattern phổ biến)

- Văn hóa **design review / RFC** ở các công ty lớn chính là **thể chế hóa** hai năng lực trên: bắt người đề xuất trả lời *"vỡ ở đâu, đo bằng gì, chi phí bao nhiêu"*.
- Thực hành **chaos engineering** (cố tình gây lỗi để **lộ tử huyệt** trước khi production lộ) là biểu hiện của tư duy *"đúng trên giấy chưa đủ"*.
- **RAG vs fine-tune** giờ là chủ đề phổ biến: cộng đồng đồng thuận **RAG cho tri thức hay-đổi**, **fine-tune cho phong cách/kỹ năng** — đúng như khung phản biện ở trên.

> 🎯 **Quy tắc TL chốt mục A:** bạn **không cần thuộc mọi con số**; bạn cần biết **câu hỏi nào vạch ra điểm yếu** và **tra cứu phần chi tiết** khi cần.

---

## ⑤ Code/Checklist thực hành

Bài này "phán đoán" nên "code" là **checklist review** bạn dán lên mỗi thiết kế:

```text
# DESIGN REVIEW CHECKLIST (lăng kính Tech Lead) — dùng cho mọi đề xuất

[ Bài toán & lựa chọn ]
□ Bản chất bài toán đã nêu rõ? (tri thức động? phong cách? throughput? consistency?)
□ Công cụ đề xuất KHỚP bản chất? (vd: tri thức hay-đổi -> RAG, KHÔNG fine-tune)
□ Có phương án đơn giản hơn đạt ~80% kết quả?

[ Quy mô & số liệu ]
□ Ước lượng tải/QPS/dung lượng/băng thông? (Bài 2)
□ Hot key / partition lệch đã tính? Ai là "celeb/flash-sale" của hệ này? (Bài 11,15)

[ Điểm vỡ production ]
□ Cold start / pool DB cạn khi scale ngang? (Bài 8)
□ Cache stampede / thundering herd / retry storm? (Bài 4,14)
□ Network partition -> chọn C hay A? split-brain? (Bài 6)
□ Strong vs eventual đặt đúng chỗ? (Bài 6)

[ Vận hành ]
□ Observability: có metrics/log/trace để biết khi nào vỡ?
□ Rollback / migration an toàn (không khóa bảng giờ cao điểm)?
□ Chi phí ở TẢI THẬT (không chỉ tải demo)? egress/cross-AZ/serverless?

[ Đo lường ]
□ Mỗi quyết định gắn 1 metric để biết đúng/sai?
```

> 🔎 **Cách dùng checklist này như TL:** đừng đọc nó như danh sách "để có". Mỗi ô `□` là **một câu hỏi bạn hỏi ra tiếng** trong design review. Người đề xuất trả lời trôi chảy → thiết kế chín. Lúng túng ở ô nào → đó chính là **tử huyệt chưa nghĩ tới**.

---

## ⑥ Keywords cần nhớ

🧠 **Cần HIỂU:**
- vì sao **RAG ≠ fine-tune** theo **bản chất bài toán**.
- vì sao *"đúng trên giấy"* hay vỡ (**hot key, cold start, partition, stampede, cost**).
- vì sao mọi quyết định cần **metric**.

📌 **Cần THUỘC:**
- **khung phản biện 5 câu hỏi**.
- **checklist tử huyệt production**.
- **RAG cho tri thức động / fine-tune cho phong cách**.

🛠️ **Cần LÀM ĐƯỢC:**
- phản biện một đề xuất kiến trúc trong **2 phút**.
- chỉ ra **≥3 điểm sẽ vỡ ở production** cho một sơ đồ "đẹp".
- gắn **metric** cho mỗi quyết định.

---

## ⑦ Mental model

> 🎯 **"Người mới hỏi 'làm thế nào'; Tech Lead hỏi 'sai ở đâu, đắt ở đâu, vỡ khi nào, đo bằng gì'. Hiểu để chỉ huy — tra cứu phần còn lại."**

---

## ⑧ Câu hỏi phỏng vấn & thách đố

**(TB)** Có người muốn **fine-tune LLM** cho chatbot tra cứu **tài liệu nội bộ hay đổi**. Bạn phản biện sao?
→ **sai công cụ**; tài liệu động nên dùng **RAG/lookup** (cập nhật index không train lại, trích nguồn, ít hallucinate); **fine-tune** chỉ hợp **phong cách/kỹ năng**.

**(khó)** Kể 5 thứ *"đúng trên giấy nhưng vỡ ở production"* và cách phát hiện sớm.
→ **hot key, cold start, cache stampede, pool cạn, network partition / retry storm**; phát hiện qua **load test, chaos, observability, ước lượng tải**.

**(TB)** Vì sao mọi quyết định kiến trúc nên gắn **metric**?
→ tránh tranh luận cảm tính; có cơ sở **rollback**; biết khi nào giả định sai.

**(khó)** Một sơ đồ **microservices** "đẹp" — bạn hỏi **3 câu** nào đầu tiên để tìm điểm yếu?
→ (a) **hot key/điểm nghẽn đơn** ở đâu? (b) **strong/eventual** đặt đúng chỗ chưa? (c) **chi phí & observability** ở tải thật ra sao?

**(TB)** Khi nào **fine-tune** thực sự hợp lý?
→ khi cần đổi **phong cách/định dạng/giọng** hoặc **kỹ năng ổn định**, dữ liệu ít đổi — không phải để nhồi tri thức hay-đổi.

> 🧩 **Thách đố (trick):** Sếp nói: *"thiết kế này đã chạy ổn ở môi trường staging 1 tuần, cứ lên production."* Bạn — Tech Lead — phản biện thế nào?
>
> ```
> ❌ "staging ổn" KHÔNG tái hiện production:
> (a) tải staging nhỏ      → hot key / partition lệch / pool cạn CHƯA lộ
> (b) không traffic thật   → cold start, cache stampede, retry storm CHƯA xảy ra
> (c) data staging sạch/nhỏ→ query chậm trên bảng lớn CHƯA thấy
> (d) chưa trải mạng đứt   → CAP chưa bị thử
>
> ✅ "chạy ổn staging" = không có lỗi HIỂN NHIÊN, KHÔNG = chịu được tải/sự cố thật
> ```
> 🔎 **Đề xuất TL:** **load test** theo tải đỉnh dự kiến, **canary/rollout từng phần**, có **rollback + observability** trước khi full production.

---

## ⑨ Bài tập + tiêu chí tự chấm

**BT1.** Lấy MỘT case study bất kỳ (Bài 9→15), đóng vai TL viết **design review** chỉ ra **≥4 điểm có thể vỡ ở production** + cách đo từng điểm.

> ✅ **Đạt khi:** dùng được **checklist tử huyệt**, mỗi điểm có cách phát hiện/metric.

**BT2.** Viết **phản biện 1 trang** cho đề xuất *"fine-tune LLM cho FAQ sản phẩm cập nhật hàng tuần"*.

> ✅ **Đạt khi:** nêu đúng **bản chất** (tri thức động), đề xuất **RAG**, so chi phí cập nhật & khả năng **trích nguồn**, nêu khi nào **fine-tune** mới hợp.

**BT3.** Tự ráp **"thẻ phán đoán"**: 10 tử huyệt production, mỗi cái 1 dòng *"dấu hiệu sớm + cách phát hiện"*.

> ✅ **Đạt khi:** phủ **hot key, cold start, stampede, pool, partition, retry storm, cost, observability, rollback, clock skew**.

---

## ⑩ Đọc thêm

- **The Pragmatic Programmer** — tư duy phản biện & *"đừng lập trình theo trùng hợp"*.
- **Google SRE Book** — *observability, rollout, postmortem culture*; **Netflix** — *chaos engineering*.
- **AWS/Microsoft Well-Architected Framework** — 5 trụ (vận hành, bảo mật, độ tin cậy, hiệu năng, chi phí) như một **checklist review**.
- **RAG vs fine-tuning** — tài liệu giới thiệu khi nào dùng cái nào (**tri thức động** vs **phong cách**).

---

> 🎯 **Khẩu quyết khép bài (và khép cả mục A):** *Không nhớ để gõ — hiểu để chỉ huy. Hỏi "sai ở đâu, đắt ở đâu, vỡ khi nào, đo bằng gì". Khớp công cụ với bản chất bài toán. "Đúng trên giấy" chưa bao giờ là đủ.*
