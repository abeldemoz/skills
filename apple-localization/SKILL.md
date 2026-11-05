---
name: apple-localization
description: Make Swift/UIKit/SwiftUI code localizable (i18n) for iOS, iPadOS, macOS, watchOS and visionOS apps — turning hardcoded strings into String Catalog entries with keys, comments, real plurals and locale-safe formatting. Writes only the source language; not for translating into other languages. Use this whenever the user says localize, localise, internationalize, i18n, l10n, or asks to audit an app for localization issues. Also apply it proactively, without being asked, any time you write or edit user-facing text in Swift UI code (labels, buttons, alerts, accessibility labels, empty states, Info.plist prompts) — new copy should be born localizable, not added to the backlog.
---

# Apple platform localization

Localizing an app is two separate jobs. **Internationalization (i18n)** is making code *capable* of showing translated text — extracting strings into a catalog, giving them stable keys and comments, fixing plurals and concatenation, using formatters instead of hand-built strings. **Translation (l10n)** is writing the actual German, Japanese, Arabic, etc. This skill does only the first job.

**Hard rule: this skill writes only the source-language value of each catalog entry. It never adds a language, writes a translation, or edits or removes another locale's values.** If a request also asks for translation ("localize into German", "add French"), do the source-side work, then say plainly that translation is out of scope.

So this skill's job, in order: **find hardcoded/fragile strings → make them localizable → verify the catalog picked them up.**

## 0. Orient before touching anything

Read the project's `AGENTS.md`/`CLAUDE.md` (root and any nested one covering the files you're touching) — a project's own conventions always beat this skill's defaults. Then detect what the project already does, because retrofitting a second convention onto a codebase is worse than picking either one consistently:

```bash
# Generated symbols in use? (Xcode 26+ workflow)
grep -rn 'String(localized: \.\|LocalizedStringResource\.[A-Z]' --include='*.swift' .

# Extraction in use? (the default this skill reaches for)
grep -rn 'String(localized: "' --include='*.swift' .

# A custom wrapper already exists? Don't invent a second one.
grep -rnE '\bNSLocalizedString\(|func LocalizedString\(' --include='*.swift' .

# What catalogs/tables/bundles already exist?
find . -name '*.xcstrings' -o -name '*.strings' -o -name '*.stringsdict'
```

If the project has an established pattern (a key grammar, a helper function, a place copy lives — e.g. a `<SceneModel>+Formatting.swift` convention), **follow it exactly** instead of the defaults below. The defaults below are for greenfield decisions the project hasn't made yet.

## 1. Pick the mode

- **(A) Writing or editing any user-facing text.** Do this inline, without being asked — see §2–3. This is the common case: you're building a screen and it needs a label.
- **(B) "Localize `<file/screen/package>`."** Migrate that scope: every hardcoded string in it becomes a real catalog entry. See §2–5.
- **(C) "Localize the app" / a whole codebase with no existing localization.** Don't do this in one pass — see §6.
- **(D) "Audit for localization issues."** Report only, no edits — see §7.

## 2. The rules for every string

**Key grammar:** `<surface>.<owner>.<specifier>[.<specifier>…]`
- `surface` is `ui` for visible copy or `accessibility` for VoiceOver-only copy.
- `owner` is the screen or component name, lowerCamelCase. If the project already names things for accessibility identifiers (Credits does: `ui.<owner>.<element>`, `owner` = type name minus `ViewController`/`View`), reuse that exact `owner` — one vocabulary for "what screen is this" beats two.
- After `owner`, add **as many specifiers as it takes to disambiguate and no more** — there's no fixed slot count or fixed last-segment "role." `ui.errorView.retryButton`, `ui.delayCompensationRanges.header.title`, `accessibility.mediaRow.label` are all valid shapes.
- One English word needing different translations in different places (a "Save" *button* vs. a "Save" *file*, an "Archive" *folder* vs. the *act* of archiving) is exactly what separate keys are for — never rely on one key doing double duty.

