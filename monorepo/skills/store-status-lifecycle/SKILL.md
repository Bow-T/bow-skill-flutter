---
name: store-status-lifecycle
description: Catch the recurring <PROJECT_KEY> store/business status-lifecycle bug class BEFORE QC does — one entity's live state (a store being active / paused-by-partner / suspended-by-admin / removed) must read IDENTICALLY and re-sync on EVERY surface that shows or gates on it: the customer store list + profile (mobile), the admin Business Management list/filter/detail/audit, the partner suspension banner, courier dispatch of an already-placed order, and the DB visibility RLS. A Sprint-8 sweep logged ~9 tickets that were each one surface of this single concept failing (a courier couldn't get an order from a suspended store, the store list didn't refresh after reactivate, admin showed empty for a paused store, the partner banner was stale/vanished, a filter option was missing, suspend/pause history was wiped on reactivate). The trap: the state is NOT a stored enum — it is DERIVED from orthogonal timestamp columns on `public.stores` (`paused_at`, `suspended_at`, `removed_at`, `suspension_lifted_at`, `is_active`), and the "is it suspended right now" derivation is re-implemented in FOUR places (SQL predicate, admin view CASE, partner TS, Flutter model) that drift. Use when adding or changing a store/business status value or precedence, a suspend/pause/reactivate/remove path, any surface that displays store availability or "Temporarily Unavailable", the courier dispatch/visibility of a store's orders, or when a QC bug says "status wrong/stale on THIS surface", "doesn't refresh after reactivate", "history lost". Generalises [[express-realtime-review]] §1 (status → every surface) to a timestamp-derived entity state; pairs with [[impact-sweep]], [[flutter-realtime]], [[bow-read-path]], [[parity-with-sibling-surface]] and [[supabase-migration]].
---

# Store/business status lifecycle — one state, five surfaces, four derivations

A store's live state (active / paused / suspended / removed) is **not** a column you
can grep for one token. It is **derived** from timestamp columns on `public.stores`,
and every surface re-derives it in its own language. When you touch the lifecycle,
`analyze`/`tsc`/`bun test` stay green while one surface reads the state differently or
never re-reads it — and QC files each surface as its own ticket. Walk the code you
touched against these five checks.

## Know the source of truth before you touch a leg
- **Truth = `public.stores` timestamp columns** — there is **no `status` enum / no
  `status IN (...)` CHECK**. State is encoded as: `paused_at` (partner self-pause,
  `is_active` stays true), `suspended_at`+`suspension_lifted_at` (admin), `removed_at`
  (terminal), `is_active` (visibility flag).
