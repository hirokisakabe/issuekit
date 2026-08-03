# AGENTS.md

This file provides guidance to coding agents (Claude Code, Codex, etc.) when working with code in this repository.

`CLAUDE.md` is a symlink to this file, so editing `AGENTS.md` updates both. **Edit `AGENTS.md` only** — do not edit `CLAUDE.md` directly or replace the symlink.

## What this repository is

issuekit is an **Agent Skills bundle**, not an application. It contains 9 skills as `skills/<name>/SKILL.md` markdown files, distributed via `gh skill install hirokisakabe/issuekit` (version-pinnable via GitHub Releases) and `npx skills add hirokisakabe/issuekit` (always HEAD). There is no build, test, or lint toolchain — the artifacts are the SKILL.md files themselves.

The bundle codifies an **issue-driven development** workflow where the GitHub issue body is the rich plan (with `Status: Ready/Draft`, `## 受け入れ条件`, `## スコープ外`, `Depends on:`, `親: #N`), investigation results default to issue comments, and the repository contains only durable artifacts. See `README.md` for the philosophy and the comparison vs. Spec Kit / cc-spex / superpowers.

## Skill graph

`issue-dispatch` is the upper-level implementation scheduler:

- `issue-dispatch` → N × `issue-implement` (one dedicated worker / worktree / branch / PR per issue; dependencies and high-conflict issues are serialized)
- `issue-implement` → `issue-dispatch` only for a single PR-shaped Ready issue invoked from Codex CLI on the default branch. The dispatcher creates one ordinary worktree and launches `codex exec -C <path>`; the linked-worktree worker re-enters `issue-implement` and continues without dispatching again.
- Direct multi-issue implementation requests enter `issue-dispatch`. `issue-pick` remains read-only and does not chain into it without a new explicit implementation request from the user.

`issue-implement` is the orchestrator of the implementation cycle and **calls** the other skills:

- `issue-implement` → `acceptance-check` (verifies `## 受け入れ条件` against the final repo state after implementation+commits, **before** `cross-review` so an acceptance ✗ does not waste a cross-review pass)
- `issue-implement` → `cross-review` (second-opinion code review of the `base...HEAD` diff after `acceptance-check` passes, before PR creation; review fixes land as additional commits, not amends)
- `issue-implement` → `worktree-start` (**conditional**, inside the mandatory isolation preflight before implementation): fires only for a Claude Code interactive session on the repository's default branch when `EnterWorktree` is available. An existing linked worktree is reused; an existing non-default feature branch is preserved for a single implementation; unsafe runtime/location combinations stop before writes or commits. `Status: Draft` still aborts in step 1 before this preflight.
- `issue-implement` → `issue-dispatch` (**conditional**, inside the mandatory isolation preflight): fires only for Codex CLI on the default branch and passes exactly the current issue. The parent session stays in place while the dispatcher owns the worker worktree and waits through PR / CI.
- `issue-implement` guards its direct-entry path with the same completion-shape rule: only PR-shaped Ready issues continue; comment-shaped issues stop with an `issue-investigate` recommendation, and ambiguous issues stop with an `issue-refine` recommendation.
- `worktree-start` → `issue-implement` or `issue-investigate` (**only** when input is an issue URL/number with `Status: Ready` and a clear completion shape; PR-shaped issues route to `issue-implement`, comment-shaped issues route to `issue-investigate`, and ambiguous issues stop after the worktree switch with an `issue-refine` recommendation)
- `issue-create` / `issue-refine` / `issue-pick` are entry points; they do not chain into other skills. `issue-pick` is a triage entry point and does not chain (see its "やらないこと" — handing off to `issue-implement` or `issue-investigate` is via user only).

`issue-investigate` is the separate orchestrator for comment-complete investigation, design, and technical-validation issues. It posts a structured result comment, calls `acceptance-check`, and closes the issue only after the acceptance check succeeds. It does not commit, open a PR, or call `cross-review`. `issue-pick` suggests it via the user, while `worktree-start` may chain to it when a Ready issue's acceptance criteria require only an issue comment and no durable repo change.

