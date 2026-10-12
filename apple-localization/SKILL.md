---
name: apple-localization
description: Make Swift/UIKit/SwiftUI code localizable (i18n) for iOS, iPadOS, macOS, watchOS and visionOS apps — turning hardcoded strings into String Catalog entries with keys, comments, real plurals and locale-safe formatting, then handing off to Xcode's own translation tools. Use this whenever the user says localize, localise, internationalize, i18n, l10n, "add a language", "translate the app", or asks to audit an app for localization issues. Also apply it proactively, without being asked, any time you write or edit user-facing text in Swift UI code (labels, buttons, alerts, accessibility labels, empty states, Info.plist prompts) — new copy should be born localizable, not added to the backlog.
---

# Apple platform localization

Localizing an app is two separate jobs. **Internationalization (i18n)** is making code *capable* of showing translated text — putting strings into a catalog with stable keys and comments, fixing plurals and concatenation, using formatters instead of hand-built strings. **Translation (l10n)** is writing the actual German, Japanese, Arabic, etc. Xcode 27+ already ships purpose-built skills for the second job (`translation-coordinator`, `translation` — see §9). This skill exists because nothing else covers the first job, and the second job is worthless without it: Xcode can only translate strings that are already in a catalog.

So this skill's job, in order: **find hardcoded/fragile strings → put them in a catalog with a stable key and a comment → call them in the project's own style → verify → hand off to translation.**

## 0. Orient before touching anything

Read the project's `AGENTS.md`/`CLAUDE.md` (root and any nested one covering the files you're touching) — a project's own conventions always beat this skill's defaults. Then detect what the project already does, because retrofitting a second convention onto a codebase is worse than picking either one consistently:

```bash
# Generated symbols in use?
grep -rnE 'String\(localized: \.|LocalizedStringResource\.[A-Z]|Text\(\.' --include='*.swift' .

# Extraction-style calls in use?
grep -rn 'String(localized: "' --include='*.swift' .

# A custom wrapper or legacy API already exists? Don't invent a second one.
grep -rnE '\bNSLocalizedString\(|func LocalizedString\(' --include='*.swift' .

# What catalogs/tables/bundles already exist?
find . -name '*.xcstrings' -o -name '*.strings' -o -name '*.stringsdict' | grep -v '/\.build/'
```

**Call style.** Two are supported equally; use whichever the project uses, and never mix them in one target:

| Style | Call site | Where the string's source of truth is |
|---|---|---|
| **Generated symbols** | `String(localized: .errorViewRetryButton)` | The catalog entry, written *first*; Xcode generates the symbol from it |
| **Extraction** | `String(localized: "errorView.retryButton", defaultValue: "Try again", bundle: #bundle, comment: "…")` | The Swift call; a build extracts the key into the catalog |

A custom wrapper (`NSLocalizedString`, a project helper) means follow the wrapper. If the project has **no localization at all**, there is no convention to follow: **ask the user which style they want** and continue in that style. The trade-offs are in `references/api.md`. Say which style you used in your summary.

Then learn the rest of the conventions from the catalogs themselves, not from guesses:
- **Table layout:** one shared table, or one per screen/feature plus a `Shared` table? Which table does a string like yours belong in?
- **Key shape:** read ten keys from the closest existing table and match them.
- **Locales:** which locales do the existing entries already carry? That is the set new copy must be translated into (§1, mode A). `CFBundleLocalizations` in the app's Info.plist lists them too.

If the project has an established pattern (a table layout, a key grammar, a place copy lives — e.g. a `<SceneModel>+Formatting.swift` convention), **follow it exactly** instead of the defaults below. The defaults are for decisions the project hasn't made yet.

## 1. Pick the mode

- **(A) Writing or editing any user-facing text.** Do this inline, without being asked — see §2–4. This is the common case: you're building a screen and it needs a label. If the catalog already carries non-source locales, finish by translating just the new keys into those locales (§9) — a translated app never ships an English-only string.
- **(B) "Localize `<file/screen/package>`."** Migrate that scope: every hardcoded string in it becomes a real catalog entry. See §2–5.
- **(C) "Localize the app" / a whole codebase with no existing localization.** Don't do this in one pass — see §7.
- **(D) "Audit for localization issues."** Report only, no edits — see §8.
- **(E) "Add a language" / "translate to X."** This is translation, not internationalization. Confirm the target strings are already in a catalog (mode B/C first if not), then read `references/translation-handoff.md` and follow it — don't attempt this yourself with locale knowledge you're guessing at.

