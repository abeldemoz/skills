# Hazards and fixes

Every example here is a real failure mode from Apple's own localization guidance (WWDC21 "Streamline your localized strings", WWDC22 "Build global apps: Localization by example"). The common thread: English happens to be forgiving about things other languages aren't, so a pattern that "just works" in English source can be silently wrong the moment translation starts, and nothing catches it at compile time.

Each fix shows the catalog entry (key = English value, plus the comment a translator needs; JSON shapes are in `references/string-catalogs.md`), then the call in both styles — **symbols** (the entry is written first and Xcode generates the symbol) and **extraction** (the call carries the key, `defaultValue:` and comment, and the build creates the entry). Use the one the project uses (`SKILL.md` §0). The comment text is abbreviated to `…` in the extraction lines; it is the same comment as in the entry.

## Concatenation

**Bad:**
```swift
var subtitle: String {
    official ? "\(typeLabel) · Official" : typeLabel
}
```
This bakes in English word order and an English-specific separator. A translator can't reorder "Official" before the type word, or use a different separator, or restructure the sentence — they only ever see the fragment `"\(x) · Official"` with no way to know what `x` will be.

**Good:** give the translator the whole sentence, with the value as a placeholder, so word order is theirs to choose:
```
video.officialTypeLabel = "%@ · Official"
  comment: "Video subtitle marking an official upload. %@ is the video type, such as Trailer or Clip."
```
```swift
// symbols
String(localized: .videoOfficialTypeLabel(typeLabel))
// extraction
String(localized: "video.officialTypeLabel", defaultValue: "\(typeLabel) · Official", bundle: #bundle, comment: "…")
```
The *entire* sentence is one catalog entry a translator can rewrite freely, not two fragments glued together in Swift.

## Fake plurals

**Bad:**
```swift
detail: episodeCount.map { $0 == 1 ? "1 Episode" : "\($0) Episodes" }
```
Two problems: (1) it never reaches the String Catalog, so it can never be translated, and (2) even translated literally, `count == 1` is the wrong test for languages with more than two plural forms. Russian needs four forms (one/few/many/other) selected by the *last digits* of the number, not by equality to 1; Arabic needs six. A binary `== 1` check can't express that, and hardcoding it in Swift forces every language through English's two-form grammar.

**Good:** put the number *in* the localized string, and let the catalog vary the wording:
```
tvShowDetail.episodeCount  — varied by plural
  one:   "%lld episode"
  other: "%lld episodes"
  comment: "Episode count for a season. %lld is the number of episodes."
```
```swift
// symbols
detail: episodeCount.map { String(localized: .tvShowDetailEpisodeCount($0)) }
// extraction
detail: episodeCount.map {
    String(localized: "tvShowDetail.episodeCount", defaultValue: "\($0) episodes", bundle: #bundle, comment: "…")
}
```
Pass the real `Int`. The source-locale entry carries English's `one`/`other` (`SKILL.md` §4 covers how to add the variation in each style); every other language gets its own plural cases when translated — Xcode already knows how many cases Russian or Arabic need, you never write that logic. Do not pre-format the count upstream (in a presenter, say): once it is a `String`, no plural rule can be applied to it.

## Count-dependent phrasing with no number

Not every count-dependent string is a plural-rules problem. If the string doesn't display the number at all, plural variation has nothing to vary on — this is just two different strings, chosen by count in code:
```
favourites.removeOneButton  = "Remove this city"
  comment: "Confirmation button when removing a single favourite city."
favourites.removeManyButton = "Remove these cities"
  comment: "Confirmation button when removing multiple favourite cities."
```
```swift
// symbols — not a plural-rules case, because there is no %lld to vary
let title = selectedCount == 1
    ? String(localized: .favouritesRemoveOneButton)
    : String(localized: .favouritesRemoveManyButton)
// extraction
let title = selectedCount == 1
    ? String(localized: "favourites.removeOneButton", defaultValue: "Remove this city", bundle: #bundle, comment: "…")
    : String(localized: "favourites.removeManyButton", defaultValue: "Remove these cities", bundle: #bundle, comment: "…")
```
Don't force this into a plural-variation shape just because it's count-dependent — that machinery exists for strings with a number *in* them.

## `String(format:)` and hand-rolled formatting

**Bad:**
```swift
let label = String(format: "%d%% complete", progress)
let price = "$\(String(format: "%.2f", amount))"
```
`String(format:)` doesn't localize digit systems (Arabic-Indic vs. Western digits), doesn't isolate interpolated numbers for correct display in right-to-left text, and hardcoded symbols like `$` or `%` ignore locale conventions entirely.

**Good:** the catalog for the sentence, Foundation's `.formatted()` for the value:
```
download.progressLabel = "%@ complete"
  comment: "Download progress label. %@ is the completion percentage, such as 45%."
```
```swift
// symbols
let label = String(localized: .downloadProgressLabel(progress.formatted(.percent)))
// extraction
let label = String(localized: "download.progressLabel",
                   defaultValue: "\(progress.formatted(.percent)) complete", bundle: #bundle, comment: "…")

let price = amount.formatted(.currency(code: "USD"))
```

