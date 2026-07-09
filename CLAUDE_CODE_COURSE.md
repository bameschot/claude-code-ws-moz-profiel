# Extending an Unknown Codebase with Claude

A hands-on course in **using Claude to work with an unfamiliar codebase** —
specifically, practising how to explore, document, and safely change a real
service without needing to understand every class before you begin.

The point of this course is not to memorise this repository up front. The point
is to learn a repeatable workflow:

1. Use Claude to explore the source base and turn what it finds into durable
   project context.
2. Use that context to make one careful cross-cutting change with minimal blast
   radius.

Each exercise has:
- a **prompting objective** — the communication skill being practised
- a **task** — what you are trying to accomplish
- a **starting prompt** — a short, natural prompt you could type right now
- **hints** — small additions that steer toward a more reliable result
- a **quality bar** — what a useful response looks like
- **iteration cues** — signals that you need a follow-up prompt

Work through the exercises in order. Exercise 1 creates the context that Exercise
2 depends on.

---

## Attribution

The Profiel Service is developed by the **Ministerie van Binnenlandse Zaken en
Koninkrijksrelaties (MinBZK)** as part of MijnOverheid Zakelijk, under the
**EUPL-1.2** license. Docs: **[docs.mijnoverheidzakelijk.nl](https://docs.mijnoverheidzakelijk.nl)**.
The extensions you build here are your own work and inherit the same license.

---

## Exploring an Unknown Source Base with Claude

When you join an unfamiliar project, you usually do not know the architecture,
API shape, domain vocabulary, persistence model, conventions, or testing strategy.
That is normal. You do not need to read the whole repository manually before you
can start being productive.

Instead, use Claude as a source-base exploration partner:

- Ask it to inspect the repository structure before proposing changes.
- Ask it to identify the main layers, packages, entry points, tests, and runtime
  configuration.
- Ask it to turn the discovered facts into committed documentation.
- Ask it to cite concrete files, classes, annotations, helpers, migrations, and
  commands so the output is grounded in the actual source.
- Ask it to extract the rules future changes must preserve.

The key habit is to make Claude produce reusable project memory, not just a long
one-off explanation. In this course that memory lives in two files:

- `README.md` — for humans: what the service does, how to run it, how to test it,
  and how to find the API documentation.
- `DESIGN.md` — for Claude and maintainers: architecture, package roles,
  conventions, cross-cutting rules, database shape, testing strategy, and known
  constraints.

---

## Before You Start

### Prerequisites

- **Java 21**
- **Maven** — use the bundled wrapper `./mvnw` if available
- **Docker** — useful when the project expects local infrastructure such as a
  database

Useful commands for this repository include:

```sh
docker compose up -d
./mvnw quarkus:dev
./mvnw verify
```

With dev mode running, inspect the project documentation and generated API docs
that are available locally. If a command fails, capture the exact output and use
that as the next prompt.

### Start with a git repository

Make an initial commit before you change anything — it is your safety net. If a
prompt breaks something, `git diff` shows exactly what changed and `git checkout .`
reverts it. Commit after each exercise passes; the history also lets you ask
Claude "what changed since the last commit?" to orient a new session.

### Ground rules

1. **You direct; Claude executes.** Your job is to give the right goal,
   constraints, and observations. Claude can read the code and explain it back.

2. **Context before code.** Before making a change, ask Claude to read
   `README.md` and `DESIGN.md` and state which conventions apply.

3. **Test after every prompt that changes code.** Run `./mvnw verify`, or start
   the application and exercise the changed behaviour. What you observe is your
   main source of follow-up questions.

4. **One concern per prompt.** Mixing unrelated tasks produces output that is
   hard to test and harder to fix.

5. **Review large output before running it.** See *The Review Prompt* below.

6. **Keep `README.md` and `DESIGN.md` current.** They are the durable memory for
   future sessions.

7. **Commit when tests pass.** A passing commit is a safe rollback point.

8. **Use plan mode for cross-cutting changes.** Before changing behaviour that
   touches multiple layers, ask for a plan and review the blast radius before
   writing code.

### The one rule to keep

Every project has rules that are expensive to rediscover. For this service, ask
Claude to identify those rules during Exercise 1 and record them in `DESIGN.md`.
One important rule to preserve is:

> **Controllers stay thin.** No database access and no business logic in the REST
> layer — all domain logic goes in a transactional service method. And PII
> identifiers must not be written to logs in raw form.

State this rule when you ask for anything touching a controller or logging, and
ensure `DESIGN.md` records the project-specific details and helpers involved.

---

## The Review Prompt

When Claude produces a large result, ask it to review its own output before you
run it:

> "Review what you just wrote. What are the most likely problems or edge cases
> you may have missed?"

A good review surfaces at least one real issue — a missing transaction, an
unhandled not-found case, a validation gap, a schema mismatch, or an accidental
change to unrelated behaviour. Handle those issues before moving on.

When something feels off after testing, review the specific behaviour instead:

> "I ran it and observed [X]. Review the relevant code and explain why it might
> behave that way."

---

## Living Documentation

Across sessions Claude loses track of what has been built. Two committed files fix
this — open every new session with them:

> "Read README.md and DESIGN.md before we continue."

Use the split deliberately:

- `README.md` — user/developer-facing usage documentation.
- `DESIGN.md` — architecture, conventions, package roles, cross-cutting rules,
  design decisions, and implementation constraints.

After each exercise, ask Claude to update both files to reflect what changed.
Then run `/compact` before the next major task if the context window has filled
with code, test output, and fixes.

---

## Exercise 1 — Explore the Source Base and Create Living Documentation

### Prompting objective
Ask Claude to turn an unfamiliar source tree into reusable project context before
making any functional changes.

### Task
Explore the repository and produce the two files that seed the rest of the work:

- `README.md` — for users and developers who need to understand, run, test, and
  use the service at a high level.
- `DESIGN.md` — for Claude and future maintainers: the architecture map, package
  responsibilities, design decisions, coding conventions, recurring patterns,
  database shape, testing strategy, and rules that must be preserved.

If `README.md` already exists, update it rather than replacing useful existing
content. If `DESIGN.md` does not exist, create it at the repository root.

### Starting prompt

> "Explore the repository before making any code changes.
>
> First, inspect the source tree, build files, configuration, migrations, tests,
> and existing documentation. Then create or update the living documentation.
>
> Update README.md so a developer can understand what the service does, how to run
> it, how to test it, where the API docs are, and what the main domain concepts
> are.
>
> Create DESIGN.md in the root of the repository. It should describe the
> architecture, package structure, API shape, tech stack, database model, testing
> approach, coding conventions, recurring implementation patterns, cross-cutting
> concerns, and the rules future changes must preserve. Cite concrete files,
> classes, annotations, helpers, migrations, and commands from this repository."

### Hints
A useful result separates user-facing information from implementation guidance:

- Put practical usage information in `README.md`: purpose, local setup, test
  commands, API documentation URLs, and a short domain/API overview.
- Put architecture and change guidance in `DESIGN.md`: layers, package map,
  dependencies, database schema, external clients, testing strategy, and rules for
  adding new features.
- Ask for a table in `DESIGN.md`: one row per package or key class, showing its
  responsibility, what it provides, and what it depends on.
- Ask for a short dependency diagram showing how the discovered layers call into
  each other.
- Ask it to cover the whole picture, not just source files: REST/API shape,
  runtime configuration, database migrations, test commands, quality gates, and
  generated clients or generated artifacts if present.

Also ask Claude to extract conventions that future changes must follow. Depending
on what it finds, this may include:

- where controller, service, persistence, mapper, validation, and test code lives
- where transactions belong
- how errors are represented
- how request bodies are validated
- how auditing and logging work
- how sensitive identifiers are handled
- how database schema changes are made and verified
- how external clients are generated or called
- how cross-cutting behaviour is shared rather than duplicated

### Quality bar
`README.md` should let a new developer run and test the service without hunting
through the codebase.

`DESIGN.md` should be specific enough that Claude can use it as startup context in
a future session. It should name real classes, packages, tables, endpoints,
constraints, commands, and conventions from this repository, not generic framework
advice.

The conventions section must include concrete examples from the existing code:
actual annotation names, helper names, transaction style, validation style,
mapping style, error-handling approach, and test style where applicable.

After Claude produces the files, use the review prompt:

> "Review README.md and DESIGN.md. Are they missing anything important that a new
> session would need before safely changing this service?"

### Iteration cues
- If `README.md` duplicates too much architecture detail, ask: "Move
  implementation guidance into DESIGN.md and keep README.md focused on running
  and using the service."
- If `DESIGN.md` is only a package summary, ask: "Add the recurring conventions
  and rules future changes must preserve."
- If the database section is thin, ask: "Add the database schema from the
  migrations, with tables, relationships, and key constraints."
- If the conventions are vague, ask: "For each convention, cite one concrete
  existing class, annotation, helper, or method that demonstrates it."
- If Claude makes unsupported claims, ask: "Which file or test proves that?"

---

## Exercise 2 — Implement a Cross-Cutting Concern With Minimal Blast Radius

### Prompting objective
Use the source-base map from Exercise 1 to make a change that cuts across the
system without rewriting everything it touches.

### Task
Add one cross-cutting concern to the service — for example pagination, filtering,
consistent request validation, shared error handling, or structured logging — so
existing behaviour continues to work.

The goal is not to touch as many files as possible. The goal is to find the
smallest shared mechanism that fits the architecture discovered in `DESIGN.md`.

### Starting prompt

> "Read README.md and DESIGN.md first.
>
> I want to add pagination to the list-style endpoints so clients can request a
> page and page size. Before writing code, identify the minimal shared mechanism
> for this project, which layers it should touch, which conventions apply, and
> what existing behaviour must remain unchanged. Then propose a step-by-step
> implementation plan."

### Hints
Before asking for implementation, force the design question:

> "What is the minimal change needed so list endpoints support pagination through
> one shared mechanism, without changing every controller method individually?"

A good answer introduces a small shared helper, query utility, DTO, mapper change,
service-layer convention, or framework-level mechanism that matches the existing
architecture. If Claude proposes editing every endpoint independently, push back:

> "Is there a way to handle this in the shared query/service layer so the
> controllers barely change?"

Once the plan is clear, ask for implementation in small steps:

1. Add or adapt the shared mechanism.
2. Apply it to one representative endpoint.
3. Add tests for default behaviour and paged behaviour.
4. Apply it to the remaining relevant endpoints only after the first path works.
5. Update `README.md` and `DESIGN.md`.

### Quality bar
The change should be easy to explain from the architecture in `DESIGN.md`.

For pagination specifically:

- A list endpoint with `?page=&size=` returns the expected slice.
- The same endpoint without paging parameters behaves exactly as before.
- Invalid paging parameters fail consistently and predictably.
- Existing tests still pass.
- Controllers remain thin.
- The implementation does not duplicate the same logic across many endpoints.

For any other cross-cutting concern, the equivalent bar is: one shared mechanism,
minimal blast radius, tests for both old and new behaviour, and updated living
documentation.

### Iteration cues
- If existing calls break: "Calling the endpoint without paging params now returns
  [X]. It should behave as before. What changed in the default path?"
- If the change was bolted onto each controller: "This edited every controller
  method. Can the shared behaviour live in a helper, query abstraction, service
  layer, interceptor, or mapper instead?"
- If tests only cover the new path: "Add tests proving the old no-parameter path
  still behaves exactly as before."
- If the docs are unchanged: "Update README.md for user-visible behaviour and
  DESIGN.md for the shared implementation pattern."

---

## The Session Pattern

```text
1. Context first    Read README.md and DESIGN.md.
2. One thing        One task or question per prompt.
3. Constraints      State what must not happen.
4. Plan             For cross-cutting changes, ask for the minimal shared design first.
5. Review           Use the review prompt before running large results.
6. Test             Run ./mvnw verify or the relevant local command and observe.
7. Iterate          Describe the observed symptom; ask for the cause.
8. Document         Ask Claude to update README.md and DESIGN.md.
```

### When to change approach

| What you observe | What to do |
|---|---|
| Claude makes generic claims | "Which file, class, test, or migration proves that?" |
| Claude changes things you did not ask about | "Why did you change [X]? Is that needed for [Y]?" |
| Same problem after two fix attempts | "Explain what the code is supposed to do here, step by step." |
| Output is too large to know where to start | Use the review prompt first, then test the areas it flags. |
| Unsure whether the result is correct | "What would a test that catches a bug here look like?" |
| Existing behaviour changes unexpectedly | "Compare the old and new paths. Where did behaviour diverge?" |

---

## Prompts to Avoid

| Avoid | Because | Use instead |
|---|---|---|
| Asking for code before exploration | Claude guesses at architecture and conventions | "Explore the repository and cite the files that establish the pattern." |
| Asking for everything at once | Too large to test meaningfully | One shared mechanism or endpoint group at a time |
| "Fix this" with no observation | Claude guesses at the cause | Describe what you ran and what you saw |
| "Write tests for X" alone | Produces happy-path tests only | Add "cover old behaviour, new behaviour, and edge cases" |
| Accepting output without reviewing | Misses predictable problems | Use the review prompt on any large result |
| Letting cross-cutting logic sprawl | Creates duplicated behaviour | "Find the smallest shared place this belongs." |
| Starting a new session without context | Claude starts from scratch | Always open with DESIGN.md and README.md |
