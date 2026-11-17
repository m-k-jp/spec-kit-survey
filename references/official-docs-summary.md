# 公式ドキュメント要約

## 調査メタ情報

| 項目 | 内容 |
| --- | --- |
| 調査日 | 2026-09-07 |
| 一次情報源 | GitHub 公式リポジトリ `github/spec-kit`（`docs/` 配下が公式サイトのソース） |
| 公式サイト | <https://github.github.io/spec-kit/> ※`.io` が正。`.com` は誤記 |
| 参照コミット | `4a7341a`（2026-09-04） |
| CLI バージョン | `pyproject.toml` = `1.0.5.dev0`（直近リリースは 1.0.4 / 2026-09-02） |
| ドキュメント最終更新 | ランディングページ表記「Last updated: August 21, 2026」 |
| Python 要件 | `>=3.11` |

> 本要約はすべて公式リポジトリ内のドキュメント・テンプレート・ソースコードを直接読解して作成しています。二次情報は使用していません。

## ドキュメント読破チェックリスト

- [x] Getting Started（`docs/index.md`, `docs/quickstart.md`, `docs/installation.md`, `docs/install/*.md`）
- [x] Concepts（`docs/concepts/sdd.md`, `spec-persistence.md`, `complex-features.md`, `spec-of-specs.md`）
- [x] Workflows（`docs/reference/workflows.md`, `workflows/speckit/workflow.yml`, `workflows/catalog*.json`）
- [x] CLI Reference（`docs/reference/overview.md`, `core.md`, `authentication.md`, `docs/upgrade.md`）
- [x] AI Agent Integration（`docs/reference/integrations.md`, `src/specify_cli/integrations/`）
- [x] Extensibility（`docs/reference/extensions.md`, `presets.md`, `bundles.md`, `extensions/EXTENSION-API-REFERENCE.md`）
- [x] Guides（`docs/guides/existing-projects.md`, `evolving-specs.md`, `monorepo.md`）
- [x] Community（`docs/community/*.md`, 各 `catalog.community.json`）
- [x] GitHub spec-kit README・CONTRIBUTING・AGENTS.md・spec-driven.md
- [x] GitHub spec-kit CHANGELOG（220 リリースを最新版から遡行）→ `references/changelog-summary.md`
- [x] History（`docs/history.md`：プロジェクト沿革・ステュワードシップ）

## 全体像

Spec Kit は「仕様駆動開発（SDD）ツール」から「**任意のコーディングエージェントを SDLC 全体に拡張する、意図駆動のハーネス**」へと定義が拡大しています。公式ランディングページの記述は次のとおりです。

> Spec Kit is an extensible, intent-driven harness that pushes any coding agent beyond code, guiding it across your SDLC or any business process.

すなわち SDD は「既定で同梱される 1 つのプロセス」であり、プロセス自体を差し替え可能な構成要素として扱う点が現行版の中核思想です。ソフトウェア開発以外（小説執筆プリセット等）も想定範囲に含まれます。

公式が掲げる 5 つのプリミティブ（`docs/history.md`）:

| プリミティブ | 役割 |
| --- | --- |
| Integrations | コーディングエージェントとの接続 |
| Extensions | コマンド・テンプレート・スクリプト・フックの追加 |
| Presets | 既存挙動のカスタマイズ・置換 |
| Workflows | 複数ステップの自動化 |
| Workflow steps | ワークフローの再利用可能な単位 |
| （Bundles） | 上記 4 種を役割単位で束ねた配布物 |

## セクション別要約

### Getting Started

インストールは PyPI 公開パッケージ `specify-cli` を `uv tool install` するのが推奨です。`pipx`、`uvx` 単発実行、ソース導入、エアギャップ環境向けホイール配布もサポートされます。

```bash
uv tool install specify-cli
specify init my-project --integration copilot   # または specify init .
```

推奨プロセスは 2 系統が明示されています。

- **短経路（小規模機能）**: `specify → plan → tasks → implement → converge`
- **完全経路（本番機能）**: `constitution → specify → clarify → plan → checklist → tasks → analyze → implement → converge`

重要な仕様として、**アクティブな機能は Git ブランチではなく `.specify/feature.json` の `feature_directory` で決まります**。環境変数 `SPECIFY_FEATURE_DIRECTORY` で上書き可能です。`git checkout` だけでは対象機能は切り替わりません。Git は必須ではありません。

### Concepts

#### SDD の哲学（`concepts/sdd.md`）

- 意図駆動（"what" を "how" より先に定義）
- ガードレールと組織原則に基づく仕様作成
- ワンショット生成ではなく多段階の精緻化
- 開発フェーズは 3 種：0-to-1（Greenfield）／Creative Exploration（並行実装比較）／Iterative Enhancement（Brownfield）

#### 仕様の永続化モデル（`concepts/spec-persistence.md`）

Spec Kit は成果物の維持方針を**強制しません**。チーム規約として明示的に選ぶことを求めています。

