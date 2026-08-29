# Running & Operating This App

## Dependencies (docker-compose.yml)

```
docker compose up -d
```

Brings up:
- **mongodb** (`mongo:latest`, port 27017, named volume `mongodb_data`)
- **redis** (`redis:latest`, port 6379)
- **zookeeper** + **kafka** (Confluent images, kafka on port 9092, advertised listener `localhost:9092`)

Nothing else is containerized — the app itself runs from your machine (`mvn spring-boot:run` or your IDE), and Gmail SMTP / the weather API / Google OAuth are real external services even in local dev (or left as harmless placeholders — see below).

## Configuration: `application.yml` vs `application-local.yml`

`application.yml` is the base config. Almost every value is `${AN_ENV_VAR}` — this is the shape meant for a real deployment (CI/prod), where those env vars are injected (e.g. `MONGODB_URI`, `REDIS_HOST`, `REDIS_PASSWORD`, `KAFKA_SERVERS`, `SERVER_PORT`, `GOOGLE_CLIENT_ID`/`SECRET`, `JAVA_EMAIL`/`JAVA_EMAIL_PASSWORD`, `WEATHER_API_KEY`). Notably it also hardcodes `spring.data.mongodb.database: journaldb` and a Kafka `SASL_SSL`/`PLAIN` security setup meant for a managed Kafka broker.

`application-local.yml` is the `local` Spring profile — it overrides the above with `localhost` values matching the docker-compose services (Mongo on `27017`, Redis on `6379` with no password, Kafka `PLAINTEXT` on `9092`, disables SASL), runs the server on port `8081`, and fills weather/Google-OAuth/mail settings with placeholder values so the app boots without real credentials. **Run with the `local` profile active** (`SPRING_PROFILES_ACTIVE=local` or `--spring.profiles.active=local`) for local development against docker-compose — otherwise Spring will try to resolve the base config's env vars and fail to start (or connect to nothing).

Because `spring.data.mongodb.auto-index-creation: true`, the unique index on `User.userName` is created automatically the first time the app connects — no manual migration step.

Note: the checked-in `application-local.yml` contains what look like real-looking (but course-provided/placeholder) OAuth client id/secret values and dummy mail credentials. Treat anything in this file as non-production and don't assume it's a live secret, but also don't copy it into a real deployment's config.

## Tests

`src/test/java/...` has JUnit 5 tests for `UserService`, `UserRepositoryImpl`, `EmailService`, `RedisService` (via `RedisTests`), `UserDetailsServiceImpl`, and a Spring context load test (`JournalAppApplicationTests`). **Most are annotated `@Disabled`** — they're course scaffolding (e.g. a parameterized `test(int a, int b, int expected)` that just asserts `a + b == expected`, disabled) rather than a maintained regression suite. Run `./mvnw test` to see current state; don't assume green tests mean broad coverage, and don't assume `@Disabled` tests reflect intentionally-skipped-but-relevant coverage — read each one before re-enabling it.

## CI

`.github/workflows/build.yml` — a single GitHub Actions workflow, "SonarCloud", triggered on pushes to `master` and on PR open/sync/reopen. It runs `mvn -B verify org.sonarsource.scanner.maven:sonar-maven-plugin:sonar` against SonarCloud project `engineering-digest_journalapp` (JDK 17 in CI, even though `pom.xml` targets Java 8 — CI just needs a JDK new enough to run the build/Sonar tooling). There's no separate "run tests" job; `mvn verify` runs the test phase as part of the same command. No deployment step exists in this repo.
