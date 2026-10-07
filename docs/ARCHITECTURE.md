# Gurudev Ashram Appointment, Seva & Access Management System

## Architecture, Database Schema, Screens & Implementation Plan

Status: **Proposal (pre-Phase 1)**. Review this document before any application code is written.

Core idea:

```
ONE PERSON → ONE PROFILE → QR CREDENTIAL(S) → APPOINTMENTS / ZONE PERMISSIONS → GATES → COMPLETE SCAN HISTORY
```

---

## 1. Architecture

### 1.1 Stack

| Layer | Choice | Why |
|---|---|---|
| Frontend + server | **Next.js 15 (App Router) + TypeScript** | One codebase for admin UI, public booking form, scanner PWA and server API routes / Server Actions |
| UI | **Tailwind CSS + shadcn/ui** (Radix) | Accessible, simple components; easy calm theme |
| Database | **Supabase Postgres** | Relational, FKs, transactions, row-level security (RLS), SQL functions |
| Auth | **Supabase Auth** (email + password, optional magic link / phone OTP) | Hashing, sessions, password reset handled by the provider |
| Realtime | **Supabase Realtime** (Postgres changes) | Live dashboard & live security feed without refresh |
| Storage | **Supabase Storage** (private bucket `person-photos`) | Profile photos served via short-lived signed URLs |
| Hosting | **Vercel** (Edge/Node functions + Vercel Cron) | HTTPS by default, cron for reminders and notification outbox |
| QR generation | `qrcode` (server & client) | Mature, renders SVG/PNG |
| QR scanning | Native `BarcodeDetector` where available, fallback `@zxing/browser` | Works on Android Chrome and iOS Safari |
| PWA / offline | **Serwist** (Workbox for Next.js) + IndexedDB (`idb`) | Cached scanner shell, offline scan queue |
| Validation | **Zod** (shared client/server schemas) | One source of truth for input rules |
| Exports | CSV (native), Excel (`exceljs`), PDF (`@react-pdf/renderer`) | All three formats requested |
| Rate limiting | Postgres-backed limiter (default) or Upstash Redis (optional) | No extra vendor required for V1 |
| i18n (ready, not built) | `next-intl` with `en` only | Gujarati / Hindi can be added by dropping in message files |
| Testing | Vitest (unit), pgTAP or SQL tests via Supabase CLI (DB functions), Playwright (E2E acceptance tests) | Acceptance tests in §47 of the brief become automated E2E tests |

### 1.2 High-level diagram

```
                 ┌──────────────────────────── Vercel ─────────────────────────────┐
 Visitor ──────▶ │  /book (public form)      /pass/[token] (public pass page)      │
                 │                                                                 │
 Admin/Coord ──▶ │  /admin/*  (dashboard, people, appointments, slots, zones,      │
 View-only       │            permissions, reports, audit, settings, users)        │
                 │                                                                 │
 Security ─────▶ │  /scan     (mobile PWA, camera, offline queue)                  │
                 │                                                                 │
                 │  Server Actions / Route Handlers  ──┐   Vercel Cron:            │
                 │   - zod validation                  │    - notification outbox  │
                 │   - role check                      │    - reminders            │
                 │   - rate limiting                   │    - no-show marking      │
                 └─────────────────────────────────────┼──────────────────────────┘
                                                       ▼
                 ┌──────────────────────── Supabase ───────────────────────────────┐
                 │ Auth │ Postgres (tables + RLS + SECURITY DEFINER functions)      │
                 │      │   scan_credential(), book_appointment(), ...             │
                 │      │ Realtime (access_logs, appointments, zone_presence)      │
                 │      │ Storage (person-photos, private)                         │
                 └─────────────────────────────────────────────────────────────────┘
                                                       │
                                     Messaging provider adapters (pluggable)
                                     WhatsApp BSP · Email (Resend/SMTP) · SMS later
```

### 1.3 Key architectural decisions

