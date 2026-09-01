# Forja — case study

**Offline-first hybrid-training app.** Guided strength sessions, cardio logging, a daily
habit protocol, and progress history — in one app that stays fully usable with no connection.

**Try it:** [app.forjahybrid.com](https://app.forjahybrid.com) · **Site:** [forjahybrid.com](https://forjahybrid.com) · **iOS:** TestFlight (closed beta)

> This repository is a write-up. It contains **no product source code** — Forja is a
> commercial product under a proprietary license. What follows is the engineering
> reasoning behind it.

**Role:** sole engineer — architecture, implementation, CI, release.
**Stack:** React Native (Expo SDK 54) · React 19 · TypeScript 5.9 strict · Zustand · Supabase (Auth + Postgres + RLS)

---

## The constraint that shaped everything

People train in basements, in gyms with no signal, on trails. An app that needs the network
mid-workout is an app that fails exactly when it is being used.

So the device is the **source of truth**, not a cache. Local storage is authoritative; the
server is a backup that reconciles in the background. Every design decision downstream falls
out of that one.

---

## Three problems worth writing about

### 1. Deletion is the hard half of sync

Last-write-wins on updates is easy. The trap is deletion: if a delete is modeled as "row is
gone", any peer that still has the row will helpfully sync it back on the next round, and the
record resurrects. The user deletes a workout, closes the app, reopens it — and it is there again.

Forja stores deletions as **tombstones**, and runs them through the *same* last-write-wins rule
as any other write, compared on `updatedAt`. A delete is not the absence of a record; it is a
record that says "deleted, at this time". Resurrection stops being possible by construction
rather than by patch.

### 2. A test suite that runs in UTC cannot see date bugs

A habit tracker lives on the concept of "today". "Today" is a local-timezone question, and the
CI machine runs in UTC — where local day and UTC day are the same day, so **no test can tell
the two apart**. Every off-by-one-day bug passes CI and ships.

The suite therefore has a guard that runs the date-sensitive tests under `TZ=Asia/Tokyo`, far
enough from UTC that any test conflating the two fails immediately. It runs on every push,
alongside the main suite.

### 3. Architecture that erodes silently unless something stops it

The codebase is organized by feature, with strict layers: `service → hook → component → screen`.
Services never import React; screens never hold business logic.

Written down, that rule decays within a month. So it is **enforced**: an automated gate fails
the pull request when a layer boundary is crossed, alongside checks for oversized files,
function-level debt, and orphaned modules. Nine stages run on a pre-push hook before anything
reaches the remote — the same ones CI runs, in the same order, so the feedback arrives before
the push rather than after.

---

## Data protection

Every table carries row-level security with **both** `USING` and `WITH CHECK` scoped to the
authenticated user. `USING` alone is the common mistake: it filters what a user can read, and
leaves them able to *write* rows attributed to someone else. Authorization is verified by a
two-user test that asserts each side cannot reach the other's data, run before every release.

Only the anonymous key ever reaches the client. Build and deploy secrets exist solely in CI.

---

## What is in the product

Strength templates with sets, reps, load, rest and per-exercise notes, run as a guided session
(exercise → set → rest timer → next), reorderable mid-workout. Cardio plans and logs for running,
cycling, swimming and brick, by workout type and intensity, with HR zones and RPE. Customizable
daily habits with streaks — where logging a workout checks off "exercise" on its own. A monthly
progress calendar, personal records by estimated 1RM (Epley) and top load, volume by muscle group,
and estimated recovery. Body measurements, 43 achievements, CSV import/export, reminders,
Sign in with Apple, and account deletion in-app.

---

<!-- Screenshots go here. Add 3-4 PNGs (session runner, progress calendar, habits, PR chart)
     to an images/ folder and reference them below. This is the highest-value addition to
     this page — recruiters look at pictures before they read. -->

## Decisions on record

Architecture decisions are recorded as ADRs in the private repository — framework choice,
offline strategy, backend, state management, UI library, navigation, i18n, and the tombstone
LWW rule that resolved problem 1 above.

---

*Questions about any of this are welcome — [LinkedIn](https://www.linkedin.com/in/eduardo-visconti/).*