The `issue-implement ↔ worktree-start` and `issue-implement ↔ issue-dispatch` edges are **bidirectional but not looping**:

- When `worktree-start` is the entry point and chains forward into `issue-implement`, the latter sees that it is already in a linked worktree and continues without re-invoking `worktree-start`.
- When `issue-implement` is the entry point and calls `worktree-start` from step 4, it must pass a pre-generated branch-name slug (`<title>-<issue番号>`), **not** the issue number. Passing the number would re-enter `worktree-start`'s Status-detection path and re-chain back into `issue-implement` unnecessarily. The recursion would still terminate via the no-op check, but the redundant invocation is avoided by routing through the task-description mode of `worktree-start`.
- When Codex CLI `issue-implement` on the default branch calls `issue-dispatch`, the dispatcher passes the issue number to a new worker in a dedicated linked worktree. That worker's isolation preflight recognizes its assignment and continues locally, so it does not call `issue-dispatch` again.

When editing one skill, check whether others reference it. Cross-references appear in two forms:

- Plugin mode: `issuekit:<skill-name>` (e.g. `issuekit:cross-review`)
- APM plain-skill mode: bare `<skill-name>` (e.g. `cross-review`)

Both forms must stay in sync — `issue-dispatch`, `issue-implement`, `issue-investigate`, `issue-pick`, and `worktree-start` document each form explicitly.

## Hardcoded Japanese keywords

Skills mechanically parse Japanese section headers from issue bodies:

- `Status: Ready` / `Status: Draft` (must be at the **top** of the body)
- `Depends on: #N, #M`
- `親: #N`
- `## 概要` / `## 背景 / モチベーション` / `## 受け入れ条件` / `## Ready にするための未決事項` / `## スコープ外` / `## 参考` / `## 実装方針` / `## 再現手順` / `## 期待する挙動` / `## 実際の挙動` / `## 調査メモ`
- Result comments: `## 調査結果` / `### 結論` / `### 根拠` / `### 検証内容` / `### Blocker` / `### 却下案` / `### 後続候補`

These strings are not localizable in the current implementation. Forking is required to use English issues (per README).

## Status semantics (single source of truth: `issue-create`)

`Status` is judged on **acceptance-criteria certainty only**, not implementation-plan certainty. A bug issue with a prioritized list of fix candidates and verifiable acceptance criteria is `Ready`. Acceptance criteria containing 「仮」/「要検討」 or that are too vague to self-verify → `Draft`. Draft issues must include `## Ready にするための未決事項`, listing only the concrete decisions needed to finalize acceptance criteria. `issue-refine`, `issue-dispatch`, `issue-implement`, and `issue-investigate` defer to `issue-create` for this rule — do not duplicate the definition; update `issue-create` and reference it.

## Depends on / parent semantics

- Dependency state is **never** written into issue bodies. Always resolve via `gh issue view <N> --json state` (see `issue-create` "依存 issue").
- Parents are linked via GitHub's sub-issue feature and resolved with the REST sub-issues endpoints (`GET /repos/{owner}/{repo}/issues/{issue_number}/parent` and `GET /repos/{owner}/{repo}/issues/{issue_number}/sub_issues`). `親: #N` in the body is a fallback for legacy issues. `issue-pick` takes the union of both. (`issue-create` step 6 — `gh api repos/.../sub_issues` — is mandatory, not optional.)

## Acceptance check is read-only

`acceptance-check` reports `✓ / ✗ / ?` and never writes. It does not flip `- [ ]` to `- [x]`, never edits issue bodies, and does not perform actual UI/CLI verification (only suggests how). `?` items are explicitly delegated to the caller.

## Runtime worktree isolation

`issue-implement` step 4 is a mandatory isolation preflight. It resolves the default branch dynamically, checks `git rev-parse --git-common-dir` against `--git-dir`, and classifies the runtime/location before any implementation write or commit:

