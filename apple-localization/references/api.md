# Localization APIs

Two call styles exist and the skill supports both equally (`SKILL.md` §0). Use the project's; if it has none, ask the user and pick with this table:

| | Generated symbols | Extraction |
|---|---|---|
| Call site | `String(localized: .errorViewRetryButton)` | `String(localized: "errorView.retryButton", defaultValue: "Try again", bundle: #bundle, comment: "…")` |
| Entry is written | First, by hand, in the catalog | By the build, from the call |
| Typo in a key | Compile error | Silent: a new key appears, the intended one is untouched |
| Placeholders | Typed function parameters | Interpolation in `defaultValue:` |
| Bundle | Resolved by the symbol | You pass it (`#bundle`) |
| Setup cost | `STRING_CATALOG_GENERATE_SYMBOLS`, Xcode 26+ | None beyond a catalog |
| Cost per string | A catalog edit before the call compiles | A long call, usually wrapped in a helper |

Don't mix the two in one target.

## Generated symbols

Xcode generates a typed symbol for every catalog entry at build time. Write the entry first (`references/string-catalogs.md`), build, then call it:
```swift
String(localized: .errorViewRetryButton)            // key "errorView.retryButton", Localizable table
String(localized: .Shared.errorViewRetryButton)     // same key in Shared.xcstrings
Text(.Search.screenTitle)                           // SwiftUI
LocalizedStringResource.Search.screenTitle          // the resource itself
```

**Naming.** The symbol is `.<Table>.<camelCasedKey>`. The default `Localizable` table adds no namespace, and dots in a key flatten into camelCase (`errorView.retryButton` → `errorViewRetryButton`) rather than becoming nested types.

**Placeholders become parameters.**

| Value in the catalog | Generated symbol | Call |
|---|---|---|
| `No results for "%@".` | function, one unlabeled parameter | `.Search.noResultsMessage(query)` |
| `%1$@ • %2$@` | function, two unlabeled parameters, in argument order | `.Search.mediaRowSubtitle("Movie", "1999")` |
| plural-varied `%lld item` | function taking an `Int` | `.Library.deleteItemCount(count)` |
| `%1$@ %#@moreCount@` with a `moreCount` substitution | function, the substitution is a **labeled** parameter | `.Shared.infoCardOverflowValue("Drama", moreCount: 3)` |

Use positional specifiers (`%1$@`, `%2$@`) whenever a translator might need to reorder arguments. Pass real typed values — an `Int`, not `String(count)` — so the catalog can apply plural rules.

**Bundles.** Pass no `bundle:`. In a Swift package or framework the generated symbol resolves its own bundle (`Bundle.module`), so it can't hit the `.main` pitfall described under extraction below.

**Setup.** `STRING_CATALOG_GENERATE_SYMBOLS = YES` at project level. A Swift package additionally needs `defaultLocalization` in `Package.swift`; see `references/string-catalogs.md`. Prove a package symbol resolves with a test (`SKILL.md` §5) the first time you add one.

## Extraction

The call is the source of truth; a build extracts its key into the catalog (`SWIFT_EMIT_LOC_STRINGS = YES`).
```swift
String(
    localized: "errorView.retryButton",       // key: semantic, stable across rewording
    defaultValue: "Try again",                // source-locale text; interpolate typed values here
    table: nil,                               // nil/omitted -> "Localizable" table
    bundle: #bundle,                          // see "Which bundle" below
    locale: .current,                         // almost always leave this at the default
    comment: "Button on the error view that retries the failed request."
)
```
Full signature (Foundation):
```swift
init(localized keyAndValue: String.LocalizationValue,
     table: String? = nil,
     bundle: LocalizedStringResource.BundleDescription = .main,
     locale: Locale = .current,
     comment: StaticString? = nil)

init(localized key: StaticString,
     defaultValue: String.LocalizationValue,
     table: String? = nil,
     bundle: Bundle? = nil,
     locale: Locale = .current,
     comment: StaticString? = nil)
```
There are two overloads because there are two key strategies (source-text keys vs. semantic keys + `defaultValue`); this skill uses the second. They take *different* bundle types (`LocalizedStringResource.BundleDescription` vs. plain `Bundle?`). Interpolate typed values into `defaultValue:` (`"\(count) episodes"`) so the catalog sees the `%lld`.

**`table:`** — omit it for the default `Localizable.xcstrings`. Pass `table: "Discover"` (creating/using `Discover.xcstrings`) when a feature's strings should live in their own catalog.

### Which bundle

