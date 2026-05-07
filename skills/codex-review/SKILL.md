---
name: codex-review
description: 実装完了後、PR作成前に Codex CLI へコードレビューを委譲する。Claude Code が自身の実装に対してセカンドオピニオンを得るために使用する。
---

# Codex Review Skill

実装済みの変更を Codex CLI にレビューさせ、異なる視点からのフィードバックを得る。

## 利用タイミング

- Claude Code による実装が完了した後、PR 作成前に呼び出す
- ユーザーが明示的にレビューを依頼した場合

## 実行手順

1. リポジトリの default branch 名を動的に取得し、ローカルで実際に解決可能な ref を `BASE_REF` に確定する。`BASE_BRANCH` は `codex exec review --base` 用、`BASE_REF` は `git diff` 用に使い分ける。

```bash
BASE_BRANCH=$(gh repo view --json defaultBranchRef --jq '.defaultBranchRef.name')
[ -n "$BASE_BRANCH" ] || { echo "default branch を取得できませんでした。gh の認証状態 / repo 内での実行か確認してください。" >&2; exit 1; }

# git diff 用の base ref: local ref → origin/<branch> の順で探す
if git rev-parse --verify --quiet "$BASE_BRANCH" >/dev/null; then
  BASE_REF="$BASE_BRANCH"
elif git rev-parse --verify --quiet "origin/$BASE_BRANCH" >/dev/null; then
  BASE_REF="origin/$BASE_BRANCH"
else
  echo "'$BASE_BRANCH' も 'origin/$BASE_BRANCH' も resolve できません。git fetch origin を実行してから再試行してください。" >&2
  exit 1
fi
```

`master` / `develop` / `trunk` を default branch とするリポジトリでも、この `BASE_BRANCH` / `BASE_REF` 経由で同じ手順がそのまま動く（例: `BASE_BRANCH=master`, `BASE_REF=master` または `BASE_REF=origin/master` → `git diff "$BASE_REF"...HEAD` / `--base master`）。`main` をデフォルトにフォールバックさせない（後述「失敗時の対応」参照）。

2. 現在のブランチの差分を確認する（コミット済み + 未コミットの両方）。

```bash
# コミット済みの変更（ブランチの差分）
git diff "$BASE_REF"...HEAD --stat
git diff "$BASE_REF"...HEAD

# 未コミットの変更がある場合（unstaged + staged）
git diff --stat
git diff
git diff --cached --stat
git diff --cached
```

3. 差分のファイル数と行数を確認し、レビュー方法を決定する。
   - **差分が 500 行以下**: 一括レビュー（ステップ 4 へ）
   - **差分が 500 行超**: ファイル単位で分割レビュー（ステップ 5 へ）

4. 一括レビュー: 差分の概要を整理し、`codex exec review` でレビューを委譲する。

```bash
codex exec review --base "$BASE_BRANCH" --uncommitted --model gpt-5.4 --full-auto "First, read the repository's AGENTS.md (if it exists) to understand project conventions and coding standards.

Then evaluate the diff from these perspectives:

1. **Correctness**: Are there logic errors, edge cases, or incorrect assumptions?
2. **Readability**: Are names clear and intent obvious? Is the code self-documenting?
3. **Consistency**: Do the changes follow existing patterns, naming conventions, and style in the codebase?
4. **Security**: Is there proper input validation? Are secrets or credentials exposed?
5. **Performance**: Are there unnecessary computations or inefficient patterns?
6. **Tests**: If the project has tests, is coverage adequate for the changes?
7. **Documentation**: Are related docs (README, CLAUDE.md, AGENTS.md, inline comments, etc.) updated to reflect the changes? Flag missing or outdated documentation.

Output format (respond in Japanese):
- List each finding with severity: critical / warning / info
- For each finding, include: file path, line number or range, description, and a concrete fix suggestion
- If no issues found, state that the code looks good
- End with a summary table: total findings by severity"
```

5. 分割レビュー: 差分が大きい場合はファイル単位で分割してレビューする。

```bash
# 変更ファイル一覧を取得（コミット済み + unstaged + staged の和集合）
git diff "$BASE_REF"...HEAD --name-only
git diff --name-only
git diff --cached --name-only

# ファイルごとに個別レビューを実行
codex exec review --base "$BASE_BRANCH" --uncommitted --model gpt-5.4 --full-auto "Review the changes to <file-path>.

Evaluate from: Correctness, Readability, Consistency, Security, Performance, Tests, Documentation.

Output (respond in Japanese):
- Each finding with severity (critical / warning / info), file path, line number, description, fix suggestion
- If no issues, state the file looks good"
```

最後に全ファイルのレビュー結果を集約してサマリーを作成する。

6. Codex のレビュー結果を確認し、ユーザーへ報告する。
   - **critical** の指摘がある場合: 修正案を提示し、ユーザーに対応方針を確認する
   - **warning** の指摘がある場合: Claude 自身で対応要否を判断する。妥当な指摘は自律的に修正し、見送る場合は理由を添えて報告する（ユーザー確認は不要）
   - **info** のみの場合: 指摘を共有し、PR 作成に進む

## 失敗時の対応

- `codex` コマンドが見つからない場合は `command -v codex` で確認し、未導入であれば `brew install --cask codex` でのインストールを案内する。初回利用時は `codex login` で OpenAI アカウントでの認証が必要。
- **default branch の取得失敗時**: `gh repo view --json defaultBranchRef --jq '.defaultBranchRef.name'` が空文字を返す、もしくは `gh` がエラーを返した場合は、その時点で停止しエラーメッセージを出す。`main` への暗黙フォールバックは行わない（誤った base に対する diff / `--base` 指定でレビュー結果が破綻するため）。よくある原因は、`gh` 未認証 (`gh auth status` で確認) / git repo 外での実行 / リモートが GitHub 以外。原因を解消してから再実行する。
- **base ref の resolve 失敗時**: `BASE_BRANCH` 名は取れたが、ローカルに該当 ref も `origin/$BASE_BRANCH` も存在しない場合（例: 浅い clone / default branch を local 側で削除した worktree 等）も停止する。`git fetch origin` で remote-tracking ref を取得すれば多くの場合解消する。
- 差分がない場合はレビュー不要としてスキップする。
- Codex CLI がタイムアウトした場合は、差分を分割して再試行する。
