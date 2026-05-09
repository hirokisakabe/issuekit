---
name: cross-review
description: 実装・commit 後、`acceptance-check` 通過後・PR 作成前に別の AI (Codex CLI または Claude CLI headless) に diff を渡してセカンドオピニオンのコードレビューを得る。backend は環境変数 `CROSS_REVIEW_BACKEND` で `codex` / `claude-self` から選択でき、未指定時は利用可能な CLI を自動検出する。
---

# Cross Review Skill

実装済みの変更を「実装した agent とは別の backend」にレビューさせ、異なる視点からのフィードバックを得る。Agent Skills は agent-portable な open standard であり、本 skill も特定の Claude Code 固有 primitive (Sub-Agent / Task tool) には依存せず、外部 CLI のみで動作する。

## 利用タイミング

- 実装・commit 後、`acceptance-check` を通過した時点で PR 作成前に呼び出す（cycle 内では実装/commit → acceptance-check → cross-review → PR の順）。レビュー指摘の修正は **追加 commit** として残し、`git commit --amend` / `rebase` 等で履歴整形しない。
- ユーザーが明示的にレビューを依頼した場合

## 依存 / 互換性

- **Codex CLI**: 0.125.0 以降で動作確認済み (`brew install --cask codex`)。codex backend では `codex exec` + stdin diff pipe 方式を採用しており、`codex exec review` の `--base / --uncommitted / [PROMPT]` 三者排他 (0.125.0 以降の制約) を回避している。0.125.0 未満でも `codex exec [PROMPT]` への stdin pipe は基本機能として古くから存在するため動作する想定だが、明示的なサポート下限は 0.125.0 とする。`codex exec review` のサブコマンド固有の挙動には依存しない。
- **Claude CLI**: claude-self backend が利用する `claude --bare -p` のために必要 (`npm install -g @anthropic-ai/claude-code`)。stdin の 10MB 上限は Claude CLI v2.1.128 以降で明示的にエラー停止する (4. 分割レビューに切り替える)。
- **`gh` CLI**: default branch 解決に必要。未認証 / repo 外実行では本 skill が明示的に停止する (失敗時の対応参照)。

## Backend 選択

### 環境変数で明示指定する場合

`CROSS_REVIEW_BACKEND` で backend を選ぶ。

| 値             | 使用 CLI            | 想定ユースケース                                          |
| -------------- | ------------------- | --------------------------------------------------------- |
| `codex`        | OpenAI Codex CLI    | Claude Code から実装し、別モデル (GPT 系) に交差レビュー  |
| `claude-self`  | Claude CLI headless | Codex CLI / Cursor 等から実装し、Claude にレビュー依頼    |

### 未指定時の自動検出

`CROSS_REVIEW_BACKEND` が空 / 未設定の場合は、agent が以下の順で `command -v` を実行して backend を確定する。これは「実装した agent 自身に自分以外を選ばせる」優先順位ではなく、**利用可能なものから順に試す availability check** として機能する。

> **注意**: 自動検出は実行中の agent を判別しない。Codex CLI で実装中に両 CLI がインストールされていると、codex backend (= 同系統 backend) が選ばれて事実上「自己レビュー」になる可能性がある。「別 backend に必ず投げる」ことを保証したい場合は、ユーザー側で `CROSS_REVIEW_BACKEND=claude-self` (Codex から実装する場合) / `CROSS_REVIEW_BACKEND=codex` (Claude から実装する場合) のように明示指定する運用を推奨する。

```bash
if [ -n "${CROSS_REVIEW_BACKEND:-}" ]; then
  BACKEND="$CROSS_REVIEW_BACKEND"
elif command -v codex >/dev/null 2>&1; then
  BACKEND="codex"
elif command -v claude >/dev/null 2>&1; then
  BACKEND="claude-self"
else
  echo "cross-review: 利用可能な backend が見つかりません。codex CLI (\`brew install --cask codex\`) または Claude CLI (\`npm install -g @anthropic-ai/claude-code\`) を導入してください。" >&2
  exit 1
fi
```

明示指定された backend に対応する CLI が見つからない場合は、暗黙の fallback はせず明示的なエラーで停止する（ユーザーが意図した backend と異なる結果を返さないため）。

