# KDE Plasma integration

Implementation and native Plasma smoke verification: 2026-10-21.

## Behaviour

The existing `menubar::menu_bar` command model feeds both the local menu and KDE's
Global Menu. `native_menu::snapshot` translates it into `craft-desktop::Menu`;
external clicks are queued to the UI thread, checked again, then sent through
`menubar::run_item`. Delete confirmations, photo auto-advance, and Compare/Survey
targets use the existing handlers. Replacing a library retires its queued actions.
Open dialogs, native file pickers, quit prompts and shortcut recording block menu actions.
The local menu is never hidden, and shortcuts remain owned by egui on Linux.

The Linux-only transport exports `com.canonical.dbusmenu`. X11 uses the window's
AppMenu properties and `com.canonical.AppMenu.Registrar`. Native Wayland uses
`org_kde_kwin_appmenu_manager` on the same connection as the application's surface.
The D-Bus object exists before its address is attached to the surface. Winit owns
socket reads; the adapter processes only its own queued events without blocking.
Missing services/protocols leave the local menu functional.

`vendor/craft-desktop` is the reusable portable menu model and optional Linux
transport. `vendor/winit/CRAFT_PATCH.md` documents the sole upstream patch: a safe
surface-proxy accessor, with no new unsafe code. Workspace-local snapshots keep
standalone checkouts buildable; update them together with the ecosystem's canonical
`shared/` sources. Other OS and web builds do not enable the Linux transport.

## Controls and preserved functionality

- Add Plasma's **Global Menu** widget to the desired panel yourself; LightCraft
  does not edit desktop configuration.
- `LIGHTCRAFT_NO_GLOBAL_MENU=1` or `KDE_NO_GLOBAL_MENU=1` disables the adapter.
- Existing display policy is unchanged: XWayland when `DISPLAY` is available;
  `LIGHTCRAFT_WAYLAND=1` selects the native path. Native Wayland file drag-and-drop
  is not supported by the pinned winit version; use Import/Open dialogs.
- File pickers already use rfd's XDG portal backend, parented to the window and
  invoked on a worker. No GTK or Qt dependency was introduced.
- Settings → Interface → Appearance Mode → **Auto** follows the portal's live
  light/dark preference. Dark/Light and Charcoal/Midnight/Silver/Paper selections
  remain intact. This is not an exact Breeze palette or system accent implementation.
- Native window decorations remain enabled on Linux. The titlebar settings in
  `main.rs` that hide the macOS title have no Linux effect.
- Flatpak retains Wayland and fallback-X11 sockets; the explicit D-Bus permissions
  allow its own AppMenu name and the X11 registrar. Packaging/runtime verification
  is still needed; permissions alone are not evidence of a working Flatpak build.

## Automated checks

Run one Cargo process at a time per target directory. On memory-limited systems:

```sh
cargo fmt --all -- --check
CARGO_BUILD_JOBS=1 cargo check --locked -p lightcraft
CARGO_BUILD_JOBS=1 cargo test --locked -p lightcraft -p lightcraft-ui-egui --lib --bins global_menu_
CARGO_BUILD_JOBS=1 cargo clippy --locked -p lightcraft -p lightcraft-ui-egui --all-targets -- -D warnings
CARGO_BUILD_JOBS=1 cargo test --locked --workspace
```

The adapter tests cover shared toggle state, stale-library/modal rejection,
unadvertised parameters, rating/Undo and delete confirmation. Shared transport
tests include D-Bus layout/state signals and wire-level Wayland association/lifetime
checks against a fake peer. These tests do not establish actual Plasma rendering.
The repository's complete gate remains `cargo xtask ci`, including WASM/layers/assets.

## Verified results — 2026-10-21

Automated checks used the normal production dependency graph and default features:

- `cargo check -p lightcraft`: passed. The first offline attempt lacked the cached
  `dhat` index entry; an online check populated dependencies without upgrading existing
  locked versions. Subsequent tests, build and Clippy ran offline.