1. **All access decisions happen in one Postgres function** (`scan_credential`). It reads the credential, person, permissions/appointment and presence, decides, writes the `access_logs` row and updates presence **in a single transaction**. The browser never decides; the API route only authenticates the scanner user and forwards. This makes Rule 6 ("every scan is logged") impossible to bypass and avoids race conditions when two gates scan the same card.
2. **Booking goes through `book_appointment`**, which locks the slot row (`SELECT … FOR UPDATE`) before counting, so two coordinators cannot both take the last seat. Override requires an authorized role + reason and is audited.
3. **QR credentials are a separate table**, not a column on `people`. A person can have a long-lived ID-card credential and one pass per appointment; each can be revoked or rotated independently (a lost card is re-issued without touching history).
4. **Presence is materialized** in `zone_presence` (updated by the scan function) instead of re-scanning the whole log for every dashboard tick. It can always be rebuilt from `access_logs`.
5. **Every table carries `event_id`** (directly or via parent) and events belong to a `site` (Ashram). Multiple Ashrams / events / future modules need no restructuring.
6. **Soft delete everywhere important** (`archived_at`), and logs are append-only (no UPDATE/DELETE grants).
7. **Timezone:** all timestamps are `timestamptz`; each event has a `timezone` (default `Asia/Kolkata`) used for slot times, windows and "today".
8. **Providers behind interfaces** (`MessagingProvider`, `EmailProvider`) with a DB outbox, so a WhatsApp Business provider can be connected later without code changes elsewhere.

---

## 2. Roles & authorization

| Capability | Super Admin | Admin / Coordinator | Security | View-only |
|---|:-:|:-:|:-:|:-:|
| Dashboard / reports | ✅ | ✅ | — (own recent scans only) | ✅ |
| People: create / edit | ✅ | ✅ | — | — |
| Block / unblock person | ✅ | ✅ (configurable) | — | — |
| Appointments: create / approve / reschedule / cancel | ✅ | ✅ | — | read |
| Slot capacity override | ✅ | only if `can_override_capacity` | — | — |
| Slots / zones / gates config | ✅ | read | — | read |
| Permissions: grant / revoke | ✅ | ✅ | — | read |
| Issue / revoke / rotate QR | ✅ | ✅ | — | — |
| Scan QR, record entry/exit | ✅ | ✅ | ✅ | — |
| Override scan decision | ✅ | ✅ | only if `can_override_scan` | — |
| Users & roles, settings, event setup | ✅ | — | — | — |
| Audit log | ✅ | read (configurable) | — | — |

Enforcement is layered:

1. **Postgres RLS** on every table, using `auth.uid()` → `user_profiles.role` (and per-event membership). This is the real boundary.
2. **Server Actions / route handlers** re-check role before any mutation and validate input with Zod.
3. **UI** hides what the role can't do (convenience only; never relied on).

Fine-grained flags (`can_override_capacity`, `can_override_scan`, `can_block_people`) live on the user profile so the Super Admin can grant them individually without inventing new roles.

---

## 3. Database schema

Conventions: `uuid` primary keys (`gen_random_uuid()`), `created_at` / `updated_at` / `created_by` on mutable tables, `archived_at` for soft delete, human-readable codes (`SEVA-1042`, `APT-26-0042`) generated by per-prefix sequences and **never** used inside QR codes.

### 3.1 Enums

```sql
create type user_role         as enum ('super_admin','admin','security','viewer');
create type person_type       as enum ('appointment_visitor','seva_volunteer','staff','coordinator','security','special_guest','other');
create type person_status     as enum ('active','blocked');
create type event_status      as enum ('draft','active','completed','archived');
create type appointment_status as enum ('pending','confirmed','rescheduled','cancelled','rejected',
                                        'arrived','entered','in_progress','completed','no_show');
create type appointment_source as enum ('public_form','admin');
create type credential_kind   as enum ('person_card','appointment_pass');
create type credential_status as enum ('active','revoked');
create type scan_action       as enum ('entry','exit');
create type scan_result       as enum ('granted','denied','warning');
create type notification_channel as enum ('whatsapp','email','sms');
create type notification_status  as enum ('queued','sending','sent','failed','cancelled');
create type offline_policy    as enum ('deny','allow_cached');
```

