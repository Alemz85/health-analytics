# Persistent Agent Workspace Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Build a durable, long-form Alke Chat workspace whose local Claude agent can safely edit the Sports app repository outside `app/` while drafts, streams, tool activity, and interrupted runs survive navigation and restart.

**Architecture:** The Electron main process owns one generation at a time, records sequenced stream events in a bounded atomic `userData` snapshot, and exposes snapshot/send/stop/continue IPC. A renderer-level provider mounted above tab routing reconciles that runtime with a versioned local UI snapshot; the Chat view becomes a document-first workspace with responsive history and work-log drawers. Claude receives a canonical repo path and a fail-closed macOS sandbox whose built-in and Bash write boundaries both deny `app/`, `.git/`, secrets, and all paths outside the repo.

**Tech Stack:** Electron 34, React 19, TypeScript, Vite/electron-vite, Vitest, Claude Code CLI 2.1.x settings/sandbox, React Markdown, CSS tokens from `DESIGN.md`, Playwright over Electron CDP for rendered verification.

---

## File structure

- Create `app/src/main/chatWorkspace.ts`: canonical repo discovery and validation; no Claude process concerns.
- Create `app/src/main/chatRuntime.ts`: bounded generation state, sequencing, atomic snapshot persistence, and interruption recovery; no Electron UI concerns.
- Modify `app/src/main/chatPolicy.ts`: build separate interactive and fixed-goal Claude policies from canonical paths.
- Modify `app/src/main/chat.ts`: orchestrate Claude, DB messages, runtime store, and stream envelopes.
- Modify `app/src/main/index.ts`: register the new runtime/continue IPC and initialize the chat service.
- Modify `app/src/preload/index.ts`: expose typed runtime/continue APIs and sequenced stream subscription.
- Modify `app/src/shared/types.ts`: single source of truth for runtime, event, IPC, and `HealthApi` contracts.
- Create `app/src/renderer/src/chat/chatUiState.ts`: pure reducer and versioned local snapshot validation.
- Create `app/src/renderer/src/chat/ChatRuntimeProvider.tsx`: persistent subscription, hydration, send/stop/continue actions, and query invalidation.
- Create `app/src/renderer/src/views/chat/ChatHistory.tsx`: history rail/drawer and session row controls.
- Create `app/src/renderer/src/views/chat/ChatWorkLog.tsx`: compact disclosure and accessible work-log drawer.
- Create `app/src/renderer/src/views/chat/ChatComposer.tsx`: persistent draft, mode, attachments, send/stop controls.
- Modify `app/src/renderer/src/views/ChatView.tsx`: document/workspace composition and session fetching only.
- Replace `app/src/renderer/src/views/ChatView.css`: responsive, token-only long-form workspace styling.
- Modify `app/src/renderer/src/App.tsx`: mount `ChatRuntimeProvider` above active-tab rendering.
- Modify `DESIGN.md`: document the durable analysis-workspace pattern without changing tokens.
- Add focused tests beside main and renderer modules; keep the existing source-contract tests only for rendered structure that cannot execute under the Node test environment.

### Task 1: Resolve and validate the source repository

**Files:**
- Create: `app/src/main/chatWorkspace.ts`
- Test: `app/src/main/__tests__/chatWorkspace.test.ts`

- [ ] **Step 1: Write the failing resolver tests**

```ts
import { mkdtemp, mkdir, realpath, symlink } from 'node:fs/promises'
import { tmpdir } from 'node:os'
import { join } from 'node:path'
import { describe, expect, it } from 'vitest'
import { resolveChatWorkspace } from '../chatWorkspace'

async function fixture(): Promise<string> {
  const root = await mkdtemp(join(tmpdir(), 'alke-workspace-'))
  await Promise.all(['.git', 'app', 'chatctx'].map((name) => mkdir(join(root, name))))
  return root
}

describe('resolveChatWorkspace', () => {
  it('prefers and canonicalizes ALKE_REPO_ROOT', async () => {
    const root = await fixture()
    const alias = `${root}-alias`
    await symlink(root, alias)
    await expect(resolveChatWorkspace({ configuredRoot: alias })).resolves.toMatchObject({
      repoRoot: await realpath(root),
      source: 'configured'
    })
  })

  it('derives dev root from source chatctx', async () => {
    const root = await fixture()
    await expect(resolveChatWorkspace({ sourceChatctxDir: join(root, 'chatctx') })).resolves
      .toMatchObject({ repoRoot: root, source: 'development' })
  })

  it('uses the validated owner fallback only when packaged', async () => {
    const root = await fixture()
    await expect(resolveChatWorkspace({ packaged: true, ownerFallback: root })).resolves
      .toMatchObject({ repoRoot: root, source: 'owner-fallback' })
  })

  it('rejects missing markers and never broadens to a parent directory', async () => {
    const root = await mkdtemp(join(tmpdir(), 'alke-invalid-'))
    await expect(resolveChatWorkspace({ configuredRoot: root })).rejects.toThrow(
      'repository not connected'
    )
  })
})
```

- [ ] **Step 2: Run the resolver test and confirm red**

Run: `cd app && npx vitest run src/main/__tests__/chatWorkspace.test.ts`

