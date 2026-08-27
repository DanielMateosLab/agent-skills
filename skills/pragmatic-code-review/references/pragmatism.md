# Pragmatism: what is worth raising

Reviewer and skeptic subagents read this file directly from disk — it is never pasted into their
prompts. Sections marked **[SHARED]** apply to both roles: the reviewer uses them to decide what
to raise, and the skeptic uses the same list to check that a finding was allowed to be raised at
all. Holding both to one rubric is what stops the skeptic confirming things the reviewer was
forbidden to report.

Sections 4–7 are lens-specific; each reviewer reads only the one matching its lens.

**Contents**

1. The five tests
2. [SHARED] Do not report
3. [SHARED] Precedents — calibration by domain
4. Frontend / React
5. Hexagonal architecture and layering
6. Data layer and migrations
7. Security
8. Tone

---

## 1. The five tests

Every candidate finding passes all five before it is written down. They exist because a reviewer
model's measured bias runs toward hypothetical corner cases, not away from them — this is a
correction for a known direction of error, not a hypothetical one.

**1. The falsifying-input test.** Can you write the literal input, request body, or state that
produces the wrong behaviour? Not "under concurrency this could" — the actual sequence. If you
cannot, it is not a finding. This is the operational form of *plausible in practice, not merely
possible in theory*, and it is the single strongest filter here.

**2. The reachability test.** Name a caller. If no code path reaches this with the values your
scenario needs, drop it. Most real refutations come from a file other than the one flagged — the
caller, the middleware, the schema, the type — so go look before you write.

**3. The diff-attribution test.** Does *this* change introduce or worsen it, or was it already
true on the base ref? `git blame -L <start>,<end> <file>` settles it. Pre-existing problems are
out of scope; if one is genuinely alarming, it belongs in a single "noticed in passing" line at
the end, not as a finding against this change.

**4. The convention test.** Before flagging a pattern as wrong, grep for it. If it appears three
or more times elsewhere in the codebase, it is a deliberate convention and you are proposing a
refactor of the codebase, not a review of this change. Cite the occurrence count either way.
Never assert "this project's convention is X" from impression.

**5. The counterfactual-cost test.** If the author merges this untouched, what actually happens?
If the honest answer is "nothing observable", it is a nit at most. The reviewer's product is the
author's attention, and attention spent on a non-event is the cost this skill exists to avoid.

Optional sixth, for anything shaped like "this should be extracted / abstracted / generalised":
**the second-caller test.** Does a second consumer exist *today*? If not, do not propose the
abstraction. Solve the problem that needs solving now, not the one that might arrive.

---

## 2. [SHARED] Do not report

These are false positives regardless of how confident they feel. Do not include them.

- **Pre-existing issues.** Anything true on the base ref, and any real problem sitting on lines
  this change did not touch.
- **Anything CI already catches**: lint, formatting, type errors, missing or unused imports,
  undefined variables, unused variables, broken builds. Assume the pipeline runs. Do not run it
  yourself to check.
- **Pedantic nitpicks a senior engineer would not raise in a real PR.**
- **Style, naming and structural preference**, unless a loaded rule in `rules.md` says otherwise
  and you can quote the rule.
- **Theoretical race conditions and timing issues.** Only report a race if you can state the
  concrete interleaving, name the two paths that reach the shared state, and say what breaks.
  "Could race under concurrent access" is not a finding.
- **Speculative breakage elsewhere.** Do not claim a change might break other code unless you
  identified the specific affected path.
- **Missing input validation** on fields with no proven security or correctness impact.
- **Missing hardening / defence-in-depth** where the primary defence is adequate. Code is not
  expected to implement every best practice.
- **Missing docstrings, comments, type hints, logging, or observability**, unless a loaded rule
  requires them.
- **Test coverage in the abstract.** "Needs more tests" is not a finding. "This branch is
  reachable and untested and here is the input that reaches it" is.
- **More specific exception types**, defensive null checks on unreachable branches, and
  micro-optimisations that trade readability for nothing measurable.