**Call shape (extraction, the default):**
```swift
let title = String(
    localized: "ui.errorView.retryButton",
    defaultValue: "Try again",
    bundle: #bundle,
    comment: "Button on the error view that retries the failed request."
)
```
- The **key is semantic, not the English text** — the reasons: (1) fixing a typo or rewording the English doesn't orphan every existing translation, it just re-flags them "Needs Review" so a translator revisits them deliberately; (2) it's how you disambiguate the same English word above.
- `defaultValue:` carries the actual English shown in this source locale.
- **Interpolate typed values, never pre-stringified ones.** `defaultValue: "\(count) episodes"` lets the catalog see the `%lld` and apply real plural rules later. `defaultValue: "\(String(count)) episodes"` or building the count into a display string upstream (in a presenter, say) throws that information away before it reaches the string — see `references/patterns.md` for the fake-plural failure mode this causes.
- If the project already uses generated symbols (detected in §0), the call is `String(localized: .errorViewRetryButton)` instead (`.Shared.errorViewRetryButton` for a key `errorView.retryButton` in `Shared.xcstrings`), and the catalog entry is written by hand first. The symbol is `.<Table>.<camelCasedKey>`; the default `Localizable` table adds no namespace, and dots in a key flatten into camelCase rather than nesting. In a Swift package the symbol resolves `Bundle.module` itself, so pass no `bundle:`. Read `references/api.md` for that shape — don't mix the two styles in one target.

**Bundle:** use `#bundle` (Foundation macro, Xcode 16+/toolchains that support it) — it resolves to the right bundle whether the code runs in the app, a framework, or a Swift package, and it back-deploys to older OS versions. If the toolchain doesn't support `#bundle`, use `.module` inside a Swift package or `Bundle(for: SomeTypeInThisModule.self)` inside a framework. Never default to `Bundle.main` from a package or framework target — that silently fails to find the string at runtime, not at compile time. `references/api.md` has the pitfall version of this with `LocalizedStringResource` specifically, which defaults its own `bundle:` to `.main` unless you override it.

**Comments are not optional.** A translator sees the key and the English in isolation, with none of the surrounding UI. The comment is their only context, so it should say what the string *is* (button, label, alert title, section header) and what any placeholder holds:
```swift
comment: "Alert title shown after a delay claim is submitted. %@ is the train operator's name."
```
A comment that just restates the English (`comment: "Retry"`) is worse than no comment — it costs a line without adding information.

**Placement:** follow the project's existing pattern if it has one (check §0). Absent one, put a screen's or component's copy in a `private extension String` block at the bottom of the file that owns the UI, one property or function per string:
```swift
private extension String {
    static var retryButtonTitle: String {
        String(localized: "ui.errorView.retryButton", defaultValue: "Try again",
               bundle: #bundle, comment: "Button on the error view that retries the failed request.")
    }
}
// call site: configuration.title = .retryButtonTitle
```
This keeps call sites short and keeps a screen's copy in one scannable place, without scattering it into a separate strings file disconnected from the view that uses it.

