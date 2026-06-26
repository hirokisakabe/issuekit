---
name: cross-review
description: 実装・commit 後、`acceptance-check` 通過後・PR 作成前に、実装セッションから独立した reviewer session を実行中 agent runtime に対応する CLI で起動し、diff への second opinion を得る。
version: 1.0.5
---

# Cross Review Skill

実装済みの変更を、実装セッションから独立した **reviewer session** に渡して second opinion を得る。目的は「別 backend / 別モデルであること」の機械的保証ではなく、実装時の会話文脈・自己正当化・途中判断から切り離したレビュー専用セッションを作ること。

Agent Skills は agent-portable な open standard であり、本 skill は特定の Claude Code 固有 primitive (Sub-Agent / Task tool) には依存しない。実行中 agent runtime に対応する外部 CLI を使い、Codex 利用時は Codex CLI、Claude Code 利用時は Claude CLI だけで完結させる。

## 利用タイミング

- 実装・commit 後、`acceptance-check` を通過した時点で PR 作成前に呼び出す（cycle 内では実装/commit → acceptance-check → cross-review → PR の順）。レビュー指摘の修正は **追加 commit** として残し、`git commit --amend` / `rebase` 等で履歴整形しない。
- ユーザーが明示的にレビューを依頼した場合。

## 依存 / 互換性

- **Codex CLI**: Codex runtime で本 skill を使う場合に必要 (`brew install --cask codex`)。`codex exec` + stdin diff pipe 方式で独立 reviewer session を起動する。`codex exec review` のサブコマンド固有の挙動には依存しない。
- **Claude CLI**: Claude Code runtime で本 skill を使う場合に必要 (`npm install -g @anthropic-ai/claude-code`)。`claude -p` で独立 reviewer session を起動する。stdin の 10MB 上限に当たる場合は 4. 分割レビューに切り替える。
- **`gh` CLI**: default branch 解決に必要。未認証 / repo 外実行では本 skill が明示的に停止する（失敗時の対応参照）。

## Reviewer Session の起動方針

実行中の agent runtime に対応する CLI で reviewer session を起動する。backend の自動検出や環境変数 override による切り替えは扱わない。runtime は実行中 agent が自分の環境情報として明示的に把握している値だけで判定し、`command -v codex` / `command -v claude` の存在順から推測しない。

| 実装中の runtime | 使用 CLI | 起動方法 |
| ---------------- | -------- | -------- |
| Codex CLI | Codex CLI | `codex exec --sandbox read-only` |
| Claude Code | Claude CLI | `claude -p --allowedTools "Read"` |

この対応は「同じ製品ファミリーの CLI で別セッションを起動する」ためのものであり、別モデルレビューを保証するものではない。別 backend / 別モデルレビューが必要な場合は、本 skill の責務として残さず別 issue で設計する。

## 実行手順

### 0. runtime と CLI の事前確認

実行中 agent runtime を明示的に判定する。Codex CLI で実装している場合は Codex CLI、Claude Code で実装している場合は Claude CLI を使う。判定できない場合、または Cursor / Gemini など本 skill に手順が定義されていない runtime の場合は、別 CLI へ自動的に切り替えず停止する。

runtime 判定後、対応 CLI の存在だけを確認する。以下はいずれか一方だけを実行し、両方を連続実行しない。

```bash
# Codex CLI で実装している場合
command -v codex >/dev/null 2>&1 || { printf '%s\n' 'Codex CLI runtime ですが `codex` コマンドが見つかりません。`brew install --cask codex` で導入してください。' >&2; exit 1; }
```

```bash
# Claude Code で実装している場合
command -v claude >/dev/null 2>&1 || { printf '%s\n' 'Claude Code runtime ですが `claude` コマンドが見つかりません。`npm install -g @anthropic-ai/claude-code` で導入してください。' >&2; exit 1; }
```

### 1. base ref の確定

