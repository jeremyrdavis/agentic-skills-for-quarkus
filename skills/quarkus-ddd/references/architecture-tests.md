# Canonical Example — ArchUnit Layer Test for a Bounded Context (Java / Quarkus)

Load this file when adding the boundary test to a bounded context. The layering
rules are `ddd-foundations` Rules 1–8 and 12; this test makes them fail the
build instead of a review. One test class per bounded context.

## Dependency (test scope)

```xml
<dependency>
    <groupId>com.tngtech.archunit</groupId>
    <artifactId>archunit-junit5</artifactId>
    <version><!-- current release --></version>
    <scope>test</scope>
</dependency>
```

Plain JUnit 5 — no `@QuarkusTest`; it reads class files, not a running app.

## The test

```java
package com.example.orders;

import com.tngtech.archunit.core.importer.ImportOption;
import com.tngtech.archunit.junit.AnalyzeClasses;
import com.tngtech.archunit.junit.ArchTest;
import com.tngtech.archunit.lang.ArchRule;
import jakarta.transaction.Transactional;

import static com.tngtech.archunit.base.DescribedPredicate.not;
import static com.tngtech.archunit.core.domain.JavaClass.Predicates.resideInAPackage;
import static com.tngtech.archunit.core.domain.JavaClass.Predicates.resideInAnyPackage;
import static com.tngtech.archunit.core.domain.JavaClass.Predicates.simpleNameEndingWith;
import static com.tngtech.archunit.lang.syntax.ArchRuleDefinition.*;
import static com.tngtech.archunit.library.Architectures.layeredArchitecture;

@AnalyzeClasses(packages = "com.example.orders", importOptions = ImportOption.DoNotIncludeTests.class)
class OrdersArchitectureTest {

    // ddd-foundations Rules 1–6: four layers, arrows point inward only.
    // Infrastructure may call Application: inbound adapters (observers, Kafka consumers) invoke use cases.
    // The one tolerated api → domain edge: exception mappers name the domain exceptions they map
    // (ddd-foundations placement table; ddd-persistence Rule 7's 409 mapping).
    @ArchTest
    static final ArchRule layers = layeredArchitecture().consideringOnlyDependenciesInLayers()
            .ignoreDependency(resideInAPackage("..orders.api.."), simpleNameEndingWith("Exception"))
            .layer("Domain").definedBy("..orders.domain..")
            .layer("Application").definedBy("..orders.application..")
            .layer("Infrastructure").definedBy("..orders.infrastructure..")
            .layer("Api").definedBy("..orders.api..")
            .whereLayer("Api").mayNotBeAccessedByAnyLayer()
            .whereLayer("Infrastructure").mayNotBeAccessedByAnyLayer()
            .whereLayer("Application").mayOnlyBeAccessedByLayers("Api", "Infrastructure")
            .whereLayer("Domain").mayOnlyBeAccessedByLayers("Application", "Infrastructure");

    // ddd-foundations Rule 2: domain depends on nothing in your code — and no framework.
    // jakarta.enterprise.context is tolerated: domain services are @ApplicationScoped (ddd-services Rule 9).
    @ArchTest
    static final ArchRule domainIsPlain = noClasses().that().resideInAPackage("..orders.domain..")
            .should().dependOnClassesThat().resideInAnyPackage(
                    "jakarta.persistence..", "jakarta.transaction..", "jakarta.ws.rs..",
                    "io.quarkus..", "org.eclipse.microprofile..", "org.hibernate..");

    // ddd-foundations Rule 12: resources see application only — the same exception-mapper carve-out as above.
    @ArchTest
    static final ArchRule apiSeesApplicationOnly = noClasses().that().resideInAPackage("..orders.api..")
            .should().dependOnClassesThat(resideInAnyPackage("..orders.domain..", "..orders.infrastructure..")
                    .and(not(simpleNameEndingWith("Exception"))));

    // quarkus-ddd mechanic 1: the transaction boundary is the application service, nowhere else.
    @ArchTest
    static final ArchRule transactionsLiveInApplication = methods().that().areAnnotatedWith(Transactional.class)
            .should().beDeclaredInClassesThat().resideInAPackage("..orders.application..")
            .orShould().beDeclaredInClassesThat().haveSimpleNameEndingWith("Relay");   // the outbox relay — see domain-events.md

    // ddd-foundations Rules 9–11: the suffix and the package agree; no bare *Service.
    @ArchTest
    static final ArchRule applicationServicesInApplication = classes().that().haveSimpleNameEndingWith("ApplicationService")
            .should().resideInAPackage("..orders.application..");

    // allowEmptyShould: ArchUnit fails a rule whose selection is empty, and a context with no
    // domain service — or no misnamed service — is a valid context.
    @ArchTest
    static final ArchRule domainServicesInDomain = classes().that().haveSimpleNameEndingWith("DomainService")
            .should().resideInAPackage("..orders.domain..")
            .allowEmptyShould(true);

    @ArchTest
    static final ArchRule noUnqualifiedService = noClasses().that().haveSimpleNameEndingWith("Service")
            .and().haveSimpleNameNotEndingWith("ApplicationService")
            .and().haveSimpleNameNotEndingWith("DomainService")
            .should().resideInAPackage("..orders..")
            .allowEmptyShould(true);

    // ddd-foundations Rule 7 / ddd-persistence Rule 3: the interface is domain, the Panache class is infrastructure.
    @ArchTest
    static final ArchRule panacheStaysInInfrastructure = noClasses().that().resideOutsideOfPackage("..orders.infrastructure..")
            .should().dependOnClassesThat().resideInAPackage("io.quarkus.hibernate.orm.panache..");
}
```

## Notes

- `consideringOnlyDependenciesInLayers()` ignores JDK and library dependencies
  in the layer rule; the separate `domainIsPlain` rule is what keeps the
  framework out of `domain`.
- `*Exception` is the only domain name `api` may touch, and only so the
  `@ServerExceptionMapper` can map it; the carve-out is by suffix, so keep
  domain exceptions named `*Exception`.
- If the api layer needs a typed ID in a signature, convert at the edge
  (`new OrderId(UUID.fromString(id))` happens in the application service's
  command constructor, not in the resource) — keep `apiSeesApplicationOnly`
  green rather than weakening it.
- One class per bounded context; for several contexts in one module, add a
  rule that `..orders..` and `..payments..` do not depend on each other's
  `domain` packages — cross-context data flows through integration events.
