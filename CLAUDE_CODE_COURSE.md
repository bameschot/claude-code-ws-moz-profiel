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

You do not need to hold this in your head — Exercise 1 has you build `DESIGN.md`, a
map of exactly this, which then seeds every later session.

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

`README.md` and `quarkus.md` give you a quick orientation. You will not find a
`DESIGN.md` yet — producing that architecture map is Exercise 1, and from then on
it is the fastest way to orient a new session.

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

7. **Use plan mode for complex exercises.** Before a large or high-stakes change,
   switch into plan mode (Shift+Tab) so Claude designs and gets your approval
   before writing anything. Especially useful for Exercises 4 and 7, where a
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
and record it in `DESIGN.md` (Exercise 2) so it applies in every future session
without repeating it.

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
  what each package does, and the rules to keep (such as the thin-controller rule).

After each exercise, ask Claude to *"update README.md and DESIGN.md to reflect what
was just added,"* then run `/compact` before the next one. Each exercise fills the
context window with code, test output, and fixes; `/compact` summarises and
discards the detail. Because the two docs are already committed and current, almost
nothing important is lost — Claude reconstructs the project state from them.

---

## Exercise 1 — Get a Map of the Codebase

### Prompting objective
Ask Claude to turn an unfamiliar source tree into something you can navigate.

### Task
Explore the source code and give Claude an overview of the structure it can work
with: the contents, structure, API, tech stack, database model and structure, and
testing methods and requirements — captured in a `DESIGN.md` at the repo root.
There is no such document yet; you are creating it.

### Starting prompt

> "Explore the repository and describe the contents, structure, API and tech
> stack, database model and structure, and testing methods and requirements in a
> document called DESIGN.md placed in the root of the repository."

### Hints
That prompt works but produces a wall of prose. A few additions make the output
navigable and reusable:

- Ask for a **specific format**: "for the code, use a table — one row per package
  (or key class), showing its responsibility, what it provides, and what it
  depends on."
- Ask for a **dependency diagram**: "add a short diagram showing how the layers
  call into each other — controller → service → entity, plus the external
  clients and cross-cutting filters."
- Ask it to **cover the whole picture, not just the Java**: "include the REST API
  (endpoints, base path, error format), the tech stack (Quarkus, Hibernate
  Panache, Flyway, ...), the database schema from the Flyway migrations (tables,
  keys, important constraints), and how the tests are run and what the coverage
  gate requires."

