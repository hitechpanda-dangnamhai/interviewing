# 🎯 BÀI 6 — Consistency Models & Eventual Consistency

**Phủ:** I-CONS-001 → I-CONS-012 · **Phổ nhất quán chi tiết — mở rộng chữ "C" của Bài 5.**

> 💡 **Cách đọc bài này.** Bài 5 nói "consistency là per-operation". Bài 6 đưa cho bạn *cái thước đo* để chọn đúng mức: một **phổ** từ mạnh đến yếu. Câu cần khắc cốt: ***chọn mức YẾU NHẤT mà nghiệp vụ vẫn đúng — vì mạnh hơn = phối hợp nhiều hơn = chậm/đắt hơn.*** Và một bẫy thuật ngữ đắt giá nhất bộ: **linearizability ≠ serializability** (hai chữ "serial..." khác trục hoàn toàn).

---

## ① Mục tiêu & vị trí trong mạch

Học xong bài này bạn sẽ làm được 5 việc:

1. **Xếp được phổ nhất quán** từ **mạnh → yếu**.
2. **Định nghĩa linearizability** và **phân biệt với serializability**.
3. **Hiểu các session guarantee** (**read-your-writes**, **monotonic reads**, **causal**).
4. **Đọc được công thức quorum R+W>N.**
5. **Gắn đúng mức consistency vào đúng nghiệp vụ.**

> 🎯 **Vị trí trong mạch học.** Bài này **làm sâu chữ "C"** của **CAP** (Bài 5); **quorum** học ở đây sẽ được **consensus thực thi** ở **Bài 7**.

**Ẩn dụ định vị:** Bài 5 cho bạn la bàn "đúng hay sống". Bài 6 cho bạn *cái thước có vạch* — không chỉ "mạnh/yếu" mà *mạnh tới mức nào*, mỗi vạch tốn bao nhiêu, và vạch nào vừa đủ cho việc của bạn.

---

## ② Giảng cơ bản → nâng cao

### (a) Phổ nhất quán — **I-CONS-001**

> **Ví dụ trực giác:** Như độ "tươi" của thông tin. *Bảng tỉ số trực tiếp* phải đúng tới từng giây (mạnh); *lượt xem một video* hơi trễ vài giây cũng chẳng sao (yếu). Bạn trả tiền cho độ tươi — càng tươi càng đắt.

Từ **mạnh → yếu**:

```
linearizable  →  sequential  →  causal  →  eventual
   (mạnh nhất)                            (yếu nhất)
   đắt/chậm nhất  ──────────────────────►  rẻ/nhanh nhất
   coordination nhiều ───────────────────► coordination ít
```

> 🎯 **Nguyên tắc vàng:** **Càng mạnh càng cần coordination (phối hợp giữa các node) nhiều hơn → càng đắt/chậm. CHỌN MỨC YẾU NHẤT mà nghiệp vụ vẫn đúng.**

---

### (b) Hai mức mạnh & phân biệt khó

#### 🔍 Linearizability — **I-CONS-002**

> **Ví dụ trực giác:** Dù dữ liệu có 3 bản sao, hệ *giả vờ* như chỉ có **một cuốn sổ duy nhất**: ai ghi xong, người đọc *ngay sau đó* chắc chắn thấy.

**Linearizability** = mọi **op** (thao tác) như **xảy ra tức thời tại một điểm** giữa lúc gọi và lúc trả về, **tôn trọng real-time order** — *như thể chỉ có một bản sao*. Nó nói về **recency** (độ mới): *đọc thấy ghi mới nhất*.

> 🔎 **Vì sao đắt?** Vì cần **round-trip tới majority/leader** mỗi op để chắc chắn mình có giá trị mới nhất.

---

#### 🚨 Linearizability vs Serializability — **I-CONS-003** (⭐⭐⭐⭐⭐, bẫy hay nhất)

> **Ví dụ trực giác:**
> - **Linearizability** lo chuyện *"đồng hồ"* — đọc *một món* có thấy phiên bản **mới nhất** theo *thời gian thực* không.
> - **Serializability** lo chuyện *"sổ sách"* — *nhiều giao dịch* đan xen có cho **kết quả như thể chạy lần lượt** không (bất kể thứ tự thật theo đồng hồ).

