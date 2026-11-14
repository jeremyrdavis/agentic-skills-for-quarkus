---
name: quarkus-tdd
description: 'Quarkus test mechanics for the tdd-* skills — use alongside them (never instead of them) when writing or running tests in a Quarkus project. Supplies: which tests run on the plain JVM and which need @QuarkusTest, and what "all previous tests pass" costs per tier, the runner (quarkus test continuous testing, ./mvnw test, ./mvnw verify), the first red as a javac error and the stub convention, where the [TEST] list lives, @QuarkusTest with Dev Services, REST Assured, @InjectMock versus QuarkusMock, @TestTransaction, the *IT extends *Test companion, and test naming. Trigger on @QuarkusTest, @QuarkusIntegrationTest, @InjectMock, QuarkusMock, REST Assured, Dev Services, [TEST] comments in a Java test, a request to run the tests in a Quarkus project, @QuarkusTest on a class with no container dependency, a mock of the class the test is named after, or an assertion on Hibernate, Jackson, or CDI behavior. Defines no TDD rule — the list is tdd-test-list, the loop is tdd-cycle — and no DDD concept. Excludes creating the project and managing extensions and versions (quarkus-app).'
---

# Quarkus TDD Mechanics

The Quarkus tier under the `tdd-*` skills. `tdd-test-list` decides what goes on the list and in what order; `tdd-cycle` decides when a test is red, what green may change, when to refactor and commit. This skill decides **how Quarkus runs that loop**: which tier a test belongs to, what to run at each step, how the first red looks in Java, and which test tooling to use. If a rule here appears to conflict with a `tdd-*` rule, the `tdd-*` rule wins and this file has a bug.

**Load order.** `tdd-test-list` (the list exists), `tdd-cycle` (the loop), then this file, then exactly one reference: `references/tdd-cycle.md` for the loop in Java/Quarkus, `references/testing.md` for test tooling.

**Red Flags — STOP if you find yourself thinking:**

- "`@QuarkusTest` on a plain Java class's test." (If a class with no container dependency needs the container to be tested, it has acquired one — fix the class, not the test.)
- "Run the whole suite, `@QuarkusTest` included, after every red." (The plain-JVM tier runs every step; the container tiers run at green and after refactoring — continuous testing makes that free.)
- "Mock the application service in `CancelOrderApplicationServiceTest`." (Never the class the test is named after.)
- "`@InjectMock` *and* `QuarkusMock.installMockForType` for the same bean, to be safe." (One mechanism per mock.)
- "The pure-JUnit rule tests come later." (Later is never; the rules are the next list item on the plain JVM.)
- "I'll test the repository through the resource." (Three tiers, three classes; a failure must name its tier.)
- "`return null;` to make it compile." (Stubs throw `UnsupportedOperationException` so a stub reached by accident is a red, not a silent pass.)

---

## Stack defaults

| Concern | Default | Alternative (say why) |
| --- | --- | --- |
| Runner during the loop | `quarkus test` (continuous testing) in a second terminal; it reruns affected tests on save | `./mvnw test -Dtest=OrderTest` for one class on demand |
| Runner at the end | `./mvnw verify -DskipITs=false` before a PR | `./mvnw verify` when the project runs ITs by default |
| Assertions | AssertJ (`assertThat`, `assertThatThrownBy`); Hamcrest only inside REST Assured `body(...)` | JUnit `Assertions` if AssertJ is not on the classpath — add it |
| Plain-JVM tier (no container dependency) | Plain JUnit 5, no Quarkus annotations, no container | — |
| Container tier (beans, repositories) | `@QuarkusTest` + Dev Services PostgreSQL with the **real** repository implementation; `@TestTransaction` | A hand-rolled in-memory fake of the repository interface when the test is about orchestration only |
| HTTP tier (resources) | `@QuarkusTest` + REST Assured; `@InjectMock` on the service the resource calls | — |
| Doubles | `@InjectMock` for collaborators; `QuarkusMock.installMockForType/Instance` for the three cases in `references/testing.md`; never a domain type, never the class under test | — |
| Integration | `class FooIT extends FooTest {}` with `@QuarkusIntegrationTest`; check `skipITs` in `pom.xml` | — |
| Test dependencies | `quarkus-junit5`, `quarkus-junit5-mockito`, `rest-assured`, `org.assertj:assertj-core` (test scope), `archunit-junit5` — added by `quarkus-app`, never by hand | — |

---

## Core mechanics

