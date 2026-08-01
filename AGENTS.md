# AGENTS.md

This file provides guidance to coding agents (Claude Code, Codex, etc.) when working with code in this repository.

`CLAUDE.md` is a symlink to this file, so editing `AGENTS.md` updates both. **Edit `AGENTS.md` only** — do not edit `CLAUDE.md` directly or replace the symlink.

## What this repository is

issuekit is an **Agent Skills bundle**, not an application. It contains 8 skills as `skills/<name>/SKILL.md` markdown files, distributed via `gh skill install hirokisakabe/issuekit` (version-pinnable via GitHub Releases) and `npx skills add hirokisakabe/issuekit` (always HEAD). There is no build, test, or lint toolchain — the artifacts are the SKILL.md files themselves.

The bundle codifies an **issue-driven development** workflow where the GitHub issue body is the rich plan (with `Status: Ready/Draft`, `## 受け入れ条件`, `## スコープ外`, `Depends on:`, `親: #N`), investigation results default to issue comments, and the repository contains only durable artifacts. See `README.md` for the philosophy and the comparison vs. Spec Kit / cc-spex / superpowers.

## Skill graph

`issue-implement` is the orchestrator of the implementation cycle and **calls** the other skills:

- `issue-implement` → `acceptance-check` (verifies `## 受け入れ条件` against the final repo state after implementation+commits, **before** `cross-review` so an acceptance ✗ does not waste a cross-review pass)
- `issue-implement` → `cross-review` (second-opinion code review of the `base...HEAD` diff after `acceptance-check` passes, before PR creation; review fixes land as additional commits, not amends)
- `issue-implement` → `worktree-start` (**conditional**, inside the mandatory isolation preflight before implementation): fires only for a Claude Code interactive session on the repository's default branch when `EnterWorktree` is available. An existing linked worktree is reused; an existing non-default feature branch is preserved for a single implementation; unsafe runtime/location combinations stop before writes or commits. `Status: Draft` still aborts in step 1 before this preflight.
- `issue-implement` guards its direct-entry path with the same completion-shape rule: only PR-shaped Ready issues continue; comment-shaped issues stop with an `issue-investigate` recommendation, and ambiguous issues stop with an `issue-refine` recommendation.
- `worktree-start` → `issue-implement` or `issue-investigate` (**only** when input is an issue URL/number with `Status: Ready` and a clear completion shape; PR-shaped issues route to `issue-implement`, comment-shaped issues route to `issue-investigate`, and ambiguous issues stop after the worktree switch with an `issue-refine` recommendation)
- `issue-create` / `issue-refine` / `issue-pick` are entry points; they do not chain into other skills. `issue-pick` is a triage entry point and does not chain (see its "やらないこと" — handing off to `issue-implement` or `issue-investigate` is via user only).

`issue-investigate` is the separate orchestrator for comment-complete investigation, design, and technical-validation issues. It posts a structured result comment, calls `acceptance-check`, and closes the issue only after the acceptance check succeeds. It does not commit, open a PR, or call `cross-review`. `issue-pick` suggests it via the user, while `worktree-start` may chain to it when a Ready issue's acceptance criteria require only an issue comment and no durable repo change.

The `issue-implement ↔ worktree-start` edge is **bidirectional but not looping**:

- When `worktree-start` is the entry point and chains forward into `issue-implement`, the latter sees that it is already in a linked worktree and continues without re-invoking `worktree-start`.
- When `issue-implement` is the entry point and calls `worktree-start` from step 4, it must pass a pre-generated branch-name slug (`<title>-<issue番号>`), **not** the issue number. Passing the number would re-enter `worktree-start`'s Status-detection path and re-chain back into `issue-implement` unnecessarily. The recursion would still terminate via the no-op check, but the redundant invocation is avoided by routing through the task-description mode of `worktree-start`.

When editing one skill, check whether others reference it. Cross-references appear in two forms:

- Plugin mode: `issuekit:<skill-name>` (e.g. `issuekit:cross-review`)
- APM plain-skill mode: bare `<skill-name>` (e.g. `cross-review`)

Both forms must stay in sync — `issue-implement`, `issue-investigate`, `issue-pick`, and `worktree-start` document each form explicitly.

## Hardcoded Japanese keywords

Skills mechanically parse Japanese section headers from issue bodies:

- `Status: Ready` / `Status: Draft` (must be at the **top** of the body)
- `Depends on: #N, #M`
- `親: #N`
- `## 概要` / `## 背景 / モチベーション` / `## 受け入れ条件` / `## Ready にするための未決事項` / `## スコープ外` / `## 参考` / `## 実装方針` / `## 再現手順` / `## 期待する挙動` / `## 実際の挙動` / `## 調査メモ`
- Result comments: `## 調査結果` / `### 結論` / `### 根拠` / `### 検証内容` / `### Blocker` / `### 却下案` / `### 後続候補`

These strings are not localizable in the current implementation. Forking is required to use English issues (per README).

## Status semantics (single source of truth: `issue-create`)

`Status` is judged on **acceptance-criteria certainty only**, not implementation-plan certainty. A bug issue with a prioritized list of fix candidates and verifiable acceptance criteria is `Ready`. Acceptance criteria containing 「仮」/「要検討」 or that are too vague to self-verify → `Draft`. Draft issues must include `## Ready にするための未決事項`, listing only the concrete decisions needed to finalize acceptance criteria. `issue-refine`, `issue-implement`, and `issue-investigate` defer to `issue-create` for this rule — do not duplicate the definition; update `issue-create` and reference it.

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
- Codex CLI stops and instructs the user to run ordinary `git worktree add`, then `codex -C <path>` in a new session. The running session is not assumed to migrate cwd.
- Codex App managed worktrees and Handoff are App-owned. Skills may verify that the chat is isolated or tell the user to use the App UI, but must not claim to create or control App-managed worktrees.
- Claude Code interactive sessions may invoke `worktree-start`, which owns the in-session `EnterWorktree` call. `claude --worktree`, subagent `isolation: worktree`, Agent view background-session isolation, and Desktop automatic session worktrees remain runtime-owned paths.

`worktree-start` is therefore still Claude Code-only, but its no-op inside an existing linked worktree is an **issuekit policy**, not a general `EnterWorktree` limitation. Current Claude Code can switch to another existing worktree under `.claude/worktrees/`; issuekit intentionally does not do so because it would displace a session already assigned to a task. Resume and cleanup follow the current [Claude Code worktree documentation](https://code.claude.com/docs/en/worktrees): resumes return to the associated worktree when it exists, interactive exit cleanup depends on whether work is present, and non-interactive `-p` worktrees require manual cleanup.

Worktrees are fresh checkouts. Document dependency/environment initialization and disk usage where relevant. `.worktreeinclude` is for ignored local files needed by Claude Code-created and Codex App managed worktrees; it does not apply to ordinary `git worktree add`.

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
- `close #<番号>` is added to a PR description **only** when the user explicitly specifies the issue number (or when invoked via `issue-implement <番号>`, which counts as explicit).
- Browser automation uses `agent-browser --engine lightpanda`.
