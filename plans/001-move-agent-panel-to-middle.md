# Make Agent Panel the Primary Center View

This ExecPlan is a living document. The sections `Progress`, `Surprises & Discoveries`, `Decision Log`, and `Outcomes & Retrospective` must be kept up to date as work proceeds.

No `PLANS.md` existed in this repository when this plan was created, so this document is the single source of truth for this change.

## Purpose / Big Picture

Make the Agent Panel the main, center view of the editor window so the traditional text editor is no longer the primary middle view. The user-visible result is that launching Zed shows the Agent Panel in the center, and the old toggle shortcut (cmd-?) is removed because the panel is always present. A person can verify this by launching Zed and confirming that the center view is the Agent Panel and that cmd-? no longer triggers any action.

## Progress

- [x] (2026-01-13 04:53Z) Identified the workspace center render path, panel docking system, and Agent Panel initialization/keymap locations.
- [x] (2026-01-13 05:24Z) Added a workspace main-view override and rendered it in place of the center pane group.
- [x] (2026-01-13 05:24Z) Marked the Agent Panel as not dock-visible and updated dock focus/open behavior.
- [x] (2026-01-13 05:24Z) Wired Agent Panel initialization to set/clear the main view based on AI enablement.
- [x] (2026-01-13 05:24Z) Removed the Agent Panel toggle keybinding from default keymaps and updated docs.
- [x] (2026-01-13 05:24Z) Ran `./script/check-keymaps`.
- [ ] Manual UI validation with `./script/zed-local` (confirm main view and no Agent Panel dock button).

## Surprises & Discoveries

- The workspace crate is compiled independently of the agent UI crate, so the center render override must be generic (accepting `AnyView`) rather than referencing `AgentPanel` directly.
- Agent Panel is currently a dock panel accessed via `Workspace::panel::<AgentPanel>()`, so hiding it from docks without breaking call sites requires a panel-level “dock visibility” flag.

## Decision Log

- Decision: Add a generic “main view” override to `Workspace` and set it from Agent Panel initialization rather than refactoring Agent Panel into a workspace item.  
  Rationale: Keeps dependencies one-way (workspace stays agent-agnostic) and avoids wider changes to the item system.  
  Date/Author: 2026-01-13 / Codex

- Decision: Keep Agent Panel registered as a panel for lookup and actions, but mark it as not dock-visible and reroute focus/open calls to the main view.  
  Rationale: Preserves existing call sites while preventing duplicate rendering in docks.  
  Date/Author: 2026-01-13 / Codex

- Decision: Remove the default Agent Panel toggle keybinding in all default keymaps, not just macOS, to keep behavior consistent across platforms.  
  Rationale: The panel is always present, so a toggle shortcut is no longer meaningful on any OS.  
  Date/Author: 2026-01-13 / Codex

## Outcomes & Retrospective

Not started.

## Context and Orientation

The main editor layout is built in `crates/workspace/src/workspace.rs`. The center area is rendered from a `PaneGroup` (`self.center.render(...)`) that hosts editor items, and three docks render panels. Panels implement the `Panel` trait in `crates/workspace/src/dock.rs` and are added to docks via `Workspace::add_panel`. The Agent Panel is implemented in `crates/agent_ui/src/agent_panel.rs` as a dock panel and is loaded from `crates/zed/src/zed.rs` in `initialize_agent_panel`. Default keybindings live in `assets/keymaps/default-macos.json`, `assets/keymaps/default-linux.json`, and `assets/keymaps/default-windows.json`.

No new comments should be added to code during implementation.

## Plan of Work

First, add a generic main-view override to the workspace so the center area can render any `AnyView`. In `crates/workspace/src/workspace.rs`, add fields to `Workspace` for `main_view: Option<AnyView>` and `main_view_focus_handle: Option<FocusHandle>`, initialize them to `None` in `Workspace::new`, and add methods `set_main_view`, `clear_main_view`, and `has_main_view`. Update `impl Focusable for Workspace` to return the main view focus handle when present. In `Workspace::render`, introduce a small helper (a private method or closure) that chooses between the stored `main_view` and `self.center.render(...)`, and replace each `self.center.render(...)` usage in the bottom dock layout branches with that helper. Also gate `centered_layout` and padding calculations behind `self.main_view.is_none()` so editor-centric padding does not apply to the Agent Panel view.

Second, extend the panel system to support panels that should not appear in docks. In `crates/workspace/src/dock.rs`, add a new `fn shows_in_dock(&self, cx: &App) -> bool` method to the `Panel` trait, defaulting to `true`. Add a corresponding method to `PanelHandle` and implement it for `Entity<T>`. Update dock UI and activation logic to skip panels where `shows_in_dock` is false: filter `PanelButtons` rendering so hidden panels never show as buttons, make `first_enabled_panel_idx` ignore hidden panels, and ensure `restore_state` ignores hidden panels when selecting the active panel. Keep the panel entry in the dock’s internal list so `Workspace::panel::<T>()` still works, but do not allow hidden panels to become visible or active in dock UI.

