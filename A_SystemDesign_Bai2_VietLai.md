# 🎯 BÀI 2 — Capacity & back-of-envelope estimation

**Phủ:** A-CAP-001 → A-CAP-006

> 💡 **Tư tưởng cốt lõi của cả bài:** **Estimation** (ước lượng) **không phải để ra con số đúng**, mà để **ra quyết định kiến trúc**. Bạn nhẩm vài phép tính thô để trả lời những câu hỏi nhị phân: *"Có cần shard không? Có cần cache không? 1 hay nhiều datacenter?"*

---

## ① Mục tiêu & vị trí trong mạch

Bài này tiếp khối **"Nền tảng tư duy"**.

- **Bài 1** dạy bạn **khi nào** ước lượng (đó là **bước 2** trong khung 6 bước).
- **Bài 2** dạy bạn **cách** ước lượng — và quan trọng hơn — **dùng con số để ra quyết định kiến trúc**.

> 🎯 **Mạch nối:** mỗi con số bạn tính ra ở đây sẽ "chỉ đường" sang các bài sau:
> - → **Bài 3:** có cần **scale ngang** (sharding) không?
> - → **Bài 4:** có cần **CDN / cache** không?
> - → **Bài 6:** đánh đổi nào là hợp lý?

> ⚠️ **Lằn ranh dễ nhầm:** Bài này *không* dạy bạn làm kế toán chính xác. Nó dạy bạn ước lượng **đủ thô để quyết định, đủ nhanh để không tốn thời gian phỏng vấn**.

---

## ② Giảng cơ bản → nâng cao

### (a) Trực giác

> **Ẩn dụ:** Bạn đứng trước một hội trường và cần biết *"có cần thuê thêm phòng không?"*. Bạn **không** đếm từng cái ghế — bạn ước "khoảng 500 người, phòng chứa 200" → **cần thêm phòng**. Đếm chính xác 487 hay 512 đều dẫn tới *cùng một quyết định*.

> 💡 **Chốt trực giác:** Mục tiêu là **bậc độ lớn (order of magnitude)**, không phải độ chính xác.
> - **Sai số 2×** → không sao (vẫn ra đúng quyết định).
> - **Sai bậc độ lớn (10×)** → mới chết (chọn nhầm kiến trúc).

> 🔎 **Vì sao chấp nhận sai 2× mà sợ sai 10×?** Vì quyết định kiến trúc có "bậc thang" rộng. "1.000 hay 2.000 QPS" thường cùng nằm trong vùng *"một DB + cache là đủ"*. Nhưng "1.000 hay 100.000 QPS" thì nhảy hẳn sang vùng *"phải shard"*. Sai 10× là sai *bậc thang*, tức sai kiến trúc.

---

### (b) Cơ chế — 4 phép tính lõi

#### 1️⃣ **QPS** (Queries Per Second) 📘 **A-CAP-001**

```
avg QPS  = DAU × (số request/user/ngày) ÷ 86.400
peak QPS ≈ 2–3 × avg QPS        (traffic dồn vào giờ cao điểm)
```

> **Ví dụ cụ thể:**
> ```
> 10M DAU × 10 req/ngày = 100M req/ngày
> 100.000.000 ÷ 86.400  ≈ ~1.150 avg QPS
> peak ≈ 2–3×           ≈ ~2.300 – 3.500 QPS
> ```

> ⚠️ **Nguyên tắc vàng:** **Luôn nói giả định ra tiếng** — *"giả sử mỗi user 10 request/ngày"* — thay vì bịa số im lặng.
>
> 🔍 **Vì sao?** Interviewer chấm **cách suy luận**, không chấm con số. Một số *sai-có-cơ-sở* (giả định rõ ràng, phép tính đúng) vẫn **đạt**; một số *đúng-không-cơ-sở* (bịa) lại trượt.

#### 2️⃣ **Storage** 📘 **A-CAP-002**

```
storage/năm = (item/năm) × (kích thước trung bình) × hệ số
hệ số = replication (×2–3) + metadata
```

> **Ví dụ cụ thể:** `500M ảnh/năm × 1MB × 3 (replica) ≈ 1.5 PB/năm`.

> 🔍 **Vì sao phải nhân hệ số replication?** Vì dữ liệu thật **không lưu một bản**. Để chịu lỗi (durability), mỗi item được sao thành **2–3 bản** (replica) trên các máy khác nhau, cộng thêm **metadata** (index, thông tin phụ). **Quên hệ số này = sai bậc độ lớn ngay.** Và nhớ **dự trù nhiều năm tăng trưởng**, đừng tính một năm rồi dừng.

