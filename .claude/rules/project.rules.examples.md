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