- Global Menu adapter tests: **4 passed**, covering local/global toggle state,
  stale-library/modal/injected-action rejection, rating/Undo/delete confirmation,
  and user labels/shortcut projection.
- Existing `menubar::` tests, run directly from the already built UI test executable:
  **24 passed in 16.46 seconds**, including local menu switching, scrolling and bounds.
  The full 1,120-test UI suite was not run.
- `cargo build -p lightcraft`, host/UI `cargo clippy --all-targets -- -D warnings`,
  `cargo fmt --all -- --check` and `git diff --check`: passed.
- `xtask assets`: 50 assets, all attributed. `xtask layers`: 33 crates, no violations.
- A test-only host constructor was updated to supply the new optional Linux bridge.
  The build-script warning about missing optional `craft-fonts` remains: Chinese and
  Japanese font coverage was not validated. WASM and full CI were not run.

Native Wayland smoke testing used a throwaway in-memory session on the live Plasma
desktop, not a headless renderer. The actual `plasmashell` D-Bus owner (`:1.24` in
that session) requested `GetLayout`, `AboutToShow` and `Event` from LightCraft's
exporter. The bridge reported no X11 window binding; its menu was associated with
the native Wayland surface. Filmstrip changed from checked to unchecked, matching
the application state, while the local menu remained enabled (`native_menu=false`).
Opening About disabled menu commands. Appearance Auto resolved to the portal's dark
preference; explicit Light was also checked in native application screenshots.
Quitting removed the AppMenu service. No desktop/user configuration was modified.

These results establish the native Wayland menu path and basic live appearance
behaviour, not the entire matrix below. Flatpak, native X11/XWayland, missing-widget
fallback, locale/shortcut live changes, portal file workflows and the full DPI /
mixed-monitor matrix still need their own runtime records.

## Manual verification matrix

Use a throwaway in-memory library, never real photos for destructive checks:

```sh
LIGHTCRAFT_NO_PREFS=1 LIGHTCRAFT_WAYLAND=1 cargo run -p lightcraft -- --memory --control 18983
```

1. In a Plasma Wayland session, focus this window and verify the six top-level
   Global Menu sections. Confirm that no duplicate client-side titlebar appears.
2. Toggle Filmstrip; verify local and global checkmarks agree. Rate a demo photo;
   confirm Undo becomes available and changes the same document when invoked.
3. Invoke Delete and confirm the existing prompt appears. Cancel; do not delete
   user files. Try another queued action while the prompt is open: it must not run.
4. Change the application language and key binding; labels/shortcuts must refresh.
   Open a different temporary library; stale menu clicks must not affect it.
5. Repeat without `LIGHTCRAFT_WAYLAND` for XWayland. This is not a native-X11-session
   test; perform that separately when a Plasma X11 login is available.
6. Repeat with `LIGHTCRAFT_NO_GLOBAL_MENU=1`, or without a Global Menu widget:
   the local menu and shortcuts must remain usable.
7. In Appearance Auto, switch the desktop between light/dark; verify dialogs,
   menus, tooltips and fields. Explicit custom themes must not be replaced.
8. At 100%, 125%, 150%, 175% and 200%, inspect menu/title text, icon sharpness,
   sliders and dialog bounds; resize/maximize/fullscreen and transfer between
   differently scaled monitors. Record actual display scale/backend and results.
9. Open a portal picker from a fullscreen window; verify parenting, cancellation
   and a disposable open/save workflow. Test clipboard text/image as supported.

Drive only LightCraft's control channel for automation; never send desktop-wide
synthetic keyboard or pointer input. Capture only its own `ui.screenshot` output.

## Remaining limitations

No exact Breeze palette/system-accent implementation, no new notifications or tray,
and no claimed 5-scale/mixed-monitor GUI pass from unit tests. Native file-drop,
Linux monitor-profile following and tablet events remain upstream/application gaps.
Keep test outcomes in the integration report; do not infer success from compilation.
