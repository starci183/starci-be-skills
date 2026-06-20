# StarCi Backend Code Cannon

The StarCi Academy backend is a NestJS + TypeORM + CQRS monolith where **every cross-cutting concern is centralized behind one canonical mechanism** — globality lives in `AppModule`, response shape lives in one interceptor, routes live in `httpConfig()`, env lives in `parseEnv*`, cache lives in `CacheService`, errors live in `AbstractException`. Code that re-implements any of these locally is wrong by definition, no matter how correct it looks in isolation. This document is the grounded standard: every rule is extracted from real source with `(src/...:NN)` evidence. Follow it exactly when writing or reviewing.

**GOLDEN RULE:** Never re-invent a cross-cutting mechanism locally — route through the one canonical abstraction (module-definition, transform interceptor, `httpConfig()`, `EntityManager`, `CacheService`, `AbstractException`, `parseEnv*`), and let the type system, not runtime checks, catch your mistakes.

---

## 1. Module & Dependency Injection

### 1.1 Every module owns a `*.module-definition.ts` built with `ConfigurableModuleBuilder`
**MUST define a `*.module-definition.ts` using `ConfigurableModuleBuilder().setExtras({ isGlobal: false }, ...)` and export `ConfigurableModuleClass`, `MODULE_OPTIONS_TOKEN`, and `OPTIONS_TYPE`. NEVER use `@Global()`.**
Why: callers decide globality at registration time instead of baking it into the class decorator.
```ts
export const { ConfigurableModuleClass, MODULE_OPTIONS_TOKEN, OPTIONS_TYPE } =
    new ConfigurableModuleBuilder().setExtras(
        { isGlobal: false },
        (definition, extras) => ({ ...definition, global: extras.isGlobal }),
    ).build()
```
Anti-pattern: `@Global()` on the class, or `global: true` hardcoded in the module body.
Evidence: `(src/modules/ai/ai.module-definition.ts:6-14)`, `(src/modules/bussiness/bussiness.module-definition.ts:7-14)`, `(src/modules/bussiness/achievements/achievements.module-definition.ts:7-14)`.

### 1.2 `setClassMethodName("forRoot")` is reserved for singleton infrastructure
**MUST use `forRoot` ONLY for app-level singleton infra (EnvModule, BullModule); all feature/business modules MUST use the default `register()`.**
Why: `forRoot` signals one-time bootstrapping (env, pools); using it for features muddies API intent.
```ts
// bullmq.module-definition.ts
export const { ConfigurableModuleClass, MODULE_OPTIONS_TOKEN, OPTIONS_TYPE } =
    new ConfigurableModuleBuilder().setClassMethodName("forRoot").build()
```
Anti-pattern: `forRoot()` on a feature module.
Evidence: `(src/modules/env/env.module-definition.ts:8-11)`, `(src/modules/bullmq/bullmq.module-definition.ts:9-12)`.

### 1.3 Dynamic modules keep `@Module({})` empty and put metadata in the override
**MUST decorate dynamic-only modules with empty `@Module({})`, `extends ConfigurableModuleClass`, and place all providers/imports/exports in the static override return — NEVER in both places.**
Why: declaring providers in `@Module()` AND the override double-registers them.
```ts
@Module({})
export class BussinessModule extends ConfigurableModuleClass {
    static register(options: typeof OPTIONS_TYPE): DynamicModule {
        const dynamicModule = super.register(options)
        return { ...dynamicModule, imports: [...modules], exports: [...modules] }
    }
}
```
Anti-pattern: providers in both `@Module()` and the override, or omitting `extends ConfigurableModuleClass`.
Evidence: `(src/modules/bussiness/bussiness.module.ts:79-82)`, `(src/modules/init/init.module.ts:39-41)`, `(src/modules/elasticsearch/elasticsearch.module.ts:22-24)`.

### 1.4 Static overrides spread `super.register()` first, then extend
**MUST call `super.register(options)` (or `super.forRoot`) first, spread the result, then add keys. NEVER build a `DynamicModule` literal from scratch.**
Why: the base populates `module`, `global`, and the `MODULE_OPTIONS_TOKEN` provider — from-scratch literals silently drop them.
```ts
static register(options: typeof OPTIONS_TYPE): DynamicModule {
    const dynamicModule = super.register(options)
    return {
        ...dynamicModule,
        providers: [...(dynamicModule.providers ?? []), MyService],
        exports: [MyService],
    }
}
```
Anti-pattern: `return { module: MyModule, providers: [MyService], exports: [MyService] }` — loses base tokens.
Evidence: `(src/modules/elasticsearch/elasticsearch.module.ts:25-43)`, `(src/modules/bussiness/jobs/jobs.module.ts:40-90)`, `(src/modules/init/init.module.ts:42-71)`.

### 1.5 Leaf modules with a fixed provider set use `@Module({ providers, exports })` — no override
**MUST declare providers/exports directly in `@Module()` and `extends ConfigurableModuleClass` with NO static override when the provider list is compile-time fixed.**
Why: a no-op override that just re-spreads `super.register()` adds indirection with zero benefit.
```ts
@Module({
    providers: [EmailBloomFilterService],
    exports: [EmailBloomFilterService],
})
export class BloomFiltersModule extends ConfigurableModuleClass {}
```
Anti-pattern: a `register()` that calls `super.register()` then copies a never-varying provider list.
Evidence: `(src/modules/bussiness/bloom-filters/bloom-filters.module.ts:14-22)`, `(src/modules/bussiness/achievements/achievements.module.ts:26-42)`, `(src/modules/ai/balancer/ai-balancer.module.ts:31-47)`.

### 1.6 Globality is decided exclusively in `AppModule`
**Module-definitions MUST default `isGlobal: false`; `AppModule` MUST be the SOLE place passing `isGlobal: true`. Aggregators MUST forward the received `options` object unchanged to every child `register()`.**
Why: centralizing globality keeps the DI graph explicit and prevents leak when a module is reused in tests or a secondary app; an aggregator that hardcodes `isGlobal: true` overrides a non-global call-site.
```ts
// app.module.ts — the only place
ElasticsearchModule.register({ isGlobal: true })
BussinessModule.register({ isGlobal: true })

// BussinessModule.register — forward the same reference
const modules = [JobsModule.register(options), TransactionsModule.register(options)]
```
Anti-pattern: a definition defaulting `isGlobal: true`, or `UserModule.register({ isGlobal: true })` hardcoded inside a possibly-non-global aggregator.
Evidence: `(apps/core/src/app.module.ts:196-519)`, `(src/modules/ai/ai.module-definition.ts:8)`, `(src/modules/bussiness/bussiness.module.ts:86-127)`.

