# 04 - 既存プロジェクトへの適用

## 調査対象

- 既存プロジェクトへの段階的導入方針
- 既存 Issue・マイルストーン・ラベルとの共存
- マイグレーション戦略
- チーム導入時の学習曲線

**参照**: `docs/guides/existing-projects.md`、`docs/guides/evolving-specs.md`、`docs/concepts/spec-persistence.md`、`docs/upgrade.md`、`docs/community/walkthroughs.md`

---

## 調査ノート

### 導入前の準備

公式ガイド `guides/existing-projects.md` が示す 5 段階が唯一の一次情報です。要点は「**既存システムを仕様から作り直す必要はない**」という 1 文に集約されます。

#### 段階 1：レビュー可能なベースラインを作る

```bash
git switch -c chore/adopt-spec-kit     # 導入専用ブランチ
git status                             # 作業中の変更は commit または stash
```

初期化で生成される全ファイルが通常のコードレビューに乗る状態を先に作ります。

#### 段階 2：その場で初期化し、差分をレビューする

```bash
specify init --here --force --integration <key>
```

- `--here` はカレントディレクトリを対象にします。
- `--force` は非空ディレクトリでの初期化を許可します。**管理対象パスが競合する場合はファイルを置換し得る**ため、ベースライン作成後にのみ使用します。アプリケーション本体を削除することはありません。
- 追加されるのは `.specify/` 配下の共有ファイルと、選択した統合のコマンド／skill ファイルのみです。**既存コードの書き換えや、既存挙動からの仕様推定は行いません。**

Git 初期化と機能ブランチは `git` 拡張の担当であり、既定では導入されません（`specify extension add git`）。既存リポジトリのブランチ運用を尊重できる構造です。

#### 段階 3：ガードレールを捕捉する

`/speckit.constitution` には「**すでに真である規約、またはチームが明示的に採用に合意した規約**」のみを書きます。

公式の指示は明確です。

> Do not invent standards merely to fill the constitution template. The constitution governs later planning and analysis, so unrealistic rules create noise instead of useful constraints.

証拠として README、アーキテクチャ決定記録、コントリビューションガイド、CI 設定を用いることが推奨されています。

#### 段階 4：境界の明確な最初の変更を 1 つ選ぶ

**「既存システム全体の文書化」を最初の機能にしてはならない**、と明記されています（その棚卸し自体が成果物である場合を除く）。

仕様には要求された成果に加えて、**維持すべき互換性の境界**を書きます。公式の例文が示唆的です。

```text
/speckit.specify Add CSV export to the existing orders page. Preserve current
filters and authorization behavior. Export only the rows visible to the signed-in
user, and do not change the existing JSON API response.
```

既存コードは「実装コンテキスト」として扱われ、`spec.md` は既存全挙動の遡及仕様ではなく、これから行う変更を定義するものです。

#### 段階 5：仕様の経年管理モデルを決める

`concepts/spec-persistence.md` の 3 モデルから選び、constitution またはオンボーディング資料に明記します。

### 段階的導入パターン

公式の 5 段階に加え、コミュニティ拡張が既存コードベース向けの入口を提供しています（いずれも未検証）。

| 拡張 | 内容 |
| --- | --- |
| `brownfield` (v1.0.0) | 既存コードベース向けブートストラップ。アーキテクチャを自動発見し、SDD を漸進的に採用 |
| `brownkit` (v1.0.1) | 証拠駆動のケイパビリティ発見、既存コードベースのセキュリティ・QA リスク評価 |
| `time-machine` (v1.1.0) | 既存コードベースへ SDD を遡及適用。機能単位で analyse → spec → ship |
| `repoindex` (v1.0.0, read-only) | リポジトリのインデックス（概要・アーキテクチャ・モジュール）を生成 |
| `improve` (v1.0.1) | コードベースをシニア視点で監査し、優先度付きの自己完結型 spec プロンプトを `specs/` へ書き出す |
| `discovery` (v0.2.0) | 実現可能性、技術選定、レガシーコードベース評価などの技術ディスカバリ |
| `keel` (v0.2.0) | `/speckit.specify` の上流での証拠付きディスカバリと、実装後の往復ドリフト監査 |
| `onboard` (v2.1.0) | Spec Kit プロジェクトに不慣れな開発者向けの文脈的オンボーディング |
| `fx-to-dotnet` (v0.8.0) | .NET Framework から .NET への移行を 7 フェーズで統括 |

公式が示す漸進導入の骨格は次のとおりです。

