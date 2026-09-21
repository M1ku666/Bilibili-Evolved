# AGENTS.md

This file gives coding agents repository-specific guidance for working on Bilibili Evolved. Read `CONTRIBUTING.md` first, then use this file as the practical checklist for implementation and verification.

## Project Structure

- `src/` contains the userscript core, built-in components, shared runtime APIs, settings UI, and other code shipped with the main script.
- `registry/lib/components/` contains installable components. New components should be placed under the appropriate category directory and use `index.ts` as the webpack entry.
- `registry/lib/plugins/` contains plugins. Use a plugin when the feature only makes sense as an extension of another component.
- Third-party components that are meant to remain outside the main repository should be added through `registry/lib/docs/third-party.ts` instead of mixing external component registration with in-repo source changes.
- `dist/` and `registry/dist/` are build outputs. Development branches should not keep generated dist files; release output branches such as `preview`, `master`, and `master-cdn` get them from CI builds.
- Feature documentation outputs under `doc/features/` are generated from metadata. Update them only when the task is explicitly about generated docs or release preparation.

## Development Setup

- The package manager is `pnpm`.
- Install root dependencies with `pnpm install`.
- Install registry dependencies from `registry/` when building registry features.
- Use the VS Code tasks from `.vscode/tasks.json` as the source of truth for local task commands when in doubt.

Common commands:

```sh
pnpm run type
pnpm run lint-check
pnpm tsx dev-tools/dev-server/index.ts
pnpm tsx dev-tools/dev-server/command.ts sessions
pnpm tsx dev-tools/dev-server/command.ts shutdown
```

The dev server starts and watches the development core build, serves the local userscript, and provides registry feature URLs for browser testing. Registry feature URLs are virtual build-on-request outputs served from memory. WebSocket is the command and event control plane; use `dev-tools/dev-server/command.ts` for single-feature build, watch, stop, start-debug, stop-debug, scaffold, and graceful server shutdown commands. See `dev-tools/dev-server/README.md` for protocol details and `CONTRIBUTING.md` for Tampermonkey setup details.

## Rebuild After Every Change

**Always compile the feature you changed before reporting the work as done.** A registry component/plugin does not take effect in the browser until it is rebuilt and reloaded.

Workflow:

1. Identify the changed feature ids from the edited paths:
   - `registry/lib/components/<category>/<id>/...` → feature id `<category>/<id>` (kind `component`)
   - `registry/lib/plugins/<category>/<id>/...` → feature id `<category>/<id>` (kind `plugin`)
   - A single edit session may touch several features; build **each** of them.
2. Build every changed feature with the registry webpack config. This works without a dev server and writes the artifact to `registry/dist/components/<category>/<id>.js` (or `registry/dist/plugins/...`):

   ```sh
   pnpm webpack --config ./registry/webpack/components.ts
   pnpm webpack --config ./registry/webpack/plugins.ts
   ```

   `./registry/webpack/components.ts` loads *all* components and then shows an interactive `Select build target` autocomplete prompt, so it cannot take the target as a CLI argument (`builders.component()` in `registry/webpack/build.ts` only understands `{ buildAll }`, which is not wired to argv). Pick the target by piping its feature id into stdin and reading the tail of the output for the result:

   ```powershell
   "video/barline`n" | pnpm webpack --config ./registry/webpack/components.ts
   "video/music-replace`n" | pnpm webpack --config ./registry/webpack/components.ts
   ```

   Success ends with `webpack 5.97.1 compiled successfully` and exits `0`; the log line `Build target · ./registry/lib/components/video/barline/index.ts` confirms which feature was built. Any `ERROR in ...` line is a real build failure — fix it before finishing.
3. The output is cached (`cache: false` is set in the registry config, but the build still compares against the on-disk artifact), so delete the stale `registry/dist/components/<category>/<id>.js` first if you need to be certain the artifact is regenerated.
4. Report the exact build command(s) and the feature ids you compiled in your final answer, so the user knows what to reload.

Notes:

- There is no need to start the dev server (`pnpm tsx dev-tools/dev-server/index.ts`) for a one-off compile, and no need to shut anything down afterwards. The dev server's `build`/`watch` commands (`pnpm tsx dev-tools/dev-server/command.ts build component <id>`) are an alternative that first requires a running dev server — `sessions` fails with `无法连接 DevServer: ws://localhost:23333` when it is down. Prefer the standalone webpack command above when nothing else needs the dev server.
- Declarative edits that do not change shipped code (e.g. `index.md` prose, `AGENTS.md`) do not need a rebuild, but say so explicitly instead of skipping silently.
- Do not run `pnpm run build-core` / `pnpm run build-features` for this; those are CI/release validations.
- Validate types and lint before or alongside the build: `pnpm exec eslint "registry/lib/components/video/<id>/**/*.{ts,vue}"` and `pnpm exec tsc --noEmit --skipLibCheck` (filter the output for the component path — the full-repo `pnpm run type` reports pre-existing errors in unrelated registry features when the checkout is incomplete).

