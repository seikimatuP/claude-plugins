---
name: verify-diff
description: Empirically verify that a code diff achieves its stated objective by dispatching a bias-free subagent. The subagent returns suggested edits as structured JSON, and this skill applies them iteratively up to max-iterations. Non-interactive — no user prompts. Use after applying an Edit when you need a dynamic cross-check that complements static reviewers like skill-review.
allowed-tools: Read, Edit, Agent, TodoWrite, Bash(git diff *), Bash(git checkout HEAD -- *)
---

# Verify Diff

Empirical check of whether a code diff actually resolves the objective it claims to. A fresh subagent reads the post-diff file and the original problem description without the implementer's bias, returns a JSON verdict (with `suggested_edits` if gaps remain), and this skill applies those edits autonomously — looping until the diff is judged converged, the subagent cannot make further progress, or a safety rail trips.

Designed to be called from non-interactive routines such as `dev-workflow-triage`. It never prompts the user; it either returns a structured summary or terminates early with a machine-readable reason code.

## Invocation contract

The caller passes these fields in natural language (the skill extracts them from the invocation text):

- `Description` — the original problem the diff is supposed to address
- `Suggested fix direction` — how the diff was meant to be shaped
- `Target file` — one relative path (single-file scope; multi-file diffs are out of scope)
- `Base ref` *(optional, default `HEAD`)* — git ref to diff against
- `Max iterations` *(optional, default `3`)* — upper bound on the refinement loop

The caller must **not** stage changes while this skill is running. The skill reads the working tree vs `Base ref`; staged content would mix into the diff and corrupt the verdict.

## Workflow

### Step 1 — Extract context

1. Parse the five fields from the invocation text. If `Target file` is missing or empty, return early:
   ```json
   {"status": "skipped", "reason": "missing target", "iterations_used": 0, "applied_edits_count": 0, "unresolved_gaps": [], "reverted_paths": [], "objective_met": "unknown"}
   ```
2. Run `git diff <Base ref> -- <Target file>`. This captures working-tree-vs-base; no staging is assumed.
3. If the diff is empty, return early with `status=conflict` and `reason="empty diff"`. An empty diff means the caller's Edit did not actually change the file, which is a bug signal the parent should surface as a conflict, not a warning.

### Step 2 — Iteration loop (i = 1 .. Max iterations)

**Pre-register iteration TodoWrite items** — before entering the loop, create `iteration 1`, `iteration 2`, ..., `iteration <Max iterations>` TodoWrite items. Mark `in_progress` before each dispatch, `completed` after parse+apply (for a `converged` verdict, "apply" is a no-op — mark `completed` immediately after parsing the verdict). On early convergence (verdict `status=converged`) or safety-rail triggered exit (`skipped` / `conflict`), mark remaining iteration items `completed` with note matching the exit reason (e.g. `skipped: converged at iter 2`). The "note" lives in the TodoWrite item's `content` field — append as `— <reason>`; TodoWrite has no dedicated note field. Pre-registration is load-bearing: without it, the subagent-driven loop tends to stop after the first iteration that looks acceptable, even when gaps remain that further iterations could close.

#### (a) Dispatch subagent

Invoke the `Agent` tool with a fresh subagent. Pass:

