# Style guide templates

Use the JSON file whose name matches the style guide's `templateLanguage`. Each file contains the category titles, template rules, and available options used by the Fly UI.

The templates are an optional catalog, not a set of rules to implement wholesale. Read a template only to find the exact UI-compatible representation for a rule that the user has requested or that is supported by evidence. Do not create rules merely because they appear in the template.

To create or update a non-custom template rule, send the selected template's exact values to `create_styleguide_rule` or `update_styleguide_rule`:

```json
{
  "category": "<category.title>",
  "description": "<rule.rule or selected option.rule>"
}
```

The UI identifies template rules by exact equality of these two values. A category or description that does not exactly match one of the bundled templates is displayed as a custom rule.

Available templates: `de`, `en`, `es`, `fr`, `hu`, `it`, `ja`, `ko`, `nl`, and `zh`.

Source: `apps/fly/src/data/style-guide-templates` in the Blackbird repository. Keep these files synchronized with that source when the UI templates change.
