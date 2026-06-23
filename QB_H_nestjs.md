# 🧪 QB_H — Ngân hàng câu hỏi: NestJS Framework Depth

> **Mục H** trong roadmap Tech Lead Backend (🟡 ~55%) · Sinh theo **WORKFLOW 2 — Bước A**.
> Đây là **mục LỚN** → bộ đề phủ HẾT 11 mục con + nhóm runtime/kiến trúc Nest để đủ chiều sâu TL.
> **Chỉ câu hỏi — KHÔNG kèm đáp án.** Mỗi câu có dòng *"dò cái gì"* = tiêu chí ĐẠT, dùng để chấm **live** ở Bước D.
> **Chống trùng:** đã đọc `QB_G_nodejs.md` — H giữ phần *đặc thù Nest* (DI container, lifecycle, decorator, transport...), không lặp lại security generic / event loop của G.
> **Tổng: 82 câu.**

## Cách đọc
- **ID:** `H-<tag>-<số>`. Tag mục con liệt kê ở mục lục dưới.
- **Độ khó:** ⭐ định nghĩa cơ bản → ⭐⭐⭐⭐⭐ phân biệt senior với TL thật.
- **"Dò cái gì":** điều người trả lời PHẢI chạm tới mới tính ĐẠT (không phải đáp án — chỉ là mốc chấm).

## Bối cảnh phiên bản (tính tại 6/2026 — *verify lại tại docs.nestjs.com khi học*)
- **NestJS 11** = bản hiện hành (11.1.x). Điểm đáng nhớ: **Express v5** là default; **SWC** là compiler mặc định; `ConsoleLogger` hỗ trợ **JSON logging**; `IntrinsicException` (bỏ qua auto-log); transporter có `unwrap()`; bootstrap được **không cần root AppModule**.
- **NestJS 12** (roadmap ~Q3 2026, *CHƯA ra tại 6/2026*): full **ESM**, **Vitest thay Jest**, **oxlint** thay ESLint, **Rspack** thay Webpack, **Standard Schema** validation trong route decorator. → Khi bị hỏi "Nest đang đi về đâu", đây là câu trả lời cập nhật.
- ⚠️ Mọi câu chạm version/tooling đổi nhanh đều có dấu *(verify)* — đối chiếu docs chính thức trước khi tin.

---

## Mục lục mục con (ID prefix → số câu)

| Tag | Mục con | Số câu |
|---|---|---|
| **H-DI** | Dependency Injection (container, provider, scope) | 9 |
| **H-MOD** | Module system & scope (circular dep, forwardRef) | 8 |
| **H-LIFE** | Request lifecycle (middleware→guard→interceptor→pipe→controller) | 7 |
| **H-PIPE** | Pipes & validation (DTO + class-validator/transformer) | 7 |
| **H-GRD** | Guards & authorization (RBAC) | 6 |
| **H-INT** | Interceptors / Exception filters | 6 |
| **H-DEC** | Custom decorators | 5 |
| **H-DYN** | Dynamic Modules & ConfigModule (forRoot/forRootAsync) | 7 |
| **H-MS** | Microservices module (transport TCP/Redis/Kafka/gRPC...) | 8 |
| **H-CQRS** | CQRS + Event Sourcing (@nestjs/cqrs) | 6 |
| **H-TST** | Testing Nest sâu (unit/e2e Supertest + TestContainers) | 7 |
| **H-RT** | Runtime/kiến trúc/version Nest (chiều sâu TL) | 6 |

---

## H-DI — Dependency Injection

**H-DI-001** ⭐⭐⭐
"DI container của Nest hoạt động thế nào? Một provider được resolve ra sao tại runtime?"
> *Dò cái gì:* Nest đọc metadata (decorator + reflection), dựng dependency graph, khởi tạo provider theo thứ tự phụ thuộc, inject qua constructor.

**H-DI-002** ⭐⭐
"Provider trong Nest là gì? `@Injectable()` đánh dấu điều gì?"
> *Dò cái gì:* provider = thứ có thể inject (service, repo, factory...); `@Injectable` cho phép Nest quản lý vòng đời + inject.

