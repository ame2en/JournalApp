# Business Workflows

Five end-to-end flows cover essentially everything the app does. Each is written as "request/trigger → code path → data touched."

## Signup & login

**Signup** — `POST /journal/public/signup`
`PublicController.signup` builds a `User` from the incoming `UserDTO`, then `UserService.saveNewUser`: BCrypt-hashes the password, hardcodes `roles = ["USER"]`, saves via `UserRepository`. `saveNewUser` catches any exception, logs it, and returns `false` — but `signup` is declared `void` and never inspects that return value. Net effect: a duplicate `userName` (unique index violation) fails the Mongo write, gets swallowed inside `saveNewUser`, and the client still receives a plain `200 OK` with an empty body, with no account actually created. The only trace is a server-side log line (`log.error("Error saving user: ...")`).

**Login** — `POST /journal/public/login`
`PublicController.login` calls `AuthenticationManager.authenticate(new UsernamePasswordAuthenticationToken(username, password))`. Spring Security delegates to `UserDetailsServiceImpl.loadUserByUsername` (looks up by `userName` via `UserRepository.findByUserName`) and checks the password against the stored BCrypt hash using the `PasswordEncoder` bean from `SpringSecurity`. On success, `JwtUtil.generateToken(username)` returns a signed JWT (1 hour expiry) as the response body. On any failure (bad credentials, user not found), the exception is caught and the endpoint returns `400 Bad Request` with the literal string `"Incorrect username or password"`.

**Google OAuth login** — `GET /journal/auth/google/callback?code=...`
A separate path to the same destination (a JWT). `GoogleAuthController` exchanges the `code` for a Google `id_token` (server-to-server call to `oauth2.googleapis.com/token`), extracts the email from Google's tokeninfo endpoint, and either finds an existing user by that email or provisions a new one (`roles=["USER"]`, a random UUID as the BCrypt-hashed password — the user can never log in with a password, only via Google, unless it's reset). Issues the same kind of JWT as the password flow, so downstream everything (JwtFilter, controllers) treats both login paths identically.

## Authenticated journal CRUD

`JournalEntryController` is mapped at `/journal`, and the whole app has a `/journal` context path, so every route here is actually `/journal/journal...` (e.g. create is `POST /journal/journal`, not `POST /journal` — easy to trip over when testing by hand). All of them resolve "current user" the same way: `SecurityContextHolder.getContext().getAuthentication().getName()` → `UserService.findByUserName(...)`. See [architecture/overview.md](../architecture/overview.md#security-filter-chain) for what happens when there's no valid token.

- **Create** (`POST /journal/journal`): `JournalEntryService.saveEntry(entry, userName)` — stamps `date = now()`, saves the entry to `journal_entries`, then appends it to the user's `journalEntries` list and re-saves the `User` document. Two writes, wrapped in `@Transactional`, backed by the `MongoTransactionManager` bean (`JournalApplication.falana`). **Unverified from reading alone, flagging for a live check**: MongoDB multi-document transactions require the server to be a replica set (or sharded cluster) member — a bare standalone `mongod`, which is exactly what `docker-compose.yml`'s `mongodb` service is (no `--replSet` flag, no `rs.initiate()` anywhere in this repo), rejects transactions outright with `"Transaction numbers are only allowed on a replica set member or mongos"`. If that holds, creating a journal entry against the shipped docker-compose would throw on every call, caught by the controller's `try/catch` and returned as `400 Bad Request` — i.e. the core "create an entry" flow may not work out of the box against local docker-compose. This is inferred from config, not confirmed by running the app (Docker wasn't available in this session) — run it yourself to confirm: `docker compose up -d && ./mvnw spring-boot:run -Dspring-boot.run.profiles=local`, sign up, log in, then `POST /journal/journal` with a bearer token and see whether you get a `201` or a `400`.
- **Read all** (`GET /journal/journal`): returns `user.getJournalEntries()` directly — no pagination.
- **Read one** (`GET /journal/journal/id/{myId}`): checks the entry ID is present in *this user's* `journalEntries` list before fetching it by ID — this is the ownership check (there's no `userId` field on `JournalEntry` to query by directly, see [data-models/collections.md](../data-models/collections.md)).
- **Update** (`PUT /journal/journal/id/{myId}`): same ownership check, then partially patches `title`/`content` only if the incoming value is non-null and non-empty (`content`/`title` you can't clear this way — sending `""` is treated as "no change").
- **Delete** (`DELETE /journal/journal/id/{myId}`): removes the entry from the user's `journalEntries` list first; only if that succeeds does it delete the document from `journal_entries` — so it can't delete an entry that isn't already linked to the caller.

## Weekly sentiment digest

The most cross-cutting flow — touches Mongo, the scheduler, Kafka, and email.

1. **Trigger**: `UserScheduler.fetchUsersAndSendSaMail`, cron `0 0 9 * * SUN` (every Sunday 9am server time, via `@EnableScheduling` on `JournalApplication`).
2. **Select users**: `UserRepositoryImpl.getUserForSA()` — a `MongoTemplate` query for users with a syntactically-valid `email` *and* `sentimentAnalysis == true`.
3. **Compute mood**: for each user, filter their (already-hydrated-via-`@DBRef`) `journalEntries` to the last 7 days, tally `Sentiment` values, pick the most frequent.
4. **Caveat**: because nothing in this codebase ever sets `JournalEntry.sentiment` (see [data-models/collections.md](../data-models/collections.md#journal_entries-collection--entityjournalentryjava)), this tally is currently always over a list of `null`s — `mostFrequentSentiment` stays `null` for every real user today, and step 5 never fires. The pipeline is fully wired but starved of its key input.
5. **Publish**: if a most-frequent sentiment was found, build a `SentimentData{email, sentiment}` and `kafkaTemplate.send("weekly-sentiments", email, sentimentData)`.
6. **Fallback**: if the Kafka send throws, the scheduler sends the email *directly* via `EmailService` in the same method — so Kafka being down degrades to synchronous email rather than failing the job.
7. **Consume**: `SentimentConsumerService` (`@KafkaListener` on `weekly-sentiments`, group `weekly-sentiment-group`) receives the `SentimentData` and calls `EmailService.sendEmail(...)` (Gmail SMTP, configured in `application.yml`).

A second, unrelated scheduled job lives in the same class: `UserScheduler.clearAppCache`, cron `0 0/10 * ? * *` (every 10 minutes) — just re-runs `AppCache.init()` to refresh the in-memory config map from `config_journal_app`.

## Weather lookup (cache-aside)

Triggered by `GET /journal/user` (`UserController.greeting`) calling `WeatherService.getWeather(city)` (city is hardcoded to `"Mumbai"` today — not derived from the user).

1. Check Redis for `weather_of_<city>`. Hit → return immediately (see [data-models/collections.md](../data-models/collections.md#non-mongo-stores) for the key/TTL shape).
2. Miss → build the real API URL from the `WEATHER_API` template in `AppCache` (populated from Mongo — see [architecture/overview.md](../architecture/overview.md#config--integration-beans)), substituting `<apiKey>` (from `weather.api.key` config) and `<city>`.
3. Call the external weather API via the shared `RestTemplate` bean (`JournalApplication.restTemplate()`).
4. On success, write the response into Redis with a 300s TTL, then return it.

This is the textbook cache-aside pattern — useful as the reference implementation if you add caching to another read path.