## 2. The rules for every string

These hold in either call style. The project's own convention wins over every default here.

**Tables:** follow the project's layout; with none, use a single `Localizable.xcstrings`. In the generated-symbol style a non-default table namespaces the symbol (`.Shared.errorViewRetryButton`); in the extraction style it is the `table:` argument.

**Key grammar (default when the project has none):** `<owner>.<element><Role>`
- `owner` is the screen or component name, lowerCamelCase. If the project already names things for accessibility identifiers, reuse that exact `owner` — one vocabulary for "what screen is this" beats two.
- `element` names what the string is attached to (`retryButton`, `emptyState`); `Role` says what kind of copy it is and is **required**: `Title`, `Label`, `Message`, `Button`, `Option`, `Action`, `Placeholder`, `Footer`, and for VoiceOver copy `AccessibilityLabel`/`AccessibilityHint`/`AccessibilityValue` (`rowMenuButton.accessibilityLabel`, `video.accessibilityLabel`).
- No `ui.`/`accessibility.` prefix; the role carries that distinction.
- In a per-screen table the table name already is the owner, so keys drop it: `Search.xcstrings` holds `screenTitle`, `noResultsMessage`. Shared tables keep the owner: `errorView.retryButton`. A `Shared` table holds only strings whose meaning and placement are identical everywhere.
- One English word needing different translations in different places (a "Save" *button* vs. a "Save" *file*, an "Archive" *folder* vs. the *act* of archiving) is exactly what separate keys are for — never rely on one key doing double duty.
- The **key is semantic, not the English text**: rewording the English re-flags existing translations "Needs Review" for a translator to revisit deliberately, instead of orphaning them.

**Interpolate typed values, never pre-stringified ones.** A count passed as an `Int` lets the catalog see the `%lld` and apply real plural rules (§3). `String(count)`, or building the count into a display string upstream (in a presenter, say), throws that information away before it reaches the string — see `references/patterns.md`.

**Comments are not optional.** A translator sees the key and the English in isolation, with none of the surrounding UI. The comment is their only context, so it should say what the string *is* (button, label, alert title, section header), where it appears, and what each placeholder holds:
```
"Alert title shown after a delay claim is submitted. %@ is the train operator's name."
```
A comment that just restates the English (`"Retry"`) is worse than no comment — it costs a line without adding information.

**Never localize:** accessibility *identifiers* (they're for test lookup, not display), log messages, analytics event names, values that come from an API (movie titles, usernames), brand/product names, or SwiftUI preview/mock data (use `Text(verbatim:)` for the latter so preview content doesn't clutter the catalog).

**Never rewrite English copy while localizing.** A localization change moves the existing text into the catalog unchanged; copy edits go in a separate change.

### Call shape — generated symbols

Write the catalog entry first (key, English value, comment), then call the symbol Xcode generates from it. The key only exists in the catalog, so a typo at the call site is a compile error, and a placeholder in the value becomes a typed function parameter.
```swift
String(localized: .errorViewRetryButton)             // Foundation
Text(.errorViewRetryButton)                          // SwiftUI
String(localized: .Search.noResultsMessage(query))   // key in Search.xcstrings, one placeholder
```
- The symbol is `.<Table>.<camelCasedKey>`; the default `Localizable` table adds no namespace, and dots in a key flatten into camelCase.
- Pass **no `bundle:`** — in a Swift package the symbol resolves `Bundle.module` itself.
- `%@`/`%lld` placeholders generate positional, unlabeled parameters in order (`.mediaRowSubtitle("Movie", "1999")`); a named plural substitution (`%#@moreCount@`) generates a labeled one (`.infoCardOverflowValue("Drama", moreCount: 3)`).
- **Placement:** call the symbol where the text is used; a `private extension String` wrapper adds nothing.

