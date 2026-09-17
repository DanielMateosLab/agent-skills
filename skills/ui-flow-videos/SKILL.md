---
name: ui-flow-videos
description: Record a user flow as a video a person will actually watch — visible pointer, eased mouse travel, a pause before every click, typing at a human rhythm, chapter cards and captions. Use whenever the user asks for a video, a screen recording, a demo or a walkthrough of a flow or a feature, for a PR, a ticket, release notes, a handover or a stakeholder demo. Covers the flow-recorder setup, the record() hooks, how to script a flow so it reads as a person using the app, and what to know before planning a long one. Needs github.com/DanielMateosLab/flow-recorder.
license: MIT
metadata:
  version: "1.0"
---

# UI Flow Videos

A recording is finished when someone who was not there can watch it once, at normal speed, and
know what the feature does.

Screenshots show states; a video shows the path between them. Pick video when the ordering, the
interaction or the motion is the point, or when the audience will watch rather than follow. When
the audience has to reproduce the steps themselves, give them an ordered screenshot set instead
(`ui-flow-screenshots`) — you can scroll back through a folder, you cannot scrub a webm with one
hand on the keyboard.

## Setup

The recorder lives in its own repo, outside whatever you are filming:

```bash
git clone https://github.com/DanielMateosLab/flow-recorder ~/coding/flow-recorder
cd ~/coding/flow-recorder && npm install
```

Node 22.18 or newer. It runs straight from TypeScript source with no build step, so Node's
strip-only mode applies: no parameter properties, no enums, and `.ts` extensions on relative
imports.

Flows are throwaway — one per thing you are showing, deleted when it ships. They live in
`flows/`, which is gitignored:

```bash
npm run record -- flows/my-flow.ts
```

Output is `videos/<name>.webm`. A take that throws is still written, as `<name>.failed.webm` —
watch that one first, it shows exactly where the flow came apart.

## Writing a flow

One function does everything:

```ts
import { record } from '../src/index.ts'

await record({
  name: 'batch_signing',
  baseURL: 'http://localhost:5174/signatures',
  locale: 'es-ES',

  setup: async () => seedFixtures(),
  session: async (context, data) => { /* cookies, tokens, addInitScript */ },
  teardown: async (data) => retire(data),

  async scenario({ page, hand, chapter, caption, beat }, data) {
    await chapter('Signing a batch', 'Two payments, one code')
    await caption('Everything waiting for this person')
    await hand.click(page.getByRole('checkbox', { name: /Jardines/ }))
    await beat()
    await caption()
  },
})
```

`setup` seeds and its return value reaches every other hook. `session` runs before the first page
exists, which is the only place to plant a token an SPA reads at boot. `teardown` always runs,
including when the scenario throws, so a failed take does not leave fixtures behind.

`hand` has `click`, `hover`, `type`, `scrollTo` and `moveTo`, all taking a selector or a locator,
all moving the drawn pointer with the real one. Timings adjust per flow with
`hand: { travelMs, aimMs, typeMs }`.

## Making it look like a person

- **Never call `page.click` or `page.fill` in a scenario.** They teleport: the state changes with
  nothing on screen explaining why. Every visible action goes through `hand`.
- **One action, then a beat.** A person pauses after a screen changes. Without that the video is
  a series of jump cuts and nobody follows it.
- **Caption the intent before the action, not after it.** "Selects both payments" then the
  clicks. A caption that arrives after the fact reads as a subtitle for something already missed.
- **Do not caption every click.** Caption the step. If the caption repeats what the pointer just
  did, cut it.
- **Wait with the page, pace with `beat`.** `waitFor` is for correctness, `beat` is for the
  viewer. Sleeping instead of waiting is how a video breaks on a slow machine.

## Before planning a long one

- **The recording is wall clock**, 25fps, and cannot be paused. Every wait for a slow API is dead
  air. That is what `chapter()` is for: it covers the screen while something slow happens, the
  way a human editor would.
- **One take, one file.** A four-minute video is a four-minute test and a single timing flake
  loses the whole take. Build each section so it runs on its own, and only then chain them.
- **Playwright records per page.** A second tab produces a second video file. For two roles,
  reuse one page and swap the session instead of opening a tab.
- **Fixture names are on camera.** A test suite avoids locator collisions with a unique suffix
  per record; a video cannot, because the suffix is legible on screen. Give the flow names it
  owns and that nothing else in the environment uses. A name shared with seed data fails as a
  strict-mode violation three minutes into the take, not as a wrong-looking video.
- **Do not import project helpers that import `@playwright/test` at module scope.** Two copies
  loaded in one process is a fatal error by design. If the project's fixtures pull it in for
  something incidental, make that import dynamic and local to where it is used.

## Gotchas

- **Playwright writes webm only.** Anything else needs ffmpeg afterwards; so does stitching two
  takes together.
- **The pointer is drawn by the recorder, not by Chromium.** A native recording has no cursor at
  all, which is why a raw `recordVideo` of a test is unwatchable.
- **The overlay is injected at document-start**, before `document.body` exists, and it is
  serialised by `toString()` — it cannot close over anything outside itself.
- **Video size is the viewport size.** 1280x800 or 1440x900 read well; a phone-width recording
  is a tall thin video that nobody can see in a PR.
- **Run the flow in pieces while you write it.** Every locator that resolves to three elements
  costs you the whole take.

## Before you finish

- Play it. Pacing, overlapping captions and a pointer that lands on the wrong element are not
  visible in the run log.
- No `.failed.webm` left in `videos/`.
- Teardown really removed the fixtures — re-run the flow and see that it starts clean.
- The file is a sane size: a minute of UI is a megabyte or two. Kilobytes means a blank page.

## Report back

The path, the duration, and what each chapter covers. Separately, anything odd you noticed in the
product while filming it: a list that does not refresh, a label that contradicts the state, a
control that jumps as the page settles. Filming a flow at human speed shows things a test never
does — report them, do not fix them.
