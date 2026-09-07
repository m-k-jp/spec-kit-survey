# 03 - 開発ライフサイクル・継続的デリバリー・ホットフィックス

## 調査対象

- Spec Kit が対応する開発ライフサイクル（要件 → 実装 → デプロイ）
- 継続的デリバリー（CD）パイプラインとの統合
- ホットフィックスフロー
- ブランチ戦略との相性

**参照**: `extensions/git/`、`extensions/bug/`、`docs/reference/workflows.md`、`workflows/speckit/workflow.yml`、`.github/workflows/*.md`、`extensions/catalog.community.json`

---

## 結論（先に要約）

Spec Kit がカバーするのは **要件 → 設計 → タスク分解 → 実装 → 収束確認** までです。**デプロイと運用はコアの範囲外**です。

| SDLC 段階 | コア | 同梱オプトイン拡張 | コミュニティ |
| --- | --- | --- | --- |
| アイデア評価 | — | `assess`（intake→research→define→shape→decide） | 多数 |
| 要件・設計・実装 | **SDD 9 コマンド** | — | プリセットで置換可能 |
| バグ対応 | — | `bug`（assess→fix→test） | `bugfix-workflow` ほか |
| ブランチ運用 | — | `git`（機能ブランチ・自動コミット） | `branch-convention`, `multi-repo-branching` |
| CI ゲート | — | — | `ci-guard`, `gates`, `maqa-ci`, `architecture-guard` |
| リリース・デプロイ | — | — | `ship`, `spec2cloud`, `changelog`, `verify-review-ship` |
| ワークフロー自動化 | `specify workflow`（エンジン本体） | — | `pipeline`, `yolo` |

---

## 調査ノート

### 開発ライフサイクルの標準フロー

公式が示す 3 つの第一級プロセスは次のとおりです（`docs/reference/overview.md`）。

1. **Agentic SDD**（コア）— `constitution → specify → clarify → plan → checklist → tasks → analyze → implement → converge`
2. **Agentic Bug Fix**（同梱 `bug` 拡張、要 `specify extension add bug`）— `bug.assess → bug.fix → bug.test`
3. **Idea Assessment**（同梱 `assess` 拡張）— intake → research → define → shape → decide

`/speckit.converge` の存在が収束の定義を与えています。実装後にコードを spec/plan/tasks と突き合わせ、

- **Converged** — `tasks.md` はバイト単位で不変。レビューまたは PR へ進む
- **Tasks appended** — 差分を Convergence セクションへ追記。`implement` → `converge` を繰り返す

という 2 値の判定を返します。「毎回のパスで検出件数が減り、converged と出るまで繰り返す」ことが明記されており、**完了条件が客観的に定義されている**点が CD との接続で重要です。

### 継続的デリバリーへの対応

#### コアが提供する接続点

Spec Kit 本体に CD 機能はありませんが、**ワークフローエンジンが CI から呼べる形で設計されています**。

| 機能 | CD 上の意味 |
| --- | --- |
| `specify workflow run <src> --json` | 実行結果を単一 JSON で stdout に出力。進捗表示は stderr へ分離されるため機械可読 |
| `status`: `created/running/completed/paused/failed/aborted` | パイプラインの分岐条件に使える |
| 状態の永続化（`.specify/workflows/runs/<run_id>/state.json` ほか） | ジョブ間で中断・再開できる |
| `specify workflow resume <run_id> --input k=v` | 承認待ちで止めたランを別ジョブから再開できる |
| `gate` ステップ + `verdict_input` | 人間承認を入力値へバインド。`--input spec_verdict=approve` で無人承認も可能 |
| `shell` ステップ（`output_format: json` で stdout を構造化） | ビルド・テスト・デプロイコマンドの組み込み |
| `if` / `switch` / `while` / `fan-out` / `fan-in` | 条件分岐と並列化 |
| `SPECIFY_INIT_DIR` / `SPECIFY_FEATURE_DIRECTORY` | `cd` せずに対象プロジェクト・機能を指定（CI 向けに明示的に用意された経路） |
| `specify init --non-interactive` | TTY 無し環境で対話プロンプトを回避 |
| `specify version --features --json` | インストール済み CLI の機能をスクリプトから判定 |