- **Anything already silenced deliberately** — a lint-ignore, an explicit comment explaining the
  choice, an `// @review-ok:` marker.
- **Intentional design choices** and changes that are obviously part of the broader intent of
  the change, even if you would have done it differently.
- **Symbols you cannot see.** You are reading a diff. A variable, import, helper or type you
  cannot find may well be defined elsewhere — grep before flagging, and if you cannot resolve
  it, say so rather than asserting.
- **Truncated scope.** A hunk ending at an opening brace or the start of a block is not
  incomplete code. Analyse what is shown.
- **Documentation and markdown files.** No findings in `.md`, `.txt`, `LICENSE`.
- **Generated files and lockfiles** — `*.lock`, `*.pb.go`, `*_pb2.py`, `*.generated.*`,
  `dist/`, `build/`, vendored code, snapshots.
- **Performance claims without a measurement.** LLM reviewers are near-blind to real performance
  bugs. If you have not measured it and cannot point at an obvious algorithmic blowup on a real
  input size, do not raise it.

A clean result is a valid result. If your lens finds nothing, return an empty list — that is
signal, not failure. **An invented issue is far worse than a missed nitpick.**

---

## 3. [SHARED] Precedents — calibration by domain

Settled calls. Do not relitigate them per review.

1. UUIDs and cryptographic tokens can be assumed unguessable and do not need validating.
2. Environment variables and CLI flags are trusted input. An attack that requires controlling
   them is not a finding.
3. A lack of permission or authentication checks in client-side JS/TS is not a vulnerability.
   The server is responsible for authorising and for validating everything it receives.
4. React, Vue and Angular are XSS-safe by construction. Do not report XSS in components unless
   the code uses `dangerouslySetInnerHTML`, `v-html`, `bypassSecurityTrust*`, direct `innerHTML`,
   or `document.write`.
5. Path traversal and SSRF are server-side concerns. Neither is a finding in client-side code.
   SSRF that only controls a URL path, not the host or protocol, is not a finding.
6. Logging URLs and non-PII data is fine. Logging secrets, credentials, tokens, request headers,
   or PII is a real finding.
7. Missing rate limiting, DoS, resource exhaustion, memory or file-descriptor leaks, and regex
   DoS are out of scope.
8. Missing audit logs is not a vulnerability.
9. Outdated or vulnerable dependencies are handled by other tooling. Not a finding here.
10. Low-impact web issues — tabnabbing, open redirect, prototype pollution, XS-Leaks — only if
    you are very confident and can name the impact.
11. Command injection in shell scripts and CI workflows is rarely reachable. Require a concrete
    path from untrusted input before reporting.
12. Memory-safety findings do not apply in memory-safe languages.
13. Code that crashes but is not exploitable is a correctness finding, not a security one. File
    it under the right lens.
14. Test files: only report things that make the test *wrong* — asserting the wrong thing,
    passing when it should fail, or hiding a real failure. Test code is allowed to violate
    production rules.

---

## 4. Frontend / React

The three things AI reviewers reliably get wrong on frontend code are all over-engineering, and
they are the reason frontend review output becomes unusable:

- **Premature abstraction.** Do not propose extracting a hook or component unless a second
  consumer exists today. Extracting for one caller adds a file and an indirection to solve a
  problem nobody has.
- **Reflexive memoization.** Do not suggest `useMemo`, `useCallback` or `React.memo` unless you
  can name the specific expensive computation, or the specific referentially-unstable dependency
  and the component that re-renders because of it. Wrapping a static array is pure cost.
- **Misdiagnosed prop drilling.** Passing props through two levels is passing props. Do not
  recommend Context or a state library for it.

Also out of scope: component file length, folder structure preferences, "this should be a
custom hook", CSS approach, and inline-style-vs-class debates — unless a loaded convention rule
covers it, in which case quote the rule.

What is worth raising in frontend code, because it is real and easy to miss:

