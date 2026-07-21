---
name: parity-with-sibling-surface
description: Catch the recurring <PROJECT_KEY> "new surface built without mirroring its sibling" bug class BEFORE QC does. A Sprint-7 sweep logged ~20 bugs that were all one shape — Express Delivery (admin + mobile) shipped as a sibling of the established Ride/Booking surface but was NOT diffed against it, so it was missing columns, filters, action buttons, totals, tooltips, page titles, ID fields, timeline grouping, and used different role terminology. Each gap was logged as its own "inconsistent with Ride booking" ticket. Use when building or editing a surface that mirrors an existing one — delivery↔ride, courier↔user, admin↔mobile, express↔store, a new list/detail modeled on an existing list/detail — or when a QC bug says "inconsistent with X", "missing column/field/button vs X", "different title/terminology than X". Pairs with [[impact-sweep]] and [[bow-ui]].
---

# Parity with a sibling surface — diff against the reference, don't rebuild from memory

When you build surface **B** (Express Delivery bookings) as a sibling of an existing
surface **A** (Ride bookings), the spec is usually "same as A, for deliveries". The
failure mode is building B from memory of what a list/detail "should have" instead
of **diffing against A's actual code** — so B silently drops half of A's fields,
columns, filters, actions and copy. `analyze`/tests pass (B is internally valid); QC
opens A and B side by side and files each difference as a separate ticket.

> This skill is a **review lens**, not a build guide. For building the UI use
> [[bow-ui]]; for blast-radius mapping use [[impact-sweep]].

Sprint-7 receipts — **one** missing diff, logged ~20 times:

| A (reference) | B (new sibling) | What B dropped | Tickets |
|---|---|---|---|
| Ride booking **list** (admin) | Express Deliveries list | Action/View (eye) col, Route/Vehicle/Fare cols, total count, filter+reset, partial search, "No Courier Found" filter, ID tooltip | 2415, 2400, 2401, 2404, 2402/2403, 2524, 2525 |
| Ride booking **detail** (admin) | Delivery booking detail | table format, page title, Sender/Courier ID, vehicle type, destination, distance/duration, cancellation fee, per-stop timeline, no dup statuses | 2416, 2424, 2409, 2407, 2408, 2417, 2418 |
| Ride booking **detail** (mobile) | Delivery booking detail | layout, back button, status-timeline order, cancellation-reason field | 2462, 2456, 2484, 2348 |
| Ride role terminology | Delivery terminology | "Rider/Driver" vs "Sender/Courier" used inconsistently | 2526 |

Anchors: admin `apps/admin/src/features/{bookings,express-deliveries}` +
`apps/admin/src/apis/{bookings,express-deliveries}`; mobile ride vs delivery booking
detail under `apps/mobile/lib/src/pages/main`.

---

## The rule: open the reference, enumerate its surface, diff line by line

Before you call a sibling surface done, **open A's code** (not your memory of it) and
build the checklist from A's actual definitions:

1. **Columns / fields** — list every column A's table/list renders and every field
   A's detail shows. For admin lists this is the column array; for detail, the row
   set. B must have each one or a deliberate, noted reason it doesn't.
   *(2400 missing Route/Vehicle/Fare; 2407 missing destination/distance/duration;
   2409 missing Sender/Courier ID.)*
2. **Actions** — every row action and header button on A (View/eye, edit, filters,
   reset, export). *(2415 missing View; 2404 missing filter+reset.)*
3. **Aggregates / affordances** — totals, counts, tooltips, empty states, pagination.
   *(2401 missing total; 2525 missing ID tooltip.)*
4. **Search / filter semantics** — match A's behavior, not just the box: partial vs
   full-match, which fields are searched, which statuses are filterable.
   *(2402/2403 partial search + search-by-name; 2524 missing "No Courier Found".)*
5. **Copy, titles, terminology, layout** — page title format, section titles,
   table-vs-card layout, back button, timeline ordering, role nouns. Reuse A's
   shared component/label helper rather than re-authoring. *(2416, 2424, 2456, 2462,
   2484, 2526.)*

**Reuse over re-author.** If A and B render the same concept, they should share the
component / column config / label map, not two copies that drift. A shared source is
the parity guarantee — see [[bow-lean]] (don't copy logic >5 lines) and
[[impact-sweep]]. When you genuinely must diverge (deliveries have stops, rides
don't), make the divergence explicit and keep everything else shared.

## Two surfaces showing the "same" value must read it the same way
Parity isn't only visual. If A and B (or courier-view vs user-view of one booking)
read the same value from different sources or derive it differently, they disagree
at runtime — the "two surfaces, two sources" read-path miss, and the
[[express-realtime-review]] §2 active-leg class (courier↔sender must derive the
current stop identically). Diff the **read path**, not just the layout.

---

## Before you say a sibling surface is done
- [ ] Opened the **reference surface's code** and enumerated its columns/fields,
      actions, aggregates, search/filter semantics, and copy/titles/terminology.
- [ ] B has each — or a noted, deliberate reason it doesn't.
- [ ] Shared component/column-config/label-map reused, not a drifting copy
      ([[bow-lean]], [[impact-sweep]]).
- [ ] Same value on both surfaces reads from the same source / same derivation
      ([[express-realtime-review]] §2).
- [ ] Verified by viewing A and B side by side, not just analyze/test green.
