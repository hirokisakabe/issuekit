---
name: issue-implement
description: 特定の GitHub issue への実装着手と PR 作成を依頼されたときに使う。issue 番号・URL・会話内で選んだ issue のいずれかを起点に、runtime と worktree の実装隔離を preflight で保証してから、実装・commit・lint・受け入れ条件チェック・cross-review・PR 作成・CI 確認まで一気通貫で自動進行する。コードを書いてプルリクを出す作業全般が対象で、issue 選定相談・タイトル編集・クローズ操作・PR レビュー単体には使わない。
version: 1.1.0
---

# Issue Implement Skill

GitHub issue を起点とした issue-driven 開発サイクルの中核 skill。issue 番号を受け取り、Status 確認から PR 作成・CI 確認までを定型的に実行する。

実装中の試行錯誤や中間段階を **commit 履歴として残す** 設計。commit を細かく区切ったうえで、最終状態に対して `acceptance-check`（受け入れ条件検査）→ `cross-review`（base...HEAD diff へのレビュー）→ PR 作成の順で回す。レビュー由来の修正も独立した commit として履歴に残るため、後から「なぜそう直したか」を追えるようになる。

## 依存

- **`issuekit:cross-review` skill**: 実装・commit 後、PR 作成前に、実装セッションから独立した reviewer session による second opinion を得る。APM plain-skill mode では `cross-review` として呼び出す。実装前に runtime と対応 CLI を事前確認し、未対応 runtime や CLI 未導入の場合は明確に失敗させる（該当 skill 側の失敗時対応に従う）。
- **`issuekit:acceptance-check` skill**: 実装・commit 後、cross-review より前に受け入れ条件の自動検査を実施する。APM plain-skill mode では `acceptance-check` として呼び出す。
- **`issuekit:worktree-start` skill**: Claude Code の対話 session が default branch 上にいる場合に、実装直前で `EnterWorktree` による専用 worktree への切り替えに使用する (後述 step 4)。APM plain-skill mode では `worktree-start` として呼び出す。他 runtime では呼ばず、runtime 別の安全な再開手順を案内して停止する。
- **`gh` CLI**: GitHub 操作全般に使用する。

## スコープ

- **含む**: Status 確認、Depends on の close 確認、親 issue の文脈取り込み、runtime / branch / worktree の実装隔離 preflight、Claude Code で可能な場合の worktree 切り替え、実装と適宜 commit、lint/format/型チェック、受け入れ条件チェック、cross-review、PR 作成、CI 確認・修正。
- **含まない**:
  - default branch 名を hardcode した branch ガード。default branch 名はリポジトリにより異なる (main / master / develop / trunk 等) ため、`gh repo view --json defaultBranchRef --jq '.defaultBranchRef.name'` で動的に解決した値と現在ブランチを比較する。
  - Codex App の managed worktree / Handoff の作成・操作。これらは App が所有する機能であり、skill は App 管理 worktree を作成したふりをしない。
  - Codex CLI の起動済み session を別 cwd へ安全に移せるという仮定。default branch 上では停止し、通常の `git worktree add` と `codex -C <path>` で新しい session を開始する手順を返す。
  - ユーザーが既に手動で feature ブランチに切り替えているケースの上書き。default branch 以外にいる場合は worktree 化を行わず既存ブランチを尊重する。
  - レビュー指摘の修正を `git commit --amend` / `rebase` / `fixup` で履歴整形すること。指摘対応は **追加 commit** で行い、試行錯誤やレビュー対応の経緯を履歴に残す。

## 実行手順

### 1. issue の取得と Status 確認

```bash
ISSUE_NUMBER=<issue 番号>
gh issue view "$ISSUE_NUMBER" --comments
gh issue view "$ISSUE_NUMBER" --json title,body,updatedAt,comments
```