リポジトリの default branch 名を動的に取得し、ローカルで実際に解決可能な ref を `BASE_REF` に確定する。`BASE_REF` は `git diff "$BASE_REF"...HEAD` の base として使う。

```bash
BASE_BRANCH=$(gh repo view --json defaultBranchRef --jq '.defaultBranchRef.name')
[ -n "$BASE_BRANCH" ] || { echo "default branch を取得できませんでした。gh の認証状態 / repo 内での実行か確認してください。" >&2; exit 1; }

if git rev-parse --verify --quiet "$BASE_BRANCH" >/dev/null; then
  BASE_REF="$BASE_BRANCH"
elif git rev-parse --verify --quiet "origin/$BASE_BRANCH" >/dev/null; then
  BASE_REF="origin/$BASE_BRANCH"
else
  echo "'$BASE_BRANCH' も 'origin/$BASE_BRANCH' も resolve できません。git fetch origin を実行してから再試行してください。" >&2
  exit 1
fi
```

`master` / `develop` / `trunk` を default branch とするリポジトリでも、この `BASE_REF` 経由で同じ手順がそのまま動く。`main` への暗黙フォールバックは行わない（後述「失敗時の対応」参照）。

### 2. 差分の確認

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

差分のファイル数と行数を確認し、レビュー方法を決定する。

- **差分が 500 行以下**: 一括レビュー（ステップ 3 へ）
- **差分が 500 行超**: ファイル単位で分割レビュー（ステップ 4 へ）
- **差分が 0 行**: レビュー不要としてスキップする

### 3. 一括レビュー

#### 3-a. Codex CLI で実行中の場合

`codex exec` に diff を stdin から流し込み、レビュー指示を `[PROMPT]` 引数として渡す。stdin が piped されかつ `[PROMPT]` も指定された場合、codex は stdin を `<stdin>` ブロックとして prompt に append する仕様 (`codex exec --help` 参照)。`codex exec review` のサブコマンド固有の挙動には依存しない。

`--sandbox read-only` を明示することで、汎用 `codex exec` を使いながらも cross-review の「報告のみ・自動修正しない」原則を CLI レイヤーで担保する（`--full-auto` は workspace-write が付くため使わない）。

```bash
{
  echo "=== Committed diff ($BASE_REF...HEAD) ==="
  git diff "$BASE_REF"...HEAD
  echo
  echo "=== Unstaged diff ==="
  git diff
  echo
  echo "=== Staged diff ==="
  git diff --cached
} | codex exec --sandbox read-only "You are a senior code reviewer providing a second opinion. Do not modify any files; output the review only. The diff is supplied via stdin (codex wraps it as a <stdin> block). First, read the repository's AGENTS.md (if it exists) to understand project conventions and coding standards.

Then evaluate the diff from these perspectives:

1. **Correctness**: Are there logic errors, edge cases, or incorrect assumptions?
2. **Readability**: Are names clear and intent obvious? Is the code self-documenting?
3. **Consistency**: Do the changes follow existing patterns, naming conventions, and style in the codebase?
4. **Security**: Is there proper input validation? Are secrets or credentials exposed?
5. **Performance**: Are there unnecessary computations or inefficient patterns?
6. **Tests**: If the project has tests, is coverage adequate for the changes?
7. **Documentation**: Are related docs (README, CLAUDE.md, AGENTS.md, inline comments, etc.) updated to reflect the changes? Flag missing or outdated documentation.
8. **Related-file consistency**: When a file changes, are sibling/peer files that must stay in sync also updated? Examples: cross-references between skills, generated files, lock files, schema and its migration, config and its documentation. Flag any consistency gap where one side of a known pair was changed but the other was not.

Output format (respond in Japanese):
- List each finding with severity: critical / warning / info
- For each finding, include: file path, line number or range, description, and a concrete fix suggestion
- If no issues found, state that the code looks good
- End with a summary table: total findings by severity"
```

#### 3-b. Claude Code で実行中の場合

