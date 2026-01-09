<!--
MongoDB Document Metadata:
- Original File Path: kavia-docs/CodeWiki/Specs/FeatureSpecs/roadrescue-admin-panel-ui-screen-spec.md
- Operation: write
- Timestamp: 2026-01-07T12:43:58.681134+00:00
- Restored At: 2026-01-09T13:15:04.413269+00:00
- Task ID: cg9d1ee1eb
-->

# RoadRescue Admin Panel UI and Screen Specification

## Purpose and scope

This document specifies the intended user experience and screen-level requirements for the RoadRescue **Admin Panel** frontend. The intent is to describe the UI in a way that is professional, modern, and business-grade while remaining consistent with the current implementation.

The Admin Panel supports admin authentication, reviewing and approving mechanics, managing requests (status and assignment), configuring fee parameters, and viewing basic analytics.

## Information architecture and navigation model

The top navigation is rendered by `src/components/layout/Navbar.js` and is shown when a `user` is present.

Authenticated admin navigation includes:
- “Dashboard” → `/dashboard`
- “Users” → `/users`
- “Requests” → `/requests`
- “Fees” → `/fees`
- “Analytics” → `/analytics`
- Right-side chip: “Admin”
- “Log out” button that calls `dataService.logout()` and navigates to `/login`.

Unauthenticated users see the brand header but no navigation links.

## Visual style (theme)

Theme constants are defined in `src/theme.js` and match the Ocean Professional palette:
- Primary: `#2563EB`
- Secondary/accent: `#F59E0B`
- Background: `#f9fafb`
- Surface: `#ffffff`
- Text: `#111827`
- Muted text: `#6B7280`
- Border: `#E5E7EB`
- Error: `#EF4444`

The UI uses shared building blocks (`Card`, `Input`, `Button`, `Table`) and consistent layout classes (`.container`, `.hero`, `.h1`, `.lead`, `.grid2`, `.grid4`).

## Screen specifications

### Screen: Login (Admin)

#### Route
`/login`

#### Primary user goal
Authenticate as an admin to access administrative functionality.

#### UI structure
- Hero:
  - Title: “Admin Panel”
  - Lead: “Manage approvals, requests, fees, and basic analytics.”
- Card:
  - Title: “Login”
  - Subtitle: “Demo admin: admin@example.com / password123”
  - Form fields:
    - Email (required; defaults to `admin@example.com`)
    - Password (required; defaults to `password123`)
  - Error region:
    - Shows “Login failed.” or the specific error string
  - Primary action:
    - “Sign in” (disabled while busy; label changes to “Signing in...”)

#### Validation and constraints
- Email is required (non-empty after trim).
- Password must be at least 6 characters.
- After authentication, the user must have role `admin`. Otherwise the page shows: “This portal is for admins only.”

#### Post-conditions
On success, navigate to:
- `/dashboard`

### Screen: Dashboard (Admin KPIs)

#### Route
`/dashboard`

#### Primary user goal
Get a quick overview of platform activity.

#### UI structure
- Hero:
  - Title: “Dashboard”
  - Lead: “Quick view of platform activity.”
- Error region:
  - Displays load failures.
- KPI grid (`.grid4`) with four KPI tiles:
  - Total users (role `user`)
  - Total mechanics (roles `mechanic` and `approved_mechanic`)
  - Open requests (status not equal to “Completed”)
  - Completed requests
- Card “Notes”:
  - Subtitle: “This MVP uses manual forms and basic persistence (mock or Supabase).”
  - Bullet list directing the admin to Users, Requests, and Fees sections.

#### Behaviors
- Loads users and requests concurrently:
  - `dataService.listUsers()`
  - `dataService.listRequests()`

### Screen: User Management

#### Route
`/users`

#### Primary user goal
Approve mechanics before they can fully operate.

#### UI structure
- Hero:
  - Title: “User Management”
  - Lead: “Approve mechanics before they can fully operate.”
- Card:
  - Title: “Users”
  - Subtitle: “Mechanics with approved=false should be reviewed and approved.”
  - Error banner on load/approve failure
  - Table columns:
    - Email
    - Role (with label mapping: `approved_mechanic` displayed as “mechanic (approved)”)
    - Approved (Yes/No)
    - Action:
      - If role is `mechanic` and approved is false: show “Approve” button (busy label “Approving...”)
      - Otherwise: show em dash

#### Behaviors
- Loads via `dataService.listUsers()`.
- Approves via `dataService.approveMechanic(id)` and reloads the list.

### Screen: Request Management

#### Route
`/requests`

#### Primary user goal
Manage request status and assignment and close cases.