Expected: FAIL because `../chatWorkspace` does not exist.

- [ ] **Step 3: Implement canonical resolution**

```ts
import { realpath, stat } from 'node:fs/promises'
import { join } from 'node:path'

export interface ChatWorkspaceOptions {
  configuredRoot?: string
  sourceChatctxDir?: string
  packaged?: boolean
  ownerFallback?: string
}

export interface ChatWorkspace {
  repoRoot: string
  appRoot: string
  gitRoot: string
  chatctxRoot: string
  source: 'configured' | 'development' | 'owner-fallback'
}

async function isDirectory(path: string): Promise<boolean> {
  try { return (await stat(path)).isDirectory() } catch { return false }
}

export async function resolveChatWorkspace(options: ChatWorkspaceOptions): Promise<ChatWorkspace> {
  const candidates: Array<[string | undefined, ChatWorkspace['source']]> = [
    [options.configuredRoot, 'configured'],
    [options.sourceChatctxDir ? join(options.sourceChatctxDir, '..') : undefined, 'development'],
    [options.packaged ? options.ownerFallback : undefined, 'owner-fallback']
  ]
  for (const [candidate, source] of candidates) {
    if (!candidate) continue
    try {
      const repoRoot = await realpath(candidate)
      const appRoot = join(repoRoot, 'app')
      const gitRoot = join(repoRoot, '.git')
      const chatctxRoot = join(repoRoot, 'chatctx')
      if (await Promise.all([appRoot, gitRoot, chatctxRoot].map(isDirectory)).then((v) => v.every(Boolean))) {
        return { repoRoot, appRoot, gitRoot, chatctxRoot, source }
      }
    } catch { /* try the next explicit candidate */ }
  }
  throw new Error('Sports app repository not connected. Set ALKE_REPO_ROOT to its absolute path.')
}
```

- [ ] **Step 4: Run resolver tests and typecheck**

Run: `cd app && npx vitest run src/main/__tests__/chatWorkspace.test.ts && npm run typecheck:node`

Expected: resolver tests PASS and node typecheck exits 0.

- [ ] **Step 5: Commit**

```bash
git add app/src/main/chatWorkspace.ts app/src/main/__tests__/chatWorkspace.test.ts
git commit -m "feat(chat): resolve canonical agent workspace"
```

### Task 2: Replace the interactive helper allowlist with a fail-closed repo sandbox

**Files:**
- Modify: `app/src/main/chatPolicy.ts`
- Modify: `app/src/main/__tests__/chatPolicy.test.ts`

- [ ] **Step 1: Replace narrow-policy assertions with path-boundary tests**

```ts
const paths = {
  repoRoot: '/Users/owner/Projects/Sports app',
  appRoot: '/Users/owner/Projects/Sports app/app',
  gitRoot: '/Users/owner/Projects/Sports app/.git',
  chatctxRoot: '/Users/owner/Projects/Sports app/chatctx',
  runtimeRoot: '/Applications/Alke.app/Contents/Resources/chatctx',
  homeRoot: '/Users/owner',
  attachmentPaths: ['/Users/owner/Desktop/plan.pdf']
}

it('grants interactive repo tools without permission bypass', () => {
  const args = buildStreamingClaudeArgs('hello', paths)
  expect(args).toContain('acceptEdits')
  expect(args).toContain('Read,Glob,Grep,Edit,Write,Bash,Skill')
  expect(args).toEqual(expect.arrayContaining(['--add-dir', paths.repoRoot]))
  expect(args).not.toContain('--dangerously-skip-permissions')
})

it('duplicates protected writes in permission and sandbox denies', () => {
  const settings = streamingSettings(buildStreamingClaudeArgs('hello', paths))
  expect(settings.permissions.deny).toEqual(expect.arrayContaining([
    expect.stringContaining('/app/**)'),
    expect.stringContaining('/.git/**)'),
    expect.stringMatching(/Read\(.+\.env/)
  ]))
  expect(settings.sandbox).toMatchObject({
    enabled: true,
    autoAllowBashIfSandboxed: true,
    allowUnsandboxedCommands: false,
    failIfUnavailable: true
  })
  expect(settings.sandbox.filesystem.allowWrite).toEqual([paths.repoRoot])
  expect(settings.sandbox.filesystem.denyWrite).toEqual(expect.arrayContaining([
    paths.appRoot, paths.gitRoot
  ]))
  expect(settings.sandbox.filesystem.denyRead).toContain(paths.homeRoot)
  expect(settings.sandbox.filesystem.allowRead).toEqual(expect.arrayContaining([
    paths.repoRoot, paths.attachmentPaths[0]
  ]))
})

it('keeps the noninteractive goal builder on fixed helper commands', () => {
  expect(buildGoalClaudeArgs('goal')).toEqual(expect.arrayContaining([
    '--permission-mode', 'dontAsk', '--allowedTools', ...CHAT_ALLOWED_TOOLS
  ]))
})
```

- [ ] **Step 2: Run policy tests and confirm red**

Run: `cd app && npx vitest run src/main/__tests__/chatPolicy.test.ts`

Expected: FAIL because the streaming builder does not accept workspace paths and still uses `dontAsk`.

