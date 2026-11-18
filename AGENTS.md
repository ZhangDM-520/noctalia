# AGENTS.md

This is the agent-facing README for this checkout: what Noctalia is, the mental model you need before editing it, how to work here, the rules that are not negotiable, and a lean index of the pitfalls that cost time. `CONTRIBUTING.md`, `ARCHITECTURE.md`, and `SCOPE.md` are authoritative and are not restated here.

## What this repo is

Noctalia is a native Wayland desktop shell: C++23, OpenGL ES, no Qt and no GTK, built with Meson and Ninja. The shell layer around a compositor is `src/` (`noctalia_core_dep`); the self-contained UI library it builds on is `lib/` (`noctalia_ui_dep`); unit tests are in `tests/`; vendored code is in `third_party/`. End-user documentation is `docs/user/**/*.mdx`, generated into the docs site by `tools/sync-docs.sh`.

## Mental model

- **Two layers, one direction.** `lib/` (`core`, `i18n`, `render`, `ui`, `util`, `wayland`) builds without `src/` on its include path and never includes shell code or shell protocols. Headers sit beside their sources; there is no `include/` tree.
- **The shell wraps the compositor.** Behavior that differs per compositor goes through `CompositorPlatform` (`compositors::detect()`, `isNiri()`, `isHyprland()`, `isSway()`, `isMango()`). Never branch on environment variables, and never add a compositor branch in the transport layer.
- **Everything is polled.** `MainLoop` polls `PollSource` subclasses that `Application::buildPollSources()` registers. A new service implements `PollSource` and returns it from there.
- **Frames run in phases.** Every surface runs `PrepareFrame -> Update -> Layout -> Render`. Debug builds abort on scene mutation during `Render`. Setters schedule work with `requestUpdate()`, `requestLayout()`, `requestRedraw()`, or `requestFrameTick()`; overrides implement `doLayout`/`doMeasure`/`doArrange`/`doUpdate` and chain to the parent hook, not the public wrapper.
- **Controls own raw nodes, rendering owns GL.** `src/shell/` composes `ui::` controls and builders; `InputArea` is the only raw-node exception. Paint goes through `RenderContext` -> `RenderBackend` (`createDefaultRenderBackend()`) and `TextureManager`, and the shell holds typed handles only (`TextureId`, `TextureHandle`, `RenderFramebuffer`).
- **Surfaces.** Panels own `noctalia-panel` layer surfaces and draw their own background, shadow, blur, and reveal; "attached" is a placement mode, not a host surface. Bars are created before docks, and dock surfaces are recreated after bars on hot reload.
- **Two config stores.** `ConfigService` owns `settings.toml` and hot reload; UI and internal state belongs to `StateStore` in `state.toml`. No ad hoc state files, and no internal state hidden under config sections.
- **Plugins are trusted Luau.** A plugin is a directory with one static `plugin.toml`; code reads it through `getConfig()`. Plugin-facing API changes bump `kCurrentPluginApiVersion` (`src/scripting/plugin_api.h`) and update `docs/plugin-api.json`, the plugin API page, and `noctalia.d.luau` in the official-plugins repository.

## How to work here

1. Read the matching page in `ARCHITECTURE.md` before changing an invariant-bearing area, and check the pitfalls index below before touching tooling or CI.
2. One change per branch and per pull request. `main` tracks upstream; work on a branch, and use a separate worktree for concurrent work.
3. Verify with the narrowest check first: `bash -n` for shell, `python3 tools/i18n-check.py` for strings, then `just build` and `just test [mode] [name]`. Escalate to `just lint` and the full suite only when the targeted run does not settle it.
4. Judge by artifacts and measured output, never by exit codes.
5. Falsify every fix: remove the guard, watch the named test fail, restore it. A check that cannot fail its caller is not a gate.
6. Review the final diff against the standards and the spec, and resolve or justify every finding in writing.
7. Leave changes dirty and hand disposition to the owner. Do not commit, push, or open a PR without explicit approval.

## Durable rules

- **One canonical name.** No aliases, no "try X then Y", no migration shims, no silent default chains. Fail loudly on bad input.
- **Never patch vendored code.** `third_party/` and `lib/third_party/` are replaced wholesale; put compatibility code in Noctalia-owned sources.
- **Palette roles, no invented alpha.** Fills and borders use a role; alpha is for real transparency only. `just role-alpha-check` ratchets the count.
- **Tests earn their place.** Permanent tests cover a deterministic contract that a plausible regression would break and that the running shell would not catch as cheaply. Use `TEST_CHECK` (`tests/test_check.h`), never `assert`.
- **Docs travel with the change.** User-facing behavior updates `docs/user/**/*.mdx` in the same commit; new strings go only into `assets/translations/en.json`.
- **Formatting is pinned.** CI enforces clang-format 23 and unwrapped Markdown/MDX. Format only the files you touched.
- **Commits and PRs follow the template.** Conventional Commits (`type(scope): summary`); the PR template's headings and Checklist wording are matched exactly by a bot that demotes a non-conforming PR to draft.

## Pitfalls index

Mechanisms live in the maintainer's private warehouse; the handle after the arrow names the entry.

- **clang tooling is pinned to 23.** A newer local binary rewrites untouched files, so formatting the tree before a focused commit creates unrelated drift → `2026-10-27:toolchain:CI-PINS-AND-GCC17-LINK`.
- **clang-tidy's header filter must be build-relative.** `compile_commands.json` stores paths relative to the build directory, so an absolute anchor silently drops every header diagnostic → same entry.
- **Out-of-tree test runs need the assets directory.** `NOCTALIA_ASSETS_DIR` (or the test argument) is otherwise missing and the asset-lookup tests fail, which is not a regression → `2026-10-29:testing:OUT-OF-TREE-ASSETS-DIR`.
- **`cp -p` poisons Ninja's mtime graph.** A restored probe file looks older than the objects built from the modified version, so Ninja skips the rebuild and leaves an ODR-skewed binary. Restore with plain `cp` and touch the headers → `2026-11-17:build:NINJA-MTIME-CP-P`.
- **A local libstdc++ older than the build's compiler leaves `std::filesystem` symbols unresolved at link.** Clear it per build directory with `-Dcpp_link_args=-static-libstdc++`; that is a local build setting, never a repo change → `2026-10-27:toolchain:CI-PINS-AND-GCC17-LINK`.
- **The PR-template bot matches wording exactly.** Paste the template, leave the Checklist lines untouched, then edit prose; `gh pr ready` re-runs the check → `2026-10-26:github:PR-TEMPLATE-BOT`.
- **Fedora 44 in CI pins an older `sdbus-c++`.** Code that only compiles against a newer release fails there and nowhere else → `2026-10-27:toolchain:CI-PINS-AND-GCC17-LINK`.

## Memory

Durable memory for this fork — lessons, measurements, research, action logs, plans, decisions, reviews, and briefs — lives outside this public repository in the maintainer's private warehouse, one directory per repository. Read its lessons index before debugging a build, test, or tooling failure there. Host-specific facts (paths, hardware, timings) stay there and never land here; cite a warehouse entry by handle instead of copying it.
