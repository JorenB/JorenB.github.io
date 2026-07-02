# Safe testing part 1 — rendering fixes needed before publishing

## Warning box

The blockquote (`> **⚠️ ...**`) renders with distill's default blockquote style: italic text, left border. Looks off for a callout. Options:
- Use a Bootstrap alert: `<div class="alert alert-warning" role="alert">...</div>`
- Add a custom CSS class in `assets/css/` and use a styled `<div>`

## `{% details %}` title quotation marks

The title passed as `{% details "Derivation" %}` renders with literal quotation marks around the word in the distill template. Fix by either:
- Dropping the quotes: `{% details Derivation %}` — check if the plugin supports unquoted titles
- Patching `_plugins/details.rb` to strip surrounding quotes from the title argument

## Figure–caption gap

The `<div class="caption">` sits too far below the figure. Options:
- Use the `caption=` parameter directly on the include: `{% include figure.html ... caption="..." %}` and check if that has tighter spacing
- Add to `_sass/` or `assets/css/`: `.caption { margin-top: -1rem; }` (or similar)
