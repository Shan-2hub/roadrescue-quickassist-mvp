<!--
MongoDB Document Metadata:
- Original File Path: kavia-docs/CodeWiki/Specs/FeatureSpecs/roadrescue-mechanic-portal-ui-screen-spec.md
- Operation: write
- Timestamp: 2026-01-07T12:43:29.492633+00:00
- Restored At: 2026-01-09T13:15:04.413208+00:00
- Task ID: cg9d1ee1eb
-->

# RoadRescue Mechanic Portal UI and Screen Specification

## Purpose and scope

This document specifies the intended user experience and screen-level requirements for the RoadRescue **Mechanic Portal** frontend. It is written to match the current codebase and its implemented screens, navigation, and behaviors.

The Mechanic Portal supports mechanics logging in, viewing unassigned requests, accepting work, managing assigned requests, updating request status, and managing a small profile (display name and service area).

## Information architecture and navigation model

The top navigation is rendered by `src/components/layout/Navbar.js` and is only populated when a `user` is present.

Authenticated mechanic navigation includes:
- “Dashboard” → `/dashboard`
- “My Assignments” → `/assignments`
- “Profile” → `/profile`
- Right-side status chip:
  - “Approved” if `user.approved` is true
  - “Pending approval” if `user.approved` is false
- “Log out” button, which calls `dataService.logout()` and navigates to `/login`.

Unauthenticated users only see the brand header; no navigation links are rendered.

## Visual style (theme)

Theme constants are defined in `src/theme.js` and align with the Ocean Professional palette:
- Primary: `#2563EB`
- Secondary/accent: `#F59E0B`
- Background: `#f9fafb`
- Surface: `#ffffff`
- Text: `#111827`
- Muted text: `#6B7280`
- Border: `#E5E7EB`
- Error: `#EF4444`

The UI is implemented using shared components (`Card`, `Input`, `Button`, `Table`) and common page layout classes (such as `.container`, `.hero`, `.h1`, `.lead`, `.grid2`).

## Core user workflows

## Authentication
Mechanics authenticate via the Mechanic Portal login screen. After successful login, the portal enforces that the logged-in user role is mechanic-oriented (`mechanic` or `approved_mechanic`).

## Work intake and assignment
Unassigned requests are listed on the Dashboard. A mechanic can accept a request, which moves it into “My Assignments” and enables status progression.

## Status progression
Mechanics update request status using a constrained set of statuses:
- Accepted
- En Route
- Working
- Completed

Status changes can optionally include a progress note, which is appended to request history.

## Screen specifications

### Screen: Login (Mechanic)

#### Route
`/login`

#### Primary user goal
Log in as a mechanic to manage requests.

#### UI structure
- Hero:
  - Title: “Mechanic Portal”
  - Lead: “Accept new requests and update statuses through completion.”
- Card:
  - Title: “Login”
  - Subtitle: “Use demo mechanic: mech@example.com / password123”
  - Form fields:
    - Email (required; defaults to `mech@example.com`)
    - Password (required; defaults to `password123`)
  - Primary action:
    - “Sign in” (disabled during submission; label changes to “Signing in...”)

#### Validation and constraints
- Email is required (non-empty after trim).
- Password must be at least 6 characters.
- After login, the user must have role `mechanic` or `approved_mechanic`. Otherwise the page displays: “This portal is for mechanics only.”

#### Post-conditions
On successful login, navigate to:
- `/dashboard`

### Screen: Dashboard (Available Requests)

#### Route
`/dashboard`

#### Primary user goal
See unassigned requests and accept work.

#### UI structure
- Hero:
  - Title: “Dashboard”
  - Lead: “Available requests awaiting a mechanic.”
- Approval banner:
  - When `user.approved` is false, show an info alert indicating pending admin approval.
- Card:
  - Title: “Available requests”
  - Subtitle: “Accept to move into My Assignments.”
  - Error region:
    - Displays errors from loading or accepting.
  - Table columns:
    - Request: short id link to `/requests/:id`
    - Created: locale date/time from `createdAt`
    - Vehicle: composed as “Make Model” with safe fallback to em dash
    - Status: rendered as a colored badge (Submitted, In Review, Accepted, En Route, Working, Completed)
    - Action: “Accept” button (shows “Accepting...” while busy)