- issue 本文だけでなく、コメントも必ず確認する。コメントには本文更新前の補足、後続議論で決まった方針変更、未反映の制約、追加の再現情報が残っていることがあるため、本文のみを唯一の文脈として扱わない。
- コメントに本文と矛盾する内容がある場合は、`updatedAt` やコメント時系列を踏まえて最新の意図を推定し、判断できなければ実装前にユーザーへ確認する。
- コメントに未解決 blocker、方針保留、受け入れ条件の未反映変更がある場合は、`Status: Ready` であっても着手を止める。ユーザーに確認するか、必要なら `issuekit:issue-refine` skill（APM plain-skill mode では `issue-refine`）で本文へ反映してから再開する。
- 本文先頭の `Status:` を確認する。Status の判定軸は **受け入れ条件の確定度** 一本（実装方針の確定度は問わない）であり、その意味を踏まえて分岐する。
  - **`Status: Ready`**: 受け入れ条件が「やったかどうか自分で判定できる」形になっている。コメント上の未解決 blocker / 本文矛盾 / 方針保留 / 受け入れ条件の未反映変更がなければ続行。
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

issue 本文に `親: #<番号>` の記載、または GitHub の sub-issue として親が存在する場合は、必ず親 issue をコメント込みで取得し、文脈を踏まえる。

```bash
# 親 issue が本文に記載されている場合
gh issue view <親 issue 番号> --comments

# sub-issue として登録されているかの確認 (任意)
REPO=$(gh repo view --json nameWithOwner --jq '.nameWithOwner')
gh api "repos/${REPO}/issues/${ISSUE_NUMBER}/parent" --jq '{number, title, state}' || true
```

### 4. 実装隔離 preflight (必須)

実装・ファイル書き込み・commit の **前** に runtime、現在 branch、worktree 状態、並列 worker かを分類する。default branch を直接変更しないことと、書き込みを伴う並列 worker が同じ working tree を共有しないことをここで保証する。後続 step 8 の `cross-review` に必要な CLI も同時に確認する。runtime は実行中 agent が明示的に把握している値を使い、`PATH` 上の CLI の存在順から推測しない。

```bash
DEFAULT_BRANCH=$(gh repo view --json defaultBranchRef --jq '.defaultBranchRef.name')
[ -n "$DEFAULT_BRANCH" ] || { echo "default branch を取得できませんでした。" >&2; exit 1; }
CURRENT_BRANCH=$(git rev-parse --abbrev-ref HEAD)
GIT_COMMON_DIR=$(git rev-parse --git-common-dir)
GIT_DIR=$(git rev-parse --git-dir)
```

分類後は次の表に従う。ここでいう「専用 worktree」は `GIT_COMMON_DIR` と `GIT_DIR` が異なる linked worktree を指す。

| 現在位置 / 呼び出し方 | 判定 |
| --- | --- |
| すでに専用 worktree 内 | 二重作成せず、その worktree で続行する。 |
| default branch 以外の既存 feature branch、かつ単独実装 | ユーザーの branch を上書きせず、そのまま続行する。 |
| 並列 worker | **1 worker = 1 worktree を必須**とする。専用 worktree 内でなければ、branch 名にかかわらず実装・commit 前に停止する。同じ worktree を別 worker と共有しない。 |
| default branch | runtime 別手順で専用 worktree へ移る。安全に移行できなければ停止する。 |

default branch 上の runtime 別分岐:

- **Claude Code 対話 session**: `EnterWorktree` が利用できる場合だけ `issuekit:worktree-start` (APM plain-skill mode では `worktree-start`) を呼ぶ。issue title から作った `<title-slug>-<issue 番号>` を **タスク説明モード**で渡し、切り替え後に `GIT_COMMON_DIR != GIT_DIR` を再確認してから続行する。`EnterWorktree` が無い旧版や、切り替えに失敗した場合は停止し、`claude --worktree <title-slug>-<issue 番号>` で新しい session を開始して `issue-implement <issue 番号>` を再実行するよう案内する。
- **Codex CLI**: worktree 作成を skip して続行してはならない。起動済み session の cwd を skill が安全に移せると仮定せず停止し、衝突しない実パスと branch 名を決めたうえで次の再開例を返す。

  ```bash
  git worktree add ../<repo>.<title-slug>-<issue番号> -b <title-slug>-<issue番号> "$DEFAULT_BRANCH"
  codex -C ../<repo>.<title-slug>-<issue番号>
  # 新しい session で issue-implement <issue番号> を再実行
  ```

