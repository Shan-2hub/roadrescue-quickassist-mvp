# RoadRescue Design System and UI/UX Specification (Ocean Professional)

## Purpose

This specification defines a single, professional, SaaS-grade design system and UI/UX standard for the RoadRescue QuickAssist platform. The intent is to align the three React SPAs:

- User Website (mobile-first)
- Mechanic Portal (mobile-first)
- Admin Panel (desktop-first)

The design system documented here is grounded in the current codebase styling baseline (“Ocean Professional”), which is implemented primarily in each app’s `src/App.css` and supported by shared UI components under `src/components/ui/`.

## Scope and principles

The scope covers foundational tokens, layouts, component usage patterns, interaction states, responsive behavior, and cross-app alignment guidance. The system prioritizes:

- Visual consistency across apps while allowing density/layout differences per audience.
- High clarity and fast scanning (especially for tables and dashboards).
- Predictable states (loading, empty, error, success) and consistent status taxonomy presentation.
- Accessibility fundamentals: label associations, focus visibility, readable contrast, and touch targets.

## Current implementation baseline (“Ocean Professional”)

All three apps currently implement a closely matched CSS token set via `:root` variables and global classes in `src/App.css`. The most important shared primitives are:

- CSS variables (colors, radii, shadows, focus ring)
- Layout classes (`.app-shell`, `.container`, `.hero`, `.grid2`, `.grid4`, `.row`)
- Components styled by class (`.card`, `.btn`, `.input`, `.table`, `.badge`, `.alert`, `.modal`)

While each app has a `src/theme.js` export, styling is primarily driven by CSS variables in `App.css`. The React components (`Button`, `Card`, `Input`, `Table`, `Modal`) generally emit the standardized class names that bind into the global CSS.

## Design tokens

### Color tokens

The design system uses a “blue primary + amber accent” palette, with a neutral surface and subtle radial background gradients.

#### Core palette (as implemented)

These are the canonical tokens used by the current CSS:

- Primary: `#2563EB`
- Secondary (accent): `#F59E0B`
- Success (currently mapped to secondary): `#F59E0B`
- Error: `#EF4444`
- Background: `#f9fafb`
- Surface: `#ffffff`
- Surface-2: `#fbfdff`
- Text: `#0f172a` (note: `theme.js` uses `#111827`, but CSS uses `#0f172a` as the real baseline)
- Muted text: `#64748b` (note: `theme.js` uses `#6B7280`, but CSS uses `#64748b`)
- Border: `#E5E7EB`
- Border strong: `#d1d5db`
- Focus ring: `0 0 0 4px rgba(37,99,235,0.14)`

#### Status and semantic colors

Status presentation is expressed with badge variants:

- Blue badge: “informational / early-stage” states (e.g., Submitted, In Review)
- Amber badge: “active / in-progress” states (e.g., Accepted, En Route, Working)
- Green badge: “done / success” states (e.g., Completed)

These badges are implemented through `.badge-blue`, `.badge-amber`, `.badge-green`.

### Typography

Typography is implemented via the body font stack:

`Inter, -apple-system, BlinkMacSystemFont, "Segoe UI", Roboto, Helvetica, Arial, sans-serif`

The system emphasizes a modern SaaS feel with strong weight for headings and labels:

- Page title (`.h1`): 30px, tight line height, negative letter spacing
- Card title (`.card-title`): 16px, heavy weight
- Label (`.label`): 12px, uppercase, heavy weight, spaced letter tracking
- Body base: 14–15px (depending on element), line-height ~1.45–1.5
- Muted text uses `--muted`

### Spacing and layout scale

The system is built on small, consistent increments:

- Card paddings: 16–18px
- Grid gaps: 12px
- Page main padding: `28px 0 56px`
- Hero padding: `16px 0 12px`

This yields a compact, professional SaaS density while remaining touch-friendly on mobile.

### Radius and shadows

- Default radius: `--radius: 14px`
- Small radius: `--radius-sm: 12px`
- Shadow small: `--shadow-sm: 0 1px 2px rgba(0,0,0,0.06)`
- Shadow large: `--shadow: 0 16px 40px rgba(17,24,39,0.12)`

Buttons and cards use `--shadow-sm` for subtle elevation.

## Layout system

### App shell

All apps implement a similar root layout:

- `.app-shell` provides min-height and flex column layout.
- `.main` provides vertical padding for the content region.
- `.container` constrains line length and ensures consistent horizontal padding.

Container widths differ by app:

- User Website: `min(1040px, calc(100% - 32px))`
- Mechanic Portal: `min(1140px, calc(100% - 32px))`
- Admin Panel: `min(1280px, calc(100% - 32px))`

This divergence is acceptable and intentionally reflects the audience:
- User/Mechanic are form-driven and benefit from narrower, focused reading widths.
- Admin is data-dense and benefits from wider tables and KPI grids.

### Page header (“Hero”)

Most pages follow a predictable pattern:

- `.hero` wrapper
- `.h1` page title
- `.lead` supporting text

