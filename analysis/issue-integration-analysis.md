# Issue・Project 統合の詳細分析

## 分析対象

`docs/02-issue-project-integration.md` の深堀り版。実装レベルの機構、ユースケース別の設計、制限事項を扱います。

**参照**: `templates/commands/taskstoissues.md`（コア実装）、`extensions/EXTENSION-API-REFERENCE.md`、`.github/workflows/{bug-assess,bug-fix,bug-test,feature-assess}.md`

---

## 技術的な連携メカニズム

### 1. コア経路：`/speckit.taskstoissues` の実装解析

#### 依存関係

```yaml
tools:
  - github/github-mcp-server/list_issues
  - github/github-mcp-server/issue_write
scripts:
  sh: scripts/bash/check-prerequisites.sh --json --require-tasks --include-tasks
```

**`gh` CLI ではなく GitHub MCP サーバーに依存**します。この選択には次の含意があります。

| 観点 | 影響 |
| --- | --- |
| 前提条件 | エージェント側に GitHub MCP サーバーの構成が必要。`gh` の認証状態は無関係 |
| 権限 | MCP サーバーに与えたトークンのスコープが上限。Issue 書き込み権限が必要 |
| CI 実行 | MCP はエージェントセッション前提のため、素の CI ジョブから直接は呼べない |
| エージェント差 | MCP 非対応のエージェントではこのコマンドが機能しない |

CI から Issue 生成を行いたい場合、このコマンドは適しません。`gh` を使う自前スクリプトを `after_tasks` フックまたは Actions 側に置く設計になります。

#### 重複排除アルゴリズム

```text
1. tasks.md からタスク ID 集合 S を構築（正規表現 \bT\d{3,}\b）
2. list_issues（state 未指定 = open+closed 双方、perPage: 100、after カーソルでページング）
3. 各 Issue タイトルを \bT\d{3,}\b で照合し、S の要素にマッチしたら「Issue 済み」とマーク
4. S の全要素がマッチした時点、またはページ終端でページングを停止
5. 未マッチの ID についてのみ issue_write を実行
```

正規表現の設計意図がテンプレート内に詳細に記述されています。

- `\d{3,}`（3 桁以上）— `converge` が `T{M+1:03d}` で採番するため 1000 件超で 4 桁になる。`\d{3}` では末尾の `\b` が数字の間に入れず `T1000` が一切マッチしないため、**そのタスクは重複排除も作成もされず静かに脱落する**
- 単語境界 `\b` — `ST001` のようなトークンの内部一致を防ぐ。また数字列全体の消費を強制するため `T100` が `T1000` の内部にマッチすることはない
- `T001 ...` / `T001: ...` / `[T001] ...` の 3 つの表記を認識する

#### 安全機構

テンプレートには 2 つの `> [!CAUTION]` ブロックが置かれています。

1. **リモートが GitHub URL でなければ以降のステップに進んではならない**
2. **いかなる状況でも、リモート URL と一致しないリポジトリに Issue を作成してはならない**

これらは実行環境の制約ではなく**プロンプト上の指示**です。エージェントがこの指示に従うことが前提であり、技術的な強制力はありません。組織として重要なリポジトリを扱う場合、MCP サーバー側のリポジトリスコープ制限を併用すべきです。

#### 生成される Issue の内容

| 項目 | 値 |
| --- | --- |
| タイトル | `T001: Create project structure`（ID + コロン + 説明） |
| 本文 | **規定なし** |
| ラベル | 付与しない |
| アサイニー | 設定しない |
| マイルストーン | 設定しない |
| 親子関係 | 設定しない（サブ Issue 化しない） |
| `[P]` / `[US1]` マーカー | タイトル生成時に**除去される**（情報が失われる） |

`[US1]`（ユーザーストーリー帰属）と `[P]`（並列可能）が Issue 側に残らないため、**Issue だけを見てもストーリー構造と並列性が分からない**という情報欠落があります。これは実務上の主要な制約です。

### 2. フック経路：`.specify/extensions.yml`

外部トラッカー連携の標準的な差し込み口は次の 4 イベントです。

| イベント | 典型的な用途 |
| --- | --- |
| `after_tasks` | `tasks.md` 生成直後に外部トラッカーへ反映（Jira / Linear 拡張が使用） |
| `before_taskstoissues` | Issue 同期前のコミット・検証 |
| `after_taskstoissues` | 生成した Issue へのラベル・マイルストーン付与、Project への追加 |
| `after_implement` | 完了タスクに対応する Issue のクローズ |

制約として、**現行のコマンドテンプレートは `condition` を評価せず、`priority` による並べ替えも行いません**（`docs/reference/extensions.md` に明記）。条件付きフックを設計しても、コマンドテンプレート経由の実行経路ではスキップされます。

### 3. GitHub Actions 経路：ラベル駆動エージェンティックワークフロー

