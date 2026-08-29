# Architecture Overview

## The layers

Standard Spring Boot layering, one package per layer under `net.engineeringdigest.journalApp`:

| Layer | Package | Role |
|---|---|---|
| Entry point | `JournalApplication.java` | `@SpringBootApplication`, `@EnableScheduling`, `@EnableTransactionManagement`. Also defines two `@Bean`s: a `MongoTransactionManager` (named `falana` — an odd/leftover name, but it's just the transaction manager bean) and a shared `RestTemplate`. |
| Web | `controller/` | `@RestController`s. Thin — pull the authenticated username off `SecurityContextHolder`, delegate to a service, wrap the result in a `ResponseEntity`. |
| DTO | `dto/` | Request-shape objects distinct from persistence entities (currently just `UserDTO`, used by signup). |
| Business logic | `service/` | `@Service` classes. Where validation, password hashing, cross-entity coordination (e.g. "save a journal entry AND append it to the user") and external calls (email, weather, Kafka) live. |
| Persistence | `repository/` | Spring Data MongoDB. Interfaces extending `MongoRepository<T, ObjectId>` get CRUD + derived-query methods for free (`findByUserName`, `deleteByUserName`). One repository (`UserRepositoryImpl`) is a plain class using `MongoTemplate` directly for a query Spring Data's method-name derivation can't express. |
| Domain model | `entity/` | `@Document`-annotated Mongo collections: `User`, `JournalEntry`, `ConfigJournalAppEntity`. |
| Cross-cutting | `config/`, `filter/`, `utilis/`, `cache/`, `scheduler/`, `constants/` | Security config, the JWT filter, JWT encode/decode, an in-memory config cache, cron jobs, string placeholders. |

Everything is wired with field-level `@Autowired` (not constructor injection) — the tutorial-era Spring Boot style. When you add a dependency to a class, follow the existing pattern rather than introducing constructor injection in just one place.

## Request lifecycle (a typical authenticated call)

1. Request hits the servlet container at `/journal/**` (context path from `application.yml`).
2. **`JwtFilter`** (`filter/JwtFilter.java`) runs once per request — it's registered via `http.addFilterBefore(jwtFilter, UsernamePasswordAuthenticationFilter.class)` in `SpringSecurity`. It reads the `Authorization: Bearer <token>` header, and *if* a token is present and `JwtUtil.validateToken(...)` says it isn't expired, it loads the `UserDetails` via `UserDetailsServiceImpl` and puts an authenticated `UsernamePasswordAuthenticationToken` into `SecurityContextHolder`.
3. Spring Security's filter chain continues. Because authorization is `permitAll()` (see below), the request reaches the controller regardless of whether step 2 actually authenticated anyone.
4. The `@RestController` method calls `SecurityContextHolder.getContext().getAuthentication().getName()` to get "the current user" and passes that username into a service method (e.g. `JournalEntryController.createEntry` → `journalEntryService.saveEntry(myEntry, userName)`).
5. The service loads/mutates the `User`/`JournalEntry` via a repository and returns.

## Security filter chain

`config/SpringSecurity.java` (still on the deprecated `WebSecurityConfigurerAdapter` API, matching Spring Boot 2.7):

```java
http.authorizeRequests().anyRequest().permitAll();
http.sessionManagement().sessionCreationPolicy(SessionCreationPolicy.STATELESS).and().csrf().disable();
http.addFilterBefore(jwtFilter, UsernamePasswordAuthenticationFilter.class);
```

**Read this carefully — it's the single most important gotcha in the codebase:** every request is permitted regardless of authentication state. The `JwtFilter` is the only thing that ever populates who-is-logged-in, but nothing stops a request *without* a token (or with a garbage token) from reaching any controller, including `AdminController`. There is no `hasRole("ADMIN")` check anywhere. `User.roles` (`USER`, `ADMIN`) exists and is set correctly by `UserService.saveAdmin` vs `saveNewUser`, but nothing reads it for authorization — it would need `.authorities()`/`hasRole` wiring in `SpringSecurity` plus method-level checks to actually matter.

A practical consequence: if a request has no valid JWT, `SecurityContextHolder` still has *some* authentication (Spring Security's `AnonymousAuthenticationFilter` runs later in the chain and installs an anonymous principal), so `authentication.getName()` returns `"anonymousUser"` rather than throwing immediately — but then `userService.findByUserName("anonymousUser")` returns `null`, and the very next call (e.g. `user.getJournalEntries()`) throws an unchecked `NullPointerException`, which most controllers don't catch. That's why unauthenticated calls to protected-feeling endpoints don't 401 — they blow up with a 500 (or a generic 400 in the one controller that wraps the body in try/catch).

Other security-relevant pieces:
- **Password hashing**: `BCryptPasswordEncoder`, wired both as the `@Bean PasswordEncoder` used by Spring Security's `AuthenticationManagerBuilder`, and re-instantiated directly (`new BCryptPasswordEncoder()`) inside `UserService` — two instances, functionally equivalent, just not the same object.
- **JWT** (`utilis/JwtUtil.java`, JJWT 0.12.x API): HMAC-signed, 1-hour expiry (the code comment says "5 minutes" but the math — `1000 * 60 * 60` — is 1 hour; trust the code over the comment). Signing key is a hardcoded string literal in the class, not read from config — anyone with the source can mint valid tokens.
- **Google OAuth** (`controller/GoogleAuthController.java`) is a separate, parallel login path: it exchanges an authorization `code` for a Google `id_token`, resolves the user's email, auto-provisions a `User` with a random password if one doesn't exist, and issues the app's own JWT — same downstream token format as password login.

## Config & integration beans

- `config/RedisConfig.java` — a `RedisTemplate` with `StringRedisSerializer` for both keys and values (so callers, e.g. `RedisService`, must serialize/deserialize JSON themselves via Jackson — see [data-models/collections.md](../data-models/collections.md)).
- `config/SwaggerConfig.java` — OpenAPI metadata, declares a `bearerAuth` HTTP Bearer scheme so Swagger UI can send JWTs, and lists the two Swagger servers (`8081` local / `8082` live).
- `cache/AppCache.java` — an in-process `Map<String,String>` loaded from the `config_journal_app` Mongo collection on startup (`@PostConstruct`) and refreshed every 10 minutes by `UserScheduler.clearAppCache()`. Currently holds one key, `WEATHER_API`, storing a URL template with `<apiKey>`/`<city>` placeholders (`constants/Placeholders.java`).
- `application.yml`'s Kafka block configures JSON (de)serializers and restricts trusted deserialization packages to `net.engineeringdigest.journalApp.model` — relevant if you ever add a new Kafka message type, since it must live in that package or the consumer will reject it.

## Where to start when changing something

- Adding a new authenticated endpoint? Follow the `JournalEntryController` pattern: pull the username from `SecurityContextHolder`, don't invent a new auth mechanism.
- Adding a new persisted concept? Add an `@Document` entity + a `MongoRepository` interface (see [data-models/collections.md](../data-models/collections.md)) — don't reach for `MongoTemplate` unless the query genuinely can't be expressed as a derived method (that's why `UserRepositoryImpl` exists).
- Touching security/authorization? Read the "Security filter chain" section above twice — the `permitAll()` behavior is easy to miss and easy to accidentally rely on.
