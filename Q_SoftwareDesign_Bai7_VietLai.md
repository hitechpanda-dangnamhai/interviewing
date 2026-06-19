# 🎯 BÀI 7 — **GoF Creational**: **Factory** · **Abstract Factory** · **Builder** · **Singleton** · **Prototype**

**Phủ:** Q-GOFC-001 → Q-GOFC-002 → Q-GOFC-003 → Q-GOFC-004 → Q-GOFC-005 → Q-GOFC-006 → Q-GOFC-007 → Q-GOFC-008

> 💡 **Mental model mở đầu:**
> **Creational pattern** = nhóm pattern lo **chuyện "đẻ" object** — tách *"quyết định tạo cái gì"* khỏi *"nơi dùng object đó"*. Vì sao cần tách? Vì nếu nơi dùng tự `new` ra class cụ thể, nó **dính chặt** vào class đó (coupling — Bài 1).
> Điểm thú vị: trong **TS/Node**, nhiều pattern Java **co lại** nhờ **first-class function / module** (nối Bài 3). Đừng bê nguyên xi cây class kiểu Java vào TS.

---

## ① Mục tiêu & vị trí trong mạch

**GoF** (**Gang of Four**, 1994) chia **23 pattern** làm **3 nhóm**. Bài này lo **Creational** — cách **tạo object** để tách *"quyết định tạo cái gì"* khỏi *"nơi dùng"*.

```
   Bài 3 (FP thay GoF) ──┐
                         ▼
   Bài 7: CREATIONAL (tạo object)   ← bài này
                         │
              ┌──────────┴──────────┐
   Bài 8: STRUCTURAL          Bài 9: BEHAVIORAL
   (ghép object)             (giao tiếp giữa object)
```

> 🎯 **Chốt vị trí:** hiểu xong Creational, bạn sẽ thấy **nhiều pattern Java co lại trong TS** nhờ **first-class function / module** (Bài 3). Đây là nền cho **Structural** (Bài 8) và **Behavioral** (Bài 9).

---

## ② Giảng cơ bản → nâng cao

### (a) Ba nhóm GoF (Q-GOFC-001)

| Nhóm | Lo việc gì | Ví dụ | Đại diện |
|---|---|---|---|
| **Creational** | **tạo object** | Factory Method, Abstract Factory, Builder, Singleton, Prototype | **Factory** |
| **Structural** | **ghép / cấu trúc** object | Adapter, Decorator, Facade… | **Adapter** |
| **Behavioral** | **giao tiếp & phân chia trách nhiệm** | Strategy, Observer… | **Strategy** |

---

### (b) Factory Method (Q-GOFC-002)

> **Ẩn dụ:** bạn gọi món "cho tôi một ly nước ép" (abstraction). **Quầy pha chế** (factory) quyết định dùng máy ép nào, cam loại gì — bạn **không cần biết**. Bạn chỉ cầm ly nước ép ra về.

**Factory Method** = tách **quyết định "tạo concrete class nào"** khỏi **nơi dùng**, ẩn logic khởi tạo/lựa chọn.

> 🚨 **CẢNH BÁO — `new X()` trực tiếp KHÔNG phải factory:**
> ```
> const e = new CsvExporter();   // ❌ LỘ concrete type ngay tại nơi dùng → KHÔNG decouple
> ```
> Khi bạn gõ thẳng `new CsvExporter()`, nơi dùng **biết chính xác** class nào → dính chặt vào nó. **Factory phải trả về abstraction**, người gọi **không biết class cụ thể**:
> ```
> const e: Exporter = createExporter('csv');   // ✅ chỉ biết Exporter, không biết CsvExporter
> ```

---

### (c) Factory kiểu TS (Q-GOFC-003)

> **Ví dụ trực giác:** thay vì xây cả một cây class Factory kiểu Java, trong TS bạn thường chỉ cần một **cuốn danh bạ**: `{ pdf: makePdf, csv: makeCsv }` — tra key, gọi hàm.

Thực tế trong **Node/TS**, "factory" thường chỉ là **object map** `{ pdf: makePdf, csv: makeCsv }` thay vì **cây class + switch**.

| | **object map factory** (TS idiom) | **class hierarchy + switch** |
|---|---|---|
| ✅ Được | **gọn**; thêm loại ≈ **thêm 1 entry** (gần **OCP**); tận dụng **first-class function** | khung chặt cho biến thể phức tạp |
| ❌ Mất | **ràng buộc kiểu lỏng hơn**; ít "khung" cho **đa sản phẩm** phức tạp | dài dòng, nhiều boilerplate |

