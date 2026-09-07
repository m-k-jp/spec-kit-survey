# 02 - Issue・Project との連携

## 調査対象

- Spec Kit が GitHub Issue・Project とどう連携するか
- 双方向同期の有無と仕組み
- Issue テンプレート、Project フィールドの構成
- ワークフロー・トリガー

**参照**: `templates/commands/taskstoissues.md`、`extensions/EXTENSION-API-REFERENCE.md`、`extensions/catalog.community.json`、`presets/catalog.community.json`、`.github/workflows/*.md`

---

## 結論（先に要約）

| 論点 | コア機能 | 補完手段 |
| --- | --- | --- |
| Task → GitHub Issue 生成 | **あり**（`/speckit.taskstoissues`） | — |
| Issue → Spec 取り込み | なし | コミュニティ拡張 `github-issues` / `issue` |
| Issue 状態 → Spec Kit への反映 | **なし**（片方向のみ） | コミュニティ拡張が独自実装 |
| GitHub Projects（v2）連携 | **なし** | コミュニティ拡張 `tasks-to-project` / `maqa-github-projects` |
| Jira / Linear / Azure DevOps | なし | コミュニティ拡張・プリセットが多数存在 |
| Issue 起点の自動実行 | コア機能ではない | `.github/workflows/*.md` のラベル駆動エージェンティックワークフローが参照実装 |

**Spec Kit のコアが提供する Issue 連携は、`tasks.md` から GitHub Issue を作る片方向・冪等な生成のみです。** 状態の逆流や Project ボード連携は、すべてコミュニティ拡張の領域です。

---

## 調査ノート

### Issue との連携：`/speckit.taskstoissues`

コア同梱コマンドであり、README の「Core Commands」表に掲載されています。テンプレート実体は `templates/commands/taskstoissues.md` です。

#### 実行機構

フロントマターで **GitHub MCP サーバー**のツールを 2 つ宣言しています。

```yaml
tools: ['github/github-mcp-server/list_issues', 'github/github-mcp-server/issue_write']
scripts:
  sh: scripts/bash/check-prerequisites.sh --json --require-tasks --include-tasks
```

`gh` CLI ではなく **MCP 経由**である点が特徴です。したがって利用にはエージェント側で GitHub MCP サーバーが構成されている必要があります。

#### 処理フロー

1. `check-prerequisites` で `tasks.md` の存在と `FEATURE_DIR` を取得
2. `.specify/memory/constitution.md` があれば読み込み
3. `git config --get remote.origin.url` でリモートを取得
   - **`> [!CAUTION] ONLY PROCEED TO NEXT STEPS IF THE REMOTE IS A GITHUB URL`**
4. **重複排除**：`list_issues` で既存 Issue を取得（`state` を渡さず open/closed 双方を対象、`perPage: 100`、カーソル `after` でページング）。タイトルを正規表現 `\bT\d{3,}\b` で照合し、既に Issue のあるタスク ID を除外
5. 残ったタスクについて `issue_write` で Issue を作成
   - `- [ ] T001 Create project structure` → タイトル `T001: Create project structure`
   - 先頭の `- [ ]`、`[P]`、`[US#]` マーカーを除去してから ID と説明を復元
6. **`> [!CAUTION] UNDER NO CIRCUMSTANCES EVER CREATE ISSUES IN REPOSITORIES THAT DO NOT MATCH THE REMOTE URL`**

#### 設計上の要点

- **冪等性**：タスク ID による重複排除により、`tasks.md` 再生成後の再実行やコマンド再呼び出しで重複 Issue が生まれません。
- **ページング打ち切り**：全タスク ID が照合できた時点でページングを止めます。Issue 履歴の大きなリポジトリで全件走査しないための配慮です。
- **4 桁以上の ID に対応**：`\d{3,}` としているのは、`/speckit.converge` が `T{M+1:03d}` で採番するため 999 件超で 4 桁になり得るからです。`\d{3}` では `T1000` が一切マッチせず、重複排除も作成も行われない不具合になる、とテンプレート内に明記されています。
- **本文の生成規則は未規定**：テンプレートが規定するのはタイトル形式のみです。Issue 本文・ラベル・アサイニー・マイルストーンの指定は含まれません。
- **逆方向の同期はありません**。Issue をクローズしても `tasks.md` の `[X]` は変わらず、`tasks.md` を更新しても既存 Issue のタイトル・本文は更新されません（新規タスクの追加分だけが追加作成されます）。

### フック機構による連携点

コアは 18 のフックイベントを定義しています。

