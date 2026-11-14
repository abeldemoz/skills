# Design smells

A smell is a symptom, not a verdict. Confirm it by reading callers and neighbors before acting on it or reporting it. A pattern the language or framework makes idiomatic is not a smell (see [languages.md](languages.md)). Each row: what you see, what it usually means, what to do.

## Structure

| Symptom | Usually means | Fix |
|---|---|---|
| A type described only with "and" | Two responsibilities | Split along the reasons to change |
| `Manager`, `Helper`, `Utils`, `Service` holding unrelated functions | Missing concept | Name the concept; move each function to its owner |
| A change fans out across many files for one idea (shotgun surgery) | One concept is smeared across units | Gather it into one unit |
| One unit is edited for unrelated reasons (divergent change) | Several concepts share a unit | Split it |
| A method uses another type's data more than its own (feature envy) | Behavior is in the wrong place | Move it to the data's owner |
| `a.b.c.d` chains | Caller knows the internals of distant collaborators | Ask the nearest collaborator to do it |
| Pass-through wrapper: methods that only forward | Shallow unit | Delete it, or make it own real behavior |
| Interface with one implementation, no seam (`Foo` + `FooImpl`) | Speculative abstraction | Inline it; reintroduce at a real seam |
| Inheritance used to share code | Wrong relationship | Compose; share through an interface plus a value or helper |
| Two units that depend on each other, or import cycles | Hidden shared concept | Extract it so dependencies point one way |
| Data structure with only getters and setters, and all logic in a separate "service" (anemic model) | Behavior separated from the data it protects | Move invariants into the type that owns the data |

## Functions and data

| Symptom | Usually means | Fix |
|---|---|---|
| Boolean parameter that switches behavior | Two functions in one | Split, or take a named option |
| Long parameter list, or the same group of arguments traveling together (data clump) | Missing value type | Introduce the type |
| Parallel optionals or flags that must agree (`isLoading` + `error` + `data`) | Invalid states are representable | One closed set of cases with payloads |
| String or integer standing for a domain value (primitive obsession) | Missing type | Wrap it; validate once at construction |
| Function mixes high-level steps with low-level detail | Two abstraction levels | Extract the detail behind an intention-revealing name |
| Function both changes state and returns an answer | Command and query mixed | Separate them |
| Hidden temporal coupling (call `a()` before `b()`) | Order is in the caller's head | Make the dependency explicit in a type or parameter |
| Deep nesting, early-exit tangles | Too many branches in one place | Guard clauses; extract; model the cases as a type |
| A switch on a kind repeated in several places | Missing polymorphism or a missing case type | One mapping point; exhaustive matching with no catch-all where the language checks it |

## Errors and state

| Symptom | Usually means | Fix |
|---|---|---|
| Empty catch block, ignored error value, optional-conversion of an error, default value on an expected failure | Error swallowed | Propagate, or handle with a visible outcome |
| Catch-all branch over an error or state type | New cases will be silently mishandled | Enumerate every case |
| Matching on error message strings | No stable error contract | Match on type or code |
| One "empty" value (null, nil, None, zero value) for both "absent" and "failed" | Two meanings in one value | Distinguish them in the type |
| Re-validating internal values at every layer | Boundary not established | Validate once at the edge; trust the type after |
| Singleton or global read inside logic | Hidden dependency | Pass it in |
| Work done in a constructor or initializer beyond assigning fields | Construction mixed with behavior | Move it to an explicit method or a factory |
| Background work with no owner, no cancellation and no error path | Leaked concurrency | Give it an owner that cancels and observes it |

## Tests

| Symptom | Usually means | Fix |
|---|---|---|
| Test mirrors the implementation line by line | Pins structure, not behavior | Assert on observable outcomes |
| Test passes with the behavior deleted | Cannot fail | Fix the assertion or the setup |
| Everything is mocked, nothing real runs | Wiring and logic untested | Use real collaborators; replace only the seams |
| Assertions on call counts and order of internal calls | Over-specified | Assert on results and externally visible effects |
| One test with many unrelated assertions | Several concepts | One concept per test |
| Tests share state or depend on execution order | Hidden coupling | Each test builds its own dependencies |
| Sleeps, real clocks or real network in tests | Nondeterminism | Inject the clock and the transport |

## Agent failure modes

Watch for these in your own output first.

- **Over-building.** Extra interfaces, factories, options, generics and layers nobody asked for. Ask what breaks if each is deleted.
- **Under-reading.** A new helper that duplicates one already in the repo, or a pattern that differs from the neighbors. Search before adding; read the closest sibling.
- **Imported idiom.** Patterns from another language or framework: Java-style interface-and-implementation pairs in Go, class hierarchies in Rust or TypeScript, builders where defaults and named arguments exist.
- **Scope creep.** Renames, reformatting or refactors unrelated to the task, which bury the real change in the diff.
- **Defensive padding.** Fallbacks, null checks and swallowed errors for situations the types already rule out, which hide real failures.
- **Leftovers.** Dead code, commented-out code, unused imports and parameters, backwards-compatibility shims for callers that don't exist.
- **Confident guesses.** Inventing an API, a flag or a project rule instead of reading the source. Verify it exists.
- **Tests written to pass.** Assertions copied from the output the code produced, not from the behavior it should have.
- **Wrong layer.** Business rules in the view or the controller, display formatting in the domain, vendor vocabulary beyond the boundary that owns it. Check the project's rules for where each kind of logic lives.
- **Stopping at the diff.** Judging a change without reading its callers, its neighbors or the rule that governs the directory.