### 3.2 Tables

```sql
-- Multi-Ashram ready
sites (id, name, address, timezone default 'Asia/Kolkata', created_at)

-- Staff/app users (1:1 with auth.users)
user_profiles (
  id uuid pk references auth.users,
  name, email, phone,
  role user_role not null,
  can_override_capacity bool default false,
  can_override_scan     bool default false,
  can_block_people      bool default false,
  status ('active','disabled'),
  created_at
)

events (
  id, site_id fk, name, location,
  start_date date, end_date date, timezone,
  status event_status,
  created_at, archived_at
)

-- All operational settings for an event (one row per event)
event_settings (
  event_id pk fk,
  slot_duration_minutes int default 30,
  default_slot_capacity int default 5,
  arrival_early_minutes int default 15,   -- window opens 15 min before
  arrival_late_minutes  int default 15,   -- window closes 15 min after
  appointment_pass_mode ('single_use','reusable') default 'single_use',
  scan_auto_record bool default false,    -- false = show ALLOW ENTRY button
  offline_policy offline_policy default 'deny',
  reminder_offsets_minutes int[] default '{1440,120}',
  venue_text, pass_instructions text,
  meeting_zone_id fk zones,               -- zone where appointment passes are valid
  public_booking_enabled bool default true,
  updated_at, updated_by
)

people (
  id, person_code text unique,            -- e.g. SEVA-1042, VIS-0007
  full_name, photo_path,
  phone, whatsapp, email,
  phone_normalized, email_normalized,     -- E.164 / lower-case, indexed for dedupe
  city, address, organization,
  person_type person_type,
  status person_status default 'active',
  blocked_reason, blocked_at, blocked_by,
  emergency_contact_name, emergency_contact_phone,
  notes,
  is_demo bool default false,
  search_vector tsvector generated,       -- name, code, phone, email
  created_at, updated_at, created_by, archived_at
)

-- Slots are concrete per date (generated from a template, then editable)
slot_templates (id, event_id, label, start_time time, end_time time, capacity, sort_order, active)
slots (
  id, event_id fk, template_id fk null,
  slot_date date, start_time time, end_time time,
  starts_at timestamptz generated/denormalized,   -- in event timezone
  capacity int not null check (capacity >= 0),
  active bool default true,
  notes,
  unique (event_id, slot_date, start_time),
  archived_at
)

appointments (
  id, appointment_code unique,            -- APT-26-0042
  event_id fk, person_id fk, slot_id fk null,   -- null while pending w/o slot
  preferred_date date, preferred_time text,     -- what the visitor asked for
  purpose, description, special_notes,
  accompanying_count int default 0,
  status appointment_status,
  source appointment_source,
  capacity_override bool default false, override_reason, override_by,
  confirmed_at, confirmed_by,
  cancelled_at, cancelled_by, cancel_reason,
  arrived_at, entered_at, meeting_started_at, completed_at,
  is_demo bool,
  created_at, updated_at, archived_at
)
appointment_status_history (id, appointment_id, from_status, to_status, from_slot_id, to_slot_id, changed_by, reason, changed_at)

zones (id, event_id fk, name, description, capacity int null, is_restricted bool, active, sort_order, archived_at)

gates (
  id, zone_id fk, name, scanner_code unique,
  default_action scan_action null,        -- entry-only / exit-only gate if set
  active, archived_at
)
-- Which security user is posted at which gate (optional, for audit & defaults)
gate_assignments (id, gate_id, user_id, valid_from, valid_until)

-- Zone access permissions (Seva / staff / guests)
permissions (
  id, event_id fk, person_id fk, zone_id fk,
  permission_type text,                    -- 'seva','staff','guest','vip', free text/enum later
  valid_from timestamptz, valid_until timestamptz,
  check (valid_until > valid_from),
  entry_allowed bool default true,
  exit_allowed  bool default true,
  max_entries int null, max_scans int null,
  active bool default true,
  revoked_at, revoked_by, revoke_reason,
  notes, created_by, created_at
)

-- QR credentials (the ONLY thing a QR points at)
credentials (
  id,
  token text unique not null,              -- 128-bit CSPRNG, base64url (22 chars)
  kind credential_kind,
  person_id fk,
  appointment_id fk null,                  -- set for appointment_pass
  status credential_status default 'active',
  issued_at, issued_by, revoked_at, revoked_by, revoke_reason,
  expires_at timestamptz null,
  replaced_by uuid null                    -- rotation chain (lost card)
)

access_logs (                               -- append-only
  id bigint identity,
  event_id, person_id null, credential_id null, permission_id null, appointment_id null,
  zone_id, gate_id,
  action scan_action,
  result scan_result,
  reason_code text,                        -- 'OK','EXPIRED','BLOCKED','NOT_AUTHORIZED','ALREADY_INSIDE',...
  reason_text text,
  is_override bool default false, override_by, override_reason,
  overrides_log_id bigint null,            -- links override to the original denied scan
  scanned_by uuid fk user_profiles,
  device_id text,
  scanned_at timestamptz,                  -- device time (may differ when offline)
  received_at timestamptz default now(),   -- server time
  was_offline bool default false,
  raw_token_hash text                      -- for unknown/forged tokens (never store raw)
)

zone_presence (
  person_id, zone_id, primary key (person_id, zone_id),
  inside bool, last_entry_at, last_exit_at, last_log_id
)

audit_logs (                                -- append-only
  id bigint identity, user_id, action text, entity_type, entity_id,
  summary text,                            -- "Coordinator changed APT-26-0042 from 10:00 to 10:30"
  before jsonb, after jsonb, ip, user_agent, created_at
)

notification_templates (id, event_id, key, channel, language default 'en', subject, body, active)
notifications (                             -- also acts as the outbox
  id, event_id, person_id, appointment_id null,
  channel notification_channel, template_key,
  kind ('confirmation','reminder','cancellation','reschedule','custom'),
  scheduled_for timestamptz, status notification_status,
  attempts int, provider, provider_message_id, error,
  sent_at, created_at
)

rate_limits (key text pk, window_start timestamptz, count int)   -- if not using Upstash
```

