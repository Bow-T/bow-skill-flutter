---
name: codemagic-build-prod
description: Trigger Codemagic production builds of the mobile app on the develop branch — BOTH Android APK and iOS IPA → Firebase App Distribution. Use when the user says "build production trên develop codemagic", "build production codemagic", "build prod develop", "build APK production", or otherwise asks to kick a Codemagic production/Firebase build.
---

# Codemagic — Build Production (develop → Firebase)

Trigger **production-flavor** builds of `apps/mobile` on Codemagic via the
`codemagic` MCP. Result: `app-production-release.apk` **and** an ad-hoc `.ipa`
(both built from `lib/app/main_prod.dart` + `.env`) distributed to **Firebase App
Distribution** (tester group `du-test`). The Slack `notify_slack_success` step
updates a single "latest build" message in place per platform.

> This is the **Firebase** production path. It is NOT Google Play / TestFlight —
> those are the `android-play` / `ios-testflight` workflows, triggered by
> `release/*` tags, and are out of scope here.

## Platform scope — read this first

Codemagic runs **exactly one workflow per `trigger_build` call**, and
`codemagic.yaml` has no combined workflow. So "build production" = **two
separate triggers**.

| what the user said | trigger |
| --- | --- |
| "build production" (no platform named) | **BOTH** `android-firebase` + `ios-firebase` |
| names Android only ("chỉ android", "build APK") | `android-firebase` only |
| names iOS only ("chỉ iOS", "build IPA") | `ios-firebase` only |

Fire the two triggers **back-to-back** (they run in parallel on Codemagic) —
don't wait for Android to finish before starting iOS.

## Steps

### 1. Trigger the build(s) — `mcp__codemagic__trigger_build`

Shared arguments for **both** workflows:

| arg | value | why |
| --- | --- | --- |
| `app_id` | `6a2fd0d4a19317a8091fcc73` | the registered app "monorepo" |
| `branch` | `develop` | default build branch (override only if the user names another) |
| `variables` | `{"FLAVOR": "production"}` | both workflows default to `FLAVOR: development` (`codemagic.yaml:247` / `:313`) — this override is what makes it a **production** build |
| `instance_type` | `mac_mini_m2` | The Codemagic token is a personal account (no team) → `linux_x2` / `linux_x4` / `mac_mini_m4` are unavailable and the build dies in ~15s with 0 steps. Both workflows already declare `mac_mini_m2` in the yaml; pass it anyway so a yaml drift can't silently break the trigger. |

Per-workflow:

| `workflow_id` | produces | typical duration |
| --- | --- | --- |
| `android-firebase` | `app-production-release.apk` | ~10–12 min |
| `ios-firebase` | ad-hoc `.ipa` (needs the App Store Connect integration + `ios_certificate` group named in the workflow's `integrations:` / `groups:` block) | ~20–25 min |

> `workflow_id` does NOT show up in `list_workflows` until a workflow has run
> once — pass the string directly, don't look it up.

### 2. Confirm each one actually started — `mcp__codemagic__get_build`
Pass each returned build ID. A healthy build reaches `initializing` → `fetching`
→ `building`. If it shows `failed` with **"not started"** and **0 steps in ~15s**,
the instance type was rejected — re-trigger making sure `instance_type` is
`mac_mini_m2`.

### 3. Report
Give the user **both** build URLs
(`https://codemagic.io/app/6a2fd0d4a19317a8091fcc73/build/<id>`) with the status
of each, and note iOS finishes noticeably later than Android. Offer to poll and
report when each finishes (artifact link + Slack confirmation).

If only one platform was triggered because the user named it, say so explicitly
so the missing platform is never a silent omission.

## Notes
- Codemagic reads `codemagic.yaml` **from the branch being built** — a change to
  the build/Slack config only takes effect once it is merged into `develop`.
- Before triggering a fresh build, if a prior identical build is still running,
  point the user to it instead of spawning a duplicate (avoids double Firebase
  upload + Slack races). Check per platform — Android may be free to re-run while
  iOS is still going.
