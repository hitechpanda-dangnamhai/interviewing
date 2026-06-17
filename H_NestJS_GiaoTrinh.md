# 🎓 H — NestJS Framework Depth — Giáo trình & Bài giảng (WORKFLOW 2: Bước B + C)

> Sinh từ ngân hàng câu hỏi `QB_H_nestjs.md` (82 câu / 12 mục con).
> **Bước B** = dựng giáo trình *ngược* từ bộ đề (gom câu → bài, sắp cơ bản → nâng cao).
> **Bước C** = giảng từng bài theo **Hợp đồng đầu ra 10 mục**.
> Ngôn ngữ: tiếng Việt, giữ thuật ngữ Anh. Mọi mốc version đối chiếu **docs.nestjs.com**.

**Bối cảnh phiên bản (verify tại 6/2026):** NestJS **11** là bản hiện hành (11.1.x). Express **v5** là adapter mặc định; SWC là compiler mặc định *(verify)*; Jest vẫn là test runner mặc định ở 11 (Vitest là hướng dự kiến cho **v12**, chưa ra tại 6/2026). Mọi dòng chạm version/tooling đều có dấu *(verify)*.

> **Câu thần chú:** *"Tôi không nhớ để gõ, tôi hiểu để chỉ huy — và tra cứu phần còn lại."*

---

# 📚 BƯỚC B — GIÁO TRÌNH NGƯỢC TỪ BỘ ĐỀ

Bộ đề H có 12 mục con, nhưng thứ tự *hỏi* không phải thứ tự *học*. Dưới đây gom theo **cụm khái niệm** và sắp theo **dependency order** (cái sau cần cái trước), không theo tần suất hỏi.

## Sơ đồ phụ thuộc (vì sao học theo thứ tự này)

```
DI (lõi tuyệt đối)
  └─> Module system (DI sống trong module)
        └─> Request lifecycle (xương sống xử lý request)
              ├─> Pipes & Validation   (1 chặng trong lifecycle)
              ├─> Guards & Authz        (1 chặng trong lifecycle)
              └─> Interceptors & Filters(1 chặng trong lifecycle)
                    └─> Custom Decorators (chất keo: gắn/đọc metadata cho các chặng trên)
                          └─> Dynamic Modules & Config (đóng gói cấu hình cho mọi thứ trên)
                                ├─> Microservices  (mở rộng lifecycle sang transport khác HTTP)
                                ├─> CQRS / ES       (kiến trúc trong-app, dựa trên DI+module+bus)
                                └─> Testing         (test mọi thứ trên qua DI override)
                                      └─> Runtime / Version (góc nhìn TL, gom toàn cảnh)
```

## Mục lục bài học → ID câu hỏi nó phủ

| Bài | Tên bài | Phase¹ | Phủ các ID |
|---|---|---|---|
| **1** | DI nền tảng: provider, container, constructor injection | nền | H-DI-001, 002, 007, 009 |
| **2** | DI nâng cao: scope, custom provider, injection token, async | nâng | H-DI-003, 004, 005, 006, 008 |
| **3** | Module system: imports/exports, @Global, circular dep, ModuleRef, tổ chức codebase | nền→nâng | H-MOD-001→008 |
| **4** | Request lifecycle: thứ tự & ExecutionContext & binding | lõi | H-LIFE-001→007 |
| **5** | Pipes & Validation: DTO, class-validator/transformer, custom pipe | nền→nâng | H-PIPE-001→007 |
| **6** | Guards & Authorization: canActivate, RBAC, Reflector, ABAC | nền→nâng | H-GRD-001→006 |
| **7** | Interceptors & Exception filters: RxJS, envelope, error mapping | nâng | H-INT-001→006 |
| **8** | Custom decorators: param decorator, SetMetadata, applyDecorators, reflection | nâng | H-DEC-001→005 |
| **9** | Dynamic modules & Config: forRoot(Async), forFeature, ConfigModule, registerAsync | nâng | H-DYN-001→007 |
| **10** | Microservices: transport, message/event pattern, hybrid, gRPC, RpcException | nâng | H-MS-001→008 |
| **11** | CQRS & Event Sourcing: command/query/event bus, saga, projection | cao | H-CQRS-001→006 |
| **12** | Testing Nest sâu: TestingModule, e2e Supertest, TestContainers, override | nâng | H-TST-001→007 |
| **13** | Runtime / kiến trúc / version (TL depth): adapter, lifecycle hooks, SWC, monorepo, chọn version | cao | H-RT-001→006 |

¹ "Phase" ở đây là độ trưởng thành nội bộ của mục H (nền / nâng / lõi / cao), không phải Phase của giáo trình GenAI.

**Tổng kiểm:** 4+5+8+7+7+6+6+5+7+8+6+7+6 = **82 câu** — phủ trọn bộ đề, không bỏ ID nào.

---

# 📖 BƯỚC C — BÀI GIẢNG

# Bài 1 — DI nền tảng: provider, container, constructor injection
> Phủ: **H-DI-001, H-DI-002, H-DI-007, H-DI-009**

## ① Mục tiêu & vị trí trong mạch
Đây là **lõi tuyệt đối** của Nest — mọi thứ khác (module, guard, service, test) đều dựng trên DI. Học xong bài này bạn: giải thích được "provider" là gì, vì sao chỉ cần khai báo constructor là Nest tự "bơm" dependency vào, và vì sao DI làm code dễ test. Kết nối: bài sau (Bài 2) đào sâu *scope* và *custom provider*; Bài 3 đặt DI vào *module*.

## ② Giảng cơ bản → nâng cao

**(a) Trực giác.** Hãy hình dung DI như một **quán cà phê có barista**. Bạn (class) không tự đi mua sữa, mua máy pha; bạn chỉ ghi vào "đơn đặt hàng" (constructor) rằng "tôi cần `CoffeeService` và `MilkProvider`". Barista (DI container) đọc đơn, tự pha sẵn rồi đặt lên bàn cho bạn. Bạn không *new* thủ công — bạn *khai báo nhu cầu*, container lo phần tạo.

**(b) Cơ chế chi tiết (📘).** Provider = bất kỳ thứ gì Nest có thể *inject* vào nơi khác: service, repository, factory, giá trị hằng... Đánh dấu bằng `@Injectable()` để báo "class này do Nest quản lý vòng đời, có thể được inject và có thể inject thứ khác".

Khi app khởi động, container thực hiện:
1. **Đọc metadata** từ decorator (`@Injectable`, `@Module`) qua `reflect-metadata` (TypeScript phát ra type của tham số constructor nhờ `emitDecoratorMetadata`).
2. **Dựng dependency graph** — ai cần ai.
3. **Topological sort** — khởi tạo theo thứ tự phụ thuộc (cái không phụ thuộc gì tạo trước).
4. **Inject qua constructor** — tạo instance, truyền các dependency đã sẵn sàng vào.

```typescript
// users.service.ts
import { Injectable } from '@nestjs/common';

@Injectable()                       // (b) đánh dấu Nest quản lý
export class UsersService {
  // constructor injection: chỉ cần KHAI BÁO, không new
  constructor(private readonly mailer: MailerService) {}

  notify(id: string) {
    this.mailer.send(`Hello ${id}`); // mailer đã được container bơm vào
  }
}
```

**(c) Mép giới hạn & sai lầm thường gặp.**
- **Constructor injection là mặc định/khuyến nghị.** Chỉ dùng `@Inject(TOKEN)` khi dependency *không phải class* (token là string/Symbol) hoặc khi cần property injection trong base class (H-DI-007).
- Quên `@Injectable()` trên một provider được inject vào nơi khác → lỗi resolve.
- Tưởng `private readonly x: Foo` trong constructor là "khai báo biến" — thực ra đó là *cú pháp parameter property* của TS, và chính kiểu `Foo` là thứ Nest đọc để biết phải inject gì.

> 📘 docs: cơ chế provider + `@Injectable`. ➕ bổ sung: giải thích `reflect-metadata`/`emitDecoratorMetadata` (vì sao Nest "biết" type để inject).

## ③ ⚠️ Kiến thức cũ / bị thay thế

| Cái cũ (phổ biến / hay gặp) | Thay bằng (hiện nay) | Vì sao |
|---|---|---|
| `new UsersService(new MailerService())` thủ công trong code | Khai báo qua constructor + provider, để container resolve | Tách khởi tạo khỏi dùng → test/được swap dễ |
| Service factory tự viết, singleton thủ công | DI container quản singleton mặc định | Bớt boilerplate, đúng vòng đời |
| Inject bằng property gán tay `this.x = container.get(...)` | Constructor injection khai báo | Rõ phụ thuộc, không phụ thuộc ngầm |

(Không có version/giá ở bài này → không cần search.)

## ④ Áp dụng thực tế + So sánh bigtech
Use case thật: một `OrderService` cần `PaymentGateway` + `InventoryRepo` + `Logger`. Thay vì `OrderService` tự tạo ba thứ này (và phải biết cách cấu hình từng cái), nó chỉ khai báo trong constructor. Khi cần đổi `PaymentGateway` từ Stripe sang mock trong test → chỉ đổi ở provider, không đụng `OrderService`.

Bigtech/pattern phổ biến: DI là pattern nền của Spring (Java), Angular (Google), .NET. Các team lớn dùng DI để giữ **dependency một chiều** và **biên rõ ràng** giữa hàng chục module — đây là lý do Nest được chọn cho codebase đông kỹ sư (DI có sẵn, không cần container ngoài). *(Đây là pattern phổ biến, không phải số liệu cần cite.)*

## ⑤ Code thực hành + cấu hình

```typescript
// mailer.service.ts
import { Injectable } from '@nestjs/common';

@Injectable()
export class MailerService {
  send(msg: string) { console.log('[MAIL]', msg); }
}
```
```typescript
// users.service.ts  — service nhận MailerService qua DI
import { Injectable } from '@nestjs/common';
import { MailerService } from './mailer.service';

@Injectable()
export class UsersService {
  constructor(private readonly mailer: MailerService) {}
  welcome(name: string) { this.mailer.send(`Welcome ${name}`); }
}
```
```typescript
// app.module.ts — đăng ký provider để container biết tạo gì
import { Module } from '@nestjs/common';
import { UsersService } from './users.service';
import { MailerService } from './mailer.service';

@Module({ providers: [UsersService, MailerService] })
export class AppModule {}
```

**Cấu hình:**
```bash
npm i -g @nestjs/cli            # CLI (verify version tại docs.nestjs.com)
nest new my-app                 # scaffold (NestJS 11.x hiện hành)
# tsconfig phải bật:
#   "experimentalDecorators": true, "emitDecoratorMetadata": true
```
`package.json` (pin tham khảo — *luôn verify bản mới*):
```jsonc
{
  "dependencies": {
    "@nestjs/common": "^11.0.0",   // verify tại docs chính thức
    "@nestjs/core": "^11.0.0",
    "reflect-metadata": "^0.2.0",
    "rxjs": "^7.8.0"
  }
}
```

## ⑥ Keywords cần nhớ (glossary)
- 🧠 **Cần HIỂU:** DI container · dependency graph · inversion of control (IoC) · vì sao constructor injection làm code testable.
- 📌 **Cần THUỘC:** `@Injectable()` · constructor injection là mặc định · `providers` array trong `@Module` · `reflect-metadata`/`emitDecoratorMetadata`.
- 🛠️ **Cần LÀM ĐƯỢC:** khai báo 1 service, inject nó vào service khác, đăng ký vào module và chạy được.

## ⑦ Mạch tư duy cần nhớ (mental model)
> **"Đừng `new`, hãy khai báo nhu cầu ở constructor — container lo việc tạo và bơm."** DI = bạn nói *cần gì*, không nói *tạo thế nào*.

## ⑧ Câu hỏi phỏng vấn & thách đố
1. *(dễ)* `@Injectable()` đánh dấu điều gì? — Gợi ý: cho phép Nest quản vòng đời + inject. **(H-DI-002)**
2. *(TB)* Mô tả các bước container resolve một provider lúc runtime. — Gợi ý: đọc metadata → graph → sort → inject. **(H-DI-001)**
3. *(TB)* Khi nào buộc dùng `@Inject` thay vì constructor injection mặc định? — Gợi ý: token không phải class / property injection base class. **(H-DI-007)**
4. *(khó)* Vì sao DI giúp unit test? Bạn thay dependency thật bằng mock qua cơ chế nào? — Gợi ý: `overrideProvider().useValue()`. **(H-DI-009)**

**🎯 Thách đố:** *"Bạn viết `constructor(private mailer)` nhưng quên annotate type `MailerService`. Code compile nhưng app crash lúc boot — vì sao?"*
> Đáp: Nest dựa vào **type của tham số** (do `emitDecoratorMetadata` phát ra) để biết inject gì. Mất type → metadata là `Object`, container không resolve được token. Bài học: TS type ở đây *không chỉ để type-check*, nó là dữ liệu runtime của DI.

**🤖 "Hiểu để chỉ huy & kiểm tra":** AI có thể sinh service `new MailerService()` ngay trong `UsersService` — chạy được nhưng *phá DI* (không swap/test được). Người hiểu sẽ sửa thành inject qua constructor. *(Cú pháp đúng, kiến trúc sai — hãy kiểm tra mọi chỗ `new` một dependency.)*

## ⑨ Bài tập thực hành + tiêu chí tự chấm
**BT1:** Tạo `GreeterService` inject `MailerService`, expose `greet(name)`. **Đạt khi:** app chạy, gọi `greet` in ra mail, *không* có `new MailerService()` nào trong code.
**BT2:** Cố tình bỏ `@Injectable()` ở `MailerService` rồi đọc lỗi boot; thêm lại và giải thích lỗi. **Đạt khi:** đọc đúng thông điệp "can't resolve dependencies" và lý do.

## ⑩ Đọc thêm / nguồn chuẩn
- docs.nestjs.com → *Providers*, *Fundamentals → Custom providers* (phần intro).
- docs.nestjs.com → *Fundamentals → Injection scopes* (chuẩn bị cho Bài 2).

---

# Bài 2 — DI nâng cao: scope, custom provider, injection token, async
> Phủ: **H-DI-003, H-DI-004, H-DI-005, H-DI-006, H-DI-008**

## ① Mục tiêu & vị trí trong mạch
Sau khi hiểu DI cơ bản (Bài 1), bài này dạy bốn câu hỏi senior/TL hay hỏi: provider sống *bao lâu* (scope), làm sao *tùy biến* cách container tạo provider (custom provider), inject thứ *không phải class* (token), và xử lý dependency *async lúc khởi tạo*. Đây là ranh giới giữa "dùng được Nest" và "hiểu Nest".