```text
before_/after_ × constitution, specify, clarify, plan, tasks, checklist, analyze, implement, taskstoissues
```

設定は `.specify/extensions.yml` に置かれます。

```yaml
hooks:
  after_tasks:
    - extension: jira
      command: speckit.jira.specstoissues
      enabled: true
      optional: true
      prompt: "Create Jira issues from tasks?"
```

`optional: true` なら確認プロンプト付き、`false` なら `EXECUTE_COMMAND:` マーカー付きの自動実行として提示されます。**`after_tasks` と `before_/after_taskstoissues` が、外部トラッカー連携の標準的な差し込み口**です。

同梱の `git` 拡張は、全コアコマンドの前後に `speckit.git.commit` フックを登録する実例になっています（`before_taskstoissues` / `after_taskstoissues` を含む）。

### Project との連携

**コアには GitHub Projects（v2）連携は一切ありません。** コミュニティカタログに以下が存在します。

| 拡張 ID | 名称 | 内容 | 必要ツール |
| --- | --- | --- | --- |
| `tasks-to-project` | Tasks to GitHub Project (v0.2.0) | Spec Kit のタスクを GitHub Projects v2 のカードとして公開・同期。`spec.md`/`tasks.md` とボード間で **priority と status を双方向同期**。コマンド 2 個、フック 2 個 | `gh`, `python3` |
| `maqa-github-projects` | MAQA GitHub Projects (v0.1.0) | 仕様から draft issue を生成し、機能の進行に合わせて Status 列を移動。タスクリストのリアルタイム反映 | MAQA 拡張前提 |

Project のビュー・フィールド構成は GitHub Projects 側の機能そのものであり、Spec Kit がフィールドスキーマを規定することはありません。

### 他トラッカーとの連携（コミュニティ）

コミュニティ拡張カタログ 165 件のうち、カテゴリ `integration` は 23 件です。Issue トラッカー系の主なものは次のとおりです。

| ID | 対象 | 特徴 |
| --- | --- | --- |
| `linear` (v0.8.0) | Linear | 仕様 1 件 = Issue 1 件、タスクフェーズ = サブ Issue。コマンド 5・フック 6 で作業中も同期 |
| `linear-weave` (v1.0.1) | Linear | 要件の取り込み、`tasks.md` のサブ Issue 化、ステータス同期 |
| `jira` (v2.1.0) | Jira | Epic / Story / Issue 階層を設定可能。カスタムフィールド対応 |
| `jira-sync` (v0.4.0) | Jira | **冪等・ドリフト検知・フェイルクローズ**の reconcile エンジン。Epic=リポジトリ、Story=仕様、Subtask=フェーズ |
| `jira-mirror` (v0.24.0) | Jira | team-managed / company-managed 双方、Scrum/SAFe 階層、マルチプロジェクト対応 |
| `azure-devops` (v1.0.0) | Azure DevOps | User Story と Task を Work Item へ同期（OAuth 認証） |
| `github-issues` (v1.0.0) | GitHub Issues | **Issue から仕様成果物を生成**。インポート、更新同期、双方向トレーサビリティを標榜 |
| `issue` (v1.0.0) | GitHub Issues | 既存 Issue を起点にローカル仕様を作成・同期 |

さらに**プリセット**として `jira` が存在し、これは `speckit.taskstoissues` を上書きして GitHub Issue の代わりに Jira の Epic/Story/Task を Atlassian 経由で作成します。プリセットによる置換であるため、コマンド名も操作手順も変わりません。

> **注意**: これらはいずれも `verified: false` です。公式ドキュメントは「カタログ掲載は発見可能性を意味するのみで、コードの監査・推奨・サポートを意味しない」と明記しています。導入前に各リポジトリのコード確認が必須です。

### スペック → Issue → Task の流れ

現行の標準的な流れは次のとおりです。

```text
/speckit.specify   →  spec.md（ユーザーストーリー P1..Pn）
/speckit.plan      →  plan.md ほか設計成果物
/speckit.tasks     →  tasks.md（T001.. / [P] / [US#]）
        │
        ├─ after_tasks フック（任意）
        │
/speckit.taskstoissues → GitHub Issue（T001: <説明> を 1 タスク 1 Issue で作成）
        │
/speckit.implement →  コード実装、tasks.md に [X] を記録
```

**Issue は「実行の記録先」ではなく「追跡の写像先」**です。実装の進行は `tasks.md` の `[X]` が真実源であり、Issue のクローズはコアの関知するところではありません。

### Issue 起点の自動化：ラベル駆動エージェンティックワークフロー

