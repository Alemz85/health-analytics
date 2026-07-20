# Chat Headless Policy Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Make packaged Claude Code chat honor the health-helper allowlist without workspace trust and eliminate the unused-stdin warning.

**Architecture:** A small pure policy module builds both streaming and goal-metric CLI arguments from one exact allowlist. The main launcher consumes that policy, and packaging ships the routing skill without relying on project permission settings.

**Tech Stack:** Electron main process, Node child processes, TypeScript, Vitest, electron-builder.

---

### Task 1: Shared headless invocation policy

**Files:**
- Create: `app/src/main/chatPolicy.ts`
- Create: `app/src/main/__tests__/chatPolicy.test.ts`

- [ ] **Step 1: Write failing policy tests**

Assert the policy adds `--permission-mode dontAsk`, exactly eight narrow `Bash(...)` allow rules, no generic Bash/Edit/Write rule, streaming JSON flags when requested, and resume only when supplied. Assert streaming stdin mode is `ignore` and `closeChildStdin()` calls `end()`.

- [ ] **Step 2: Prove the tests fail**

Run `cd app && npx vitest run src/main/__tests__/chatPolicy.test.ts`. Expect failure because the module does not exist.

- [ ] **Step 3: Implement the pure policy**

Export `CHAT_ALLOWED_TOOLS`, `CLAUDE_STREAM_STDIO`, `buildStreamingClaudeArgs(prompt, resumeId?)`, `buildGoalClaudeArgs(prompt)`, and `closeChildStdin(child)`. Preserve the existing streaming flags and append the explicit headless permission arguments to both builders.

- [ ] **Step 4: Verify policy tests pass**

Run the focused test and expect all cases to pass.

### Task 2: Wire the policy into Electron and packaging

**Files:**
- Modify: `app/src/main/chat.ts`
- Modify: `app/electron-builder.yml`
- Modify: `app/src/main/__tests__/chatPolicy.test.ts`

- [ ] **Step 1: Add a failing integration/source contract**

Assert `chat.ts` uses both argument builders, passes `CLAUDE_STREAM_STDIO` to streaming `spawn`, and closes the `execFile` child's stdin. Assert electron-builder maps `.claude/skills` rather than the whole `.claude` directory.

- [ ] **Step 2: Prove the contract fails**

Run the focused test and expect failures against the current direct argument arrays and packaging mapping.

- [ ] **Step 3: Wire the streaming path**

Replace the inline streaming args with `buildStreamingClaudeArgs()`, use `stdio: CLAUDE_STREAM_STDIO`, and preserve stdout/stderr parsing and session resumption.

- [ ] **Step 4: Wire the goal path**

Replace `['-p', prompt]` with `buildGoalClaudeArgs(prompt)`, capture the `execFile` child, and call `closeChildStdin(child)` immediately.

- [ ] **Step 5: Narrow packaged settings**

Change electron-builder's resource mapping from `../chatctx/.claude` to `../chatctx/.claude/skills`, targeting `chatctx/.claude/skills`. Do not change the development settings file.

- [ ] **Step 6: Verify and package**

Run the focused test, `npm run typecheck`, full Vitest, and `npm run dist:mac`. Inspect the app bundle for the health skill and absence of packaged `.claude/settings.json`; verify DMG and ZIP integrity.
