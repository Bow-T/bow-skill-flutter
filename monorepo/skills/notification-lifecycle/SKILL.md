---
name: notification-lifecycle
description: Catch the recurring <PROJECT_KEY> notification bug class BEFORE QC does — the whole push+in-app lifecycle for a status/event, not just one leg of it. A Sprint-7 sweep logged ~20 notification bugs that were each a different leg of the SAME five questions failing: (1) is a push EMITTED on every transition (incl. per-stop legs); (2) in the RECIPIENT's language, not the sender's/app's; (3) exactly ONCE (the app does NOT dedupe by type); (4) does tapping it route to the LIVE target and get WITHDRAWN when that target dies; (5) do the in-app list + bell badge re-sync in realtime. Use when adding or editing anything that emits a notification (a new status transition, an enqueue trigger, fcm-send, notification_templates), when adding a notification `type`, when wiring notification tap→route, when touching the bell badge / notification list / unread count, or when a QC bug says "no push / wrong language / duplicate / opens wrong screen / badge stale". Pairs with [[bow-i18n]] (template localization), [[express-realtime-review]] (badge realtime filter), [[flutter-realtime]] and [[impact-sweep]].
---

# Notification lifecycle — the five questions every notification must pass

`fvm flutter analyze` + unit tests were **green** on every Sprint-7 notification
bug. They are cross-layer lifecycle defects: a status flips in the DB, but one leg
of the pipeline (enqueue trigger → `fcm-send` → device → tap-route → in-app list →
badge) drops it. QC logs each dropped leg as its own ticket, so **one** missing
concept becomes five tickets. Before declaring any notification change done, walk
the code you touched against these five questions. Each cites the real Sprint-7
site so you can see the shape.

> This skill is a **review lens**, not a build guide. For building the UI use
> [[bow-ui]]; for template localization use [[bow-i18n]]; for the runtime audit use
> [[verify-runtime-not-just-static-green]].

The pipeline (know it before you touch a leg):

```
DB status/event change
  └─ enqueue trigger (supabase/migrations/*_notify_*, *_notification_enqueue_triggers.sql)
       └─ INSERT public.notifications (type, user_id, data)   ← in-app row + trigger to push
            └─ fcm-send (supabase/functions/fcm-send/index.ts) ← localizes + fans to devices
                 └─ device push  ──tap──▶ notification_service.dart / _notification_routing.dart
                 └─ realtime INSERT ──▶ notification_vm list + notification_badge unread count
```

---

## 1. EMIT on EVERY transition — including per-stop legs a booking-level trigger skips
A push is enqueued by a trigger that reacts to **one** column. If the user-visible
event can happen more than once (multi-stop delivery, re-tries, sub-states), the
single booking-level trigger fires once and every later leg is silent.

- **Trap (<PROJECT_KEY>-2376 / 2375):** the dropoff phase is *stop-driven* —
  `_express_stop_sequencing` flips the next stop to `in_transit`/`arrived` on
  `express_delivery_stops`, but the "In transit"/"Arrived" push was enqueued by the
  **booking-level** `_notify_express_deliveries_status_change` on
  `express_deliveries.status`, which reaches `in_transit` exactly once. Sender got a
  ping for Stop 1 and nothing for Stops 2, 3, …
- **Rule:** when a phase is driven by a child table (stops, items, attempts), the
  notification trigger belongs on the **child transition**, not only the parent
  status. Ask: "can this event recur? does a trigger fire on each recurrence?"
- Also check the transition even *fires* the enqueue: <PROJECT_KEY>-2507 (no push at
  all on a booking status update) was a status path with no enqueue trigger wired.

## 2. Localize to the RECIPIENT's language, not the app's / caller's
`fcm-send` reads `users.language_code` for the **recipient** and, when
`locale !== 'en'`, swaps in the `notification_templates (type, locale)` row;
**if that row is missing it falls back to the caller-supplied English string**.
So a new `type` that ships an English body but **no FR `notification_templates`
row** silently delivers English to every FR user.

- **Trap:** <PROJECT_KEY>-2374 ("No Courier Found" EN under FR), 2268 (push FR while
  app EN), 2255 ("New delivery request" not localized), 2152 (courier pickup push EN).
