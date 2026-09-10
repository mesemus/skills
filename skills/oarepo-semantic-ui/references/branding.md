# Repository branding — colors, fonts, logos, and page chrome

Read this when customizing a repository's overall look and feel — colors,
fonts, logo, header/footer/frontpage, or site copy — as opposed to a
record model's forms/search/detail pages (covered elsewhere). Grounded in a
real repository's actual files, not just the general docs.

## File locations

**Don't assume LESS overrides live under `assets/semantic-ui/less/...`** by
analogy with how a Python package nests its own theme assets
(`theme/assets/semantic-ui/js/<pkg>/`, documented elsewhere in this skill)
— at the top-level repository instance, there is no `semantic-ui/` segment.
Branding files live directly at:

```
myrepo/
  assets/
    less/site/
      globals/{site.variables, site.overrides}
      elements/{button,header,input,label,container}.{variables,overrides}
      modules/{checkbox,dropdown}.{variables,overrides}
      views/{card,item}.{variables,overrides}
      collections/{form,menu}.{variables,overrides}
      fonts/*.woff2
      images/*.svg
  static/
    images/*.svg, *.png   # logo, favicons, funder/partner logos
  templates/
    page.html, header.html, footer.html, frontpage.html
    css.html, javascript.html
    macros/card.html
```

This is the top-level **instance** override root (distinct from a Python
package's own `theme/assets/semantic-ui/js/<pkg>/` structure documented
elsewhere in this skill) — `invenio-cli`'s asset build picks it up directly
by convention, with no `webpack.py`/entry-point registration needed for
the LESS/template tree itself (unlike the model-specific webpack bundles in
[wiring.md](wiring.md)).

