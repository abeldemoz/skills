# Language idioms

How each principle usually looks in a given language. The project's own conventions and the surrounding code win over this file; use it when they are silent. A pattern listed under "Idiomatic" must not be reported as a defect.

Slots: **Seams** (abstraction at a boundary), **Invalid states**, **Errors**, **Wiring** (dependencies), **State** (mutation and concurrency), **Traps** (what agents get wrong in this language).

## Swift

- **Seams:** protocols, owned by the consumer; prefer value types (`struct`, `enum`) and composition over class inheritance. Mark classes `final` by default.
- **Invalid states:** enums with associated values; wrapper types for identifiers and units; failable or throwing initializers that validate once.
- **Errors:** `throws` (typed throws in Swift 6 where the project uses them) or `Result`; use project-defined error enums. Don't use `try?` or `try!` to discard an expected failure.
- **Wiring:** initializer injection; build the graph in one composition root (app entry point). Avoid `.shared` singletons read from logic.
- **State:** actors and `Sendable` for shared mutable state; `@MainActor` for UI state; structured concurrency (`async let`, task groups) over detached tasks; support cancellation.
- **Traps:** a protocol per class "for mocking", force unwraps (`!`) on expected absence, business logic in view controllers or views, `Any`/`AnyObject` where a generic or protocol fits, unowned fire-and-forget `Task {}`.

## Kotlin

- **Seams:** interfaces at real seams; classes are final by default, and mocking libraries can mock them, so an interface for mocking alone is unneeded. Prefer functions and extension functions to one-method classes.
- **Invalid states:** sealed interfaces/classes with exhaustive `when`; `data class` and `@JvmInline value class` for domain values; non-null types by default.
- **Errors:** exceptions are unchecked; use them for exceptional failures and sealed result types for expected ones. Never swallow `CancellationException`.
- **Wiring:** constructor injection; a DI framework (Hilt, Koin, Dagger) is the composition root where the project uses one.
- **State:** `val` over `var`; coroutines with structured concurrency; `StateFlow`/`Flow` for observable state; confine mutable state to one scope or dispatcher.
- **Traps:** `!!` on expected absence, `GlobalScope`, `object` singletons holding mutable state, catching `Exception` broadly in suspending code, `lateinit` to dodge construction order.

## Java

- **Seams:** interfaces at real seams. A lone `Foo`/`FooImpl` pair that exists for no second implementation and no test need is noise. Favor composition over inheritance; package-private visibility hides internals.
- **Invalid states:** records (16+), sealed interfaces with pattern-matching `switch` (17+/21+); validate in the canonical constructor; immutable fields (`final`). Use `Optional` for return values only, not fields or parameters.
- **Errors:** follow the codebase's checked/unchecked policy; always preserve the cause (`new X(msg, e)`); never catch `Exception` or `Throwable` to continue silently.
- **Wiring:** constructor injection; a framework container (Spring, Guice, Dagger, Jakarta CDI) is the composition root where the project uses one. Field injection and service-locator lookups are the smell, not the framework itself.
- **State:** immutable objects, thread-safe collections or confinement to an executor; avoid shared mutable statics; use virtual threads or executors with an owner, never bare `new Thread`.
- **Traps:** getters and setters on every field, all logic in `*Service` classes over passive data objects, static utility classes growing unbounded, inheritance for reuse, over-use of `Optional` chaining, null returned for collections.

## C#

- **Seams:** interfaces at real seams; an `IFoo`/`Foo` pair with a single implementation and no seam is noise. Be cautious wrapping an ORM or `DbContext` in a repository that adds nothing.
- **Invalid states:** records, `init`-only and `required` members, nullable reference types enabled, pattern matching with `switch` expressions; validate at construction.
- **Errors:** exceptions with specific types; preserve the cause and the stack (`throw;`, inner exceptions); don't catch `Exception` to continue.
- **Wiring:** constructor injection through the built-in container (`Microsoft.Extensions.DependencyInjection`) as the composition root; injecting `IServiceProvider` into logic is the service-locator smell.
- **State:** `async`/`await` end to end with a `CancellationToken` parameter; never block with `.Result` or `.Wait()`; no `async void` except event handlers; dispose `IDisposable` owners.
- **Traps:** anemic domain objects with logic in managers, static helper classes, regions to hide size, `Task.Run` around already-async work, exceptions as control flow.

## Go