## ② Giảng cơ bản → nâng cao

**(a) Trực giác.** Ba scope giống ba kiểu "sở hữu đồ vật":
- **DEFAULT (singleton)** = cái loa chung phòng họp — cả app dùng *một* instance.
- **REQUEST** = ly giấy mỗi khách một cái — mỗi HTTP request tạo *mới*.
- **TRANSIENT** = bút bi phát cho từng người — mỗi *nơi inject* nhận instance riêng.

**(b) Cơ chế chi tiết (📘).**

*Scope (H-DI-003, H-DI-004):* singleton là mặc định và rẻ nhất (tạo 1 lần, dùng lại). REQUEST tạo lại mỗi request — đắt. Điểm chí mạng: **scope "lây ngược lên" (bubble up)**. Nếu `ControllerA` phụ thuộc `ServiceB` REQUEST-scoped, thì `ControllerA` *cũng* thành REQUEST-scoped — mất tính singleton, khởi tạo lại toàn chain mỗi request → ảnh hưởng hiệu năng đáng kể ở traffic cao.

*Custom provider (H-DI-005):* bốn cách dạy container "tạo provider thế nào":

| Cú pháp | Dùng khi | Ví dụ |
|---|---|---|
| `useClass` | đổi implementation theo điều kiện | prod dùng `StripeGateway`, dev dùng `FakeGateway` |
| `useValue` | hằng / object có sẵn / mock trong test | config object, mảng constants |
| `useFactory` | tạo *động*, cần tính toán hoặc inject thứ khác | tạo client từ `ConfigService` |
| `useExisting` | tạo *alias* cho provider đã có | đặt tên cũ trỏ tới service mới |

*Injection token (H-DI-006):* TypeScript **interface biến mất lúc runtime** (chỉ là compile-time). Vì vậy không thể `constructor(private repo: IUserRepo)` rồi mong Nest inject — không có "thứ" runtime để làm key. Giải pháp: dùng **token** (string hoặc `Symbol`) làm khóa:
```typescript
export const USER_REPO = Symbol('USER_REPO');           // token bền hơn string
@Injectable() export class S { constructor(@Inject(USER_REPO) private repo: IUserRepo) {} }
```

*Async provider (H-DI-008):* khi giá trị provider cần chờ (vd kết nối DB đã sẵn sàng), dùng **`useFactory` async**. Nest **chờ factory resolve xong** rồi mới coi app sẵn sàng — nên đừng để route nhận request trước khi connection mở.

**(c) Mép giới hạn & sai lầm.**
- Lạm dụng REQUEST scope "cho chắc" → giết hiệu năng. Chỉ dùng khi *thực sự* cần state-per-request (vd tenant context) và cân nhắc giải pháp khác (AsyncLocalStorage).
- Inject interface trực tiếp (lỗi kinh điển của người từ Java sang) — phải qua token.
- `useFactory` async nhưng không `await` đúng bên trong → app "sẵn sàng" với connection chưa mở.

> 📘 docs: injection scopes, custom providers, async providers. ➕ bổ sung: nhấn mạnh chi phí "scope bubble-up" và gợi ý AsyncLocalStorage thay REQUEST scope.

## ③ ⚠️ Kiến thức cũ / bị thay thế

| Cái cũ | Thay bằng (hiện nay) | Vì sao |
|---|---|---|
| Dùng REQUEST scope để giữ context per-request | **AsyncLocalStorage** (hoặc CLS) giữ context, provider vẫn singleton | Tránh bubble-up giết perf |
| Inject bằng string token tùy tiện (`'USER_REPO'`) | **`Symbol`** hoặc hằng token export chung | Tránh va token trùng tên |
| Tạo client (Redis/DB) trong constructor đồng bộ rồi "hy vọng đã connect" | **async `useFactory`** chờ resolve | App chỉ ready khi dependency thật sự sẵn sàng |

## ④ Áp dụng thực tế + So sánh bigtech
Use case: app multi-tenant cần biết "request này thuộc tenant nào". Người mới chọn REQUEST-scoped `TenantContextService` — đúng nhưng đắt. Cách production phổ biến: middleware/interceptor set tenant vào **AsyncLocalStorage**, service singleton đọc ra → giữ singleton + vẫn per-request. Đây là pattern các team high-traffic ưa dùng (giống thread-local của Java nhưng cho async). *(Pattern phổ biến, không phải số liệu cố định.)*

## ⑤ Code thực hành + cấu hình
```typescript
// custom provider: useFactory async tạo Redis client từ ConfigService
import { Module } from '@nestjs/common';
import Redis from 'ioredis';                  // verify version tại docs ioredis

export const REDIS = Symbol('REDIS');

@Module({
  providers: [{
    provide: REDIS,
    useFactory: async (config: ConfigService) => {
      const client = new Redis(config.get('REDIS_URL'));
      await client.ping();                     // chờ connection thật sự lên
      return client;                           // Nest chờ Promise này resolve
    },
    inject: [ConfigService],                   // inject vào factory
  }],
  exports: [REDIS],
})
export class RedisModule {}
```
```typescript
// inject token (không phải class) qua @Inject
import { Inject, Injectable } from '@nestjs/common';
@Injectable()
export class CacheService {
  constructor(@Inject(REDIS) private readonly redis: Redis) {}
}
```
> `// kiểm chứng tại docs chính thức`: API ioredis/cấu hình client có thể đổi theo bản.

## ⑥ Keywords cần nhớ
- 🧠 **Cần HIỂU:** scope bubble-up · vì sao REQUEST scope đắt · vì sao interface không inject được · async readiness.
- 📌 **Cần THUỘC:** `Scope.DEFAULT/REQUEST/TRANSIENT` · `useClass/useValue/useFactory/useExisting` · `@Inject(TOKEN)` · `inject: [...]` trong factory.
- 🛠️ **Cần LÀM ĐƯỢC:** viết 1 async `useFactory` provider + inject qua token; chọn đúng loại custom provider cho 1 tình huống.

## ⑦ Mental model
> **"Provider không chỉ là *cái gì*, mà còn là *sống bao lâu* (scope) và *tạo bằng cách nào* (use*)."** Token là chìa khóa khi 'cái gì' không tồn tại lúc runtime.

## ⑧ Câu hỏi phỏng vấn & thách đố
1. *(TB)* Ba scope khác nhau gì? Trade-off của REQUEST? **(H-DI-003)**
2. *(khó)* REQUEST scope "lây" lên cha thế nào, vì sao lo perf? **(H-DI-004)**
3. *(khó)* Bốn loại custom provider, mỗi loại 1 ví dụ đúng. **(H-DI-005)**
4. *(khó)* Vì sao inject interface trực tiếp không được, fix sao? **(H-DI-006)**
5. *(rất khó)* Provider cần giá trị async lúc khởi tạo — cung cấp qua DI thế nào? **(H-DI-008)**

**🎯 Thách đố:** *"Một `LoggerService` (DEFAULT) inject một `RequestIdService` (REQUEST). Bạn thấy `LoggerService` bỗng được tạo lại mỗi request dù bạn không đổi scope của nó — vì sao?"*
> Đáp: **bubble-up** — phụ thuộc REQUEST-scoped kéo `LoggerService` thành REQUEST-scoped. Fix: lấy request-id qua AsyncLocalStorage để `LoggerService` ở lại singleton.

**🤖 "Hiểu để chỉ huy":** AI thường mặc định mọi service là singleton — đúng phần lớn, nhưng *sai* khi state thực sự phụ thuộc request. Ngược lại, AI cũng có thể "rắc" REQUEST scope khắp nơi. Người hiểu mới quyết được chỗ nào cần scope nào.

## ⑨ Bài tập + tiêu chí tự chấm
**BT1:** Viết provider `CONFIG_VALUE` bằng `useValue` và inject vào 1 service. **Đạt khi:** inject qua token chạy đúng.
**BT2:** Đổi nó sang `useFactory` async (giả lập `await sleep(50)` rồi trả config). **Đạt khi:** app chỉ "Nest application successfully started" sau khi factory resolve.

## ⑩ Đọc thêm
- docs.nestjs.com → *Fundamentals → Injection scopes*, *Custom providers*, *Async providers*.
- docs.nestjs.com → *Techniques → Async local storage* (thay thế REQUEST scope).

---

# Bài 3 — Module system: imports/exports, @Global, circular dep, ModuleRef, tổ chức codebase
> Phủ: **H-MOD-001 → H-MOD-008**

## ① Mục tiêu & vị trí trong mạch
DI (Bài 1–2) cần một "nơi để sống" — đó là **module**. Module quyết định *provider nào thấy provider nào*. Học xong: tổ chức được app theo feature, chia sẻ provider đúng cách, gỡ circular dependency, và tư duy TL khi codebase phình to.

## ② Giảng cơ bản → nâng cao

**(a) Trực giác.** Module như **một phòng ban trong công ty**. Mỗi phòng có nhân sự nội bộ (`providers`), có người tiếp khách (`controllers`), nhập tài nguyên từ phòng khác (`imports`), và *chỉ* cho mượn ai mình *công khai cho mượn* (`exports`). Không export = nhân sự nội bộ, phòng khác không gọi được.

**(b) Cơ chế chi tiết (📘).**

*4 field của `@Module` (H-MOD-001):*

| Field | Vai trò |
|---|---|
| `imports` | kéo module *khác* vào để dùng provider chúng **export** |
| `controllers` | khai báo route handler của module |
| `providers` | đăng ký provider *nội bộ* module |
| `exports` | công khai provider cho module nào **import** mình |

*Chia sẻ provider (H-MOD-002):* để `ModuleB` dùng `FooService` của `ModuleA`: A phải `exports: [FooService]` **và** B phải `imports: [ModuleA]`. Thiếu một trong hai → không thấy.

*`@Global()` (H-MOD-003):* đánh dấu module global → provider export của nó khả dụng *toàn app* không cần import lại. Tiện cho cross-cutting (config, logger) nhưng **dùng tiết chế**: lạm dụng làm mờ phụ thuộc (không thấy ai import gì), khó trace, phá tính module hóa.

*Circular dependency (H-MOD-004):* A cần B, B cần A → container không biết tạo ai trước. `forwardRef(() => X)` ở *cả hai phía* trì hoãn resolve để phá vòng. **Nhưng** thông điệp quan trọng hơn: circular dep thường là **mùi thiết kế** — nên tách phần chung ra module thứ ba thay vì vá bằng forwardRef.

*`ModuleRef` (H-MOD-005):* "service locator" của Nest — lấy provider theo token *tại runtime* (`moduleRef.get(Token)` / `moduleRef.resolve(Token)` cho scoped/transient). Dùng khi *không biết token lúc compile* (plugin/factory động) — không lạm dụng vì làm phụ thuộc ngầm.

*Static vs dynamic module (H-MOD-006):* static = cấu hình cứng trong `@Module`. Dynamic = method (vd `forRoot(options)`) **trả về metadata module được cấu hình lúc import** → truyền options vào. (Đào sâu ở Bài 9.)

*Lazy loading (H-MOD-008):* `LazyModuleLoader` nạp module *theo nhu cầu* thay vì lúc boot → giảm cold-start/RAM. Lợi ích lớn với **serverless/CLI**; với HTTP server long-running thì nhỏ (đằng nào cũng nạp hết). *(verify API hiện hành.)*

**(c) Mép giới hạn & sai lầm (H-MOD-007 — tư duy TL).**
- **God module** (một module ôm hết) → coupling cao, khó test. Chia theo **bounded context/feature**.
- Export bừa mọi provider → mất ranh giới. Chỉ export thứ *thật sự* là API của module.
- Dependency hai chiều giữa feature module → tách `SharedModule`/`CoreModule` cho cross-cutting, giữ **dependency một chiều**.

> 📘 docs: module, exports/imports, @Global, dynamic/lazy, ModuleRef, circular dep. ➕ bổ sung: nguyên tắc tổ chức codebase lớn (bounded context, one-way dependency).

## ③ ⚠️ Kiến thức cũ / bị thay thế

| Cái cũ | Thay bằng (hiện nay) | Vì sao |
|---|---|---|
| Nhồi hết vào `AppModule` | Tách feature module + Shared/Core module | Tránh God module, dễ test/scale |
| Vá circular dep bằng `forwardRef` rồi để đó | Tách concept chung ra module thứ 3 | forwardRef là *triệu chứng*, không phải *thuốc* |
| `@Global()` cho nhiều thứ "cho tiện" | Import tường minh; @Global chỉ cho cross-cutting thật | Giữ phụ thuộc tường minh, trace được |

## ④ Áp dụng thực tế + So sánh bigtech
App e-commerce: `OrdersModule`, `PaymentsModule`, `UsersModule`, cộng `SharedModule` (logger/config) và `DatabaseModule` (`@Global` connection). Orders import Payments + Users; Payments **không** import Orders (one-way). Pattern này phản ánh cách team lớn map **bounded context** (DDD) → module Nest. Bigtech dùng monorepo + module boundary để 50+ kỹ sư không giẫm chân nhau (xem Bài 13).

## ⑤ Code thực hành + cấu hình
```typescript
// shared.module.ts — export để dùng chung
import { Module } from '@nestjs/common';
import { LoggerService } from './logger.service';
@Module({ providers: [LoggerService], exports: [LoggerService] }) // exports mới chia sẻ được
export class SharedModule {}
```
```typescript
// orders.module.ts — import để xài LoggerService
import { Module } from '@nestjs/common';
import { SharedModule } from '../shared/shared.module';
import { OrdersService } from './orders.service';
@Module({ imports: [SharedModule], providers: [OrdersService] })
export class OrdersModule {}
```
```typescript
// gỡ circular dep bằng forwardRef (cả hai phía) — nhưng cân nhắc tách module
@Module({ imports: [forwardRef(() => PaymentsModule)] })
export class OrdersModule {}
// trong service: constructor(@Inject(forwardRef(() => PaymentsService)) private p: PaymentsService) {}
```

## ⑥ Keywords cần nhớ
- 🧠 **Cần HIỂU:** ranh giới visibility (export/import) · vì sao circular dep là mùi thiết kế · bounded context.
- 📌 **Cần THUỘC:** 4 field `@Module` · `forwardRef()` · `@Global()` · `ModuleRef.get/resolve` · static vs dynamic.
- 🛠️ **Cần LÀM ĐƯỢC:** chia 1 app thành ≥3 module với one-way dependency; export/import đúng; gỡ 1 circular dep.

## ⑦ Mental model
> **"Provider 'private' trong module; muốn dùng chung phải `export` + `import`. Circular dep = đi tách module, không phải đi vá."**

