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
| `.wb-home-route`                                             | solid white, 972×734 — **home route only**                                         |
| `#workbuddy-menubar-container`                               | `rgb(242, 242, 242)`, the top 30 px strip                                          |
| `.workbuddy-window-controls`                                 | `--cb-panel-bg-primary` — the strip behind the minimise / maximise / close buttons |
| `.cr-input-container`, `.cr-input-toolbar__right`            | solid white, inside the composer                                                   |
| `.collapsible-section-header`, `.conversation-section-label` | `rgb(242, 242, 242)` — sidebar group headers (**see the note below: not clearable**) |
| `[class*="cb-agent-card"]`                                   | white / `rgb(230, 230, 230)` — sidebar conversation cards                          |

Two traps found while deriving this list:

- **`.wb-home-route` is home-only, and it is not an ancestor of the home content.** It really is `<main class="wb-home-route">`, but on 5.5.6 it is painted as a *sibling underlay* — below the home UI, above your background. Walking up from `document.elementFromPoint` never reaches it, so the ancestor walk returns a clean stack while the window stays washed out. Only full enumeration finds it. This is the exact mechanism behind "looks correct in a conversation, white on home".
- **Do not use `[class*="grid_"]`.** It also matches unrelated class names such as `artifact-slot-panel__grid`.

Write the selector without a tag name — `.wb-home-route`, not `main.wb-home-route`. The element happens to be a `<main>`, but the plate is worth clearing whether or not that stays true across releases.

`.collapsible-section-header` is the one plate on this list that **cannot be cleared from a theme**. The app paints it from `.conversation-section-content [class^="collapsible-section"] > [class*="header"] { background: var(--wb-sidebar-bg, var(--cb-sidebar-bg, var(--vscode-sideBar-background, #fff))) !important }`. Raising your own selector past that — a `(0,4,1)` `!important` was tried — still loses, and redefining `--cb-sidebar-bg` / `--wb-sidebar-bg` does not help either, so specificity alone is not the deciding factor. Leave it native and say so in the theme, rather than shipping a rule that silently does nothing.

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
- **Do not let the scrim go so heavy that it inverts the image's own tonality.** On a bimodal wallpaper this bites in the dark appearance. A left-to-right ramp heavy enough to crush the image's light half (`.93`) took it from 253 down to 28, while the image's dark half — under a much lighter `.62` — landed at 46. The light half ended up *darker* than the dark half, and the sidebar, which sits over that light half, collapsed into a flat near-black band with no texture left in it. The user reads that as a rendering bug, not as a wallpaper. Pick the floor so the light-art region lands in roughly the same tone band as the rest of the window: at `.83` the left end sits at 52 against 52–60 across the content area, and the band disappears. Measure this rather than eyeballing it — sample one horizontal row of the screenshot across the side bar and content, and check the step between them.

  Rough target for light text on a dark theme: keep the whole window under ~110 luminance with the darkest and lightest areas within ~10–15 of each other, or the wallpaper's own structure starts reading as a UI defect.

## Diagnosing "apply succeeded but nothing changed"

A passing apply only proves the style reached the renderer. If the window looks untouched, an ancestor is painting an opaque background. Rather than guessing:

1. Walk up from `document.elementFromPoint(x, y)`, printing `backgroundColor` for each ancestor — this shows which layer covers the image.
2. Enumerate every element whose `backgroundColor` alpha exceeds 0.85 and that intersects the viewport, sorted by visible area. **Do not filter by area threshold** — a size filter misses small chrome such as the window-control strip.
3. Do not trust step 1 on its own. The ancestor walk has a blind spot: a solid **sibling** layer painted below the content but above the background is invisible to it, and its stack comes back clean while the window is still washed out. Whenever step 1 comes back clean and the result is still wrong, run step 2 — that is what surfaces `.wb-home-route`.

Confirm the fix the same way: after applying, read the computed `backgroundColor` of every plate on your clear list and check it is actually `rgba(0, 0, 0, 0)`. A rule can look right in the source and still be losing a cascade fight; only the computed value proves it landed.

## Do not verify colours through a scripted appearance switch

Setting `data-theme` by hand (or swapping the appearance classes) does not update colours the app writes during its own render pass. Reading them straight afterwards returns the *previous* appearance's value, which looks exactly like "this colour ignores the theme" and invites a wrong conclusion. Scripted switches are fine for previewing background and layout; verify text contrast by switching appearance through the app (avatar → 外观) and measuring after that.

## Surfaces to leave opaque

Clearing backgrounds is not a blanket operation. Keep these as they are:

- **Right detail panel** (`detail-panel`, `detail-main`, `sidebar-next`, `detail-layout`). It exists to preview documents and artefacts; with a transparent background the artwork sits behind the content and hurts reading.
- **Conversation content cards** (`.cr-tool-exp__content`, `.cr-code-like-box`, …), buttons and avatars — content and controls, not window chrome.

Clear chrome only: sidebar, top bar, menu bar, window controls, composer shell.

## A wallpaper-only theme contains no geometry

Not every request wants the UI re-skinned. "Change the wallpaper, leave my interface alone" is a normal ask, and it is a *different* theme shape: paint the root, clear the plates, stop there.

Keep sizing and spacing out of the file entirely. A theme that also sets `width`, `max-width`, `gap`, `min-height`, `padding`, `border-radius`, `box-shadow`, `transform`, or `border` on app chrome will visibly restructure surfaces the user never asked you to touch. On 5.5.6 the home screen is where it shows first, because the hero is a centred title that the plate paints behind:

- `.wb-home-page` — `width` / `max-width` / `gap` resize the whole column.
- `.wb-home-header` — `min-height` / `padding` / `overflow` grow the hero and clip its content.
- `.wb-home-header__title` — a `width` constraint re-wraps the centred title.
- `.teams-content-wrapper`, `.wb-scene-tabs`, `.quick-actions__item` — border radius, borders and shadows relocate and restyle chips that were already fine.

The symptom is distinctive: the hero title gets **cut off at the top edge** and the scene tabs appear **detached from the card**, floating over the wallpaper. If you see that, you are not looking at a clearing bug — there is a geometry declaration in the theme. Delete it; the layout returns by itself.

So the whole file reduces to two jobs, and it is worth keeping it that small:

```css
/* 1. paint the wallpaper on the window root, with one veil per appearance */
html.codedrobe-host-workbuddy,
html.codedrobe-host-workbuddy body,
html.codedrobe-host-workbuddy #root {
  background-color: #fbfbfa !important;
  background-image: var(--ink-veil), var(--codedrobe-image-wallpaper, none) !important;
  background-repeat: no-repeat, no-repeat !important;
  background-position: center center, center 20% !important;
  background-size: 100% 100%, cover !important;
  background-attachment: fixed, fixed !important;
}

/* 2. clear the plates from the table above, and nothing else */
html.codedrobe-host-workbuddy .teams-container,
html.codedrobe-host-workbuddy [class*="_gridViewItem_"],
html.codedrobe-host-workbuddy .conversation-shell,
html.codedrobe-host-workbuddy .wb-home-route {
  background-color: transparent !important;
  background-image: none !important;
}
```

Crop note: a 1236×764 window is wider than a 1186×856 reference image, so `cover` scales by width in both directions and there is no horizontal freedom — `background-position` can only move the image vertically. Anchor it to keep the focal point (`center 20%` held the face) and accept that the bottom is what gets cut.
