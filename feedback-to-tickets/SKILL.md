---
name: feedback-to-tickets
description: Convert a dump of user/beta feedback into de-duplicated Jira tickets, epics, and spikes, filed under the right epic. Use when the user wants to turn feedback into Jira issues, triage TestFlight/beta feedback, or file user feedback as tickets.
---

Turn a raw text dump of user/beta feedback into a clean, de-duplicated set of Jira issues —
checking for existing tickets before creating new ones, filing everything under the right
epic, and never writing to Jira without the user's sign-off on the plan first.

## Defaults (apply unless the user overrides them)

- **Approval-first.** Build a proposed mapping table and present it. Do not create, comment
  on, or edit any Jira issue until the user approves. Creating tickets is hard to undo.
- **Duplicates get a comment, not a description edit.** When feedback matches an existing
  ticket, add it as a corroborating comment (`Beta feedback corroborates this: "<quote>"`).
  Skip entirely only if the ticket already fully captures it with nothing new to add. Add the
  batch label to the existing ticket too.
- **Match on meaning, not wording.** Confident semantic matches are acted on automatically.
  Borderline matches are marked `possible dup of KEY` in the table for the user to decide.
  Feedback that matches a ticket already **Done** or **Won't Do** is flagged as a likely
  **regression / reopen candidate** — never silently recreated as a new ticket.
- **Classify issue type per item:** broken/incorrect behavior → **Bug**; feature request or
  enhancement → **Story**; chore/config/non-user-facing → **Task**; **large or open-ended
  scope with no clear implementation path → Spike** (an investigation, not a commitment to
  build). Show the pick in the table so the user can override.
- **Group within-batch duplicates.** If multiple feedback items describe the same underlying
  issue, merge them into one ticket and include each distinct quote.
- **Non-actionable feedback gets no ticket, but gets listed.** Praise, too-vague, or
  out-of-scope items go in the table under "no ticket" with a reason — nothing is silently
  dropped. Vague-but-plausible items are flagged for the user to clarify.
- **Descriptions = restatement + raw quote.** Write a clear problem statement (or repro, for
  bugs), then include the original wording verbatim in a blockquote:
  `> Beta feedback: "<quote>"`.
- **Label everything from the batch.** Every ticket created or commented on gets a shared
  label (default `beta-feedback`) so the whole batch is queryable in one JQL.

## Process

1. **Resolve the environment.** Call `getAccessibleAtlassianResources` for the cloudId, then
   `getVisibleJiraProjects` (`action: "create"`, `expandIssueTypes: true`) for the project key
   and available issue types. If working in a known environment (see appendix below), you can
   skip straight to using those values — but re-verify if anything seems off (e.g. an epic
   key resolves to the wrong summary).

2. **Build the dedup surface.** Pull existing epics (`issuetype = Epic`) so you know the
   routing targets and can spot a fallback/misc epic. You don't need to load the entire
   backlog up front — targeted searches per topic (step 4) are cheaper and avoid token limits.

3. **Parse the feedback** into discrete items. Note which ones overlap or describe the same
   underlying request — those get grouped per the defaults above.

4. **Dedup-match each item.** For each feedback item (or group), run a targeted
   `searchJiraIssuesUsingJql` text search across the *whole* project — not just open issues —
   e.g. `project = X AND text ~ "trailer"`. Check both open and closed/Done/Won't-Do issues.
   Only request the fields you need (see gotchas below). Classify each item: **NEW**,
   **DUP-of-KEY**, **POSSIBLE-DUP**, **REGRESSION**, or **NO-TICKET**.

5. **Present the mapping table and stop.** One table, columns:
   `# | Feedback (short) | Decision | Epic | Type | Proposed summary | Notes`.
   Surface every possible-dup, regression, and no-ticket row explicitly with reasoning.
   Do not proceed until the user approves or edits it.

6. **Apply after approval.**
   - New tickets: `createJiraIssue` (see gotchas for `parent` and `labels`).
   - Duplicates: `addCommentToJiraIssue` with the quote, then `editJiraIssue` to add the
     batch label to the existing ticket.
   - Only act on regression/possible-dup rows per what the user decided in the table — never
     re-create something they flagged as already covered.

7. **Report back.** List tickets created (with keys and epics), tickets commented on,
   regressions/possible-dups flagged, and items intentionally left without a ticket (with
   reasons). Give the user a verification JQL, e.g.
   `project = X AND labels = "beta-feedback" ORDER BY created DESC`.

## Tool reference & gotchas

- The Atlassian MCP tools are usually deferred — load them first with
  `ToolSearch("select:getAccessibleAtlassianResources,getVisibleJiraProjects,searchJiraIssuesUsingJql,createJiraIssue,addCommentToJiraIssue,editJiraIssue")`
  (or a keyword search) before calling them.
- **`mcp__atlassian-rovo__search` (Rovo full-text search) may 403 with "app not installed"**
  even though it's listed. Don't rely on it — use `searchJiraIssuesUsingJql` with `text ~`
  clauses instead; it works even without the Rovo app.
- JQL results are often large and get saved to a file instead of returned inline. Reduce this
  by requesting only the fields you actually need, e.g.
  `fields: ["summary","status","issuetype","parent"]` instead of the default field set.
  When a result *does* spill to a file, parse it with `jq` or `python3` rather than reading it
  raw — and check the shape first: some tool outputs wrap the real JSON as
  `[{"type":"text","text":"<json-encoded string>"}]`, which needs an extra decode step before
  a normal `jq '.issues[]'` will work.
- In a **next-gen (team-managed/simplified)** Jira project, file a new issue under an epic by
  passing the epic key as **`parent`** to `createJiraIssue` — there's no separate epic-link
  field to set.
- Labels, priority, components, and other non-core fields go through **`additional_fields`**
  on `createJiraIssue`, e.g. `additional_fields: {"labels": ["beta-feedback"]}`. There is no
  top-level `labels` parameter.
- To label an *existing* ticket, use `editJiraIssue` with `fields: {"labels": [...]}` — this
  **replaces** the label array, so include any labels you want to keep.
- `Spike` is a distinct issue type in Jira Software projects (alongside Task/Bug/Story) — use
  it explicitly for open-ended investigations rather than forcing them into Task or Story.

## Known environment: Credits (abeldemoz.atlassian.net)

Fast-path defaults for this project — treat as a snapshot to verify, not gospel, since epics
and keys can change:

- **cloudId:** `9372fe21-40b6-4b0d-8cff-f0b68ae5d491`
- **Project:** `CRED` (Credits), single project on the site, next-gen/simplified.
- **Issue types:** Task, Bug, Story, Spike, Epic, Subtask.
- **Fallback epic:** `Misc` = **CRED-22** — file here when nothing else fits.
- **Epic routing map** (non-exhaustive, re-check for new epics):
  Movie Details `CRED-1` · TV Show Details `CRED-2` · Tech Debt `CRED-21` · Preferences
  `CRED-26` · Discovery: Custom Categories `CRED-39` · Search `CRED-47` · Person `CRED-48` ·
  Credits `CRED-71` · Season Details `CRED-74` · Episode details `CRED-87` · Watch options
  screen `CRED-96` · DevEx `CRED-100` · Watchlist `CRED-122` · Saved `CRED-126` · Testing
  `CRED-151` · Onboarding Flow `CRED-156` · Durable persistence for Saved Items `CRED-166` ·
  Prepare Credits for TestFlight `CRED-174` · Business & Legal / Monetization readiness
  `CRED-186` · Settings `CRED-206` · About screen `CRED-220` · CloudKit sync for Saved Items
  `CRED-243`.
