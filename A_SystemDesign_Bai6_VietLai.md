# 🎯 BÀI 6 — Trade-off articulation, CAP & PACELC

**Phủ:** A-TO-001 → A-TO-007

> 💡 **Tư tưởng cốt lõi của cả bài:** Trong system design **không có "tốt nhất"**, chỉ có **"phù hợp nhất với requirement"**. Mọi lựa chọn đều **cắt một thứ để được thứ khác**. Bài này dạy bạn **nói ra điều đó thành lời** — kỹ năng phân biệt **senior** với một **TL (tech lead)** thật.

---

## ① Mục tiêu & vị trí trong mạch

Bài này **mở khối "Đánh đổi & ranh giới"** — và là bài phân biệt **senior** với **TL** thật.

> 🎯 **Vì sao quan trọng đến vậy?** Vì mọi quyết định ở **case study (Bài 9–15)** đều phải **phát biểu được bằng ngôn ngữ trade-off**. Học xong, bạn có **khung biện luận** để trả lời thuyết phục câu kinh điển: *"Vì sao A mà không B, anh đánh đổi gì?"* — câu mà interviewer hỏi để tách người *biết làm* khỏi người *biết vì sao*.

---

## ② Giảng cơ bản → nâng cao

### (a) Trực giác

> **Ẩn dụ:** Chọn xe. Không có "xe tốt nhất" — xe thể thao nhanh nhưng tốn xăng & chở được ít; xe tải chở khoẻ nhưng chậm. Người sành không khoe *"xe A tốt hơn"*, họ nói *"chở hàng nặng đường xa thì tôi chọn xe tải, chấp nhận đi chậm"*.

> 🎯 **Chốt:** **TL giỏi không khoe "A tốt hơn"** — họ nói *"với requirement X, tôi chọn A, chấp nhận mất Y"*.

---

### (b) Cơ chế

#### 📘 **A-TO-001 — CAP theorem & vì sao "chọn 2 trong 3" gây hiểu lầm**

> **Ví dụ trực giác:** Hai chi nhánh ngân hàng mất liên lạc với nhau (**network partition**). Lúc đó bạn buộc phải chọn: *vẫn cho rút tiền* (ưu **availability**, rủi ro hai bên cùng rút quá số dư) hay *khoá lại chờ nối lại* (ưu **consistency**, nhưng tạm không phục vụ).

**CAP** gồm **C** (consistency), **A** (availability), **P** (partition tolerance). Vì sao *"chọn 2 trong 3"* sai?

> 🔍 **Mấu chốt:** **P (network partition) luôn có thể xảy ra** trong hệ phân tán — nó **không phải lựa chọn**. Thực chất: **khi CÓ partition, phải chọn C hay A**. Còn lúc **không** partition, vẫn có đánh đổi C/A khác.

> ➕ **PACELC mở rộng đúng hơn:**
> ```
> if Partition  then (C or A)      ← khi mạng đứt: chọn nhất quán hay sẵn sàng
> Else                (L or C)      ← khi mạng BÌNH THƯỜNG: chọn latency thấp hay nhất quán cao
> ```
> 🎯 **Ý quan trọng:** kể cả lúc **bình thường**, **strong consistency vẫn đánh đổi latency**. Không phải *"vứt hẳn 1 cái vĩnh viễn"* như cách hiểu "2 trong 3".

#### 📘 **A-TO-002 — Strong vs eventual consistency**

> **Ví dụ trực giác:** Số dư tài khoản **phải đúng ngay** (không ai chịu thấy tiền sai) — **strong**. Còn số like một bài post trễ vài giây thì **chẳng ai chết** — **eventual**.

| Tiêu chí | **Strong consistency** | **Eventual consistency** |
|---|---|---|
| Dùng cho | **tiền / tồn kho / đặt vé** (phải đúng ngay) | **feed / like / đếm view** (trễ vài giây vô hại) |
| Latency | ↑ cao hơn | nhanh |
| Availability khi partition | ↓ giảm | sẵn sàng |
| Rủi ro | (chậm hơn) | có thể **đọc data cũ (stale)** |

> ⚠️ **Cảnh báo:** Phải **nêu ví dụ cụ thể** (tồn kho, like…), **không** nói lý thuyết suông. Câu trả lời "tôi chọn strong consistency" mà không gắn ví dụ = rỗng.

#### 📘 **A-TO-004 — Tứ giác latency / throughput / cost / consistency**

> **Ẩn dụ:** Như cái chăn hẹp — kéo che chân thì hở vai. Cải thiện một góc thường **hy sinh** góc khác.

| Lựa chọn | Được | Mất |
|---|---|---|
| **strong consistency** | đúng tuyệt đối | ↑ **latency** |
| **cache** | ↓ latency | **stale** (data cũ) |
| **replica** | ↑ **availability** | ↑ **cost** & ↑ độ phức tạp đồng bộ |

> 🎯 **Chốt:** **TL chọn theo ưu tiên nghiệp vụ**, không mơ *"tốt mọi mặt"*.

