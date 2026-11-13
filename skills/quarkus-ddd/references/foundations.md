# Canonical Example — `payments` Bounded Context (Java / Quarkus)

Load this file when scaffolding a new bounded context in a Quarkus project. For
placement and review decisions, the Core Rules and Quick Reference tables in
`ddd-foundations/SKILL.md` are sufficient on their own.

Use case: *authorize a payment* — load the `Payment` aggregate, run a
domain-service fraud check, advance the aggregate's state machine, persist via an
infrastructure repository implementation, return a `PaymentDTO`.

## Module layout

Layers are Java packages under one bounded-context package:

```
com.example.payments
├── domain
│   ├── PaymentMethod.java              (value object — record)
│   ├── Payment.java                    (aggregate root — plain class, no framework imports)
│   ├── PaymentRepository.java          (interface — uses only domain types)
│   ├── FraudDetectionDomainService.java
│   └── PaymentAuthorizedEvent.java     (domain event — record, past tense, *Event suffix)
├── application
│   ├── AuthorizePaymentCommand.java    (input DTO — record)
│   ├── PaymentDTO.java                 (output DTO — record)
│   └── AuthorizePaymentApplicationService.java
├── infrastructure
│   ├── PaymentRecord.java              (JPA-mapped mirror entity — jakarta.persistence lives here)
│   └── PaymentRepository.java          (same name as the domain interface; implements it via Panache, translates)
└── api
    ├── PaymentsResource.java           (HTTP resource)
    └── PaymentNotFoundMapper.java      (exception mapper)
```

Allowed dependency directions (one way only):

```
api ──▶ application ──▶ domain
                           ▲
                infrastructure
```

## The key files

### 1. Repository interface — `domain`, speaks only domain language

```java
package com.example.payments.domain;

import java.util.Optional;

public interface PaymentRepository {
    Optional<Payment> findById(PaymentId id);
    Optional<Payment> findByOrderId(OrderId orderId);
    void persist(Payment payment);
}
```

No `jakarta.persistence`, no Panache, no Hibernate *in the interface*. Its
vocabulary is the aggregate's vocabulary.

### 2. Repository implementation — `infrastructure`, brings the framework (Panache)

```java
package com.example.payments.infrastructure;

import com.example.payments.domain.OrderId;
import com.example.payments.domain.Payment;
import com.example.payments.domain.PaymentId;
import io.quarkus.hibernate.orm.panache.PanacheRepositoryBase;
import jakarta.enterprise.context.ApplicationScoped;
import java.util.Optional;
import java.util.UUID;

@ApplicationScoped
// Same simple name as the domain interface (ddd-persistence Rule 3); the package
// distinguishes, so the implements clause is fully qualified. Panache's verbs come
// from PanacheRepositoryBase and never appear on the domain interface.
public class PaymentRepository implements com.example.payments.domain.PaymentRepository,
                                          PanacheRepositoryBase<PaymentRecord, UUID> {

    @Override
    public Optional<Payment> findById(PaymentId id) {
        return findByIdOptional(id.value())
                .map(PaymentRecord::toDomain);                       // persistence -> domain
    }

    @Override
    public Optional<Payment> findByOrderId(OrderId orderId) {
        return find("orderId", orderId.value()).firstResultOptional()
                .map(PaymentRecord::toDomain);
    }

    @Override
    public void persist(Payment payment) {
        getEntityManager().merge(PaymentRecord.fromDomain(payment)); // domain -> persistence, upsert
        flush();                                                     // surface @Version conflicts here — see references/persistence.md
    }
}
```

`PaymentRecord` is the JPA-mapped mirror entity (`@Entity`, `@Table`, `@Version`,
column mappings) and it lives in this package. The `toDomain`/`fromDomain`
translation keeps ORM concerns out of `domain`. CDI wires this class as *the*
domain `PaymentRepository`; nothing outside `infrastructure` ever names the
implementation. The lock-conflict translation and 409 mapping are in
`references/persistence.md`.

