---
name: bow-read-path
description: Catch the recurring <PROJECT_KEY> "read-path miss" bug class BEFORE QC does — a value renders wrong, empty, or mis-routes because one layer of the read path (DB SELECT → model field → getter → UI render / router) drops it, OR a surface reads from a non-source-of-truth column, OR two surfaces read the same value from two different sources. These shipped analyze-green + test-green (cancel reason, phone country-code, offer ETA, push deep-link type, login role). The SAME class hits the web apps too (admin/partner Next.js, read path DB → Supabase select → React query hook → cell/detail render): phone shown without the `+` prefix, a missing field, a rating rendering 0 when there are no reviews, list items displayed without images/prices and mislabeled, a raw enum token shown instead of its label, a value attributed to the wrong party. Use when adding or editing a field that a screen displays or a router keys off, when a QC bug says "X shows blank / wrong / stale on THIS screen but right on another", when binding a Supabase column into a Flutter widget OR a React cell/detail field (admin/partner), or when the same value appears on two surfaces (mobile↔web included). Pairs with [[bow-model]], [[impact-sweep]], [[parity-with-sibling-surface]] and [[verify-runtime-not-just-static-green]].
---

# Read-path review — why a value shows wrong on ONE surface

Every bug below passed `fvm flutter analyze` / `tsc` + unit tests when it shipped.
They are not type errors — the value never travelled the full read path, or it
travelled a **different** path than the surface next to it. QC finds them one
surface at a time and files them as unrelated tickets; they are the same class.

> This is a **review lens** for the *read* direction. For the model mechanics use
> [[bow-model]]; for full blast-radius use [[impact-sweep]]; for DB/runtime truth
> use [[verify-runtime-not-just-static-green]].

The read path is almost always four layers:

```
DB column ──▶ query SELECT ──▶ Model field (fromJson) ──▶ getter/VM ──▶ Text()/router
```

A value shows wrong when **any one link is missing**, or when a sibling surface
uses a **shorter/different** chain. Walk all four before saying "done".

---

## 1. New displayed field → add it to ALL FOUR layers, not just the widget
The most common shape (a courier cancellation reason). The column existed, the
sender screen and the list card already showed it — only the courier **detail**
surface was blank, because the fix was needed in three separate places:

1. **Query SELECT** — the assigned-delivery query never requested
   `cancellation_reason`. A column absent from `SELECT` arrives as `null` in the
   payload; `fromJson` happily parses `null`; nothing errors.
