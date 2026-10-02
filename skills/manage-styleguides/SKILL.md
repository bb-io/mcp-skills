---
name: manage-styleguides
description: Create, assess, refine, or apply Blackbird style guides when a user needs governed writing guidance for a brand, language, market, audience, or channel.
---

Use this skill when the Blackbird MCP server is available and the user asks to manage writing rules or style guidance. Do not use it for terminology management; use a termbase workflow when the request is about approved, discouraged, or forbidden terms.

Style guides are governed context used by downstream language workflows. They can express voice and tone, capitalization, punctuation, formatting, audience, channel, market, and language-specific guidance.

## MCP field vocabulary

Translate MCP field names into their product meaning before proposing a change:

| MCP field | Product meaning |
| --- | --- |
| `lakeId` | The Lake that owns the assets. |
| `styleguideId` | The specific style guide being read or changed. |
| `templateLanguage` | The UI's **Template language**: a choice of predefined rule categories and options, not a translation target or the style guide's scope. |
| style guide `description` | The optional overview shown for the style guide. |
| rule `description` | The actual writing instruction. For a template rule, this must exactly equal the template's `rule` text. |
| rule `category` | The UI section for that instruction. For a template rule, this must exactly equal the template category title; use `Other` for a freeform rule. |
| `variantId` | The language, locale, market, or other content variant to which a style guide is attached. |

## Assess before changing

1. Use `list_accessible_resources` to identify the lake when it is unknown.
2. Use `list_styleguides`, `get_styleguide`, and `list_styleguide_rules` to inspect the current configuration before proposing an update.
3. If the user wants a guide applied to a language or market, use `list_variants` and `list_variant_styleguides` to establish the target variant and existing associations.
4. State the evidence, the proposed change, and any expected downstream impact before performing a write, unless the user has already explicitly requested the precise change.

## Design guidance

- Keep one guide focused on a coherent concern. Separate brand-level rules from language-, region-, audience-, or channel-specific guidance when those instructions can differ.
- Use custom rules for organization-specific instructions that do not fit a language template, such as product naming, brand voice, or editorial constraints.
- Write rules as clear, testable instructions. Include the preferred behavior and, when useful, a short counterexample or prohibited behavior.
- Avoid duplicating terminology rules in a style guide. Recommend a termbase when the user needs variant-specific preferred, discouraged, or forbidden terms.

## Template language

The template language provides predefined language-specific categories and choices; it does not translate custom rules. Changing it clears the previously enabled template-rule structure and replaces it with the selected language's structure. Custom rules are managed separately.

Do not change `templateLanguage` unless the user explicitly requests it or approves the stated impact. Use custom rules when no suitable template exists.

Templates are an optional catalog of available rules, not a default style guide or a checklist. Do not enable, create, or recommend every template rule. Add only the specific rules supported by the user's stated requirements or by evidence from the existing guide, content, and workflow. Leave every other template rule disabled.

For a guide with a supported template language, read the matching file in [references/style-guide-templates](references/style-guide-templates/README.md) only when a needed rule maps to that template. The template is an exact contract with the Fly UI:

- Create a rule using the template category's `title` as `category` and the selected rule or option's exact `rule` value as `description`.
- For a rule with `options`, create only the one option that represents the user's requested preference.
- Do not paraphrase, translate, combine, or otherwise alter a template `rule` value. Any mismatch makes the rule appear as a custom rule in the UI.
- Treat rules not present in the selected language template as custom rules, normally using category `Other`.

The bundled template files cover `de`, `en`, `es`, `fr`, `hu`, `it`, `ja`, `ko`, `nl`, and `zh`. They are a snapshot of the Fly UI templates; update this skill's copies when those UI templates change.

## Apply changes

- Use `create_styleguide`, `update_styleguide`, and `delete_styleguide` only for explicitly requested changes.
- Use `create_styleguide_rule`, `update_styleguide_rule`, and `delete_styleguide_rule` for custom rules. Read the existing rule list first when changing or deleting rules.
- To apply a guide, confirm the intended variant, then use `add_variant_styleguide`. Do not remove an existing variant association without explicit user approval; list associations first with `list_variant_styleguides`.

After a write, re-read the affected guide, rules, or variant associations and report the completed change and any remaining recommendations.
