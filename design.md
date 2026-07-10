# Profiel Service — Design & Architecture

> MijnOverheid Zakelijk (MOZa) **Profiel Service** — a Quarkus/Java microservice that lets citizens and businesses manage their contact details (`contactgegevens`) and communication preferences (`voorkeuren`) in one trusted place, and lets government service providers (`dienstverleners`) retrieve up-to-date, reusable profile information via federated integrations.
>
> Owner: Ministerie van Binnenlandse Zaken (MinBZK). License: EUPL-1.2. Lifecycle: **development** (POC, targeting Logius Private Cloud).

---

## 1. Contents & Purpose

The service exposes a REST API for:

- **Partijen (parties)** — retrieving a party profile by identifier (BSN, KVK, RSIN), single and in bulk.
- **Contactgegevens (contact details)** — add/update/delete email, phone, etc. Email details trigger an external verification flow.
- **Voorkeuren (preferences)** — add/update/delete communication preferences, optionally scoped to a specific service provider/service.
- **Dienstverleners & Diensten** — register service providers and their services, used as "scopes" on contact details/preferences.
- **Email verification** — request a verification code and verify an email against an external verification service (NotifyNL), protected by a shared circuit breaker.
- **Data retention** — a scheduled job that sets a `te_verwijderen_op` (to-be-deleted-on) date on records that have been unused for a long period.

Domain language is **Dutch**; class/field names, API paths, and DB columns follow Dutch domain terms.

---

## 2. Tech Stack

| Layer | Technology |
|-------|-----------|
| Language / Runtime | **Java 21** |
| Framework | **Quarkus 3.37.0** (`quarkus-bom` platform) |
| Build | **Maven** (wrapper `./mvnw`); `maven-compiler-plugin` with MapStruct annotation processing |
| REST | `quarkus-rest` + `quarkus-rest-jackson` (JAX-RS, non-reactive) |
| REST clients | `quarkus-rest-client-jackson` + `quarkus-openapi-generator` (clients generated from OpenAPI specs) |
| Persistence | **Hibernate ORM with Panache** (active-record style), `quarkus-hibernate-envers` (auditing, currently disabled), `quarkus-hibernate-orm-panache` |
| Database | **PostgreSQL** (prod/dev, `quarkus-jdbc-postgresql`); **H2** in-memory for tests (`quarkus-jdbc-h2`) |
| Migrations | **Flyway** (`quarkus-flyway`), scripts in `src/main/resources/db/migration` |
| Object mapping | **MapStruct 1.6.3** (`PartijMapper`) |
| Validation | **Hibernate Validator 9.1.0** + Jakarta Bean Validation; custom `@ValidIdentificatieNummer` |
| API docs | **SmallRye OpenAPI** (`/openapi.json`, Swagger UI at `/docs`) |
| Errors | **RFC 9457 problem+json** via `quarkus-http-problem` (`io.quarkiverse.httpproblem`) |
| Resilience | **SmallRye Fault Tolerance** — shared circuit breaker (`VerificatieServiceGuard`) |
| Scheduling | **Quartz** (`quarkus-quartz`), JDBC-clustered store in prod, RAM in tests |
| Observability | **Micrometer + Prometheus** (`/q/metrics`), **SmallRye Health** (`/q/health`), **OpenTelemetry** tracing (pinned to 1.62.0), JSON logging (`quarkus-logging-json`) |
| Audit logging | `logboekdataverwerking-wrapper` (LDV) — "Logboek Dataverwerking" processing-activity logging to ClickHouse |
| Packaging / Deploy | **JIB** container images, **Kubernetes** manifests (`quarkus-kubernetes`, `quarkus-kubernetes-config`); native image supported via `native` profile |
| Testing | JUnit 5, Quarkus Test, Mockito, REST Assured, WireMock, Pact (provider), Atlassian swagger-request-validator, **Jazzer** (fuzzing), JaCoCo (coverage) |

External dependencies:
- **Basisprofiel API** (KVK) — generated client, `nl.rijksoverheid.moz.external.clients.basisprofiel`.
- **Verificatie service** (NotifyNL) — generated client, `nl.rijksoverheid.moz.external.clients.verificatie_service`.