| Context | Use |
|---|---|
| App target | `#bundle` (resolves to `Bundle.main`) or omit `bundle:` entirely |
| Swift package target | `#bundle`, or `.module` if the toolchain predates `#bundle` |
| Framework target | `#bundle`, or `Bundle(for: SomeTypeInThisModule.self)` as a fallback |
| Passing a `LocalizedStringResource` from a package/framework | `.atURL(#bundle.bundleURL)` — never leave it at the `.main` default |

`#bundle` is a `@freestanding(expression)` macro from Foundation that "expands to a bundle instance that's most likely to contain resources for the calling code," and back-deploys to older OS versions than its own availability would suggest. Prefer it over picking `.module` vs. `Bundle(for:)` by hand whenever the toolchain supports it.

**Pitfall:** `LocalizedStringResource`'s string-literal initializer (`"some key"` used directly where a `LocalizedStringResource` is expected) defaults `bundle:` to `.main`. Inside a package or framework that is the wrong bundle — the lookup fails at runtime and falls back to the raw key or default value with no compile-time warning. Construct it explicitly with `bundle: .atURL(#bundle.bundleURL)` (or `.forClass(SomeTypeInThisModule.self)`).

## `LocalizedStringResource` — passing a localizable string around

Use it when a type needs to *hold* a localizable string before it's rendered, or when a test needs to resolve a string in a chosen locale:
```swift
struct RetryPromptView: View {
    let title: LocalizedStringResource
    var body: some View { Text(title) }
}
RetryPromptView(title: .errorViewRetryButton)             // generated symbols
RetryPromptView(title: LocalizedStringResource(           // extraction
    "errorView.retryButton", defaultValue: "Try again",
    bundle: .atURL(#bundle.bundleURL),
    comment: "Button on the error view that retries the failed request."
))

var resource = LocalizedStringResource.errorViewRetryButton
resource.locale = Locale(identifier: "de")
String(localized: resource)        // German copy, regardless of the device locale
```
If the project's architecture keeps scene models free of display strings, don't store resources in them either — the view maps a typed enum case to a string.

## SwiftUI: `Text`, `Button`, and the `LocalizedStringKey` trap

A **string literal** passed to `Text`/`Button`/`Label`/etc. is inferred as `LocalizedStringKey` and looked up in the catalog (in an extraction-style project this is also what extraction picks up). A **`String` variable** passed to the same initializer resolves to the `Text(verbatim:)` overload and is *not* localized, even though the code looks identical at the call site:
```swift
let title = "Settings"
Text(title)          // NOT localized — silently shows raw English on every locale
Text("Settings")     // localized — literal, inferred as LocalizedStringKey
Text(.settingsTitle) // localized — generated symbol
```
This is the most common silent SwiftUI localization bug. If you're building the string dynamically, pass a generated symbol, a `LocalizedStringResource`, or the *result* of `String(localized:)` — don't pass a plain interpolated `String` and hope. Use `Text(verbatim:)` deliberately for content that must never be localized: preview/mock data, user-entered text, raw API values.

## `AttributedString(localized:)` — inline emphasis and links

When copy needs bold, italic, or a link, don't hand-roll `NSAttributedString` ranges — write Markdown in the localized value:
```swift
let confirmation = AttributedString(
    localized: "checkout.confirmationMessage",
    defaultValue: "Your order is **complete**!",
    comment: "Confirmation banner after an order finishes."
)
```
The `**bold**`/`*italic*`/`` `code` ``/`[text](url)` markup survives translation as structure — a translator moves the asterisks to wherever their language's word order puts the emphasized word. The example is the extraction form; in a generated-symbol project, put the same Markdown in the catalog value and check that a symbol-taking `AttributedString(localized:)` compiles on the deployment target.

## Relevant build settings

| Setting | What it does | When to check it |
|---|---|---|
| `SWIFT_EMIT_LOC_STRINGS` | Whether the Swift compiler extracts `String(localized:)`/`Text(...)` calls into catalogs at all | Extraction style: `YES` on any target whose Swift should be extracted; `NO` is correct for test targets |
| `LOCALIZATION_PREFERS_STRING_CATALOGS` | Makes `Export Localizations` / `xcodebuild -exportLocalizations` emit String-Catalog-flavored XLIFF instead of legacy `.stringsdict` XLIFF | Set `YES` once the project has moved off `.strings`/`.stringsdict` |
| `STRING_CATALOG_GENERATE_SYMBOLS` | Turns on the generated-symbols workflow project-wide | Generated-symbols style only |
| `LOCALIZED_STRING_MACRO_NAMES` | Registers a **custom C/Objective-C macro** (not Swift) as an extraction source, for a wrapper like `#define MyLocalizedString(key, comment) ...` | Only for Objective-C/C code with a custom wrapper macro — Swift extraction doesn't use it |
