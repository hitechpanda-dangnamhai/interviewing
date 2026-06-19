# 🎯 BÀI 8 — **GoF Structural**: **Adapter** · **Facade** · **Proxy** · **Decorator** · **Composite** · **Bridge**

**Phủ:** Q-GOFS-001 → Q-GOFS-002 → Q-GOFS-003 → Q-GOFS-004 → Q-GOFS-005 → Q-GOFS-006 → Q-GOFS-007 → Q-GOFS-008

> 💡 **Mental model mở đầu:**
> **Structural pattern** = cách **ghép object** thành cấu trúc lớn hơn **mà vẫn linh hoạt**. Bài này chứa **hai bẫy phỏng vấn kinh điển**:
> 1. **Adapter vs Facade vs Proxy** — cả ba đều "bọc" một object, nhưng **khác mục đích hoàn toàn**.
> 2. **GoF Decorator ≠ TS/Nest decorator** — trùng tên, khác bản chất.
> Nắm đúng *ý đồ* của mỗi pattern quan trọng hơn nhớ tên.

---

## ① Mục tiêu & vị trí trong mạch

**Structural** = **ghép object thành cấu trúc lớn hơn mà vẫn linh hoạt**.

```
   Bài 6 (DIP / port-adapter) ──┐
                                ▼
   Bài 8: STRUCTURAL  ← bài này (ghép object)
        bẫy 1: Adapter vs Facade vs Proxy (cùng "bọc", khác ý đồ)
        bẫy 2: GoF Decorator ≠ TS/Nest decorator
                                │
                                ▼
              chuẩn bị cho kiến trúc (Bài 11)
```

> 🎯 **Chốt vị trí:** bài này có **bẫy phỏng vấn kinh điển** (Adapter/Facade/Proxy & GoF Decorator vs TS decorator — **verify ở đầu tài liệu**). Nó nối **Bài 6** (**DIP / port-adapter**) và chuẩn bị cho **kiến trúc** (Bài 11).

---

## ② Giảng cơ bản → nâng cao

### (a) Adapter (Q-GOFS-001)

> **Ẩn dụ — phích cắm chuyển đổi.** Bạn mang sạc chân dẹt (Mỹ) sang ổ chân tròn (EU) — cần **cục adapter** chuyển *giao diện* cho khớp, chứ không thay cả cái sạc.

**Adapter** = chuyển **interface** của một class sang **interface client mong đợi**.

> 💡 **Use case backend:** bọc **SDK bên thứ ba / legacy API** về **interface domain** của bạn → nối ý với **Anti-Corruption Layer** (ACL — DDD, **Bài 13/14**): giữ cho "ngôn ngữ lạ" của bên ngoài không lọt vào làm bẩn domain của bạn.

---

### (b) ⚠️ Adapter vs Facade vs Proxy (Q-GOFS-002) — cùng "wrap", khác ý đồ

> 🚨 **Đây là bẫy phỏng vấn số 1 của Structural.** Cả ba đều "bọc" một object, nên người mới tưởng giống nhau. **Khác nhau ở MỤC ĐÍCH, không phải ở "có bọc hay không".**

| Pattern | Interface so với gốc | Mục đích | Ẩn dụ |
|---|---|---|---|
| **Adapter** | **ĐỔI** (A → B) | làm hai thứ **tương thích** | **"Phích cắm chuyển đổi"** |
| **Facade** | mặt **gọn mới** | **đơn giản hoá** subsystem phức tạp (nhiều thứ → một cửa) | **"Lễ tân khách sạn"** |
| **Proxy** | **GIỮ NGUYÊN** | **kiểm soát truy cập / thêm hành vi** (lazy/cache/quyền) | **"Người đại diện"** |

> 💡 **Mẹo phân biệt nhanh:** hỏi *"interface sau khi bọc có giống gốc không?"*
> - **đổi** → Adapter
> - **gọn lại / khác hẳn** → Facade
> - **y nguyên** → Proxy

---

### (c) Decorator pattern (Q-GOFS-003)

> **Ẩn dụ:** mặc thêm áo khoác. Bạn vẫn là bạn (cùng interface), nhưng **dán thêm một lớp** (ấm hơn). Mặc thêm áo mưa nữa → chồng lớp tiếp.

**Decorator** = bọc object **cùng interface**, **thêm trách nhiệm lúc runtime**, **có thể chồng nhiều lớp**. **Hợp OCP** (mở rộng không sửa class gốc).

