# Canonical Example — Repository, Mirror, and Use-Case-Optimal Query (Java / Quarkus, Panache)

Load this file when implementing a repository or adding a non-trivial query in
a Quarkus project. For placement and review decisions, the Core Rules and Quick
Reference tables in `ddd-persistence/SKILL.md` are sufficient on their own.
Comments cite `ddd-persistence` rule numbers.

## 1. The interface — `<bc>/domain`, ubiquitous language only

```java
package com.example.orders.domain;

import java.util.Optional;

public interface OrderRepository {
    Optional<Order> findById(OrderId id);                    // aggregate out (Rule 2)
    void persist(Order order);                                // whole aggregate, upsert (Rule 6)
    void remove(OrderId id);
    OrderPage findRecent(int page, int size);                 // use-case-optimal VO (Rules 9, 10)
}
```

No `io.quarkus`, no `jakarta.persistence`, no Panache in the interface. Its
vocabulary is the aggregate's vocabulary.

## 2. The purpose-built query VOs — `<bc>/domain`, per `ddd-value-objects`

Domain-specific, immutable, equality by value — explicitly **not** DTOs (Rule 9).

```java
package com.example.orders.domain;

public record OrderSummary(OrderId id, CustomerId customerId,
                           Money total, Order.Status status) {}

public record OrderPage(java.util.List<OrderSummary> items,
                        long totalCount, int page, int size) {
    public OrderPage { items = java.util.List.copyOf(items); }   // defensive immutability
}
```

## 3. The mirror — `<bc>/infrastructure`, carries every framework concern (Rule 4)

```java
package com.example.orders.infrastructure;

import com.example.orders.domain.*;
import jakarta.persistence.*;
import java.time.Instant;
import java.util.UUID;

@Entity
@Table(name = "orders")
public class OrderRecord {

    @Id private UUID id;                              // assigned by the domain, not generated
    @Version private long version;                    // the lock column (Rule 7)
    private UUID customerId;
    @Embedded private MoneyColumns total;             // value object → @Embeddable, no identity
    @Enumerated(EnumType.STRING) private Order.Status status;
    private Instant placedAt;
    // lines: @ElementCollection or a child @Entity — still inside the mirror

    protected OrderRecord() {}                        // JPA's constructor — the mirror's problem, not the aggregate's

    static OrderRecord fromDomain(Order order) {      // outbound translation (Rule 5)
        var r = new OrderRecord();
        r.id = order.id().value();
        r.version = order.version();                  // carry the loaded version into the compare-and-set
        r.customerId = order.customerId().value();
        r.total = MoneyColumns.of(order.total());
        r.status = order.status();
        r.placedAt = order.placedAt();
        return r;
    }

    Order toDomain() {                                // inbound translation — the only rehydrate caller (Rule 5)
        return Order.rehydrate(new OrderId(id), version, new CustomerId(customerId),
                               total.toDomain(), status, placedAt /*, lines */);
    }

    OrderSummary toSummary() {
        return new OrderSummary(new OrderId(id), new CustomerId(customerId), total.toDomain(), status);
    }
}
```

`MoneyColumns` is an `@Embeddable` with `amount` and `currency` columns and a
`toDomain()` that calls the `Money` constructor, so value-object validation
re-runs on the way in.

## 4. The implementation — `<bc>/infrastructure`, same name, Panache behind the interface (Rule 3)

