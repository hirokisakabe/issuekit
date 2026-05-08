---
name: worktree-start
description: Claude Code 専用。素の `claude` で起動した直後に、タスク説明から命名した git worktree へ `EnterWorktree` で切り替えて作業を開始する。`cwt` (dotfiles の wezterm + worktrunk + claude 起動 alias) の代替として、issue 不要なタスク起点での並列セッション立ち上げに使う。
---

# Worktree Start Skill

Claude Code が v2.1.49 で導入した `EnterWorktree` ツールを使い、起動済みセッションの cwd を新規 git worktree に切り替えて並列タスクを開始する skill。`issue-implement` が「issue 起点」の orchestrator なのに対し、本 skill は **issue 不要なタスク起点の entry point** として並ぶ。

## スコープ

- **含む**: タスク説明からのブランチ名生成 (LLM 命名 or ユーザー明示指定の受領)、`EnterWorktree` ツール呼び出しによるセッション cwd 切り替え、既存 worktree 内での no-op 判定。
- **含まない**:
  - **`wezterm cli spawn` 等の外部タブ管理**: 並列タブの起動はユーザー操作のまま。
  - **`issue-implement` への自動連鎖**: 本 skill は worktree 切り替えのみを行い、issue 駆動サイクルへの引き継ぎはユーザーが明示的に行う。
  - **Codex CLI / 他 agent 用の fallback 実装**: `EnterWorktree` は Claude Code 固有で、他 runtime には対応 primitive が存在しない。
  - **`EnterWorktree` の `path` パラメータでクリーン命名する回避策**: `worktree-` prefix 強制を許容する方針 (issue #13 スコープ外)。
  - **作成済み worktree のクリーンアップ**: `ExitWorktree` / `git worktree remove` 等は呼ばない。

## 利用タイミング

- ユーザーが「worktree でタスクを始めたい」「並列タブで別タスクを切り出したい」「cwt の代わり」のような起動指示を与えたとき。
- すでに wezterm 等で素の `claude` が起動しており、これから worktree に入りたい状況。

## Claude Code 限定であること

本 skill は **Claude Code 専用** で、Codex CLI / Cursor / Gemini など他の agent runtime では利用できない。

- `EnterWorktree` ツールは Claude Code v2.1.49 (2026-02-19) で CLI に追加された Claude Code 固有の primitive である。
- Codex CLI には worktree の概念がなく ([openai/codex#13120](https://github.com/openai/codex/issues/13120))、Codex Worktrees は Desktop app 専用機能のため CLI からは呼び出せない。
- 他 agent からは fallback 実装を提供しない（issue #13 のスコープ外）。Codex 等で動かす場合は、ユーザーが手元で `git worktree add` を実行してから当該 worktree で agent を起動する従来手順に従うこと。

## 依存

- **Claude Code v2.1.49 以上**: `EnterWorktree` ツールを必要とする。古いバージョンでは tool not found となるため、`claude --version` で確認しユーザーへアップデートを案内する。
- **`git`**: worktree 作成のために必要（`EnterWorktree` の内部で利用される）。
- **git リポジトリ内であること**: 起動 cwd が git working tree でない場合、`EnterWorktree` は失敗する。

## 入力

ユーザーから受け取るのは「これから始めるタスクの説明」一つだけ。例:

- 「Slack 連携の OAuth フローを実装したい」
- 「issue #42 のバグを直したい」
- 「dependabot PR をまとめてレビューする」

issue 番号を含む依頼であっても、本 skill は issue 本文を取りに行かず、ブランチ名生成の入力としてのみ扱う。issue 駆動の実装サイクルを回したい場合は、worktree に入った後でユーザーが `issue-implement <番号>` を呼ぶ運用とする（連鎖 trigger は行わない。「やらないこと」参照）。

## 実行手順

### 1. 既存 worktree への再進入チェック (no-op 判定)

`EnterWorktree` ツール自体が「既に worktree 内にいるセッション」からの再進入を拒否する。本 skill ではこれに依存し、**現在のセッションが worktree 内であることを検知できた場合は何もせず終了する**（呼び出し時点で no-op）。

判定は以下のいずれかで行う:

- `git rev-parse --git-common-dir` と `git rev-parse --git-dir` を比較し、異なれば worktree 内。
- もしくは `git worktree list` の現在 path がメイン working tree と異なるかを確認。

`EnterWorktree` を投機的に呼んでツール側のエラーで気付く運用は避ける。skill 側で先に判定し、ユーザーには「すでに worktree 内のため何もしません」と返す。

### 2. ブランチ名の決定

ユーザーが渡したタスク説明から、Claude Code 自身が短いブランチ名を生成する。`cwt` の `claude -p` 相当の役割を skill 内に取り込んだもの。

- 形式: 英小文字 + 数字 + ハイフン (`kebab-case`) で 3〜5 単語程度。
- 内容: タスク説明の核となる名詞・動詞を抽出し、要約。
- prefix: `EnterWorktree` 側で `worktree-` プレフィックスが強制付与される仕様 ([Claude Code worktrees docs](https://code.claude.com/docs/en/worktrees)) を許容する。skill 側でクリーンな命名のために `path` パラメータで回避することはしない（issue #13 スコープ外）。
- ユーザーから「ブランチ名は `xxx` にして」のように明示指定があった場合は LLM 命名を行わずそのまま採用する。

例:
- 入力: 「Slack 連携の OAuth フロー実装」 → `slack-oauth-flow`
- 入力: 「dependabot PR の取りまとめレビュー」 → `dependabot-roundup-review`

### 3. `EnterWorktree` の呼び出し

確定したブランチ名を `name` 引数に渡して `EnterWorktree` を呼ぶ。

```
EnterWorktree({ name: "slack-oauth-flow" })
```

成功すれば現在のセッションの cwd が新規 worktree (`worktree-slack-oauth-flow` 系のブランチ + 対応ディレクトリ) に切り替わる。以降のツール呼び出しは新 worktree 上で動作する。

### 4. 完了報告

ユーザーには以下を返す:

- 入った worktree のパス (新 cwd)
- 作成されたブランチ名 (`worktree-` prefix 込み)
- 「次は何をしますか?」の確認 (issue 番号を伴う依頼なら `issue-implement` を、汎用タスクならそのまま実装に入る、など)

## 失敗時の対応

- **`EnterWorktree` ツールが見つからない**: Claude Code が v2.1.49 未満。`claude --version` を確認し、`npm install -g @anthropic-ai/claude-code` でアップデートしてから再起動するよう案内する。
- **「Already inside a worktree」系のエラーが返る**: skill 側の事前チェックを通り抜けてしまったケース。報告のみ行い、再呼び出しは試みない（既に目的の状態にある可能性が高い）。
- **git リポジトリ外で呼ばれた**: `git rev-parse --is-inside-work-tree` で先に検知し、git repo 内で再実行するようユーザーへ案内する。
- **ブランチ名衝突**: `EnterWorktree` 側のエラー出力をそのままユーザーに見せ、別のブランチ名を提示してもらう（自動でサフィックス付与等は行わない。意図しない命名を避けるため）。

## やらないこと

- **既に worktree 内にいるセッションでの再進入**: 上記 step 1 で no-op として返す。`EnterWorktree` 自体も再進入を拒否するため、二重チェック構造で安全側に倒す。
- **`wezterm cli spawn` 等の外部タブ管理の自動化**: 並列タブの起動はユーザー操作のまま。skill から wezterm / tmux / iTerm 等を直接操作しない。
- **`issue-implement` への自動連鎖**: タスク説明に issue 番号が含まれていても、本 skill は worktree 切り替えだけを行い `issue-implement` は呼ばない。issue 起点で進めたい場合はユーザーが明示的に `issue-implement <番号>` を呼ぶ。
- **Codex CLI / 他 agent 用の fallback 実装**: 本 skill は Claude Code 専用。「Claude Code 限定であること」セクション参照。
- **`EnterWorktree` の `path` パラメータでクリーン命名を試みる**: `worktree-` prefix 強制を許容する方針 (issue #13 スコープ外)。
- **`cwt` (dotfiles) 側の削除や移行スクリプト提供**: 本 skill は新規追加のみで、dotfiles の整理はユーザーの別作業。
- **作成済み worktree のクリーンアップ**: `ExitWorktree` / `git worktree remove` 等は呼ばない。worktree のライフサイクル管理はユーザー責務。
