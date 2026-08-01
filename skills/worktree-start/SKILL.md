---
name: worktree-start
description: "Claude Code 専用。タスク説明または issue URL / 番号から命名した git worktree へ `EnterWorktree` で切り替える。Ready issue は完了形を判定し、PR なら `issue-implement`、コメント完結型なら `issue-investigate` へ連鎖し、要確認なら `issue-refine` を案内する。"
version: 1.1.0
---

# Worktree Start Skill

Claude Code が v2.1.49 で導入した `EnterWorktree` ツールを使い、起動済みセッションの cwd を新規 git worktree に切り替えて並列タスクを開始する skill。`issue-implement` は PR 完結、`issue-investigate` はコメント完結の orchestrator であり、本 skill は **issue 起点・タスク起点どちらでも入れる entry point** として完了形に応じた経路へつなぐ。

## スコープ

- **含む**: タスク説明 / issue URL / issue 番号からのブランチ名生成、`EnterWorktree` による cwd 切り替え、既存 worktree 内での no-op 判定、issue 入力時の Status・コメント・完了形の確認、着手可能な Ready issue の適切な orchestrator への引き継ぎ。
- **含まない**:
  - **外部タブ管理ツール (ターミナルマルチプレクサ等) との連携**: 並列タブの起動はユーザー操作のまま。
  - **Codex CLI / 他 agent 用の fallback 実装**: `EnterWorktree` は Claude Code 固有で、他 runtime には対応 primitive が存在しない。
  - **`EnterWorktree` の `path` パラメータでクリーン命名する回避策**: `worktree-` prefix 強制を許容する方針 (issue #13 スコープ外)。
  - **作成済み worktree のクリーンアップ**: `ExitWorktree` / `git worktree remove` 等は呼ばない。
  - **Draft / フォーマット不完全 / 完了形が要確認な issue の着手連鎖**: worktree 作成のみ行い `issue-refine` を案内する。

## 利用タイミング

- ユーザーが「worktree でタスクを始めたい」「並列タブで別タスクを切り出したい」のような起動指示を与えたとき。
- ユーザーが **issue URL** (`https://github.com/<owner>/<repo>/issues/<N>`) または **issue 番号** をセッション冒頭に貼り、新規 worktree で着手したいとき。
- すでにターミナルエミュレータで素の `claude` が起動しており、これから worktree に入りたい状況。
- `issuekit:issue-implement` skill (APM plain-skill mode では `issue-implement`) の冒頭ステップから呼ばれたとき。この場合は **Status / Depends on / 親 issue の確認は上流で完了済み**であり、本 skill 側では入力を「タスク説明モード」(後述 step 2) として扱って worktree 切り替え機能のみを提供する。詳細は後述「上流 skill (`issue-implement`) からの呼び出し」を参照。

## Claude Code 限定であること

本 skill は **Claude Code 専用** で、Codex CLI / Cursor / Gemini など他の agent runtime では利用できない。

- `EnterWorktree` ツールは Claude Code v2.1.49 (2026-02-19) で CLI に追加された Claude Code 固有の primitive である。
- Codex CLI には worktree の概念がなく ([openai/codex#13120](https://github.com/openai/codex/issues/13120))、Codex Worktrees は Desktop app 専用機能のため CLI からは呼び出せない。
- 他 agent からは fallback 実装を提供しない（issue #13 のスコープ外）。Codex 等で動かす場合は、ユーザーが手元で `git worktree add` を実行してから当該 worktree で agent を起動する従来手順に従うこと。

## 上流 skill (`issue-implement`) からの呼び出し

本 skill は単独 entry point として呼ばれるほか、`issuekit:issue-implement` skill の冒頭ステップ (issue-implement の step 4) から呼ばれることがある。後者の場合、上流が以下を **既に確認済み**であることを前提に動作する:

- 対象 issue の `Status: Ready` と、コメント上に未解決 blocker / 本文矛盾 / 方針保留 / 受け入れ条件の未反映変更がないこと
- `Depends on:` がすべて close 済み
- 親 issue の文脈取り込み

そのため `issue-implement` から呼ばれた際は、本 skill 側で再度 `gh issue view` による Status / Depends on / 親 issue の検証は行わない（**Status チェックは上流に委譲**）。具体的には、上流から渡されるのは **issue 番号ではなく事前生成済みのブランチ名 slug** (`<title-slug>-<issue 番号>` 形式) のみであり、本 skill はそれを step 2 の「タスク説明モード」と同じ経路で扱う。issue 番号を伴う入力経路 (Status・完了形判定 → 対応 orchestrator への連鎖) には入らないため、`issue-implement → worktree-start → issue-implement` の再帰連鎖は発生しない。

万一上流が誤って issue 番号を渡してしまった場合、step 2 で完了形を再判定して対応 orchestrator を余分に呼ぶことになる。PR 経路で `issue-implement` が再度本 skill を呼び出しても、step 1 の no-op 判定で循環は止まる。とはいえ無駄な再呼び出しと誤ルーティングを避けるため、上流からは必ずブランチ名 slug のみを渡す運用にすること (詳細は `issue-implement` の "やらないこと" 節を参照)。

## 依存

- **Claude Code v2.1.49 以上**: `EnterWorktree` ツールを必要とする。古いバージョンでは tool not found となるため、`claude --version` で確認しユーザーへアップデートを案内する。
- **`git`**: worktree 作成のために必要（`EnterWorktree` の内部で利用される）。
- **git リポジトリ内であること**: 起動 cwd が git working tree でない場合、`EnterWorktree` は失敗する。

## 入力

ユーザーから受け取るのは以下のいずれか:

- **タスク説明** (issue 不要): 「Slack 連携の OAuth フローを実装したい」「dependabot PR をまとめてレビューする」など。
- **issue URL**: `https://github.com/<owner>/<repo>/issues/<N>` 形式。
- **issue 番号**: `#42` / `42` 単体。

issue URL / 番号が渡された場合は、本 skill 側で `gh issue view --comments` を呼び出して title (ブランチ名生成用)、Status と完了形 (連鎖判定用)、コメントの補足文脈を取得する。**Status と完了形は `issue-create` の定義に従う**。

## 実行手順

### 1. 既存 worktree への再進入チェック (no-op 判定)

`EnterWorktree` ツール自体が「既に worktree 内にいるセッション」からの再進入を拒否する。本 skill ではこれに依存し、**現在のセッションが worktree 内であることを検知できた場合は何もせず終了する**（呼び出し時点で no-op）。

判定は以下のいずれかで行う:

- `git rev-parse --git-common-dir` と `git rev-parse --git-dir` を比較し、異なれば worktree 内。
- もしくは `git worktree list` の現在 path がメイン working tree と異なるかを確認。

`EnterWorktree` を投機的に呼んでツール側のエラーで気付く運用は避ける。skill 側で先に判定し、ユーザーには「すでに worktree 内のため何もしません」と返す。

### 2. 入力タイプ、Status、完了形の判定

ユーザー入力を以下に分類する:

- **タスク説明 (issue なし)**: そのまま step 3 のブランチ名生成へ進む。issue 連鎖は行わない (step 5 はスキップ)。`issue-implement` 上流から呼ばれた場合 (= 事前生成済みブランチ名 slug が渡される) もこの経路で扱い、Status 判定や `gh issue view` は走らせない。
- **issue URL / 番号**: URL から番号を抽出し、`gh issue view <N> --comments` で本文、title、コメントを取得する。本文先頭の `Status:` を確認し、コメントに本文未反映の補足や矛盾がないかも確認して以下に分岐する。本文とコメントが矛盾する場合は、`updatedAt` やコメント時系列を踏まえて最新の意図を推定し、判断できないものだけを要確認として扱う:
  - **`Status: Ready` かつコメントに未解決事項がない**: 受け入れ条件と `## スコープ外` を `issue-create` の「成果物と完了形」に照らす。repo 変更が完了条件なら **PR**、結果コメントだけなら **issue コメント**、両方または判別不能なら **要確認** とする。
  - **`Status: Ready` だがコメントに未解決 blocker / 本文との矛盾 / 方針保留 / 受け入れ条件の未反映変更がある**: 完了形にかかわらず着手連鎖を行わず、ユーザー確認または `issue-refine` を案内する。
  - **`Status: Draft`**: 着手連鎖を行わず `issuekit:issue-refine`（APM plain-skill mode では `issue-refine`）を案内する。
  - **`Status:` 表記なし / フォーマット不完全**: 同様に連鎖せず、`issue-refine` を案内する。

### 3. ブランチ名の決定

- **タスク説明から**: Claude Code 自身が短い slug を生成する。形式は英小文字 + 数字 + ハイフン (`kebab-case`) で 3〜5 単語程度。タスク説明の核となる名詞・動詞を抽出して要約する。
  - 例: 「Slack 連携の OAuth フロー実装」 → `slack-oauth-flow`
  - 例: 「dependabot PR の取りまとめレビュー」 → `dependabot-roundup-review`
- **issue 入力から**: issue title から同形式の slug を生成し、末尾に `-<issue 番号>` を付与する。issue とブランチを後から照合できるようにする狙い。
  - 例: issue #42「Slack 連携の OAuth フロー」 → `slack-oauth-flow-42`
- **ユーザー明示指定**: 「ブランチ名は `xxx` にして」と渡された場合は LLM 命名を行わずそのまま採用する。
- prefix: `EnterWorktree` 側で `worktree-` プレフィックスが強制付与される仕様 ([Claude Code worktrees docs](https://code.claude.com/docs/en/worktrees)) を許容する。skill 側でクリーンな命名のために `path` パラメータで回避することはしない（issue #13 スコープ外）。

### 4. `EnterWorktree` の呼び出し

確定したブランチ名を `name` 引数に渡して `EnterWorktree` を呼ぶ。

```
EnterWorktree({ name: "slack-oauth-flow-42" })
```

成功すれば現在のセッションの cwd が新規 worktree (`worktree-slack-oauth-flow-42` 系のブランチ + 対応ディレクトリ) に切り替わる。以降のツール呼び出しは新 worktree 上で動作する。

### 5. (issue Ready 時のみ) 完了形に応じた引き継ぎ

step 2 で **issue URL / 番号 + `Status: Ready` + コメント上の未解決事項なし** だった場合、完了形に応じて分岐する。

- **PR**: `issuekit:issue-implement <N>`（APM plain-skill mode では `issue-implement <N>`）へ引き継ぐ。
- **issue コメント**: `issuekit:issue-investigate <N>`（APM plain-skill mode では `issue-investigate <N>`）へ引き継ぐ。
- **要確認**: 自動連鎖せず `issuekit:issue-refine <N>`（APM plain-skill mode では `issue-refine <N>`）を案内する。

- PR または issue コメントと判定できた Ready issue は、引き継ぎ前にユーザー確認を挟まない。Ready は受け入れ条件が確定したシグナルであり、完了形が要確認なら `issue-refine` へ戻す。
- 連鎖後の Status / Depends on / 親 issue 再確認と各 cycle の完了処理は呼び出し先 orchestrator の責務。本 skill は worktree 切り替えと引き継ぎだけを行う。

それ以外 (タスク説明 / Draft / 表記なし / 完了形が要確認) は着手 orchestrator へ引き継がず、step 6 の完了報告のみで終わる。

### 6. 完了報告

ユーザーには以下を返す:

- 入った worktree のパス (新 cwd) と作成されたブランチ名 (`worktree-` prefix 込み)。
- **issue Ready で連鎖した場合**: 判定した完了形と、`issue-implement <N>` または `issue-investigate <N>` を起動済みであること。
- **issue Ready だが完了形が要確認の場合**: 自動連鎖せず、`issue-refine` で完了形を整理する案内。worktree は既に作成済み。
- **issue Ready だがコメント上の要確認点があり連鎖しなかった場合**: 要確認点を列挙し、ユーザー確認または `issue-refine` で整理してから再度呼ぶ案内。worktree は既に作成済み。
- **issue Draft / 表記なしの場合**: `issue-refine` で整理してから再度呼ぶ案内。worktree は既に作成済み。
- **タスク説明の場合**: 「次は何をしますか?」の確認 (そのまま実装に入る、別 skill を呼ぶ、など)。

## 失敗時の対応

- **`EnterWorktree` ツールが見つからない**: Claude Code が v2.1.49 未満。`claude --version` を確認し、`npm install -g @anthropic-ai/claude-code` でアップデートしてから再起動するよう案内する。
- **「Already inside a worktree」系のエラーが返る**: skill 側の事前チェックを通り抜けてしまったケース。報告のみ行い、再呼び出しは試みない（既に目的の状態にある可能性が高い）。
- **git リポジトリ外で呼ばれた**: `git rev-parse --is-inside-work-tree` で先に検知し、git repo 内で再実行するようユーザーへ案内する。
- **ブランチ名衝突**: `EnterWorktree` 側のエラー出力をそのままユーザーに見せ、別のブランチ名を提示してもらう（自動でサフィックス付与等は行わない。意図しない命名を避けるため）。

## やらないこと

- **既に worktree 内にいるセッションでの再進入**: 上記 step 1 で no-op として返す。`EnterWorktree` 自体も再進入を拒否するため、二重チェック構造で安全側に倒す。
- **外部タブ / ペインの自動起動**: 並列タブの起動はユーザー操作のまま。skill から外部のターミナルマルチプレクサ等を直接操作しない。
- **Draft / フォーマット不完全 / 完了形が要確認な issue の着手連鎖**: worktree 作成までで止め、`issue-refine` を案内する。Status と完了形は `issue-create` の定義に従う。
- **タスク説明 (issue なし) 入力時の着手 orchestrator 連鎖**: issue 番号が文字列として登場しても、URL / 番号として明示入力されていなければ `gh issue view` を呼ばずタスク説明として扱う。連鎖は行わない。
- **Codex CLI / 他 agent 用の fallback 実装**: 本 skill は Claude Code 専用。「Claude Code 限定であること」セクション参照。
- **`EnterWorktree` の `path` パラメータでクリーン命名を試みる**: `worktree-` prefix 強制を許容する方針 (issue #13 スコープ外)。
- **作成済み worktree のクリーンアップ**: `ExitWorktree` / `git worktree remove` 等は呼ばない。worktree のライフサイクル管理はユーザー責務。
