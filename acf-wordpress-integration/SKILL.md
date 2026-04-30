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

**Stack:** WordPress + ACF Pro · Polylang (multilingual) · Timber/Twig templating · HTML layouts by `<section>`

---

## acf-json

Field groups live in `wp-content/themes/{theme}/acf-json/group_*.json`. One file per group.

**Rule: bump `modified` to `date +%s` on every JSON edit.** ACF shows the Sync button only when file `modified` > DB value.

```json
{ "key": "group_home_page", "title": "Home Page", "modified": 1777580708 }
```

**Sync workflow:** edit JSON → update `modified` → WP admin → Custom Fields → Sync.

### JSON snippets

```json
// Tab
{ "key": "field_tab_hero", "label": "Hero", "name": "", "type": "tab", "placement": "top", "endpoint": 0 }

// Text field
{ "key": "field_hero_title", "label": "Hero Title", "name": "hero_title", "type": "text",
  "instructions": "Main heading.", "default_value": "Professional IT Solutions", "required": 0 }

// Repeater
{ "key": "field_services_items", "label": "Services", "name": "services_items", "type": "repeater",
  "sub_fields": [
    { "key": "field_service_title", "name": "service_title", "type": "text", "parent_repeater": "field_services_items" }
  ]
}
```

---

## Step 1 — Analyze the Layout

Each `<section>` → one **Tab**. Map elements to field types:

| Layout element | ACF type | Notes |
|---|---|---|
| `<h1>`–`<h6>`, `<span>` | Text | Default = tag text |
| `<p>` simple | Textarea | Default = tag text |
| `<p>` with HTML | Wysiwyg / Textarea | Note allowed tags in Instructions |
| `<img>` | Image | Instructions = dimensions |
| `<a href>` | Text (url) + Text (label) | — |
| Repeating blocks | Repeater | `copy_once` in Polylang |
| SVG icon | Textarea / Image | Textarea for inline SVG |
| Toggle | True/False | Default = 1 or 0 |
| Color | Color Picker | — |
| File download | File | Instructions = formats |

---

## Step 2 — Field Group Structure

```
Field Group: "Page Name"
├── Tab: Hero          → hero_title [Text], hero_subtitle [Textarea], hero_btn_label [Text], hero_btn_url [Text]
├── Tab: About         → about_title [Text], about_text [Textarea], about_image [Image]
└── Tab: Services      → services_items [Repeater] → service_icon [Image], service_title [Text], service_text [Textarea]
```

**Naming:** `snake_case` · prefix = section name (`hero_`, `about_`) · repeaters suffix `_items` / `_list`

---

## Step 3 — Default Values

- **Text / Textarea / True-False:** always set from the HTML layout copy.
- **Image:** no default.
- **Repeater:** no default on the repeater itself; set defaults on child Text/Textarea fields.

---

## Step 4 — Polylang

- Regular fields (Text, Image, etc.): no extra config — Polylang duplicates on translation create.
- **Repeater fields:** set sync to `copy_once` (copy once on first translation, then independent).

---

## Step 5 — Instructions

Always add Instructions for:
- Image → `Recommended size: 1200×600 px, JPG/WebP`
- Textarea with HTML → `Basic HTML supported: &lt;b&gt;, &lt;br&gt;, &lt;a&gt;`
- URL fields → `Full URL including https:// or relative path (/contact)`

> **Escape HTML in `instructions` JSON.** ACF renders it as HTML — `<b>` becomes a real tag and breaks the admin UI. Write `&lt;b&gt;` in the JSON string. `default_value` is not affected and keeps real HTML; render it in Twig with `|raw`.

---

## Step 6 — Twig (Timber)

```twig
{# Text #}
<h1>{{ post.meta('hero_title') }}</h1>

{# HTML field #}
<p>{{ post.meta('hero_lead')|raw }}</p>

{# Image #}
{% set img = post.meta('about_image') %}
{% if img %}<img src="{{ img.url }}" alt="{{ img.alt }}">{% endif %}

{# Repeater — always guard with {% if %} #}
{% if post.meta('services_items') %}
  {% for item in post.meta('services_items') %}
    <h3>{{ item.service_title }}</h3>
  {% endfor %}
{% endif %}

{# True/False #}
{% if post.meta('show_banner') %}<div class="banner">…</div>{% endif %}
```

---

## Pre-delivery Checklist

- [ ] Every `<section>` → separate Tab
- [ ] All text fields have Default Values from the layout
- [ ] Repeater fields → `copy_once` in Polylang
- [ ] Image fields have dimension Instructions
- [ ] Repeater loops wrapped in `{% if post.meta(...) %}`
- [ ] Field names: `snake_case` with section prefix
- [ ] Field Group location assigned to correct page template
- [ ] `modified` timestamp updated to current Unix timestamp
- [ ] ACF Sync run in WordPress admin
- [ ] Rendered and checked in all language versions