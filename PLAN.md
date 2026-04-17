# Implementation Plan: PCWD Client Intake Portal

## Overview
A Next.js 16 client intake and project-status portal for Portage Creek Web Designs.
Linked from the "Sign In" button on https://www.portagecreekwebdesigns.com/.

---

## Phase 1 — Project Scaffolding

### Step 1.1 — Initialize Next.js App
```bash
npx create-next-app@latest . \
  --typescript \
  --tailwind \
  --app \
  --src-dir \
  --import-alias "@/*"
```
- Use the `src/` directory convention.
- App Router (Next.js 15/16 default).

### Step 1.2 — Install Core Dependencies
```bash
# Database
npm install drizzle-orm @neondatabase/serverless
npm install -D drizzle-kit

# Auth
npm install @clerk/nextjs

# UI / Design System
npm install @fontsource/epilogue @fontsource/manrope
npm install clsx tailwind-merge

# Forms & Validation
npm install react-hook-form zod @hookform/resolvers

# Notifications (see Phase 6)
npm install resend          # transactional email
npm install @novu/node      # optional: multi-channel notification layer
```

### Step 1.3 — Environment Variables
Create `.env.local` with the following keys (values provided by user):
```
# Neon / Drizzle
DATABASE_URL=

# Clerk
NEXT_PUBLIC_CLERK_PUBLISHABLE_KEY=
CLERK_SECRET_KEY=
NEXT_PUBLIC_CLERK_SIGN_IN_URL=/sign-in
NEXT_PUBLIC_CLERK_SIGN_UP_URL=/sign-in
NEXT_PUBLIC_CLERK_AFTER_SIGN_IN_URL=/onboarding
NEXT_PUBLIC_CLERK_AFTER_SIGN_UP_URL=/onboarding

# Admin
ADMIN_EMAIL=portagecreekwebdesigns@gmail.com

# Notifications
RESEND_API_KEY=
```

### Step 1.4 — Tailwind & Design System Configuration
Extend `tailwind.config.ts`:
- `fontFamily`: `epilogue` → Epilogue, `manrope` → Manrope.
- `colors`: `primary: #00B4D8`, surface/neutral tokens.
- `borderRadius`: set `DEFAULT` and all keys to `9999px` (max roundedness = pill shapes).
- `spacing`: keep Tailwind's default scale-2 base.

Add Google Fonts or `@fontsource` imports in `src/app/layout.tsx`.

---

## Phase 2 — Database Schema (Neon + Drizzle ORM)

### Step 2.1 — Drizzle Config
Create `drizzle.config.ts` at project root pointing to `src/db/schema.ts` and `DATABASE_URL`.

### Step 2.2 — Schema (`src/db/schema.ts`)

#### `clients` table
| Column | Type | Notes |
|---|---|---|
| id | uuid PK | default `gen_random_uuid()` |
| clerk_user_id | text unique | synced from Clerk webhook |
| email | text unique | |
| full_name | text | |
| phone | text | format `(XXX)-XXX-XXXX` |
| business_name | text | |
| created_at | timestamp | |
| updated_at | timestamp | |

#### `intake_responses` table
| Column | Type | Notes |
|---|---|---|
| id | uuid PK | |
| client_id | uuid FK → clients.id | |
| products_services | text | Step 2 Q1 |
| ideal_customer | text | Step 2 Q2 |
| problems_solved | text | Step 2 Q3 |
| differentiators | text | Step 2 Q4 |
| brand_preferences | text | Step 3 Q1 |
| reference_sites | text | Step 3 Q2 |
| budget | text | Step 3 Q3 |
| created_at | timestamp | |

#### `project_statuses` table
| Column | Type | Notes |
|---|---|---|
| id | uuid PK | |
| client_id | uuid FK → clients.id | unique (one status record per client) |
| current_step | integer | 1–5 (see status stages) |
| step1_note | text nullable | Under Review note |
| step2_note | text nullable | Design & Scaffolding note |
| step3_note | text nullable | Website is Live note |
| step4_note | text nullable | Payment Status note |
| step5_note | text nullable | Launched note |
| updated_at | timestamp | |

**Status stages enum:**
1. Under Review
2. Design & Scaffolding
3. Website is Live
4. Payment Status
5. Launched

