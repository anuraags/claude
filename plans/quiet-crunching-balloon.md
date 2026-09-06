# REM-657: Reopen a REM Task workflow

## Context

[REM-657](https://nearmap.atlassian.net/browse/REM-657): implement a reopen path in
`rem-task-workflows`. Today `remTaskWorkflow` always geocodes an address, checks Nearmap 3D
coverage, then calls geom's `create` endpoint. Reopening an already-completed geom task needs a
second entry point: skip geocoding/coverage, and call geom's reopen endpoint directly with an
existing `taskId`. Everything downstream (wait-for-completion, geometry upload, ESX/PDF
generation, webhook notifications, cancel-while-processing) should behave identically regardless
of how the task got started.

**Endpoint identification (confirmed, not `reopen` by name):** geom's `TaskEndpoint` has no
literal `reopen` REST method. `TaskEndpoint.reject` (`@ApiMethod(path = "reject")`,
`src/main/java/us/eave/backend/TaskEndpoint.java:1478`) is what actually reopens a completed
task — `rejectHelper` sets `task.endDate = null` (makes the task "incomplete" again) and resets
the report's status to `StatusNew`; a comment there literally says "Reset wwdMode so a reopened
Roof+WWD task starts at roof review." So the client method backing this feature calls
`POST /api/task/v1/reject`, matching the existing `.../cancel` mapping precedent in
`packages/rem-geometry-client/src/gcp-rem-geometry-client/index.ts`.

**Constraint carried over from geom:** `TaskEndpoint.reject` throws if `instructions` is null/empty
and separately enforces `MaxLengthValidator`. Per your answer, `instructions` becomes a required
(non-optional) field on `WorkflowInput` for **both** branches (create and reopen) — this is scoped
to the workflow's own input schema in `packages/rem-task-workflows/src/workflows/types.ts`; the
shared `createTaskInputSchema` in `@nearmap/rem-geometry-client` (used directly by other
consumers of the client) is left with `instructions` optional, since that's a lower-level API
contract this ticket isn't asked to change.

**Input shape:** per your answer, `WorkflowInput` becomes a `zod.discriminatedUnion('type', …)`
with an explicit `type: 'create' | 'reopen'` literal.

**Out of scope:** `TaskStatus.reopenEligible` (`workflows/types.ts`) stays `false` with its
existing TODO — that field is about signalling a *running* workflow to reopen via the Temporal
UI (the follow-up to REM-656's cancel signal), not this taskId-input path. Not touching it.

**Completion-signal note (verified, not a blocker):** the workflow waits on
`condition(() => remTaskCompleted || cancelRequested)` before calling `handleCompletedTask`.
`remTaskCompletedSignal` is only sent, in this repo, by a manual CLI tool
(`tools/utils/complete-rem-task.ts`) which looks up workflows via
`ExternalTaskId = "<taskId>"` search attribute and signals every matching execution. In
production this must be wired externally to geom's listener webhook — not present in this repo
and out of scope to change. Since the reopen branch reuses the exact same
`upsertSearchAttributes([{ ExternalTaskId, taskId }])` + `condition(...)` pattern with the same
geom `taskId`, whatever already signals completion for create-flow workflows will equally reach
a reopen-flow workflow (the old, now-closed execution also matches the search query, but
signalling a closed workflow just fails — already handled by the existing try/catch in
`complete-rem-task.ts`). No new risk introduced here; flagging so it's a known assumption, not a
silent one.

## Changes

### 1. `@nearmap/rem-geometry-client` — add `reopenGeometryTask`

- **`src/types.ts`**: add to `REMGeometryClient`:
  ```ts
  reopenGeometryTask(taskId: string, instructions: string, urls?: string[]): Promise<void>;
  ```
- **`src/gcp-rem-geometry-client/server-side-types.ts`**: add
  `export interface TaskRejectResponse { success: boolean }` (mirrors `TaskCancelResponse`).
- **`src/gcp-rem-geometry-client/index.ts`**: implement using the existing `postTaskRequest`
  helper (so the `There is no task with identifier X.` 503→404 normalization via
  `normalizeTaskNotFoundError` applies here too, same as `cancelGeometryTask`/`fetchTaskGeometry`):
  ```ts
  reopenGeometryTask: async (taskId, instructions, urls): Promise<void> => {
    const params = url.serializeSearchParams({ instructions, taskId, url: urls });
    const response = await postTaskRequest<TaskRejectResponse>(`/api/task/v1/reject?${params}`, taskId);
    if (!response.data.success) {
      throw new Error(`Reopen request for task ${taskId} was not successful`);
    }
  },
  ```
  No special-cased "already such" swallowing like `cancelGeometryTask`'s
  `isTaskAlreadyCompletedError` — unlike cancel, there's no expected race here; geom's
  "Cannot reject an incomplete/canceled task" failures are genuine caller errors and should
  surface as errors, same as any other activity failure (handled generically by
  `handleWorkflowError` in `main.ts`, same as today's `createTask` errors).
- **`src/dummy.ts`**: add a `reopenGeometryTask` stub (log + resolve), following the existing
  console.log pattern for the other dummy methods.
- Tests: `src/gcp-rem-geometry-client/index.test.ts` (green path, not-found→404 normalization,
  not-successful throws) and `src/dummy.test.ts`, following this repo's Jest conventions
  (`beforeEach` mock setup, green path first, conditions in `describe`).

### 2. `rem-task-workflows` activities — add `reopenTask`

- **`src/activities/types.ts`**: add to `Activities`:
  ```ts
  reopenTask: (input: { taskId: string; instructions: string; urls?: string[] }) => Promise<void>;
  ```
- **`src/activities/reopen-task.ts`** (+ `.test.ts`): mirror `src/activities/cancel-task.ts`:
  ```ts
  export const reopenTask: Activities['reopenTask'] = async ({ taskId, instructions, urls }) =>
    ServerContext.instance.remGeometryClient.reopenGeometryTask(taskId, instructions, urls);
  ```
- **`src/activities/index.ts`**: register `reopenTask` in `createActivities`.
- Update every test file that type-checks a full `mockRemGeometryClient: REMGeometryClient`
  object to add `reopenGeometryTask: jest.fn()` (compile-only requirement, no behavior change):
  `cancel-task.test.ts`, `create-task.test.ts`, `upload-task-geometry.test.ts`,
  `generate-and-upload-esx.test.ts`, `wait-for-task-completion.test.ts`.

### 3. `workflows/types.ts` — discriminated `WorkflowInput`

```ts
const workflowCommonFields = {
  instructions: zod.string(),
  callbackUrl: zod.url({ protocol: /^https?$/ }).optional(),
  skipCleanup: zod.boolean().optional(),
};

const createWorkflowInputSchema = createTaskInputSchema.extend({
  ...workflowCommonFields,
  type: zod.literal('create'),
  country: zod.enum(['US', 'AU']),
  coordinates: createTaskInputSchema.shape.coordinates.optional(),
});

const reopenWorkflowInputSchema = zod.object({
  ...workflowCommonFields,
  type: zod.literal('reopen'),
  taskId: zod.string(),
  urls: zod.array(zod.string()).optional(),
  dataType: dataTypeSchema, // re-exported from @nearmap/rem-geometry-client, same as today
});

export const workflowInputSchema = zod.discriminatedUnion('type', [
  createWorkflowInputSchema,
  reopenWorkflowInputSchema,
]);
export type WorkflowInput = zod.infer<typeof workflowInputSchema>;
```

Reopen mode intentionally omits `address`/`coordinates`/`propertyType`/`priority`/`customerId`/
`orderId` — none of them are consumed once geocoding, coverage-check, and `createTask` are
skipped. `dataType` stays required in both: it still drives `handleCompletedTask` /
`generateAndUploadESX`'s `type` param and isn't recoverable from geom's task-status response.

`workflowOutputSchema`/`WorkflowOutput`/`TaskStatus` are unchanged.

### 4. `workflows/main.ts` — branch on `input.type`

Extract the "how do we get a running geom task" step out of `runRemTaskWorkflow` into a small
helper so the two branches share everything after that point unchanged (webhook-submitted
notification, the `condition(remTaskCompleted || cancelRequested)` wait, `cancelIfRequested`,
`handleCompletedTask`, output construction, `handleWorkflowError`):

```ts
const { reopenTask } = proxyActivities<Activities>({ startToCloseTimeout: '1 minute' }); // alongside createTask etc.

type EstablishedTask =
  | { kind: 'started'; taskId: string; dataType: WorkflowInput['dataType']; coordinates?: { latitude: number; longitude: number } }
  | { kind: 'canceled'; output: WorkflowCanceledOutput };

const establishTask = async (input: WorkflowInput): Promise<EstablishedTask> => {
  if (input.type === 'reopen') {
    await reopenTask({ taskId: input.taskId, instructions: input.instructions, urls: input.urls });
    return { kind: 'started', taskId: input.taskId, dataType: input.dataType };
  }

  const coordinates = await establishCoordinates(input);
  if (!coordinates) {
    return { kind: 'canceled', output: { state: 'canceled', errorMessage: 'failed to find coordinates for address', problem: 'address not found' } };
  }

  const coverageDecision = getCoverageDecision({ dataType: input.dataType, ...(await determineCoverage(coordinates)) });
  if (coverageDecision === 'extended') {
    return { kind: 'canceled', output: { state: 'canceled', errorMessage: 'address is not covered by Nearmap 3D', problem: 'not covered' } };
  }
  if (coverageDecision !== 'nearmap') {
    return { kind: 'canceled', output: coverageDecision };
  }

  const { callbackUrl, skipCleanup, type, ...taskInput } = input;
  const remTaskId = await createTask({ ...taskInput, coordinates });
  return { kind: 'started', taskId: remTaskId, dataType: input.dataType, coordinates };
};
```

`runRemTaskWorkflow` calls `establishTask(input)` once, sets `status.latitude`/`status.longitude`
when `coordinates` is present, returns early via `cancelAndReturn(output)` on `kind === 'canceled'`,
and otherwise continues with the existing tail using the returned `taskId`/`dataType` in place of
`remTaskId`/`input.dataType`. No behavior change for the existing create path — this is a
same-behavior refactor plus the new branch.

`testCoverage` (exported helper) and `EstablishCoordinatesInput` are unaffected — coverage-testing
is inherently create-mode-only. `main.ts` keeps its `/* istanbul ignore file */` per ADR-0009.

### 5. Fix up existing callers/tests broken by the new required `type` field

- `packages/rem-task-workflows/tools/utils/start-rem-workflow.ts`: add `type: 'create'` to
  `buildDummyInput`'s returned object.
- `packages/rem-task-workflows/src/workflows/types.test.ts`: update existing valid/invalid input
  fixtures for the new discriminated shape; add cases for the reopen variant.
- `packages/rem-task-workflows/src/integration-tests/rem-task-workflow.test.ts`: add `type: 'create'`
  to existing input fixtures; add new test case(s) covering the reopen path end-to-end (reopen →
  same completion/upload/webhook/cancel behavior as create).

## Verification

- `pnpm --filter @nearmap/rem-geometry-client exec jest --coverage=false`
- `pnpm --filter @nearmap/rem-task-workflows exec jest --coverage=false`
- `pnpm nx lint rem-geometry-client && pnpm nx lint rem-task-workflows` (catches any mock/type
  ripple from the new interface method and the new discriminant field)
- Full `pnpm --filter @nearmap/rem-geometry-client test` and
  `pnpm --filter @nearmap/rem-task-workflows test` for the 100%-coverage gate before opening a PR.