```
   LoggingService( CachingService( RealService ) )
        │              │              │
        │              │              └─ object thật làm việc
        │              └─ thêm caching (cùng interface)
        └─ thêm logging (cùng interface)
   → mỗi lớp wrap quanh lớp trong, cùng interface, RealService KHÔNG đổi
```

---

### (d) ⚠️⚠️ GoF Decorator ≠ TypeScript/Nest decorator (Q-GOFS-004) — bẫy cực phổ biến

> 🚨 **CẢNH BÁO — trùng tên, KHÁC bản chất hoàn toàn:**

| | **GoF Decorator** | **TS/Nest decorator** (`@Injectable`, `@Get`) |
|---|---|---|
| Là gì | **pattern wrap object** thêm hành vi | **cú pháp annotation** gắn metadata / biến đổi declaration |
| Khi nào | **runtime**, **composition** | lúc **define class** |
| Ví dụ | `new CachingDecorator(real)` | `@Injectable() class Foo {}` |

> 🔎 **(Đã verify ở đầu tài liệu):** **Nest** dùng **legacy `experimentalDecorators` + `reflect-metadata`**; **TC39 Stage 3** là chuẩn mới mặc định từ **TS 5.0** nhưng **không có parameter decorator** nên **Nest chưa chuyển**. → khi học, **verify** phiên bản & cấu hình decorator bạn đang dùng.

---

### (e) Proxy — 3 biến thể (Q-GOFS-005)

> **Ẩn dụ:** Proxy như **người gác cổng / đại diện** — bạn nói chuyện với họ y như nói với người thật (cùng interface), nhưng họ *kiểm soát* việc gì được qua.

| Biến thể | Làm gì | Ví dụ |
|---|---|---|
| **Virtual** | **lazy init** — chỉ tạo object đắt khi **thật sự cần** | lazy-load entity nặng |
| **Protection** | kiểm tra **quyền** trước khi cho gọi | authz wrapper |
| **Remote** | đại diện object **ở xa** | RPC stub/client |

> 💡 JS có cả **`Proxy` built-in** để **intercept** (chặn/can thiệp) truy cập thuộc tính/method — một công cụ ngôn ngữ riêng, đừng nhầm với *Proxy pattern* (dù cùng tinh thần).

---

### (f) Facade vs service layer / God object (Q-GOFS-006)

> **Ẩn dụ:** **lễ tân khách sạn** đặt phòng, gọi taxi, báo dọn phòng giúp bạn — nhưng lễ tân **không tự nấu ăn, không tự lái taxi**. Họ **điều phối**, không **làm thay**.

**Facade** = mặt gọn cho subsystem. **Rủi ro:** phình thành **God object** (Bài 10) khi nó **ôm cả business logic**.

> 🎯 **Ranh giới vàng:** **Facade ĐIỀU PHỐI** (gọi các thành phần), **KHÔNG chứa rule nghiệp vụ lõi**. Vượt ranh giới này → **cohesion sụp**, biến thành God object.

---

### (g) Composite (Q-GOFS-007)

> **Ví dụ trực giác:** thư mục máy tính. Một **folder** có thể chứa **file** *và* chứa **folder con**. Khi tính "tổng dung lượng", bạn xử lý folder và file **như nhau** (đệ quy xuống).

**Composite** = xử lý **cấu trúc cây** (object chứa con **cùng kiểu**) bằng cách cho **leaf** (lá) và **composite** (nút chứa con) **chung một interface** → **client xử lý đồng nhất** (đệ quy).

> 💡 **Ví dụ ngoài UI:** **filesystem** (folder/file), **org chart** (sơ đồ tổ chức), **BOM** (bill of materials — danh mục vật tư), **cây quyền**.

---

### (h) Bridge (Q-GOFS-008)

> **Ví dụ trực giác:** bạn có **2 trục** chọn độc lập: *loại báo cáo* (PDF/HTML) **×** *nơi xuất* (Màn hình/Máy in). Nếu làm mỗi tổ hợp một class → bùng nổ số class.

**Bridge** = **tách abstraction khỏi implementation** để **cả hai biến thiên độc lập**.

> 🚨 **Nhận diện:** có **2 trục biến thiên độc lập** (vd `Report {PDF, HTML}` **×** `Renderer {Screen, Printer}`).

