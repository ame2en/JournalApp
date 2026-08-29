# Journal App — Quickstart

## What this is

A backend-only journaling API built with Spring Boot. A user signs up, logs in, and gets a JWT. With that JWT they create/read/update/delete personal journal entries. In the background, a weekly scheduler looks at each user's journal entries, works out their dominant mood for the last 7 days, and emails them a summary — routed through Kafka so the "compute" and "send email" steps are decoupled. There's also a small weather lookup (Redis-cached, backed by an external weather API) surfaced in a `/user` greeting endpoint, and a Google OAuth login path as an alternative to username/password.

This is the "Journal App" tutorial project from the Engineering Digest course (see `pom.xml` description "E2EE Journal App", `SwaggerConfig` byline "By Vipul"). That matters for how you read it: some pieces are deliberately minimal/scaffolded (most tests are `@Disabled`), and a few features are half-wired (see "Known gaps" below) — don't assume everything referenced is fully implemented end to end.

## Tech stack

- **Spring Boot 2.7.16**, Java 8 target, Maven build (`pom.xml`).
- **MongoDB** (Spring Data MongoDB) — primary datastore, database name `journaldb`.
- **Redis** (Spring Data Redis) — response cache (weather lookups today).
- **Kafka** (spring-kafka) — decouples "detect weekly sentiment" from "send email".
- **Spring Security + JJWT 0.12.5** — stateless JWT auth, plus a Google OAuth2 callback flow.
- **springdoc-openapi** — Swagger UI / OpenAPI docs (`SwaggerConfig`).
- **Lombok** — `@Data`/`@Builder` on entities/DTOs.

## Running it locally

`docker-compose.yml` at the repo root brings up the three stateful dependencies: `mongodb` (27017), `redis` (6379), and `kafka`+`zookeeper` (9092). Start those first:

```
docker compose up -d
```

The app itself reads config from `src/main/resources/application.yml` (base config, values pulled from env vars like `${MONGODB_URI}`, `${REDIS_HOST}`, `${KAFKA_SERVERS}`) layered with `application-local.yml` (the `local` Spring profile — points at `localhost` for everything, runs on port `8081`). Activate the `local` profile when running so you don't need real credentials for Mongo/Redis/Kafka/Gmail/Google OAuth. See [operations/running-locally.md](operations/running-locally.md) for the full breakdown of every config value and what it's for.

All routes are served under the context path `/journal` (set in `application.yml`), so the login endpoint is really `/journal/public/login`, etc.

## Map of the codebase

- [architecture/overview.md](architecture/overview.md) — the layers (controller → service → repository), the security filter chain, and the config/bean setup. Start here to understand *how a request moves through the app*.
- [data-models/collections.md](data-models/collections.md) — the three Mongo collections, how `User` and `JournalEntry` relate, and the Redis/Kafka shapes. Start here to understand *what the app stores*.
- [workflows/business-flows.md](workflows/business-flows.md) — signup/login, journal CRUD, the weekly sentiment-digest pipeline, weather caching, Google OAuth. Start here to understand *what the app does*, end to end.
- [operations/running-locally.md](operations/running-locally.md) — env vars, docker-compose services, test suite status, CI.

## Known gaps worth knowing before you change anything

- **Security is effectively informational, not enforced.** `SpringSecurity.configure(HttpSecurity)` calls `.anyRequest().permitAll()`. The `JwtFilter` still runs and populates `SecurityContextHolder` when a valid Bearer token is present, but Spring Security itself does not block unauthenticated or wrong-role requests — including `/admin/**`. See [architecture/overview.md](architecture/overview.md#security-filter-chain).
- **Sentiment is read but never written.** `JournalEntry.sentiment` is consumed by the weekly scheduler (`UserScheduler`) but no code path in this repo ever calls `setSentiment(...)` on a journal entry. New entries are always created with a `null` sentiment. See [workflows/business-flows.md](workflows/business-flows.md#weekly-sentiment-digest).
- **JWT signing key is a hardcoded literal** in `JwtUtil.java`, not sourced from config/secrets.
- **Most test classes are `@Disabled`** — treat the test suite as scaffolding, not a safety net.