### 3.3 Indexes (main ones)

- `people (phone_normalized)`, `people (email_normalized)`, `people (whatsapp)` – dedupe & search
- `people using gin (search_vector)` + `pg_trgm` on `full_name` – fast fuzzy search
- `appointments (event_id, status)`, `appointments (slot_id) where status in (active statuses)`, `appointments (person_id)`
- `slots (event_id, slot_date, start_time)`
- `permissions (person_id, zone_id, valid_from, valid_until) where active`
- `credentials (token)` unique, `credentials (person_id)`
- `access_logs (zone_id, scanned_at desc)`, `access_logs (person_id, scanned_at desc)`, `access_logs (event_id, scanned_at desc)`
- `zone_presence (zone_id) where inside`
- `audit_logs (entity_type, entity_id)`, `audit_logs (created_at desc)`

### 3.4 Key views (for dashboards / reports)

- `v_slot_stats` – per slot: capacity, booked (confirmed/rescheduled/arrived/entered/in_progress/completed), arrived, entered, completed, no_show, available, fill state (`available` / `almost_full` ≥80% / `full` / `in_progress` / `done`).
- `v_zone_stats_today` – per zone: authorized people (active permissions overlapping today), inside now, entries/exits/denied today.
- `v_today_kpis` – the KPI cards.
- `v_person_current_status` – currently inside which zones.

### 3.5 Database functions (SECURITY DEFINER, role-checked inside)

| Function | Purpose |
|---|---|
| `book_appointment(person_id, slot_id, purpose, …, override bool, override_reason)` | Locks slot, counts active bookings, rejects when full unless caller may override; creates appointment + history + audit |
| `transition_appointment(id, to_status, reason, new_slot_id?)` | Enforces state machine (below); on cancel/reject revokes pass credential; on confirm issues pass + queues notifications |
| `issue_credential(person_id, kind, appointment_id?)` / `revoke_credential(id, reason)` / `rotate_credential(id)` | QR lifecycle |
| `scan_credential(token, gate_id, action, device_time, device_id, offline bool)` | **The access decision** (see §4) |
| `override_scan(log_id, reason)` | Admin/permitted security override; writes new log row linked to original |
| `set_person_status(person_id, status, reason)` | Block / unblock (immediate effect, audited) |
| `mark_no_shows(event_id)` | Cron: confirmed appointments whose window closed with no entry → `no_show` |
| `rebuild_presence(event_id)` | Recompute `zone_presence` from logs (repair tool) |

