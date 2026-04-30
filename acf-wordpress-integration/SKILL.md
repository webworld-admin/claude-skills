---
name: acf-wordpress-integration
description: >
  Use this skill whenever the user needs to integrate HTML/CSS layouts into WordPress using
  Advanced Custom Fields (ACF) plugin, especially on multilingual sites powered by Polylang.
  Triggers include: "интегрировать верстку", "acf поля", "wordpress acf", "сделать поля acf",
  "интеграция страницы wordpress", "acf repeater", "acf tabs", "polylang acf", "верстка в wordpress",
  "acf field group", "сделать шаблон страницы". Always use this skill when the task involves
  mapping HTML sections/components to ACF field groups, creating field groups, writing Twig/Timber
  templates, or configuring Polylang-compatible ACF fields. Do NOT use for general WordPress
  development unrelated to ACF field creation or template integration.
---

# ACF WordPress Integration Skill

This skill covers the full process of integrating a finished HTML layout into WordPress using
Advanced Custom Fields (ACF), with multilingual support via Polylang and Timber/Twig templating.

---

## Stack & Environment

- **WordPress** + **ACF Pro** (tabs, repeaters, flexible content)
- **Polylang** — multilingual support
- **Timber** — Twig templating engine for PHP templates
- Layout: HTML/CSS structured by `<section>` tags

---

## acf-json — Working with Field Groups as Files

ACF field groups are stored as JSON files in the theme folder at `acf-json/`. This enables version
control and the ACF **Sync** feature in the WordPress admin to push file changes into the database.

### File location

```
wp-content/themes/your-theme/
└── acf-json/
    └── group_xxxxxxxxxx.json   ← one file per field group
```

### `modified` timestamp — update on every edit

**Every time you edit a JSON file, update the `modified` field** at the root of the JSON object
to a Unix timestamp greater than the current value. ACF compares this against the database record —
if `modified` in the file is higher, the Sync button appears in the admin.

Get the current Unix timestamp in terminal:

```bash
date +%s
```

The value must be a plain integer (no quotes):

```json
{
  "key": "group_home_page",
  "title": "Home Page",
  "modified": 1771700000,
  ...
}
```

**Rule: after any change to a JSON file — adding a field, renaming, changing a default value,
updating instructions — always bump `modified` to a fresh timestamp before saving.**

### Sync workflow

1. Edit the JSON file
2. Update `modified` to the current Unix timestamp
3. Commit to git
4. WordPress admin → **Custom Fields → Sync** → click Sync on the changed group
5. ACF imports the file into the database — changes are live

### JSON field structure reference

Tab field:

```json
{
  "key": "field_tab_hero",
  "label": "Hero Section",
  "name": "tab_hero",
  "type": "tab",
  "placement": "top",
  "endpoint": 0
}
```

Regular field:

```json
{
  "key": "field_hero_title",
  "label": "Hero Title",
  "name": "hero_title",
  "type": "text",
  "instructions": "Main page heading. Displayed at the top of the Hero block.",
  "default_value": "Professional IT Solutions",
  "required": 0
}
```

Repeater field with `sub_fields`:

```json
{
  "key": "field_services_items",
  "label": "Services Items",
  "name": "services_items",
  "type": "repeater",
  "sub_fields": [
    {
      "key": "field_service_title",
      "label": "Service Title",
      "name": "service_title",
      "type": "text",
      "default_value": ""
    }
  ]
}
```

---

## Step 1 — Analyze the Layout

Before creating fields:

1. Split the page by `<section>` tags — each section becomes a separate **Tab** in ACF.
2. Inside each section, identify all editable elements:
    - Text (`h1–h6`, `p`, `span`) → `Text` or `Textarea`
    - Links / buttons → `text` (url) + `text` (label), or a `link` group
    - Images → `Image`
    - Repeating blocks (cards, list items) → `Repeater`
    - Icons / SVG → `Textarea` or `Image`
    - Boolean toggles → `True/False`

---

## Step 2 — Field Group Structure

### Tab rule

Each `<section>` maps to a **Tab** field in ACF. Example structure:

```
Field Group: "Home Page"
├── Tab: Hero Section
│   ├── hero_title         [Text]
│   ├── hero_subtitle      [Textarea]
│   └── hero_button_label  [Text]
├── Tab: About Section
│   ├── about_title        [Text]
│   ├── about_text         [Textarea]
│   └── about_image        [Image]
└── Tab: Services Section
    └── services_items     [Repeater]
        ├── service_icon   [Image]
        ├── service_title  [Text]
        └── service_text   [Textarea]
```

### Field naming

- `snake_case` only, lowercase, no spaces
- Prefix = section name: `hero_`, `about_`, `services_`
- Repeater fields: suffix `_items` or `_list`

---

## Step 3 — Default Values

Always fill in `Default Value` for text fields. Pull the text directly from the HTML layout or the Figma mockup.

| Field type | Set Default Value? | Where to get the text        |
|------------|--------------------|-------------------------------|
| Text       | ✅ Yes             | From the tag in HTML          |
| Textarea   | ✅ Yes             | From the tag in HTML          |
| Image      | ❌ No              | —                             |
| Repeater   | ❌ No (on itself)  | Set defaults on child fields  |
| True/False | ✅ Yes             | Based on layout logic         |

Example:
```
Field: hero_title [Text]
Default Value: "Professional IT Solutions"   ← taken from <h1> in HTML
```

---

## Step 4 — Polylang: Translation Settings

