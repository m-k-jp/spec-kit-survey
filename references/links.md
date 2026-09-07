# 参考リンク集

> URL は 2026-09-07 時点で確認したものです。公式サイトのドメインは **`github.github.io`**（`.com` ではありません）。

## 公式（一次情報）

- [GitHub Spec Kit リポジトリ](https://github.com/github/spec-kit) — MIT ライセンス、GitHub, Inc. 著作
- [Spec Kit Documentation（公式サイト）](https://github.github.io/spec-kit/) — 実体は `docs/` ディレクトリ
- [CHANGELOG](https://github.com/github/spec-kit/blob/main/CHANGELOG.md)
- [GitHub Releases](https://github.com/github/spec-kit/releases)
- [PyPI: specify-cli](https://pypi.org/project/specify-cli/)

### ドキュメント個別ページ

| ページ | URL |
| --- | --- |
| Quick Start | <https://github.github.io/spec-kit/quickstart.html> |
| Installation | <https://github.github.io/spec-kit/installation.html> |
| Upgrade Guide | <https://github.github.io/spec-kit/upgrade.html> |
| What is SDD? | <https://github.github.io/spec-kit/concepts/sdd.html> |
| Spec Persistence Models | <https://github.github.io/spec-kit/concepts/spec-persistence.html> |
| Handling Complex Features | <https://github.github.io/spec-kit/concepts/complex-features.html> |
| Spec of Specs | <https://github.github.io/spec-kit/concepts/spec-of-specs.html> |
| Reference Overview | <https://github.github.io/spec-kit/reference/overview.html> |
| Core Commands | <https://github.github.io/spec-kit/reference/core.html> |
| Agentic SDD | <https://github.github.io/spec-kit/reference/agentic-sdd.html> |
| Agentic Bug Fix | <https://github.github.io/spec-kit/reference/agentic-bugfix.html> |
| Integrations | <https://github.github.io/spec-kit/reference/integrations.html> |
| Extensions | <https://github.github.io/spec-kit/reference/extensions.html> |
| Presets | <https://github.github.io/spec-kit/reference/presets.html> |
| Workflows | <https://github.github.io/spec-kit/reference/workflows.html> |
| Bundles | <https://github.github.io/spec-kit/reference/bundles.html> |
| Authentication | <https://github.github.io/spec-kit/reference/authentication.html> |
| Adopting in Existing Projects | <https://github.github.io/spec-kit/guides/existing-projects.html> |
| Evolving Specs | <https://github.github.io/spec-kit/guides/evolving-specs.html> |
| Monorepo | <https://github.github.io/spec-kit/guides/monorepo.html> |
| Enterprise / Air-Gapped | <https://github.github.io/spec-kit/install/air-gapped.html> |
| History | <https://github.github.io/spec-kit/history.html> |
| Community Extensions | <https://github.github.io/spec-kit/community/extensions.html> |
| Community Presets | <https://github.github.io/spec-kit/community/presets.html> |
| Community Walkthroughs | <https://github.github.io/spec-kit/community/walkthroughs.html> |

### リポジトリ内の重要ファイル（一次情報）

| ファイル | 内容 |
| --- | --- |
| `spec-driven.md` | SDD の完全な方法論（約 26 KB） |
| `AGENTS.md` | エージェント向けリポジトリガイド（約 31 KB） |
| `CONTRIBUTING.md` | 開発ワークフローと検証要件 |
| `templates/` | コアテンプレート 5 種 + コマンドプロンプト 11 種 |
| `scripts/{bash,powershell,python}/` | 機能作成・前提条件チェック・テンプレート解決 |
| `extensions/EXTENSION-API-REFERENCE.md` | 拡張マニフェスト仕様、Python API、フックイベント一覧 |
| `extensions/EXTENSION-DEVELOPMENT-GUIDE.md` | 拡張開発手順 |
| `extensions/EXTENSION-PUBLISHING-GUIDE.md` | 拡張公開手順 |
| `presets/ARCHITECTURE.md` | プリセットの内部構造 |
| `workflows/ARCHITECTURE.md` | ワークフローエンジンの内部構造 |
| `.github/workflows/{bug-assess,bug-fix,bug-test,feature-assess}.md` | ラベル駆動エージェンティックワークフローの参照実装 |
| `examples/bundles/` | 役割別バンドルのサンプル 4 種 |

### カタログ（機械可読な一次情報）

- 統合: <https://raw.githubusercontent.com/github/spec-kit/main/integrations/catalog.json>
- 拡張（同梱）: <https://raw.githubusercontent.com/github/spec-kit/main/extensions/catalog.json>
- 拡張（コミュニティ）: <https://raw.githubusercontent.com/github/spec-kit/main/extensions/catalog.community.json>
- プリセット（同梱）: <https://raw.githubusercontent.com/github/spec-kit/main/presets/catalog.json>
- プリセット（コミュニティ）: <https://raw.githubusercontent.com/github/spec-kit/main/presets/catalog.community.json>
- ワークフロー: <https://raw.githubusercontent.com/github/spec-kit/main/workflows/catalog.json>
- ワークフローステップ: <https://raw.githubusercontent.com/github/spec-kit/main/workflows/step-catalog.json>
- バンドル（コミュニティ）: <https://raw.githubusercontent.com/github/spec-kit/main/bundles/catalog.community.json>

## 学習リソース

- [Microsoft Learn — Spec-driven development with GitHub Spec Kit for enterprise developers](https://learn.microsoft.com/ja-jp/training/modules/spec-driven-development-github-spec-kit-enterprise-developers/)
- [Community Walkthroughs](https://github.github.io/spec-kit/community/walkthroughs.html) — .NET / Java / Go の brownfield 事例を含む
  - 第三者作。GitHub による審査・推奨・サポートはありません

## プロジェクト沿革（一次情報に準ずる）

- [Manfred Riem（リードメンテナ）— Six months leading Spec Kit](https://www.manorrock.com/blog/2026/07/22/six_months_leading_spec_kit.html)
- [Manfred Riem — Spec Kit turns one](https://www.manorrock.com/blog/2026/08/21/spec_kit_turns_one.html)

## 関連概念

- [Martin Fowler — Exploring Gen AI: SDD tools](https://martinfowler.com/articles/exploring-gen-ai/sdd-3-tools.html) — 公式ドキュメントが spec-first / spec-anchored / spec-as-source の分類を引用

## 注意事項

コミュニティカタログに掲載された拡張・プリセット・ワークフロー・バンドルは、いずれも **`verified: false`** です。公式ドキュメントは次のとおり明記しています。

> A catalog listing made a component visible, but did not mean its code had been audited or endorsed.

導入前にリポジトリのコードを確認してください。
