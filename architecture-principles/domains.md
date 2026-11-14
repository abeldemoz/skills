# Domain lenses

The principles apply everywhere; their weight does not. Find the kind of software you are building, read what matters most there, and treat the rest as secondary. A system can be several at once (a mobile app with a sync engine); apply each section that fits. The project's own rules win.

## Client applications (mobile, desktop, frontend web)

- **UI is the volatile edge.** Keep decisions and data shaping out of views and components; views render state and report user intent.
- **State ownership is the central question.** Separate server data (cached, can be stale), app state and transient UI state. Derive instead of copying, and keep one owner per piece of state. State that is copied and kept in sync by hand drifts.
- **Every async boundary has loading, empty, error and (where relevant) offline states**, and the user can retry. Model them in a type, not as parallel flags.
- **Lifecycle and cancellation.** Screens, components and views come and go while async work continues. Work started for a view is cancelled or ignored when the view goes away; results never write into a dead view.
- **The main or UI thread is a shared resource.** Keep it free of I/O and heavy work; make thread confinement of UI state explicit.
- **Installed versions never go away.** On-device storage needs versioned schemas and migrations; clients must tolerate older and newer servers (version skew). Treat stored data and persisted preferences as a contract.
- **Presentation concerns belong to the view layer**: localization, locale-aware formatting, accessibility, layout. Don't push display strings through the domain.
- **Web only:** keep server and client rendering responsibilities explicit; watch bundle size and import graphs; components should be pure functions of their inputs, with side effects confined to effects and handlers.

## Backend services and APIs

- **The API is a contract.** Version it, never expose third-party shapes, return stable machine-readable error codes, and keep old versions working until consumers have moved.
- **Trust boundary at the edge.** Validate and authenticate at the boundary; derive identity from the verified session, never from client-supplied identifiers; authorize in one place.
- **Transactions and consistency.** Keep transactions short and never hold one open across a network call. Decide per operation what must be atomic, and which parts can be eventually consistent.
- **Design for failure and retry.** Timeouts on every outbound call; retries only for idempotent operations, with backoff; idempotency keys for operations that cannot be repeated.
- **Schema changes are deployments.** Make migrations backward-compatible (expand, migrate, contract) so old and new code run side by side during rollout.
- **Data ownership.** One service or module owns each piece of data; others ask. Prefer a well-structured single deployable until a concrete need (scaling, team boundaries, isolation) justifies splitting it; a distributed system with shared state is the worst of both.
- **Observability is part of the design**: structured logs with correlation IDs, metrics on the boundaries, no secrets or personal data in logs.
- **Concurrency model is explicit**: what runs in parallel, what is shared, what is guarded.

## Libraries, SDKs and frameworks

- **The public API is the product.** Keep the surface minimal; everything exported is a promise. Hide internals with visibility rules and review any new public symbol twice.
- **Semantic versioning and deprecation paths.** Breaking changes are rare, announced and migrated.
- **Don't impose dependencies or global state on consumers.** Keep dependencies few and stable; no singletons or import-time side effects; take configuration and collaborators as parameters.
- **Errors are API.** Their types and codes are stable and documented.
- **Extension points only for demonstrated needs**, because every hook is a permanent constraint.
- **Examples and tests exercise the public surface**, so they double as usability checks.

## CLIs, tools and scripts

- **Separate argument parsing and I/O from the logic**, so the logic is testable without a process.
- **Respect the conventions of the platform**: data on stdout, diagnostics on stderr, meaningful exit codes, non-interactive by default when output is piped, a clear configuration precedence (flags, environment, files, defaults).
- **Make operations safe to re-run (idempotent)**, and destructive ones explicit, with a dry-run where it matters.
- **Keep the surface composable**: small commands with predictable output beat one command with many modes.

## Data pipelines, batch jobs and ML systems

- **Steps are idempotent and re-runnable**, so a failure is recovered by running again; partial failure and backfills are designed in, not discovered.
- **Separate transformation logic (pure, testable) from I/O.**
- **Schemas and contracts at every boundary** between stages or teams; validate data on entry and fail loudly or quarantine bad records, never silently coerce.
- **Reproducibility.** Pin inputs, code and configuration versions so a result can be explained and regenerated; record lineage.
- **Resource and cost limits are part of the design** (memory, partition size, retries, parallelism).

## Performance-critical, embedded and game code

- **Measure before restructuring.** Abstraction costs are real here: indirection, allocation and cache behavior can justify breaking layering rules in a hot path.
- **Separate hot and cold paths.** Keep the clean design for the cold path and isolate the optimized code behind a narrow interface.
- **Budget what matters** (latency, memory, power, frame time) and verify with benchmarks or profiling; write down why an optimization exists, because it will look wrong later.
- **Prefer data layout and ownership clarity over deep object graphs**, and define the lifetime and thread-safety contract of every shared resource.