#### 📘 **A-TO-005 — SQL vs NoSQL** *(giao với E)*

> 🚨 **Cấm:** chọn NoSQL chỉ vì *"NoSQL nhanh hơn"* — câu này **vô nghĩa** nếu không gắn workload.

Quyết theo **access pattern** + yêu cầu **consistency** + **mô hình dữ liệu** + **scale ghi**.

> 💡 **Mặc định an toàn:** **Postgres** đúng cho phần lớn ca. Chọn **NoSQL** khi có **lý do cụ thể**: **scale ghi cực lớn**, **schema linh hoạt**, hoặc **key-value đơn giản**.

#### 📘 **A-TO-006 — Monolith vs microservices**

> **Ví dụ trực giác:** Một quán nhỏ thì một đầu bếp lo hết (monolith) là gọn. Tách thành 5 quầy chuyên biệt (microservices) chỉ hợp lý khi quán đủ lớn — nếu không, 5 quầy chỉ tốn thêm người điều phối.

> 🚨 **"Microservices mặc định" là sai.** **Micro** thêm phức tạp **network / vận hành / consistency xuyên service**. Chỉ chia khi **CÓ lý do** (**scale độc lập**, **team boundary**). **Modular monolith / monolith** hợp **team nhỏ & giai đoạn sớm** *(đào sâu Bài 7)*.

---

### (c) Khung trả lời trade-off thuyết phục 📘 **A-TO-003**

> 🎯 **Bốn bước (khẩu quyết):** **Nêu tiêu chí → So sánh → Gắn requirement → Nhận nhược điểm.**

1. **Nêu tiêu chí** đang cân (**latency / cost / consistency / complexity / operability**).
2. **So 2 phương án** theo từng tiêu chí (không chỉ nói chung).
3. **Gắn với requirement** đã chốt (*"vì ta cần strong consistency cho tồn kho nên…"*).
4. **Thừa nhận nhược điểm** của phương án chọn (*"đổi lại latency cao hơn ~Xms, chấp nhận được vì…"*).

> 🚨 **KHÔNG bao giờ** nói *"A tốt hơn B"* trống rỗng. Đó là **red flag** lớn nhất ở vòng TL.

---

### Mép giới hạn — over-engineering 📘 **A-TO-007**

> **Ví dụ trực giác:** Mua xe tải 18 bánh để đi chợ mua rau. "Hoành tráng" nhưng vô lý.

Thêm **shard / microservice / queue / đa vùng** khi **chưa cần** = **điểm trừ** ở phỏng vấn TL: **chi phí phức tạp > lợi ích**.

