---
paths:
  - "skills/**/*.md"
---

`skills/` 配下のファイルを変更した場合、PR を作成する前に変更した SKILL.md の frontmatter `version:` を必ずバンプすること。

## バージョンバンプのルール

semver (`MAJOR.MINOR.PATCH`) に従う:

- **patch** (`x.y.Z`): バグ修正・文言修正・誤字訂正など、動作に影響しない変更
- **minor** (`x.Y.0`): 後方互換のある機能追加・手順の追加・やらないこと欄の拡充など
- **major** (`X.0.0`): 既存の動作を壊す変更・スコープの大幅な変更など

## 対象ファイル

変更した skill の SKILL.md のみバンプする。変更していない skill の `version:` は触らない。

## 理由

`version:` の変化を GHA (`.github/workflows/release.yml`) が検知し、自動で GitHub Release を作成する。バンプしないままマージすると新バージョンのリリースが作成されず、`gh skill install hirokisakabe/issuekit@<version>` でのバージョン固定インストールが機能しなくなる。
