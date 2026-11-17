# 05 - アーキテクチャ・プリセット・拡張

## 調査対象

- Spec Kit の全体アーキテクチャ
- プリセットとエクステンションの仕組み
- CLI：specify コマンドの全体像
- カスタマイズ・拡張ポイント

**参照**: `src/specify_cli/`、`docs/reference/{extensions,presets,bundles,integrations,workflows}.md`、`extensions/EXTENSION-API-REFERENCE.md`、各 `catalog*.json`

---

## 調査ノート

### コアアーキテクチャ

Spec Kit は「**エージェントに渡すファイルを生成・解決するツール**」です。エージェント本体を内包せず、LLM API も呼びません。生成されるのは Markdown のコマンドプロンプトとテンプレート、およびシェル／PowerShell／Python のスクリプトです。

```text
┌──────────────────────────────────────────────────────────┐
│ specify CLI（Python 3.11+ / PyPI: specify-cli）           │
│  init / check / version / status / doctor / self-*        │
│  integration / extension / preset / workflow / bundle     │
└───────────────┬──────────────────────────────────────────┘
                │ 生成・解決
                ▼
┌──────────────────────────────────────────────────────────┐
│ .specify/  （プロジェクト状態・テンプレート・スクリプト） │
│ <agent>/   （.claude/skills, .github/prompts などへ出力） │
└───────────────┬──────────────────────────────────────────┘
                │ 読み込み・実行
                ▼
┌──────────────────────────────────────────────────────────┐
│ コーディングエージェント（Claude Code / Copilot / …）     │
│  スラッシュコマンド or skill として /speckit.* を実行     │
│  → スクリプトを呼び、Markdown 成果物を生成する            │
└──────────────────────────────────────────────────────────┘
```

この分離により、**エージェントを乗り換えてもプロセスと成果物が残る**という中心的な価値が成立しています。

`src/specify_cli/` の主要モジュール構成:

| モジュール | 役割 |
| --- | --- |
| `integrations/` | エージェント統合（`base.py` の抽象基底 + 41 のエージェント別実装） |
| `extensions/` | 拡張のインストール・レジストリ・フック実行 |
| `presets/` | プリセットの解決・優先度・合成戦略 |
| `workflows/` | ワークフローエンジン、`steps/`、`overlays/` |
| `bundler/` | バンドルの検証・ビルド・公開（`models/` `services/` `lib/`） |
| `authentication/` | オプトイン認証レジストリ |
| `catalogs.py` | カタログスタックの解決 |
| `events.py` | エージェントネイティブランタイムフック |
| `shared_infra.py` | 共有インフラ（管理対象ファイルのマニフェスト・ハッシュ管理） |
| `_assets.py` | wheel 埋め込みコアパック（エアギャップ対応の要） |

### CLI：specify コマンド体系

| 系統 | 主なサブコマンド |
| --- | --- |
| コア | `init` / `check` / `version [--features] [--json]` / `status` / `doctor` / `self check` / `self upgrade` |
| 統合 | `list` / `search` / `info` / `install` / `uninstall` / `use` / `switch` / `upgrade` / `status` / `catalog` |
| 拡張 | `add` / `remove` / `list` / `info` / `update` / `enable` / `disable` / `set-priority` / `catalog` |
| プリセット | `add` / `remove` / `list` / `info` / `resolve` / `enable` / `disable` / `set-priority` / `catalog` |
| ワークフロー | `run` / `resume` / `status` / `list` / `add` / `update` / `enable` / `disable` / `remove` / `search` / `info` / `validate` / `overlay` / `catalog` |
| バンドル | `search` / `info` / `install` / `update` / `remove` / `list` / `init` / `validate` / `build` / `publish` / `catalog` |

全系統に `catalog list|add|remove` が存在する点が構造的な特徴です。**5 種のプリミティブすべてがカタログ経由で配布される**という一貫した設計になっています。

### 5 つのプリミティブ + バンドル