### 1.7 Inject a polymorphic service set via a token + `useFactory` + `inject`
**MUST collect same-type sibling services (e.g. badge strategies) under a string/symbol token wired with `useFactory` injecting each by class token. NEVER assemble the array in a constructor or via `MODULE_OPTIONS_TOKEN`.**
Why: NestJS cannot natively inject an array of siblings; the factory keeps each strategy independently injectable while exposing the collected array under one token.
```ts
{
    provide: ACHIEVEMENT_BADGES,
    useFactory: (...badges: Array<AbstractBadge>) => badges,
    inject: ACHIEVEMENT_BADGE_PROVIDERS,
}
```
Anti-pattern: constructor-spreading concrete services or building the array inside the service.
Evidence: `(src/modules/bussiness/achievements/achievements.module.ts:28-35)`.

---

## 2. GraphQL (Apollo)

### 2.1 Every response is a class extending `AbstractGraphQLResponse`
**MUST define each Query/Mutation response `@ObjectType()` as a concrete class that `extends AbstractGraphQLResponse implements IAbstractGraphQLResponse<T>`. NEVER return a raw entity or ad-hoc object.**
Why: the base supplies the mandatory `{ success, message, error }` envelope that `GraphQLTransformInterceptor` injects; the interface binds the generic payload to `data` at compile time.
```ts
@ObjectType()
export class UserXpResponse
    extends AbstractGraphQLResponse
    implements IAbstractGraphQLResponse<UserXpData> {
    @Field(() => UserXpData, { nullable: true })
    data: UserXpData
}
```
Anti-pattern: returning a raw entity or `{ success, data }` literal.
Evidence: `(src/features/api/core/graphql/queries/users/user-xp/graphql-types/response.ts:77-88)`, `(src/features/api/core/graphql/mutations/challenge-submissions/submit-challenge-submission/graphql-types/response.ts:26-34)`.

### 2.2 Pair `@GraphQLSuccessMessage` + `@UseInterceptors(GraphQLTransformInterceptor)` on every operation
**MUST decorate every `@Query`/`@Mutation` method with both `@GraphQLSuccessMessage({ [Locale.En]: ..., [Locale.Vi]: ... })` AND `@UseInterceptors(GraphQLTransformInterceptor)`, on the method (never the class).**
Why: the interceptor reads the message metadata via Reflector; without the decorator i18n is lost (falls back to `"Success"`), without the interceptor the result is returned unwrapped.
```ts
@GraphQLSuccessMessage({
    [Locale.En]: 'User XP fetched successfully',
    [Locale.Vi]: 'Lấy điểm kinh nghiệm thành công',
})
@UseInterceptors(GraphQLTransformInterceptor)
@Query(() => UserXpResponse, { name: 'userXp' })
async execute(...): Promise<UserXpResult>
```
Anti-pattern: interceptor without message decorator, or message decorator on the class.
Evidence: `(src/features/api/core/graphql/queries/users/user-xp/user-xp.resolver.ts:53-64)`, `(src/modules/api/apollo/server/interceptors/graphql-transform.interceptor.ts:60-68)`.

### 2.3 The single resolver handler is always named `execute()`
**MUST name the one public resolver method `execute()`; the GraphQL operation name comes from the `name:` option, not the method name.**
Why: decouples the TS identifier from the schema name and signals exactly one handler per resolver class.
```ts
@Query(() => UserXpResponse, { name: 'userXp', description: '...' })
async execute(@Args('userId', { type: () => ID }) userId: string): Promise<UserXpResult>
```
Anti-pattern: `getUserXp()` / `query()` relying on NestJS name inference.
Evidence: `(src/features/api/core/graphql/queries/users/user-xp/user-xp.resolver.ts:65)`, `(src/features/api/core/graphql/queries/users/user-following/user-following.resolver.ts:69)`.

### 2.4 Throttle every resolver with an explicit tier
**MUST apply `@UseThrottler(ThrottlerConfig.<Soft|Medium|Strict>)` on every resolver method, choosing the tier by operation sensitivity. Order matters — place it before `@UseInterceptors`.**
Why: all public GraphQL endpoints are unauthenticated or optional-auth; the tier is the explicit risk classification.
```ts
@UseThrottler(ThrottlerConfig.Soft)
@UseGuards(KeycloakOptionalAuthGraphQLGuard)
@GraphQLSuccessMessage({...})
@UseInterceptors(GraphQLTransformInterceptor)
@Query(() => UserProfileResponse, { name: 'userProfile' })
```
Anti-pattern: omitting `@UseThrottler`, or placing it after `@UseInterceptors`.
Evidence: `(src/features/api/core/graphql/queries/users/user-xp/user-xp.resolver.ts:50)`, `(src/features/api/core/graphql/mutations/two-factor/setup-two-factor/setup-two-factor.resolver.ts:58)`.

### 2.5 Pagination always extends the shared abstract bases
**MUST extend `PaginationPageFilters<T>` / `PaginationCursorFilters` for inputs and `PaginationPageResponseData` / `PaginationCursorResponseData` for response data. NEVER hand-roll `pageNumber`, `limit`, `cursor`, or `count`.**
Why: the shared bases keep field names/descriptions/defaults uniform so `ValidateService`/`PaginateService` operate without coupling to each feature.
```ts
@InputType()
export class ChallengesRequestPaginationFilters extends PaginationPageFilters<ChallengesSortBy> {
    @Field(() => [ChallengesRequestSort], { defaultValue: [...] })
    sorts: Array<ChallengesRequestSort>
}
```
Anti-pattern: declaring `pageNumber`/`limit`/`cursor` directly on a feature input/object type.
Evidence: `(src/features/api/core/graphql/queries/challenges/challenges/graphql-types/request.ts:63-77)`, `(src/modules/api/apollo/server/graphql-types/inputs/pagination-page.ts:11-37)`.

### 2.6 Inject locale via `@GraphQLLocale()`
**MUST inject request locale with the `@GraphQLLocale()` parameter decorator. NEVER read it from the raw request or `GqlExecutionContext` inside a resolver.**
Why: the decorator calls `resolveLocale(context)` centrally, keeping resolution in one place aligned with the interceptor.
```ts
async execute(
    @KeycloakGraphQLUser() user: UserEntity,
    @Args('request') request: SubmitChallengeSubmissionRequest,
    @GraphQLLocale() locale: Locale,
): Promise<SubmitChallengeSubmissionResponseData>
```
Anti-pattern: `GqlExecutionContext.create(context).getContext().req.headers['accept-language']`.
Evidence: `(src/features/api/core/graphql/mutations/challenge-submissions/submit-challenge-submission/submit-challenge-submission.resolver.ts:78)`, `(src/modules/api/apollo/server/decorators/locale.decorators.ts:12-15)`.

