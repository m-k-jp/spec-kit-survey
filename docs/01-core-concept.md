# 01 - Spec Kit のコア概念

## 調査対象

- 仕様（Spec）、プラン（Plan）、タスク（Task）の定義と役割
- マークダウン形式での永続化
- プロジェクト構造と命名規則
- コア機能の全体像

**参照**: `github/spec-kit` @ `4a7341a`（CLI 1.0.5.dev0 / 直近リリース 1.0.4）、`docs/concepts/*`、`templates/*`、`scripts/python/*`

---

## 調査ノート

### 全体像：9 コマンドの直列プロセス

```text
constitution → specify → clarify → plan → checklist → tasks → analyze → implement → converge
```

必須の依存関係は `specify` → `plan` のみです。`clarify` / `checklist` / `analyze` は「意味のある曖昧さがある場合に追加する品質ゲート」と公式に位置づけられています。小規模機能では `specify → plan → tasks → implement → converge` の 5 コマンドで完結します。

各コマンドは**書き込み権限が明確に分離**されている点が設計上の要です。

| コマンド | 生成・更新する成果物 | 書き込み権限 |
| --- | --- | --- |
| `constitution` | `.specify/memory/constitution.md` | 自身の成果物のみ（PR #3790 以降、テンプレートへ伝播しない） |
| `specify` | `spec.md`、`checklists/requirements.md` | 機能ディレクトリ |
| `clarify` | `spec.md`（最大 5 問の回答を反映）、`checklists/requirements.md` | 同上 |
| `plan` | `plan.md`, `research.md`, `data-model.md`, `quickstart.md`, `contracts/` | 同上 |
| `checklist` | `checklists/<名前>.md`（カスタム） | 同上 |
| `tasks` | `tasks.md` | 同上 |
| `analyze` | なし（レポート出力のみ） | **読み取り専用** |
| `implement` | ソースコード、`tasks.md` の `[X]` マーカー | チェックリストのマーカーは**変更禁止** |
| `converge` | `tasks.md` の Convergence セクション | **追記のみ**（コードの編集・削除は不可） |

この権限分離により、「エージェントが自分で品質ゲートを通したことにしてしまう」自己承認を構造的に防いでいます。

### 仕様（Spec）

`templates/spec-template.md` が定義する必須セクションは 3 つです。

1. **User Scenarios & Testing**（必須）— 優先度付き（P1, P2, P3…）のユーザーストーリー。各ストーリーは **独立してテスト可能**（Independent Test）でなければならず、1 つだけ実装しても価値を届ける MVP になることが求められます。受け入れ条件は Given/When/Then 形式。
2. **Requirements**（必須）— `FR-001` 形式の機能要件。曖昧な箇所は `[NEEDS CLARIFICATION: ...]` として明示的にマークします。データを扱う場合は Key Entities を実装非依存で記述。
3. **Success Criteria**（必須）— `SC-001` 形式の**技術非依存かつ測定可能**な成功基準（例：「2 分以内にアカウント作成を完了できる」）。

補助セクションとして Edge Cases と Assumptions があります。Assumptions は「機能記述が指定していなかった箇所に、妥当な既定値を選んだ」ことを明示するための欄です。

技術スタックの記述は仕様には含めません。「what と why を書き、how は書かない」が原則です。

### プラン（Plan）

`templates/plan-template.md` の構成:

- **Technical Context** — 言語／バージョン、主要依存、ストレージ、テスト、対象プラットフォーム、プロジェクト種別、性能目標、制約、規模。未定の項目は `NEEDS CLARIFICATION` と書きます。
- **Constitution Check** — *GATE: Phase 0 research 前に通過必須、Phase 1 design 後に再チェック*。テンプレートには `[Gates determined based on constitution file]` というポインタのみが入り、実行時に生きた constitution から導出されます（ランタイム解決）。
- **Project Structure** — 機能ディレクトリの文書構成と、ソースコードの配置（単一プロジェクト／Web アプリ／モバイル+API の 3 案から選択し、実パスへ具体化）。
- **Complexity Tracking** — Constitution Check 違反がある場合のみ記入。「なぜ必要か」「より単純な代替をなぜ却下したか」を表で残します。

Phase 0（research.md）→ Phase 1（data-model.md / quickstart.md / contracts/）という段階構成を持ち、`tasks.md` は Phase 2 として `/speckit.tasks` が別途生成します。

### タスク（Task）

`templates/tasks-template.md` の書式は `[ID] [P?] [Story] Description` です。

