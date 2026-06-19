# 🎯 BÀI 15 — Judgment Tech Lead: chống **over-engineering** · **enforce design** · **ADR** · **evolutionary architecture** · kiểm soát **AI-code**

**Phủ:** Q-TL-001 → Q-TL-002 → Q-TL-003 → Q-TL-004 → Q-TL-005 → Q-TL-006 → Q-TL-007 → Q-TL-008

> 💡 **Mental model mở đầu:**
> Đây là **tầng meta trên TOÀN BỘ mục Q** — bài hội tụ. Bài 1–14 dạy bạn *biết các công cụ*; bài này dạy **judgment**:
> - **Khi nào DÙNG / KHÔNG dùng** mọi thứ đã học.
> - **Enforce** nguyên tắc trong team mà không thành bottleneck.
> - **Kiểm soát chất lượng thiết kế của code AI**.
> Đây là phần **phân biệt senior với Tech Lead thật**: không phải biết nhiều pattern hơn, mà biết *chọn đúng và biện minh được*.

---

## ① Mục tiêu & vị trí trong mạch

**Tầng meta trên toàn bộ mục Q**: biết khi nào **DÙNG / KHÔNG dùng** tất cả những thứ **Bài 1–14**, **enforce** nguyên tắc trong team, và **kiểm soát chất lượng thiết kế của code AI**.

```
   Bài 1–14: các CÔNG CỤ (coupling/cohesion, SOLID, GoF, kiến trúc, DDD...)
                    │  TẦNG META
                    ▼
   Bài 15: JUDGMENT — chọn đúng, enforce, ghi lại, kiểm soát AI  ← bài này
                    │
              phân biệt SENIOR với TECH LEAD thật
```

> 🎯 **Chốt vị trí:** đây là phần **phân biệt senior với Tech Lead thật**. Senior *dùng tốt* công cụ; Tech Lead *quyết định khi nào dùng*, *enforce*, và *biện minh được*.

---

## ② Giảng cơ bản → nâng cao

### (a) Khi nào KHÔNG dùng một design pattern (Q-TL-001 — nguyên tắc chung)

> **Ví dụ trực giác:** có cả hộp đồ nghề xịn không có nghĩa mỗi cái đinh đều phải dùng máy khoan. Đôi khi cây búa (hoặc bàn tay) là đủ.

**Khi nào KHÔNG dùng:**
- pattern thêm **indirection/abstraction** mà **chưa có biến thiên / đau thật** (**YAGNI**).
- đội **chưa quen** → **giảm dễ đọc**.

> 🎯 **Chốt:** chọn pattern để **giải vấn đề cụ thể**, **không vì "cho xịn"**. Ưu tiên **đơn giản (KISS)**.

---

### (b) Over- vs under-engineering (Q-TL-002)

> 🔎 **Hai cực ngược nhau — phải tránh CẢ HAI:**

| | **Over-engineering** | **Under-engineering** |
|---|---|---|
| Dấu hiệu | nhiều **layer** / **interface 1-1** / **abstraction không có implementation thứ 2** / **flexibility cho tương lai chưa đến** | **copy-paste** / **coupling chặt** / **không có seam để test** |
| Bệnh | làm **thừa** | làm **thiếu** |

> 🎯 **Tìm điểm cân theo bối cảnh:** **độ phức tạp / vòng đời / đội**. Không có "mức đúng tuyệt đối" — chỉ có "đúng cho bối cảnh này".

---

### (c) Dẫn dắt thảo luận thiết kế (Q-TL-003)

> **Tình huống:** một dev giỏi muốn áp **DDD + hexagonal + CQRS** cho một **CRUD nội bộ nhỏ**.

> 🎯 **Cách xử lý (kỹ năng Tech Lead):** **dẫn dắt bằng câu hỏi giá trị/chi phí**, gắn với **độ phức tạp thực + vòng đời + đội**. **Không chặn cứng cũng không chiều.** Có thể đề xuất **"bắt đầu đơn giản, để seam để tiến hoá"**. **Ra quyết định có lý do.**

> 💡 **Vì sao dẫn dắt thay vì ra lệnh?** Vì dev giỏi cần *hiểu lý do* để tâm phục; ra lệnh "không, dùng layered đi" làm họ ức chế. Hỏi *"CRUD này dự kiến phức tạp lên không? Có invariant nào không?"* giúp họ **tự thấy** khi nào cái đắt là không đáng.

---

### (d) ⚠️ Enforce nguyên tắc thiết kế không thành bottleneck review (Q-TL-004)

> **Ẩn dụ:** thay vì cử một người **đứng gác cổng** soi từng PR (mệt, dễ sót, thành nút thắt), hãy **lắp cửa từ tự động** — ai vi phạm thì cửa không mở.

**Công cụ hoá thay vì phụ thuộc con người:**

