# Detached `$EDITOR` (Ctrl+Alt+O) — working notes

Branch: `detached-editor`, rebased onto upstream `339b61a5` (0.7.2, 2026-11-18).
Commits: `2a2d3556` (feature), `061c94f0` (temp-file fix), `4f167b03` (input freeze),
`fb24df4d` (key moved to Ctrl+Alt+O), `83b27c19` (status-bar hint).
This notes file is tracked in the docs commit on top of them: keep notes-only edits
in their own commit and exclude the file from the upstream PR.

## What was built

`Ctrl+Alt+O` — "open the input in `$EDITOR` without hiding the UI" (the existing `Alt+O`
suspends the TUI and blocks on the editor; that is unusable with a GUI editor).

Flow:

1. `Action::EditInputDetached` (routed from the key handler and from
   `maki.ui.action("edit_input_detached")`, priced like `edit_input`: no permission) →
   event loop writes the chat input to a `maki-input-*.md` temp file and spawns
   `$VISUAL`/`$EDITOR` with `Stdio::null()`. The UI keeps running.
2. A detached std thread reaps the child (`child.wait()`) and sends `EditorExit { id }`
   over a tokio `unbounded_channel` wired into the event loop's existing
   `tokio::select!` (0.7.2 moved `event_loop` off flume; the 0.7.1 port used flume)
   (`Wake::EditorExited`).
3. On editor exit: the file is read back and unlinked, and synced into the input via
   `rewrite_input` **only if** `input_box.buffer.version()` is still at
   `DetachedDraft { id, path, baseline_version }`'s baseline. Any mutation of the
   buffer in the meantime means the user took over the input → draft dropped, flash
   hint.
4. While the draft is in flight the chat input is **frozen**: chat keys and pastes
   are dropped, but `Esc` and the bash `Tab` toggle stay alive, and scrolling is
   untouched (mouse is answered in the event loop, the page keys above the chat
   handler). The version check in step 3 remains as the safety net.

Key files: `maki-ui/src/terminal.rs` (`resolve_editor`, `open_in_editor_detached`),
`maki-ui/src/event_loop.rs` (channel, `Wake`, `handle_editor_exit`, action arm),
`maki-ui/src/app/mod.rs` (`DetachedDraft`, key check, routing),
`maki-ui/src/components/{mod,keybindings}.rs`, `maki-lua/src/api/util/command.rs`
(`BuiltinAction::EditInputDetached` + `needs()` arm), `maki-lua/src/api/ui/mod.rs`
(doc table), `maki-ui/src/splash.rs`, generated docs.

## Strategy — and why