- [ ] **Step 3: Implement separate interactive and fixed-goal policies**

```ts
export interface ChatPolicyPaths {
  repoRoot: string
  appRoot: string
  gitRoot: string
  chatctxRoot: string
  runtimeRoot: string
  homeRoot: string
  attachmentPaths: string[]
}

function absoluteRulePath(path: string): string {
  return `//${path.replace(/^\/+/, '')}`
}

export function buildInteractiveSettings(paths: ChatPolicyPaths): Record<string, unknown> {
  const repo = absoluteRulePath(paths.repoRoot)
  const app = absoluteRulePath(paths.appRoot)
  const git = absoluteRulePath(paths.gitRoot)
  return {
    permissions: {
      allow: [`Read(${repo}/**)`, `Edit(${repo}/**)`, `Write(${repo}/**)`, 'Bash', 'Skill'],
      deny: [
        `Edit(${app}/**)`, `Write(${app}/**)`,
        `Edit(${git}/**)`, `Write(${git}/**)`,
        `Read(${repo}/**/.env*)`, `Edit(${repo}/**/.env*)`, `Write(${repo}/**/.env*)`,
        `Read(${absoluteRulePath(paths.homeRoot)}/.ssh/**)`,
        `Read(${absoluteRulePath(paths.homeRoot)}/.claude/**)`
      ]
    },
    sandbox: {
      enabled: true,
      autoAllowBashIfSandboxed: true,
      allowUnsandboxedCommands: false,
      failIfUnavailable: true,
      filesystem: {
        allowWrite: [paths.repoRoot],
        denyWrite: [paths.appRoot, paths.gitRoot, paths.runtimeRoot, `${paths.repoRoot}/**/.env*`],
        denyRead: [paths.homeRoot],
        allowRead: [paths.repoRoot, ...paths.attachmentPaths]
      },
      network: { allowedDomains: ['mgghhfoppexwemxqvgrn.supabase.co'] }
    },
    disableAllHooks: true,
    autoMemoryEnabled: false
  }
}

export function buildStreamingClaudeArgs(
  prompt: string,
  paths: ChatPolicyPaths,
  resumeId?: string
): string[] {
  const args = ['-p', prompt, '--output-format', 'stream-json', '--verbose', '--include-partial-messages']
  if (resumeId) args.push('--resume', resumeId)
  return [...args,
    '--permission-mode', 'acceptEdits',
    '--setting-sources', 'project',
    '--settings', JSON.stringify(buildInteractiveSettings(paths)),
    '--tools', 'Read,Glob,Grep,Edit,Write,Bash,Skill',
    '--add-dir', paths.repoRoot,
    '--strict-mcp-config', '--disallowedTools', 'mcp__*'
  ]
}
```

Keep `buildGoalClaudeArgs()` on the existing eight helper commands and `dontAsk`; factor its settings into `buildGoalSettings()` rather than broadening the background goal worker.

- [ ] **Step 4: Run red/green policy verification**

Run: `cd app && npx vitest run src/main/__tests__/chatPolicy.test.ts && npm run typecheck:node`

Expected: policy tests PASS; node typecheck exits 0.

- [ ] **Step 5: Commit**

```bash
git add app/src/main/chatPolicy.ts app/src/main/__tests__/chatPolicy.test.ts
git commit -m "feat(chat): sandbox repo agent writes"
```

### Task 3: Add a bounded, atomic main-process runtime store

**Files:**
- Modify: `app/src/shared/types.ts`
- Create: `app/src/main/chatRuntime.ts`
- Test: `app/src/main/__tests__/chatRuntime.test.ts`

- [ ] **Step 1: Define failing state-transition and persistence tests**

```ts
it('sequences text and work events into one snapshot', () => {
  const store = runtimeStore()
  const started = store.begin({ sessionId: 's1', message: 'Analyze', mode: 'analysis', attachments: [] })
  const text = store.appendText('Answer')
  const tool = store.appendWork({ kind: 'tool', label: 'Read recovery plan', detail: 'db.py' })
  expect([started.sequence, text.sequence, tool.sequence]).toEqual([1, 2, 3])
  expect(store.snapshot()).toMatchObject({ phase: 'running', assistantText: 'Answer', lastSequence: 3 })
})

it('bounds work details, entries, partial text, and serialized snapshots', () => {
  const store = runtimeStore()
  store.begin({ sessionId: 's1', message: 'Analyze', mode: 'analysis', attachments: [] })
  for (let i = 0; i < 240; i += 1) store.appendWork({ kind: 'tool', label: `Tool ${i}`, detail: 'x'.repeat(4096) })
  store.appendText('y'.repeat(1024 * 1024 + 100))
  expect(store.snapshot()?.workLog).toHaveLength(200)
  expect(store.snapshot()?.workLog.at(-1)?.detail.length).toBeLessThanOrEqual(2048)
  expect(store.snapshot()?.assistantText.length).toBeLessThanOrEqual(1024 * 1024)
})

it('atomically restores an in-flight run as interrupted', () => {
  const first = runtimeStore(filePath)
  first.begin({ sessionId: 's1', message: 'Analyze', mode: 'analysis', attachments: [] })
  first.appendText('Partial')
  first.flush()
  const restored = runtimeStore(filePath)
  expect(restored.restore()).toMatchObject({ phase: 'interrupted', assistantText: 'Partial' })
})