Audit rows are written by these functions plus generic `AFTER UPDATE/DELETE` triggers on `people`, `permissions`, `slots`, `zones`, `gates`, `event_settings`, `user_profiles`.

### 3.6 Appointment state machine

```
pending ──approve──▶ confirmed ──scan──▶ arrived/entered ──▶ in_progress ──▶ completed
   │                  │   ▲                                   
   │                  │   └── reschedule (→ 'rescheduled', treated as confirmed for new slot)
   ├──reject──▶ rejected
   └──cancel──▶ cancelled ◀──cancel── confirmed / rescheduled
                confirmed/rescheduled ──(window closed, cron)──▶ no_show
```

`arrived` vs `entered`: `arrived` = passed the outer gate (Main Ashram); `entered` = scanned at the meeting-zone gate. If only one gate is used, a single scan sets `entered` (with `arrived_at` stamped too). `in_progress` / `completed` are set by the coordinator at the meeting door with one tap.

---

## 4. QR design & scan decision logic

### 4.1 QR content

- Token: 16 random bytes from `crypto.getRandomValues` → base64url (22 chars). Unguessable (2¹²⁸), non-sequential.
- Encoded as a URL: `https://<domain>/q/<token>`.
  - A normal phone camera opening it lands on a harmless public page ("This is a Gurudev Ashram pass. Please show it at the gate.") – optionally showing the visitor's own pass when it's an appointment pass. **No personal data is in the QR.**
  - The in-app scanner extracts the token and calls the scan API.
- The token only *points at* a `credentials` row. Everything (person, status, permissions, appointment) is read live from the DB on every scan, so blocking, revoking or changing permissions takes effect instantly.
- Copied/forged QRs: unknown tokens are denied and logged (`raw_token_hash`); revoked tokens show "Pass revoked"; a person card scanned at two different zones' entry gates without an exit, or repeated denied scans of the same token, are flagged as **suspicious** on the dashboard. Security always sees the photo + name to compare with the physical person.

### 4.2 `scan_credential` algorithm

```
input: token, gate_id, action (entry|exit), override? (separate call)
zone := gate.zone

1. credential := lookup(token)
     none        → DENIED  "Unknown QR"            (log, rate-limit device)
     revoked     → DENIED  "Pass revoked/replaced"
     expired     → DENIED  "Pass expired"
2. person := credential.person
     blocked     → DENIED  "ID is blocked"
     archived    → DENIED  "ID inactive"
3. if credential.kind = appointment_pass:
     appt := credential.appointment
     appt.status in (cancelled, rejected)  → DENIED "Appointment cancelled"
     appt.status = pending                 → DENIED "Appointment not confirmed"
     zone ∉ appointment zones (meeting zone + main ashram)  → DENIED "Not valid for this zone"
     appt date ≠ today                     → DENIED "Appointment is on <date>"
     now < start − early                   → WARNING "Entry window begins at 10:15"
     now > start + late                    → DENIED/WARNING "Time window expired" (override allowed)
     single_use and already entered meeting zone → WARNING "Pass already used at 10:21"
   else (person_card):
     perms := active permissions for (person, zone) where not revoked
     none at all                → DENIED "Not authorized for this zone"
     none where valid_from ≤ now ≤ valid_until:
         all in future          → DENIED "Access starts <time>"
         else                   → DENIED "Access expired"
     action=entry and not entry_allowed / action=exit and not exit_allowed → DENIED
     max_entries / max_scans reached → DENIED "Entry limit reached"
     (if the person also has an appointment today and zone is the meeting zone,
      the appointment rules above are applied instead – one card works for both)
4. presence := zone_presence(person, zone)
     entry & inside      → WARNING "Already inside <zone> since 08:32" (override allowed)
     exit  & not inside  → WARNING "Not currently inside"             (override allowed)
5. GRANTED → (if auto-record or after ALLOW ENTRY tap) update presence,
             update appointment status/timestamps, write log
6. ALWAYS write access_logs row (granted, denied or warning) in the same transaction
return {result, reason, person{name, code, type, photo_url (signed, 60s)}, zone, valid_until, appointment{time, code, purpose}, log_id}
```