| Cơ chế | Ví dụ |
|---|---|
| **Architecture lint / boundary trong CI** | **dependency-cruiser**, **eslint-plugin-boundaries**, **ESLint no-restricted-imports**, **Sheriff**, **Nx boundaries** (**đã verify**) — chặn `domain/` import `infra/` tự động |
| Quy trình nhẹ | **ADR + template**, **design review nhẹ**, **pairing/mentoring** |

> 🎯 **Biến nguyên tắc thành guardrail tự động**, giảm phụ thuộc **người gác cổng**. *(verify tên/cấu hình tooling khi học.)*

---

### (e) ADR — Architecture Decision Record (Q-TL-005)

> **Ví dụ trực giác:** 6 tháng sau, người mới hỏi *"sao chỗ này dùng hexagonal?"* — không ai nhớ. **ADR** là cuốn nhật ký ghi lại *vì sao* quyết định, để không phải tranh luận lại từ đầu.

**ADR** = tài liệu **ngắn** ghi **quyết định + bối cảnh + các lựa chọn + hệ quả**. Giúp **nhớ "vì sao"**, **onboard nhanh**, **tránh tranh luận lặp**.

> 📌 **ADR tối thiểu gồm 4 phần:** **Context · Decision · Alternatives · Consequences/Status**.

---

### (f) Evolutionary architecture / design for change (Q-TL-006)

> **Ví dụ trực giác:** xây nhà chừa sẵn *ống chờ* ở chỗ *chắc chắn* sau này lắp thêm điều hoà — nhưng **không** lát sẵn sân bay phòng khi "biết đâu cần".

**Evolutionary architecture** = thiết kế **dễ đổi** mà **không đoán trước mọi tương lai** (chống **speculative generality**): tạo **seam/abstraction tại điểm THỰC SỰ biến động**, **để hở chỗ chưa rõ** thay vì khoá sớm. **YAGNI + refactor liên tục** khi biến thiên xuất hiện.

> 🎯 **Phân biệt:** **"để dễ đổi"** (seam tại **điểm đau thật**) ≠ **"đoán mò tương lai"** (abstraction cho **nhu cầu chưa tồn tại**). Cái đầu là khôn ngoan; cái sau là over-engineering trá hình.

---

### (g) ⚠️🤖 Kiểm soát chất lượng thiết kế của AI-code (Q-TL-007)

**AI** (Copilot/Cursor) sinh code **"chạy được"** nhưng **vi phạm thiết kế** (nhét logic vào controller, import DB vào domain, lặp pattern sai).

**TL kiểm soát thế nào:**

```
   AI lo CÚ PHÁP  ───────────┐
                             ▼
   NGƯỜI chỉ huy & kiểm tra THIẾT KẾ
        │
        ├─ Guardrail: architecture lint / boundary test
        ├─ PR checklist
        ├─ review tập trung BOUNDARY/COUPLING (KHÔNG sa vào style vặt)
        └─ conventions trong prompt/repo (cho AI biết quy ước kiến trúc)
```

> 🎯 **Câu thần chú:** **"hiểu để chỉ huy, không thuộc để gõ".** *(R-trend; verify công cụ.)*

---

### (h) ⚠️ Principle vs Tool/Pattern/Framework (Q-TL-008) — vì sao TL lý luận ở tầng nguyên tắc

> **Ví dụ trực giác:** **nguyên tắc** là **la bàn** (chỉ hướng Bắc bất biến); **pattern/framework** là **bản đồ một vùng** (đổi khi địa hình đổi). Đi đường dài thì tin la bàn.

| | **Principle** (nguyên tắc) | **Tool/Pattern/Framework** |
|---|---|---|
| Ví dụ | **SOLID**, **coupling/cohesion** | Nest, Prisma, GoF pattern |
| Tính chất | **ổn định**, là cơ sở **"vì sao"** | **đổi theo thời gian**, là **phương tiện** |

> 🎯 **TL phải biện minh quyết định bằng trade-off** (**coupling/cohesion/đổi-được**), **KHÔNG "vì framework bảo thế"**.

> 🔎 **Khi phỏng vấn:** nhận ra ứng viên **thuộc-pattern** (đọc tên vanh vách) vs **hiểu-nguyên-tắc** (giải thích **vì sao** và **khi nào KHÔNG**). Người thứ hai mới là Tech Lead.

---

## ③ ⚠️ Cũ → Mới

