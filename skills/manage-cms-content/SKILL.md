---
name: manage-cms-content
description: Build or update Blackbird workflows that localize or process content from a CMS, PIM, commerce platform, or another system that manages page- or product-level content.
---

# Manage CMS content workflows

Use this skill for content localization, translation, enrichment, publishing, or synchronization from a CMS in the broad sense: a conventional CMS, PIM, e-commerce platform such as Shopify, knowledge base like Zendesk, or another system that manages content entities. An entity might be an article, entry, product, page, help-center item, or blog post.

Use this skill together with [Build and manage Blackbird Birds](../build-birds/SKILL.md). That skill governs API/MCP discovery, action schemas, connections, bindings, loops, checkpoints, error handling, and safe complete-definition updates. Use this skill to make the content and Blacklake choices.

## Understand the content model first

Before building, establish:

- The content system, entity types, source locale, target locales, and the fields or embedded content that are actually localizable.
- Whether the system supports locales natively, needs a localization plugin, or needs a separate target entity. Do not assume all fields, entity types, or locales are localizable.
- How content should be selected: an event/workflow state, a tag or custom field, a manual/scheduled search, or a backfill.
- What should process the content: machine translation, AI transformation, a TMS, human review, or another approved service.
- How the CMS identifies the source entity, target locale, and completion state so that translated content returns to the right place.

Treat CMS features such as tags, custom fields, workflow states, categories, and locale settings as part of the workflow design. In particular, prevent event-trigger loops: publishing a translated entity can otherwise retrigger the source-content Bird. Filter for source-locale content or a dedicated "ready for localization" marker, and remove or transition that marker only after successful completion.

## Discover the real app capabilities

Use `search_apps`, `list_app_actions`, `get_app_action_schema`, `list_app_events`, and `get_app_event_schema` for the selected content system. Look specifically for its equivalent of:

1. An event or search action to identify eligible entities.
2. An action that exports a source entity as a translation-ready HTML/XLIFF file, preserving the entity and locale context needed for re-import.
3. An action that imports the processed file into the correct target entity and locale.
4. Locale, language, tag, workflow, or metadata actions needed for selection and completion.

Do not use an action name from another CMS as a contract. The schema defines fields, outputs, supported embedded content, and app-specific limitations. For Contentful, for example, discover which referenced or rich-text embedded entities should be recursively included and bound the source locale explicitly when its schema requires it.

Blackbird handles textual file interoperability for you. Export, Prepare content, translation, and CMS upload steps can pass their file outputs directly to one another; do not add a format-conversion step just because preparation or translation produces bilingual XLIFF 2. CMS upload actions accept that intermediary and convert it back to the CMS representation. This does not apply to arbitrary binary assets such as video, which must be handled according to the CMS action schema.

## Make Blacklake part of the round trip

For every translation or localized-content workflow, try to include Blacklake. Discover the Blacklake app actions, schemas, accessible Lake, and any required connection before composing the Bird. A Blacklake connection requires the user's Blacklake server URL and project key. If Blacklake is not available to the user, explain what resource, credential, or permission is missing rather than silently claiming its benefits.

The standard order is:

1. Export the CMS entity to its translation-ready file.
2. Call Blacklake **Prepare content** on that file before any translation or language processing. It diffs known content, pre-fills unchanged units, applies the selected leverage strategy when one is configured, and enriches the file with applicable style guides, terminology, and other governed context.
3. Send the prepared file to the translation or processing service. For a TMS, create the job, wait with a checkpoint scoped to that job, then download the completed file. For AI call a relevant **Translate** Action depending on the service the user wants to use.
4. Import the completed file into the CMS at the intended entity and target locale.
5. Call Blacklake **Store content** with the output of the successful CMS upload action, as required by its discovered schema. This preserves source-system context such as the uploaded-content link while recording the result for future diffs and leverage.

Keep the Blacklake-prepared artifact intact between preparation and the processing service. Do not replace it with a separately re-exported CMS file before translation; that would discard the prepared context. After upload, bind Store content to that upload result rather than directly to the translation-service download.

### Configure Blacklake from the file first

The translation-ready file normally carries the source/target variant and content identity. Leave Blacklake's variant-code and source-content-ID overrides empty unless the CMS export lacks or misstates that metadata: explicit inputs take precedence over file metadata. Provide a source content ID only for a CMS that stores source and target content without a link between them.

Select a strategy when the user needs governed reuse; without one, Prepare content still performs a diff but does not leverage by strategy. Choose whether preparation is for translation or editing, and select only the termbases that should contribute translations, definitions, keywords, or forbidden terms. Store content can also take a monolingual CMS export to synchronize in-context edits; when doing so, use its align-with-variant option only if the discovered schema and content model require it.

When a user wants to alter the context applied by Prepare content, use the dedicated skills for [style guides](../manage-styleguides/SKILL.md), [termbases](../manage-termbases/SKILL.md), or [strategies](../manage-strategies/SKILL.md). Explain these as writing guidance, approved terminology, and rules for eligible reusable content - not their API field names.

## Choose trigger and language patterns deliberately

- Use an event or CMS workflow-state trigger for continuous localization when the source system can reliably distinguish source content from translated updates.
- Use a scheduled or manual trigger plus a search action for backfills, controlled batches, or historical content. Loop over the returned entities and make processing idempotent where possible.
- Read available locales from the CMS when the target locales should follow its configuration. Map locale codes between the CMS and translation service when their conventions differ.
- A TMS commonly accepts several targets in one project; a one-language-at-a-time translation action requires a target-locale loop. Keep entity looping and locale looping explicit so an agent never confuses one file with a list of files.
- Use event checkpoints for human or TMS completion, binding the exact project/job/entity identifier produced earlier. Do not use an arbitrary time delay as a substitute for known completion.

## Preserve content integrity

- Retain the source entity ID, source locale, target locale, and translation-ready file throughout the Flight. These are what let the CMS importer update the correct content.
- Do not translate system identifiers, locale codes, markup, placeholders, or embedded context by hand. Use the CMS export/import actions and the processing service's file workflow.
- Include embedded or referenced content only when it should be localized with the parent entity; apply the CMS schema's recursion and exclusion controls to prevent unnecessary traversal or cycles.
- Make publishing, workflow-state changes, and marker removal the final successful side effects. Do not mark content complete after a failed translation or failed CMS import.
- For updates to an existing Bird, preserve its app-specific selection filters, fields, connection IDs, and file hand-offs unless the request explicitly changes them. Re-read and test a controlled/manual Flight before expanding a CMS workflow to continuous operation.

Describe the completed workflow in content terms: what starts it, which entities and locales it handles, how Blacklake prepares and stores the content, how processing completes, where the translated content returns, and how repeat processing is prevented.