> 🎯 **Nguyên tắc YAGNI** (*You Aren't Gonna Need It*): **bắt đầu đơn giản, scale khi requirement đòi**. Thể hiện **phán đoán**, không **khoe kiến thức**.

> 💡 **Phân biệt 📘 vs ➕:** **CAP**, **strong/eventual**, **SQL/NoSQL**, **monolith/micro**, **over-engineering** đều 📘. **PACELC** và **khung 4 bước articulation** là ➕ làm sâu.

---

## ③ ⚠️ Kiến thức cũ / hay bị làm sai

| Quan niệm sai phổ biến (❌) | ✅ Đúng hơn | Vì sao |
|---|---|---|
| *"CAP: chọn 2 trong 3"* | Khi có **P** thì chọn **C/A**; dùng **PACELC** (kể cả lúc không partition) | **P không phải lựa chọn** — nó luôn có thể xảy ra |
| *"NoSQL nhanh hơn SQL"* | Chọn theo **access pattern / consistency / scale ghi** | *"Nhanh hơn"* tuỳ workload; **Postgres** an toàn mặc định |
| *"Microservices = hiện đại = nên dùng"* | Chia khi **CÓ lý do**; mặc định **modular monolith** | **Micro** thêm chi phí phân tán lớn |
| *"A tốt hơn B"* (không nêu tiêu chí) | Nêu tiêu chí → so → gắn requirement → nhận nhược điểm | **Trade-off trống rỗng = red flag** |

---

## ④ Áp dụng thực tế + So sánh bigtech

**PACELC** giúp phân loại hệ thật:

| Hệ | Nghiêng về | Nghĩa |
|---|---|---|
| **DynamoDB / Cassandra** | **PA / EL** | ưu **availability/latency**, **eventual** |
| Hệ kiểu **Spanner** | **PC / EC** | ưu **consistency**, chấp nhận **latency** cao hơn |

> 💡 Đây là cách **đọc một datastore qua lăng kính trade-off** (pattern phổ biến — **verify** chi tiết sản phẩm).

> ➕ **Monolith-first** là khuyến nghị phổ biến (**Martin Fowler — "MonolithFirst"**): nhiều công ty lớn **tách microservices sau** khi **seam** (đường nối tự nhiên giữa các phần) đã rõ, **không** phải từ ngày đầu.

---

## ⑤ Code thực hành + cấu hình

Bài này là **tư duy**, không code. Dùng **bảng quyết định** điền khi gặp lựa chọn (mang vào phỏng vấn):

```text
# TRADE-OFF DECISION TABLE (điền cho mỗi quyết định lớn)
Quyết định: <SQL vs NoSQL? strong vs eventual? mono vs micro?>
Requirement liên quan đã chốt: <...>

| Tiêu chí       | Phương án A | Phương án B |
|----------------|-------------|-------------|
| Latency        |             |             |
| Consistency    |             |             |
| Cost           |             |             |
| Complexity/Ops |             |             |
| Scale (đọc/ghi)|             |             |

=> CHỌN: <A/B> vì requirement <...>; CHẤP NHẬN mất: <nhược điểm>.
```

> 🔎 **Vì sao điền bảng tốt hơn nói miệng?** Vì bảng **ép bạn so theo từng tiêu chí** thay vì cảm tính, và dòng cuối **ép thừa nhận nhược điểm** — chính là bước 3 và 4 của khung articulation. Cấu trúc này làm câu trả lời của bạn nghe như một TL, không như người mới.

---

## ⑥ Keywords cần nhớ

🧠 **Cần HIỂU:**
- **CAP** đúng nghĩa (**P luôn có thể xảy ra**).
- **PACELC**.
- vì sao **mọi lựa chọn là đánh đổi**.
- **YAGNI / over-engineering**.

📌 **Cần THUỘC:**
- ví dụ **strong** (tiền/tồn kho/vé) vs **eventual** (feed/like/view).
- **4 trục** (**latency/throughput/cost/consistency**).
- **khung 4 bước articulation**.

🛠️ **Cần LÀM ĐƯỢC:**
- trả lời *"vì sao A không B"* theo khung.
- phân loại một **datastore** theo **PACELC**.
- nhận ra **over-engineering**.

---

## ⑦ Mental model

> 🎯 **"Không có tốt nhất, chỉ có phù hợp nhất."**
> **Nêu tiêu chí → so sánh → gắn requirement → nhận nhược điểm. PACELC > CAP "2 trong 3".**

---

## ⑧ Câu hỏi phỏng vấn & thách đố

**(TB)** **CAP** nói gì, vì sao *"chọn 2 trong 3"* gây hiểu lầm?
→ khi có **partition** mới phải chọn **C/A**; **P không phải lựa chọn**; dùng **PACELC**.

**(TB)** **Strong vs eventual** — ví dụ mỗi loại?
→ **strong**: tồn kho/thanh toán; **eventual**: feed/like/view.

**(khó)** **SQL vs NoSQL** quyết theo gì?
→ **access pattern + consistency + mô hình data + scale ghi**; không phải *"nhanh hơn"*.

**(khó)** Vì sao *"microservices mặc định"* là sai?
→ thêm **chi phí phân tán**; chia khi có lý do (**scale / team boundary**).

**(rất khó)** Khung trả lời *"vì sao A không B"*?
→ **tiêu chí → so sánh → gắn requirement → thừa nhận nhược điểm**.

> 🧩 **Thách đố (trick):** Ứng viên nói *"em chọn microservices + Kafka + đa region cho MVP 100 user."* Phản biện?
>
> ```
> ❌ Khoe kiến trúc lớn:  micro + Kafka + multi-region  cho 100 user
>        ↓
> Soi chi phí/lợi ích:
>   - 100 user  → KHÔNG cần phân tán
>   - chi phí vận hành + consistency xuyên service  >  lợi ích
>        ↓
> ✅ TL chọn:  modular monolith + 1 DB
>             → scale KHI requirement đòi (YAGNI)
> ```
> 🔎 **Vì sao là bẫy?** Đây là **over-engineering** kinh điển: **100 user không cần phân tán**; chi phí vận hành/consistency **lớn hơn** lợi ích.

---

## ⑨ Bài tập + tiêu chí tự chấm

**BT1.** Cho yêu cầu *"đếm view video tỷ lượt"*. Dùng **decision table** chọn **strong vs eventual consistency**.

> ✅ **Đạt khi:** chọn **eventual**, gắn lý do (**trễ vài giây vô hại, ưu latency/availability**), nêu **nhược điểm** (**đếm xấp xỉ**).

**BT2.** Phân loại **Postgres, DynamoDB, Cassandra** theo **PACELC** (P_/E_).

> ✅ **Đạt khi:** giải thích mỗi cái nghiêng **C** hay **A/L** và **vì sao**.

---

## ⑩ Đọc thêm

- **Brewer** — *CAP Twelve Years Later*; **Abadi** — *Consistency Tradeoffs in Modern Distributed Database System Design* (**PACELC**).
- **Martin Fowler** — *MonolithFirst*, *MicroservicePremium*.

---

> 🎯 **Khẩu quyết khép bài:** *Không có "tốt nhất", chỉ có "phù hợp nhất". Nêu tiêu chí → so → gắn requirement → nhận nhược điểm. Đơn giản trước, scale khi cần (YAGNI).*