| Cũ / phản xạ | Tech Lead nên | Vì sao |
|---|---|---|
| **Áp pattern "cho xịn"/cho đủ bộ** | Pattern để **giải vấn đề cụ thể**; mặc định **KISS/YAGNI** | Tránh **over-engineering** |
| **Enforce thiết kế bằng review thủ công** | **Architecture lint/boundary test** trong CI | **Guardrail tự động**, không bottleneck |
| **Quyết định kiến trúc "trong đầu", không ghi** | **ADR** (context/decision/alternatives/consequences) | Nhớ **vì sao**, tránh tranh luận lặp |
| **Abstraction sẵn cho mọi tương lai** | **Seam tại điểm biến động thật**; để hở phần chưa rõ | Chống **speculative generality** |
| **Tin code AI vì "chạy được"** | **Review boundary/coupling + guardrail lint** | **AI đúng cú pháp ≠ đúng thiết kế** |

---

## ④ Áp dụng thực tế + So sánh bigtech

### Enforce
Thêm **`depcruise --validate`** + **rule boundaries** vào **GitHub Actions**; PR vi phạm `domain → infra` bị **fail CI tự động**.

### ADR
Thư mục **`docs/adr/0001-use-hexagonal.md`** theo **template Michael Nygard** — chuẩn phổ biến nhiều công ty.

### Bigtech / ecosystem (pattern phổ biến — không tuyệt đối)

| Thực hành | Ghi chú |
|---|---|
| **Google** | **readability** + **large-scale lint** |
| **Spotify/ThoughtWorks** | phổ biến **ADR** |
| **"fitness functions"** | (**Building Evolutionary Architecture** — ThoughtWorks) = **test tự động cho thuộc tính kiến trúc** |

> 🔎 **⚠️ Insight sâu:** **AI-code guardrails đang là chủ đề nóng 2025–2026** (**verify công cụ**). Khi AI sinh code ngày càng nhiều, **bottleneck dịch chuyển** từ *"viết code"* sang *"kiểm soát thiết kế của code"*. **fitness functions** + **architecture lint** chính là cách *tự động hoá việc kiểm soát đó* — biến nguyên tắc kiến trúc thành test mà máy chạy mỗi commit.

---

## ⑤ Code thực hành (enforce kiến trúc trong CI)

```javascript
// .dependency-cruiser.cjs — chặn domain import infra (đã verify công cụ tồn tại)
module.exports = {
  forbidden: [
    {
      name: 'domain-no-infra',
      comment: 'Domain layer KHÔNG được phụ thuộc infra (Dependency Rule)',
      severity: 'error',
      from: { path: '^src/domain' },
      to:   { path: '^src/(infra|adapters)' },   // vi phạm -> FAIL CI tự động
    },
    { name: 'no-circular', severity: 'error', from: {}, to: { circular: true } },  // chặn phụ thuộc vòng
  ],
};
// CI:  npx depcruise --config .dependency-cruiser.cjs src
```

```markdown
<!-- ADR tối thiểu: docs/adr/0007-hexagonal-for-order-service.md -->
# 7. Dùng Hexagonal cho Order Service
## Status: Accepted
## Context: domain đặt hàng nhiều invariant, dự kiến thay datastore.
## Decision: tách domain/port/adapter; enforce bằng dependency-cruiser.
## Alternatives: layered đơn giản (loại vì domain phức tạp/biến động).
## Consequences: thêm mapping/interface; bù lại test/thay hạ tầng dễ.
```

> 💡 **Đọc kỹ:** rule `domain-no-infra` biến **Dependency Rule (Bài 11)** từ "lời nhắc trong review" thành **luật máy bắt** — PR nào để `domain/` import `infra/` là **đỏ CI ngay**, không cần ai nhớ soi. ADR 4 phần ghi lại *vì sao* chọn hexagonal, để 6 tháng sau không tranh luận lại.

---

## ⑥ Keywords cần nhớ

> 🧠 **Cần HIỂU:**
> - **khi nào KHÔNG dùng pattern**
> - **over/under-engineering**
> - **dẫn dắt thảo luận**
> - **enforce bằng tooling**
> - **ADR**
> - **evolutionary architecture / speculative generality**
> - **AI-code guardrails**
> - **principle vs tool**

> 📌 **Cần THUỘC:**
> - **4 phần ADR** (Context/Decision/Alternatives/Consequences).
> - Dấu hiệu **over-engineering** (interface 1-1, abstraction không impl thứ 2).

> 🛠️ **Cần LÀM ĐƯỢC:**
> - Cấu hình một **rule boundary lint**.
> - Viết một **ADR**.
> - Phản hồi **"áp full DDD cho CRUD"** mang tính **dẫn dắt**.

---

## ⑦ Mental model

> 🎯 **Khẩu quyết:**
> **Nguyên tắc là la bàn (ổn định), pattern/framework là phương tiện (thay được).** TL **biện minh bằng trade-off**, **enforce bằng guardrail tự động** chứ không bằng **người gác cổng**, và **ghi lại "vì sao" (ADR)**. Với AI: **hiểu để chỉ huy & kiểm tra, không thuộc để gõ.**

