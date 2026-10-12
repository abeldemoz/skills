# String Catalogs: setup, states, authoring, and variations

A `.xcstrings` file is the single source of truth for one table's strings across every language — keys, source text, comments, translations, and states, all in one JSON file that Xcode compiles down to `.strings`/`.stringsdict` at build time. This reference covers the mechanics; `SKILL.md` §4 covers when to do each step.

## One-time setup (mode C, or any package/target with none yet)

**Swift package:** add `defaultLocalization` — without it, Xcode has no development-region baseline for the package's catalogs, and `Product > Export/Import Localizations` won't offer the package's strings at all:
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
Catalogs go anywhere under the directory `resources: [.process(...)]` points at (e.g. `Sources/MobileUI/Resources/Localization/`).

**An empty catalog**, if the target has none yet — File > New > String Catalog in Xcode, or write the minimal valid file:
```json
{
  "sourceLanguage" : "en",
  "strings" : { },
  "version" : "1.0"
}
```
Xcode may bump `version` the first time it saves the file; that's expected.

**App target Info.plist** — list every locale the app ships so strings hosted in packages resolve at runtime:
```xml
<key>CFBundleDevelopmentRegion</key>
<string>en</string>
<key>CFBundleLocalizations</key>
<array>
    <string>en</string>
    <string>de</string>
    ...
</array>
```
Add new locales here (and to the project's known regions) whenever a language is added. Prove it works with a test that resolves one translated string from the package bundle (`SKILL.md` §5).

**Build settings** (check with `mcp__xcode__GetTargetBuildSettings` or by grepping the `.pbxproj`):
| Setting | Expected value |
|---|---|
| `STRING_CATALOG_GENERATE_SYMBOLS` | Generated-symbols style only: `YES`. Set it at project level; a package target has so far needed only `defaultLocalization` — confirm with the `SKILL.md` §5 test. Don't flip it in a project that hasn't chosen that style |
| `SWIFT_EMIT_LOC_STRINGS` | Extraction style only: `YES` on the app/framework/package targets whose Swift should be extracted (not test targets) |
| `LOCALIZATION_PREFERS_STRING_CATALOGS` | Either style: `YES` once the project is on catalogs, so exported XLIFF matches the catalog format |

## Translation states

Each key × locale pair carries a state, visible in Xcode's catalog editor and via `mcp__xcode__StringCatalogRead`'s `requestedState`:

| State | Meaning | What triggers it |
|---|---|---|
| **New** | No translation yet for this locale | A key was just added, or a language was just added to the project |
| **Needs Review** | Has a translation, but the source text changed since | Editing the English wording of an existing key — this is *why* semantic keys beat source-text keys: the key is untouched, so existing translations survive, just flagged for a human to revisit |
| **Translated** | Confirmed correct for this locale | Manual translation, "Mark as Reviewed", or writing `"state": "translated"` by hand |
| **Machine translated** (`machine_translated` in the JSON) | Written by Xcode's translation tools, not yet reviewed | `mcp__xcode__StringCatalogEdit` and the bundled `translation` skill |
| **Stale** | The key no longer appears anywhere in code | An extraction-style build ran and the extractor didn't find this key |
| *(manually managed)* (`"extractionState": "manual"` in the JSON) | Exempted from automatic add/update/remove | Every entry authored by hand for generated symbols |

**On orphans:** with manually managed entries Xcode never marks a key Stale, so removing the last call site leaves the entry behind. Delete the entry from the catalog too, or, if the string is expected to come back, leave it and say why in your summary.

## Authoring entries

**Whenever you hand-edit a catalog, tell the user to close it in Xcode first** — Xcode holds its own in-memory copy while a catalog is open in the editor and can overwrite a concurrent on-disk edit when it next saves.

**Match Xcode's formatting** so diffs stay minimal: two-space indent, a space on both sides of the colon (`"key" : value`), and keys sorted alphabetically (entries under `"strings"`, locales under `"localizations"`). Insert new entries as text in their sorted position rather than re-serializing the file with a JSON library, whose default output (`"key": value`) would rewrite every line. Afterwards validate with `python3 -m json.tool <file> >/dev/null`.

### Generated symbols — write the whole entry by hand

The entry must exist before the call compiles, so the source-language entry is hand-written JSON (shapes below), with `"extractionState" : "manual"` and `"state" : "translated"`. Build afterwards so Xcode regenerates the symbols.

### Extraction — the build creates the entry; you add variations

The call creates the key, `defaultValue:` and comment when the project builds. A string with a numeric placeholder then needs its **source-locale** entry varied by plural before there's anything for a translator to work from — translators vary their *own* language's cases off of what the source shows them; they don't invent the concept of "this needs plural forms".

**Preferred: Xcode MCP.** Use `mcp__xcode__StringCatalogEdit` with `targetLocaleIdentifier` set to the **source** locale (e.g. `"en"`), even though this feels like "translating into English" — you're writing the source-language plural cases:
```json
{
  "filePath": "MobileUI/Sources/MobileUI/Resources/Localizable.xcstrings",
  "stringKey": "tvShowDetail.episodeCount",
  "targetLocaleIdentifier": "en",
  "variationTranslation": {
    "topLevelVariation": {
      "plural.one": "%lld episode",
      "plural.other": "%lld episodes"
    }
  }
}
```
The tool saves `machine_translated`; the source language isn't machine translated, so set the source entry's state back to `translated` afterwards. **Fallback:** hand-edit the JSON using the shapes below, minus `extractionState` (Xcode manages that field for extracted entries; `manual` marks hand-authored ones).

English only needs `one`/`other`. Don't guess at other languages' plural case sets (Arabic needs six: zero/one/two/few/many/other) — that is the `translation` skill's job once the key reaches it (see `references/translation-handoff.md`), and it can ask the catalog what cases a target locale needs.

### JSON shapes

The examples carry `"extractionState" : "manual"` as a hand-authored entry for generated symbols would; omit that line when adding variations to an extracted entry.

**Simple string:**
```json
"errorView.retryButton" : {
  "comment" : "Button on the shared error view that retries the failed request.",
  "extractionState" : "manual",
  "localizations" : {
    "en" : {
      "stringUnit" : {
        "state" : "translated",
        "value" : "Try again"
      }
    }
  }
}
```

**Plural of the whole string** — the value contains a bare `%lld` (a generated symbol then takes the count: `.deleteItemCount(Int)`):
```json
"deleteItemCount" : {
  "comment" : "Item count in a delete confirmation. The number is supplied by the placeholder.",
  "extractionState" : "manual",
  "localizations" : {
    "en" : {
      "variations" : {
        "plural" : {
          "one" : { "stringUnit" : { "state" : "translated", "value" : "%lld item" } },
          "other" : { "stringUnit" : { "state" : "translated", "value" : "%lld items" } }
        }
      }
    }
  }
}
```
**Plural inside a larger string (substitution)** — the sentence has other placeholders, and only one part varies. The template references the substitution as `%#@name@`; `argNum` is the position of the numeric argument in the whole format string, and `%arg` inside a variant stands for that number. With generated symbols the substitution name becomes a labeled parameter (`.infoCardOverflowValue(first, moreCount: n)`):
```json
"infoCard.overflowValue" : {
  "comment" : "Info chip value when an item has several values. %1$@ is the first value; the rest is how many more there are.",
  "extractionState" : "manual",
  "localizations" : {
    "en" : {
      "stringUnit" : { "state" : "translated", "value" : "%1$@ %#@moreCount@" },
      "substitutions" : {
        "moreCount" : {
          "argNum" : 2,
          "formatSpecifier" : "lld",
          "variations" : {
            "plural" : {
              "one" : { "stringUnit" : { "state" : "translated", "value" : "+%arg more" } },
              "other" : { "stringUnit" : { "state" : "translated", "value" : "+%arg more" } }
            }
          }
        }
      }
    }
  }
}
```
Both variants may legitimately read the same in English; the variation still has to exist so translators can differ per plural category.

**Device variation** (e.g. "tap" on iPhone vs. "click" on Mac) uses the same shape with `"variations" : { "device" : { "iphone" : {...}, "mac" : {...}, "other" : {...} } }` in place of `"plural"`. A string needing both nests them: `"device" : { "iphone" : { "variations" : { "plural" : {...} } }, ... }`.

## `InfoPlist.xcstrings`

The app name (`CFBundleDisplayName`) and permission-usage strings (camera, location, etc.) localize through a dedicated catalog, not through code symbols — create `InfoPlist.xcstrings` at the app target level and add it to the target; Xcode auto-populates known Info.plist localization keys on the next build, and custom ones can be added like any other entry.

## Migrating existing `.strings`/`.stringsdict`

If a project (or an imported dependency) still has legacy files: right-click the `.strings`/`.stringsdict` in Xcode → "Migrate to String Catalog." This preserves every existing translation and plural rule — it's a lossless format conversion, not a retranslation — and can be done incrementally, one table/target at a time, while both formats coexist elsewhere in the project.