2. **Model field** — the courier model had no `cancellationReason` field (its
   sibling list-item model did — that's why the *list* was fine).
3. **UI render** — the page never emitted the row and the VM never exposed a getter.

**Do:** when you add a field to a screen, grep the value from the DB column outward
and confirm each link exists:
```bash
grep -rn "cancellation_reason" apps/mobile/lib   # SELECT strings + fromJson keys
grep -rn "cancellationReason"  apps/mobile/lib   # model field + getters + widgets
```
If a **sibling surface already shows the value** (list card, sender screen), that
surface's chain IS your checklist — copy its SELECT columns and model field. The
bug is almost always "the new surface has a shorter chain than the old one".

## 2. Read from the SOURCE OF TRUTH, not a convenient copy
A value can live in two columns; reading the wrong one compiles clean and is wrong
for a subset of rows.

- **Login role** — signin/signup read the role from `auth.users.user_metadata['role']`,
  but `public.users.role` is the source of truth (the sync-to-metadata trigger omits
  role). Social signups had the role in `public.users`, `NULL` in metadata → existing
  users bounced to *Select Role*. Prod proved it: most rows had `public.users.role`
  set, fewer had metadata set.

**Do:** before binding a column, ask *"is this the column the write path writes, or a
mirror that can drift?"* Prefer the table the mutation updates. If you must read a
mirror, verify on prod that it's fully backfilled (`count(*) where mirror is null`),
and fix the trigger/backfill — don't just read around it.

## 3. Two surfaces, one value → one source, one formatter
If surface A and surface B show "the same" value from **different** inputs, they will
disagree and QC files the discrepancy.

- **Offer ETA** — the courier offer card rendered a fixed admin SLA window
  (1h/2h/4h/8h) while Booking Details used the real routed duration. A 2.1 km trip
  showed "~2h" on the card and "8 min" on details. Both were "correct" for their own
  source — the sources disagreed.
- **Receiver phone** — the pending-review snapshot echoed the raw national-only
  controller text (the `+33` lives in a separate country-prefix picker), while the
  persisted booking stored the canonical `+33…` via the shared phone formatter. Same
  number, two renderings.

**Do:** when a value appears on ≥2 surfaces, make them share the **same input and the
same formatter**. Route the review snapshot through the SAME formatter the persist
path uses. Never re-derive a display value from raw controller text when a canonical
helper already exists.

## 4. Crossing a boundary (edge fn / FCM / router) → carry the routing key
A field dropped while crossing a service boundary breaks *every* consumer that keys
off it, but QC only tests one.

- **Push deep-link** — `fcm-send` built `message.data` from `body.data` only and never
  merged the `type` (it arrived as a separate enqueue param). The device got `data`
  without `type`; the mobile router keys **every** deep-link off `data['type']` →
  `null` → fell to `default` → opened the feed instead of chat. It broke message /
  ride_* / order_* / support / express_* alike — QC tested only "tap a message push".

**Do:** at any boundary that reassembles a payload (edge fn, FCM data map, navigation
args, RPC return), list the keys the **receiver** reads and confirm each survives the
crossing. For a router, the switch key (`data['type']`) is load-bearing — assert it in
a test on the *builder* side, not just the router side.

## 5. The web apps (admin/partner) have the SAME four links
The path just has React names, and every link fails the same way:

```
DB column ──▶ Supabase .select() ──▶ query hook (apis/<domain>) ──▶ cell / detail field
```

- **Missing SELECT column** (§1's twin): a column absent from the `.select('…')` in
  `apps/{admin,partner}/src/apis/<domain>/request.ts` arrives `undefined`; the cell
  renders blank (a missing VAT field; an owner phone absent from the payload).
- **Formatter on the render side, not the raw value** (§3's twin): a phone shown
  without `+` is the raw national number reaching the cell instead of the canonical
  `+…` — route it through the **same** phone formatter the write path uses
  (`libphonenumber-js`), don't render `row.phone`. A raw enum token is a status/type
  printed instead of its label map.
- **Empty vs zero vs missing** — a rating of 0 for a business with **no** reviews is
  "no data", not the number 0; items with valid data mislabeled "Sold out" / no
  image/price read an availability/stock field that is `null`≠`0`≠`out`. Distinguish
  `null`/empty from a real 0 at the cell.
- **Wrong source of truth** (§2's twin): a commission attributed to the wrong party
  read the party from a derived/convenient field instead of the ledger's own column.

**Do:** for a new admin/partner cell, grep the snake_case column through
`apps/*/src/apis/<domain>` (is it in the `.select`?) and the camelCase field through
the feature, exactly like the Flutter grep. Reuse the shared formatter/label map;
never print a raw column or a bare `0`.

---

## The 30-second read-path check (run before "Ready for QA")
For the value you touched:
- [ ] **SELECT** — every column the screen needs is in the query string (grep the
      snake_case column; a missing column silently parses to null).
- [ ] **Model** — the field exists on the model the surface uses (not just its
      sibling model), and `.g.dart` is regenerated.
- [ ] **Getter/VM** — exposed to the widget; one getter, one meaning.
- [ ] **Source of truth** — reading the column the write path writes, not a mirror
      that can drift (verify backfill on prod if it's a mirror).
- [ ] **Sibling surfaces** — every other screen/card showing this value reads the
      SAME source through the SAME formatter (grep the value across `apps/`).
- [ ] **Boundaries** — any routing/display key survives every edge-fn / FCM /
      nav-arg reassembly; assert it on the builder side.

Green here = the value is correct on **all** surfaces, not just the one you opened.
"analyze pass" proves none of these — see [[verify-runtime-not-just-static-green]].
