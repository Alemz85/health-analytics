# Persistent Agent Workspace and Scoped Repo Autonomy Design

## Goal

Turn Chat into a durable, long-form analysis workspace and give its local Claude agent enough authority to complete repository-backed health tasks without approval dead ends.

The result must satisfy three properties together:

- moving between Alke tabs does not lose a draft, selected conversation, streamed answer, progress state, or tool activity;
- quitting and reopening Alke restores the useful chat state, while honestly marking a killed generation as interrupted rather than pretending it is still running;
- the agent can read the Sports app repository and write outside `app/`, but cannot modify the desktop application's frontend or escape into the rest of the machine.

This is approach 2 from design exploration: a stateful analysis workspace with a renderer-level runtime provider, a main-process generation registry and bounded snapshot, and an OS-sandboxed Claude invocation. It is intentionally more robust than keeping the current component mounted and intentionally smaller than a durable database event log or unrestricted host agent.

## Authority model

### Repository resolution

The main process resolves one canonical Sports app repository root before starting a chat generation:

1. use an absolute `ALKE_REPO_ROOT` value when configured in the environment loaded by the app;
2. in development, derive the repository from the source `chatctx/` directory;
3. for the owner's packaged installation, try `~/Projects/Github/Sports app` as a convenience fallback;
4. otherwise report an actionable “repository not connected” error instead of widening access or silently treating the packaged resources directory as the source repository.

Every candidate is canonicalized and must contain the expected `.git/`, `chatctx/`, and `app/` markers. The canonical root, not an unchecked environment string or symlink path, is used in the permission and sandbox policy. The packaged `chatctx/` resource remains the instruction/runtime fallback for existing health reads, but repository-editing work only runs against a validated source checkout.

### Agent working directory and tools

Repository-backed chat runs with the canonical repository root available and `chatctx/` as the initial working directory, preserving the current health-mode routing and relative helper commands. The Claude invocation exposes:

- `Read`, `Glob`, and `Grep` across the repository;
- `Edit` and `Write` on permitted repository paths;
- sandboxed `Bash` for ordinary local commands, including Python or Node scripts that create temporary contract files;
- the existing project `Skill` router.

The permission mode becomes edit-accepting for permitted paths, and sandboxed Bash commands are allowed without a headless approval prompt. This fixes the workout-template failure: the agent can create a JSON document, invoke the template contract, verify the result, and remove its temporary file without resorting to a throwaway database write.

The current exact eight-command allowlist is retired for interactive chat because it cannot express safe, multi-step repository work. Goal-metric generation uses the same sandbox foundation but keeps only the capabilities needed by its fixed workflow.

### Hard write boundary

The following policy is enforced by machinery, not only by `CLAUDE.md` prose:

- **Readable:** the validated Sports app repository, except secret files.
- **Writable:** repository paths outside the deny set.
- **Never writable:** the complete canonical `app/` tree, including renderer, preload, main-process, packaging, and app tests.
- **Never writable:** `.git/`, root and nested `.env` files, credential files, and the packaged application/resources tree.
- **Never readable by the model:** `.env`, SSH/keychain/cloud credentials, Claude configuration, browser profiles, and other explicit credential locations. Existing health helpers may continue receiving the narrowly required Supabase values through their process environment.
- **Outside the repository:** no write access is added. The sandbox denies reads from the owner's home directory by default, then uses the more-specific `allowRead` rule for the canonical repository and explicit user-selected attachments. System/runtime reads required to launch sandboxed tools remain available. Attachments retain their existing canonicalization, extension, size, and read-only treatment.

The whole `app/` tree is denied rather than trying to classify individual files as “frontend.” That stricter interpretation is deliberate: it gives the user the requested guarantee that an in-app agent cannot destabilize the running desktop app through a renderer, shared-contract, preload, build-config, or main-process edit.

The boundary is duplicated at both relevant layers:

1. Claude permission rules allow built-in repository reads/edits, explicitly deny built-in `Read`, `Edit`, and `Write` operations on credential/protected paths, and never grant built-in writes outside the repository.
2. Claude's macOS sandbox allows filesystem writes in the canonical repository root, then applies higher-priority write denials for `app/`, `.git/`, and secrets. It pairs a home-directory `denyRead` with more-specific repository and attachment `allowRead` entries. Shell subprocesses inherit the same boundary.