### 2.7 Every `@ObjectType`/`@InputType`/`@Field` carries a description
**MUST supply a description string on every type and field decorator. No decorator may be left undescribed.**
Why: descriptions are the public schema's living docs surfaced by introspection/codegen.
```ts
@ObjectType({ description: "A user's XP aggregate (per-source XP + total/reward balances)." })
export class UserXpData {
    @Field(() => Int, { description: 'Total XP earned from passed challenges.' })
    challengeXp: number
}
```
Anti-pattern: `@Field(() => Int)` or `@ObjectType()` with no description.
Evidence: `(src/features/api/core/graphql/queries/users/user-xp/graphql-types/response.ts:15-66)`, `(src/features/api/core/graphql/shared/discussion/object-types/reaction-summary.object.ts:12-91)`.

### 2.8 Enums reach GraphQL only via `createEnumType` + `registerEnumType`
**MUST wrap every TS enum with `createEnumType(MyEnum)` to produce a `GraphQLTypeX` token, then call `registerEnumType(GraphQLTypeX, { name, description, valuesMap })`. NEVER pass the raw enum to `@Field(() => MyEnum)`.**
Why: code-first needs a registered token (not the raw enum); `createEnumType` lowercases keys and `registerEnumType` attaches per-value descriptions for introspection.
```ts
export const GraphQLTypeChallengesSortBy = createEnumType(ChallengesSortBy)
registerEnumType(GraphQLTypeChallengesSortBy, {
    name: 'ChallengesSortBy',
    description: '...',
    valuesMap: { [ChallengesSortBy.Title]: { description: 'Sort by title' } },
})
@Field(() => GraphQLTypeChallengesSortBy)
by: ChallengesSortBy
```
Anti-pattern: `@Field(() => ChallengesSortBy)` — skips registration and loses descriptions.
Evidence: `(src/features/api/core/graphql/queries/challenges/challenges/graphql-types/request.ts:24-45)`, `(src/modules/databases/postgresql/primary/enums/ai-mode.ts:28-47)`, `(src/modules/common/utils/enum.ts:1-12)`.

---

## 3. REST (Controllers, DTOs, Guards)

### 3.1 Wrap every success handler with `RestTransformInterceptor` + `@RestSuccessMessage`
**MUST apply `@UseInterceptors(RestTransformInterceptor)` and `@RestSuccessMessage('...')` on every handler returning a success payload. NEVER return a raw object unwrapped.**
Why: the interceptor is the single place shaping `{ success, message, data, error? }` and normalizing errors into `HttpException` with the same envelope.
```ts
@RestSuccessMessage("User tokens after successful login.")
@UseInterceptors(RestTransformInterceptor)
@Post(httpConfig().keycloak().auth().login().path)
async login(@Body() body: KeycloakLoginRequest): Promise<KeycloakAuthResponse> { ... }
```
Anti-pattern: returning a plain object without the interceptor, or omitting the message (empty `message`).
Evidence: `(src/features/api/core/http/keycloak/auth/auth.controller.ts:38-50)`, `(src/modules/api/rest/interceptors/rest-transform.interceptor.ts:39-41)`.

### 3.2 Controller declared object-form with `version: '1'`
**MUST declare `@Controller({ path: httpConfig().<domain>().tags, version: '1' })`. NEVER omit the version or use a bare string path.**
Why: URI versioning prefixes routes with `/v1/`; pinning the string keeps the URL stable against global-default drift.
```ts
@Controller({ path: httpConfig().payos().tags, version: "1" })
```
Anti-pattern: `@Controller('payos')`.
Evidence: `(src/features/api/core/http/payos/create-payment-link/create-payment-link.controller.ts:34-39)`, `(src/features/api/core/http/keycloak/auth/auth.controller.ts:29-32)`.

### 3.3 Route paths and `@ApiTags` come only from `httpConfig()`
**MUST source controller path, method sub-path, and `@ApiTags` from the typed `httpConfig()` factory. NEVER hardcode route-segment string literals.**
Why: one typed factory (`src/features/api/core/http/http.ts`) lets paths be renamed in one place; scattered literals drift and typo.
```ts
@ApiTags(httpConfig().admin().tags)
@Controller({ path: httpConfig().admin().tags, version: "1" })
...
@Post(httpConfig().admin().presignedUrl().path)
```
Anti-pattern: `@Controller('admin') ... @Post('presigned-url')`.
Evidence: `(src/features/api/core/http/admin/presigned-url/presigned-url.controller.ts:33-37,59)`, `(src/features/api/core/http/http.ts:1-166)`.

### 3.4 Every DTO field pairs class-validator with `@ApiProperty`
**MUST pair every request-DTO field with class-validator decorators (`@IsString`, `@IsNotEmpty`, `@IsOptional`, ...) AND `@ApiProperty`/`@ApiPropertyOptional`. Neither may be omitted.**
Why: class-validator drives the global `ValidationPipe`; `@ApiProperty` drives Swagger. Dropping either silently breaks validation or hides the field.
```ts
@ApiProperty({ description: "...", example: "..." })
@IsString()
@IsNotEmpty()
key: string

@ApiPropertyOptional({ description: "...", example: "..." })
@IsString()
@IsOptional()
contentType?: string
```
Anti-pattern: a field with only `@ApiProperty` and no validators, or vice versa.
Evidence: `(src/features/api/core/http/admin/presigned-url/dtos/presigned-url.request.ts:15-29)`, `(src/features/api/core/http/payos/create-payment-link/dtos/request.ts:14-61)`.

### 3.5 Response DTO extends `AbstractRestResponse<TData>` and re-`declare`s `data`
**MUST extend `AbstractRestResponse<TData>` and redeclare `data` with `@ApiProperty` via `declare data: TData`. NEVER return the raw data interface from a controller.**
Why: the base supplies the universal `{ success, message, data?, error? }` Swagger envelope; `declare` narrows the generic to a concrete Swagger type without emitting the field twice.
```ts
export class CreatePaymentLinkResponse
    extends AbstractRestResponse<CreatePaymentLinkResponseData> {
    @ApiProperty({ type: CreatePaymentLinkResponseData })
    declare data: CreatePaymentLinkResponseData
}
```
Anti-pattern: returning a plain interface, or extending the base without re-declaring `data` (Swagger sees generic `unknown`).
Evidence: `(src/features/api/core/http/payos/create-payment-link/dtos/response.ts:101-109)`, `(src/modules/api/rest/dtos/abstracts.ts:16-41)`.