典型的なパイプライン構成は次のようになります。

```yaml
# 概念図（workflow.yml）
steps:
  - id: implement
    command: speckit.implement
  - id: test
    type: shell
    run: "pytest -q"
  - id: gate-deploy
    type: gate
    message: "Deploy to staging?"
    options: [approve, reject]
    verdict_input: deploy_verdict     # CI からは --input deploy_verdict=approve
    on_reject: abort
  - id: deploy
    type: shell
    run: "./deploy.sh {{ inputs.target }}"   # inputs.target は enum で制約すること
```

#### セキュリティ上の制約（重要）

公式ドキュメントが明示的に警告しています。

- `shell` ステップは**サンドボックスなし**でユーザー権限のまま実行されます。
- `{{ }}` は**単純な文字列置換**であり、クォートもエスケープも行われません。`run` に埋め込まれた値はシェル構文として解釈されます。
- `requires` は助言的な事前条件ブロックであり実行時ゲートではありません。`requires.permissions` は「存在しないサンドボックスを示唆するため」バリデーションで拒否されます。
- **エージェント出力（`prompt` ステップの stdout）は信頼できない入力として扱うこと**。ファイル・チケット・Web の内容に影響され得ます。
- 対策の第一は「入力を `enum` で制約する」ことです。クォートはセキュリティ境界ではないと明記されています。
- `gate` は次のステップを検査しません。`message` はそのまま出力されるため、信頼できないデータを埋め込むと端末エスケープによる表示改ざんの余地があります。信頼できない内容は `show_file`（制御文字・ANSI が除去される）で提示します。

**CD へ持ち込む際の実務指針**: デプロイ先やターゲット名は必ず `enum` 化し、エージェント生成テキストは `run` に流さない。ここは自組織のルールとして明文化すべき箇所です。

#### CI ゲート系のコミュニティ拡張

| 拡張 | 内容 |
| --- | --- |
| `ci-guard` (v1.0.0, read-only) | CI/CD 向けの仕様コンプライアンスゲート。仕様の存在確認、ドリフト検査、ギャップがあればマージをブロック |
| `gates` (v0.3.3) | 1 つのポリシーファイルで、エージェントフック・git チェック・CI パイプラインを横断する決定論的な品質強制 |
| `maqa-ci` (v0.1.0) | GitHub Actions / CircleCI / GitLab CI / Bitbucket Pipelines を自動検出し、QA 未通過をブロック |
| `architecture-guard` (v2.3.6) | フレームワーク非依存のアーキテクチャガバナンス。ドリフト検知とルール強制 |
| `security-review` (v2.0.0) | ブランチ／PR／plan／task 単位の段階的セキュリティレビュー |
| `trace` (v1.0.0) | `spec.md` とテストスイートから要件 → テストのトレーサビリティ行列を構築。未テスト要件と孤立テストを検出 |

`ci-guard` が read-only であることは重要です。CI 上では成果物を書き換えず、判定だけを返す設計が推奨されるということです。

#### リリース系

| 拡張 | 内容 |
| --- | --- |
| `ship` (v1.0.0) | プリフライトチェック、ブランチ同期、CHANGELOG 生成、CI 検証、PR 作成の自動化 |
| `changelog` (v1.0.0, read-only) | 仕様の git 履歴と要件差分から CHANGELOG・リリースノートを生成 |
| `spec2cloud` (v1.1.0, 拡張＋プリセット) | Azure 向けに `spec → plan → tasks → implement → deploy` を拡張したワークフロー |
| `verify-review-ship` (v0.4.2) | 収束後の運用検証・技術レビュー・トランザクショナルなデリバリー |

`spec2cloud` は**コアの 4 段階に `deploy` を追加した**構成であり、「デプロイ段階は各社が拡張で足す」という Spec Kit の設計思想を体現しています。

### ホットフィックス対応パターン

#### 同梱 `bug` 拡張が正規の経路

```bash
specify extension add bug
```

```text
/speckit.bug.assess "TypeError: cannot read properties of undefined ..." slug=callback-token
/speckit.bug.fix    slug=callback-token
/speckit.bug.test   slug=callback-token
```

