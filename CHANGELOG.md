# Changelog

## 0.5.0

Bumped `fretboard-renderer-core` to 0.4.0: adds note shapes `diamond`/`octagon`/`doublecircle` (◎)/`x`, note `opacity`, `paths[].arrow`/`curve` (arrowheads, smooth curves through 2+ points), `stringNotes[].side` (leading/trailing), a System `size` multiplier, and a Local `omittedStringBehavior` override. New Settings UI entries: `stringNoteDefaultShape`, `size`; `defaultShape`'s enum now includes the new shapes. Note: `fretboard-renderer-core` 0.4.0 also adds chord-progression sheets (`diagrams.progression`) — that feature isn't wired up for VSCode yet and shows a "not supported here yet" message if used.

## 0.4.0

Bumped `fretboard-renderer-core` to 0.3.0: adds Degree Name root notation (`rootNotation`/`key`, Roman-numeral titles relative to a key), polygon/fill/color/rounded-corner support for `boxes`, `style`/`color` on `paths` (dashed, barre-weight `thick`), per-string `stringNotes` annotations outside the grid, and fixes `(omit1)`/`(omit3)`/`(omit5)` typography (no longer mistakenly rendered as raised/superscript).

## 0.3.1

Bumped `fretboard-renderer-core` to 0.2.0: fixes chord-tension naming (e.g. a slash chord's bass no longer double-counts as an upper-structure tension; `m7`/`maj7` tension folding now requires the full chain below the target degree, matching the dominant-7th branch).

## 0.2.0

Added a VSCode Settings UI (`contributes.configuration`, namespaced `fretboardRenderer.*`) for the System settings layer — every field from the Obsidian plugin's settings tab is now configurable from VSCode's own Settings, no need to hand-edit `settings.json`. Workspace-wide `fretboard-renderer.yaml` still overrides it, same as before.

## 0.1.0 — MVP

Initial MVP: renders ```fretboard code blocks in VSCode's built-in Markdown preview, reusing the same parser/model/rendering core as the [Obsidian plugin](https://github.com/hiroshi-watanabe/fretboard-renderer). See the local `fretboard-renderer.yaml` (workspace root) for global config; no settings UI yet.