it('ignores corrupt and unknown-version files', () => {
  writeFileSync(filePath, '{broken')
  expect(runtimeStore(filePath).restore()).toBeNull()
  writeFileSync(filePath, JSON.stringify({ version: 99 }))
  expect(runtimeStore(filePath).restore()).toBeNull()
})
```

- [ ] **Step 2: Run the runtime test and confirm red**

Run: `cd app && npx vitest run src/main/__tests__/chatRuntime.test.ts`

Expected: FAIL because the runtime types/store do not exist.

- [ ] **Step 3: Add the shared runtime contract**

```ts
export type ChatRuntimePhase = 'starting' | 'running' | 'stopping' | 'completed' | 'failed' | 'interrupted'
export interface ChatWorkLogEntry {
  sequence: number
  at: string
  kind: 'status' | 'tool'
  label: string
  detail: string
}
export interface ChatRuntimeSnapshot {
  version: 1
  generationId: string
  sessionId: string
  originalMessage: string
  mode: ChatMode
  attachments: ChatAttachment[]
  startedAt: string
  updatedAt: string
  phase: ChatRuntimePhase
  assistantText: string
  workLog: ChatWorkLogEntry[]
  error: string | null
  resumeAvailable: boolean
  lastSequence: number
}
export interface ChatRuntimeEnvelope {
  generationId: string
  sessionId: string
  sequence: number
  event: ChatStreamEvent
}

// Extend the existing unions/interfaces in the same shared contract:
// ChatStreamEvent gains { kind: 'status'; label: string; detail?: string }.
// ChatMessage gains attachments?: ChatAttachment[].
```

- [ ] **Step 4: Implement `ChatRuntimeStore`**

Implement a dependency-injected class with `restore`, `begin`, `markRunning`, `appendText`, `appendWork`, `markStopping`, `complete`, `fail`, `snapshot`, `flush`, and `dispose`. `flush()` writes `<path>.tmp`, verifies the 2 MiB cap, then renames it. Text writes schedule one 200ms flush; lifecycle transitions flush immediately. `restore()` validates every field, bounds untrusted arrays/strings, and converts nonterminal phases to `interrupted`.

```ts
export class ChatRuntimeStore {
  private current: ChatRuntimeSnapshot | null = null
  private timer: ReturnType<typeof setTimeout> | null = null
  constructor(
    private readonly filePath: string,
    private readonly now = () => new Date(),
    private readonly id = randomUUID
  ) {}

  snapshot(): ChatRuntimeSnapshot | null {
    return this.current ? structuredClone(this.current) : null
  }

  begin(input: BeginRuntimeInput): ChatRuntimeEnvelope {
    const at = this.now().toISOString()
    this.current = {
      version: 1, generationId: this.id(), sessionId: input.sessionId,
      originalMessage: input.message, mode: input.mode, attachments: input.attachments,
      startedAt: at, updatedAt: at, phase: 'starting', assistantText: '', workLog: [],
      error: null, resumeAvailable: false, lastSequence: 0
    }
    return this.emit({ kind: 'status', label: 'Starting' })
  }

  appendText(text: string): ChatRuntimeEnvelope {
    const current = this.requireCurrent()
    current.assistantText = (current.assistantText + text).slice(-MAX_PARTIAL_TEXT_BYTES)
    const envelope = this.emit({ kind: 'text', text })
    this.scheduleFlush()
    return envelope
  }

  flush(): void {
    if (!this.current) return
    const json = JSON.stringify(this.current)
    if (Buffer.byteLength(json) > MAX_RUNTIME_BYTES) throw new Error('chat runtime snapshot exceeded 2 MiB')
    writeFileSync(`${this.filePath}.tmp`, json, { mode: 0o600 })
    renameSync(`${this.filePath}.tmp`, this.filePath)
  }

  // appendWork applies both entry/detail caps before emit(). The phase methods
  // use one private transition helper that cancels the throttle and flushes.
  // restore() assigns current only after validateRuntimeSnapshot() succeeds.
}
```

- [ ] **Step 5: Run runtime tests and shared typechecks**

Run: `cd app && npx vitest run src/main/__tests__/chatRuntime.test.ts && npm run typecheck`

Expected: runtime tests PASS; both typechecks exit 0.

- [ ] **Step 6: Commit**

```bash
git add app/src/shared/types.ts app/src/main/chatRuntime.ts app/src/main/__tests__/chatRuntime.test.ts
git commit -m "feat(chat): persist generation runtime"
```

### Task 4: Wire runtime ownership, replay, continuation, and IPC

**Files:**
- Modify: `app/src/main/chat.ts`
- Modify: `app/src/main/index.ts`
- Modify: `app/src/preload/index.ts`
- Modify: `app/src/shared/types.ts`
- Modify: `app/src/main/__tests__/chatPolicy.test.ts`
- Create: `app/src/main/__tests__/chatIntegration.test.ts`

- [ ] **Step 1: Write failing integration-contract tests**

```ts
it('records before broadcasting and exposes snapshot IPC', () => {
  expect(chatSource).toMatch(/runtime\.appendText\([^)]+\)[\s\S]*webContents\.send/)
  expect(indexSource).toContain('IPC_CHANNELS.chatGetRuntime')
  expect(preloadSource).toContain('chatGetRuntime: () => ipcRenderer.invoke(IPC_CHANNELS.chatGetRuntime)')
})