| Tiêu chí | **Linearizability** | **Serializability** |
|---|---|---|
| Phạm vi | **single-object** (một đối tượng) | **multi-object** (nhiều object, về **transaction**) |
| Về cái gì | **real-time ordering** / **recency** (đọc thấy ghi mới nhất) | Kết quả **tương đương một thứ tự tuần tự nào đó** |
| Có khớp real-time? | **Có** (bắt buộc) | **Không bắt buộc** khớp real-time |

> 🎯 **Chốt:** **Strict serializable = CẢ HAI** (serializable + tôn trọng real-time). Đây là điểm **nhiều người nhầm** — *hai chữ "serial..." khác TRỤC hoàn toàn*: một trục về **độ mới của 1 object**, một trục về **thứ tự của nhiều transaction**.

---

### (c) Eventual & session guarantees

#### 🔍 Eventual consistency — **I-CONS-004**

> **Ví dụ trực giác:** Tin đồn trong làng. Nếu *ngừng phát tin mới*, cuối cùng cả làng *biết cùng một câu chuyện* — nhưng **không ai hứa lúc nào** mọi người đồng bộ.

**Eventual consistency** = **nếu ngừng ghi**, các replica **cuối cùng hội tụ** về cùng giá trị. *"Eventual"* thường **ms → giây**, bị chặn bởi **replication lag / network**; **KHÔNG đảm bảo thời điểm cụ thể**.

> ⚠️ **Hiểu lầm phổ biến:** *"eventual = đồng bộ trong X giây"* là **sai** — không có deadline, chỉ có *"sẽ hội tụ nếu ngừng ghi"*.

---

#### 🔍 Read-your-own-writes — **I-CONS-005**

> **Ví dụ trực giác:** Bạn vừa đổi ảnh đại diện, reload trang → *vẫn thấy ảnh cũ*. Bực mình! Không phải lưu hỏng — mà lần đọc *trúng một replica chưa kịp cập nhật*.

**Read-your-own-writes** = một **session guarantee** (đảm bảo trong phạm vi phiên): *đọc thấy ghi của chính mình*. **Vi phạm khi** read route sang **replica chưa cập nhật** ("vừa update mà không thấy").

**Timeline — vì sao "vừa ghi không thấy":**

```
T1  user: WRITE avatar='new.jpg' → ghi vào PRIMARY ✅
       ↓  (primary đang replicate sang các replica, có độ trễ)
T2  user: reload → READ route sang REPLICA-2 (chưa nhận bản mới)
       → trả avatar='old.jpg' ❌ user tưởng mất dữ liệu
```

> 💡 **Fix:** **đọc từ primary sau write** / **sticky theo session** / **chờ version token**. → **cross-ref E-REP.**

---

#### 🔍 Monotonic reads — **I-CONS-006**

> **Ví dụ trực giác:** Bạn đọc tin "đã có 50 bình luận", reload lại thấy "30 bình luận" — *thời gian như chạy ngược*. Do hai lần đọc trúng *hai replica lệch nhau*.

**Monotonic reads** = lần đọc sau **không thấy state cũ hơn** lần đọc trước. Thiếu nó → *"thời gian chạy ngược"*: reload thấy data **biến mất** (trúng **replica lệch khác nhau**).

**Timeline — thời gian chạy ngược:**

```
T1  user: READ → trúng REPLICA-1 (đã cập nhật) → thấy "50 bình luận"
T2  user: reload → trúng REPLICA-2 (còn lag)    → thấy "30 bình luận" ❌ lùi về quá khứ
```

> 💡 **Fix:** **sticky replica** (ghim phiên vào *một* replica) / **version**.

---

#### 🔍 Causal consistency — **I-CONS-007**

> **Ví dụ trực giác:** Trong group chat, *câu trả lời* không được hiện ra **trước** *câu hỏi*. Quan hệ nhân–quả phải giữ.

**Causal consistency** = giữ **thứ tự nhân-quả** — **reply không xuất hiện trước message gốc**. *Yếu hơn linearizable* nhưng **đủ cho chat/comment**.

**Timeline — eventual đảo thứ tự nhân quả:**

