# Localization foundation

Status: Revised discussion draft. No architecture or dependency is approved.

This proposal favors readable catalogs, a small API and existing Hew conventions.
It describes a future implementation, not the current Japanese prototype.
Repository reference: `b561df8f49758068058636a5b275008e256d5e9a`.

## 1. Start small

Agree on the foundation with maintainers, demonstrate one translated welcome
screen, and expand Japanese coverage after the foundation is accepted. Keep the
current prototype intact as reference material; do not submit all its changes as
one foundation contribution.

The first contribution needs only:

- English and demonstration Japanese catalogs with descriptive message keys.
- A pure lookup/formatting function and one subscribing React hook.
- Language selection using Hew's existing settings pattern.
- A small validator and focused tests for the demonstration surface.

Native menus and feature-by-feature extraction follow separately. Model formats,
geometry, measurement parsing, command IDs and shortcuts remain unchanged.

## 2. Catalogs that people can read

Use feature-level JSON files, initially `locales/en/welcome.json` and
`locales/ja/welcome.json`. Keep English as the complete source catalog.
Use shallow semantic keys, two-space indentation, UTF-8 and LF. A wording fix
should not rename a key. Different meanings receive different keys even when
English uses the same word, such as the verb Open and an open surface.

Illustrative English catalog:

```json
{
  "startModeling": "Start modeling",
  "openFile": "Open a file…",
  "openFailure": "Could not open {fileName}."
}
```

Illustrative Japanese demonstration:

```json
{
  "startModeling": "モデリングを始める",
  "openFile": "ファイルを開く…",
  "openFailure": "{fileName}を開けませんでした。"
}
```

Keep complete sentences together. Substitute named values as literal text;
never execute catalog contents or insert translation output as HTML.
English translator notes belong next to the feature in its developer document,
using a small key/context/placeholder table. No metadata service or fingerprint
manifest is needed initially. Source meaning changes require explicit translation
review in the same PR.

JSON is recommended because it is editable without TypeScript knowledge and can
later be read by the native shell. An `as const` TypeScript catalog is a valid
alternative if maintainers prioritize inline English comments and compiler
checks; it complicates direct Rust consumption and translation-tool interchange.
Choose one format before implementation, not two synchronized source catalogs.

## 3. A small application-facing API

Provide `useT(namespace)` for React and `translate(locale, namespace, key, values)`
for non-React callers. The hook owns its locale subscription; callers should not
need a separate subscription call just to make text update.

Illustrative API only; this is not compiled implementation:

```tsx
function StartButton({ onStart }: { onStart: () => void }) {
  const t = useT('welcome')

  // Translate the label without changing the document action.
  return <button onClick={onStart}>{t('startModeling')}</button>
}
```

A missing or empty translation falls back to the English message. Unknown
English keys fail validation. Derive key types from the English catalog; verify
placeholder names separately. Keep static metadata as keys and IDs, and resolve
messages at render time. All comments and developer guidance are English.

### Runtime choice

| Option | Strength | Maintenance cost |
| --- | --- | --- |
| Small internal adapter | No added runtime dependency; ordinary lookup, named interpolation and useSyncExternalStore | Hew owns the deliberately limited formatter and its tests |
| i18next + react-i18next behind the same API | Established plural handling and React integration | Added dependencies, configuration and message-syntax adaptation |

Prefer the small adapter for the welcome-screen demonstration. It supports only
plain text and named interpolation: no HTML, nesting, expressions or custom
message language. Keep plural-bearing messages outside that first slice; do not
pretend English never needs plural forms. Before migrating a count-bearing
surface, decide between an explicit plural entry shape selected with
Intl.PluralRules and the library option. Preserve whole-message English fallback
with English plural rules, rather than selecting an English form using Japanese
rules. Do not make every component implement its own plural selection.

Before choosing the production runtime, measure both options in an isolated
frontend entry under the pinned toolchain: minified and gzip bytes for the same
welcome example, plus initialization and fallback tests. No size has been measured
for this revision. A small hook adapter is the compatibility boundary; if the
library wins, it adapts catalog interpolation and owns subscriptions internally.
Do not expose both formatter syntaxes to application code.

## 4. Reuse the existing language-setting pattern

Use localStorage plus the existing Tauri `settings-changed` event, as theme and
units already do. Initial reads recover the persisted setting; an event is a
notification, not the only copy of the value. Confirm shared-storage behavior in
actual supported desktop webviews instead of assuming a missed event loses data.

Preference values are `en`, `ja`, and `system`. Start with English while the
translation is partial. Preserve explicit choices. Match system preferences in
order against supported locales, with English fallback. Update the mounted UI
and document language without reloading or resetting the model. Treat unavailable
storage as best-effort persistence; keep the current in-memory selection.