```
❌ KẾ THỪA THẲNG → m × n class (combinatorial explosion)
   PdfScreen, PdfPrinter, HtmlScreen, HtmlPrinter...  (2×2 = 4, rồi 3×4 = 12...)

✅ BRIDGE → m + n class (nối 2 phân cấp bằng composition)
   Report {PDF, HTML}  ──cầm──▶  Renderer {Screen, Printer}
   (2 + 2 = 4 nhưng thêm trục chỉ +1, không nhân lên)
```

| | **Bridge** | **Adapter** |
|---|---|---|
| Thời điểm | **thiết kế TRƯỚC** (chủ động tách 2 trục) | **chữa SAU** (làm 2 thứ có sẵn tương thích) |

---

## ③ ⚠️ Cũ → Mới / Phân biệt

| Hay nhầm | Phân biệt đúng | Vì sao |
|---|---|---|
| **Adapter = Facade = Proxy** ("đều bọc") | **Adapter đổi interface · Facade đơn giản hoá · Proxy kiểm soát truy cập** | Khác **mục đích**, không khác "có bọc hay không" |
| **GoF Decorator = `@Decorator`** của TS/Nest | **Pattern wrap object ≠ annotation gắn metadata** | Trùng tên, khác bản chất |
| **Kế thừa m×n** cho 2 trục biến thiên | **Bridge** (composition, **m+n**) | Tránh **class explosion** |

---

## ④ Áp dụng thực tế + So sánh bigtech

### Adapter
Bọc **Stripe/Twilio SDK** về **`PaymentGateway`/`SmsSender`** domain → **đổi vendor không lan** (**ACL**).

### Decorator (pattern)
**middleware / interceptor** wrap handler thêm **logging/auth/caching** — **Nest Interceptor** về ý tưởng **là decorator-pattern** (dù khai báo bằng `@`).

### Bigtech / ecosystem (pattern phổ biến — không tuyệt đối)

| Trường hợp | Pattern |
|---|---|
| **ORM repository** wrap driver | **Adapter** |
| **CDN / cache layer** trước origin | **Proxy** |
| **facade SDK** gom nhiều micro-API thành một client | **Facade** |

> 🔎 **Insight sâu:** CDN đứng trước origin server *cùng giao diện HTTP* (bạn gọi URL y như gọi origin) nhưng *kiểm soát*: cache, chặn, rate-limit. Đó chính là **Proxy pattern ở quy mô hạ tầng** — pattern GoF không chỉ sống trong code mà còn ở tầng kiến trúc.

---

## ⑤ Code thực hành

```typescript
// structural.ts — TypeScript 5.x

// --- Adapter: bọc SDK lạ về interface DOMAIN ---
interface SmsSender { send(to: string, text: string): Promise<void>; }     // interface DOMAIN (cái ta muốn)

// SDK bên thứ ba có signature KHÁC:
class TwilioSdk { async messages_create(p: { To: string; Body: string }) {/*...*/} }

class TwilioSmsAdapter implements SmsSender {
  constructor(private sdk: TwilioSdk) {}
  async send(to: string, text: string) {
    await this.sdk.messages_create({ To: to, Body: text });                // CHUYỂN interface domain → SDK
  }
}

// --- Decorator pattern: chồng hành vi RUNTIME, CÙNG interface ---
interface DataService { read(id: string): Promise<string>; }
class RealDataService implements DataService { async read(id){ return `data:${id}`; } }

class CachingDecorator implements DataService {                            // CÙNG interface DataService
  private cache = new Map<string, string>();
  constructor(private inner: DataService) {}
  async read(id: string) {
    if (this.cache.has(id)) return this.cache.get(id)!;                    // HIT cache → thêm caching, KHÔNG sửa Real
    const v = await this.inner.read(id);                                   // MISS → gọi lớp trong
    this.cache.set(id, v);
    return v;
  }
}
// const svc = new CachingDecorator(new LoggingDecorator(new RealDataService())); // CHỒNG lớp (OCP)

// --- Composite: cây file ---
interface Node { size(): number; }
class FileLeaf implements Node { constructor(private bytes: number){} size(){ return this.bytes; } }
class Folder implements Node {
  private children: Node[] = [];
  add(n: Node){ this.children.push(n); return this; }
  size(){ return this.children.reduce((s, c) => s + c.size(), 0); }        // ĐỆ QUY đồng nhất leaf + composite
}
```

> 💡 **Đọc kỹ `Folder.size()`:** nó gọi `c.size()` trên *mỗi con* mà **không cần biết** con là `FileLeaf` hay `Folder` khác — vì cả hai **cùng interface `Node`**. Đó là sức mạnh của **Composite**: client (và chính composite) xử lý cây **đồng nhất**.

