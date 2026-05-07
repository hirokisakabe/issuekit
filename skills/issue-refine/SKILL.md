---
name: issue-refine
description: 既存の GitHub issue を `issue-create` skill のフォーマットに沿って整理する。タイトルのみで起票された issue や、フォーマット不完全な issue を後から rich plan に仕上げ直したい場合に使用する。
---

# Issue Refine Skill

既存 issue を取得し、不足セクションを対話で埋め、Status を再評価して `gh issue edit` で更新する。issue-driven 開発サイクルにおける Draft → Ready 移行や、雑に起票された issue の整理に使う。

## `issue-create` との違い

| 観点 | `issue-create`    | `issue-refine`             |
| ---- | ----------------- | -------------------------- |
| 対象 | 新規 issue        | 既存 issue                 |
| 操作 | `gh issue create` | `gh issue edit`            |
| 起点 | 白紙              | 既存本文を尊重して差分整理 |

フォーマット観点（共通セクション、Status 判定、親子 issue の扱い）は **`issue-create` skill を参照する**。本 skill では重複定義しない。

## 実行手順

### 1. 既存 issue の取得

```bash
ISSUE_NUMBER=<issue 番号>
gh issue view "$ISSUE_NUMBER"
```

タイトル、本文、ラベル、親子関係を確認する。

### 2. ギャップ分析

`issue-create` skill のフォーマット（共通セクション: 概要 / 背景・モチベーション / 受け入れ条件 / スコープ外 / 参考）に対して、現状の本文に **何が欠けているか** を洗い出す。

- 必須セクションの欠落
- `Status:` 表記の有無
- 親 issue がある場合の親リンクと「実装前に親 issue を必ず読むこと」明記
- 追加セクション（実装方針 / 再現手順 / 調査メモ等）の必要性

### 3. 対話で不足を埋める

ユーザーに対し、欠けているセクションごとに必要な情報を確認する。一度にまとめて聞かず、優先度の高いもの（概要・受け入れ条件・実装方針）から順に確認するのが望ましい。

既存の本文は **可能な限り尊重する**。表現を勝手に書き換えず、追記・補完を中心に進める。意図が不明な記述があればユーザーに確認してから整える。

### 4. Status の再評価

`issue-create` skill の「Status > Draft とする条件」に従い、整理後の本文で Status を再判定する。判定軸は **受け入れ条件の確定度** 一本。

- 受け入れ条件が「仮」「要検討」を含む / 検証不能なほど曖昧（やったかどうか自分で判定できない）→ `Status: Draft`
- 受け入れ条件が「やったかどうか自分で判定できる」形になっている → `Status: Ready`

実装方針の確定度では判定しない。複数案を優先順位付きで列挙していれば `Status: Ready` でよい。

整理前後で Status が変わる場合（例: Draft → Ready）は、その旨をユーザーに伝える。

### 5. issue 更新

`gh issue edit` で本文（必要ならタイトルも）を更新する。本文は `--body-file -` で標準入力から渡し、記号を含むタイトルでも安全に扱う。

```bash
gh issue edit "$ISSUE_NUMBER" --body-file - <<'EOF'
Status: Ready

## 概要
...
EOF
```

タイトルも変更する場合:

```bash
gh issue edit "$ISSUE_NUMBER" --title "<新タイトル>" --body-file - <<'EOF'
...
EOF
```

### 6. 親子関係の整備（必要時）

整理の過程で親 issue の存在が判明した場合は、`issue-create` skill の「親子 issue の扱い」に従い sub-issue として紐づける。

```bash
REPO=$(gh repo view --json nameWithOwner --jq '.nameWithOwner')
CHILD_ID=$(gh issue view "$ISSUE_NUMBER" --json id --jq '.id')
PARENT_ISSUE_NUMBER=<親 issue 番号>

gh api "repos/${REPO}/issues/${PARENT_ISSUE_NUMBER}/sub_issues" \
  -X POST \
  -f sub_issue_id="$CHILD_ID"
```

### 7. 完了報告

更新後の issue URL と Status をユーザーに返す。`Status: Ready` になった場合は、続けて `issue-implement` skill で実装に進めることを案内してよい。

## やらないこと

- close キーワード（`close #123` 等）を issue 本文・コメントに書かない。
- 既存本文の意図を勝手に書き換えない。あくまで補完・整形を中心に行う。
- 完全に空の issue を本 skill で扱わない。新規起票相当の場合は `issue-create` skill を使う。
