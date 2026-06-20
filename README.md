<div align="center">

# ⭐ StarCi · Backend Skills

### Agent-native engineering standard for a production NestJS monorepo

[![Claude Code](https://img.shields.io/badge/Claude_Code-Plugin-d97757?logo=anthropic&logoColor=white)](https://docs.claude.com/en/docs/claude-code/plugins)
[![NestJS](https://img.shields.io/badge/NestJS-monorepo-e0234e?logo=nestjs&logoColor=white)](https://nestjs.com)
[![TypeScript](https://img.shields.io/badge/TypeScript-strict-3178c6?logo=typescript&logoColor=white)](https://www.typescriptlang.org)
[![CQRS](https://img.shields.io/badge/CQRS-+_projections-6366f1)](#)
[![License: MIT](https://img.shields.io/badge/License-MIT-22c55e)](LICENSE)

*The skills + the **Code Cannon** that keep StarCi Academy's backend consistent across 60+ modules — extracted from the real source by agents, reviewed by humans, open-sourced.*

</div>

---

## 👋 Who is StarCi?

We build at the intersection of three things — and automate everything in between:

| Pillar | What it means |
|---|---|
| 🎓 **Tech KOL & Education** | We teach Fullstack, System Design & DevOps the way they're really practiced. These are the exact skills our agents use to build the academy's backend. |
| 🤖 **Automation Corp** | Agent-first engineering. The **Code Cannon** below wasn't hand-written — it was *mined* from the live codebase by 7 parallel scan agents and synthesized by a reasoning model, every rule carrying `file:line` evidence. |
| ⛓️ **Blockchain & DeFi** | On-chain execution, market-making, high-throughput trading. The same discipline we apply to settlement logic, we apply to a module's DI graph. |

---

## 📜 The Code Cannon

[`cannon/CODE-CANNON.md`](cannon/CODE-CANNON.md) — the authoritative, **grounded** code-pattern standard for the backend. **53 strict rules** across 7 domains, **120 `(src/...:NN)` evidence refs**, zero hand-waving.

> **GOLDEN RULE** — Never re-invent a cross-cutting mechanism locally. Route through the one canonical abstraction (module-definition · transform interceptor · `httpConfig()` · `EntityManager` · `CacheService` · `AbstractException` · `parseEnv*`), and let the type system — not runtime checks — catch your mistakes.

| § | Domain | Covers |
|---|---|---|
| 1 | **Module & DI** | `ConfigurableModuleBuilder`, globality decided only in `AppModule`, dynamic-module overrides |
| 2 | **GraphQL (Apollo)** | `AbstractGraphQLResponse` envelope, resolver→service boundary, Input/ObjectType |
| 3 | **REST** | controllers, DTOs, guards, `httpConfig()` routing |
| 4 | **TypeORM data access** | `InjectEntityManager` (never `InjectRepository`), transactions |
| 5 | **CQRS & projections** | command/query handlers, projection read-models, CDC self-heal |
| 6 | **Type-safety & naming** | `export const` arrow, `{Action}Params/{Action}Result`, branded primitives, JSDoc |
| 7 | **Cross-cutting infra** | `CacheService` cache-aside, BullMQ, NATS events, exceptions, env config |

*How it was built: no theory — 7 **Sonnet** agents scanned the real `src/modules/**` in parallel, an **Opus** agent synthesized the findings. Re-runnable as the codebase evolves.*

---

## 🧩 Skills

| Skill | Use when |
|---|---|
| **`/starci-be-cannon-apply`** | Writing **new** backend code — auto-applies the Cannon from the first keystroke, self-checks before done. |
| **`/starci-be-cannon-audit`** | Auditing **existing** code / a PR / a legacy module for drift from the Cannon — reports violations with `file:line` + severity (never silently rewrites). |
| **`be-conventions`** | The everyday backend style guide — module structure, naming, params/result, JSDoc, TypeORM, env, exceptions, ESLint. |
| **`be-new-module`** | Scaffolding a new NestJS GraphQL/REST leaf module that follows the conventions. |
| **`be-local-run`** | Bringing the backend up locally — Docker infra (Postgres/Redis/ES/Qdrant/Kafka/MinIO/NATS/Keycloak), `.mount` config + secrets, `npm run start:dev`. |

---

## 🚀 Install

```bash
# inside Claude Code
/plugin marketplace add starci183/starci-be-skills
/plugin install starci-be-skills
```

Or clone into your project's `.claude/skills/` for a local copy. Read the standard directly: [`cannon/CODE-CANNON.md`](cannon/CODE-CANNON.md).

---

## 🔁 Keeping the Cannon honest

Rules are never edited in place mid-build. New conventions land in `cannon/drafts/<name>.md` (newest wins); `/merge` folds them into `CODE-CANNON.md`. Re-run the scan workflow when the architecture shifts — the Cannon stays grounded, never aspirational.

---

## 📜 License

MIT © [StarCi](https://github.com/starci183) — use them, fork them, ship faster.

<div align="center">

**Part of the StarCi skills suite:**
[fe](https://github.com/starci183/starci-fe-skills) ·
[be](https://github.com/starci183/starci-be-skills) ·
[ops](https://github.com/starci183/starci-ops-skills) ·
[content](https://github.com/starci183/starci-content-skills)

*Built by agents. Reviewed by humans. Shipped at KOL speed.*

</div>