Spec Kit リポジトリ自身の運用実装です。`.github/workflows/<name>.md` に定義し `<name>.lock.yml` へコンパイルする方式（GitHub Agentic Workflows）。

#### 権限モデル

```yaml
permissions:
  contents: read
  issues: read          # 書き込み権限を与えない

safe-outputs:           # 書き込みは宣言された安全出力のみ
  add-comment:
    max: 1
  add-labels:
    allowed: [needs-assessment, needs-reproduction, fix-proposed, fix-blocked]
    max: 1
  create-pull-request:
    draft: true
    title-prefix: "[bug-fix] "
    labels: [bug-fix, automated]
    max: 1
    protected-files:
      policy: blocked
      exclude: [README.md, CHANGELOG.md]
```

この構造が示す設計原則は次の 3 点です。

1. **トークンには読み取り権限のみを与え、書き込みは種類・件数・対象を宣言した安全出力に限定する**
2. **人間のラベル付与を唯一のゲートとする**（`bug-fix.md`：「メンテナがゲートキーパーである。明示的に `bug-fix` ラベルが付いていない Issue には決して作用しない」）
3. **保護ファイルへの変更をブロックする**（除外リストで明示的に許可したファイルのみ変更可）

#### 段階的承認モデル

```text
Issue 起票
   ↓ 人間が bug-assess ラベルを貼る
[bug-assess] コードベース評価 → assessment を Issue コメントとして投稿 + severity ラベル
   ↓ 人間が内容を読み、bug-fix ラベルを貼る
[bug-fix] assessment コメントを読み取り → 修正適用 → ドラフト PR 作成
   ↓ 人間が PR をレビュー、bug-test ラベルを貼る
[bug-test] 再現とテストを実行 → verified / partial / failed を記録
```

**各段階の間に必ず人間の判断が入る**構造です。自律エージェントを Issue 運用に組み込む際の参照モデルとして価値があります。

`bug-assess` の実装には現実的な工夫も見られます。「GitHub Issues API は真のファイル添付をサポートしないため、`assessment.md` の全文を単一の Issue コメントとして投稿し、そのコメント自体を添付物として扱う」という記述です。

また `bug-fix` は「直前の assessment コメント」を探す際、**ボット／サービスアカウントが著者であり、期待される構造（assessment ヘッダ + Proposed Remediation / Files likely to change / Tests to add or update の各節）を持つもの**に限定します。人間が手書きした偽の assessment に反応しないための照合条件です。

---

## ユースケース別の統合方法

### ユースケース A：GitHub のみ、進捗の可視化が目的

**推奨構成**: コアのみ（`/speckit.taskstoissues`）

```text
tasks.md（真実源）→ Issue（可視化用の写像）
```

- 実装の進捗は `tasks.md` の `[X]` で管理する
- Issue は顧客・チームへの可視化目的と割り切る
- Issue のクローズは手動、または `after_implement` フックで自作コマンドを走らせる

**利点**: 追加インストール不要。未検証拡張への依存が無い。
**欠点**: タスク数 = Issue 数。ストーリー構造が Issue 側に残らない。

### ユースケース B：GitHub Projects でカンバン管理したい

**推奨構成**: コア + コミュニティ拡張 `tasks-to-project`（v0.2.0、要 `gh` と `python3`）

- `spec.md` / `tasks.md` とボードの間で priority と status を双方向同期
- コマンド 2 個、フック 2 個を提供

**留意点**: `verified: false`。導入前にリポジトリのコード確認が必要です。GitHub Projects v2 の GraphQL API を叩く実装になるため、トークンスコープ（`project` スコープ）の確認も要ります。

### ユースケース C：Issue 粒度を粗くしたい（推奨）

**推奨構成**: プリセットで `speckit.taskstoissues` を上書き

タスク 1 件 1 Issue は多くの現場で細かすぎます。プリセットで次のように変更できます。

- **ユーザーストーリー単位**で Issue を作る（`[US1]` マーカーで集約）→ Issue 数がストーリー数に一致
- **フェーズ単位**で Issue を作る（Setup / Foundational / US1 / US2 / Polish）
- 本文にタスク一覧をチェックボックスで埋め込む → GitHub 側でタスクリストの進捗バーが出る

プリセットは `wrap` 戦略でコアの安全チェック（リモート検証・重複排除）を保持したまま、タイトル生成規則だけを差し替える形が安全です。

### ユースケース D：Jira / Linear を使っている

**推奨構成**: プリセット `jira`（コマンド互換）または拡張 `jira-sync` / `linear`