- A linked worktree dedicated to the current issue/task continues without double creation. A worktree assigned to another task, or with unverifiable assignment, stops. A non-default feature branch is preserved for a single implementation. A main working tree in detached HEAD stops as unclassifiable; a runtime-owned detached HEAD linked worktree (such as Codex App) is allowed when its current-task assignment is established.
- A write-capable parallel worker is evaluated first and requires **one worker = one worktree** even if it is already on a feature branch. Continue only when runtime/session context establishes that the linked worktree is dedicated to that worker; otherwise stop.
- Default-branch execution must move to a dedicated worktree or stop before implementation. There is no skip-and-continue path.
- Codex CLI on the default branch hands a single issue to `issue-dispatch`. The parent remains in its current cwd; the dispatcher creates an ordinary worktree and launches `codex exec -C <path>` with one issue-specific worker. If dispatch preflight cannot guarantee sandbox, approval, authentication, or isolation, it stops before implementation.
- Codex App managed worktrees and Handoff are App-owned. Skills may verify that the chat is isolated or tell the user to use the App UI, but must not claim to create or control App-managed worktrees.
- Claude Code interactive sessions may invoke `worktree-start`, which owns the in-session `EnterWorktree` call. `claude --worktree`, subagent `isolation: worktree`, Agent view background-session isolation, and Desktop automatic session worktrees remain runtime-owned paths.

