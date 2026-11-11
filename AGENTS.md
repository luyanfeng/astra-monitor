# AGENTS.md — Astra Monitor

GNOME Shell top-bar system-monitor extension (uuid `monitor@luyanfeng.github.io`), TypeScript compiled to GJS. GNOME Shell 45–51.

## Architecture

- Only two TS entry points are compiled (`tsconfig.json` → `files`): `extension.ts` (extension) and `prefs.ts` (preferences window). Everything in `src/` is reached via imports from these.
- Per-resource folders (`src/processor|gpu|memory|storage|network|sensors/`) each follow the same pattern: `*Monitor.ts` (data collection), `*Header.ts` (top-bar widget), `*Menu.ts`, plus `*Bars/*Graph` components. Prefs panels live in `src/prefs/`.
- `src/utils/utils.ts` is a singleton: `Utils.init(...)` is called in `extension.ts` `enable()` and wires settings, metadata, all monitors, and shared helpers (`CommandSubprocess`, `CancellableTaskManager`, ...). Access monitors/services through `Utils`, not by re-instantiating.
- `src/config.ts` wraps all GSettings access (schema `org.gnome.shell.extensions.astra-monitor`).
- Data sources are external commands (gtop, nethogs, amdgpu_top, nvidia-smi, intel_gpu_top, hwmon) spawned via `CommandSubprocess`; types for `gi://` imports come from the `@girs/*` packages plus `types/ambient.d.ts`.

## Commands

- Install deps: `npm install`. Note: `package-lock.json` is gitignored, so installs are not reproducible; `@girs` beta versions drift.
- Compile: `npx tsc` → outputs to `build/` (NOT `dist/`, despite README's `compile.sh` description; `dist/` is transient staging, deleted after pack/test). TypeScript is not a devDependency; the shell scripts require a global `tsc` on PATH, but `node_modules/.bin/tsc` exists, so prefer `npx tsc`.
- **Known tsc baseline:** `npx tsc` currently exits 2 with a large set of pre-existing type errors — `@girs/*` ambient-d.ts conflicts in `node_modules` plus GObject-class typing errors in `src/` (many in `*Graph.ts`). JS is still emitted to `build/`. Don't assume your change caused this; check whether your files appear in the error list and whether the error count grew. All shell scripts gate on tsc's exit code, so they abort while this baseline exists.
- Schemas: `bash ./schemas.sh` (`glib-compile-schemas`). Scripts auto-compile if `schemas/gschemas.compiled` is missing (gitignored).
- Translations: run `bash ./i18n.sh` after adding user-facing strings (xgettext over `build/**/*.js` → `po/monitor@luyanfeng.github.io.pot`, then msgmerge all `po/*.po`; needs `jq` and gettext).
- Pack: `bash ./pack.sh` — schemas → wipes `build/` → tsc → **`npm run format` (rewrites source files)** → i18n → stages to `dist/` → `gnome-extensions pack` → `monitor@luyanfeng.github.io.shell-extension.zip` at repo root.
- Live test: `bash ./test.sh` — compiles, packs, and **installs into your real GNOME session** (`gnome-extensions install --force`), then launches an isolated nested shell: GNOME ≥49 requires `mutter-devkit` (`gnome-shell --devkit`); older versions use Xephyr on `:2`. Sets `SHELL_DEBUG=all` for logs.
- **Updating the running extension (MANDATORY order, user rule):** when changing anything under `~/.local/share/gnome-shell/extensions/monitor@luyanfeng.github.io/`, you must (1) disable the extension, (2) **remove the old extension directory entirely** (move it aside as backup), (3) **move the newly built package in as a whole**, (4) enable the extension. NEVER replace individual files inside a live extension directory and NEVER toggle disable/enable as a hot-reload trick — this has crashed the GNOME Shell session (observed full session logout). Note: JS code changes still require a logout/login to take effect (GJS module cache survives disable/enable).
- EGO upload validation: `bash ./ego.sh` — build + pack + `shexli` (mirrors extensions.gnome.org checks; creates a python venv).
- Format: `npm run format` (prettier + eslint --fix). There is no separate lint script, no CI, and no unit-test suite (`benchmarks/` are manual perf scripts backing `COMPARISON.md`).

## Conventions that differ from defaults

- TS imports use `.js` extensions (module NodeNext): `import X from './src/foo.js'` inside `.ts` files.
- Formatting: prettier — 4-space indent, single quotes, printWidth 100, `arrowParens: avoid`.
- ESLint enforces GJS restrictions: use `console.log/warn/error` (no `log`/`logError` globals), `constructor()`/`super()` instead of `_init`, arrow functions instead of `Lang.bind`, template strings instead of `imports.format`, and `no-await-in-loop` is an error.
- Commits follow Conventional Commits (`fix:`, `chore:`).
- Releases: bump the numeric `version` (and human `version-name`) in `metadata.json`; build artifacts (`build/`, `dist/`, zips, `schemas/gschemas.compiled`, `po/*.mo`) are gitignored — never commit them.
