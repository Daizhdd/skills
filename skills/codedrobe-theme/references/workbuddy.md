# WorkBuddy target

Use app id `workbuddy`. Read current defaults and the last verified application version from `codedrobe apps --json`. The built-in default CDP port is currently `9336`, but an explicit `--port` must win.

## Apply behavior

WorkBuddy currently uses renderer-only theming and does not require Codex-style host appearance settings. A reachable renderer can normally be themed without restarting the application.

- Use `--app-path` for a nonstandard installation.
- Do not patch `WorkBuddy.app`, its Electron resources, or `app.asar`.
- Apply and restore through Core so image object URLs, observers, styles, and root markers are cleaned consistently.

## Verification surface

Keep the adapter limited to stable cross-route landmarks:

- root: teams container
- sidebar: conversation sidebar/list
- workspace: teams main content, main content, or chat container
- composer: editable textbox

Keep home layout rules in the theme package. For a theme that styles the WorkBuddy home and conversation screens, verify at least:

Capture separate home and conversation snapshots before adapting `assets/theme-starter/workbuddy.css` or `assets/examples/miku-future-beats/workbuddy.css`. Prefer the live semantic classes over any stale example selector.

1. Home header/hero, scene tabs, quick actions, home composer, and named images.
2. A conversation with long text, tables or code, scrolling, and the conversation composer shell.
3. Sidebar selection, hover states, menus, input, microphone, model selector, and send controls.
4. No horizontal overflow or hidden native actions.

## Opaque backplates that hide the wallpaper

An apply can report complete success while the window looks untouched: the renderer paints its own solid surfaces on top of the injected background. On **5.5.6** the wallpaper reaches the screen only after these are cleared.

| Element                                                      | What it painted                                                                    |
| ------------------------------------------------------------ | ---------------------------------------------------------------------------------- |
| `.conversation-shell`                                        | solid white                                                                        |
| `[class*="gridView"]`, `[class*="_grid_"]`                   | solid white — CSS-Module hash classes for the grid layout cells                    |
| `main.wb-home-route`                                         | solid white — **home route only**                                                  |
| `#workbuddy-menubar-container`                               | `rgb(242, 242, 242)`, the top 30 px strip                                          |
| `.workbuddy-window-controls`                                 | `--cb-panel-bg-primary` — the strip behind the minimise / maximise / close buttons |
| `.cr-input-container`, `.cr-input-toolbar__right`            | solid white, inside the composer                                                   |
| `.collapsible-section-header`, `.conversation-section-label` | `rgb(242, 242, 242)` — sidebar group headers                                       |
| `[class*="cb-agent-card"]`                                   | white / `rgb(230, 230, 230)` — sidebar conversation cards                          |

Two traps found while deriving this list:

- **`.wb-home-route` is home-only.** Because that plate does not exist on the conversation route, a theme can look correct in a conversation and washed out on the home screen. Verify both routes.
- **Do not use `[class*="grid_"]`.** It also matches unrelated class names such as `artifact-slot-panel__grid`.

The list is version-bound. When the app updates, re-derive it rather than trusting the table.

## Routes and appearance

- The conversation container is `.conversation-shell` and the home route is `main.wb-home-route`. `.chat-container` no longer exists.
- Appearance is driven by the **`data-theme` attribute on `<html>`** (`light` / `dark`) plus a semantic token set (`--cb-text-primary`, `--cb-panel-bg-primary`, `--cb-bg-secondary`, `--sk-*`, several thousand custom properties in total). Swapping only the `light` / `cb-light` classes changes nothing.
- A theme can therefore follow the host appearance with `html[data-theme="dark"] { … }`, which outranks a plain host selector — so a single package can adapt instead of hard-coding one appearance.

## Wallpaper brightness has to match the host appearance

Text colour follows the host appearance, so when a wallpaper sits behind the UI its brightness decides whether the text survives: a dark image with the light appearance gives dark text on dark artwork, and the reverse. Matched pairs need no treatment at all; mismatched pairs need a scrim.

Solving for the scrim is more predictable than guessing. Sample the wallpaper's mean luminance `L` (scale to roughly 48×48, average `0.2126R + 0.7152G + 0.0722B`), then

- white scrim `a = (196 − L) / (255 − L)` when `L < 196`
- black scrim `a = 1 − 82 / L` when `L > 82`

clamp to ≤ 0.82, and emit one value per appearance. Two notes from practice:

- Mean luminance does not describe how *busy* a wallpaper is. A bright but finely detailed illustration still needs a small white scrim (~0.40), because fine detail behind text reads as noise.
- Because the pairing is what matters, the computed values are a good basis for telling the user which appearance their image wants — and for flagging when it does not match the one they are using.

## Diagnosing "apply succeeded but nothing changed"

A passing apply only proves the style reached the renderer. If the window looks untouched, an ancestor is painting an opaque background. Rather than guessing:

1. Walk up from `document.elementFromPoint(x, y)`, printing `backgroundColor` for each ancestor — this shows which layer covers the image.
2. Enumerate every element whose `backgroundColor` alpha exceeds 0.85 and that intersects the viewport, sorted by visible area. **Do not filter by area threshold** — a size filter misses small chrome such as the window-control strip.

## Do not verify colours through a scripted appearance switch

Setting `data-theme` by hand (or swapping the appearance classes) does not update colours the app writes during its own render pass. Reading them straight afterwards returns the *previous* appearance's value, which looks exactly like "this colour ignores the theme" and invites a wrong conclusion. Scripted switches are fine for previewing background and layout; verify text contrast by switching appearance through the app (avatar → 外观) and measuring after that.

## Surfaces to leave opaque

Clearing backgrounds is not a blanket operation. Keep these as they are:

- **Right detail panel** (`detail-panel`, `detail-main`, `sidebar-next`, `detail-layout`). It exists to preview documents and artefacts; with a transparent background the artwork sits behind the content and hurts reading.
- **Conversation content cards** (`.cr-tool-exp__content`, `.cr-code-like-box`, …), buttons and avatars — content and controls, not window chrome.

Clear chrome only: sidebar, top bar, menu bar, window controls, composer shell.