**H-DI-003** ⭐⭐⭐⭐
"Ba injection scope (DEFAULT/singleton, REQUEST, TRANSIENT) khác nhau gì? Trade-off của REQUEST scope?"
> *Dò cái gì:* singleton dùng chung 1 instance; REQUEST tạo mới mỗi request (đắt, ảnh hưởng performance, 'bubble up' lên cả chain); TRANSIENT mỗi nơi inject 1 instance.

**H-DI-004** ⭐⭐⭐⭐
"REQUEST-scoped provider 'lây' scope lên controller/provider cha như thế nào, và vì sao đáng lo về hiệu năng?"
> *Dò cái gì:* bất cứ thứ gì phụ thuộc provider REQUEST-scoped cũng thành REQUEST-scoped → mất singleton, khởi tạo lại mỗi request.

**H-DI-005** ⭐⭐⭐⭐
"Custom provider: useClass / useValue / useFactory / useExisting khác nhau và dùng khi nào?"
> *Dò cái gì:* useClass (đổi implementation), useValue (hằng/mảng mock), useFactory (tạo động, có inject), useExisting (alias). Cho ví dụ đúng tình huống.

**H-DI-006** ⭐⭐⭐⭐
"Injection token là gì? Vì sao cần inject bằng token (string/Symbol) thay vì class, ví dụ với interface?"
> *Dò cái gì:* TypeScript interface không tồn tại runtime → không inject được trực tiếp; dùng token + `@Inject(TOKEN)` để map.

**H-DI-007** ⭐⭐⭐
"Property injection vs constructor injection — Nest ưu tiên cái nào, khi nào buộc dùng `@Inject`?"
> *Dò cái gì:* constructor là mặc định/khuyến nghị; `@Inject` khi token không phải class hoặc property injection trong base class.

**H-DI-008** ⭐⭐⭐⭐⭐
"Provider của bạn cần một giá trị async (vd kết nối đã sẵn sàng) lúc khởi tạo — bạn cung cấp qua DI thế nào?"
> *Dò cái gì:* async useFactory (provider async) hoặc lifecycle hook; hiểu Nest chờ factory resolve trước khi app sẵn sàng.

**H-DI-009** ⭐⭐⭐⭐
"Vì sao DI giúp testability? Trong unit test bạn thay một dependency thật bằng mock qua cơ chế nào?"
> *Dò cái gì:* DI cho phép override provider; Test module `overrideProvider().useValue()` để bơm mock thay vì instance thật.

---

## H-MOD — Module system & scope

**H-MOD-001** ⭐⭐
"Module trong Nest là gì? `@Module({imports, controllers, providers, exports})` mỗi field làm gì?"
> *Dò cái gì:* đơn vị tổ chức feature; imports (module khác), providers (đăng ký nội bộ), exports (chia sẻ ra ngoài), controllers (route).

**H-MOD-002** ⭐⭐⭐
"Provider của module A muốn dùng được ở module B thì cần gì?"
> *Dò cái gì:* A phải `exports` provider đó VÀ B phải `imports` module A; không export thì private trong A.

**H-MOD-003** ⭐⭐⭐
"`@Global()` module để làm gì? Vì sao nên dùng tiết chế?"
> *Dò cái gì:* provider khả dụng toàn app không cần import lại; lạm dụng làm mờ phụ thuộc, khó trace, phá tính module hóa.

**H-MOD-004** ⭐⭐⭐⭐
"Circular dependency giữa hai module/provider xảy ra thế nào và `forwardRef()` giải quyết ra sao?"
> *Dò cái gì:* hai bên phụ thuộc lẫn nhau → Nest không resolve được thứ tự; forwardRef trì hoãn resolve; đồng thời nhận ra đây thường là *dấu hiệu thiết kế cần tách*.

**H-MOD-005** ⭐⭐⭐⭐
"`ModuleRef` dùng để làm gì? Khi nào cần lấy provider động thay vì inject thẳng?"
> *Dò cái gì:* lấy instance theo token tại runtime (kể cả scoped/lazy), cho factory/plugin động hoặc khi không biết token lúc compile.