> 🎯 **Chốt:** **chọn theo độ phức tạp** — **map** cho đơn giản, **hierarchy** khi cần cấu trúc cho biến thể phức tạp.

---

### (d) Abstract Factory (Q-GOFC-004)

> **Ví dụ trực giác:** mua **bộ nội thất cùng tông**. Bạn không chọn lẻ ghế chỗ này, bàn chỗ kia — bạn chọn "bộ Scandinavian" và *cả bộ* (ghế + bàn + kệ) đều **đồng bộ phong cách**.

| | **Factory Method** | **Abstract Factory** |
|---|---|---|
| Tạo gì | **một** sản phẩm | **cả họ** sản phẩm liên quan, **nhất quán cùng "gia đình"** |
| Ví dụ | tạo một `Button` | bộ UI theme: **Button + Checkbox** cùng **Dark/Light**; hoặc bộ **driver** cùng **vendor** |

> 💡 **Khi nào cần Abstract Factory?** Khi phải **đảm bảo các object đi cùng nhau đúng bộ** — không lẫn Button Dark với Checkbox Light. Nó *gom việc tạo cả gia đình* vào một chỗ để giữ tính nhất quán.

---

### (e) Builder (Q-GOFC-005)

> **Ẩn dụ:** gọi pizza — bạn nói từng bước *"size L, thêm phô mai, thêm nấm, không hành"* (Builder), thay vì đọc một tràng 6 tham số mà người nghe dễ nhầm thứ tự.

**Builder** = tạo object **phức tạp từng bước**, tránh **telescoping constructor** (`new X(a,b,c,d,e,f)` — chuỗi tham số dài dễ nhầm), đặt **tên rõ** từng phần, **bất biến sau `build()`**.

> 🚨 **CẢNH BÁO — đừng lạm dụng Builder trong TS:**
> Trong TS, **options object** `new X({a, b, c})` thường **đã đủ** (đã có tên rõ, không sợ nhầm thứ tự). **Builder chỉ đáng** khi có **thứ tự / validation phức tạp giữa các bước** hoặc **nhiều biến thể build**. Dựng Builder cho thứ chỉ cần options object là **over-engineering** (Bài 3).

---

### (f) Singleton — vì sao bị coi anti-pattern (Q-GOFC-006 & 007)

> **Ví dụ trực giác:** **Singleton** như một **cái tủ chung cả công ty** mà ai cũng thò tay vào lấy/sửa đồ. Nghe tiện, nhưng: không ai biết *ai* đã đổi gì, lúc test thì đồ của test trước còn sót lại, và lúc đông người cùng mở tủ thì loạn.

**Singleton** = **global state ẩn** → kéo theo loạt vấn đề:

| Vấn đề | Vì sao |
|---|---|
| **khó test** | **state rò giữa test** — test A để lại state, test B nhận nhầm |
| **coupling ngầm** | ai cũng `Singleton.getInstance()` → phụ thuộc ẩn, không thấy trong constructor |
| **concurrency / lifecycle / thứ tự khởi tạo** | nhiều luồng cùng tạo, hoặc khởi tạo sai thứ tự |

> 🔎 **⚠️ Trong Node, "module là singleton SẴN":** module được **cache** sau lần `require`/`import` **đầu tiên**.
> ```
> // db.ts
> export const db = new Pool();   // ← chạy ĐÚNG MỘT LẦN, mọi import sau dùng lại instance này
> ```
> Cơ chế cache module:
> ```
> T1  file A:  import { db } from './db'   → Node CHẠY db.ts, tạo Pool, cache lại
> T2  file B:  import { db } from './db'   → Node KHÔNG chạy lại, trả db ĐÃ cache
> T3  file C:  import { db } from './db'   → cùng instance db
>                    │
>                    ▼
> ✅ Cả app dùng CHUNG một instance — KHÔNG cần tự cài Singleton thủ công
> ```

> 🎯 **Connection pool "chỉ một" (Q-GOFC-007):** ưu tiên **cung cấp qua DI** (một instance do **container** quản lý) thay vì **Singleton cổ điển** — vẫn **test / override / khởi tạo có thứ tự** được.
> **Nhớ khẩu quyết:** **"một instance" ≠ "phải dùng Singleton pattern".** (cross-ref **H-DI**.)

---

### (g) Prototype (Q-GOFC-008)

> **Ví dụ trực giác:** thay vì *xây* một ngôi nhà mới từ đầu (tốn kém), bạn **photocopy** một bản thiết kế mẫu rồi chỉnh sửa.

**Prototype** = tạo object mới bằng **clone một mẫu** thay vì khởi tạo tốn kém / không biết concrete type. JS vốn **prototype-based**; có **`Object.create`**, **`structuredClone`**.