```java
package com.example.orders.infrastructure;

import com.example.orders.domain.*;
import io.quarkus.hibernate.orm.panache.PanacheQuery;
import io.quarkus.hibernate.orm.panache.PanacheRepositoryBase;
import io.quarkus.panache.common.Page;
import io.quarkus.panache.common.Sort;
import jakarta.enterprise.context.ApplicationScoped;
import jakarta.persistence.OptimisticLockException;
import java.util.Optional;
import java.util.UUID;

@ApplicationScoped
// Same simple name as the domain interface — the package is the distinguisher,
// so the implements clause is fully qualified. No technology in the name (Rule 3).
// Panache's own verbs arrive through PanacheRepositoryBase; they are NOT on the
// domain interface. The domain-typed findById(OrderId)/persist(Order) and the
// Panache-typed findById(UUID)/persist(OrderRecord) are ordinary overloads.
public class OrderRepository implements com.example.orders.domain.OrderRepository,
                                        PanacheRepositoryBase<OrderRecord, UUID> {

    @Override
    public Optional<Order> findById(OrderId id) {
        return findByIdOptional(id.value())                  // Panache
                .map(OrderRecord::toDomain);                 // rehydrate inside (Rule 5)
    }

    @Override
    public void persist(Order order) {                       // whole aggregate, upsert (Rule 6)
        OrderRecord record = OrderRecord.fromDomain(order);  // detached mirror carrying the loaded version
        try {
            getEntityManager().merge(record);                // the one verb Panache lacks; merge inserts a new
            flush();                                         // assigned-id row or updates an existing one.
        } catch (OptimisticLockException stale) {            // flush() here makes the compare-and-set fail
            throw new ConcurrencyConflictException(order.id(), stale);   // in this call, not at commit —
        }                                                    // domain-neutral conflict (Rule 7)
    }

    @Override
    public void remove(OrderId id) {
        deleteById(id.value());                              // Panache
    }

    @Override
    public OrderPage findRecent(int page, int size) {        // Vernon query (Rules 9, 10)
        PanacheQuery<OrderRecord> query = findAll(Sort.descending("placedAt"));
        var items = query.page(Page.of(page, size)).list()
                         .stream().map(OrderRecord::toSummary).toList();
        return new OrderPage(items, query.count(), page, size);   // count() ignores paging
    }
}
```

**Conflict surfacing (Rule 7):** `ConcurrencyConflictException` is a domain
exception. The api layer maps it, and nothing else does:

```java
// com.example.orders.api
public class ConflictMapper {
    @ServerExceptionMapper
    public RestResponse<ProblemDTO> onConflict(ConcurrencyConflictException e) {
        return RestResponse.status(Response.Status.CONFLICT, ProblemDTO.of(e));
    }
}
```

The repository never retries — the use case decides what a conflict means.

## 5. An atomic operation (Rule 8)

`thumbsUp` has no invariant, so load-mutate-persist would be a lost-update
hazard. The interface gets an intention-revealing method; the implementation is
one Panache update:

```java
// thoughts/domain — on ThoughtRepository:
void incrementThumbsUp(ThoughtId id);

// thoughts/infrastructure — on the implementation (PanacheRepositoryBase<ThoughtRecord, UUID>):
@Override
public void incrementThumbsUp(ThoughtId id) {
    update("thumbsUp = thumbsUp + 1 where id = ?1", id.value());   // one UPDATE, atomic under concurrency
}
```

## What this demonstrates

- **Interface in domain vocabulary (Rule 2):** typed IDs in, aggregates and
  domain VOs out; zero framework imports in `<bc>/domain`.
- **Same-name implementation (Rule 3):** package is the distinguisher;
  fully-qualified `implements` is Java's spelling of it. Panache is a detail of
  the implementation class, not of the contract.
- **Translation is the whole job (Rule 5):** `toDomain` calls
  `Order.rehydrate(...)`; VO constructors re-validate on the way in.
- **Upsert (Rule 6):** `merge` + `flush` saves the whole aggregate in one call.
- **Locking (Rule 7):** the mirror's `@Version` does the compare-and-set; the
  boundary surfaces a domain-neutral conflict → 409.
- **Atomic op (Rule 8):** one `update(...)` behind a named method.
- **Vernon query (Rules 9, 10):** `OrderPage`/`OrderSummary` are immutable
  domain records — not DTOs; `Page`/`Sort`/`PanacheQuery` never cross the
  interface.

## Quarkus idioms

- `@Version` on the mirror gives compare-and-set for free; keep the aggregate's
  own `version` a plain `long` (per `ddd-aggregates` Rule 10) and copy it into
  the mirror in `fromDomain`.
- Assigned UUID ids mean `merge` selects-then-inserts for a new aggregate and
  updates an existing one — one code path for Rule 6's upsert.
- Prefer Panache's `find("…", Sort, params)` / `update("…")` short forms to
  JPQL strings; drop to `getEntityManager()` only for `merge` or a query Panache
  cannot express.
- Repository tests split: contract tests against an in-memory fake for callers;
  `@QuarkusTest` with Dev Services PostgreSQL for the mapping, the lock, and
  the atomic update.