- `Description` and `Suggested fix direction` (verbatim)
- The unified diff (on iter 1, reuse Step 1's output; on iter ≥ 2, re-run `git diff <Base ref> -- <Target file>` so the diff reflects edits that landed in prior iterations)
- The full current contents of `Target file` — re-`Read` at the start of every iteration so the snapshot reflects prior edits
- The judgment rubric and response format below

**Judgment rubric (include verbatim in the subagent prompt):**

> Judge whether the diff resolves the problem in `Description` AND follows `Suggested fix direction`. Also check for regressions — changes that break behavior the original file relied on. Return `objective_met: "yes"` only if there are **no remaining gaps AND no regressions**. Otherwise return `"partial"` (direction is right but gaps remain) or `"no"` (diff does not address the objective).

**Response format (include verbatim in the subagent prompt):**

> Write your reasoning in natural language, then end your response with a single fenced JSON block matching this schema:
>
> ````
> ```json
> {
>   "objective_met": "yes|partial|no",
>   "remaining_gaps": ["<short phrase>"],
>   "regressions": ["<short phrase>"],
>   "suggested_edits": [
>     {"old_string": "<unique snippet>", "new_string": "<replacement>", "rationale": "<why>"}
>   ],
>   "confidence": "high|medium|low"
> }
> ```
> ````
>
> `old_string` must match exactly one location in the current file. Include **1–3 lines of surrounding context** so the snippet is unique — short one-liners collide and cause the Edit to fail.

#### (b) Parse & apply — evaluate in this order, first match wins

1. **Verdict missing or malformed** — no fenced JSON block found, or JSON parse fails → return `status=skipped`, `reason="verdict parse failure"`.
2. **Schema violation** — `objective_met` is not one of `yes|partial|no`, or required keys are missing → return `status=skipped`, `reason="verdict schema violation"`.
3. **Divergence** — only when `i >= 2`: if both `remaining_gaps` AND `regressions` contain the same elements as the previous iteration's values (compare as multisets — sort each array textually before comparison so a reordered-but-identical report still counts as divergence), the loop is not making progress → return `status=skipped`, `reason="divergent gaps"`. (Skip on `i = 1`. Comparing the `(remaining_gaps, regressions)` pair catches a subagent that reports regressions-only with empty `suggested_edits`, which would otherwise loop on empty-equal `remaining_gaps` alone.)
4. **Converged** — `objective_met == "yes"` AND `regressions` is empty → exit loop with `status=converged` and proceed directly to Step 4. Safety rails (c) do not run (their "at least one edit was applied" precondition is not met this iteration).
5. **Otherwise** — apply `suggested_edits` in order:
   - Re-Read the target file before each Edit so `old_string` matches current contents.
   - If an `old_string` is not found, skip that edit and continue with the next. This is expected when the subagent returned multiple edits from a single snapshot and a later edit overlaps a region an earlier edit already rewrote — the skip is a no-op fallback, not an error.
   - After the edits (applied or skipped), run the safety rails in (c), then continue to iteration `i + 1`.

#### (c) Per-iteration safety rails — run only if at least one edit was applied

- **Frontmatter integrity** — Re-Read the file. If the file begins with a `---`-delimited YAML frontmatter block, parse it; if parsing fails:
  ```
  git checkout HEAD -- <Target file>
  ```
  Return `status=conflict`, `reason="frontmatter broken"`, `reverted_paths=[<Target file>]`. If the file has no frontmatter block at all (e.g., a plain source file), skip this rail — there is nothing to corrupt.
- **Scope** — Run `git diff --name-only`. If any returned path is not `<Target file>`:
  ```
  git checkout HEAD -- <each offending path>
  ```
  Return `status=conflict`, `reason="scope violation"`, `reverted_paths=[<each offending path>]`.

### Step 3 — Max iterations reached without convergence

Set `status=unresolved`, `unresolved_gaps = <last remaining_gaps>`. `applied_edits_count` reflects edits that actually landed (not skipped).

### Step 4 — Emit structured summary

End your response with a single fenced JSON block matching this schema:

```json
{
  "status": "converged|unresolved|skipped|conflict",
  "iterations_used": N,
  "objective_met": "yes|partial|no|unknown",
  "applied_edits_count": N,
  "unresolved_gaps": ["..."],
  "reverted_paths": ["..."],
  "reason": "verdict parse failure|verdict schema violation|divergent gaps|frontmatter broken|scope violation|missing target|empty diff|dispatch error|null"
}
```

The `|null` token at the end of the `reason` enum means JSON `null` (not the string `"null"`).

Field semantics by status:

- `reason`: JSON `null` for `converged` and `unresolved`; the matching enumerated string otherwise.
- `objective_met`: the last verdict's value for `converged` (always `"yes"`), `unresolved`, and `skipped (divergent gaps)`. `"unknown"` for all other skipped/conflict paths (verdict was never received, or the verdict itself was unparseable).
- `unresolved_gaps`: the last verdict's `remaining_gaps` for `unresolved` and `skipped (divergent gaps)`; `[]` otherwise.
- `applied_edits_count`: count of `suggested_edits` whose `Edit` call succeeded. Edits skipped because their `old_string` did not match do not count. Applies to all statuses.
- `iterations_used`: the number of iterations whose subagent dispatch returned a verdict, whether or not edits landed — **including the iteration whose verdict triggered `converged`** (which applies no edits itself). Step 1 early returns (missing target, empty diff) count as `0`.

## Dispatch failure

If the `Agent` tool call itself errors, times out, or returns an empty response, return `status=skipped`, `reason="dispatch error"`. Do not re-read the file yourself as a fallback — self-review reintroduces the bias this skill exists to avoid.

## No structure-only mode

This skill does not have a static-review fallback. Callers that want best-practices checking (prose quality, naming, description rigor) should chain `Skill(skill-review)` after `verify-diff`; the two skills are complementary, not substitutes.

## Scope check boundary

`verify-diff` runs its scope check per iteration (inside (c)) to catch leaks the moment they appear. A caller running its own final scope check per Finding provides a last-resort backstop — two independent gates, different granularities.

## Related

- `prompt-tuning` — iterative empirical evaluation of a whole prompt against multi-scenario requirement checklists. Shares the anti-self-review philosophy (dispatch a fresh subagent; never self-review) but operates at prompt-quality granularity, while `verify-diff` operates on a single diff with a single objective.
