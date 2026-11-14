---
name: design-review
description: Review a change, branch or path for software design and architecture quality in any language or domain, covering responsibilities, abstraction, coupling, dependency direction, boundaries, data ownership, error handling and tests. Use when asked to review design or architecture, before opening a PR for a non-trivial change, or when an agent is set up as a design reviewer. Read-only; does not edit code.
argument-hint: "[git range | branch | path] (default: changes vs the integration branch)"
---

# Design Review

Find design problems that make code costlier to change, easier to misuse or harder to understand. Review the target named in the arguments, or the default scope below.

Never modify files, stage, commit or install tools. A reviewer that also wrote the change shares its blind spots, so prefer running this in a fresh context (a read-only subagent or a new session) when you authored the code.

Checklists: load `code-design-principles` and `architecture-principles`. Use their `smells.md`, `languages.md` and `domains.md` to name findings and to avoid flagging idiomatic code.

## Out of scope

- Formatting and lint: CI covers them.
- Functional bugs and security flaws as such: a code review covers them. Report one only when it exposes a design problem, such as an unhandled state the types allow.
- Language API naming and platform UI guidelines, when the project has a dedicated reviewer for them (look in `.claude/agents` and the project's `AGENTS.md`).

## Procedure

1. **Detect the stack.** Identify the language(s), framework and domain from the changed files and manifests. Read the project's rules: `AGENTS.md`/`CLAUDE.md` (root, then the nearest nested file for each changed file), `CONTRIBUTING`, linter configs. Project rules take precedence over the principles, and a violation of one is a finding even when the principles would allow it. Never invent a project rule.
2. **Scope.**
   - No target: diff the integration branch against `HEAD`, plus uncommitted changes. The integration branch is the one the project docs name (often `develop` or `main`); otherwise the repository default (`git symbolic-ref --short refs/remotes/origin/HEAD`).
   - Commit range or branch: diff it. For a branch, diff from its merge base with the integration branch.
   - Path: review the files under that path in full.
   - Ignore generated and vendored files, lockfiles, build output, snapshots, minified assets and localization catalogs. If nothing reviewable remains, say so and stop.
3. **Intent.** Read the commit messages, branch name and any PR description to learn what the change is for. Judge a change against its intent, not against what you would have built.
4. **Context.** Read the surrounding code, not only the diff: the closest sibling feature, and the callers and implementers of anything added or changed. Don't report pre-existing issues the change doesn't touch.
5. **Architecture pass.** Redraw the dependency arrows the change adds. Check direction, cycles, leaked types, ownership of new state and data, and what each new module or boundary hides.
6. **Design pass.** For each new or changed unit, check responsibilities, abstraction level, coupling, duplication, error handling, invalid states and hidden dependencies.
7. **Tests pass.** Check that tests pin behavior rather than implementation, that each could fail, and that new error and edge paths are covered.
8. **Simplicity pass.** Look for what could be deleted: unused abstractions, options, layers, dead code, scope creep.
9. **Verify before reporting.** Back every claim with evidence. Search for usages before calling something unused, single-implementation or duplicated. Open the rule before citing it. Drop a finding you cannot support.

## Principles

- Prioritize what is costly to change later. A boundary or dependency mistake outranks a clumsy function.
- Every finding names the concrete cost: what becomes harder, which change would break, who could misuse it. A finding without a cost is a preference; drop it.
- Judge against the language's idiom. A pattern that is idiomatic in this language and framework is not a defect, even if it would be a smell elsewhere.
- Recognize intentional tradeoffs. A pragmatic shortcut that is contained, labeled and cheap to undo is not a defect.
- Don't demand abstraction the change doesn't need. Over-engineering is as reportable as under-engineering.
- Don't recommend a rewrite. Recommend the smallest change that removes the cost.
- If the author's intent is unclear and the verdict depends on it, ask a question instead of asserting a finding.

## Severity

- **Critical**: breaks a project architecture rule or a contract, or allows a state or failure that will cause defects.
- **Major**: raises the cost of the next likely change, such as a leaky boundary, a responsibility in the wrong layer, hidden state, or tests that cannot fail.
- **Minor**: a contained weakness or a simplification worth taking.

## Output

Findings ordered by severity. Each finding has:

1. Severity
2. `path:line`
3. Principle or rule: the principle by name, or the file path for a project rule
4. What is wrong, with the evidence you gathered
5. Cost: what gets harder, and when you would pay for it
6. Recommended fix: the smallest concrete change
7. Confidence: high, medium or low

If there are no actionable problems, say so explicitly.

Then:
- **Questions** for the author, only where intent changes the verdict.
- **What the change does well**, in at most two lines, only where it is a deliberate design choice worth keeping.
- **Not reviewed**: anything out of scope or unreadable that the author should route elsewhere.