### Step 2.3 — DB Client (`src/db/index.ts`)
Initialize `neon()` + `drizzle()` singleton for use in Server Actions and API routes.

### Step 2.4 — Migrations
```bash
npx drizzle-kit generate
npx drizzle-kit migrate
```

---

## Phase 3 — Authentication (Clerk)

### Step 3.1 — Clerk Provider
Wrap `src/app/layout.tsx` in `<ClerkProvider>`.

### Step 3.2 — Sign-In Page (`src/app/sign-in/[[...sign-in]]/page.tsx`)
Render Clerk's `<SignIn>` component styled to match the PCWD design system (teal accent, pill shapes).

### Step 3.3 — Clerk Webhook (`src/app/api/webhooks/clerk/route.ts`)
- Listen for `user.created` event.
- Insert a new row into `clients` (clerk_user_id, email).
- Create a default `project_statuses` row (current_step = 1).
- Verify webhook signature with `svix`.

### Step 3.4 — Route Guard Middleware (`src/middleware.ts`)
Use `clerkMiddleware()` with `createRouteMatcher`:
- Public: `/`, `/sign-in(.*)`, `/api/webhooks/(.*)`.
- Protected: everything else.
- Admin-only: `/admin/(.*)` — check email against `ADMIN_EMAIL` env var inside the route handler.

### Step 3.5 — Post-Login Redirect Logic (`src/app/api/auth/redirect/route.ts` or layout)
After sign-in, server-side check:
- If `email === ADMIN_EMAIL` → redirect `/admin`.
- Else if `clients.full_name` is null (intake not completed) → redirect `/onboarding`.
- Else → redirect `/status`.

---

## Phase 4 — User Flows

### Step 4.1 — Onboarding / Intake (`src/app/onboarding/page.tsx`)

Three-step wizard using `react-hook-form` + `zod`:

**Step 1 — Business Contact**
- Business Name (required)
- Full Name (required)
- Phone Number (required, regex `^\(\d{3}\)-\d{3}-\d{4}$`)

**Step 2 — Tell Me More About Your Business**
- Products/Services you're promoting
- Who is your ideal customer?
- What problems are you solving?
- What makes you different from competitors?

**Step 3 — Design, Build & Launch Context**
- Do you have brand designs and preferences?
- Any reference sites you like or dislike?
- What's your budget?

**Wizard behavior:**
- Progress indicator (step 1/3, 2/3, 3/3) with pill-styled step dots.
- "Next" / "Back" navigation; final step has "Submit".
- On submit: Server Action saves to `clients` (name, phone, business_name) and `intake_responses`; redirects to `/status`.

### Step 4.2 — Status Page (`src/app/status/page.tsx`)
- Fetch `clients` row and `project_statuses` row for the logged-in user.
- Display: "Project Status for {Full Name}" / "Track the progress of {Business Name}".
- Vertical stepper showing all 5 stages; current stage highlighted in teal.
- Each stage card shows the admin's note (if set) below the stage description.
- Auto-refresh or optimistic polling (every 60 s) via `router.refresh()` or SWR.

---

## Phase 5 — Admin Dashboard

### Step 5.1 — Admin Layout (`src/app/admin/layout.tsx`)
- Server-side email guard: redirect non-admins to `/status`.
- Sidebar nav: Dashboard, Clients.

### Step 5.2 — Dashboard Page (`src/app/admin/page.tsx`)

**Section A — Stats Cards (4 cards)**
| Stat | Query |
|---|---|
| Total Clients | `COUNT(clients)` |
| Pending Sign Up | clients with no `full_name` (intake not completed) |
| In Progress | `current_step` 1–4 |
| Launched | `current_step = 5` |

**Section B — Client Records Table**
Columns: Full Name, Business Name, Email, Phone, Current Status, Actions.
Actions: View Details, Update Status, Delete.

**Section C — Pending Sign-Up Clients Table**
Clients who signed up via Clerk but have not completed intake.
Columns: Email, Signed Up At.

### Step 5.3 — Update Status Modal / Drawer
- Triggered by "Update Status" action.
- Select new step (1–5) via pill-button group.
- Textarea for per-step note.
- On save: Server Action updates `project_statuses`; triggers notification (Phase 6).

### Step 5.4 — Delete Client Action
- Confirmation dialog ("This will permanently delete all records for {name}.").
- Server Action: delete `project_statuses`, `intake_responses`, `clients` rows; also delete Clerk user via Clerk backend SDK.