## ⑧ Câu hỏi phỏng vấn & thách đố
1. *(dễ)* 4 field của `@Module` làm gì? **(H-MOD-001)**
2. *(TB)* B dùng provider của A cần gì? **(H-MOD-002)**
3. *(TB)* `@Global()` để làm gì, vì sao tiết chế? **(H-MOD-003)**
4. *(khó)* Circular dep xảy ra sao, `forwardRef` giải quyết thế nào, và nó *báo hiệu* gì? **(H-MOD-004)**
5. *(khó)* `ModuleRef` khi nào cần thay vì inject thẳng? **(H-MOD-005)**
6. *(rất khó)* Tổ chức codebase Nest lớn tránh God module & coupling chéo? **(H-MOD-007)**

**🎯 Thách đố:** *"Bạn export một provider từ ModuleA, ModuleB import ModuleA nhưng vẫn 'can't resolve' — có thể vì lý do gì khác ngoài quên export?"*
> Đáp: provider có thể là **REQUEST-scoped** trong khi nơi dùng là static/eager; hoặc bị **shadow** bởi một provider cùng token đăng ký lại trong B (B tự `providers` lại → instance khác). Hoặc circular import khiến A chưa init xong.

**🤖 "Hiểu để chỉ huy":** AI hay "sửa" circular dep bằng cách rải `forwardRef` — chạy được nhưng giấu vấn đề thiết kế. Người hiểu sẽ hỏi "tại sao hai module này biết nhau?" và tách module.

## ⑨ Bài tập + tiêu chí tự chấm
**BT1:** Dựng `AppModule` gồm `UsersModule` + `SharedModule(LoggerService)`. Users dùng Logger. **Đạt khi:** chỉ export Logger, Users import Shared, chạy đúng; thử bỏ `exports` để thấy lỗi.
**BT2:** Tạo cố ý circular dep A↔B, gỡ bằng forwardRef, rồi *refactor* tách phần chung ra `CommonModule`. **Đạt khi:** bản refactor không còn forwardRef.

## ⑩ Đọc thêm
- docs.nestjs.com → *Modules*, *Fundamentals → Circular dependency*, *Module reference*, *Lazy-loading modules*.

---

# Bài 4 — Request lifecycle: thứ tự, ExecutionContext, binding
> Phủ: **H-LIFE-001 → H-LIFE-007**

## ① Mục tiêu & vị trí trong mạch
Đây là **xương sống** của Nest: một request đi qua những chặng nào, theo thứ tự nào, và mỗi chặng "biết" được gì. Hiểu bài này thì 3 bài kế (Pipes/Guards/Interceptors) chỉ là "phóng to" từng chặng. Nếu không nắm thứ tự, bạn sẽ đặt auth sai chỗ, validate sai thời điểm.

## ② Giảng cơ bản → nâng cao

**(a) Trực giác.** Request như khách vào tòa nhà: **bảo vệ cổng (middleware)** → **kiểm tra thẻ ra vào (guard)** → **camera ghi hình vào (interceptor pre)** → **soi hành lý/đổi tiền tệ (pipe)** → **gặp người cần gặp (controller handler)** → **camera ghi hình ra (interceptor post)** → nếu có sự cố thì **đội xử lý sự cố (exception filter)**.

**(b) Cơ chế chi tiết (📘) — thứ tự chuẩn (H-LIFE-001):**
```
Incoming request
  → Middleware
    → Guards
      → Interceptors (pre-controller)
        → Pipes
          → Controller handler (method)
        ← Interceptors (post-controller, map response)
  ← Exception filter (nếu có exception bị ném bất kỳ đâu phía trên)
Response out
```

*Middleware vs Guard (H-LIFE-002):* middleware là khái niệm Express thuần — **không** biết `ExecutionContext`/handler metadata. Guard có `ExecutionContext` + `Reflector` → đọc được route/handler/metadata (vd `@Roles`) để quyết định. Vì vậy **auth/authz đặt ở guard**, không ở middleware.

*Pipe sau guard (H-LIFE-003):* guard chạy **trước** pipe. Nên nếu validation cần biết user đã auth, lấy user từ guard (đã gắn vào request), **đừng** nhét logic auth vào pipe.

*ExecutionContext (H-LIFE-004):* abstraction cho ngữ cảnh hiện tại — có thể là HTTP, RPC (microservice), hay WebSocket. Cho phép lấy `getHandler()` + `getClass()` để đọc metadata, và `switchToHttp()/switchToRpc()/switchToWs()`. Nhờ nó, một guard/interceptor chạy được trên *nhiều transport*.

*Exception rẽ nhánh (H-LIFE-005):* khi handler ném lỗi, luồng **nhảy thẳng sang exception filter**; phần "map response" của interceptor (sau handler) **không chạy bình thường** — trừ khi bạn dùng `catchError` trong RxJS pipe của interceptor để bắt trước. Hiểu rẽ nhánh này để không kỳ vọng interceptor post luôn chạy.

*Binding 3 cấp (H-LIFE-006):* guard/pipe/interceptor/filter gắn được ở **global / controller / method**. Phạm vi hẹp hơn ghi đè/được áp thêm tùy loại.

*useGlobalGuards vs APP_GUARD (H-LIFE-007 — câu khó nhất):*

| Cách | Qua DI? | Inject service được? |
|---|---|---|
| `app.useGlobalGuards(new MyGuard())` | ❌ không | ❌ (tự `new`, không có DI) |
| Provider `{ provide: APP_GUARD, useClass: MyGuard }` | ✅ có | ✅ inject `Reflector`/service |

→ Nếu guard cần inject (gần như luôn cần `Reflector`), **dùng `APP_GUARD`**.

**(c) Mép giới hạn & sai lầm.**
- Đặt auth ở middleware rồi ngạc nhiên vì không đọc được `@Roles`.
- Tưởng interceptor post luôn chạy kể cả khi lỗi.
- `app.useGlobalGuards()` với guard cần `Reflector` → guard nhận `undefined`.

## ③ ⚠️ Kiến thức cũ / bị thay thế

| Cái cũ | Thay bằng | Vì sao |
|---|---|---|
| Auth bằng Express middleware | **Guard** (`canActivate` + Reflector) | Đọc được metadata route/role |
| `app.useGlobalGuards(new X())` cho guard cần service | Provider **`APP_GUARD`** | Mới inject được qua DI |
| Validate trong handler thủ công | **Pipe** chạy trước handler | Tách concern, khai báo |

## ④ Áp dụng thực tế + So sánh bigtech
API có rate-limit (middleware) → auth JWT (guard) → log + timeout (interceptor) → validate body (pipe) → handler → chuẩn hóa response `{data}` (interceptor) → lỗi nào cũng về 1 format (filter). Đây gần như là "kiến trúc tối thiểu" của mọi REST service production trên Nest — và là lý do Nest được ưa ở team lớn: các cross-cutting concern có *chỗ chuẩn* để đặt thay vì rải rác.

## ⑤ Code thực hành + cấu hình
```typescript
// main.ts — pipe global qua app (đơn giản, không cần DI)
const app = await NestFactory.create(AppModule);
app.useGlobalPipes(new ValidationPipe({ whitelist: true, transform: true }));
await app.listen(3000);
```
```typescript
// guard global qua DI (APP_GUARD) — để inject Reflector/service
import { APP_GUARD } from '@nestjs/core';
@Module({ providers: [{ provide: APP_GUARD, useClass: JwtAuthGuard }] }) // chạy trong DI
export class AppModule {}
```
```typescript
// guard đọc ExecutionContext + Reflector
@Injectable()
export class RolesGuard implements CanActivate {
  constructor(private reflector: Reflector) {}
  canActivate(ctx: ExecutionContext): boolean {
    const roles = this.reflector.getAllAndOverride<string[]>('roles',
      [ctx.getHandler(), ctx.getClass()]);   // Nest 11: trả T | undefined (verify)
    if (!roles) return true;
    const { user } = ctx.switchToHttp().getRequest();
    return roles.some(r => user?.roles?.includes(r));
  }
}
```

## ⑥ Keywords cần nhớ
- 🧠 **Cần HIỂU:** thứ tự lifecycle · vì sao auth ở guard không ở middleware · rẽ nhánh exception · vì sao APP_GUARD inject được.
- 📌 **Cần THUỘC:** middleware→guard→interceptor(pre)→pipe→handler→interceptor(post)→filter · `ExecutionContext` · `APP_GUARD/APP_PIPE/APP_INTERCEPTOR/APP_FILTER`.
- 🛠️ **Cần LÀM ĐƯỢC:** vẽ lại sơ đồ lifecycle từ trí nhớ; bind 1 guard global qua APP_GUARD.

## ⑦ Mental model
> **"middleware → guard → interceptor → pipe → handler → (interceptor) → filter. Guard biết metadata, middleware thì không. Cần inject → APP_*, không phải useGlobal*()."**

## ⑧ Câu hỏi phỏng vấn & thách đố
1. *(khó)* Kể đúng thứ tự lifecycle cả chiều vào & ra. **(H-LIFE-001)**
2. *(TB)* Middleware vs guard, vì sao auth ở guard? **(H-LIFE-002)**
3. *(khó)* Pipe trước hay sau guard? Validation cần user thì sao? **(H-LIFE-003)**
4. *(TB)* ExecutionContext là gì, vì sao cần? **(H-LIFE-004)**
5. *(rất khó)* `useGlobalGuards` vs `APP_GUARD` khác gì, khi nào buộc dùng cách nào? **(H-LIFE-007)**

**🎯 Thách đố:** *"Bạn ném `ForbiddenException` trong handler nhưng interceptor logging (post) không in 'response sent' — vì sao, và làm sao vẫn log được cả khi lỗi?"*
> Đáp: lỗi rẽ sang exception filter, nhánh map-response của interceptor không chạy. Muốn log cả lỗi → dùng `catchError`/`tap({error})` trong RxJS pipe của interceptor, hoặc log ở exception filter.

**🤖 "Hiểu để chỉ huy":** AI có thể gắn AuthGuard bằng `useGlobalGuards(new AuthGuard())` — chạy ở demo nhưng guard không inject được `Reflector` → kiểm tra: guard cần service thì **phải** qua `APP_GUARD`.

## ⑨ Bài tập + tiêu chí tự chấm
**BT1:** Vẽ (text) lại sơ đồ lifecycle, chú thích chặng nào đọc được metadata. **Đạt khi:** đúng thứ tự + đúng chỗ guard/pipe.
**BT2:** Triển khai `LoggingInterceptor` log cả success lẫn error (dùng `tap`/`catchError`). **Đạt khi:** request lỗi vẫn có log.

## ⑩ Đọc thêm
- docs.nestjs.com → *Overview → Middleware/Guards/Interceptors/Pipes/Exception filters*; *Fundamentals → Execution context*; *FAQ → Request lifecycle*.

---

# Bài 5 — Pipes & Validation: DTO, class-validator/transformer, custom pipe
> Phủ: **H-PIPE-001 → H-PIPE-007**

## ① Mục tiêu & vị trí trong mạch
Pipe là chặng *ngay trước handler* trong lifecycle (Bài 4). Bài này dạy cách chặn input bẩn ở biên (validate) và ép kiểu (transform) — lá chắn đầu tiên của API. Kết nối: validation tốt là tiền đề cho security (chống mass-assignment) ở mục M roadmap.

## ② Giảng cơ bản → nâng cao

**(a) Trực giác.** Pipe như **trạm hải quan**: hành lý (input) phải khai đúng *form* (validate) và đôi khi *quy đổi tiền tệ* (transform `'42'` → `42`) trước khi vào nội địa (handler).

**(b) Cơ chế chi tiết (📘).**

*Hai vai trò pipe (H-PIPE-001):* **transform** (đổi/ép kiểu input) và **validate** (chặn input sai) — chạy trước khi giá trị tới handler.

*DTO + class-validator + ValidationPipe (H-PIPE-002):* DTO là class mô tả shape request; gắn decorator (`@IsString()`, `@IsEmail()`, `@Min()`...). `ValidationPipe` đọc metadata đó, validate, ném **400** nếu sai.

*Options quan trọng (H-PIPE-003):*

| Option | Tác dụng | Vì sao bật |
|---|---|---|
| `whitelist: true` | **loại** field không khai trong DTO | chống mass-assignment, payload rác |
| `forbidNonWhitelisted: true` | **báo lỗi** nếu có field lạ | chặt hơn whitelist |
| `transform: true` | ép body thành **instance DTO** + ép kiểu primitive | DTO thành object thật, `@Type` chạy |

*class-transformer (H-PIPE-004):* payload HTTP là **plain object** (JSON parse ra), không phải instance DTO → một số validator nested/array không chạy đúng nếu chưa transform. `@Type(() => ChildDto)` báo class-transformer cách dựng nested instance; `plainToInstance` chuyển plain → class. Đây là lý do "chỉ class-validator là chưa đủ".

*Custom pipe (H-PIPE-005):* implement `PipeTransform`, method `transform(value, metadata: ArgumentMetadata)` — nhận value + metadata (type/metatype/data), **trả** giá trị đã transform **hoặc ném** exception. Pipe chỉ tác động *một tham số được bind*.

**(c) Mép giới hạn & sai lầm.**
- *Validation biên vs domain (H-PIPE-006):* pipe lo **shape/format** ở biên (HTTP); **invariant nghiệp vụ** (vd "số dư không âm") thuộc **domain layer**, đừng dồn hết vào DTO.
- Quên `transform: true` → `@Type` không chạy → nested DTO không được validate (lỗ hổng âm thầm).
- Đặt rule nghiệp vụ phức tạp vào DTO → DTO phình, khó tái dùng.

*Hướng tương lai (H-PIPE-007, ➕):* **Nest 12 dự kiến** hỗ trợ **Standard Schema** (Zod/Valibot) ngay trong route decorator — tức validate **schema-based** bên cạnh **decorator-based** (class-validator). Trade-off: Zod cho **type-inference** mạnh (1 nguồn sự thật cho type + validate); class-validator quen thuộc, gắn liền DTO class. *(verify trạng thái v12 — chưa ra tại 6/2026.)*

## ③ ⚠️ Kiến thức cũ / bị thay thế

| Cái cũ | Thay bằng (hiện nay/sắp tới) | Vì sao |
|---|---|---|
| Validate thủ công trong handler (`if (!body.email)`) | DTO + class-validator + ValidationPipe | Khai báo, tái dùng, tự ném 400 |
| ValidationPipe không bật whitelist | `whitelist`+`forbidNonWhitelisted`+`transform` | Chống field rác/mass-assignment |
| Chỉ class-validator cho nested | + class-transformer `@Type` / `transform:true` | Nested/array mới validate đúng |
| (xu hướng) chỉ decorator-based | + **Standard Schema/Zod** (Nest 12, *verify*) | Type-inference, 1 nguồn type+validate |

