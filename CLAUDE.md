# Claude Code Plugins Marketplace

Claude Code用プラグインを公開するためのマーケットプレイスリポジトリです。

## リポジトリ構成

```
.
├── .claude-plugin/
│   └── marketplace.json      # プラグインマニフェスト
├── skills/                   # SKILL.mdの実体（正規ファイル）
│   └── <skill-name>/
│       └── SKILL.md
├── plugins/                  # 全プラグインのsourceディレクトリ
│   └── <plugin-name>/
│       ├── skills/
│       │   └── <skill-name>  # → ../../../skills/<skill-name> (シンボリックリンク)
│       ├── .claude-plugin/   # (エージェント依存プラグインのみ)
│       │   └── plugin.json
│       ├── agents/           # (エージェント依存プラグインのみ)
│       │   └── <agent-name>.md
│       └── README.md         # (エージェント依存プラグインのみ)
├── .claude/
│   ├── commands/             # 開発用コマンド
│   ├── skills/               # 開発・テスト用シンボリックリンク
│   └── agents/               # エージェントへのシンボリックリンク
├── CHANGELOG.md              # 変更履歴
└── README.md                 # リポジトリ説明
```

**注意:** 各プラグインは `plugins/` 配下に専用のsourceディレクトリを持つ必要がある。`source: "./"` を使うと、`skills/` 配下の全スキルが自動発見されて重複登録される。

## スキル追加フロー（エージェント非依存）

スキルが `allowed-tools` を持ち、エージェントに依存しない場合:

### 1. skills/ディレクトリにスキル作成

```
skills/<skill-name>/
└── SKILL.md
```

### 2. SKILL.md

```markdown
---
name: <skill-name>
description: スキルの説明
allowed-tools: Read, Glob, Grep
---

# スキル名

スキルの詳細な説明と使い方
```

### 3. プラグインのsourceディレクトリ作成

```bash
mkdir -p plugins/<plugin-name>/skills
ln -s ../../../skills/<skill-name> plugins/<plugin-name>/skills/<skill-name>
```

### 4. marketplace.json に追加

`.claude-plugin/marketplace.json` の `plugins` 配列に追加:

```json
{
  "name": "<plugin-name>",
  "source": "./plugins/<plugin-name>",
  "description": "スキルの説明",
  "version": "1.0.0",
  "author": { "name": "hiropon" },
  "category": "workflow"
}
```

### 5. 開発・テスト用シンボリックリンク作成

```bash
ln -s ../../skills/<skill-name> .claude/skills/<skill-name>
```

### 6. CHANGELOG.md 更新

## プラグイン追加フロー（エージェント依存）

エージェントを必要とするプラグインの場合:

### 1. skills/ディレクトリにスキル作成＋プラグインディレクトリ作成

```
skills/<skill-name>/
└── SKILL.md

plugins/<plugin-name>/
├── .claude-plugin/
│   └── plugin.json
├── skills/
│   └── <skill-name>  # → ../../../skills/<skill-name> (シンボリックリンク)
├── agents/
│   └── <agent-name>.md
└── README.md
```

```bash
# スキル実体は skills/ に配置
mkdir -p skills/<skill-name>
# プラグインディレクトリからシンボリックリンク
mkdir -p plugins/<plugin-name>/skills
ln -s ../../../skills/<skill-name> plugins/<plugin-name>/skills/<skill-name>
```

### 2. plugin.json

```json
{
  "name": "<plugin-name>",
  "version": "1.0.0",
  "description": "プラグインの説明",
  "author": {
    "name": "hiroro-work",
    "url": "https://github.com/hiroro-work"
  },
  "homepage": "https://github.com/hiroro-work/claude-plugins",
  "repository": "https://github.com/hiroro-work/claude-plugins",
  "license": "MIT",
  "keywords": ["keyword1", "keyword2"]
}
```

### 3. marketplace.json に追加

`.claude-plugin/marketplace.json` の `plugins` 配列に追加:

```json
{
  "name": "<plugin-name>",
  "source": "./plugins/<plugin-name>",
  "description": "プラグインの説明",
  "version": "1.0.0",
  "author": { "name": "hiropon" },
  "category": "workflow"
}
```

### 4. 開発・テスト用シンボリックリンク作成

```bash
# スキル
ln -s ../../skills/<skill-name> .claude/skills/<skill-name>

# エージェント
ln -s ../../plugins/<plugin-name>/agents/<agent-name>.md .claude/agents/<agent-name>.md
```

### 5. CHANGELOG.md 更新

## 検証コマンド

```bash
/verify-plugins        # 構造・バージョン・動作テスト
/verify-plugins --full # 完全検証（CLI更新確認を含む）
/test-skills           # スキル・エージェント動作テスト
```

## コーディング規約

### 命名規則

- スキル名: kebab-case（例: `security-scanner`, `ask-claude`）
- プラグイン名: kebab-case（例: `peer`, `translate`）
- エージェント名: kebab-case（例: `peer`, `tr`）

### allowed-tools

- 必要最小限の権限のみ付与
- `Bash(*)` は避け、具体的なコマンドを指定（例: `Bash(git *)`, `Bash(jq *)`）
- セキュリティスキャンで警告される可能性のあるパターンは正当な理由がある場合のみ使用

### バージョン管理

- セマンティックバージョニング（SemVer）を使用
- `marketplace.json` と `plugin.json` のバージョンは常に一致させる

### ドキュメント

- README.md: ユーザー向けのドキュメント（使い方、機能、設定など）
- SKILL.md: Claude向けの指示（処理フロー、出力形式など）

## セキュリティ

プラグイン追加時は `/security-scanner --project` でセキュリティスキャンを実行し、問題がないことを確認してください。
