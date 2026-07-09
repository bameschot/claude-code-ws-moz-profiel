# Extending the MOZa Profiel Service with Claude

A hands-on course in **using Claude to work with an unfamiliar codebase** —
specifically, navigating and extending the MOZa Profiel Service, a Quarkus/Java
microservice for the Dutch government (MijnOverheid Zakelijk).

The service in `src/main/java/` is a real, production-shaped codebase: JAX-RS
controllers, Hibernate Panache entities, Flyway migrations, generated OpenAPI
clients, a circuit breaker, a Quartz scheduler, and a Dutch domain vocabulary
most developers have never seen. That is the point. You are not expected to
already know Quarkus, Hibernate, or the domain. The exercises here are about
developing the habits that let you use Claude productively on *any* codebase you
do not fully understand.

Each exercise has:
- a **prompting objective** — the communication skill being practised
- a **task** — what you are trying to accomplish
- a **starting prompt** — a short, natural prompt you could type right now
- **hints** — small additions that steer toward a more reliable result
- a **quality bar** — what a useful response looks like
- **iteration cues** — signals that you need a follow-up prompt

Work through these in order. Each exercise's output becomes input to the next.

---

## Attribution

The Profiel Service is developed by the **Ministerie van Binnenlandse Zaken en
Koninkrijksrelaties (MinBZK)** as part of MijnOverheid Zakelijk, under the
**EUPL-1.2** license. Docs: **[docs.mijnoverheidzakelijk.nl](https://docs.mijnoverheidzakelijk.nl)**.
The extensions you build here are your own work and inherit the same license.

---

## What You Are Working On

The Profiel Service lets citizens and businesses (`partijen`) manage their
contact details (`contactgegevens`) and communication preferences (`voorkeuren`),
and lets government service providers (`dienstverleners`) retrieve reusable
profile information. It is organised in layers — the same layering you exploit to
scope every exercise:

- **Controllers** (`controller/`) — the REST layer under `/api/profielservice/v1`:
  validate input, map outcomes to status codes and RFC 9457 `problem+json`, record
  audit entries. No business logic or DB access; they delegate to services.
  (`ProfielController`, `DienstverlenerController`, `EmailVerificatieController`.)
- **Services** (`services/`) — the domain logic: `@Transactional` beans that read
  and write entities and enforce invariants. (`PartijService`,
  `DienstverlenerService`, `EmailVerificatieService`, plus the shared circuit
  breaker `VerificatieServiceGuard`.)
- **Persistence** (`entity/` + `db/migration/`) — **Hibernate Panache** entities
  (active-record, static finders) over a **Flyway**-owned schema. In production
  Hibernate only *validates* against it, so schema and entities must always agree.
- **External clients** — the KVK Basisprofiel and NotifyNL verification services,
  *generated* from OpenAPI specs and called through the shared circuit breaker.
- **Cross-cutting** — `SecurityHeadersFilter`, the `@RequireBody` interceptor, the
  `problem+json` exception mappers, LDV audit logging (`@Logboek`), and the
  `RetentieScheduler`.

You do not need to hold this in your head — Exercise 1 has you create or update
`README.md` and `DESIGN.md`: a user-facing project guide plus a Claude-facing
architecture and conventions map that seeds every later session.

---

## Before You Start

### What is in the repository

```
src/main/java/nl/rijksoverheid/moz/   Java source — controllers, services, entities, ...
src/main/resources/db/migration/       Flyway SQL migrations (V1, V2, V3)
src/main/resources/openapi/            OpenAPI specs for generated external clients
src/test/java/                         Test suite (integration, unit, contract, fuzz)
README.md                              User-facing docs (Dutch)
quarkus.md                             Quarkus dev-mode notes
FUZZING.md                             How the fuzz tests work
pom.xml                                Maven build
```

`README.md` and `quarkus.md` give you a quick orientation. Exercise 1 updates
`README.md` where needed and creates `DESIGN.md`: the architecture, conventions,
and rules Claude should read before every future session.

### Prerequisites

- **Java 21**
- **Maven** — use the bundled wrapper `./mvnw` (no local install needed)
- **Docker** — for a local PostgreSQL via `docker compose up -d`, or rely on the
  in-memory **H2** database that the test profile uses automatically

```sh
docker compose up -d        # start local PostgreSQL
./mvnw quarkus:dev          # dev mode with live reload, http://localhost:8080
./mvnw verify               # full build + tests (uses H2)
```

With dev mode running, the Swagger UI is at `http://localhost:8080/docs` and the
OpenAPI spec at `http://localhost:8080/openapi.json`. Health and metrics live on
a separate management port: `http://localhost:9090/q/health`.

### Start with a git repository

Make an initial commit before you change anything — it is your safety net. If a
prompt breaks something, `git diff` shows exactly what changed and `git checkout .`
reverts it. Commit after each exercise passes; the history also lets you ask Claude
"what changed since the last commit?" to orient a new session.

### Ground rules

1. **You direct; Claude executes.** You do not need to read or understand the
   Java, the Hibernate mappings, or the Dutch domain terms — Claude will. Your
   job is to give it the right goal, the right constraints, and to test and
   observe the result.

2. **Test after every prompt.** Run `./mvnw verify`, or start `./mvnw quarkus:dev`
   and hit the endpoint with the Swagger UI or `curl`. What you observe is your
   main source of follow-up questions — not the code itself.

3. **One concern per prompt.** Mixing two tasks in one message produces output
   that is hard to test and harder to fix.

4. **Review large output before running it.** See *The Review Prompt* below.

5. **Keep `README.md` and `DESIGN.md` current.** See *Living Documentation* below.

6. **Commit when tests pass.** A passing commit is a safe rollback point. If the
   next prompt breaks something, `git checkout .` gets you back instantly.

7. **Use plan mode for the vertical feature.** Before a large or high-stakes
   change, switch into plan mode (Shift+Tab) so Claude designs and gets your
   approval before writing anything. Especially useful for Exercise 2, where a
   structural mistake is expensive to undo.

8. **Match the model to the task.** Use **Opus / high effort** for analysis,
   documentation, and architecture — anything whose output shapes what follows.
   Use **Sonnet / medium effort** for implementation, tests, and bug fixes: fast
   to iterate, and enough for most prompts.

### The one rule to keep

The Profiel Service has one architectural rule that is painful to fix if broken:

> **Controllers stay thin.** No database access and no business logic in the REST
> layer — all domain logic goes in a `@Transactional` service method. And PII
> (identificatienummers such as BSN/KVK/RSIN) must be hashed via `HashHelper`
> before it is written to a log.

State this rule when you ask for anything touching a controller or adding logging,
and record it in `DESIGN.md` during Exercise 1 so it applies in every future
session without repeating it.

---

## The Review Prompt

When Claude produces a large result, ask it to review its own output before you
run it — it reasons more critically when asked directly:

> "Review what you just wrote. What are the most likely problems or edge cases
> you may have missed?"

A good review surfaces at least one real issue — a missing `@Transactional`, an
unhandled 404, a validation gap — handle it before moving on. When something feels
off after testing, review the specific behaviour instead:

> "I ran it and observed [X]. Review the relevant code and explain why it might
> behave that way."

This complements running the code, it does not replace it. Use it after every
exercise that produces non-trivial output.

---

## Living Documentation

Across sessions Claude loses track of what has been built. Two committed files fix
this — open every new session with them ("Read DESIGN.md and README.md before we
continue"):

- `README.md` — for a **user**: what the service does, how to run it, what
  endpoints exist.
- `DESIGN.md` — for **Claude**: the layered architecture, key design decisions,
  what each package does, recurring conventions, and the rules to keep (such as
  the thin-controller rule).

After each exercise, ask Claude to *"update README.md and DESIGN.md to reflect what
was just added,"* then run `/compact` before the next one. Each exercise fills the
context window with code, test output, and fixes; `/compact` summarises and
discards the detail. Because the two docs are already committed and current, almost
nothing important is lost — Claude reconstructs the project state from them.

---

## Exercise 1 — Create the Living Documentation

### Prompting objective
Ask Claude to turn an unfamiliar source tree into reusable project context: a
human-facing `README.md` and a Claude-facing `DESIGN.md`.

### Task
Explore the repository and produce the two files that seed every later session:

- `README.md` — for users and developers who need to understand, run, test, and
  use the service at a high level.
- `DESIGN.md` — for Claude and future maintainers: the architecture map, package
  responsibilities, design decisions, coding conventions, recurring patterns, and
  rules that must be preserved.

If `README.md` already exists, update it rather than replacing useful existing
content. If `DESIGN.md` does not exist, create it at the repository root.

### Starting prompt

> "Explore the repository and create or update the living documentation.
>
> Update README.md so a developer can understand what the service does, how to run
> it, how to test it, where the API docs are, and what the main domain concepts
> are.
>
> Create DESIGN.md in the root of the repository. It should describe the
> architecture, package structure, API shape, tech stack, database model, testing
> approach, coding conventions, recurring implementation patterns, and the rules
> future changes must preserve."

### Hints
A useful result separates user-facing information from implementation guidance:

- Put practical usage information in `README.md`: purpose, local setup, test
  commands, Swagger/OpenAPI URLs, and a short domain/API overview.
- Put architecture and change guidance in `DESIGN.md`: layers, package map,
  dependencies, database schema, external clients, testing strategy, and rules for
  adding new features.
- For the codebase map, ask for a table: one row per package or key class,
  showing its responsibility, what it provides, and what it depends on.
- Ask for a short dependency diagram showing how the layers call into each other:
  controller → service → entity, plus external clients and cross-cutting filters.
- Ask it to cover the whole picture, not just Java: REST endpoints, base path,
  error format, Quarkus, Hibernate Panache, Flyway migrations, database tables,
  test commands, and coverage requirements.

Also ask Claude to extract the conventions that future changes must follow:

- **Controllers stay thin**: no database access and no business logic in the REST
  layer.
- **Transactions**: writes happen inside `@Transactional` service methods.
- **Errors**: RFC 9457 `problem+json`, built through the existing `Problems`
  helper.
- **Request bodies**: mutating endpoints use `@RequireBody`; request DTOs are
  Java `record`s validated with Bean Validation.
- **Auditing and PII**: operations use `@Logboek(...)`; identifiers such as
  BSN/KVK/RSIN are hashed with `HashHelper` before logging.
- **Persistence**: entities use Hibernate Panache active-record patterns and must
  match Flyway migrations because schema validation is strict.
- **Mapping**: entity-to-DTO conversion follows the existing mapper style.
- **Scopes**: document how `dienstverlener_dienst` models service-provider scope.

### Quality bar
`README.md` should let a new developer run and test the service without hunting
through the codebase.

`DESIGN.md` should be specific enough that Claude can use it as startup context in
a future session. It should name real classes, packages, tables, endpoints,
constraints, commands, and conventions from this repository, not generic Quarkus
advice.

The conventions section must include concrete examples from the existing code:
the actual annotation names, helper names, transaction style, validation style,
mapping style, and error-handling approach.

After Claude produces the files, use the review prompt:

> "Review README.md and DESIGN.md. Are they missing anything important that a new
> session would need before safely changing this service?"

### Iteration cues
- If `README.md` duplicates too much architecture detail, ask: "Move
  implementation guidance into DESIGN.md and keep README.md focused on running
  and using the service."
- If `DESIGN.md` is only a package summary, ask: "Add the recurring conventions
  and rules future changes must preserve."
- If the database section is thin, ask: "Add the database schema from the Flyway
  migrations, with tables, relationships, and key constraints."
- If the conventions are vague, ask: "For each convention, cite one concrete
  existing class, annotation, helper, or method that demonstrates it."
- Keep both files current — the vertical feature exercise should update
  `README.md` and `DESIGN.md` when behaviour, architecture, or conventions
  change.

---

## Exercise 2 — Implement a Vertical Feature Across Layers

### Prompting objective
Scope a large feature into layers you can test one at a time.

### Background
The service enforces its schema strictly: `schema-management.strategy=validate`
means Hibernate refuses to start if an entity does not match the database. So a new
domain concept is not one file — it is a Flyway migration, an entity, service
logic, a controller, and tests, and they must all agree. Building bottom-up gets
each layer right before the next depends on it.

### Task
Add a new domain concept end-to-end. For example, a `notitie` (a free-text note
attached to a partij) or a `toestemming` (a consent record). Build it bottom-up:
**migration + entity → service → controller → tests.**

### Starting prompt

> "Add the ability to attach notes to a partij. A note has text and a timestamp,
> belongs to one partij, and can be listed and added. Use the existing layering
> and conventions."

### Hints
Use plan mode (ground rule 7): "Think through how to structure this feature before
writing any code — the migration, the entity, the service methods, and the
endpoint." As part of that, ask what could go wrong at runtime — the failure modes
to design around — before requesting code; naming them up front is cheaper than
finding them in testing. Review and confirm the plan, then let Claude implement in
one pass. Two additions keep the large task manageable:

- Start at the bottom: "Begin with the Flyway migration (`V4__...`) and the entity
  only, and get the mapping to match the table exactly."
- Ask for tests before moving up: "Write a service test against H2 before adding
  the controller."

Run `./mvnw verify` at each layer before moving on, and review each layer before
running it:

> "Review the layer you just wrote. What are the most likely correctness or schema
> problems?"

When it's complete, update `README.md` and `DESIGN.md`.

### Quality bar
`./mvnw verify` passes at each layer. The service starts cleanly (proving the
migration and entity agree — a mismatch fails `validate` at boot). Adding and then
listing a note round-trips the data through the running service.

### Iteration cues
- If the app fails to start after adding the entity: "Startup fails with a schema
  validation error. What is the difference between the migration and the entity
  mapping?"
- If writes do not persist: "The note is returned but not saved. Is the service
  method `@Transactional`?"

---


## The Session Pattern

```
1. Context first    Read README.md and DESIGN.md.
2. One thing        One task or question per prompt.
3. Constraints      State what must not happen — Claude will respect them.
4. Review           Use the review prompt before running large results.
5. Test             Run ./mvnw verify or quarkus:dev and observe; describe what you see.
6. Iterate          Describe the observed symptom; ask for the cause.
7. Document         Ask Claude to update README.md and DESIGN.md.
```

### When to change approach

| What you observe | What to do |
|---|---|
| Claude changes things you did not ask about | "Why did you change [X]? Is that needed for [Y]?" |
| Same problem after two fix attempts | "Explain what the code is supposed to do here, step by step" |
| Output is too large to know where to start | Use the review prompt first, then test the areas it flags |
| Unsure whether the result is correct | "What would a test that catches a bug here look like?" |
| App fails to start after a schema change | "Does the entity mapping match the Flyway migration? `validate` is strict" |

---

## Prompts to Avoid

| Avoid | Because | Use instead |
|---|---|---|
| Asking for everything at once | Too large to test meaningfully | One layer or feature at a time |
| "Fix this" with no observation | Claude guesses at the cause | Describe what you ran and what you saw |
| "Write tests for X" alone | Produces happy-path tests only | Add "focus on edge cases and error conditions" |
| Two unrelated tasks in one prompt | Hard to test separately | One topic per prompt |
| Accepting output without reviewing | Misses predictable problems | Use the review prompt on any large result |
| Letting logic creep into a controller | Breaks the thin-controller rule | "Move this into a `@Transactional` service method" |
| Starting a new session without context | Claude starts from scratch | Always open with DESIGN.md and README.md |
