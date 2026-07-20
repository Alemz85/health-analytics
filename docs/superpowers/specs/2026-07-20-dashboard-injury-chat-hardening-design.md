# Dashboard, Injury, and Chat Hardening Design

## Goal

Finish three focused desktop-app corrections without changing the underlying health-data contracts:

- normalize spacing between the Dashboard's opening cards while preserving their current proportions;
- make the current injury-plan week expose the adherence score and the efficacy dosages already used for historical scoring, and separate recovery-plan navigation from injury-status logging;
- make packaged chat launches independent of Claude Code's path-specific workspace-trust state.

## Dashboard spacing

The glance grid and the Sessions/RHR grid remain separate so their proportions and responsive behavior do not change. A new parent card-stack groups them with `{spacing.md}` between the grids. The surrounding view keeps `{spacing.xl}` between the page header, opening card cluster, and later sections.

This produces a consistent 16px gap between adjacent opening cards in both axes without negative margins, global `.view` overrides, or a risky grid rewrite.

## Current-week injury adherence

The current-week table gains a week-to-date adherence summary. It reuses the existing current-week pace calculation, which compares completed checks with the acceptable dose expected through the elapsed accountable days rather than penalizing an unfinished week against seven full days.

Each accountable exercise header exposes the values behind the score:

- completed checks this ISO week;
- acceptable therapeutic dose (`green_min`, falling back to `weekly_target` for legacy plans);
- minimum-effective dose (`yellow_min`) when assigned;
- prescribed weekly target when it differs from the acceptable dose.

Activities remain visible and checkable but are explicitly unscored. Future-phase items remain visible; early checks remain full-strength and do not affect the current adherence aggregate. When no exercise is accountable, the summary shows an unavailable state rather than `0%`.

## Injury full-view actions

Compact preview cards keep the existing three actions because plan access is useful at a glance. The full view stops reusing that exact action composition:

- the status-logging row contains only “Feeling fine” and “Log flare-up”;
- a neutral “View recovery plan” action sits with the plan-start and current-plan-week metadata;
- resolved injuries keep the same plan access in read-only form;
- lifecycle controls remain together in the existing bottom footer.

This keeps plan navigation available without presenting it as a third kind of injury-status log.

## Packaged Claude CLI policy

Both streaming chat and goal-metric generation use one shared, explicit headless invocation policy:

- the eight existing health-helper command patterns are passed through Claude Code's `--allowedTools` option;
- `--permission-mode dontAsk` makes an unlisted operation fail immediately instead of waiting for an approval that a headless process cannot answer;
- streaming stdin is ignored, and callback-based goal generation closes its child stdin immediately;
- packaged resources include the health routing skill but do not depend on the packaged `.claude/settings.json` permission file.

The app does not edit `~/.claude.json`, does not ask the user to trust an application-bundle path, and does not use permission bypass mode.

## Error handling and safety

Existing renderer mutation rollback and chat stderr reporting remain unchanged. The explicit CLI policy grants only the current health helper commands; generic Bash, Edit, and Write access are not added. The injury changes are derived entirely from data already returned by the current API.

## Verification

- focused red/green tests for Dashboard layout structure, current-week injury dosage/adherence semantics, full-view action placement, and Claude invocation policy;
- full app Vitest suite and TypeScript checks;
- dark/light screenshots of Dashboard and active/resolved Injury full views at desktop and narrow widths;
- a packaged-app chat run that executes an allowed health read without trust or stdin warnings;
- a fresh arm64 app, DMG, and ZIP build with integrity checks.

## Non-goals

- changing Dashboard card proportions;
- changing adherence thresholds or database schema;
- redesigning the recovery-plan modal;
- silently trusting Claude workspaces or broadening Claude's filesystem/shell authority.