`worktree-start` is therefore still Claude Code-only, but its no-op inside an existing linked worktree is an **issuekit policy**, not a general `EnterWorktree` limitation. Current Claude Code can switch to another existing worktree under `.claude/worktrees/`; issuekit intentionally does not do so because it would displace a session already assigned to a task. Resume and cleanup follow the current [Claude Code worktree documentation](https://code.claude.com/docs/en/worktrees): resumes return to the associated worktree when it exists, interactive exit cleanup depends on whether work is present, and non-interactive `-p` worktrees require manual cleanup.

Worktrees are fresh checkouts. Document dependency/environment initialization and disk usage where relevant. `.worktreeinclude` is for ignored local files needed by Claude Code-created and Codex App managed worktrees; it does not apply to ordinary `git worktree add`.

## Dispatch isolation and scheduling

`issue-dispatch` owns only cross-issue orchestration. It must fetch each candidate's current body and comments, resolve real dependency state through GitHub, read parent context, estimate paths from the issue contract, and publish the launch plan before starting workers.

- The invariant is **1 issue = 1 worker = 1 worktree = 1 branch = 1 PR**. Never share a write-capable checkout between workers or mix multiple issues into one branch / PR.
- `Depends on:` is a DAG. Open dependencies outside the candidate set block the issue. Dependencies inside the set create scheduling edges, but a downstream worker still waits for the dependency issue to close and land on the default branch; PR + CI success alone is not a merge substitute.
- High-overlap changes are serialized with the same merge barrier. If independence cannot be established, show the uncertain path estimate before implementation and ask the user whether to serialize or exclude.
- Multiple-issue concurrency defaults to 3 and is capped by the user's value, runtime limit, and currently independent Ready issue count. A failed worker blocks only its dependents; unrelated workers continue.
- Codex CLI write workers use ordinary `git worktree` checkouts and `codex exec -C`. Their non-interactive sandbox must write the assigned worktree and shared git common dir without granting broad repository access. Fresh approvals cannot be requested mid-run, so approval, sandbox, `gh`, and Codex authentication are preflight requirements.
- Current Codex native subagents may be used for read-only analysis, but not parallel writes unless the runtime explicitly guarantees a dedicated cwd / worktree per worker. Claude Code write workers use `isolation: worktree`, Agent view isolation, or an equivalent official primitive; non-isolated Agent teams are not used.
- Codex App top-level Worktree chats and Handoff remain App-owned. When the surface cannot guarantee automated per-issue worktrees, return the plan and launch prompts; do not automate the UI.
- The parent waits for every worker to succeed, fail, block, or remain waiting, then aggregates issue number, state, branch, PR URL, CI, and blocker. It never auto-merges or auto-cleans worker state.

## Cross-review reviewer session selection

`cross-review` is defined as a second-opinion review from an independent reviewer session, not as a guarantee that a different backend or different model is used.

- Codex runtime uses Codex CLI (`codex exec --sandbox read-only` with stdin diff pipe) to start a fresh reviewer session.
- Claude Code runtime uses Claude CLI headless (`claude -p` with stdin diff) to start a fresh reviewer session.

The runtime must be determined from the running agent's explicit environment, not inferred from whichever CLI exists on `PATH`. Environment-variable backend overrides and auto-detection fallback are intentionally not part of the workflow. If a different backend / different model review is needed, track that as a separate issue instead of keeping it inside `cross-review`.

## Cross-review base branch resolution

`cross-review` resolves the base branch dynamically via `gh repo view --json defaultBranchRef --jq '.defaultBranchRef.name'` and feeds the result into `git diff "$BASE_REF"...HEAD` (the diff is piped to `codex exec` / `claude -p` via stdin). `master` / `develop` / `trunk` repos work without modification. The skill stops with an explicit error (no silent fallback to `main`) when default-branch resolution fails — see its "失敗時の対応" section. Override (env var / arg) is intentionally out of scope.

## External dependencies

- `gh` CLI — all GitHub operations. Must be authenticated against the target repo.
- The CLI for the current agent runtime: Codex CLI (`brew install --cask codex`) when implementing from Codex, or Claude CLI (`npm install -g @anthropic-ai/claude-code`) when implementing from Claude Code. `cross-review` must fail loudly (not silently skip) when the corresponding CLI is unavailable or the current runtime has no documented reviewer-session launch step.
- Codex CLI dispatch additionally requires authenticated non-interactive `codex exec`, ordinary `git worktree` support, and sandbox write access to each worker checkout plus the shared git common dir.
- Claude Code with `EnterWorktree` support — required by `worktree-start`. If unavailable, the skill instructs users to update/restart or start a new isolated session with `claude --worktree` rather than continuing on the default branch.

## Editing skills

- Frontmatter `name:` and `description:` drive how Claude Code triggers the skill. Keep `description:` specific and trigger-oriented (it is matched against user utterances).
- When changing any `skills/*/SKILL.md`, bump the changed skill's frontmatter `version:` before opening a PR. Use semver: patch for bug fixes, wording fixes, and typo fixes; minor for backward-compatible feature or workflow additions; major for breaking behavior or substantial scope changes.
- Bump only the `version:` of SKILL.md files that changed. Do not touch unchanged skills' versions.
- The release workflow detects `version:` changes and creates GitHub Releases. Merging a skill change without a bump prevents version-pinned installs such as `gh skill install hirokisakabe/issuekit <skill-name>@v<version>` from receiving that change.
- Prefer editing existing SKILL.md files over adding new ones. New skills should fit the existing graph (orchestrator vs. entry-point vs. verifier) and follow the structure: スコープ → 依存 → 入力 → 実行手順 → 失敗時の対応 → やらないこと.
- "やらないこと" sections are load-bearing — they prevent scope creep across cycles. When in doubt, expand "やらないこと" rather than the implementation surface.

## Repository conventions inherited from the user's global CLAUDE.md

- PR descriptions are written in Japanese.
- `close #<番号>` is added to a PR description only when the user uniquely selects the issue in the conversation and requests its complete implementation. This includes direct issue numbers / URLs and unambiguous references to a previously presented numbered selection; machine-selected issues, partial implementations, epic subsets, and relation-only PRs do not close. Dispatcher workers inherit `ISSUE_CLOSE_INTENT=true|false` together with `ISSUE_CLOSE_INTENT_REASON` and never infer intent from the mechanical `issue-implement <番号>` prompt.
- Browser automation uses `agent-browser --engine lightpanda`.