The driving requirement (user's actual use case): **Emacs signals "done" by exiting**
when the buffer/Emacs is closed. That makes *process exit* the change signal, which
eliminates every other design we discussed:

- **Not mtime polling.** First proposal checked the temp file's mtime every frame,
  gated on the buffer being untouched. Worked, but the user (rightly) disliked
  per-frame `stat` calls; it also needed extra state (baseline mtime, baseline
  content) to distinguish "editor saved" from "editor touched the file".
- **Not a file watcher** (inotify/FSEvents via `notify`): a new dependency for a
  signal we don't need.
- **Exit-signal wins**: the reaper thread turns exit into a one-shot channel event.
  Zero per-frame work, one buffer-version comparison total, no watcher. The whole
  feature is ~190 lines including docs.

Design decisions worth restating in a PR:

- **Input freeze, then drop-on-local-edit.** While the draft is in flight the chat
  keys and pastes are frozen (`handle_chat_key` early-returns on
  `detached_draft.is_some()`; the `Chat` arm of `route_text_paste` drops its fallback
  after on-screen windows), so the buffer cannot diverge from the file the editor
  holds. The version check on exit is the safety net for what the freeze cannot
  cover: `Ctrl+C`'s first-press discard and plugin bindings that write the input.
  Original drop-on-local-edit alternative (keep syncing and clobber typing) was
  rejected: an autosave landing mid-keystroke would eat user text.
- **`Stdio::null()`, never inherit**: the editor runs beside the UI; inheriting would
  hand a terminal editor maki's live tty and corrupt the screen. Consequence: a *terminal*
  Emacs spawned this way has no tty to render in — `$EDITOR` must be a launcher
  (GUI emacs, `emacsclient`, a wrapper). Deliberately not maki's problem to solve.
- **Key is `Ctrl+Alt+O`, matched exactly** (`key::EDIT_INPUT_DETACHED.matches(key)`,
  `app/mod.rs:2191`). Chosen 2026-11-18 after a precedent survey (see Key convention
  below). Tradeoff accepted: `terminal.rs:559` pushes only
  `DISAMBIGUATE_ESCAPE_CODES`, so the Ctrl+Alt pair is reported as one key event only on
  kitty-protocol terminals. Elsewhere there is no such code point, so the bind simply
  never fires (the bytes a legacy terminal sends for it are not a letter plus both
  modifier bits; crossterm ORs `ALT` onto whatever the byte after `ESC` parses to).
  The existing `Ctrl+Alt+G` (scroll to bottom) has the same gate and no fallback, so
  the precedent tolerates it. The old loose matcher (`code == 'O' && ALT && !CTRL`, needed
  because Alt+Shift+O was ambiguous with Alt+O on legacy terminals) is gone; Alt+Shift+O
  is now unbound, with a test for that.
- **Monotonic draft ids**: re-opening while an editor is still running replaces the
  draft and unlinks the old file immediately; the old editor's late exit event is
  ignored by id mismatch (no resurrection, no double-unlink).
- **BuiltinAction moved to `maki-lua`** upstream during the 71-commit pull; the new
  variant is additive (snake_case name is the Lua-facing API; only renames break
  configs).

## Tradeoffs (accepted, worth flagging upstream)

1. **Sync is on editor exit, not autosave.** Emacs: closing the buffer *is* the done
   signal, so this is right. A GUI editor left open forever never syncs — by design.
2. **One blocking std thread per open detached editor** (the reaper). Negligible (a
   parked thread), but noted.
3. **Emacs prompt on exit**: with null stdio Emacs cannot ask "Save file?" for a
   modified buffer. If the user's config exits cleanly on buffer close (the reported
   use case) there is no issue; otherwise they lose the last unsaved state or Emacs
   reads EOF from the prompt. Document in the PR.
4. **Windows**: unlinking the temp file while the editor still has it open fails
   silently (`let _ =`) and the file lingers until the handle closes. Fine for MVP.
5. **Non-zero editor exit is synced anyway** — the file may be complete; no special
   handling. (The exit code was dropped from the event for KISS after clippy review.)
6. **Editors that return immediately** (`code`, `subl`, `gvim`): the spawned process
   exits while the GUI stays open, so maki syncs back and unlinks right away and the
   editor then opens a nonexistent path. `emacsclient --create-frame` was verified to
   stay alive while the frame is open (blocks until the buffer is killed), so the
   exit-signal design holds for the user's setup. The PR should say `$EDITOR` must
   block until "done".

## Issues found (environment, pre-existing — not caused by this work)

1. **`cargo clippy` is broken on this machine.** Two Rust installs: Homebrew **1.100.0**
   (what `cargo`/`rustc` resolve to) plus a stale rustup 1.98.0 whose shims sit
   in `~/.cargo/bin` (rustup itself not on PATH; `~/.cargo/bin` not on PATH either,
   but cargo probes `$CARGO_HOME/bin` first for *subcommands*, so `cargo clippy`
   runs the 1.98.0 `cargo-clippy`). Homebrew's own
   `/opt/homebrew/bin/{cargo-clippy,clippy-driver}` symlinks are *also* stale 1.98.0
   leftovers; the live 1.100.0 binaries are in
   `/opt/homebrew/Cellar/rust/1.100.0/bin/`. Symptom: "rustc 1.98.0 is not
   supported" (project needs 1.99). Plain `cargo check/test` unaffected. Workaround
   used: `CARGO_HOME=/tmp/cargo-home` (empty dir; registry caching makes it fine)
   — subcommand lookup falls through to the Homebrew cargo's sibling 1.100.0
   `cargo-clippy`. Permanent fix: refresh the brew links
   (`brew unlink rust && brew link rust`) and delete the stale `~/.cargo/bin`
   rustup shims.
