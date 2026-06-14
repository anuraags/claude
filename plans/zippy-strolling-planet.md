# REM-359 — Confirm flow: endpoint, token consume, parked state, task reopen

## Context

Part of the Customer-Driven Roof Location Correction work (design doc:
`workflows/docs/design-docs/REM-283-roof-location-correction.md`, §8.3–8.5). When the system resolves
the **wrong location** for a property, instead of cancelling we want to recover: the customer drops a
pin on the correct property in the `property-locator` web view, and that corrected location feeds back
into roof geometry processing.

REM-357 (the base branch for both repos) already built the token foundation: the `PropertyLocatorToken`
DynamoDB table, mint/verify/consume/revoke persistence, the `locationCorrectionService`, and the
`property-locator` correction page that injects a `confirmUrl`.

**This ticket closes two of the design's open questions and builds the recovery loop's back half:**
- **OQ2 (resolved by the user):** the Roofing Service API does **not** support update-coordinates-and-reopen.
  We take the fallback path — **create a new roof geometry task at the corrected coordinates and re-link
  the Xactimate subscriptions** to it.
- **OQ1 (design leans this way):** the public confirm endpoint lives on the **workflows** server, with CORS
  scoped to the `property-locator` origin.

What we deliver: (1) a `WaitingForCustomerLocation` parked state so a task can leave active processing
until a customer corrects it; (2) the create-new-task + re-link "reopen" path; (3) the public
`POST /location-correction/confirm` endpoint that ties token-consume → new task → Slack together.

Scope confirmed with user: **build the parked-state mechanism only** (the §8.2 auto-trigger that decides
to park stays in the Phase-4 ticket); **re-link all subscriptions** to the new task.

Base branch for both repos: `REM-357`. Work branch: `REM-359` (already created in both).

---

## Workflows repo (`/Users/anu.sridhar/Development/Projects/workflows`)

### 1. Parked task state (§8.3)

- **`src/server/services/persistence-service/types.ts`** — add
  `WaitingForCustomerLocation = 'waiting-for-customer-location'` to the `TaskStatus` enum.
- **`src/server/process-task/check-task-is-in-db.ts`** — in `checkRoofGeometryTaskInDb`, when
  `taskInfo.status === TaskStatus.WaitingForCustomerLocation`, log "task parked, awaiting customer
  location" and return `{ state: 'finished-checking' }`. This is what makes a parked task **leave the
  active polling loop**: even if a roofing webhook fires, the state machine short-circuits and never
  polls/cancels it. (The processing loop in `process-task/index.ts` runs to `finished-checking`; parking
  = the run ends cleanly without cancel.)
- **New `src/server/process-task/park-task-for-location-correction.ts`** — small reusable helper
  `parkTaskForLocationCorrection(taskInfo, configuration)` that calls
  `persistenceService.trackRoofGeometryTask({ status: WaitingForCustomerLocation, creator, tags: [],
  metadata: { problem } })`. Ready for the Phase-4 trigger to call; not wired into the problem handler
  here. Unit-tested.

### 2. Create-new-task + re-link "reopen" path (§8.5)

- **New `src/server/process-task/recreate-task-at-corrected-location.ts`** —
  `recreateTaskAtCorrectedLocation(originalTaskId, confirmedCoords, configuration): Promise<string>`:
  1. `getTaskById(originalTaskId)`; read `metadata.input` as `CreateRoofingTaskInput` (same pattern as
     `create-extended-coverage-task.ts:33`). If missing, log + throw.
  2. Build a new `CreateRoofingTaskInput` from the original with `latitude/longitude` replaced by
     `confirmedCoords.lat/lng` (keep address, propertyType, dataType, instructions, externalCustomerId,
     externalOrderId, priority).
  3. `roofingService.createRoofGeometryTask(input)` → `newTaskId` (reuses the existing client method at
     `roofing-service/index.ts:65`).
  4. `trackRoofGeometryTask({ roofGeometryTaskId: newTaskId, status: Pending, creator, pricingInfo,
     tags, metadata: { input, correctedFrom: originalTaskId } })` copying original creator/pricing.
  5. Re-link **all** subscriptions: `getTaskSubscriptions(originalTaskId)` → for each,
     `subscribeToTask(newTaskId, creator.id, xactimateProjectId, claimNumber, type,
     { xactimateBody: sub.xactimateProject })` (mirrors `xactimate/callback.ts:220`).
  6. Return `newTaskId`. Caller kicks off `processTask(newTaskId, 'roof-geometry', configuration)`.
- No new roofing-service client method is needed — `createRoofGeometryTask` already exists; the existing
  `reopenTask` (PUT status=reopened) can't change coordinates, which is exactly why we take this path.

### 3. Public confirm endpoint (§8.4, OQ1 → workflows)

