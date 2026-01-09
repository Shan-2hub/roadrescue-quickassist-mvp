<!--
MongoDB Document Metadata:
- Original File Path: kavia-docs/CodeWiki/Specs/FeatureSpecs/roadrescue-user-website-ui-screen-spec.md
- Operation: write
- Timestamp: 2026-01-07T12:42:58.186386+00:00
- Restored At: 2026-01-09T13:15:04.413134+00:00
- Task ID: cg9d1ee1eb
-->

# RoadRescue User Website UI and Screen Specification

## Purpose and scope

This document specifies the intended user experience and screen-level requirements for the RoadRescue “QuickAssist” **User Website** frontend. The scope is limited to what is present in the current codebase, with screen behaviors and UI elements described to match the implementation.

The User Website provides an authenticated user flow to submit a vehicle breakdown request, view a list of requests, and view request details. It also provides an About page describing data/auth mode (Supabase vs local mock storage).

## Information architecture and navigation model

The top navigation is rendered by `src/components/layout/Navbar.js` and varies by authentication state.

When the user is authenticated, the primary navigation contains:
- “Submit Request” → `/submit`
- “My Requests” → `/requests`
- “About” → `/about`
- A right-side chip showing the user role label (User/Mechanic/Admin) and a “Log out” button.

When the user is not authenticated, the primary navigation contains:
- “Login” → `/login`
- “Register” → `/register`
- “About” → `/about`

Logging out uses `dataService.logout()` and redirects to `/login`.

## Visual style (theme)

The application theme constants are defined in `src/theme.js`.

The UI is designed as a clean, modern, business-grade interface with:
- Primary color: `#2563EB`
- Secondary / accent color: `#F59E0B`
- Error: `#EF4444`
- Background: `#f9fafb`
- Surface: `#ffffff`
- Text: `#111827`
- Muted text: `#6B7280`
- Border: `#E5E7EB`
- Subtle blue background utility: `rgba(37, 99, 235, 0.10)`
- Rounded corners: 10px–18px tokens
- Subtle shadows: small and medium tokens

The app uses shared UI components (`Card`, `Input`, `Button`, `Table`) and page-level layout helpers (such as `.container`, `.hero`, `.h1`, `.lead`) to present consistent spacing and hierarchy.

## Common layout patterns

Pages generally follow these patterns:

### Page container
Each screen wraps content in a `.container` which constrains width and provides consistent padding.

### Hero header
Most pages begin with a `.hero` section containing:
- A primary H1 title (`.h1`)
- A supporting line of explanatory text (`.lead`)

### Card sections
Primary content blocks are placed in a `Card` with an optional `title`, `subtitle`, and optional `actions` region for contextual links.

### Alerts and validation
Inline errors appear as an alert area (`.alert.alert-error`). Non-error informational messages may use `.alert` or `.alert.alert-info` depending on the page.

### Responsive grids
The UI uses `.grid2` for two-column layouts, which should collapse to one column on narrow screens.

## Screen specifications

### Screen: Login

#### Route
`/login`

#### Primary user goal
Authenticate to submit and track breakdown requests.

#### UI structure
- Hero:
  - Title: “Welcome back”
  - Lead: “Log in to submit and track your roadside assistance requests.”
- Card:
  - Title: “Login”
  - Subtitle: “Use the seeded demo account or your own.”
  - Action link: “Create account” → `/register`
  - Form fields:
    - Email (required)
    - Password (required, minimum length validation of 6 characters)
  - Inline links:
    - “About data & auth” → `/about`
  - Primary action:
    - “Sign in” (disabled during submission, label changes to “Signing in...”)

#### Validation rules
- Email is required (non-empty after trim).
- Password must be at least 6 characters.
- Authentication errors show as an error alert.

#### Post-conditions
On successful login, the app navigates to:
- `/submit`

### Screen: Register

#### Route
`/register`

#### Primary user goal
Create a new account for subsequent logins (Supabase if configured, otherwise local demo storage).

#### UI structure
- Hero:
  - Title: “Create your account”
  - Lead: “Start a new request in under a minute.”
- Card:
  - Title: “Register”
  - Subtitle: “Email/password auth (Supabase if configured; otherwise local demo).”
  - Action link: “Back to login” → `/login`
  - Form fields:
    - Email (required)
    - Password (required, minimum length validation of 6 characters, hint: “At least 6 characters.”)
    - Confirm password (required)
  - Primary action:
    - “Create account” (disabled during submission, label changes to “Creating...”)

#### Validation rules
- Email is required (non-empty after trim).
- Password must be at least 6 characters.
- Confirm password must match password.
- Registration errors show as an error alert.

#### Post-conditions
On successful registration, the app navigates to:
- `/submit`

### Screen: Submit Request

#### Route
`/submit`

#### Access control
This screen is intended for authenticated users. (In current routing, authenticated state is passed down and the page uses the current `user` in request creation.)