## ④ Áp dụng thực tế + So sánh bigtech
API tạo user: `CreateUserDto { @IsEmail() email; @MinLength(8) password; @Type(()=>AddressDto) @ValidateNested() address }`. Bật ValidationPipe global với whitelist+transform → client gửi thừa `isAdmin: true` cũng bị loại sạch (chặn privilege escalation). Pattern "DTO ở biên, domain rule ở trong" là chuẩn ở team lớn; nhiều team mới đang chuyển sang **Zod** cho type-safety end-to-end (chia sẻ schema FE↔BE).

## ⑤ Code thực hành + cấu hình
```typescript
// create-user.dto.ts
import { IsEmail, MinLength, ValidateNested, IsOptional } from 'class-validator';
import { Type } from 'class-transformer';

class AddressDto { @MinLength(1) city: string; }

export class CreateUserDto {
  @IsEmail() email: string;
  @MinLength(8) password: string;
  @IsOptional() @ValidateNested() @Type(() => AddressDto) // @Type cần cho nested
  address?: AddressDto;
}
```
```typescript
// main.ts — bật ValidationPipe global đúng cách
app.useGlobalPipes(new ValidationPipe({
  whitelist: true,             // bỏ field thừa
  forbidNonWhitelisted: true,  // báo lỗi nếu có field lạ
  transform: true,             // ép thành instance DTO + ép kiểu
}));
```
```typescript
// custom pipe: ParseObjectIdPipe
import { PipeTransform, BadRequestException, ArgumentMetadata } from '@nestjs/common';
export class ParseObjectIdPipe implements PipeTransform {
  transform(value: string, _meta: ArgumentMetadata) {
    if (!/^[a-f\d]{24}$/i.test(value)) throw new BadRequestException('Invalid id');
    return value; // hoặc new ObjectId(value)
  }
}
```
```jsonc
// requirements (verify bản mới): class-validator, class-transformer
{ "dependencies": { "class-validator": "^0.14.0", "class-transformer": "^0.5.0" } }
```

## ⑥ Keywords cần nhớ
- 🧠 **Cần HIỂU:** plain object vs instance · vì sao cần transform cho nested · biên vs domain validation.
- 📌 **Cần THUỘC:** `ValidationPipe({whitelist,forbidNonWhitelisted,transform})` · `@Type` · `@ValidateNested` · `PipeTransform.transform(value, metadata)`.
- 🛠️ **Cần LÀM ĐƯỢC:** viết 1 DTO nested validate đúng + 1 custom pipe ném 400.

## ⑦ Mental model
> **"DTO = hợp đồng input ở biên; ValidationPipe = người gác cổng đọc hợp đồng. Nested → nhớ `@Type` + `transform:true`."**

## ⑧ Câu hỏi phỏng vấn & thách đố
1. *(dễ)* Hai vai trò của pipe? **(H-PIPE-001)**
2. *(TB)* DTO+class-validator+ValidationPipe phối hợp sao? **(H-PIPE-002)**
3. *(khó)* whitelist/forbidNonWhitelisted/transform làm gì, vì sao bật? **(H-PIPE-003)**
4. *(khó)* Vì sao chỉ class-validator chưa đủ cho nested? **(H-PIPE-004)**
5. *(rất khó)* Standard Schema/Zod (Nest 12) đổi cách validate thế nào? **(H-PIPE-007)**

**🎯 Thách đố:** *"DTO của bạn có `address: AddressDto` với `@ValidateNested()` nhưng client gửi address sai vẫn lọt qua. Thiếu gì?"*
> Đáp: thiếu `@Type(() => AddressDto)` (và/hoặc `transform: true`). Không có `@Type`, class-transformer không dựng nested thành instance → validator nested không chạy.

**🤖 "Hiểu để chỉ huy":** AI sinh DTO + `@ValidateNested` nhìn "đủ" nhưng quên `@Type` → validate nested **âm thầm bỏ qua**. Cú pháp đúng, hành vi sai — phải tự test 1 payload nested bẩn để chắc nó bị chặn.

## ⑨ Bài tập + tiêu chí tự chấm
**BT1:** DTO có nested array `items: ItemDto[]`. **Đạt khi:** gửi item sai bị 400 (chứng tỏ `@Type` + `@ValidateNested({each:true})` đúng).
**BT2:** Bật `forbidNonWhitelisted`, gửi field thừa → nhận 400 liệt kê field lạ. **Đạt khi:** đúng hành vi.

## ⑩ Đọc thêm
- docs.nestjs.com → *Techniques → Validation*; *Pipes*. Trang class-validator/class-transformer chính thức. Theo dõi RFC Standard Schema cho Nest 12 *(verify)*.

---

# Bài 6 — Guards & Authorization: canActivate, RBAC, Reflector, ABAC
> Phủ: **H-GRD-001 → H-GRD-006**

## ① Mục tiêu & vị trí trong mạch
Guard là chặng *quyết định cho-qua-hay-chặn* trong lifecycle (Bài 4), nơi đặt **authorization**. Bài này dạy luồng RBAC khai báo (`@Roles` + Reflector) và khi nào RBAC không đủ thì lên ABAC/policy. Kết nối Bài 8 (decorator) và mục M roadmap (security).

## ② Giảng cơ bản → nâng cao

**(a) Trực giác.** Guard là **người soát vé**: nhìn vé (request + user) và *luật của cửa này* (metadata `@Roles`) rồi nói "vào được / mời ra". Khác pipe (đổi/soi hành lý) — guard chỉ phán **yes/no**.

**(b) Cơ chế chi tiết (📘).**

*`canActivate` (H-GRD-001):* trả `boolean | Promise<boolean> | Observable<boolean>`. `true` → qua; `false`/throw → chặn (mặc định **403**). Dùng cho authz và precondition.

*Luồng RBAC (H-GRD-002):*
```
@Roles('admin')            // (1) decorator gắn METADATA lên handler/class
   → RolesGuard.canActivate // (2) guard chạy
      → reflector.getAllAndOverride('roles', [handler, class])  // (3) đọc metadata
      → so role metadata với user.roles (từ request đã auth)    // (4) quyết định
```

*Reflector (H-GRD-003):* `getAllAndOverride` lấy giá trị **ưu tiên theo cấp** (method > class); `getAllAndMerge` **gộp** nhiều cấp. ⚠️ **Nest 11 (đã verify):** `getAllAndOverride` trả `T | undefined` (phản ánh khả năng không có metadata) và `getAllAndMerge` trả về **object** thay vì mảng 1 phần tử khi chỉ có một metadata kiểu object — code cũ giả định luôn có giá trị có thể vỡ type. *(verify khi nâng cấp.)*

*Authn vs Authz (H-GRD-004):* **authentication** (xác thực — *bạn là ai*, gắn `req.user`) chạy **trước** **authorization** (phân quyền — *bạn được làm gì*). Thường `AuthGuard` (passport) chạy trước `RolesGuard`. Đừng trộn hai việc vào một guard mơ hồ.

**(c) Mép giới hạn & sai lầm.**
- *Không nhét authz vào handler (H-GRD-005):* để tách concern, tái dùng, test riêng, và khai báo bằng metadata; handler chỉ lo nghiệp vụ.
- *RBAC không đủ (H-GRD-006, ➕):* khi quyền phụ thuộc **tài nguyên/thuộc tính** (vd "chủ sở hữu mới sửa được bài viết X") thì role-only thất bại → lên **ABAC/policy-based** (CASL, OPA): kiểm theo bộ ba **(user, action, resource)**. Nhận ra giới hạn RBAC là dấu hiệu tư duy TL.
- Quên rằng `canActivate` trả `false` cho 403 *mặc định* — nếu muốn 401/thông điệp khác phải tự throw đúng exception.

## ③ ⚠️ Kiến thức cũ / bị thay thế

| Cái cũ | Thay bằng (hiện nay) | Vì sao |
|---|---|---|
| Check `if (user.role !== 'admin')` trong handler | `@Roles` + RolesGuard + Reflector | Khai báo, tái dùng, test riêng |
| Giả định `reflector.getAllAndOverride` luôn có giá trị | Xử lý `T \| undefined` (Nest 11) | Type mới phản ánh "không có metadata" |
| RBAC cho mọi bài toán quyền | **ABAC/policy** (CASL/OPA) khi quyền theo resource | RBAC không biểu diễn được quyền theo thuộc tính |

## ④ Áp dụng thực tế + So sánh bigtech
Admin panel: `@Roles('admin')` trên route xóa user. Nhưng "user chỉ sửa post *của mình*" → RBAC bó tay, dùng CASL: `ability.can('update', subject('Post', post))`. Bigtech quy mô lớn thường tách hẳn **policy engine** (OPA/Cedar) khỏi app để quản quyền tập trung — Nest guard chỉ là điểm *gọi* policy. *(Pattern phổ biến; chi tiết engine thì verify theo tài liệu từng hãng.)*

## ⑤ Code thực hành + cấu hình
```typescript
// roles.decorator.ts
import { SetMetadata } from '@nestjs/common';
export const ROLES_KEY = 'roles';
export const Roles = (...roles: string[]) => SetMetadata(ROLES_KEY, roles);
```
```typescript
// roles.guard.ts
@Injectable()
export class RolesGuard implements CanActivate {
  constructor(private reflector: Reflector) {}
  canActivate(ctx: ExecutionContext): boolean {
    const required = this.reflector.getAllAndOverride<string[] | undefined>( // Nest 11: T | undefined
      ROLES_KEY, [ctx.getHandler(), ctx.getClass()]);
    if (!required?.length) return true;                 // route không yêu cầu role
    const { user } = ctx.switchToHttp().getRequest();
    return required.some(r => user?.roles?.includes(r));// thiếu quyền → false → 403
  }
}
```
```typescript
// dùng kèm AuthGuard (authn) trước RolesGuard (authz)
@UseGuards(AuthGuard('jwt'), RolesGuard)
@Roles('admin')
@Delete(':id') remove() { /* ... */ }
```

## ⑥ Keywords cần nhớ
- 🧠 **Cần HIỂU:** authn vs authz · vì sao authz ở guard không ở handler · giới hạn RBAC → ABAC.
- 📌 **Cần THUỘC:** `canActivate` trả boolean/Promise/Observable · `SetMetadata`+`Reflector` · `getAllAndOverride` (Nest 11: `T | undefined`) vs `getAllAndMerge`.
- 🛠️ **Cần LÀM ĐƯỢC:** dựng RBAC guard hoàn chỉnh; phác 1 policy-based check cho quyền theo resource.

## ⑦ Mental model
> **"Authn gắn *bạn là ai*, authz phán *bạn được làm gì*. Role không đủ → chuyển sang (user, action, resource)."**

## ⑧ Câu hỏi phỏng vấn & thách đố
1. *(TB)* `canActivate` trả gì, quyết định gì? **(H-GRD-001)**
2. *(TB)* Luồng `@Roles`→Reflector→RolesGuard. **(H-GRD-002)**
3. *(khó)* `getAllAndOverride` vs `getAllAndMerge`; Nest 11 đổi gì? **(H-GRD-003)**
4. *(khó)* Authn vs authz đặt ở đâu, AuthGuard vs RolesGuard phối hợp sao? **(H-GRD-004)**
5. *(rất khó)* RBAC không đủ thì mở sang ABAC/policy thế nào? **(H-GRD-006)**

**🎯 Thách đố:** *"RolesGuard của bạn chặn cả route công khai không gắn `@Roles`. Bug ở đâu?"*
> Đáp: không xử lý case metadata `undefined` → coi như "không có role hợp lệ" → chặn. Phải `if (!required) return true;` cho route không yêu cầu role (đặc biệt sau khi Nest 11 trả `T | undefined`).

**🤖 "Hiểu để chỉ huy":** AI sinh RolesGuard "đủ" nhưng hay quên nhánh `undefined` → khóa nhầm route public. Người hiểu test cả route có và không có `@Roles`.

## ⑨ Bài tập + tiêu chí tự chấm
**BT1:** RBAC guard + `@Roles`. **Đạt khi:** admin qua, user thường nhận 403, route không `@Roles` vẫn mở.
**BT2:** Thêm 1 check ownership đơn giản (user chỉ sửa resource có `ownerId === user.id`). **Đạt khi:** đúng/sai owner cho kết quả đúng — nhận ra đây là ABAC sơ khai.

## ⑩ Đọc thêm
- docs.nestjs.com → *Guards*; *Security → Authentication/Authorization*; trang CASL chính thức. (Liên hệ mục M roadmap.)

---

# Bài 7 — Interceptors & Exception filters: RxJS, envelope, error mapping
> Phủ: **H-INT-001 → H-INT-006**

## ① Mục tiêu & vị trí trong mạch
Interceptor là chặng *bọc cả trước & sau handler* (Bài 4); exception filter là *đội xử lý sự cố* cuối luồng. Bài này dạy can thiệp response stream bằng RxJS, chuẩn hóa envelope `{data}`, và biến mọi lỗi thành một format thống nhất. Đây là nơi Nest "khác" Express rõ nhất.

## ② Giảng cơ bản → nâng cao

**(a) Trực giác.** Interceptor như **lớp bọc quà**: nó cầm gói hàng *trước* khi giao (thêm logging/timeout) và *sau* khi nhận lại (gói lại response, đổi format). Exception filter như **bộ phận khiếu nại**: bất kỳ ai ném lỗi đều bị nó đón và trả lời theo mẫu chuẩn.

**(b) Cơ chế chi tiết (📘).**

*Interceptor làm được gì (H-INT-001):* bọc cả **trước + sau** handler — transform response, logging, caching, timeout, mapping error — thứ guard/pipe không làm được (chúng chỉ ở một phía).

*`intercept(context, next)` + RxJS (H-INT-002):* `next.handle()` trả **Observable** của response stream. Bạn dùng pipe RxJS để xử lý *sau* handler: `map` (đổi response), `tap` (side-effect như log), `catchError` (bắt lỗi), `timeout` (giới hạn thời gian). Nest dùng RxJS ở đây để biểu diễn luồng "trước→sau" như một stream có thể compose.

*Response envelope (H-INT-003):* gói mọi response thành `{data, meta}` nên đặt ở **global interceptor** + `map` → tập trung, không lặp ở từng handler.

