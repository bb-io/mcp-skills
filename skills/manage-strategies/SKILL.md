---
name: manage-strategies
description: Create, assess, and maintain Blackbird reuse strategies and custom metadata when a user needs governed rules for which content may be reused in a workflow.
---

Use this skill when the Blackbird MCP server is available and the user asks which stored content may be reused, trusted, excluded, or prioritized during content preparation. Use it also for metadata-field definitions that support those decisions. Do not use it for style or terminology rules.

Strategies are reusable governance policies. They decide which stored units are eligible to participate in reuse during Prepare Content; they do not merely filter results after a match is found. Metadata fields provide business context that strategies can use for those decisions.

## Speak in product language

Use the plain-language terms below with the user. Use camelCase names only inside MCP tool arguments or when explaining a technical integration detail.

| MCP field | Say to the user |
| --- | --- |
| `lakeId` | The Lake that holds the content and governance settings. |
| `strategyId` | The reuse strategy being reviewed or changed. |
| `filters` | The strategy's reuse eligibility rules. |
| `diff` | **Compare incoming content with the latest stored version first.** Unchanged content can be finalized without further work. |
| `leverage` | **Search for eligible stored matches to reuse.** Disable it for a change-only workflow with no reuse search. |
| `inactive` | **Include inactive or unpublished content in reuse.** |
| `translatedByPerson` | **Require human-translated content.** |
| `reviewedByPerson` | **Require human-reviewed content.** |
| `minimumQualityScore` | **Minimum quality score required for reuse.** |
| `allowedContentSystems` / `notAllowedContentSystems` | **Trusted source systems** / **excluded source systems**. |
| `translatedByTools` / `notTranslatedByTools` | **Allowed tools or engines** / **excluded tools or engines**. |
| `translatedByPersons` / `notTranslatedByPersons` | **Allowed translators or reviewers** / **excluded translators or reviewers**. |
| `createdAfter` / `createdBefore` | **Only reuse content created on or after / on or before a date.** |
| `metadataFilters` | **Business-context conditions for reuse.** |
| `metaFieldId` | The selected **metadata field** used by a business-context condition. |
| metadata field `type` | The field's data kind: **Text**, **Number**, or **Yes/No**. |

`metaFieldId` is an implementation ID. Before using it, find the field with `list_metadata_fields`, then explain its name and purpose to the user instead.

## Assess before changing

1. Use `list_accessible_resources` to identify the Lake when it is unknown.
2. Use `list_strategies` and `get_strategy` to understand existing reuse policies before proposing a new one or modifying one.
3. For a business-context condition, use `list_metadata_fields` first. Create a metadata field only when it supports a concrete reuse, prioritization, exclusion, governance, or reporting decision.
4. Describe the policy in workflow terms before writing it: what content is trusted, what must be excluded, whether reuse is enabled, and the quality, date, source, tool, people, or business-context boundaries.

## Design strategies

- Prefer a small set of named policies for distinct purposes, such as strict production reuse, broader operational reuse, and experimental AI-assisted reuse.
- Enable comparison with the latest stored version in most workflows. Enable reuse only when the workflow should search stored content for eligible matches.
- Require human translation and/or review for high-visibility or high-risk content when that is the intended quality bar.
- Use date limits or inactive content deliberately—for example, to avoid legacy messaging or support a controlled rollback.
- Use source-system, tool, and people restrictions to express trust boundaries, not as arbitrary technical filters.

## Metadata and business-context conditions

A metadata field is a reusable Lake-level schema definition. It is not a value on a piece of content. Blackbird workflows usually populate values later from source systems, analytics, or enrichment processes.

- Create a field with a clear name, data kind, and description that state its intended business decision. Use **Text** for labels and categories, **Number** for scores and KPIs, and **Yes/No** for flags such as deprecated or needs review.
- Do not create a field merely because it may be useful someday. Prefer fields that answer whether content should be reused, prioritized, excluded, or assigned to a business segment.
- When defining a business-context condition, use exact matching for Text and Yes/No fields. Number fields can use exact or range comparisons. In MCP input, `=>` means **greater than or equal to**.
- Explain the condition using the metadata field's human name, not its ID or the `metadataFilters` implementation name.

## Apply changes safely

- `create_strategy` only creates the name and optional description with default eligibility rules. It does not accept the policy itself or return its ID. After creating one, use `list_strategies` to identify it, then call `update_strategy` with the complete intended policy.
- `update_strategy` replaces the strategy's entire name, description, and eligibility-rule set. Always read the current strategy first and carry forward every rule that the user has not explicitly asked to change.
- Use `create_metadata_field`, `update_metadata_field`, and `delete_metadata_field` only to manage field definitions. The available MCP tools do not set metadata values on individual content.
- Delete a strategy or metadata field only with explicit confirmation after explaining its expected impact.

After a write, re-read the strategy or metadata field and report the resulting policy in product language: eligible content, exclusions, trust requirements, time window, and business-context conditions.