#### UI structure
- Hero:
  - Title: “Request Management”
  - Lead: “Reassign requests, change status, or close cases.”
- Card:
  - Title: “All requests”
  - Subtitle: “Edits are saved per-row.”
  - Error banner for load or save failures
  - Table columns:
    - Request: short id (non-link in the current table)
    - Created: locale date/time from `createdAt`
    - Vehicle: displayed as `${r.vehicle.make} ${r.vehicle.model}`
    - Status: displayed as a colored pill
    - Customer: `userEmail`
    - Edit: inline editor block containing:
      - Status dropdown with a fixed status option list:
        - Submitted, In Review, Assigned, In Progress, Completed, Accepted, En Route, Working
      - Assign mechanic dropdown:
        - “Unassigned”
        - A list of approved mechanics (role `approved_mechanic` or `mechanic` with approved flag)
      - Action buttons:
        - “Save” (busy label “Saving...”)
        - “Close (Completed)” which sets status to Completed
- Secondary section:
  - Card “Quick reassign helper”
  - Subtitle: “Paste request ID and mechanic email to update faster (mock workflow).”
  - Form:
    - Request ID (required)
    - Mechanic email (optional)
    - Status dropdown (same status set)
    - “Update” button with busy label “Updating...”
    - Inline success (“Updated.”) and error messages

#### Behaviors
- Loads requests and users concurrently:
  - `dataService.listRequests()`
  - `dataService.listUsers()`
- Initializes per-row editor state with current status and assigned mechanic id.
- Saving a row calls:
  - `dataService.updateRequest(req.id, { status, assignedMechanicId, assignedMechanicEmail })`
- Closing a request calls:
  - `dataService.updateRequest(req.id, { status: "Completed" })`
- Quick reassign resolves mechanic by email against approved mechanics and then calls:
  - `dataService.updateRequest(requestId, { assignedMechanicId, assignedMechanicEmail, status })`

### Screen: Fee Settings

#### Route
`/fees`

#### Primary user goal
Manage numeric fee parameters stored by the data layer.

#### UI structure
- Hero:
  - Title: “Fee Settings”
  - Lead: “Simple numeric parameters stored in the data layer.”
- Card:
  - Title: “Fees”
  - Subtitle: “Used for pricing logic later (MVP stores values only).”
  - Fields:
    - Base fee ($) (required, numeric)
    - Per-mile fee ($) (required, numeric)
    - After-hours multiplier (required, numeric; hint “Example: 1.25 for +25%.”)
  - Feedback:
    - Success message “Fees saved.”
    - Error messages for validation and persistence failures
  - Primary action:
    - “Save fees” (disabled while busy; label changes to “Saving...”)

#### Validation rules
- Base fee must be a valid number >= 0.
- Per-mile fee must be a valid number >= 0.
- After-hours multiplier must be a valid number >= 1.0.

#### Behaviors
- Loads existing fees on mount via `dataService.getFees()`, falling back to defaults if retrieval fails.
- Saves via `dataService.setFees({ baseFee, perMile, afterHoursMultiplier })`.

### Screen: Analytics

#### Route
`/analytics`

#### Primary user goal
Review simple analytics about requests without external chart libraries.

#### UI structure
- Hero:
  - Title: “Analytics”
  - Lead: “Lightweight analytics with simple visual bars.”
- Error banner for load failures
- Two-column grid:
  - Card “Requests per day”
    - Subtitle: “Counts by created date.”
    - Displays a simple bar row per day bucket based on `createdAt`
    - Empty state: “No data”
  - Card “Status breakdown”
    - Subtitle: “Most common statuses first.”
    - Displays bar rows by status frequency
    - Empty state: “No data”
- Secondary section:
  - Card “Recent activity”
  - Subtitle: “Latest requests table.”
  - Table showing up to 20 requests:
    - Request short id
    - Created
    - Status
    - Customer

#### Behaviors
- Loads requests via `dataService.listRequests()`.
- Buckets dates by day using a local bucketing helper and builds proportional bar widths relative to the maximum bucket size.

## Non-functional UI requirements

## Consistency and professionalism
The admin UI should maintain consistent spacing, readable typography, and predictable placement of actions. The inline per-row editor must remain legible without introducing clutter; controls should wrap appropriately on small screens.

## Responsiveness
The `.grid2` layout for analytics should stack on smaller viewports. Tables should remain readable; horizontal overflow should be handled by the table component/container.

## Status taxonomy
The Admin Panel recognizes a superset of statuses that span user, mechanic, and administrative flows. The UI should treat the status strings as authoritative enumerations for filtering and display consistency.