## Component And Plugin Guidelines

- Define components with `defineComponentMetadata`.
- Follow existing plugin definitions for code under `registry/lib/plugins/`.
- New components should include `author` metadata.
- Component `name` values should be specific and camelCase. Display names and option names should clearly describe the feature instead of using generic labels.
- If a component has an `index.md`, do not duplicate the same description in metadata unless the existing pattern for that component requires it.
- Use `urlInclude` and `urlExclude` for page matching instead of manually repeating page checks in `entry`.
- Keep `entry` focused on the work that must happen when the component starts.
- Do not rely on the return value of `entry` for cleanup. Put teardown logic in `unload`; use `reload` together with `unload` when a component must support disable and re-enable behavior.
- Use existing helpers such as `styledComponentEntry`, `toggleStyle`, shared core APIs, component APIs, and plugin APIs before adding new infrastructure.
- Put fixed component styles in SCSS files instead of constructing style text in business logic. Use `instantStyles`, `styledComponentEntry`, or `toggleStyle` so styles are only applied when intended.
- When writing SCSS, check shared Sass files before adding local helpers. For example, `ui/_common.scss` is available through `@import "common"` and provides common mixins such as fullscreen and centering helpers.
- For options, follow existing `defineOptionsMetadata` and `options` patterns so defaults, labels, and validators stay close to metadata.
- Model exclusive choices as one option, usually with an enum or dropdown, instead of multiple mutually exclusive boolean options.
- If settings need to control CSS, prefer toggling a class on `html` or `body` and writing SCSS against that class.

## Project-Specific Components

Two registry components in this repository carry a large amount of local behavior. Read this section before changing them.

### `registry/lib/components/video/barline` — 音乐小节线

- Files: `index.ts` (entry, options, control-bar buttons, metronome wiring), `config.ts` (data model + persistence + timeline math), `metronome.ts` (Web Audio metronome), `ui.ts` (progress-bar marker overlay), `swipe.ts` (touch swipe to jump bars), `video-fetch.ts` (Bilibili API metadata), `BarLineEditor.vue` (per-video config table), `BarLineManager.vue` (all-configs manager), `BarLineWidget.vue` (feature-panel button), `styles.scss`, `swipe.scss`.
- Feature-panel widget exposes two buttons: 「设置音乐小节线」opens `BarLineEditor.vue`, 「管理小节线」opens `BarLineManager.vue` (search / sort / multi-select delete / import-export).
- Data model: `BarConfig { segments[], metronomeMuted, metronomeVolume, createdAt?, openedAt?, videoTitle?, videoCover?, bvid?, aid? }`; `TimeSignatureSegment { bpm, numerator, denominator, firstBeatTime?, startBar? }`. Beat length is `(60 / bpm) * (4 / denominator)` — the denominator really participates in time math.
- Config storage key is the video's `cid || aid` (not BV). Video title/cover/bv are fetched lazily via `video-fetch.ts` and cached back into the config.
- Options: `configs` (hidden JSON string), `hideBarNavButtons`, `loopDisableCountIn`, `countInMaxVolume`, `promptMetronomeVolume`.

### `registry/lib/components/video/music-replace` — 本地音频替换

- Files: `index.ts` (entry, sync, dialog openers), `config.ts` (data model + GM persistence + IndexedDB blobs/handles), `player.ts` (`SyncedAudio` rAF sync loop), `MusicReplaceSettings.vue` (per-video settings), `MusicFilesManager.vue` (all-audio manager), `MusicReplaceWidget.vue` (feature-panel button), `styles.scss`.
- Feature-panel widget exposes 「本地音频替换」and 「管理音频替换」.
- Two storage layers: config JSON in settings (`musicReplaceConfigs`), and audio `Blob`s / `FileSystemHandle`s in IndexedDB (`barline-music-replace`, stores `audio` + `handles`). Audio playback reads the IndexedDB blob; the handle store is only a secondary backup and is **not** serializable, so it is excluded from import/export.
- Import/export packages config + audio into a zip (via `fflate`), with audio entries named `audio/<key>.<ext>` derived from `fileName`.
- `pickAudioFile()` must never throw: `showOpenFilePicker` is skipped inside cross-origin iframes and wrapped in try/catch with an `<input type="file">` fallback, because Chromium can raise `SecurityError` / `NotAllowedError` / `TypeError` there.

