# Alke Rebrand Design

## Goal

Replace the generic “Health Analytics” / “Health” identity and ECG-wave icon with the approved **Alke** name and **Three Flows** mark. This is a brand-only change. The app remains the same private personal-health analytics product and keeps its existing design system, behavior, data contracts, and navigation.

## Brand idea

Alke (`AL-kay`) is the Greek personification of prowess, courage, and battle-strength. The product interpretation is durable personal capability: the strength to train, recover, adapt, and make informed decisions. It is not positioned as aggressive, competitive, military, or clinical.

The product name is written **Alke** in prose and system metadata. The display wordmark is lowercase **alke**. The functional descriptor is **personal performance intelligence**. The optional external brand line is **Strength, in context.** The descriptor and brand line do not appear repeatedly in the working dashboard; they are reserved for identity contexts such as documentation, packaging, or a future About surface.

The name had a preliminary web collision scan during design. A small product named “ALKE Performance Fitness” exists. This does not block the private personal-tool rebrand, but a public launch requires a formal trademark, company-name, domain, and app-store clearance before treating the name as commercially exclusive.

## Three Flows mark

The approved mark consists of three parallel, rounded trajectories moving from left to right. Each line rises, dips, and resolves upward. The three paths suggest training stimulus, current body state, and adaptation over time without assigning a permanent one-to-one meaning to an individual path.

The geometry is abstract and instrument-like. It must not be embellished with a heart, ECG trace, athlete silhouette, Greek column, helmet, shield, laurel wreath, or mythology illustration. Stroke ends and joins are round. The paths retain generous negative space and remain distinguishable at small sizes.

The primary app-icon treatment places the full-color mark on a black square icon field with the platform-appropriate rounded mask:

- top path: `#FFFFFF`;
- middle path: `{colors.aerobic}` (`#2DD4BF`);
- bottom path: `{colors.load}` (`#6366F1`);
- icon field: visually consistent with the black app canvas, using `#050607` only where a slight separation from a surrounding pure-black field is required.

No gradient, glow, shadow, bevel, texture, or extra brand color is introduced. The icon source must be maintained as vector geometry and exported to the PNG and ICNS assets required by Electron packaging. Exports must remain legible at 16, 32, 64, 128, 256, 512, and 1024 pixels.

## In-app identity

The expanded sidebar’s existing brand slot becomes a compact lockup containing the Three Flows micro-mark and lowercase `alke` wordmark. It continues to use Space Grotesk at weight 500, `{colors.text}`, and the current spacing scale. The lockup must fit the existing sidebar width and vertical rhythm without moving navigation items or changing the sidebar’s padding.

Inside analytical UI, the mark is monochrome: white in dark mode and black in light mode. This preserves `DESIGN.md`’s rule that accent colors communicate metric domains rather than decorate app chrome. The full-color mark is limited to the app icon, packaging, documentation identity, and any future About surface. At the existing collapsed-sidebar breakpoint, the entire brand lockup remains hidden, matching current behavior.

The renderer document title, packaged product display name, and user-visible app/window naming change to **Alke**. Internal identifiers may remain unchanged when changing them would move the Electron `userData` directory, lose the packaged `.env`, reset preferences, or otherwise break continuity. If an internal identifier must change, implementation must migrate the old application-support data safely before switching to it.

## Existing design system remains authoritative

`DESIGN.md` continues to govern every UI decision. The rebrand does not alter:

- Space Grotesk or Inter, their weights, sizes, tracking, or hierarchy;
- canvas, surface, text, semantic accent, or light-theme tokens;
- cards, charts, navigation, buttons, inputs, radii, spacing, or shadows;
- theme behavior, responsive breakpoints, animations, or interaction patterns;
- tab names, information architecture, health models, data, or copy outside identity references.

The logo does not create a sixth accent family. It does not authorize colored navigation chrome or decorative reuse of the full-color mark inside dashboard views.

## Identity replacement surface

Implementation replaces user-visible legacy identity references, including:

- `Health Analytics` in Electron packaging metadata and the renderer title;
- `Health` in the expanded sidebar brand slot;
- the current teal ECG-wave app icon in build and runtime icon assets;
- the top-level README product heading and packaged-app instructions where they name the product.

Generic prose describing Apple Health, health data, or health analytics remains unchanged when it describes functionality rather than the old product name. Internal package names, database identifiers, Supabase resources, environment variables, and migrations are not renamed merely for brand consistency.

## Accessibility and rendering

The full lockup has accessible text from the visible `alke` wordmark; the adjacent decorative mark is hidden from assistive technology. Icon-only contexts use the accessible label **Alke** where the platform supports one. The monochrome mark must remain visible against both `{colors.canvas}` themes, and no meaning depends on distinguishing teal from indigo.

The mark must be visually inspected rather than approved from source alone. Verification includes the packaged macOS icon in Finder and the Dock, the expanded sidebar in both themes, the renderer/window title, and small-size raster exports. The sidebar check covers normal desktop width and the existing collapsed breakpoint to confirm that navigation placement and responsive behavior are unchanged.

## Verification

- TypeScript typecheck and the full app Vitest suite.
- Electron/Vite production build.
- Dark- and light-theme screenshots of the expanded sidebar.
- A narrow-width screenshot confirming the existing collapsed-sidebar behavior.
- Visual inspection of icon exports at every required raster size.
- Packaged macOS app inspection in Finder and the Dock.
- A search for user-visible `Health Analytics` and sidebar `Health` identity references, with functional Apple Health prose intentionally retained.
- Confirmation that existing application-support data, credentials, and preferences remain available after launching the renamed packaged app.

## Non-goals

- redesigning any view, card, chart, component, or interaction;
- changing typography, colors, layout, spacing, or responsive behavior;
- adding a splash screen, About screen, marketing page, or repeated tagline;
- changing health calculations, ingestion, database schema, chat behavior, or user data;
- claiming trademark ownership or commercial exclusivity without formal clearance.
