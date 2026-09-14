# agent-skills

Agent skills I use day to day, installable into any repo: **pragmatic-code-review** and
**ui-flow-screenshots**.

## pragmatic-code-review

A two-role code review loop. A **reviewer** proposes findings; an adversarial **skeptic** gets each
one blinded and tries to disprove it with freshly retrieved evidence. Only findings that survive
reach you. The point is to beat the usual AI-reviewer failure mode — a wall of speculative race
conditions, hypothetical edge cases and premature-abstraction nits that you learn to ignore.

The calibration lives in `references/pragmatism.md`: five "is this worth raising" tests, an
explicit do-not-report list, and per-domain precedents for correctness, security, the data layer
and frontend. Subagents read it from disk themselves, so it stays out of the orchestrator's
context.

**Nothing it produces leaves your machine.** No PR comment, no review, no approval, no commit, no
status check, no Slack or Jira or Notion update. It writes a markdown report and prints a short
summary in chat. You decide what happens next.

### Install

With the [skills CLI](https://github.com/vercel-labs/skills):

```bash
npx skills add DanielMateosLab/agent-skills
```

Add `-g` for global instead of the current project, and `-a claude-code` (or any other supported
agent) to skip the picker.

As a Claude Code plugin — this also installs the two subagents below, which the skills CLI does
not:

```
/plugin marketplace add DanielMateosLab/agent-skills
/plugin install pragmatic-code-review@danielmateoslab
```

Then just ask: "review this branch", "review PR 412", "review my changes before I commit".

### The two subagents

`assets/agents/` holds `review-lens` and `review-skeptic`, which pin `model: opus`. If you
installed via the skills CLI, copy them in yourself:

```bash
cp ~/.claude/skills/pragmatic-code-review/assets/agents/review-*.md ~/.claude/agents/
```

Without them the skill still runs, it just falls back to the session model and asks first if that
model is weaker than Opus. That check matters: both roles are judgement calls, and a cheap skeptic
rubber-stamps, which quietly collapses the two-role loop back into a single-pass reviewer that
looks like it's working.

### Requirements

`git` always. `gh`, authenticated, for `pr` mode — `branch` and `worktree` mode don't need it.
Nothing else; no jq, no python, no npm install.

### What it writes to your repo

`scripts/collect.sh` builds a review pack under `.claude/reviews/pack-<slug>/` (the diff,
per-file patches, the project's own rules at the base ref, and a computed size tier), and the
report lands in `.claude/reviews/`. On first run the script appends `.claude/reviews/` to
`.git/info/exclude` so none of it shows up in your working tree — your `.gitignore` is left alone.

### Project conventions it picks up

The review judges against your project's own rules, not a generic opinion. `collect.sh` loads
`CLAUDE.md`, `AGENTS.md` and `REVIEW.md` at the base ref, plus every `CLAUDE.md` in any ancestor
directory of a changed file, so monorepo package rules come along. It also loads any installed
skill whose name looks like it carries conventions — matching `*convention*`, `*architect*`,
`*hexagon*`, `*clean-arch*` or `*frontend*`. If nothing matches, the architecture lens is skipped
and the report says so, rather than inventing a boundary you never agreed to.

### Known limitations

The remote is assumed to be `origin`. `pr` mode fetches `pull/N/head`, which is GitHub-specific,
so on GitLab it degrades to reviewing the diff alone without reading source files — use `branch`
mode there instead.

## ui-flow-screenshots

The format guide for documenting a user flow with screenshots, so a set is readable by someone who
wasn't there. One directory per flow, `desktop/` and `mobile/` inside every one, files named
`NN_step_name.png` numbered by the order a person performs the steps — not the order you happened
to shoot them, because you always double back and timestamps lie.

It also covers how to split a feature into flows by role without re-shooting the same happy path
twice, optional red highlighting injected into the live DOM (never post-processed onto the PNG),
and the handful of gotchas that each cost an hour the first time: `outline` not painting on a
`<tr>`, ancestors with `overflow` clipping it, toasts that dismiss before the capture lands, and
leftover app state between passes masquerading as a broken app.

Tool-agnostic — Playwright, Puppeteer, or an MCP wrapper around either.

Ask: "screenshot this flow", "document the checkout flow for the PR", "capture these screens on
desktop and mobile".

## Layout

```
skills/ui-flow-screenshots/
  SKILL.md                      # the whole thing: layout, naming, viewports, highlighting

skills/pragmatic-code-review/
  SKILL.md                      # the orchestrator: target resolution, lenses, rounds, gate
  references/
    pragmatism.md               # the calibration rubric — the part that does the work
    reviewer-prompt.md          # reviewer template, lens briefs, finding schema
    skeptic-prompt.md           # skeptic template, verdict enum, termination table
    report-template.md          # report and chat-summary format
  scripts/collect.sh            # builds the review pack
  assets/agents/                # optional review-lens / review-skeptic, pinned to Opus
```

## License

MIT