### Quality bar
The output should name specific roles ("shared circuit breaker guarding the
NotifyNL verification calls", "retention scheduler that sets `te_verwijderen_op`
on stale records") not vague ones ("handles verification", "manages data"). The
database section should list real tables and constraints drawn from the Flyway
migrations, not a generic guess, and the testing section should name the actual
commands (`./mvnw verify`) and the coverage requirement. If a section feels
generic, follow up: "What does this do that nothing else does?"

### Iteration cues
- If two classes seem to overlap, ask: "What is the boundary between
  `PartijService` and `DienstverlenerService`?"
- If a section is missing or thin (say the database or testing part), ask for it
  directly: "Add the database schema from the Flyway migrations, with tables and
  key constraints."
- Keep this file — you will refer back to it throughout the project.

---

## Exercise 2 — Build a Conventions & Patterns Reference

### Prompting objective
Generate a reusable document before a large task so Claude applies consistent
choices throughout, rather than reinventing them for each new endpoint.

### Task
Create a reference that captures how *this* service is built — the recurring
patterns a new endpoint or feature must follow to fit in.

### Starting prompt

> "Create a document describing the coding conventions and recurring patterns in
> this service, so that new endpoints and features can be added consistently.
> Place it in the root of the repository."

### Hints
The starting prompt gets you a general description. Ask Claude to capture the
specific patterns that make this service consistent:

- **Errors**: RFC 9457 `problem+json`, always built with the `Problems` helper
  (`Problems.notFound(...)`, `Problems.missingBody()`), never raw exceptions.
- **Request bodies**: mutating endpoints are annotated `@RequireBody`; request
  DTOs are Java `record`s validated with Bean Validation (`@Valid`, and the custom
  `@ValidIdentificatieNummer`).
- **Auditing & PII**: every operation is annotated `@Logboek(...)` with a
  processing-activity id, and identifiers are hashed with `HashHelper` before
  logging.
- **Persistence**: entities are Panache active-record with static finders; all
  writes happen inside a `@Transactional` service method.
- **Mapping**: entity→DTO conversion goes through MapStruct (`PartijMapper`).
- **Scopes**: the `dienstverlener_dienst` join models a scope; a `null` dienst
  means the scope applies to the whole dienstverlener.

Include the thin-controller rule from *The one rule to keep* verbatim, so you can
read it into future prompts.

After getting the document, use the review prompt:

> "Review this conventions document. Is there anything important for adding a new
> endpoint to this service that is missing?"

### Quality bar
The document should show a *concrete* example of each pattern (the actual
annotation, the actual helper call), not a vague description. If you later see a
controller calling a Panache finder directly, or an error returned as a plain
`500`, the convention was not applied.

### Iteration cues
- Save this document in the repository. At the start of each subsequent session,
  ask Claude to read it: "Read the conventions document before we continue."
- If Claude makes an inconsistent choice later, point back to it: "The conventions
  doc says errors go through `Problems`. Revise to match it."

---

## Exercise 3 — Add a Read Endpoint

### Prompting objective
Describe a component by what it must do, not how to implement it.

### Background
Read endpoints are the safest place to practise behaviour-first prompting: no data
changes, so a wrong result is obvious and cheap to fix. The service already has
read endpoints (`POST /partij`, `GET /dienstverlener/{naam}`) you can point Claude
at as examples of the house style.

### Task
Add a new read-only endpoint — for example, one that lists all registered
`dienstverleners`, or one that returns a party's default contact detail per type.

### Starting prompt

> "Add a read-only endpoint that lists all registered dienstverleners with their
> services. Follow the existing conventions in this service."

### Hints
Describe the *behaviour and the contract*, and let Claude choose the
implementation:

- "It should return 200 with a JSON array, and an empty array (not a 404) when
  there are none."
- "Reuse the existing response DTO shape used by `GET /dienstverlener/{naam}` if
  one fits."

Ask Claude to confirm where the endpoint belongs before writing much: "Which
controller should this live in, and is there an existing service method I can
reuse?" This keeps the thin-controller rule intact.

After it compiles, test it: start `./mvnw quarkus:dev`, open
`http://localhost:8080/docs`, and call the new endpoint. Then update `README.md`
and `DESIGN.md`.

### Quality bar
With dev mode running, the endpoint appears in Swagger UI, returns the expected
JSON, and a not-found variant (if any) returns `application/problem+json` rather
than a stack trace. The controller method delegates to a service — it does not
query the database itself.

### Iteration cues
- If the response shape is wrong: "I called it and got [X]. I expected the same
  shape as `GET /dienstverlener/{naam}`. What differs?"
- If it returns 404 for an empty list: "An empty result should be `200 []`, not
  404. What should the endpoint do when nothing matches?"

---

## Exercise 4 — Implement a Vertical Feature Across Layers

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

## Exercise 5 — Build a Full CRUD Resource

### Prompting objective
Describe behaviour precisely so Claude does not have to guess what the client
sees.

### Background
The Profiel Service is deliberate about HTTP semantics: adding a contact detail
returns **201 Created** with a `Location` header when new, but **200 OK** when it
already existed (an idempotent upsert); deletes return **204 No Content**; missing
resources return **404** as `problem+json`; conflicting writes return **409**.
`ProfielController.addContactgegeven` is the reference for this style.

### Task
Extend the resource from Exercise 4 into full CRUD — add update and delete — with
status codes and semantics that match the rest of the service.

### Starting prompt

> "Add update and delete for notes. Match the HTTP semantics the rest of this
> service uses — look at how `ProfielController` handles contactgegeven."

### Hints
Spell out the contract precisely; this is where guessing produces subtle bugs:

- "POST is an upsert: 201 with a `Location` header when created, 200 when it
  already existed."
- "DELETE returns 204 on success and 404 `problem+json` when the note or partij
  does not exist."
- "A conflicting update returns 409 `problem+json`, using the `Problems` helper."

Test each verb and describe what you observe if something is off. Try deleting a
note that does not exist and confirm you get a `problem+json` 404, not a 500.

When it's complete, update `README.md` and `DESIGN.md`.

### Quality bar
Each verb returns the correct status code; a duplicate create returns 200 (not a
second row); a missing target returns `application/problem+json` 404. The
controller stays thin — all the branching lives in the service.

### Iteration cues
- If delete returns 500 on a missing id: "Deleting a non-existent note returns
  500. It should be a 404 `problem+json`. Where is that handled?"
- If create always returns 201: "Creating the same note twice makes two rows. It
  should upsert and return 200 the second time. What decides created-vs-existing?"

---

## Exercise 6 — A Small, Well-Scoped Addition

### Prompting objective
Write a minimal prompt for a well-scoped addition — just enough to be
unambiguous.

### Task
Add one small, optional field to the note resource you built in Exercises 4 and 5
— for example a `categorie` label — validated and mapped like the note's existing
fields. This is the "small addition to the thing you just built" step: after
decomposing a large feature, you now practise recognising when a change is genuinely
small.

### Starting prompt

> "Add an optional category label to notes."

### Hints
That terse prompt is deliberately enough. The lesson here is the opposite of
Exercise 4: when a change really is small, over-specifying the prompt just gets in
the way — state the goal and let Claude fit it to the existing note code.

One thing worth checking after: does the new field flow through every layer the
note already touches — the entity, a Flyway migration, the request/response DTOs,
the `PartijMapper`-style mapping, and validation — and is it rejected cleanly when
invalid? Remember the Exercise 4 lesson: `schema-management` is `validate`, so the
new column needs a matching migration or the app will not start.

This exercise also introduces a useful pattern for any change that touches a name
in several places: before making the change, ask **"list every place a note is
represented in the codebase — entity, migration, DTOs, mapper, service, controller,
tests"** first, then ask for the field in a second prompt. Seeing the full list up
front tells you whether the change is really small or secretly large.

Then update `DESIGN.md` (and `README.md` if it lists the note fields).

### Quality bar
The new field can be sent to the note endpoints and is accepted, validated, and
returned like the existing fields; an invalid value still returns a `problem+json`
400; and the app starts (proving the new column and the entity agree). No unrelated
code changed.

### Iteration cues
- If the field is accepted but not persisted or returned: "The category is accepted
  but does not come back on read. Which layer dropped it — the mapping, the entity,
  or the migration?"
- If Claude changed more than expected: "Was that change needed for the new field,
  or is it a separate concern?"

---

## Exercise 7 — A Larger Multi-Step Feature: Audit History API

### Prompting objective
Break a large feature into a sequence of steps, each small enough to test before
the next one starts.

### Background
The service is wired for auditing but does not expose it: entities are `@Audited`
and `quarkus-hibernate-envers` is a dependency, but `application.properties` has
`quarkus.hibernate-envers.active=false` and the audit tables are not migrated yet.
So this is a realistic multi-step feature: turn on Envers, migrate its tables, and
expose a read API over the revision history.

### Task
Add an audit-history API: given a partij, return the history of changes to its
contact details and preferences over time.

### Starting prompt

> "Enable Hibernate Envers auditing and add a read endpoint that returns the
> change history of a partij."

### Hints
Use plan mode (as in Exercise 4): "Think through how to enable Envers and expose
its history safely — the migration for the audit tables, the config change, and
the query API." Agree on the structure, then proceed step by step.

An audit feature is too large to produce and verify in one go. Ask for it in four
steps, testing after each before moving on:

1. "Enable Envers and add the Flyway migration for the `_AUD` and revision-info
   tables. Confirm the app still starts (`validate` must pass)."
2. "Add a read endpoint that lists the revisions of a given partij."
3. "Add a point-in-time detail view: the state of the partij at a chosen
   revision."
4. "Add filtering — by date range and by change type."

> **Side note — let Claude write the test data.** After each step, rather than
> hand-crafting a scenario, ask Claude: "Write a short test that creates a partij,
> changes a contact detail twice, and then checks the history endpoint." It knows
> the shape it just built and will exercise it directly. You just run it and
> observe.

Use the review prompt after step 1, since the migration is the risky part:

> "Review the Envers migration and config change. What is most likely to break
> startup or schema validation?"

When complete, update `README.md` and `DESIGN.md`.

### Quality bar
The app starts with Envers enabled (proving the audit-table migration matches what
Envers expects). After changing a contact detail twice, the history endpoint shows
both revisions in order.

### Iteration cues
- If startup fails after enabling Envers: "Startup fails with a schema error after
  enabling Envers. Which audit tables or columns does it expect that the migration
  did not create?"
- If history is empty: "I changed a record but the history is empty. Is the entity
  actually `@Audited`, and did the change commit in a transaction?"

---

## Exercise 8 — A Cross-Cutting Change With Minimal Blast Radius

### Prompting objective
Ask Claude to find the minimal change before making one that touches everything.

### Task
Add pagination (and optionally filtering) to the list-style endpoints — such as
the bulk party lookup — so large result sets are returned in pages, with all
existing endpoints continuing to work.

### Starting prompt

> "Add pagination to the list endpoints so clients can request a page and a page
> size."

### Hints
Before asking for the implementation, ask one design question:

> "What is the minimal change needed so that the list endpoints support pagination
> through one shared mechanism, without changing the signature of every controller
> method individually?"

A good answer introduces a small shared helper or query utility and threads it
through the service layer, rather than rewriting each endpoint. If Claude proposes
editing every controller method separately, push back: "Is there a way to handle
paging in the service/query layer so the controllers barely change?"

Once you have the design confirmed, ask for the implementation:

> "Implement it. Existing endpoints without paging parameters should keep their
> current behaviour."

Then update `README.md` and `DESIGN.md`.

### Quality bar
A list endpoint with `?page=&size=` returns the right slice; the same endpoint
without those parameters behaves exactly as before. Existing tests still pass —
the change did not alter unrelated behaviour.

### Iteration cues
- If existing calls break: "Calling the endpoint without paging params now returns
  [X]. It should behave as before. What changed in the default path?"
- If paging was bolted onto each method: "This edited every controller method. Can
  the paging live in the query layer instead so the controllers stay thin?"

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
