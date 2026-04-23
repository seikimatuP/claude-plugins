# Project Rules

## SKILL.md設計

- シンプルな箇条書き形式を優先
- エビデンス・信頼度等の詳細情報は不要（コンテキスト削減のため）
- 一般知識で判断可能なことはルールに含めない
- AIに判断を委ねる方針（細かく書きすぎない）
- SKILL.md 内外の相互参照は、サブステップ番号（例: `Step 2.7`）ではなく安定した節見出し・フレーズ（例: `Step 2 difficulty assessment`）で記述する。サブステップ追加/挿入時にリンク切れが多発するため

## allowed-tools設計

- `Bash(*)`は避け、具体的なコマンドを指定（例: `Bash(git ls-files *)`）
- 検索/参照系のBashコマンドは許可してよい
- 必要な権限のみ追加（mkdir -p等）

## セキュリティ

- `.gitignore`に設定されているファイル/ディレクトリはデフォルトで除外
- `.env`等の機密情報は抽出対象外
- `git ls-files`を使用して追跡ファイルのみを対象にする

## ワークフロー

- peerに相談する（作業計画レビュー、完了時チェック）
- codex等の外部ツールにレビューを依頼

## ドキュメント言語

- プラン文書（`.claude/plans/*.md` 等）は日本語で記述する
- 実装物（SKILL.md、README.md、CHANGELOG.md、plugin.json の description 等、配布される成果物）は英語で記述する
- ユーザーとの会話は日本語で行う（グローバル設定）

## SKILL.md の配布性

- このリポジトリは配布用マーケットプレイスのため、SKILL.md にユーザー固有の情報（特定リポジトリ名 `owner/repo`、絶対パス、個人識別子等）をハードコードしない
- リポジトリ情報やパスが必要な機能は設定ファイル（`.claude/<skill>.local.md` 等の frontmatter）経由で受け取る

## ローカルスキル設計

- ルーチン用途（Claude Code on the Web の定期実行など非対話環境）を想定するローカルスキル（`.claude/skills/<name>/`）は、外部プラグインスキル（`Skill(document-skills:*)` 等）への依存を避ける。Routine 環境に当該プラグインが install されていないと無条件失敗するため、参照したいベストプラクティスは `skills/<name>/references/` 配下に要旨を抽出して自己完結させる
- スキル自身を修正する種類のルーチン（triage 等）では、1 改善 = 1 commit の粒度で落とす。複数 Finding が同一ファイルに当たる場合も、Finding ごとに対象ファイルを直前に再 Read → Edit を組み直し → commit を繰り返す。事前に「2 件目以降は conflict」と落とす過剰防衛はしない（日次ルーチンで issue が滞留する）

## プラグイン構造

- 単体スキルプラグインは direct-skill 方式（`source: "./skills/<skill-dir>"` + `skills: ["./"]`）を使う。`plugins/` 配下のラッパーディレクトリを作らない
- direct-skill 方式ではプラグイン名と skill ディレクトリ名が異なっても OK（例: plugin `peer` → skill `ask-peer`）
- wrapper 方式（`plugins/<name>/`）は以下のいずれかに該当する場合のみ: (1) `agents/` を持つエージェント依存プラグイン、(2) `plugin.json` にフック定義を持つプラグイン、(3) 複数スキル bundle
- wrapper には 2 サブパターン: (A) エージェント/フック wrapper（`plugin.json` 必須）、(B) bundle wrapper（`plugin.json` 不要、marketplace.json 側の `skills` 配列で参照スキルを明示）
- bundle では marketplace.json の `skills` 配列と `plugins/<bundle>/skills/` 配下の symlink セットを必ず一致させる。ずれると配布が壊れるため、`/verify-plugins` と `run-tests` で整合性を検証すること
- フックの自動設定が必要な場合はプラグイン化
- PreCompactだけでなくStopフックも検討（Compactが発生しない場合に対応）
- 設定が複雑なスキルには README.md を用意する。`skills/<name>/README.md` に置けば direct-skill 方式で source 直下に配置されるため、利用者に自動的に届く
- プラグイン構造を変更する場合、`.claude-plugin/marketplace.json` だけでなく検証ツール（`.claude/skills/run-tests/SKILL.md`、`.claude/commands/verify-plugins.md`）とドキュメント（`CLAUDE.md`）の該当箇所もセットで更新する。片方だけ更新すると見落としが発生する