---

## ⑥ Keywords cần nhớ

> 🧠 **Cần HIỂU:**
> - **Adapter / Facade / Proxy** (khác **ý đồ**)
> - **Decorator pattern** + **OCP**
> - **GoF Decorator ≠ TS decorator**
> - **Composite** (cây)
> - **Bridge** (2 trục, **m+n vs m×n**)

> 📌 **Cần THUỘC:**
> - **3 biến thể Proxy** (Virtual/Protection/Remote).
> - **Bridge-trước vs Adapter-sau**.

> 🛠️ **Cần LÀM ĐƯỢC:**
> - Viết **Adapter** bọc SDK.
> - Chồng **Decorator** caching/logging.
> - Dựng **Composite** cây.

---

## ⑦ Mental model

> 🎯 **Khẩu quyết (một câu mỗi pattern):**
> **Adapter** đổi phích cắm · **Facade** làm lễ tân · **Proxy** làm người gác cổng · **Decorator** dán thêm lớp · **Composite** gộp cây thành một · **Bridge** tách hai trục để khỏi nổ số class.

---

## ⑧ Câu hỏi phỏng vấn & thách đố

**(dễ)** *Adapter làm gì? Use case backend?*
→ **đổi interface**; **bọc SDK / legacy**.

**(TB)** *Adapter vs Facade vs Proxy khác mục đích sao?*
→ **tương thích / đơn giản hoá / kiểm soát truy cập**.

**(khó)** *Phân biệt GoF Decorator với TS/Nest decorator.*
→ **wrap object** vs **annotation metadata**.

**(rất khó)** *Bridge: cho tình huống không dùng Bridge thì class nổ tổ hợp.*
→ **Report × Renderer = m×n**; Bridge nối bằng composition → **m+n**.

> 🧩 **🔥 Thách đố:** *"Facade cho gọn — nhét luôn business logic vào cho tiện?"*
>
> **Bẫy:** **Facade chỉ ĐIỀU PHỐI.** Nhét logic → thành **God object** (Bài 10), **cohesion sụp**. Trả lời "nhét cho tiện" là rơi bẫy — Facade gọi các thành phần, *không chứa rule nghiệp vụ lõi*.

---

## ⑨ Bài tập + tiêu chí tự chấm

**BT1.** Bọc một **SDK bên thứ ba** (mail/sms/payment) bằng **Adapter** về **interface domain**.
→ ✅ **Đạt khi:** **đổi vendor chỉ cần adapter mới**, **business không đổi**.

> 💡 *Gợi ý:* định nghĩa interface theo *ngôn ngữ domain của bạn* (`send(to, text)`), rồi để adapter dịch sang signature lạ của SDK (`messages_create({To, Body})`).

**BT2.** Thêm **caching** cho một service bằng **Decorator** (**không sửa service gốc**).
→ ✅ **Đạt khi:** **bật/tắt caching = bọc/không bọc**, code Real **giữ nguyên**.

---

## ⑩ Đọc thêm

- **Design Patterns (GoF)** — **Structural**. **Refactoring Guru**.
- **TypeScript Handbook** — **Decorators**; **tc39/proposal-decorators** (để phân biệt với **GoF Decorator**).

---

## 🤖 Hiểu để chỉ huy (Kiểm tra AI)

> 🚨 **AI rất hay:**
> 1. **Lẫn "decorator"** — sinh `@SomeDecorator` khi bạn muốn **pattern wrap**, hoặc ngược lại.
> 2. **Biến Facade thành God object** — nhét business logic vào cái "mặt gọn".
>
> 🎯 **Hai câu soát mỗi lần nhận code từ AI:**
> - *"Cái 'decorator' này là **wrap object** (pattern) hay **annotation metadata** (TS)?"* → sai loại → ép sửa đúng ý đồ.
> - *"Facade/service này đang **điều phối** hay đang **chứa rule nghiệp vụ lõi**?"* → chứa logic → ép tách (giữ ranh giới điều-phối-vs-chứa-logic).
>
> AI dễ bị "trùng tên" đánh lừa (decorator), và dễ gom hết vào một Facade "cho gọn". Vai trò chỉ huy của bạn: luôn hỏi **đúng ý đồ pattern** và **giữ ranh giới điều-phối-vs-chứa-logic**.