- **Rule:** every new notification `type` must `INSERT ... ON CONFLICT (type,
  locale) DO UPDATE` a row for **each** supported locale (EN **and** FR) in the same
  migration — see the `ON CONFLICT (type, locale)` template in [[bow-i18n]]. Dynamic
  parts (`{entity}`, `stop_order`, amounts) render per-locale
  (`Intl.NumberFormat(locale, …)`), never string-concatenated in one language. If
  the app re-localizes off `type` + `data` (the `stop_order` pattern), the
  switch/localizer must carry EN **and** FR.

## 3. Exactly ONCE — the app does NOT dedupe by type
Two triggers that both react to overlapping conditions, or one trigger that fires on
a transition already announced elsewhere, deliver a **duplicate** push. There is no
client-side dedupe — suppress at the emit site.

- **Trap:** <PROJECT_KEY>-2376 had to guard the *first* leg (already announced by the
  booking-level `in_transit` push) with a "subsequent-leg only" predicate (a sibling
  stop is already terminal) to avoid double-firing. <PROJECT_KEY>-2512 (duplicate
  "Account Approved" on first login), 2444 (wrong/extra push on vehicle-update reject).
- **Rule:** when you add an enqueue trigger, grep the peers on the same table
  (`grep -rln "_notify_" supabase/migrations | xargs grep -l "<table>"`) and prove no
  other trigger already emits for this exact transition. If two can both fire, add a
  suppression predicate at the **enqueue** site, not the client.

## 4. Tap routes to the LIVE target — and the notification is WITHDRAWN when it dies
A notification's tap-target is a snapshot. If the underlying entity changes state
(offer expires, booking cancelled/accepted, row deleted) the stale notification
either mis-routes or opens a dead screen — and it must be removed, not linger.

- **Trap:** <PROJECT_KEY>-2501 (expired request notif not removed, opens unavailable
  offer), 2499 (accepted-booking notif opens the offer instead of the active
  booking), 2373 (courier returns to an expired request after accepting a new one),
  2440 (nav not prevented when the delivery no longer exists), 2459/2382 (tap opens
  map / wrong Booking Detail).
- **Rule:** (a) map `type` → destination in a unit-tested table
  (`_notification_routing.dart`), never inline in the router — add the new `type`
  there so the regression test catches missing wire-up; (b) the destination must
  **re-resolve the entity live** and route by its *current* state (offer still open?
  → offer; accepted? → active booking; gone? → list + toast), not by the payload
  snapshot; (c) when the entity reaches a terminal/!offer state, the enqueue side
  must **dismiss** the now-dead notification (the "dismiss resolved offer notifs"
  migration pattern). This is the query→model→getter→UI read-path miss class applied
  to routing: verify every layer forwards the value, not just the model.

## 5. In-app list + bell badge re-sync in REALTIME with the server's filter
The notification list and the unread badge are fed by a `postgres_changes` stream;
they must reflect inserts/reads live and count exactly what the server counts.

- **Trap:** <PROJECT_KEY>-2380 (list not updated live), 2383 (badge not updated after
  "Mark All Read"), 2275/2490 (bell shows no unread indicator / overlapped). And the
  badge over-count class: the realtime handler did `unreadNotifCount += 1` on
  **every** INSERT while `countUnread()` chains `.neq('type','message')` — see
  [[express-realtime-review]] §3-4.
- **Rule:** the realtime handler applies the **same predicate** the authoritative
  query uses (share it); "Mark all read" updates the local count *and* the stream
  reconciles it; the badge derives from the same `countUnread()` source on refresh.
  Subscribe in a service, dispose the channel in the VM ([[flutter-realtime]]).

---

## Before you say a notification change is done
- [ ] Push fires on **every** occurrence of the event, incl. per-stop/child-table
      legs — not just the once-only parent status (§1).
- [ ] EN **and** FR `notification_templates` rows exist for the new `type`; dynamic
      parts localized per-recipient-locale (§2, [[bow-i18n]]).
- [ ] No duplicate: no peer trigger already emits this transition; first-leg /
      overlap guarded at the enqueue site (§3).
- [ ] Tap maps in a unit-tested table, re-resolves the entity LIVE, routes by
      current state, and the stale notif is dismissed when the target dies (§4).
- [ ] List + badge update in realtime with the server's filter; "mark read"
      reconciles (§5, [[express-realtime-review]], [[flutter-realtime]]).
- [ ] Runtime verified end-to-end (enqueue → fcm-send → device → tap → list/badge),
      not just analyze/test green ([[verify-runtime-not-just-static-green]]).