```
Alice: post "Mất chìa khóa rồi 😭"  (message gốc)
Bob:   reply "Tìm thấy chưa?"        (nhân quả: phụ thuộc message của Alice)

Dưới EVENTUAL (không giữ nhân quả), Carol có thể thấy:
   "Tìm thấy chưa?"   ← reply hiện TRƯỚC ❌ (vô nghĩa)
   "Mất chìa khóa rồi 😭"

Dưới CAUSAL: luôn thấy message gốc TRƯỚC reply ✅
```

---

### (d) Quorum & quyết định nghiệp vụ

#### 🔍 Quorum R+W>N — **I-CONS-008**

> **Ví dụ trực giác:** N=3 thùng phiếu. Bạn *bỏ phiếu* (write) vào ít nhất 2 thùng, và *kiểm phiếu* (read) ở ít nhất 2 thùng. Vì 2+2 > 3, **chắc chắn có ÍT NHẤT một thùng vừa được bỏ phiếu vừa được kiểm** → bạn không thể bỏ sót phiếu mới nhất.

**Quorum R+W>N** = nếu **tập node đọc (R)** + **tập node ghi (W)** vượt **tổng N** thì hai tập **giao nhau** → đọc **chắc chắn chạm ít nhất một node có ghi mới nhất**.

**Timeline — vì sao giao điểm cứu stale read (N=3, W=2, R=2):**

```
node:   [n1]  [n2]  [n3]
WRITE x=5 → ghi n1, n2 ✅ (W=2)
READ  x   → đọc n2, n3       (R=2)
                ↑
          n2 nằm trong CẢ tập ghi lẫn tập đọc → đọc thấy x=5 ✅ (không stale)
```

**Tunable (điều chỉnh R/W):**

| Cấu hình | Đặc tính | Khi nào |
|---|---|---|
| **W=N** | **Đọc nhanh** (R nhỏ), ghi chậm (mọi replica phải nhận) | Đọc-tối-ưu |
| **R=N** | **Ghi nhanh** (W nhỏ), đọc chậm | Ghi-tối-ưu |
| **QUORUM** (R=W=⌈(N+1)/2⌉) | **Cân bằng** | Mặc định an toàn |

---

#### 🔍 Chọn mức theo nghiệp vụ — **I-CONS-009**

| Nghiệp vụ | Mức nên chọn | Vì sao |
|---|---|---|
| **Số dư ví** | **strong / linearizable** | Tiền **không được sai** |
| **Đếm like** | **eventual** | Lệch tạm **OK** |
| **Trạng thái đơn hàng** | **strong** cho trạng thái *quyết định*, **eventual** cho hiển thị *phụ* | Tách phần quan trọng khỏi phần phụ |

> 🎯 **Chốt:** **Gắn lựa chọn vào yêu cầu, KHÔNG cảm tính.**

---

#### 🔍 Tunable consistency — **I-CONS-010**

| | Lợi | Rủi ro |
|---|---|---|
| **Tunable consistency** | Trả **đúng chi phí cho đúng nhu cầu** | Dev **chọn sai mức** cho path quan trọng → **bug khó tái hiện** |

> ⚠️ **Cần:** **policy rõ** + **test cho path nhạy cảm** (tiền, tồn kho).

---

#### 🚨 "Cứ strong cho chắc" — phản biện — **I-CONS-011**

> 🚨 **Đừng mặc định strong toàn cục.** **Strong** = **latency cao** + **giảm availability khi partition** + **throughput thấp**. **Nhiều path KHÔNG cần.**

---

#### 🔍 Replication lag gây lớp lỗi nào — **I-CONS-012**

**Replication lag** (độ trễ sao chép) là *thủ phạm chung* của ba lớp lỗi:
- **stale read** (đọc dữ liệu cũ),
- **mất read-your-writes** (vừa ghi không thấy),
- **non-monotonic reads** (thời gian chạy ngược).

> 💡 **Giảm bằng:** **read-from-primary** cho path nhạy cảm, **sticky session**, **version/token** chờ replica, hoặc **chấp nhận + UX phù hợp**. → **cross-ref E-REP.**

> 📘 **vs** ➕ — *docs gốc* chạm **"eventual consistency"** sơ; **phổ đầy đủ**, **session guarantees**, **linearizable-vs-serializable**, **quorum** là phần ➕ chiều sâu.

---

## ③ ⚠️ Kiến thức cũ / cách làm bị thay thế

