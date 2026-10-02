---
name: debug-flights
description: Diagnose why a Blackbird Flight failed, behaved unexpectedly, or produced the wrong result by inspecting Flight history, logs, output, and the Bird definition before applying a safe fix.
---

# Debug Blackbird Flights

Use this skill when a user asks why a Flight failed, what an error means, why a Bird behaved unexpectedly, or why its output is wrong. A **Bird** is the reusable automation; a **Flight** is one execution of that Bird. Explain findings in the user's workflow language, naming the relevant step by its human-readable label rather than leading with API field names.

## Investigate from evidence outward

Start with the smallest scope that identifies the affected Flight.

1. If the user supplies a Blackbird Flight URL such as `https://dev.blackbird.io/n/1/flights/61025ecb-ea50-4a8e-b6d0-b4d000bc8ab7`, parse the Nest ID (`1`) and Flight ID from it, then call `get_flight`. This is the essential first step when the URL does not contain a Bird ID: its result provides the `birdId`, full Flight error detail, and ordered execution history. Each history entry includes the executed step's inputs and outputs when recorded, so use it as the primary evidence for what each step actually received and produced. Likewise, use `get_flight` whenever the user knows the Nest and Flight ID but not the Bird ID.
2. If the user provides a Nest, Bird, and Flight ID, use them directly. If they know the Bird but not the Flight, call `list_bird_flights` with the Bird ID and, when applicable, `status: "failed"`. If they only know the Nest or need to locate a Flight across Birds, call `search_flights`, using its name, status, date, and pagination filters to narrow the result; it returns each matching `birdId`.
3. Use the Flight's execution history to identify the last completed step, failing step, actual inputs, and captured outputs. For looped work, inspect `loopStepId` and `loopIteration` so evidence is attributed to the correct iteration. A missing input or output can mean it was not recorded (for example, a trigger or a step that failed before producing output), not that a value was necessarily absent at design time.
4. Call `get_flight_logs` when the execution history does not explain the outcome or when retry, skip, cancellation, propagation, error type, or timing needs confirmation. Request the maximum useful page size (up to 100), read in execution order, and inspect later pages when the result indicates more records.
5. Call `get_flight_output` only when the final completed-step output itself needs separate confirmation. It is a convenience view of the last completed step, not a substitute for the per-step outputs returned by `get_flight`.
6. Correlate Flight step IDs with the Bird only when the Flight data is insufficient or the behavior involves routing, bindings, looping, a checkpoint, or configuration. Call `get_bird_definition` first, then `get_bird_schema` to understand the configured step inputs and outputs. The definition is editable; the schema is a read-only authoring reference.
7. If a specific connector input/output remains ambiguous, inspect its app action or event schema before inferring a field name, type, or output path.
8. When a webhook- or polling-triggered Bird has repeated failures or has become inactive, also inspect `get_bird_logs`. Blackbird automatically deactivates such a Bird after five consecutive failed Flights, so resolve the structural cause before proposing reactivation.

Never claim a root cause that the logs and configuration do not support. Separate a confirmed cause, a likely cause, and what additional evidence is needed.

## Read the symptoms correctly

| Symptom                                            | What to verify                                                                                                                                                                   |
| -------------------------------------------------- | -------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| A Flight failed at an action                       | The action history entry's error, actual inputs, selected connection, required input bindings, and preceding step output. Distinguish an app/API rejection from a missing or mismatched Bird input. |
| A Flight took the unexpected decision branch       | The decision history entry and the inputs/outputs immediately before it, then the branch comparison operator, branch order, and fallback branch.                                |
| A loop skipped or mishandled items                 | Whether `forEach` received an array, the `loopItem` binding points to the right loop, body steps actually belong in the loop, and `skipLoopIteration` did not suppress the work. |
| A Bird did not start                               | Trigger type and configuration, event connection/subscription state, polling interval, schedule/timezone, and whether a checkpoint is intentionally waiting.                     |
| A Flight stopped after a checkpoint                | Whether it is waiting for a delay or the exact expected event/object, whether the event connection is valid, and whether a link-mode webhook URL was registered.                 |
| Output is wrong despite success                    | The relevant step's captured output and inputs, input bindings that built them, any composable/object/loop-item paths, and any later branch or action that transformed them.       |
| Repeated, duplicated, or unexpectedly skipped work | Trigger cadence/bucketing, retry and skip settings, loop boundaries, and whether side-effecting actions are idempotent.                                                          |