Initialize once per window, reconcile after attaching listeners and on focus,
and dispose listeners explicitly. Apply external changes without rebroadcasting.
Test late-opening windows, restart persistence and simultaneous changes. No
revision protocol or new Rust settings store belongs in the first contribution.
Do not add upstream migration code solely for unpublished prototype values.
When full Japanese coverage and desktop checks pass, separately propose switching
new users to the system default; never override an explicit English preference.

## 5. Native menus are a separate slice

First compare two approaches before implementing either:

| Approach | Benefit | Remaining question |
| --- | --- | --- |
| Embed the same static menu JSON in Rust with include_str! | One source dictionary; can translate during menu construction | Persisted language must be available before the first webview |
| Resolve labels in the frontend and apply them to existing handles | Uses the frontend formatter and current settings | Startup menu may remain English until the first update |

Prefer embedded JSON for static menu labels if first-paint localization is
required. Keep native labels plain text and parameter-free initially; do not
build a second interpolation/plural engine in Rust. Add a small persisted shell
preference only if startup language requires it, with a clear synchronization
contract and tests. The broader theme/units settings architecture stays outside
this slice. If frontend updates are sufficient, reuse existing focus ownership
and avoid a new snapshot/revision/rollback subsystem.

Preserve accelerators, checkmarks, enabled state and OS menu roles. Localize fixed
Recent-menu labels after rebuilds, but preserve filenames and user window titles.
Verify the pinned Tauri setters, including CheckMenuItem::set_text: the existing
source comment saying that setter is unavailable is not proof of the API limit.
Both approaches require a macOS spike; neither is verified by this document.

Undo/Redo text affects HTML and native menus. Current history entries contain
English labels. If stable operation kinds are unavailable at the UI boundary,
leave operation-specific text English temporarily. Adding kind to
history_entries_json changes the public WASM contract and needs explicit
maintainer discussion; do not parse English labels or sneak in an API change.

## 6. Migration boundaries

| Current coupling | Migration rule |
| --- | --- |
| ToolSpec.name in tools/toolRegistry.ts | Keep the existing tool ID; add a display key |
| palette/registry.ts labels/descriptions | Separate display/search text from action ID and icon lookup |
| health/healthCopy.ts MENU constants and MenuBar.tsx | Resolve display keys without changing behavior or weakening nativeMenuParity tests |
| panels/dockLogic.ts dockVerbsFor | Keep verb IDs and selection gates; localize display copy |
| settings/units.ts option labels and similar settings | Translate labels; retain stored values and unit parsing |
| App.tsx undoRedoMenuLabels | Handle English history labels as the explicit WASM boundary above |
| kernelErrors.ts | Translate the existing typed-error-code mapping; preserve diagnostic details |

When migrating palette search, score English and translated labels at the same
label tier; descriptions remain description matches, not label synonyms. Normalize
search inputs with NFKC and case folding consistently on query and candidate
copies, never on stored user content or IDs. NFKC handles width variants; it does
not equate hiragana and katakana. Kana aliases can be a later deliberate feature.
Preserve IME composition: Enter/Escape/arrows must not dispatch during composition,
including the WebKit final-confirmation behavior. Keep user-created labels verbatim.

## 7. Validation in proportion to scope

The first slice requires key/empty-value checks, matching named placeholders,
English fallback, key typing, preference synchronization, no model reload, and
component tests for visible and accessible translated text. Compare missing keys
only within the declared demonstration surface; incomplete unrelated namespaces
are not failures. If plural messages are introduced later, validate logical
messages and locale-specific plural categories rather than identical raw keys.

Use the project's existing tools where possible. Duplicate-key diagnostics may
use an existing parser; a TypeScript catalog could instead use compiler checks.
Do not introduce a new parser dependency without a concrete need and comparison.
Start with human source review for the small surface. Add scoped literal scanning
when migration grows; defer AST enforcement, source fingerprint manifests,
pseudolocalization, RTL infrastructure and a translation platform.

## 8. Delivery and review

1. Agree on the small API, catalog format and initial language behavior.
2. Build the foundation and one welcome-screen demonstration; compare runtime
   options in a bounded spike before choosing production dependencies.
3. Pass focused tests and required repository checks under the pinned toolchain.
4. Agree on the native-menu approach and verify it on macOS.
5. Migrate one feature at a time, then expand reviewed Japanese catalogs after
   foundation acceptance. Track visible gaps; do not call the demo a full release.

Keep this draft at its current path so existing review links remain valid. For
an accepted upstream plan, propose placing the detailed design in docs/dev/ and
adding a planned section to docs/agents/ROADMAP.md. Do not present an unapproved
feature as committed roadmap work.

The prior prototype's 174 passing focused tests do not verify this revised design.
Rust/WASM tooling and pinned Node checks remain implementation prerequisites.
This revision changes documentation only; no new benchmark or desktop verification
is claimed. Existing prototype files and audit records are preserved. No push,
commit, Issue or PR publication is authorized by this design revision.
