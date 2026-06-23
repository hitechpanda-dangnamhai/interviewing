# 🧪 QB_Q — Ngân hàng câu hỏi: Software Design & Patterns

> **Mục Q** trong roadmap Tech Lead Backend (🟡 ~40% · nền tảng xuyên suốt) · Sinh theo **WORKFLOW 2 — Bước A**.
> Đây là **mục LỚN** → bộ đề phủ HẾT 5 mục con của roadmap (SOLID · GoF patterns · Clean/layered architecture · Code smell & refactoring · DDD nhập môn) + mở rộng nền tảng OOP và nhóm **quyết định cấp Tech Lead** (khi nào KHÔNG dùng pattern, chống over-engineering, enforce design trong team).
> **Chỉ câu hỏi — KHÔNG kèm đáp án.** Mỗi câu có dòng *"dò cái gì"* = tiêu chí ĐẠT, dùng để chấm **live** ở Bước D.
> **Tổng: 90 câu.**

> **Chống trùng (đã đọc `QB_E_database.md`, `QB_F_apidesign.md`, `QB_G_nodejs.md`, `QB_H_nestjs.md`):**
> - **Dependency Inversion (DIP)** ở đây là *nguyên tắc thiết kế* (phụ thuộc vào abstraction, interface + đảo chiều phụ thuộc, vì sao giúp testability/swappability), **KHÔNG lặp** phần cơ chế **DI container của Nest** (provider/scope/`useFactory`...) đã nằm trọn ở **H-DI**. Khi câu Q chạm DI, hiểu ở mức *vì sao thiết kế thế*, để chiều *cách wiring* cho H.
> - **Layered/clean architecture** ở đây là *nguyên tắc kiến trúc code* (dependency rule, ports & adapters, tách domain khỏi infra). Cách **tổ chức module Nest cụ thể** (controller/service/repository, tránh 'God module') ở **H-MOD/H-107** → Q chỉ trỏ sang.
> - **Repository pattern** ở đây nhìn từ *DDD tactical* (làm việc với aggregate root, nói ngôn ngữ domain, ẩn ORM). Phần *persistence/transaction/N+1 phía DB* ở **mục E** → không lặp.
> - **API Gateway / BFF** là *kiến trúc hệ thống* → thuộc **mục F (F-TL-…)** và mục K, KHÔNG nằm ở Q (Q là *thiết kế code/object-level*). Q trỏ sang khi cần.
> - **Observer pattern** ở đây là *pattern (subject/observer, observer vs pub/sub về coupling)*. Cơ chế **EventEmitter/MessagePattern–EventPattern** ở **G/H** → Q chỉ liên hệ, không lặp wiring.

## Cách đọc
- **ID:** `Q-<tag>-<số>`. Tag mục con liệt kê ở mục lục dưới.
- **Độ khó:** ⭐ định nghĩa cơ bản → ⭐⭐⭐⭐⭐ phân biệt senior với TL thật (trade-off, judgment, "khi nào KHÔNG dùng", enforce trong team).
- **"Dò cái gì":** điều người trả lời PHẢI chạm tới mới tính ĐẠT (không phải đáp án — chỉ là mốc chấm live ở Bước D).

## Bối cảnh "độ ổn định" của chủ đề (tính tại 6/2026)
- Kiến thức Q phần lớn **ổn định** (GoF *Design Patterns* 1994; SOLID — Robert C. Martin; *Clean Architecture* 2017; *Domain-Driven Design* — Eric Evans 2003; *Refactoring* — Martin Fowler). → ít cần search hơn mục model/giá; **verify chủ yếu ở phần tooling/cú pháp đổi nhanh**.
- ⚠️ **TypeScript decorators**: cú pháp/ngữ nghĩa đã đổi giữa *legacy experimental decorators* (`experimentalDecorators`, dựa trên đề xuất cũ) và **TC39 decorators (Stage 3)** đã vào TS 5.0. Nest hiện vẫn dựa trên `experimentalDecorators` + `reflect-metadata`. → khi nhắc decorator để minh họa pattern, ghi *(verify trạng thái TS/TC39 khi học)*.
- ⚠️ "DDD 2026 standards" (tách Domain Service vs Application Service nghiêm ngặt theo Clean/Hexagonal) là *diễn giải cộng đồng*, không phải chuẩn cố định → trình bày như **pattern phổ biến**, không khẳng định tuyệt đối.
- Mọi câu chạm tên tooling/đời cú pháp đổi nhanh có dấu *(verify)*.

---

## Mục lục mục con (ID prefix → số câu)

| Tag | Mục con | Số câu |
|---|---|---|
| **Q-SOLID** | 5 nguyên tắc SOLID + vi phạm + trade-off | 14 |
| **Q-OOP** | Nền OOP đỡ pattern: coupling/cohesion, composition vs inheritance, abstraction, DRY/KISS/YAGNI, Law of Demeter | 10 |
| **Q-GOFC** | GoF Creational: Factory, Abstract Factory, Builder, Singleton, Prototype | 8 |
| **Q-GOFS** | GoF Structural: Adapter, Decorator, Facade, Proxy, Composite, Bridge | 8 |
| **Q-GOFB** | GoF Behavioral: Strategy, Observer, Command, Template Method, State, Chain of Responsibility | 10 |
| **Q-ARCH** | Layered / Clean / Hexagonal / Onion, dependency rule, ports & adapters | 12 |
| **Q-DDD** | DDD nhập môn: bounded context, aggregate, entity/value object, repository, domain vs application service | 10 |
| **Q-SMELL** | Code smell, anti-pattern, refactoring | 10 |
| **Q-TL** | Judgment cấp Tech Lead: khi nào KHÔNG dùng pattern, chống over-engineering, enforce design | 8 |

---

## Q-SOLID — 5 nguyên tắc SOLID

**Q-SOLID-001** ⭐
"SOLID là viết tắt của 5 nguyên tắc nào? Mỗi chữ một câu một dòng."
> *Dò cái gì:* SRP / OCP / LSP / ISP / DIP — gọi đúng tên và nêu được ý chính từng cái, không nhầm lẫn thứ tự/nghĩa.

