# AGENTS.md

This file provides guidance to coding agents (Claude Code, Codex, etc.) when working with code in this repository.

`CLAUDE.md` is a symlink to this file, so editing `AGENTS.md` updates both. **Edit `AGENTS.md` only** — do not edit `CLAUDE.md` directly or replace the symlink.

## What this repository is

issuekit is a **Claude Code skill bundle**, not an application. It contains 7 skills as `skills/<name>/SKILL.md` markdown files, distributed through skills.sh (`npx skills add hirokisakabe/issuekit`). There is no build, test, or lint toolchain — the artifacts are the SKILL.md files themselves.

The bundle codifies an **issue-driven development** workflow where the GitHub issue body is the rich plan (with `Status: Ready/Draft`, `## 受け入れ条件`, `## スコープ外`, `Depends on:`, `親: #N`), and the repository contains only durable code. See `README.md` for the philosophy and the comparison vs. Spec Kit / cc-spex / superpowers.

## Skill graph

`issue-implement` is the orchestrator of the implementation cycle and **calls** the other skills:

- `issue-implement` → `acceptance-check` (verifies `## 受け入れ条件` against the final repo state after implementation+commits, **before** `cross-review` so an acceptance ✗ does not waste a cross-review pass)
- `issue-implement` → `cross-review` (second-opinion code review of the `base...HEAD` diff after `acceptance-check` passes, before PR creation; review fixes land as additional commits, not amends)
- `issue-implement` → `worktree-start` (**conditional**, before implementation in `issue-implement` step 4): fires only when **all four** conditions hold — `EnterWorktree` is available (= Claude Code runtime), the session is outside any worktree (`git rev-parse --git-common-dir` == `--git-dir`), the current branch is the repo's default branch (`gh repo view --json defaultBranchRef`), and `Status: Ready`. `Status: Draft` triggers an early abort in step 1, so the worktree is never created for Draft issues.
- `worktree-start` → `issue-implement` (**only** when input is an issue URL/number with `Status: Ready`; with a generic task description, `Status: Draft`, or unformatted issues it stops at the worktree switch)
- `issue-create` / `issue-refine` / `issue-pick` are entry points; they do not chain into other skills. `issue-pick` is a triage entry point and does not chain (see its "やらないこと" — handing off to `issue-implement` is via user only).

The `issue-implement ↔ worktree-start` edge is **bidirectional but not looping**:

- When `worktree-start` is the entry point and chains forward into `issue-implement`, the latter would re-invoke `worktree-start`, but the second call hits the "already inside a worktree" no-op check and returns immediately.
- When `issue-implement` is the entry point and calls `worktree-start` from step 4, it must pass a pre-generated branch-name slug (`<title>-<issue番号>`), **not** the issue number. Passing the number would re-enter `worktree-start`'s Status-detection path and re-chain back into `issue-implement` unnecessarily. The recursion would still terminate via the no-op check, but the redundant invocation is avoided by routing through the task-description mode of `worktree-start`.

When editing one skill, check whether others reference it. Cross-references appear in two forms:

- Plugin mode: `issuekit:<skill-name>` (e.g. `issuekit:cross-review`)
- APM plain-skill mode: bare `<skill-name>` (e.g. `cross-review`)

Both forms must stay in sync — `issue-implement` and `issue-pick` document each form explicitly.

## Hardcoded Japanese keywords

Skills mechanically parse Japanese section headers from issue bodies:

- `Status: Ready` / `Status: Draft` (must be at the **top** of the body)
- `Depends on: #N, #M`
- `親: #N`
- `## 概要` / `## 背景 / モチベーション` / `## 受け入れ条件` / `## スコープ外` / `## 参考` / `## 実装方針` / `## 再現手順` / `## 期待する挙動` / `## 実際の挙動` / `## 調査メモ`

These strings are not localizable in the current implementation. Forking is required to use English issues (per README).

## Status semantics (single source of truth: `issue-create`)

`Status` is judged on **acceptance-criteria certainty only**, not implementation-plan certainty. A bug issue with a prioritized list of fix candidates and verifiable acceptance criteria is `Ready`. Acceptance criteria containing 「仮」/「要検討」 or that are too vague to self-verify → `Draft`. `issue-refine` and `issue-implement` defer to `issue-create` for this rule — do not duplicate the definition; update `issue-create` and reference it.

## Depends on / parent semantics

