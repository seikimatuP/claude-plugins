# Project Rules - Examples

## Principles Examples

### bundle スキルと dev-workflow-bundle のペア bump
**Good** (CHANGELOG.md):
```markdown
## 2026-04-25

### dev-workflow v1.34.2 / dev-workflow-bundle v1.34.2

- fix(dev-workflow): ...
```
**Bad:**
```markdown
## 2026-04-25

### dev-workflow v1.34.2

- fix(dev-workflow): ...
```
（`dev-workflow-bundle` の対が抜けると bundle 配布の version が静かに古いまま残る）

### `Edit` での marketplace.json version 書き換え
**Good:**
```jsonc
// old_string (name の閉じる " と trailing , まで含める)
"name": "dev-workflow",
      "source": "./plugins/dev-workflow",
      "version": "1.34.2",

// new_string
"name": "dev-workflow",
      "source": "./plugins/dev-workflow",
      "version": "1.34.3",
```
**Bad:**
```jsonc
// old_string が "name": "dev-workflow" だけ → "dev-workflow-bundle" の prefix と被って not-unique error
"name": "dev-workflow"
```
Edit 直後に `jq empty .claude-plugin/marketplace.json` で syntax 確認する。`replace_all` は禁止。

### bookkeeping commit の分離
**Good** (commit log):
```text
feat(dev-workflow-triage): release bookkeeping after accepted Findings
fix(dev-workflow): Step 7.5 semantic-judgment branch
chore(release): bump dev-workflow / dev-workflow-bundle (auto-triage 2026-04-25)
```
（per-Finding fix と version bump が別コミット）
**Bad:**
```text
fix(dev-workflow): Step 7.5 semantic-judgment branch + bump version
```
（同じコミットに混ぜると「1 accepted Finding = 1 commit」「scope check」の意味が薄れる）

### Routine スキルの per-invocation 件数 cap heuristic
**Good** (`dev-workflow-triage` SKILL.md Step 2):
```text
gh issue list --repo <owner>/<repo> --state open --limit 50
# Exactly 50 ⇒ overflow=true; report "50-issue cap reached" in Step 4 summary
```
（subagent dispatch コストを考慮して保守的に 50 に設定。cap 到達時は overflow フラグで人に追加 invocation を促す）
**Bad:**
```text
gh issue list --repo <owner>/<repo> --state open --limit 200
# Exactly 200 ⇒ overflow=true
```
（200 件を 1 routine 走行で順次 triage すると verify-diff/skill-review の subagent dispatch が積み重なって walltime が膨らむ）

### 0-item 経路の multi-row flip
**Good** (per-issue ループが 0 件で skip される SKILL.md 記述):
```markdown
If Step 2 reported 0 open issues, mark Step 2 / Step 3 / Step 3.7 / Step 4 all
`completed` in a **single TodoWrite call** and proceed directly to Completion summary.
```
（phase 行を 1 call で同時遷移させて stall 誘発点を減らす）
**Bad:**
```markdown
If 0 issues, mark Step 2 completed.
Then mark Step 3 completed.
Then mark Step 3.7 completed.
Then mark Step 4 completed.
```
（phase 行ごとに別 call にすると、call 間でターン跨ぎの停止誘惑が入る）

### Forward jump pointer for skip path
**Good** (`§ Title match` の skip path 末尾):
```markdown
The per-issue row is flipped pending → completed here per § Title-mismatch skip path.
Skipping does not bypass the reminder dispatch — apply the dispatch at the end of § Close decision.
```
（短絡 path から下流 dispatch への forward jump を明示）
**Bad:**
```markdown
On title mismatch, skip the issue and continue.
```
（「skip = 何もしない」と誤読されると下流の必須 reminder dispatch が抜ける）

### 並列 reminder dispatch（runtime variant 選択）
**Good** (`§ No-Stall Principle` issue-loop boundary):
```markdown
**Reminder #1 (more issues remain)**: Ensure the just-finished per-issue row is
completed (regardless of outcome — accepted, rejected, parse-error, title-mismatch-skip,
any non-error result), then proceed to the next issue with the next tool call.
See § No-Stall Principle.

**Reminder #2 (last issue)**: Ensure the just-finished per-issue row is completed
(regardless of outcome — accepted, rejected, parse-error, title-mismatch-skip,
any non-error result), then proceed to Step 3.7 with the next tool call.
See § No-Stall Principle.
```
（両 variant を並列に prose 記述、closed-list 形式で structural 整合）
**Bad:**
```markdown
After the last issue is processed, jump to Step 3.7.
(For non-last issues, reminder #1 in another section applies.)
```
（dispatch 位置が分散すると agent が決定点で参照しにくくなる）