**Q-SOLID-002** ⭐⭐
"Single Responsibility Principle (SRP) thật ra nói về *cái gì* — 'một class làm một việc' có phải cách hiểu đúng không?"
> *Dò cái gì:* SRP = 'một class chỉ có MỘT lý do để thay đổi' (một actor/stakeholder), KHÔNG phải 'một method/một việc'; phân biệt 'lý do thay đổi' với 'số lượng hành vi'.

**Q-SOLID-003** ⭐⭐
"Cho một class `UserService` vừa validate, vừa lưu DB, vừa gửi email xác nhận. Nó vi phạm nguyên tắc nào và sửa hướng nào?"
> *Dò cái gì:* SRP (ba lý do thay đổi: rule nghiệp vụ, persistence, hạ tầng mail); tách thành các collaborator tách biệt; hiểu lợi ích test/maintain.

**Q-SOLID-004** ⭐⭐⭐
"Open/Closed Principle: 'mở để mở rộng, đóng để sửa đổi' nghĩa là gì? Cơ chế nào của OOP giúp đạt OCP?"
> *Dò cái gì:* thêm hành vi mới mà không sửa code cũ đã chạy ổn; đạt qua abstraction/polymorphism (interface, strategy); nối được OCP ↔ giảm rủi ro regression.

**Q-SOLID-005** ⭐⭐⭐
"Một hàm `calculatePrice` có chuỗi `if (type === 'A') … else if (type === 'B') …` cứ mỗi loại mới lại phải sửa. Vi phạm gì? Pattern nào trị?"
> *Dò cái gì:* vi phạm OCP; thay switch-on-type bằng polymorphism/Strategy hoặc map handler; nhận ra mùi 'switch statement' báo thiếu abstraction.

**Q-SOLID-006** ⭐⭐⭐⭐
"Liskov Substitution Principle nói gì? Cho ví dụ kinh điển khi kế thừa 'đúng cú pháp' nhưng phá LSP."
> *Dò cái gì:* subtype phải thay thế được supertype mà không phá hành vi/contract; ví dụ Square-extends-Rectangle hoặc Ostrich-extends-Bird (`fly()` ném exception); hiểu hậu điều kiện không được mạnh hơn, tiền điều kiện không được yếu hơn.

**Q-SOLID-007** ⭐⭐⭐⭐
"Vì sao 'is-a' theo nghĩa đời thường (chim cánh cụt *là* chim) lại có thể vi phạm LSP trong code? Bài học rút ra cho việc dùng inheritance?"
> *Dò cái gì:* quan hệ phân loại đời thực ≠ quan hệ thay thế hành vi; ưu tiên composition khi 'is-a' không kéo theo 'behaves-like-a'; LSP là test cho việc inheritance có hợp lệ không.

**Q-SOLID-008** ⭐⭐⭐
"Interface Segregation Principle: một interface `Worker` có cả `work()` và `eat()`, robot phải implement `eat()` rỗng. Sai ở đâu, sửa sao?"
> *Dò cái gì:* client không nên bị buộc phụ thuộc method nó không dùng; tách thành interface nhỏ (`Workable`, `Eatable`); nhận ra 'fat interface' là mùi.

**Q-SOLID-009** ⭐⭐⭐⭐
"Dependency Inversion Principle phát biểu hai vế nào? 'High-level' và 'low-level module' là gì?"
> *Dò cái gì:* (1) module cấp cao không phụ thuộc module cấp thấp, cả hai phụ thuộc abstraction; (2) abstraction không phụ thuộc chi tiết, chi tiết phụ thuộc abstraction; cho ví dụ business logic vs DB/HTTP driver.

**Q-SOLID-010** ⭐⭐⭐⭐
"DIP và Dependency Injection (DI) khác nhau thế nào? Có DI là tự động có DIP không?"
> *Dò cái gì:* DIP là *nguyên tắc* (phụ thuộc abstraction); DI là *kỹ thuật* cung cấp dependency từ ngoài; inject một *concrete class* vẫn là DI nhưng KHÔNG đạt DIP (phải inject qua interface/abstraction). *(cross-ref H-DI cho cơ chế container Nest.)*

**Q-SOLID-011** ⭐⭐⭐
"Vì sao DIP/lập trình theo interface lại làm code dễ test hơn? Liên hệ tới mock/stub."
> *Dò cái gì:* phụ thuộc abstraction → trong test thay implementation thật bằng test double; tách business logic khỏi I/O thật (DB/network) để unit test nhanh, xác định.

**Q-SOLID-012** ⭐⭐⭐⭐⭐
"SOLID có bao giờ là *over-engineering* không? Khi nào bạn cố tình KHÔNG áp dụng triệt để?"
> *Dò cái gì:* nhận ra trade-off — dự án nhỏ/script/throwaway, abstraction sớm khi chưa rõ trục biến thiên → thêm class/indirection vô ích; biết 'áp dụng khi đau thật', không giáo điều.

**Q-SOLID-013** ⭐⭐⭐⭐
"SOLID liên hệ thế nào với hai khái niệm coupling và cohesion? Nguyên tắc nào kéo cohesion lên, nguyên tắc nào kéo coupling xuống?"
> *Dò cái gì:* SRP/ISP ↑ cohesion (gom đúng trách nhiệm, interface gọn); DIP/OCP ↓ coupling (phụ thuộc abstraction, không sửa lan); hiểu mục tiêu cuối là loose coupling + high cohesion.

**Q-SOLID-014** ⭐⭐⭐⭐⭐
"Trong code review, một junior 'áp SOLID' bằng cách tạo interface cho mọi class (mỗi class một interface 1-1). Bạn phản hồi sao?"
> *Dò cái gì:* nhận ra 'interface 1-1 vô nghĩa' là over-abstraction (YAGNI); interface chỉ có giá trị khi có ≥2 implementation hoặc cần đảo chiều phụ thuộc/test seam thật; cách góp ý mang tính dẫn dắt, không chỉ chê.

