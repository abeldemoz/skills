# String Catalogs: setup, states, variations, and how to edit them

A `.xcstrings` file is the single source of truth for one table's strings across every language — keys, source text, comments, translations, and states, all in one JSON file that Xcode compiles down to `.strings`/`.stringsdict` at build time. This reference covers the mechanics; `SKILL.md` §4 covers when to do each step.

## One-time setup (mode C, or any package/target with none yet)

**Swift package:** add `defaultLocalization` — without it, Xcode has no development-region baseline to extract into, and `Product > Export/Import Localizations` won't offer the package's strings at all:
```swift
let package = Package(
    name: "MobileUI",
    defaultLocalization: "en",   // <-- add this
    platforms: [.iOS(.v18)],
    ...
    targets: [
        .target(name: "MobileUI", ..., resources: [.process("Resources")])
    ]
)
```

**An empty catalog**, if the target has no `Localizable.xcstrings` yet — Xcode's UI creates one via File > New > String Catalog, or write the minimal valid file directly:
```json
{
  "sourceLanguage": "en",
  "strings": {},
  "version": "1.0"
}
```
Place it under the target's resources directory (e.g. `Sources/MobileUI/Resources/Localizable.xcstrings`) so `resources: [.process("Resources")]` picks it up. Building the target afterward is what actually populates `"strings"` — Xcode's compiler extraction runs on every build and adds every new `String(localized:)`/`Text(...)` call it finds, so an empty catalog immediately after setup is expected and correct.

**App-target build settings** (check with `mcp__xcode__GetTargetBuildSettings` or by grepping the `.pbxproj`):
| Setting | Expected value |
|---|---|
| `SWIFT_EMIT_LOC_STRINGS` | `YES` on the app/framework/package targets whose Swift should be extracted (not test targets) |
| `LOCALIZATION_PREFERS_STRING_CATALOGS` | `YES` once the project is on catalogs, so exported XLIFF matches the catalog format |
| `STRING_CATALOG_GENERATE_SYMBOLS` | Matches whatever workflow §0 of `SKILL.md` detected — don't flip this without the project owner's intent |

## Translation states

Each key × locale pair carries one of these states, visible in Xcode's catalog editor and via `mcp__xcode__StringCatalogRead`'s `requestedState`:

| State | Meaning | What triggers it |
|---|---|---|
| **New** | No translation yet for this locale | A key was just added, or a language was just added to the project |
| **Needs Review** | Has a translation, but the source (`defaultValue`) changed since | Editing the English wording of an existing key — this is *why* semantic keys beat source-text keys: the key is untouched, so history and existing translations survive, just flagged for a human to revisit |
| **Translated** | Confirmed correct for this locale | Manual translation, or "Mark as Reviewed" |
| **Stale** | The key no longer appears anywhere in code | A build ran and the extractor didn't find this key this time |
| *(manually managed)* | Exempted from automatic add/update/remove | Set explicitly for keys built dynamically (e.g. a runtime-constructed key) that the compiler can't see statically |

**On stale keys:** if you remove a Swift call, don't walk away — the catalog now has an orphaned stale entry. If the key holds only the source-language value, delete it from the catalog too (it's dead weight and can confuse a future audit). If any other locale has a value for it, leave the key in place — deleting it would destroy someone else's translation — and name it in your summary so a human decides.

## Plural and device variation (source locale)

A key with a numeric placeholder needs its **source-locale** entry varied by plural before there's anything for a translator to work from — translators vary their *own* language's cases off of what the source shows them, they don't invent the concept of "this needs plural forms" themselves.

**Preferred: Xcode MCP.** Use `mcp__xcode__StringCatalogEdit` with `targetLocaleIdentifier` set to the **source** locale (e.g. `"en"`), even though this feels like "translating into English" — you're writing the source-language plural cases, which is exactly what this tool call does for any locale, source included:
```json
{
  "filePath": "MobileUI/Sources/MobileUI/Resources/Localizable.xcstrings",
  "stringKey": "ui.tvShowDetail.episodeCount",
  "targetLocaleIdentifier": "en",
  "variationTranslation": {
    "topLevelVariation": {
      "plural.one": "%lld episode",
      "plural.other": "%lld episodes"
    }
  }
}
```
English only needs `one`/`other`. Don't write plural cases for any other language (Arabic needs six: zero/one/two/few/many/other) — those are filled in when the key is translated, which is outside this skill. Leave other locales alone.

**Fallback: direct JSON edit**, only if the Xcode MCP tools aren't available. **Tell the user to close that catalog in Xcode first** — Xcode holds its own in-memory copy while a catalog is open in the editor and can overwrite a concurrent on-disk edit when it next saves. The shape inside `"strings"."<key>"."localizations"."<locale>"`:
```json
"ui.tvShowDetail.episodeCount": {
  "comment": "Episode count for a season. %lld is the number of episodes.",
  "localizations": {
    "en": {
      "variations": {
        "plural": {
          "one":   { "stringUnit": { "state": "translated", "value": "%lld episode" } },
          "other": { "stringUnit": { "state": "translated", "value": "%lld episodes" } }
        }
      }
    }
  }
}
```
For a device-specific variant (e.g. "tap" on iOS vs. "click" on macOS), the same shape uses `"variations": { "device": { "iphone": {...}, "mac": {...}, "other": {...} } }` instead of `"plural"`. A string needing both nests them: `"device": { "iphone": { "variations": { "plural": {...} } }, ... }`.

For a **non-varied** key, the JSON is simpler — a plain `stringUnit` instead of `variations`:
```json
"ui.errorView.retryButton": {
  "comment": "Button on the error view that retries the failed request.",
  "localizations": {
    "en": { "stringUnit": { "state": "translated", "value": "Try again" } }
  }
}
```

## `InfoPlist.xcstrings`

The app name (`CFBundleDisplayName`) and permission-usage strings (camera, location, etc.) localize through a dedicated catalog, not the code-extraction path — create `InfoPlist.xcstrings` at the app target level and add it to the target; Xcode auto-populates known Info.plist localization keys on the next build, and any custom ones can be added the same way as a regular catalog entry.

## Migrating existing `.strings`/`.stringsdict`

If a project (or an imported dependency) still has legacy files: right-click the `.strings`/`.stringsdict` in Xcode → "Migrate to String Catalog." This preserves every existing translation and plural rule — it's a lossless format conversion, not a retranslation — and can be done incrementally, one table/target at a time, while both formats coexist elsewhere in the project.