### 3.6 Auth guards use an abstract base with a `getRequest()` template method
**MUST implement auth guards by extending an abstract base that owns `canActivate` and defines `protected abstract getRequest(context)` per transport. NEVER duplicate JWT verification per transport.**
Why: splits the invariant (token verify, user upsert, session check) from the variant (HTTP vs GQL extraction) so every transport runs identical security checks.
```ts
protected abstract getRequest(context: ExecutionContext): KeycloakAuthGuardRequest
// REST
protected getRequest(context: ExecutionContext) {
    return context.switchToHttp().getRequest<KeycloakAuthGuardRequest>()
}
// GraphQL
protected getRequest(context: ExecutionContext) {
    return GqlExecutionContext.create(context).getContext<{ req?: ... }>().req
}
```
Anti-pattern: one guard with `if/else` on `context.getType()` mixing JWT logic with extraction.
Evidence: `(src/modules/keycloak/guards/abstract.ts:36-116)`, `(src/modules/keycloak/guards/keycloak-auth-rest.guard.ts:31-53)`, `(src/modules/keycloak/guards/keycloak-auth-graphql.guard.ts:35-63)`.

### 3.7 Controller → thin Service → CQRS Command
**MUST route the controller action through a dedicated Service whose only job is `commandBus.execute(new XCommand(params))`. NEVER put business logic or direct infra calls in the controller or service.**
Why: the Command + Handler isolates side-effects and is independently testable; the service only maps the HTTP contract to the command envelope.
```ts
// service
async execute(params: PresignedUrlCommandParams): Promise<Array<PresignedUrlItem>> {
    return this.commandBus.execute(new PresignedUrlCommand(params))
}
// controller
async presignedUrl(@Body() body: PresignedUrlRequest) {
    return this.presignedUrlService.execute({ key: body.key, contentType: body.contentType })
}
```
Anti-pattern: injecting S3/DB infra directly into the controller or service.
Evidence: `(src/features/api/core/http/admin/presigned-url/presigned-url.service.ts:14-26)`, `(src/features/api/core/http/admin/presigned-url/presigned-url.controller.ts:63-70)`.

### 3.8 Action-style `@Post` handlers pin `@HttpCode(200)`
**MUST add `@HttpCode(200)` to any `@Post` that is an action/query rather than a resource creation, matching the `@ApiResponse` status.**
Why: NestJS defaults `@Post` to 201; action endpoints (process-video, presigned-url, webhook) are lookups/jobs — 201 is semantically wrong and breaks status-checking clients.
```ts
@Post(httpConfig().admin().presignedUrl().path)
@HttpCode(200)
async presignedUrl(@Body() body: PresignedUrlRequest): Promise<...> { ... }
```
Anti-pattern: action `@Post` without `@HttpCode(200)` while Swagger documents 200.
Evidence: `(src/features/api/core/http/admin/presigned-url/presigned-url.controller.ts:59-62)`, `(src/features/api/core/http/minio/webhook/webhook.controller.ts:57)`.

---

## 4. TypeORM Data Access

### 4.1 Inject `EntityManager`, never a `Repository`
**MUST inject the `EntityManager` via `@InjectPrimaryPostgreSQLEntityManager()`. NEVER use `@InjectRepository(Entity)`.**
Why: a named connection (`POSTGRESQL_PRIMARY`) plus a single `EntityManager` gives access to all entities and transactions without per-entity tokens.
```ts
@InjectPrimaryPostgreSQLEntityManager()
private readonly entityManager: EntityManager
```
Anti-pattern: `@InjectRepository(SomeEntity) private readonly repo: Repository<SomeEntity>`.
Evidence: `(src/modules/databases/postgresql/primary/primary.decorators.ts:1-8)`, `(src/modules/session/session.service.ts:81)`, `(src/modules/membership/membership.service.ts:42-45)`.

### 4.2 Multi-step writes go inside `entityManager.transaction(...)`
**MUST wrap any multi-step write in `entityManager.transaction(async (entityManager) => { ... })` and use the callback-scoped manager for ALL reads/writes inside. NEVER use the outer manager in the callback.**
Why: callback operations share one Postgres transaction with automatic rollback on error.
```ts
await this.entityManager.transaction(async (entityManager) => {
    await entityManager.findOne(TransactionEntity, { where: { id } })
    await entityManager.update(TransactionEntity, { id }, { status })
})
```
Anti-pattern: sequential `save()`/`update()` outside a transaction — partial failure corrupts state.
Evidence: `(src/modules/membership/membership.service.ts:59-93)`, `(src/modules/init/seeders/coding-problems/inserts/coding-problem-insert.service.ts:60-172)`.

### 4.3 UUID-PK tables extend `UuidAbstractEntity`
**MUST extend `UuidAbstractEntity` for every UUID-PK entity. NEVER redeclare `id`, `createdAt`, or `updatedAt`.**
Why: the base centralizes the PK, `timestamptz` timestamps, and `toDto`/`toPlain` helpers; duplication diverges the schema.
```ts
@Entity('achievements')
export class AchievementEntity extends UuidAbstractEntity { ... }
```
Anti-pattern: `@PrimaryGeneratedColumn('uuid')` / `@CreateDateColumn` in a concrete class that already extends the base.
Evidence: `(src/modules/databases/postgresql/primary/entities/abstract.ts:52-65)`, `(src/modules/databases/postgresql/primary/entities/achievement.entity.ts:41)`.

### 4.4 Composite-key / translation tables extend `AbstractEntity` with `@PrimaryColumn`
**MUST extend `AbstractEntity` (not `UuidAbstractEntity`) for composite-PK / translation entities and declare each PK column with `@PrimaryColumn`.**
Why: translation rows are keyed by `(parentId, locale, field)`; an auto UUID PK contradicts the composite design and wastes an index.
```ts
@Entity('challenge_translations')
export class ChallengeTranslationEntity extends AbstractEntity {
    @PrimaryColumn({ name: 'challenge_id', type: 'uuid' }) challengeId: string
    @PrimaryColumn({ name: 'locale', ... }) locale: Locale
    @PrimaryColumn({ name: 'field', ... }) field: string
}
```
Anti-pattern: `UuidAbstractEntity` plus extra `@PrimaryColumn`s.
Evidence: `(src/modules/databases/postgresql/primary/entities/abstract.ts:14-47)`, `(src/modules/databases/postgresql/primary/entities/challenge-translation.entity.ts:31-69)`.

### 4.5 Projection tables extend `AbstractProjectionEntity` — aggregate goes in jsonb `value`
**MUST extend `AbstractProjectionEntity` for read-model projections; the whole aggregate goes in the inherited jsonb `value` column. NEVER add per-metric `@Column`s.**
Why: storing the snapshot in one jsonb column avoids migrations when metrics change and drives TTL lazy-refresh via inherited `updatedAt`.
```ts
@Entity('user_stats_projections')
export class UserStatsProjectionEntity extends AbstractProjectionEntity {
    @PrimaryColumn({ name: 'user_id', type: 'uuid' }) userId: string
}
```
Anti-pattern: `@Column` per metric instead of the inherited `value: Record<string, unknown>`.
Evidence: `(src/modules/databases/postgresql/primary/entities/abstract-projection.ts:28-46)`, `(src/modules/databases/postgresql/primary/entities/user-stats-projection.entity.ts:26-58)`.