---

## Q-OOP — Nền OOP đỡ pattern

**Q-OOP-001** ⭐⭐
"Phân biệt coupling và cohesion. Vì sao mục tiêu là 'loose coupling, high cohesion'?"
> *Dò cái gì:* coupling = mức phụ thuộc giữa các module; cohesion = mức 'thuộc về nhau' của các phần trong một module; loose+high → đổi một chỗ ít lan, dễ hiểu/test/tái dùng.

**Q-OOP-002** ⭐⭐⭐
"'Favor composition over inheritance' nghĩa là gì và vì sao? Khi nào inheritance vẫn đúng?"
> *Dò cái gì:* inheritance tạo coupling chặt + fragile base class + chỉ 'is-a'; composition linh hoạt, ghép hành vi runtime; inheritance hợp khi quan hệ 'is-a' thật + thoả LSP + ổn định.

**Q-OOP-003** ⭐⭐⭐⭐
"'Fragile base class problem' là gì? Cho tình huống sửa class cha làm vỡ class con dù không đụng class con."
> *Dò cái gì:* class con phụ thuộc chi tiết implementation của cha; đổi method nội bộ ở cha → đổi hành vi con ngầm; lý do nên đóng gói/ưu tiên composition.

**Q-OOP-004** ⭐⭐
"Encapsulation là gì, khác gì với chỉ 'để biến private'? Vì sao getter/setter cho mọi field không hẳn là encapsulation?"
> *Dò cái gì:* encapsulation = ẩn trạng thái + bảo vệ invariant qua hành vi, không chỉ access modifier; 'anemic' getter/setter phơi state ra ngoài → mất bảo vệ bất biến.

**Q-OOP-005** ⭐⭐
"Abstraction và encapsulation khác nhau ở đâu? Nói một câu phân biệt rõ."
> *Dò cái gì:* abstraction = phơi 'cái gì' (interface/khái niệm), giấu 'như thế nào'; encapsulation = che giấu/bảo vệ chi tiết trạng thái+implementation; hai mặt nhưng không đồng nhất.

**Q-OOP-006** ⭐⭐⭐
"DRY, KISS, YAGNI mỗi cái cảnh báo điều gì? Cho một ví dụ áp DRY *quá đà* gây hại."
> *Dò cái gì:* DRY tránh lặp kiến thức; KISS chuộng đơn giản; YAGNI đừng làm trước cái chưa cần; ví dụ gom hai đoạn 'giống ngẫu nhiên' thành một abstraction sai → coupling giả, khó sửa (premature abstraction).

**Q-OOP-007** ⭐⭐⭐⭐
"DRY thường bị hiểu nhầm là 'không có dòng code trùng nhau'. Phát biểu đúng của DRY là gì?"
> *Dò cái gì:* DRY về *kiến thức/quy tắc* có một nguồn chân lý duy nhất, KHÔNG phải về ký tự code; hai đoạn giống nhau nhưng thay đổi vì lý do khác nhau thì KHÔNG nên gộp.

**Q-OOP-008** ⭐⭐⭐
"Law of Demeter ('principle of least knowledge') nói gì? `order.getCustomer().getAddress().getCity()` vi phạm thế nào?"
> *Dò cái gì:* chỉ 'nói chuyện với bạn bè trực tiếp', không reach sâu vào chuỗi object; train wreck `a.b().c().d()` lộ cấu trúc nội bộ → coupling; hướng sửa: 'Tell, Don't Ask' / thêm method ở đúng tầng.

**Q-OOP-009** ⭐⭐⭐⭐
"Phân biệt 'programming to an interface, not an implementation' với việc dùng từ khoá `interface`. Hai cái có giống nhau không?"
> *Dò cái gì:* nguyên tắc nói về *phụ thuộc kiểu trừu tượng/contract*, không nhất thiết là keyword `interface`; có thể là abstract class/duck typing; mục tiêu là tách client khỏi concrete type.

**Q-OOP-010** ⭐⭐⭐⭐⭐
"Trong TypeScript/Node, OOP-thuần có phải lựa chọn duy nhất để 'thiết kế tốt'? Khi nào functional/closure thay được pattern OOP?"
> *Dò cái gì:* nhiều pattern GoF trong JS/TS thu gọn nhờ first-class function/closure (Strategy = truyền hàm; Command = closure; Singleton = module); judgment chọn paradigm theo bài toán, không ép OOP. *(verify minh hoạ cú pháp.)*

---

## Q-GOFC — GoF Creational Patterns

**Q-GOFC-001** ⭐⭐
"Ba nhóm GoF (Creational / Structural / Behavioral) mỗi nhóm giải quyết loại vấn đề gì? Cho một pattern đại diện mỗi nhóm."
> *Dò cái gì:* creational = tạo object; structural = ghép/cấu trúc object; behavioral = giao tiếp/phân chia trách nhiệm; ví dụ đúng nhóm (Factory / Adapter / Strategy).

**Q-GOFC-002** ⭐⭐⭐
"Factory Method giải quyết vấn đề gì? Vì sao 'gọi constructor trực tiếp' KHÔNG phải là factory?"
> *Dò cái gì:* tách quyết định 'tạo class nào' khỏi nơi dùng, ẩn logic khởi tạo/lựa chọn concrete type; `new X()` lộ concrete → không decouple → không phải factory.

**Q-GOFC-003** ⭐⭐⭐
"Trong Node/TS, một 'factory' thường chỉ là object map `{ pdf: makePdf, csv: makeCsv }` thay vì switch. Cách này được/mất gì so với hierarchy Factory Method kiểu Java?"
> *Dò cái gì:* map/registry gọn, dễ thêm loại (gần OCP), tận dụng first-class function; mất: ràng buộc kiểu lỏng hơn, ít 'khung' cho biến thể phức tạp; biết chọn theo độ phức tạp.

