# InMyControl — Project Plan

Cloud-first web app to help users focus on actions they can control while pursuing a motivating Big Goal.

## Table of contents

- [Project overview](#project-overview)
- [Scope and priorities](#scope-and-priorities)
- [High-level architecture](#high-level-architecture)
- [Data model (table schemas)](#data-model-table-schemas)
- [Feature set (detailed)](#feature-set-detailed)
- [API surface (overview)](#api-surface-overview)
- [Google Calendar flow](#google-calendar-flow)
- [UI / UX notes and onboarding](#ui--ux-notes-and-onboarding)
- [Security, privacy, and RLS](#security-privacy-and-rls)
- [Deployment, infra, and operational plan](#deployment-infra-and-operational-plan)
- [Roadmap and milestones](#roadmap-and-milestones)
- [Testing and QA](#testing-and-quality-assurance)
- [Acceptance criteria](#acceptance-criteria)
- [Next steps](#next-steps)

## Project overview

InMyControl is a cloud-first web application hosted on Vercel, using Supabase (Postgres/Auth/Storage) and GitHub for repository and CI. The product helps users pursue a motivating "Big Goal" by tracking actions they can control: recurring tasks (Daily, Weekly, Monthly) and one-time tasks. The app's internal calendar is the canonical timeline. Google Calendar integration is strictly one-way and user-prompted: when a user creates a one-time task they may choose to add it to Google Calendar. Templates are authored outside the app and imported as seed content.

## Scope and priorities

### Primary objectives

- Enable users to identify and consistently complete controllable actions aligned with their Big Goal.
- Provide clear daily/weekly/monthly progress tracking and an internal canonical calendar.
- Offer a simple, privacy-respecting one-way flow to add new one-time tasks to Google Calendar.

### MVP priorities

- Authentication and profile
- Big Goal onboarding
- Create/edit/delete recurring tasks and one-time tasks
- Task logging and internal calendar view
- One-time → Google Calendar add prompt (user-driven)
- Templates import (seed-only)
- Quotes popup and basic settings
- Export/import user data

### Non-goals for MVP

- Local-first / full offline mode
- Two-way Google Calendar sync
- Background calendar polling
- Template-driven custom runtime logic

## High-level architecture

- **Frontend**: React + TypeScript SPA (hosted on Vercel). Components for Dashboard, Daily/Weekly/Monthly pages, internal Calendar, Task creation/edit modals, Settings.
- **Backend**: Serverless endpoints on Vercel for privileged operations (optional server-side Google Calendar creation). Supabase client used directly from frontend for core CRUD and auth.
- **Database**: Supabase Postgres (managed). Use UUID primary keys, JSONB for recurrence rules and settings.
- **Auth**: Supabase Auth; Google OAuth available for optional calendar creation. Consider email/password fallback.
- **Storage**: Supabase Storage buckets for profile images and attachments.
- **Logging/Monitoring**: Sentry (opt-in) or alternative; serverless logs in Vercel.
- **CI/CD**: GitHub + Vercel integration; PR checks for lint/tests.

## Data model (table schemas)

**Notes:**
- Use UUID for primary keys and timestamptz for timestamps.
- JSONB fields are used for flexible recurrence rules and settings.
- The following schema list is descriptive; literal SQL statements are intentionally omitted.

### users
- `id` (UUID) — primary key (Supabase Auth user id)
- `email` (text)
- `display_name` (text)
- `profile_image_url` (text) — Supabase Storage path or signed URL
- `created_at` (timestamptz)
- `settings` (jsonb) — UI preferences, calendar prompt policy, quote preferences

### big_goals
- `id` (UUID)
- `owner_id` (UUID) — FK users.id
- `title` (text)
- `description` (text)
- `image_url` (text)
- `is_active` (boolean)
- `created_at` (timestamptz)
- `last_modified` (timestamptz)

### task_types
- `id` (text) — e.g., D, W, M, O (One-time)
- `description` (text)
- `icon` (text) — icon identifier or storage path

### categories
- `id` (text) — e.g., PH, MH, SH, RH or user-defined slug
- `description` (text)
- `owner_id` (UUID NULL) — NULL = global
- `color` (text) — optional hex string for UI

### recurring_tasks
- `id` (UUID)
- `owner_id` (UUID) — FK users.id
- `type_id` (text) — FK task_types.id (D, W, M)
- `category_id` (text) — FK categories.id
- `title` (text)
- `description` (text)
- `repeat_rule` (jsonb) — examples: `{"days":["Mon","Wed","Fri"]}` or `{"weekly_target":3}`
- `target_count` (integer NULL) — for weekly/monthly tallies
- `active` (boolean)
- `created_at` (timestamptz)
- `last_modified` (timestamptz)

### one_time_tasks
- `id` (UUID)
- `owner_id` (UUID)
- `category_id` (text)
- `title` (text)
- `description` (text)
- `due_date` (date)
- `calendar_added` (boolean DEFAULT false)
- `calendar_event_id` (text NULL) — saved only when calendar_added and creation succeeded
- `completed` (boolean DEFAULT false)
- `created_at` (timestamptz)
- `last_modified` (timestamptz)

### task_logs
- `id` (UUID)
- `task_id` (UUID) — FK to recurring_tasks.id
- `owner_id` (UUID)
- `date` (date)
- `completed` (boolean)
- `count` (integer DEFAULT 1) — useful for weekly tally increments
- `note` (text NULL)
- `created_at` (timestamptz)

### quotes
- `id` (UUID)
- `owner_id` (UUID NULL) — NULL for global quotes
- `quote` (text)
- `reference` (text NULL)
- `created_at` (timestamptz)

### attachments
- `id` (UUID)
- `owner_id` (UUID)
- `parent_type` (text) — e.g., user, task, one_time_task
- `parent_id` (UUID) — FK to related record
- `storage_path` (text) — Supabase Storage path
- `public` (boolean)
- `created_at` (timestamptz)

### templates
- `id` (UUID)
- `owner_id` (UUID) — typically your admin account
- `name` (text)
- `description` (text)
- `seed_payload` (jsonb) — categories, recurring_tasks, one_time_tasks skeletons
- `created_at` (timestamptz)

### audit_logs (optional but recommended)
- `id` (UUID)
- `owner_id` (UUID)
- `event_type` (text) — e.g., task.create, calendar.add
- `payload` (jsonb)
- `created_at` (timestamptz)

## Feature set (detailed)

### Authentication & profile
- Supabase Auth for signup/signin.
- Google OAuth optional for calendar creation.
- Profile page with display name, profile image upload (Supabase Storage).
- Settings store: calendar prompt preference, quote popup toggle, data export.

### Big Goal
- Onboarding asks user for a single motivating Big Goal (text + optional image).
- Big Goal shown on Dashboard and summary cards; editable in profile or settings.

### Recurring tasks
- Create/edit/delete recurring tasks for Daily, Weekly, Monthly use-cases.
- Repeat rule UI:
  - Daily: choose weekdays
  - Weekly: either checklist (complete once per week) or weekly-target tally (e.g., target_count = 3)
  - Monthly: checklist or monthly-target tally
- Task metadata: title, description, category, active flag, target_count.
- Logs for each completion event; repeat_rule stored as JSONB.

### One-time tasks
- Create/edit/delete one-time tasks with optional due_date.
- After creation prompt to add to Google Calendar (user decides).
- Track calendar_added and calendar_event_id when created.
- One-time tasks shown on internal calendar on their due_date.

### Internal calendar
- Derived from app data (recurring task logs and one-time tasks).
- Day cells display completed/total counts for recurring tasks and list one-time tasks.
- Week/month banners show aggregated progress counts.

### Dashboard
- Default landing: top Daily (today's recurring tasks + today's one-time tasks), then Weekly section (upcoming week non-today one-time tasks + recurring weekly items), then Monthly and Big Goal snapshot.
- Quick add (+) to create a one-time task.

### Google Calendar integration (one-way, user-prompted)
- Prompt on creating a one-time task: Add / Don't add; settings control prompt frequency.
- If user adds, create event in Google Calendar via OAuth (client-side or serverless).
- Persist calendar_event_id to allow future unlinking.
- Event title format: `IMC | One-time | [Category] | [Task Title]`

### Templates
- Import JSON template files authored by you.
- Applying a template seeds categories, recurring tasks, and one-time placeholders.
- Templates editable post-import; no special runtime logic in v1.

### Quotes popup
- Optional daily popup showing a random quote from global list, user list, or external API.
- User-configurable behavior in Settings.

### Attachments
- Support images or attachments for profiles and tasks.
- Store files in Supabase Storage; store path references in attachments table.

### Export / Import
- Users can export full data as JSON and import to another account.

## API surface (overview)

### Auth
- Handled by Supabase client.

### Core endpoints
Suggested serverless endpoints on Vercel in addition to direct Supabase usage:

- `GET /api/dashboard` — aggregated data for dashboard
- `GET /api/recurring_tasks` — list; `POST /api/recurring_tasks` — create; `PUT /api/recurring_tasks/:id` — update; `DELETE /api/recurring_tasks/:id` — delete
- `GET /api/one_time_tasks` — list; `POST /api/one_time_tasks` — create; `PUT /api/one_time_tasks/:id` — update; `DELETE /api/one_time_tasks/:id` — delete
- `POST /api/one_time_tasks/:id/add_to_google` — create Google Calendar event (serverless or client-side)
- `GET /api/calendar/internal?start=YYYY-MM-DD&end=YYYY-MM-DD` — derived internal calendar events
- `POST /api/task_logs` — create log; `GET /api/task_logs?date=YYYY-MM-DD` — fetch logs
- `POST /api/templates/:id/apply` — seed tasks & categories from template
- `GET /api/quotes/random` — random quote
- `GET /api/export` — user export JSON
- `POST /api/import` — import user export JSON

### Authorization

- Every endpoint must validate Supabase JWT; serverless endpoints should verify auth as well.
- Ensure serverless endpoints protect any stored refresh tokens if used.

### Implementation considerations

- Core CRUD can be performed directly against Supabase from the frontend using the Supabase client, with serverless functions reserved for Google Calendar creation (if server-side refresh token storage is chosen) and any other privileged actions.

## Google Calendar flow (one-way, user-prompted)

### Prompt behavior

- After saving a one-time task, show a non-blocking prompt: "Add this to your Google Calendar?" with actions: Add / Don't add.
- Settings allow user to toggle prompt frequency: Always ask / Ask once per session / Never.

### Add action

If the user chooses Add:

- Use client-side Google API OAuth flow to create the event (recommended for simplicity and to avoid persisting refresh tokens), or
- Use serverless function to create event if you prefer server-side creation and will store refresh tokens (must be encrypted).

Event metadata:

- Title: `IMC | One-time | [Category] | [Task Title]`
- Date/time: derived from one-time task (due_date) — optionally allow all-day vs. time selection
- Description: include link back to the app task or summary

On success:

- Set `one_time_tasks.calendar_added = true`
- Store `calendar_event_id` for potential unlinking or open-in-calendar actions

On failure:

- Surface a clear error and allow retry

### Implementation note

For MVP avoid background sync and imports from Google. Keep the internal calendar the canonical timeline.

## UI / UX notes and onboarding

### Onboarding

- Short flow to set the Big Goal and whether user wants Google Calendar integration.
- Explain cloud-first model, optional Google Calendar adds, and privacy controls.

### Dashboard layout

- Single-column: Daily (today's recurring + today's one-time), Weekly (rest of week), Monthly summary, Big Goal snapshot.
- Quick add button to create one-time tasks.

### Task creation modal

- Fields: Type (One-time or Recurring type selected by route), Title, Category, Description, Due Date (for one-time), Repeat Rule (for recurring), Target Count (if applicable), Calendar add toggle (or prompt after save).

### Internal calendar

- Month view with day cells showing completed/total for recurring tasks.
- Clicking a day opens a side panel with details (task list, logs, one-time tasks).
- Week banner and month header summarize progress.

### Visual affordances

- Badge on one-time task cards when calendar_added is true (with "Open in Google Calendar" link or "Unlink" action).
- Progress bars for weekly/monthly tally targets.
- Clear use of category colors/icons.

### Settings

- Calendar prompt policy
- Quotes popup toggle and preference
- Export / Delete account

## Security, privacy, and Row-Level Security (RLS)

### Supabase RLS policies

Enforce owner-based access:

- `recurring_tasks`, `one_time_tasks`, `task_logs`, `attachments`, `quotes`: allow access only when `owner_id = auth.uid()`
- `users`: allow select/update only for current user; admin-only for cross-user ops
- Templates and global categories are readable by all users; template application seeds owned rows.

### Tokens and secrets

- If storing Google refresh tokens for server-side flows, encrypt them at application layer and store encryption keys in Vercel environment variables (rotate regularly).
- Prefer client-only OAuth for calendar creation in MVP to avoid storing refresh tokens.

### Transport & storage

- TLS for all traffic.
- Use Supabase-managed encryption-at-rest; consider application-layer encryption for highly sensitive fields if required.

### Privacy features

- Clear privacy notice and setting to disable profile image storage (or store as private).
- Export and delete endpoints to satisfy user data portability and deletion requests.

## Deployment, infra, and operational plan

### Vercel

- Host SPA and serverless endpoints
- Use Vercel environment variables for Supabase keys and cryptographic keys

### Supabase

- Managed Postgres, Auth, Storage
- Configure buckets for attachments and profile images
- Create migrations and seed initial task_types and global categories

### GitHub + CI

- Branch protection and PR checks
- Run linting, tests, builds on PRs before merge

### Monitoring

- Error tracking (Sentry) — opt-in
- Basic serverless logs in Vercel
- Supabase dashboard for DB usage; create alerts for storage and row growth

### Backups & cost control

- Rely on Supabase backups for DB; consider exporting JSON user backups for redundancy
- Monitor free-tier limits and set alerts for usage spikes; add rate limits or quotas if needed

## Roadmap and milestones

### Sprint 0 — Preparation
- Create Supabase project and configure Auth/Storage
- Set up Vercel + GitHub repo + CI
- Define RLS policies and seed data for task_types and global categories

### Sprint 1 — Core auth & profile
- Implement Supabase Auth in frontend
- Profile page, settings persistence, image upload to Supabase Storage

### Sprint 2 — Recurring tasks & logging
- UI for Daily/Weekly/Monthly recurring tasks
- Repeat rule editor and JSONB mapping
- Task logging endpoints and internal calendar derivation

### Sprint 3 — One-time tasks & internal calendar
- One-time create/edit/delete
- Internal calendar rendering and dashboard integration
- Quick add flow

### Sprint 4 — Google Calendar add flow & attachments
- Implement OAuth decision (client-side recommended)
- Implement Add to Google Calendar flow and persist calendar metadata
- Attachments upload and attachments table

### Sprint 5 — Templates, quotes, export/import
- Template import UI (from your authored JSON)
- Quotes popup and settings
- Export/import endpoints and UI

### Sprint 6 — Polish, testing, release
- Accessibility, responsive layouts, unit/integration tests
- E2E tests for critical flows
- Monitoring and error tracking
- Alpha release and feedback loop

### Post-MVP
- Optional server-side background jobs (if storing refresh tokens)
- Local caching for temporary offline use
- Template-driven custom logic and richer template features
- Advanced analytics, habit streaks, community templates

## Testing and Quality Assurance

### Unit tests
- Recurrence parsing and repeat_rule handling
- Tally/target logic for weekly/monthly

### Integration tests
- Supabase endpoints and RLS behaviors
- Template import and seed application

### End-to-end tests
- Onboarding → create recurring task → log → calendar view
- Create one-time task → Add to Google Calendar flow
- Export / Import workflows

### Security testing
- Verify RLS prevents cross-user access
- Test attempts to access or modify other users' data directly via API

## Acceptance criteria (MVP)

- Users can sign up/sign in and set a Big Goal
- Users can create/edit/delete recurring tasks (Daily/Weekly/Monthly) and log completions
- Users can create/edit/delete one-time tasks and optionally add them to Google Calendar via a prompt
- Internal calendar shows per-day completed/total counts and one-time tasks on due dates
- Templates can be imported by the user to seed content
- Export endpoint provides full user JSON; account deletion supported
- RLS and auth enforced; attachments stored in Supabase Storage

## Next steps

1. Choose Google OAuth approach for calendar creation:
   - Client-side (recommended for MVP) — simpler, no long-term token storage
   - Server-side (if you want future background capabilities) — requires encrypted refresh token storage

2. Create Supabase project and seed initial data (task_types, global categories)

3. Scaffold frontend (React + TypeScript) and integrate Supabase client + Auth

4. Implement DB schemas and RLS policies in Supabase

5. Build Sprint 1 features and iterate on UX

#   i n M y C o n t r o l  
 