| プリミティブ | 追加するもの | 同時有効数 | 配布 |
| --- | --- | --- | --- |
| Integration | エージェント接続（コマンド／skill ファイルの出力先と形式） | **1 プロジェクト 1 つ**（原則） | `integrations/catalog.json`（41 件） |
| Extension | 新しいコマンド・テンプレート・スクリプト・フック | 複数可 | 同梱 4 件 + コミュニティ 165 件 |
| Preset | 既存コマンド・テンプレート・スクリプトの上書き／合成 | 複数可（priority で階層化） | 同梱 2 件 + コミュニティ 34 件 |
| Workflow | 複数ステップの自動化定義（YAML） | 複数可 | 同梱 1 件（`speckit`）+ コミュニティ 2 件 |
| Workflow step | ワークフローの再利用可能なステップ型 | 複数可 | ステップカタログ |
| Bundle | 上記を役割単位で束ねた構成一式 | 複数可 | バンドルカタログ |

> 数値は参照コミット `4a7341a`（2026-09-04）時点のカタログ実測値です。公式ドキュメントの表記（38 統合 / 157 拡張 / 33 プリセット）は 2026-08-21 時点のものです。

同梱（オプトイン）拡張は 4 件です。

| 拡張 | 内容 |
| --- | --- |
| `git` | 機能ブランチ作成・命名規則・検証・リモート検出・自動コミット（全 18 フックに登録） |
| `bug` | バグトリアージ（assess → fix → test）、`.specify/bugs/<slug>/` に保存 |
| `assess` | アイデア評価（intake → research → define → shape → decide） |
| `agent-context` | エージェントコンテキストの更新（0.12.0 で完全オプトイン化） |

同梱プリセットは `lean`（最小構成コマンド）と `constitution-sync`（旧 `/constitution` 伝播挙動の復元）の 2 件です。

### プリセット（Presets）

プリセットは**既存の挙動を置き換える**仕組みです。新しいコマンドを増やさずに、仕様書の書式、コマンドの指示内容、スクリプトの中身を差し替えます。

#### 合成戦略

| 戦略 | 対象 | 動作 |
| --- | --- | --- |
| `replace`（既定） | テンプレート / コマンド / スクリプト | 優先度スタックの最初の一致を丸ごと採用 |
| `prepend` | テンプレート / コマンド | 下位レイヤの内容の**前**に自分の内容を置く |
| `append` | テンプレート / コマンド | 下位レイヤの内容の**後**に自分の内容を置く |
| `wrap` | テンプレート / コマンド | `{CORE_TEMPLATE}` を下位レイヤの内容で置換 |
| `wrap`（スクリプト） | スクリプト | `$CORE_SCRIPT` を下位レイヤの内容で置換 |

`wrap` は「コアの指示を保持したまま前後に自組織の規約を足す」用途に適します。0.8.0（2026-04-23）で導入されました。

#### 適用例（コミュニティ）

- `jira` — `speckit.taskstoissues` を上書きし、GitHub Issue の代わりに Jira の Epic/Story/Task を作成
- `toc-navigation` — 生成される `spec.md` / `plan.md` / `tasks.md` に目次を付与
- `explicit-task-dependencies` — `tasks.md` に `(depends on T###)` の明示的依存と実行波 DAG を追加
- `claude-ask-questions` — Claude Code 上の `/speckit.clarify` と `/speckit.checklist` を Markdown 表から `AskUserQuestion` へ置換
- `pirate` — 全出力を海賊口調に変換（プリセットの表現力を示すデモ）
- `fiction-book-writing` / `screenwriting` — SDD を小説・脚本執筆へ転用

`pirate` と `fiction-book-writing` の存在は「プロセスそのものが差し替え可能である」という設計思想の証左です。

### エクステンション（Extensions）

拡張は**新しい能力を追加**します。マニフェストは `extension.yml`（`schema_version: "1.0"`）です。

```yaml
extension:
  id: git
  version: "1.0.0"
requires:
  speckit_version: ">=0.2.0"      # 互換範囲の宣言
  tools:
    - name: git
      required: false
provides:
  commands: [...]                  # コマンド定義
  config: [...]                    # 設定テンプレート
  # 0.16.2 以降: templates, scripts も宣言可能
hooks:
  before_specify:
    command: speckit.git.feature
    optional: false
```