**Q-GOFC-004** ⭐⭐⭐⭐
"Abstract Factory khác Factory Method ở đâu? Khi nào cần Abstract Factory?"
> *Dò cái gì:* Factory Method tạo *một* sản phẩm; Abstract Factory tạo *họ sản phẩm liên quan* nhất quán (vd theme/driver-set); cần khi phải đảm bảo các object đi cùng nhau cùng 'gia đình'.

**Q-GOFC-005** ⭐⭐⭐
"Builder pattern dùng khi nào? So với constructor nhiều tham số / object literal."
> *Dò cái gì:* tạo object phức tạp từng bước, tránh 'telescoping constructor', đặt tên rõ từng phần, bất biến sau build; trong TS đôi khi options object đã đủ → biết khi nào Builder mới đáng.

**Q-GOFC-006** ⭐⭐⭐⭐
"Singleton bị coi là anti-pattern trong nhiều trường hợp. Vì sao? Trong Node, 'module là singleton sẵn' nghĩa là gì?"
> *Dò cái gì:* Singleton = global state ẩn → khó test, coupling ngầm, vấn đề concurrency/lifecycle; trong Node, `module` được cache nên export instance đã là singleton; thường nên dùng DI thay vì Singleton thủ công.

**Q-GOFC-007** ⭐⭐⭐⭐⭐
"Bạn cần một connection pool 'chỉ một' toàn app. Singleton cổ điển vs cung cấp qua DI container — chọn cách nào và vì sao?"
> *Dò cái gì:* ưu tiên DI (single instance được container quản lý) để vẫn test/override/khởi tạo có thứ tự được; Singleton tự cài tạo global+hidden dependency; hiểu 'một instance' ≠ 'phải dùng Singleton pattern'. *(cross-ref H-DI.)*

**Q-GOFC-008** ⭐⭐⭐
"Prototype pattern (clone) giải quyết gì? Trong JS, `Object.create`/structuredClone liên hệ thế nào?"
> *Dò cái gì:* tạo object mới bằng cách clone mẫu thay vì khởi tạo tốn kém/không biết concrete type; JS có prototype-based + clone API; cảnh giác shallow vs deep clone.

---

## Q-GOFS — GoF Structural Patterns

**Q-GOFS-001** ⭐⭐
"Adapter pattern làm gì? Cho một use case backend thật."
> *Dò cái gì:* chuyển interface của một class sang interface client mong đợi; ví dụ bọc SDK bên thứ ba/legacy API về interface domain; nối ý với 'anti-corruption layer'.

**Q-GOFS-002** ⭐⭐⭐
"Adapter vs Facade vs Proxy: cùng 'bọc' một thứ, khác nhau ở *mục đích* thế nào?"
> *Dò cái gì:* Adapter = đổi interface cho tương thích; Facade = đơn giản hoá/hợp nhất nhiều thứ phức tạp thành một mặt gọn; Proxy = cùng interface nhưng kiểm soát truy cập/thêm hành vi (lazy/cache/quyền).

**Q-GOFS-003** ⭐⭐⭐⭐
"Decorator pattern bổ sung hành vi *không sửa class gốc* thế nào? Liên hệ OCP."
> *Dò cái gì:* bọc object cùng interface, thêm trách nhiệm ở runtime, có thể chồng nhiều lớp; hợp OCP (mở rộng không sửa); ví dụ logging/caching/auth wrap quanh một service.

**Q-GOFS-004** ⭐⭐⭐⭐
"Phân biệt **Decorator pattern (GoF)** với **TypeScript/Nest decorator** (`@Injectable`, `@Get`). Chúng có cùng một thứ không?"
> *Dò cái gì:* GoF Decorator = wrap object thêm hành vi; TS/Nest decorator = cú pháp annotation gắn metadata/biến đổi declaration lúc define; KHÁC nhau dù trùng tên — bẫy phổ biến. *(verify trạng thái TC39/experimentalDecorators.)*

**Q-GOFS-005** ⭐⭐⭐
"Proxy pattern: kể 3 biến thể (virtual/protection/remote) và một ví dụ mỗi loại."
> *Dò cái gì:* virtual = lazy init/đắt mới tạo; protection = kiểm tra quyền trước khi cho gọi; remote = đại diện object ở xa; ví dụ hợp lý từng loại (lazy-load entity / authz wrapper / RPC stub).

**Q-GOFS-006** ⭐⭐⭐
"Facade khác gì với một 'service layer' gom nhiều call? Khi nào Facade là chính đáng, khi nào thành 'God object'?"
> *Dò cái gì:* Facade = mặt gọn cho subsystem phức tạp; rủi ro phình thành God object khi facade ôm cả business logic; ranh giới: facade *điều phối*, không *chứa* logic nghiệp vụ.

**Q-GOFS-007** ⭐⭐⭐⭐
"Composite pattern xử lý cấu trúc cây (object có con cùng kiểu) thế nào? Cho ví dụ ngoài UI."
> *Dò cái gì:* coi 'leaf' và 'composite' qua cùng một interface để client xử lý đồng nhất; ví dụ filesystem (folder/file), org chart, bom/cây quyền; lợi ích đệ quy đồng nhất.

**Q-GOFS-008** ⭐⭐⭐⭐⭐
"Bridge pattern tách 'abstraction' khỏi 'implementation' để cả hai biến thiên độc lập. Cho tình huống nếu *không* dùng Bridge thì số class nổ tổ hợp (combinatorial explosion)."
> *Dò cái gì:* nhận ra 2 trục biến thiên độc lập (vd loại Report × loại Renderer) → kế thừa thẳng cho m×n class; Bridge tách 2 phân cấp nối bằng composition → m+n; phân biệt với Adapter (Bridge thiết kế trước, Adapter chữa sau).

---

## Q-GOFB — GoF Behavioral Patterns

**Q-GOFB-001** ⭐⭐⭐
"Strategy pattern giải quyết gì? Trong TS, 'truyền một hàm vào' đã là Strategy chưa?"
> *Dò cái gì:* đóng gói các thuật toán hoán đổi được sau cùng một interface, chọn runtime; trong JS/TS truyền callback/function chính là Strategy bản gọn — nối với việc thay switch-on-type (OCP).