Two-step flow when `scan_auto_record = false` (default, matches "ALLOW ENTRY" button): the first call is `mode='check'` (logs a check), the confirm call `mode='commit'` records the entry/exit. A decision token (signed, 60 s TTL) binds the confirm to the check so the confirm cannot be replayed on another person.

Exit scans are always permitted for valid identities even after permission expiry (people must be able to leave); this is configurable.

### 4.3 Offline behaviour

- Scanner page and assets are precached (installable PWA). Banner shows **ONLINE** / **OFFLINE** (+ queued count).
- Default `offline_policy = deny`: when offline, scans show **🟡 CANNOT VERIFY – OFFLINE**, the scan is stored in an IndexedDB queue and replayed on reconnect (logged with `was_offline = true`, server re-evaluates and flags any mismatch). Access is never silently granted.
- Optional `allow_cached` (admin-enabled per event): scanner downloads a signed snapshot of *currently valid* token-hash → {name, code, photo, zone permissions, valid_until} for its gate's zone, refreshed every few minutes, max age configurable (e.g. 30 min). Offline grants are clearly marked "OFFLINE – CACHED DECISION" and reconciled on sync. Blocked/revoked after the snapshot are reported on sync.

---

## 5. Screens / pages

### Public (no login)

| Route | Screen |
|---|---|
| `/book/[eventSlug]` | Public appointment request form (name, contacts, city, purpose, description, preferred date, preferred slot – only open slots shown, accompanying count). Captcha (Turnstile) + rate limit. Result: "Request received – status PENDING". |
| `/q/[token]` | Landing page for a QR opened with a normal camera; for appointment passes shows the pass (name, date, time, venue, QR, instructions). |
| `/login` | Staff login. |

### Admin / Coordinator / View-only (`/admin`, desktop-first, responsive)

| Route | Screen |
|---|---|
| `/admin` | **Dashboard**: event header (name, dates), date picker (defaults today), KPI cards, **slot-wise table** with colour states, **live zone cards**, **live security feed** (Realtime), suspicious-activity panel. |
| `/admin/appointments` | List with filters (date, slot, status, purpose, source), quick actions (approve, reject, reschedule, cancel, mark in-progress/completed), bulk approve. Pending queue tab. |
| `/admin/appointments/new` | Admin booking: search/create person (dedupe warning), pick date → slot grid with live availability, override toggle (authorized users only, reason required). |
| `/admin/appointments/[id]` | Detail, status timeline, pass preview, resend WhatsApp/email, history. |
| `/admin/slots` | Calendar/day grid of slots per date; generate slots from templates for a date range; edit capacity per date; disable; booked/available counts. |
| `/admin/people` | Search (name, code, phone, WhatsApp, email, appointment code), filter by type/status, import CSV. |
| `/admin/people/new`, `/admin/people/[id]` | **Person profile**: header (photo, code, type, status, block/unblock), contacts, **current status** (inside which zones), permissions list (active/expired/revoked + "not authorized" zones), appointments, full access history, audit trail, credentials (issue / revoke / rotate, print ID card). |
| `/admin/people/[id]/card` & `/admin/cards/print` | Printable ID card (CR80 85.6×54 mm) single or batch sheet (A4, 10 per page). |
| `/admin/permissions` | List & bulk-grant (select many people × zones × validity window), revoke, filter expired/active. |
| `/admin/zones` | Zones CRUD + gates CRUD (name, scanner code, default action), assign security users to gates. |
| `/admin/reports` | Tabs: Appointments, Slots, Access, Person Activity, Zones. Date filters → table + export CSV / Excel / PDF. |
| `/admin/audit` | Read-only audit log with filters (user, entity, action, date). |
| `/admin/settings` | Event (name, dates, location, timezone), slot duration/capacity defaults, arrival tolerance, pass mode, scan auto-record, offline policy, reminder timings, notification templates, provider config status. Super Admin only. |
| `/admin/events` | Create / switch / archive events (reuse for future visits, optionally clone slots/zones/settings). |
| `/admin/users` | Staff users, roles, override flags, disable user. |