### Call shape — extraction

Write the call; the build extracts the key into the catalog.
```swift
let title = String(
    localized: "errorView.retryButton",
    defaultValue: "Try again",
    bundle: #bundle,
    comment: "Button on the error view that retries the failed request."
)
```
- `defaultValue:` carries the English shown in the source locale; interpolate typed values into it (`"\(count) episodes"`).
- **Bundle:** use `#bundle` (Foundation macro; resolves correctly in an app, framework, or package and back-deploys). If the toolchain lacks it, use `.module` in a package or `Bundle(for:)` in a framework. Never default to `Bundle.main` from a package or framework — the lookup fails at runtime, not compile time. `references/api.md` has the `LocalizedStringResource` version of this pitfall.
- **Placement:** follow the project. Absent a pattern, keep a screen's copy in a `private extension String` at the bottom of the file that owns the UI, one property or function per string, so call sites stay short:
```swift
private extension String {
    static var retryButtonTitle: String {
        String(localized: "errorView.retryButton", defaultValue: "Try again",
               bundle: #bundle, comment: "Button on the error view that retries the failed request.")
    }
}
// call site: configuration.title = .retryButtonTitle
```

Everything else about the API — `LocalizedStringResource`, the SwiftUI `Text(String)` trap, `AttributedString(localized:)`, `table:` — is in `references/api.md`.

## 3. Hazards to fix while you're in a string

These come straight from Apple's own localization guidance (WWDC21 "Streamline your localized strings", WWDC22 "Build global apps"). Each one looks harmless in English and breaks in at least one shipping language. Full before/after examples: `references/patterns.md`.

- **Concatenation.** `"\(typeLabel) · Official"` assumes English word order and an English separator. Write the whole sentence as one catalog entry with the pieces as placeholders, so a translator can reorder them.
- **Fake plurals.** `count == 1 ? "1 Episode" : "\(count) Episodes"` is wrong in every language with more than two plural forms (Russian has four; Arabic has six) and is invisible to Xcode's tooling, so it never gets fixed. Put the typed count into one string and vary the *catalog* entry by plural (§4) — never branch in Swift.
- **`String(format:)` and hand-rolled formatting.** Loses locale-aware digit systems, RTL isolation, and grouping. Use the catalog for text and Foundation's `.formatted()` for numbers, dates, currency, lists, and durations — see `references/patterns.md` for the formatter reference.
- **One string glued from a verb and a variable that changes grammar.** `"Show weather in \(location)"` reads fine for a city but wrong for "My Location" in languages where the preposition depends on what follows. When two call sites feed genuinely different grammar into the same template, split them into two full strings.
- **Count-dependent phrasing with no number in it.** `"Remove this city"` vs `"Remove these cities"` isn't a plural-rules problem (there's no `%lld` to vary on) — it's just two different strings, chosen in code by the count.
- **Server/remote content in a different language from the UI.** Decide deliberately which language remote content follows rather than letting each piece guess; a project may even define its own content-language setting separate from the UI language. If it follows the UI, match against the app's resolved localization (`Bundle.preferredLocalizations(from:)`), not the raw device locale.

## 4. Making the catalog changes stick

Full mechanics, JSON shapes, and the tooling caveats: `references/string-catalogs.md`. Whenever you hand-edit a catalog, **tell the user to close it in Xcode first** — Xcode holds its own in-memory copy and can overwrite a concurrent on-disk edit.

