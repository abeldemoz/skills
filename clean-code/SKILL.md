---
name: clean-code
description: Review code against Robert C. Martin's Clean Code principles (naming, functions, comments, error handling, boundaries, unit tests, classes, systems). Use when asked to enforce Clean Code, check Uncle Bob violations, or review for clean code principles. Supports --fix to apply changes automatically.
---

Review the code for violations of Robert C. Martin's Clean Code principles.

## Scope
- If a path argument is provided, review that file or directory
- Otherwise, review the current `git diff` (staged and unstaged changes)

## Principles to enforce

### 1. Naming
- Variables, functions, and classes use intention-revealing names
- No single-letter names except loop counters
- No abbreviations or encodings (Hungarian notation, type prefixes)
- Names are pronounceable and searchable
- Classes are nouns; methods are verbs
- No misleading names (e.g., `accountList` that isn't a List)

### 2. Functions
- Functions do one thing only
- Functions are small (ideally ≤20 lines)
- One level of abstraction per function
- No more than 3 arguments; use parameter objects when more are needed
- No flag arguments (boolean params that control behavior)
- No side effects
- Command-query separation: functions either do something or answer something, not both

### 3. Comments
- No redundant comments that restate the code
- No noise comments (obvious getters/setters, constructors)
- No commented-out code
- No journal/changelog comments (use git for that)
- Legal and truly explanatory comments (why, not what) are acceptable

### 4. Error Handling
- Use exceptions rather than error codes
- No null returns; prefer Optional, empty collections, or exceptions
- No null passing as arguments
- Error messages are informative
- Don't swallow exceptions silently

### 5. Boundaries
- Third-party APIs are wrapped, not used directly throughout the codebase
- Interfaces define what the code needs, not what the library provides

### 6. Unit Tests
- Tests follow FIRST: Fast, Independent, Repeatable, Self-validating, Timely
- One concept per test
- Tests are as clean as production code
- No magic numbers in tests; use named constants
- Test names clearly describe what they test

### 7. Classes
- Classes are small and focused on a single responsibility (SRP)
- High cohesion: instance variables used by most methods
- Open for extension, closed for modification (OCP)
- Depend on abstractions, not concretions (DIP)

### 8. Systems
- Separate construction (wiring/initialization) from use (runtime logic)
- Use dependency injection
- Don't mix startup logic with business logic

## Output format

Group findings by file and location. For each violation:
- `file:line` reference
- Principle violated
- Offending code (quoted)
- Concise explanation
- Suggested fix

## Flags

`--fix` — after reporting all violations, automatically apply all fixes. No per-violation confirmation; the user reviews the resulting diff.

## Behavior

Always run at full depth across all 8 principles regardless of file size or violation count.