```
   Định thêm pattern?  → giải vấn đề gì? chưa đau → ĐỪNG (KISS/YAGNI)
   Enforce nguyên tắc? → boundary lint trong CI (máy gác), không người gác
   Quyết định lớn?     → ghi ADR 4 phần (nhớ vì sao)
   Code AI gửi tới?    → review BOUNDARY/COUPLING, không style vặt
   Biện minh?          → bằng trade-off nguyên tắc, không "framework bảo thế"
```

---

## ⑧ Câu hỏi phỏng vấn & thách đố

**(rất khó)** *Khi nào KHÔNG nên dùng một design pattern (nguyên tắc chung)?*
→ khi thêm **indirection** mà chưa có **biến thiên/đau thật**; đội chưa quen; **KISS/YAGNI**.

**(khó)** *Dấu hiệu over-engineering cụ thể?*
→ **interface 1-1**, **abstraction không impl thứ 2**, **flexibility cho tương lai chưa đến**.

**(khó)** *ADR là gì, gồm gì?*
→ **context/decision/alternatives/consequences**; nhớ **"vì sao"**.

**(rất khó)** *Enforce dependency rule trong team đông không thành bottleneck — cách?*
→ **công cụ hoá** (**boundary lint trong CI**) + **ADR** + **mentoring**.

**(rất khó)** *Vì sao TL phải lý luận ở tầng nguyên tắc thay vì framework?*
→ **nguyên tắc ổn định** và là cơ sở **"vì sao"**; **framework đổi** theo thời gian.

> 🧩 **🔥 Thách đố (Q-TL-007):** *"AI sinh code chạy được — merge luôn cho nhanh?"*
>
> **Bẫy:** **"chạy được" ≠ đúng thiết kế**; AI hay **nhét logic vào controller / import DB vào domain**. Cần **guardrail lint + review boundary/coupling**. **Người chỉ huy & kiểm tra, AI lo cú pháp.** Trả lời "merge cho nhanh" là rơi bẫy nguy hiểm nhất của thời AI-code.

---

## ⑨ Bài tập + tiêu chí tự chấm

**BT1.** Thêm **1 rule dependency-cruiser/eslint-plugin-boundaries** chặn `domain → infra` vào repo bạn + chạy trong **CI**.
→ ✅ **Đạt khi:** một PR **cố tình vi phạm** bị **fail tự động**.

> 💡 *Gợi ý:* dùng đúng config ở mục ⑤, rồi tự tạo một file `domain/` import thử `infra/` để xác nhận CI đỏ.

**BT2.** Viết **1 ADR** cho một quyết định kiến trúc gần đây (đủ **4 phần**).
→ ✅ **Đạt khi:** người mới đọc **hiểu vì sao chọn**, **đã cân nhắc gì**.

---

## ⑩ Đọc thêm

- **Michael Nygard** — **"Documenting Architecture Decisions"** (ADR).
- **Neal Ford et al.** — **Building Evolutionary Architecture** (**fitness functions**).
- **R. C. Martin** — **SOLID/Clean**.
- **Công cụ (đã verify):** **dependency-cruiser**, **eslint-plugin-boundaries**, **Sheriff**, **Nx module boundaries**.

---

## 🤖 Hiểu để chỉ huy (toàn mục Q hội tụ ở đây)

> 🎯 **Cả mục Q hội tụ ở đây:** AI viết **cú pháp pattern** rất nhanh, nhưng **chọn pattern nào**, **khi nào KHÔNG dùng**, và **giữ boundary/coupling** là việc của **bạn**.

**Cặp ví dụ kinh điển "cú pháp đúng, thiết kế/hành vi sai":**

| AI làm | Sai ở đâu | Bạn phải bắt |
|---|---|---|
| để **`temperature=0.7`** cho **phân loại** | cú pháp đúng, **hành vi sai** (phân loại cần xác định) | hiểu *vì sao* cần `temperature` thấp |
| chọn **fine-tune** cho dữ liệu **hay đổi** (đáng lẽ **RAG**) | hiểu sai *định hình yêu cầu* | nhận ra dữ liệu động → RAG |
| **import ORM vào domain** | **cú pháp đúng, KIẾN TRÚC sai** (vi phạm Dependency Rule — Bài 11) | bắt bằng **boundary lint** |

> 🎯 **Chốt cả mục Q:** AI là thợ gõ cú pháp xuất sắc. Bạn là **người chỉ huy** — cầm **la bàn nguyên tắc** (coupling/cohesion, SOLID), **chọn phương tiện đúng**, **enforce bằng guardrail tự động**, **ghi lại vì sao (ADR)**, và **kiểm tra thiết kế** chứ không chỉ "chạy được". Đó là toàn bộ điều phân biệt một người *dùng AI* với một *Tech Lead chỉ huy AY*.