it('returns generation ownership from send and enforces one active run', () => {
  expect(chatSource).toContain('generationId')
  expect(chatSource).toContain('A chat response is already running')
})

it('supports continue and flushes interruption before quit', () => {
  expect(chatSource).toContain('continueMessage')
  expect(chatSource).toContain('runtime.flush()')
  expect(chatSource).toContain("app.on('before-quit'")
})
```

- [ ] **Step 2: Run the integration test and confirm red**

Run: `cd app && npx vitest run src/main/__tests__/chatIntegration.test.ts`

Expected: FAIL because snapshot/continue IPC and runtime wiring are absent.

- [ ] **Step 3: Extend `IPC_CHANNELS` and `HealthApi`**

```ts
chatGetRuntime: 'chat:get-runtime',
chatContinue: 'chat:continue',
// HealthApi
chatGetRuntime(): Promise<ChatRuntimeSnapshot | null>
chatContinue(sessionId: string): Promise<{ sessionId: string; generationId: string }>
chatSend(...): Promise<{ sessionId: string; generationId: string }>
onChatStream(listener: (payload: ChatRuntimeEnvelope) => void): () => void
```

- [ ] **Step 4: Make `chat.ts` own one generation**

Resolve the source checkout once per send, build `ChatPolicyPaths` from canonical roots plus validated attachments, begin the runtime before spawning, and append each text/tool/status event before broadcasting its returned envelope. Remove `killOtherSessions` and the three-child ceiling in favor of one `activeChild`. Persist the user message only after validation/runtime ownership succeeds. On close, reconcile the latest DB session, save non-empty assistant text once, then mark the runtime completed/failed. `stopMessage` marks stopping before `SIGTERM`.

```ts
export function getChatRuntime(): ChatRuntimeSnapshot | null { return runtime.snapshot() }

export async function continueMessage(window: BrowserWindow, sessionId: string) {
  const interrupted = runtime.snapshot()
  if (!interrupted || interrupted.phase !== 'interrupted' || interrupted.sessionId !== sessionId) {
    throw new Error('There is no interrupted response to continue.')
  }
  const prompt = `Continue the interrupted response. Original request:\n${interrupted.originalMessage}\n\nPartial response already shown:\n${interrupted.assistantText}`
  return sendMessage(window, sessionId, prompt, [], interrupted.mode, { displayMessage: 'Continue interrupted response' })
}
```

- [ ] **Step 5: Register and preload runtime APIs**

```ts
ipcMain.handle(IPC_CHANNELS.chatGetRuntime, () => chat.getChatRuntime())
ipcMain.handle(IPC_CHANNELS.chatContinue, (event, sessionId: string) => {
  const win = BrowserWindow.fromWebContents(event.sender)
  if (!win) throw new Error('no window for chat continuation')
  return chat.continueMessage(win, sessionId)
})
```

Use `IPC_CHANNELS.chatStream` rather than the literal `chat:stream` in main and preload. Restore the runtime after `app.getPath('userData')` is available and synchronously flush/interrupt before the child is killed during `before-quit`.

During startup reconciliation, load the runtime's DB session. If a final assistant message already contains the runtime answer at or after `updatedAt`, mark the runtime completed and render the DB message instead of a stale partial. Otherwise retain the interrupted partial and set `resumeAvailable` from `claude_session_id`, which drives Continue versus Retry.

- [ ] **Step 6: Run focused main/preload tests**

Run: `cd app && npx vitest run src/main/__tests__/chatRuntime.test.ts src/main/__tests__/chatPolicy.test.ts src/main/__tests__/chatIntegration.test.ts && npm run typecheck`

Expected: focused tests PASS; typecheck exits 0.

- [ ] **Step 7: Commit**

```bash
git add app/src/main/chat.ts app/src/main/index.ts app/src/preload/index.ts app/src/shared/types.ts app/src/main/__tests__
git commit -m "feat(chat): own and replay live generations"
```

### Task 5: Persist renderer chat state above tab navigation

**Files:**
- Create: `app/src/renderer/src/chat/chatUiState.ts`
- Create: `app/src/renderer/src/chat/ChatRuntimeProvider.tsx`
- Create: `app/src/renderer/src/chat/__tests__/chatUiState.test.ts`
- Modify: `app/src/renderer/src/App.tsx`
- Test: `app/src/renderer/src/views/__tests__/ChatView.test.ts`

- [ ] **Step 1: Write failing reducer/persistence tests**

```ts
it('keeps independent drafts and attachments per session', () => {
  let state = initialChatUiState()
  state = chatUiReducer(state, { type: 'set-draft', key: NEW_CHAT_KEY, text: 'new draft' })
  state = chatUiReducer(state, { type: 'select', sessionId: 's1' })
  state = chatUiReducer(state, { type: 'set-draft', key: 's1', text: 'follow up' })
  expect(state.drafts).toEqual({ [NEW_CHAT_KEY]: 'new draft', s1: 'follow up' })
})