---

## 3. Project Structure

```
src/main/java/nl/rijksoverheid/moz/
├── ApiVersion.java                 # Central API version constant
├── OpenApiConfig.java              # @OpenAPIDefinition metadata (NL GOV ADR compliance)
├── common/                         # Enums & constants: ContactType, IdentificatieType,
│                                   #   VoorkeurType, Taal, MediaTypes, ApiResponseDescriptions
├── controller/                     # JAX-RS resources + exception mappers
│   ├── ProfielController.java          # /partij, /contactgegeven, /voorkeur, bulk, te-verwijderen-op
│   ├── DienstverlenerController.java    # /dienstverlener, /diensten
│   ├── EmailVerificatieController.java  # /emailverificatie(/code)
│   ├── DomainExceptionMapper.java       # maps BusinessException/etc. → problem+json
│   └── DatabaseConstraintViolationMapper.java
├── dto/
│   ├── request/                    # *Request records (validated input DTOs)
│   └── response/                   # *Response records (output DTOs)
├── entity/                         # Panache entities: Partij, Identificatie, Contactgegeven,
│                                   #   Voorkeur, Dienstverlener, Dienst, DienstverlenerDienst,
│                                   #   ScopeContactgegeven, ScopeVoorkeur
├── exception/                      # BusinessException (Kind enum), AuthorizationException, TechnicalException
├── filter/                         # SecurityHeadersFilter, @RequireBody interceptor
├── helper/                         # HashHelper (SHA-256), Problems (problem+json factory)
├── job/                            # RetentieScheduler (Quartz cron job)
├── mapper/                         # PartijMapper (MapStruct)
├── services/                       # PartijService, DienstverlenerService,
│                                   #   EmailVerificatieService, VerificatieServiceGuard
└── validation/                     # @ValidIdentificatieNummer + IdentificatieNummerValidator

src/main/resources/
├── application.properties          # Config (profiles: default, %dev, %test, %prod)
├── db/migration/                   # Flyway: V1 gegevensmodel, V2 te_verwijderen_op, V3 quartz tables
└── openapi/                        # Specs consumed by openapi-generator (verificatie_service.yaml)
src/main/openapi/api_basisprofiel.yaml
src/main/docker/                    # Dockerfile.{jvm,native,native-micro,legacy-jar}

src/test/java/nl/rijksoverheid/moz/
├── controller/                     # *IntegrationTest (@QuarkusTest), OpenApiValidationTest, DomainExceptionMapperTest
├── services/                       # *ServiceTest (unit + mock)
├── contract/                       # PactProviderVerificationTest
├── fuzzing/                        # EndpointFuzzTest + standalone Jazzer fuzzers
├── job/                            # RetentieSchedulerTest
├── helper/, validation/            # unit tests
└── StandardErrorResponsesTest.java
src/test/resources/
├── application.properties          # H2 config, Flyway off, drop-and-create
├── mappings/                       # WireMock stub mappings for verificatie service
└── pacts/                          # Pact contract files
```

---

## 4. API

- **Base path**: `/api/profielservice/v1`
- **OpenAPI spec**: `/openapi.json` · **Swagger UI**: `/docs`
- **Health & metrics**: separate management port **9090** under `/q/health` and `/q/metrics` (not on the public port).
- Follows **NL GOV API Design Rules 2.1.0**; errors are **RFC 9457** (`application/problem+json`).
- API version surfaced via the `API-Version` response header (`ApiVersion.CURRENT`).

### Endpoints

**Profiel** (`ProfielController`)

| Method | Path | Purpose |
|--------|------|---------|
| POST | `/partij` | Get one party profile (body: `identificatieType`, `identificatieNummer`, optional filters) → 200 / 404 |
| POST | `/partijen/bulk` | Get multiple profiles; missing ones silently dropped → 200 / 206 / 404 |
| POST | `/contactgegeven` | Add contact detail (auto-creates party); 201 created / 200 already existed |
| PUT | `/contactgegeven` | Update type/value/scope/isDefault of a contact detail |
| DELETE | `/contactgegeven/{contactgegevenId}` | Delete a contact detail → 204 |
| PATCH | `/contactgegeven/te-verwijderen-op` | Set deletion date (dienstverlener with scope only) → 200/403/404 |
| POST | `/voorkeur` | Add/upsert a preference (one row per party+type+scope) |
| PUT | `/voorkeur` | Update a preference |
| DELETE | `/voorkeur/{voorkeurId}` | Delete a preference → 204 |
| PATCH | `/voorkeur/te-verwijderen-op` | Set deletion date (dienstverlener with scope only) |

