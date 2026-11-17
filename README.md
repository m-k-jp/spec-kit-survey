# spec-kit-survey

GitHub Spec Kit の仕様駆動開発（SDD）に関する調査・分析プロジェクト。

## 調査目的

1. コア概念の理解（仕様・プラン・タスク管理）
2. Issue・Project との連携方式の把握
3. 開発ライフサイクル、CD、ホットフィックスへの対応
4. 既存プロジェクトへの段階的導入方針
5. イクシス受託開発への適用可能性の検討

## 調査方針

- **一次情報主導** — 公式リポジトリのドキュメント・テンプレート・ソースコード・CHANGELOG を直接読解
- **必要に応じてソースコード参照** — ロジック詳細が必要な場合に `src/specify_cli/` および `scripts/` を確認
- **実装サンプルは後段階** — まずは理解に専念

## 調査対象バージョン

| 項目 | 内容 |
| --- | --- |
| 調査日 | 2026-09-07 |
| 参照コミット | `github/spec-kit` @ `4a7341a`（2026-09-04） |
| CLI バージョン | `1.0.5.dev0`（直近リリース 1.0.4 / 2026-09-02） |
| ライセンス | MIT（GitHub, Inc.） |

## ディレクトリ構成

```
spec-kit-survey/
├── README.md                          # このファイル
├── docs/                              # 調査ドキュメント
│   ├── 01-core-concept.md             # コア概念（Spec/Plan/Task、ディレクトリ体系）
│   ├── 02-issue-project-integration.md# Issue・Project 連携
│   ├── 03-sdlc-patterns.md            # SDLC・CD・ホットフィックス
│   ├── 04-existing-projects.md        # 既存プロジェクトへの適用
│   └── 05-architecture-notes.md       # アーキテクチャ・拡張機構
├── analysis/                          # 詳細分析ノート
│   ├── issue-integration-analysis.md  # Issue 連携の実装レベル分析
│   └── compatibility-check.md         # イクシス適合性評価と導入シナリオ
└── references/                        # 参考資料
    ├── official-docs-summary.md       # 公式ドキュメント要約
    ├── changelog-summary.md           # CHANGELOG 分析（220 リリース）
    └── links.md                       # リンク集
```

## 調査進捗

- [x] 公式ドキュメント読破 → `references/official-docs-summary.md`
- [x] CHANGELOG 確認 → `references/changelog-summary.md`
- [x] Issue・Project 連携の仕組み理解 → `docs/02` / `analysis/issue-integration-analysis.md`
- [x] CD・ホットフィックスパターン調査 → `docs/03`
- [x] 既存プロジェクト適用方針作成 → `docs/04`
- [x] イクシス適合性評価 → `analysis/compatibility-check.md`
- [ ] 実装サンプルによる検証（社内ツール 1 件でのパイロット）

## 調査結果の要約

### Spec Kit とは

「仕様駆動開発ツール」から「**任意のコーディングエージェントを SDLC 全体へ拡張する、意図駆動のハーネス**」へ定義が拡大しています。SDD は既定で同梱される 1 プロセスにすぎず、プロセス自体が差し替え可能な構成要素です。

コアの 9 コマンド:

```
constitution → specify → clarify → plan → checklist → tasks → analyze → implement → converge
```

必須依存は `specify → plan` のみ。`clarify` / `checklist` / `analyze` は品質ゲートとして任意に追加します。

### 押さえるべき 6 点

1. **アクティブ機能は `.specify/feature.json` が決める。** Git ブランチではありません（0.10.0 以降、`git` はオプトイン拡張）。
2. **Issue 連携は片方向のみ。** `/speckit.taskstoissues` が GitHub MCP 経由で `tasks.md` → Issue を冪等生成します。逆同期・Project 連携はコアに存在しません。
3. **書き込み権限が分離されている。** `analyze` は読み取り専用、`converge` は `tasks.md` への追記のみ、`implement` はチェックリストのマーカーを変更禁止。エージェントの自己承認を構造的に防いでいます。
4. **デプロイはコアの範囲外。** ワークフローエンジンが CI から呼べる形（`--json` / `resume` / `gate` + `verdict_input`）で用意されており、デプロイ段階は拡張で足す設計です。
5. **カスタマイズの主軸はプリセット。** 4 層の解決スタック（overrides > presets > extensions > core）と 4 つの合成戦略（replace / prepend / append / wrap）でコア更新に追随しやすい形の改変ができます。
6. **コミュニティ拡張はすべて未検証。** カタログ掲載は発見可能性を意味するのみで、監査・推奨・サポートを意味しません。

### イクシスへの適用（暫定結論）

| 案件種別 | 適用可否 | 推奨する永続化モデル |
| --- | --- | --- |
| 社内ツール開発 | 適用可 | Living Spec |
| 受託・製品化フェーズ | 適用可 | Flow-forward Spec |
| 受託・原理試作フェーズ | 条件付き | Flow-back Spec（試作サイクル末に仕様を追随させるルールが必須） |
| 極小規模のスクリプト | 非推奨 | — |

詳細と 4 フェーズの導入シナリオは `analysis/compatibility-check.md` を参照してください。

## 参考リンク

- [GitHub Spec Kit 公式リポジトリ](https://github.com/github/spec-kit)
- [Spec Kit Documentation](https://github.github.io/spec-kit/)
- [Microsoft Learn - Spec-driven development with GitHub Spec Kit](https://learn.microsoft.com/ja-jp/training/modules/spec-driven-development-github-spec-kit-enterprise-developers/)

その他は `references/links.md` にまとめています。