it('reconciles newer sequenced runtime events and ignores duplicates', () => {
  const running = runtimeSnapshot({ lastSequence: 4, assistantText: 'A' })
  let state = chatUiReducer(initialChatUiState(), { type: 'hydrate-runtime', runtime: running })
  state = chatUiReducer(state, { type: 'runtime-event', envelope: textEnvelope(5, 'B') })
  state = chatUiReducer(state, { type: 'runtime-event', envelope: textEnvelope(5, 'B') })
  expect(state.runtime?.assistantText).toBe('AB')
})

it('validates versioned local storage and falls back on corruption', () => {
  expect(parseChatUiSnapshot('{bad')).toEqual(initialChatUiState())
  expect(parseChatUiSnapshot(JSON.stringify({ version: 99 }))).toEqual(initialChatUiState())
})
```

- [ ] **Step 2: Run renderer state tests and confirm red**

Run: `cd app && npx vitest run src/renderer/src/chat/__tests__/chatUiState.test.ts`

Expected: FAIL because reducer/persistence modules do not exist.

- [ ] **Step 3: Implement the pure versioned reducer**

```ts
export const CHAT_UI_STORAGE_KEY = 'alke-chat-ui:v1'
export const NEW_CHAT_KEY = '__new__'
export interface ChatUiState {
  version: 1
  selectedSessionId: string | null
  drafts: Record<string, string>
  modes: Record<string, ChatMode>
  attachments: Record<string, ChatAttachment[]>
  runtime: ChatRuntimeSnapshot | null
  historyOpen: boolean
  workLogOpen: boolean
  notice: string | null
}
```

Reducer actions are `select`, `new-chat`, `set-draft`, `set-mode`, `set-attachments`, `hydrate-runtime`, `runtime-event`, `set-history-open`, `set-work-log-open`, and `set-notice`. Persistence serializes every field except the main-owned runtime body; only its selected session/generation reference is retained locally.

- [ ] **Step 4: Implement the persistent provider**

`ChatRuntimeProvider` initializes from `localStorage`, subscribes once to `onChatStream`, fetches `chatGetRuntime`, reconciles by generation/sequence, and saves the UI snapshot after reducer changes. It exposes state plus `selectSession`, `newAnalysis`, `setDraft`, `setMode`, `setAttachments`, `send`, `stop`, `continueInterrupted`, and drawer setters. Terminal events invalidate `['chat','sessions']`, `['goals']`, and `['goal-progress']` even while Chat is not rendered.

```tsx
export function ChatRuntimeProvider({ children }: PropsWithChildren): ReactElement {
  const queryClient = useQueryClient()
  const [state, dispatch] = useReducer(chatUiReducer, undefined, loadChatUiState)
  useEffect(() => window.api.onChatStream((payload) => dispatch({ type: 'runtime-event', envelope: payload })), [])
  useEffect(() => { void window.api.chatGetRuntime().then((runtime) => dispatch({ type: 'hydrate-runtime', runtime })) }, [])
  useEffect(() => persistChatUiState(state), [state])
  return <ChatRuntimeContext.Provider value={buildActions(state, dispatch, queryClient)}>{children}</ChatRuntimeContext.Provider>
}
```

`buildActions` is a local factory. Its send path awaits `chatSend` before clearing only the accepted composition's draft and attachments, then selects the returned session. Rejected sends leave the draft and attachments untouched. Stop and continue assert that the runtime belongs to the selected session before invoking IPC.

- [ ] **Step 5: Mount provider above active-tab rendering**

Wrap the complete `.app-shell` return in `<ChatRuntimeProvider>`. Do not render `ChatView` while another tab is active; persistence belongs to the provider, not hidden DOM.

- [ ] **Step 6: Run reducer, source-contract, and type tests**

Run: `cd app && npx vitest run src/renderer/src/chat/__tests__/chatUiState.test.ts src/renderer/src/views/__tests__/ChatView.test.ts && npm run typecheck`

Expected: focused tests PASS; typecheck exits 0.

- [ ] **Step 7: Commit**

```bash
git add app/src/renderer/src/chat app/src/renderer/src/App.tsx app/src/renderer/src/views/__tests__/ChatView.test.ts
git commit -m "feat(chat): persist state across app navigation"
```

### Task 6: Rebuild Chat as a long-form analysis workspace

**Files:**
- Create: `app/src/renderer/src/views/chat/ChatHistory.tsx`
- Create: `app/src/renderer/src/views/chat/ChatWorkLog.tsx`
- Create: `app/src/renderer/src/views/chat/ChatComposer.tsx`
- Modify: `app/src/renderer/src/views/ChatView.tsx`
- Replace: `app/src/renderer/src/views/ChatView.css`
- Modify: `app/src/renderer/src/views/__tests__/ChatView.test.ts`

- [ ] **Step 1: Write failing workspace structure/responsive tests**

```ts
it('uses a compact session header and document column without TabHeader', () => {
  expect(source).not.toContain('<TabHeader')
  expect(source).toContain('chat-session-header')
  expect(source).toContain('chat-document')
  expect(source).toContain('ChatHistory')
  expect(source).toContain('ChatWorkLog')
  expect(source).toContain('ChatComposer')
})

