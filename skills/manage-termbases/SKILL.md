---
name: manage-termbases
description: Create, organize, and maintain Blackbird termbases and their concepts and terms when terminology must be governed across languages, locales, markets, or workflows.
---

Use this skill when the Blackbird MCP server is available and the user asks to manage approved, preferred, discouraged, keyword, or forbidden terminology. Use a styleguide instead for general writing, voice, punctuation, or formatting instructions.

Termbases are governed context: they represent concepts and their terms per variant, and can enrich downstream translation, rewriting, generation, and other language workflows. A termbase is not merely a flat word list.

## MCP field vocabulary

Translate the MCP's camelCase fields into their product meaning before proposing or making a change:

| MCP field | Product meaning |
| --- | --- |
| `lakeId` | The Lake that owns the terminology assets. |
| `termbaseId` | The termbase that contains the concepts and terms. |
| `termGroupId` | A **concept group**: one underlying concept that can contain source terms, translations, alternatives, and prohibited forms across variants. Used in term data and when filtering `list_terms`. |
| `termgroupId` | The same concept-group ID, using the spelling required by the group-management MCP tools. |
| `variantId` | The specific Blackbird Variant for a term: normally a language, locale, market, or other content variant. It is an ID, not a language code or a termbase name. |
| `term` | The actual word or phrase for that variant. |
| `description` | The UI's **Definition** or disambiguating context. It is included in downstream context when the term is leveraged. |
| `notes` | The UI's **Notes**: internal editorial or operational comments. They are never included in downstream context. Use an empty string when there are no notes. |
| termbase `description` | The optional overview of the termbase, not a definition for each term. |
| `status` | The governance treatment of this individual term; see the status mapping below. |

### Status mapping

| MCP value | UI label | Meaning |
| --- | --- | --- |
| `admitted` | Accepted | A valid possible term or translation; included when the term or translation is in scope. |
| `preferred` | Preferred | A valid term favored over accepted alternatives; included when in scope. |
| `not recommended` | Not recommended | In scope but generally avoided; pair it with an accepted or preferred alternative for the same concept. |
| `desired` | Keyword | A term to use where possible, often for SEO; always included in full context during preparation. |
| `forbidden` | Forbidden | A blocked term that must not be used; always included in full context during preparation. |

## Assess and structure terminology

1. Use `list_accessible_resources` to identify the Lake when it is unknown.
2. Use `list_termbases`, `get_termbase`, `list_termbase_groups`, and `list_terms` to inspect existing terminology before adding or changing it.
3. Use `list_variants` to find the intended target variant before creating a term. Do not substitute a language label for `variantId`.
4. Decide whether the request is a new concept or another expression of an existing concept. Keep translations, preferred forms, discouraged alternatives, and forbidden forms for the same meaning in one concept group.
5. State the proposed terminology, variant, definition, status, and expected downstream effect before a write, unless the user has already explicitly requested those precise changes.

## Create and maintain concepts

- To start a new concept, call `create_term` without `termGroupId`. Blackbird creates its concept group; then use `list_termbase_groups` or the returned term's `termGroupId` to identify it.
- To add a translation or alternative to an existing concept, pass that concept's `termGroupId` to `create_term`.
- Use `update_termbase_group` only to rename the human-facing concept-group name. Read the group and its terms first.
- Do not treat `description` as a private comment. Put context that should guide downstream language work in `description`, and put workflow, ownership, or review information in `notes`.
- Keep brand-specific lexical choices in a termbase; keep broader grammatical, tone, formatting, or writing behavior in a style guide.

## Apply changes safely

- Use `create_termbase`, `update_termbase`, and `delete_termbase` only for explicitly requested changes. A deleted termbase is permanent.
- Use `create_term`, `update_term`, and `delete_term` only after locating the intended term or concept. Preserve the associated variant and status when an update is not meant to change them.
- Before deleting a concept group with `delete_termbase_group`, list its terms and obtain explicit confirmation that the whole concept is no longer needed.

After a write, re-read the affected termbase, group, or terms and report the user-facing result: concept, variants, definitions, notes, and term statuses.
