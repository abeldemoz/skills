---
name: architecture-principles
description: Language- and platform-agnostic software architecture principles for designing and reviewing structure above the single type. Use before building a feature or vertical slice; when adding a package, module, layer, dependency, cross-boundary type, persistence or network integration, or cross-cutting concern; when moving code between layers; and when reviewing dependency direction, boundaries or data ownership. Applies to mobile, frontend, backend, libraries, CLIs and data pipelines.
---

# Architecture Principles

Goal: a structure where a typical change touches one place, volatile details can be swapped without rippling, and the rules of the system live in one spot each. Architecture is the set of decisions that are expensive to reverse; spend design effort in proportion to that cost.

Companion: [domains.md](domains.md) says which principles weigh most in each kind of software. For code inside a unit (functions, types, errors, tests), load `code-design-principles`.

## Detect the system first

1. Identify what you are building and in what ecosystem: a client app (mobile, desktop, frontend web), a backend service, a library or SDK, a CLI or tool, a data pipeline, performance-critical code. Read the matching section of [domains.md](domains.md).
2. Read the project's own architecture rules: `AGENTS.md`/`CLAUDE.md` (root, then the nearest nested one), `CONTRIBUTING`, architecture docs, module or build files that enforce boundaries. They override everything below.
3. Read the existing structure before adding to it: the module layout, how a similar feature is organized, which direction dependencies already point. Extend the existing architecture; don't import another one.
4. Where no rule exists for a decision, don't invent one silently. Apply the principles below, then state the decision, the alternatives and the reason in the plan or PR description so the project owner can overrule it. If the choice is large and the project gives no signal, ask first.
5. Don't restructure existing code to fit these principles unless asked; report the mismatch instead.

## Before building anything non-trivial: the design brief

Write this down (in the plan or in your reply) before the first file. Every line states its reason.

1. **Problem and constraints**: what must be true when this works, and what is out of scope.
2. **Responsibilities and owners**: each responsibility in a sentence, and the layer or module that owns it. Reuse the existing owner before creating one.
3. **Dependency direction**: who depends on whom, drawn as arrows. Check each arrow against the project's rules and the principles below.
4. **Seams and boundaries**: where data crosses a boundary, which type crosses it, and who owns that type.
5. **Data and state**: the single source of truth for each piece of state, who may mutate it, and what is derived rather than stored.
6. **Failure and edge states**: what happens on error, empty, slow, offline, partial and concurrent use, at each boundary that applies to this kind of system.
7. **Options considered**: at least one real alternative, why it lost, and what the chosen option gives up.
8. **Deferred**: what you are deliberately not building, and the signal that would make it worth building.

For a small change inside one existing unit, a sentence for items 2 and 3 is enough. Skip the brief only when the change adds no responsibility, dependency or boundary.

## Principles

**Fit the style to the problem**
- Match the weight of the architecture to the weight of the problem. A simple CRUD service or a script doesn't need ports, adapters and a layer per concept. A layer that only forwards calls is a cost.
- Prefer the simplest structure that keeps the next likely change local. Add structure when a real second case, a real boundary or real pain shows up, not before.

**Dependency direction**
- Dependencies point toward stable, abstract, domain-level code. Volatile details (UI, frameworks, vendors, storage, wire formats) sit at the edge and depend inward.
- Domain logic should be testable without the framework, network, disk or clock. Where the ecosystem makes a framework the wiring mechanism (Spring, ASP.NET, Dagger, a UI framework), use it, but keep logic out of the framework-facing code.
- Wrap a third-party dependency behind an interface you own when it is volatile, hard to test, or its types would spread through your code. Don't wrap stable, ubiquitous ones (the standard library, basic collection, serialization or date utilities).
- Dependencies are acyclic. A cycle means a concept is missing; extract it so both sides depend on it.
- Depend on another module's public surface, never its internals. Enforce it with the strongest tool available: visibility modifiers, build-level module or package boundaries, lint rules for imports.

