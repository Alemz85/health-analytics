# Injury Current-Week Adherence Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Show current-week adherence and its acceptable/minimum/prescribed dosages, and move full-view recovery-plan access out of the status-logging row.

**Architecture:** Add one pure summary model in `injuryStats.ts`, render that model above and within `ThisWeekTable`, and make full-view action composition explicit while leaving preview-card actions unchanged.

**Tech Stack:** React, TanStack Query, TypeScript, Vitest, CSS tokens.

---

### Task 1: Current-week summary model

**Files:**
- Modify: `app/src/renderer/src/lib/injuryStats.ts`
- Modify: `app/src/renderer/src/lib/__tests__/injuryStats.test.ts`

- [ ] **Step 1: Write failing behavioral tests**

Add tests for `currentWeekAdherenceSummary()` asserting: thresholded exercises expose distinct done/acceptable/minimum/prescribed counts; aggregate percentage uses elapsed accountable days; duplicate same-day checks count once; activities are returned as unscored; future-only plans return `pct: null`.

```ts
expect(currentWeekAdherenceSummary(items, checks, TODAY, planStart)).toMatchObject({
  pct: 100,
  items: [{ itemId: 'ankle', done: 4, acceptable: 5, minimum: 3, prescribed: 7, scored: true }]
})
```

- [ ] **Step 2: Prove the tests fail**

Run `cd app && npx vitest run src/renderer/src/lib/__tests__/injuryStats.test.ts`. Expect an import/export failure for the missing helper.

- [ ] **Step 3: Implement the pure model**

Export `CurrentWeekDoseSummary`, `CurrentWeekAdherenceSummary`, and `currentWeekAdherenceSummary()`. Reuse `weeklyAdherence()` for the pace-adjusted aggregate, `checkedDays()` for distinct ISO-week completions, `doseTarget()` for acceptable dose, and `isPlanItemAccountable()` for phase state. Return only active exercise/activity rows, and return `pct: null` when no targeted exercise is currently accountable.

- [ ] **Step 4: Verify the analytics tests pass**

Run the focused injury-stat test and expect all cases to pass.

### Task 2: Current-week presentation

**Files:**
- Modify: `app/src/renderer/src/views/InjuriesView.tsx`
- Modify: `app/src/renderer/src/views/InjuriesView.css`
- Modify: `app/src/renderer/src/views/__tests__/injuryAdherenceHeader.test.ts`
- Modify: `app/src/renderer/src/views/__tests__/injuryAdherenceGrid.test.ts`

- [ ] **Step 1: Write failing renderer contracts**

Assert `ThisWeekTable` calls `currentWeekAdherenceSummary`, renders `Week-to-date adherence`, exposes `acceptable`, `minimum`, and `prescribed` labels, and continues preserving full-strength early checks in future cells.

- [ ] **Step 2: Prove the renderer tests fail**

Run the two focused view tests. Expect failures for the missing summary/labels.

- [ ] **Step 3: Render the summary and dosage model**

Add a compact summary row above the table with a threshold-colored percentage chip or `Not yet scored`. In exercise headers render `done/acceptable acceptable`, plus `minimum N` and `prescribed N` only when available and non-duplicative. Mark activities `Unscored` while retaining their progress text.

- [ ] **Step 4: Add tokenized responsive styling**

Use existing surface/text/radius/spacing variables. Keep the table horizontally scrollable and allow the summary row to wrap at narrow widths.

- [ ] **Step 5: Verify focused tests and typecheck**

Run the focused view tests and `npm run typecheck`; expect success.

### Task 3: Full-view recovery-plan placement

**Files:**
- Modify: `app/src/renderer/src/views/InjuriesView.tsx`
- Modify: `app/src/renderer/src/views/InjuriesView.css`
- Modify: `app/src/renderer/src/views/__tests__/injuryCardActions.test.ts`

- [ ] **Step 1: Write failing placement tests**

Assert preview cards retain the Recovery plan action, active full-view status actions omit it, and active/resolved full views render `View recovery plan` in `injury-plan-access-row` beside plan metadata.

- [ ] **Step 2: Prove the placement tests fail**

Run the focused action test and expect the new placement assertions to fail.

- [ ] **Step 3: Implement the placement**

Make the status action row able to omit its plan button. Use that form only in the full view. Add a neutral plan-access row containing `PlanStartControl` when active plan data exists and an always-available `View recovery plan` button. Remove the resolved-only top action row.

- [ ] **Step 4: Verify injury behavior**

Run all injury renderer/stat tests, typecheck, and the full Vitest suite. Capture active and resolved full views in dark/light themes at 1440×900 and 900×600.
