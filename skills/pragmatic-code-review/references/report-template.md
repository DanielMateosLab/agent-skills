# Report template

Written to `.claude/reviews/<YYYY-MM-DD>-<slug>.md`. Nothing here is posted anywhere.

Keep it dense. The user is reading it to decide what to do next, not to admire the process.
Omit any section that would be empty, except **Dropped** and **Coverage**.

**Dropped** always appears when anything was dropped, because what was cut and why is how the
user calibrates the gates. **Coverage** always appears, full stop — and it carries the weight
when a review found nothing, since at that point the dropped table is empty too and the report
would otherwise be indistinguishable from a review that silently failed. On a clean change,
Coverage is the report: one line per lens saying what it actually looked at.

---

```markdown
# Review — <target> · <YYYY-MM-DD>

<base ref> → <head ref> · <N> files, +<A>/−<B> · tier: <trivial|standard|large>
Lenses: <list> · Rules loaded: <CLAUDE.md@base, packages/api/CLAUDE.md, convention skills, …>
<M> raw findings → <K> after dedup → <J> verified → <R> reported

**Verdict: <Ready to merge | Not ready | Ready with fixes>** — <one or two sentences>

---

## Blocking (P0)

### 1. <one-line title> — `path/to/file.ts:214`
`confidence 100 · P0 · correctness`

<What is wrong, in two or three sentences. Lead with the trigger.>

```ts
const d = order.discount;
```

**Trigger** <the concrete input or state>
**Impact** <what the user or the system actually experiences>
**Verified** <what the skeptic went looking for and what it found — one line, with a file:line>
**Fix** <the smallest change that resolves it>

---

## Worth fixing (P1)

<same shape>

## Optional (P2)

<one line each — file:line, the issue, the fix. No expanded blocks.>

## Contested

### <title> — `path:line`
**Reviewer** <the claim and the evidence it cited — ~3 lines>
**Skeptic** <the refutation and the evidence it cited — ~3 lines>
**Unresolved after <N> rounds.** <what would settle it — a test to run, a question to answer>

## Unverified

<findings whose skeptic could not reach a verdict, or that exceeded the round budget. One line
each, with what is missing. These are kept deliberately: a skeptic running out of road is not a
refutation.>

## Dropped

| Finding | Verdict | Why |
|---|---|---|
| `src/a.ts:9` quoted code does not match the file | REFUTED | fabricated |
| `src/b.ts:31` null deref in the retry path | OUT_OF_SCOPE | pre-existing, root cause untouched |
| `src/c.ts:88` concurrent writes could interleave | UNFALSIFIABLE | no input produces an observable failure |
| `src/d.ts:12` extract this into a hook | — | single consumer; second-caller test |
| `src/e.ts:55` naming inconsistent with neighbours | P3 | below the severity gate |

## Coverage

- correctness: <N> files, <what it checked> — <K findings | nothing found>
- security: <same>
- <files not reviewed and why: generated, lockfiles, vendored, or truncated>
- <rules expected but missing, e.g. "no convention skills found — architecture lens skipped">
- <anything about the change itself: it is incoherent, it does two things, the ticket asked for
  something it does not do, it is too large to review well>
```

---

## Notes on writing it

**Lead with the trigger, not the taxonomy.** "Returns 500 when the coupon code is unknown" tells
the user more in six words than a category label does.

**One finding, one entry.** If the same defect appears at four call sites, that is one finding
with four locations, not four findings.

**The Dropped table is not an apology, it is the calibration surface.** It is how the user sees
whether the gates are cutting the right things. Keep the "why" column short and specific — the
verdict name plus five words.

**Do not include a "Strengths" section.** Praise from a machine reviewer carries no information
and costs the user reading time. If the change is genuinely good, the verdict line says so.

**Coverage honesty.** If the diff was too large to review properly, or a file was skipped, say
which and why. A confident review of 30% of a change is worse than an admission.

**An unverified P0 or P1 belongs in the verdict line**, not buried in the Unverified section.
"We could not determine whether this crashes production" is exactly the thing a person needs
before merging, and it is the one case where a capped-confidence finding still blocks.

## The chat summary

Short. Severity-ordered. The user has the file for details.

```
Review written to .claude/reviews/2026-08-11-pr-412.md

  P0  src/orders/apply-discount.ts:214   500 on unknown coupon — order is null
  P1  migrations/0042_add_status.sql:8   non-concurrent index on orders (2.1M rows), blocks writes
  P1  src/hooks/useOrders.ts:33          stale response overwrites newer one on fast re-filter

  1 contested · 2 unverified · 9 dropped (6 refuted, 3 below gate)

Ready with fixes. Nothing posted — say the word if you want PR comments drafted.
```

Then one line offering what is next: fix the blockers, re-review after fixes, or draft PR
comments for approval. Offer; do not start.