**Boundaries and information hiding**
- Draw a boundary around each design decision likely to change (a vendor, a format, a policy, a storage choice) so that decision lives in one module.
- Each boundary has its own vocabulary and types. Translate at the boundary (wire format → domain → view or response); don't let one layer's shapes, names or sentinel values leak into the next.
- Keep a boundary's interface smaller than its implementation. A boundary that needs many calls to be used correctly is in the wrong place.
- Data that crosses a boundary carries data, not behavior from the layer that produced it.

**Cohesion of modules**
- Organize by feature or capability where the ecosystem allows, so code that changes together lives together. Follow the project's existing layout; a mixed layout is worse than either.
- Shared code earns its place by having two or more real consumers. One consumer means it belongs to that consumer. A shared module that grows into a dumping ground (`common`, `utils`, `shared`) is the main way boundaries erode.
- A module that everything imports is a liability: every edit retests and risks everything. Keep it small, stable and free of policy.

**Data ownership and state**
- Exactly one owner for each piece of state, and one place that enforces each invariant. Derived values are computed, not stored twice.
- Prefer a flow where data moves one way and changes are requests to the owner, not writes from afar.
- Concurrency is a design decision made per piece of state: confine it to one owner (actor, task, queue, transaction, event loop) rather than guarding shared state with locks everywhere.

**Wiring**
- Separate construction from use. Build the object graph in one place near the program's entry point; everything else receives its dependencies. No service locators and no singletons read from inside logic.
- Cross-cutting concerns (logging, metrics, feature flags, auth) go through a dedicated seam, not ad hoc calls scattered through feature code.

**Contracts and evolution**
- Anything crossing a process, device or release boundary (HTTP API, event, stored data, file format, public library API, deep link) is a contract: version it, keep old versions working until consumers have moved, give errors a stable machine-readable code and never expose a third party's shapes.
- Make a change easy, then make the easy change: prefer a small preparatory refactor, then the feature, over a feature bent around the existing structure.
- Prefer reversible decisions and delay irreversible ones until the information exists. Don't add distribution, queues, caches or plugin systems before a concrete need shows up.
- Design every network and storage interaction for failure: timeouts, retries only for idempotent operations, partial failure, and an explicit way for the caller to see each outcome.

**Testability as an architecture property**
- If domain logic cannot be tested without the framework, network, disk or clock, a dependency points the wrong way.
- Replace dependencies at the seams the architecture already has; don't add seams only for tests.
- Test each layer against its own contract: pure rules in isolation, adapters against the real thing or a faithful fake, wiring with a few end-to-end paths.

## Smells at this level

| Symptom | Usually means |
|---|---|
| A feature edit requires changes in several modules | A boundary is in the wrong place, or a concept is smeared |
| A vendor, framework or wire type appears in a domain-level signature | Boundary leak |
| Business decisions made in a view, controller, handler or serialization type | Wrong layer |
| Display formatting or presentation copy in a layer that doesn't render | Wrong layer |
| Two modules each keep their own copy of the same state | No owner |
| A new shared module used by one consumer | Premature sharing |
| A module reaching into another's internals | Boundary not enforced |
| Constructors that build their own collaborators | Construction mixed with use |
| An abstraction layer with no second implementation and no boundary to protect | Speculative layer |
| A contract change that breaks existing consumers | Missing versioning |
| An integration with no timeout, no failure path or no idempotency story | Happy-path design |
| A heavyweight architecture around a simple problem | Style doesn't fit the problem |

## When finished

- Redraw the dependency arrows from the diff. Does every new arrow point in an allowed direction? Is anything new imported that shouldn't be?
- Name what each new module or boundary hides. If it hides nothing, delete it.
- State what the next likely change costs: how many places does it touch?
- Report any project rule you had to bend, and any decision the project owner may want to overrule.

For a non-trivial change, get an independent review (the `design-review` skill).
