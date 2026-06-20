# starci-be-skills

Backend skills for **StarCi Academy** — the `starci-academy-backend` NestJS/TypeScript monorepo.
Packaged as a [Claude Code plugin](https://docs.claude.com/en/docs/claude-code/plugins) so the
skills auto-load when you work on backend code.

## Skills

| Skill | Use when |
|-------|----------|
| **be-conventions** | Writing/editing any `.ts` under `src/` or `apps/` — module structure, file naming, params/result, JSDoc, TypeORM, env config, exceptions, unit tests, ESLint. The authoritative backend style guide. |
| **be-local-run** | Bringing the backend up locally — Docker infra (Postgres/Redis/ES/Qdrant/Kafka/MinIO/NATS/Keycloak), `.mount` config + secrets, `npm run start:dev`. |
| **be-new-module** | Scaffolding a new NestJS GraphQL/REST leaf module that follows the conventions (module-definition + global registration + folder layout). |

## Install

Add this repo as a plugin marketplace / install directly:

```bash
# inside Claude Code
/plugin marketplace add starci183/starci-be-skills
/plugin install starci-be-skills
```

Or clone into your project's `.claude/skills/` for a local copy.

## Canonical source

`be-conventions` mirrors `.cursor/rules/starci-academy.mdc` in the backend repo — keep both in
sync when a rule changes.

---

© StarCi Academy
