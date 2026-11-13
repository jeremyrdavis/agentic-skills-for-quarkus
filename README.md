# agentic-skills-for-quarkus

Claude Code skills for building Quarkus applications.

Three skills, three jobs:

- **`quarkus-app`** — bootstrap and maintain a Quarkus project: platform version, build tool, extensions, Red Hat Build of Quarkus.
- **`quarkus-ddd`** — the Quarkus mechanics tier under the [`ddd-*` skills](https://github.com/jeremyrdavis/agentic-skills-for-ddd): how Quarkus realizes the decisions those skills make.
- **`quarkus-tdd`** — the Quarkus mechanics tier under the [`tdd-*` skills](https://github.com/jeremyrdavis/agentic-skills-for-tdd): how Quarkus runs the red–green–refactor loop.

## Available skills

### `quarkus-app`

Use when creating or maintaining a Quarkus application. It covers creating a new project, choosing the build tool and Java version, adding or updating extensions, keeping versions aligned with the latest platform, and RHBQ. It does not generate domain code.

### `quarkus-ddd`

Use **alongside** the `ddd-*` skills when implementing DDD patterns in a Quarkus project. The DDD skills decide the layers, the aggregate shape, the repository contract, who owns the transaction, and when events are delivered. `quarkus-ddd` supplies the Quarkus side of each decision:

- package layout mirroring the four `ddd-foundations` layers, enforced by an ArchUnit test;
- the extension bundle for a DDD-shaped service;
- `@Transactional` on application-service methods and nowhere else;
- Panache repositories implemented on the infrastructure class behind the hand-written domain interface; `@Version` compare-and-set and its mapping to 409;
- after-commit event delivery with CDI `TransactionPhase` observers, and a transactional outbox to Kafka for events that leave the service;
- test tiers: plain JUnit for `domain`, `@QuarkusTest` with Dev Services for `application` and `api`.

It defines no DDD concepts of its own. Its `references/` directory holds the Quarkus worked examples that the `ddd-*` skills point to: `foundations.md`, `services.md`, `persistence.md`, `domain-events.md`, `architecture-tests.md`.

### `quarkus-tdd`

Use **alongside** the `tdd-*` skills when writing or running tests in a Quarkus project. `tdd-test-list` decides what goes on the test list and in what order; `tdd-cycle` decides when a test is red, what green may change, when to refactor and commit. `quarkus-tdd` supplies the Quarkus side:

- the test tier per `ddd-foundations` layer — plain JUnit 5 for `domain`, `@QuarkusTest` with Dev Services for `application` and `api` — and what "all previous tests pass" costs per tier;
- the runner: `quarkus test` continuous testing during the loop, `./mvnw verify -DskipITs=false` before a PR;
- the first red as a `javac` error and the stub convention (`throw new UnsupportedOperationException()`);
- where the `[TEST]` list lives and how test classes and methods are named;
- `@InjectMock` versus `QuarkusMock`, REST Assured, `@TestTransaction`, the `*IT extends *Test` companion, and what never to mock.

It defines no TDD rule of its own. Its `references/` directory holds `tdd-cycle.md` (one worked loop on the canonical `Order`) and `testing.md` (the test tooling, absorbed from the earlier standalone `quarkus-testing` skill).

## Install

There is no `claude mcp add-skill` command. Use one of the following.

### Plugin, per project (recommended)

Install into each Quarkus project rather than user-wide, so the descriptions are only in context where they apply. `quarkus-ddd` needs the `ddd-skills` plugin installed in the same project; `quarkus-tdd` needs `tdd-skills`.

```bash
claude plugin marketplace add jeremyrdavis/agentic-skills-for-ddd
claude plugin marketplace add jeremyrdavis/agentic-skills-for-tdd
claude plugin marketplace add jeremyrdavis/agentic-skills-for-quarkus
cd <your-quarkus-project>
claude plugin install ddd-skills@agentic-skills-for-ddd --scope project
claude plugin install tdd-skills@agentic-skills-for-tdd --scope project
claude plugin install quarkus-skills@agentic-skills-for-quarkus --scope project
```

One-step form from inside a Claude Code session:

```
/plugin install quarkus-skills --marketplace jeremyrdavis/agentic-skills-for-quarkus
```

A Quarkus project that doesn't use DDD or TDD can install `quarkus-skills` alone; it carries the unused `quarkus-ddd` and `quarkus-tdd` descriptions, which cost a few hundred tokens per session each and nothing else.

### Copy, per project (no manifests)

```bash
git clone https://github.com/jeremyrdavis/agentic-skills-for-quarkus.git
mkdir -p <your-project>/.claude/skills
cp -r agentic-skills-for-quarkus/skills/* <your-project>/.claude/skills/
```

## How the skills work together

For a new service: `quarkus-app` creates the project and adds `rest-jackson, hibernate-orm-panache, jdbc-postgresql` (plus `messaging-kafka, scheduler` if events leave the service). Then the `ddd-*` skill for the concern at hand loads (for example `ddd-aggregates` for a new aggregate), `quarkus-ddd` loads with it, and the assistant reads the one Quarkus reference file for that concern. With the `tdd-*` skills installed, `tdd-test-list` writes the `[TEST]` list first, `tdd-cycle` runs the loop, and `quarkus-tdd` says which tier each test is in and what to run at each step. Precedence: `tdd-cycle` governs the sequence; the `ddd-*` skills constrain the interface each test invokes and the layering; `quarkus-ddd` and `quarkus-tdd` supply mechanics.

Example prompts:

- "Create a new Quarkus REST API with PostgreSQL persistence." → `quarkus-app`
- "Create an orders bounded context with an `Order` aggregate." → `ddd-foundations`, `ddd-aggregates`, `quarkus-ddd`
- "Add a repository for `Order` with optimistic locking." → `ddd-persistence`, `quarkus-ddd`
- "Publish `OrderCancelledEvent` to Kafka." → `ddd-domain-events`, `quarkus-ddd`
- "Add `cancel` to `Order`, test-first." → `tdd-test-list`, `tdd-cycle`, `ddd-aggregates`, `quarkus-ddd`, `quarkus-tdd`
- "Write a `@QuarkusTest` for the orders resource." → `quarkus-tdd` (and `tdd-cycle` if the resource is being built)

## Using the skills with Codex

Codex can use the same files as guidance. Point it at the repository or copy the relevant `skills/*/SKILL.md` into its context. For DDD work, give it the matching `ddd-*` skill as well, and for TDD the `tdd-*` skills; `quarkus-ddd` and `quarkus-tdd` on their own are deliberately incomplete.

## Repository structure

```text
.
├── .claude-plugin/
│   ├── plugin.json
│   └── marketplace.json
└── skills/
    ├── quarkus-app/
    │   ├── SKILL.md
    │   └── references/            (extensions.md, rhbq.md)
    ├── quarkus-ddd/
    │   ├── SKILL.md
    │   └── references/            (foundations, services, persistence, domain-events, architecture-tests)
    └── quarkus-tdd/
        ├── SKILL.md
        └── references/            (tdd-cycle, testing)
```

## Prerequisites

- Java 25 or your preferred version
- Maven or Gradle
- Quarkus CLI recommended for app scaffolding
- Claude Code or Codex, depending on your workflow

## Contributing

Keep `quarkus-app` about the project, `quarkus-ddd` about DDD mechanics, and `quarkus-tdd` about test mechanics. If a change to `quarkus-ddd` would redefine a DDD concept, it belongs in the `ddd-*` repository; if a change to `quarkus-tdd` would redefine a TDD rule, it belongs in the `tdd-*` repository. The `references/*.md` files cite `ddd-skills` and `tdd-skills` Core Rule numbers; they are written against the `1.x` versions.