**Q-GOFB-002** ⭐⭐⭐
"Cho hệ thống tính phí ship theo nhiều hãng. So sánh giải bằng `if/else` vs Strategy. Strategy được/mất gì?"
> *Dò cái gì:* Strategy: thêm hãng = thêm class, không sửa code cũ (OCP), dễ test từng strategy; mất: nhiều class/indirection nếu chỉ 2-3 case đơn giản; biết cân theo số biến thể.

**Q-GOFB-003** ⭐⭐⭐
"Observer pattern: subject/observer hoạt động thế nào? Cho ví dụ."
> *Dò cái gì:* quan hệ một-nhiều, subject đổi state → notify các observer đã subscribe; observer thêm/bớt runtime; ví dụ UI update, cache invalidation, domain event.

**Q-GOFB-004** ⭐⭐⭐⭐
"Phân biệt **Observer** với **Pub/Sub**. Khác nhau cốt lõi về *coupling* nằm ở đâu?"
> *Dò cái gì:* Observer: subject biết & gọi trực tiếp observer (cùng tiến trình, coupling vừa); Pub/Sub: qua broker/event channel trung gian, publisher không biết subscriber (decouple mạnh, thường async/cross-process); nối với EventEmitter (in-proc) vs message broker. *(cross-ref G/H cho cơ chế.)*

**Q-GOFB-005** ⭐⭐⭐⭐
"Trong Node, `EventEmitter` là hiện thân của Observer. Rủi ro thường gặp khi lạm dụng emitter cho luồng nghiệp vụ là gì?"
> *Dò cái gì:* khó lần vết luồng (control flow ẩn), lỗi nuốt mất (`error` event chưa handle → crash), memory leak do quên `removeListener`, thứ tự/đồng bộ khó suy luận; biết khi nào nên dùng call trực tiếp/queue thay vì event.

**Q-GOFB-006** ⭐⭐⭐
"Command pattern đóng gói 'một yêu cầu' thành object để làm gì? Kể 2 năng lực nó mở ra."
> *Dò cái gì:* biến hành động thành object có thể truyền/lưu/hoãn; mở ra undo/redo, queue/lập lịch, logging/replay, audit; ví dụ job queue.

**Q-GOFB-007** ⭐⭐⭐
"Template Method pattern là gì? Nó dựa trên cơ chế OOP nào và rủi ro coupling ra sao?"
> *Dò cái gì:* lớp cha định 'khung' thuật toán, để lớp con override vài bước (hook); dựa trên inheritance → coupling base-class chặt; cân nhắc Strategy (composition) khi cần linh hoạt hơn.

**Q-GOFB-008** ⭐⭐⭐⭐
"State pattern khác Strategy thế nào, dù cấu trúc class trông giống nhau?"
> *Dò cái gì:* State: object tự *chuyển trạng thái* và hành vi đổi theo state, các state biết transition; Strategy: client/bên ngoài chọn thuật toán, các strategy độc lập không tự đổi nhau; khác ở *ai điều khiển chuyển* và *ý đồ*.

**Q-GOFB-009** ⭐⭐⭐⭐
"Chain of Responsibility dùng khi nào? Liên hệ với middleware của Express/Nest."
> *Dò cái gì:* chuỗi handler, mỗi handler xử lý hoặc chuyển tiếp; middleware pipeline (`req → mw1 → mw2 → handler`) chính là CoR; lợi ích: thêm/bớt/đổi thứ tự khâu xử lý linh hoạt.

**Q-GOFB-010** ⭐⭐⭐⭐⭐
"Cho một workflow duyệt đơn nhiều bước (tạo → chờ duyệt → duyệt → từ chối → đóng) với hành vi khác nhau mỗi bước. Bạn mô hình bằng pattern nào và vì sao không chỉ dùng enum + if?"
> *Dò cái gì:* State machine/State pattern để gom hành vi + transition hợp lệ theo state, tránh `if(status===…)` rải khắp; nêu lợi ích: chặn transition sai, dễ thêm state; biết khi đơn giản thì enum+guard cũng đủ (không over-engineer).

---

## Q-ARCH — Layered / Clean / Hexagonal / Onion

**Q-ARCH-001** ⭐⭐
"Layered architecture (presentation / business / data) cơ bản là gì? Quy tắc phụ thuộc giữa các tầng?"
> *Dò cái gì:* tách trách nhiệm theo tầng; phụ thuộc đi một chiều xuống dưới; tầng trên không bị tầng dưới gọi ngược; lợi ích thay tầng (vd đổi DB) ít ảnh hưởng.

**Q-ARCH-002** ⭐⭐⭐
"'The Dependency Rule' (Clean Architecture) phát biểu thế nào? Vì sao domain/business KHÔNG được phụ thuộc framework/DB?"
> *Dò cái gì:* phụ thuộc chỉ trỏ vào trong (về phía policy/domain); domain độc lập với chi tiết (DB/web/framework) để ổn định, dễ test, thay hạ tầng không đụng nghiệp vụ.

**Q-ARCH-003** ⭐⭐⭐⭐
"Hexagonal Architecture (ports & adapters): 'port' và 'adapter' là gì? Phân biệt driving (primary) vs driven (secondary)."
> *Dò cái gì:* port = interface domain định nghĩa; adapter = implementation cụ thể nối ra ngoài; driving = thứ gọi vào app (HTTP/CLI/test); driven = thứ app gọi ra (DB/mail/queue); app core không biết adapter nào.

**Q-ARCH-004** ⭐⭐⭐⭐
"Clean, Onion, Hexagonal — ba cái này khác nhau bản chất hay chỉ là cách vẽ khác của *một* ý tưởng?"
> *Dò cái gì:* cùng tư tưởng lõi: tách domain khỏi infra + đảo chiều phụ thuộc vào trong; khác ở thuật ngữ/cách phân lớp/nhấn mạnh; không nên tranh cãi tên mà hiểu nguyên tắc chung.