- **Seams:** small interfaces (often one method) defined by the consumer, not next to the implementation. "Accept interfaces, return concrete types." No interface until a second implementation or a test needs one. Passing function values is a valid seam.
- **Invalid states:** there are no sum types. Use unexported fields with constructors that validate, named types for domain values, `iota` constants with a `String` method, and make the zero value useful or explicitly invalid.
- **Errors:** errors are values: check every one, wrap with context (`fmt.Errorf("…: %w", err)`), test with `errors.Is`/`errors.As`, use sentinel or typed errors only for callers that branch on them. Panic only for programmer error.
- **Wiring:** explicit construction in `main`; DI frameworks are uncommon and not needed. Avoid package-level mutable state and `init()` side effects.
- **State:** pass `context.Context` as the first parameter and honor cancellation; every goroutine has an owner, a stop condition and an error path; confine state to one goroutine or guard it with a mutex local to its owner; prefer `errgroup` to loose goroutines.
- **Traps:** Java-style `Foo`/`FooImpl` pairs, producer-side interfaces "for mocking", large interfaces, `util`/`common`/`models` packages, stuttering names (`user.UserService`), getters named `GetX`, `interface{}`/`any` where a generic or concrete type fits, panics for expected failures, ignoring errors with `_`.

## Rust

- **Seams:** traits at real seams; use generics for static dispatch and `dyn Trait` when the type set is open or compile time matters. Visibility (`pub(crate)`, private modules) hides internals.
- **Invalid states:** enums (sum types) with data; newtypes for identifiers and units; builders or constructors that return `Result` and uphold invariants; `Option` for absence.
- **Errors:** `Result` with `?`; a typed error enum per module or feature in libraries (`thiserror`), contextual errors at application edges (`anyhow`). No `unwrap`/`expect`/`panic!` for expected failures outside tests; keep the source error.
- **Wiring:** pass dependencies in as parameters or generics; build the graph in `main`; avoid global mutable state (`static mut`, `lazy_static` with interior mutability).
- **State:** ownership is the design: one owner, borrows for access. Reach for channels or an owning task before `Arc<Mutex<T>>`; the lock-everywhere shape is the smell. `Send`/`Sync` bounds are part of the API.
- **Traps:** a trait per struct, cloning to appease the borrow checker, `Rc<RefCell<T>>` graphs, premature generics and lifetime gymnastics, `String` where `&str` or a newtype fits, stringly-typed errors.

## TypeScript / JavaScript

- **Seams:** functions and object types as seams (structural typing means a consumer-defined type is enough); classes only where state and behavior belong together. Modules are the unit of hiding: export the minimum.
- **Invalid states:** discriminated unions with exhaustive `switch` (a `never` check), `readonly`, branded types for identifiers. Parse unknown data at the boundary with a schema validator and trust the types after. Avoid `any` and unchecked `as` casts.
- **Errors:** follow the project (exceptions or result types); always handle promise rejections; never leave a floating promise; `throw` `Error` objects, not strings.
- **Wiring:** pass dependencies as arguments or closures (factory functions); avoid import-time side effects and module-level mutable state.
- **State (UI):** keep server data, UI state and derived values separate; derive instead of storing a copy and syncing it in an effect; keep components pure and put side effects in effects or event handlers; lift state only as far as needed.
- **Traps:** `any` everywhere, class hierarchies, barrel files that create cycles, prop-drilling replaced by a global store for local state, effects that mirror props into state, giant components, deep optional chaining hiding missing data.

## Python

- **Seams:** duck typing; use `typing.Protocol` for a structural seam and an ABC only to share implementation. A class with only `__init__` and one method is a function.
- **Invalid states:** `dataclass(frozen=True)`, `enum.Enum`, `Literal` and unions, validated models (pydantic or similar) at the boundary; type hints checked by the project's type checker.
- **Errors:** specific exceptions, `raise … from err` to keep the cause, no bare `except:` and no `except Exception: pass`; use `contextlib` and `with` for cleanup.
- **Wiring:** pass dependencies as parameters; construct them in an entry point; avoid module-level singletons and import-time side effects.
- **State:** no mutable default arguments; prefer immutable data; use `asyncio` structured concurrency (`TaskGroup`) or a process/thread pool with an owner; never mix blocking calls into async code.
- **Traps:** `Manager`/`Helper` classes, `**kwargs` bags that hide a contract, `dict` standing in for a domain type, patching imports deep in tests instead of passing the dependency, deep inheritance, catching and returning `None`.

## C and C++

- **Seams:** C: opaque handles and a documented header API, with ownership stated per function. C++: abstract classes or templates at real seams; headers expose the smallest interface (pimpl only for a stated build or ABI reason).
- **Invalid states:** C++: strong types, `enum class`, `std::variant`, `std::optional`, `std::expected` (C++23); constructors establish invariants. C: validate at the API boundary.
- **Errors:** follow the project's policy (exceptions, error codes or `expected`); never ignore a returned status; clean up on every path (RAII in C++, a single exit label in C).
- **Wiring:** pass dependencies in (constructor or parameter); avoid mutable globals.
- **State:** RAII for every resource; `std::unique_ptr` by default, `std::shared_ptr` only for genuinely shared ownership; `const` correctness; value semantics; define the thread-safety contract for every shared object.
- **Traps:** raw `new`/`delete`, `shared_ptr` everywhere, deep inheritance for reuse, macros where `constexpr` or templates fit, templates for a single use, undefined behavior hidden behind casts.