### 4.6 Every `@ManyToOne` gets a `@RelationId` FK scalar
**MUST declare a `@RelationId` property right after each `@ManyToOne`. NEVER read the FK by touching the relation object.**
Why: TypeORM doesn't populate `@ManyToOne` by default; `@RelationId` projects the FK scalar with no extra join, safe to pass to other services.
```ts
@ManyToOne(() => UserEntity, { onDelete: 'CASCADE', nullable: false })
@JoinColumn({ name: 'user_id', foreignKeyConstraintName: 'fk_user_id_activities_users' })
user: UserEntity

@RelationId((activity: ActivityEntity) => activity.user)
userId: string
```
Anti-pattern: `activity.user.id` (triggers a join or is `undefined` when unloaded) instead of `activity.userId`.
Evidence: `(src/modules/databases/postgresql/primary/entities/activity.entity.ts:59-78)`, `(src/modules/databases/postgresql/primary/entities/user-follow.entity.ts:31-68)`.

### 4.7 Name every FK and unique constraint explicitly
**MUST supply `foreignKeyConstraintName` on every `@JoinColumn` (`fk_<col>_<table>_<ref_table>`) and name every `@Unique` (`UQ_<table>_<columns>`). NEVER rely on auto-generated names.**
Why: auto-generated names are unstable across TypeORM versions and environments; named constraints are required for reliable migrations, audits, and error messages.
```ts
@JoinColumn({ name: 'user_id', foreignKeyConstraintName: 'fk_user_id_user_achievements_users' })
...
@Unique('UQ_achievement_slug', ['slug'])
```
Anti-pattern: `@JoinColumn({ name: 'user_id' })` or `@Unique(['slug'])` — anonymous.
Evidence: `(src/modules/databases/postgresql/primary/entities/user-achievement.entity.ts:63-66)`, `(src/modules/databases/postgresql/primary/entities/achievement.entity.ts:38-39)`, `(src/modules/databases/postgresql/primary/entities/user-follow.entity.ts:22-28)`.

---

## 5. CQRS & Projections

### 5.1 Handlers extend `ICQRSHandler` and override only `process()`
**EVERY handler MUST extend `ICQRSHandler<TEvent, TResponse>` and implement only `protected process()`. NEVER override `execute()`.**
Why: the base centralizes async delegation and retry/tracing hooks; subclasses expose only domain logic.
```ts
export class SendMailEventHandler
    extends ICQRSHandler<SendMailEvent, void>
    implements ICommandHandler<SendMailEvent, void> {
    protected override async process(event: SendMailEvent): Promise<void> {
        await this.enqueueSendMailJobService.enqueue(event.payload)
    }
}
```
Anti-pattern: implementing `ICommandHandler` directly with an `execute(event)` method.
Evidence: `(src/modules/cqrs/icqrs-handler.ts:6-16)`, `(src/modules/cqrs/event-bus/send-mail/send-mail.handler.ts:31-46)`.

### 5.2 Event handlers bridge to a BullMQ enqueue service
**CQRS event handlers MUST delegate to a BullMQ enqueue service. NEVER run side effects (email, GitHub API, ScyllaDB write) inline.**
Why: the EventBus is in-memory and synchronous; BullMQ survives restarts and provides retry semantics the EventBus cannot.
```ts
protected override async process(event: SyncScyllaDBEvent): Promise<void> {
    const { entityType, id } = event.payload
    await this.enqueueSyncScyllaDBJobService.enqueue({ entityType, id })
}
```
Anti-pattern: calling the mailer/GitHub/ScyllaDB driver directly in `process()`.
Evidence: `(src/modules/cqrs/event-bus/sync-scylladb/sync-scylladb.handler.ts:35-41)`, `(src/modules/cqrs/event-bus/add-github-user-to-team/add-github-user-to-team.handler.ts:44-49)`.

### 5.3 Events carry a single `readonly payload`
**Event classes MUST hold all data in one `readonly payload` typed to an imported payload interface. NEVER spread flat fields on the class.**
Why: a single payload keeps the event extensible without changing handler signatures and matches the downstream BullMQ job payload type.
```ts
export class SendMailEvent {
    constructor(readonly payload: SendMailPayload) {}
}
```
Anti-pattern: `readonly to: string, readonly subject: string` directly on the class.
Evidence: `(src/modules/cqrs/event-bus/send-mail/send-mail.event.ts:5-8)`, `(src/modules/cqrs/event-bus/sync-scylladb/sync-scylladb.event.ts:7-14)`.

### 5.4 CDC projectors extend `AbstractProjectionListener`
**Every CDC projection MUST extend `AbstractProjectionListener<TTarget>` and implement only `groupId`, `topics`, `deriveTargets()`, `recomputeTarget()`. ALL Kafka plumbing lives in the base.**
Why: centralizing subscribe/run/error-swallow gives every projector identical at-least-once delivery and per-message fault isolation without copy-paste.
```ts
export abstract class AbstractProjectionListener<TTarget> implements OnModuleInit {
    protected abstract readonly groupId: string
    protected abstract readonly topics: Array<string>
    protected abstract deriveTargets(message: ProjectionCdcMessage): Promise<Array<TTarget>> | Array<TTarget>
    protected abstract recomputeTarget(target: TTarget): Promise<void>
}
```
Anti-pattern: a projector building its own KafkaJS consumer + `eachMessage` loop.
Evidence: `(src/modules/projection/abstract-projection.listener.ts:32-62)`.

### 5.5 CDC message loop swallows per-message errors
**The CDC loop MUST catch ALL per-message errors, wrap in `KafkaCdcMessageException`, log at Error, and swallow. NEVER rethrow.**
Why: rethrowing inside `eachMessage` stalls the partition consumer; since `recomputeTarget` is an idempotent UPSERT, the next message self-heals.
```ts
} catch (error) {
    const exception = new KafkaCdcMessageException({
        topic, originalError: error instanceof Error ? error : undefined,
    })
    this.logger.error(exception.message, exception.stack)
}
```
Anti-pattern: letting the error propagate out of `eachMessage`.
Evidence: `(src/modules/projection/abstract-projection.listener.ts:130-138)`.