**Q-ARCH-005** ⭐⭐⭐⭐
"Trong hexagonal, vì sao `interface` của repository lại được *định nghĩa ở tầng domain* mà implementation lại ở tầng infra? Cơ chế gì đảo được chiều phụ thuộc?"
> *Dò cái gì:* domain sở hữu port (abstraction); infra implement port → phụ thuộc trỏ từ ngoài vào domain (DIP ở cấp kiến trúc); domain không import infra, chỉ import abstraction của chính nó.

**Q-ARCH-006** ⭐⭐⭐
"Cho code: trong một entity domain bạn thấy `import { PrismaClient }`. Vì sao đây là dấu hiệu kiến trúc sai (theo dependency rule)?"
> *Dò cái gì:* domain rò rỉ phụ thuộc ORM/infra → đảo ngược dependency rule; khó test/thay DB, business trộn persistence; sửa: repository port + adapter ở ngoài.

**Q-ARCH-007** ⭐⭐⭐⭐
"'Anemic Domain Model' là gì và vì sao Martin Fowler coi là anti-pattern trong DDD? Nó mâu thuẫn ra sao với layered service-heavy?"
> *Dò cái gì:* entity chỉ là túi getter/setter, mọi logic dồn vào service → mất encapsulation/OOP; DDD muốn hành vi+bất biến nằm trong domain object; biết đây là *trade-off có chủ đích* tuỳ team/độ phức tạp, không phải luôn sai.

**Q-ARCH-008** ⭐⭐⭐⭐⭐
"Hexagonal/clean thêm nhiều layer/mapping/interface. Với một microservice CRUD mỏng, áp full clean có đáng không? Bạn quyết định dựa trên gì?"
> *Dò cái gì:* judgment — chi phí ports/adapters/mapping chỉ đáng khi domain đủ phức tạp/biến động; CRUD mỏng → over-engineering; cân theo độ phức tạp nghiệp vụ, vòng đời, đội ngũ; không giáo điều.

**Q-ARCH-009** ⭐⭐⭐⭐
"'Screaming Architecture' (Uncle Bob) nói cấu trúc thư mục nên 'hét lên' điều gì? So với tổ chức theo technical layer (controllers/, services/, models/)."
> *Dò cái gì:* cấu trúc nên phản ánh *domain/use case* (orders/, billing/) chứ không phải framework/layer; lợi ích: thấy hệ thống *làm gì* ngay; nối với feature-module (cross-ref H-MOD).

**Q-ARCH-010** ⭐⭐⭐⭐
"Trong Clean/Onion thường có 'use case'/'application service'. Tầng này khác 'domain service' và khác 'controller' thế nào?"
> *Dò cái gì:* use case/application service = điều phối luồng (orchestrate), quản transaction/unit-of-work, không chứa rule lõi; domain service = rule nghiệp vụ liên nhiều aggregate; controller = adapter I/O. *(verify: phân định cộng đồng, không tuyệt đối.)*

**Q-ARCH-011** ⭐⭐⭐⭐⭐
"CQRS (Command Query Responsibility Segregation) ở mức code-design giải quyết gì? Khi nào tách read/write model là hợp lý, khi nào là phức tạp thừa?"
> *Dò cái gì:* tách model ghi (giữ invariant) khỏi model đọc (tối ưu truy vấn/projection); đáng khi read/write rất khác nhau/khác tải; thừa khi CRUD đơn giản; phân biệt CQRS với Event Sourcing (không bắt buộc đi kèm).

**Q-ARCH-012** ⭐⭐⭐⭐⭐
"Bạn được giao một monolith 'big ball of mud' tầng trộn lẫn. Lộ trình refactor về clean/hexagonal *không* big-bang rewrite là gì?"
> *Dò cái gì:* nêu cách incremental — bọc seam bằng interface, tách domain dần, đưa I/O ra adapter, dùng test characterization làm lưới an toàn, strangler-fig theo module; tránh rewrite toàn bộ rủi ro.

---

## Q-DDD — Domain-Driven Design nhập môn

**Q-DDD-001** ⭐⭐⭐
"Bounded Context là gì? Vì sao 'một model duy nhất cho toàn hệ thống' lại là kỳ vọng sai?"
> *Dò cái gì:* ranh giới trong đó một model/ngôn ngữ nhất quán; cùng từ ('Customer') có nghĩa khác giữa context (Sales vs Billing); một model toàn cục → phình, mâu thuẫn; chia context giảm phức tạp.

**Q-DDD-002** ⭐⭐
"Ubiquitous Language là gì và giải quyết vấn đề gì giữa dev và domain expert?"
> *Dò cái gì:* ngôn ngữ chung dùng nhất quán trong cả trò chuyện *và* code; tránh dịch sai giữa nghiệp vụ và kỹ thuật; tên class/method phản ánh đúng thuật ngữ domain.