**Never localize:** accessibility *identifiers* (they're for test lookup, not display — see the project's own accessibility-identifier rules if it has them), log messages, analytics event names, values that come from an API (movie titles, usernames), brand/product names, or SwiftUI preview/mock data (use `Text(verbatim:)` for the latter so preview content doesn't clutter the catalog).

**Never rewrite English copy while localizing.** A localization change moves the existing text into the catalog unchanged; copy edits go in a separate change.

## 3. Hazards to fix while you're in a string

These come straight from Apple's own localization guidance (WWDC21 "Streamline your localized strings", WWDC22 "Build global apps"). Each one looks harmless in English and breaks in at least one shipping language. Full before/after examples: `references/patterns.md`.

- **Concatenation.** `"\(typeLabel) · Official"` assumes English word order and an English separator. Write the whole sentence as one localizable string with the pieces interpolated in, so a translator can reorder them.
- **Fake plurals.** `count == 1 ? "1 Episode" : "\(count) Episodes"` is wrong in every language with more than two plural forms (Russian has four; Arabic has six) and is invisible to Xcode's tooling, so it never gets fixed. Interpolate the count into one `defaultValue` and vary the *catalog* entry by plural (§4) — never branch in Swift.
- **`String(format:)` and hand-rolled formatting.** Loses locale-aware digit systems, RTL isolation, and grouping. Use `String(localized:)` for text and Foundation's `.formatted()` for numbers, dates, currency, lists, and durations — see `references/patterns.md` for the formatter reference.
- **One string glued from a verb and a variable that changes grammar.** `"Show weather in \(location)"` reads fine for a city but wrong for "My Location" in languages where the preposition depends on what follows. When two call sites feed genuinely different grammar into the same template, split them into two full strings.
- **Count-dependent phrasing with no number in it.** `"Remove this city"` vs `"Remove these cities"` isn't a plural-rules problem (there's no `%lld` to vary on) — it's just two different strings, chosen in code by the count.
- **Server/remote content matched to the wrong locale.** Match remote content to the app's own resolved localization (`Bundle.preferredLocalizations(from:)`), not the raw device locale — otherwise the UI can end up part English, part something else.

## 4. Making the catalog changes stick

Editing Swift is half the job — the String Catalog has to actually pick the string up, and plural/device variations have to be added to the *source* locale before there's anything for a translator to vary. Full mechanics, JSON shapes, and the Xcode-MCP-first-JSON-fallback rule: `references/string-catalogs.md`. In short:

1. Build so Xcode's extractor discovers new/changed keys (`mcp__xcode__BuildProject`, or the project's own build command from its `AGENTS.md`).
2. If a string needs plural or device variation, add that to the **source locale** — `mcp__xcode__StringCatalogEdit` with `targetLocaleIdentifier` set to the source language, or hand-edit the `.xcstrings` JSON if Xcode's MCP isn't available (tell the user to close that catalog in Xcode first so it doesn't clobber the edit).
3. Re-check with `mcp__xcode__StringCatalogRead`/`StringCatalogContext` (or by opening the JSON) that the key exists with the right default value, comment, and variation — and that keys you removed are deleted from the catalog too, unless another locale has a value for them. Leave those in place and name them in your summary.

## 5. Layout, RTL, and Dynamic Type

Localizable text and localizable layout are different bugs — a perfectly extracted string still breaks if its container is a fixed-width `UILabel` or a hardcoded `.leading`-as-`.left`. Read `references/layout.md` when touching layout for: leading/trailing vs. left/right, flexible container sizing, RTL mirroring rules (what mirrors automatically vs. what needs `semanticContentAttribute`), Dynamic Type headroom, and optional pseudolanguage launch arguments for manual testing.

## 6. Localizing a whole app (mode C)

Don't migrate 250 strings in one diff. Two-phase:

1. **Setup + inventory, one pass.** Add `defaultLocalization: "en"` to the relevant `Package.swift`(s), create an empty `Localizable.xcstrings` (`{"sourceLanguage":"en","strings":{},"version":"1.0"}`) if none exists, confirm build settings (`references/string-catalogs.md`). Then produce an inventory table: one row per feature/owner, with a rough count of unlocalized strings and any hazards from §3 you spotted while scanning (fake plurals, concatenation, etc.). Don't fix anything yet.
2. **Migrate feature by feature.** Take the features in the order that makes sense (smallest/highest-traffic first is a reasonable default, but ask if it's not obvious). For each feature: apply §2–4, build, verify the catalog, then **stop and report** before starting the next feature — each feature is naturally its own PR, and a human should see one land before the next starts.

## 7. Audit mode (mode D) — report, don't edit

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
Present findings grouped by file/feature with a one-line description of each hazard type present (see §3) — no fix, no catalog changes.