#### 3️⃣ **Bandwidth** 📘 **A-CAP-005**

```
bandwidth = peak QPS × payload trung bình   (bytes/giây)
```

> 💡 Ra được **throughput** thì biết: có cần **CDN** không, **egress** (dữ liệu đi ra) bao nhiêu — **egress là tiền thật** — và chọn **instance / network** loại nào.

> 🎯 **Chốt:** Đây là tinh thần xuyên suốt — **nối ước lượng với một quyết định cụ thể**, đừng để con số "mồ côi".

#### 4️⃣ **Read:write ratio** 📘 **A-CAP-003**

> **Ví dụ trực giác:** Một tờ báo online — hàng triệu người *đọc*, chỉ vài chục phóng viên *ghi*. Đó là **read-heavy** điển hình.

| **read:write** | Nỗ lực tối ưu đặt ở đâu | Vì sao |
|---|---|---|
| **read ≫ write** (read-heavy) | **cache** + **read replica** + **CDN** | nhiều lượt đọc → nhân bản dữ liệu ra gần người đọc |
| **write ≫ read** (write-heavy) | **sharding** + **queue** + **LSM store** (vd **Cassandra**) | nhiều lượt ghi → chia tải ghi & hấp thụ burst |

> 🔍 **Vì sao ratio "định hình" kiến trúc?** Vì nó cho biết **bạn nên đổ công sức tối ưu vào đâu**. Tối ưu đường ghi cho một hệ read-heavy = lãng phí; và ngược lại. **LSM store** (Log-Structured Merge tree, nền tảng của Cassandra) tối ưu cho ghi tuần tự nhanh — đúng nhu cầu write-heavy.

---

### (c) Latency orders of magnitude 📘⬆️ **A-CAP-004**

> **Cần THUỘC bậc độ lớn** (mức quan trọng ⭐ cao). Đây là các con số kinh điển của **Jeff Dean** — dùng để **biện luận**, **không phải SLA**.

> ⚠️ **Lưu ý (đã verify):** Đây là **bậc độ lớn để lập luận**, không phải cam kết hiệu năng. Đừng trích như con số tuyệt đối.

| Thao tác | Bậc độ lớn |
|---|---|
| **L1 cache** | ~0.5 ns |
| **Main memory (RAM)** | ~100 ns |
| Đọc 1MB tuần tự từ **RAM** | ~250 µs |
| **SSD** random read | ~150 µs |
| Đọc 1MB từ **SSD** | ~1 ms |
| **Round-trip** trong **cùng datacenter** | ~0.5 ms |
| **HDD** seek | ~10 ms |
| **Round-trip xuyên lục địa** (CA ↔ EU) | ~100–150 ms |

> 🎯 **Quy tắc nhớ (khẩu quyết):**
> **RAM (ns) ≪ SSD (µs) ≪ disk/network (ms) ≪ cross-region (chục–trăm ms).**

Dùng để biện luận những điều như:

- **Vì sao cache RAM đáng giá?** Vì RAM nhanh hơn DB disk **~1000×** (ns vs ms) → một **cache** RAM trước DB là đòn bẩy khổng lồ.
- **Vì sao cross-region call phải cẩn thận?** Một round-trip xuyên lục địa ~150ms — gấp **~300×** round-trip trong cùng datacenter. Vài cú call như vậy nối tiếp = trải nghiệm ì ạch.
- **Vì sao đặt replica gần user?** Để biến cross-region (~150ms) thành local (~0.5ms).

---

### Mép giới hạn 📘 **A-CAP-006**

> 🚨 **Cấm:** Đừng ước lượng "trang trí". Chỉ tính con số **dẫn tới một quyết định**.
>
> 🔎 **Tự hỏi mỗi lần định tính:** *"Con số này khiến tôi chọn khác đi không?"* — Nếu **không**, **bỏ**. Mỗi phút trong phỏng vấn là quý; tính số vô dụng = tự cắt thời gian deep-dive *(liên hệ Bài 1 — không sa lầy)*.

---

## ③ ⚠️ Kiến thức cũ / hay bị làm sai