1. **Tier by dependency, not by package.** A test runs on the plain JVM — JUnit 5, no Quarkus annotations — when the class under test needs no CDI, database, or HTTP: domain objects, pure logic, mappers. It is a `@QuarkusTest` when the class needs the container: beans with injected collaborators, repositories against Dev Services, resources through REST Assured. One test class per production class, in the matching test package.
2. **What "all previous tests pass" (`tdd-cycle` Rule 7) costs per tier.** The plain-JVM tier runs at every red and green — milliseconds. The `@QuarkusTest` tiers run at every green commit and after each refactoring; with `quarkus test` running they are already green or already red by the time you look. `verify` with ITs runs before the PR, not inside the loop.
3. **The first red is a compile error.** `tdd-cycle` Rule 5's two steps in Java: the test names a class or method that doesn't exist → `javac` fails → add the type or method with a body of `throw new UnsupportedOperationException();` (never `return null`) → the AssertJ failure is the second red. Constructors and records that must exist get their parameters and nothing else.
4. **The list lives at the bottom of the test class** as `// [TEST] ...` comments, in list order; the line is deleted when its test is written. Items discovered while going green are appended there (`tdd-test-list` Rule 6).
5. **Test names are snake_case sentences that state the behavior**: `cancelling_a_fulfilled_order_is_refused`. The class is `<ProductionClass>Test`; no `should`, no `test` prefix.
6. **`@InjectMock` is the default double.** Declare the field, stub in `@BeforeEach` with `Mockito.when(...)`. `QuarkusMock.installMockForType` only for the three cases (programmatic construction, normal-scoped replacement visible to other beans, a different whole instance per test — paired with `installMockForInstance`). Never both for one bean. Both `QuarkusMock` forms are incompatible with parallel execution.
7. **Never mock the class the test is named after, nor a plain domain object.** Entities, value types, and pure-logic classes are exercised as real instances; mock collaborators only — another application service, an external client, a `Clock`.
8. **REST Assured: status first, then JSON paths.** `given().when().post(...).then().statusCode(200).body("status", equalTo("CANCELLED"))`. The path in the test is the path the resource declares; fix the test, not the resource.
9. **Repository tests use Dev Services and `@TestTransaction`.** Real PostgreSQL, rolled back per test. Inject the domain interface, not the Panache class. Never a manual Testcontainer beside Dev Services.
10. **Every `@QuarkusTest` class gets an `*IT extends *Test` companion** so the packaged artifact runs the same suite. Know whether the project sets `<skipITs>true</skipITs>`.
11. **Don't test the framework.** No test that Hibernate persists a field, Jackson serializes a record, or CDI injects a bean. The mapping, the lock, and the atomic update are yours and get tests; the ORM is not.
12. **Commits follow `tdd-cycle` Rule 12 with `git`**: at green, `feat:`/`fix:` naming the behavior; after refactoring, `refactor:` naming the move. Commit the cycle's files by path (`git commit -m "..." -- <paths>`), never `-a`, so unrelated changes in the tree are left alone. In human mode the red step is `test: <behavior>` with the failing test alone, so the history shows the test came first.

---

## Reference files

Load exactly one, for the concern at hand.

| Doing… | Read | Pairs with |
| --- | --- | --- |
| The loop in Java/Quarkus: a `[TEST]` list, a plain-JVM cycle, a `@QuarkusTest` cycle, the commits, a "passed without red" callout | `references/tdd-cycle.md` | `tdd-cycle`, `tdd-test-list` |
| A `@QuarkusTest`: doubles, REST Assured, Dev Services, `@TestTransaction`, the IT companion, `QuarkusMock` | `references/testing.md` | `tdd-cycle` Rule 13 |


---

## Quick Reference

- [ ] Each `[TEST]` item has a tier, and the test class sits in the matching package.
- [ ] `quarkus test` is running; the plain-JVM tier is green before every commit; container tiers are green at every green commit.
- [ ] A new class or method was first seen as a `javac` failure; stubs throw `UnsupportedOperationException`.
- [ ] The `[TEST]` list is at the bottom of the test class and shrinks as tests are written.
- [ ] Test names are snake_case sentences; classes are `<Production>Test` with an `*IT` companion.
- [ ] Doubles: `@InjectMock` on collaborators only; no mock of the class under test or of a domain type; no bean mocked two ways.
- [ ] Repository tests: real Dev Services PostgreSQL, `@TestTransaction`, domain interface injected.
- [ ] REST Assured asserts status, then JSON paths, at the resource's declared path.
- [ ] Nothing asserts Hibernate, Jackson, or CDI behavior.

### Companion skills

- **`tdd-test-list`, `tdd-cycle`** — the process this skill runs. Written against `tdd-skills` 1.x; Core Rule numbers cited here and in `references/` refer to that version.
- **`quarkus-app`** — the project, the extensions, the test dependencies.
