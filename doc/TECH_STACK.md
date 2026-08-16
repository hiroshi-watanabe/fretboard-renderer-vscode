# Tech Stack

This document describes the toolchain and libraries used to build, package, and (manually) release the **Fretboard Renderer VSCode extension**. This repo used to be a `vscode-extension/` subfolder inside the [fretboard-renderer](https://github.com/hiroshi-watanabe/fretboard-renderer) Obsidian-plugin monorepo; it was split into its own repo on 2026-08-16 to keep its source out of Obsidian's automated plugin-review scanning (see that repo's [doc/TECH_STACK.md](https://github.com/hiroshi-watanabe/fretboard-renderer/blob/main/doc/TECH_STACK.md#why-the-split-2026-08-16) for the full story) — this repo is otherwise unrelated to that scanning concern, since VSCode has no equivalent automated review.

## Requirements

| Tool | Version | How to get it | Notes |
| :--- | :--- | :--- | :--- |
| Node.js | 18 or later recommended | Installer for your OS: [nodejs.org](https://nodejs.org/) | Only needed for development (running `npm`, esbuild, `vsce`). Not a runtime requirement for the extension itself — the built `dist/extension.js` runs inside VSCode's own Node-based extension host. |
| npm | Bundled with Node.js | Included with the Node.js installer above | Used for dependency management and running scripts. |
| TypeScript | `^5.4.3` (devDependency, see `package.json`) | `npm install` pulls it in automatically | Compiled to plain JavaScript by esbuild; `tsc` itself is only used for type-checking (`tsc -noEmit`), not for emitting output. |
| VSCode | `engines.vscode: ^1.74.0` (see `package.json`) | App download: [code.visualstudio.com](https://code.visualstudio.com/) | Minimum VSCode version the extension declares support for. Needed to actually run/debug the extension (Extension Development Host), not to build it. |
| `@vscode/vsce` | `^2.24.0` (devDependency) | `npm install` pulls it in automatically | CLI that builds the `.vsix` package (`vsce package`); see "Packaging & release" below for why `vsce publish` itself isn't used here. |

## Language & module system

- **TypeScript**, `strict: true` (see `tsconfig.json`). Compiles against `target: ES2020`, `module: CommonJS`, `moduleResolution: Node`, `lib: ["ES2020"]` — no `"DOM"` needed; this extension never touches a DOM (it emits SVG as a plain string via `fretboard-renderer-core`'s `toSvgString()`, not a live DOM tree).
- No `"types"` override in `tsconfig.json` — TS's default auto-include of everything under `@types/*` picks up `@types/node` and `@types/vscode` (both devDependencies) on its own. (Until 2026-08-16 this needed an explicit `"types": ["obsidian", "node"]` — the old shared `render-fretboard.ts`, before the `fretboard-renderer-core` extraction, mixed in Obsidian-only rendering functions that this project's `tsc` still had to type-check transitively. That's no longer the case: `fretboard-renderer-core` has zero Obsidian dependency.)
- No UI framework — the extension has no webview/custom UI at all. Its only user-facing surfaces are VSCode's built-in Markdown preview (via `markdown.markdownItPlugins`) and its native Settings UI (via `contributes.configuration`).

## Build

- **[esbuild](https://esbuild.github.io/)** `^0.20.2` — bundles `src/extension.ts` (and, transitively, `fretboard-renderer-core` + its own dependencies, e.g. `yaml`) into a single `dist/extension.js`.
  - Config: `esbuild.config.mjs`. Output format `cjs`, `platform: "node"`, `target: "node16"`. `vscode` and Node builtins are marked `external` (provided by the extension host at runtime); everything else, including `fretboard-renderer-core`, is bundled in.
  - `npm run dev` — watch mode (inline sourcemaps, unminified).
  - `npm run build` — runs `tsc -noEmit -skipLibCheck` for type-checking first, then a minified production build (no sourcemap).

## Runtime dependencies

- **[fretboard-renderer-core](https://www.npmjs.com/package/fretboard-renderer-core)** `^0.1.0` — the platform-agnostic parser, diagram model resolver, SVG renderer, chord/scale-naming logic, autocomplete context resolver, and settings-parsing, shared with the Obsidian plugin. This extension calls its `toSvgString()` path (not `toDom()`, which needs a live `document` this Node-hosted extension doesn't have). Bundled directly into `dist/extension.js` by esbuild.
- `yaml` is a transitive dependency of `fretboard-renderer-core` (used for `fretboard-renderer.yaml` Global-config parsing); not declared directly here, still bundled automatically.

## Testing

There is **no test suite in this repo**. The parsing/rendering/music-theory logic this extension calls into is tested in [fretboard-renderer-core's own test suite](https://github.com/hiroshi-watanabe/fretboard-renderer-core) (Vitest, 289 tests as of `0.1.0`). This repo's own code (`extension.ts`'s markdown-it plugin wiring, `render-fence.ts`'s error-wrapping, `completion-provider.ts`'s VSCode `CompletionItemProvider` glue, `settings.ts`'s System/Global settings resolution) is verified by building, packaging a `.vsix`, and manually exercising it in a real VSCode window (Extension Development Host via F5, or installing the built `.vsix` directly).

## Source layout

```text
src/
  extension.ts              activate() — registers the markdown-it plugin (extendMarkdownIt)
                             overriding the ```fretboard fence renderer, and the
                             CompletionItemProvider for in-editor autocomplete
  render-fence.ts           Parses a fence's raw YAML, resolves it, and renders to an
                             HTML string via fretboard-renderer-core (parseFretboardBlock
                             → resolveFretboardModel → buildFretboardSvg → toSvgString).
                             Zero `vscode` import — unit-testable in isolation, though no
                             tests exist for it yet in this repo (see "Testing" above).
  completion-provider.ts    VSCode CompletionItemProvider wrapping
                             fretboard-renderer-core's getFretboardCompletions()
  settings.ts               System layer = VSCode's own Settings UI
                             (contributes.configuration, namespaced fretboardRenderer.*).
                             Global layer = a fretboard-renderer.yaml at the workspace
                             root, via fretboard-renderer-core's parseVaultConfig.
```

- **Styling**: `media/fretboard-vscode.css` mirrors the Obsidian plugin's `styles.css` class names, remapped to VSCode's webview theme variables (e.g. `--text-normal` → `--vscode-editor-foreground`). Loaded via `markdown.previewStyles` in `package.json`.
- **What's excluded from the package** (`.vscodeignore`): `src/**` (TS source), `.vscode/**`, `node_modules/**`, `tsconfig.json`, `esbuild.config.mjs`, `.gitignore`, `**/*.map`.

### What's inside the `.vsix`

A `.vsix` is a zip; `vsce package`'s own output for this project lists exactly what ships:

```
extension.vsixmanifest      Marketplace metadata (from package.json)
[Content_Types].xml
extension/
  CHANGELOG.md
  LICENSE.txt
  README.md                 ← the same README shown on the Marketplace listing page
  package.json
  dist/
    extension.js             The one esbuild bundle — this repo's own code plus
                              fretboard-renderer-core and its transitive deps, all in one file
  media/
    fretboard-vscode.css
    icon.png
```

No `node_modules/`, no TypeScript sources, no test files — `dist/extension.js` is fully self-contained (everything not marked `external` in `esbuild.config.mjs` is bundled in, including `fretboard-renderer-core`).

## Packaging & release

Releases are **manual VSIX upload**, not `vsce publish` from the command line — see the `technical-notes` vault (`memo/VscodeExtensionRelease.md`) for the full story of why: `vsce publish` needs an Azure DevOps Personal Access Token, which in turn needs an Azure DevOps *Organization* to exist first, and creating one led into an unrelated Azure Subscription signup flow that never completed (2026-08-13/14).

1. Bump `version` in `package.json` (Claude owns this step by prior agreement with the maintainer — there's no `vsce publish` step to tie a git tag to an automatic bump).
2. `npm run package` (= `npm run build` + `vsce package`) — produces `fretboard-renderer-vscode-<version>.vsix` locally.
3. Open the **Manage Publishers & Extensions** page (Microsoft account sign-in, no PAT needed): https://marketplace.visualstudio.com/manage/publishers/hiroshi-watanabe
4. For a first-ever publish: "New extension" → drag-and-drop the `.vsix`. For an update: use the existing extension row's `…` (kebab) menu → **Update** → upload the new `.vsix`. (Using "New extension" for an update fails — it conflicts on the existing name+publisher.)
5. The listing shows "Verifying" briefly, then flips to the new version number once Marketplace's own automated check passes.

There is no GitHub Actions workflow for this repo's release — unlike the Obsidian plugin side (tag-push-triggered), this side has no CI automation by design.

## Debugging

- Press **F5** in VSCode (with this repo open at its root, not a parent folder) to launch an Extension Development Host — a child VSCode window with this extension loaded. Console output appears in the *parent* window's Debug Console.

## Why these choices

- **esbuild over Rollup/webpack**: fast, minimal config, first-class TypeScript support, matches the tooling already used on the Obsidian-plugin side of this project.
- **Manual VSIX upload over `vsce publish`**: not a preference — the intended CLI path (Azure DevOps PAT) was blocked by an unrelated signup flow at the time; see above.
- **`fretboard-renderer-core` as a normal npm dependency, not a relative path into a sibling repo**: keeps this repo genuinely standalone (a fresh `git clone` + `npm install` is all that's needed to build), and means a `fretboard-renderer-core` bugfix/feature only needs a version bump here, not a monorepo/workspace setup spanning two repos.