| Hay gặp (❌ sai) | ✅ Nên làm | Vì sao |
|---|---|---|
| Tính ra con số chính xác nhiều chữ số | Làm tròn về **bậc độ lớn** | Mục tiêu là **quyết định**, không phải kế toán |
| Bịa số, không nói **giả định** | Nêu **giả định** to, rõ | Interviewer chấm cách suy luận; số **sai-có-cơ-sở** vẫn đạt |
| Quên hệ số **replication** / **metadata** khi tính **storage** | ×2–3 cho replica + metadata | Thiếu hệ số ⇒ **sai bậc độ lớn** |
| Bỏ qua **peak**, chỉ tính **avg** | **peak ≈ 2–3× avg** | Hệ **sập vì peak**, không sập vì avg |

> 🔍 **Đào sâu dòng cuối:** Vì sao "sập vì peak"? Vì một hệ thống phải sống sót ở *lúc đông nhất* (giờ cao điểm, sự kiện sale, viral). Thiết kế theo **avg** giống may áo vừa lúc bụng đói rồi mặc lúc bụng no — rách ở đúng lúc cần nhất.

---

## ④ Áp dụng thực tế + So sánh bigtech

**Estimation** không chỉ là trò phỏng vấn — nó xảy ra thật ở:

- **Capacity planning** (trước khi launch): tính trước để chuẩn bị hạ tầng.
- **Cost review / FinOps**: rà chi phí định kỳ.

> **Ví dụ thực tế — egress cost:** **bandwidth estimation** quyết định bài toán **egress cost**. Đẩy ảnh/video qua **CDN** thay vì kéo thẳng từ **origin** có thể **cắt chi phí egress đáng kể**. Đây là lý do mọi nền tảng **media-heavy** đều theo pattern **CDN-first** (pattern phổ biến).

> ➕ **So sánh bigtech:** Ở công ty thật, con số **QPS / storage** từ chính phép tính Bài 2 còn được dùng để:
> - chọn **instance type** (loại máy ảo phù hợp), và
> - đặt **autoscaling threshold** (ngưỡng tự co giãn).
>
> 🎯 **Chốt:** Cùng một phép nhẩm — học cho phỏng vấn, dùng luôn cho vận hành thật.

---

## ⑤ Code thực hành + cấu hình

Script luyện **trực giác** (chạy được, không cần thư viện ngoài). Đây là **công cụ tập nhẩm**, không phải thứ gõ trong phòng phỏng vấn:

```python
# capacity_estimate.py — chạy: python capacity_estimate.py
# Không có dependency ngoài; số ra là BẬC ĐỘ LỚN để biện luận, KHÔNG phải SLA.

def estimate(dau, req_per_user_per_day, payload_kb,
             item_per_year, item_size_mb, replication=3, peak_factor=3):
    sec_per_day = 86_400                                    # số giây trong 1 ngày (cần THUỘC)
    avg_qps = dau * req_per_user_per_day / sec_per_day      # B1: DAU -> avg QPS
    peak_qps = avg_qps * peak_factor                        # B2: peak ≈ 2-3× avg
    bandwidth_mb_s = peak_qps * payload_kb / 1024           # B3: throughput MB/s ở PEAK
    storage_pb_year = item_per_year * item_size_mb * replication / 1024 / 1024 / 1024  # B4: MB -> PB

    print(f"avg QPS   ≈ {avg_qps:,.0f}")
    print(f"peak QPS  ≈ {peak_qps:,.0f}  (x{peak_factor})")
    print(f"bandwidth ≈ {bandwidth_mb_s:,.1f} MB/s ở peak  -> cần CDN nếu lớn")
    print(f"storage   ≈ {storage_pb_year:,.3f} PB/năm (đã x{replication} replica)")

    # Quy tắc QUYẾT ĐỊNH (heuristic, KHÔNG phải luật cứng):
    if peak_qps > 10_000: print("=> peak QPS cao: cân nhắc shard / scale ngang DB")
    if bandwidth_mb_s > 100: print("=> bandwidth lớn: CDN gần như bắt buộc")

if __name__ == "__main__":
    # Ví dụ: 10M DAU, 10 req/ngày, payload 5KB; 500M ảnh/năm @1MB
    estimate(dau=10_000_000, req_per_user_per_day=10, payload_kb=5,
             item_per_year=500_000_000, item_size_mb=1)
```

> ⚠️ **Cảnh báo:** Đây là công cụ **luyện trực giác**; trong phỏng vấn bạn **nhẩm tay**, không gõ script. Mục đích là để qua vài lần chạy, con số "in vào đầu" và bạn nhẩm được tức thì.

