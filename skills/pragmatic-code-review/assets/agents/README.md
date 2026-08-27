# Optional agent definitions

Copy these into `.claude/agents/` (project) or `~/.claude/agents/` (global) so the review loop
pins its own models instead of inheriting the session's:

```bash
cp assets/agents/review-lens.md assets/agents/review-skeptic.md ~/.claude/agents/
```

Without them the skill still works — it just has to fall back to whatever the session is running
on, and it will ask before proceeding on a non-Opus session. Detection and refutation are both
judgement calls; a cheap model rubber-stamps, which silently collapses the two-role loop back
into a single-pass reviewer that looks like it is working.