### 5.6 Kafka startup is best-effort (non-fatal `onModuleInit`)
**`onModuleInit` in every CDC listener MUST wrap full Kafka setup in try/catch, log on failure, and NOT rethrow. Kafka unavailability MUST NOT block boot.**
Why: in local/CI the broker may be absent; blocking bootstrap on it breaks every non-Kafka test and local dev.
```ts
async onModuleInit(): Promise<void> {
    try {
        await this.projectionKafkaService.ensureTopics({ topics: this.topics })
        // subscribe + run
    } catch (error) {
        const cause = error instanceof Error ? error : new Error(String(error))
        this.logger.error(`${this.groupId} CDC listener disabled (Kafka unavailable)`, cause.stack)
    }
}
```
Anti-pattern: letting `ensureTopics`/`subscribe` errors crash DI bootstrap.
Evidence: `(src/modules/projection/abstract-projection.listener.ts:68-98)`.

### 5.7 `KafkaService` owns consumer lifecycle
**`KafkaService` MUST track every consumer it creates and disconnect all in `onModuleDestroy`. Callers MUST NOT call `consumer.disconnect()`.**
Why: centralized lifecycle guarantees clean disconnect on shutdown so the broker rebalances promptly and tests don't hang.
```ts
this.consumers.push(consumer)
...
async onModuleDestroy(): Promise<void> {
    for (const consumer of this.consumers) {
        try { await consumer.disconnect() } catch (error) { this.logger.error(...) }
    }
}
```
Anti-pattern: a projector holding and disconnecting its own consumer ad-hoc.
Evidence: `(src/modules/kafka/kafka.service.ts:43)`, `(src/modules/kafka/kafka.service.ts:148-164)`.

### 5.8 Debezium envelope unwrapped defensively
**CDC listeners MUST unwrap the Debezium envelope as `envelope.payload ?? envelope`. NEVER assume the row is always nested or always top-level.**
Why: the `ExtractNewRecordState` SMT may emit the row either way depending on connector version; the fallback handles both without code changes.
```ts
const envelope = JSON.parse(message.value.toString()) as ProjectionCdcEnvelope
const row = envelope.payload ?? envelope
```
Anti-pattern: `envelope.payload.after` or `envelope.after` with no fallback.
Evidence: `(src/modules/projection/abstract-projection.listener.ts:119-121)`, `(src/modules/projection/types/index.ts:14-20)`.

---

## 6. Type-Safety & Naming

### 6.1 Input/output interfaces are `{Action}Params` / `{Action}Result`
**MUST name every input interface `{Action}Params` and every output `{Action}Result`. NEVER use bare objects, anonymous types, or ad-hoc names.**
Why: consistent suffixes make input/output instantly recognizable and enable unambiguous `{@link}` references.
```ts
export interface AiInvokeParams { messages: Array<BaseMessage>; category?: AiModelCategory }
export interface AiInvokeResult { text: string; model: string; provider: ModelProvider; attempts: number }
```
Anti-pattern: `invoke(messages, category?): Promise<{ text: string; model: string }>`.
Evidence: `(src/modules/ai/types/ai-invoke.ts:23)`, `(src/modules/ai/utils/resolve-grading-invoke-options.ts:34)`.

### 6.2 Multi-variant data is a discriminated union of named interfaces
**MUST model variants as named interfaces (one per variant, literal discriminant), then `type`-alias the union. NEVER write inline `{ kind: 'a' } | { kind: 'b' }` on a generic or parameter.**
Why: named branches give each a JSDoc home, make exhaustive narrowing reliable, and keep generics readable.
```ts
export interface AiJobSelectionAuto    { mode: AiMode.Auto }
export interface AiJobSelectionPremium { mode: AiMode.Premium; model: string; provider: ModelProvider }
export interface AiJobSelectionByok    { mode: AiMode.Byok; model: string; provider: ModelProvider; apiKey?: string }
export type AiJobSelection = AiJobSelectionAuto | AiJobSelectionPremium | AiJobSelectionByok
```
Anti-pattern: an inline `{ mode: ... } | { mode: ... }` alias with no named branches.
Evidence: `(src/modules/ai/types/ai-job-selection.ts:8-51)`, `(src/modules/ai/balancer/types/use-api.ts:77-165)`.

### 6.3 Provider keys are branded primitives
**MUST wrap raw string API keys in `Brand<string, ProviderName>`. NEVER pass an unbranded `string` where a provider-specific key is expected.**
Why: branding makes mixing an OpenAI key into a Gemini slot a compile-time error, with zero runtime overhead.
```ts
declare const apiKeyBrand: unique symbol
export type Brand<T, B extends string> = T & { readonly [apiKeyBrand]: B }
export type OpenAiApiKey = Brand<string, "OpenAi">
export type GeminiApiKey = Brand<string, "Gemini">
```
Anti-pattern: `key: string` accepting any provider's raw key.
Evidence: `(src/modules/ai/balancer/types/use-api.ts:12-31)`, `(src/modules/ai/balancer/use-api.service.ts:376-399)`.

### 6.4 Exceptions carry typed `{Action}ExceptionMetadata`
**MUST define a `{Action}ExceptionMetadata extends AbstractExceptionMetadata` for every custom exception; the constructor MUST accept exactly that interface (destructured) and pass structured metadata to `super()`.**
Why: named metadata gives log/Sentry consumers stable keys, propagates `originalError`, and blocks unrelated fields.
```ts
export interface AllModelsExhaustedExceptionMetadata extends AbstractExceptionMetadata {
    attempts: number
    modelsTried: number
}
export class AllModelsExhaustedException extends AbstractException {
    constructor({ attempts, modelsTried, originalError }: AllModelsExhaustedExceptionMetadata) {
        super(`AI Balancer: all ${modelsTried} fallback models exhausted after ${attempts} attempts`,
            "ALL_MODELS_EXHAUSTED_EXCEPTION", { attempts, modelsTried, originalError })
    }
}
```
Anti-pattern: `extends Error` with positional args and no metadata interface.
Evidence: `(src/modules/exceptions/errors/ai/all-models-exhausted.ts:11-38)`, `(src/modules/exceptions/errors/ai/no-active-balancer-key.ts:11-40)`. (See §7.4 for the cross-cutting `AbstractException` contract.)

### 6.5 Standalone exports are arrow consts
**MUST use `export const fn = (...) => ...` for standalone utilities. Reserve `export function` ONLY for overloaded helpers.**
Why: arrow consts block hoisting, make tree-shaking explicit, and match the NestJS method style.
```ts
export const parseEnvInt = ({ key, defaultValue }: ParseEnvIntParams): number =>
    parseInt(process.env[key] ?? defaultValue.toString(), 10)
```
Anti-pattern: `export function parseEnvInt(...) { ... }` for a non-overloaded helper.
Evidence: `(src/modules/env/utils/parse-env.ts:19,30,40,52)`, `(src/modules/locale/resolve-locale.ts:16,61,89)`, `(src/modules/common/utils/computations/pow-10.ts:15-19)` (the overload exception).

