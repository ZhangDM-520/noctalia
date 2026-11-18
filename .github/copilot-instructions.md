# Noctalia — Copilot instructions

Native Wayland desktop shell (C++23, OpenGL ES, no Qt/GTK; Meson + Ninja). Self-contained UI library in `lib/` (`noctalia_ui_dep`, `lib/meson.build`), shell in `src/` (`noctalia_core_dep`, root `meson.build`), unit tests in `tests/`, vendored code in `third_party/`.

Canonical rules live in [CONTRIBUTING.md](../CONTRIBUTING.md) (style, naming, testing, commits, PR template) and [ARCHITECTURE.md](../ARCHITECTURE.md) (invariants reviewers enforce). Do not restate them here; read the matching section before editing.

## Commands

Build modes are `debug` (default), `release`, `asan`, in `build-<mode>` dirs.

```sh
just configure [release|asan [prefix]]     # meson setup --reconfigure + compile_commands.json
just build                                 # noctalia + noctalia-pam-helper; no test targets
just run                                   # ./build-<mode>/noctalia
just test [mode] [test_name]               # meson test -C build-<mode>
just lint | just fix | just format | just format-check
just role-alpha-check | just test-workflows
python3 tools/i18n-check.py [--all]
```

A targeted run beats the suite: `meson test -C build-debug <name> --print-errorlogs`. Out of tree, prefix `NOCTALIA_ASSETS_DIR=$PWD/assets` or pass the assets dir as the test argument, or the asset-lookup tests fail.

clang-tidy takes touched files, and the header filter must stay build-relative:

```sh
run-clang-tidy -quiet -p build-debug -header-filter='\.\./(src|lib|tests)/.*' '-warnings-as-errors=*' <files...>
```

CI (`.github/workflows/ci.yml`) runs `just format-code-check`, `just role-alpha-check`, `just format-docs-check`, `just build`, `just test debug --print-errorlogs`, `just lint`, and a Fedora 44 build against its older `sdbus-c++`.

## Architecture boundaries

- **`lib/` vs `src/`**: `lib/` (`core`, `i18n`, `render`, `ui`, `util`, `wayland`) builds without `src/` on its include path and never includes shell code or shell protocols. Headers sit beside sources; no `include/` tree.
- **Event loop**: `MainLoop` polls `PollSource` subclasses (`lib/core/poll_source.h`) returned from `Application::buildPollSources()`.
- **UI phases**: `PrepareFrame -> Update -> Layout -> Render`; debug builds abort on scene mutation during `Render` (`lib/core/ui_phase.cpp`). Setters call `requestUpdate()`/`requestLayout()`/`requestRedraw()`/`requestFrameTick()`; overrides implement `doLayout`/`doMeasure`/`doArrange`/`doUpdate` and chain to the parent hook.
- **Nodes and render**: `src/shell/` composes `ui::` controls and builders (`InputArea` is the only raw-node exception). Paint goes through `RenderContext` -> `RenderBackend` (`createDefaultRenderBackend()`) and `TextureManager`; the shell holds typed handles only (`TextureId`, `TextureHandle`, `RenderFramebuffer`).
- **Fractional scaling**: keep `set_buffer_scale(1)`, render at the fractional scale, set `wp_viewport.set_destination()`, and call `makeCurrent()` + `syncContentScale()` before configure-time layout, measurement, or texture work. Panels own `noctalia-panel` layer surfaces; bars are created before docks.
- **Compositors**: route behavior through `CompositorPlatform` (`compositors::detect()`, `isNiri()`, `isHyprland()`, `isSway()`, `isMango()`), never environment sniffing or transport-layer branches.
- **Config and state**: `ConfigService` owns `settings.toml` and hot reload; UI and internal state belongs to `StateStore` in `state.toml`. CLI commands register in `src/cli/schema_*.h` (guarded by `tests/cli_schema_test.cpp`); D-Bus services release their names on disable.
- **Plugins**: plugins are trusted Luau reading a static `plugin.toml` through `getConfig()`. API changes bump `kCurrentPluginApiVersion` (`src/scripting/plugin_api.h`), `docs/plugin-api.json`, `docs/user/plugins/development/plugin-api.mdx`, and `noctalia.d.luau` in the official-plugins repository.

## Conventions

- **Tests**: shell tests register in `_cpp_test_names` (root `meson.build`); `lib/` tests link `noctalia_ui_dep` plus `tests/test_identity.cpp`. Use `TEST_CHECK` (`tests/test_check.h`), never `assert`; compare floats with a tolerance. Add a permanent test only for a deterministic contract a plausible regression would break.
- **One canonical name**: no aliases, fallback chains, or migration shims; fail loudly on bad input (`sdbus-c++` async handlers take `std::optional<sdbus::Error>` by value with `// NOLINT(performance-unnecessary-value-param)`).
- **User docs**: update `docs/user/**/*.mdx` in the same commit; never edit docs-site copies (`tools/sync-docs.sh` owns them). New plugin API levels enter `docs/plugin-api.json` with `"noctaliaVersion": null`.
- **Translations**: English keys only in `assets/translations/en.json`; verify with `tools/i18n-check.py`.
- **Palette roles**: use a role instead of role-plus-hardcoded-alpha; `just role-alpha-check` is a ratchet, `just role-alpha-update` lowers the baseline.
- **Vendored code**: never patch `third_party/` or `lib/third_party/` for Noctalia behavior.
- **Formatting**: clang-format 23 (`PATH=/usr/lib/llvm23/bin:$PATH just format-code-check`) and unwrapped Markdown/MDX (`tools/remark/format.mjs`). Format only touched files; `just format` rewrites the tree.
- **Commits and PRs**: Conventional Commits (`type(scope): summary`). `.github/workflows/enforce-pr-template.yml` matches the exact headings (`## Summary`, `## Motivation`, `## Type of Change`, `## Testing`, `## Checklist`) and Checklist wording, and demotes a non-conforming PR to draft.

## Manual verification

Set `NOCTALIA_ASSETS_DIR` when running out-of-tree binaries. Debug a running shell with `noctalia msg <cmd>`, `noctalia config {validate,export full}`, `noctalia plugins lint <dir>`, the `dev.noctalia.Debug` D-Bus service, or `ASAN_OPTIONS=log_path=/tmp/noctalia-asan` after `just configure asan && just build asan`.