| モデル | 変更ルール | 適合ケース | 注意点 |
| --- | --- | --- | --- |
| Flow-back Spec | 任意の成果物を編集し、後で整合させる | 小規模・高速反復 | 静かな乖離（silent drift） |
| Flow-forward Spec | 要件変更時は新しい feature ディレクトリを作る（既存は不変） | 監査証跡・履歴重視 | 文脈の重複・分散 |
| Living Spec | `spec.md` を先に更新し、下流を再生成 | 仕様を契約として扱う | 設計根拠の消失 |

Martin Fowler 系の分類（spec-first / spec-anchored / spec-as-source）は「仕様がどれだけ長く意味を持つか」という時間軸の問いであり、上記 3 モデルは「要件変更時に成果物集合をどう変異させるか」という別問題である、と整理されています。

#### 大規模機能の扱い（`concepts/complex-features.md`, `spec-of-specs.md`）

実装フェーズの破綻原因はコンテキストウィンドウの枯渇であると明言されています。対処は次の順で軽い順に試すよう指示されています。

1. 実行タスク数・フェーズを限定する（`/speckit.implement only execute tasks T001-T010`）
2. サブエージェントへ `[P]` タスクを委譲する
3. 1 と 2 の併用
4. 「Spec of Specs」= ロードマップ表で機能を独立スライスへ分解（最もオーバーヘッドが大きい最終手段）

Spec of Specs はツール機能ではなく **Markdown の表による規約**です（`specs/<epic>/roadmap.md` に ID・意図・スコープ境界・依存・ステータス・サブ仕様パスを持つ表を置き、双方向リンクを張る）。

### Workflows（`reference/workflows.md`）

`specify workflow run <source>` で YAML 定義のワークフローを実行します。同梱の `speckit` ワークフローは specify → gate → plan → gate → tasks → implement という構成です。

ステップ種別は 12 種：`command` / `prompt` / `shell` / `init` / `slot` / `gate` / `if` / `switch` / `while` / `do-while` / `fan-out` / `fan-in`。

CD 観点で重要な性質:

- 実行状態は `.specify/workflows/runs/<run_id>/` に `state.json` / `inputs.json` / `log.jsonl` として永続化され、`specify workflow resume <run_id>` で**中断地点から再開**できます。
- `--json` で機械可読出力が得られ、`status` は `created|running|completed|paused|failed|aborted`。
- `gate` ステップは人間承認の一時停止点です。`verdict_input` で承認結果を入力値にバインドでき、CI から `--input spec_verdict=approve` の形で無人承認も可能です。
- **セキュリティ上の明示的な警告**があります。`shell` ステップはサンドボックスなしでユーザー権限のまま実行され、`{{ }}` 展開は単純な文字列置換のためシェルインジェクションの余地があります。`requires` は助言的な事前条件であり実行時ゲートではありません。エージェント出力を `run` に埋め込まないこと、入力は `enum` で制約することが推奨されています。
- **Overlays**：インストール済みワークフローの `workflow.yml` を編集せずにプロジェクト側で拡張・上書きでき、更新時にローカル改変が失われません。

### CLI Reference（`reference/core.md` ほか）

主要コマンド体系:

| 系統 | コマンド |
| --- | --- |
| コア | `specify init` / `check` / `version` / `status` / `doctor` / `self check` / `self upgrade` |
| 統合 | `specify integration list|search|info|install|uninstall|use|switch|upgrade|status|catalog` |
| 拡張 | `specify extension add|remove|list|info|update|enable|disable|set-priority|catalog` |
| プリセット | `specify preset add|remove|list|info|resolve|enable|disable|set-priority|catalog` |
| ワークフロー | `specify workflow run|resume|status|list|add|update|enable|disable|remove|search|info|validate|catalog|overlay` |
| バンドル | `specify bundle search|info|install|update|remove|list|init|validate|build|publish|catalog` |

`specify init` の主なオプション: `--integration` / `--integration-options` / `--script sh|ps|py` / `--here` / `--force` / `--ignore-agent-tools` / `--preset` / `--extension` / `--non-interactive`。

環境変数（重要な 2 軸）:

- `SPECIFY_INIT_DIR` — **プロジェクト**（`.specify/` を含むディレクトリ）を選択。モノレポで `cd` せずに対象を指定する用途。存在しない／`.specify/` を持たないパスはエラーで停止し、カレントへフォールバックしません。
- `SPECIFY_FEATURE_DIRECTORY` — 選択されたプロジェクト内の**機能**を選択（`.specify/feature.json` より優先）。
- `SPECIFY_FEATURE` — 非 Git リポジトリでの機能検出上書き。
- `SPECKIT_INTEGRATION_DEFAULT` — `--integration` 省略時の既定統合。

認証（`reference/authentication.md`）はオプトインです。`~/.specify/auth.json` を作成しない限り、カタログ取得・拡張ダウンロード・リリース確認のすべてが無認証で行われます。プロバイダは `github` と `azure-devops`、方式は `bearer` / `basic-pat` / `azure-ad` 等。トークンは `token_env` による環境変数参照が推奨され、ファイルは `chmod 600` を求めています。

### AI Agent Integration（`reference/integrations.md`）

