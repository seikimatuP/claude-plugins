---
description: Verify all plugins structure, versions, and execute skill/agent tests
argument-hint: [--full]
allowed-tools: Read, Glob, Grep, Bash(jq *), Bash(for *), Bash(echo *), Bash(if *), Bash(head *), Bash(do *), Bash(ls *), Bash(readlink *), Bash(test *), Skill(test-skills), Skill(check-cli-updates)
---

# Verify Plugins

このリポジトリの全プラグイン/スキルの構造、バージョン整合性、および動作を検証します。

## 使い方

```text
/verify-plugins        # 基本検証（構造・バージョン・動作テスト）
/verify-plugins --full # 完全検証（CLI更新確認を含む）
```

## 対象ファイル

- `.claude-plugin/marketplace.json` - プラグインマニフェスト
- `skills/*/` - スキル実体ディレクトリ
- `plugins/*/` - プラグインsourceディレクトリ

## 検証項目

### 1. スキル実体の存在確認（skills/配下）

`skills/*/SKILL.md` が存在することを確認。

### 2. プラグインsourceディレクトリの構造確認（plugins/配下）

marketplace.json の全プラグインについて:
- `source` で指定されたディレクトリが存在すること
- `plugins/<name>/skills/` 配下にスキルへのシンボリックリンクが存在すること
- シンボリックリンクの参照先（`skills/` 配下の実体）が存在すること

### 3. バージョン整合性

marketplace.json の全プラグインについて:
- `plugin.json` が存在するプラグイン: marketplace.json と plugin.json のバージョンが一致すること
- `plugin.json` が存在しないプラグイン: marketplace.json のバージョンのみ確認

### 4. 構文検証

- 各 `plugin.json` が有効なJSONであること
- 各 `SKILL.md` にYAMLフロントマターが存在すること
- 各エージェント定義ファイルにYAMLフロントマターが存在すること

### 5. スキル・エージェント動作確認

各プラグイン/スキルの機能が正常に動作することを確認。

## 作業手順

### Step 1: marketplace.json の読み込み

`.claude-plugin/marketplace.json` を読み込み、登録されている全プラグインをリストアップしてください。

### Step 2: スキル実体の存在確認

`skills/*/SKILL.md` を Glob で列挙し、marketplace.json に登録された全スキルの実体が存在することを確認してください。

### Step 3: プラグインsourceディレクトリの構造確認

marketplace.json の各プラグインについて:
1. `source` ディレクトリが存在すること
2. `plugins/<name>/skills/` 配下のエントリがシンボリックリンクであること
3. シンボリックリンクの参照先が存在し、SKILL.md を含むこと

```bash
# シンボリックリンク確認例
ls -la plugins/<name>/skills/
readlink plugins/<name>/skills/<skill-name>
test -f plugins/<name>/skills/<skill-name>/SKILL.md
```

### Step 4: バージョン整合性チェック

marketplace.json の全プラグインについて:
1. marketplace.json のバージョンを取得
2. `plugins/<name>/.claude-plugin/plugin.json` が存在する場合、そのバージョンを取得し一致を確認

不一致がある場合は警告として記録してください。

### Step 5: JSON構文検証

各 `plugin.json` について `jq` で構文チェック：

```bash
jq . plugins/<plugin>/.claude-plugin/plugin.json > /dev/null
```

### Step 6: フロントマター存在確認

各 `SKILL.md` と エージェントファイルの先頭が `---` で始まることを確認してください。

### Step 7: スキル・エージェント動作テスト

**Skillツールを使って `test-skills` スキルを呼び出してください。**

```text
Skill(skill: "test-skills")
```

このスキルでは以下がテストされます：

- 各スキル動作（/ask-claude, /ask-codex, /ask-gemini, /ask-copilot, /ask-peer, /tr, /caffeinate, /security-scanner）
- 各エージェント動作（tr, tr-hq）
- 外部CLI依存のスキルは、CLIがインストールされていない場合スキップ

### Step 8: CLI更新確認（`--full` 指定時のみ）

**`--full` オプションが指定された場合のみ**、このステップを実行してください。
指定されていない場合は、このステップをスキップして Step 9 に進んでください。

```text
Skill(skill: "check-cli-updates")
```

このスキルでは以下が確認されます：

- 各CLIの最新バージョンとインストール済みバージョンの比較
- SKILL.mdに記載されているオプションの有効性
- 非推奨オプションや新機能の確認

### Step 9: 結果サマリー

以下の形式で結果を報告してください：

```
## 検証結果

### スキル実体
| スキル | SKILL.md | 状態 |
|--------|----------|------|
| ask-claude | ✅ | ✅ |
| ... | ... | ... |

### プラグインsourceディレクトリ
| プラグイン | source存在 | skills/シンボリックリンク | 参照先存在 | 状態 |
|-----------|-----------|----------------------|----------|------|
| ask-claude | ✅ | ✅ | ✅ | ✅ |
| ... | ... | ... | ... | ... |

### バージョン整合性
| プラグイン | marketplace | plugin.json | 状態 |
|-----------|-------------|-------------|------|
| translate | 1.1.1 | 1.1.1 | ✅ |
| caffeinate | 1.0.0 | 1.0.0 | ✅ |
| ask-claude | 1.1.3 | N/A | ✅ |
| ... | ... | ... | ... |

### 構文検証
| 対象 | JSON | フロントマター | 状態 |
|------|------|---------------|------|
| translate | ✅ | ✅ | ✅ |
| caffeinate | ✅ | ✅ | ✅ |
| ask-claude | N/A | ✅ | ✅ |
| ... | ... | ... | ... |

### 動作テスト
| 対象 | スキル/エージェント | 結果 | 備考 |
|------|-------------------|------|------|
| ask-claude | /ask-claude | ✅/⚠️/N/A | 正常動作/エラー内容/CLI未インストール |
| ... | ... | ... | ... |

### 総合結果
✅ 全スキル/プラグインが正常です / ⚠️ N件の問題が見つかりました

問題がある場合は詳細を記載：
- [対象名]: 問題の内容と推奨対応
```