- Dependency state is **never** written into issue bodies. Always resolve via `gh issue view <N> --json state` (see `issue-create` "依存 issue").
- Parents are linked via GitHub's sub-issue feature (GraphQL `issue.parent`); REST `/issues/{n}` does not expose `parent`. `親: #N` in the body is a fallback for legacy issues. `issue-pick` takes the union of both. (`issue-create` step 6 — `gh api repos/.../sub_issues` — is mandatory, not optional.)

## Acceptance check is read-only

`acceptance-check` reports `✓ / ✗ / ?` and never writes. It does not flip `- [ ]` to `- [x]`, never edits issue bodies, and does not perform actual UI/CLI verification (only suggests how). `?` items are explicitly delegated to the caller.

## Worktree-start is Claude Code only

`worktree-start` invokes the `EnterWorktree` tool added to Claude Code in v2.1.49 (2026-02-19). This primitive is Claude Code-specific:

- Codex CLI has no worktree concept ([openai/codex#13120](https://github.com/openai/codex/issues/13120)); Codex Worktrees ship only in the Desktop app, not the CLI.
- The skill therefore does **not** provide a fallback for non-Claude-Code agents — when run under another runtime the `EnterWorktree` tool will simply not exist. Users on Codex / Cursor / Gemini should fall back to plain `git worktree add` outside the agent.
- Branch naming is the skill's responsibility (LLM-named in kebab-case, or user-supplied verbatim). The `worktree-` prefix forced by `EnterWorktree` is intentionally accepted; the `path` parameter escape hatch is out of scope (see issue #13).
- The skill is a no-op when the current session is already inside a worktree — `EnterWorktree` itself rejects re-entry, and the skill double-checks via `git rev-parse --git-common-dir` / `--git-dir` before calling the tool.

## Cross-review backend selection

`cross-review` supports two backends, selected via the `CROSS_REVIEW_BACKEND` environment variable:

- `codex` — OpenAI Codex CLI (`codex exec` with stdin diff pipe; the `review` sub-command is avoided because of the 0.125.0 `--base` / `--uncommitted` / `[PROMPT]` mutual exclusion). Intended for "implemented with Claude Code → reviewed by GPT".
- `claude-self` — Claude CLI headless (`claude --bare -p` with stdin diff). Intended for "implemented with Codex CLI / Cursor → reviewed by Claude".

When the env var is unset, the skill falls back to `command -v` auto-detection (`codex` first, then `claude`). When the env var is **set** but the corresponding CLI is missing, the skill fails explicitly — there is no silent fallback to the other backend, since that would silently change the reviewer model the user asked for.

## Cross-review base branch resolution

`cross-review` resolves the base branch dynamically via `gh repo view --json defaultBranchRef --jq '.defaultBranchRef.name'` and feeds the result into `git diff "$BASE_REF"...HEAD` for both backends (the diff is piped to `codex exec` / `claude --bare -p` via stdin). `master` / `develop` / `trunk` repos work without modification. The skill stops with an explicit error (no silent fallback to `main`) when default-branch resolution fails — see its "失敗時の対応" section. Override (env var / arg) is intentionally out of scope.

## External dependencies

- `gh` CLI — all GitHub operations. Must be authenticated against the target repo.
- At least one of: Codex CLI (`brew install --cask codex`) or Claude CLI (`npm install -g @anthropic-ai/claude-code`), required by `cross-review`. The skill must fail loudly (not silently skip) when neither is available, or when an explicitly-selected backend's CLI is missing.
- Claude Code v2.1.49 or newer — required by `worktree-start` for the `EnterWorktree` tool. Older versions surface this as "tool not found"; the skill instructs users to upgrade rather than attempting any workaround.

## Editing skills

- Frontmatter `name:` and `description:` drive how Claude Code triggers the skill. Keep `description:` specific and trigger-oriented (it is matched against user utterances).
- Prefer editing existing SKILL.md files over adding new ones. New skills should fit the existing graph (orchestrator vs. entry-point vs. verifier) and follow the structure: スコープ → 依存 → 入力 → 実行手順 → 失敗時の対応 → やらないこと.
- "やらないこと" sections are load-bearing — they prevent scope creep across cycles. When in doubt, expand "やらないこと" rather than the implementation surface.

## Repository conventions inherited from the user's global CLAUDE.md

- PR descriptions are written in Japanese.
- `close #<番号>` is added to a PR description **only** when the user explicitly specifies the issue number (or when invoked via `issue-implement <番号>`, which counts as explicit).
- Browser automation uses `agent-browser --engine lightpanda`.
