---
name: pr-description
description: Write the title and body of a pull request, and only that. It returns text for the caller to publish and never runs `gh pr create`/`gh pr edit`, commits or pushes. Use this whenever someone asks to write, draft, rewrite, tighten, fix or update a PR description or title, says a branch is ready for review or to "fill in the PR", and also before ANY agent opens a pull request or edits an existing PR's body, even when the user only said "open a PR" or "raise a PR", so the text is written deliberately instead of auto-filled from commit messages. Not for commit messages, release notes or changelogs, or reviewing someone else's PR.
---

# Writing a PR description

A PR description is read by a reviewer who has the diff but not your context, and months later by someone following `git blame` back to this PR. The diff already shows which lines changed. The description carries what the diff cannot: why the change exists, the alternatives that lost, how it was verified, and what is still open.

Reviewers learn where to look when every PR has the same shape, so the order of sections never changes. But a typo fix and a data migration need different things said. Every PR gets the same spine, and extra sections appear only when the change calls for them.

## What you return

1. **Title**: one line.
2. **Body**: in a fenced `markdown` block, so the caller can save it and pass `--body-file`.
3. **Open items**: what the user must do or decide that you could not (attach screenshots, a ticket key you could not find, checks not run, a question). Leave the list out if there are none.
4. **Conventions followed**: template, merged-PR pattern or these defaults.

Do not publish. Whoever called you opens or edits the PR, and adds any attribution line their instructions call for. If the user asked for the PR to be opened or updated, that is a separate step you take with this text after the skill's work is done.

## 1. Find the project's conventions

A project's own style beats the defaults below. Stop at the first source that answers each question:

1. **A PR template**: `.github/pull_request_template.md` (any case), `.github/PULL_REQUEST_TEMPLATE/`, or the same names in the repo root or `docs/`. Fill in its headings. If a section below has no heading in the template, put it under the closest one rather than dropping it. The evidence rules in step 2 still apply inside a template.
2. **Written guidance**: `AGENTS.md`, `CLAUDE.md`, `CONTRIBUTING.md`, a README section on PRs.
3. **Recent merged PRs**: `gh pr list --state merged --limit 20 --json number,title,body`. Read them for title format (ticket prefix, casing, tense), tracker-link format and footer lines. Copy the form, not the length, because one of them may be an outlier.
4. **The defaults in this skill.**

Do not import conventions from other repos you have seen.

## 2. Gather the evidence

Take each fact from the first source that has it:

1. **The conversation**: what the user asked for, what was run and what it printed.
2. **The ticket**: through whatever tracker tool is available. Find the key in the branch name, the commits and the user's message. Never invent one.
3. **The commits**: `git log <base>..HEAD`. The base is the existing PR's base, otherwise the default branch, unless the branch is clearly stacked on another.
4. **The diff**: `git diff <base>...HEAD --stat`, then the hunks that matter. A description written from commit subjects alone misses the point.

For an existing PR, also run `gh pr view --json title,body,baseRefName,headRefOid` and `gh pr checks`.

Two things must not be guessed:

- **Why.** If a reviewer would ask "why this way?" and no source answers, ask the user one short question before writing. For a small change the what is the why, and that is fine.
- **What was verified.** Only a command you ran or output you saw in this session, or a CI result you checked, counts as done. A description that claims unrun checks teaches reviewers to distrust the ticks that are real.

## 3. Decide which sections apply

Order is fixed. Include a section only when its condition holds; several can hold at once. Work type (feature, bug fix, refactor, chore, docs, revert) only predicts which conditions will fire.

| # | Section | Include when |
|---|---|---|
| 1 | Opener (one sentence) | always |
| 2 | Links line | ticket found, stacked series, or revert |
| 3 | `## Summary` | always |
| 4 | `## Root cause` | bug fix |
| 5 | `## Breaking changes` | public API, contract or file format changes |
| 6 | `## Migration` | schema, data model or persisted-format change |
| 7 | `## Rollout` | anything must happen besides merging |
| 8 | `## Copy & translations` | user-facing text added or changed |
| 9 | `## Performance` | the PR claims a performance change |
| 10 | `## Review notes` | anything to flag |
| 11 | `## Testing` | always |

Detect the conditions from the diff and the conversation: migration or schema files and persisted models; locale, string-catalog or i18n files; flags, environment config, secrets, entitlements or deploy files; exported or public API surface; benchmark claims.

## 4. Write each section

**Opener.** One sentence in the present tense that states the outcome for the user or the codebase, not where you edited. "Exports now include archived projects." beats "Update ExportService." Make it stand alone, because with squash merging it may become the commit message.

**Links line.** The tracker link in the project's format. Write `Fixes #123` only when the tracker is GitHub Issues. For a stacked series write `Part 2 of 5, depends on #41`. For a revert write `Reverts #88` and say why.

**Summary.** Start with the problem: what was slow, missing or broken before. Then, only if the change spans several layers (UI, data, API), one short bullet per layer. Do not walk through files.

**Root cause.** Why it broke, not just where, and how this fix addresses it. Say so if the same defect could exist elsewhere.

