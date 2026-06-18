# firefox-extension-dev — Claude Code skill

A [Claude Code](https://claude.com/claude-code) skill with field-verified, hard-won notes for
**developing and live-debugging Firefox WebExtensions** — including privileged Experiment APIs
(`experiment_apis`) and driving a running Firefox over the Remote Debugging Protocol.

Most of these notes are *silent failure modes* that cost hours to diagnose: the extension looks
loaded, the manifest parses, `typeof browser.myApi === "object"` — yet nothing fires. The skill
captures the exact symptoms and fixes (verified live on Firefox 149).

## What it covers

- **Building** `.xpi` with `web-ext`, and the expected (non-fatal) `MANIFEST_FIELD_PRIVILEGED` lint warning.
- **`experiment_apis` gotchas:** the required `parent.paths`, why top-level imports in `api.js` break the
  whole extension, the cached-parent-module reload trap, and where background vs. parent code actually runs.
- **Live debugging via RDP:** connecting to a running Firefox's chrome process, evaluating JS, enumerating
  observers, installing temp add-ons, all over a tiny dependency-free socket client.
- **Detecting Firefox's native screenshot tool** (a worked example of hooking a `[ChromeOnly]` feature),
  including why the obvious signals are wrong and the correct phase-polling approach.
- **Content-script page detection:** direct media documents, "dark by default" pages via `color-scheme`,
  and timing for late-applied styles.
- **Testing in a separate, isolated Firefox** without disturbing your primary browser.

## Install

Copy the skill directory into your Claude Code skills folder:

```sh
# user-global
cp -r skills/firefox-extension-dev ~/.claude/skills/

# or per-project
cp -r skills/firefox-extension-dev /path/to/project/.claude/skills/
```

Claude Code discovers it automatically; it activates when you work on a Firefox extension.

## License

MIT — see [LICENSE](LICENSE).

The skill content is original engineering notes. It references public Firefox internals
(`ScreenshotsUtils`, `getUIPhase`, XPCOM `Services`, the WebExtensions Experiment API) documented at
[firefox-source-docs.mozilla.org](https://firefox-source-docs.mozilla.org/) and
[searchfox.org](https://searchfox.org/); those names belong to Mozilla.