> 🚨 **CẢNH BÁO — shallow vs deep clone:**
> ```
> shallow clone:  bản sao và bản gốc CHIA SẺ cùng nested object/refs
>                 → sửa nested ở bản sao → bản gốc CŨNG đổi  ❌ (rò rỉ)
> deep clone:     sao chép ĐỆ QUY mọi tầng → hoàn toàn độc lập  ✅
> ```
> Cảnh giác với **nested object / refs**: `{...obj}` chỉ shallow. Dùng **`structuredClone()`** để deep clone giữ được Date/Map/Set.

---

## ③ ⚠️ Kiến thức cũ → Mới

| Cũ | Nay (TS/Node) | Vì sao |
|---|---|---|
| **`getInstance()` Singleton thủ công** | **Module export** hoặc **DI single-scope** | Module đã **cache**; **DI** vẫn **test/override** được |
| **Telescoping constructor** `new X(a,b,c,d,e)` | **Options object** (hoặc **Builder** khi phức tạp) | Rõ tên, tránh **nhầm thứ tự tham số** |
| **`switch(type){ new A()… }`** rải rác | **Object map factory** `{a:makeA}` | Gần **OCP**, gọn, **first-class function** |
| **Deep copy bằng `JSON.parse(JSON.stringify())`** | **`structuredClone()`** (Node ≥17) | Giữ **Date/Map/Set**, không mất kiểu (**verify** Node version) |

---

## ④ Áp dụng thực tế + So sánh bigtech

### Use case Factory: `createExporter(format)`

```
   controller gọi: createExporter('pdf')
                    │  trả về ABSTRACTION Exporter (không biết class cụ thể)
                    ▼
   map: { pdf: makePdf, csv: makeCsv, xlsx: makeXlsx }
   → thêm 'xlsx' = thêm 1 entry, controller KHÔNG đổi
```

### Use case Singleton-via-DI

> **DB pool**, **Redis client**, **HTTP client** tái dùng — **Nest provider scope DEFAULT** (singleton) quản lý. Một instance, nhưng qua **DI** nên vẫn override/test được.

### Bigtech / ecosystem (pattern phổ biến — không tuyệt đối)

| Hệ sinh thái | Cách làm |
|---|---|
| **driver DB** (`pg`, `mongodb`) | export **factory/client** tái dùng |
| **SDK đám mây** (**AWS SDK v3**) | tạo **client một lần**, **inject khắp nơi** |

> 🔎 **Insight sâu:** AWS SDK v3 khuyến nghị *tạo client một lần rồi inject* — đúng tinh thần **"một instance qua DI"**, **không Singleton thủ công**. Cả ngành đã rời bỏ `getInstance()` cổ điển vì nó tạo **hidden global dependency**.

---

## ⑤ Code thực hành

```typescript
// creational.ts — TypeScript 5.x

// --- Factory bằng OBJECT MAP (idiom TS) ---
interface Exporter { export(rows: object[]): string; }
class CsvExporter  implements Exporter { export(r){ return '/* csv */'; } }
class JsonExporter implements Exporter { export(r){ return JSON.stringify(r); } }

const exporters: Record<string, () => Exporter> = {
  csv:  () => new CsvExporter(),
  json: () => new JsonExporter(),   // thêm 'xlsx' = thêm 1 DÒNG, KHÔNG sửa createExporter
};

function createExporter(format: string): Exporter {
  const make = exporters[format];
  if (!make) throw new Error(`Unsupported format: ${format}`);
  return make();                    // trả ABSTRACTION (Exporter), người gọi KHÔNG biết concrete
}

// --- "Singleton" ĐÚNG CÁCH trong Node: module export (đã được CACHE) ---
// db.ts
// import { Pool } from 'pg';                 // verify ở docs chính thức
// export const pool = new Pool({ /* config từ ENV, KHÔNG hardcode */ });
// các file khác: import { pool } from './db'  -> CÙNG MỘT instance toàn app (nhờ module cache)

// --- Builder CHỈ KHI thật phức tạp; đa số dùng options object ---
class Pizza {
  private constructor(readonly opts: { size: string; toppings: string[] }) {}
  static builder() {
    const t: string[] = []; let size = 'M';
    return {
      size: (s: string) => { size = s; return this as any; },       // từng bước, tên rõ
      add:  (x: string) => { t.push(x); return this as any; },
      build: () => new (Pizza as any)({ size, toppings: t }),       // bất biến SAU build()
    };
  }
}
```