```bash
case "$BACKEND" in
  codex)
    command -v codex >/dev/null 2>&1 || { echo "CROSS_REVIEW_BACKEND=codex ですが \`codex\` コマンドが見つかりません。\`brew install --cask codex\` で導入してください。" >&2; exit 1; } ;;
  claude-self)
    command -v claude >/dev/null 2>&1 || { echo "CROSS_REVIEW_BACKEND=claude-self ですが \`claude\` コマンドが見つかりません。\`npm install -g @anthropic-ai/claude-code\` で導入してください。" >&2; exit 1; } ;;
  *)
    echo "cross-review: 未対応の CROSS_REVIEW_BACKEND=$BACKEND (対応値: codex / claude-self)" >&2; exit 1 ;;
esac
```

## 実行手順

### 1. base ref の確定 (backend 共通)

リポジトリの default branch 名を動的に取得し、ローカルで実際に解決可能な ref を `BASE_REF` に確定する。`BASE_REF` は両 backend が `git diff "$BASE_REF"...HEAD` の base として共通で使う。

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

### 2. 差分の確認 (backend 共通)

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

#### 3-a. backend = `codex` の場合

`codex exec` に diff を stdin から流し込み、レビュー指示を `[PROMPT]` 引数として渡す。stdin が piped されかつ `[PROMPT]` も指定された場合、codex は stdin を `<stdin>` ブロックとして prompt に append する仕様 (`codex exec --help` 参照)。`codex exec review --base / --uncommitted / [PROMPT]` の三者排他 (codex-cli 0.125.0 以降) を回避するため、`review` サブコマンドではなく汎用の `codex exec` を使う。claude-self backend と同じ「diff pipe + custom prompt」の構造に揃え、保守コストも下げている。

`--sandbox read-only` を明示することで、汎用 `codex exec` を使いながらも cross-review の「報告のみ・自動修正しない」原則を CLI レイヤーで担保する (`--full-auto` は workspace-write が付くため使わない)。

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
} | codex exec --sandbox read-only --model gpt-5.4 "You are a senior code reviewer providing a second opinion. Do not modify any files; output the review only. The diff is supplied via stdin (codex wraps it as a <stdin> block). First, read the repository's AGENTS.md (if it exists) to understand project conventions and coding standards.

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

#### 3-b. backend = `claude-self` の場合

`claude --bare -p` で Claude CLI に diff を stdin 経由で渡す。`--bare` は CI / scripted call の推奨モードで、hooks / skills / MCP / CLAUDE.md auto-discovery を skip する（cross-review が再帰的に skill chain に巻き込まれる事故を防ぎ、レビュー結果の再現性も高まる）。`AGENTS.md` を読ませるために `--allowedTools "Read"` を付与する。

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
} | claude --bare -p \
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

Output format (respond in Japanese):
- List each finding with severity: critical / warning / info
- For each finding, include: file path, line number or range, description, and a concrete fix suggestion
- If no issues found, state that the code looks good
- End with a summary table: total findings by severity"
```

> piped stdin は Claude CLI v2.1.128 以降 10MB cap がある。500 行超の差分は次の分割レビューに切り替えれば 1 ファイルあたりは十分小さくなる。

### 4. 分割レビュー (差分が大きい場合)

差分をファイル単位に分割し、ファイルごとに backend を呼ぶ。最後に全ファイルのレビュー結果を集約してサマリーを作成する。

```bash
# 変更ファイル一覧を取得（コミット済み + unstaged + staged の和集合）
git diff "$BASE_REF"...HEAD --name-only
git diff --name-only
git diff --cached --name-only
```

#### 4-a. backend = `codex`

`FILE_PATH` でファイルパスを変数化し、空白や shell メタ文字を含むファイル名でも壊れないようにする (3-a と同じく `--sandbox read-only` で書き込みを禁止)。`<file-path>` は placeholder で、実利用時は単一引用符付きで実パスに置き換える (例: `FILE_PATH='skills/cross-review/SKILL.md'`)。

```bash
FILE_PATH='<file-path>'
{
  echo "=== Diff for $FILE_PATH ==="
  git diff "$BASE_REF"...HEAD -- "$FILE_PATH"
  git diff -- "$FILE_PATH"
  git diff --cached -- "$FILE_PATH"
} | codex exec --sandbox read-only --model gpt-5.4 "You are a senior code reviewer providing a second opinion. Do not modify any files; output the review only. The diff for a single file is supplied via stdin (codex wraps it as a <stdin> block). Review the changes to $FILE_PATH.