**H-MOD-006** ⭐⭐⭐
"Static module vs dynamic module khác nhau gì (ở mức khái niệm, chưa đi vào forRoot)?"
> *Dò cái gì:* static cấu hình cứng; dynamic trả về metadata module được cấu hình tại thời điểm import (truyền options).

**H-MOD-007** ⭐⭐⭐⭐
"Bạn tổ chức một codebase Nest lớn theo module thế nào để tránh 'God module' và coupling chéo?"
> *Dò cái gì:* chia theo bounded context/feature, shared module cho cross-cutting, hạn chế export, hướng dependency một chiều; tư duy TL.

**H-MOD-008** ⭐⭐⭐⭐⭐
"Lazy-loading module trong Nest (LazyModuleLoader) có ý nghĩa gì với app HTTP thường vs serverless/microservice?"
> *Dò cái gì:* giảm cold start/bộ nhớ khi chỉ cần module theo nhu cầu; với HTTP server long-running lợi ích nhỏ, với serverless/CLI rõ hơn. *(verify API hiện hành.)*

---

## H-LIFE — Request lifecycle

**H-LIFE-001** ⭐⭐⭐⭐
"Kể đúng thứ tự một request đi qua: middleware, guard, interceptor, pipe, controller, rồi interceptor/exception filter ở chiều ra."
> *Dò cái gì:* middleware → guards → interceptors (pre) → pipes → handler → interceptors (post) → exception filter (khi lỗi); không đảo thứ tự guard/pipe.

**H-LIFE-002** ⭐⭐⭐
"Middleware và guard khác nhau ở chỗ nào? Vì sao auth thường đặt ở guard chứ không middleware?"
> *Dò cái gì:* middleware không biết execution context/handler metadata; guard có ExecutionContext + Reflector → quyết định theo route/role.

**H-LIFE-003** ⭐⭐⭐⭐
"Pipe chạy trước hay sau guard? Nếu validation cần thông tin user đã auth thì sao?"
> *Dò cái gì:* guard chạy trước pipe; nên dựa vào user từ guard, không nhét logic auth vào pipe.

**H-LIFE-004** ⭐⭐⭐
"ExecutionContext là gì và vì sao guard/interceptor cần nó?"
> *Dò cái gì:* abstraction cho ngữ cảnh hiện tại (HTTP/RPC/WS), lấy được handler + class để đọc metadata, hỗ trợ đa transport.

**H-LIFE-005** ⭐⭐⭐⭐
"Khi exception ném ra ở handler, luồng đi tiếp thế nào? Interceptor 'post' còn chạy không?"
> *Dò cái gì:* lỗi bị exception filter bắt; phần map response của interceptor (sau handler) không chạy bình thường — hiểu rẽ nhánh error.

**H-LIFE-006** ⭐⭐⭐
"Binding ở các cấp: global / controller / method — áp dụng cho guard/pipe/interceptor thế nào, ưu tiên ra sao?"
> *Dò cái gì:* có thể gắn 3 cấp; hiểu phạm vi áp dụng và cách kết hợp (global qua APP_GUARD provider để vẫn dùng DI).

**H-LIFE-007** ⭐⭐⭐⭐⭐
"Vì sao đăng ký global guard bằng `app.useGlobalGuards()` khác với provider `APP_GUARD`? Khi nào buộc dùng cách nào?"
> *Dò cái gì:* useGlobalGuards không qua DI (không inject được service); APP_GUARD provider chạy trong DI → inject được Reflector/service.

---

## H-PIPE — Pipes & validation

**H-PIPE-001** ⭐⭐
"Pipe trong Nest làm gì? Hai vai trò chính của nó?"
> *Dò cái gì:* transform (đổi/ép kiểu input) và validate (chặn input sai) trước khi vào handler.

**H-PIPE-002** ⭐⭐⭐
"DTO + class-validator + ValidationPipe phối hợp thế nào để validate request body?"
> *Dò cái gì:* DTO class gắn decorator (@IsString...); ValidationPipe đọc metadata, validate, ném 400 nếu sai.