Sandbox startup uses fail-closed behavior. Unsandboxed command escape is disabled. MCP tools and other global tool bypasses remain disabled, and permission-bypass mode is not used. If the OS sandbox is unavailable or any protected path cannot be canonicalized, the generation fails with a visible explanation.

Network access stays limited to the existing Supabase project endpoint required by the health helper scripts. General web access, arbitrary package installation, pushing to remotes, and external messaging are not granted by this change.

## Chat state architecture

### Main-process generation registry

The main process becomes the authority for the live Claude run. It maintains a single active generation record containing:

- a unique generation id and monotonically increasing event sequence;
- the Alke chat session id, original user message, mode, validated attachment metadata, and start time;
- lifecycle state: `starting`, `running`, `stopping`, `completed`, `failed`, or `interrupted`;
- accumulated assistant text;
- a normalized work log of tool use and safe progress/status events;
- the latest error, when present.

Every renderer event is appended to this record before it is broadcast. A new snapshot IPC method returns the current record and the latest event sequence, so a renderer that mounted late or reloaded can reconcile before listening for newer events. Events carry the generation id and sequence number; duplicate or stale events are ignored by the renderer reducer.

Only one chat generation runs at a time. The user may browse other conversations and other Alke tabs while it runs, but sending in a second conversation remains disabled until the active run finishes or is stopped. This makes ownership, recovery, and stop behavior unambiguous.

### Bounded runtime persistence

The main process atomically stores a versioned runtime snapshot under Electron `userData`. Writes are throttled during text streaming and flushed at lifecycle transitions and before quit. The file holds the active or most recently terminal generation, not the complete conversation history.

The work log is normalized and bounded to the latest 200 entries, 2 KiB of detail per entry, and 256 KiB total. The retained partial answer is capped at 1 MiB, and the complete runtime file at 2 MiB; the durable database copy is not truncated when a response completes. Tool inputs are summarized and truncated; raw model chain-of-thought is neither requested nor stored. The retained log is a concise audit trail such as “Read recovery plan,” “Wrote template document,” and the tool/detail rows already exposed by the stream.

On normal completion, the assistant answer remains persisted in the existing chat-session database and the runtime snapshot retains the latest turn's collapsed work log for continuity. On a deliberate stop, any non-empty partial answer is stored with the existing session and labeled as stopped in the runtime record.

If Alke exits during a generation, the child receives `SIGTERM`, the latest snapshot is flushed, and the next launch converts `starting`, `running`, or `stopping` to `interrupted`. The partial answer and work log remain visible, but the UI is not busy and does not claim the old process is alive. A **Continue** action starts a new turn in the same Claude session with the original request and retained partial result identified as interrupted, asking the agent to finish without repeating completed work. If no resumable Claude session id exists, the action becomes **Retry** and resends the original request.

Snapshot reconciliation checks the database before showing a partial answer. If the matching final assistant message already exists, the durable database copy wins and the stale partial is discarded. Corrupt, oversized, or unknown-version snapshot files are quarantined or ignored with a recoverable local error; they never prevent Chat from opening.

### Renderer-owned UI persistence

A `ChatRuntimeProvider` mounts above the active-tab switch in `App`, subscribes to chat events once, and remains alive while the user navigates through Dashboard, Injury, Gym, or any other Alke tab. `ChatView` renders provider state instead of owning the live run.

The provider keeps state per composition/session where appropriate:

- selected conversation or “new analysis” state;
- unsent draft text;
- selected mode for a new conversation;
- validated attachment metadata;
- active answer and work log supplied by the main-process snapshot;
- errors and connection/runtime status;
- history-rail and work-log disclosure preferences.

Drafts are keyed by session id, with a separate key for a new conversation, so inspecting an old conversation does not overwrite the current composition. Selected conversation, drafts, mode, attachments, and disclosure preferences are stored in a versioned local UI snapshot. Restored attachments are revalidated by the main process; missing or changed files are removed with a non-blocking notice.

This creates the following behavior:

