# Bird definition JSON patterns

These are concise, editable-definition patterns for MCP `create_bird` and `update_bird_definition`. They are not complete Birds and do not supply real connector names, connection IDs, input names, output paths, or comparison operators. Discover those values first with the app schema tools; call `get_bird_operator_catalog` for operator contracts.

All IDs below are illustrative. For an update, preserve IDs from `get_bird_definition`; for a new Bird, use unique IDs consistently throughout the definition. The `name` in an action or event reference must be the externally resolved name returned by discovery, not a made-up display label or stale numeric connector ID.

## Manual and event triggers

A manual trigger is a safe starting point for a newly built Bird:

```json
{
  "name": "Send selected content for translation",
  "trigger": {
    "id": "manual-start",
    "type": "manual",
    "next": "download-content"
  },
  "steps": []
}
```

An event trigger needs the exact discovered event, a connected app account, and any event inputs required by its schema. Link-mode events can return a webhook URL after a publishing operation.

```json
{
  "trigger": {
    "id": "content-created",
    "type": "webhook",
    "event": {
      "name": "Example.App.Events.ContentCreated"
    },
    "connection": 12345,
    "mode": "subscription",
    "inputs": {
      "projectId": {
        "source": "custom",
        "value": "project-123"
      }
    },
    "next": "download-content"
  }
}
```

Use `type: "polling"` with `interval` only for an event that is exposed as polling. Use `type: "bucketing"` with `amount` and/or `timer` only when batching events is intentional.

## Action and input bindings

An action invokes one exact app capability. The action schema determines its real `name`, required input keys, and output paths. This example sends an earlier trigger output to an app action and uses a fixed value for another input.

```json
{
  "id": "download-content",
  "type": "action",
  "name": "Download content",
  "action": {
    "name": "Example.App.Actions.DownloadContent"
  },
  "connection": 12345,
  "inputs": {
    "contentId": {
      "source": "step",
      "stepId": "content-created",
      "output": "content.id"
    },
    "includeMetadata": {
      "source": "custom",
      "value": true
    }
  },
  "next": "after-download",
  "onError": {
    "retry": {
      "count": 3,
      "intervalSeconds": 30
    }
  }
}
```

Other valid binding patterns are:

```json
{
  "source": "loopItem",
  "loopId": "for-each-document",
  "path": "file"
}
```

```json
{
  "source": "composable",
  "template": "Translation for {{title}}",
  "variables": {
    "title": {
      "source": "step",
      "stepId": "download-content",
      "output": "title"
    }
  }
}
```

Use `dynamic` or `static` bindings only with a value returned by the field's configured data source. Use an `object` binding when the action schema expects a structured value and its individual fields need separate bindings.

## Decision

A decision has routed branches rather than one ordinary linear next step. Every decision needs exactly one fallback branch. The actual comparison operator must be supported by the relevant field/value types.

```json
{
  "id": "is-translatable",
  "type": "decision",
  "name": "Content is translatable?",
  "next": null,
  "branches": [
    {
      "name": "Translatable",
      "when": {
        "left": {
          "source": "step",
          "stepId": "download-content",
          "output": "isTranslatable"
        },
        "operator": "equal",
        "right": {
          "source": "custom",
          "value": true
        }
      },
      "next": "create-translation"
    },
    {
      "name": "Not translatable",
      "otherwise": true,
      "next": "record-skip"
    }
  ]
}
```

Do not use an `otherwise` branch together with `when`. Point both successful paths to a shared step only when that step should run for either result.

## Loop

Use a loop when an earlier output is an array but the next action takes one item. The loop body contains all repeated step IDs. The loop's `next` runs only after all items have been processed.

```json
{
  "id": "for-each-document",
  "type": "loop",
  "name": "For each source document",
  "forEach": {
    "source": "step",
    "stepId": "download-content",
    "output": "documents"
  },
  "itemName": "Source document",
  "body": ["translate-document", "finish-document"],
  "next": "notify-complete"
}
```

An action inside the loop consumes the current item. The flow-ending operator illustrates the loop-body endpoint; use the exact contract returned by `get_bird_operator_catalog`.

```json
{
  "id": "translate-document",
  "type": "action",
  "action": {
    "name": "Example.Tms.Actions.TranslateDocument"
  },
  "connection": 67890,
  "inputs": {
    "document": {
      "source": "loopItem",
      "loopId": "for-each-document"
    }
  },
  "next": "finish-document"
}
```

```json
{
  "id": "finish-document",
  "type": "operator",
  "operator": "flow.endLoop",
  "loopId": "for-each-document",
  "next": null
}
```

## Checkpoint

Use a delay checkpoint for a fixed wait. Use an event checkpoint for a real external completion or approval, and bind the exact prior object identifier if the event schema supports it.

```json
{
  "id": "wait-for-project-completion",
  "type": "checkpoint",
  "name": "Wait for translation project completion",
  "mode": "event",
  "event": {
    "name": "Example.Tms.Events.ProjectCompleted"
  },
  "connection": 67890,
  "webhookMode": "subscription",
  "inputs": {
    "projectId": {
      "source": "step",
      "stepId": "create-translation",
      "output": "project.id"
    }
  },
  "next": "download-translation"
}
```

For a delay, use `"mode": "delay"` and a `duration`, for example `{"hours": 2}`. Use checkpoint polling or bucketing only when supported by the selected event and required by the workflow.

## Operators and error handling

Operators are built-in flow operations. Call `get_bird_operator_catalog` before authoring one; it defines the exact inputs and output behavior.

```json
{
  "id": "save-project-id",
  "type": "operator",
  "operator": "variable.create",
  "inputs": {
    "value": {
      "source": "step",
      "stepId": "create-translation",
      "output": "project.id"
    }
  },
  "output": {
    "name": "translationProjectId",
    "type": "string"
  },
  "next": "wait-for-project-completion"
}
```

Attach error handling only where it represents the intended recovery behavior:

```json
{
  "onError": {
    "retry": {
      "count": 3,
      "intervalSeconds": 30
    }
  }
}
```

Use retry for likely transient failures. `skipStep: true` is suitable only for a genuinely optional step. `skipLoopIteration: true` is suitable only inside a loop where a bad item can safely be omitted. Do not combine skip behavior with work that must succeed to avoid data loss or inconsistent external state.