**Generated symbols** — the catalog is written first:
1. Add the entry to the `.xcstrings` JSON: key, comment, the English value, `"extractionState": "manual"`, `"state": "translated"`, plus plural variations or `%#@name@` substitutions for any string with a number in it.
2. Build (`mcp__xcode__BuildProject`, or the project's own build command from its `AGENTS.md`) so Xcode regenerates the symbols; the call site won't compile until it does.
3. Delete orphans by hand — manually managed entries are never marked Stale, so a key you stopped calling stays until you remove it.

**Extraction** — the call is written first:
1. Build so Xcode's extractor discovers the new/changed keys.
2. If a string needs plural or device variation, add it to the **source locale**: `mcp__xcode__StringCatalogEdit` with `targetLocaleIdentifier` set to the source language, or hand-edit the JSON if the Xcode MCP isn't available. `StringCatalogEdit` saves `machine_translated`; set the source entry back to `translated`.
3. Check that a key you removed isn't left behind as a Stale entry, and delete it if so.

**Either style**, finish by confirming the catalog still parses (`python3 -m json.tool <file> >/dev/null`) and the key exists with the right English value, comment, and variation (`mcp__xcode__StringCatalogRead`/`StringCatalogContext`, or open the JSON). Then, if the catalog carries other locales, translate the new keys (mode A, §9).

## 5. Prove it resolves

Add a test the first time a package or target is set up for localization, and for any string with a plural or substitution. Build a `LocalizedStringResource`, override its locale, and compare:
```swift
var resource = LocalizedStringResource.errorViewRetryButton      // generated symbols
// extraction style:
// var resource = LocalizedStringResource("errorView.retryButton", defaultValue: "Try again",
//                                        bundle: .atURL(#bundle.bundleURL))
resource.locale = Locale(identifier: "en")
#expect(String(localized: resource) == "Try again")
```
Repeat with one translated locale (`"de"`) to prove the bundle actually resolves translations — the failure mode where everything compiles but a package shows English everywhere. For plurals, assert both the `one` and `other` forms.

## 6. Layout, RTL, and Dynamic Type

Localizable text and localizable layout are different bugs — a perfectly extracted string still breaks if its container is a fixed-width `UILabel` or a hardcoded `.leading`-as-`.left`. Read `references/layout.md` when touching layout for: leading/trailing vs. left/right, flexible container sizing, RTL mirroring rules (what mirrors automatically vs. what needs `semanticContentAttribute`), Dynamic Type headroom, and optional pseudolanguage launch arguments for manual testing.

## 7. Localizing a whole app (mode C)

Don't migrate 250 strings in one diff. Two-phase:

1. **Setup + inventory, one pass.** Settle the call style with the user (§0), do the one-time setup in `references/string-catalogs.md` (package `defaultLocalization`, an empty `Localizable.xcstrings`, `CFBundleLocalizations`, and the build setting for the chosen style), and add the §5 test to prove a string resolves. Then produce an inventory table: one row per feature/owner, with a rough count of unlocalized strings and any hazards from §3 you spotted while scanning (fake plurals, concatenation, etc.). Don't fix anything yet.
2. **Migrate feature by feature.** Take the features in the order that makes sense (smallest/highest-traffic first is a reasonable default, but ask if it's not obvious). For each feature: apply §2–4, build, verify the catalog, then **stop and report** before starting the next feature — each feature is naturally its own PR, and a human should see one land before the next starts.

## 8. Audit mode (mode D) — report, don't edit

Grep for the hazards, don't touch code:
```bash
# Hardcoded UI text (heuristic — expect false positives, skim results)
grep -rnE '\.(text|title|placeholder) = "[A-Z]' --include='*.swift' .
grep -rnE 'UIAlertController\(title: "|Text\("[A-Z]' --include='*.swift' .

# Fake plurals
grep -rnE '== 1 \? "|count == 1' --include='*.swift' .

# Hand-built formatting
grep -rnE 'String\(format:' --include='*.swift' .
```
Also report any target where both call styles from §0 appear — mixed styles are a hazard in themselves. Present findings grouped by file/feature with a one-line description of each hazard type present (see §3) — no fix, no catalog changes.

## 9. Handing off to translation (mode E, and the end of mode A)

Xcode 27+ bundles two skills purpose-built for this — `translation-coordinator` (plans the work, splits it into batches, delegates) and `translation` (translates one batch, reading source, comments, similar strings, and a locale style guide before writing anything). They are not registered as Claude Code skills, but their instructions and the MCP tools they drive (`LocalizationPlanner`, `StringCatalogRead`, `StringCatalogContext`, `StringCatalogEdit`) are available. Read `references/translation-handoff.md` for exactly how to locate and follow them from here — including the scoped run for a handful of new keys — and don't improvise a translation workflow when Apple has already shipped one with 58 locale style guides behind it.