#### Primary user goal
Create a new breakdown request via a manual form (no maps/AI).

#### UI structure
- Hero:
  - Title: “Submit a breakdown request”
  - Lead: “Tell us what happened. A mechanic will review and accept it.”
- Card:
  - Title: “Request details”
  - Subtitle: “No maps/AI—manual details only.”
  - Form sections:
    1) Vehicle details (two-column grid):
       - Make (required)
       - Model (required)
       - Year (optional; placeholder “e.g., 2018”)
       - License plate (optional; placeholder “Optional”)
       The state includes year and plate, but other screens treat vehicle as `{ make, model }` for display.
    2) Issue description (textarea):
       - Required; placeholder “Describe the symptoms, warning lights, noises, etc.”
    3) Contact (two-column grid):
       - Contact name (required)
       - Contact phone (required)
  - Error banner region:
    - Shows the first validation or submission error encountered.
  - Actions row:
    - Primary: “Submit request” (disabled during submission; label changes to “Submitting...”)
    - Secondary: “View my requests” → navigates to `/requests`

#### Validation rules
- Vehicle make is required.
- Vehicle model is required.
- Issue description is required.
- Contact name is required.
- Contact phone is required.

#### Post-conditions
On successful creation, the app navigates to:
- `/requests/:id` for the created request.

### Screen: My Requests (Request List)

#### Route
`/requests`

#### Primary user goal
See all submitted requests and track their statuses.

#### UI structure
- Hero:
  - Title: “My requests”
  - Lead: “Track status updates as mechanics accept and work your case.”
- Card:
  - Title: “Requests”
  - Subtitle: “Newest first.”
  - Actions:
    - “+ New request” → `/submit`
  - Error region:
    - Shows load errors in an error alert.
  - Table columns:
    - Request: short id link to `/requests/:id`
    - Created: locale date/time from `createdAt`
    - Vehicle: displayed as `<make> <model>` with safe fallback if missing values
    - Status: rendered as a badge with color mapping

#### Status badges
Status strings map to colored badges (blue/amber/green). The UI currently recognizes multiple statuses including:
- Submitted
- In Review
- Assigned
- In Progress
- Completed
- Accepted
- En Route
- Working

### Screen: Request Detail (User)

#### Route
`/requests/:requestId`

#### Primary user goal
View the details and current status of a specific request.

#### Access control and security behavior
On load, the page fetches the request by id and enforces that `req.userId === user.id`. If the request is not found or access is denied, an error card is displayed.

#### UI structure
- Hero:
  - Title: “Request <short id>”
  - Lead: “Status: <status>”
- Content:
  - Two-column grid of cards:
    - Vehicle card:
      - Make: `req.vehicle.make` (fallback to empty string)
      - Model: `req.vehicle.model` (fallback to empty string)
      - Year: displayed as em dash (not read from the request here)
      - Plate: displayed as em dash (not read from the request here)
      This reflects the current canonical display expectation of vehicle being `{ make, model }` in most views.
    - Contact card:
      - Name: `req.contact.name`
      - Phone: `req.contact.phone`
      - Created: locale date/time from `createdAt`
  - Issue description card:
    - Full issue description text
    - Link: “← Back to My Requests” → `/requests`

#### Loading and error states
- While loading: a skeleton placeholder is shown.
- On error: a Card with an error alert and “Back to list” link.

### Screen: About

#### Route
`/about`

#### Primary user goal
Understand the MVP constraints and whether the app is running in Supabase mode or mock mode.

#### UI structure
- Hero:
  - Title: “About QuickAssist MVP”
  - Lead: “Manual dashboards and forms—no maps, AI, or external location APIs.”
- Card:
  - Title: “Auth & data mode”
  - Subtitle:
    - “Supabase mode detected.” when `isSupabaseConfigured()` is true
    - “Mock mode detected (localStorage).” otherwise
  - Content:
    - Bullet list describing:
      - Supabase configured behavior: uses `supabase.auth` and tables `profiles` and `requests`
      - Mock mode behavior: uses localStorage with seeded demo users/requests
      - Demo accounts and password
    - A note instructing that `REACT_APP_SUPABASE_URL` and `REACT_APP_SUPABASE_KEY` enable Supabase mode

## Non-functional UI requirements

## Responsiveness
The grid-based layouts must gracefully collapse for mobile widths. Controls should remain tappable and legible with adequate spacing.

## Accessibility
- Navigation is labeled with `aria-label="Primary navigation"`.
- Forms use explicit `label` elements and required indicators.

## Data-mode clarity
The About page is the authoritative UI location for clarifying mock vs Supabase mode. In mock mode, demo credentials are displayed for ease of testing.

## Known implementation constraints (as currently implemented)

The request submission form stores vehicle year and license plate, but the user request detail screen displays Year and Plate as placeholders. This is a current inconsistency in display behavior and should be considered when refining the product, but this document reflects the present implementation.