同梱カタログには **41 統合**が登録されています（ドキュメント本文の表記は 38、2026-08-21 時点）。Copilot、Claude、Gemini、Codex、Cursor、Zed、Kilo Code、Kiro、Forge、Devin、Droid、Qwen、Kimi、Trae、Junie、Goose、Amp、opencode、Cline、Firebender、Docker Agent、DeepSeek Harness などが含まれ、該当がない場合の逃げ道として `generic` 統合が用意されています。

- **プロジェクト内でアクティブな統合は 1 つ**が原則です。`specify integration use|switch <key>` でいつでも切り替えられます。
- 提供形式は 2 系統あります。従来の Markdown スラッシュコマンド（`/speckit.specify`）と、エージェントネイティブな **skills** 形式（`speckit-specify`、`$speckit-*`、`/skill:speckit-*`）。Copilot は 0.16.0 で skills が既定になりました。
- エージェント CLI がインストールされていなくても統合の導入自体は可能です（`--ignore-agent-tools`）。

### Extensibility（extensions / presets / bundles）

**解決スタック（優先度：高 → 低）**

1. プロジェクトローカル上書き `.specify/templates/overrides/`
2. インストール済みプリセット（priority 昇順）
3. インストール済み拡張（priority 昇順）
4. Spec Kit コア `.specify/templates/`

テンプレートとスクリプトは**実行時**に解決され、コマンドファイルは**インストール時**にアクティブ統合のディレクトリへ書き出されます。合成戦略は `replace`（既定）／`prepend`／`append`／`wrap`（`{CORE_TEMPLATE}` または `$CORE_SCRIPT` を置換）。

**フックイベント**（コア定義、`extensions/EXTENSION-API-REFERENCE.md`）

`before_/after_` × `constitution` / `specify` / `clarify` / `plan` / `tasks` / `checklist` / `analyze` / `implement` / `taskstoissues` の 18 種。設定は `.specify/extensions.yml` に保持され、`optional: true` なら実行確認プロンプト、`false` なら `EXECUTE_COMMAND` 付きの自動実行として提示されます。

**カタログの信頼モデル**：カタログ登録は「発見可能になる」ことのみを意味し、コードの監査・推奨を意味しません。ソースごとに `install-allowed` と `discovery-only` の 2 種のインストールポリシーがあり、企業は自前カタログを立てて配布対象を統制できます。拡張の配布物は sha256 検証されます（0.11.7 以降）。

### 既存プロジェクトへの適用（`guides/existing-projects.md`）

公式の推奨手順は次の 5 段階です。

1. レビュー可能なベースラインを作る（コミットまたは stash → 導入用ブランチ作成）
2. `specify init --here --force --integration <key>` を実行し、**差分をレビュー**する
3. `/speckit.constitution` で「すでに真である規約」を記述する（テンプレートを埋めるために規約を捏造しないこと）
4. 境界の明確な最初の変更を 1 つ選ぶ（「既存システム全体の文書化」を最初の機能にしない）
5. 仕様の経年管理モデル（前掲 3 モデル）をチームで決める

公式は「既存システムを仕様から作り直す必要はない」と明言しています。`spec.md` は既存挙動の遡及仕様ではなく、これから行う変更を定義するものです。

### モノレポ（`guides/monorepo.md`）

Spec Kit プロジェクトは**ディレクトリスコープ**です（`.specify/` を含むディレクトリが単位）。ルート解決は Git トップレベルより**最も近い `.specify/`** を優先します。メンバープロジェクトごとに constitution と機能番号が独立します。ただし Git 操作は共有ワークツリーで行われるため、ブランチ名前空間はモノレポ全体で共有されます。constitution の継承機構は提供されないため、共有規約は自前で同期する必要があります。

### エンタープライズ／エアギャップ（`install/air-gapped.md`）

接続環境でホイールと依存を `pip download` し、`dist/` を搬入して `pip install --no-index --find-links=./dist specify-cli` で導入します。同一 OS・同一 Python バージョンでのビルドが必要です。`specify init` はバンドル済みアセットを使うためネットワーク不要です（0.4.0 でコアパックが wheel に埋め込まれました）。Windows のオフライン scaffolding には PowerShell 7+（`pwsh`）が必要です。

## 一次情報として押さえるべき要点

1. アクティブ機能の決定は `.specify/feature.json`。Git ブランチではない（0.10.0 以降、git は拡張としてオプトイン）。
2. `/speckit.taskstoissues` はコア同梱コマンドで、GitHub MCP サーバー経由の**片方向（Task → Issue）**生成。逆同期は無い。
3. `/speckit.analyze` は読み取り専用、`/speckit.converge` は追記専用（`tasks.md` への追記のみ）という書き込み権限の分離が明示されている。
4. カスタムチェックリストは**レビュアー所有**であり、`[x]` は「要件品質基準を満たした」ことを意味し、実装完了を意味しない。`/speckit.implement` はマーカーを変更してはならない。
5. ワークフローの `shell` ステップにサンドボックスは無い。CI へ持ち込む際は入力を `enum` で制約する。
6. コミュニティカタログは監査済みを意味しない。