**Dienstverlener** (`DienstverlenerController`)

| Method | Path | Purpose |
|--------|------|---------|
| GET | `/dienstverlener/{naam}` | Get provider + its services → 200 / 404 |
| POST | `/dienstverlener/` | Add provider → 201 / 409 |
| POST | `/dienstverlener/{dienstverlenerNaam}/diensten` | Add a service to a provider → 201 / 409 |

**EmailVerificatie** (`EmailVerificatieController`)

| Method | Path | Purpose |
|--------|------|---------|
| POST | `/emailverificatie/code` | (Re)request an email verification code → 200 / 404 / 503 |
| POST | `/emailverificatie` | Verify email with code → 200 / 400 |

### Cross-cutting API behavior
- `@RequireBody` interceptor rejects empty request bodies.
- `SecurityHeadersFilter` sets NL GOV / NCSC security headers (`Cache-Control: no-store`, CSP `frame-ancestors 'none'`, HSTS, `X-Content-Type-Options`, `X-Frame-Options: DENY`, `Referrer-Policy`, `API-Version`).
- CORS: explicit origin allowlist (never `*`), configured per environment.
- Exception mappers translate domain exceptions to problem+json (`DomainExceptionMapper`, `DatabaseConstraintViolationMapper`).
- Every mutating/reading operation is annotated with `@Logboek` (LDV processing-activity audit), and sensitive identifiers are SHA-256 hashed (`HashHelper`) before logging.

---

## 5. Data Model / Database Structure

The schema is defined by Flyway migrations in `src/main/resources/db/migration`:

- **V1** `init_gegevensmodel.sql` — core domain model (parties, contact details, preferences, providers, scopes).
- **V2** `add_te_verwijderen_op.sql` — retention columns + partial retention indexes.
- **V3** `add_quartz_tables.sql` — standard Quartz scheduler tables (`QRTZ_*`) for the clustered JDBC job store.

