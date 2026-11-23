# Smart Roster — Roster Management System

A browser-based tool for an operations manager to **plan, generate, validate and change monthly rosters** across multiple brands, processes and pods. Built for the Datastraw hiring assignment.

- **Stack:** React 18 + Vite, plain JavaScript. No backend — the scheduling engine runs in the browser; data is kept in `localStorage` (a "Reset sample data" button restores the original).
- **Sample data:** 36 people (32 agents + 4 pod leads), 4 pods, 25 brands (20 single-resource = 80%, 5 multi-person), 42 processes, holidays, leaves, comp-off. All fictional.
- **Live app:** _add your deployed URL here_

## Run locally

```bash
npm install
npm run dev        # http://localhost:5173
npm test           # engine tests (data shape, generator invariants, change handling)
npm run build      # static build in dist/
```

No environment variables are needed (so there is no `.env.example`).

## Deploy (free, ~2 minutes)

1. Push this folder to a GitHub repo.
2. On [vercel.com](https://vercel.com) (or Netlify): **Add New → Project → import the repo**. Framework preset: *Vite*. Build command `npm run build`, output `dist`. Deploy.

## Tour (what to look at)

| Tab | What it shows |
|---|---|
| **Roster** | Company / Pod / Brand / Process / Employee views. Click any cell to edit (validated instantly); click a date header for "who is working today"; click a red coverage cell for fixes. |
| **Issues** | Every problem, split into *mandatory* (error) vs *preference/risk* (warning) vs info, with a plain-language "why" and ranked fixes you can apply. |
| **Change Lab** | Preview a change *before* committing: employee leave (single owner / 1-of-3 / no-backup presets) or a brand requirement change. |
| **Manage Data** | Add / edit / remove employees, pods, brands, processes, calendars, minimums, assignments, holidays, leaves — no code changes. |

Header actions: **Regenerate** (keeps manual edits and unchanged rest days), **Auto-repair**, **CSV export**, **Publish** (blocked while mandatory issues remain unless explicitly acknowledged), Undo/Redo, change log.

## Assumptions (all configurable in the app)

- Weeks are Mon–Sun. A week with ≥ 5 days in the month needs at least one rest day (weekly off, leave or comp-off).
- A process's **operating calendar** is one of: 365 days · every day except public holidays · closed Sundays & public holidays. It is separate from each employee's schedule.
- When *all* of a person's processes are closed (Sunday-type closures), that day is their weekly off; public holidays are "PH" and don't consume the weekly off.
- Minimum staffing can be set per process, per weekday (e.g. 2 on Sat/Sun) and for holidays.
- Working a public holiday creates a comp-off entitlement (shown as info; a manager approves it).
- Max 6 consecutive working days. Allocation per person ≤ 100% (per *brand*; processes of one brand share that slice).
- Pod leads have no process assignments and follow Mon–Sat, non-holiday.
- Backups must be in the same pod, have the process-type skill, be working that day and share the shift. Cross-pod backups are suggested but flagged as needing approval.
- Sample holidays are illustrative.

---

## Tell us how you think

### 1. How did you model employees, brands, processes, pods and their relationships?
Plain data, no hard-coded logic:
`Pod → Employees` (a pod has a lead), `Brand → Processes` (each process has a type CS/ACC/RET/NDR, an **operating calendar**, minimum staffing, day-specific minimums, a shift), and `Assignment = (employee, process, allocation %)`. Assignments are the many-to-many link, which naturally supports: one person on all processes of a brand, several people on different processes, several people sharing one process (e.g. NutriNest CS needs 2 of 3), and one person across two brands (Ishita 60/40). Leaves, comp-offs and holidays are lists. The roster itself is a map `employee|date → {status, backup-for}`, so it is separate from the rules that produced it.

### 2. How does the system generate a roster and decide whether it satisfies the constraints?
Two separate pure functions:
- **Generator** (deterministic, explainable): (1) fix approved leave/comp-off and closed days; (2) rotate holiday work fairly, only as many people as required; (3) place weekly offs for the hardest-to-cover people first, each on the cheapest day — cost = coverage shortfall (heavy) + teammates already off (stagger) + non-preferred day (light) − stability with the previous roster; (4) fill remaining gaps with eligible backups; (5) cap consecutive working days.
- **Validator**: takes *any* roster — generated or hand-edited — and checks coverage per process per open day, leave conflicts, weekly rest, consecutive days, allocation, shift clashes, invalid backups, plus preferences and risks. The generator is allowed to fail; the validator is the source of truth, so an invalid roster is never silently produced.

### 3. How did you handle single-resource brands vs brands with multiple employees?
Same model, different behaviour that falls out of the data. A single-owner brand on a *Mon–Sat* calendar is trivial: Sunday is the weekly off, no gap. A single owner on a *365* calendar can never take a weekly off without a backup, so the generator looks for a skilled same-pod backup (who can take over every process of that brand for the day) and, if none exists, leaves a **visible gap** plus a structural "no backup" warning. For multi-person brands the generator staggers offs (penalises two teammates off together) and respects higher minimums (e.g. 2 on weekends).

### 4. What if one person manages all processes of a brand but is unavailable on a required day?
The validator raises a **coverage gap** for each affected process/day with the reason ("Pooja is the only person on it and is on leave"). The app then simulates candidate fixes against the whole roster and ranks them: a same-pod backup covering all processes of that brand, swapping a teammate's weekly off, or calling someone in on a holiday (comp-off due). Only fixes that genuinely reduce total blocking issues are offered; cross-pod options are labelled as needing approval. If nothing works it says so and tells the manager what decision is needed (add a skilled backup, approve overtime, relax the requirement). Try it: *Change Lab → "Single owner unavailable"*.

### 5. How does it handle conflicting requirements or insufficient staffing?
By priority: **mandatory beats preference**. Required coverage, leave, rest days, shift and allocation limits are errors and block publishing; a preferred weekly-off day is only a warning and is overridden (and reported) when coverage needs it. Insufficient staffing is never hidden — it appears as an error with the cause and the best available mitigation. Publishing with errors needs an explicit "acknowledge & publish" and the acknowledged items are recorded.

### 6. What did you automate, and what is left under human control?
Automated: first-draft generation, fair holiday rotation, backup selection, validation, fix ranking, impact preview, optional auto-repair. Human-controlled: approving leave and comp-offs, choosing among fixes (nothing is applied without a click, except the opt-in "Auto-repair"), cross-pod borrowing, accepting unresolved gaps, and publishing. Manual cell edits are *locked* so regeneration never overrides a manager's decision. The reasoning: scheduling is mechanical, but trade-offs between people and clients are judgment calls with real-world consequences.

### 7. If the company doubled its team and brand count, what would you change?
Move from browser-only to a database + API with auth and per-pod permissions; run generation server-side (or in a worker) and store versioned rosters/audit logs. Replace the greedy generator with a constraint solver (OR-Tools CP-SAT) for global optimality and fairness, keeping the current validator as the independent checker. Index coverage incrementally instead of re-validating the whole month on each edit, virtualise the grid, and add pod-lead approval and real notifications.

### 8. With one more week, what would I improve first?
Multi-month awareness: rest and consecutive-day rules across month boundaries (currently the month start assumes a rest day before it). After that: pod-lead approval workflow, fairness reporting (Sundays/holidays worked per person over time), comparing alternative rosters side by side, and real notifications/email when a published roster changes.

---

## Project layout

```
src/engine/    dates · rules · context · generate · validate · backup · suggest   (pure JS, no React)
src/data/      sample.js            (fictional dataset)
src/components Grid · Views · Issues · ChangeLab · Manage · About
src/store.jsx  state, undo/redo, persistence, actions
scripts/       test-engine.mjs
```