成果物は `.specify/bugs/<slug>/` に段階ごとの Markdown として保存されます（`assessment.md` / `fix.md` / `test.md`）。

設計上の要点:

- **`specs/` とは別ディレクトリ**に保存されます。ホットフィックスが機能仕様の番号体系を汚しません。
- `assess` は URL も受け付けます（GitHub Issue の URL を直接渡せる）。
- **`fix` だけがソースコードを編集**します。`assess` と `test` はソースに対して読み取り専用です。
- `fix` は assessment に列挙されたファイルの範囲内に留まり、逸脱する場合は **Deviations from Assessment** として記録します。
- `test` の判定は `verified` / `partial` / `failed` の 3 値。assessment に挙がった再現手順が実際には実行されなかった場合、`verified` と主張せず `partial` へ**格下げ**します。過大主張を構造的に防ぐ設計です。
- slug はケバブケースへ正規化されます。既存 slug への上書きは対話実行では確認、自動実行では拒否して新しい slug を採ります。

#### ホットフィックス時に Spec をどう扱うか

公式は明示していませんが、機構から導かれる整理は次のとおりです。

| 状況 | 推奨経路 | 理由 |
| --- | --- | --- |
| 仕様どおりに動いていない（実装バグ） | `bug` 拡張（`.specify/bugs/`） | 仕様は正しい。仕様を変更する必要がない |
| 仕様が誤っていた／要件が変わった | `/speckit.specify`（新規機能ディレクトリ）または既存 `spec.md` の改訂 | 永続化モデル（Flow-forward / Living / Flow-back）の選択に従う |
| 緊急度が高く分析の時間がない | まず `bug.fix` で止血し、事後に `bug.assess` 相当の記録を残す | 記録が残る構造を優先 |

**注意点**: `bug` 拡張の成果物は `specs/` の外にあるため、`/speckit.analyze` や `/speckit.converge` の整合性検査の対象外です。ホットフィックスによる挙動変更が `spec.md` と乖離した場合、それを検出する自動機構はありません。**ホットフィックス後に仕様を追随させる運用ルールが別途必要**です。

#### Issue 起点のホットフィックス自動化

Spec Kit リポジトリ自身の `.github/workflows/bug-{assess,fix,test}.md` が参照実装です。ラベル付与を人間のゲートとし、`bug-fix` は**ドラフト PR** を作って必ず人間レビューを挟みます。`permissions` は読み取りのみで、書き込みは `safe-outputs` に限定、保護ファイルへの変更はブロックされます。

`bug-fix` ワークフローは意図的に Spec Kit 非依存（`.specify/` に依存しない）で書かれており、他リポジトリへそのまま移植できると明記されています。

### 並行開発・マルチストリーム管理

#### ブランチ戦略

`git` 拡張（`specify extension add git`）が提供する設定は次のとおりです。

```yaml
branch_numbering: sequential   # または timestamp（YYYYMMDD-HHMMSS）
branch_template: ""            # 既定は "{number}-{slug}"
                               # トークン: {author}, {app}, {number}, {slug}
                               # 制約: {slug} は {number} より前に置けない
                               #       最終パスセグメントは {number}- で始まること
branch_prefix: ""              # 例 "features/{app}" → features/{app}/{number}-{slug}
commit_style: fixed            # または conventional（差分から Conventional Commit を生成）
auto_commit:
  default: false               # 全コマンド前後の自動コミット。既定は無効
```

- **機能ブランチ戦略**：標準対応です。`001-feature-name` 形式が既定。
- **Git Flow / モノレポ**：`branch_template` と `branch_prefix` で `features/web/001-foo` や `{author}/{app}/{number}-{slug}` のような名前空間が作れます（0.7.0 で gitflow 向けの単一セグメント接頭辞が追加）。
- **リリースブランチ戦略**：**専用の対応はありません**。Spec Kit はリリースブランチ・タグ・バージョン付けの概念を持ちません。`release/x.y` の運用は Spec Kit の外側で行い、Spec Kit は機能ブランチのみを扱うのが素直な使い方です。
- 自動コミットは全 18 フック分が用意されていますが、**既定はすべて無効**です。有効化するとエージェントの各ステップがコミットとして残り、レビュー時の追跡性が上がります。