- Prod/dev: Flyway migrates at start; Hibernate `schema-management.strategy=validate`. UUID PKs default via pgcrypto `gen_random_uuid()`; naming strategy is `CamelCaseToUnderscoresNamingStrategy`.
- Tests: Flyway **off**, Hibernate `drop-and-create` builds the schema from entities on H2 (trade-off: tests don't exercise the actual SQL migrations).

### Entity–Relationship overview

```
                         ┌───────────────┐
                         │    partij     │  (root aggregate)
                         └───────┬───────┘
          ┌──────────────┬───────┴────────┬───────────────┐
          │ 1:N          │ 1:N            │ 1:N           │
          ▼              ▼                ▼               │
  ┌───────────────┐ ┌───────────────┐ ┌───────────────┐  │
  │ identificatie │ │ contactgegeven│ │   voorkeur    │  │
  │ (BSN/KVK/RSIN)│ └───────┬───────┘ └───────┬───────┘  │
  └───────────────┘         │ 1:N             │ 1:N       │
                            ▼                 ▼           │
                  ┌──────────────────┐ ┌──────────────┐  │
                  │scope_contactgegev│ │scope_voorkeur│  │
                  └─────────┬────────┘ └──────┬───────┘  │
                            │ N:1             │ N:1       │
                            └────────┬────────┘           │
                                     ▼                    │
                          ┌──────────────────────┐        │
                          │ dienstverlener_dienst │◄───────┘ (scope target)
                          └───────┬───────┬───────┘
                          N:1     │       │  N:1 (nullable → provider-wide scope)
                            ┌─────▼──┐  ┌──▼──────┐
                            │dienst- │  │ dienst  │
                            │verlener│  └─────────┘
                            └────────┘
```

### Domain tables (V1 + V2)

**partij** — root aggregate.

| Column | Type | Notes |
|--------|------|-------|
| id | uuid | PK, `gen_random_uuid()` |

**identificatie** — a party's identifier(s).

| Column | Type | Notes |
|--------|------|-------|
| id | uuid | PK |
| partij_id | uuid | FK → partij, `ON DELETE CASCADE` |
| identificatie_type | text | NOT NULL (BSN / KVK / RSIN) |
| identificatie_nummer | text | NOT NULL |
- Unique: `uk_identificatie (identificatie_type, identificatie_nummer)`. Index `idx_identificatie_partij (partij_id)`.

**contactgegeven** — contact detail (email, phone, …).

| Column | Type | Notes |
|--------|------|-------|
| id | uuid | PK |
| partij_id | uuid | FK → partij, `ON DELETE CASCADE` |
| type | text | NOT NULL (e.g. Email) |
| waarde | text | NOT NULL (email lowercased before storage) |
| is_geverifieerd | boolean | NOT NULL DEFAULT false |
| geverifieerd_at | timestamptz | nullable |
| verificatie_referentie_id | text | nullable (NotifyNL reference) |
| is_default | boolean | NOT NULL DEFAULT false |
| created_at | timestamptz | NOT NULL DEFAULT now() |
| last_updated | timestamptz | NOT NULL DEFAULT now() |
| last_used_at | timestamptz | nullable |
| te_verwijderen_op *(V2)* | timestamptz | nullable — deletion date |
| te_verwijderen_op_automatisch *(V2)* | boolean | NOT NULL DEFAULT false — set by retention scheduler |
- Unique: `uk_contactgegeven_dedup (partij_id, type, waarde)`.
- Partial unique index `contactgegeven_default_per_type (partij_id, type) WHERE is_default = true` — at most one default per type per party.
- Partial index `idx_contactgegeven_retention (last_used_at) WHERE te_verwijderen_op IS NULL` — for the retention scan.

**voorkeur** — communication preference (invariant: one row per party + type + scope).

| Column | Type | Notes |
|--------|------|-------|
| id | uuid | PK |
| partij_id | uuid | FK → partij, `ON DELETE CASCADE` |
| voorkeur_type | text | NOT NULL |
| waarde | text | NOT NULL |
| created_at | timestamptz | NOT NULL DEFAULT now() |
| last_updated | timestamptz | NOT NULL DEFAULT now() |
| last_used_at | timestamptz | nullable |
| te_verwijderen_op *(V2)* | timestamptz | nullable |
| te_verwijderen_op_automatisch *(V2)* | boolean | NOT NULL DEFAULT false |
- Index `idx_voorkeur_partij (partij_id)`; partial `idx_voorkeur_retention (last_used_at) WHERE te_verwijderen_op IS NULL`.

**dienstverlener** — service provider.

| Column | Type | Notes |
|--------|------|-------|
| id | uuid | PK |
| naam | text | NOT NULL |
| beschrijving | text | nullable |
- Case-insensitive unique index `uk_dienstverlener_naam_ci (lower(naam))`.

**dienst** — a service.

| Column | Type | Notes |
|--------|------|-------|
| id | uuid | PK |
| naam | text | NOT NULL |
| beschrijving | text | nullable |

**dienstverlener_dienst** — provider↔service join; the concrete scope target.

| Column | Type | Notes |
|--------|------|-------|
| id | uuid | PK |
| dienstverlener_id | uuid | FK → dienstverlener, `ON DELETE CASCADE` |
| dienst_id | uuid | FK → dienst, `ON DELETE SET NULL`, **nullable** (NULL = provider-wide scope) |
- Unique: `uk_dienstverlener_dienst (dienstverlener_id, dienst_id)`.
- Partial unique index `uk_dvdienst_dv_broad (dienstverlener_id) WHERE dienst_id IS NULL` — one provider-wide row per provider (NULLs aren't deduped by the plain unique constraint).
- Indexes `idx_dienstverlener_dienst_dv`, `idx_dienstverlener_dienst_dienst`.

**scope_contactgegeven** — links a contact detail to a scope.

| Column | Type | Notes |
|--------|------|-------|
| id | uuid | PK |
| contactgegeven_id | uuid | FK → contactgegeven, `ON DELETE CASCADE` |
| dienstverlener_dienst_id | uuid | FK → dienstverlener_dienst |
- Unique: `uk_scope_contactgegeven (contactgegeven_id, dienstverlener_dienst_id)`. Indexes on both FK columns.

**scope_voorkeur** — links a preference to a scope.

| Column | Type | Notes |
|--------|------|-------|
| id | uuid | PK |
| voorkeur_id | uuid | FK → voorkeur, `ON DELETE CASCADE` |
| dienstverlener_dienst_id | uuid | FK → dienstverlener_dienst |
- Unique: `uk_scope_voorkeur (voorkeur_id, dienstverlener_dienst_id)`. Indexes on both FK columns.

### Scheduler tables (V3)

Standard Quartz PostgreSQL DDL required by `quarkus-quartz` with the `jdbc-cmt`, clustered store: `QRTZ_JOB_DETAILS`, `QRTZ_TRIGGERS` (+ `SIMPLE`/`CRON`/`SIMPROP`/`BLOB` sub-tables), `QRTZ_CALENDARS`, `QRTZ_PAUSED_TRIGGER_GRPS`, `QRTZ_FIRED_TRIGGERS`, `QRTZ_SCHEDULER_STATE`, `QRTZ_LOCKS`, plus their supporting indexes. In tests the store is RAM-based, so these tables aren't used.

### Indexing rationale
PostgreSQL does not auto-index FK columns, so V1 adds explicit indexes on the hot lookup/scope-filter paths (`identificatie.partij_id`, `voorkeur.partij_id`, both scope FKs, and the `dienstverlener_dienst` FKs). `contactgegeven(partij_id)` is already covered by the `uk_contactgegeven_dedup` unique index. Partial indexes back the default-per-type invariant, the provider-wide-scope invariant, and the retention scheduler scan.

---

## 6. Key Design Decisions

- **Panache active-record** entities with static finder queries (e.g. `Partij.findByIdentificatie`).
- **Email normalization** — email `waarde` is lowercased (`Locale.ROOT`) before dedup/storage.
- **Upsert semantics** — POST `/contactgegeven` and `/voorkeur` are idempotent upserts on their natural keys; adding a new scope to an existing record returns 200.
- **Email verification lifecycle** — verification code is re-issued only when the email value actually changes; type changes away from Email clear stale verification state.
- **Default demotion ordering** — `demoteCurrentDefault` runs as a bulk JPQL update *before* mutating the entity, relying on Hibernate `FlushModeType.AUTO` to avoid tripping the partial unique index (documented in `PartijService`).
- **Shared circuit breaker** — one `VerificatieServiceGuard` guards both verification-service calls; failures on one endpoint protect the other. Configurable via `verificatie-service.circuit-breaker.*` properties.
- **Retention** — `RetentieScheduler` (cron `0 0 2 * * ?`) sets `te_verwijderen_op` = now + 6 months on records unused for ≥ 78 months (6.5 yrs); manual dienstverlener overrides validated to ≤ 7 years from the reference date.
- **Management port separation** — health/metrics/OpenAPI split so infra scrapes them cluster-internally.

---

## 7. Testing

### Approach
- **Integration tests** (`*IntegrationTest`, `@QuarkusTest`) — full Quarkus boot on H2, REST Assured drives real HTTP; external verification service mocked via **WireMock** (`quarkus-wiremock`, stubs in `src/test/resources/mappings/`) and Mockito `@InjectMock`.
- **Unit tests** (`*ServiceTest`, `helper`, `validation`) — Mockito for collaborators.
- **OpenAPI schema validation** (`OpenApiValidationTest` + Atlassian `swagger-request-validator-restassured`) — every integration request/response validated against the live `/q/openapi` spec.
- **Contract testing (Pact)** — `PactProviderVerificationTest` verifies the provider against pact files in `src/test/resources/pacts/` (`moza-profiel-service.json` is a provider self-test contract). Surefire `workingDirectory` is pinned to the module root so `@PactFolder` resolves consistently.
- **Standard error responses** (`StandardErrorResponsesTest`) — asserts problem+json conformance.
- **Fuzz testing (Jazzer)** — two flavors (see `FUZZING.md`):
  - JUnit `@FuzzTest` (`EndpointFuzzTest`) run in the normal suite with few iterations; extend duration via `-Djazzer.duration=<time>`.
  - Standalone `fuzzerTestOneInput` targets (`EndpointFuzzer`, `HashHelperFuzzer`, `JsonDeserializationFuzzer`) for **ClusterFuzzLite** continuous fuzzing (PR: 5 min, batch: 1 h daily), configured under `.clusterfuzzlite/` and `.github/workflows/cflite_*.yml`.

### Requirements / gates
- **JaCoCo coverage gate**: BUNDLE minimum **50% line** and **50% branch** coverage (`jacoco-check`, checked at `test` phase). Generated external clients (`nl/rijksoverheid/moz/external/**`) are excluded from coverage.
- Integration tests (`*IT`) are skipped by default (`skipITs=true`); enabled under the `native` profile.
- Java 21 required; `java.util.logging.manager` set to JBoss LogManager for tests.

### Commands
```bash
docker compose up -d                 # local PostgreSQL
./mvnw quarkus:dev                    # dev mode, live reload, http://localhost:8080
./mvnw verify                         # full build + tests (H2)
mvn test -Dtest=EndpointFuzzTest -Djazzer.duration=1m -Djacoco.skip=true   # extended fuzzing
mvn -Pnative verify                   # native build + integration tests
```

---

## 8. Configuration & Profiles

Config lives in `src/main/resources/application.properties` with Quarkus profiles:

- **default** — PostgreSQL, Flyway on, validate schema, CORS locked to `mijnoverheidzakelijk.nl` origins, verificatie URL a placeholder invalid host.
- **`%dev`** — verificatie service → `localhost:8089` (WireMock devservice), localhost CORS, non-JSON console logs, secrets in a gitignored `application-dev.properties`.
- **`%test`** — H2 in-memory, Flyway off + drop-and-create, scheduler off, RAM Quartz, mock NotifyNL creds.
- **`%prod`** — datasource, LDV, and NotifyNL values injected from deployed secrets; CORS via `MOZA_CORS_ORIGINS`.

Secrets (never committed): `notifynl.emailverificatie.{api-key,template-id,reference}`, datasource credentials, LDV/ClickHouse settings.

---

## 9. Build, CI & Deployment

- **CI** (`.github/workflows/maven.yml`) — on push/PR to `main`: JDK 21 (Temurin), `mvn -B package`, dependency-graph submission for Dependabot. Additional workflows: `scorecard.yml` (OpenSSF Scorecard), `cflite_{pr,cron,batch}.yml` (fuzzing).
- **Dependency hygiene** — `dependabot.yml`; `pom.xml` pins security-overrides (OpenTelemetry BOM 1.62.0 ahead of the platform, swagger-parser 2.1.40, tika-core, commons-configuration2, rhino).
- **Containerization** — JIB (`quarkus-container-image-jib`); Dockerfiles for jvm/native/native-micro/legacy-jar; `docker-compose.yml` for local PostgreSQL.
- **Kubernetes** — manifests generated by `quarkus-kubernetes`; runtime config via `quarkus-kubernetes-config`. Target platform: Logius Private Cloud (LPC).
- **Native image** — `native` Maven profile enables Quarkus native build and integration tests.

---

## 10. Governance & Compliance

- **License**: EUPL-1.2 · **publiccode.yml** describes the project (lifecycle: development).
- Community files: `CONTRIBUTING.md`, `GOVERNANCE.md`, `SUPPORT.md`, issue/PR templates (NL + EN).
- **NL GOV API Design Rules 2.1.0** compliance (URI versioning, OpenAPI contact/license, security headers).
- **RFC 9457** error format.
- **Privacy/audit**: LDV "Logboek Dataverwerking" logs each processing activity (`PS-*` activity IDs) with hashed data-subject identifiers; supports GDPR-style retention via the retention scheduler.
