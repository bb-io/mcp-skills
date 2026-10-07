---
name: build-birds
description: Create, inspect, update, or troubleshoot Blackbird Birds when a user needs an automated workflow with triggers, connector actions, decisions, loops, checkpoints, or data bindings.
---

# Build and manage Blackbird Birds

Use this skill when a user wants to build a Blackbird automation (a **Bird**), change an existing one, or understand its workflow. Speak in the user's workflow language - such as "start when a project is created", "for each file", or "wait for approval" - rather than exposing camel-cased API field names. Use API names only while making the tool call or when a technical explanation is useful.

A Bird belongs to a **Nest** (the user's workspace) and produces a **Flight** each time it runs. A Bird consists of one trigger and a directed set of steps. Prefer a manual trigger while building and testing, then move to the requested scheduled or event trigger once the workflow is proven.

## Choose the right building blocks

| User intent                               | Blackbird building block | Important behavior                                                                                                                 |
| ----------------------------------------- | ------------------------ | ---------------------------------------------------------------------------------------------------------------------------------- |
| Start on demand or test a workflow        | Manual trigger           | A user starts a Flight with the Fly button. Good for one-off work, debugging, and safe initial testing.                            |
| Run on a regular cadence                  | Scheduled trigger        | Use an interval, daily, weekly, or monthly schedule and a timezone. Consider overlap protection if a run can outlive its schedule. |
| React to an app change                    | Event trigger            | Select a specific app event and connection. Webhook events may return a URL that must be registered in the external app.           |
| Find changes where an app has no webhooks | Polling trigger          | Blackbird checks for changes at the configured interval.                                                                           |
| Collect related events before one run     | Bucketed event trigger   | Starts when the requested event count or maximum wait time is reached, whichever comes first.                                      |
| Do something in an app                    | Action                   | Choose the exact action and valid inputs; select a connection in the same Nest when that app requires one.                         |
| Continue down different paths             | Decision                 | Give each condition its own branch and include exactly one fallback branch.                                                        |
| Repeat work for every item in a list      | Loop                     | Put only per-item steps inside; steps that should happen once go after the loop.                                                   |
| Wait in the middle of a workflow          | Checkpoint               | Either delay for a duration or wait for a specific event, normally narrowed to the object created earlier.                         |
| Transform data or control execution       | Operator                 | Use only a documented/observed operator definition; do not guess operation names or parameters.                                    |

## Discover before composing

1. Identify the target Nest with `list_accessible_resources` and confirm the Bird's purpose, trigger, source app, destination app, and desired error behavior.
2. Use `search_apps`, then `list_app_actions` and `get_app_action_schema` for every action in the Bird. For a trigger or event checkpoint, use `list_app_events` and `get_app_event_schema`. In the Bird definition, copy the selected element's `externalFullName` into `action.name` or `event.name`; its short `name` and `displayName` are human-readable labels, not resolution keys. Never infer input names, required fields, trigger outputs, or binding types from an action/event name, a previous Bird, or general product knowledge. The schemas are the sole source of truth.
3. Inspect every action/event input's `dataSource`. A non-null `dataSource` whose `type` is `0` (`KeyValue`) is a dynamic input, for example `{"type": 0, "declaringTypeFullName": "Apps.Slack.DataSourceHandlers.ChannelUserHandler"}`. For each such input, call `get_app_action_input_options` or `get_app_event_input_options` before filling it. These tools do not support file/folder data sources. They return the actual current values accepted by the connected app and are essential for correctly authoring `dynamic` bindings. Pass the selected connection; for context-dependent choices, also pass the raw values already selected for sibling inputs (not Bird binding objects). Do not guess identifiers or labels from names, prior Birds, or general product knowledge. Call the tool again after changing an upstream sibling value, because its available options can change.
4. Before adding an operator, call `get_bird_operator_catalog`. It is the source of truth for supported operator names, exact input keys, required flags, outputs, examples, and flow-control constraints.
5. Use `list_app_connections` to select an existing connected app account in the target Nest. If none is suitable, call `get_app_connection_schema` before `create_app_connection`. Only token/key connection types can be created through MCP; OAuth connection types must be connected through the appropriate interactive flow.
6. For an existing Bird, always call `get_bird_definition` before editing. Call `get_bird_schema` as a human-readable reference for what its trigger and steps expose, but never submit that read-only schema as an update.
7. Plan the data flow before writing: which step produces each value, whether it is one value or a list, and which later step consumes it. Give steps and loops meaningful names so the workflow remains maintainable.

When composing a definition, read [JSON definition patterns](references/definition-patterns.md). They show the editable shape for common trigger and step types; replace every placeholder with names, IDs, inputs, and outputs confirmed by the discovery tools. Immediately before `create_bird` or `update_bird_definition`, call `preflight_bird_definition` with this exact definition and discovery references for every action and event. Do not write while it reports `valid: false`.

## Use the Utilities app for data preparation

The **Blackbird Utilities** app is always available in every Nest and does **not** require a connection. Treat it as a useful first option when a Bird needs to prepare or inspect data between app actions: date arithmetic and formatting, text or file-name processing, regular-expression extraction or replacement, number ranges for loops, array membership or de-duplication, XML values, file/text conversion, and web-page content extraction are examples, not a complete catalog.

Utilities are connectionless and appear in `search_apps` results like other apps. Search by name, then use their normal action/event schema tools. Omit `connection` when composing a utility action or event checkpoint. Prefer a Utilities action over adding avoidable external-system calls or manually emulating a common transformation. Use the operator catalog for built-in workflow operators; Utilities actions are app actions and remain schema-driven.

## Pass textual files between apps directly

Blackbird files are interoperable. When an action produces textual content, pass that file directly to the next action that needs a file; do not add conversion steps merely to reconcile file formats. Blackbird natively handles XLIFF 1.2, XLIFF 2.x, HTML, JSON, PO, XML, DITA, plain text, DOCX, and PPTX.

Some actions, including translation and Prepare content, turn a monolingual file into a bilingual XLIFF 2 file to preserve context and interoperability. This is an internal hand-off, not a problem to solve: downstream actions such as CMS uploads accept XLIFF 2 and convert it back to the representation they need. This rule applies only to textual content. Treat arbitrary cloud-drive assets, video, and other non-textual binaries according to the receiving action's schema instead.

## Build a new Bird

Create a complete definition with `create_bird`. Each element has a unique ID; `next`, branch targets, and loop membership - not the JSON array order - define execution order.

Use the selected action or event's full `externalFullName` as its reference `name`; do not use its short `name` or `displayName`. Do not rely on a numeric connector ID from discovery responses: it is informational and can become stale.

For every action:

- Set `type` to `action`, select its action reference, add a connection when its app requires one (omit it for connectionless Utilities), and supply all required inputs from its discovered schema.
- Link ordinary linear work with `next` to the next step ID. Omit it only when the action is deliberately terminal or execution is routed elsewhere.
- Use outputs from earlier steps only when their type and shape match the input. A result that looks like `[ ... ]` is a list, not one item.
- Configure `onError` deliberately for side-effecting work: retry transient failures, skip only when it is safe, and never hide a failure that would leave data inconsistent.

`create_bird` saves a draft unless `publish: true` is supplied. For a draft, review it with `get_bird_definition`, then call `publish_bird_definition` when the user authorizes it. To confirm what is live, call `get_bird_definition` with `published: true`. A Bird can be `active` while `isPublished` is false: its previous version is running and a draft is pending. If a publishing response includes `webhookPayloadUrls`, register each returned URL with the relevant external app immediately. The URL map is keyed by the trigger or checkpoint step ID and rotates on every republish.

For an authorized manual test, `fly_manual_bird` accepts the start request without returning a Flight ID. Find the new Flight with `list_bird_flights` or `search_flights`, then call `get_flight`. Its ordered step history shows the actual inputs and captured outputs for each executed step, including loop iterations. Compare that evidence with the intended bindings and values before treating the Bird as proven; use `get_flight_logs` only when retry, skip, error, or timing detail needs further investigation.

## Control Bird activation deliberately

Use `activate_bird` to resume a published, suspended Bird, and `deactivate_bird` to suspend a published, active Bird so it accepts no new trigger events or Flights. Both change lifecycle state and require Bird-manager access. Do not change activation state unless the user explicitly requests it or it is necessary to complete an explicitly requested workflow change. Read the Bird first when its current status is unknown; these operations are not idempotent and fail when the Bird is already in the target state or has not been published.

## Bind data to inputs

Each input is a binding. Use the smallest appropriate binding rather than copying values between steps.

- **Literal value** (`custom`): text, number, boolean, or other fixed value chosen while designing the Bird.
- **Earlier output** (`step`): a value from the trigger or an earlier step. Use its producer step ID and output path.
- **Connection-backed choice** (`dynamic`): a value from a data source that depends on the selected connection. First retrieve it with `get_app_action_input_options` or `get_app_event_input_options`; bind the returned `value` and, when available, `displayName`. The display name is cosmetic; the engine uses only `value`.
- **Fixed catalog choice** (`static`): a value from a non-connection-specific data source. Its `displayName`, when present, is cosmetic; the engine uses only `value`.
- **Interpolated value** (`composable`): use this to build text from literal text and one or more values from earlier steps. `template` contains placeholders such as `{{ messageText }}`, while `variables` maps each placeholder name to a full input binding. Every placeholder in the template must have a matching entry in `variables`; a variable can use a `step`, `loopItem`, `custom`, or other supported binding.
- **Current loop item** (`loopItem`): the single item currently being processed, optionally followed by a property path.
- **Structured value** (`object`): bind its named properties or array items independently when the action expects a complex object.

Prefer a direct `step` binding when an input should receive one earlier output unchanged. Use a composable binding when you need surrounding text or several values, for example a template such as `The text is {{ messageText }} from {{ channelId }}` whose variables bind those placeholders to trigger outputs.

Mark an input optional only when the app schema says it is optional: set binding `required` to `false` to explicitly make it optional; `true` and omission both follow the connector-defined requiredness. Input types are inferred during publish—do not add a `type` property to a binding. The only caller-declared type is an operator output: use canonical `string`, `integer`, `double`, `boolean`, `date`, `file`, `object`, or `array` (the API also accepts documented aliases). Do not send a list to an action expecting one item: add a loop or use an app action that explicitly accepts a list.

## Decisions and branches

Use a decision when the workflow must choose a route based on run-time data. A branch contains a condition with a left value, comparison operator, and right value, plus the ID of the first step on that route.

- Put the most specific conditions first when conditions overlap.
- Add exactly one `otherwise` branch as the fallback; it has no comparison condition.
- Route every branch to a valid next step or a deliberate terminal end.
- Keep shared follow-up work outside the decision and point successful branches to that common step where appropriate.
- A decision tests data; it does not create or transform it. Use an action or verified operator for that work.

## Loops

A loop processes every item in an array or list, one at a time. Its `forEach` binding must refer to a list-producing trigger/action output or another valid list binding. The loop `body` lists the IDs of steps that execute for each item.

- Use `loopItem` bindings inside the body to supply the current item (or one of its properties) to an action.
- Put every step that must repeat per item in the body. Create-once work - such as creating one translation project for many documents - belongs before the loop; aggregate or final work belongs after it.
- Ensure the body has a valid internal path for every iteration and that the loop itself has a `next` target for work after all items are complete.
- A loop can contain decisions and nested flow only when their IDs and routes are unambiguous. Keep it simple unless the user needs nested behavior.
- Name the loop after what it iterates, for example "For each source document", rather than "Loop 1".

## Checkpoints

A checkpoint pauses one Flight and then resumes it. Use a **delay** checkpoint only when a time wait is truly sufficient. Use an **event** checkpoint for a real completion, approval, or external-system state change.

For an event checkpoint, select the event app, event, connection, and - where supported - the exact object identifier produced by an earlier action. This prevents an unrelated event from resuming the wrong Flight. A checkpoint that returns multiple items provides them as a list for later use, so add a loop if subsequent work expects one item at a time.

Event checkpoints can also return webhook setup URLs. Register new URLs after both creation and update, because an update may rotate them.

## Operators and variables

An operator is a built-in workflow operation, distinct from a connector action. `get_bird_operator_catalog` provides the supported operators and their exact contracts.

Therefore:

- Use only an operator returned by the catalog, with its exact input keys.
- `variable.create` and `variable.update` are supported. Declare a created variable's name and type in `output` as specified by the catalog. `variable.create` accepts exactly one initial-value input; its key is free, while a legacy `name` input is ignored.
- Use a flow-ending operator only where it is appropriate for the intended branch or loop behavior; do not use it as a substitute for correctly wiring `next`, decision branches, or loop bodies.
- `flow.endLoop` and `flow.endFlight` are unconditional in the API definition dialect. To make either conditional, use a decision step that routes one branch to the flow operator.

### Variables are mutable workflow state

Use a direct output binding when a value is consumed once without changing. Create a variable when a value must be reused across later steps, given a meaningful name, accumulated over time, or replaced after a decision. A variable's name and type are declared by the `output` on `variable.create`; use that created output to identify the variable in a later `variable.update`.

An update replaces the current value. To build text or collect items, bind both the current variable value and the new value into the update as the operator catalog prescribes. For an array accumulated inside a loop, initialise it before the loop, update it once per iteration, and consume it after the loop. Updating the collection that a loop is already iterating does not change that loop's iteration set.

### Correlate records across separate Birds

Prefer an event checkpoint when one Bird can wait for the specific external event it needs. When the connected system supports custom properties, persist the corresponding record ID there instead. Use entity links only as a fallback when separate Birds must later connect two business records and neither approach is viable; this capability is being phased out.

With that fallback, use `entityLink.set` when both records are created or known, then `entityLink.get` or `entityLink.getMany` in the later Bird. Give entity types durable, recognizable names and link stable record identifiers, not display text. Confirm the exact operator input names and output shape from `get_bird_operator_catalog` before composing the definition.

## Safely update an existing Bird

`update_bird_definition` replaces the complete editable definition. Follow this sequence:

1. Read the current definition, then make the smallest requested change in that full definition.
2. Preserve existing trigger and step IDs, connections, inputs, routes, error settings, and unknown-but-valid fields unless the user asked to change them. IDs are the stable references used by bindings, `next`, loop bodies, and branches.
3. Do not replace the definition with `get_bird_schema`: that response describes fields as arrays for display, whereas an editable definition uses binding maps.
4. Validate every reference: each `next`, branch target, loop body item, source step ID, and loop ID must point to an element that still exists.
5. Submit the full edited definition as a draft unless immediate publication is explicitly intended. Review the saved definition, then call `publish_bird_definition` to make it live. If the publishing response includes webhook payload URLs, re-register them in the external app; they rotate on every republish.
6. Confirm the live definition with `get_bird_definition` using `published: true` and, for a manual Bird, run a controlled Flight. Locate it in the flight list, then inspect `get_flight`'s per-step inputs and outputs before treating the change as complete. Inspect Flight logs as needed for retry, skip, error, or timing detail.

## Practical quality checks

Before creating or updating, confirm all of the following:

- The trigger matches how and when the user actually wants the workflow to start.
- Every selected action/event exists in the chosen app and every connection belongs to the target Nest.
- All required action inputs have compatible bindings.
- Decisions have one fallback and valid branch destinations.
- Loops receive a list and use the current item only in repeated work.
- Checkpoints wait for the right duration or the right specific object/event.
- Event trigger/checkpoint URLs have a registration plan.
- The exact proposed definition passed `preflight_bird_definition` with `valid: true`.
- Side effects are not repeated accidentally by a schedule, retry, or loop.
- The workflow is explained back in plain language, including where it starts, what it repeats or decides, when it waits, and what happens on failure.