Spec Kit リポジトリ自身が `.github/workflows/` に 4 つのエージェンティックワークフローを持っています（`*.md` で定義し `*.lock.yml` へコンパイルする GitHub Agentic Workflows 方式）。**これは Spec Kit の機能ではなく、リポジトリ運用のリファレンス実装**ですが、Issue 連携の設計例として価値があります。

| ワークフロー | トリガー | 動作 | 安全出力 |
| --- | --- | --- | --- |
| `bug-assess` | Issue に `bug-assess` ラベル | コードベースに対しバグ報告を評価し、`assessment.md` 相当を Issue コメントとして投稿 | コメント 1 件、ラベル最大 2 件 |
| `bug-fix` | Issue に `bug-fix` ラベル | 直前の assessment コメントを読み、修正を適用して**ドラフト PR** を作成 | PR 1 件（draft、`[bug-fix] ` 接頭辞、保護ファイルはブロック）、コメント 1 件 |
| `bug-test` | Issue に `bug-test` ラベル | 再現と追加テストを実行し検証結果を記録 | — |
| `feature-assess` | Issue に `feature-assess` ラベル | Spec Kit CLI を CI 上でインストールし `assess` 拡張のアイデア評価パイプラインを実行、各段階を Issue へ投稿 | コメント最大 5 件、ラベル 1 件 |

設計上の特徴:

- **人間がラベルを貼る行為が唯一のゲート**です。`bug-fix` のプロンプトには「メンテナがゲートキーパーである。明示的に `bug-fix` ラベルが付いていない Issue には決して作用しない」と明記されています。
- `permissions` は `contents: read` / `issues: read` に絞られ、書き込みは `safe-outputs`（コメント・ラベル・PR 作成）に限定されます。
- `bug-fix` は **Spec Kit 非依存**に設計されており（`.specify/` に依存しない）、他リポジトリへ移植可能と明記されています。
- 3 段階（assess → fix → test）それぞれで人間のラベル付けを要求する設計は、自律エージェントに対する段階的承認モデルとして参考になります。

---

## キー質問への回答

### Spec Kit の Task から GitHub Issue を自動作成できるか？

**できます。** コア同梱の `/speckit.taskstoissues` が `tasks.md` の各タスクを 1 Issue として作成します。GitHub MCP サーバー経由で動作し、タスク ID による重複排除で冪等性を保ちます。ただし作成されるのはタイトル `T001: <説明>` のみで、ラベル・マイルストーン・アサイニー・本文構造の指定はありません。それらが必要な場合はプリセットでコマンドを上書きするか、拡張を導入する必要があります。

### Issue の状態更新は Spec Kit に反映されるか？

**コア機能では反映されません。** `/speckit.taskstoissues` は片方向生成であり、Issue のクローズ・編集を検知する機構はコアに存在しません。

双方向を求める場合の選択肢は 3 つです。

1. コミュニティ拡張を導入する（`github-issues` は双方向トレーサビリティを標榜、`tasks-to-project` は priority/status の双方向同期を実装）
2. `after_implement` フックに自前コマンドを登録し、`[X]` の付いたタスクに対応する Issue をクローズする
3. GitHub Actions 側で Issue クローズをトリガーに `tasks.md` を更新する（Spec Kit の外側で実装）

実務上は 2 が最も見通しが良いと考えられます。`tasks.md` を真実源とし、Issue は表示用の写像と割り切る運用です。

### Project ビューはどの程度カスタマイズ可能か？

**Spec Kit 側にはカスタマイズ機構がありません。** GitHub Projects v2 のビュー・フィールド・自動化はすべて GitHub 側の機能です。

Spec Kit との接続を担うのはコミュニティ拡張 `tasks-to-project` で、`spec.md` / `tasks.md` とボードの間で priority と status を同期します。カラム構成やカスタムフィールドは Projects 側で自由に定義でき、拡張はそれに値を書き込む役割です。

---

## 実務上の留意点

- **GitHub MCP サーバーの構成が前提**です。社内でエージェントを使う際、MCP サーバーの認証（PAT のスコープ）を設計に含める必要があります。
- Issue が「1 タスク 1 件」で作られるため、タスク数がそのまま Issue 数になります。50 タスクの機能なら 50 Issue です。粒度によっては Issue 一覧が実用的でなくなるため、**フェーズ単位・ユーザーストーリー単位で Issue 化するようプリセットで上書きする**運用も検討に値します。
- 外部トラッカー連携拡張はいずれも未検証（`verified: false`）です。`jira-sync` の「冪等・ドリフト検知・フェイルクローズ」という設計方針は参考になりますが、採用時はコード確認と小規模検証が必要です。
