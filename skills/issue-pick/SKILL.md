---
name: issue-pick
description: "Use when the user has NOT yet decided which issue to work on and needs help choosing. This is the pre-decision advisory phase: the user is weighing multiple open issues and wants structured guidance — not implementation. Key triggers: asking which issue to prioritize or tackle next, identifying which issues are blocked vs. ready to start independently, selecting issues that fit limited capacity (small/high-impact), or finding independent issues for parallel worktree sessions. The user's state is \"I have several candidates and don't know where to start.\" Provides ranked recommendation (1 pick + 1-2 alternates) across impact/dependencies/size/urgency — read-only, no state changes."
version: 1.0.4
---

# Issue Pick Skill

複数の open issue を統一観点で整理し、「次に着手すべきもの」の判断材料をユーザーに提示する read-only な advisory skill。重み付けや優先度の永続化は行わず、最終判断は人間に委ねる。

## スコープ

- **含む**: open issue の一覧取得、4 観点 (影響範囲 / 依存・blocker / 規模 / 緊急度) での構造化、推奨 1 件 + 補欠 1〜2 件の提示。
- **含まない**:
  - **state 変更**: issue body / labels / assignees / Projects v2 等への書き込みは一切行わない。
  - **ranking の永続化**: 出力は揮発的な advisory に留め、優先度を保存する仕組みを持たない。
  - **重み付け**: 観点ごとの重み付けやスコアリングはしない。文脈依存の判断 (好み・気分・直近の関心) は人間に委ねる。
  - **assigned filter**: 個人リポジトリでは無意味のため非対応。
  - **自動着手**: `issuekit:issue-implement` skill との連携は user 経由のみ。APM plain-skill mode では `issue-implement` として案内する。skill 内で chain しない。

## 依存

- **`gh` CLI**: issue 一覧取得・本文取得に使用する。

## 入力

- 引数なし: デフォルトで open + `Status: Ready` の issue のみを対象とする。
- `--include-draft`: Draft の issue も対象に含める。

## 実行手順

### 1. issue 一覧の取得 (本文込み)

`gh issue list` で本文 (`body`) も含めて **すべての open issue** を一括取得する。`Status: Ready` は issue 本文の先頭にしかなく、タイトル / labels だけでは判別できないため、ここで本文を取得しておく必要がある。

```bash
gh issue list --state open --limit 1000 --json number,title,labels,body,createdAt,updatedAt
```

> **注意 1**: タイトル / labels だけで先に上位 10 件へ絞り込むと、見た目が urgent な Draft issue が残って本文を読む段階で除外され、本来対象にすべき Ready issue を一度も評価しないまま「候補なし」になる可能性がある。Status 判定は必ず候補絞り込みより **前** に行う。
>
> **注意 2**: `--limit` を小さく設定すると、古い Ready issue が `gh issue list` の段階で truncate され、Status フィルタを通過する前に消える。リポジトリの open issue 数を上回る十分大きな値 (例: `--limit 1000`) を指定する。`--limit` 上限を超えるリポジトリでは `gh issue list --search "is:open sort:created-asc"` 等で並べ替えてページング取得し、全件評価を担保する。

### 2. Status フィルタリング

各 issue の本文先頭の `Status:` 行を確認し、対象を絞り込む。

- **デフォルト**: `Status: Ready` の issue のみを残す。`Status:` 表記が無い issue や `Status: Draft` の issue は **除外** する (それらは `issuekit:issue-implement`、または APM plain-skill mode の `issue-implement` で Ready 確認時にも弾かれるため、推奨に含めると整合しない)。
- **`--include-draft` 指定時**: `Status: Ready` と `Status: Draft` の issue を残す。`Status:` 表記が無い issue は依然として除外する。
- フォーマット不完全な issue があった場合は、出力末尾の hint に「`/issuekit:issue-refine <番号>` で整理を促す」旨を案内してよい (推奨候補としては扱わない)。

### 3. 候補の絞り込み (上限 10 件)

Status フィルタ後の対象 issue が **10 件を超える場合のみ** 一次選別を行い、上位 10 件に絞り込む。手順 1 で本文も取得しているため、本文の中身も含めて判断してよい (本文未読のままタイトル / labels だけで切り詰めると、本文に重要な受け入れ条件・blocker 情報が書かれた issue が deep-read 対象から永久に外れるため)。

