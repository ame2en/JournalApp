# Data Models

MongoDB is schemaless, so "tables" here means `@Document`-annotated Java classes under `entity/` — each maps to one collection. There are three.

## `users` collection — `entity/User.java`

```java
@Document(collection = "users")
public class User {
    @Id private ObjectId id;
    @Indexed(unique = true) @NonNull private String userName;
    private String email;
    private boolean sentimentAnalysis;
    @NonNull private String password;       // BCrypt hash, never plaintext
    @DBRef private List<JournalEntry> journalEntries = new ArrayList<>();
    private List<String> roles;             // e.g. ["USER"] or ["USER", "ADMIN"]
}
```

- `userName` has a unique index (`@Indexed(unique = true)`), enforced by Mongo because `application.yml` sets `spring.data.mongodb.auto-index-creation: true`. Two signups with the same username will fail at the DB layer.
- `password` is always a BCrypt hash — set by `UserService.saveNewUser`/`saveAdmin` via `PasswordEncoder.encode(...)` before the entity is saved. Nothing in the codebase stores a raw password.
- `sentimentAnalysis` is a per-user opt-in flag. It's the filter used by the weekly digest job (`UserRepositoryImpl.getUserForSA`) to decide who gets a mood-summary email.
- `roles` is set correctly (`["USER"]` for normal signup, `["USER","ADMIN"]` for `UserService.saveAdmin`) but — see [architecture/overview.md](../architecture/overview.md#security-filter-chain) — nothing currently *enforces* it; it's descriptive, not (yet) access-controlling.
- `journalEntries` is a `@DBRef` list, **not an embedded array**. Mongo stores each entry as a reference (collection name + `_id`), and Spring Data MongoDB resolves ("hydrates") the full `JournalEntry` documents whenever a `User` is loaded. Practically: reading a `User` always pulls all of their journal entries too (there's no lazy/paged loading), and `journalEntries` is the *only* place the app tracks "which entries belong to this user" — `JournalEntryRepository` itself has no `userId` field to query by.

## `journal_entries` collection — `entity/JournalEntry.java`

```java
@Document(collection = "journal_entries")
public class JournalEntry {
    @Id private ObjectId id;
    @NonNull private String title;
    private String content;
    private LocalDateTime date;      // set server-side to "now" on create
    private Sentiment sentiment;     // HAPPY | SAD | ANGRY | ANXIOUS — see caveat below
}
```

- `date` is set by `JournalEntryService.saveEntry(entry, userName)` at creation time (`journalEntry.setDate(LocalDateTime.now())`) — the client never supplies it.
- **`sentiment` is read but never written anywhere in this codebase.** `enums/Sentiment.java` defines the four values, `UserScheduler` reads `entry.getSentiment()` when computing the weekly digest, but no controller/service ever calls `setSentiment(...)`. Every entry created through the current API has `sentiment == null`. If you're picking up this feature, this is the gap to fill (likely: some text-classification step between entry creation and save). See [workflows/business-flows.md](../workflows/business-flows.md#weekly-sentiment-digest).
- There's no foreign key back to `User` on this side — ownership is tracked only via `User.journalEntries` (see above). Deleting a `User` would orphan their entries in `journal_entries` (no cascade delete is implemented); deleting an entry (`JournalEntryService.deleteById`) does correctly remove it from both the `User.journalEntries` list *and* the `journal_entries` collection.

## `config_journal_app` collection — `entity/ConfigJournalAppEntity.java`

```java
@Document(collection = "config_journal_app")
public class ConfigJournalAppEntity {
    private String key;
    private String value;
}
```

A generic key/value store, loaded in full into the in-process `AppCache` map on startup and every 10 minutes (see [architecture/overview.md](../architecture/overview.md#config--integration-beans)). Today it holds exactly one row in practice: key `WEATHER_API` → a URL template string with `<apiKey>`/`<city>` placeholders, used by `WeatherService`. Adding a new config-driven value means inserting a new `{key, value}` document and reading it via `AppCache.appCache.get(...)`.

## Non-Mongo stores

**Redis** — used by `RedisService` as a generic cache, currently only for weather lookups:
- Key shape: `weather_of_<city>` (e.g. `weather_of_Mumbai`).
- Value: the `WeatherResponse` object, JSON-serialized manually via Jackson before being handed to `RedisTemplate` (both key and value serializers are plain `StringRedisSerializer` — `RedisConfig.java` — so Redis just sees strings; the JSON encode/decode is `RedisService`'s job, not the template's).
- TTL: 300 seconds, set at `WeatherService.getWeather(...)`'s cache-write call site.

**Kafka** — one topic, `weekly-sentiments`:
- Producer: `UserScheduler.fetchUsersAndSendSaMail` (weekly cron), message value type `model/SentimentData.java` (`{email, sentiment}`), JSON-serialized (`spring.kafka.producer.value-serializer: JsonSerializer`).
- Consumer: `service/SentimentConsumerService.java`, `@KafkaListener(topics = "weekly-sentiments", groupId = "weekly-sentiment-group")`, deserializes back into `SentimentData` (trusted-packages restricted to `net.engineeringdigest.journalApp.model` in `application.yml`).
- If the Kafka send itself throws, `UserScheduler` falls back to sending the email directly (bypassing Kafka) in the same method — see [workflows/business-flows.md](../workflows/business-flows.md#weekly-sentiment-digest).

## Entity relationship at a glance

```
User (users) ──@DBRef list──> JournalEntry (journal_entries)
   ^ owns 0..N entries, entries have no back-reference to User

ConfigJournalAppEntity (config_journal_app)  ── loaded wholesale into AppCache (in-memory map), not joined to anything

Redis: weather_of_<city> -> WeatherResponse JSON   (5 min TTL, independent of Mongo)
Kafka: topic "weekly-sentiments"  User+JournalEntry data (via UserScheduler) -> SentimentData -> SentimentConsumerService -> email
```