### Shared persistence rules (both components)

- Configs live in a settings string option (`barline.options.configs`, `musicReplace.options.musicReplaceConfigs`), so they are **shared across tabs through GM storage**, one tab per JS context.
- Registry components run in the page context, so GM APIs are reached through `unsafeWindow.bilibiliEvolved.monkeyApis.GM_getValue` / `GM_setValue` — the bare `GM_getValue` global is not defined there.
- Reads must prefer the latest GM value, and writes must **not** go through the local settings proxy. Assigning `settingsRef.options.configs` triggers the framework's whole-`components` write-back and clobbers other tabs' changes. Instead use the local `writeConfigsToGM(json)` helper: read the latest GM `components`, replace only this component's field, then `GM_setValue`. Keep it in sync when adding new persistence paths.
- Manager panels refresh on `visibilitychange` (tab becomes visible) so cross-tab edits show up without a manual refresh button.

### Keeping this section up to date

- This section is the source of truth for how `barline` and `music-replace` are structured. Whenever a change affects the described facts — files added/removed/renamed, data model or storage keys, options, feature-panel buttons, persistence rules, or the import/export format — **ask the user for confirmation first, then update the matching bullets here** in the same piece of work.
- Do not silently rewrite or restructure existing bullets; propose the edit and wait for the user to confirm before applying it.
- Small internal refactors that leave the documented behavior, files, storage, and options unchanged do not require an update.

## Code Style

- Follow the repository ESLint and Prettier configuration.
- Use named exports except in files covered by the existing ESLint overrides, such as Vue single-file components and build-related files.
- Keep control-flow bodies braced.
- Preserve the existing TypeScript, Vue 2, and SCSS conventions around the code being changed.
- Avoid unnecessary defensive branches, empty error handling, duplicated state, and one-off abstractions.
- Prefer existing Bilibili API wrappers, request helpers, settings helpers, observer utilities, style utilities, and UI components over reimplementing equivalent behavior.
- Prefer stable Bilibili APIs and existing wrappers over reading page-internal globals or framework internals when the data is available from an API.
- Scope DOM selectors to the nearest stable parent class or page region. Avoid broad selectors that can match unrelated Bilibili UI.
- Keep names and types aligned with behavior. If a function starts returning a broader shape, update the type and name instead of overloading a misleading contract.
- Be careful with data units and API failure states. Do not cache a failed request as if it were a successful value, and keep byte, bit, count, id, and URL semantics explicit.

## Generated Files

- Do not manually edit generated docs or dist outputs as part of ordinary feature or bug-fix work.
- If generated files must be updated, use the project’s existing build or docs generation workflow and include only the intended generated changes.
- Do not commit unrelated cache, profile, local debug, or package-manager output files.

## Validation

Choose validation based on the risk and scope of the change.

- Run `pnpm run type` for TypeScript type checking.
- Run `pnpm run lint-check` for lint validation.
- For core userscript changes, use the dev server's core watcher, which automatically compiles the development build.
- For registry feature changes, run `pnpm tsx dev-tools/dev-server/command.ts build <component|plugin> <id>` while the dev server is running.
- Run `pnpm exec tsc -p dev-tools/dev-server/tsconfig.json` for dev server TypeScript changes.
- For browser-facing behavior, verify the changed feature in a real browser with the local userscript installed.
- For API-shape or Bilibili-rollout-dependent changes, record what page or account state was actually self-tested.

The pull request CI workflow runs the following production builds. Do not repeat them as routine local validation; run them locally only when explicitly requested, reproducing a CI failure, changing shared build infrastructure, or preparing a release.

1. `pnpm run type`
2. `pnpm run lint-check`
3. `pnpm run build-core`
4. install dependencies in `registry/`
5. `pnpm run build-features`

When a change affects page behavior, also check the browser console for new runtime errors and test the relevant Bilibili page type, such as video, bangumi, live, feeds, space, settings panel, or the affected component area.

## Branches And Commits

- Use `preview-features` as the base for new features.
- Use `preview-fixes` as the base for bug fixes.
- Commit messages only need to clearly describe the change; the repository does not require a strict conventional commit format.
- Do not create release tags, push release branches, or perform release steps unless the task is explicitly about releasing.