**Q-DDD-003** ⭐⭐⭐
"Phân biệt Entity và Value Object. Cho ví dụ mỗi loại và tiêu chí phân biệt."
> *Dò cái gì:* Entity có *identity* riêng (tồn tại qua thời gian dù thuộc tính đổi, vd User#42); Value Object định danh bằng *giá trị*, bất biến, thay vì sửa thì tạo mới (vd Money, Address); tiêu chí: có cần identity không.

**Q-DDD-004** ⭐⭐⭐⭐
"Aggregate và Aggregate Root là gì? Quy tắc 'tham chiếu aggregate khác bằng ID, không nhúng object' để làm gì?"
> *Dò cái gì:* aggregate = cụm object là một consistency boundary; root là cửa duy nhất truy cập/sửa bên trong; tham chiếu aggregate khác bằng ID giữ boundary + tránh load/sửa chéo + cho phép eventual consistency.

**Q-DDD-005** ⭐⭐⭐⭐
"'Invariant' trong DDD là gì? Vì sao aggregate root là nơi *bảo vệ* invariant?"
> *Dò cái gì:* invariant = quy tắc nghiệp vụ luôn phải đúng cho aggregate; mọi thay đổi đi qua root để root kiểm tra/giữ nhất quán nội bộ; ví dụ 'tổng item ≤ hạn mức đơn'.

**Q-DDD-006** ⭐⭐⭐⭐
"Aggregate boundary thường nên trùng với transaction boundary. Vì sao? Hệ quả khi một thao tác phải đổi *nhiều* aggregate?"
> *Dò cái gì:* một aggregate = một đơn vị nhất quán giao dịch (một tx sửa một aggregate); đổi nhiều aggregate → dùng eventual consistency/domain event/saga thay vì một tx khổng lồ; nối với ranh giới to/nhỏ ảnh hưởng contention.

**Q-DDD-007** ⭐⭐⭐
"Repository trong DDD khác một DAO/ORM thường thế nào? Vì sao repository nên 'nói ngôn ngữ domain'?"
> *Dò cái gì:* repository là collection-like cho aggregate root, ẩn persistence, trả về domain object (không leak ORM entity); interface theo nhu cầu domain ('findActiveOrdersFor(customer)') không theo khả năng DB. *(cross-ref E cho persistence chi tiết.)*

**Q-DDD-008** ⭐⭐⭐⭐
"Domain Service vs Application Service khác nhau ra sao? Một logic 'gửi email xác nhận sau khi đặt đơn' nên ở đâu?"
> *Dò cái gì:* Domain Service = rule nghiệp vụ thuần liên nhiều aggregate, không chạm hạ tầng; Application Service = điều phối use case, quản transaction, gọi ra ngoài; gửi email = side-effect hạ tầng → application service (qua port), KHÔNG nhét vào domain.

**Q-DDD-009** ⭐⭐⭐⭐
"Domain Event là gì và vì sao nó giúp giảm coupling giữa các bounded context/aggregate?"
> *Dò cái gì:* sự kiện 'đã xảy ra' trong domain (OrderPlaced), phát ra cho phần khác phản ứng async; aggregate không gọi trực tiếp aggregate/context khác → loose coupling, eventual consistency.

**Q-DDD-010** ⭐⭐⭐⭐⭐
"DDD có chi phí học/triển khai cao. Khi nào bạn *khuyên không* áp DDD chiến thuật đầy đủ cho một service? Dấu hiệu domain 'đủ phức tạp để đáng'?"
> *Dò cái gì:* CRUD đơn giản/ít rule/đội nhỏ → DDD đầy đủ là overhead; đáng khi nghiệp vụ phức tạp, nhiều invariant, ngôn ngữ domain giàu, thay đổi liên tục; phân biệt strategic DDD (luôn hữu ích: bounded context) với tactical (chọn lọc).

---

## Q-SMELL — Code Smell, Anti-pattern, Refactoring

**Q-SMELL-001** ⭐⭐
"Code smell là gì? Nó có phải là bug không? Vì sao vẫn phải quan tâm?"
> *Dò cái gì:* smell = dấu hiệu bề mặt báo vấn đề thiết kế tiềm ẩn, không phải bug/không làm sai chức năng; quan trọng vì báo trước nợ kỹ thuật, khó bảo trì/mở rộng nếu bỏ qua.

**Q-SMELL-002** ⭐⭐
"God Object / God Class là gì? Vì sao nó nguy hiểm và liên hệ nguyên tắc nào?"
> *Dò cái gì:* một class ôm quá nhiều trách nhiệm/biết quá nhiều; vi phạm SRP + cohesion thấp + coupling cao; mọi thay đổi đụng vào nó → rủi ro; sửa: tách trách nhiệm.

**Q-SMELL-003** ⭐⭐⭐
"'Feature Envy' là smell gì? Cách nhận ra và hướng refactor."
> *Dò cái gì:* method quan tâm dữ liệu/method của class *khác* hơn class của chính nó; báo hiệu method đặt sai chỗ; refactor: move method sang đúng class sở hữu dữ liệu.

**Q-SMELL-004** ⭐⭐⭐
"Phân biệt 'Divergent Change' và 'Shotgun Surgery'. Mỗi cái vi phạm trực giác nào?"
> *Dò cái gì:* Divergent Change = một class đổi vì nhiều lý do khác nhau (thiếu SRP → nên tách); Shotgun Surgery = một thay đổi buộc sửa rải rác nhiều class (coupling cao/thiếu gom → nên gộp lại); ngược nhau về phương.

**Q-SMELL-005** ⭐⭐⭐
"'Primitive Obsession' và 'Data Clumps' là gì? Cho ví dụ và cách sửa."
> *Dò cái gì:* lạm dụng kiểu nguyên thuỷ thay vì kiểu domain (string cho email/tiền); nhóm tham số luôn đi cùng nhau (lat/long/alt); sửa: tạo Value Object/object gói lại → encapsulate validation.

**Q-SMELL-006** ⭐⭐⭐⭐
"Một method 200 dòng nhiều cấp lồng + comment chia section. Bạn liệt kê những smell nào và refactor theo bước nào (giữ hành vi)?"
> *Dò cái gì:* Long Method, Deep Nesting, section-comment; refactor: extract method theo block/abstraction level, guard clause giảm nesting, đặt tên ý nghĩa; nhấn 'giữ behavior + có test trước'.

**Q-SMELL-007** ⭐⭐⭐⭐
"Refactoring (theo Fowler) định nghĩa chặt là gì? Vì sao 'phải có test' là điều kiện đi kèm? Refactor có được đổi hành vi không?"
> *Dò cái gì:* thay đổi *cấu trúc nội bộ* mà KHÔNG đổi hành vi quan sát được; test là lưới an toàn để chứng minh behavior không đổi; phân biệt refactor với 'sửa kèm thêm tính năng'.

**Q-SMELL-008** ⭐⭐⭐⭐
"Anti-pattern khác code smell thế nào? Cho 2 anti-pattern bạn hay gặp ở backend và vì sao chúng 'có vẻ đúng nhưng sai'."
> *Dò cái gì:* anti-pattern = 'giải pháp' phổ biến nhưng phản tác dụng (vs smell là dấu hiệu); ví dụ Golden Hammer, Lava Flow, Spaghetti, Magic Numbers/Strings, premature optimization; giải thích vì sao bẫy.

**Q-SMELL-009** ⭐⭐⭐⭐
"Lạm dụng *design pattern* cũng tạo smell. Cho một ví dụ 'pattern đúng nhưng dùng sai chỗ' gây hại."
> *Dò cái gì:* vd Decorator chồng quá sâu → object chain khó debug/overhead; Singleton tạo global state; Factory cho thứ chỉ `new` một lần; biết pattern là công cụ có chi phí, không phải huy chương.

**Q-SMELL-010** ⭐⭐⭐⭐⭐
"Trong code review bạn thấy smell nhưng deadline gấp. Là Tech Lead bạn quyết định fix-now / log nợ / bỏ qua dựa trên tiêu chí nào? Ghi nhận nợ kỹ thuật ra sao?"
> *Dò cái gì:* phân loại theo rủi ro lan/tần suất đụng/độ rủi ro thay đổi; quyết định có chủ đích (không cầu toàn), log nợ tường minh (ticket/TODO có chủ + lý do), tránh 'boy-scout' phá scope PR; cân chất lượng vs tốc độ giao.

---

## Q-TL — Judgment cấp Tech Lead

**Q-TL-001** ⭐⭐⭐⭐⭐
"'Khi nào KHÔNG nên dùng một design pattern?' Trả lời bằng nguyên tắc chung, không phải case lẻ."
> *Dò cái gì:* khi pattern thêm indirection/abstraction mà chưa có biến thiên/đau thật (YAGNI); khi đội chưa quen → giảm dễ đọc; chọn pattern *để giải vấn đề cụ thể*, không vì 'cho xịn'; ưu tiên đơn giản (KISS).

**Q-TL-002** ⭐⭐⭐⭐
"Over-engineering vs under-engineering: làm sao bạn nhận ra một thiết kế đang quá-thiết-kế? Dấu hiệu cụ thể."
> *Dò cái gì:* nhiều layer/interface 1-1/abstraction không có implementation thứ 2/flexibility cho tương lai chưa đến; ngược lại under = copy-paste, coupling, không seam test; biết tìm điểm cân theo bối cảnh.

**Q-TL-003** ⭐⭐⭐⭐
"Một developer giỏi muốn áp DDD + hexagonal + CQRS cho một CRUD nội bộ nhỏ. Bạn xử lý cuộc thảo luận này thế nào?"
> *Dò cái gì:* dẫn dắt bằng câu hỏi giá trị/chi phí, gắn với độ phức tạp thực + vòng đời + đội; không chặn cứng cũng không chiều; có thể đề 'bắt đầu đơn giản, để seam để tiến hoá'; ra quyết định có lý do.

**Q-TL-004** ⭐⭐⭐⭐⭐
"Làm sao bạn *enforce* nguyên tắc thiết kế (dependency rule, không import infra vào domain) trong một team đông mà không trở thành bottleneck review?"
> *Dò cái gì:* công cụ hoá — lint/ArchUnit-style/dependency-cruiser/eslint boundaries rule, module boundary trong CI, ADR + template, design review nhẹ, pairing/mentoring; biến nguyên tắc thành *guardrail tự động* thay vì phụ thuộc con người. *(verify tên tooling khi học.)*

**Q-TL-005** ⭐⭐⭐⭐
"ADR (Architecture Decision Record) là gì và vì sao TL nên dùng? Một ADR tối thiểu gồm gì?"
> *Dò cái gì:* tài liệu ngắn ghi *quyết định + bối cảnh + các lựa chọn + hệ quả*; giúp nhớ 'vì sao', onboard, tránh tranh luận lặp; gồm context, decision, alternatives, consequences/status.

**Q-TL-006** ⭐⭐⭐⭐⭐
"'Evolutionary architecture / design for change': làm sao thiết kế để dễ đổi *mà không* đoán trước mọi tương lai (chống speculative generality)?"
> *Dò cái gì:* tạo seam/abstraction tại điểm *thực sự* biến động, để hở chỗ chưa rõ thay vì khoá sớm; YAGNI + refactor liên tục khi biến thiên xuất hiện; phân biệt 'để dễ đổi' với 'đoán mò tương lai'.

**Q-TL-007** ⭐⭐⭐⭐
"AI coding assistant (Copilot/Cursor) sinh code 'chạy được' nhưng vi phạm thiết kế (vd nhét logic vào controller, import DB vào domain, lặp pattern sai). Là TL bạn kiểm soát chất lượng *thiết kế* thế nào?"
> *Dò cái gì:* AI lo cú pháp nhưng người *chỉ huy & kiểm tra* thiết kế; nêu guardrail (architecture lint/boundary test, PR checklist, review tập trung vào boundary/coupling, conventions trong prompt/repo); câu thần chú 'hiểu để chỉ huy, không thuộc để gõ'. *(R-trend, verify công cụ.)*

**Q-TL-008** ⭐⭐⭐⭐⭐
"Phân biệt 'design principle' (SOLID, coupling/cohesion) với 'tool/pattern/framework' khi đánh giá ứng viên hoặc khi tranh luận thiết kế. Vì sao TL phải lý luận ở tầng *nguyên tắc*?"
> *Dò cái gì:* pattern/framework là phương tiện, đổi theo thời gian; nguyên tắc ổn định và là cơ sở *vì sao* chọn phương tiện; TL phải biện minh quyết định bằng trade-off (coupling/cohesion/đổi-được) chứ không 'vì framework bảo thế'; nhận ra ứng viên thuộc-pattern vs hiểu-nguyên-tắc.

---

## Tự kiểm (đếm câu)

| Tag | Số câu |
|---|---|
| Q-SOLID | 14 |
| Q-OOP | 10 |
| Q-GOFC | 8 |
| Q-GOFS | 8 |
| Q-GOFB | 10 |
| Q-ARCH | 12 |
| Q-DDD | 10 |
| Q-SMELL | 10 |
| Q-TL | 8 |
| **Tổng** | **90** |