- Unstable object, array or function identities in dependency arrays causing render loops.
- `useEffect` with a missing cleanup — subscriptions, timers, listeners, aborted fetches.
- Stale closures over props or state in callbacks and effects.
- Missing or index-based `key` on lists whose items reorder or are removed.
- Conflated state: one variable carrying both loading state and data state, so an impossible
  combination becomes representable.
- Race conditions between overlapping requests where a slow earlier response overwrites a newer
  one — this one is concrete and common; it clears the falsifying-input test easily.
- Accessibility on interactive elements: a `div` with `onClick` and no keyboard handler or role,
  form inputs with no label, focus lost after a modal closes.
- Error and empty states that render `undefined`, `NaN`, or a blank screen.

---

## 5. Hexagonal architecture and layering

Only apply this section if `rules.md` actually contains the project's architecture rules. If it
does, judge against what those rules say and quote them. If it does not, say "architecture rules
not available" in the report and skip the lens — do not substitute your own opinion of what
hexagonal architecture ought to mean. Different teams draw the ports differently and inventing a
boundary the project never agreed to is exactly the kind of finding that erodes trust.

When the rules are loaded, the violations worth catching are the ones that actually leak:

- Domain or application code importing an adapter, a framework, an ORM entity, an HTTP type, or
  anything from infrastructure.
- Business rules implemented inside a controller, repository, or React component instead of in
  the domain.
- A port defined in terms of its implementation — a "repository" interface that leaks SQL,
  pagination cursors, or an ORM's query builder into the domain's vocabulary.
- Domain models mutated directly by an adapter, or persistence models used as domain models.
- A use case reaching for infrastructure directly instead of through its injected port.
- Transaction or session handling escaping into the domain.

Judge the boundary that was crossed, not the file layout. "This file is in the wrong folder" is
a finding only if a rule says so. And a *pre-existing* violation this change merely touches is
out of scope — the diff-attribution test applies here like everywhere else.

---

## 6. Data layer and migrations

Concrete and high value, since these fail in production and not in CI:

- A migration that is not backward compatible with the currently-deployed code — a column
  dropped or renamed while the old version still reads it.
- A migration that cannot be rolled back, or whose down-migration loses data.
- A blocking lock on a large table: adding a non-null column with a default, an index built
  without `CONCURRENTLY`, a table rewrite.
- A backfill running in the same transaction as a schema change, or unbatched over a large
  table.
- A query newly unscoped by tenant, user, or account in a multi-tenant path.
- N+1 introduced by a loop over a relation, where you can name the loop and the query.
- A write path that is no longer atomic — two statements that used to be one transaction, or a
  side effect fired before the commit.
- Unique constraints or nullability that the application code contradicts.

Not worth raising: index suggestions without a measured slow query, "consider denormalising",
ORM-versus-raw-SQL preference, missing `created_at`.

---

## 7. Security

Only flag what you would confidently raise in a real PR. Every finding needs an attack path:
untrusted input → the sink → what the attacker gets. If you cannot trace all three, do not
report it.

Highest value in most web applications: broken tenant or ownership scoping (an ID from the
request used without checking it belongs to the caller), authorisation checked in the UI but not
on the server, a new endpoint missing the auth middleware its neighbours have, secrets or tokens
reaching logs or error responses, PII in a payload that did not carry it before, and
unparameterised SQL built from request data.

Severity: something is HIGH only if it is directly exploitable into unauthorised access, data
exposure, or code execution. Being reachable only from the internal network does not lower that.
MEDIUM only when obvious and concrete. Everything below that is out of scope — see §3.

---

## 8. Tone

Blunt and factual. State the issue, the trigger, the impact, the fix. No praise padding, no
"great work", no softening, and no accusation either. Say how narrow a trigger is upfront rather
than overstating impact — a reviewer that oversells a finding once is discounted on every
finding after.

Uncertainty is stated, not hidden. "I could not verify X" is a legitimate thing to write. A
fabricated definite claim is not.
