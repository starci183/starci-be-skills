---
name: be-new-module
description: >
  Scaffold a new NestJS GraphQL/REST leaf module in starci-academy-backend that follows the
  house conventions. Use whenever creating a new query/mutation module, resolver, service, or
  feature folder. Covers the ConfigurableModuleBuilder pattern, global registration via
  .register({ isGlobal: true }), the module-root folder layout, and TypeORM EntityManager wiring.
  Pairs with be-conventions (the full style guide). Trigger on "new module", "tạo module",
  "add a query/mutation", "scaffold resolver".
---

# Scaffold a new module — starci-academy-backend

Follow this for every new leaf GraphQL module. The full rules live in **be-conventions**; this is
the actionable scaffold. Run `npm run lint` when done.

## 1. Pick the shape

| Shape | Class name | When |
|-------|-----------|------|
| Single query | `<X>SingleQueryModule` | exactly one `@Query` resolver |
| Single mutation | `<X>SingleMutationModule` | exactly one `@Mutation` resolver |
| Aggregator | `<Resource>QueriesModule` / `<Resource>MutationsModule` | bundles several leaves |

## 2. Folder layout

```
foo-bar/
├─ index.ts                     # re-export public API
├─ foo-bar.module.ts
├─ foo-bar.module-definition.ts # ConfigurableModuleBuilder + setExtras({ isGlobal })
├─ foo-bar.resolver.ts          # @Query / @Mutation only — no inline types
├─ foo-bar.service.ts           # business logic — no inline types
├─ types/        index.ts + <feature>.ts   (CreateFooParams, FooResult, *Row)
├─ enums/        index.ts + <feature>.ts
├─ classes/      index.ts + <feature>.ts
└─ graphql-types/
   ├─ inputs/        create-foo.input.ts + index.ts   (@InputType)
   └─ object-types/  foo.object.ts + index.ts         (@ObjectType)
```

Every folder has an `index.ts` barrel. NestJS files (`*.service.ts`, `*.resolver.ts`, `*.module.ts`)
contain **zero** `interface`/`type`/`enum`/`class`/exported `const` — only imports.

## 3. `.module-definition.ts`

```ts
import { ConfigurableModuleBuilder } from "@nestjs/common"

export const { ConfigurableModuleClass, MODULE_OPTIONS_TOKEN, OPTIONS_TYPE } =
    new ConfigurableModuleBuilder()
        .setExtras({ isGlobal: false }, (definition, extras) => ({
            ...definition,
            global: extras.isGlobal,
        }))
        .build()
```

## 4. `.module.ts`

```ts
import { Module } from "@nestjs/common"
import { ConfigurableModuleClass } from "./foo-bar.module-definition"
import { FooBarResolver } from "./foo-bar.resolver"
import { FooBarService } from "./foo-bar.service"

@Module({
    providers: [FooBarResolver, FooBarService],
})
export class FooBarSingleQueryModule extends ConfigurableModuleClass {}
```

No `TypeOrmModule.forFeature(...)` — the global `PrimaryPostgreSQLModule` already exposes the
EntityManager.

## 5. Register in the parent — `.register({ isGlobal: true })`

```ts
// ✅ — leaf becomes global, GraphQLSchemaBuilder picks up the resolver
imports: [
    FooBarSingleQueryModule.register({ isGlobal: true }),
]

// ❌ — leaf not global → resolver invisible to the schema
imports: [FooBarSingleQueryModule]
```

## 6. Service — EntityManager, not Repository

```ts
import { Injectable } from "@nestjs/common"
import { EntityManager } from "typeorm"
import { FooEntity, InjectPrimaryPostgreSQLEntityManager } from "@modules/databases"

@Injectable()
export class FooBarService {
    constructor(
        @InjectPrimaryPostgreSQLEntityManager()
        private readonly entityManager: EntityManager,
    ) {}

    // list enabled rows — manager works with any entity, no per-entity wiring
    async list() {
        return this.entityManager.find(FooEntity, { where: { enabled: true } })
    }
}
```

## 7. Before saving — checklist (see be-conventions §12 for the full list)

- [ ] params object for ≥2 args, destructured in signature; types named `{Action}Params` / `{Action}Result`
- [ ] no inline `{...}` in generics/casts/returns — extract to `types/` (`*Row` for raw SQL)
- [ ] `Array<T>` not `T[]`; `import type` for type-only imports
- [ ] JSDoc on every exported type/enum/class **and every member**; dense `//` why-comments in bodies
- [ ] throw `@modules/exceptions` classes, never raw `new Error(...)`
- [ ] tunables via `envConfig()`; ops-editable config in `app.yaml`
- [ ] `index.ts` barrels updated; `npm run lint` clean