### 6.6 `import type` is separate and grouped by origin
**MUST keep `import type { ... }` blocks separate from value imports, grouped: NestJS first, then `@modules/*`, then local relative.**
Why: type-only imports are erased; mixing them hides runtime cost and triggers spurious circular-dep warnings.
```ts
import { Injectable } from "@nestjs/common"
import { AiMode, ModelProvider } from "@modules/databases"
import { UseApiService } from "./balancer"
import type { UseApiActionContext } from "./balancer"
import type { AiInvokeParams, AiInvokeResult } from "./types"
```
Anti-pattern: types and values in one block.
Evidence: `(src/modules/ai/ai-invoke.service.ts:1-33)`, `(src/modules/ai/balancer/use-api.service.ts:1-45)`.

### 6.7 Every exported interface field and enum value has JSDoc
**MUST add `/** ... */` to every member of every exported `interface` and `enum`; cross-reference with `{@link ClassName.member}` where non-obvious.**
Why: IDE hover surfaces the comment; `{@link}` is TypeDoc-checked and prevents stale references.
```ts
export interface AiInvokeResult {
    /** The model response content as a string. */
    text: string
    /** The provider matching {@link AiInvokeResult.model}. */
    provider: ModelProvider
}
```
Anti-pattern: an exported interface with undocumented fields.
Evidence: `(src/modules/ai/types/ai-invoke.ts:45-54)`, `(src/modules/ai/balancer/types/key-state.ts:15-30)`, `(src/modules/databases/postgresql/primary/enums/ai-mode.ts:19-26)`.

> Note: §2.8 (GraphQL enum registration via `createEnumType` + `registerEnumType`) is the canonical rule for any DB/domain enum exposed to GraphQL — it lives in the enum's own file alongside the enum.

---

## 7. Cross-Cutting Infrastructure

### 7.1 All cache I/O goes through `CacheService` + `CacheKey` + `configMap`
**MUST route every get/set/del through `CacheService` using a `CacheKey` enum value; TTL comes from the central `configMap`. NEVER pass raw Redis strings or inline TTL numbers.**
Why: the `configMap` couples each logical key to its TTL (from `envConfig`) AND its TS result shape, giving compile-time-typed `get<K>` with no casting.
```ts
await cacheService.set({ key: CacheKey.UserEnrolledCourses, args: [userId], cacheResult: data })
// TTL pulled from configMap[CacheKey.UserEnrolledCourses].ttl
```
Anti-pattern: `redisClient.set('user:enrolled-courses:123', JSON.stringify(data), { ttl: 3600 })`.
Evidence: `(src/modules/cache/cache.service.ts:133)`, `(src/modules/cache/constants/config.ts:30-124)`, `(src/modules/cache/enums/cache-key.ts:5-28)`.

### 7.2 Del-on-write is the correctness path; TTL is only a safety net
**MUST call `cacheService.del` immediately after any mutation that changes a cached entity. NEVER rely on TTL expiry as the correctness mechanism for mutable user-scoped data.**
Why: user-scoped caches (enrollments, profile locks) must reflect mutations instantly; TTL only self-heals a missed invalidation.
```ts
async invalidateEnrolledCourses(userId: string): Promise<void> {
    await this.cacheService.del({ key: CacheKey.UserEnrolledCourses, args: [userId] })
}
```
Anti-pattern: assuming a short TTL covers an enroll/refund without an explicit `del`.
Evidence: `(src/modules/bussiness/user/user.service.ts:144-151)`, `(src/modules/bussiness/user/user.service.ts:211-217)`, `(src/modules/env/config.ts:124-134)`.

### 7.3 Cache values serialize with SuperJSON
**MUST serialize all cache values with SuperJSON. NEVER use `JSON.stringify`/`JSON.parse` for cache I/O.**
Why: cache-manager stores strings; plain JSON drops Dates, Maps, Sets, and class instances — SuperJSON preserves full type metadata.
```ts
const serialized = this.superjson.stringify(cacheResult)
await cacheManager.set(cacheKey, serialized, ttl)
return this.superjson.parse<...>(serializedCachedResult)
```
Anti-pattern: `JSON.stringify`/`JSON.parse` round-trip, silently dropping Dates.
Evidence: `(src/modules/cache/cache.service.ts:92-93)`, `(src/modules/cache/cache.service.ts:126)`, `(src/modules/event/nats/nats-message-factory.service.ts:67-71)`.

### 7.4 Custom errors extend `AbstractException` with a SCREAMING_SNAKE_CASE code
**MUST extend `AbstractException`, pass a SCREAMING_SNAKE_CASE code as the 2nd constructor arg, and define a typed `*ExceptionMetadata extends AbstractExceptionMetadata` for all debug fields. NEVER throw `new Error(...)` or ad-hoc objects.**
Why: `AbstractException` guarantees a stable `{ message, code, metadata }` envelope (`toJSON`/`fromJSON`) consumed identically by HTTP filters, BullMQ processors, and the NATS bridge.
```ts
export interface CourseNotFoundExceptionMetadata extends AbstractExceptionMetadata { id?: string }
export class CourseNotFoundException extends AbstractException {
    constructor({ id, originalError }: CourseNotFoundExceptionMetadata) {
        super('Course not found', 'COURSE_NOT_FOUND_EXCEPTION', { id, originalError })
    }
}
```
Anti-pattern: `throw new Error('Course not found')`.
Evidence: `(src/modules/exceptions/errors/abstract.ts:5-61)`, `(src/modules/exceptions/errors/courses/course-not-found.ts:9-34)`, `(src/modules/exceptions/errors/cache/not-found.ts:8-24)`.

### 7.5 All env reads go through typed `parseEnv*` helpers
**MUST read every env var through a `parseEnv*` helper (`parseEnvInt`, `parseEnvBoolean`, `parseEnvMs`, `parseEnvSecond`, `parseEnvString`, `parseEnvStringList`, `parseEnvJson`). NEVER call `process.env[key]` directly in app code.**
Why: helpers force a default at the call site, coerce to the right type, and use the `ms` library so durations stay human-readable (`'15m'`, `'3s'`). All 327 reads in `envConfig` use them.
```ts
ttl: parseEnvMs({ key: 'CACHE_TTL_KEYCLOAK_OIDC_PKCE', defaultValue: '15m' })
attempts: parseEnvInt({ key: 'BULLMQ_ATTEMPTS', defaultValue: 3 })
enabled: parseEnvBoolean({ key: 'CACHE_DEBUG_ENABLED', defaultValue: false })
```
Anti-pattern: `parseInt(process.env.REDIS_PORT || '6379')` scattered in constructors; missing defaults that crash on cold start.
Evidence: `(src/modules/env/utils/parse-env.ts:19-127)`, `(src/modules/env/config.ts:63-90)`.

