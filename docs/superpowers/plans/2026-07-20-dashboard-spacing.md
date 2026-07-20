# Dashboard Spacing Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Give every adjacent card in the Dashboard's opening cluster the same 16px tokenized gap without changing card proportions.

**Architecture:** Keep the glance and 12-column grids intact. Wrap them in a flex stack that owns the inter-grid gap, leaving `.view` to separate the opening cluster from page-level sections.

**Tech Stack:** React, CSS custom properties, Vitest source-contract tests, Electron/Vite.

---

### Task 1: Opening card-cluster spacing

**Files:**
- Create: `app/src/renderer/src/views/__tests__/dashboardLayout.test.ts`
- Modify: `app/src/renderer/src/views/DashboardView.tsx`
- Modify: `app/src/renderer/src/views/DashboardView.css`

- [ ] **Step 1: Write the failing layout contract**

Create a Vitest test that reads the TSX and CSS and asserts both grids live inside `dashboard-card-stack`, whose CSS is a column flex container with `gap: var(--space-md)`.

```ts
const source = readFileSync(new URL('../DashboardView.tsx', import.meta.url), 'utf8')
const css = readFileSync(new URL('../DashboardView.css', import.meta.url), 'utf8')
expect(source).toMatch(/dashboard-card-stack[\s\S]*dashboard-glance-grid[\s\S]*dashboard-grid/)
expect(css).toMatch(/\.dashboard-card-stack\s*\{[^}]*display:\s*flex[^}]*flex-direction:\s*column[^}]*gap:\s*var\(--space-md\)/s)
```

- [ ] **Step 2: Prove the test fails**

Run `cd app && npx vitest run src/renderer/src/views/__tests__/dashboardLayout.test.ts`. Expect failure because `dashboard-card-stack` does not exist.

- [ ] **Step 3: Add the minimal wrapper**

Wrap only `.dashboard-glance-grid` and the following `.dashboard-grid` in `<div className="dashboard-card-stack">`. Add:

```css
.dashboard-card-stack {
  display: flex;
  flex-direction: column;
  gap: var(--space-md);
}
```

- [ ] **Step 4: Verify focused and full checks**

Run the focused test, `npm run typecheck`, and `npx vitest run` from `app/`. Expect all to pass.

- [ ] **Step 5: Verify rendered spacing**

Launch with `HEALTH_APP_DISPLAY=external`, capture the Dashboard in dark and light themes at desktop width and around the 1100px/760px breakpoints, and confirm adjacent opening-card gaps remain 16px without proportion changes.