| Cái cũ / hiểu lầm | Thay bằng (đúng) | Vì sao |
|---|---|---|
| *"Eventual = sẽ đồng bộ trong X giây"* (đảm bảo thời điểm) | **Eventual = hội tụ nếu ngừng ghi**, không có deadline | Bị chặn bởi **lag/network**, không hứa mốc |
| Lẫn **linearizability** với **serializability** | **single-object/real-time** vs **multi-object/transaction-order** | Hai trục khác nhau; **strict-serializable = cả hai** |
| *"User không thấy update của mình → bug ghi"* | Thường là route sang **replica lag** (mất **read-your-writes**) | Sửa ở **tầng routing**, không phải ghi |
| Mặc định **strong consistency** toàn hệ | **Chọn mức yếu nhất đủ đúng per-path** | Strong **đắt/chậm/giảm availability** |
| Đọc **1 node** coi là *"đủ mới"* | Hiểu **R+W>N** để đảm bảo **giao điểm** | Đọc thiếu quorum → **stale** |

---

## ④ Áp dụng thực tế + So sánh bigtech

**Use case — bình luận biến mất:** sau khi user đăng bình luận, họ reload mà **không thấy** → kinh điển **mất read-your-writes** do đọc **replica chưa kịp đồng bộ**.
**Fix:** route đọc *của chính user đó* về **primary** trong **N giây** sau write, hoặc gửi kèm **version token** để client *chờ replica bắt kịp*.

**Bigtech pattern:**
- **Dynamo-style** (**DynamoDB / Cassandra**) cho chọn **R/W quorum per-operation**.
- Nhiều **hệ social** cố tình dùng **eventual** cho **feed/counter** để đạt **throughput khổng lồ**, và **causal/strong** cho phần nhạy cảm (**tin nhắn, thanh toán**).
- **Đồng hồ logic / version vector** dùng để giữ **thứ tự nhân quả**.

> ⚠️ **(Pattern phổ biến; verify từng hệ.)**

---

## ⑤ Code thực hành + cấu hình

**Read-your-own-writes — route về primary sau khi vừa ghi:**

```javascript
// ✅ Read-your-own-writes: route đọc về primary ngay sau khi user vừa ghi
async function getProfile(userId: string, justWrote: boolean) {
  // Nếu user vừa ghi trong cửa sổ ngắn → đọc PRIMARY để thấy ngay (không trúng replica lag)
  const conn = justWrote ? primaryDb : replicaDb;   // sticky-to-primary TẠM THỜI
  return conn.query('SELECT * FROM profiles WHERE user_id = $1', [userId]);
}

// 🧠 Chỉ-huy-AI: AI hay route MỌI read sang replica để "scale". Người hiểu biết
//    phải tách path "vừa-ghi-của-tôi" về primary, nếu không user kêu "mất data".
```

**Quorum R + W > N (minh hoạ N=3):**

```
W=2, R=2  → 2+2=4 > 3  ✅ đảm bảo đọc chạm node có ghi mới nhất (cân bằng, QUORUM)
W=3, R=1  → đọc nhanh, ghi chậm (mọi replica phải nhận write)   (đọc-tối-ưu)
W=1, R=3  → ghi nhanh, đọc chậm                                  (ghi-tối-ưu)
W=1, R=1  → 1+1=2 < 3  ❌ KHÔNG đảm bảo — có thể stale (eventual)
```

> 🔧 **Cấu hình:**
> - **Tài liệu hoá** mức consistency + **routing policy** cho từng nhóm endpoint.
> - **Test path tiền/tồn kho** ở mức **strong**.
> - **Theo dõi replication lag** (đặt **alert**) để biết **staleness window thực tế** và chọn **cửa sổ sticky-to-primary** hợp lý.

---

## ⑥ Keywords cần nhớ (glossary)

**🧠 Cần HIỂU (nắm bản chất):**
- **phổ** **linearizable → sequential → causal → eventual**.
- **linearizability vs serializability** — recency-1-object vs order-nhiều-transaction.
- **read-your-writes** — đọc thấy ghi của chính mình.
- **monotonic reads** — không thấy state cũ hơn lần trước.
- **causal consistency** — giữ thứ tự nhân quả.
- **quorum intersection** — R+W>N → hai tập giao nhau.
- **"yếu nhất mà vẫn đúng"** — nguyên tắc chọn mức.