*Exception filter (H-INT-004):* `@Catch(SomeException)` (hoặc `@Catch()` bắt tất) + implement `ExceptionFilter.catch(exception, host)` → format error response thống nhất, log. Phân biệt `HttpException` (đã có status) vs lỗi không lường.

*HttpException vs lỗi unknown (H-INT-005):* `HttpException`/built-in (`NotFoundException`, `BadRequestException`...) **map sẵn status code**. Lỗi *không phải* HttpException → mặc định **500**, và **cần filter** để không lộ stack trace ra client.

*Thứ tự & quan hệ với filter (H-INT-006):* nhiều interceptor lồng nhau như **onion** — chạy theo thứ tự đăng ký ở chiều vào, ngược lại ở chiều ra. Khi lỗi xảy ra, luồng rẽ sang filter; nhưng `catchError` trong một interceptor **có thể chặn lỗi trước khi tới filter** (bắt và map lại) — hiểu điểm này để biết "ai bắt lỗi trước".

**(c) Mép giới hạn & sai lầm.**
- Dùng interceptor để *validate* (sai vai trò — đó là pipe) hoặc *authz* (đó là guard).
- Quên rằng interceptor post **không chạy bình thường khi handler ném lỗi** (Bài 4) trừ khi bắt bằng `catchError`.
- Filter `@Catch()` bắt tất nhưng vô tình "nuốt" cả HttpException và đổi status → mất ngữ nghĩa lỗi.

## ③ ⚠️ Kiến thức cũ / bị thay thế

| Cái cũ | Thay bằng (hiện nay) | Vì sao |
|---|---|---|
| `try/catch` + `res.status().json()` rải khắp handler | **Exception filter** tập trung | 1 format lỗi, không lặp |
| Gói `{data}` thủ công ở từng controller | **Global interceptor** + `map` | DRY, nhất quán |
| Lộ stack trace lỗi 500 ra client | Filter chuẩn hóa + log nội bộ | Bảo mật, không rò chi tiết |

## ④ Áp dụng thực tế + So sánh bigtech
Mọi API trả `{data, meta:{requestId}}` (interceptor) và mọi lỗi trả `{error:{code,message}}` + status đúng (filter), kèm log có `requestId` để trace. Đây là "API contract" chuẩn của các team lớn — client chỉ phải xử lý *một* hình dạng success và *một* hình dạng error. Kèm `timeout` interceptor để cắt request treo (chống tê liệt do downstream chậm).

## ⑤ Code thực hành + cấu hình
```typescript
// transform.interceptor.ts — envelope {data} toàn app
import { CallHandler, ExecutionContext, Injectable, NestInterceptor } from '@nestjs/common';
import { map, timeout } from 'rxjs/operators';
@Injectable()
export class TransformInterceptor implements NestInterceptor {
  intercept(_ctx: ExecutionContext, next: CallHandler) {
    return next.handle().pipe(
      timeout(10_000),                       // cắt request treo > 10s
      map(data => ({ data })),               // chiều RA: gói envelope
    );
  }
}
```
```typescript
// all-exceptions.filter.ts — chuẩn hóa lỗi, không lộ stack
import { ArgumentsHost, Catch, ExceptionFilter, HttpException, HttpStatus } from '@nestjs/common';
@Catch()
export class AllExceptionsFilter implements ExceptionFilter {
  catch(ex: unknown, host: ArgumentsHost) {
    const res = host.switchToHttp().getResponse();
    const status = ex instanceof HttpException ? ex.getStatus() : HttpStatus.INTERNAL_SERVER_ERROR;
    const message = ex instanceof HttpException ? ex.getResponse() : 'Internal server error';
    // log chi tiết NỘI BỘ (không trả ra client)
    res.status(status).json({ error: { status, message } });
  }
}
```
```typescript
// đăng ký global qua DI (inject được nếu cần)
@Module({ providers: [
  { provide: APP_INTERCEPTOR, useClass: TransformInterceptor },
  { provide: APP_FILTER, useClass: AllExceptionsFilter },
]})
export class AppModule {}
```

## ⑥ Keywords cần nhớ
- 🧠 **Cần HIỂU:** vì sao interceptor bọc 2 đầu mà guard/pipe không · onion nesting · catchError chặn trước filter · vì sao không lộ stack.
- 📌 **Cần THUỘC:** `intercept(ctx, next)` + `next.handle()` Observable · `map/tap/catchError/timeout` · `@Catch()` + `ExceptionFilter.catch` · `HttpException`.
- 🛠️ **Cần LÀM ĐƯỢC:** viết 1 interceptor envelope + timeout, 1 exception filter bắt-tất an toàn.

## ⑦ Mental model
> **"Interceptor = lớp onion bọc trước/sau (RxJS). Filter = mẫu trả lời chuẩn cho mọi lỗi. HttpException có status, lỗi lạ = 500 + đừng lộ stack."**

## ⑧ Câu hỏi phỏng vấn & thách đố
1. *(TB)* Interceptor làm gì mà guard/pipe không? **(H-INT-001)**
2. *(khó)* `next.handle()` Observable hoạt động sao, vì sao RxJS? **(H-INT-002)**
3. *(TB)* Exception filter chuẩn hóa lỗi thế nào? **(H-INT-004)**
4. *(khó)* HttpException vs lỗi JS thường; lỗi unknown bị xử lý sao? **(H-INT-005)**
5. *(rất khó)* Thứ tự nhiều interceptor + quan hệ với filter khi lỗi? **(H-INT-006)**

**🎯 Thách đố:** *"TransformInterceptor gói mọi response thành `{data}`, nhưng response lỗi 400 lại KHÔNG bị gói. Vì sao đúng/sai?"*
> Đáp: khi handler ném lỗi, nhánh `map` (post) không chạy (luồng rẽ sang filter) → lỗi không bị gói envelope success. Đó thực ra *đúng* (lỗi nên theo format error riêng). Nếu muốn cả lỗi cũng có format chung → xử lý ở filter, không ở interceptor map.

**🤖 "Hiểu để chỉ huy":** AI có thể đặt logic validate/authz vào interceptor vì "nó chạy trước handler" — sai vai trò. Người hiểu giữ validate ở pipe, authz ở guard, interceptor cho cross-cutting response.

## ⑨ Bài tập + tiêu chí tự chấm
**BT1:** Interceptor thêm `meta.durationMs` (đo thời gian xử lý qua `tap`/`Date.now`). **Đạt khi:** response có duration đúng.
**BT2:** Filter map lỗi không-HttpException thành 500 generic, log stack nội bộ. **Đạt khi:** client *không* thấy stack, log server *có*.

## ⑩ Đọc thêm
- docs.nestjs.com → *Interceptors*, *Exception filters*. RxJS docs (operators `map/tap/catchError/timeout`).

---

# Bài 8 — Custom decorators: param decorator, SetMetadata, applyDecorators, reflection
> Phủ: **H-DEC-001 → H-DEC-005**

## ① Mục tiêu & vị trí trong mạch
Decorator là **chất keo** nối các chặng lifecycle: nó *gắn* dữ liệu (metadata) để guard/interceptor *đọc*, và *rút gọn* truy cập request. Bài này dạy tạo `@CurrentUser()`, `@Roles()`, gộp decorator, và hiểu reflection bên dưới — nền để "kiểm tra AI" khi nó sinh decorator.

## ② Giảng cơ bản → nâng cao

**(a) Trực giác.** Decorator như **nhãn dán** lên hộp (handler/class). Có nhãn *mang dữ liệu* để người khác đọc (`@Roles('admin')` → guard đọc), và nhãn *rút gọn thao tác* (`@CurrentUser()` thay cho moi `req.user` thủ công).

**(b) Cơ chế chi tiết (📘).**

*Param decorator (H-DEC-001):* `createParamDecorator((data, ctx: ExecutionContext) => ...)` — lấy dữ liệu từ `ctx` (thường `ctx.switchToHttp().getRequest()`) trả về giá trị inject vào tham số handler. Thay cho `@Req()` rồi tự bóc.

*SetMetadata + Reflector (H-DEC-002):* `SetMetadata(key, value)` gắn cặp key–value lên handler/class; `Reflector` đọc lại trong guard/interceptor. `@Roles` chính là wrapper của `SetMetadata` (Bài 6).

*`applyDecorators` (H-DEC-003):* **compose** nhiều decorator thành một cho DRY. Vd `@Auth(...roles)` = `applyDecorators(UseGuards(AuthGuard, RolesGuard), Roles(...roles), ApiBearerAuth())` → 1 dòng trên handler thay vì 3.

*Cơ chế reflection (H-DEC-004):* Nest dựa trên **metadata reflection** (`reflect-metadata` + `emitDecoratorMetadata` của TS) để đọc type/metadata lúc runtime → dựng DI và route. ⚠️ Lưu ý: TS đang dịch chuyển giữa **decorator legacy** (experimentalDecorators) và **decorator stage-3** (chuẩn ECMAScript) — Nest hiện dựa trên legacy + reflect-metadata; cần *verify* khi migrate TS/decorator mới.

**(c) Mép giới hạn & sai lầm (H-DEC-005 — "hiểu để chỉ huy").**
Một custom decorator có thể *chạy được* nhưng *phá DI/scope*. Khi AI sinh decorator, phải kiểm:
- **Sai context theo transport:** dùng `ctx.switchToHttp()` cứng → vỡ khi chạy trên RPC/WS. Decorator dùng chung nhiều transport phải nhận biết `ctx.getType()`.
- **Không tương thích scoped provider:** decorator giả định singleton có thể sai trong REQUEST scope.
- **Trùng metadata key:** key string tùy tiện va với key khác → đọc nhầm.

## ③ ⚠️ Kiến thức cũ / bị thay thế

| Cái cũ | Thay bằng (hiện nay) | Vì sao |
|---|---|---|
| `@Req() req` rồi `req.user` thủ công khắp nơi | `@CurrentUser()` param decorator | Gọn, tái dùng, dễ test |
| Lặp `@UseGuards()+@Roles()+@ApiBearerAuth()` mọi route | `@Auth()` qua `applyDecorators` | DRY, đọc gọn |
| Metadata key string rải rác | Hằng key export tập trung (`ROLES_KEY`) | Tránh trùng/typo |

## ④ Áp dụng thực tế + So sánh bigtech
`@CurrentUser()` + `@Auth('admin')` xuất hiện trên hầu hết controller của một app thật → giảm boilerplate, tăng nhất quán. Pattern "decorator = API khai báo cho cross-cutting" giống Angular/Spring annotations — team lớn dùng để route/security/docs (Swagger) đọc cùng một nguồn metadata.

## ⑤ Code thực hành + cấu hình
```typescript
// current-user.decorator.ts
import { createParamDecorator, ExecutionContext } from '@nestjs/common';
export const CurrentUser = createParamDecorator(
  (data: string | undefined, ctx: ExecutionContext) => {
    const req = ctx.switchToHttp().getRequest();   // ⚠️ chỉ đúng cho HTTP
    return data ? req.user?.[data] : req.user;      // @CurrentUser('id') lấy field
  },
);
```
```typescript
// auth.decorator.ts — gộp nhiều decorator
import { applyDecorators, UseGuards } from '@nestjs/common';
import { Roles } from './roles.decorator';
export function Auth(...roles: string[]) {
  return applyDecorators(
    UseGuards(AuthGuard('jwt'), RolesGuard),
    Roles(...roles),
  );
}
// dùng: @Auth('admin') @Delete(':id') remove(@CurrentUser('id') uid: string) {}
```

## ⑥ Keywords cần nhớ
- 🧠 **Cần HIỂU:** metadata-mang-dữ-liệu vs decorator-rút-gọn · reflection runtime · vì sao decorator có thể phá scope/transport.
- 📌 **Cần THUỘC:** `createParamDecorator((data, ctx)=>...)` · `SetMetadata` · `applyDecorators` · `reflect-metadata`/`emitDecoratorMetadata`.
- 🛠️ **Cần LÀM ĐƯỢC:** viết `@CurrentUser()` + 1 decorator gộp `@Auth()`.

## ⑦ Mental model
> **"Decorator gắn nhãn để guard/interceptor đọc, hoặc rút gọn truy cập request. Nest 'thấy' nhãn nhờ reflect-metadata."**

## ⑧ Câu hỏi phỏng vấn & thách đố
1. *(TB)* `@CurrentUser()` tạo bằng gì, lấy data từ đâu? **(H-DEC-001)**
2. *(TB)* `SetMetadata` cặp với Reflector ra sao? **(H-DEC-002)**
3. *(khó)* `applyDecorators` để làm gì, ví dụ? **(H-DEC-003)**
4. *(khó)* Decorator dựa cơ chế TS/JS nào? **(H-DEC-004)**
5. *(rất khó)* AI sinh decorator chạy được — kiểm gì để chắc không phá DI/scope? **(H-DEC-005)**

**🎯 Thách đố:** *"`@CurrentUser()` chạy ngon ở REST nhưng crash khi bạn tái dùng trong một microservice handler. Vì sao?"*
> Đáp: nó gọi `ctx.switchToHttp()` cứng; ở RPC context không có HTTP request. Fix: nhánh theo `ctx.getType()` (`'http' | 'rpc' | 'ws'`) hoặc tạo decorator riêng cho transport.

**🤖 "Hiểu để chỉ huy":** AI sinh param decorator gần như luôn `switchToHttp()` — đúng cho REST, sai khi dùng đa transport. Người hiểu kiểm `ctx.getType()` trước khi tin.

## ⑨ Bài tập + tiêu chí tự chấm
**BT1:** `@CurrentUser('email')` trả đúng email user. **Đạt khi:** handler nhận đúng giá trị, không dùng `@Req()`.
**BT2:** `@Auth('admin')` gộp guard+roles. **Đạt khi:** 1 dòng decorator thay được 2–3 dòng cũ, hành vi y hệt.

## ⑩ Đọc thêm
- docs.nestjs.com → *Custom decorators*, *Custom route decorators*; trang `reflect-metadata`. Theo dõi tình trạng TS decorators stage-3 *(verify)*.

---

# Bài 9 — Dynamic modules & Config: forRoot(Async), forFeature, ConfigModule, registerAsync
> Phủ: **H-DYN-001 → H-DYN-007**

## ① Mục tiêu & vị trí trong mạch
Đến giờ module của bạn là *tĩnh*. Bài này dạy module **cấu hình được lúc import** — nền của mọi thư viện Nest (`@nestjs/config`, JwtModule, TypeOrmModule...). Dùng đúng DI (Bài 1–2) + module (Bài 3). Là kỹ năng để viết "thư viện nội bộ" tái dùng giữa nhiều service.

## ② Giảng cơ bản → nâng cao