**H-PIPE-003** ⭐⭐⭐⭐
"`whitelist`, `forbidNonWhitelisted`, `transform` trong ValidationPipe làm gì? Vì sao nên bật?"
> *Dò cái gì:* whitelist bỏ field thừa, forbidNonWhitelisted báo lỗi nếu có field lạ, transform ép kiểu/instance hóa DTO; chống mass-assignment.

**H-PIPE-004** ⭐⭐⭐⭐
"class-transformer (`@Type`, `plainToInstance`) cần thiết ở đâu, vì sao chỉ class-validator là chưa đủ với nested/array?"
> *Dò cái gì:* payload JSON là plain object → cần transform thành instance + `@Type` cho nested để validator chạy đúng.

**H-PIPE-005** ⭐⭐⭐
"Viết một custom pipe (vd ParseObjectIdPipe) — `transform(value, metadata)` nhận gì, trả gì?"
> *Dò cái gì:* nhận value + ArgumentMetadata, trả giá trị đã transform hoặc ném exception; hiểu pipe chỉ tác động 1 tham số được bind.

**H-PIPE-006** ⭐⭐⭐
"Validation ở pipe (boundary) vs validation ở business/domain layer khác nhau gì? Đặt cái nào ở đâu?"
> *Dò cái gì:* pipe lo shape/format ở biên; invariant nghiệp vụ thuộc domain; không dồn hết vào DTO.

**H-PIPE-007** ⭐⭐⭐⭐⭐
"Nest 12 dự kiến hỗ trợ Standard Schema (vd Zod) trong route decorator — điều này đổi cách validate so với class-validator thế nào?"
> *Dò cái gì:* biết hướng dịch chuyển sang schema-based (Zod/Valibot) bên cạnh decorator-based; trade-off type-inference vs decorator. *(verify trạng thái v12.)*

---

## H-GRD — Guards & authorization

**H-GRD-001** ⭐⭐⭐
"Guard là gì? `canActivate()` trả về gì và quyết định điều gì?"
> *Dò cái gì:* trả boolean/Promise/Observable; true cho qua, false/throw chặn (403); dùng cho authz/precondition.

**H-GRD-002** ⭐⭐⭐
"Phân quyền RBAC bằng guard + metadata: luồng `@Roles()` → Reflector → RolesGuard hoạt động ra sao?"
> *Dò cái gì:* `@Roles` gắn metadata, RolesGuard dùng Reflector đọc metadata + so với role của user (từ request), quyết định.

**H-GRD-003** ⭐⭐⭐⭐
"`Reflector.getAllAndOverride` vs `getAllAndMerge` khác nhau gì? (Nest 11 đổi return type — verify)"
> *Dò cái gì:* override lấy giá trị ưu tiên theo cấp; merge gộp; biết v11 cải tiến type inference (getAllAndOverride trả `T | undefined`). *(verify)*

**H-GRD-004** ⭐⭐⭐⭐
"Authentication vs Authorization trong Nest đặt ở đâu? AuthGuard (passport) vs custom RolesGuard phối hợp thế nào?"
> *Dò cái gì:* authn (xác thực, gắn req.user) trước authz (kiểm quyền); thường AuthGuard chạy trước RolesGuard.

**H-GRD-005** ⭐⭐⭐
"Vì sao không nên nhét logic phân quyền phức tạp vào controller handler?"
> *Dò cái gì:* tách concern, tái dùng, test được, áp metadata khai báo; handler chỉ lo nghiệp vụ.

**H-GRD-006** ⭐⭐⭐⭐⭐
"Khi RBAC cứng không đủ (quyền theo tài nguyên/thuộc tính), bạn mở rộng guard sang ABAC/policy thế nào trong Nest?"
> *Dò cái gì:* policy-based guard (CASL/OPA), kiểm quyền theo (user, action, resource) thay vì chỉ role; nhận ra giới hạn RBAC. *(liên hệ mục M roadmap.)*

---

## H-INT — Interceptors / Exception filters

**H-INT-001** ⭐⭐⭐
"Interceptor làm được gì mà guard/pipe không? Nêu vài use case."
> *Dò cái gì:* bọc cả trước+sau handler (transform response, logging, caching, timeout, mapping); can thiệp luồng RxJS trả về.