it('collapses both auxiliary rails at narrower widths', () => {
  expect(css).toMatch(/@media \(max-width: 1180px\)[\s\S]*\.chat-history/s)
  expect(css).toMatch(/@media \(max-width: 760px\)[\s\S]*\.chat-worklog/s)
  expect(css).toContain('max-width: 760px')
})

it('never uses decorative gradients or shadows', () => {
  expect(css).not.toMatch(/linear-gradient|radial-gradient|box-shadow/)
})
```

- [ ] **Step 2: Run Chat view tests and confirm red**

Run: `cd app && npx vitest run src/renderer/src/views/__tests__/ChatView.test.ts`

Expected: FAIL on the compact-header/component/responsive contracts.

- [ ] **Step 3: Extract history, work log, and composer components**

`ChatHistory` owns session row rename/delete UI and marks the main-owned runtime session with “Working” or “Interrupted.” It refuses deletion for a nonterminal runtime session and calls `onRequestStop` instead. `ChatWorkLog` renders one inline summary button plus a drawer `<aside aria-label="Work log">`, with chronological entries and `<details>` for truncated detail. `ChatComposer` reads/writes the provider's selected draft/mode/attachments, autosizes the textarea, revalidates restored attachments, and never disables reading the draft while another session runs.

```tsx
<ChatComposer
  draft={draft}
  mode={mode}
  attachments={attachments}
  runningHere={runtime?.sessionId === selectedSessionId && isRunning(runtime)}
  blockedByOtherRun={Boolean(runtime && isRunning(runtime) && runtime.sessionId !== selectedSessionId)}
  onDraftChange={setDraft}
  onModeChange={setMode}
  onAttachmentsChange={setAttachments}
  onSend={send}
  onStop={stop}