`claude -p` で Claude CLI に diff を stdin 経由で渡す。`--bare` は OAuth / keychain のログイン状態を読まず `ANTHROPIC_API_KEY` または `--settings` の `apiKeyHelper` 前提になるため、ローカルの Claude.ai ログイン運用でも動くように使わない。`AGENTS.md` を読ませるために `--allowedTools "Read"` を付与する。

```bash
{
  echo "=== Committed diff ($BASE_REF...HEAD) ==="
  git diff "$BASE_REF"...HEAD
  echo
  echo "=== Unstaged diff ==="
  git diff
  echo
  echo "=== Staged diff ==="
  git diff --cached
} | claude -p \
  --allowedTools "Read" \
  --append-system-prompt "You are a senior code reviewer providing a second opinion. The diff is supplied via stdin. First, read the repository's AGENTS.md (if it exists) to understand project conventions and coding standards." \
  "Evaluate the diff from these perspectives:

1. **Correctness**: Are there logic errors, edge cases, or incorrect assumptions?
2. **Readability**: Are names clear and intent obvious? Is the code self-documenting?
3. **Consistency**: Do the changes follow existing patterns, naming conventions, and style in the codebase?
4. **Security**: Is there proper input validation? Are secrets or credentials exposed?
5. **Performance**: Are there unnecessary computations or inefficient patterns?
6. **Tests**: If the project has tests, is coverage adequate for the changes?
7. **Documentation**: Are related docs (README, CLAUDE.md, AGENTS.md, inline comments, etc.) updated to reflect the changes? Flag missing or outdated documentation.
8. **Related-file consistency**: When a file changes, are sibling/peer files that must stay in sync also updated? Examples: cross-references between skills, generated files, lock files, schema and its migration, config and its documentation. Flag any consistency gap where one side of a known pair was changed but the other was not.

Output format (respond in Japanese):
- List each finding with severity: critical / warning / info
- For each finding, include: file path, line number or range, description, and a concrete fix suggestion
- If no issues found, state that the code looks good
- End with a summary table: total findings by severity"
```

### 4. 分割レビュー（差分が大きい場合）

差分をファイル単位に分割し、ファイルごとに reviewer session を呼ぶ。最後に全ファイルのレビュー結果を集約してサマリーを作成する。

```bash
# 変更ファイル一覧を取得（コミット済み + unstaged + staged の和集合）
git diff "$BASE_REF"...HEAD --name-only
git diff --name-only
git diff --cached --name-only
```

#### 4-a. Codex CLI で実行中の場合

`FILE_PATH` でファイルパスを変数化し、空白や shell メタ文字を含むファイル名でも壊れないようにする（3-a と同じく `--sandbox read-only` で書き込みを禁止）。`<file-path>` は placeholder で、実利用時は単一引用符付きで実パスに置き換える（例: `FILE_PATH='skills/cross-review/SKILL.md'`）。

```bash
FILE_PATH='<file-path>'
{
  echo "=== Diff for $FILE_PATH ==="
  git diff "$BASE_REF"...HEAD -- "$FILE_PATH"
  git diff -- "$FILE_PATH"
  git diff --cached -- "$FILE_PATH"
} | codex exec --sandbox read-only "You are a senior code reviewer providing a second opinion. Do not modify any files; output the review only. The diff for a single file is supplied via stdin (codex wraps it as a <stdin> block). Review the changes to $FILE_PATH.

Evaluate from: Correctness, Readability, Consistency, Security, Performance, Tests, Documentation, Related-file consistency.

Output (respond in Japanese):
- Each finding with severity (critical / warning / info), file path, line number, description, fix suggestion
- If no issues, state the file looks good"
```

#### 4-b. Claude Code で実行中の場合

