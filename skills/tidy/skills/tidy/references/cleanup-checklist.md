# Cleanup Checklist

The reviewer walks each CHANGED FILE against the items below. Classify each finding as `mechanical_edit` (textual replacement) or `structural_note` (needs human moving / deleting / large rewrite).

Each item is **judgment-style**, not a regex — describe the cleanup opportunity in your own words on the way to producing a fix.

## Items

1. **Redundancy / duplication** — same logic appearing at 2+ sites within the diff (extract or inline); the same data structure declared in multiple places (consolidate).

2. **Dead code** — unused imports, unused variables / parameters, unreachable branches, code that the user explicitly removed earlier in the session and a later edit re-introduced (deletion is authoritative, not a gap to fill).

3. **Over-abstraction / premature generalization** — a helper called from exactly one site, a class with exactly one instantiation, a parameter that always receives the same value.

4. **Defensive guards on already-safe paths** — null checks where the call site guarantees non-null, redundant guard layers where an upstream guard already handles the case (double-coverage).

5. **Speculative features** — functionality beyond the stated requirement; features added "for future use" without an explicit trigger (user requirement, known bug, documented rule).

6. **Comment narration / preamble** — line-by-line paraphrase of code, restating surrounding context, comments that explain *what* the code does instead of non-obvious *why*.

7. **Redundant prose in docs / SKILL.md** — re-statement of what well-named identifiers already convey, repeated rationale that adds no new constraint, paragraphs that paraphrase an adjacent paragraph.

8. **Naming consistency drift (within-diff scope)** — a rename that did not propagate to all call sites within the same change; a function whose name no longer matches its current behavior after the change. Out of scope: pre-existing name mismatches the diff does not touch.

## Overlap handling

When a finding could match more than one item, apply these rules and emit only the preferred classification. **First-match-wins per finding** — never split one finding across multiple items.

- **(1) + (3)** — a single-call-site helper that looks like both duplication and over-abstraction: **prefer (3) over-abstraction**. The defect is that the abstraction was unnecessary, not that the same code repeats.
- **(2) + (5)** — code added "for future use" but never called within the diff: **prefer (5) speculative-features** when the intent reads as "future use" in adjacent prose / comments; **prefer (2) dead code** when no such intent is signaled and the code simply has no caller.
- **(4) + (5)** — a try/catch / null-guard around an operation that currently cannot fail, added "for a future path": **prefer (5) speculative-features**.
- **(6) + (7)** — prose redundancy in SKILL.md / `references/*.md`: **prefer (7) redundant prose** by default; **prefer (6) comment narration** only when the redundant content has the line-by-line paraphrase structure of narration / preamble (not just general repetition).
- **(8)** is independent — rename propagation gaps are a **consistency** layer, not a duplication layer. Apply (8) on its own; do not collapse it into (1).

## Reviewer judgment notes

- The checklist is a starting frame, not a constraint. If a real cleanup opportunity does not fit any item cleanly, surface it as a `structural_note` with the reason — the caller can route it to a human.
- Project conventions under `.claude/rules/` and `CLAUDE.md` override this checklist where they conflict.
- Don't chase aesthetic preferences — fix concrete cleanup wins, leave style-only edits to the formatter.