一次選別の観点:

- タイトル・本文から推測される影響範囲・緊急度
- 本文に明記された blocker / 依存の有無
- labels (例: `bug`, `priority:high` 等が付いていれば優先)
- 直近の `updatedAt` (議論が活発なものは文脈が新鮮)

選別基準は **機械的な重み付けではなく**、観点を提示するための候補ピックアップに留める。

### 4. deep-read (親子関係の取得)

絞り込んだ候補について、コメントと親子関係を取得する。手順 1 で本文は既に取得済みだが、コメントには本文未反映の補足・最新方針・blocker が残ることがあるため、deep-read 対象では本文だけで判断しない。

```bash
for N in <候補番号>; do
  gh issue view "$N" --comments
done
```

コメントに本文と矛盾する内容、未解決 blocker、方針保留、受け入れ条件の未反映変更がある場合は、その issue を `Status: Ready` の推奨候補から外し、出力では別枠の「要確認」候補として扱う。本文とコメントが矛盾する場合は、`updatedAt` やコメント時系列を踏まえて最新の意図を推定し、判断できないものだけを要確認として扱う。推測で断定せず「コメント上の補足 / 要確認」として理由を明示する。

続けて親 issue を 2 系統で取得する。

1. **GitHub の sub-issue 機能 (推奨)**: REST の sub-issues endpoint (`GET /repos/{owner}/{repo}/issues/{issue_number}/parent`) を使う。親 issue が存在しない場合は 404 になるため、その場合は空扱いにして fallback へ進む。
2. **本文中の `親: #N` 記載 (fallback)**: 旧形式の issue や sub-issue 紐づけが漏れている issue では、本文の `親: #<番号>` リンクしか残っていないケースがある。本文を正規表現で走査し、親番号を抽出する。

```bash
REPO=$(gh repo view --json nameWithOwner --jq '.nameWithOwner')

for N in <候補番号>; do
  # (1) REST で親 issue を取得 (sub-issue 機能で紐づけられている場合)
  gh api "repos/${REPO}/issues/${N}/parent" \
    --jq '.number' || true

  # (2) 本文中の `親: #<番号>` (sub-issue 未紐づけ時の fallback)
  gh issue view "$N" --json body --jq '.body' | grep -oE '親: #[0-9]+' | grep -oE '[0-9]+'
done
```

両系統で得られた親番号の和集合を取り、重複を排除する。

親子関係の解釈は以下とする。`issuekit:issue-create` skill の運用では子 issue は「親 epic 配下の独立した実装タスク」として起票されるため、open な親 issue があっても **即 blocker とは限らない**。

- **親が closed**: 依存は解消済み。blocker 扱いしない。
- **親が open かつ `Status: Draft`**: 親側の受け入れ条件が確定しておらず、子 issue の前提が揺らぐ可能性がある。**blocker 候補** として注記する。
- **親が open かつ `Status: Ready` / それ以外**: 親が大きな epic として open のまま残っているだけのケースが多い。blocker ではなく **文脈情報** として「親: #N」を記載するに留める。

親 issue 本文の取得には別途 `gh issue view <親番号>` が必要。blocker 判定が出力に効く場合のみ取得し、不要な fetch は避ける。

### 5. 4 観点での構造化

各候補 issue について、以下 4 観点で判断材料を整理する。

| 観点              | 内容                                                               |
| ----------------- | ------------------------------------------------------------------ |
| **影響範囲**      | 変更が及ぶパッケージ / ファイル / ユーザー体験の広さ               |
| **依存・blocker** | 親 issue・先行 issue・外部リソース等への依存。blocker があれば明示 |
| **規模**          | 想定される変更行数・ファイル数・実装ステップ数の目安               |
| **緊急度**        | 期限・障害影響・他作業のブロッカー性などの時間的要素               |

各観点は issue 本文・コメント・labels・親子関係から読み取れる範囲で整理し、推測が必要な場合は「(推測)」と明示する。コメント上の要確認点がある issue は、推奨 / 補欠ではなく要確認候補として分離する。

### 6. 推奨と補欠の提示

整理結果から **推奨 1 件 + 補欠 1〜2 件** を選び、理由とともに提示する。

- ranking 全件 (3 件超の順位付け) は出さない。advisory に徹し、優先度の永続化と紛らわしくしないため。
- 推奨理由は 4 観点のどれを重視したかを明記する (例: 「影響範囲が広く、blocker もないため」)。
- コメント上の要確認点がある issue は推奨 / 補欠に含めない。出力に含める場合は「要確認」セクションへ分離し、先にコメント内容の確認または `issue-refine` が必要なことを示す。
- **重み付けはしない**: 「総合スコア」「優先度ポイント」のような数値化はせず、観点ごとの状態を並べた上で「これが妥当」と説明する形に留める。

### 7. 出力フォーマット

markdown 散文 + 観点別の箇条書きで出力する。出力末尾には次のアクションを促す **hint 行** を必ず含める。推奨 issue の Status により hint を分岐させる:

- 推奨が `Status: Ready`: plugin mode では `/issuekit:issue-implement <番号>`、APM plain-skill mode では `issue-implement <番号>` を案内する。
- 推奨が `Status: Draft` (`--include-draft` 指定時のみ): plugin mode では `/issuekit:issue-refine <番号>`、APM plain-skill mode では `issue-refine <番号>` で先に Ready 化を促す (`issuekit:issue-implement` / `issue-implement` は Draft を弾くため)。

```md
## 着手候補

