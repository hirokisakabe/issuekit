---
name: issue-implement
description: GitHub issue 番号を起点に、Status 確認 → worktree 化 → 実装・commit → 受け入れ条件チェック → cross-review → PR 作成 → CI 確認まで一気通貫で進める。ユーザーから「issue #N を実装して」「#N お願い」などの依頼があった場合に使用する。
---

# Issue Implement Skill

GitHub issue を起点とした issue-driven 開発サイクルの中核 skill。issue 番号を受け取り、Status 確認から PR 作成・CI 確認までを定型的に実行する。

実装中の試行錯誤や中間段階を **commit 履歴として残す** 設計。commit を細かく区切ったうえで、最終状態に対して `acceptance-check`（受け入れ条件検査）→ `cross-review`（base...HEAD diff へのレビュー）→ PR 作成の順で回す。レビュー由来の修正も独立した commit として履歴に残るため、後から「なぜそう直したか」を追えるようになる。

## 依存

- **`issuekit:cross-review` skill**: 実装・commit 後、PR 作成前に cross-review を実施する。APM plain-skill mode では `cross-review` として呼び出す。backend は環境変数 `CROSS_REVIEW_BACKEND` で `codex` / `claude-self` から選択でき、未指定時は利用可能な CLI を自動検出する。対応する CLI がいずれも未導入な場合は明確に失敗させる（該当 skill 側の失敗時対応に従う）。
- **`issuekit:acceptance-check` skill**: 実装・commit 後、cross-review より前に受け入れ条件の自動検査を実施する。APM plain-skill mode では `acceptance-check` として呼び出す。
- **`issuekit:worktree-start` skill**: Claude Code 環境かつ default branch 上で起動された場合に、実装直前で worktree への自動切り替えに使用する (条件付き、後述 step 4)。APM plain-skill mode では `worktree-start` として呼び出す。Claude Code 以外の runtime ではこの step は skip される。
- **`gh` CLI**: GitHub 操作全般に使用する。

## スコープ

- **含む**: Status 確認、Depends on の close 確認、親 issue の文脈取り込み、worktree への自動切り替え (Claude Code 環境かつ default branch 上のときのみ、条件付き)、実装と適宜 commit、lint/format/型チェック、受け入れ条件チェック、cross-review、PR 作成、CI 確認・修正。
- **含まない**:
  - default branch 名を hardcode した branch ガード。default branch 名はリポジトリにより異なる (main / master / develop / trunk 等) ため、`gh repo view --json defaultBranchRef --jq '.defaultBranchRef.name'` で動的に解決した値と現在ブランチを比較する。
  - 非 Claude Code 環境 (Codex CLI / Cursor / Gemini) 向けの worktree 化フォールバック。`EnterWorktree` ツールが無い環境ではこの step を skip し、ユーザーが事前に切った worktree / branch でそのまま続行する。
  - ユーザーが既に手動で feature ブランチに切り替えているケースの上書き。default branch 以外にいる場合は worktree 化を行わず既存ブランチを尊重する。
  - レビュー指摘の修正を `git commit --amend` / `rebase` / `fixup` で履歴整形すること。指摘対応は **追加 commit** で行い、試行錯誤やレビュー対応の経緯を履歴に残す。

## 実行手順

### 1. issue の取得と Status 確認

```bash
ISSUE_NUMBER=<issue 番号>
gh issue view "$ISSUE_NUMBER"
```

- 本文先頭の `Status:` を確認する。Status の判定軸は **受け入れ条件の確定度** 一本（実装方針の確定度は問わない）であり、その意味を踏まえて分岐する。
  - **`Status: Ready`**: 受け入れ条件が「やったかどうか自分で判定できる」形になっている。続行。
  - **`Status: Draft`**: 受け入れ条件が未確定（「仮」「要検討」を含む / 検証不能なほど曖昧）。着手しない。**worktree 化 (step 4) より前にここで early abort する**ため、worktree は作成されない。ユーザーに受け入れ条件の確認を促し、必要なら `issuekit:issue-refine` skill（APM plain-skill mode では `issue-refine`）で整理する。
  - **`Status:` 表記なし / フォーマット不完全**: 同様に worktree 化前に abort し、`issuekit:issue-refine` skill（APM plain-skill mode では `issue-refine`）での整理を案内する。