### Formatter reference

| Need | API |
|---|---|
| Percent | `value.formatted(.percent)` |
| Currency | `amount.formatted(.currency(code: "USD"))` |
| Number with fixed precision | `value.formatted(.number.precision(.fractionLength(2)))` |
| Date | `date.formatted(.dateTime.year().month())` |
| Relative date | `date.formatted(.relative(presentation: .numeric))` → "2 minutes ago" |
| Duration / time range | `(start..<end).formatted(.components(style: .wide))` |
| Measurement (respects unit preference) | `measurement.formatted(.measurement(width: .narrow, usage: .rainfall))` |
| Byte count | `bytes.formatted(.byteCount(style: .file))` |
| List of items | `["A", "B", "C"].formatted(.list(type: .and))` → "A, B, and C"; `.list(type: .or)` → "A, B, or C" |
| Person name | `PersonNameComponentsFormatter().string(from: components)` — adapts to cultural name order |

Every one of these already knows the target locale's rules; don't reimplement any of them with string interpolation.

## The same English word needing different translations

**Bad:** one key, two meanings:
```swift
Text("Archive")   // used both as a sidebar folder name and as a menu action
```
"Archive" the noun (a folder) and "archive" the verb (an action) translate to different words in many languages. If both uses share one catalog entry, translating one meaning breaks the other.

**Good:** separate keys, even though the English text is identical:
```
sidebar.archiveFolderLabel   = "Archive"
  comment: "Name of the Archive folder shown in the sidebar (a place, not an action)."
messageActions.archiveAction = "Archive"
  comment: "Menu item that moves the selected message into the Archive folder (an action)."
```
```swift
// symbols
Text(.sidebarArchiveFolderLabel)
Text(.messageActionsArchiveAction)
// extraction
Text("sidebar.archiveFolderLabel", bundle: #bundle, comment: "…")   // or String(localized:defaultValue:…)
Text("messageActions.archiveAction", bundle: #bundle, comment: "…")
```
This is the main argument for semantic keys over source-text keys: with source-text keys, Xcode would merge these into one catalog row.

## One template needing different grammar per call site

**Bad:**
```swift
// called with place = "Berlin" and place = "My Location"
String(localized: "Show weather in \(place)")
```
In several languages the preposition before a city name ("in Berlin") differs grammatically from the preposition before "my current location" — one template can't hold both correctly no matter how it's translated.

**Good:** two full strings, each with its own comment describing exactly what fills the slot:
```
weather.titleForCityLabel            = "Show weather in %@"
  comment: "Activity title for viewing weather at a specific named city. %@ is the city name."
weather.titleForCurrentLocationLabel = "Show weather in My Location"
  comment: "Activity title for viewing weather at the user's current location (fixed phrase, not a variable place name)."
```
```swift
// symbols
String(localized: .weatherTitleForCityLabel(cityName))
String(localized: .weatherTitleForCurrentLocationLabel)
// extraction
String(localized: "weather.titleForCityLabel", defaultValue: "Show weather in \(cityName)", bundle: #bundle, comment: "…")
String(localized: "weather.titleForCurrentLocationLabel", defaultValue: "Show weather in My Location", bundle: #bundle, comment: "…")
```

## Remote content in a different language from the UI

**Bad:** localizing the UI to the app's language but requesting server content with some other language source (the raw device locale, a hardcoded default, a separate setting nobody reconciled) — the two can disagree, producing a screen that's part one language, part another.

**Good:** decide deliberately which language remote content follows, and use that one source everywhere. A project may legitimately define a content-language setting that is independent of the UI language (so a user can read the UI in one language and metadata in another); then that is the documented rule and the code must use it consistently. If remote content should follow the UI, resolve the best match explicitly:
```swift
let serverLanguages = ["bg", "de", "en", "es", "kk", "uk"]
let bestMatch = Bundle.preferredLocalizations(from: serverLanguages).first
```
Never ship a partially localized screen by accident — if the server can't support the language the rule calls for, fall back to a supported one consistently rather than letting different pieces of the screen independently guess.

## Inflection markdown (optional, narrow use)

For simple singular/plural agreement in the languages that support it, Foundation offers inline inflection markup as an alternative to a full plural-varied catalog entry:
```swift
String(localized: "\(amount) in last ^[\(hours) hour](inflect: true)")
```
This is a narrower tool than catalog plural variation — it only covers automatic grammatical agreement in supported languages, and translators see the `^[...]()` markup inline rather than clean prose. It is also an extraction-style literal, so it doesn't fit a generated-symbols project. Default to catalog plural variation (`SKILL.md` §4); reach for this only when a project has already adopted it or when the string is simple enough that the markup overhead is clearly worth it.
