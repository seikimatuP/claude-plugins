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
- 非対話／ルーチン実行を想定するスキル（`dev-workflow` / `dev-workflow-triage` 等）には `No-Stall Principle` 節（または同等の "途中停止禁止" 節）を SKILL.md 冒頭に明記する。`Skill(...)` のサブスキル復帰点・ループ境界・非致命エラー処理点で「サマリして一区切りつける」誘惑が入り込むため、(i) 許容される唯一の非完走経路（fatal-abort exits）を closed list で列挙し、(ii) サブスキル復帰時は戻り値を意味的に判定して即座に既存分岐へ進むこと、(iii) 非致命エラー（`*-failed` 系、`overflow` 系）は記録して続行する旨、を明文化する。複数スキルで同名節を使う場合は表記を一致させ、各 SKILL.md 内のクロスリファレンスはサブステップ番号ではなく安定した節見出し（例: `§ No-Stall Principle`）で行う
- Claude Code on the Web などの ephemeral 実行環境を主戦場とするルーチンスキルが作業用ファイル（`.claude/plans/<skill>-*.md` 等の staging 文書）を生成する場合、デフォルトを「成功時削除（retrospective pattern の mirror）」にしない。(i) session 終了で workspace ごと破棄されるため蓄積が起きない、(ii) ファイルを残すことで Web file viewer から in-session 確認が可能になる、(iii) 外部（GitHub issue comment 等）に canonical な永続記録が既にある、の 3 条件が揃う場合は「ファイルを残す + `.gitignore` で commit 混入だけをブロック」を優先する。SKILL.md には「gitignored なので commit されない／外部コメントが canonical record」である旨を明記し、`rm` を allowed-tools に足さない（権限最小化）
- 非対話／ルーチン実行を想定するスキルが staging ファイル（`triage-<date>-issue<N>.md` 等）を生成する際、書き込み先は `.claude/` 配下を避ける。Claude Code は `.claude/*` を「sensitive file」として扱うため、`allowed-tools` に `Write` があっても permission dialog が発生し、ルーチン（非対話）実行で事実上の停止点になる。代わりに repo root 直下の dedicated directory（例: `.triage/`）を使い、`.gitignore` で commit 混入だけをブロックする。SKILL.md には「`.claude/` 外に置くのは sensitive-path treatment 回避のため」と明記する
- 非対話／ルーチン実行を想定するスキルでは、サブスキル復帰時の No-Stall Principle 違反が観測された箇所（`Skill(verify-diff)` / `Skill(skill-review)` 等の return point）に「return-point no-stall reminder」をインライン bullet として配置する。SKILL.md 冒頭の `§ No-Stall Principle` 節と意図的に重複させる — 抽象節だけでは agent が決定の瞬間に参照しないため、決定点の bullet 直後に置くことで違反確率を下げる。reminder は `(regardless of outcome — <列挙>, any non-error result)` のように非致命結果を closed list で列挙し、次の action を「next tool call で発行」と明示し、`§ No-Stall Principle` への安定参照（節見出しベース）を含める
- 非対話／ルーチン実行を想定するスキルが GitHub issue / PR / 検索結果等の collection をループ処理する場合、per-invocation の件数 cap は subagent dispatch（`Skill(verify-diff)` / `Skill(skill-review)` / `Skill(ask-peer)` 等）のオーバーヘッドを織り込んで保守的に設定する。`gh issue list --limit 200` のような上限値は「routine 1 走行で順次処理」観点では非現実的（1 件あたり複数 subagent dispatch があるため walltime が膨らむ）。経験則として `--limit 50` 程度を初期値とし、`overflow=true`（戻り値が cap に張り付いた場合）を summary に明記する設計を採る。cap 引き下げに伴い TodoWrite 行数上限の懸念も実質的に消える
- 非対話／ルーチン実行を想定するスキルでは、`§ No-Stall Principle` に「Phase / per-X TodoWrite transitions are non-stalling」段落を明記する。TodoWrite 書き込みは in-memory state 操作で file 書き込みではないため sensitive-path treatment / permission dialog が発生しない。phase 行と per-item 行の `pending → in_progress → completed` 遷移は同一 tool-call burst 内で発行可能であり、ターン跨ぎの「summary 出力 → 次ターンで status flip」誘惑を排除する旨を SKILL.md prose で明示する
- 非対話／ルーチン実行を想定するスキルで collection ループが 0 件で skip される経路（例: `gh issue list` が 0 件 → per-issue ループ非実行 → 直接 termination phase へ）では、複数 phase 行（`Step 2 / Step 3 / Step 3.7 / Step 4` 等）を **1 回の TodoWrite call で同時遷移**（`completed` flip を一括）させる慣行を採る。phase 行ごとに別 call を発行すると stall 誘発点が増えるため、0-item 経路は「multi-row flip in single call」として SKILL.md prose に明記する
- 非対話／ルーチン実行を想定するスキルで分岐 path（例: `title-mismatch-skip` 短絡 path）が下流の dispatch ブロック（reminder / status flip 等）を経由しない構造になっている場合、上流 sub-step 末尾に **forward jump pointer**（例: `Skipping does not bypass the reminder dispatch — apply the dispatch at the end of § Close decision`）を明示的に挿入する。短絡 path に分岐したまま「skip = 何もしない」と誤解されると下流の必須 transition が抜ける
- 非対話／ルーチン実行を想定するスキルの `§ No-Stall Principle` 節で、同じ境界に対して条件で分かれる reminder（例: issue-loop boundary で「more issues remain → reminder #1」「last issue → reminder #2」）を配置する場合、**両 variant を SKILL.md 上で並列に prose 記述**し agent が runtime で applicable variant を選ぶ形を採る。dispatch 位置を分散させると agent が決定点で当該節を参照しにくくなるため、closed-list 形式の reminder を同一位置に並べる（reminder #2 にも `, any non-error result` を含めて #1/#3 と structural 整合させる）

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