This structure should remain consistent across all apps to reduce cognitive friction.

### Grids

The system defines responsive grids:

- `.grid2`: 2-column grid with a 12px gap and mobile collapse
- `.grid4`: Admin-only grid for KPIs with responsive reduction on narrower screens

Mobile responsiveness is handled by media queries in each `App.css`:
- User and Mechanic collapse `.grid2` at 720px
- Admin collapses `.grid4` at 920px and further at 560px, and collapses `.grid2` at 560px

## Core UI components

This section documents how the shared components are expected to behave and how they are styled today.

### Button

Implemented in each app under `src/components/ui/Button.js` and styled by:

- `.btn`
- `.btn-primary`, `.btn-secondary`, `.btn-danger`, `.btn-ghost`
- `.btn-md`, `.btn-sm`

Interaction behavior:
- Hover lifts slightly via transform
- Disabled reduces opacity and removes transform
- Primary hover adds a stronger shadow

Usage guidance:
- Primary is the default for the main action on a screen (submit, save, sign in).
- Secondary is used for “positive but not primary” actions, or where emphasis should be lower.
- Ghost is used for secondary navigation or dismiss actions (e.g., modal close).
- Danger should be used for destructive actions only (rare in this MVP).

### Card

Implemented as `src/components/ui/Card.js`, emitting:

- `.card`
- optional `.card-header` with `.card-title`, `.card-subtitle`, `.card-actions`
- `.card-body`

Usage guidance:
- Every primary page section should typically be a card for predictable hierarchy.
- Use `actions` for contextual secondary actions such as “New request” or “Back to login”.

### Input

Implemented as `src/components/ui/Input.js`, emitting:

- `.field`, `.label`, `.req`, `.hint`, `.error`
- `.input` and optionally `.input-error`

Usage guidance:
- Always pass a stable `name` to bind label → input (accessibility and click-target).
- Prefer inline validation messaging using the existing `.error` and `.input-error` pattern.
- Use hint text sparingly; reserve it for non-obvious constraints and examples.

### Table

Implemented as `src/components/ui/Table.js`, emitting:

- `.table-wrap` for horizontal overflow handling
- `.table` for standardized table look-and-feel
- `.table-empty` for empty state

Constraints:
- The table uses a minimum width and will scroll horizontally on smaller screens, which is acceptable for the MVP. Ensure critical columns appear earlier so the first columns remain visible.

### Modal (User Website only)

The user website includes `src/components/ui/Modal.js` and corresponding modal CSS in `App.css`:

- `.modal-backdrop`, `.modal`, `.modal-header`, `.modal-title`, `.modal-body`, `.modal-footer`

Behavior:
- Renders only when `open` is true.
- Closes on Escape.
- Backdrop click handling is implemented in a simplified way and should remain consistent if ported to other apps.

Guidance:
- Use modals for confirmation or short workflows. Avoid embedding complex forms inside modals for mobile-first flows unless necessary.

## Shared states and messaging patterns

### Alerts

Alerts are standardized:

- `.alert` base
- `.alert-error` for failures
- `.alert-info` for non-blocking informational states

Use cases:
- Authentication errors
- Data loading failures
- Mechanic “Pending approval” banner
- Admin load/save errors

Guidance:
- Error text should be concise, actionable, and avoid internal jargon.
- When possible, include a recovery action (retry, back link) near the alert.

### Loading states

The system uses `.skeleton` blocks in some screens (for example, request detail pages). This is an accepted baseline.

Guidance:
- For data tables, keep a consistent behavior: either show skeleton rows or show a loading indicator area above the table. Do not oscillate between patterns within the same app.

### Empty states

Tables render “No data” via `.table-empty`. For cards or panels, prefer a short sentence and a direct next step (for example, “No requests yet. Create your first request.”) while retaining the same typographic hierarchy.

## Status taxonomy alignment

Each app has status utilities in `src/services/statusUtils.js` that normalize and label statuses, and return CSS classes used for badges.

Cross-app alignment requirement:
- The three apps must display statuses consistently as a shared taxonomy, with consistent color semantics.

A practical alignment approach for the current MVP is:

- Early pipeline / queue: Submitted, In Review → blue
- Active work: Assigned, In Progress, Accepted, En Route, Working → amber
- Done: Completed → green

If an app currently supports a subset of statuses, it should still render unknown statuses as a neutral default badge, but the canonical list should remain consistent in admin workflows.

## Cross-app alignment guidance

### What must be identical across all apps

The following must be consistent to maintain a single product identity:

- CSS variable names and semantic meaning in `:root` (colors, radii, focus ring, borders).
- Base typography stack and use of muted vs text colors.
- Card, button, input, table, badge, and alert styling classes and meaning.
- Page structure pattern: hero header, then one or more cards.

### What can differ (by product needs)

Differences are allowed when they serve the user and do not break brand consistency:

- Container max width (already differs by app).
- Density and number of columns shown in tables (admin is data-dense).
- Admin-only KPI tile styling and dashboard grid.