### Security scanner (`/scan`, mobile-first PWA)

| Screen | Content |
|---|---|
| Gate selection (first time / change) | Pick gate (pre-selected if assigned). Shows Zone + Gate in a fixed top bar. |
| Main | ONLINE/OFFLINE pill, gate name, **ENTRY / EXIT** big toggle, giant **📷 SCAN QR** button, manual code entry fallback (Person ID / Appointment ID). |
| Result | Full-screen **green / red / yellow** with large text: result, name, photo, type/code, zone, valid until / appointment time, reason. Buttons: **ALLOW ENTRY / RECORD EXIT**, **OVERRIDE** (if permitted, requires reason), **SCAN NEXT**. Optional vibration + sound. |
| Recent scans | Last 20 scans by this device/gate. |
| Today's list | Authorized people for this zone today (searchable), useful when the camera fails. |

---

## 6. Notifications

```ts
interface MessageChannelProvider {
  channel: 'whatsapp' | 'email' | 'sms';
  send(msg: { to: string; templateKey: string; variables: Record<string,string>; mediaUrl?: string }):
    Promise<{ providerMessageId: string }>;
}
```

- Implementations: `ConsoleProvider` (dev/demo), `ResendEmailProvider` / `SmtpEmailProvider`, `WhatsAppCloudApiProvider` (Meta) as a reference; Gupshup / Interakt / Twilio can be added as new adapters. Selected by env config.
- Appointment confirm / reschedule / cancel → rows inserted into `notifications` (outbox). Vercel Cron (every minute) sends queued rows with retry + backoff.
- Reminders: on confirm, rows are scheduled at `slot.starts_at − offset` for each configured offset; cancelled automatically on cancel/reschedule.
- Messages contain name, date, time, venue, instructions and the **pass link** `/q/<token>` (WhatsApp templates must be pre-approved; the link renders the QR).

---

## 7. Security checklist

- Supabase Auth (bcrypt hashing by provider), HTTPS via Vercel, secure HTTP-only cookies (`@supabase/ssr`).
- RLS on every table; service-role key only on the server, never shipped to the browser.
- All mutations via Server Actions / route handlers → Zod validation → role check → DB function.
- QR tokens: CSPRNG, non-sequential, revocable, rotatable; no PII in QR.
- Rate limiting: scan API per user/device (e.g. 60/min), public booking per IP (e.g. 5/hour) + Turnstile captcha, login throttled by Supabase.
- Photos in private bucket, short-lived signed URLs.
- Append-only logs (no UPDATE/DELETE privileges on `access_logs`, `audit_logs`).
- Soft delete; no hard deletes from the UI.
- CSP & security headers; dependency audit in CI.
- PII minimisation in exports (export action itself is audited).

---

## 8. Project structure (planned)

```
/app
  /(public)/book/[eventSlug]  /q/[token]  /login
  /(admin)/admin/...           (layout with sidebar, role guard)
  /(scanner)/scan/...          (minimal layout, PWA manifest scope)
  /api/scan  /api/cron/notifications  /api/cron/no-shows  /api/export/[report]
/components  (ui/, dashboard/, scanner/, forms/, cards/)
/lib
  /supabase   (server, browser, admin clients; generated types)
  /auth       (getUserWithRole, requireRole)
  /qr         (token generation, QR render)
  /scan       (client: camera, offline queue)
  /notifications (provider interface + adapters + templates)
  /reports    (queries + csv/xlsx/pdf exporters)
  /validation (zod schemas)
/supabase
  /migrations  (SQL: enums, tables, indexes, RLS, functions, views)
  seed.sql     (DEMO data)
  /tests       (SQL tests for scan & booking functions)
/tests/e2e     (Playwright – the 10 acceptance tests)
/docs
```