- **Switch Alke tabs:** nothing is unmounted at the state/stream layer; typing and generation continue unchanged.
- **Switch chat conversations:** each draft remains intact; the running session carries a visible working indicator in history; its stream continues to accumulate even while another session is open.
- **Reload the renderer:** hydrate the local UI snapshot, fetch the main runtime snapshot, then subscribe from its last sequence.
- **Quit and reopen after an idle session:** restore the selected conversation and drafts; completed messages come from the database.
- **Quit and reopen during a response:** restore the partial response and work log as interrupted, with Continue or Retry.

The database remains the source of truth for completed user and assistant messages. Transient work-log entries do not become chat messages and are not added to Supabase.

## Long-form analysis workspace

### Overall composition

Chat becomes a full-height working surface within the existing Alke shell. It no longer uses a large marketing-style page heading followed by a rounded card floating in empty space. The canvas is divided by hairlines and luminance steps, using the exact `DESIGN.md` tokens and preserving the app's no-shadow, no-gradient, grayscale-clean rules.

At wide desktop sizes the workspace has:

- a slim, collapsible conversation rail;
- a flexible central document area with a 720–760px reading column;
- an optional narrow work-log drawer opened from the session header.

The conversation rail contains one quiet **New analysis** action, search-free chronological history, active/working indicators, and existing rename/delete controls. It is visually subordinate to the document. The work-log drawer is closed by default after a completed answer, remains easy to reopen, and shows the active turn's chronological progress without competing with the answer.

At intermediate widths the conversation rail becomes a drawer. At narrow widths both history and work log are overlay drawers, leaving the document and composer the full usable width. No fixed 256px rail is allowed to squeeze the reading column, and suggestions or composer controls must not clip at any supported window width.

### Session header

A compact sticky session header replaces the oversized Chat tab header. It contains:

- the current conversation title or “New analysis”;
- the fixed mode for an existing Claude session, or the mode picker for a new one;
- a neutral connection/runtime label (`Ready`, `Working`, `Stopped`, `Interrupted`, or `Offline`);
- compact controls for conversation history and work log when those panels are collapsed.

The header uses `{colors.canvas}` or the existing surface ladder, a bottom `{colors.hairline}`, and established label/body tokens. Runtime state is communicated by text and icon as well as motion; no new accent color is introduced.

### Conversation document

Assistant responses remain unbubbled and document-like, as required by `DESIGN.md`, but receive a more deliberate reading treatment:

- `{typography.body-md}` with comfortable measure and paragraph rhythm;
- clear Space Grotesk section headings without billboard sizing;
- restrained list spacing, horizontal table containment, and legible code blocks on `{colors.surface}`;
- stable copy controls and timestamps that do not shift the document on hover;
- generous separation between turns without wrapping each exchange in a card.

User messages remain right-aligned `{colors.surface-elevated}` bubbles with `{rounded.md}`, but become more compact and are capped so they do not dominate the document. Attachments appear as neutral metadata rows beneath the corresponding user message.

During generation, answer text grows in place. The work log is represented by one calm inline disclosure near the active response and by the optional drawer; it does not inject raw tool JSON between answer paragraphs. The disclosure shows a concise current action and an event count. Opening it reveals chronological entries and truncated technical detail. Completion collapses it but does not erase it.

### Composer

The composer is anchored to the bottom of the document pane and aligned with the reading column. It uses the existing `{colors.surface}` input well, `{colors.hairline}`, `{rounded.md}`, and circular white send/neutral stop actions. A solid canvas/surface boundary and hairline separate it from scrolling content; no fade or gradient is used.

The composer supports multi-line long prompts, drag/drop and file selection, attachment chips, keyboard send behavior, and the existing attachment limits. The draft is written to local persistence while typing. While a response is active, the prompt remains readable, the send action becomes Stop for the running conversation, and other conversations clearly explain why sending is temporarily unavailable.

The empty state is compact and useful: a short explanation plus the existing analysis suggestions, laid out so they never sit beneath the composer. Offline and repository-not-connected states occupy the document pane and keep conversation history accessible.

## Failure and edge-case behavior