### Mobile-first guidance (User Website and Mechanic Portal)

For the User and Mechanic experiences, the system should prioritize:

- Single-column stacking at mobile widths with clear vertical rhythm.
- Large enough touch targets (current button/input padding is appropriate).
- Avoid multi-column forms unless the grid collapses cleanly (the `.grid2` does this at 720px).
- Make secondary links visible but not visually dominant (use `.link`).

### Desktop-first guidance (Admin Panel)

The Admin panel should prioritize:

- Higher information density, with wider container and stable table layouts.
- Clear separation of actions, filters, and editor blocks in request management.
- Wrapping behavior for row-level editors to avoid overflow and keep controls accessible.
- Maintain table scroll behavior for very wide tables, but keep primary identifiers and status visible early in the column order.

## Implementation notes for engineers

### Canonical source of design tokens

In the current codebase, the canonical styling is defined in each app’s `src/App.css`. While `src/theme.js` exists, it currently diverges slightly from CSS token values (notably `text` and `mutedText`). For strict alignment, the product should treat the CSS `:root` values as authoritative unless the codebase is refactored to unify tokens.

### Component contract stability

The UI components (`Button`, `Card`, `Input`, `Table`, `Modal`) use class names that map directly into global CSS. Any future refactors should preserve these class names or introduce a compatibility layer to avoid cross-app divergence.

### Navigation, header, and footer

The global nav and footer styling is consistent via `.navbar*` and `.footer*` classes. Each app’s `Navbar` differs in links (because the information architecture differs), but the styling and behavior should remain consistent.

## UX consistency rules for key screens

This section describes consistency rules that should be applied when refining screens.

### Authentication screens

Across all apps, login should follow the same visual pattern:

- Hero title and lead describing the portal.
- One primary card containing the form.
- One clear primary submit button.
- Errors shown as `.alert.alert-error` at the top of the card.

### List screens (requests, assignments, users)

- Use a card wrapper with title/subtitle.
- Put primary action in `Card` actions region (e.g., “New request”).
- Use `Table` for lists and badge styling for statuses.
- Provide a clear empty state and a next step.

### Detail screens

- Hero title includes short id.
- Hero lead displays current status.
- Use `.grid2` to present detail cards; ensure it collapses to one column on mobile.
- Provide a “Back” link at the end of the primary content.

## Known gaps and alignment risks (current state)

- `theme.js` token values do not perfectly match `App.css` tokens. This can cause confusion if engineers reference `theme.js` expecting it to drive UI visuals.
- Modal is only present in the user app; if future UI patterns require modals in mechanic/admin, the component and CSS should be promoted consistently.
- Table min-width differs across apps, which is acceptable; however, key columns should stay consistent in order and meaning to reduce mental switching.

## Appendix: File map of current design system implementation

### User Website

- Global styling tokens and utility classes: `frontend_user_website/src/App.css`
- Minimal globals: `frontend_user_website/src/index.css`
- Theme object (not primary styling source): `frontend_user_website/src/theme.js`
- UI components:
  - `frontend_user_website/src/components/ui/Button.js`
  - `frontend_user_website/src/components/ui/Card.js`
  - `frontend_user_website/src/components/ui/Input.js`
  - `frontend_user_website/src/components/ui/Table.js`
  - `frontend_user_website/src/components/ui/Modal.js`

### Mechanic Portal

- Global styling tokens and utility classes: `frontend_mechanic_portal/src/App.css`
- Minimal globals: `frontend_mechanic_portal/src/index.css`
- Theme object (not primary styling source): `frontend_mechanic_portal/src/theme.js`
- UI components:
  - `frontend_mechanic_portal/src/components/ui/Button.js`
  - `frontend_mechanic_portal/src/components/ui/Card.js`
  - `frontend_mechanic_portal/src/components/ui/Input.js`
  - `frontend_mechanic_portal/src/components/ui/Table.js`

### Admin Panel

- Global styling tokens and utility classes: `frontend_admin_panel/src/App.css`
- Minimal globals: `frontend_admin_panel/src/index.css`
- Theme object (not primary styling source): `frontend_admin_panel/src/theme.js`
- UI components:
  - `frontend_admin_panel/src/components/ui/Button.js`
  - `frontend_admin_panel/src/components/ui/Card.js`
  - `frontend_admin_panel/src/components/ui/Input.js`
  - `frontend_admin_panel/src/components/ui/Table.js`
  - KPI styles are defined in `frontend_admin_panel/src/App.css` under `.kpi*`

## Relationship to existing screen specifications

This design system is intended to be used in conjunction with the app-specific UI/screen specifications:

- `kavia-docs/roadrescue-user-website-ui-screen-spec.md`
- `kavia-docs/roadrescue-mechanic-portal-ui-screen-spec.md`
- `kavia-docs/roadrescue-admin-panel-ui-screen-spec.md`

Those documents define what each screen does. This document defines how screens should look and behave in a consistent, SaaS-grade manner.
