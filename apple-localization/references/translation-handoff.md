# Handing off to Xcode's bundled translation skills

**Precondition:** everything in scope is already extracted into a String Catalog with real keys, `defaultValue`s, and comments (mode B or C from `SKILL.md` has already run on this scope). Apple's translation tooling can only translate what's already in a catalog — it does not read raw Swift source looking for hardcoded strings. If you're not sure the scope is ready, run an inventory (`SKILL.md` §6) before going further.

## Why this exists as a separate step

Xcode 27+ ships two skills purpose-built for translation, with 58 bundled locale style guides, that this skill deliberately does not duplicate:
- **`translation-coordinator`** — plans the work: adds the target language to the project, builds to trigger extraction, lists untranslated keys, splits them into batches, and delegates each batch to a sub-agent running the `translation` skill.
- **`translation`** — translates one batch: for each key, reads the source text, the developer's comment, similar strings elsewhere in the catalog (for terminology consistency), the code location the string is used at (to tell a button-verb from a noun), and the target locale's style guide, before writing a translation.

They are not registered as Claude Code skills — they ship inside Xcode's app bundle for Xcode's own AI features — but their instructions and the MCP tools they drive (`LocalizationPlanner`, `StringCatalogRead`, `StringCatalogContext`, `StringCatalogEdit`) are reachable from here. Read and follow their actual instructions rather than reimplementing translation logic from memory — they encode real per-locale grammar rules (verb-vs-noun for buttons, part-of-speech consistency across a toggle's two labels, quote/apostrophe conventions per language) that are easy to get subtly wrong from general knowledge.

## Locating the skills

```bash
xcode-select -p
# e.g. /Applications/Xcode-27.0.app/Contents/Developer
```
The skills live at:
```
"$(xcode-select -p)/../PlugIns/IDEXCStringsSupport.framework/Versions/A/Resources/Skills/translation-coordinator/SKILL.md.packaged"
"$(xcode-select -p)/../PlugIns/IDEXCStringsSupport.framework/Versions/A/Resources/Skills/translation/SKILL.md.packaged"
"$(xcode-select -p)/../PlugIns/IDEXCStringsSupport.framework/Versions/A/Resources/Skills/translation/references/styleguide_<locale>.md.packaged"
"$(xcode-select -p)/../PlugIns/IDEXCStringsSupport.framework/Versions/A/Resources/Skills/translation/references/plural-variations.md.packaged"
"$(xcode-select -p)/../PlugIns/IDEXCStringsSupport.framework/Versions/A/Resources/Skills/translation/references/device-variations.md.packaged"
```
If `xcode-select -p` points at a Xcode without this framework (pre-27), fall back to the newest `/Applications/Xcode*.app` that has it:
```bash
ls -d /Applications/Xcode*.app/Contents/PlugIns/IDEXCStringsSupport.framework 2>/dev/null
```
If none exists on the machine at all, skip straight to the manual XLIFF fallback at the bottom of this file.

## Running it from Claude Code

The packaged files are plain Markdown (despite the `.md.packaged` extension) and reference each other and their tools by the same names Claude Code exposes. Concretely:

1. **Read** `translation-coordinator/SKILL.md.packaged` in full and follow it as written.
2. It will call `mcp__xcode__LocalizationPlanner` to add the target language and create any missing catalogs, then `mcp__xcode__StringCatalogRead` to list untranslated keys.
3. It says to delegate batches of up to 15 keys to sub-agents "using the `xcode-integration:translation` Skill." Since that skill isn't registered here, **tell each sub-agent explicitly to read `<Skills>/translation/SKILL.md.packaged` (full absolute path) and follow it**, rather than naming a skill it can invoke by name. Give each sub-agent the same inputs the coordinator's own examples specify: target locale, String Catalog path, and the exact key list (verbatim, including any `\uXXXX` escaping — see the coordinator's own critical rules about not losing escaping in transit).
4. Wherever the `translation` skill's instructions reference `references/styleguide_{locale}.md`, `references/plural-variations.md`, or `references/device-variations.md`, resolve those against the **same `translation/references/` directory located in step 1** and append `.md.packaged` to the filename Claude expects — that's the only naming difference from what the packaged instructions literally say.
5. Keep the coordinator's stated limits: batches of ≤15 strings, at most 5 concurrent sub-agents at a time, waiting for one wave to finish before starting the next.
6. After all sub-agents finish, use `mcp__xcode__StringCatalogRead` yourself to verify every requested key actually moved out of the "new" state, exactly as the coordinator's own workflow specifies.

The Claude Code MCP tools take an optional `workspaceIdentifier` rather than the packaged instructions' `tabIdentifier`. When Xcode is running with its UI, pass the identifier `XcodeListWorkspaces` returns (e.g. `windowtab-…`) and check it maps to the intended checkout or worktree; an absolute path only works in headless mode. Omit it only when a call doesn't need to identify a workspace.

Gotchas:
- Call `LocalizationPlanner` once per target locale before translating. A locale listed in `knownRegions` isn't registered with the catalog tools until you do.
- `LocalizationPlanner` can create empty `InfoPlist.xcstrings` and `mul.lproj/LaunchScreen.xcstrings` files. Delete them if they're out of scope.
- `StringCatalogEdit` saves `machine_translated`, not `translated`. When verifying, check the `machine_translated` state.

## Before starting: two things to get from the user

The coordinator's own instructions say to forward "the user's request, terminology and style choices" to sub-agents, since sub-agents never see the original conversation. Get these explicitly rather than guessing:
- **A do-not-translate list** — brand names, product names, trademarked terms, anything that should appear identically in every language.
- **The app name's translation policy** — many apps keep their name untranslated everywhere; some don't. Whichever the user decides, it must be applied consistently across every string that mentions it, and the coordinator's instructions already say to translate any `CFBundleDisplayName`/`CFBundleName` keys *first* in the first batch so later batches can reuse that decision.

State plainly to each sub-agent that this guidance overrides the style guide, but that the style guide is still the baseline for anything this guidance doesn't cover — this mirrors exactly what the `translation` skill's own instructions say about priority order (explicit instructions > existing translations > style guide).

## Fallback: no Xcode 27+, or a human translator is preferred

```bash
xcodebuild -exportLocalizations -workspace App.xcworkspace -localizationPath ~/Desktop -exportLanguage de
# send ~/Desktop/de.xcloc to a human translator or translation vendor
xcodebuild -importLocalizations -workspace App.xcworkspace -localizationPath ~/Desktop/de.xcloc
```
This produces/consumes the same industry-standard XLIFF format the bundled skills work with internally — a vendor's own tooling or a human translator can open the `.xcloc` directly. Use this path when the user specifically wants a human in the loop, or when Xcode 27's bundled skills genuinely aren't available on the machine.