2. **`nextest` not installed** (`cargo nextest` → no such command), so `just test`
   can't run; plain `cargo test` was used.
3. **Pre-existing flaky tests** (verified identical on clean `main` via `git stash`
   runs; re-verified after the 0.7.1 rebase, 2026-11-07, and on the 0.7.2 base
   `339b61a5` via a throwaway worktree, 2026-11-18): maki-lua lib — model-tier
   fallback, loader/JIT, plugin_permissions (~5, membership varies run-to-run);
   maki-ui — `agent::tests::background_e2e`, `code_view` diff tests, `input`
   decoration/shell-prompt tests, `messages`, `streaming_content`, `theme`
   (~14-15, membership varies; identical module set on branch and base — base
   2709 passed/15 failed vs branch 2718 passed/14 failed, the +9 are this feature's
   tests). These are the timing-sensitive tests `.config/nextest.toml` has slow-timeout /
   `threads-required` workarounds for; plain `cargo test` lacks those. `theme::tests`
   even fail in isolation, with a different member failing each run →
   shared-state/timing flake in that module. Re-counted 2026-11-18 after the key change:
   14 / 16 / 17 failures across three back-to-back `cargo test -p maki-ui --lib` runs,
   same module set each time, and no key or editor test among them.

## Lessons learned

- **Check git state before implementing** — the working tree "surprise" was our own
  planned edit; but the 71-commit pull behind `main` meant every hook point had to be
  re-verified after the pull (most moved: `BuiltinAction` → `maki-lua`, keybindings
  restructured, `app/mod.rs` split into modules). Always re-grep the anchor sites
  after a large pull, not just assume.
- **Ask about the real editor early.** The feature design flipped twice (polling →
  exit-signal) once the user named Emacs. The "done" semantics of the target editor
  *is* the design.
- **`Bind::matches` is exact** (`code == code && modifiers == modifiers`). Combined
  bitsets *are* const via `.union()` (`keybindings.rs:73` does
  `KeyModifiers::CONTROL.union(KeyModifiers::ALT)` in a const); only the runtime `|`
  operator fails in const, which is what led us astray into `from_bits_truncate`.
- **Shift is not a maki convention for secondary actions** (surveyed 2026-11-18, saved
  as memory `keybinding-modifier-conventions.md`): see the Key convention section below.
  Any future shift-based bind still needs a custom match, since shift is protocol-dependent
  (legacy omits the bit, kitty adds it) and collides with the unshifted letter.
- **`tempfile` 3.x `TempPath` deletes on drop.** `into_temp_path()` returns a
  `TempPath` with drop-cleanup *enabled* (default `disable_cleanup: false`);
  `into_temp_path().to_path_buf()` deleted the file mid-statement, so the editor
  opened a nonexistent path and a fresh empty buffer. The file persists only via
  `TempPath::keep()` (or `disable_cleanup(true)`). The write+keep step is extracted
  to `write_kept_temp_file` so the regression is unit-testable without spawning an
  editor.
- **A live repro with a watcher beats static analysis.** A background loop
  (200 ms) logging `$TMPDIR` contents plus `ps` for the editor process, while the
  user ran the repro, produced the exact timeline (file absent until the editor's
  own save; the client process alive the whole session) and pointed at the drop in
  one glance.
- **clippy `needless_borrow`**: pass `String` where `&str` is expected (deref
  coercion) rather than `&text`.
- **Diagnosing cargo oddities**: `CARGO_LOG=debug` + grepping for the toolchain path
  exposed the dual-toolchain routing in one shot.

## Key convention (why Ctrl+Alt+O)

Surveyed the built-ins on 2026-11-18.