Evaluate from: Correctness, Readability, Consistency, Security, Performance, Tests, Documentation.

Output (respond in Japanese):
- Each finding with severity (critical / warning / info), file path, line number, description, fix suggestion
- If no issues, state the file looks good"
```

#### 4-b. backend = `claude-self`

```bash
{
  echo "=== Diff for <file-path> ==="
  git diff "$BASE_REF"...HEAD -- <file-path>
  git diff -- <file-path>
  git diff --cached -- <file-path>
} | claude --bare -p \
  --allowedTools "Read" \
  --append-system-prompt "You are a senior code reviewer providing a second opinion. The diff for a single file is supplied via stdin." \
  "Review the changes to <file-path>.

Evaluate from: Correctness, Readability, Consistency, Security, Performance, Tests, Documentation.

Output (respond in Japanese):
- Each finding with severity (critical / warning / info), file path, line number, description, fix suggestion
- If no issues, state the file looks good"
```

### 5. 結果の集約と報告

backend のレビュー結果を確認し、ユーザーへ報告する。
- **critical** の指摘がある場合: 修正案を提示し、ユーザーに対応方針を確認する
- **warning** の指摘がある場合: agent 自身で対応要否を判断する。妥当な指摘は自律的に修正し、見送る場合は理由を添えて報告する（ユーザー確認は不要）
- **info** のみの場合: 指摘を共有し、PR 作成に進む

## 失敗時の対応

- **backend CLI が見つからない場合**: `command -v` 結果と `CROSS_REVIEW_BACKEND` の値を踏まえて明示的に停止する。`codex` 未導入なら `brew install --cask codex` を、`claude` 未導入なら `npm install -g @anthropic-ai/claude-code` を案内する。Codex CLI 初回利用時は `codex login` で OpenAI アカウント認証が必要。Claude CLI 初回利用時は `claude` を一度起動して認証する。
- **default branch の取得失敗時**: `gh repo view --json defaultBranchRef --jq '.defaultBranchRef.name'` が空文字を返す、もしくは `gh` がエラーを返した場合は、その時点で停止しエラーメッセージを出す。`main` への暗黙フォールバックは行わない（誤った base に対する diff でレビュー結果が破綻するため）。よくある原因は、`gh` 未認証 (`gh auth status` で確認) / git repo 外での実行 / リモートが GitHub 以外。原因を解消してから再実行する。
- **base ref の resolve 失敗時**: `BASE_BRANCH` 名は取れたが、ローカルに該当 ref も `origin/$BASE_BRANCH` も存在しない場合（例: 浅い clone / default branch を local 側で削除した worktree 等）も停止する。`git fetch origin` で remote-tracking ref を取得すれば多くの場合解消する。
- **codex backend で `codex exec review --base ... [PROMPT]` 系のエラーに遭遇した場合**: 古い呼び出し方式が残ったローカル環境の可能性が高い。本 skill は 0.125.0 の三者排他制約を踏まえて `codex exec` + stdin diff pipe に移行済み。SKILL.md を最新版に更新するか、shell history に残った古いコマンドを破棄する。
- **差分がない場合**: レビュー不要としてスキップする。
- **backend がタイムアウトした場合**: 差分を分割して再試行する（ステップ 4）。
- **claude-self backend で stdin が 10MB を超える場合**: Claude CLI が明示的にエラーで停止するので、ステップ 4 の分割レビューに切り替える。

## やらないこと

- 暗黙の backend fallback（明示指定された backend が利用不可な場合に勝手に他の backend に切り替えること）。ユーザー意図と異なるレビュー結果を返すことを避けるため。
- レビュー結果の自動修正適用（報告のみ）。
- backend 出力フォーマットの統一（各 backend の出力をそのまま使う）。
- 追加 backend (Gemini / OpenAI 直 API 等) の実装。構造を残しつつ別 issue 化。
- codex-cli 0.124 系以前への downgrade 案内。upstream の意思決定に追従しない一時しのぎになり、依存 CLI のバージョン分岐が発散するため採らない (`codex exec` + stdin diff pipe で 0.125.0 以降を正面突破する)。
