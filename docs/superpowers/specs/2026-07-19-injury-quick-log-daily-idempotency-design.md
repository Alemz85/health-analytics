# Injury quick-log daily idempotency

## Problem

The Injuries view derives “today” in the user’s configured timezone, but `addInjuryLog` currently defaults an omitted `entry_date` from UTC. Around local midnight, an optimistic Jul 19 entry can therefore be reconciled with a persisted Jul 18 row. The button becomes available again because the returned row is not “today,” and replacing only the temporary ID can leave repeated references to the same persisted row in the cache.

The database already has the partial unique index `injury_notes_user_daily_unique`, so this is a date-contract and cache-reconciliation bug rather than a missing uniqueness constraint.

## Behavioral contract

For app-authored, single-day injury quick logs (`source = 'user'`, no `entry_end_date`):

- There is at most one row per injury per user-local calendar day.
- Repeating “Feeling fine” on the same day is an idempotent no-op.
- A flare-up submitted later that day replaces the “Feeling fine” row.
- “Feeling fine” never replaces an existing same-day flare-up.
- A later flare-up submission updates the same daily row with the newly submitted details; it does not append another row.
- Chat-authored history notes and period entries remain append-only and outside this rule.

## Data flow

Both quick-log callers—“Feeling fine” and the flare-up form—send the renderer’s explicit `todayYMD` as `entry_date`. This makes the operation stable across IPC, offline queueing, delayed replay, and UTC midnight.

`addInjuryLog` keeps a timezone-aware fallback for callers that omit `entry_date`. It looks up the user’s configured timezone and derives the local date from the current instant instead of slicing the UTC ISO string.

When a same-day row exists, persistence applies the directional precedence rule:

1. Incoming “Feeling fine” keeps the existing row unchanged.
2. An identical flare-up keeps the existing row unchanged.
3. A changed flare-up updates the existing row in place.
4. With no daily row, the incoming quick log is inserted.

The existing partial unique index remains the final concurrency guard.

## Client cache behavior

One pure helper applies a quick log optimistically by replacing the existing user-authored single-day row for that local date, or prepending when none exists. Both the fine action and flare form use it.

A second pure helper reconciles the server result by removing the temporary row and any duplicate cached representation of the same logical daily row before inserting the returned result once. This also repairs duplicate references already produced within the current cache session.

The “Feeling fine” action treats any same-day quick log as already logged. After a fine entry it shows the existing logged state; after a flare-up it remains unavailable, preventing a downgrade.

## Error and offline behavior

Optimistic mutations keep the existing snapshot rollback behavior on permanent failure. A queued offline write retains its explicit `entry_date`, so replay on a later date cannot move the log to another day. Mutation scoping remains per injury.

## Verification

Automated coverage will prove:

- local midnight in a positive UTC offset resolves to the local date;
- repeated fine logs collapse to one optimistic row;
- a flare replaces a same-day fine row;
- a fine log cannot replace a same-day flare;
- server reconciliation leaves one cached daily row;
- both renderer write paths send `entry_date: todayYMD`;
- the action is disabled after any same-day quick log;
- existing analytics and injury tests remain green.

The rendered-flow check will use an isolated fixture or mocked API state to exercise repeated “Feeling fine” submissions and a same-day flare replacement without adding synthetic health entries to the live personal database.

## Non-goals

- Changing chat-authored injury history semantics.
- Adding or replacing database constraints.
- Automatically deleting historical notes.
- Redesigning the Injuries view.