## バージョン管理 / リリース運用

- bundle に含まれるスキル（`dev-workflow` / `ask-peer` / `extract-rules` / `rules-review`）の version bump は、対応するスキル plugin と `dev-workflow-bundle` plugin を **常にペアで bump** する。CHANGELOG の version subsection 見出しも `### <skill> vX.Y.Z / dev-workflow-bundle vX.Y.Z` の対形式で書く（既存 CHANGELOG の不変条件）
- bump 直前に `dev-workflow` plugin と `dev-workflow-bundle` plugin の現 version 一致を `jq -r ...` で確認する version-skew guard を入れる。不一致なら abort して per-Finding コミットは preserve する
- `marketplace.json` の version 書き換えは `jq | mv` ではなく `Edit` ツールで version 行を直接書き換える。`Bash(mv *)` を allowed-tools に追加せずに済むため、「allowed-tools は必要最小限」原則と整合する。Edit 直後に `jq empty` で構文整合性を再確認する
- `Edit` で plugin の version を書き換えるときの `old_string` には plugin name + 周辺 + version を含める塊を取り、name の閉じる double-quote と末尾 `,` まで必ず含める（例: `"name": "dev-workflow",`）。`"name": "dev-workflow"` のように trailing カンマを含めないと `"name": "dev-workflow-bundle"` の prefix と被って not-unique error になる。`replace_all` は禁止
- 自動化スクリプトやルーチンスキルでスキルを更新した場合、CHANGELOG.md / marketplace.json の version bump は **per-Finding コミットとは別の bookkeeping commit** にまとめる（`chore(release): bump <plugins> (auto-triage YYYY-MM-DD)` のような subject）。「1 accepted Finding = 1 commit」ルールを維持し、scope check の意味を保つため
- CHANGELOG.md のエントリは新しい version subsection を `## YYYY-MM-DD` 直下に **prepend** する（既存スタイル「新しい version が上」と整合）。同日複数 invocation が起きうる場合、commit subject 末尾に `(auto-triage YYYY-MM-DD)` 等のサフィックスを付けて commit log で区別できるようにする

## Examples

When in doubt: ./project.rules.examples.md
