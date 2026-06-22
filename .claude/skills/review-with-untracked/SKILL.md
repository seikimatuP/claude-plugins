---
name: review-with-untracked
description: Temporarily visualize untracked new files into the git diff so a chain of diff-deriving review callees can see them, then restore the files to untracked. Runs `git add -N` on untracked new files (intent-to-add), dispatches the requested review callees (rules-review / skill-review / publicity-review / verify-diff / tidy) in order, then `git reset -- <paths>` to restore the untracked state. Holds a closed list of each callee's visualization and restore-safety assumptions. Non-interactive — no user prompts. Project-local routine — not for marketplace distribution.
allowed-tools: Read, Bash(git status --porcelain*), Bash(git add -N *), Bash(git reset -- *), Bash(git rev-parse *), Skill(rules-review), Skill(skill-review), Skill(publicity-review), Skill(verify-diff), Skill(tidy)
---

# Review With Untracked

Review callees that derive their review scope from the git working-tree diff are **blind to untracked new files** — `git diff <base>` excludes paths that are not yet tracked, so a brand-new file created during a task is invisible to them. This skill brackets a review chain so those callees can see untracked new files: it runs `git add -N` (intent-to-add) on the untracked new files so they appear in `git diff <base>` as `+` additions, dispatches the requested review callees in order, then runs a path-scoped `git reset -- <paths>` to return the files to untracked.

