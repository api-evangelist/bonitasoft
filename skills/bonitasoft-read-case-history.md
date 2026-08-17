---
name: Read Bonita case history from the archive
description: Find completed cases, tasks and variables after they have left the live resources, and join archived records back to the live ids they came from — the single most common source of unexpected 404s in the Bonita API.
api: openapi/bonitasoft-bonita-openapi.yml
operations: [searchArchivedProcessInstances, getArchivedProcessInstanceById, getContextByArchivedProcessInstanceId, searchArchivedHumanTasks, searchArchivedTasks, searchArchivedActivities, searchArchivedFlowNodes, getArchivedProcessInstanceVariables, searchLogs]
---

# Read Bonita case history from the archive

Bonita moves completed BPMN elements out of the live tables. This is by design,
and it is the reason an agent that treats Bonita like ordinary REST resources
404s constantly.

**Rule of thumb: if it finished, it is in the archive.** Nearly every runtime
resource has an archived twin.

| Live | Archived |
|---|---|
| `GET /API/bpm/case` | `GET /API/bpm/archivedCase` |
| `GET /API/bpm/humanTask` | `GET /API/bpm/archivedHumanTask` |
| `GET /API/bpm/task` | `GET /API/bpm/archivedTask` |
| `GET /API/bpm/activity` | `GET /API/bpm/archivedActivity` |
| `GET /API/bpm/flowNode` | `GET /API/bpm/archivedFlowNode` |
| `GET /API/bpm/caseVariable` | `GET /API/bpm/archivedCaseVariable` |
| `GET /API/bpm/caseDocument` | `GET /API/bpm/archivedCaseDocument` |

## Steps

1. **Try live, then archive.** On a 404 from `getProcessInstanceById`
   (`GET /API/bpm/case/{id}`), call `getArchivedProcessInstanceById` —
   `GET /API/bpm/archivedCase/{id}`. Do not report "not found" until both miss.

2. **Search the archive.** `searchArchivedProcessInstances` —
   `GET /API/bpm/archivedCase?p=0&c=20&f=<filter>&o=<order>`. Same `p`/`c`/`f`/`o`/`s`
   contract as the live search, same `content-range` header.

3. **Join back to the live id.** Archived records carry **`sourceObjectId`** —
   the id the entity had while live. That is the join key between the two halves
   of the model. Note the spelling drift: the variable schemas use
   `sourcedObjectId` instead.

4. **Reconstruct what happened.** `getContextByArchivedProcessInstanceId` —
   `GET /API/bpm/archivedCase/{id}/context` — for the business data and documents
   the case ended with. `searchArchivedHumanTasks`, `searchArchivedActivities` and
   `searchArchivedFlowNodes` for the steps it took, filtered on the case id.
   `getArchivedProcessInstanceVariables` — `GET /API/bpm/archivedCaseVariable` —
   for the final variable values.

5. **Fall back to the queriable log for the audit trail.** `searchLogs` —
   `GET /API/system/log?p=0&c=50`. This is Bonita's own event log; it records
   platform-level events, including `BUSINESS_DATA_CLEANED_UP` when a BDM
   retention rule deletes expired business data (Bonita 2026.1+).

## Rules

- **Archived records are immutable.** There is no PUT on any archived resource.
  The only write is `deleteArchivedProcessInstanceById` —
  `DELETE /API/bpm/archivedCase/{id}` — which is destructive and irreversible.
- **The archive is not forever.** BDM data retention rules (2026.1+, subscription
  editions) delete expired business data on a schedule, cascading down the
  composition tree. If a case's business data is missing but the case is present,
  a retention rule may have run. Check with the deployment's administrators.
- **Beware the spec typo when reading failures.** Archived failures are served at
  `/API/bpm/achivedFailure/...` — "achived", missing the `r`. It is spelled that
  way in the published contract, unlike every other archived resource.
- **No `d=` on some archived resources.** Verify before relying on field
  expansion in the archive; `d=` is documented in prose, not declared per
  operation.

See `data-model/bonitasoft-data-model.yml`,
`errors/bonitasoft-problem-types.yml`.