主要フィールド:

- `category` — `process` / `integration` / `docs` / `code` / `visibility`（0.10.2 で第一級フィールド化）
- `effect` — `read-only` / `read-write`（同上）
- `requires.speckit_version` — 互換範囲。コアの更新で拡張が非互換になることは構造的に起こり得ます

コミュニティ拡張 165 件の内訳（参照コミット時点）:

| カテゴリ | 件数 |
| --- | --- |
| `process` | 77 |
| `docs` | 32 |
| `integration` | 23 |
| `code` | 18 |
| `visibility` | 14 |

副作用の内訳は `read-write` 134 件、`read-only` 30 件です。CI ゲート系（`ci-guard`、`verify`、`trace`、`speckit-inventory`）が read-only である点は、導入リスク評価の目安になります。

### フック機構

コアが定義するイベントは `before_/after_` × 9 コマンド = 18 種です。設定は `.specify/extensions.yml` に集約されます。

| フィールド | 意味 |
| --- | --- |
| `extension` / `command` | 実行対象 |
| `enabled` | 無効化フラグ |
| `optional` | `true` なら `prompt` 付きで確認、`false` なら `EXECUTE_COMMAND:` 付きの自動実行として提示 |
| `priority` | 整数（1 以上）。既定 10 |
| `condition` | `config.<path>` / `env.<VAR>` を `is set` / `==` / `!=` で評価 |

**実装上の注意点が 2 つ**あります。公式ドキュメントが明記しています。

1. `HookExecutor.get_hooks_for_event()` は priority 昇順で返しますが、**現行のコマンドテンプレートは YAML の記述順のまま提示**しており、priority による並べ替えを行いません。
2. **現行のコマンドテンプレートは `condition` を評価せず、非空の condition を持つフックをスキップ**します。条件評価は `HookExecutor` 実装に委ねられています。

すなわち、フックの優先度と条件は「マニフェスト仕様としては存在するが、コマンドテンプレート経由の実行経路では未活用」という状態です。自組織で条件付きフックを設計する場合、この差異を前提にする必要があります。

0.15.0（2026-07-30）で追加された**エージェントネイティブランタイムフック**は別系統です。Claude Code の `SessionStart` / `PreToolUse` / `PostToolUse` / `SessionEnd` / `UserPromptSubmit` / `Stop` といったエージェント自身のイベントへ接続します。

### AI エージェント統合

`src/specify_cli/integrations/base.py` に 1 つの抽象基底と 4 つの出力形式クラスが定義されています。

```text
IntegrationBase (ABC)
├── MarkdownIntegration   … Markdown のスラッシュコマンドファイルを出力
├── TomlIntegration       … TOML 形式（Codex 等）
├── YamlIntegration       … YAML 形式
└── SkillsIntegration     … エージェントネイティブ skill を出力（現行の主流）
```

Claude Code 統合（`integrations/claude/__init__.py`）の実装例:

```python
class ClaudeIntegration(SkillsIntegration):
    key = "claude"
    config = {
        "name": "Claude Code",
        "folder": ".claude/",
        "commands_subdir": "skills",
        "requires_cli": True,
    }
    registrar_config = {
        "dir": ".claude/skills",
        "format": "markdown",
        "args": "$ARGUMENTS",
        "extension": "/SKILL.md",
    }
    multi_install_safe = True
    CANONICAL_TO_NATIVE = {          # ランタイムフックのイベント名対応
        "session_start": "SessionStart",
        "pre_tool_use": "PreToolUse",
        ...
    }
    events_config_file = ".claude/settings.json"
    events_format = "json-nested"
```

`ARGUMENT_HINTS` 辞書により、`/speckit.specify` に「Describe the feature you want to specify」といったインラインヒントが付与されます。