Third, adjust workspace panel focus helpers to respect the new dock visibility. In `crates/workspace/src/workspace.rs`, update `focus_or_unfocus_panel` and `open_panel` to short-circuit when the target panel exists but `shows_in_dock` is false: focus the panel’s focus handle without opening any dock and return the panel. This keeps existing `workspace.focus_panel::<AgentPanel>(...)` calls working while preventing dock activation. `toggle_panel_focus` should use the same behavior; it may always focus the panel for hidden panels instead of toggling away, which is acceptable because the toggle keybinding is removed.

Fourth, mark the Agent Panel as main-view-only. In `crates/agent_ui/src/agent_panel.rs`, implement `shows_in_dock` in the `Panel` impl to return `false`. Do not add any new comments.

Fifth, wire the main view into initialization. In `crates/zed/src/zed.rs`, update `setup_or_teardown_ai_panel` so that whenever a panel is present and `panel.read(cx).shows_in_dock(cx)` is false, it calls `workspace.set_main_view(panel.clone().into(), panel.read(cx).focus_handle(cx), window, cx)`. When AI is disabled or the panel is removed, clear the main view with `workspace.clear_main_view(cx)`. This must happen both on initial load and when settings change via the existing `SettingsStore` observer.

Sixth, remove the default Agent Panel toggle bindings. In `assets/keymaps/default-macos.json`, `assets/keymaps/default-linux.json`, and `assets/keymaps/default-windows.json`, remove the `agent::ToggleFocus` keybinding entries. Keep the JSON structure valid and do not leave trailing commas or blank lines.

Seventh, update documentation that instructs users to open the Agent Panel with the toggle shortcut. In `docs/src/ai/external-agents.md` and `docs/AGENTS.md`, replace the “open the agent panel with {#kb agent::ToggleFocus}” phrasing with “the agent panel is the main view” and remove the keybinding references. Only adjust wording that is now inaccurate.

## Concrete Steps

1. From the repo root, re-open the key files for quick reference.

    cd /Users/khoi/Developer/code/github.com/khoi/zed
    rg "struct Workspace" crates/workspace/src/workspace.rs
    rg "trait Panel" crates/workspace/src/dock.rs
    rg "AgentPanel" crates/agent_ui/src/agent_panel.rs
    rg "ToggleFocus" assets/keymaps/default-*.json

2. Implement the workspace main-view override in `crates/workspace/src/workspace.rs`, including render selection and focus handle changes.

3. Implement the dock visibility flag in `crates/workspace/src/dock.rs` and update the dock rendering/activation paths to skip hidden panels.

4. Update workspace panel focus/open helpers in `crates/workspace/src/workspace.rs` to short-circuit for hidden panels.

5. Mark the Agent Panel as hidden-from-dock in `crates/agent_ui/src/agent_panel.rs` and update `setup_or_teardown_ai_panel` in `crates/zed/src/zed.rs` to set/clear the main view.

6. Remove `agent::ToggleFocus` bindings from default keymaps and update docs.

7. Run keymap validation.

    cd /Users/khoi/Developer/code/github.com/khoi/zed
    ./script/check-keymaps

    Expected output is a clean exit; if any keymap JSON error appears, fix the file and rerun until the script exits without errors.

8. Manually validate the UI by launching a local build (choose one command and stick to it).

    cd /Users/khoi/Developer/code/github.com/khoi/zed
    ./script/zed-local

    After launch, the center view should be the Agent Panel, the Agent Panel button should not appear in dock buttons, and cmd-? (or other removed bindings) should do nothing.

## Validation and Acceptance

- Launching Zed shows the Agent Panel in the center and no editor text area is visible by default.
- The Agent Panel no longer appears as a dock button and does not open a dock when focus actions are invoked.
- Default keymaps contain no `agent::ToggleFocus` bindings, and `./script/check-keymaps` passes.
- The Agent Panel still responds to actions like creating a new thread, confirming it remains functional as the main view.

## Idempotence and Recovery

All steps are safe to repeat. If the UI does not render, clear the main view by temporarily calling `workspace.clear_main_view(cx)` in initialization to confirm the center pane renders, then reapply the main-view wiring. If keymap validation fails, fix the JSON and rerun `./script/check-keymaps` until it passes.

## Artifacts and Notes

After keymap validation, a successful run should exit cleanly without errors. If it fails, expect output similar to:

    keymap: invalid JSON in assets/keymaps/default-macos.json: ...

Use that path and error to repair the JSON.

## Interfaces and Dependencies

Additions in `crates/workspace/src/dock.rs`:

    trait Panel {
        fn shows_in_dock(&self, cx: &App) -> bool { true }
    }

    trait PanelHandle {
        fn shows_in_dock(&self, cx: &App) -> bool;
    }

Additions in `crates/workspace/src/workspace.rs`:

    pub fn set_main_view(&mut self, view: AnyView, focus: FocusHandle, window: &mut Window, cx: &mut Context<Self>);
    pub fn clear_main_view(&mut self, cx: &mut Context<Self>);
    pub fn has_main_view(&self) -> bool;

Agent Panel implementation in `crates/agent_ui/src/agent_panel.rs`:

    fn shows_in_dock(&self, _cx: &App) -> bool { false }

Change Note (2026-01-13 05:24Z): Updated Progress to reflect completed implementation steps and keymap validation results.
