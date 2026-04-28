# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Repository Purpose

`jquery.fileDownload` is a single-file jQuery plugin that provides an Ajax-like
file download experience (progress dialog, success/failure callbacks, promise
support) while still triggering the browser's native save dialog. The plugin
itself is plain JavaScript with no build step; the rest of the repository is a
small ASP.NET MVC 4 / C# demo application that also doubles as the reference
implementation of the required server-side cookie pattern.

## Build / Test / Lint

There is no build, test, or lint tooling configured in this repository:

- `package.json` declares only metadata (no `scripts`, no `devDependencies`).
- `bower.json` is the primary package manifest (`main: src/Scripts/jquery.fileDownload.js`).
- `index.js` is a CommonJS shim that re-exports the same script so the package
  also works via `require('jquery-file-download')`.
- The demo app (`src/jquery.fileDownload.csproj`, `.sln`) is a Visual Studio /
  MSBuild project targeting .NET Framework 4.0 + ASP.NET MVC 4. It is intended
  to be opened in Visual Studio on Windows; it cannot be built on Linux with
  the default toolchain available here.

When editing the plugin, verify changes by reading the code — there is no
test suite to run.

## Versioning

Three places carry the version number and must be kept in sync when bumping:

- `package.json` → `version`
- `bower.json` → `version`
- `src/Scripts/jquery.fileDownload.js` → the `v1.4.x` banner comment at the top

`CHANGELOG.md` uses HTML (`<h4>`, `<ul>`, `<li>`) rather than Markdown — match
that style when appending new entries.

## Architecture

### Client/server contract (read this before changing anything)

The whole plugin is built around one contract between browser and server:

1. The plugin issues the download request in a hidden iframe (GET/desktop), a
   hidden form in an iframe (POST/desktop), or a new window (iOS/Android) —
   **not** via XHR. This is why the browser's native save dialog appears, but
   it is also why the plugin **cannot send custom headers** and why downloads
   **must come from the same origin as the page**.
2. Because the iframe/window gives no reliable success signal, the plugin polls
   `document.cookie` every `checkInterval` ms (default 100) looking for
   `cookieName=cookieValue` (default `fileDownload=true`).
3. The server is responsible for setting that cookie on a successful file
   response. The plugin then resolves its Deferred, clears the cookie, and
   stops polling.
4. Failure is detected by inspecting the iframe/window document body; if it
   contains HTML after the poll interval, it is treated as an error response
   and the Deferred is rejected with that HTML.

Any change to cookie naming, polling, iframe/form handling, or mobile
detection has to preserve this contract end-to-end.

### Client code

- **`src/Scripts/jquery.fileDownload.js`** — the entire plugin. A single IIFE
  extending `$` with `$.fileDownload(fileUrl, options)`. Key internals:
  - `settings` — user options merged over defaults; see the top of the
    function for the authoritative option list and defaults.
  - Browser branching (`isIos`, `isAndroid`, `isOtherMobileBrowser`) picks
    between iframe, new-window, and direct-navigation strategies. Android is
    explicitly blocked from non-GET downloads (known Android browser bug).
  - `checkFileDownloadComplete` — recursive `setTimeout` loop that polls for
    the success cookie and inspects the iframe for failure HTML. Do not
    convert to `setInterval`; the recursive form is load-bearing for the
    "wait 100ms for IE to finish writing error HTML" path.
  - `cleanUp` — note the preserved comment that iframe removal "appears to
    randomly cause the download to fail"; leaving the iframe in the DOM is
    intentional.
  - Returned promise has an extra `abort()` method attached to it.
- **`src/Scripts/jquery.fileDownload.d.ts`** — TypeScript ambient declarations
  for `JQueryStatic.fileDownload` and `FileDownloadOptions`. Keep in sync
  whenever an option is added, removed, or renamed in the JS.
- **`src/Scripts/Support/`** — third-party libraries (gritter, SyntaxHighlighter)
  used only by the demo page. Do not treat as part of the plugin.

### Server-side demo (ASP.NET MVC 4, C#)

The `src/` folder (outside `Scripts/`) is the demo/reference MVC app:

- **`src/Common/FileDownloadAttribute.cs`** — the reference server-side
  implementation of the cookie contract. It is an `ActionFilterAttribute`:
  after an action executes, if the result is a `FileResult`, it appends the
  `fileDownload=true` cookie; otherwise it expires any stale cookie. This is
  what consumers are expected to copy (or port to their stack) to make the
  plugin work.
- **`src/Controllers/FileDownloadController.cs`** — demo actions showing both
  GET and POST downloads; odd `id` values deliberately throw to exercise the
  failure path.
- **`src/Views/FileDownload/Index.cshtml`** — the demo page. Each usage
  pattern (simple dialog, promise, POST form, custom dialog) is shown twice:
  once as live wiring in a `<script>` block, once as a `<pre class="brush:...">`
  source display. When updating an example, update both copies.
- **`src/Global.asax.cs`** — default route points to `FileDownload/Index`.

### JS ↔ C# coupling

The plugin's defaults (`cookieName: "fileDownload"`, `cookiePath: "/"`) match
`FileDownloadAttribute`'s defaults. Changing a default on one side without the
other silently breaks every downstream consumer that relied on defaults, so
change both together or not at all.

## Conventions

- Preserve support for jQuery 1.6+ and legacy browsers (IE 6+, iOS 5+,
  Android 4+). The script contains several IE-specific workarounds (e.g. the
  `e.number == -2146828218` permission-denied catch, the
  `htmlSpecialCharsPlaceHolders` map that uses `&#39;` because IE8 lacks
  `&apos;`) — do not "modernize" these away.
- No ES6+ syntax in `jquery.fileDownload.js`. Stay within the ES3/ES5 subset
  already in use.
- Comments in the plugin deliberately document *why* each branch exists
  (browser bugs, iframe quirks). Keep that level of explanation when adding
  new branches; terse code here is a hazard, not a virtue.