`FORK_CONTEXT_COMMANDS` のコメントは実装判断の記録として示唆的です。`/speckit.analyze` を fork サブエージェントで走らせる最適化（#2511）が一度導入されましたが、analyze の出力が 300〜500 行に及び、それが本体の会話へ注入されるため、fork ごとにコンテキストが累積して長時間セッションが破綻する（#3185）ことが判明し、**撤回**されています。現在この辞書は意図的に空です。

統合の運用ルール:

- プロジェクト内でアクティブな統合は 1 つ。`specify integration use|switch <key>` で切り替え
- 一部の統合は `multi_install_safe = True` で共存可能
- `upgrade` は同一統合のファイル更新、`switch` は別統合への切り替え
- エージェント CLI 未インストールでも `--ignore-agent-tools` で導入可能

### バンドル（Bundles）

バンドルは**新しい挙動を持ちません**。既存の拡張・プリセット・ワークフロー・ステップを curate してバージョン固定で一括導入する配布単位です。

保証事項として公式が挙げているのは 4 点です。

1. `info` が表示する内容と `install` が追加する内容が完全に一致する（透明性）
2. インストールは冪等で、プロジェクトルート内に限定される
3. `remove` は他のバンドルが依存しているコンポーネントに触れない
4. 消費側・作成側のコマンドはローカルまたは固定ソースに対して**オフラインで動作**する

`examples/bundles/` に 4 つの役割別サンプル（プロダクトマネージャ、ビジネスアナリスト、セキュリティリサーチャ、開発者）があります。

### カタログとセキュリティモデル

すべてのプリミティブは**優先度付きカタログスタック**（project > user > built-in）から解決されます。各ソースはインストールポリシーを持ちます。

| ポリシー | 動作 |
| --- | --- |
| `install-allowed` | `search` / `info` に加えてインストール可能 |
| `discovery-only` | 一覧・詳細表示のみ。インストールは拒否 |

セキュリティ上の重要事項:

- **カタログ掲載は監査・推奨・サポートを意味しません。** 公式ドキュメントの複数箇所で明記されています
- 配布アーカイブは sha256 検証されます（0.11.7 以降）
- URL 由来の拡張インストールには確認プロンプトが出ます（0.8.18 以降）
- 認証はオプトイン（`~/.specify/auth.json`）。未設定なら全 HTTP リクエストが無認証
- `bundle` と `workflow run <file>` は、書き込み境界を守るためシンボリックリンクされた `.specify/` を拒否します
- 1.0 前後で入力検証の強化が集中しています（不正なカタログペイロード・マニフェスト・URL ポートの拒否、プロヴェナンス読み取り失敗時のフェイルクローズ）

企業導入では、**自組織のカタログを立てて `install-allowed` を限定し、公式カタログを `discovery-only` にする**構成が想定されています。

---

## キー質問への回答

### AI エージェント統合（Claude、Copilot など）の詳細実装は？

`IntegrationBase` を継承した 41 個のエージェント別クラスが `src/specify_cli/integrations/<key>/` に実装されています。各クラスが宣言するのは主に次の情報です。

- 出力先ディレクトリと形式（例：Claude は `.claude/skills/<name>/SKILL.md`、Markdown 形式、引数プレースホルダ `$ARGUMENTS`）
- CLI ツールの要否と導入 URL
- 複数統合の共存可否（`multi_install_safe`）
- ランタイムフックのイベント名対応と設定ファイル（Claude は `.claude/settings.json`、`json-nested` 形式）
- コマンド固有のフロントマター注入（Claude の `argument-hint` など）

出力形式は 4 系統（Markdown / TOML / YAML / Skills）で、現行の主流は **skills 形式**です。Copilot は 0.16.0（2026-08-05）で skills が既定になりました。skills 形式では呼び出しが `/speckit.specify` ではなく `speckit-specify`、`$speckit-*`、`/skill:speckit-*` などエージェント固有の形になります。

テンプレート内の `__SPECKIT_COMMAND_PLAN__` 等のプレースホルダが、この呼び出し形式の差異を吸収しています。