**H-INT-002** ⭐⭐⭐⭐
"`intercept(context, next)` và `next.handle()` (Observable) hoạt động thế nào? Vì sao Nest dùng RxJS ở đây?"
> *Dò cái gì:* next.handle() trả Observable của response stream; dùng pipe RxJS (map/tap/catchError/timeout) để xử lý sau handler.

**H-INT-003** ⭐⭐⭐
"Chuẩn hóa response envelope toàn app (vd `{data, meta}`) nên đặt ở interceptor hay ở đâu, vì sao?"
> *Dò cái gì:* global interceptor map mọi response; tập trung, không lặp ở từng handler.

**H-INT-004** ⭐⭐⭐
"Exception filter là gì? `@Catch()` + `ExceptionFilter` chuẩn hóa error response ra sao?"
> *Dò cái gì:* bắt exception (theo loại hoặc tất cả), format response thống nhất, log; phân biệt HttpException vs lỗi không lường.

**H-INT-005** ⭐⭐⭐⭐
"HttpException và built-in exception (NotFoundException...) khác lỗi JS thường thế nào trong Nest? Lỗi unknown bị xử lý ra sao?"
> *Dò cái gì:* HttpException map status code; lỗi không phải HttpException → 500 mặc định; cần filter để không lộ stack.

**H-INT-006** ⭐⭐⭐⭐⭐
"Thứ tự áp dụng nhiều interceptor (nesting) và quan hệ với exception filter khi lỗi xảy ra — giải thích."
> *Dò cái gì:* interceptor chạy theo thứ tự đăng ký, lồng nhau như onion; lỗi ném ra rẽ sang filter, hiểu catchError trong interceptor có thể chặn trước filter.

---

## H-DEC — Custom decorators

**H-DEC-001** ⭐⭐⭐
"Custom param decorator (vd `@CurrentUser()`) tạo bằng gì và lấy dữ liệu từ đâu?"
> *Dò cái gì:* `createParamDecorator((data, ctx) => ...)`, lấy từ ExecutionContext (request); thay cho `@Req()` thủ công.

**H-DEC-002** ⭐⭐⭐
"`SetMetadata` / decorator factory để gắn metadata (vd `@Roles`) — cặp với Reflector thế nào?"
> *Dò cái gì:* SetMetadata gắn key-value lên handler/class; Reflector đọc lại trong guard/interceptor.

**H-DEC-003** ⭐⭐⭐⭐
"`applyDecorators()` để làm gì? Cho ví dụ gộp nhiều decorator thành một."
> *Dò cái gì:* compose nhiều decorator (vd @Auth = @UseGuards + @ApiBearerAuth + @Roles) cho DRY/đọc gọn.

**H-DEC-004** ⭐⭐⭐
"Decorator của Nest dựa trên cơ chế nào của TypeScript/JS để hoạt động (reflection)?"
> *Dò cái gì:* metadata reflection (reflect-metadata, emitDecoratorMetadata); Nest đọc metadata để dựng DI/route. *(lưu ý decorator stage-3 vs legacy — verify khi migrate.)*

**H-DEC-005** ⭐⭐⭐⭐⭐
"AI sinh một custom decorator dùng được, nhưng bạn cần kiểm tra điều gì để chắc nó không phá DI/scope?"
> *Dò cái gì:* nhận ra điểm dễ sai: lấy sai context theo transport, không tương thích scoped provider, metadata key trùng; *hiểu để kiểm tra* không chỉ chạy được.

---

## H-DYN — Dynamic Modules & ConfigModule

**H-DYN-001** ⭐⭐⭐⭐
"Dynamic module là gì? `forRoot()` trả về cái gì và khác module tĩnh ra sao?"
> *Dò cái gì:* method trả về `DynamicModule` (module + providers cấu hình theo options); cho phép cấu hình lúc import.

**H-DYN-002** ⭐⭐⭐⭐
"`forRoot()` vs `forRootAsync()` khác nhau gì? Khi nào buộc dùng async?"
> *Dò cái gì:* async khi options phụ thuộc thứ khác (vd ConfigService) → dùng useFactory/inject; sync khi options tĩnh.

