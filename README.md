# Tignite Studio

Native Android app (Kotlin + Jetpack Compose, Material3) that wraps
`https://aistudio.google.com` in a WebView, lets you type a prompt from a native
chat bar, and mirrors what AI Studio generates into a native Timeline/Files UI
backed by Room.

## Open in Android Studio

1. Open this folder (`TigniteStudio/`) as a project in Android Studio (Koala or newer).
2. Let Gradle sync — it will pull the Compose BOM, Room, Navigation-Compose, and KSP.
3. Run on a device or emulator with **API 26+** and Play Services (needed for the
   Google login flow to behave normally inside the WebView).

No API keys or secrets are needed — everything runs through the WebView's own
session, the same way a browser would.

## What's implemented

- **Home screen** — header, empty state, Projects list (`ShadcnCard` + `ShadcnBadge`),
  and a fixed bottom `ChatInputBar` (tall multi-line field, black circular send button).
- **Studio screen** — top bar (back / refresh / open-externally), full-screen WebView,
  a floating pill-shaped bottom nav (`FloatingPillNav`) with Chat/Studio/Settings.
- **Studio panel** — slide-up overlay with Timeline and Files tabs, backed by Room.
- **Data layer** — Room entities/DAOs for `projects`, `project_files`, `timeline_entries`,
  with upsert-by-filename so follow-up prompts patch existing files instead of duplicating.
- **Settings** — sign-out (clears WebView cookies) and an About/attribution section.
- **Theme** — shadcn-inspired neutral palette (white/black/gray + one muted slate
  accent), 8dp ("rounded-md") corners everywhere, 1dp borders instead of elevation.

## Two things that will need real-device tuning

**1. Frame-embedding may be blocked.**
The spec's own note is correct to flag this: if `aistudio.google.com` sends
`X-Frame-Options` or a CSP `frame-ancestors` header that rejects being embedded,
the WebView will show a blank or broken page. `AiStudioWebViewFactory` includes a
sanity check (`pageLoadedSanityCheckScript`) that inspects `document.body` after
load and reports `AiStudioLoadState.Blocked` with a clear on-screen message
instead of silently failing — but if this fires, the whole embed-and-scrape
approach needs a different strategy (e.g., driving a real Chrome Custom Tab and
using the Google Drive/Gemini API instead of scraping the web UI).
**Test this first**, exactly as the original spec suggested, before relying on
the rest of the app.

**2. The DOM scraping selectors are best-effort placeholders.**
AI Studio's actual markup (element IDs, ARIA labels, Angular component classes)
isn't public and changes over time. All of the guesswork lives in one file —
`webview/JsInjectionScripts.kt` — in three functions:

- `injectPromptAndSubmit()` — `findComposerInput()` / `findSendButton()`
- `installObserverScript` — `guessFilename()`, and the `pre code` selector used
  to detect generated code blocks

To fix these for real: open AI Studio in desktop Chrome, use DevTools to find
the actual composer textarea and "Run" button selectors, then do the same for
the code-block containers. Update the CSS selectors in that one file — nothing
else needs to change, since every other layer (Room, Timeline UI, Files UI)
just consumes whatever `TigniteJsBridge.onCodeBlock()` reports.

## Security note

`addJavascriptInterface` (used to expose `window.TigniteBridge` to the page) is
scoped to two narrow, side-effect-light methods (`onCodeBlock`, `onPromptSubmitted`)
specifically because that interface is reachable by any JavaScript running in the
WebView's origin — keep it that minimal if you extend the bridge.