```bash
FILE_PATH='<file-path>'
{
  echo "=== Diff for $FILE_PATH ==="
  git diff "$BASE_REF"...HEAD -- "$FILE_PATH"
  git diff -- "$FILE_PATH"
  git diff --cached -- "$FILE_PATH"
} | claude -p \
  --allowedTools "Read" \
  --append-system-prompt "You are a senior code reviewer providing a second opinion. The diff for a single file is supplied via stdin." \
  "Review the changes to $FILE_PATH.

Evaluate from: Correctness, Readability, Consistency, Security, Performance, Tests, Documentation, Related-file consistency.

Output (respond in Japanese):
- Each finding with severity (critical / warning / info), file path, line number, description, fix suggestion
- If no issues, state the file looks good"
```

### 5. 結果の集約と報告

reviewer session の結果を確認し、ユーザーへ報告する。

- **critical** の指摘がある場合: 修正案を提示し、ユーザーに対応方針を確認する
- **warning** の指摘がある場合: agent 自身で対応要否を判断する。妥当な指摘は自律的に修正し、見送る場合は理由を添えて報告する（ユーザー確認は不要）
- **info** のみの場合: 指摘を共有し、PR 作成に進む

## 失敗時の対応

- **実行中 runtime に対応する CLI が見つからない場合**: Codex CLI で実装しているなら `codex`、Claude Code で実装しているなら `claude` が必要。該当 CLI が無ければ明示的に停止し、Codex CLI は `brew install --cask codex`、Claude CLI は `npm install -g @anthropic-ai/claude-code` を案内する。Codex CLI 初回利用時は `codex login` で OpenAI アカウント認証が必要。Claude CLI 初回利用時は `claude` を一度起動して認証する。
- **実行中 runtime を判定できない場合**: 自動検出で別 CLI へ切り替えず停止する。Codex CLI / Claude Code 以外の runtime 向け手順は別 issue で扱う。
- **default branch の取得失敗時**: `gh repo view --json defaultBranchRef --jq '.defaultBranchRef.name'` が空文字を返す、もしくは `gh` がエラーを返した場合は、その時点で停止しエラーメッセージを出す。`main` への暗黙フォールバックは行わない（誤った base に対する diff でレビュー結果が破綻するため）。よくある原因は、`gh` 未認証 (`gh auth status` で確認) / git repo 外での実行 / リモートが GitHub 以外。原因を解消してから再実行する。
- **base ref の resolve 失敗時**: `BASE_BRANCH` 名は取れたが、ローカルに該当 ref も `origin/$BASE_BRANCH` も存在しない場合（例: 浅い clone / default branch を local 側で削除した worktree 等）も停止する。`git fetch origin` で remote-tracking ref を取得すれば多くの場合解消する。
- **Codex CLI で `codex exec review --base ... [PROMPT]` 系のエラーに遭遇した場合**: 古い呼び出し方式が残ったローカル環境の可能性が高い。本 skill は `codex exec` + stdin diff pipe を使う。SKILL.md を最新版に更新するか、shell history に残った古いコマンドを破棄する。
- **差分がない場合**: レビュー不要としてスキップする。
- **reviewer session がタイムアウトした場合**: 差分を分割して再試行する（ステップ 4）。
- **Claude CLI で stdin が 10MB を超える場合**: Claude CLI が明示的にエラーで停止するので、ステップ 4 の分割レビューに切り替える。

## やらないこと

- backend / CLI の自動検出や環境変数 override による切り替え。実行中 runtime に対応する CLI で独立 reviewer session を起動する。
- 別 backend / 別モデルレビューの保証。必要なら別 issue として扱う。
- レビュー結果の自動修正適用（報告のみ）。
- reviewer session の出力フォーマットの統一（各 CLI の出力をそのまま使う）。
- 追加 backend (Gemini / OpenAI 直 API 等) の実装。構造を残しつつ別 issue 化。
- codex-cli 0.124 系以前への downgrade 案内。upstream の意思決定に追従しない一時しのぎになり、依存 CLI のバージョン分岐が発散するため採らない（`codex exec` + stdin diff pipe で正面突破する）。
