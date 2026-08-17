---
name: Diagnose and recover a failed Bonita process
description: Find out why a case is stuck, read the underlying BPM and connector failures, and replay the flow node — the operational skill that keeps a Bonita deployment running.
api: openapi/bonitasoft-bonita-openapi.yml
operations: [getProcessInstanceInfoById, getBPMFailuresByCaseId, getBPMFailuresByRootCaseId, getBPMFailuresByFlowNodeInstanceId, getArchivedBPMFailuresByCaseId, searchFlowNodes, getFlowNodeById, updateFlowNodeById, searchConnectorInstances, getConnectorFailureById, searchTimerEventTriggers, updateTimerEventTriggerById, searchLogs]
---

# Diagnose and recover a failed Bonita process

When a Bonita case stops moving, the answer is almost never in the HTTP error —
it is in the engine's failure records. Bonita models failure as first-class data,
which is one of the genuinely strong parts of this API.

Authenticate as an administrator. Writes need the `X-Bonita-API-Token` header.

## Steps

1. **Locate the stuck element.** `getProcessInstanceInfoById` —
   `GET /API/bpm/caseInfo/{id}`. Returns `flowNodeStatesCounters`: how many flow
   node instances sit in each state (ready, executing, waiting, initializing,
   **failed**, completing, completed, skipped, cancelled, aborted). A non-zero
   `failed` counter is your signal.

2. **Read the failures for the case.** `getBPMFailuresByCaseId` —
   `GET /API/bpm/failure/case/{caseId}`. For a parent case whose children may have
   failed, `getBPMFailuresByRootCaseId` —
   `GET /API/bpm/failure/case/{rootCaseId}/childCases`. Each `BPMFailure` carries
   `caseId`, `rootCaseId`, `flowNodeInstanceId` and `processDefinitionId`.

3. **Narrow to the flow node.** `getBPMFailuresByFlowNodeInstanceId` —
   `GET /API/bpm/failure/flowNode/{flowNodeInstanceId}` — then `getFlowNodeById` —
   `GET /API/bpm/flowNode/{id}?d=processId&d=caseId&d=assigned_id` — to see the
   node in context.

4. **If a connector failed, read the connector failure.**
   `searchConnectorInstances` —
   `GET /API/bpm/connectorInstance?p=0&c=20&f=containerId%3D<flowNodeOrCaseId>` —
   then `getConnectorFailureById` — `GET /API/bpm/connectorFailure/{id}` — for the
   stack trace. Connectors are where external systems break a process, so this is
   the most common root cause.
   Note that `containerId` is polymorphic: a connector instance may hang off a
   flow node **or** a process instance, and the spec does not distinguish them by
   field.

5. **Replay the flow node.** `updateFlowNodeById` — `PUT /API/bpm/flowNode/{id}`
   — with the state change that retries or skips the node. Fix the cause first:
   replaying a connector that will fail again just re-fails it.

6. **If the case is waiting on a timer, inspect it.**
   `searchTimerEventTriggers` — `GET /API/bpm/timerEventTrigger?p=0&c=20` — and
   `updateTimerEventTriggerById` — `PUT /API/bpm/timerEventTrigger/{id}` — to
   change a trigger's execution date. This is also the nearest thing Bonita has to
   a test clock, but it is a **production scheduler operation**, not a simulation.

7. **If the case has already been archived**, use
   `getArchivedBPMFailuresByCaseId` —
   `GET /API/bpm/achivedFailure/case/{caseId}` — and
   `getArchivedBPMFailuresByRootCaseId` —
   `GET /API/bpm/achivedFailure/case/{rootCaseId}/childCases`.
   **Note the spelling: `achivedFailure`, missing the `r`.** It is spelled that way
   in the published contract.

8. **Correlate with the platform log.** `searchLogs` —
   `GET /API/system/log?p=0&c=50` — and `getLogById` for detail.

## Rules

- **Do not confuse an HTTP error with a process failure.** A 500 from the API is a
  request-level fault; a `BPMFailure` is a durable engine record of a process step
  that could not execute. They need different responses.
- **`5XX` is declared on all 224 operations** and its body is only
  `{ "message": "..." }`, frequently a wrapped Java exception. Never parse it as a
  contract — use the failure resources instead.
- **Replay is not idempotent and there is no dry run.** There is no
  `Idempotency-Key` and no simulate mode. Re-issuing a replay can double-execute a
  side effect the connector already performed. Check the connector's target system
  before replaying a second time.
- **`p` and `c` are required on every search.** Omitting them returns 400.
- **Nothing pushes these failures to you.** There is no webhook, no AsyncAPI and
  no event delivery — you must poll `caseInfo` or the failure resources. Budget for
  that in any monitoring you build.

See `errors/bonitasoft-problem-types.yml`,
`conventions/bonitasoft-conventions.yml`,
`data-model/bonitasoft-data-model.yml`.