### 7.6 Event transport is declared in the central event `configMap`
**MUST declare every event's `useNats`/`useLocal` in the central event `configMap`. NEVER hardcode transport decisions in producers/consumers or call `natsProducerService.publish()` directly from a feature service.**
Why: the `configMap` is the canonical record of which events need cross-pod NATS fan-out vs. single-instance local; callers may override per-emit but must not invent transport logic.
```ts
[EventName.JobStatusUpdated]: { useNats: true,  useLocal: false, eventPayload: {} as JobStatusUpdatedEventPayload }
[EventName.CommentCreated]:   { useNats: false, useLocal: true,  eventPayload: {} as CommentChangedEventPayload }
```
Anti-pattern: a feature service calling `natsProducerService.publish()` and bypassing `EventEmitterService` + configMap.
Evidence: `(src/modules/event/config.ts:16-92)`, `(src/modules/event/event-emitter.service.ts:48-76)`, `(src/modules/event/nats/nats-bridge.service.ts:106-121)`.

### 7.7 NATS messages are digest-stamped and deduplicated
**MUST stamp every outbound NATS message with a content-hash digest via `NatsMessageFactoryService`, and deduplicate inbound messages against a short-lived (default 3s) memory-cache entry.**
Why: NATS delivers at-least-once; the digest + memory-cache window prevents re-emitting the same event twice to the local EventEmitter within a delivery burst.
```ts
// producer
const serialized = natsMessageFactoryService.create({ message: payload }) // adds .digest = hash(payload)
// consumer
const cached = await cacheService.get({ key: CacheKey.NatsMessageDigest, args: [parsed.digest], cacheType: CacheType.Memory })
if (cached) continue
await cacheService.set({ key: CacheKey.NatsMessageDigest, args: [parsed.digest], cacheResult: true, cacheType: CacheType.Memory })
```
Anti-pattern: publishing raw payloads without a digest, or skipping the dedup check.
Evidence: `(src/modules/event/nats/nats-message-factory.service.ts:43-51)`, `(src/modules/event/nats/nats-bridge.service.ts:226-245)`, `(src/modules/env/config.ts:87-90)`.

### 7.8 GraphQL resolver caching uses the decorator + interceptor pair, evicting failures
**MUST gate resolver caching behind both `@GraphQLCacheResponse({ key, argsExtractor })` AND `GraphQLCacheInterceptor`, and MUST evict failed/falsy (`success: false`) responses rather than store them.**
Why: separating cache config (decorator) from execution (interceptor) lets each resolver declare its own key without the interceptor knowing domain types; evicting failures prevents poisoning the cache with error payloads.
```ts
@GraphQLCacheResponse({ key: CacheKey.EnrollmentMilestones, argsExtractor: (req, user) => [req.courseId, user?.id] })
@UseInterceptors(GraphQLCacheInterceptor)
async getEnrollmentMilestones(...) { ... }
```
Anti-pattern: manual get/set/del in the service layer, or caching `{ success: false }` results.
Evidence: `(src/modules/cache/interceptors/graphql-cache.interceptor.ts:41-43)`, `(src/modules/cache/interceptors/graphql-cache.interceptor.ts:99-107)`, `(src/modules/cache/interceptors/graphql-cache.interceptor.ts:127-135)`.

---

## AVOID — Code Smells (quick reject list)

- `@Global()` or `global: true` anywhere outside `AppModule`'s `isGlobal: true` calls.
- Building a `DynamicModule` literal instead of spreading `super.register()`.
- A leaf module with a no-op `register()` that just re-spreads providers.
- Resolver returning a raw entity / `{ success, data }` object, or a method not named `execute()`.
- `@UseInterceptors(...Transform)` without its paired `@...SuccessMessage`.
- A resolver with no `@UseThrottler`.
- `@Field(() => RawEnum)` — raw TS enum without `createEnumType` + `registerEnumType`.
- Any `@ObjectType`/`@InputType`/`@Field`/interface field/enum value missing a description/JSDoc.
- Hand-rolled `pageNumber`/`limit`/`cursor`/`count` instead of the shared pagination bases.
- Hardcoded route strings instead of `httpConfig()`; bare-string `@Controller('x')` with no `version: '1'`.
- DTO field with only `@ApiProperty` or only class-validator (never both missing).
- Returning a raw interface from a controller instead of `AbstractRestResponse<TData>` with `declare data`.
- Action `@Post` without `@HttpCode(200)`.
- `@InjectRepository(...)` instead of the `EntityManager`.
- Multi-step writes outside `entityManager.transaction(...)`, or using the outer manager inside the callback.
- Redeclaring `id`/`createdAt`/`updatedAt`, per-metric `@Column` on a projection, anonymous FK/unique constraints, or reading a FK via the relation object instead of `@RelationId`.
- A CQRS handler overriding `execute()`, running side effects inline, or spreading flat fields instead of one `payload`.
- A CDC projector with its own KafkaJS consumer, a rethrowing `eachMessage`, or a boot-blocking `onModuleInit`.
- `process.env[...]` direct reads; inline TTL numbers; `JSON.stringify` for cache; `new Error(...)` instead of `AbstractException`.
- Direct `natsProducerService.publish()` from a feature service; un-digested / un-deduped NATS messages.
- Inline `{ a } | { b }` unions, unbranded provider keys, mixed type+value import blocks, `export function` for non-overloaded utilities.

---

## How an Agent Applies This Cannon

**AUDIT mode** (reviewing existing code / a diff / a PR): Walk the changed files section-by-section against the rules above. For each violation, report the rule id (e.g. `§3.5`), the `file:line`, a severity (BLOCKER for envelope/security/transaction/globality breaks; WARN for naming/JSDoc/description gaps), and the one-line fix. Do NOT auto-edit — produce the findings list and let the human approve. Confirm cross-cutting routing (cache, env, errors, NATS) actually flows through the canonical abstraction, not a local re-implementation.

**APPLY mode** (writing new code): Build to the rule from the first keystroke — start every module from a `*.module-definition.ts`, every resolver from the `execute()` + decorator stack, every entity from the right abstract base with named constraints, every service from `EntityManager`/`CacheService`/`parseEnv*`/`AbstractException`. Before declaring done, self-audit the new file against the AVOID list and the relevant section; if any cross-cutting concern is implemented locally, replace it with the canonical mechanism. Let `tsc` (branded types, `{@link}`, declared `data`) catch what review would otherwise have to.