---

## ⑥ Keywords cần nhớ

🧠 **Cần HIỂU:**
- **"estimate để ra quyết định, không để ra con số đúng"** — linh hồn cả bài.
- Vì sao **read:write ratio** định hình kiến trúc.
- **latency hierarchy** — thứ bậc độ trễ RAM → SSD → disk/network → cross-region.

📌 **Cần THUỘC:**
- **86.400** giây/ngày.
- **peak ≈ 2–3× avg**.
- **RAM ~100ns** / **SSD ~µs** / **DC round-trip ~0.5ms** / **cross-region ~100ms**.

🛠️ **Cần LÀM ĐƯỢC:**
- Nhẩm **QPS / storage / bandwidth** từ **DAU**.
- Phát biểu **giả định** to, rõ.
- Rút ra kết luận **"cần cache / shard / CDN không"**.

---

## ⑦ Mental model

> 🎯 **"DAU → QPS → bytes → quyết định."**
> **Con số chỉ đáng tính nếu nó đổi thiết kế.** **RAM rẻ hơn disk ~1000× về latency.**

---

## ⑧ Câu hỏi phỏng vấn & thách đố

**(dễ)** 10M DAU, mỗi user 10 request/ngày — avg & peak QPS?
→ ~**1.150** avg, ~**2.300–3.500** peak.

**(TB)** **Read:write ratio** ảnh hưởng kiến trúc thế nào?
→ **read-heavy** ⇒ **cache + replica + CDN**; **write-heavy** ⇒ **shard + queue + LSM**.

**(TB)** Vì sao **RAM cache** đáng giá so với đọc DB?
→ **RAM ~100ns** vs **disk ~ms** ⇒ nhanh hơn **~1000×** bậc độ lớn.

**(khó)** **Storage** cho 500M ảnh/năm @1MB?
→ ×3 **replica** ≈ **1.5PB/năm**; nhớ dự trù nhiều năm.

**(rất khó)** Khi nào **estimation** là lãng phí?
→ khi con số **không đổi** một quyết định kiến trúc nào.

> 🧩 **Thách đố (trick):** Bạn tính ra **1.150 QPS** và kết luận "cần 50 server". Có gì sai?
>
> ```
> ❌ BẪY:    QPS  ───nhảy thẳng──►  số server
>            (chưa biết 1 server gánh được bao nhiêu QPS!)
>
> ✅ ĐÚNG:   QPS  ──►  per-server capacity  ──►  số server
>            + phải dùng PEAK QPS, không phải avg
> ```
> 🔎 **Vì sao là bẫy?** Nhảy từ QPS sang số server mà chưa biết **per-server capacity** (1 server gánh bao nhiêu QPS — phụ thuộc **payload**, **CPU/IO**). Phải **ước lượng per-server capacity trước**, và phải tính trên **peak**, không phải avg.

---

## ⑨ Bài tập + tiêu chí tự chấm

**BT1.** Ước lượng **QPS**, **storage/năm**, **bandwidth peak** cho **"Instagram-lite"**: 50M DAU, mỗi user xem 50 ảnh + đăng 0.2 ảnh/ngày, ảnh ~1.5MB.

> ✅ **Đạt khi:** **tách read (xem) và write (đăng)**, nêu **giả định**, ra **bậc độ lớn** hợp lý, kết luận **"cần CDN + object storage"**.
>
> 🔎 *Gợi ý:* read và write có QPS rất khác nhau (50 vs 0.2 / user/ngày) — đây chính là ví dụ **read-heavy** điển hình.

**BT2.** Cho một con số bạn vừa tính, chỉ ra **quyết định kiến trúc** nó dẫn tới.

> ✅ **Đạt khi:** mỗi con số gắn với **đúng một quyết định** (**cache** / **shard** / **CDN** / **replica**). Con số "mồ côi" không gắn quyết định nào = chưa đạt.

---

## ⑩ Đọc thêm

- **Jeff Dean** — *"Latency Numbers Every Programmer Should Know"* (gist `jboner/2841832`; bản tương tác `colin-scott.github.io/.../interactive_latency.html`).
- **System Design Interview** (Alex Xu) — chương *"Back-of-the-envelope estimation"*.

---

> 🎯 **Khẩu quyết khép bài:** *Nhẩm thô, không đếm chính xác. Nói giả định to. Mỗi con số phải đẻ ra một quyết định. RAM ≪ SSD ≪ disk/network ≪ cross-region.*