---

## Phase 6 — Notifications

### Recommended Stack
- **Resend** (email, simple REST API): send transactional emails to the client and admin.
- No SMS or push needed at MVP; Resend covers both parties cleanly.

### Step 6.1 — Notification Helper (`src/lib/notifications.ts`)
Two functions:
- `notifyClientStatusUpdate(client, newStep, note)` — email to client.
- `notifyAdminNewIntake(client)` — email to admin when intake is completed.

### Step 6.2 — Email Templates (`src/emails/`)
Use React Email (optional) or plain HTML strings:
- `StatusUpdateEmail` — subject "Your project status has been updated", body shows new stage name + note.
- `NewIntakeEmail` — subject "New client intake received", body shows client name, business, contact.

### Step 6.3 — Trigger Points
| Event | Notification |
|---|---|
| Intake submitted | `notifyAdminNewIntake` |
| Admin updates status | `notifyClientStatusUpdate` |

---

## Phase 7 — UI Component Library (`src/components/`)

Build reusable components aligned with DESIGN.md:

| Component | Description |
|---|---|
| `Button` | pill-shaped, variants: primary (teal fill), outline, ghost |
| `Input` | pill-shaped text input with label + error state |
| `Textarea` | rounded (large radius), label + error state |
| `Card` | soft-corner container, neutral background |
| `StepIndicator` | numbered circles with connecting line; active = teal fill |
| `StatusStepper` | vertical stepper for status page |
| `StatCard` | stat number + label card for admin dashboard |
| `Modal` | centered overlay with pill-shaped action buttons |
| `Badge` | pill chip for status labels |

All components use Manrope for body text, Epilogue for headings, teal `#00B4D8` as accent.

---

## Phase 8 — File / Folder Structure

```
src/
├── app/
│   ├── layout.tsx                  # ClerkProvider, fonts, global styles
│   ├── page.tsx                    # Root → redirect logic
│   ├── sign-in/[[...sign-in]]/
│   │   └── page.tsx
│   ├── onboarding/
│   │   └── page.tsx                # 3-step intake wizard
│   ├── status/
│   │   └── page.tsx                # Client status page
│   ├── admin/
│   │   ├── layout.tsx              # Admin guard
│   │   └── page.tsx                # Admin dashboard
│   └── api/
│       ├── webhooks/clerk/
│       │   └── route.ts
│       └── auth/redirect/
│           └── route.ts
├── components/
│   ├── ui/                         # Primitive components (Button, Input…)
│   └── features/                   # Composed feature components
├── db/
│   ├── index.ts                    # Drizzle client
│   └── schema.ts
├── lib/
│   ├── notifications.ts
│   └── utils.ts                    # cn() helper
├── actions/                        # Next.js Server Actions
│   ├── intake.ts
│   ├── status.ts
│   └── admin.ts
└── emails/                         # Email templates
    ├── StatusUpdateEmail.tsx
    └── NewIntakeEmail.tsx
```

---

## Phase 9 — Testing & QA

- **Manual flows**: new user sign-in → intake → status page; admin login → dashboard → update status → client receives email; admin delete client.
- **Validation**: phone regex, required fields, step gating in wizard.
- **Auth guards**: direct URL access to `/admin`, `/status`, `/onboarding` by unauthenticated users → redirect to `/sign-in`.
- **Responsiveness**: mobile-first layout for client-facing pages; admin dashboard responsive table.

---

## Phase 10 — Deployment

- Deploy to **Vercel** (native Next.js support).
- Set all `.env.local` keys as Vercel environment variables.
- Configure Clerk webhook endpoint: `https://<domain>/api/webhooks/clerk`.
- Add Vercel domain to Clerk's allowed origins.
- Run `npx drizzle-kit migrate` against the production Neon database on first deploy.

---

## Implementation Order (Recommended Sequence)

1. Phase 1 — Scaffold & config
2. Phase 2 — DB schema + migrations
3. Phase 3 — Clerk auth + webhook + middleware
4. Phase 7 — UI component library
5. Phase 4.1 — Intake wizard
6. Phase 4.2 — Status page
7. Phase 5 — Admin dashboard
8. Phase 6 — Notifications
9. Phase 9 — QA
10. Phase 10 — Deploy