**H-DYN-003** ⭐⭐⭐
"`forFeature()` trong các module (vd TypeORM/Mongoose) khác `forRoot()` chỗ nào?"
> *Dò cái gì:* forRoot cấu hình toàn cục (connection); forFeature đăng ký theo feature (entity/repo) ở module con.

**H-DYN-004** ⭐⭐⭐
"ConfigModule (`@nestjs/config`) load env thế nào? Vì sao nên dùng nó thay vì đọc `process.env` rải rác?"
> *Dò cái gì:* load .env + validate, inject ConfigService có type; tập trung, test được, tránh undefined ngầm.

**H-DYN-005** ⭐⭐⭐⭐
"Validate biến môi trường lúc khởi động (Joi/zod schema) trong ConfigModule — vì sao nên fail-fast?"
> *Dò cái gì:* phát hiện thiếu/sai env ngay khi boot thay vì lỗi runtime mơ hồ sau này.

**H-DYN-006** ⭐⭐⭐⭐
"`registerAsync` (vd JwtModule, BullModule) cho phép inject dependency vào cấu hình — cơ chế useFactory ở đây là gì?"
> *Dò cái gì:* factory nhận inject (ConfigService...) trả options async; hiểu đây là dynamic module + DI.

**H-DYN-007** ⭐⭐⭐⭐⭐
"Bạn viết một dynamic module tái dùng (vd MailModule.forRoot(options)) cho nhiều dự án — thiết kế options + token + async thế nào?"
> *Dò cái gì:* định nghĩa options interface + injection token, hỗ trợ cả forRoot/forRootAsync, export service; tư duy thư viện nội bộ.

---

## H-MS — Microservices module

**H-MS-001** ⭐⭐⭐
"`@nestjs/microservices` cho phép giao tiếp giữa service Nest bằng những transport nào?"
> *Dò cái gì:* TCP, Redis, NATS, RabbitMQ, Kafka, gRPC, MQTT; chọn theo nhu cầu (queue vs streaming vs RPC).

**H-MS-002** ⭐⭐⭐⭐
"Message pattern (`@MessagePattern`) vs event pattern (`@EventPattern`) khác nhau gì? Request-response vs fire-and-forget."
> *Dò cái gì:* MessagePattern chờ phản hồi (RPC-like); EventPattern phát sự kiện không chờ; chọn đúng theo coupling.

**H-MS-003** ⭐⭐⭐⭐
"Hybrid application (vừa HTTP vừa microservice) trong Nest là gì, dựng thế nào?"
> *Dò cái gì:* app HTTP + `connectMicroservice` + `startAllMicroservices`; một process phục vụ cả hai.

**H-MS-004** ⭐⭐⭐⭐
"Khi nào chọn gRPC giữa các service Nest thay vì REST/TCP transport?"
> *Dò cái gì:* contract Protobuf, hiệu năng/streaming, đa ngôn ngữ; trade-off so với REST (dễ debug) — liên hệ mục F/K roadmap.

**H-MS-005** ⭐⭐⭐⭐
"Transport Kafka/RabbitMQ qua Nest: ack, consumer group/queue, retry được Nest abstraction tới đâu, còn lại bạn lo gì?"
> *Dò cái gì:* Nest lo bind pattern + serialize; ack/at-least-once/idempotent consumer/DLQ vẫn là trách nhiệm thiết kế. *(verify option v11.)*

**H-MS-006** ⭐⭐⭐
"`ClientProxy` / `@nestjs/microservices` client gửi message thế nào, và `unwrap()` (Nest 11) cho bạn gì?"
> *Dò cái gì:* ClientProxy.send/emit trả Observable; unwrap() lấy native client để thao tác nâng cao ngoài API chuẩn. *(verify)*

**H-MS-007** ⭐⭐⭐⭐
"Exception/timeout trong microservice Nest xử lý khác HTTP thế nào (RpcException)?"
> *Dò cái gì:* RpcException + filter riêng cho RPC context; timeout phía client; không trả HTTP status như REST.

