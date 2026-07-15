# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project overview

QuiVient? is a French-language event RSVP app ("Organisez. Invitez. Gérez facilement." — organize events, share a link, let people sign up). Organizers create an event with one or more date/time slots and a capacity per slot; participants pick a slot and register with first/last name (and optional email); slots that fill up automatically move new registrants to a waitlist that promotes people as spots open up.

## Architecture

**The entire application is a single file: `index.html`.** There is no build step, no bundler, no package.json, and no test suite — it's opened directly in a browser or served as a static file. All CSS, HTML, and JavaScript live in that one file, structured as:

1. `<style>` — all CSS, organized by section with comment headers (HERO, CARDS, FORMS, SLOT GRID, EDIT MODAL, etc.)
2. `<body>` — markup for four "pages" (`#page-home`, `#page-create`, `#page-event`, `#page-dashboard`, `#page-registrations`), only one visible at a time via the `.page.active` class
3. `<script>` — all application logic, in one global scope (no modules, no framework)

### Client-side "SPA" routing

There's no router library. `nav(page, eventId)` (search for `// ── ROUTING`) toggles the `.active` class between the `#page-*` divs and calls the loader function for that page (`loadHome`, `loadDashboard`, `loadEvent`, `loadRegistrations`). Deep links use a `?event=<slug>` query param resolved by `handleDeepLink()` on `DOMContentLoaded`, which looks up the event by slug and navigates to it.

### Data layer: Supabase

- The Supabase client is initialized lazily by `initDB()`, which checks that `SUPABASE_URL`/`SUPABASE_KEY` (top of `<head>`) have been configured; if not, it shows a config banner instead of loading data.
- Tables used directly via the JS client (no ORM, no server-side code): `events`, `event_slots`, `registrations`.
  - `events`: title, description, category, location, max_spots, slug, is_recurring, days_label, created_at
  - `event_slots`: belongs to an event; date, time_start, time_end, max_spots
  - `registrations`: belongs to a slot/event; first_name, last_name, email, status (`confirmed` / `waitlist` / `cancelled`), waitlist_pos, registered_at
- Waitlist logic (promotion when someone cancels, renumbering remaining waitlist positions) is done client-side with sequential Supabase queries — see `cancelMyReg()` and `removeReg()`. There's no DB trigger/RPC for this; any change to waitlist behavior must keep these two functions consistent since they duplicate the same promotion/renumbering logic.

### Auth model

There is no real backend auth — "admin" access (create/edit/delete events, view the dashboard and registrations list) is gated by a hardcoded password (`ADMIN_PASS`, currently `'2183'`) checked client-side in `loginAdmin()`, with the logged-in flag stored in `sessionStorage` (`qv_admin`). This is not secure and should not be treated as a real authorization boundary — anyone with the page source can read the password. `supprimerEvenement()` (delete event) additionally re-prompts for the same code inline.

Participant self-service (viewing "you're registered" state, cancelling your own registration) is tracked via `sessionStorage` keys per slot (`qv_reg_<slotId>`, `qv_slot_<slotId>`), not a real user session — it only works on the same browser/device that made the registration.

### Email notifications

EmailJS (loaded via CDN, `@emailjs/browser`) sends a notification when a waitlisted participant is promoted to confirmed (`sendPromotionEmail()`). Configured via `EMAILJS_SERVICE_ID` / `EMAILJS_TEMPLATE_ID` / `EMAILJS_PUBLIC_KEY` near the top of the `<script>` block. If the public key is left as the placeholder, sending is silently skipped.

## Configuration

Supabase credentials are hardcoded at the top of `index.html`:

```js
const SUPABASE_URL  = '...'
const SUPABASE_KEY  = '...'  // anon/public key
```

There is no `.env` or config file — to point the app at a different Supabase project, edit these two lines directly. The anon key is meant to be public (RLS policies on the Supabase project, not this key, are what should restrict access), but avoid weakening that assumption when editing.

## Development workflow

- No install, build, or dev server step is required — open `index.html` in a browser (or serve the directory with any static file server) to run the app against the live Supabase backend defined in the file.
- No linter or test suite exists in this repo; verify changes manually in a browser.
- Because everything is one file, prefer targeted edits over restructuring; keep new CSS near its related section and new functions near the feature they belong to (routing, event creation, event detail, dashboard, registrations, edit modal) to match the existing organization.
- The UI strings are in French; keep new user-facing text consistent with the existing tone/language unless told otherwise.