---

## 9. Implementation plan (phases, each ends with tests passing)

| Phase | Scope | Exit test |
|---|---|---|
| **1. Foundation** | Next.js + Tailwind + shadcn scaffold, Supabase project config & CLI, migrations for **all** tables/enums/indexes, RLS policies, `user_profiles` + role guard, login/logout, admin layout shell, events + settings CRUD (Super Admin), audit-log trigger infrastructure, CI (lint, typecheck, unit, SQL tests). | Each role can log in and only reaches allowed routes; RLS tests prove a viewer/security user cannot write. |
| **2. People, slots, appointments** | People CRUD + dedupe + search + photo upload; slot templates → generate slots, capacity per date; `book_appointment` with locking & override; appointment list/detail/state machine; public booking form; pending queue approve/reject/reschedule. | Acceptance **Test 8** (capacity), concurrent-booking SQL test, state-machine tests. |
| **3. QR appointment passes** | `credentials` lifecycle, pass issued on confirm and revoked on cancel, `/q/[token]` pass page, downloadable pass PNG/PDF, appointment rules inside `scan_credential` (time window, single-use). | **Tests 1, 2** at function level. |
| **4. Seva permissions & zones** | Zones & gates CRUD, gate assignments, permissions CRUD + bulk grant, person-card credentials, printable ID cards (single + batch), block/unblock. | **Tests 3, 4, 7** at function level. |
| **5. Scanner & entry/exit** | `/scan` PWA (camera, gate select, entry/exit toggle, colour result screens, confirm/override), presence tracking, duplicate entry/exit warnings, offline queue + status pill, rate limiting. | **Tests 1–7, 9** end-to-end in Playwright (fake camera input / manual code). |
| **6. Dashboard & reports** | KPI cards, slot-wise table, zone cards, live feed via Realtime, person profile history, 5 reports with filters, CSV/Excel/PDF export, audit log viewer, suspicious-activity panel. | **Test 10** (counts update live after scans), report totals reconcile with logs. |
| **7. Notifications, demo data, deployment** | Provider interface + console/email/WhatsApp adapters, outbox + reminders cron, no-show cron, DEMO seed (20 people, 10 appointments, 5 seva volunteers, 4 zones, 3 gates, permissions, granted/denied scans, mixed statuses), Vercel deployment, runbook for Ashram staff. | Full acceptance suite (Tests 1–10) green against a deployed preview; manual run-through on Android + iPhone. |

Clock control for tests: `scan_credential` accepts an injected "now" only in test mode (SQL tests set it via a session variable), so Test 3 (scan at 10 AM vs 7 PM) is deterministic.

---

## 10. Decisions needing confirmation

These have sensible defaults (in **bold**) and will be used unless changed:

1. **Single QR per person also works for their appointment** (in addition to a per-appointment pass link sent by WhatsApp)? Default: **yes – both work; appointment pass is revoked on cancel, the person card is not.**
2. Appointment pass behaviour: **single-use at the meeting zone** (configurable to reusable).
3. Scan recording: **show "ALLOW ENTRY" confirm button** (configurable to auto-record).
4. Offline policy: **deny / queue** (cached allowlist available as opt-in).
5. Arrival window: **15 min before → 15 min after** slot start.
6. Who may override scan decisions: **Super Admin & Admin; Security only if granted the flag.**
7. WhatsApp provider: none chosen yet → **console/log adapter + Meta Cloud API adapter scaffold**; email via **Resend** (or SMTP). Which provider/accounts does the Ashram already have?
8. Hosting accounts: a Supabase project and a Vercel project are needed (free tiers are enough for development; **Supabase Pro recommended for the live event** for backups & no pausing).
9. Language: **English UI for V1**, text externalised for Gujarati/Hindi later.
10. ID card print size: **CR80 (credit-card) size**, A4 batch sheets.