> 💡 **Cấu hình:** config DB/SDK qua **`process.env`** + (requirements-equivalent là) **`package.json`** (pin version, vd `"pg": "^8.x"` — **verify** bản mới khi học). **Không commit credential.**

---

## ⑥ Keywords cần nhớ

> 🧠 **Cần HIỂU:**
> - **3 nhóm GoF** (Creational/Structural/Behavioral)
> - **Factory Method** (tách quyết định tạo)
> - **Abstract Factory** (cả họ sản phẩm)
> - **Builder vs options object**
> - vì sao **Singleton là anti-pattern**, **"module = singleton"**
> - **Prototype / clone**

> 📌 **Cần THUỘC:**
> - **`new X()` ≠ factory**.
> - **"một instance ≠ phải Singleton"**.
> - **shallow vs deep clone**.

> 🛠️ **Cần LÀM ĐƯỢC:**
> - Viết **object-map factory**.
> - Cung cấp **một-instance qua DI** thay **Singleton**.
> - **Clone an toàn**.

---

## ⑦ Mental model

> 🎯 **Khẩu quyết:**
> **Creational = giấu chuyện "đẻ" object.** Trong TS: **factory** thường là **một map hàm**; **Singleton** thường là **một module**; **Builder** chỉ khi **options object không đủ**.

```
   Cần tạo object?         → giấu chuyện đẻ sau một factory (trả abstraction)
   Cần "một instance"?     → module export / DI single-scope (ĐỪNG getInstance)
   Constructor 6 tham số?  → options object (Builder nếu phức tạp THẬT)
   Cần copy object?        → cẩn thận shallow vs deep (structuredClone)
```

---

## ⑧ Câu hỏi phỏng vấn & thách đố

**(TB)** *3 nhóm GoF giải quyết gì, ví dụ mỗi nhóm?*
→ **tạo / ghép / giao tiếp**; **Factory / Adapter / Strategy**.

**(TB)** *Vì sao `new X()` không phải factory?*
→ **lộ concrete**, **không decouple**.

**(khó)** *Abstract Factory khác Factory Method?*
→ **một sản phẩm** vs **cả họ nhất quán**.

**(khó)** *Vì sao Singleton bị coi anti-pattern? "module là singleton sẵn" nghĩa gì?*
→ **global state ẩn**; **module được cache** sau import đầu tiên.

> 🧩 **🔥 Thách đố:** *"Cần đúng một connection pool toàn app — Singleton chứ gì nữa?"*
>
> **Bẫy:** **"một instance" KHÔNG bắt buộc Singleton pattern.** Dùng **DI single-scope** để vẫn **test / override / khởi tạo có thứ tự**. **Singleton thủ công** tạo **hidden global dependency**. Trả lời "Singleton chứ gì" là rơi bẫy — đúng là *một instance, nhưng cung cấp qua module/DI*.

---

## ⑨ Bài tập + tiêu chí tự chấm

**BT1.** Đổi một `switch(type){...new...}` thành **object-map factory**.
→ ✅ **Đạt khi:** thêm loại mới **không sửa hàm tạo**.

**BT2.** Tìm một **`getInstance()` Singleton** trong code, chuyển sang **module export** hoặc **DI**.
→ ✅ **Đạt khi:** test có thể **inject fake** thay vì dùng **global thật**.

> 💡 *Gợi ý:* sau khi đổi, thử viết một unit test chèn fake — nếu chèn được mà không cần đụng global thì bạn đã thật sự gỡ được Singleton.

---

## ⑩ Đọc thêm

- **Design Patterns (GoF)** — phần **Creational**. **Refactoring Guru** — minh hoạ TS.
- **Node.js docs** — **module caching**; **MDN** — **`structuredClone`**.

---

## 🤖 Hiểu để chỉ huy (Kiểm tra AI)

> 🚨 **AI rất thích sinh:**
> 1. **Singleton `getInstance()`** — tạo **global state ẩn** "cho tiện".
> 2. **Builder cồng kềnh** cho thứ chỉ cần **options object**.
>
> 🎯 **Hai câu soát mỗi lần nhận code từ AI:**
> - *"Cái này có biến thành **global state ẩn** không?"* → có → ép đổi sang **module export / DI**.
> - *"**Builder** này có **thật sự cần** không, hay options object là đủ?"* → đủ → ép **đơn giản hoá**.
>
> AI hay bê nguyên pattern Java (Singleton, Builder đầy đủ) vào TS, trong khi TS có **module cache** và **options object** làm phần lớn việc đó gọn hơn. Vai trò chỉ huy của bạn: nhận ra *pattern nào đã co lại trong TS* và **không dựng lại khung Java thừa thãi**.