- **Precedence: `removed` > `suspended` > `paused` > `active`** — defined once in the
  admin `businesses_admin` view CASE ("removed outranks suspended; admin suspension
  outranks the partner's own pause").
- **The vocabulary string is a mirror on every surface**: admin `apis/businesses/types.ts`
  (`'active'|'paused'|'suspended'|'removed'`), partner `lib/store-status-view.ts`,
  Flutter `store_model.dart` (`isActive`, `isPaused`). Any new value / precedence
  change must be made in the view **and** all three mirrors.

---

## 1. "Suspended right now" is ONE predicate — call it, don't re-null-check
Suspension is not `suspended_at IS NOT NULL`: a lifted suspension keeps `suspended_at`
for history, so the live test compares it against `suspension_lifted_at`.

- **Canonical predicate:** `store_suspension_in_force(suspended_at, suspension_lifted_at)`
  — its own migration header lists the **four** SQL readers that silently broke when
  they used a raw `suspended_at IS NOT NULL` instead.
- **Mirrors that must match it:** partner `lib/store-suspension.ts` `isSuspendedFrom`
  (compares timestamps, not null), `isPausedFrom` (pause shows only when not
  suspended). Flutter currently reads `paused_at` only (`store_model.dart` `isPaused`)
  and leans on RLS to hide suspended stores.
- **Rule:** every reader — SQL, admin view, partner TS, Flutter — answers "suspended
  now?" the same way. If you add a state or change precedence, grep a peer predicate
  (`grep -rn "store_suspension_in_force\|suspended_at" supabase apps`) and update each
  hit. A raw null-check anywhere is the bug waiting to re-fire.

## 2. A status flip must reach EVERY reader — the four mirrors + RLS
The same store state is consumed on five surfaces; changing the lifecycle without
sweeping all of them drops it on the one you didn't open.

- **DB/RLS visibility** — customer sees a store only when
  `is_active AND removed_at IS NULL AND NOT store_suspension_in_force(...)`; products of
  suspended stores hidden; order-insert is gated by
  `_enforce_store_not_paused_on_order_insert` → raises `store_temporarily_unavailable`.
- **Admin** — vocabulary + filter (`apis/businesses/types.ts`), filter options incl.
  "Paused by partner" (`features/businesses/components/BusinessesToolbar.tsx`), status
  tag + audit drawer (`BusinessDetailDrawer.tsx`).
- **Partner** — banner (`components/product/store-paused-banner.tsx`), derivation
  (`lib/store-suspension.ts`), load-on-login/tab-switch (`stores/use-store-status.ts`).
- **Mobile** — list/profile availability (`store_queries.dart`, `store_model.dart`,
  `search_result_item.dart`, `store_profile_page.dart`).
- **Rule:** treat a lifecycle change as Cross-cutting ([[impact-sweep]]): grep a peer
  state already wired across all five and make its hits your checklist. "Done" = every
  surface updated, not "the write path compiles".

## 3. Visibility ≠ dispatch — suspending a store must NOT strand an in-flight order
Hiding a store from customers is a different question from stopping the courier flow
of an order that was **already placed** (a courier couldn't receive the pickup offer
once the store was suspended — it SHOULD still dispatch).

- **Correct today:** `match-order-courier` reads the store via the service-role client
  for coordinates only, with **no** store-status predicate; the only gate is
  `order.status === 'ready_for_pickup'`. Don't add an `is_active`/suspension filter to
  the dispatch or the courier-side offer read.
- **Rule:** put store-status filters on the **customer visibility** path (RLS, store
  list, new-order creation), never on the fulfilment path of an order that already
  exists. When you add a suspension check, ask "customer-facing browse, or in-flight
  order?" — only the former filters. If this recurs, look for a `stores.is_active` join
  on the courier offer/order query, not on dispatch.

## 4. The customer surface must RE-SYNC when the state flips live
A store row read once and cached goes stale; the mobile store list has **no realtime**
and won't reflect a reactivate.

- **Trap:** list VMs are pull-to-refresh/pagination only — no `subscribe`/`channel`
  (`category_stores_vm.dart`). The single `stores` subscription is profile-page
  **rating-only** and short-circuits any non-rating change (`realtime_service.dart`
  `subscribeToStoreRating`, `store_profile_vm.dart`:
  `if (row['rating_avg'] == store?.ratingAvg) return;`). A `paused_at`/reactivate flip
  pushes no update anywhere.
- **Rule:** a surface that gates user action on store availability must re-read that
  state when it can change under the user — subscribe to the store row (widen the
  existing rating channel to also fire on `is_active`/`paused_at`/`suspended_at`, or
  re-fetch on screen focus), and select the columns the availability flag needs
  (`store_queries.dart` reads `is_active,paused_at` — add `suspended_at` if the surface
  must distinguish it). See [[flutter-realtime]] (own the channel, dispose it, apply
  the server's filter) and [[express-realtime-review]] §6 (re-sync on live change).

## 5. Reactivate must PRESERVE history — don't wipe the audit column-set
The reverse transition is not "clear the timestamp". Admin history needs the prior
suspend/pause to survive.

- **Trap/fix:** reactivate sets `suspension_lifted_at` and deliberately does **not**
  clear `suspended_at` (`admin-suspend-store`); partner reactivate keeps
  `last_paused_at`/`last_pause_reason`. The admin drawer reads exactly those
  ("Suspended at/by", "Paused by partner at", "Reactivated at/by"); the actor names
  come from `audit_logs`, not `stores`.
- **Rule:** when a lifecycle write reverses a state, add the *lifted/reactivated*
  column instead of nulling the *entered* one, and backfill the full related
  column-set (SQL-hygiene: `reviewed_by`⇒`reviewed_at`). A surface that shows history
  (admin) must read the preserved columns, not the live flag.

---

## Before you say a store-status change is done
- [ ] "Suspended now?" uses `store_suspension_in_force` (or its exact mirror) on every
      surface — no raw `suspended_at IS NOT NULL` (§1).
- [ ] New state/precedence swept across all five surfaces: RLS/DB, admin
      vocab+filter+detail, partner banner, mobile list+profile (§2, [[impact-sweep]]).
- [ ] Store-status filter is on customer-visibility only, never on the dispatch /
      in-flight-order path (§3).
- [ ] The customer surface re-syncs on a live flip (realtime widened or re-fetch on
      focus) and selects the columns its availability flag needs (§4, [[flutter-realtime]]).
- [ ] Reactivate/reverse preserves the suspend/pause history columns; the audit
      surface reads them (§5, backfill full column-set).
- [ ] Runtime verified end-to-end (suspend → customer hidden, order still dispatches,
      admin/partner/mobile all reflect it, reactivate restores + keeps history), not
      just analyze/tsc/test green ([[verify-runtime-not-just-static-green]]).