- `T001` 形式の連番 ID。3 桁は下限であり、1000 件を超えれば 4 桁以上になります。
- `[P]` は並列実行可能（別ファイル・依存なし）を示すマーカー。
- `[US1]` はどのユーザーストーリーに属するかを示します。
- 説明には**正確なファイルパスを含める**ことが求められます。

フェーズ構成:

```text
Phase 1: Setup（共有インフラ）
Phase 2: Foundational（ブロッキング前提条件。ここが終わるまでユーザーストーリー着手不可）
Phase 3..N: ユーザーストーリーごとに 1 フェーズ（優先度順、P1 が MVP）
Phase 最終: Polish（横断的関心事）
```

テストタスクは独立フェーズではなく、**該当ユーザーストーリーのフェーズ内**に生成されます。かつ「仕様で明示的に要求された場合のみ生成する」オプション扱いです。

各ストーリーフェーズの末尾には **Checkpoint**（「この時点で US1 は単独で動作しテスト可能であるべき」）が置かれ、増分デリバリーの検証点になっています。

### ディレクトリ構造・ファイル体系

```text
<project-root>/
├── .specify/
│   ├── memory/constitution.md          # プロジェクト憲章（唯一の真実源、実行時に参照）
│   ├── templates/                      # コアテンプレート（解決スタック最下層）
│   │   └── overrides/                  # プロジェクトローカル上書き（最上層）
│   ├── scripts/{bash,powershell,python}/
│   ├── feature.json                    # アクティブ機能ポインタ（.gitignore 対象）
│   ├── integration.json                # アクティブ統合
│   ├── extensions.yml                  # 拡張登録とフック設定
│   ├── extensions/<id>/                # インストール済み拡張
│   ├── presets/<id>/                   # インストール済みプリセット
│   ├── workflows/{<id>/,runs/<run_id>/,overlays/<id>/}
│   ├── bugs/<slug>/                    # bug 拡張の成果物（assessment/fix/test.md）
│   ├── {extension,preset,workflow,integration}-catalogs.yml
│   └── .gitignore                      # 管理下。feature.json と local-config.yml を除外
└── specs/
    └── 001-feature-name/               # 機能ディレクトリ（= 1 機能 1 ディレクトリ）
        ├── spec.md
        ├── plan.md
        ├── research.md
        ├── data-model.md
        ├── quickstart.md
        ├── contracts/
        ├── tasks.md
        └── checklists/
            ├── requirements.md         # 組み込み。specify/clarify が保守
            └── <custom>.md             # /speckit.checklist 生成。レビュアー所有
```

`scripts/python/common.py` の `FeaturePaths` データクラスが、この成果物集合をコード上で定義しています（`feature_spec` / `impl_plan` / `tasks` / `research` / `data_model` / `quickstart` / `contracts_dir`）。

**機能ディレクトリの命名**は `scripts/python/create_new_feature.py` が担います。

- 既定は `specs/` 配下の既存最大番号 + 1 による連番（`001-feature-name`）。`--timestamp` で `YYYYMMDD-HHMMSS` 形式も選択可。
- 記述文からストップワードを除いて短縮名を生成。`--short-name` で明示指定も可能。
- ブランチ名長は最大 244 文字、機能番号の上限は int64。
- 生成後に `.specify/feature.json` へ `{"feature_directory": "specs/001-feature-name"}` を書き込みます。

### アクティブ機能の決定方式（重要）

**Git ブランチではなく `.specify/feature.json` がアクティブ機能を決めます。**

優先順位は次のとおりです。

1. 環境変数 `SPECIFY_FEATURE_DIRECTORY`
2. `.specify/feature.json` の `feature_directory`
3. どちらも無ければエラー終了（フォールバックしない）

`git checkout` だけでは対象機能は切り替わりません。Git は 0.10.0 以降オプトインの `git` 拡張であり、Spec Kit 本体は Git を必要としません。モノレポでは `SPECIFY_INIT_DIR`（プロジェクト選択）と `SPECIFY_FEATURE_DIRECTORY`（機能選択）の 2 軸で対象を決定します。

### テンプレートの解決順序

| 優先度 | レイヤ | パス |
| --- | --- | --- |
| 1（最優先） | プロジェクトローカル上書き | `.specify/templates/overrides/` |
| 2 | プリセット（priority 昇順） | `.specify/presets/<id>/` |
| 3 | 拡張（priority 昇順） | `.specify/extensions/<id>/` |
| 4（最低） | Spec Kit コア | `.specify/templates/` |