The skill is **project-local** (lives under `.claude/skills/review-with-untracked/`, not registered in `.claude-plugin/marketplace.json`) and non-interactive — designed to be wrapped by review-orchestrating routines (e.g. `dev-workflow`'s `hooks.on_complete`, `dev-workflow-triage`, `triage-review`).

## No-Stall Principle

This skill is non-interactive and has **zero user-judgment gates**. Every callee return is a structured return value to parse-and-proceed-past, never a checkpoint to confirm with the user. The only user-facing output is the § Return contract verdict at the end.

**Permissible fatal-abort exits** (emit the § Return contract in `status: "error"` form and stop) — closed list:

- Not inside a git work tree, or `Base ref` does not resolve to a commit (`git rev-parse <Base ref>` fails) → `reason: "base ref unresolved"`.
- `Callees` is empty, or contains a name outside the § Closed list dispatch vocabulary → `reason: "unknown callee"`.

Everything else is non-fatal — record and continue:

- A callee dispatch error or unparseable verdict — record it under that callee's per-callee status and proceed to the next callee.
- A `corrupted_paths` detection (§ Procedure step (f)) — recorded and surfaced in the verdict, never an abort.

**Callee return-point reminder**: after a callee's verdict is parsed, the next tool call — the next callee dispatch, or step (e) restore when the last callee returned — must be issued in the same turn. Do not insert a "shall I proceed?" turn or re-render a callee verdict as a standalone deliverable.

**Restore is mandatory on every normal and callee-error exit path** (§ Procedure step (e)). A hard tool-level error or session death between step (c) and step (e) is out of scope for this principle — § Constraints names the resulting leftover-`A` window.

## Invocation contract

The caller passes these fields in natural language (the skill extracts them from the invocation text):

- `Base ref` *(optional, default `HEAD`)* — the git ref the review callees diff against. Forwarded to each callee in the callee's own argument form (see § Closed list).
- `Callees` *(required)* — an ordered list of callee names to dispatch, each drawn from the § Closed list dispatch vocabulary. Dispatched in the given order.
- `Scope` *(optional)* — additional path globs to exclude from visualization, on top of the always-excluded workflow-artifact paths (`.claude/plans/**`).

## Closed list of per-callee visualization & restore-safety assumptions

The list scope is **review callees that derive their review scope from the git working-tree diff** (`git diff <base>`, or `git status --untracked-files=all` for untracked in `tidy`'s default mode) — the only callees `git add -N` visualization affects. It records, per callee: how it scopes its diff, whether it is blind to untracked files without visualization, its **restore-safety class** (whether its internal `git checkout HEAD -- <file>` rail handles a HEAD-absent file), and the argument form this skill passes it.

`git add -N` is the bridge: `git diff <ref>` excludes untracked files but **includes** intent-to-add files, so visualizing an untracked file turns it into a reviewable `+` hunk.

| Callee | Diff-scope mechanism | Untracked visibility (without add -N) | Restore-safety class | Argument form |
|---|---|---|---|---|
| `rules-review` | `git diff --name-only <base>` | blind | safe — detect-only (no `Edit`, no rail) | `--base-commit <sha>` |
| `skill-review` | `git diff <Base ref>` | blind | **unsafe** — frontmatter rail `git checkout HEAD -- <file>` has no HEAD-absent handling | `Base ref: <ref>` |
| `publicity-review` | `git diff <Base ref>` | blind | **unsafe** — same frontmatter rail | `Base ref: <ref>` |
| `verify-diff` | `git diff <Base ref>` | blind | **unsafe** — same frontmatter rail | `Base ref: <ref>` |
| `tidy` | `git status --untracked-files=all` (default mode) | native — collects untracked itself | safe — HEAD-absent specialization in its rail | default mode (no `Base ref`) |

The restore-safety column reflects each callee's `git checkout HEAD -- <file>` safety rail — anchored in `skill-review` / `publicity-review` / `verify-diff` § "Per-iteration safety rails" (c) (the three `Base ref:` editors share one **unsafe** rail with no HEAD-absent handling), in `tidy`'s HEAD-absent specialization in its own rail, and in `rules-review`'s absence of a rail (detect-only). Re-check this column if a callee changes its safety rail.

The five rows above are also the **dispatch vocabulary**: a `Callees` name outside this set is an unknown-callee fatal abort (§ No-Stall Principle).

**Out-of-scope (not diff-deriving, so visualization is irrelevant — never valid `Callees` entries)**:

- `prompt-tuning` — evaluates a path-targeted prompt file; does not run `git diff`.
- `ask-peer` — reviews a caller-supplied payload; does not run `git diff`.
- `verify-bundle-sync` — a structural directory comparison (`diff -rq` of the canonical vs bundle-copy trees) that ignores the base ref; diff-independent, so visualization has no effect on it.

Runtime uses of the list: (a) validate each requested callee name; (b) pick the per-callee argument form; (c) decide whether visualization is needed — apply it once when any requested callee is `blind`; (d) read the restore-safety class — when any requested callee is **unsafe**, enable the step (f) detect-and-warn pass.

**Argument-form minimalism**: pass each callee only the single argument form above — no triage branch name, no changed-file list, no extra context. A long preamble can override the callee's own procedural logic and cause an empty-input early return; for `skill-review` the `Base ref: <ref>` line is the entire argument.

**`tidy` scope caveat**: `tidy`'s default mode diffs working-tree-vs-HEAD, so its review scope is `Base ref`-independent and equals the other callees' `<Base ref>` scope only when `Base ref == HEAD` — against a multi-commit base its findings cover only the uncommitted delta while the `Base ref:` callees cover the full base-to-tree range. This divergence is accepted because default mode is how `tidy` natively sees untracked files (the whole reason it is `safe` rather than needing visualization).

## Procedure

Single invocation, top to bottom:

(a) **Validate inputs, then resolve the base.** First validate `Callees` (no git command needed): confirm it is non-empty and every name is in the § Closed list dispatch vocabulary; on a miss, take the unknown-callee fatal abort (§ No-Stall Principle) with `base: null`. Only after the names pass, run `git rev-parse <Base ref>` to a concrete `<sha>`; on failure, take the base-ref-unresolved fatal abort (§ No-Stall Principle) with `base: null`. Both checks run before any git mutation — step (c) is the first mutation, so a fatal abort in this step never leaves an intent-to-add entry behind.

(b) **Collect untracked new files.** Run `git status --porcelain=v1 --untracked-files=all -z` and take the `??` entries. Subtract the always-excluded `.claude/plans/**` workflow-artifact paths and any `Scope` excludes; hold the remainder as `visualized_paths` — the *candidate* set to visualize. (The verdict's `visualized_paths` field reports only what step (c) actually runs `git add -N` on, so it is the empty set whenever visualization is skipped — including when no requested callee is `blind`.) Gitignored files never appear in this output, so they are excluded automatically (intended). If `visualized_paths` is empty, skip visualization — step (c) and the step (e) restore become no-ops — but the callees still run in step (d).

(c) **Visualize** — only when at least one requested callee is `blind` (per § Closed list) and `visualized_paths` is non-empty. Run `git add -N -- <visualized_paths>`. An empty file is visualized as `new file mode` with no `+` hunk; a callee sees the path with no content delta and treats it as a no-op, which is acceptable. `visualized_paths` is sourced exclusively from step (b)'s `??` entries, so `git add -N` never touches a tracked path.

(d) **Dispatch callees in order.** For each name in `Callees`, in order: emit a one-line Progress Visibility status, then dispatch `Skill(<callee>)` with its § Closed list argument form (substituting the step (a) `<sha>` for `rules-review`, the `<Base ref>` for the `Base ref:` callees). However the host services the call — inline in this same context, or as a separate agent — the callee emits a single fenced JSON verdict block; parse that block (ignore any prose preceding it) and record its `status`. An inline-serviced verdict is a normal result, not a failure. Only a tool-level dispatch error or a missing / unparseable verdict is the non-fatal failure path — record it under that callee's per-callee status (`dispatch-error` / `unparsed`) and proceed to the next callee. **Return-point reminder**: issue the next callee dispatch — or step (e) when this was the last callee — in the same turn (§ No-Stall Principle).

(e) **Restore (mandatory on every normal and callee-error exit path).** When visualization was applied in step (c), run `git reset -- <visualized_paths>` to drop the intent-to-add entries and return the files to untracked. **Use the path-scoped form only — never bare `git reset`**, which would also unstage any unrelated content the caller had staged before invocation. **`git reset -- <paths>` is unstage-only: it preserves working-tree content (including any corruption a callee left behind) so step (f) can inspect it. Reordering this step after (f), or replacing it with `git checkout` / `git restore`, would break detection — do not.**

(f) **Detect-and-warn** — only when a requested callee is **unsafe** (per § Closed list) and visualization was applied. The unsafe callees' frontmatter rail (`git checkout HEAD -- <file>`) cannot revert a HEAD-absent (visualized) file, so a frontmatter-corrupting edit can survive. `Read` each `visualized_paths` entry and collect into `corrupted_paths` any path that is now corrupted:

- **Primary detector (signal #2)**: re-parse the file's YAML frontmatter yourself (orchestrator-side, independent of callee behavior). A path whose `---`-delimited frontmatter no longer parses is corrupted. A file without a frontmatter block is never flagged.
- **Corroborating (signal #1)**: a callee verdict that reported a rail conflict (`status: "conflict"`, `reason: "frontmatter broken"`) referencing the path.

Signal #2 is **primary** because the unsafe callees emit `frontmatter broken` without checking their `git checkout` exit code — signal #1 works today but would silently go dark if a callee revision gated its verdict on checkout success. This pass **warns only**: it does not restore content, since auto-restore would also discard any legitimate in-scope fix a callee applied (see § Constraints).

(g) **Emit the § Return contract verdict** and stop.

## Constraints / boundary

- **Callees must not stash or reset during visualization.** This skill's callees run while `visualized_paths` are in intent-to-add state; a callee that ran `git stash` / `git reset` itself would disturb that state. The current callee set does not (only `triage-review` stashes, and it is a *caller* that wraps this skill, never a callee of it).
- **Non-breaking callee edits persist (intended).** When a callee edits a visualized file's frontmatter *without* breaking it — or edits its body — no rail fires and the edit survives on the file after step (e) returns it to untracked. This is intended: the skill exists to let callees review and fix new files. `corrupted_paths` surfaces only frontmatter *breakage*, not legitimate edits.
- **Hard-crash leftover window.** Step (e)'s restore is a procedural instruction, not a `try`/`finally` primitive: a hard tool-level error or session death between step (c) and step (e) can leave a `visualized_paths` entry in intent-to-add (` A`) state with a clean-looking working tree. The skill does **not** auto-reset orphan ` A` entries on a later run (that could clobber a caller's own intentional intent-to-add). Recover manually with `git reset -- <path>` if it occurs.
- **Permission-matcher fallback.** If a host's permission matcher rejects the scoped `Bash(git reset -- *)` / `Bash(git add -N *)` grants, broaden them to `Bash(git reset *)` / `Bash(git add *)` but keep the scoped invocation forms (`git reset -- <paths>` / `git add -N -- <paths>`) — the path scoping is the safety property; the grant pattern is only how the host authorizes it.

## Return contract

End the response with a single fenced JSON block:

```json
{
  "status": "ok|error",
  "base": "<sha>",
  "visualized_paths": ["<path>"],
  "corrupted_paths": ["<path>"],
  "callees": [
    {"name": "<callee>", "status": "<callee verdict status, or 'dispatch-error' / 'unparsed'>"}
  ],
  "reason": null
}
```

Field semantics:

- `status`: `error` only on a § No-Stall Principle fatal abort (with `reason` set); otherwise `ok`, including runs where individual callees errored (those surface in `callees[].status`) or `corrupted_paths` is non-empty.
- `base`: the concrete `<sha>` resolved in step (a), or JSON `null` when a fatal abort fired before base resolution (unknown callee, or base ref unresolved).
- `visualized_paths`: the exact set `git add -N` ran on (empty when visualization was skipped).
- `corrupted_paths`: a subset of `visualized_paths` flagged by step (f) (empty when step (f) did not run or found nothing).
- `callees`: one entry per dispatched callee, in dispatch order, carrying the callee's own verdict `status` verbatim (or `dispatch-error` / `unparsed` for a non-fatal dispatch / parse failure). Each callee defines its own status vocabulary (they are **not** a single unified enum across callees), so this is a pass-through value — a consumer must read it per-callee, not switch on it as one enum.
- `reason`: a short string on `status: "error"`, else JSON `null`.

## Sub-skill caller directive

When invoked via `Skill(review-with-untracked)` from an orchestrator, the fenced JSON verdict is the **structured return value** of this skill's procedure — not a deliverable to the user, and emitting it does **not** terminate the orchestrator's turn. The same agent must immediately issue the next tool call the orchestrator's flow dictates. Do not insert a prose summary or a "shall I proceed?" sentence between the JSON verdict and the next tool call. Closing the turn after emitting the JSON block — even with no prose between them — is the same violation as inserting prose. Only one fenced JSON block — the verdict — appears in the response, so callers can locate it unambiguously.

## Related

- `triage-review` — operator routine that dispatches the same review callees against a committed `main..HEAD` stack; the structural model for this skill's orchestrator shape. It auto-stashes, so it is a *caller* that could wrap this skill, never a callee.
- `verify-diff` / `skill-review` / `publicity-review` / `rules-review` / `tidy` — the review callees this skill brackets; their untracked-visibility and restore-safety classes are tabulated in § Closed list.