### 2. Depends on (依存 issue) の確認

issue 本文に `Depends on:` 行がある場合、列挙された依存 issue がすべて close 済みかを確認する。

- 抽出: `gh issue view "$ISSUE_NUMBER"` の出力から `Depends on:` で始まる行をパースし、`#<数字>` を列挙する。
- 判定は **本文の状態表記ではなく `gh issue view <番号> --json state` の実体で行う**。本文に状態を書き写さない設計（`issuekit:issue-create` 側）と整合させ、表記漏れの影響を受けないため。

```bash
gh issue view <依存 issue 番号> --json state --jq '.state'
# → "OPEN" or "CLOSED"
```

- **1 件でも `OPEN` の場合**: 着手しない。どの依存 issue が未 close かをユーザーに報告し、判断（先に依存を片付ける / 強行する / 中止）を仰ぐ。
- **すべて `CLOSED` の場合**: 次のステップへ進む。
- **`Depends on:` 行が無い場合**: 何もせず次のステップへ進む。

### 3. 親 issue の確認

issue 本文に `親: #<番号>` の記載、または GitHub の sub-issue として親が存在する場合は、必ず親 issue を `gh issue view` で取得し、文脈を踏まえる。

```bash
# 親 issue が本文に記載されている場合
gh issue view <親 issue 番号>

# sub-issue として登録されているかの確認 (任意)
REPO=$(gh repo view --json nameWithOwner --jq '.nameWithOwner')
gh api "repos/${REPO}/issues/${ISSUE_NUMBER}" --jq '.parent // empty'
```

### 4. worktree への自動切り替え (条件付き)

実装サイクルの冒頭で、default branch 上のまま実装を始めて main / master を直接汚す事故を機械的に防ぐためのステップ。以下の AND 条件 4 つを **すべて** 満たす場合のみ、`issuekit:worktree-start` skill (APM plain-skill mode では `worktree-start`) を呼び出して新規 worktree に切り替える。

1. **`EnterWorktree` ツールが利用可能** (= Claude Code 環境)。Codex CLI / Cursor / Gemini 等の非 Claude Code 環境では `EnterWorktree` が存在しないため自動的に false となり、本 step は skip される。
2. **現在のセッションが worktree の外**: `git rev-parse --git-common-dir` と `git rev-parse --git-dir` の出力が一致する。一致しなければ既に worktree 内なので skip。
3. **現在のブランチが default branch**: `gh repo view --json defaultBranchRef --jq '.defaultBranchRef.name'` の結果と `git rev-parse --abbrev-ref HEAD` が一致する。default branch 以外 (= ユーザーが手動で feature ブランチに切り替え済み) なら skip し、既存ブランチを尊重する。
4. **対象 issue が `Status: Ready`**: step 1 で確認済みの値を使う。`Status: Draft` は step 1 で early abort 済み (worktree も作成しない) なので、ここに到達した時点で常に Ready。

