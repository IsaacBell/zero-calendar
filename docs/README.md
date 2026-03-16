# Zero Calendar — Deployment & Fixes Report

This document summarizes the work done to deploy and fix Zero Calendar.

---

## Context

The client provided:
- Access to his Vercel workspace
- Google OAuth credentials (Client ID & Secret)
- Groq API key for AI chat features

The goal was to fork Zero Calendar, deploy it, and get it working.

---

## Step 1 — Fork & Initial Setup

Forked the [Zero Calendar repository](https://github.com/Zero-Calendar/zero-calendar) to a personal GitHub account first to work on fixes without affecting the client's repository. Created an Upstash KV database (`kv-zero-calendar`) in the Washington, D.C. (iad1) region through Vercel Storage for testing. Configured environment variables (KV credentials, NextAuth secret, Google OAuth, Groq API key).

---

## Step 2 — Fix Build Failures

The app couldn't build out of the box:

- **Missing `tsconfig.json`** — all `@/*` imports were unresolved, so the build failed on every component. Added the file with the correct path alias configuration.
- **Vulnerable Next.js version** — Vercel blocked deployment because Next.js 15.2.4 had a known vulnerability. Updated to `^15.3.0`.
- **Dependency conflicts** — React 19 conflicts with some peer dependencies. Resolved by installing with `--legacy-peer-deps`.

---

## Step 3 — Fix Core Functionality

Once the app built and deployed, several features were broken:

- **Events couldn't be created, updated, or deleted** — the KV database calls were running in the browser where server-only environment variables are unavailable. Fixed by marking `lib/calendar.ts` as a server module (`"use server"`).
- **Wrong user ID on calendar page** — the code used `session.user.sub` but NextAuth only sets `session.user.id`. Events weren't loading for the logged-in user.
- **Categories crashed the event dialog** — the category dropdown tried to render full objects instead of strings, throwing a React error.
- **Duplicate calendars in the sidebar** — the same calendar appeared twice due to duplicate entries in KV.
- **AI chat ignored events without a title** — saying "create an event tomorrow at 10 AM" did nothing because the missing title was treated as an invalid request. Now defaults to "New Event".

---

## Step 4 — Fix Google Calendar Integration

The Google Calendar integration existed in the codebase but was non-functional:

- **Credentials never stored** — Google OAuth tokens were kept in the browser session only, but the sync code reads from the KV database. Added credential persistence to KV on sign-in.
- **Events from Google never appeared** — the calendar view only read local events from KV. Added fetching and merging of Google Calendar events.
- **No sync on create/update/delete** — events created in the app never reached Google Calendar. Added automatic sync to Google after every create, update, and delete operation.
- **All-day events crashed the cache** — birthday and all-day events from Google use a different date format that caused database errors. Added proper handling.

---

## Step 5 — Apply Fixes to Client's Fork

Once all fixes were tested and working on the personal fork, added it as a remote on the client's forked repository and pulled all commits to bring in the changes. Removed the remote afterward. Switched the KV database configuration to one created under the client's Vercel account and deployed from his repository.

---

## Detailed Documentation

- [CHANGES.md](./CHANGES.md) — full technical details of every bug fix and improvement
- [GOOGLE_CALENDAR_ISSUES.md](./GOOGLE_CALENDAR_ISSUES.md) — known remaining issues with the Google Calendar integration