#### 並行開発

- **アクティブ機能は 1 プロセスにつき 1 つ**です（`.specify/feature.json` が単一のポインタ）。同一チェックアウトで 2 機能を同時に進めることはできません。
- 公式の推奨は **worktree の利用**です（`concepts/spec-of-specs.md`：「独立スライスを並行して構築するには、実行ごとにアクティブ機能状態を分離するため別々の worktree を使う」）。
- モノレポでは `.specify/` を持つディレクトリごとにプロジェクトが独立し、機能番号も独立します。ただし Git ブランチ名前空間はリポジトリ全体で共有されます。
- ワークフロー内の `fan-out` は `max_concurrency` で並列度を制御できます（0.12.2）。

---

## キー質問への回答

### Spec Kit は機能ブランチ・リリースブランチ戦略に対応しているか？

**機能ブランチは対応、リリースブランチは非対応です。**

機能ブランチは `git` 拡張で `001-feature-name` 形式が既定生成され、テンプレートと接頭辞で Git Flow 風の名前空間にも適応できます。一方、リリースブランチ・タグ・セマンティックバージョニングに関する概念は Spec Kit に存在しません。

さらに 0.10.0（2026-06-09）以降、**Git そのものがオプトイン**になりました。アクティブ機能の決定は `.specify/feature.json` が担い、`git checkout` では切り替わりません。すなわち Spec Kit は「ブランチ戦略に従属しない」設計であり、既存のブランチ運用を変えずに導入できる反面、ブランチ運用の支援も期待できないということです。

### CD パイプライン（CI/CD ツール）との連携ポイントは？

主要な接続点は 5 つです。

1. **`specify workflow run --json`** — 実行結果を機械可読な JSON で取得。stdout は JSON のみ、進捗は stderr。
2. **`gate` + `verdict_input`** — 人間承認を `--input` で外部から与えられるため、Slack 承認やデプロイ承認ボタンと接続できます。
3. **状態永続化と `resume`** — ジョブをまたいだ長時間ワークフローが成立します。
4. **`SPECIFY_INIT_DIR` / `--non-interactive`** — `cd` 無しの対象指定と、TTY 無し環境での実行。
5. **read-only な検査コマンド** — `/speckit.analyze`（成果物整合性）、`ci-guard`（仕様ドリフトでマージブロック）、`trace`（要件 → テストのトレーサビリティ行列）を CI のチェックとして走らせる形が想定されています。

デプロイそのものは `shell` ステップか、`spec2cloud` のような拡張で足す領域です。

### ホットフィックス対応時に Spec をどう管理するか？

**同梱 `bug` 拡張を使い、`.specify/bugs/<slug>/` に assess → fix → test の 3 段階記録を残すのが正規経路です。** `specs/` の機能番号体系とは分離されるため、ホットフィックスが機能仕様の履歴を汚しません。

ただし次の 2 点は運用ルールで補う必要があります。

1. `bug` 成果物は `/speckit.analyze` / `/speckit.converge` の検査対象外です。ホットフィックスで挙動が変わった場合、対応する `spec.md` を更新する責任は人間側にあります。
2. 仕様自体が誤っていた場合は `bug` ではなく `specify` の経路に乗せるべきです。判断基準は「仕様どおりに動いていないのか（= bug）」「仕様が誤っていたのか（= spec 改訂）」です。

---

## 実務上の留意点（イクシス視点）

- 受託開発では顧客側の CI 環境が使えない場合があります。その場合でも `specify workflow run` はローカル実行で完結し、状態は `.specify/workflows/runs/` に残るため、手元での再現性は確保できます。
- 原理試作フェーズでは `gate` を多めに置き、`verdict_input` を使わず対話承認にするのが安全です。本番運用フェーズで無人化する、という二段構えが取れます。
- `shell` ステップのインジェクション懸念は、AWS の認証情報を持つ環境で特に重要です。デプロイ先は必ず `enum` で制約する規約を先に決めるべきです。
