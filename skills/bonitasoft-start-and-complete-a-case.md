---
name: Start a Bonita case and drive it to completion
description: Find a deployed process, read its instantiation contract, start a case with the right payload, then find, assign and execute the human tasks that follow — the core BPM flow of the Bonita API.
api: openapi/bonitasoft-bonita-openapi.yml
operations: [searchProcesses, getProcessContractById, instanciateProcess, createProcessInstance, getProcessInstanceById, searchHumanTasks, updateHumanTaskById, getContractByUserTaskId, executeUserTask, getProcessInstanceInfoById]
---

# Start a Bonita case and drive it to completion

This is the flow Bonita exists for: instantiate a deployed BPMN process, then
work the human tasks it generates.

Authenticate first — see `skills/bonitasoft-authenticate-and-call.md`. Every
write below needs the `X-Bonita-API-Token` header.

## Steps

1. **Find the process.** `searchProcesses` — `GET /API/bpm/process?p=0&c=10&f=name%3D<processName>`.
   You need the process **definition id**. If you only know the display name and
   want the versions available, `searchProcessNames` —
   `GET /API/bpm/processName` — returns one entry per distinct name with its
   deployed versions (added in Bonita 2026.2).

2. **Read the instantiation contract before building a payload.**
   `getProcessContractById` — `GET /API/bpm/process/{id}/contract`. This returns
   the `Contract`: its `inputs` (recursive — a complex input contains further
   inputs) and its `constraints`. **Do not guess the body.** The contract is
   per-process and defined at design time; it is the only description of what
   `instanciateProcess` will accept.

3. **Start the case.** `instanciateProcess` —
   `POST /API/bpm/process/{id}/instantiation` with a JSON body matching the
   contract. Returns a `ProcessInstantiationResponse` carrying `caseId`.
   - The alternative is `createProcessInstance` — `POST /API/bpm/case` with
     `processDefinitionId` and optional `variables`. Use `instanciateProcess`
     when the process has a contract; use `createProcessInstance` for the simpler
     variable-based start.

4. **Read the case.** `getProcessInstanceById` — `GET /API/bpm/case/{id}`. For
   the flow-node state rollup (how many instances are ready, executing, waiting,
   failed, and so on) use `getProcessInstanceInfoById` —
   `GET /API/bpm/caseInfo/{id}`. That is the cheapest way to answer "where is this
   case now".

5. **Find the pending human task.** `searchHumanTasks` —
   `GET /API/bpm/humanTask?p=0&c=10&f=caseId%3D<caseId>`. Add
   `d=processId&d=assigned_id` to inline the process and the assignee instead of
   getting bare ids back.

6. **Assign it.** `updateHumanTaskById` — `PUT /API/bpm/humanTask/{id}` with
   `assigned_id` set to the user id who will act. A task that is *pending* for an
   actor is not yet assigned; most flows assign before executing.

7. **Read the task contract, then execute.** `getContractByUserTaskId` —
   `GET /API/bpm/userTask/{id}/contract` — then `executeUserTask` —
   `POST /API/bpm/userTask/{id}/execution` with the matching body. As with step 2,
   the contract is the payload specification.
   - `getContextByUserTaskId` — `GET /API/bpm/userTask/{id}/context` — returns the
     business data and documents bound to that task, which is what a form would
     render.

8. **Repeat 5–7** until the case completes. A completed case leaves the live
   resource — see `skills/bonitasoft-read-case-history.md`.

## Rules

- **`p` and `c` are required on every search.** Omitting them returns 400.
- **There is no idempotency.** Retrying step 3 starts a **second case**. If the
  first request timed out, search for the case (step 5, filtered on a business key
  you passed in the contract) before retrying. Bonita has no
  `Idempotency-Key` header, so deduplication is entirely your responsibility.
- **Watch for 429 on Community Edition.** Community 2024.3+ caps case creation
  (150 per month). `Retry-After` on that 429 is an **absolute date-time**, not
  delay-seconds — back off until that date, do not retry in seconds.
- **A 404 on a case you just saw usually means it completed**, not that it never
  existed. Check the archive.
- Use `createProcessInstanceComment` — `POST /API/bpm/comment` — to leave an audit
  trail on the case; use `updateVariableByProcessInstanceId` —
  `PUT /API/bpm/caseVariable/{id}/{variableName}` — to change a process variable
  mid-flight.

See `conventions/bonitasoft-conventions.yml`,
`rate-limits/bonitasoft-rate-limits.yml`, `data-model/bonitasoft-data-model.yml`.
