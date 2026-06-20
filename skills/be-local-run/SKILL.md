---
name: be-local-run
description: >
  Run the starci-academy-backend (apps/core) locally. Use this skill whenever asked to start,
  boot, or run the backend, set up its Docker infrastructure, or debug why it won't connect at
  boot. Covers the infra compose (Postgres/Redis/Elasticsearch/Qdrant/Kafka/MinIO/NATS/Keycloak),
  the .mount config + terraform secrets, and the npm start:dev flow. Trigger on "chạy backend",
  "run backend", "docker infra", "start:dev", "boot the app", or connection errors at startup.
---

# Run the backend locally — starci-academy-backend

The Nest app (`apps/core`) runs on the **host** (`npm run start:dev`) and connects to
infrastructure running in Docker. MongoDB is **not** needed (it is in the read-policy list but
no Mongoose connection is wired).

## 1. Services the core app connects to at boot

Derived from `apps/core/src/app.module.ts` + `src/modules/env/config.ts` (defaults) and
`.env.override` (active dev overrides):

| Service | Host port | Credentials / notes |
|---------|-----------|---------------------|
| PostgreSQL 16 | `5433` | `postgres` / `Cuong123_A`, db `starci-academy`. TypeORM `synchronize=true` builds the schema on first boot. |
| Redis 7 | `6380` | pass `Cuong123_A` — covers cache + bullmq + throttler + adapter |
| Elasticsearch 8 | `9200` | `elastic` / `123456`, security on, no TLS |
| Qdrant | `6333` | api-key `Cuong123_A` |
| Kafka (KRaft) | `29092` | CDC progress-projection listener |
| MinIO | `9000` / `9001` | `minioadmin` / `minioadmin123`, bucket `starci-academy` |
| NATS (JetStream) | `4222` | token `starci@2026` |
| Keycloak 24 | `8089` | admin `admin` / `bitnami123`, realm `master` |

> Port `5433` often collides with other local Postgres containers (e.g. a `pgvector` from another
> project). `docker stop <that-container>` before bringing this up.

## 2. Infra compose

The repo ships `.docker/compose.yaml` with all eight services, ports/creds matching the env.

```bash
docker compose -f .docker/compose.yaml up -d      # start
docker compose -f .docker/compose.yaml ps          # status
docker compose -f .docker/compose.yaml down        # stop (keep volumes)
docker compose -f .docker/compose.yaml down -v     # stop + wipe data
```

Image gotchas:
- **Kafka**: use `apache/kafka:3.7.1` (KRaft, `KAFKA_*` env vars) — the `bitnami/kafka:3.7` tag was
  removed from Docker Hub.
- **Keycloak**: `quay.io/keycloak/keycloak:24.0` with `start-dev` (embedded H2 — no extra DB).
- A one-shot `minio/mc` init service creates the `starci-academy` bucket so first upload doesn't 404.

## 3. `.mount` config + secrets (NOT in git)

`.mount/config` and `.mount/terraform` are gitignored — only the maintainer has them. Without
them, InitModule seeding + Keycloak provisioning + AI will fail at boot.

- `.mount/config/` → `app.yaml`, `seed.yaml`, `metadata.json`
- `.mount/terraform/` → 26 secret `.key` files (`data-git-token.key`, `keycloak-client-secret.key`,
  `keycloak-admin.json`, `s3-secret-access-key.key`, `encryption-key.key`, AI/payment keys, …)
- `.mount/data/` → the content SSOT (large; comes from the private `data` git repo)

Transferring secrets between machines: zip `.mount`, send via PairDrop / LocalSend / cloud, then
**merge only `config` + `terraform` (+ `assets`, `screenshots`)** into the target `.mount` —
**keep the local full `.mount/data`** (transfer zips usually carry only a data subset).

## 4. Boot

```bash
npm install
npm run start:dev          # nest start --watch → port 3001
```

Verify:

```bash
curl -s -o /dev/null -w "%{http_code}\n" -X POST http://localhost:3001/graphql \
  -H "Content-Type: application/json" -d '{"query":"{ __typename }"}'   # expect 200
```

Success log line: `Nest application successfully started`, GraphQL mapped at `POST /graphql`.

## 5. Known-benign boot noise

- `kafkajs ... Topic creation errors` — CDC topics already exist / create race. Harmless; the app
  connects to Kafka and boots fine. Set `KAFKA_AUTO_CREATE_TOPICS_ENABLE: "true"` on the Kafka
  service to quiet it (optional).
- Windows long-path errors when counting `.mount/data/**/submissions` — display only, data intact.
