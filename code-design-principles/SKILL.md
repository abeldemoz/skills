---
name: code-design-principles
description: Language-agnostic software design principles for writing and reviewing code. Use before writing or changing non-trivial code in any language (Swift, Kotlin, Java, C#, Go, Rust, TypeScript, Python, C++ and others); when adding a type, interface, function, parameter, error case, test double or test; when splitting or merging code; when deciding whether to abstract or duplicate; and when reviewing code for design quality.
---

# Code Design Principles

Goal: code that is easy to change, hard to misuse and cheap to understand. These are judgment tools, not rules to maximize. Every principle has a cost; apply one only where its cost is lower than the problem it prevents.

Companion files: [smells.md](smells.md) maps symptoms to principles and lists the failure modes agents repeat. [languages.md](languages.md) gives each principle's idiomatic form per language. For structure above the type level (modules, layers, dependency direction), load `architecture-principles`.

## Detect the stack first

1. Identify the language(s), framework and paradigm from the files you are changing and the manifests nearby (`Package.swift`, `go.mod`, `Cargo.toml`, `pom.xml`, `build.gradle`, `*.csproj`, `package.json`, `pyproject.toml`, `CMakeLists.txt`).
2. Read the project's own rules: `AGENTS.md`/`CLAUDE.md` (root, then the nearest nested one), `CONTRIBUTING`, linter and formatter configs. They override everything below.
3. Read the closest sibling code. Match its naming, idiom, error style, test style and comment density before improving on it. Consistency beats a locally better design. Don't rewrite existing code to fit these principles unless asked; report it instead.
4. Open the entry for your language in [languages.md](languages.md). When a principle as worded below conflicts with the language's idiom, the idiom wins: don't write Java in Go or Go in Python.
5. Where no project rule exists for a decision, apply these principles and state the decision and its reason in the plan or PR description. If you cannot tell which convention a project uses, ask instead of choosing for it.

## Before writing

1. Read the neighbors: the closest existing feature that does something similar, and the callers of what you will change. Copy its shape unless you can name why it is wrong.
2. Write down, in a sentence each, the responsibilities your change introduces and who owns them. If a sentence needs "and", split it.
3. Find the existing home for each responsibility before creating a new one. Reuse what exists; extend it only when the new need is the same concept, not just similar code.
4. Pick the smallest design that meets today's requirement. Decide what you are deliberately not building.

## Principles

**Cohesion and responsibility**
- A unit (type, function, module) has one reason to change. Test: describe it in one sentence without "and" or "or".
- Group what changes together; separate what changes for different reasons. Test: a typical change touches one place.
- Names say what a thing is for. `Manager`, `Helper`, `Utils`, `Handler`, `Processor` and `data`/`info` hide a missing responsibility.

**Abstraction**
- Prefer deep units: a small, simple interface that hides real complexity. A wrapper that forwards calls and adds nothing is a cost, not a layer.
- Introduce an interface (protocol, trait, abstract type, function type) at a real seam: a boundary the architecture requires, a second implementation, or a dependency a test must replace. An interface with one implementation and no seam is noise.
- Depend on the abstraction the caller needs, not the shape the implementation happens to have. Interfaces are owned by their consumers.
- Don't leak implementation types (framework, vendor, wire format) through a signature a caller shouldn't know about.

**Duplication**
- DRY means one source of truth for each piece of knowledge, not that similar-looking text must merge. Merge when the copies must change together; leave them when they only look alike today.
- Wait for the third occurrence before extracting, and extract the concept, not the lines. A wrong abstraction costs more than duplication; its parameters and flags are the symptom.

**Simplicity**
- Don't build for hypothetical needs: no unused parameters, options, configuration, extension points or generics "for later".
- Don't leave dead code, commented-out code, compatibility shims for callers that don't exist, or TODOs without an owner and a reason.
- Keep one level of abstraction per function. Extract when the name explains more than the lines do.
- Keep changes scoped. A refactor the task didn't need goes in its own change.

**Types and state**
- Make illegal states unrepresentable with the strongest tool the language gives you: a closed set of cases (sum type, enum with payloads, sealed hierarchy, discriminated union) over parallel optionals or flags; distinct types over bare strings and numbers; a validating constructor over re-checking everywhere. Where the language has none of these, hide the fields and validate once at construction.
- No boolean flag parameters that switch behavior; split the function or take a named option.
- Prefer immutable values and pure functions. Push side effects (I/O, time, randomness, global state) to the edges and pass them in.
- Shared mutable state, singletons and ambient globals are architectural decisions, not conveniences. Pass dependencies in; see `architecture-principles`.
- Tell, don't ask. Don't reach through an object's collaborators (`a.b.c.d`) to act on them.

**Errors**
- Fail fast and loud on a broken invariant; never swallow an error, and never substitute a default that hides a failure.
- Validate at the boundary where untrusted data enters. Inside it, trust the types instead of re-checking.
- Use the error mechanism the language and project already use (exceptions, error values, result types, typed throws), and keep the cause when wrapping. Handle an error where something can actually be done about it; don't catch just to log and continue.
- Error messages state what failed and what the caller can do. Callers branch on type or code, never on message text.

**Tests**
- Test behavior through the public interface, not the implementation. A refactor that keeps behavior must not break a test.
- A test must be able to fail. Watch it fail for the right reason once, or check that removing the behavior breaks it.
- One concept per test; name it by the behavior and condition it pins.
- Don't mock what you don't own, and don't mock the unit under test. Use the project's test-double vocabulary and tooling.
- Hard-to-test code is design feedback: too many collaborators, hidden dependencies, mixed levels. Fix the design instead of reaching for heavier mocks.
- Bugs get a regression test first.

**Names and comments**
- A name reveals intent at the call site. If a comment is needed to explain a name, rename.
- Follow the project's comment conventions. Without one, comments say why, never what or how.

## When finished

Re-read the diff as a reviewer would:
- Can each new unit be described in one sentence? Does each have a reason to exist?
- Is there anything you added that nothing needs yet?
- Does any new interface have exactly one implementation and no seam?
- What changes in two places that should change in one?
- Which error paths are untested? Would each test fail if the behavior broke?
- Does the change match its neighbors and the language's idiom?

For anything non-trivial, get an independent review (the `design-review` skill) and mention in your summary any decision where you chose between real alternatives and why.