**📌 Cần THUỘC (nhớ nguyên văn):**
- **R+W>N**.
- **strict-serializable = linearizable + serializable**.
- *"eventual = hội tụ nếu ngừng ghi"*.
- **QUORUM = ⌈(N+1)/2⌉**.

**🛠️ Cần LÀM ĐƯỢC (kỹ năng tay):**
- Chẩn đoán *"mất read-your-writes"* do **replica lag**.
- Chọn **R/W**.
- Gán **mức consistency per-path**.

---

## ⑦ Mạch tư duy cần nhớ (mental model)

> 🎯 **Khẩu quyết 1 — phổ, không phải công tắc:**
> *"**Consistency là một PHỔ, không phải on/off.** Mạnh hơn = coordination nhiều hơn = đắt hơn → **chọn mức YẾU NHẤT mà nghiệp vụ vẫn đúng**."*

> 🎯 **Khẩu quyết 2 — đừng lẫn hai trục:**
> *"**Linearizability** nói về **recency của 1 object**; **serializability** nói về **thứ tự của nhiều transaction** — đừng lẫn."*

---

## ⑧ Câu hỏi phỏng vấn & thách đố

**(TB)** Kể **phổ consistency** từ mạnh đến yếu, cái mạnh **đắt hơn vì sao**?
→ *Gợi ý:* linearizable→sequential→causal→eventual; mạnh cần coordination/round-trip nhiều hơn.

**(khó)** **Linearizability** là gì (1 câu)? **Vì sao đắt**?
→ *Gợi ý:* đọc thấy ghi mới nhất như một bản duy nhất; đắt vì round-trip tới majority/leader.

**(rất khó)** **Linearizability vs serializability** khác gì? **Strict serializable** là gì?
→ *Gợi ý:* 1-object/real-time vs multi-transaction/order; strict = cả hai.

**(khó)** **Read-your-writes** là gì? Vì sao *"vừa update mà không thấy"*?
→ *Gợi ý:* session guarantee; do đọc trúng replica lag → fix route primary/sticky.

**(khó)** **R+W>N** nghĩa là gì? Tune thế nào cho **đọc nhanh / ghi nhanh**?
→ *Gợi ý:* hai tập giao nhau; W=N đọc nhanh, R=N ghi nhanh.

> 🧩 **Thách đố / trick — AI hay sai ở đây:**
> *"Reload trang thấy data lúc có lúc không, 'thời gian chạy ngược' — bug gì?"*
> → thiếu **monotonic reads** do trúng các **replica lệch nhau**; fix **sticky replica / version**.
>
> *"Cứ chọn strong consistency cho chắc đúng không?"*
> → ❌ **không**; strong **đắt/chậm/giảm availability**, nhiều path không cần — đây là **AI hay mặc định strong**, người hiểu phải **tiết chế**.

---

## ⑨ Bài tập thực hành + tiêu chí tự chấm

**BT1 — Tái hiện rồi fix mất read-your-writes.**
Ghi **primary**, đọc **replica có lag** rồi fix bằng **sticky-to-primary**.
> ✅ **Đạt khi:** chứng minh được hiện tượng *"vừa ghi không thấy"* → fix → thấy ngay, **giải thích bằng lời** tại sao.

**BT2 — Với N=5, liệt kê các cặp (R,W) thỏa R+W>N.**
Và **gọi tên trade-off** mỗi cặp.
> ✅ **Đạt khi:** đúng điều kiện **giao quorum** + giải thích cặp nào **tối ưu đọc / ghi / cân bằng**.

---

## ⑩ Đọc thêm / nguồn chuẩn

- **DDIA ch.5 & 9** (consistency models, linearizability, ordering).
- **Jepsen — Consistency Models** (sơ đồ phổ kinh điển).
- **Dynamo paper** (quorum R/W).
- **Docs DB bạn dùng** (read/write concern, consistency level, replica routing).

---

> 🔚 **Hết Bài 6.** → **Bước 3 — Chốt ghi nhớ.**
>
> 🎯 **Tâm điểm:** **linearizability ≠ serializability**, **R+W>N**, **session guarantees**. Gợi ý 3 câu tự hỏi: *(1) Phổ consistency mạnh→yếu, vì sao mạnh đắt hơn? (2) Hai chữ "serial/linear" khác trục thế nào, strict-serializable là gì? (3) "Vừa ghi không thấy" là vi phạm gì, fix sao?*
