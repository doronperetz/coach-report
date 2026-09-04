# Posting updates into the Oura ecosystem

Research note: how (and whether) an outside system can push **exercise** and
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

## 5. Recommendation

**Do the crossing yourself; don't fight Oura for it.**

Oura's Meals feature exists to cross food against its sensor data. But the API
gives you the sensor half — `daily_activity`, `daily_sleep`, `daily_readiness`,
`daily_stress`, `daily_spo2`, `workout`, `session`, `heartrate`,
`daily_cardiovascular_age`, `vO2_max` — and you already own the nutrition and
training half. Pulling Oura's data down and joining it in your own pipeline is
strictly more capable than pushing food up: you get real macros instead of
low/moderate/high bands, arbitrary history instead of same-day, and the result
is readable (Oura's Meals output is not exposed anywhere).

Concretely:

1. **Sensor data in** — Oura read API. Register a webhook subscription
   (`POST /v2/webhook/subscription`) so you're pushed on change rather than
   polling.
2. **Workouts out to Oura** — only if you specifically want Oura's Activity
   Score to reflect training it can't detect. Native app → health store if one
   exists; otherwise the Strava upload bridge, same-day, with an HR stream.
3. **Meals** — keep them yours. Offer a copy/share affordance into Oura Meals
   for members who want Oura's glucose-vs-meal view (only meaningful if they
   have a Stelo connected).
4. **Watch for change.** Stelo and Health Panels show Oura builds inbound pipes
   for partners it chooses. First-class meal ingestion would come through a
   partnership conversation, not a public endpoint.

---

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

Not verifiable: `partnersupport.ouraring.com` ("Using the Oura API", Oura for
Organizations) is behind Cloudflare and returned 403 to both fetch attempts.
If an enterprise write capability exists it would be documented there — worth a
manual look before concluding the door is fully shut.
