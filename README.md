# spec-kit-survey

GitHub Spec Kit の仕様駆動開発（SDD）に関する調査・分析プロジェクト。

## 調査目的

1. コア概念の理解（仕様・プラン・タスク管理）
2. Issue・Project との連携方式の把握
3. 開発ライフサイクル、CD、ホットフィックスへの対応
4. 既存プロジェクトへの段階的導入方針
5. イクシス受託開発への適用可能性の検討

## 調査方針

- **ドキュメント主導** — 公式ドキュメント・CHANGELOG を基本
- **必要に応じてソースコード参照** — ロジック詳細が必要な場合のみ
- **実装サンプルは後段階** — まずは理解に専念

## ディレクトリ構成

```
spec-kit-survey/
├── README.md                          # このファイル
├── spec-kit/                          # GitHub spec-kit クローン（参考用）
├── docs/                              # 調査ドキュメント
│   ├── 01-core-concept.md
│   ├── 02-issue-project-integration.md
│   ├── 03-sdlc-patterns.md
│   ├── 04-existing-projects.md
│   └── 05-architecture-notes.md
├── analysis/                          # 詳細分析ノート
│   ├── issue-integration-analysis.md
│   └── compatibility-check.md
└── references/                        # 参考資料
    ├── official-docs-summary.md
    └── links.md
```

## 調査進捗

- [ ] 公式ドキュメント読破
- [ ] CHANGELOG 確認
- [ ] Issue・Project 連携の仕組み理解
- [ ] CD・ホットフィックスパターン調査
- [ ] 既存プロジェクト適用方針作成
- [ ] イクシス適合性評価

## 参考リンク

- [GitHub Spec Kit 公式リポジトリ](https://github.com/github/spec-kit)
- [Spec Kit Documentation](https://github.github.com/spec-kit/)
- [Microsoft Learn - Spec-driven development with GitHub Spec Kit](https://learn.microsoft.com/ja-jp/training/modules/spec-driven-development-github-spec-kit-enterprise-developers/)