**(a) Trực giác.** Static module = **đồ nội thất lắp sẵn**. Dynamic module = **đồ IKEA**: cùng một bộ khung, nhưng bạn *truyền tùy chọn* (màu, kích thước) lúc lắp (`forRoot(options)`). `forRootAsync` = khi tùy chọn phải *đi hỏi nơi khác* trước (vd hỏi `ConfigService` lấy URL DB) mới biết lắp thế nào.

**(b) Cơ chế chi tiết (📘).**

*Dynamic module (H-DYN-001):* một static method (`forRoot`) trả về object `DynamicModule` (`{ module, providers, exports, imports }`) được **cấu hình theo options** truyền vào lúc import. Khác static (cấu hình cứng trong `@Module`).

*forRoot vs forRootAsync (H-DYN-002):* `forRoot(opts)` khi options **tĩnh/biết trước**. `forRootAsync({ useFactory, inject })` khi options **phụ thuộc thứ khác** (vd `ConfigService`) → factory nhận inject, có thể async.

*forFeature (H-DYN-003):* `forRoot` cấu hình **toàn cục một lần** (vd connection DB); `forFeature` đăng ký **theo feature** ở module con (vd entity/repo của riêng `UsersModule`). Pattern này thấy ở TypeOrmModule/MongooseModule.

*ConfigModule (H-DYN-004):* `@nestjs/config` load `.env`, (tùy chọn) validate, cung cấp `ConfigService` có type để inject. Vì sao tốt hơn `process.env` rải rác: tập trung, type, test được, tránh `undefined` ngầm.

*Validate env fail-fast (H-DYN-005):* dùng Joi/Zod schema trong ConfigModule để **phát hiện thiếu/sai env ngay lúc boot** thay vì lỗi runtime mơ hồ giữa chừng (vd thiếu `DATABASE_URL` thì app *không khởi động* chứ không chạy rồi sập lúc query).

*registerAsync (H-DYN-006):* `JwtModule.registerAsync({ useFactory, inject:[ConfigService] })` — factory nhận inject trả options async. Bản chất = dynamic module + DI: cho phép tiêm dependency vào *cấu hình* của module.

**(c) Mép giới hạn & sai lầm (H-DYN-007 — viết module tái dùng).**
Thiết kế `MailModule.forRoot(options)` dùng cho nhiều dự án:
- Định nghĩa **options interface** rõ ràng + **injection token** cho options.
- Hỗ trợ **cả `forRoot` và `forRootAsync`** (sync cho config tĩnh, async cho config động).
- `exports` service để nơi import dùng được.
- Sai thường gặp: hardcode options trong providers (không tái cấu hình được); quên `forRootAsync` (khóa người dùng vào config tĩnh).

## ③ ⚠️ Kiến thức cũ / bị thay thế

| Cái cũ | Thay bằng (hiện nay) | Vì sao |
|---|---|---|
| `process.env.X` rải rác, ép kiểu tay | `ConfigModule` + `ConfigService` (typed) | Tập trung, test được, không undefined ngầm |
| Không validate env, lỗi runtime mơ hồ | Joi/Zod schema **fail-fast** lúc boot | Sai env thì *không* khởi động |
| Module chỉ `forRoot` (config tĩnh) | Hỗ trợ cả `forRootAsync` | Config phụ thuộc ConfigService/secret |

## ④ Áp dụng thực tế + So sánh bigtech
`ConfigModule.forRoot({ isGlobal:true, validationSchema })` + `TypeOrmModule.forRootAsync({ useFactory: c => ({ url: c.get('DB_URL') }), inject:[ConfigService] })` + `JwtModule.registerAsync(...)`. Đây là "bộ khởi động" gần như mọi app Nest production. Team lớn còn đóng gói các concern chung (mail, audit, feature-flag) thành **dynamic module nội bộ** publish lên private registry — đúng tinh thần H-DYN-007.

## ⑤ Code thực hành + cấu hình
```typescript
// config: load + validate fail-fast
import * as Joi from 'joi';
@Module({ imports: [ ConfigModule.forRoot({
  isGlobal: true,
  validationSchema: Joi.object({
    DATABASE_URL: Joi.string().uri().required(),   // thiếu → app KHÔNG boot
    JWT_SECRET: Joi.string().min(16).required(),
  }),
})]})
export class AppModule {}
```
```typescript
// forRootAsync inject ConfigService
TypeOrmModule.forRootAsync({
  inject: [ConfigService],
  useFactory: (c: ConfigService) => ({
    type: 'postgres', url: c.get<string>('DATABASE_URL'), autoLoadEntities: true,
  }),
}); // verify API TypeORM/Nest theo bản
```
```typescript
// dynamic module tái dùng: MailModule.forRoot / forRootAsync
export const MAIL_OPTIONS = Symbol('MAIL_OPTIONS');
@Module({})
export class MailModule {
  static forRoot(opts: MailOptions): DynamicModule {
    return { module: MailModule,
      providers: [{ provide: MAIL_OPTIONS, useValue: opts }, MailService],
      exports: [MailService] };
  }
  static forRootAsync(o: { useFactory: (...a:any[])=>MailOptions|Promise<MailOptions>; inject?: any[] }): DynamicModule {
    return { module: MailModule,
      providers: [{ provide: MAIL_OPTIONS, useFactory: o.useFactory, inject: o.inject ?? [] }, MailService],
      exports: [MailService] };
  }
}
```

## ⑥ Keywords cần nhớ
- 🧠 **Cần HIỂU:** dynamic = config-lúc-import · vì sao async khi options phụ thuộc · fail-fast env.
- 📌 **Cần THUỘC:** `forRoot/forRootAsync/forFeature/registerAsync` · `DynamicModule` shape · options token + `useValue/useFactory`.
- 🛠️ **Cần LÀM ĐƯỢC:** viết 1 dynamic module hỗ trợ cả forRoot & forRootAsync; cấu hình ConfigModule validate.

## ⑦ Mental model
> **"Dynamic module = đồ IKEA: truyền options lúc lắp. Options phụ thuộc thứ khác → forRootAsync (inject factory). Env sai → fail-fast lúc boot."**

## ⑧ Câu hỏi phỏng vấn & thách đố
1. *(khó)* Dynamic module là gì, `forRoot` trả gì? **(H-DYN-001)**
2. *(khó)* forRoot vs forRootAsync, khi nào buộc async? **(H-DYN-002)**
3. *(TB)* forFeature vs forRoot? **(H-DYN-003)**
4. *(TB)* ConfigModule hơn `process.env` chỗ nào? **(H-DYN-004)**
5. *(rất khó)* Thiết kế MailModule.forRoot tái dùng: options+token+async? **(H-DYN-007)**

**🎯 Thách đố:** *"JwtModule của bạn cần `secret` từ ConfigService, nhưng bạn viết `JwtModule.register({ secret: process.env.JWT_SECRET })` và thỉnh thoảng secret là `undefined`. Vì sao và fix sao?"*
> Đáp: `register` (sync) đọc `process.env` *ngay khi import*, có thể trước khi env nạp/validate xong, và bỏ qua ConfigService. Fix: `registerAsync({ useFactory: c => ({ secret: c.get('JWT_SECRET') }), inject:[ConfigService] })`.

**🤖 "Hiểu để chỉ huy":** AI hay chọn `register({ secret: process.env... })` cho nhanh — chạy ở demo nhưng giòn (thứ tự nạp env, không validate). Người hiểu chỉ huy dùng `registerAsync` + ConfigModule fail-fast.

## ⑨ Bài tập + tiêu chí tự chấm
**BT1:** ConfigModule validate `PORT` (number, required). **Đạt khi:** xóa PORT khỏi .env → app báo lỗi *lúc boot*.
**BT2:** Viết `CacheModule.forRoot(opts)` + `forRootAsync`. **Đạt khi:** cùng module dùng được cả hai kiểu, service được export & inject.

## ⑩ Đọc thêm
- docs.nestjs.com → *Fundamentals → Dynamic modules*; *Techniques → Configuration*. Trang `@nestjs/config`, Joi/Zod.

---

# Bài 10 — Microservices: transport, message/event pattern, hybrid, gRPC, RpcException
> Phủ: **H-MS-001 → H-MS-008**

## ① Mục tiêu & vị trí trong mạch
`@nestjs/microservices` mở rộng lifecycle (Bài 4) sang transport *không phải HTTP*. Cùng controller/DI/guard/pipe nhưng "đầu vào" là message qua TCP/Redis/Kafka/gRPC... Bài này dạy chọn transport, request-response vs fire-and-forget, và quan trọng nhất (TL): **khi nào KHÔNG nên tách microservice**.

## ② Giảng cơ bản → nâng cao

**(a) Trực giác.** HTTP controller = **quầy tiếp khách trực tiếp**. Microservice = cùng nhân viên đó nhưng nhận yêu cầu qua **đường dây nội bộ** (TCP), **hộp thư** (queue: RabbitMQ), hay **bảng tin phát thanh** (Kafka stream). Nest giữ nguyên cách bạn viết handler, chỉ đổi "đường" message tới.

**(b) Cơ chế chi tiết (📘).**

*Transport (H-MS-001):* TCP, Redis, NATS, RabbitMQ, Kafka, gRPC, MQTT. Chọn theo nhu cầu: **queue** (RabbitMQ) cho công việc bền/ack; **streaming/log** (Kafka) cho event lớn, replay; **RPC** (gRPC/TCP) cho request-response hiệu năng.

*Message vs Event pattern (H-MS-002):* `@MessagePattern` = **request-response** (RPC-like, *chờ* phản hồi). `@EventPattern` = **fire-and-forget** (phát sự kiện, *không chờ*). Chọn theo coupling: cần kết quả → message; chỉ thông báo "đã xảy ra" → event.

*Hybrid application (H-MS-003):* một process vừa HTTP vừa microservice: `app.connectMicroservice(...)` + `app.startAllMicroservices()` + `app.listen(port)`. Hữu ích khi service vừa phục vụ REST vừa nghe queue.

*Khi nào gRPC (H-MS-004):* contract Protobuf chặt, hiệu năng + streaming, đa ngôn ngữ. Trade-off: khó debug bằng mắt hơn REST (binary), cần build proto. (Liên hệ mục F API design + K messaging roadmap.)

*Kafka/RabbitMQ — Nest lo tới đâu (H-MS-005):* Nest abstraction phần **bind pattern + (de)serialize message**. Phần **ack, at-least-once, idempotent consumer, DLQ, consumer group/queue config** vẫn là **trách nhiệm thiết kế của bạn**. *(verify option cụ thể bản v11.)*

*ClientProxy + unwrap (H-MS-006):* `ClientProxy.send()` (message, trả Observable kết quả) / `.emit()` (event). **Nest 11**: `unwrap()` cho phép lấy **native client** (vd Kafka/AMQP client gốc) để thao tác nâng cao ngoài API chuẩn. *(verify)*

*RpcException (H-MS-007):* lỗi trong microservice dùng `RpcException` + filter riêng cho RPC context; timeout xử lý phía client. **Không** trả HTTP status như REST — ngữ nghĩa lỗi khác.

**(c) Mép giới hạn & sai lầm (H-MS-008 — câu TL quan trọng nhất).**
**Khi nào KHÔNG tách microservice dù Nest hỗ trợ sẵn:**
- Team/scale nhỏ, một **monolith module hóa tốt** (Bài 3) đáp ứng đủ.
- Tách sớm → gánh **chi phí vận hành** (deploy/observability nhiều service), **độ trễ mạng**, **distributed transaction** (mất ACID, phải saga). Tư duy TL: tách vì *nhu cầu thật* (scale độc lập, team độc lập, fault isolation), **không** vì "Nest làm được".

## ③ ⚠️ Kiến thức cũ / bị thay thế

| Cái cũ | Thay bằng (hiện nay) | Vì sao |
|---|---|---|
| Tách microservice sớm "cho hiện đại" | **Modular monolith** trước, tách khi cần | Tránh chi phí phân tán không cần thiết |
| Dùng `@MessagePattern` cho mọi giao tiếp | Phân biệt message (chờ) vs **event** (không chờ) | Đúng coupling, tránh chờ vô ích |
| Tin Nest lo hết reliability của queue | Tự lo ack/idempotent/DLQ | Nest chỉ abstraction bind+serialize |

## ④ Áp dụng thực tế + So sánh bigtech
`OrderService` (HTTP) phát event `order.created` lên Kafka (`emit`, fire-and-forget) → `EmailService` + `InventoryService` consume độc lập. Tra cứu giá vận chuyển thì `send` (message, cần kết quả) qua gRPC tới `ShippingService`. Bigtech điển hình: Kafka cho event backbone, gRPC cho RPC nội bộ service-to-service; REST/GraphQL ở edge. *(Pattern phổ biến; chi tiết hạ tầng từng hãng thì verify.)*

## ⑤ Code thực hành + cấu hình
```typescript
// microservice consumer
import { Controller } from '@nestjs/common';
import { MessagePattern, EventPattern, Payload } from '@nestjs/microservices';
@Controller()
export class OrdersMsController {
  @MessagePattern({ cmd: 'get_order' })           // request-response
  getOrder(@Payload() id: string) { return { id, status: 'PAID' }; }

  @EventPattern('order.created')                  // fire-and-forget
  onCreated(@Payload() evt: { id: string }) { /* side-effect, không return */ }
}
```
```typescript
// hybrid app: HTTP + microservice trong 1 process
const app = await NestFactory.create(AppModule);
app.connectMicroservice({ transport: Transport.KAFKA, options: { /* brokers... */ } });
await app.startAllMicroservices();
await app.listen(3000);   // verify option transport theo bản
```
```typescript
// client gửi message / phát event
constructor(@Inject('ORDERS') private client: ClientProxy) {}
this.client.send({ cmd: 'get_order' }, '123');   // Observable<kết quả>
this.client.emit('order.created', { id: '123' }); // không chờ
```

## ⑥ Keywords cần nhớ
- 🧠 **Cần HIỂU:** message (chờ) vs event (không chờ) · Nest lo gì / bạn lo gì ở queue · vì sao đừng tách microservice sớm.
- 📌 **Cần THUỘC:** transport list · `@MessagePattern/@EventPattern` · `connectMicroservice`+`startAllMicroservices` · `ClientProxy.send/emit` · `RpcException` · `unwrap()` (v11, *verify*).
- 🛠️ **Cần LÀM ĐƯỢC:** dựng 1 hybrid app; phân biệt đúng send vs emit cho 1 tình huống.

## ⑦ Mental model
> **"Cùng handler, đổi 'đường' message. send = chờ kết quả, emit = phát rồi quên. Nest lo bind+serialize; reliability (ack/DLQ/idempotent) là việc của bạn. Tách microservice vì nhu cầu, không vì framework cho phép."**