/>
```

- [ ] **Step 4: Compose the document-first Chat view**

Use a full-height `.chat-workspace` containing the history panel, `.chat-main`, compact sticky header, scrollable `.chat-document`, and anchored composer. Historical assistant messages render unbubbled Markdown. The current runtime answer renders exactly once for its owning session. The inline work-log disclosure stays visible after completion; interrupted state includes Continue/Retry beside the partial answer. Empty, offline, and disconnected-repo states preserve the history control.

- [ ] **Step 5: Replace CSS with token-only responsive layout**

Use `var(--color-canvas)`, surface/hairline/text tokens, Space Grotesk only for document headings, Inter for body/UI, 720–760px prose measure, 32px minimum controls, and 150–250ms state transitions. Wide layout uses a 208px history rail and optional 248px work-log drawer; below 1180px history becomes an overlay drawer; below 760px both drawers overlay and the document/composer use 16px side padding. Use a solid composer boundary with a hairline, never a gradient, shadow, glow, glass, or accent-colored chrome.

```css
.chat-document-column { width: min(100%, 760px); margin-inline: auto; }
.chat-assistant { font: var(--type-body-md); line-height: 1.68; max-width: 72ch; }
.chat-session-header { position: sticky; top: 0; border-bottom: 1px solid var(--color-hairline); }
.chat-composer { border-top: 1px solid var(--color-hairline); background: var(--color-canvas); }
@media (max-width: 1180px) { .chat-history { position: absolute; transform: translateX(-100%); } }
@media (max-width: 760px) { .chat-document, .chat-composer { padding-inline: var(--space-md); } }
```

- [ ] **Step 6: Run view tests and both typechecks**

Run: `cd app && npx vitest run src/renderer/src/views/__tests__/ChatView.test.ts src/renderer/src/chat/__tests__/chatUiState.test.ts && npm run typecheck`

Expected: focused tests PASS; typecheck exits 0.

- [ ] **Step 7: Commit**

```bash
git add app/src/renderer/src/views/ChatView.tsx app/src/renderer/src/views/ChatView.css app/src/renderer/src/views/chat app/src/renderer/src/views/__tests__/ChatView.test.ts
git commit -m "feat(chat): redesign long-form analysis workspace"
```

### Task 7: Harden accessibility, document the pattern, and audit the UI

**Files:**
- Modify: `DESIGN.md`
- Modify: `app/src/renderer/src/views/ChatView.tsx`
- Modify: `app/src/renderer/src/views/chat/*.tsx`
- Modify: `app/src/renderer/src/views/ChatView.css`
- Modify: relevant focused tests

- [ ] **Step 1: Add failing accessibility/state tests**

```ts
it('uses labeled drawers, coalesced status, and focus-return controls', () => {
  expect(source).toContain('aria-live="polite"')
  expect(historySource).toContain('aria-label="Conversation history"')
  expect(workLogSource).toContain('aria-label="Work log"')
  expect(historySource).toContain("event.key === 'Escape'")
  expect(workLogSource).toContain("event.key === 'Escape'")
})

it('has reduced-motion and visible focus coverage', () => {
  expect(css).toMatch(/:focus-visible/)
  expect(css).toMatch(/@media \(prefers-reduced-motion: reduce\)/)
})
```

- [ ] **Step 2: Run focused tests and confirm red**

Run: `cd app && npx vitest run src/renderer/src/views/__tests__/ChatView.test.ts`

Expected: FAIL for any missing drawer/status/focus behavior.

- [ ] **Step 3: Implement keyboard, focus, status, and Markdown hardening**

Add Escape-to-close, focus return, modal focus containment at overlay breakpoints, semantic heading order, keyboard-scrollable Markdown table wrappers, labeled copy/send/stop/delete controls, decorative `aria-hidden`, and a coalesced polite live region that announces phase/current action rather than token deltas. Ensure every interactive state has default, hover, focus-visible, active, and disabled styling.

```tsx
<button ref={historyTriggerRef} aria-expanded={historyOpen} aria-controls="chat-history" onClick={openHistory}>History</button>
<aside id="chat-history" aria-label="Conversation history" aria-modal={overlay ? true : undefined}>
  <button aria-label="Close conversation history" onClick={closeHistory}><X aria-hidden="true" /></button>
</aside>
<p className="sr-only" aria-live="polite">{runtimeAnnouncement}</p>
```

- [ ] **Step 4: Update the enduring Chat section in `DESIGN.md`**

Document the compact sticky session header, 720–760px document column, responsive history/work-log drawers, persistent anchored composer, unbubbled assistant document, compact user bubble, and neutral status vocabulary. Preserve every existing token and the explicit no-shadow/no-gradient laws.

- [ ] **Step 5: Fetch and apply the current Web Interface Guidelines**

Run: `curl -L https://raw.githubusercontent.com/vercel-labs/web-interface-guidelines/main/command.md`

Review `ChatView.tsx`, `views/chat/*.tsx`, and `ChatView.css` against every current rule. Fix each applicable issue with `apply_patch`; record non-applicable rules in the implementation notes rather than weakening the UI.

- [ ] **Step 6: Run focused tests, lint, and typecheck**

Run: `cd app && npx vitest run src/renderer/src/views/__tests__/ChatView.test.ts src/renderer/src/chat/__tests__/chatUiState.test.ts && npm run lint && npm run typecheck`

Expected: tests PASS; ESLint and both typechecks exit 0.

- [ ] **Step 7: Commit**

```bash
git add DESIGN.md app/src/renderer/src/views app/src/renderer/src/chat
git commit -m "fix(chat): harden workspace accessibility"
```

### Task 8: Full verification, integration, packaging, and handoff

**Files:**
- Modify only files required by failures found during verification.

- [ ] **Step 1: Run the complete automated gate**

Run: `cd app && npm run typecheck && npx vitest run && npm run lint && npm run build`

Expected: 0 TypeScript errors, all Vitest files/tests PASS, 0 ESLint errors, Electron/Vite build exits 0.

- [ ] **Step 2: Run the real permission smoke fixture**

Create a disposable validated repo fixture with `.git/`, `chatctx/`, and `app/`; invoke the built policy with Claude against it. Verify Claude can write/read/delete `chatctx/template-v3.json`, while attempts through both Write and Bash to create `app/forbidden.txt`, `.git/forbidden`, and an outside fixture file fail. Verify no protected file exists afterward.

- [ ] **Step 3: Launch and capture every affected rendered state**

Run: `HEALTH_APP_DISPLAY=external npm run dev -- --remoteDebuggingPort=9333`

Use Playwright/CDP to capture dark and light Chat screenshots at 1440×1000, 1000×800, and 720×800 for empty, long-answer, active-work-log, and interrupted states. Inspect every screenshot for clipped text, squeezed prose, hidden suggestions, composer overlap, unintended color, focus loss, and drawer behavior. Save evidence under `/tmp/alke-chat-verification/`.

- [ ] **Step 4: Exercise live navigation and restart recovery**

Type an unsent draft, switch through Dashboard/Injuries/Gym, and confirm it remains. Start a real response, switch Alke tabs and chat sessions, return, and confirm answer/work-log sequence continuity. Quit mid-response, relaunch, confirm `Interrupted` plus partial answer and Continue/Retry with no false busy state, then complete the continuation.

- [ ] **Step 5: Build and inspect the packaged app**

Run: `cd app && npm run dist:mac`

Expected: fresh arm64 `.app`, `.dmg`, and `.zip` artifacts. Launch the packaged app with the validated owner fallback, repeat one allowed template-file write and one denied `app/` write, then repeat draft/tab persistence and interrupted-restart smoke checks.

- [ ] **Step 6: Re-run the complete gate after any verification fixes**

Run: `cd app && npm run typecheck && npx vitest run && npm run lint && npm run build`

Expected: every command exits 0 with all tests passing.

- [ ] **Step 7: Commit verification fixes**

```bash
git add app DESIGN.md
git commit -m "fix(chat): close workspace verification gaps"
```

- [ ] **Step 8: Integrate and clean up**

From the primary checkout, merge `codex/persistent-agent-workspace` into `main`, run the full gate on merged main, push `main`, remove `.worktrees/persistent-agent-workspace`, delete the merged local branch, confirm `git worktree list` contains only main and `main...origin/main` is clean, then run `npm --prefix app run dist:mac` once more from main so the final package exactly matches the pushed commit.