### 推奨: #<番号> <タイトル>

- 影響範囲: ...
- 依存・blocker: ...
- 規模: ...
- 緊急度: ...

理由: <4 観点のどれを重視したかを含めた散文の説明>

### 補欠: #<番号> <タイトル>

- 影響範囲: ...
- 依存・blocker: ...
- 規模: ...
- 緊急度: ...

理由: <なぜ推奨ではなく補欠なのか>

(補欠は最大 2 件まで)

### 要確認: #<番号> <タイトル>

- コメント上の論点: <本文未反映の補足 / 本文との矛盾 / 未解決 blocker / 方針保留 / 受け入れ条件の未反映変更>
- 必要な確認 / 整理: <ユーザー確認、または `/issuekit:issue-refine <番号>` / APM plain-skill mode では `issue-refine <番号>` で本文へ反映>

---

着手する場合は `/issuekit:issue-implement <番号>` を呼んでください。
APM plain-skill mode では `issue-implement <番号>` を呼んでください。
(推奨が Draft の場合は代わりに `/issuekit:issue-refine <番号>`、APM plain-skill mode では `issue-refine <番号>` を案内)
```

## 重み付けについて

本 skill は **重み付けを行わない**。理由は以下:

- 「規模が小さければ優先」「緊急度が高ければ優先」のような単純なルールでは、ユーザーの直近の関心 (好み・気分・他作業との関連) を反映できない
- 重み付けを skill 内で固定すると、ユーザーが「今日は軽い作業をしたい」「今日はまとまった時間を取って大物に取り組む」のような文脈依存の判断ができなくなる

skill は **観点を統一フォーマットで提示するところまで** を担い、最終判断は人間に委ねる。

## 利用タイミング

- 複数の open issue が溜まってきて「次どれ?」と判断材料を整理したいとき。
- ユーザーが「issue 整理して」「次の作業候補出して」等を依頼したとき。

## 失敗時の対応

- `gh issue list` が失敗する (認証エラー等) 場合は、その旨を報告して終了する。
- 対象 issue が 0 件の場合は「対象 issue がない」旨を報告して終了する (`--include-draft` を促す案内をしてよい)。

## やらないこと

- **issue body / labels / Projects v2 等の state 変更**: 本 skill は完全に read-only。
- **ranking 全件の出力**: 推奨 1 + 補欠 1〜2 件のみ。優先度の永続化と紛らわしくしないため。
- **重み付け / スコアリング**: 観点を提示するに留め、数値化はしない。
- **`issuekit:issue-implement` への自動 chain**: 出力末尾の hint 行のみ。APM plain-skill mode では `issue-implement` として案内する。skill 内で自動呼び出しはしない。
- **assigned filter**: 個人リポジトリでは無意味のためサポートしない。