## ⑧ Câu hỏi phỏng vấn & thách đố
1. *(TB)* Các transport `@nestjs/microservices` hỗ trợ? **(H-MS-001)**
2. *(khó)* MessagePattern vs EventPattern? **(H-MS-002)**
3. *(khó)* Hybrid app là gì, dựng sao? **(H-MS-003)**
4. *(khó)* Kafka/RabbitMQ: Nest lo tới đâu, bạn lo gì? **(H-MS-005)**
5. *(rất khó)* Khi nào KHÔNG nên tách microservice? **(H-MS-008)**

**🎯 Thách đố:** *"Bạn dùng `@EventPattern('payment.process')` để xử lý thanh toán và thấy đôi khi một thanh toán bị xử lý 2 lần. Hai vấn đề thiết kế?"*
> Đáp: (1) thanh toán *cần kết quả/độ tin cậy* → có lẽ nên là message/command có ack, không phải event fire-and-forget; (2) thiếu **idempotent consumer** (at-least-once delivery → trùng) — cần idempotency key. Nest không tự lo phần này.

**🤖 "Hiểu để chỉ huy":** AI sẵn sàng "chẻ" app thành 5 microservice vì Nest hỗ trợ — bẫy over-engineering. Người hiểu hỏi "scale/team/fault có *thật* cần tách không?" trước khi gật.

## ⑨ Bài tập + tiêu chí tự chấm
**BT1:** Hybrid app: REST `POST /orders` → `emit('order.created')`; consumer log event. **Đạt khi:** một process phục vụ cả hai, event tới consumer.
**BT2:** Thêm idempotency cho consumer (lưu processed id). **Đạt khi:** gửi trùng event không xử lý 2 lần.

## ⑩ Đọc thêm
- docs.nestjs.com → *Microservices → Overview/Basics/Kafka/gRPC/Exception filters*. (Liên hệ mục K messaging roadmap.)

---

# Bài 11 — CQRS & Event Sourcing: command/query/event bus, saga, projection
> Phủ: **H-CQRS-001 → H-CQRS-006**

## ① Mục tiêu & vị trí trong mạch
`@nestjs/cqrs` là một *kiến trúc trong-app* dựng trên DI + module + bus. Bài này dạy tách đọc/ghi, decouple qua bus, và khái niệm Event Sourcing — kèm cảnh báo over-engineering. Đây là bài "cao" — chỉ đáng dùng khi domain đủ phức tạp.

## ② Giảng cơ bản → nâng cao

**(a) Trực giác.** CQRS như **tách quầy "đặt hàng" khỏi quầy "tra cứu"** trong ngân hàng: lệnh thay đổi tiền (command) đi một quầy có kiểm soát chặt; tra số dư (query) đi quầy khác tối ưu cho đọc nhanh. Hai quầy *không dùng chung* quy trình.

**(b) Cơ chế chi tiết (📘).**

*@nestjs/cqrs giải gì (H-CQRS-001):* tách **ghi (command)** khỏi **đọc (query)**, decouple qua **bus + handler**. CommandBus định tuyến command → CommandHandler; QueryBus → QueryHandler; **EventBus** phát event *sau khi* xử lý để các handler khác phản ứng.

*Command/Query/Event ngữ nghĩa (H-CQRS-002):*

| Loại | Mục đích | Trả data nghiệp vụ? | Thì |
|---|---|---|---|
| **Command** | *thay đổi* state | không (chỉ ack/id) | mệnh lệnh ("CreateOrder") |
| **Query** | *đọc* state, không đổi | có | câu hỏi ("GetOrderById") |
| **Event** | *báo việc đã xảy ra* | (payload sự kiện) | quá khứ ("OrderCreated") |

*Khi nào dùng / over-engineering (H-CQRS-003):* hợp khi **domain phức tạp**, **đọc-ghi lệch tải** (đọc nhiều hơn ghi, cần model đọc riêng), hoặc **cần event/audit**. Với **CRUD đơn giản** → CQRS là *thừa*, tăng boilerplate (mỗi thao tác là 1 command + handler + event).

*Event Sourcing (H-CQRS-004, ➕):* thay vì lưu **state hiện tại**, lưu **chuỗi event**; state = *replay* event. Đánh đổi: **lợi** = audit/temporal query (xem state tại thời điểm bất kỳ); **hại** = phức tạp, **versioning event** (schema event đổi theo thời gian), **rebuild/projection**, **eventual consistency**. (Liên hệ mục K roadmap.)

*Saga cqrs vs orchestration (H-CQRS-005):* saga trong `@nestjs/cqrs` là **RxJS-based, in-process** — phản ứng dòng event để phát command tiếp theo. Khác **orchestration phân tán/bền vững** (Temporal/Camunda) cho **long-running transaction** xuyên service, có persistence/retry/compensation mạnh.

*Read model / projection (H-CQRS-006):* event cập nhật **read model riêng** (denormalized, tối ưu đọc). Rủi ro: có **độ trễ** giữa write và projection → **eventual consistency**, phải xử lý **stale read** (UI đọc dữ liệu hơi cũ).

**(c) Mép giới hạn & sai lầm.**
- Áp CQRS/ES cho CRUD đơn giản → boilerplate khổng lồ, team chậm.
- Quên versioning event trong ES → không replay được khi schema đổi.
- Kỳ vọng read model nhất quán tức thì → bug "vừa tạo xong query không thấy".

## ③ ⚠️ Kiến thức cũ / bị thay thế

| Cái cũ | Thay bằng (hiện nay) | Vì sao |
|---|---|---|
| Service "thượng đế" làm cả đọc+ghi cho domain phức tạp | CQRS tách command/query | Decouple, model đọc/ghi riêng |
| CQRS/ES cho mọi app "cho chuẩn" | Chỉ khi domain phức tạp/lệch tải | Tránh boilerplate vô ích |
| Saga cqrs cho transaction xuyên service dài | Orchestrator bền vững (Temporal...) | Persistence/retry/compensation thật |

## ④ Áp dụng thực tế + So sánh bigtech
Hệ đặt vé: ghi (`ReserveSeatCommand`) qua command bus có validation chặt; đọc bảng ghế (`GetAvailabilityQuery`) từ read model denormalized cập nhật qua event → chịu tải đọc cao. Ngân hàng/fintech dùng Event Sourcing cho **audit bất biến** (mọi giao dịch là event). Long-running flow (đặt vé + thanh toán + xuất vé) → orchestrator (Temporal) thay vì saga in-process. *(Pattern phổ biến; công cụ cụ thể verify.)*

## ⑤ Code thực hành + cấu hình
```typescript
// command + handler
export class CreateOrderCommand { constructor(public readonly dto: CreateOrderDto) {} }

@CommandHandler(CreateOrderCommand)
export class CreateOrderHandler implements ICommandHandler<CreateOrderCommand> {
  constructor(private repo: OrderRepo, private bus: EventBus) {}
  async execute(cmd: CreateOrderCommand) {
    const order = await this.repo.create(cmd.dto);
    this.bus.publish(new OrderCreatedEvent(order.id)); // phát event sau khi ghi
    return order.id;                                   // command chỉ trả id/ack
  }
}
```
```typescript
// controller chỉ "gửi" command/query, không chứa logic
@Post() create(@Body() dto: CreateOrderDto) {
  return this.commandBus.execute(new CreateOrderCommand(dto));
}
@Get(':id') get(@Param('id') id: string) {
  return this.queryBus.execute(new GetOrderQuery(id)); // đọc từ read model
}
```
> Cần `@nestjs/cqrs` (verify version). Nest 11: cqrs hỗ trợ **request-scoped** + **typed** command/event/query.

## ⑥ Keywords cần nhớ
- 🧠 **Cần HIỂU:** vì sao tách đọc/ghi · eventual consistency của read model · khi CQRS/ES là over-engineering · saga in-process vs orchestration.
- 📌 **Cần THUỘC:** CommandBus/QueryBus/EventBus · `@CommandHandler/@QueryHandler/@EventsHandler` · command (không trả data) vs query vs event.
- 🛠️ **Cần LÀM ĐƯỢC:** dựng 1 command+handler+event tối thiểu; phân loại đúng command/query/event cho tình huống.

## ⑦ Mental model
> **"Command đổi state (không trả data), Query đọc (không đổi), Event báo đã-xảy-ra. ES = lưu chuỗi event, state là replay. Dùng khi domain xứng đáng — CRUD thì đừng."**

## ⑧ Câu hỏi phỏng vấn & thách đố
1. *(khó)* @nestjs/cqrs giải gì; vai trò 3 bus? **(H-CQRS-001)**
2. *(TB)* Command vs Query vs Event ngữ nghĩa? **(H-CQRS-002)**
3. *(khó)* Khi nào CQRS đáng dùng / over-engineering? **(H-CQRS-003)**
4. *(rất khó)* Event Sourcing đánh đổi gì? **(H-CQRS-004)**
5. *(khó)* Read model/projection cập nhật sao, rủi ro consistency? **(H-CQRS-006)**

**🎯 Thách đố:** *"Sau khi `CreateOrderCommand` trả về thành công, client gọi ngay `GetOrderQuery` nhưng không thấy order. Hệ thống hỏng?"*
> Đáp: không nhất thiết — nếu query đọc từ **read model** cập nhật qua event (async), có **độ trễ projection** → eventual consistency. Cần: hoặc đọc từ write model cho read-after-write, hoặc thiết kế UI chấp nhận độ trễ, hoặc chờ event ack.

**🤖 "Hiểu để chỉ huy":** AI dễ "trang bị" CQRS/ES cho một CRUD todo-app vì nghe "kiến trúc xịn" → boilerplate ngộp. Người hiểu phán: domain này *không* xứng CQRS, dùng service đơn giản.

## ⑨ Bài tập + tiêu chí tự chấm
**BT1:** Command `CreateTask` + handler + event `TaskCreated`, một EventsHandler log. **Đạt khi:** tạo task → event được handle.
**BT2:** Viết 1 đoạn lý luận (5–8 câu) "app này có nên dùng CQRS không" cho 1 CRUD đơn giản. **Đạt khi:** kết luận *không*, nêu đúng lý do boilerplate/độ phức tạp.

## ⑩ Đọc thêm
- docs.nestjs.com → *Recipes → CQRS*. Tài liệu Event Sourcing/Temporal (liên hệ mục K).

---

# Bài 12 — Testing Nest sâu: TestingModule, e2e Supertest, TestContainers, override
> Phủ: **H-TST-001 → H-TST-007**

## ① Mục tiêu & vị trí trong mạch
Test là nơi DI (Bài 1–2) "trả lương": vì dependency được inject, ta **override** chúng để test cô lập. Bài này dạy unit test qua override, e2e qua Supertest, integration thật qua TestContainers, và hướng Jest→Vitest (v12). Đây là kỹ năng phân biệt người viết code chạy được với người viết code *được kiểm chứng*.

## ② Giảng cơ bản → nâng cao

**(a) Trực giác.** `Test.createTestingModule` như **dựng lại sân khấu thu nhỏ**: cùng đạo cụ (DI container) như runtime, nhưng bạn được **thay diễn viên** (override provider bằng mock) tùy cảnh quay.

**(b) Cơ chế chi tiết (📘).**

*TestingModule (H-TST-001):* `Test.createTestingModule({...}).compile()` dựng một **DI container test** thật — đăng ký/override provider, compile để lấy instance *y như runtime*. Đây là nền của mọi loại test Nest.

*Unit test + override (H-TST-002):* `overrideProvider(Dep).useValue(mock)` (hoặc `useFactory`) bơm mock → test logic service cô lập, **không chạm DB thật**.

*e2e với Supertest (H-TST-003):* `app = moduleRef.createNestApplication()` (có thể override), `await app.init()`, rồi `request(app.getHttpServer()).post('/x').send(...)` → assert status/body. e2e **áp cả pipe/guard** → kiểm hành vi end-to-end thật của route.

*TestContainers (H-TST-004, ➕):* chạy **Postgres/Redis thật** trong container ephemeral cho integration test → kiểm query/migration/transaction đúng hành vi engine thật, **không lệch** như SQLite/mock (vd SQL dialect, constraint, transaction isolation khác nhau). (Liên hệ mục L roadmap.)

*Mock vs integration (H-TST-005):* mock **nhanh** nhưng dễ **lệch thực tế** (mock sai giả định); integration **chậm** nhưng bắt **lỗi tích hợp** thật. Cân theo **test pyramid**: nhiều unit, ít e2e/integration ở đỉnh.

*Override guard/interceptor (H-TST-006):* `overrideGuard(AuthGuard).useValue({ canActivate: () => true })` để bỏ auth khi test logic route. **Rủi ro che lỗi:** test không phản ánh authz thật → **phải có test riêng cho guard**.

**(c) Mép giới hạn & hướng tương lai (H-TST-007).**
**Nest 12 dự kiến** đổi default test runner **Jest → Vitest** *(verify — chưa ra tại 6/2026; xem cảnh báo xung đột nguồn ở đầu tài liệu)*. Ảnh hưởng suite hiện tại: Vitest dùng ESM/transform khác (SWC), API **tương thích phần lớn** (`describe/it/expect`) nhưng **mock & config đổi** (`vi.fn` thay `jest.fn`, `vitest.config` thay jest config). Chiến lược: **migrate dần**, giữ test logic, đổi lớp mock/config; không cần viết lại assertion.

## ③ ⚠️ Kiến thức cũ / bị thay thế

| Cái cũ | Thay bằng (hiện nay) | Vì sao |
|---|---|---|
| Mock DB / dùng SQLite in-memory cho integration | **TestContainers** (DB thật ephemeral) | Không lệch dialect/transaction |
| Tự new service + mock thủ công | `Test.createTestingModule` + `overrideProvider` | Test như runtime, đúng DI |
| (xu hướng) Jest mặc định | **Vitest** (Nest 12, *verify*) | Nhanh hơn (SWC/ESM), DX tốt hơn |

## ④ Áp dụng thực tế + So sánh bigtech
Pyramid điển hình: hàng trăm unit test (service + mock repo) chạy mili-giây; vài chục integration test với TestContainers (Postgres thật) cho repository/migration; một nhúm e2e Supertest cho luồng critical (auth, checkout). CI chạy unit ở mọi commit, integration/e2e ở pipeline nặng hơn. Team lớn gate merge bằng coverage + e2e critical path.

