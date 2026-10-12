# Localization APIs

## `String(localized:)` — the default call

```swift
String(
    localized: "ui.errorView.retryButton",   // key: semantic, stable across rewording
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
There are two overloads because there are two key strategies (source-text keys vs. semantic keys + `defaultValue`) — this skill defaults to the second. Note the two overloads take *different* bundle types (`LocalizedStringResource.BundleDescription` vs. plain `Bundle?`) — see the pitfall below.

## `AttributedString(localized:)` — for inline emphasis/links

When copy needs bold, italic, or a link, don't hand-roll `NSAttributedString` ranges — write Markdown in the localized value and load it as an `AttributedString`:
```swift
let confirmation = AttributedString(
    localized: "ui.checkout.confirmation",
    defaultValue: "Your order is **complete**!",
    comment: "Confirmation banner after an order finishes."
)
```
The `**bold**`/`*italic*`/`` `code` ``/`[text](url)` markup survives translation as structure — a translator moves the asterisks to wherever their language's word order puts the emphasized word, they don't need to preserve English character positions.

## SwiftUI: `Text`, `Button`, and the `LocalizedStringKey` trap

```swift
Text("ui.settings.title", bundle: #bundle, comment: "Navigation title for the Settings screen")
Button("ui.errorView.retryButton", bundle: #bundle, comment: "…") { retry() }
```
A **string literal** passed to `Text`/`Button`/`Label`/etc. is inferred as `LocalizedStringKey` and is automatically localizable — extraction picks it up like any other call. But a **`String` variable** passed to the same initializer resolves to the `Text(verbatim:)` overload and is *not* localized, even though the code looks identical at the call site:
```swift
let title = "Settings"
Text(title)          // NOT localized — silently shows raw English on every locale
Text("Settings")     // localized — literal, inferred as LocalizedStringKey
```
This is the most common silent SwiftUI localization bug. If you're building the string dynamically, build a `LocalizedStringResource` or use `String(localized:)` explicitly and pass the *result* — don't pass a plain interpolated `String` and hope.

Use `Text(verbatim:)` deliberately for content that must never be localized: preview/mock data, user-entered text, raw API values.

## `LocalizedStringResource` — passing a localizable string around

Use this when a type needs to *hold* a localizable string before it's rendered (e.g. a scene model property, per the Presentation-pattern rule that all localization happens in the view layer — a raw enum case is usually better there, but when the view layer itself needs to pass a resource between its own types, this is the type for it):
```swift
struct RetryPromptView: View {
    let title: LocalizedStringResource
    var body: some View { Text(title) }
}
RetryPromptView(title: LocalizedStringResource(
    "ui.errorView.retryButton", defaultValue: "Try again",
    bundle: .atURL(Bundle.module.bundleURL),
    comment: "…"
))
```

**Pitfall:** `LocalizedStringResource`'s string-literal initializer (`"some key"` used directly where a `LocalizedStringResource` is expected) defaults `bundle:` to `.main`. Inside a package or framework, that's the wrong bundle — the lookup fails at runtime and falls back to showing the raw key or default value with no compile-time warning. Always construct it explicitly with `bundle: .atURL(#bundle.bundleURL)` (or `.forClass(SomeTypeInThisModule.self)`) when you're inside anything other than the main app target.

## Which bundle

| Context | Use |
|---|---|
| App target | `#bundle` (resolves to `Bundle.main`) or omit `bundle:` entirely |
| Swift package target | `#bundle`, or `.module` if the toolchain predates `#bundle` |
| Framework target | `#bundle`, or `Bundle(for: SomeTypeInThisModule.self)` as a fallback |
| Passing a `LocalizedStringResource` from a package/framework | `.atURL(#bundle.bundleURL)` — never leave it at the `.main` default |

`#bundle` is a `@freestanding(expression)` macro from Foundation: "expands to a bundle instance that's most likely to contain resources for the calling code," and explicitly back-deploys to older OS versions than its own availability would suggest. Prefer it over manually picking `.module` vs. `Bundle(for:)` whenever the toolchain supports it — one spelling that's correct everywhere beats remembering which of three APIs applies to which target type.

## `table:` — splitting strings across catalogs

Omit it for the default `Localizable.xcstrings` table. Pass `table: "Discover"` (creating/using `Discover.xcstrings`) when a feature or framework's strings should live in their own catalog rather than the shared one — useful for large multi-package apps where one team owns one catalog. Generated symbols namespace under the table: `LocalizedStringResource.Discover.myKey` vs. `LocalizedStringResource.myKey` for the default table.

## Generated symbols (only when the project already uses them)

Detected in §0 of `SKILL.md` by call sites like `String(localized: .someKey)` or `LocalizedStringResource.someKey`. If the project uses this workflow, don't hand-write extraction calls — instead add the entry to the `.xcstrings` file directly (key, default value, comment) and call it as:
```swift
String(localized: .errorViewRetryButton)                     // Foundation, key "errorView.retryButton", default table
Text(.introductionTitle)                                       // SwiftUI, default table
.navigationSubtitle(.subtitle(friendsPosts: 42))                // generated function for a placeholder
String(localized: .Shared.errorViewRetryButton)                // key "errorView.retryButton" in Shared.xcstrings
LocalizedStringResource.Discover.myKey                          // non-default table, namespaced
```
Xcode generates the symbol from the catalog entry at build time; a placeholder in the value (`%lld`) generates a function parameter, not a property. The symbol is `.<Table>.<camelCasedKey>`: the default `Localizable` table adds no namespace, and dots in a key flatten into camelCase (`errorView.retryButton` → `errorViewRetryButton`) rather than becoming nested types. In a Swift package the generated symbol resolves `Bundle.module` itself, so pass no `bundle:`. This workflow trades "wording is a full app rebuild away" concerns for "the key now only exists in one file, must be added there first" — worth it in a large project with many contributors touching the same table, not worth introducing into a project that hasn't already adopted it.

## Relevant build settings

| Setting | What it does | When to check it |
|---|---|---|
| `SWIFT_EMIT_LOC_STRINGS` | Whether the Swift compiler extracts `String(localized:)`/`Text(...)` calls into catalogs at all | Must be `YES` on any target whose Swift code should be extracted; `NO` is correct for test targets |
| `LOCALIZATION_PREFERS_STRING_CATALOGS` | Makes `Export Localizations` / `xcodebuild -exportLocalizations` emit String-Catalog-flavored XLIFF instead of legacy `.stringsdict` XLIFF | Set `YES` once the project has moved off `.strings`/`.stringsdict` |
| `STRING_CATALOG_GENERATE_SYMBOLS` | Turns on the generated-symbols workflow project-wide | Only relevant if the project has opted into that workflow (§0 detection) |
| `LOCALIZED_STRING_MACRO_NAMES` | Registers a **custom C/Objective-C macro** (not Swift) as an extraction source, for a wrapper like `#define MyLocalizedString(key, comment) ...` | Only needed for Objective-C/C code with a custom wrapper macro — Swift string extraction doesn't use this setting |
