---
name: ui-flow-screenshots
description: Capture a user flow as an ordered screenshot set, at both desktop and mobile viewports, named so the files sort into the order a person performs the steps. Use whenever the user asks to screenshot, capture or document a flow, a feature, a screen, or "what this looks like" — for a PR description, a ticket, a design review, release notes, a bug report or a handover. Covers the directory layout, the NN_step_name naming convention, how to split a feature into flows by role, and optional red highlighting of the one thing that matters in each shot. Applies to any browser-automation tool (Playwright, Puppeteer, or an MCP wrapper around one).
license: MIT
metadata:
  version: "1.0"
---

# UI Flow Screenshots

A screenshot set is finished when someone who was not there can scroll the folder top to bottom
and follow what happened. That takes three things: the right split into flows, an ordering that
survives the filesystem, and both viewports.

## Output layout

```
<output-root>/
  <flow_name>/
    desktop/
      01_step_name.png
      02_step_name.png
    mobile/
      01_step_name.png
      02_step_name.png
```

One directory per flow. Inside it, one directory per viewport, always both. Ask where the
output root goes if the user has not said; default to a `screenshots/` directory outside any
repo the screenshots are about, so they never land in a commit by accident.

## Naming

`NN_step_name.png` — two digits, zero padded, underscore, then the step.

- **The number is flow order, not capture order.** You will double back for shots you missed, so
  file timestamps lie. Decide the running order once the set is complete, then rename.
- **The name says what the screenshot shows**, not what you did to get it: `03_empty_cart`,
  `07_confirm_dialog`, `09_payment_failed_banner`.
- **snake_case, lowercase, ASCII.** No spaces, no accents, no `ñ`, no `:` or `/`. Write step
  names in whatever language the user writes in, but strip diacritics from the filename.
- **Prefix steps on the same screen alike** (`cart_empty`, `cart_two_items`) so a long flow
  groups visually.
- **The same step is the same number and the same name at both viewports.** That is what lets
  someone open two files side by side and compare.
- **A step that exists at only one viewport gets a letter**: `01b_nav_drawer_open.png` in
  `mobile/` when desktop shows that navigation permanently. The letter keeps every later number
  aligned across the two sides.

## Viewports

Both, always, for every flow. Defaults, unless the user or the project says otherwise:

| | Size | Notes |
|---|---|---|
| desktop | 1440x900 | |
| mobile | 390x844 | iPhone 14 |

Keep the device scale factor at 1 so one CSS pixel is one image pixel. Shoot the viewport, not
the full page, except where the point of that step is content that does not fit — then say so in
the report, because the file will be a different height from its pair.

Do the whole flow at one viewport, then the whole flow again at the other. Reset application
state between the two passes, exactly as you would between flows.

## Splitting a feature into flows

One flow per **perspective**, not per screen. An administrator creating the thing, a customer
approving it, someone hitting the error path.

**Do not re-shoot a path you already have because a second role can also walk it.** If the happy
path is captured from one role, the second role only gets the steps that differ for them. The
test of a good split is coverage, not symmetry: every new screen and every distinct state
appears somewhere, at both viewports, exactly once.

Include the negative cases. A control that is absent because the action is unavailable is worth
a frame, next to the frame where it is present.

## Highlighting

Optional, and worth it for anything someone will follow as instructions. One or two elements per
shot: where the click goes, or the one thing to look at. Highlight half the screen and you have
highlighted nothing.

Inject it into the live page immediately before the capture and remove it after. Do not
post-process the PNG — that needs an image library, and the coordinates drift the moment the
layout does.

```js
el.style.outline = '3px solid #e5484d'
el.style.outlineOffset = '2px'
el.style.borderRadius = getComputedStyle(el).borderRadius
el.style.boxShadow = '0 0 0 6px rgba(229,72,77,.22)'
```

`outline`, never `border`: a border participates in layout and the shot stops matching the real
screen. Tighten the offset and halo to 2-3px where the target sits close to a screen edge or
another control, which is common on mobile.

**If a screen has nothing worth pointing at, capture it clean and say so.** A form where every
field must be filled is the usual case: marking one field states something false, and marking
the whole form states nothing.

Write highlighted versions to a parallel tree with identical filenames
(`<output-root>-highlighted/<flow_name>/...`) so the clean set survives. Skip the numbers and
labels; the outline plus the ordered filenames carry it.

## Gotchas that cost an hour each

- **`outline` does not paint on `<tr>` in Chromium** — you get the bottom edge only. Overlay an
  absolutely positioned div at document coordinates instead.
- **Ancestors with `overflow` clip the outline.** Set `overflow: visible` on the offender and
  revert it after, but never on a container that genuinely scrolls: that changes the screen.
- **Toasts auto-dismiss before the capture lands.** Freeze the page's dismissal timer, then
  shoot. Do not retype the toast content by hand.
- **`fullPage` does not expand a panel with its own scrollbar.** Size the viewport instead.
- **Reset application state between flows and between viewport passes.** Leftover state — a live
  one-time code, a rate-limit cooldown, a record already consumed by the previous pass — is the
  source of most "the app is broken" detours that turn out to be the harness.
- **Some steps consume state that a reset cannot restore** (something rejected, deleted,
  already used). Plan the running order so the destructive step comes last, or budget fresh
  fixtures for the second pass.

## Before you finish

- Both viewport directories hold the same step names, with the same numbers.
- No two files in a flow are byte-for-byte identical. A duplicate means two steps captured the
  same screen: either the step numbering is wrong, or those two steps are one step.
- Nothing is empty, truncated or captured mid-render.
- `find <output-root> -type f | sort` — read it, and paste it into the report.

## Report back

The tree, what each flow covers, which steps you left unhighlighted and why, anything that did
not fit the convention. Separately, anything odd you noticed in the product while walking it: a
stale list, a label that contradicts the state, a control that overflows at 390px. You have just
performed the flow more attentively than most people ever will, and that is worth writing down —
but report it, do not fix it.
