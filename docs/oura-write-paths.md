# Posting updates into the Oura ecosystem

Research note: how (and whether) an outside system — specifically the
health-native Android apps — can push **exercise** and
**meal** data into a member's Oura account, so that Oura's own crossing of
sensor data against nutrition and training works without the member typing
everything into the Oura app by hand.

Verified 2026-09-04 against the live Oura OpenAPI document and Oura's own
support articles. This is a research doc only — it is not deployed content and
is unaffected by `web/deploy-report-page.sh`.

---

## 1. The headline: Oura's public API cannot write

Fetched `https://cloud.ouraring.com/v2/static/json/openapi-1.37.json`
(the spec Oura's own `/v2/docs` page loads), API version 2.0:

| | count |
|---|---|
| Total paths | 72 |
| `GET` | 71 |
| `POST` / `PUT` / `DELETE` | 4 — **all on `/v2/webhook/subscription`** |

The only non-`GET` operations in the entire surface manage webhook
subscriptions:

```
GET,POST        /v2/webhook/subscription
PUT             /v2/webhook/subscription/renew/{id}
GET,PUT,DELETE  /v2/webhook/subscription/{id}
```

Every user-data route — `sleep`, `daily_activity`, `daily_readiness`,
`daily_stress`, `workout`, `session`, `tag`, `enhanced_tag`, `heartrate`,
`vO2_max`, `ring_configuration`, … — is `GET` only, in both the live and
sandbox namespaces.

Two consequences worth being explicit about:

- **Tags are not a back door.** `/v2/usercollection/tag` and
  `/v2/usercollection/enhanced_tag` are read-only. You can see the tags a
  member created in the app; you cannot create one. (`EnhancedTagModel` has a
  `comment` freeform field and a `custom_name` — tempting, but there is no
  route that accepts them.)
- **There is no nutrition collection at all.** No `meal`, `nutrition`, or
  `food` path, and no such schema in the spec. So Oura Meals data is not only
  un-writable, it is also **un-readable** — anything a member logs in Meals
  cannot be pulled back out through the API.

Aggregators do not change this. Terra, Vital/Junction, Rook, Spike and Validic
all wrap the same read-only Oura surface; Terra's own Oura page describes a
one-way flow ("Oura pushes to Terra on change") with no write capability. If
Oura has no write endpoint, neither does anything sitting on top of it.

---

## 2. What *does* flow into Oura

Oura ingests third-party data — just not through the developer API. Two
mechanisms exist, both consumer-facing:

### a. The OS health stores (the supported path)

**Apple Health → Oura** reads: Active Energy, Resting Energy, Blood Pressure,
Body Fat %, Cardio Fitness, DOB, Sex, Heart Rate, Height, Weight, Lean Body
Mass, Mindful Minutes, Respiratory Rate, Sleep + Sleep Stages, Steps, Workout
Routes, **Workouts**.

**Health Connect → Oura** reads: Active Calories Burned, Distance,
**Exercise**, Power, Steps, Total Calories Burned, VO2 Max, Weight, Blood
Pressure, Heart Rate.

Note what is *absent from both lists*: **nutrition and hydration**. Apple
Health carries ~80 dietary types and Health Connect has `NutritionRecord` and
`HydrationRecord`; Oura reads none of them. The health stores are a workout
pipe into Oura, not a food pipe.

### b. Partner integrations (mostly outbound)

Oura's Partner Integrations page lists ~18 partners. Nearly all are **Oura →
partner** (Cronometer, Strava, Zero, Clue, Headspace, Natural Cycles, …). Only
three carry data *into* Oura:

- **Dexcom Stelo** — continuous glucose, surfaced against meals.
- **Health Panels** — blood work results.
- **Strava** — activities (see §4).

Notably, **Cronometer is outbound only**: Oura sends sleep and activity data to
Cronometer to sharpen its expenditure model. Nutrition does not come back.

The Stelo and Health Panels slots prove Oura *does* build private inbound
pipes — but on a negotiated, per-partner basis, not via a documented API.

---

## 3. Exercise: the workable options

Ranked by robustness.

### Option A — write to HealthKit / Health Connect from a native app
**Best available path.** Write an `HKWorkout` (iOS) or `ExerciseSessionRecord`
(Android) plus the associated `activeEnergyBurned` / heart-rate samples; Oura
picks it up on its next sync and it counts toward the Activity Score and goal.

- Requires a **native mobile app** with health-store write entitlement. A web
  app or a Supabase edge function cannot do this — HealthKit has no server API.
- Member must enable the integration (Oura app → Settings → Data Sharing →
  Apple Health / Connected apps → Health Connect) *and* grant the per-type
  toggles in the OS health app. Both gates are silent when off.
- **Watch double-counting.** Oura also *writes* Active Energy and Steps into
  the health stores; a naive round trip inflates calories. Oura's own guidance
  is to set a priority data source, or stop sharing steps outbound.

### Option B — no native app: bridge into Apple Health
If you can't ship an app, third parties can write on your behalf:

- **IFTTT iOS Health** service — has a "log a workout" action triggerable from
  a webhook. Cheapest server-driven route.
- **RunGap / HealthFit / Health Auto Export** — import a `.fit` / `.tcx` /
  `.gpx` you generate and write it into Apple Health.
- **Shortcuts automation** — `Log Health Sample` is workable for quantity types
  but clumsy for full workouts (the Type field can't take a variable).

All of these put a manual or per-device step in the loop. Fine for one member,
poor for a product.

### Option C — Strava as a server-side bridge
The only path that is **fully server-side**, since Strava (unlike Oura) has a
real write API:

- `POST /api/v3/uploads` — upload a FIT/TCX/GPX file (scope `activity:write`)
- `POST /api/v3/activities` — create a manual activity

Strava activities then "automatically display as activity cards within the Oura
app" and "count towards your activity goal and Activity Score."

Two constraints decide whether this is usable:

1. **Heart rate appears to be required.** Oura documents importing "recorded
   workouts with heart rate data" from Strava. A bare manual activity
   (`POST /activities`) carries no HR stream and may be ignored — so prefer
   `POST /uploads` with a TCX/FIT that contains an HR series. *Verify this
   empirically before building on it; Oura's wording is not a precise contract.*
2. **Same-day only.** Oura's Strava article is explicit: "Oura syncs activities
   only for the current day, and activities cannot be imported retroactively."
   A nightly batch job will silently drop everything. Push within the day.

---

## 4. Nutrition: there is no inbound path

Stating it plainly, because it constrains the design:

- Not readable from Apple Health or Health Connect by Oura.
- No API endpoint, read or write.
- Cronometer — Oura's own nutrition partner — is outbound only.
- **Oura Meals accepts exactly three inputs, all in-app**: take a photo, upload
  a photo, or type what you ate. Oura Advisor then estimates a Nutrition
  Breakdown (protein, fiber, processing level, added sugars, total fats, total
  carbs) as low/moderate/high bands. There is no import, no barcode scan, no
  sync, and no third-party hook. Requires Gen3+, active membership, English.

So the only ways to get meals into Oura are human-in-the-loop:

- **Generate paste-ready text.** Meals accepts free text, so an app can produce
  a one-line description ("6oz grilled salmon, 1 cup quinoa, roasted
  broccoli") the member pastes into Meals. A share-sheet handoff or a copy
  button makes this a two-tap action rather than re-typing.
- **Hand off the photo.** If the member already photographs meals in your app,
  share that image to Oura Meals; Oura's own AI does the analysis.

Both are member actions, not automation. Neither is scriptable.

---

## 5. What this means for health-native

Investigated `doronperetz/health-native` @ `a824201`. Three sideloaded Android
apps (`com.health.workout`, `com.health.diabetes`, `com.health.creami`) over a
shared `core`, with Supabase edge functions behind them.

### The deployment model is an advantage here

Builds are signed in CI and published as **GitHub Releases, installed via
Obtainium** (`build-*.yml` → `gh release create`); `output/` holds a backup APK.
Nothing ships through Google Play.

That matters, because the gate on Health Connect data types is a *Play
publishing* requirement: you declare data-type access in the Play Console "while
preparing your app for publishing on Google Play," and undeclared types fail at
review. Off-Play, that gate doesn't apply — the permission is an ordinary
runtime grant the user approves in the Health Connect UI. This isn't a
theoretical read: `diabetes` already holds **twelve** health permissions,
including `READ_HEALTH_DATA_IN_BACKGROUND` and `READ_HEALTH_DATA_HISTORY`
(normally among the most scrutinised), and they work in production. Adding a
`WRITE_*` permission is the same shape of change.

`connect-client` is pinned at `1.1.0-alpha08` (deliberately — the comment in
`libs.versions.toml` notes alpha09+ needs compileSdk 35). `insertRecords` is
long-standing API and present in alpha08, so **no version or SDK bump is
required.**

### The plumbing already runs in the wrong direction

Today: **Oura → Health Connect → `diabetes` app → Supabase.** `HealthConnectSource`
is explicitly "Read-only view over Health Connect," with twelve `READ_*`
permissions and no writes anywhere. `HcPermissions` even documents the recovery
metrics as "what a ring-style wearable (e.g. Oura) contributes to Health
Connect." You are already consuming Oura through exactly the pipe you'd need to
push back through.

### Three concrete obstacles, all in your code

**1. The workout app has no Health Connect at all.** Sessions are logged in
`com.health.workout` (`buildSessionLog` → `SyncSessionWorker` → `session_log`),
but every line of Health Connect code lives in `com.health.diabetes`.
`workout/app/src/main/AndroidManifest.xml` declares three permissions — INTERNET,
ACCESS_NETWORK_STATE, POST_NOTIFICATIONS — and no health ones. Separate
`applicationId`s mean separate HC clients and separate user grants. Either add a
writer to `workout` (correct: it owns the data) or route sessions to `diabetes`.

**2. `SessionLog` throws away the timestamps a write needs.** An
`ExerciseSessionRecord` requires `startTime`, `endTime`, and zone offsets.
`SessionLog` carries `date` (a `LocalDate` string) and `durationMin` — and
`buildSessionLog` takes `startTimestamp` only to compute `durationMin`, then
discards it. `durationMin` is also nullable when the start is unknown. **Persisting
start/end instants on `SessionLog` and `session_log` is a prerequisite**, not a
detail.

One nice thing falls out: `id = "$date-$sessionKey"` is already a stable,
one-per-type-per-day key — exactly the shape of `metadata.clientRecordId`, which
makes the HC write idempotent for free.

**3. Writing creates a read-back loop, and one path double-counts.**
`exerciseSessions()` reads *every* `ExerciseSessionRecord` with no `dataOrigin`
filter, so the 6h `HealthSyncWorker` would re-ingest anything you write.
Severity differs by record type:

- **`ExerciseSessionRecord` — survivable.** The round-trip lands a
  `health_sessions` row with `source = "health_connect"` (a constant in
  `HealthMappers`), upserted on `(session_type, start_time, source)`. It won't
  duplicate *itself*, but it will sit alongside the `session_log` row for the
  same physical workout — and alongside Oura's own auto-detected session at a
  *different* start time, which upserts as a separate row. Anything unioning
  those sees one workout as two or three.
- **`ActiveCaloriesBurnedRecord` — actively wrong. Do not write it.**
  `hourlyActiveKcal` uses HC's `aggregate`, which sums across every origin with
  no dedup. Your own code already documents this hazard for steps and works
  around it with `bucketHourlyMaxByOrigin` — but that guard is *steps-only*.
  Active calories and `sessionAggregates` have no equivalent, so writing kcal
  inflates `active_kcal` by exactly the amount you write.

**Write `ExerciseSessionRecord` and nothing else.** No calories, no steps, no
heart rate — Oura is already the authority on all three, and each one you add
double-counts on the way back in. Then filter own-package on read: exclude
`metadata.dataOrigin.packageName == BuildConfig.APPLICATION_ID` in
`exerciseSessions()` (and add a fake-backed test — CLAUDE.md requires one).

### Recommended shape

1. `WRITE_EXERCISE` in the writing app's manifest, requested as **optional** —
   mirror the `HcPermissions.optional` pattern so a denied grant degrades to
   "workout doesn't reach Oura" rather than breaking session logging.
2. A `HealthConnectSink` alongside `HealthConnectSource` — one `insertRecords`
   call, `ExerciseSessionRecord` only, `clientRecordId` = the `SessionLog.id`,
   `exerciseType` mapped from `sessionType` (`exerciseTypeName` in
   `HealthMappers` already has the int table; you need its inverse).
3. Chain it off the existing `SyncSessionWorker` path so it retries offline.
4. Own-package filter on the read side, shipped in the same change.

Expect it to land as a manifest line, ~80 lines of writer, the timestamp change
through `SessionLog`/`session_log`, and the read-side filter.

### Meals: the mobile app does not unlock this

Worth being blunt, because "we have a mobile app so we can publish to Health
Connect" is true for exercise and **still false for food**. Health Connect has
`NutritionRecord` and `HydrationRecord` — but **Oura does not read them**
(§2a). Writing nutrition to Health Connect is a no-op with respect to Oura: the
record lands, Oura ignores it. Oura Meals accepts photo, photo upload, or typed
text, in-app, and nothing else.

So the `analyze-meal-photo` / `evaluate-meal` / `approved_meals` pipeline has no
route into Oura at any price, with or without a native app. Keep meals yours and
do the crossing in your own stack — you already hold both halves, and you get
real macros instead of Oura's low/moderate/high bands. The one member-facing
option is a share/copy affordance into Oura Meals, which is only worth building
if there's a Stelo connected to make Oura's glucose-vs-meal view meaningful.

### Is the exercise write even worth it?

The honest answer is: only for one thing. You already read Oura's data in, so
pushing workouts up adds nothing to *your* reporting. What it buys is Oura's
**own** model seeing training it can't detect — a gym session with no cadence
signal — which feeds its Activity Score and next-day readiness. If members act
on Oura's readiness number, that's a real gap worth closing. If they act on the
coach report, this is effort spent making a second system agree with the first.

## Sources

- Oura OpenAPI spec: `https://cloud.ouraring.com/v2/static/json/openapi-1.37.json` (via `https://cloud.ouraring.com/v2/docs`)
- [Oura — Partner Integrations](https://support.ouraring.com/hc/en-us/articles/10705471244947-Partner-Integrations)
- [Oura — Apple Health Integration](https://support.ouraring.com/hc/en-us/articles/360025438734-Apple-Health-Integration)
- [Oura — Health Connect by Android Integration](https://support.ouraring.com/hc/en-us/articles/10786105824531-Health-Connect-by-Android-Integration)
- [Oura — Strava Integration](https://support.ouraring.com/hc/en-us/articles/10766662499219-Strava-Integration)
- [Oura — Meals](https://support.ouraring.com/hc/en-us/articles/40264659421843-Meals)
- [Oura blog — Oura Meals](https://ouraring.com/blog/oura-meals/)
- [Oura — The Oura API](https://support.ouraring.com/hc/en-us/articles/4415266939155-The-Oura-API)
- [Strava — Oura and Strava](https://support.strava.com/hc/en-us/articles/6619564102157-Oura-and-Strava)
- [Strava API v3 reference](https://developers.strava.com/docs/reference/)
- [Terra — Oura integration](https://tryterra.co/integrations/oura)
- [Health Connect data types](https://developer.android.com/health-and-fitness/health-connect/data-types)
- [HKWorkout — Apple Developer](https://developer.apple.com/documentation/healthkit/hkworkout)
- [IFTTT — iOS Health + Webhooks](https://ifttt.com/connect/ios_health/maker_webhooks)
- [Declare access to Health Connect data types](https://developer.android.com/health-and-fitness/guides/health-connect/publish/declare-access)
- [Android Health Permissions: Guidance and FAQs (Play Console Help)](https://support.google.com/googleplay/android-developer/answer/12991134)
- `doronperetz/health-native` @ `a824201` — `diabetes/app/src/main/{AndroidManifest.xml,java/com/health/diabetes/healthconnect/*}`, `workout/app/src/main/java/com/health/workout/{domain/SessionLog.kt,data/SyncSessionWorker.kt}`, `gradle/libs.versions.toml`, `.github/workflows/build-*.yml`

Not verifiable: `partnersupport.ouraring.com` ("Using the Oura API", Oura for
Organizations) is behind Cloudflare and returned 403 to both fetch attempts.
If an enterprise write capability exists it would be documented there — worth a
manual look before concluding the door is fully shut.