- A send is accepted by the main process before the renderer performs optimistic state changes. The main process returns the generation and session ids, preventing a failed spawn from leaving an unowned optimistic message.
- A Claude spawn or sandbox failure leaves the user request and attachments recoverable and exposes Retry; it never clears the draft silently.
- A database save failure leaves the locally streamed answer visible with an explicit “not saved” status and retry path.
- Deleting the actively running conversation requires stopping it first. Deleting the selected idle conversation selects the next available session or a new composition without deleting unrelated drafts.
- Renaming a session updates the provider and query cache without disturbing its runtime record.
- A terminal event received after tab navigation still updates the provider, session list, goals queries, and saved runtime snapshot.
- Snapshot size caps prevent an unusually long answer or verbose tool from creating unbounded `userData` growth. The database remains responsible for the complete final answer.
- The policy builder rejects a repository root that resolves through or contains protected-path ambiguity. Permission construction uses canonical absolute paths and is covered by platform-specific tests.

## Accessibility and interaction quality

- Conversation history and work-log drawers have labeled controls, predictable focus return, Escape-to-close behavior, and focus containment when modal at narrow widths.
- Stream status uses a polite live region with coalesced updates; token-by-token answer text is not repeatedly announced.
- Stop, send, history, work-log, copy, rename, and delete controls keep at least 32px pointer targets and visible keyboard focus from `DESIGN.md`.
- Markdown headings retain semantic order. Tables are keyboard-scrollable, code-copy actions are labeled, and decorative activity icons are hidden from assistive technology.
- Loading and working indicators respect `prefers-reduced-motion`; status never depends on animation or color alone.
- Both dark and light themes use existing tokens exclusively. There are no shadows, glows, gradients, decorative accent colors, or reduced-contrast text invented for Chat.

## Verification

### Automated coverage

- policy-builder tests prove that permitted repository reads/writes are exposed, `app/` and `.git/` edits are denied, secrets are unreadable, MCP/bypass paths remain blocked, the sandbox fails closed, and unsandboxed escape is disabled;
- repository-resolution tests cover explicit configuration, development discovery, the validated owner-path fallback, symlinks, missing markers, and protected-path ambiguity;
- main-process runtime tests cover ordered event accumulation, snapshot replay, deduplication, bounded persistence, stop, completion, failure, before-quit interruption, corrupt snapshots, and database reconciliation;
- reducer/provider tests cover Alke-tab unmount/remount, chat-session switching, per-session drafts, active-session indicators, restored attachments, renderer reload, late terminal events, and Continue/Retry recovery;
- component tests cover responsive drawers, compact header states, composer enablement, work-log persistence, offline state, and accessible labels/focus behavior;
- the full app Vitest suite, TypeScript check, and production build remain green.

### Rendered and integration checks

- launch with `HEALTH_APP_DISPLAY=external` and capture Chat in dark and light themes at wide, intermediate, and narrow window sizes;
- inspect empty, long-answer, active-stream, expanded-work-log, interrupted, offline, and repository-not-connected states;
- start a real response, navigate through multiple Alke tabs, return, and confirm the draft/stream/work log never reset;
- switch chat conversations during a real response and confirm the running indicator and eventual completion land in the correct session;
- quit during a real response, relaunch, and confirm an interrupted partial plus Continue/Retry appears without a false loading state;
- in a disposable fixture repository, prove an agent can create and consume a temporary workout-template JSON outside `app/`, while both built-in file tools and Bash fail to write inside `app/`, `.git/`, or outside the repository;
- package the macOS app and repeat the repository-resolution, permission, persistence, and tab-navigation smoke tests against the packaged build.

## Non-goals

- allowing the in-app agent to modify any file under `app/`;
- bypassing Claude permissions, disabling the OS sandbox, or granting broad home-directory access;
- giving the agent general internet access, package-install authority, remote Git push, email, messaging, or other external side effects;
- running more than one Claude generation concurrently;
- syncing drafts or transient work logs across devices;
- storing raw chain-of-thought or every Claude stream event in Supabase;
- changing health calculations, ingestion, database schema, chat mode content, or attachment limits;
- turning Chat into a compact messenger UI or introducing a new visual system outside `DESIGN.md`.