**Breaking changes, Migration, Rollout.** Who is affected and what they must do. For migrations: what happens to existing data, and rollback or compatibility. For rollout: flags, config, secrets, entitlements, deploy order, backfills, in the order they must happen.

**Copy & translations.** Keys added or changed, translation status (machine or reviewed), and wording worth a native-speaker look.

**Performance.** Before and after numbers, how they were measured, on what.

**Review notes.** Things that need a reviewer's judgement:
- **Where to look hardest**: the subtle or risky part.
- **A over B: why**: only alternatives actually weighed, as seen in the conversation, ticket or commits. Never invent a road not taken.
- **Failures that exist without this PR**, with the evidence ("fails identically on an unmodified base checkout") or "not checked".
- Security, privacy and dependency changes: permissions, data sent elsewhere, packages added or bumped and why.
- **Follow-ups** with ticket keys, and anything deliberately left out.

**Testing.** Never "tested locally". Each line is `- [x] command or check: result (environment)`, and each unrun check is `- [ ] Not run: what (why)`. What the section looks like depends on the work:
- Bug fix: the regression test that fails before and passes after, plus steps that reproduced the bug.
- Refactor: say "no behaviour change" and how that was shown.
- Docs or config only: one line.
- Behaviour change: a **Reproduce** block with the environment, build configuration and parameters, then the main path in about 7 steps or fewer.
- UI change: a Before/After table. Embed images or URLs you were given. Agents cannot upload images to a PR body, so otherwise fill the cells with `_pending_` and add "attach screenshots" to the open items. Never imply they are attached.
- Long QA checklist (edge cases, locales, devices): put it in `<details><summary>Full QA checklist</summary>` at the end of the section.

## Keep it short

Aim for about 40 visible lines, not counting `<details>`. A small change needs a sentence or two per section, but keep the headings so the shape stays recognisable. Cut first:
- file-by-file tours, since the reviewer has the diff
- the story of how you got there ("first I tried"), since the decision belongs in the description but the process does not
- adjectives that advertise ("robust", "clean"), and claims nobody can check
- logs and full test output, since a command and its result fit on one line

## Title

Follow the project's format. Otherwise: imperative mood, sentence case, no trailing period, about 70 characters at most, naming the outcome, with the ticket key as a prefix when the branch has one. Stacked series get `(k/n)`. It must read well alone in `git log`. Leave "WIP" and "draft" out; draft is a flag on the PR.

## Rewriting an existing PR

Describe the final state of the branch as if writing it fresh. Drop "after review", "rebased onto" and other history, which belongs in comments and commits. Compare each tick with `headRefOid`: a check from an older commit is unticked, or labelled with the commit it ran on. Keep what you did not write (screenshots, reviewer-requested notes, bot sections) and mention in the open items that you did.

## Style

Plain words and specific counts ("438 tests pass"). Identifiers and commands in backticks. Do not hard-wrap prose, since GitHub renders single newlines as line breaks. No "This PR…", filler or emoji.

## Examples

Small change, the spine only:

````markdown
Title: Fix typo in rate-limit error message

Corrects "recieved" to "received" in `RateLimitError.message`.

## Summary
The message shown to users when a request is throttled was misspelled. No behaviour change.

## Testing
- [x] `npm test -- rate-limit`: 12 pass
````

Bug fix with a migration, where Root cause, Migration, Rollout and Review notes apply:

````markdown
Title: PROJ-212: Stop duplicate invoices when a payment webhook is retried

Retried payment webhooks no longer create a second invoice for the same order.

Jira: [PROJ-212](https://example.atlassian.net/browse/PROJ-212)

## Summary
Customers were billed twice when the payment provider retried a webhook after a timeout. Invoices are now unique per `(order_id, provider_event_id)`.

## Root cause
The handler inserted an invoice before checking whether the event had been seen. Two retries inside the same second both passed the check. The unique index makes the database reject the second insert, and the handler treats that rejection as success.

## Migration
Adds a unique index on `invoices(order_id, provider_event_id)`. Existing duplicates (41 in production, listed in PROJ-212) must be merged first, or the index build fails. Rollback: drop the index; no data changes.

## Rollout
1. Run `scripts/merge_duplicate_invoices.py --apply` against production.
2. Deploy. The migration runs on boot.

## Review notes
- **Unique index over an application-level lock:** the lock needs shared state between workers, and the index holds even if a worker crashes mid-request.
- Look hardest at `handleWebhook`: a rejected insert is now a no-op, not an error.
- Follow-up: PROJ-230 alerts on the rejection rate.

## Testing
- [x] `pytest tests/webhooks -k duplicate`: new test fails on `main`, passes here
- [x] CI green on `4f2a9c1`
- [ ] Not run: the merge script against a production copy (no access)

**Reproduce** (local, Postgres 15)
1. `make db-reset && make dev`
2. `curl -X POST localhost:8000/webhooks/pay -d @fixtures/paid.json` twice, back to back
3. `SELECT count(*) FROM invoices;` returns 1 (2 on `main`)
````

Open items for that example: "Merge-script dry run still outstanding" and "Confirm the 41 duplicate count is current".