- **New `src/server/express/routes/location-correction/confirm.ts`** —
  `setupLocationCorrectionConfirmEndpoint(router, wrap, configuration)`:
  - zod-validate body `{ token: string, lng: number, lat: number }` → `HTTP_CLIENT_ERROR` on bad shape.
  - `locationCorrectionService.verifyPropertyLocatorToken(token)` (existing) → if `!valid`, map reason to
    status: `not-found`→404, `expired`/`consumed`/`revoked`→`HTTP_CONFLICT` (409).
  - Bbox check: `{lng,lat}` inside `record.allowedBbox` (new helper `isWithinBbox` in
    `location-correction-service/`, alongside `derive-allowed-bbox.ts`). Out of bounds →
    `HTTP_UNPROCESSABLE_ENTITY` (422).
  - `locationCorrectionService.consumePropertyLocatorToken(token, { lng, lat })` (existing, atomic
    `active → consumed` conditional write). `null` return = lost race / already consumed → 409.
  - On success: `recreateTaskAtCorrectedLocation(record.taskId, { lng, lat }, configuration)` →
    `newTaskId`, then `processTask(newTaskId, 'roof-geometry', configuration).catch(lastResortHandler)`
    (fire-and-forget, same as webhook `roof-geometry-webhook.ts:18`).
  - `slackService?.notifyLocationCorrectionConfirmed(...)` (see §4).
  - Respond `HTTP_OK` with `{ status: 'ok' }`.
- **CORS** — use the already-present `cors` package (workflows `package.json:151`). Apply a `cors({ origin:
  configuration.propertyLocatorUrl })` middleware **only** to this route (incl. `OPTIONS` preflight), so
  the `property-locator`-served page can POST cross-origin without opening CORS globally.
- **Wiring** — register on the **public** router (token is the only credential, no auth middleware):
  add `setupLocationCorrectionConfirmEndpoint(publicRouter, ...)` in
  `src/server/express/routes/index.ts:setupPublicRoutes` (alongside the webhooks). Final path:
  `/workflows/v1/location-correction/confirm`.

### 4. Slack notification

- **`src/server/services/slack-service/types.ts`** + **`index.ts`** — add
  `notifyLocationCorrectionConfirmed(originalTaskId, newTaskId, coords)` posting to the existing
  support/xactimate channel (reuse the block style in `index.ts`). Add to the `__mocks__/configuration.ts`
  `slackService` mock.

### 5. Config

- **`src/server/types.ts`** + **`configuration.ts`** — expose `propertyLocatorUrl` on `Configuration`
  (read from the existing `PROPERTY_LOCATOR_URL` env, already used at `configuration.ts:125`) so the CORS
  middleware can scope to that origin. Add to `__mocks__/configuration.ts`.

### 6. Tests (Jest + supertest, co-located `*.test.ts`)

- `confirm.test.ts` — supertest against the route (template: `roof-geometry-webhook.test.ts`), mocking
  `processTask` and `recreateTaskAtCorrectedLocation`; cover valid confirm, bad body, not-found/expired/
  consumed token, out-of-bbox, lost-race (`consume` returns null), CORS preflight.
- `recreate-task-at-corrected-location.test.ts` — new task created with swapped coords, all subscriptions
  re-linked, missing-metadata throws.
- `park-task-for-location-correction.test.ts` — sets `WaitingForCustomerLocation`.
- `check-task-is-in-db.test.ts` — parked task short-circuits to `finished-checking` (extend if present).

---

## property-locator repo (`/Users/anu.sridhar/Development/Projects/property-locator`)

The confirm endpoint lives on workflows (cross-origin), so the only change is pointing the injected
`confirmUrl` at the workflows endpoint instead of the same-origin placeholder. (The client-side confirm
POST itself is REM-382 — not this ticket.)

- **`src/server/express/routes/correct-location/view-config.ts:9-10`** — replace the hardcoded same-origin
  `CONFIRM_URL` placeholder with the workflows confirm URL sourced from a new env var
  (`WORKFLOWS_CONFIRM_URL`, e.g. `https://<workflows-host>/workflows/v1/location-correction/confirm`),
  threaded through `buildViewConfig` from `configuration.ts` (add to `Configuration` + mock) rather than a
  module constant. Update the `view-config` test.

---

## Verification

1. **Workflows unit/integration tests** — `cd workflows && npm test` (Jest; run the new/affected suites).
   The repo has a coverage gate, so the new modules need full-branch tests.
2. **Type + lint** — `npm run build` / `tsc` and `prettier`/`eslint` as the repo configures.
3. **property-locator** — `cd property-locator && npm test` (100% coverage gate); confirm `view-config`
   test passes with the new env-driven `confirmUrl`.
4. **End-to-end (manual, local)** — mint a token (REM-357 path), open the correction page, POST
   `{ token, lng, lat }` to `/workflows/v1/location-correction/confirm`: verify the token row flips to
   `consumed` with `confirmedCoords` set, a new roofing task is created at the corrected coords with the
   subscriptions re-linked, the original task sits in `WaitingForCustomerLocation`, and a Slack message
   fires. Re-POST the same token → 409.

## Out of scope (later tickets)
- §8.2 auto-trigger that decides to park a task (Phase 4).
- Client-side pin/confirm POST UI in property-locator (REM-382).
- Email delivery, expiry sweep/reminders (Phase 4).
