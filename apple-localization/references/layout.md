# Layout, right-to-left, and Dynamic Type

A perfectly extracted, perfectly translated string still ships broken if the view around it assumes English-length text in a left-to-right layout. These are the layout-side bugs that surface only once real translations land — catch them while you're already touching the view, rather than in a bug report from a German or Arabic user later.

## Never fix width by hand; let text determine its own size

German and Finnish commonly run 30–40% longer than English for the same meaning; Japanese and Chinese often run shorter. A container sized for the English string will clip, truncate, or overlap in either direction.

- Don't set a fixed `widthAnchor`/`heightAnchor` constraint on a label, button, or anything else holding localizable text — size it from its content (intrinsic content size) and let siblings adapt.
- `label.numberOfLines = 0` wherever the design can tolerate wrapping — a single hardcoded line count is an English-length assumption in disguise.
- If a fixed height is unavoidable (e.g. a table row), verify the target languages' longest expected strings still fit, or make the row self-sizing (`UITableView.automaticDimension`).
- SwiftUI: prefer layouts that size to content by default (`HStack`/`VStack`/`Grid`) over explicit `.frame(width:)` on text.

## Leading/trailing, never left/right

`left`/`right` anchors, `textAlignment = .left`, and manually flipped image assets all assume left-to-right reading order. Use the direction-aware equivalents everywhere, and RTL locales (Arabic, Hebrew, etc.) mirror the whole layout automatically:

| Instead of | Use |
|---|---|
| `leftAnchor` / `rightAnchor` | `leadingAnchor` / `trailingAnchor` |
| `textAlignment = .left` | `textAlignment = .natural` (or `.leading` in SwiftUI's `multilineTextAlignment`) |
| `UIEdgeInsets(left:right:)` | `NSDirectionalEdgeInsets(leading:trailing:)` |
| Hardcoded `.left`/`.right` alignment in a `UIStackView`/`HStack` | Rely on the system's automatic mirroring — don't hardcode an alignment that assumes reading direction |

**What mirrors automatically** in RTL: layout direction, leading/trailing constraints, stack view ordering, navigation transitions, most SF Symbols that have an RTL variant. **What does not mirror automatically** and needs explicit handling: directional icons that encode meaning through shape (a "forward" arrow, a playback scrubber), which need `imageFlippedForRightToLeftLayoutDirection` or an explicit RTL-aware asset; and anything where you've set `semanticContentAttribute` to `.forceLeftToRight`/`.forceRightToLeft` for a reason unrelated to the user's language (e.g. displaying a left-to-right phone number or code snippet inside an RTL screen — that specific subview should force LTR, not the whole screen).

## SwiftUI-specific layout tools

- `.leading`/`.trailing` alignment (never `.left`/`.right`) on stacks and text.
- `ViewThatFits` for constrained spaces (e.g. Apple Watch, or a button row that might need to stack vertically once translated text is longer) — provide a horizontal layout as the primary option and a vertical fallback; SwiftUI picks whichever actually fits the translated content at runtime.
- `Grid` with `.leading` column alignment for tabular layouts where columns need to size to their longest translated cell rather than a hardcoded width.

## Dynamic Type

Localization and accessibility text sizing compound — a long German string at an accessibility text size is the worst case a layout will see. Anything sized in §"Never fix width by hand" above should also be checked at the largest supported Dynamic Type category, not just at the default size in English.

## Manual testing: pseudolanguages

Available on request when a human wants to eyeball layout without waiting for real translations. Not part of this skill's default verification (see `SKILL.md` §4) — a build and catalog check is the default; reach for these only when specifically asked to visually check layout robustness.

**Scheme-based, via Xcode:** Edit Scheme → Run → Options → App Language, set to "Double-Length Pseudolanguage" or "Right-to-Left Pseudolanguage."

**Launch-argument equivalents** (useful for a UI test or a scripted launch):
```
-NSDoubleLocalizedStrings YES        # doubles every string's apparent length, to surface truncation
-AppleTextDirection YES -NSForceRightToLeftWritingDirection YES   # forces RTL layout without needing an RTL translation
-NSShowNonLocalizedStrings YES       # uppercases any string that ISN'T going through localization — the fastest way to spot a missed String(localized:) call
```
`-NSShowNonLocalizedStrings YES` in particular is worth mentioning to the user after finishing a migration (mode B/C) as a fast independent check for anything the grep-based audit in `SKILL.md` §7 might have missed — a string that shows up unexpectedly uppercased on screen bypassed the catalog entirely.