- **Codex App**: App の **Worktree** で開始済み、または **Handoff** で managed worktree へ移動済みなら続行する。Local の default branch 上なら実装前に停止し、App UI で Worktree chat を開始するか Handoff してから再実行するよう案内する。managed worktree / Handoff は runtime 所有であり、skill 自身は作成・操作しない。
- **Claude Code Agent view / Desktop**: Agent view の background session と Desktop の新規 Code session は runtime が自動隔離する。実際に linked worktree へ移ったことを確認して続行する。移行前の main checkout では書き込みを始めない。
- **未対応 runtime**: default branch 上では停止する。対応する reviewer-session launch 手順も無ければ、cross-review preflight の時点でも停止する。

`Status: Draft` / フォーマット不完全 / コメント上の blocker は step 1 で early abort 済みなので、この preflight に到達しない。`worktree-start → issue-implement` で入った場合は linked worktree 判定により二重作成しない。

### 5. 実装（必要に応じて適宜 commit）

issue 本文の「実装方針」「受け入れ条件」「スコープ外」と、step 1 / step 3 で確認したコメントの文脈に従い実装する。実装中に方針の揺らぎや不明点が出た場合は、勝手に拡張せずユーザーに確認する。

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

受け入れ条件をすべて満たした最終形に対して、実装セッションから独立した reviewer session による cross-review を実施する。`issuekit:cross-review` skill を呼び出す。APM plain-skill mode では `cross-review` を呼び出す。

- **critical** の指摘がある場合: **追加 commit** で修正してから次に進む（`git commit --amend` / `rebase` / `fixup` は使わない）。修正後に受け入れ条件への影響が無いか軽く確認する（影響が疑わしい場合は step 7 をやり直す）。
- **warning** の指摘がある場合: 実装 agent 自身で対応要否を判断する。妥当な指摘は自律的に **追加 commit** で修正し、見送る場合は理由を添えて報告する（ユーザー確認は不要）。
- **info** のみの場合: 指摘を共有し、PR 作成に進む。

reviewer session は実行中 agent runtime に対応する CLI で起動する。Codex CLI で実装している場合は `codex exec --sandbox read-only`、Claude Code で実装している場合は `claude -p` を使う。base branch は `gh repo view --json defaultBranchRef` から動的に解決される（`master` / `develop` / `trunk` でもそのまま動く）。default branch 解決が失敗した場合は同 skill が明示的に停止するので、エラー出力に従って原因を解消してから再実行する。

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
- default branch 上で worktree 化を単に skip して実装へ進むこと。runtime が安全に切り替えられなければ、書き込み・commit 前に停止して再開手順を返す。
- Codex CLI の起動済み session の cwd を skill が変更すること、または Codex App の managed worktree / Handoff を skill が作成・操作すること。
- 書き込みを伴う並列 worker が同じ worktree を共有すること。並列 worker は branch 名にかかわらず 1 worker = 1 worktree とする。
- 単独実装で、すでに default branch 以外の feature branch にいるユーザーへの worktree 強制切り替え。step 4 の分類で既存 branch を尊重する。
- step 4 で `worktree-start` を呼ぶ際に issue 番号を渡すこと。issue 番号を渡すと `worktree-start` 側の Status 判定経路に入り `issue-implement` への再帰連鎖が起きるため、タスク説明モードで slug (`<title>-<issue 番号>`) のみを渡す。
- issue 本文や PR への `close` キーワードの自動付与（ユーザー明示指定時のみ）。
- 受け入れ条件を満たさない状態での PR 作成。
- 実行中 agent runtime に対応する CLI が未導入な状態での cross-review 省略（該当する `issuekit:cross-review` / `cross-review` skill の失敗時対応に従い、明確に失敗させる）。
- **`acceptance-check` / `cross-review` / CI の指摘修正のために `git commit --amend` / `git rebase` / `git rebase -i` / `--fixup` / `git reset` 等で履歴を整形すること**。レビュー対応・修正対応はすべて **追加 commit** として残し、試行錯誤と修正経緯を後から追えるようにする。issuekit リポジトリは merge commit 運用（squash ではない）なので、commit 履歴は merge 後も価値を持つ。