### Regular fields (Text, Textarea, Image, etc.)

No special ACF configuration needed — Polylang duplicates content when creating a translation.
Each language edits its fields independently.

### Repeater fields

Set the synchronization behavior to **`copy_once`** — copy values once when the translation is
first created, then allow independent editing per language.

---

## Step 5 — Field Instructions (hints)

Always add `Instructions` to fields where it helps editors. Especially:

- **Image** — recommended dimensions:  
  `Recommended size: 1200×600 px, JPG/WebP format`
- **Textarea** with HTML content:  
  `Basic HTML supported: &lt;b&gt;, &lt;br&gt;, &lt;a&gt;`
- **Text** for URLs:  
  `Enter the full URL including https://`
- **Image inside a Repeater**:  
  `Card icon. Size: 64×64 px, SVG or PNG`
- **Fields tied to a specific location on the page**:  
  `Displayed in the "Benefits" block on the homepage`

> **Important — escape HTML tags inside `instructions`.** ACF renders the
> `instructions` string as HTML in the admin form, so a literal `<b>` is parsed
> as a bold tag (and an unclosed/leaked opening tag can bleed formatting onto
> neighboring labels and visually break the field UI). When you want editors to
> see tag names like `<b>`, `<br>`, `<p>`, write them as HTML entities in the
> JSON: `&lt;b&gt;`, `&lt;br&gt;`, `&lt;p&gt;`. This applies only to
> `instructions` — `default_value` keeps real HTML, and the Twig template
> renders it through `|raw` (e.g. `{{ post.meta('hero_title')|raw }}`).

---

## Step 6 — Twig Template (Timber)

### Regular field

```twig
<h1>{{ post.meta('hero_title') }}</h1>
<p>{{ post.meta('hero_subtitle') }}</p>
```

### Repeater

Always wrap in an existence check before iterating:

```twig
{% if post.meta('services_items') %}
  {% for item in post.meta('services_items') %}
    <div class="service-card">
      <img src="{{ item.service_icon.url }}" alt="{{ item.service_title }}">
      <h3>{{ item.service_title }}</h3>
      <p>{{ item.service_text }}</p>
    </div>
  {% endfor %}
{% endif %}
```

### Image field

```twig
{% set img = post.meta('about_image') %}
{% if img %}
  <img src="{{ img.url }}" alt="{{ img.alt }}" width="{{ img.width }}" height="{{ img.height }}">
{% endif %}
```

### True/False field

```twig
{% if post.meta('show_banner') %}
  <div class="banner">...</div>
{% endif %}
```

---

## Step 7 — Full Integration Example

### Source HTML

```html
<section class="hero">
  <h1>Professional IT Solutions</h1>
  <p>We help businesses grow with modern technology</p>
  <a href="/contact" class="btn">Get in touch</a>
</section>
```

### ACF fields

```
Tab: Hero Section
├── hero_title [Text]
│   Label: "Hero Title"
│   Default: "Professional IT Solutions"
│   Instructions: "Main page heading. Displayed at the top of the Hero block."
│
├── hero_subtitle [Textarea]
│   Label: "Hero Subtitle"
│   Default: "We help businesses grow with modern technology"
│   Instructions: "Short description below the heading. Recommended: 1–2 sentences."
│   Rows: 3
│
├── hero_button_label [Text]
│   Label: "Button Text"
│   Default: "Get in touch"
│
└── hero_button_url [Text]
    Label: "Button URL"
    Default: "/contact"
    Instructions: "Button link. Use a relative path (e.g. /contact) or a full URL."
```

### Twig template

```twig
<section class="hero">
  <h1>{{ post.meta('hero_title') }}</h1>
  <p>{{ post.meta('hero_subtitle') }}</p>
  <a href="{{ post.meta('hero_button_url') }}" class="btn">
    {{ post.meta('hero_button_label') }}
  </a>
</section>
```

---

## Step 8 — Pre-delivery Checklist

- [ ] Every `<section>` → separate Tab in ACF
- [ ] All text fields have a Default Value from the layout/mockup
- [ ] Repeater fields are set to `copy_once` in Polylang sync settings
- [ ] All Image fields have Instructions with recommended dimensions
- [ ] All Repeater loops are wrapped in `{% if post.meta(...) %} ... {% endif %}`
- [ ] Field names: `snake_case` with section prefix
- [ ] Field Group is assigned to the correct page template or post type
- [ ] `modified` timestamp in the JSON file updated to current Unix timestamp
- [ ] ACF Sync run in WordPress admin after JSON changes
- [ ] Rendering verified on both language versions (e.g. EN and RU)

---

## ACF Field Type Reference

| Layout element               | ACF field type   | Note                                    |
|------------------------------|------------------|-----------------------------------------|
| `<h1>`–`<h6>`, `<span>`      | Text             | Default = text from the tag             |
| `<p>` multiline              | Textarea         | Default = text from the tag             |
| `<p>` with HTML tags         | Wysiwyg / Textarea | Add allowed tags to Instructions       |
| `<img>`                      | Image            | Instructions = size in px               |
| `<a href="...">`             | Text (url)       | Instructions = URL format               |
| Repeating block (cards)      | Repeater         | copy_once in Polylang                   |
| Icon (SVG)                   | Textarea / Image | Textarea for inline SVG                 |
| Visibility toggle            | True/False       | Default = 1 (visible) or 0             |
| Accent color                 | Color Picker     | —                                       |
| Downloadable file            | File             | Instructions = allowed formats          |