```text
[導入ブランチ作成] → [init --here --force] → [差分レビュー・マージ]
        ↓
[constitution：既存の真実を記述]
        ↓
[境界の明確な小変更 1 件を SDD で通す]  ← ここで初めてチームが手触りを得る
        ↓
[永続化モデルの決定]  → 以降の機能へ展開
```

### 既存資産の扱い

#### 既存 Issue・マイルストーン・ラベル

**コアには既存資産との共存機構も競合検出もありません。** Spec Kit は `specs/` 以下と `.specify/` 以下しか関知しません。

`/speckit.taskstoissues` は既存 Issue を破壊しませんが、これは「タスク ID による重複排除」の副作用です。タイトルが `\bT\d{3,}\b` にマッチする既存 Issue があると、無関係でも「そのタスクは Issue 済み」と判定される可能性があります。**既存 Issue のタイトル命名規則に `T001` 形式が含まれないことを事前確認**すべきです。

マイルストーン・ラベル・アサイニーは `/speckit.taskstoissues` の生成対象外です。付与が必要な場合は次のいずれかになります。

- プリセットで `speckit.taskstoissues` を上書きする
- `after_taskstoissues` フックで後処理コマンドを走らせる
- GitHub Actions 側で Issue 作成イベントを拾って付与する

#### 既存ドキュメント・仕様書

既存の設計書を `spec.md` へ機械変換する公式ツールはありません。`time-machine` や `improve` といったコミュニティ拡張が遡及生成を試みる領域です。

現実的な扱いは「既存ドキュメントは constitution とプロンプトの入力として使い、`spec.md` は新規変更分のみを記述する」ことです。

#### Spec Kit 自身の更新（`docs/upgrade.md`）

既存プロジェクトの運用では、**2 つの独立した更新ループ**があります。

1. **CLI の更新** — `specify self check`（読み取り専用の確認）→ `specify self upgrade`（uv tool / pipx を自動判別）
2. **プロジェクトファイルの更新** — `specify integration upgrade <key>` および `specify extension update`。最終手段として `specify init --here --force`

保護すべき対象として公式が挙げているのは次の 3 種です。

- `.specify/memory/constitution.md`（カスタマイズ済みの憲章）
- `.specify/templates/` および `.specify/scripts/` の改変
- IDE 系エージェントの旧コマンドディレクトリ（重複スラッシュコマンドの原因）

管理対象ファイルはハッシュ管理されており、**非強制アップグレードでは改変済みファイルはカスタマイズとみなされ上書きされません**。`--force` は差分確認後にのみ使う、という順序が推奨されています。

`specs/` はテンプレートパッケージの一部ではないため、更新で失われることはありません。

### チーム適応とトレーニング

#### 学習曲線の実態

コマンド数は core 9 + 任意 3 + CLI 6 系統です。ただし日常的に使うのは 5 つ（`specify` / `plan` / `tasks` / `implement` / `converge`）に収まります。公式が「短経路」を先に提示しているのは、この段差を意識した構成と読めます。

躓きやすい概念を整理すると次のとおりです。

| 概念 | 誤解されやすい点 | 正しい理解 |
| --- | --- | --- |
| アクティブ機能 | `git checkout` で切り替わると思い込む | `.specify/feature.json` が決める。Git 非依存 |
| チェックリスト | エージェントがチェックを付けると思う | カスタムチェックリストは**レビュアー所有**。`[x]` は要件品質の合格であり実装完了ではない |
| `/speckit.analyze` | 修正までしてくれると思う | 読み取り専用。問題は**発生源のコマンドへ戻って**直す |
| `/speckit.converge` | コードを直してくれると思う | 追記専用。`tasks.md` に不足タスクを足すだけ |
| 仕様の粒度 | 大きな機能を 1 仕様で通そうとする | コンテキスト枯渇で実装が破綻する。分割戦略が必要 |
| constitution | 理想を書く場所と思う | 実際に守られている規約のみを書く。守れない規約はノイズになる |

#### 学習リソース

公式の community walkthroughs にブラウンフィールド事例が 3 件あります（いずれも第三者作、GitHub による審査・推奨・サポートはありません）。

- **ASP.NET CMS 拡張** — CarrotCakeCMS-Core（約 307,000 行）へ Docker Compose 基盤とトークン認証付き headless REST API を追加。事前の仕様も constitution も無い状態からの適用例
- **Java ランタイム拡張** — Piranha（約 420,000 行、180 Maven モジュール）へ管理コンソールを追加
- **Go / React ダッシュボード** — NASA Hermes（Go）へ React 製テレメトリダッシュボードを追加。**GitHub Copilot CLI からターミナルのみで完結**させた例