テンプレートとスクリプトは**実行時**に解決されます。コマンドファイルは**インストール時**にアクティブ統合のディレクトリへ書き出されます。合成戦略は `replace`（既定）／`prepend`／`append`／`wrap`。

---

## キー質問への回答

### Spec、Plan、Task は 1:1:N の関係か？

**ファイルとしては 1:1:1、内容としては 1:1:N です。**

1 つの機能ディレクトリに `spec.md` / `plan.md` / `tasks.md` が各 1 つ存在します。`tasks.md` は単一ファイルの中に N 個のタスク行を持つ形式であり、タスクごとにファイルが分かれることはありません。

より正確な内部構造は次のとおりです。

```text
1 spec.md
  └─ N ユーザーストーリー（P1, P2, P3… 優先度付き、各々独立テスト可能）
       └─ 1 tasks.md 内の 1 フェーズ
            └─ M タスク（T001…、[P] 並列マーカー、[US1] 帰属マーカー付き）
```

したがって「ユーザーストーリー : タスクフェーズ」が 1:1、「タスクフェーズ : タスク」が 1:M という入れ子になります。

補助成果物（`research.md` / `data-model.md` / `quickstart.md` / `contracts/`）は plan の Phase 0・Phase 1 出力であり、`plan.md` に対して 1:N です。

複数機能をまたぐ関係は「Spec of Specs」規約でのみ表現されます。これはツール機能ではなく、`roadmap.md` に ID・依存・ステータスを持つ表を置き、サブ仕様と双方向リンクを張る**Markdown の運用規約**です。

### マークダウンファイルの粒度管理はどのように行われるか？

粒度は 3 段階で管理されます。

1. **機能ディレクトリ単位** — 1 つの `/speckit.specify` 実行が 1 ディレクトリを生成します。粒度の上限は明示的には規定されておらず、「1 サイクルで実装しきれるか」が実務上の基準です。
2. **ユーザーストーリー単位** — 仕様テンプレートが「1 つだけ実装しても MVP になること」を要求しており、これが実質的な分割単位です。
3. **コンテキストウィンドウによる制約** — 公式ドキュメント `concepts/complex-features.md` は「実装フェーズの破綻原因はコンテキスト枯渇である」と明言し、対処を 4 段階で示しています（タスク数・フェーズの限定 → サブエージェント委譲 → 併用 → Spec of Specs による分解）。分解は最もオーバーヘッドが大きいため最終手段と位置づけられています。

粒度が過大になった場合の自動検出機構は**ありません**。`/speckit.analyze` は成果物間の整合性を見るものであり、粒度の妥当性は判定しません。

### テンプレートの標準は何か？

コアが同梱するテンプレートは 5 種類です。

| ファイル | 用途 |
| --- | --- |
| `templates/spec-template.md` | 仕様（User Scenarios / Requirements / Success Criteria） |
| `templates/plan-template.md` | 実装計画（Technical Context / Constitution Check / Project Structure / Complexity Tracking） |
| `templates/tasks-template.md` | タスク一覧（フェーズ構成とタスク書式） |
| `templates/constitution-template.md` | プロジェクト憲章（Core Principles / 追加制約 / 開発ワークフロー / Governance、バージョンと批准日を持つ） |
| `templates/checklist-template.md` | カスタム品質チェックリスト（`CHK001` 形式） |

加えて `templates/commands/` に 11 個のコマンドプロンプト（analyze / checklist / clarify / constitution / converge / implement / plan / specify / tasks / taskstoissues）が入り、これがエージェントへ配布されるスラッシュコマンドまたは skill の実体です。

テンプレート内の `__SPECKIT_COMMAND_PLAN__` のようなプレースホルダは、統合ごとのコマンド呼び出し形式（`/speckit.plan` / `$speckit-plan` / `/skill:speckit-plan`）へ実行時に置換されます。これによりテンプレート本体はエージェント非依存に保たれています。

---

## 実務上の留意点（イクシス視点の初期メモ）

- **Git 非依存**であるため、パートナー建設会社との協働で Git 運用が確立していない案件でも導入自体は可能です。
- 成功基準（SC）が「技術非依存かつ測定可能」であることを要求する点は、測量新技術の原理試作と相性が良いと考えられます（例：「点群 1 億点を 5 分以内に処理」）。
- 一方で、原理試作では「作ってみないと要件が決まらない」場面が多く、Flow-back Spec モデル（実装からの逆流を許容し後で整合させる）を選ぶのが現実的です。