**Override hierarchy** (each layer overrides the one before it — "everything
under your `assets/less` wins"): Semantic UI base theme → `invenio-theme` →
`invenio-app-rdm` → `oarepo_ui` → your repository. The same layering applies
to `templates/` (relative-path matching, see §5).

## 1. Colors

The established pattern is a full numbered color scale per family, not one
flat `@brandColor` variable:

```less
// assets/less/site/globals/site.variables

// Numbered scales (light → dark), one per semantic color family
@mainColors1: #d3f6f9;  // ... through
@mainColors9: #011314;
@secondaryColors1: #fbe0e6;  // ... through @secondaryColors9
@greys1: #edf2f3;            // ... through @greys9
@confirmation1: #cafdcc;     // ... @alerts*, @errors*, @reds*, @pinks*

// Semantic aliases derived from the scale, not hardcoded literals
@primaryColor: @mainColors7;
@secondaryColor: @pinks1;
@darkGrey: #707075;
@lightGrey: #e4e4e3;

// Kept only for backwards compatibility with code expecting @brandColor
@brandColor: @primaryColor;
@brand-primary: @primaryColor;
```

Define a numbered scale per color family, derive semantic variables
(`@primaryColor`, `@secondaryColor`, ...) from the scale, and keep
`@brandColor` as an alias if anything still references it — don't just set
one flat `@brandColor` and stop there.

**Site-wide chrome colors:**

```less
@navbarBackgroundColor: @brandColor;         // solid navbar background
@navbarBackgroundImage: linear-gradient(...); // or a gradient/image instead
@footerLightColor: @primaryColor;
@footerDarkColor: @primaryDarkenColor;        // a darker shade of the brand color
```

**Component-specific colors** follow the `{componentType}/{component}.variables`
convention — the real component-type folders are `elements/`, `collections/`,
`views/`, `modules/` (standard Semantic UI taxonomy), e.g.
`elements/button.variables`:

```less
@buttonBackgroundPrimaryColor: @primaryColor;
@buttonBackgroundPrimaryHoverColor: @pinks1;
@borderRadius: 0.5em;          // maps to Semantic UI's own button variable
@backgroundColor: @buttonBackgroundSecondaryColor;
```

Look up the variable name in the upstream Semantic UI/`invenio-theme`
component source first, then redefine only that variable in your own
`.variables` file — don't copy the whole upstream file.

## 2. Style overrides beyond variables

When no variable exists for what you need, override the CSS rule directly
in the matching `.overrides` file (same `{componentType}/{component}.overrides`
naming). **Real, non-obvious gotcha found in the actual `site.overrides`**:
some `invenio-theme` styles genuinely cannot be reached through any
variable and must be force-overridden by selector, with a comment
explaining why:

```less
/* Override defaults from invenio-theme that cannot be overriden other way */
html.cover-page {
  background-color: @coverPageBackgroundColor !important;
}
.ui.page-header #invenio-burger-menu-icon {
  .navicon, .navicon::before, .navicon::after {
    background: @white !important;
  }
}
```

If you've defined a variable and it isn't taking effect, don't assume
you've misspelled it — check whether the target style is one of these
variable-proof cases and needs a direct, `!important`-qualified selector
override instead.

## 3. Fonts

```less
// assets/less/site/globals/site.overrides
@font-face {
  font-family: "Roboto";
  font-style: normal;
  font-weight: 100 900;      // variable-weight font, one @font-face for the whole range
  font-display: swap;
  src: url("../fonts/Roboto-Regular-Latin.woff2") format("woff2");
}
```

Two path styles are both used in the real codebase, for different
purposes — don't assume there's exactly one correct form:

- **A relative path** (`../fonts/Roboto-Regular-Latin.woff2`) when the
  referencing file is itself inside `site/` (here, `site/globals/` →
  `../fonts/` resolves to the sibling `site/fonts/` directory).
- **The `~@less/` alias** (resolves to the whole `assets/less/` root, so a
  `site/`-nested asset needs the `site/` segment spelled out:
  `~@less/site/images/background-image.svg`) when referencing from
  somewhere the relative path wouldn't reach as cleanly, or from outside
  the `site/` tree entirely (a `.jsx` file can also use the `@js/`-sibling
  `~@less/` alias the same way).

Apply the font by name via your own variable (`@fontName: "Roboto";` in
`site.variables`) rather than hardcoding the family name at every use site.

## 4. Logos and static assets

Two categories:

- **Static** (`myrepo/static/images/...`) — served as-is, no build step,
  referenced via `THEME_LOGO` config or `url_for('static', filename=...)`.
  Use for the logo, favicon, and any asset referenced from `invenio.cfg`.
- **Dynamic** (`myrepo/assets/less/site/images/...`) — processed by the
  webpack/Rspack build, referenced via the `~@less/` alias from LESS (or
  `@js/` from JS). Use for anything only ever referenced from LESS/JS, like
  a CSS background image.

```python
# invenio.cfg
THEME_LOGO = "images/logo.svg"   # path is relative to static/, not absolute
```

**Real, reusable pattern**: locale-conditional logo swapping, from the
actual `header.html` override:

```jinja
{%- block brand %}
  {%- if current_i18n.language == 'cs' %}
    <img src="{{ url_for('static', filename='images/logo.svg') }}"
         alt="{{ _(config.THEME_SITENAME) }} {{ _('home') }}"/>
  {%- else %}
    <img src="{{ url_for('static', filename='images/logo_en.svg') }}" .../>
  {%- endif %}
{%- endblock brand %}
```

Ship one logo variant per locale that needs different text/wordmark, and
switch on `current_i18n.language` in the `brand` block override rather than
trying to make one SVG serve every language.

## 5. Page chrome templates (header, footer, frontpage)

Your `templates/` directory mirrors the relative path of whatever you're
overriding, and Jinja's `{% extends %}`/`{% block %}` composition does the
rest — no special registration needed for path-based overrides (only
`_TEMPLATE`/`_TEMPLATES`-suffixed config keys, §6, need a config entry).

**These are the same "extend the default, override one block, keep a
doc-comment cheat-sheet of every available block" convention already
documented for model page templates in
[jinjax-components.md](jinjax-components.md) and
[wiring.md §6](wiring.md#6-the-templates-are-mostly-empty-extends-stubs)
— it's a repository-wide scaffolding convention, not something specific to
record models.** Real examples:

```jinja
{# templates/page.html — the repo's own base page #}
{% extends "oarepo_ui/base_page.html" %}
{% block page_footer %}{% include "footer.html" %}{% endblock page_footer %}
```

```jinja
{# templates/footer.html #}
{% extends "invenio_app_rdm/footer.html" %}
{%- block footer_top %}
  <div class="ui container footer-top">...</div>
{%- endblock footer_top %}
```

```jinja
{# templates/frontpage.html #}
{% extends "oarepo_ui/frontpage.html" %}
{% from "macros/card.html" import render_card %}
{% set hide_footer_mandatory_publicity = True %}

{% block page_header %}{% include "header.html" %}{% endblock page_header %}
{% block grid_section %}
  <div class="hero-section">...</div>
  {{ render_card(header=_('Upload dataset'), icon='upload', url=..., button_text=...) }}
{% endblock grid_section %}
```

Two patterns worth calling out:

- **`{% set some_var = ... %}` before an `{% include %}`** is how a page
  template passes a flag to an included partial that doesn't take explicit
  arguments — `frontpage.html` sets `hide_footer_mandatory_publicity` and
  the included `footer.html` reads it via a plain `{% if
  not hide_footer_mandatory_publicity %}`. Use this instead of trying to
  pass arguments to a plain `{% include %}` (which doesn't take any).
- **`render_card(header, description, icon, url, button_text)`** in
  `templates/macros/card.html` is a real, minimal example of the
  "write your own small macro for repeated formatting logic" pattern from
  [jinjax-components.md §4](jinjax-components.md#4-plain-jinja2-macros--the-minority-pattern) —
  a good template to copy for any "repeat this same small chunk of markup
  N times with different data" need on a branding page.

**`css.html`/`javascript.html` are how a repository-wide custom component
bundle gets loaded on every page** — this is the missing link between the
`ui.components` webpack theme bundle registered in
[wiring.md](wiring.md#4-webpackpy-and-pyprojecttoml--the-actual-last-mile)'s
`pyproject.toml` example and where it actually gets included:

```jinja
{# templates/css.html #}
{% include "oarepo_ui/css.html" %}
{{ webpack['components.css'] }}
```
```jinja
{# templates/javascript.html #}
{% include "oarepo_ui/javascript.html" %}
{{ webpack['components.js'] }}
```

If you register a new repository-wide (not model-specific) React widget or
CSS bundle, this is where to add its `webpack[...]` include — chain to the
`oarepo_ui` default first (`{% include "oarepo_ui/css.html" %}`) so you
don't lose the base framework's own assets.

## 6. Config-driven template overrides (`_TEMPLATE(S)` keys)

Some templates are swapped via a config variable instead of (or alongside)
path-based override — useful when the same override needs to be toggled
per-environment, or when the target is a *list* of includes rather than one
template:

```python
# invenio.cfg — single-template form
THEME_FOOTER_TEMPLATE = "custom/myfooter.html"

# invenio.cfg — real example: a *list* of sidebar partial includes
APP_RDM_DETAIL_SIDE_BAR_TEMPLATES = [
    "invenio_app_rdm/records/details/side_bar/manage_menu.html",
    "invenio_app_rdm/records/details/side_bar/versions.html",
    ...,
    "datarepo/records/details/side_bar/licenses.html",  # replaces the default licenses partial
    "invenio_app_rdm/records/details/side_bar/citations.html",
    ...,
]
```

Not every `_TEMPLATE`-suffixed config key takes a single string — some
(like `APP_RDM_DETAIL_SIDE_BAR_TEMPLATES`) take an ordered list of template
paths to include in sequence, and swapping one entry for your own
repository-local template (as shown above) is the standard way to replace
just one section of a composed sidebar/page without overriding the whole
list.

## 7. Site copy and text

```python
# invenio.cfg
config.configure_ui(
    code="datarepo",              # a short site identifier
    name=_("Data Catch-all Repository"),
    description=_("Catch-all repository for Czech scientific data"),
)
```

`config.configure_ui(...)` (from `oarepo_config`) is the primary way to set
the site name/description shown in the header and page title. Beyond this:

- Use `_("...")`/`lazy_gettext` for **every** piece of custom template
  text, not just user-facing labels — this is what makes the string
  translatable at all (see the sibling `invenio-semantic-ui` skill's i18n
  conventions for the equivalent React-side rule).
- For text embedded in a template (buttons, hints, labels) rather than a
  config value, override the template itself (§5) — there's no separate
  central "copy" config beyond site name/description.