**H-MS-008** ⭐⭐⭐⭐⭐
"Khi nào KHÔNG nên tách microservice trong Nest dù framework hỗ trợ sẵn? (cảnh báo over-engineering)"
> *Dò cái gì:* monolith module hóa đủ tốt cho team/scale nhỏ; tách sớm gây chi phí vận hành/độ trễ/transaction phân tán — tư duy TL chứ không 'vì Nest làm được'.

---

## H-CQRS — CQRS + Event Sourcing

**H-CQRS-001** ⭐⭐⭐⭐
"`@nestjs/cqrs` giải quyết bài toán gì? Command bus / Query bus / Event bus đóng vai trò gì?"
> *Dò cái gì:* tách ghi (command) khỏi đọc (query), decouple qua bus + handler; Event bus phát sự kiện sau khi xử lý.

**H-CQRS-002** ⭐⭐⭐
"Command vs Query vs Event khác nhau ngữ nghĩa thế nào trong CQRS?"
> *Dò cái gì:* command thay đổi state (không trả data nghiệp vụ), query đọc không đổi state, event báo việc đã xảy ra (quá khứ).

**H-CQRS-003** ⭐⭐⭐⭐
"Khi nào CQRS đáng dùng và khi nào là over-engineering cho một CRUD app?"
> *Dò cái gì:* hợp khi domain phức tạp/đọc-ghi lệch tải/cần event; CRUD đơn giản thì thừa, tăng boilerplate.

**H-CQRS-004** ⭐⭐⭐⭐⭐
"Event Sourcing là gì? Lưu state bằng chuỗi event đánh đổi gì so với lưu state hiện tại?"
> *Dò cái gì:* state = replay event; lợi: audit/temporal, hại: phức tạp, versioning event, rebuild/projection, eventual consistency. *(liên hệ mục K roadmap.)*

**H-CQRS-005** ⭐⭐⭐⭐
"Saga trong `@nestjs/cqrs` (RxJS-based) khác saga orchestration (Temporal/Camunda) ở mức nào?"
> *Dò cái gì:* saga cqrs phản ứng event trong-process bằng RxJS; orchestration phân tán/bền vững qua công cụ riêng cho long-running transaction.

**H-CQRS-006** ⭐⭐⭐⭐
"Read model / projection trong CQRS được cập nhật thế nào, và rủi ro consistency là gì?"
> *Dò cái gì:* event cập nhật read model riêng (denormalized); có độ trễ → eventual consistency, cần xử lý stale read.

---

## H-TST — Testing Nest sâu

**H-TST-001** ⭐⭐⭐
"`Test.createTestingModule()` dùng để làm gì? Vì sao nó là nền tảng test Nest?"
> *Dò cái gì:* dựng DI container test, đăng ký/override provider, compile module để lấy instance test thật như runtime.

**H-TST-002** ⭐⭐⭐
"Unit test một service Nest: mock dependency qua DI thế nào (`overrideProvider`)?"
> *Dò cái gì:* override provider bằng useValue/useFactory mock; test logic service cô lập, không chạm DB thật.

**H-TST-003** ⭐⭐⭐⭐
"e2e test với Supertest: cách dựng app test (`createNestApplication`) và bắn HTTP request giả?"
> *Dò cái gì:* build app từ AppModule (có thể override), `app.getHttpServer()` + supertest request, assert status/body; áp cả pipe/guard.

**H-TST-004** ⭐⭐⭐⭐
"TestContainers cho integration test với DB thật giải quyết hạn chế gì của mock/in-memory DB?"
> *Dò cái gì:* chạy Postgres/Redis thật trong container ephemeral → test query/migration/transaction đúng hành vi, không lệch như SQLite/mock. *(liên hệ mục L roadmap.)*

**H-TST-005** ⭐⭐⭐
"Khi nào mock, khi nào dùng integration thật? Trade-off độ tin cậy vs tốc độ."
> *Dò cái gì:* mock nhanh nhưng dễ lệch thực tế; integration chậm nhưng bắt lỗi tích hợp; cân theo test pyramid.

