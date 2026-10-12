# Hazards and fixes

Every example here is a real failure mode from Apple's own localization guidance (WWDC21 "Streamline your localized strings", WWDC22 "Build global apps: Localization by example"). The common thread: English happens to be forgiving about things other languages aren't, so a pattern that "just works" in English source can be silently wrong the moment translation starts, and nothing catches it at compile time.

## Concatenation

**Bad:**
```swift
var subtitle: String {
    official ? "\(typeLabel) · Official" : typeLabel
}
```
This bakes in English word order and an English-specific separator. A translator can't reorder "Official" before the type word, or use a different separator, or restructure the sentence — they only ever see the fragment `"\(x) · Official"` with no way to know what `x` will be.

**Good:** give the translator the whole sentence, with the value interpolated as a parameter, so word order is theirs to choose:
```swift
private extension String {
    static func officialLabel(type: String) -> String {
        String(localized: "ui.video.officialLabel",
               defaultValue: "\(type) · Official",
               bundle: #bundle,
               comment: "Video subtitle marking an official upload. %@ is the video type (Trailer, Clip, etc).")
    }
}
```
This still interpolates — the difference is the *entire* sentence is one catalog entry a translator can rewrite freely, not two fragments glued together in Swift.

## Fake plurals

**Bad:**
```swift
detail: episodeCount.map { $0 == 1 ? "1 Episode" : "\($0) Episodes" }
```
Two problems: (1) it's invisible to the String Catalog — there's no `String(localized:)` call here at all, so it can never be translated, and (2) even translated literally, `count == 1` is the wrong test for languages with more than two plural forms. Russian needs four forms (one/few/many/other) selected by the *last digits* of the number, not by equality to 1; Arabic needs six. A binary `== 1` check can't express that, and hardcoding it in Swift means every language gets forced through English's two-form grammar.

**Good:** put the number *in* the localized string, and let the catalog vary the wording:
```swift
private extension String {
    static func episodeCount(_ count: Int) -> String {
        String(localized: "ui.tvShowDetail.episodeCount",
               defaultValue: "\(count) episodes",
               bundle: #bundle,
               comment: "Episode count for a season. %lld is the number of episodes.")
    }
}
```
Then, once this is extracted, mark the **source-locale** entry "Vary by Plural" (via `StringCatalogEdit` or the JSON shape — see `references/string-catalogs.md`) so English carries `one: "%lld episode"` / `other: "%lld episodes"`. Every other language gets its own plural cases when translated, matching its own grammar — Xcode already knows how many cases Russian or Arabic need; you never write that logic.

## Count-dependent phrasing with no number

Not every count-dependent string is a plural-rules problem. If the string doesn't display the number at all, plural variation has nothing to vary on — this is just two different strings, chosen by count in code:
```swift
// Correct: not a plural-rules case, because there is no %lld to vary
let message = selectedCount == 1
    ? String(localized: "ui.favourites.removeOne", defaultValue: "Remove this city",
              bundle: #bundle, comment: "Confirmation button when removing a single favourite city.")
    : String(localized: "ui.favourites.removeMany", defaultValue: "Remove these cities",
              bundle: #bundle, comment: "Confirmation button when removing multiple favourite cities.")
```
Don't force this into a stringsdict/plural-variation shape just because it's count-dependent — that machinery exists for strings with a number *in* them.

## `String(format:)` and hand-rolled formatting

**Bad:**
```swift
let label = String(format: "%d%% complete", progress)
let price = "$\(String(format: "%.2f", amount))"
```
`String(format:)` doesn't localize digit systems (Arabic-Indic vs. Western digits), doesn't isolate interpolated numbers for correct display in right-to-left text, and hardcoded symbols like `$` or `%` ignore locale conventions entirely.

**Good:** use `String(localized:)` for the sentence and Foundation's `.formatted()` for the value:
```swift
let label = String(localized: "ui.download.progress", defaultValue: "\(progress.formatted(.percent)) complete",
                    bundle: #bundle, comment: "Download progress label.")
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
String(localized: "Archive")   // used both as a sidebar folder name and as a menu action
```
"Archive" the noun (a folder) and "archive" the verb (an action) translate to different words in many languages. If both uses share one catalog entry, translating one meaning breaks the other.

**Good:** separate keys, even though the English text is identical:
```swift
String(localized: "ui.sidebar.archiveFolder", defaultValue: "Archive", bundle: #bundle,
       comment: "Name of the Archive folder shown in the sidebar (a place, not an action).")
String(localized: "ui.messageActions.archive", defaultValue: "Archive", bundle: #bundle,
       comment: "Menu item that moves the selected message into the Archive folder (an action).")
```
This is the main argument for semantic keys over source-text keys: with source-text keys, Xcode would merge these into one catalog row unless you also varied the table, which is more machinery for the same result.

## One template needing different grammar per call site

**Bad:**
```swift
String(localized: "Show weather in \(place)")
// called with place = "Berlin" and place = "My Location"
```
In several languages the preposition before a city name ("in Berlin") differs grammatically from the preposition before "my current location" — one template can't hold both correctly no matter how it's translated.

**Good:** two full strings, each with its own comment describing exactly what fills the slot:
```swift
String(localized: "ui.weather.titleForCity", defaultValue: "Show weather in \(cityName)",
       bundle: #bundle, comment: "Activity title for viewing weather at a specific named city.")
String(localized: "ui.weather.titleForCurrentLocation", defaultValue: "Show weather in My Location",
       bundle: #bundle, comment: "Activity title for viewing weather at the user's current location (fixed phrase, not a variable place name).")
```

## Remote content matched to the wrong locale

**Bad:** localizing the UI to the app's language but requesting server content using the raw device locale, or vice versa — the two can disagree (e.g. device set to a region the server doesn't support), producing a screen that's part one language, part another.

**Good:** resolve the best-matching server language explicitly and use the same source everywhere:
```swift
let serverLanguages = ["bg", "de", "en", "es", "kk", "uk"]
let bestMatch = Bundle.preferredLocalizations(from: serverLanguages).first
```
Never ship a partially localized screen — if the server can't support the user's language, decide deliberately (fall back to a supported one consistently) rather than letting different pieces of the screen independently guess.

## Inflection markdown (optional, narrow use)

For simple singular/plural agreement in the languages that support it, Foundation offers inline inflection markup as an alternative to a full plural-varied catalog entry:
```swift
String(localized: "\(amount) in last ^[\(hours) hour](inflect: true)")
```
This is a narrower tool than catalog plural variation — it only covers automatic grammatical agreement in supported languages, and translators see the `^[...]()` markup inline rather than clean prose. Default to catalog plural variation (§4/§5 of `SKILL.md`); reach for this only when a project has already adopted it or when the string is simple enough that the markup overhead is clearly worth it.