規模の大きい既存コードベースでも成立することを示す事例として参照価値があります。

---

## キー質問への回答

### 既存 Issue を Spec に変換するツール・手法があるか？

**コアにはありません。** 変換方向はコアでは `tasks.md → GitHub Issue` の片方向のみです。

逆方向（Issue → Spec）はコミュニティ拡張が担います。

| 拡張 | 内容 |
| --- | --- |
| `github-issues` (v1.0.0) | GitHub Issues から spec 成果物を生成。インポート・更新同期・双方向トレーサビリティを標榜。要 `gh >= 2.0.0` |
| `issue` (v1.0.0) | 既存 Issue を起点にローカル仕様を作成・同期。コマンド 3 個 |

いずれも `verified: false` です。

なお同梱の `bug` 拡張の `/speckit.bug.assess` は **GitHub Issue の URL を直接引数に取れます**。バグ報告 Issue については、拡張を追加せずコアの範囲内で「Issue → 評価レポート」の変換が可能です。

現実的な代替手段は、Issue 本文をそのまま `/speckit.specify` の引数に渡すことです。Spec Kit のコマンドはいずれも自由記述の引数を受け取るため、変換ツールが無くても運用は成立します。

### 新規プロジェクトと既存プロジェクトで異なる戦略が必要か？

**必要です。** 公式は 3 つの開発フェーズを区別しています（0-to-1 / Creative Exploration / Iterative Enhancement）。既存プロジェクトは Iterative Enhancement（Brownfield）に該当します。

差異を整理すると次のとおりです。

| 観点 | 新規（Greenfield） | 既存（Brownfield） |
| --- | --- | --- |
| 初期化 | `specify init <name>` | `specify init --here --force`（要ベースライン） |
| constitution | あるべき原則を設計する | すでに真である規約を記述する |
| 最初の機能 | 中核機能そのもの | 境界の明確な小変更 1 件 |
| spec の役割 | システム全体の定義 | **変更分のみ**の定義。既存挙動の遡及仕様ではない |
| 互換性 | 考慮不要 | 維持すべき境界を spec に明記する |
| plan | 技術選定から | 既存アーキテクチャ・依存・テスト規約の**再利用を検証** |
| Git 拡張 | 導入しやすい | 既存ブランチ運用と衝突しないか要確認 |
| 永続化モデル | 最初から選べる | 最初の 1 機能を通してから決めてよい |

### Team Plan・Enterprise Plan で異なる機能があるか？

**Spec Kit 自体にプラン差はありません。** Spec Kit は GitHub, Inc. 著作の **MIT ライセンスの OSS** であり、PyPI で公開されている `specify-cli` を誰でも同条件で利用できます。有償プランによる機能差、ライセンス認証、テレメトリによる機能制限は存在しません。

プランに依存するのは Spec Kit の**周辺要素**です。

| 要素 | プラン依存 |
| --- | --- |
| コーディングエージェント本体（Copilot、Claude など） | 各製品の契約による |
| GitHub Projects v2 の機能範囲 | GitHub のプランによる |
| GitHub Actions の実行時間・同時実行数 | GitHub のプランによる |
| GitHub MCP サーバー経由の Issue 操作 | 対象リポジトリへの権限による |
| プライベートカタログのホスティング | 自組織の環境次第（Spec Kit 側の制限は無い） |

一方、エンタープライズ向けの機能は**プラン差ではなく設計として**提供されています。

- **エアギャップ導入** — wheel 同梱のコアパックにより `specify init` はネットワーク不要
- **プライベートカタログ** — 自組織のカタログを立て、`install-allowed` / `discovery-only` の 2 種のポリシーで配布対象を統制
- **オプトイン認証** — `~/.specify/auth.json` で GitHub Enterprise Server や Azure DevOps（Azure AD サービスプリンシパル含む）へ認証
- **バンドル** — 役割別の構成一式を 1 コマンドで配布

---

## 実務上の留意点（イクシス視点）

- 受託開発では顧客リポジトリへ `.specify/` を追加することになります。**成果物の帰属と、顧客側レビューの負担**について事前合意が要ります。`init` 差分をレビュー可能な状態で提示する公式手順は、この合意形成にそのまま使えます。
- 既存 Issue に `T001` 形式のタイトルが無いことを事前確認してください（重複排除の誤判定を避けるため）。
- 社内ツールへの適用が最も低リスクです。まず社内ツールで 1 機能を通し、手順とプリセットを固めてから受託案件へ展開する順序が現実的です。
