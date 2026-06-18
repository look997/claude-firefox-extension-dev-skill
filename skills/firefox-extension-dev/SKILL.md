---
name: firefox-extension-dev
description: >-
  Develop, build, and live-debug Firefox WebExtensions — including
  privileged Experiment APIs (experiment_apis) and driving a running
  browser over the Remote Debugging Protocol. Use when working on a
  Firefox extension: writing manifest.json/background/content scripts,
  building an .xpi with web-ext, debugging why an extension won't load
  or a feature doesn't fire, or inspecting Firefox internals
  (ScreenshotsUtils, XPCOM Services, chrome DOM) live.
---

# Firefox extension development & live debugging

Hard-won, field-verified notes for building and debugging Firefox WebExtensions
(verified live on Firefox 149). Most of these are silent failure modes that cost
hours to diagnose.

## Building
- `web-ext build --overwrite-dest --artifacts-dir web-ext-artifacts --ignore-files "web-ext-artifacts/**" ".git/**" "*.md"`
- An `.xpi` is just the produced `.zip` renamed. The manifest must be at the archive root.
- `web-ext lint` on an extension with `experiment_apis` emits `MANIFEST_FIELD_PRIVILEGED` — expected, NOT fatal; `build` still succeeds.

## Experiment APIs (experiment_apis) — gotchas that cost real time
- **`parent.paths` IS REQUIRED.** List every namespace the parent script serves: `"parent": { "scopes": ["addon_parent"], "paths": [["myNamespace"]], "script": "api.js" }`. WITHOUT it, Firefox never binds `browser.myNamespace` to the parent script — `getAPI` never runs, `addListener` silently no-ops (returns without error), and the whole API is a dead stub. This is the #1 silent failure: everything looks registered (manifest parsed, schema loaded, `typeof browser.myNamespace === "object"`) but nothing fires. Diagnose by comparing a WORKING experiment's parsed manifest (`WebExtensionPolicy.getByID(id).extension.manifest.experiment_apis`) — the working one has `paths` populated, the broken one has `[]`.
- **Never import `ExtensionCommon`/`ExtensionAPI`/`Services`/`ChromeUtils` at the top level of `api.js`** — they are already sandbox globals. A throwing top-level `ChromeUtils.importESModule(...)` aborts the WHOLE API registration and cascades into breaking the extension's background + content scripts (icon shows, nothing works). Get `EventManager` via `const { EventManager } = ExtensionCommon;` inside `getAPI`.
- **Don't put unsupported values in manifest `parent.events`** (e.g. `"startup"`). Lifecycle events are update/uninstall only. Register observers lazily on first `addListener`.
- **Reloading a temporary add-on does NOT refresh the parent `api.js` module** — it stays cached in the privileged loader; old observers keep running. Editing `api.js` requires a **full Firefox restart**, not just "Reload" in about:debugging. (Editing the manifest IS picked up on reinstall, so a `paths` fix can be tested without a restart. Editing a content/background script is also picked up on reload — only the experiment parent module is cached.)
- Module-level exports (e.g. `UIPhases` from `ScreenshotsUtils.sys.mjs`) are NOT properties of the main exported object — import them explicitly via the destructure.
- Background runs in the **extension process** (`useRemoteWebExtension=true`), but an `addon_parent` experiment runs in the real chrome process — `Services.wm`/chrome DOM there reflect the actual browser windows, and parent↔background is bridged automatically once `paths` is correct.
- Experiment APIs require a privileged/self-hosted context: a regular AMO-listed extension cannot ship `experiment_apis`. They run when loaded temporarily (`about:debugging`), on Developer Edition / unbranded builds, or with `extensions.experiments.enabled=true` + an unsigned-allowed build.

## Live debugging via the Remote Debugging Protocol (RDP)
You can drive a running Firefox from the inside — including the parent/chrome process — when a debug port is open. Firefox must have been launched with `--start-debugger-server <port>` (and `devtools.debugger.remote-enabled=true`).
- Find the port: `ss -ltnp | grep firefox`. Handshake on connect is `{"from":"root","applicationType":"browser"}`. Wire format is length-prefixed JSON: `<byteLength>:<json>`.
- Eval JS in the chrome process: root → `getProcess{id:0}` → `getTarget` → `consoleActor` → `evaluateJSAsync`. A tiny Python `socket` client over the port (no deps) is enough.
- Useful moves: enumerate observers (`Services.obs.enumerateObservers(topic)`), read the chrome DOM, install a temp addon (`addonsActor` → `installTemporaryAddon`), restart (`Services.startup.quit(eAttemptQuit|eRestart)` — brings tabs back via session restore but LOSES the debug port, since that only opens via the launch flag).
- Reaching a specific addon's background context uses the newer `watcher` actor, not `getTarget`.
- `evaluateJSAsync` does not auto-resolve a returned Promise; stash the result on a global and read it back, or read the `evaluationResult` packet.
- This is far faster than asking a human to paste console logs.