呼び方は **タスク説明モード** (issue 番号は渡さない)。issue title から kebab-case の slug を生成し、末尾に `-<issue 番号>` を付けたブランチ名 (例: issue #42「Slack 連携の OAuth フロー」→ `slack-oauth-flow-42`) を指定する。issue 番号を渡すと `worktree-start` 側で Status 判定経路に入り `issue-implement` への再帰連鎖が発生してしまうため、Status は本 skill 側で既に確認済みである旨を踏まえて純粋な worktree 切り替え機能だけを使う形にする。

いずれかの条件が欠ける場合は worktree を切らずにそのまま step 5 (実装) に進む:

- 非 Claude Code 環境 → ユーザーが事前に切った worktree / branch で続行。
- 既に worktree 内 → 二重発火を避けるため何もしない (`worktree-start` 側の再進入チェックでも no-op になる)。
- default branch 以外のブランチ → ユーザーが意図して feature ブランチを切っているとみなし上書きしない。

なお `worktree-start → issue-implement → worktree-start` の循環は、2 度目の `worktree-start` 呼び出しが上記条件 2 で worktree 内と判定され自動的に no-op になるため発生しない。

### 5. 実装（必要に応じて適宜 commit）

issue の「実装方針」「受け入れ条件」「スコープ外」に従い実装する。実装中に方針の揺らぎや不明点が出た場合は、勝手に拡張せずユーザーに確認する。

「実装方針」が **優先順位付き（または順序付き）の解消候補リスト** として書かれている場合は、上から順に試す。各候補の試行後に `acceptance-check` skill 相当の検証で受け入れ条件の充足を確認し、満たせない場合は次の候補へ進む。候補を恣意的に選ばず、最初から順に試すこと。すべて試しても受け入れ条件を満たせない場合は、勝手に新しい方針を追加せずユーザーに報告する。

実装完了の判定は **受け入れ条件のチェックリストをすべて満たしていること**。

#### commit 粒度のガイド

実装途中で「ここまでは固まった」という区切りがついたら、その時点で commit する。最終 PR 単位ではなく、**試行錯誤や中間段階を履歴に残す** ことを優先する。`acceptance-check` は最終状態に対して受け入れ条件を、`cross-review` は base...HEAD の最終 diff を対象に回るため、commit が何個に分かれていても判定結果は変わらない。

粒度の目安:

- **受け入れ条件 1 項目 ≒ 1 commit**: 受け入れ条件のチェック項目に対応する変更が一段落したら commit する。
- **リファクタや前処理は別 commit に分離**: 機能追加と無関係な整理（型の整え、import の並び替え、関連箇所のリネーム等）は同じ commit に混ぜず、独立した commit として切り出す。
- **試行錯誤の途中段階も残してよい**: 「方針 A を試した → やめて方針 B に切り替えた」のような経過は、後追いの価値があるなら commit として残す（ただし明らかに作業途中の壊れた状態は commit しない）。

commit メッセージは Conventional Commit-like prefix (`feat:` / `fix:` / `chore:` / `refactor:` / `docs:` 等) を使用し、scope を絞った具体的な記述にする。

### 6. lint / format / 型チェック

プロジェクトに設定されているフォーマッタ・リンタ・型チェックを実行し、すべてパスすることを確認する。これらが通らない場合は実装完了とみなさない。lint/format による自動修正が発生した場合は追加 commit として残す。

### 7. 受け入れ条件チェック (issuekit:acceptance-check skill)

実装と適宜 commit が一段落した時点で、受け入れ条件の充足を最終確認するステップ。`issuekit:acceptance-check` skill を呼び出し、issue 本文の `## 受け入れ条件` を抽出して各項目を自動検査する。APM plain-skill mode では `acceptance-check` を呼び出す。

**`acceptance-check` を `cross-review` より前に置く理由**: 受け入れ条件 ✗ の場合は実装に戻って修正することになり、追加変更が発生した時点で cross-review の対象 diff も変わる。先に cross-review を回すと、その指摘対応・受け入れ条件修正の双方で diff が動くたびに cross-review がやり直しになり無駄打ちが発生する。受け入れ条件を確定させてから cross-review を回すことで、レビュー対象 diff の安定性を確保する。

- **`✗` が 1 件でもある場合**: 受け入れ条件未達。step 5 の実装に戻って **追加 commit** で修正する（`git commit --amend` / `rebase` / `fixup` は使わない）。修正完了後にこの step 7 をやり直す。
- **`?` (要人間判定) のみの場合**: Claude 自身で動作確認方法を実行できる範囲は実行し、ブラウザ操作や主観評価など人間判定が必要なものは判断結果を添えて進む。判断が困難なものはユーザーに確認する。
- **すべて `✓` の場合**: 次の step 8 (cross-review) に進む。

`issuekit:acceptance-check` 自体は read-only であり、issue body や code を変更しない。failed 項目の修正は本 skill 側の責任。

### 8. cross-review (issuekit:cross-review skill)

受け入れ条件をすべて満たした最終形に対して、別 backend (Codex CLI または Claude CLI headless) による cross-review を実施する。`issuekit:cross-review` skill を呼び出す。APM plain-skill mode では `cross-review` を呼び出す。

- **critical** の指摘がある場合: **追加 commit** で修正してから次に進む（`git commit --amend` / `rebase` / `fixup` は使わない）。修正後に受け入れ条件への影響が無いか軽く確認する（影響が疑わしい場合は step 7 をやり直す）。
- **warning** の指摘がある場合: Claude 自身で対応要否を判断する。妥当な指摘は自律的に **追加 commit** で修正し、見送る場合は理由を添えて報告する（ユーザー確認は不要）。
- **info** のみの場合: 指摘を共有し、PR 作成に進む。

backend は環境変数 `CROSS_REVIEW_BACKEND` (`codex` / `claude-self`) で選べる。未指定時は同 skill 側で `command -v` による自動検出が走る。base branch は `gh repo view --json defaultBranchRef` から動的に解決される（`master` / `develop` / `trunk` でもそのまま動く）。default branch 解決が失敗した場合は同 skill が明示的に停止するので、エラー出力に従って原因を解消してから再実行する。

### 9. PR 作成

```bash
gh pr create --title "<title>" --body-file - <<'EOF'
<日本語の description>
EOF
```

- **PR description は日本語**で記載する（CLAUDE.md の常時適用ルール）。
- ユーザーから明示的に issue 番号を指定された場合のみ、description の先頭に `close #<issue 番号>` を記載する。本 skill のように issue 番号を起点に呼ばれた場合は、その issue 番号を「明示指定」とみなして `close #<issue 番号>` を入れてよい。
- description には目的、影響パッケージパス、ローカル検証手順を含める。

### 10. CI 確認

```bash
gh pr checks <PR 番号 or URL> --watch
```

- CI が成功するまで監視する。
- **失敗した場合**: ログを確認して修正し、追加 commit で対応する（`git commit --amend` / `rebase` / `fixup` は使わない）。
- **CI が開始されない場合**: ブランチのコンフリクトが原因の可能性が高い。`gh pr view` でコンフリクト状況を確認し、解消してから再度 CI を待つ。

### 11. 完了報告

PR URL と CI 結果（成功 / 修正後成功）をユーザーに返す。

## やらないこと

- default branch 名 (`main` / `master` / `develop` 等) を hardcode した branch ガード。step 4 の判定は `gh repo view --json defaultBranchRef --jq '.defaultBranchRef.name'` の結果と動的に比較する。
- 非 Claude Code 環境向けの worktree 化フォールバック実装。`EnterWorktree` ツールが無い環境では step 4 を skip し、ユーザーが事前に切った worktree / branch でそのまま続行する。
- 既に default branch 以外のブランチで作業しているユーザーへの worktree 強制切り替え。step 4 の条件 3 で skip する (= 既存 feature ブランチを尊重)。
- step 4 で `worktree-start` を呼ぶ際に issue 番号を渡すこと。issue 番号を渡すと `worktree-start` 側の Status 判定経路に入り `issue-implement` への再帰連鎖が起きるため、タスク説明モードで slug (`<title>-<issue 番号>`) のみを渡す。
- issue 本文や PR への `close` キーワードの自動付与（ユーザー明示指定時のみ）。
- 受け入れ条件を満たさない状態での PR 作成。
- 対応 backend CLI（codex / claude）がいずれも未導入な状態での cross-review 省略（該当する `issuekit:cross-review` / `cross-review` skill の失敗時対応に従い、明確に失敗させる）。
- **`acceptance-check` / `cross-review` / CI の指摘修正のために `git commit --amend` / `git rebase` / `git rebase -i` / `--fixup` / `git reset` 等で履歴を整形すること**。レビュー対応・修正対応はすべて **追加 commit** として残し、試行錯誤と修正経緯を後から追えるようにする。issuekit リポジトリは merge commit 運用（squash ではない）なので、commit 履歴は merge 後も価値を持つ。