- **`Ctrl+Alt` is the reserved "secondary variant" namespace.** One precedent, and it is
  exactly this shape: `SCROLL_TOP` = `Ctrl+G`, `SCROLL_BOTTOM` = `CONTROL | ALT` on the
  same letter (`keybindings.rs:70-75`, label built with `mod_key!("Alt+G")`). Structural
  support: `components/mod.rs:190` `is_ctrl()` is `CONTROL && !ALT`, so the whole `Ctrl+`
  dispatch layer hands Ctrl+Alt combos to their own namespace by construction. Lua
  notation agrees on the order `<C-M-o>` (`api/keymap.rs:1043`).
- **Plain `Alt` is a second full namespace**, used for actions with no Ctrl slot and for
  Alt spellings of pairs: `Alt+O` (edit input) vs `Ctrl+O` (plan editor), `Alt+Z` (undo)
  vs `Ctrl+Z` (suspend), `Alt+T` / `Alt+,` / `Alt+.` (thinking), `Alt+arrows`,
  `Alt+Del`, `Alt+Home/End`, `Alt+U,D`, `Alt+PageUp/Down`, `Alt+Enter`, and the docs'
  `Alt+S` file-picker example.
- **`Alt+Shift` had no built-in precedent at all.** Only the Lua key parser accepts
  `<M-S-1>` (`key.rs:454,477,495`); `Shift` never distinguishes an action in a `Bind`
  (only `Shift+Enter` newline and the generic `Shift+motion` selection row). Our bind
  was the app's only `ALT|SHIFT`.

Decision: `Ctrl+Alt+O`, replacing the `ALT|SHIFT` bind and the loose matcher. `Alt+Shift+O`
is unbound now. Unverified: `Ctrl+Alt` is AltGr on some layouts, so a layout that
composes a character on AltGr+O could shadow the bind; not tested (kitty disambiguation
usually wins, and `Ctrl+Alt+G` has lived with this since `23d1b142`).

## Remaining work before proposing upstream

**Must:**
- [x] Interactive end-to-end test on real Emacs — done 2026-11-07 via
      `emacsclient --create-frame`: buffer opens with the input's content, edit +
      save + kill frame → syncs back, file cleaned up. Typing in the TUI mid-edit is
      now frozen (`4f167b03`) instead of dropping the draft, and re-opening while the
      editor is up is blocked by the same gate. Still untested: a detached draft in a
      *background* tab (the handler scans all sessions, should work).
- [ ] Green-test confirmation via CI or a machine with `nextest` (local `cargo test`
      can't be trusted for the flaky set; none of the flakes touch this feature, but
      a clean signal helps the PR).
- [ ] Re-run the interactive Emacs repro once on `Ctrl+Alt+O`: the 2026-11-07 run typed
      Alt+Shift+O, and key routing is only covered by unit tests. Ghostty/kitty report
      the pair; if the repro is on a legacy terminal the key will not reach the app at
      all (it arrives as `Ctrl+O`).

**Nice-to-have (code quality / design questions for the PR):**
- [x] Visual indicator while a draft is in flight — done (`83b27c19`): the status
      bar draws ` editing in $EDITOR` in `status_notice` style while
      `detached_draft.is_some()`, as a persistent left-group span fed by
      `StatusBarContext.editor_open` (`app/view.rs`) rather than the flash
      machinery, so it sits next to "auto-scroll paused", survives the error
      arm, and shrinks the cwd/model to make room like a flash does. No
      cadence work: every wake already repaints while a draft is open.
- [ ] Decide/word the help text for the terminal-Emacs null-stdio caveat (splash
      currently says only "without hiding the UI").
- [ ] `DetachedDraft` could live next to `InputBox` rather than as an App field, if
      upstream prefers the input box to own input-related drafts.
- [ ] Consider a unit test for `handle_editor_exit`'s four outcomes (currently only
      key routing is tested); needs a seam (the handler is private to the event loop
      and needs a session with a planted draft + fake temp file).
- [ ] PR description should state: new keybinding, new `maki.ui.action` name, the
      input-freeze-while-open rule, the `Ctrl+C`/plugin safety net, the null-stdio
      caveat, the "$EDITOR must block until done" caveat, and the Emacs rationale.
- [ ] Optional: mention in `site/docs` hand-written pages (permissions/quick-start)
      if maintainers want it; generated pages are already in sync
      (`just gen-docs-check` passes).