- **プリセット `jira`** — `speckit.taskstoissues` を上書きするため、**コマンド名も手順も変わりません**。チームの学習コストが最小
- **拡張 `jira-sync` (v0.4.0)** — 冪等・ドリフト検知・フェイルクローズの reconcile エンジン。Epic=リポジトリ、Story=仕様、Subtask=フェーズという階層。要 `bash>=4.4`, `git`, `curl`, `jq`
- **拡張 `linear` (v0.8.0)** — 仕様 1 件 = Issue 1 件、タスクフェーズ = サブ Issue。フック 6 個で作業中も同期

`jira-sync` の設計方針（冪等・ドリフト検知・フェイルクローズ）は、自前実装する場合の設計指針としても参考になります。

### ユースケース E：Issue 起点で仕様を作りたい

**推奨構成**: 拡張不要。Issue 本文を `/speckit.specify` の引数に渡す

```text
/speckit.specify <Issue #123 の本文を貼り付け>
```

拡張を入れる場合は `github-issues`（インポート・更新同期・双方向トレーサビリティ）または `issue`（既存 Issue を起点に仕様を作成・同期）。

バグ報告 Issue については、同梱 `bug` 拡張の `/speckit.bug.assess <Issue URL>` が **URL を直接受け取れる**ため、追加拡張なしで対応できます。

---

## トラブルシューティング・制限事項

### 制限事項の一覧

| # | 制限 | 影響度 | 回避策 |
| --- | --- | --- | --- |
| 1 | Issue → `tasks.md` の逆同期が無い | 高 | `tasks.md` を真実源とする運用ルール。または `after_implement` フックで Issue をクローズ |
| 2 | Issue 本文・ラベル・マイルストーンが生成されない | 中 | プリセットで上書き、または `after_taskstoissues` フックで後処理 |
| 3 | `[US1]` / `[P]` マーカーが Issue に残らない | 中 | プリセットでタイトルまたは本文に含める |
| 4 | タスク数 = Issue 数（粒度が細かすぎる） | 中 | プリセットでストーリー／フェーズ単位に集約 |
| 5 | GitHub Projects 連携がコアに無い | 中 | `tasks-to-project` 拡張、または Projects の自動化ルール（Issue 作成時に自動追加）を使う |
| 6 | MCP 依存のため CI から直接呼べない | 中 | `gh` を使う自前スクリプトを Actions 側に置く |
| 7 | リモート検証がプロンプト上の指示にすぎない | 中 | MCP サーバー側でリポジトリスコープを制限 |
| 8 | 既存 Issue のタイトルが `T\d{3,}` にマッチすると誤検出 | 低 | 導入前に既存 Issue のタイトル規則を確認 |
| 9 | `bug` 拡張の成果物が `analyze` / `converge` の対象外 | 中 | ホットフィックス後に仕様を追随させる運用ルール |
| 10 | 連携拡張がすべて `verified: false` | 中 | 導入前のコード確認と小規模検証 |

### よくある想定トラブル

**Q. `/speckit.taskstoissues` が何も作成せず終了する**

考えられる原因は 3 つです。

1. `git config --get remote.origin.url` が GitHub URL でない（`> [!CAUTION]` により意図的に中断）
2. GitHub MCP サーバーが構成されていない
3. 全タスク ID が既存 Issue にマッチした（重複排除が正しく働いた結果）

**Q. `tasks.md` を再生成したら Issue が重複した**

タスク ID が振り直された場合に起こり得ます。重複排除は**タスク ID の一致**で判定するため、ID が変わると別タスクとして扱われます。`tasks.md` を再生成する際は、既存 Issue との ID 対応が保たれるか確認が必要です。

**Q. 1000 件を超えるタスクで一部が Issue 化されない**

現行テンプレートは `\d{3,}` で 4 桁以上に対応済みです。古いバージョンの Spec Kit（`\d{3}` 時代）を使っている場合は該当し得ます。`specify self check` でバージョンを確認してください。

---

## 設計上の総括

Spec Kit の Issue 連携は「**最小限のコア + 拡張による補完**」という一貫した方針で設計されています。コアが提供するのは片方向・冪等・安全チェック付きの生成のみで、双方向同期・Project 連携・他トラッカーはすべてエコシステム側に委ねられています。

この設計は次の利点と欠点を持ちます。

**利点**

- コアが小さく、トラッカーの選択に縛られない
- プリセットによる上書きで、コマンド名を変えずにトラッカーを差し替えられる（`jira` プリセットが実例）
- フックによる差し込み口が 18 箇所あり、自組織の運用に合わせやすい

**欠点**

- 標準的な「あるべき連携」が公式に示されていないため、各組織が設計判断を強いられる
- コミュニティ拡張はすべて未検証であり、採用判断のコストが利用者側にある
- コアだけでは Issue の粒度が細かすぎ、そのままでは実用性が低い場面が多い

**実務上の第一歩としては、コアの `/speckit.taskstoissues` をそのまま使い、粒度が問題になった時点でプリセットによる上書きへ進む**のが、リスクと学習コストの両面で妥当と判断します。