## Detecting Firefox's native screenshot tool (FF 127+)
A worked example of an experiment API hooking a chrome-only feature.
- The screenshot overlay is `[ChromeOnly]` anonymous content — invisible to content scripts. Detection must be chrome-side (experiment API).
- **The real state signal is `ScreenshotsUtils.getUIPhase(selectedBrowser)`** → `UIPhases` enum `{CLOSED:0, INITIAL:1, OVERLAYSELECTION:2, PREVIEW:3}` (a separate module export). "Active" = phase ≠ CLOSED.
- `"menuitem-screenshot"` (Services.obs) is ONLY a wake-up, and **it does NOT reliably accompany every capture** (keyboard shortcut / context-menu / direct paths may not fire it when the phase changes). **Do NOT make detection depend on it.** Keying a poll off the trigger leaves your effect stale through the PREVIEW phase when no fresh trigger fired. Use the notification only as an optional instant wake-up.
- **Do NOT key off the `.screenshotsPagePanel` element.** That panel is the *selection* UI; it hides as soon as you capture, but the flow is NOT over — the capture render and the **PREVIEW dialog** are still up, phase = PREVIEW(3). The capture (`ScreenshotsUtils.createCanvas`) renders the page in **left-to-right column tiles** AFTER the panel hides, so any page-level overlay restored then bleeds a **vertical band** into the right side of the saved image.
- **Correct approach (verified end-to-end):** run ONE continuous, low-cost monitor for the lifetime of the listeners — `setInterval(tick, ~200ms)` comparing the real `getUIPhase` against last-known state, emitting on the edge (≠ CLOSED → start; back to CLOSED → end). Independent of any trigger. There is no DOM event for the CLOSED transition (the clean `screenshots-*` notifications are `Cu.isInAutomation`-only), and `getUIPhase` is a cheap chrome read, so a 200ms poll is negligible. This is a condition-poll on real state, not a guessed delay.
- **Avoid arbitrary timeouts/sleeps to fix timing.** Wait on the real signal (a phase value, an event, a state change) and act when it actually flips. A fixed delay is simultaneously too long and too short.
- **Testing capture/PREVIEW from chrome is FLAKY:** `start()`/`showPanelAndOverlay()`/`takeScreenshot()` throw on `Glean.screenshots[...]` or `this[#overlay] is undefined` when driven from RDP outside the real UI flow (`openPanel()` opens the selection panel but not a full capture). To test the hold logic deterministically, **mock `getUIPhase`** to return 3 for ~2s then 0, fire the trigger, and assert your effect holds while 3 and releases when 0. A real user click hits none of these issues.

## Detecting page state from a content script
- **Direct media file (image/video/audio opened as the top-level document):** use `document.contentType` (real MIME, not spoofable by a `<meta>` tag) — `image/`/`video/`/`audio/` prefix. Corroborate with `document.mozSyntheticDocument` (Firefox-only; guard with `typeof === "boolean"`). Exclude `image/svg+xml` (a real SVG document, not a synthetic MediaDocument). Available from `document_start`; the document type never changes, so no re-evaluation needed.
- **"Dark by default" page:** `getComputedStyle(documentElement).colorScheme` returns the DECLARED list ("light dark"), not the resolved scheme — a `dark` token means the site declared dark support. `<meta name="color-scheme">` is the equivalent declaration. `matchMedia('(prefers-color-scheme: dark)')` is only the OS preference, never a page signal. Note: many sites go dark via `@media (prefers-color-scheme: dark)` WITHOUT declaring `color-scheme`, so the declaration check alone misses them. Do NOT scan `document.styleSheets`/`cssRules` — it throws `SecurityError` on cross-origin sheets and silently drops rules.
- **Timing for late-applied styles:** at `document_start` computed styles / theme `<meta>` may not be in place. Re-evaluate on a `MutationObserver` of `<html>`/`<head>` and finalize on the `load` event; stop watching at `load` (a real signal) rather than after a fixed timeout.

## Testing in a separate Firefox instance
To avoid disturbing a primary browser, launch an isolated instance:
- `firefox --no-remote --profile /path/to/test-profile --start-debugger-server 6001 https://example.com` on a fresh debug port.
- **A fresh profile does NOT have the debug prefs**, so `--start-debugger-server` won't open the port. Write a **`user.js`** (NOT `prefs.js` — Firefox overwrites prefs.js on exit) in the profile dir with: `devtools.debugger.remote-enabled=true`, `devtools.debugger.prompt-connection=false`, `devtools.debugger.force-local=true`, `extensions.experiments.enabled=true`, `xpinstall.signatures.required=false`. To force dark mode for testing: `ui.systemUsesDarkTheme=1` + `layout.css.prefers-color-scheme.content-override=0`. THEN launch.
- Launch detached so the process survives the shell that started it; a bare `&` in a one-off command that exits nonzero may never actually start the process.
- Point the RDP client at the test port and `installTemporaryAddon` there.