**H-TST-006** ⭐⭐⭐⭐
"Override guard/interceptor trong test (vd bỏ AuthGuard) làm thế nào và rủi ro che lỗi là gì?"
> *Dò cái gì:* `overrideGuard().useValue(...)`; rủi ro: test không phản ánh authz thật → cần test riêng cho guard.

**H-TST-007** ⭐⭐⭐⭐⭐
"Nest 12 dự kiến đổi default test runner Jest → Vitest. Việc này ảnh hưởng test suite hiện tại của bạn thế nào?"
> *Dò cái gì:* biết hướng dịch chuyển; ESM/transform khác, API tương thích phần lớn nhưng mock/config đổi; chiến lược migrate dần. *(verify trạng thái v12.)*

---

## H-RT — Runtime / kiến trúc / version Nest (chiều sâu TL)

**H-RT-001** ⭐⭐⭐
"Nest dùng platform adapter (Express vs Fastify) thế nào? Khi nào chọn Fastify?"
> *Dò cái gì:* Nest core độc lập HTTP lib qua adapter; Fastify cho throughput cao hơn, đánh đổi hệ sinh thái middleware Express. *(Nest 11 default Express v5 — verify.)*

**H-RT-002** ⭐⭐⭐⭐
"Lifecycle hooks (`onModuleInit`, `onApplicationBootstrap`, `onModuleDestroy`, `beforeApplicationShutdown`) chạy khi nào? Dùng cho gì?"
> *Dò cái gì:* thứ tự init/destroy; dùng mở/đóng kết nối, warm-up, graceful shutdown; bật `enableShutdownHooks()`.

**H-RT-003** ⭐⭐⭐⭐
"`enableShutdownHooks()` + SIGTERM cho graceful shutdown trong K8s — phối hợp thế nào? (liên hệ G-DIAG-009)"
> *Dò cái gì:* Nest gọi onModuleDestroy/beforeShutdown khi nhận tín hiệu; drain request + đóng DB/queue trước khi pod chết.

**H-RT-004** ⭐⭐⭐
"SWC là gì và vì sao Nest 11 dùng nó làm compiler mặc định? Trade-off so với tsc?"
> *Dò cái gì:* compiler Rust nhanh hơn nhiều cho dev/build; trade-off: type-check vẫn cần tsc riêng (SWC không type-check). *(verify)*

**H-RT-005** ⭐⭐⭐⭐
"Monorepo mode của Nest (apps + libs) giải quyết gì khi có nhiều service dùng chung code?"
> *Dò cái gì:* nhiều app + thư viện chia sẻ trong 1 repo, build/test chung; so sánh với Nx; tránh lặp code giữa microservice.

**H-RT-006** ⭐⭐⭐⭐⭐
"Là TL chọn version Nest cho dự án mới 6/2026: chọn 11 hay chờ 12? Lập luận thế nào?"
> *Dò cái gì:* chọn **Nest 11 (ổn định, hiện hành)** vì 12 mới ở roadmap/Q3-2026, breaking lớn (ESM/Vitest/oxlint); cân nhắc Node LTS 24 đi kèm; chỉ thử 12 ở môi trường đánh giá. *(verify ngày phát hành 12.)*

---

## ✅ Sau Bước A
Đây là **82 câu** phủ HẾT 11 mục con của H + nhóm runtime/kiến trúc Nest. Viết lại bằng lời mình, không chép danh sách nguồn; mỗi câu có mốc chấm riêng; đã chống trùng với `QB_G_nodejs.md`.

**Việc của bạn:** lưu file này vào project (kéo vào Project files). Khi bạn muốn, tôi sẽ:
- **Bước B** — dựng giáo trình *ngược* từ bộ đề (gom câu → bài học, sắp cơ bản → nâng cao, ánh xạ Bài → ID).
- **Bước C** — giảng từng bài theo Hợp đồng 10 mục.
- **Bước D** — phỏng vấn theo đợt, chấm **live**.

> *Câu thần chú:* **"Tôi không nhớ để gõ, tôi hiểu để chỉ huy — và tra cứu phần còn lại."**