#### Behaviors
- The screen loads via `dataService.listUnassignedRequests()`.
- Accepting a request calls `dataService.acceptRequest({ requestId, mechanic: user })` and reloads the list.

### Screen: My Assignments

#### Route
`/assignments`

#### Primary user goal
See work assigned to the mechanic and quickly jump into detail to update status.

#### UI structure
- Hero:
  - Title: “My assignments”
  - Lead: “Update status as you progress: Accepted → En Route → Working → Completed.”
- Card:
  - Title: “Assigned requests”
  - Subtitle: “Click into a request to update status.”
  - Error region for load failures
  - Table columns:
    - Request: short id link to `/requests/:id`
    - Vehicle: composed as “Make Model” with fallback
    - Status: badge styling for Accepted / En Route / Working / Completed
    - Customer: `userEmail`

#### Behaviors
- Loads assignments via `dataService.listMyAssignments(user.id)`.

### Screen: Request Detail (Mechanic)

#### Route
`/requests/:requestId`

#### Primary user goal
Review request information and update status with optional progress notes.

#### UI structure
- Hero:
  - Title: “Request <short id>”
  - Lead: “Current status: <status>”
- Two-column grid:
  - Card “Customer & contact”
    - Customer: `req.userEmail` (fallback em dash)
    - Contact: `req.contact.name` (fallback)
    - Phone: `req.contact.phone` (fallback)
    - Email: `req.contact.email` (fallback; may be absent)
  - Card “Vehicle”
    - Make: `req.vehicle.make` (fallback em dash)
    - Model: `req.vehicle.model` (fallback em dash)
    - Year: `req.vehicle.year` (fallback em dash)
    - Plate: `req.vehicle.plate` (fallback em dash)
- Card “Issue”
  - Issue description text
  - Progress note textarea (optional) with placeholder “e.g., On the way, ETA 15 minutes.”
  - Status update action buttons:
    - “Set: Accepted”
    - “Set: En Route”
    - “Set: Working”
    - “Set: Completed” (visually uses secondary button variant)
  - Navigation link: “← Back to assignments” → `/assignments`
  - History section (when `req.notes` exist):
    - Displays most recent first, showing timestamp, author, and text.

#### Behaviors and constraints
- The request is loaded via `dataService.getRequestById(requestId)`.
- When setting a status:
  - If the request is not assigned (`!req.assignedMechanicId`), the page attempts to accept it first via `dataService.acceptRequest`.
  - Then it calls `dataService.updateRequestStatus({ requestId, status, mechanic: user, noteText })`.
  - After updating, it reloads request data and clears the note field.
- If a load or update fails, an error card is displayed with a “← Back” link to `/dashboard`.

### Screen: Profile

#### Route
`/profile`

#### Primary user goal
Maintain basic mechanic identity information used for dispatch/assignment workflows.

#### UI structure
- Hero:
  - Title: “Profile”
  - Lead: “Keep your service area up to date for dispatching and assignments.”
- Card:
  - Title: “Mechanic details”
  - Subtitle: “Stored in Supabase profiles when configured; otherwise local demo storage.”
  - Form fields:
    - Display name (required)
    - Service area (optional; placeholder “e.g., Downtown, Northside”)
  - Feedback regions:
    - Success message: “Profile saved.”
    - Error banner for save failures
  - Primary action:
    - “Save” (disabled while busy; label changes to “Saving...”)

#### Validation rules
- Display name is required (non-empty after trim).

#### Post-conditions
- Profile changes are persisted via `dataService.updateProfile({ userId, profile })`.
- Parent state is updated through `onUserUpdated?.({ ...user, profile })`.

## Non-functional UI requirements

## Responsiveness
The `.grid2` layout on the Request Detail page should stack on mobile widths while preserving readable key/value formatting.

## Safety and access expectations
The portal explicitly rejects users that are not mechanics. It also presents a “Pending approval” state in the UI; whether approval restricts actions is presented as a policy constraint, and the UI already signals the state to the mechanic.