## ⑤ Code thực hành + cấu hình
```typescript
// unit test: override repo bằng mock
const moduleRef = await Test.createTestingModule({
  providers: [OrdersService, { provide: OrderRepo, useValue: { create: jest.fn() } }],
}).compile();
const service = moduleRef.get(OrdersService);
```
```typescript
// e2e với Supertest + override guard
const moduleRef = await Test.createTestingModule({ imports: [AppModule] })
  .overrideGuard(AuthGuard('jwt')).useValue({ canActivate: () => true }) // bỏ auth để test route
  .compile();
const app = moduleRef.createNestApplication();
app.useGlobalPipes(new ValidationPipe({ whitelist: true })); // áp pipe như prod
await app.init();
await request(app.getHttpServer()).post('/orders').send({ item: 'x' }).expect(201);
```
```typescript
// integration với TestContainers (Postgres thật)
const pg = await new PostgreSqlContainer().start();   // verify API testcontainers
process.env.DATABASE_URL = pg.getConnectionUri();
// ... build app, chạy migration, test repo thật, rồi pg.stop()
```
```jsonc
// devDeps (verify bản): @nestjs/testing, supertest, testcontainers, jest (hoặc vitest ở v12)
```

## ⑥ Keywords cần nhớ
- 🧠 **Cần HIỂU:** vì sao DI làm test dễ · test pyramid · vì sao override guard có thể che lỗi · vì sao TestContainers > SQLite mock.
- 📌 **Cần THUỘC:** `Test.createTestingModule().compile()` · `overrideProvider/overrideGuard().useValue()` · `createNestApplication()`+`getHttpServer()`+supertest.
- 🛠️ **Cần LÀM ĐƯỢC:** 1 unit test mock repo + 1 e2e Supertest có override guard.

## ⑦ Mental model
> **"DI cho phép thay diễn viên: override provider = mock, override guard = bỏ auth. Unit nhanh-nhiều, integration thật-ít. TestContainers cho DB thật, không SQLite giả."**

## ⑧ Câu hỏi phỏng vấn & thách đố
1. *(TB)* `Test.createTestingModule` để làm gì? **(H-TST-001)**
2. *(TB)* Unit test service: mock dep qua DI sao? **(H-TST-002)**
3. *(khó)* e2e Supertest: dựng app test + bắn request? **(H-TST-003)**
4. *(khó)* TestContainers giải hạn chế gì của mock/in-memory? **(H-TST-004)**
5. *(rất khó)* Jest→Vitest (Nest 12) ảnh hưởng suite thế nào? **(H-TST-007)**

**🎯 Thách đố:** *"Bạn `overrideGuard(AuthGuard).useValue({canActivate:()=>true})` cho mọi e2e và coverage rất cao. Vì sao đây có thể là 'cảm giác an toàn giả'?"*
> Đáp: bạn đã **tắt authz** trong mọi test → suite *không bao giờ* kiểm route bị chặn đúng chưa. Coverage cao nhưng lỗ hổng phân quyền lọt lưới. Phải có test **riêng** cho guard (cả case bị từ chối).

**🤖 "Hiểu để chỉ huy":** AI hay mock luôn cả tầng DB và override mọi guard cho "test chạy xanh" — xanh nhưng rỗng. Người hiểu giữ ít integration thật + test guard riêng để test *có nghĩa*.

## ⑨ Bài tập + tiêu chí tự chấm
**BT1:** Unit test `OrdersService.create` với repo mock, assert gọi repo đúng args. **Đạt khi:** test pass, không chạm DB.
**BT2:** e2e: route `@Roles('admin')` — viết 2 test: admin (200) và user thường (403, *không* override guard). **Đạt khi:** cả hai pass, chứng tỏ authz thật được kiểm.

## ⑩ Đọc thêm
- docs.nestjs.com → *Fundamentals → Testing*. Trang Supertest, Testcontainers. Theo dõi Vitest support Nest 12 *(verify)*.

---

# Bài 13 — Runtime / kiến trúc / version (TL depth): adapter, lifecycle hooks, SWC, monorepo, chọn version
> Phủ: **H-RT-001 → H-RT-006**

## ① Mục tiêu & vị trí trong mạch
Bài tổng kết ở góc nhìn **Tech Lead**: Nest *chạy* trên gì, *bắt đầu/kết thúc* thế nào (graceful shutdown), build bằng gì (SWC), tổ chức nhiều service ra sao (monorepo), và **chọn version nào** cho dự án mới. Gom toàn cảnh 12 bài trước thành quyết định kiến trúc.

## ② Giảng cơ bản → nâng cao

**(a) Trực giác.** Nest core như **bộ não** tách rời khỏi **cơ thể HTTP** (Express/Fastify): đổi cơ thể (adapter) mà não giữ nguyên. Lifecycle hooks như **nghi thức mở cửa/đóng cửa** cửa hàng (bật đèn, mở kết nối → đóng kết nối, tắt đèn). Version choice như **chọn xe cho chuyến dài**: bản ổn định đang chạy tốt vs bản mới nhiều tính năng nhưng chưa kiểm chứng.

**(b) Cơ chế chi tiết.**

*Platform adapter (H-RT-001):* Nest core **độc lập HTTP lib** qua adapter; mặc định **Express** (Nest 11: **Express v5** — đã verify), thay được bằng **Fastify** cho throughput cao hơn, đánh đổi hệ sinh thái middleware Express phong phú hơn. Chọn Fastify khi đo được Express là bottleneck/cold-start nhạy. *(verify benchmark theo workload.)*

*Lifecycle hooks (H-RT-002):* `onModuleInit` → `onApplicationBootstrap` (init); `onModuleDestroy` → `beforeApplicationShutdown` → `onApplicationShutdown` (teardown). Dùng để mở/đóng kết nối, warm-up, graceful shutdown. ⚠️ **Nest 11 (verify):** thứ tự hook **kết thúc đã bị đảo** so với trước — kiểm lại nếu phụ thuộc thứ tự teardown.

*Graceful shutdown trong K8s (H-RT-003):* `app.enableShutdownHooks()` để Nest gọi `onModuleDestroy/beforeApplicationShutdown` khi nhận **SIGTERM** (K8s gửi khi xóa pod). Trong hook: **drain request đang chạy** + đóng DB/queue/connection trước khi process chết → không rớt request giữa chừng. (Liên hệ G-DIAG-009 mục G.)

*SWC (H-RT-004):* compiler viết bằng **Rust**, nhanh hơn `tsc` nhiều cho dev/build → là compiler mặc định Nest 11 *(verify)*. **Trade-off then chốt: SWC KHÔNG type-check** — chỉ transpile. Vẫn cần `tsc --noEmit` (hoặc IDE/CI) để bắt lỗi type. Nhanh nhưng đừng tưởng nó thay luôn type-checking.

*Monorepo mode (H-RT-005):* Nest hỗ trợ **apps + libs** trong một repo (build/test/lint chung, chia sẻ code) → tránh lặp code giữa nhiều service/microservice. So với **Nx** (mạnh hơn về caching/affected/graph), Nest monorepo nhẹ và built-in. Chọn theo quy mô.

*Chọn version (H-RT-006 — quyết định TL):* dự án mới 6/2026 → **chọn Nest 11 (ổn định, hiện hành 11.1.x)**. Lý do: **Nest 12 còn ở roadmap (~Q3 2026, chưa ra)**, mang breaking lớn (full **ESM**, **Vitest** thay Jest, **oxlint** thay ESLint, **Rspack** thay Webpack, **Standard Schema** trong route decorator). Cân nhắc Node LTS đi kèm. Chỉ thử 12 ở **môi trường đánh giá**, không đưa thẳng prod. *(verify ngày phát hành 12 tại docs/GitHub trước khi quyết.)*

**(c) Mép giới hạn & sai lầm.**
- Đổi sang Fastify "cho nhanh" mà chưa đo → mất ecosystem Express, có thể không lợi.
- Quên `enableShutdownHooks()` → pod bị kill cứng, rớt request/connection rò.
- Tưởng SWC type-check → bug type lọt vào build.
- Nhảy lên version mới nhất ngay khi vừa ra → gánh breaking trong prod.

## ③ ⚠️ Kiến thức cũ / bị thay thế

| Cái cũ | Thay bằng (hiện nay) | Vì sao |
|---|---|---|
| Build bằng `tsc` mặc định | **SWC** (Nest 11) cho dev/build nhanh + `tsc --noEmit` riêng để type-check | Tốc độ, nhưng vẫn cần type-check tách |
| Tắt app bằng kill cứng | `enableShutdownHooks()` + drain trên SIGTERM | Graceful trong K8s, không rớt request |
| Copy code giữa nhiều repo service | **Monorepo** (Nest apps/libs hoặc Nx) | Chia sẻ code, build/test chung |
| (xu hướng) Jest/ESLint/Webpack | Vitest/oxlint/Rspack (Nest 12, *verify, chưa ra*) | Toolchain nhanh hơn |

## ④ Áp dụng thực tế + So sánh bigtech
Service Nest trên K8s: `enableShutdownHooks()`, readiness/liveness probe, đóng pool DB trên SIGTERM → rolling update không rớt request. Monorepo `apps/` (api, worker, cron) + `libs/` (shared domain) build chung. Phần lớn team production 2026 ở Nest 11 + Express v5 (ecosystem) hoặc Fastify (throughput); SWC cho dev nhanh + `tsc` gate type ở CI. *(Pattern phổ biến; benchmark/version cụ thể luôn verify.)*

## ⑤ Code thực hành + cấu hình
```typescript
// graceful shutdown
const app = await NestFactory.create(AppModule);
app.enableShutdownHooks();            // bật để nhận SIGTERM → gọi hooks
await app.listen(3000);

@Injectable()
export class DbService implements OnModuleDestroy {
  async onModuleDestroy() { await this.pool.end(); } // đóng kết nối trước khi chết
}
```
```typescript
// chọn Fastify adapter (khi đo được cần throughput)
import { FastifyAdapter, NestFastifyApplication } from '@nestjs/platform-fastify';
const app = await NestFactory.create<NestFastifyApplication>(AppModule, new FastifyAdapter());
// verify: một số middleware Express không tương thích trực tiếp
```
```bash
# build: SWC nhanh nhưng KHÔNG type-check → chạy tsc riêng ở CI
nest build            # dùng SWC (Nest 11, verify)
tsc --noEmit          # gate type-check tách biệt
```

## ⑥ Keywords cần nhớ
- 🧠 **Cần HIỂU:** core tách HTTP adapter · vì sao SWC không thay type-check · graceful shutdown trên SIGTERM · cân nhắc 11 vs 12.
- 📌 **Cần THUỘC:** `enableShutdownHooks()` · `OnModuleInit/OnApplicationBootstrap/OnModuleDestroy/OnApplicationShutdown` · FastifyAdapter · Nest 11 = hiện hành, 12 = roadmap.
- 🛠️ **Cần LÀM ĐƯỢC:** bật graceful shutdown đóng tài nguyên đúng; lập luận chọn version cho 1 dự án.

## ⑦ Mental model
> **"Não Nest tách khỏi thân HTTP (adapter). SWC build nhanh nhưng không type-check. SIGTERM → hooks → drain rồi mới chết. Dự án mới 6/2026: chọn 11 ổn định, ngó 12 ở môi trường thử."**

## ⑧ Câu hỏi phỏng vấn & thách đố
1. *(TB)* Platform adapter là gì, khi nào Fastify? **(H-RT-001)**
2. *(khó)* Lifecycle hooks chạy khi nào, dùng cho gì? **(H-RT-002)**
3. *(khó)* `enableShutdownHooks()` + SIGTERM trong K8s phối hợp sao? **(H-RT-003)**
4. *(TB)* SWC là gì, trade-off so với tsc? **(H-RT-004)**
5. *(rất khó)* Là TL, chọn Nest 11 hay 12 cho dự án mới 6/2026, lập luận? **(H-RT-006)**

**🎯 Thách đố:** *"CI của bạn dùng `nest build` (SWC) và 'xanh', nhưng prod crash vì một lỗi type rõ ràng. Vì sao build không bắt được?"*
> Đáp: SWC **chỉ transpile, không type-check** → lỗi type lọt qua build. Phải thêm bước `tsc --noEmit` (hoặc `nest build --type-check` nếu hỗ trợ) trong CI để gate type. *(verify cờ theo bản.)*

**🤖 "Hiểu để chỉ huy":** AI sẽ vui vẻ chọn "phiên bản mới nhất" hoặc đề xuất Fastify "cho nhanh", và tin `nest build` đã type-check. Người hiểu mới cân nhắc breaking change, ecosystem, và bổ sung type-check gate — đây đúng là vai trò TL.

## ⑨ Bài tập + tiêu chí tự chấm
**BT1:** Bật `enableShutdownHooks()` + đóng 1 "kết nối" giả trong `onModuleDestroy`; gửi SIGTERM (`kill -TERM`) và quan sát log đóng. **Đạt khi:** thấy log teardown trước khi process thoát.
**BT2:** Viết 1 đoạn quyết định (8–12 câu) chọn version + adapter cho một API mới, có nêu trade-off và bước verify. **Đạt khi:** lập luận dựa trên ổn định/breaking/ecosystem, không chỉ "mới nhất là tốt nhất".

## ⑩ Đọc thêm
- docs.nestjs.com → *Techniques → Performance (Fastify)*, *Lifecycle events*, *FAQ → Hybrid app*, *CLI → Monorepo & libraries*, *Migration guide* (11) & *Roadmap* (12, *verify*).

---

# ✅ KẾT — Sau Bước B + C
- **13 bài** phủ trọn **82 câu** của bộ đề `QB_H_nestjs.md` (không bỏ ID).
- Học theo **dependency order**: DI → Module → Lifecycle → (Pipe/Guard/Interceptor) → Decorator → Dynamic/Config → Microservices/CQRS/Testing → Runtime/Version.
- Mọi mốc version/tooling đã **search + cite** hoặc gắn *(verify)* để đối chiếu docs chính thức.

## Bước tiếp theo (theo WORKFLOW 2)
- **Bước D — Phỏng vấn theo đợt (chấm LIVE):** mỗi đợt 10–15 câu trộn dễ→khó; tôi hỏi, bạn trả lời, tôi dựng đáp án chuẩn + chấm ĐẠT/CHƯA ngay lúc đó (recall trước, đáp án sau).
- **Bước E — Cổng hoàn thành + ôn ngắt quãng:** xong = ĐẠT toàn bộ 82 câu; chèn lại câu cũ theo lịch hôm nay → mai → 3 ngày → 1 tuần.

> *Câu thần chú:* **"Tôi không nhớ để gõ, tôi hiểu để chỉ huy — và tra cứu phần còn lại."**
