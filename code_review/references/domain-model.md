# Domain Model Integrity Reference

Based on "Learning Domain-Driven Design" (Khononov, O'Reilly, 2021) and
"Mastering Domain-Driven Design" (Junker, 2025).

Apply this checklist when the code models a business domain — entities, aggregates,
services, repositories, or domain events. Skip entirely for pure infrastructure or
UI-only changes.

---

## Ubiquitous Language

The shared vocabulary of the business and the codebase should be the same. When they diverge,
the model drifts from reality and becomes harder to reason about.

- [ ] Variable, class, and function names reflect domain concepts, not technical infrastructure.
  `userData` or `activeRecord` instead of `Customer` or `Campaign` signals the domain hasn't
  been captured in the model.
- [ ] Ambiguous terms used in multiple senses within one bounded context are flagged.
  `user`, `visitor`, and `account` should not coexist unresolved in the same module.
- [ ] Domain concepts aren't mapped onto generic technical names (`Item`, `Data`, `Record`, `Handler`)
  when a more specific domain term exists.
- [ ] Method names describe domain intent, not technical mechanics
  (`placeOrder()` rather than `insertOrderRecord()`).

---

## Aggregate Boundaries

An aggregate is a cluster of domain objects treated as a single unit for data consistency.
Every aggregate has a root — the only entry point for external code.

- [ ] A single transaction modifies exactly one aggregate root. If a command touches multiple
  aggregate roots without going through sagas or domain events, the boundary is almost
  certainly wrong.
- [ ] External code holds references only to aggregate roots, never to child entities inside
  an aggregate.
- [ ] Aggregates holding more than ~3–5 closely related entities are questioned — oversized
  aggregates create concurrency bottlenecks and harm scalability.
- [ ] No aggregate directly mutates another aggregate's state. Cross-aggregate coordination
  belongs in sagas or process managers.
- [ ] Collections inside an aggregate (e.g., an `Order` with `OrderLines`) are encapsulated —
  callers add/remove items through the root's methods, not by manipulating the collection directly.

**Common boundary mistakes to flag:**
- A `save()` that accepts individual child entities rather than the full aggregate root
- A `find()` method that returns child entities detached from their root
- Methods on the root that accept external aggregate instances as parameters and mutate them
- A single HTTP request that persists changes to two or more aggregate roots in one transaction

---

## Entities and Value Objects

**Entities** have stable identity that persists across state changes (e.g., a `Customer` is
still the same customer after changing their address). **Value objects** are defined entirely
by their attributes — two value objects with the same attributes are interchangeable.

**Entities — what to check:**
- [ ] Entities have an explicit, stable identity field (typically an ID).
- [ ] Objects that change over time but lack explicit identity are likely misclassified —
  they may need to be promoted to entities with proper identity management.
- [ ] Entity identity is not derived from mutable attributes (using `email` as an entity's
  primary identity is fragile if email can change).

**Value objects — what to check:**
- [ ] Value objects are immutable. Any setter or mutable field on a class intended as a value
  object is a bug waiting to happen — mutations should return a new instance.
- [ ] Value objects compare by value (`equals`/`hashCode` overridden in Java/Kotlin;
  `__eq__` in Python; `PartialEq`/`Eq` derived in Rust; `==` operator in Go structs).
  Reference equality for value objects produces subtle, hard-to-diagnose bugs.
- [ ] Value objects contain no business-significant identity field — if they need one, they
  should be entities.
- [ ] Value objects that wrap primitives (e.g., `Money`, `EmailAddress`, `OrderId`) enforce
  their own invariants (valid range, format, etc.) rather than relying on callers to validate.

---

## Repositories

A repository abstracts the mechanism for loading and saving aggregates. It should look like
an in-memory collection of aggregate roots from the domain's perspective.

- [ ] Repositories load and save complete aggregates — never fragments of one. A `find()` that
  returns child entities without their aggregate root breaks encapsulation.
- [ ] Generic ORM queries exposed directly from a repository (e.g., `.Query().Where(...)`,
  `.findAll(Specification<T>)`) violate aggregate boundaries. Domain-specific query methods
  belong on the repository interface.
- [ ] Saving individual child entities (rather than the root) is a boundary violation.
- [ ] Repositories interact with exactly one aggregate type. A `CustomerRepository` that also
  saves `Orders` is a design smell.
- [ ] Repository interfaces are defined in the domain layer; implementations live in the
  infrastructure layer. Domain code depends on the interface, not the ORM.
- [ ] Repositories do not contain business logic — they only load and persist aggregates.
  Business rules belong on the aggregate or domain services.

---

## Transactions and Consistency

- [ ] All mutations to an aggregate are wrapped in a single atomic transaction.
  Separate `UPDATE` and `INSERT` operations on the same aggregate without a transaction
  wrapper are a consistency bug.
- [ ] Read-modify-write operations without optimistic locking (`@Version`, `rowVersion`,
  ETags, CAS) or pessimistic locking are implicit race conditions under concurrent load.
- [ ] Domain events are published *after* a successful commit, not synchronously from within
  domain logic. Publishing inside a transaction and then losing the event if the message broker
  is unavailable is a common data loss pattern.
- [ ] The **outbox pattern** (persist events in the same DB transaction as the state change,
  then relay them asynchronously) is used for reliable event delivery.
- [ ] Long-running business processes (multi-step workflows) use sagas or process managers
  rather than distributed transactions.

---

## Domain Events

Domain events represent something significant that happened in the domain — they are facts,
named in past tense (`OrderPlaced`, `PaymentFailed`, `InventoryReserved`).

- [ ] Events raised inside aggregates are actually published downstream — verify the dispatch
  path from the aggregate to the event bus/outbox.
- [ ] Event names reflect domain intent, not technical mechanics.
  `UserRecordCreatedInPostgres` does not belong in the domain model.
- [ ] Integration event schemas (crossing bounded context boundaries) are versioned and
  backward-compatible. Removing fields from an event is a breaking change.
- [ ] Event handlers are idempotent — processing the same event twice must not cause
  duplicate side effects (use idempotency keys or check-before-act patterns).
- [ ] Stale event handlers that no longer match the aggregate's current behavior indicate
  model drift and should be updated or removed.
- [ ] Event payload contains only what's necessary. Including full entity state in every event
  creates unintended coupling between bounded contexts.

---

## Bounded Contexts

A bounded context is a specific area of the domain with its own model, language, and boundaries.
Contexts integrate through well-defined contracts (APIs, integration events, anti-corruption layers).

- [ ] Direct instantiation or mutation of classes from another service or bounded context is
  a boundary violation. Use DTOs, value objects, or integration events to cross boundaries.
- [ ] Shared mutable database tables between bounded contexts without explicit governance
  indicate a shared kernel that needs an anti-corruption layer (ACL).
- [ ] Anti-corruption layers that contain complex business logic may be masking an incorrect
  aggregate boundary upstream — the ACL should translate, not compute.
- [ ] Synchronous calls from one bounded context into another for every request create temporal
  coupling. Evaluate whether asynchronous integration would be more resilient.
- [ ] Eventual consistency between bounded contexts is explicitly acknowledged in the design —
  the UI and business logic should handle the window of inconsistency gracefully.