該当統合が無い場合の逃げ道として `generic` 統合があり、`--integration-options="--commands-dir .myagent/cmds"` で任意のディレクトリへ出力できます。

### プリセット・エクステンションの優先度・競合解決は？

**解決スタックの 4 層構造**が唯一のルールです。

```text
1. .specify/templates/overrides/     プロジェクトローカル上書き（最優先）
2. .specify/presets/<id>/            プリセット（priority 昇順、小さい値が先）
3. .specify/extensions/<id>/         拡張（priority 昇順）
4. .specify/templates/               Spec Kit コア（最低優先）
```

競合解決の具体的な挙動:

- **ファイル名単位で独立に解決**されます。`plan-template.md` はプリセット A から、`spec-template.md` はプリセット B から、という状態が正常です
- 同名ファイルを複数が提供する場合、**最高優先度のものが勝ちます**（`replace` 戦略の場合）
- 優先度は `specify preset set-priority` / `specify extension set-priority` で制御します（例：`--priority 5` は `--priority 10` より先）
- **削除時は次点が自動的に復元**されます
- 解決結果の追跡には `specify preset resolve <name>` を使います
- テンプレート・スクリプトは**実行時**解決、コマンドファイルは**インストール時**にアクティブ統合のディレクトリへ書き出し
- `disable` はテンプレート／スクリプト解決から除外しますが、**登録済みコマンドは残ります**。コマンドの変更を止めるには `remove` が必要です

`prepend` / `append` / `wrap` を使う場合は「勝者総取り」ではなく下位レイヤの内容が合成されます。

### カスタムコマンドの作成・共有方法は？

段階的に 4 つの方法があります。

| 方法 | 適する場面 | 共有範囲 |
| --- | --- | --- |
| プロジェクトローカル上書き（`.specify/templates/overrides/`） | 1 プロジェクト限りの調整 | そのリポジトリのみ |
| プリセット | 既存コマンドの書式・指示を組織標準へ変更 | カタログ経由で組織・公開 |
| 拡張 | 新しいコマンドや外部ツール連携を追加 | 同上 |
| バンドル | 役割別に一式を配布 | 同上 |

**拡張の作成手順**（`extensions/EXTENSION-DEVELOPMENT-GUIDE.md`、`extensions/template/` がスケルトン）:

1. `extension.yml` に `id` / `version` / `requires.speckit_version` / `provides.commands` / `hooks` を記述
2. `commands/<name>.md` にコマンドプロンプトを記述（Universal Command Format）
3. `.extensionignore` で配布に含めないファイルを指定（0.2.1 以降）
4. ローカル検証 → ZIP または tar アーカイブ化
5. 公開：アーカイブをホスティングし、カタログソースを追加。コミュニティカタログへの掲載は Issue テンプレート経由で申請（`EXTENSION-PUBLISHING-GUIDE.md`）

**組織内配布**では、自前のカタログ JSON をホストして `specify extension catalog add <url>` するのが標準経路です。公開カタログへの掲載は不要です。

**ワークフローのカスタマイズ**には Overlays という別経路があります。インストール済みの `workflow.yml` を編集せずにプロジェクト側で拡張・上書きでき、`specify bundle update` や `specify workflow add` による更新でローカル改変が失われません。

---

## 実務上の留意点（イクシス視点）

- **プリセットが組織標準化の主軸**になります。仕様テンプレートに「測量精度」「現場環境条件」「安全要件」といった建設分野固有のセクションを `wrap` または `append` で足す形が現実的です。コアの更新に追随しやすくなります。
- 拡張は「新しいコマンドが必要になったとき」に限定するのが保守負担の観点で妥当です。`requires.speckit_version` の互換範囲を自分で管理する必要が生じるためです。
- Claude Code 統合は skills 形式で `.claude/skills/<name>/SKILL.md` へ出力されます。既存の社内 skill 資産との名前衝突を事前確認してください。
- カタログは自組織で立て、公式カタログは `discovery-only` にする構成が、顧客環境へ持ち込む際の説明責任を果たしやすい形です。