### 3. Application service — `application`, orchestrates the use case

```java
package com.example.payments.application;

import com.example.payments.domain.*;   // domain types and interfaces — never the impl
import jakarta.enterprise.context.ApplicationScoped;
import jakarta.transaction.Transactional;

@ApplicationScoped
public class AuthorizePaymentApplicationService {

    private final PaymentRepository payments;            // domain interface
    private final FraudDetectionDomainService fraud;     // domain service

    AuthorizePaymentApplicationService(PaymentRepository payments,
                                       FraudDetectionDomainService fraud) {
        this.payments = payments;
        this.fraud = fraud;
    }

    @Transactional                                       // owns the transaction boundary
    public PaymentDTO authorize(AuthorizePaymentCommand cmd) {
        Payment payment = payments.findById(cmd.paymentId())
                .orElseThrow(PaymentNotFoundException::new);
        FraudVerdict verdict = fraud.check(payment, cmd.method());
        payment.authorize(verdict);                      // aggregate enforces invariants
        payments.persist(payment);
        return PaymentDTO.of(payment);                   // DTO out — never the entity
    }
}
```

Load → call domain service → mutate aggregate → persist → map to DTO.
Transactions and DTO mapping live here and only here.

### 4. HTTP resource — `api`, depends only on `application`

```java
package com.example.payments.api;

import com.example.payments.application.AuthorizePaymentApplicationService;
import com.example.payments.application.PaymentDTO;
// never an import from ..payments.domain or ..payments.infrastructure
import jakarta.ws.rs.POST;
import jakarta.ws.rs.Path;
import jakarta.ws.rs.PathParam;
import jakarta.ws.rs.core.Response;

@Path("/payments")
public class PaymentsResource {

    private final AuthorizePaymentApplicationService authorizePayment;

    PaymentsResource(AuthorizePaymentApplicationService authorizePayment) {
        this.authorizePayment = authorizePayment;
    }

    @POST
    @Path("/{id}/authorize")
    public Response authorize(@PathParam("id") String id, AuthorizeRequest body) {
        PaymentDTO dto = authorizePayment.authorize(body.toCommand(id));
        return Response.ok(dto).build();
    }
}
```

The resource never sees `Payment`, the `PaymentRepository` interface, or its
infrastructure implementation. Request parsing, status codes, and the exception
mapper in this package are its whole job.

## What this demonstrates

- The repository **interface** lives in `domain`; its signatures use only domain types.
- The **implementation** lives in `infrastructure` and carries all framework
  concerns — including the JPA mirror entity `PaymentRecord` and the translation
  between it and the domain aggregate.
- The application service orchestrates, owns `@Transactional`, depends on the
  domain *interface*, and returns DTOs.
- The fraud check is a `DomainService`: pure domain logic, no I/O.
- The domain event is `PaymentAuthorizedEvent` — past tense, `*Event` suffix, in `domain`.
- The resource lives in `api` and imports only from `application`.

## Quarkus idioms

- Value objects, events, and DTOs are `record`s. Aggregates are plain classes
  with state transitions as methods.
- CDI is the composition root: the constructor-injected domain
  `PaymentRepository` resolves to the Panache-backed infrastructure
  `PaymentRepository` at runtime.
- Domain purity: no `jakarta.persistence.*`, `jakarta.ws.rs.*`,
  `jakarta.transaction.*`, `io.quarkus.*`, or `org.eclipse.microprofile.*`
  imports under `<bc>.domain`.
- Enforce with **ArchUnit**: the layered-architecture rule and the domain-purity
  rule in `references/architecture-tests.md`, one test class per bounded context.
- Scaffolding order when generating a context from scratch: value objects →
  domain events → aggregate → repository interface → domain services →
  application service, commands, DTOs → mirror + repository implementation →
  resource + exception mappers → ArchUnit test.