An event trigger that returns several items can create one Flight per item. In contrast, a checkpoint can return those items together as a list. Use that distinction when diagnosing apparent duplicates or list-versus-item failures.

## Diagnose error handling before changing it

Classify the error before suggesting retry or skip behavior:

| Error category       | Interpretation                                                                       | Appropriate response                                                                                                             |
| -------------------- | ------------------------------------------------------------------------------------ | -------------------------------------------------------------------------------------------------------------------------------- |
| Configuration error  | A value, file, permission, credential, or environment setting needs correction.      | Correct the configured cause. Retrying unchanged input will normally fail again.                                                 |
| App not responding   | The external service may have a temporary outage, overload, or rate-limit condition. | Check the app's status/error detail and consider a bounded retry only if repeating the action is safe.                           |
| Unexpected App issue | The connector encountered an unhandled issue.                                        | Preserve the log evidence and report it to the app developer or Blackbird support; do not mask it with speculative Bird changes. |

For each affected action, inspect its `onError` settings and reason about its external effect:

- A retry re-executes the action up to the configured count at the configured interval. It is useful for a demonstrated transient failure, not as a default for every action.
- Never add or increase retries for a non-idempotent action without user direction. An error can occur after an external project, message, file, or record was created, so retrying may duplicate it.
- `skipStep` is appropriate only for non-essential work such as a notification or status update. When an action can be skipped, its output cannot be used later in the Bird; diagnose downstream missing-data failures with that constraint in mind.
- `skipLoopIteration` is appropriate only when every item can fail independently and omitting an item is acceptable. It lets later iterations continue; it does not repair the failed item.

If a Bird was auto-deactivated after repeated event failures, do not frame reactivation as the fix. First correct the trigger, connection, action configuration, or external issue that caused the failures.

## Report a useful diagnosis

Give the user a concise incident narrative:

1. **What happened:** Bird and Flight, status, relevant time, and the step where behavior diverged.
2. **Why:** quote or accurately paraphrase the decisive log/output evidence, then relate it to the configured action, binding, route, loop, or checkpoint.
3. **Impact:** state whether any earlier side effects likely completed and whether retrying could duplicate them.
4. **Resolution:** say whether a safe targeted fix was made, or propose the smallest change that would address the evidence.

Avoid showing secrets, access tokens, complete webhook URLs, or personally sensitive payload values in the explanation. Redact them while preserving enough context to identify the failing field or system.

## Apply a fix at the right confidence level

When the evidence shows a narrow, low-risk configuration error, make the correction. Examples include correcting a confirmed input/output path, reconnecting a valid existing connection reference, fixing a clearly incorrect `next` target, or restoring a required fallback decision branch.

Before changing a Bird, read [Build and manage Blackbird Birds](../build-birds/SKILL.md) and follow its safe update procedure:

- Read the complete definition with `get_bird_definition`.
- Preserve stable IDs and every unrelated trigger, connection, input, route, and error setting.
- Update the entire editable definition with `update_bird_definition`; never send the read-only Bird schema as the update.
- Re-read the definition afterward. Register any returned webhook payload URLs with the external app because updates can rotate link-mode URLs.

Do not automatically make a wider or higher-risk change. Explain and ask for direction before changing a trigger, connection credentials, retry/skip policy, loop boundaries, an operator contract, event subscription, or any action that could repeat, omit, or mutate external data. For those cases, offer a rigorous change plan based on the user’s intended workflow.

After a targeted fix, only run a manual Bird when the user has authorized the resulting external side effects. Otherwise, state how they can validate it and what logs/output should confirm success.

## Completion checks

Finish only after you have either:

- identified a supported cause and communicated it clearly;
- applied and verified a narrowly scoped fix; or
- explained exactly what remains unknown, which tool/data would resolve it, and the proposed next change requiring user direction.

Do not mark an intermittent or external-service failure as permanently fixed solely because one later Flight succeeded.
