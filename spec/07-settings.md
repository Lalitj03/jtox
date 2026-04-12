# JTOX Settings

Version: 1.1.0-draft

---

## 1. Overview

Settings are the configuration layer of JTOX. They define what a theme author exposes
as customizable, and what a site owner can change without editing templates.

Every section, component, and block can have a settings schema. The schema defines:
- What settings exist (keys)
- What type each setting is (text, color, number, etc.)
- What the default value is
- What constraints apply (min/max, options, etc.)
- What the setting's intent is (style vs data)

Settings are the bridge between the JTOX template (structure) and the end user
(customization). A well-designed settings schema means the template never needs to be
edited — all customization happens through settings.

---

## 2. Settings Schema

A settings schema is a JSON object where each key is a setting identifier:

```json
{
  "title": {
    "type": "text",
    "intent": "data",
    "label": "Section Title",
    "description": "The main heading displayed at the top of this section",
    "value": "Featured Products",
    "required": true
  },
  "max_items": {
    "type": "number",
    "intent": "data",
    "label": "Maximum Products",
    "description": "How many products to display",
    "value": 8,
    "min": 1,
    "max": 50,
    "step": 1
  },
  "background_color": {
    "type": "color",
    "intent": "style",
    "label": "Background Color",
    "value": "#ffffff"
  }
}
```


### Setting Properties

| Property | Required | Type | Description |
|---|---|---|---|
| `type` | Yes | string | The setting type (see Section 3) |
| `intent` | Yes | string | `"style"` or `"data"` (see Section 4) |
| `label` | Yes | string | Human-readable label for the editor UI |
| `description` | No | string | Help text explaining the setting |
| `value` | Yes | any | Default value |
| `required` | No | boolean | Whether the setting must have a value (default: `false`) |
| `group` | No | string | Group identifier for organizing settings in the editor |

Additional properties depend on the setting `type` (see Section 3).

---

## 3. Setting Types

### `text`

A single-line text value.

```json
{
  "type": "text",
  "intent": "data",
  "label": "Button Text",
  "value": "Shop Now",
  "maxLength": 100,
  "placeholder": "Enter button text..."
}
```

| Property | Type | Description |
|---|---|---|
| `maxLength` | number | Maximum character count |
| `minLength` | number | Minimum character count |
| `placeholder` | string | Placeholder text for the editor input |
| `pattern` | string | Regex pattern for validation |

### `textarea`

A multi-line text value.

```json
{
  "type": "textarea",
  "intent": "data",
  "label": "Description",
  "value": "",
  "maxLength": 500,
  "rows": 4
}
```

| Property | Type | Description |
|---|---|---|
| `maxLength` | number | Maximum character count |
| `rows` | number | Suggested editor height in rows |

### `number`

A numeric value with optional range constraints.

```json
{
  "type": "number",
  "intent": "data",
  "label": "Items Per Row",
  "value": 4,
  "min": 1,
  "max": 12,
  "step": 1
}
```

| Property | Type | Description |
|---|---|---|
| `min` | number | Minimum value |
| `max` | number | Maximum value |
| `step` | number | Increment step |
| `unit` | string | Display unit (e.g., `"px"`, `"%"`, `"rem"`) |

### `range`

A numeric value displayed as a slider. Same properties as `number`.

```json
{
  "type": "range",
  "intent": "style",
  "label": "Border Radius",
  "value": 8,
  "min": 0,
  "max": 50,
  "step": 1,
  "unit": "px"
}
```

### `color`

A color value.

```json
{
  "type": "color",
  "intent": "style",
  "label": "Primary Color",
  "value": "#3b82f6",
  "format": "hex"
}
```

| Property | Type | Description |
|---|---|---|
| `format` | string | Color format: `"hex"`, `"rgb"`, `"hsl"` (default: `"hex"`) |
| `alpha` | boolean | Whether alpha/opacity is supported (default: `false`) |

### `select`

A value chosen from predefined options.

```json
{
  "type": "select",
  "intent": "data",
  "label": "Layout Style",
  "value": "grid",
  "options": [
    { "label": "Grid", "value": "grid" },
    { "label": "List", "value": "list" },
    { "label": "Carousel", "value": "carousel" }
  ]
}
```

| Property | Required | Type | Description |
|---|---|---|---|
| `options` | Yes | array | Array of `{ label, value }` objects |
| `multiple` | No | boolean | Allow multiple selections (default: `false`) |

### `boolean`

A true/false toggle.

```json
{
  "type": "boolean",
  "intent": "data",
  "label": "Show Price",
  "value": true
}
```

### `image`

An image reference.

```json
{
  "type": "image",
  "intent": "data",
  "label": "Background Image",
  "value": null,
  "accept": ["image/jpeg", "image/png", "image/webp"],
  "maxSize": 5242880
}
```

| Property | Type | Description |
|---|---|---|
| `accept` | array | Accepted MIME types |
| `maxSize` | number | Maximum file size in bytes |
| `dimensions` | object | `{ "minWidth", "maxWidth", "minHeight", "maxHeight", "aspectRatio" }` |

### `url`

A URL value.

```json
{
  "type": "url",
  "intent": "data",
  "label": "Call to Action Link",
  "value": "/products",
  "allowExternal": true
}
```

| Property | Type | Description |
|---|---|---|
| `allowExternal` | boolean | Whether external URLs are allowed (default: `true`) |

### `richtext`

Rich text content (formatted text with inline styles).

```json
{
  "type": "richtext",
  "intent": "data",
  "label": "Body Content",
  "value": [],
  "format": "slate",
  "allowedFormats": ["bold", "italic", "link", "list"]
}
```

| Property | Type | Description |
|---|---|---|
| `format` | string | Storage format of the rich text value (see table below) |
| `allowedFormats` | array | Which inline formatting options the editor exposes |

#### Rich Text Formats

The `format` property declares how the rich text value is stored. This tells the renderer
how to interpret the value and tells the editor which editing experience to provide.

| Format | Default Value | Value Shape | Description |
|---|---|---|---|
| `"plain"` | `""` | String | Plain text, no formatting. This is the default if `format` is omitted. |
| `"html"` | `""` | String | HTML string (e.g., `"<p>Hello <b>world</b></p>"`) |
| `"slate"` | `[]` | Array | Slate.js document model — array of block nodes |
| `"markdown"` | `""` | String | Markdown string (e.g., `"Hello **world**"`) |
| `"prosemirror"` | `{}` | Object | ProseMirror document model |
| `"jtox"` | `[]` | Array | JTOX-compatible content nodes — structure defined by platform |

Rules:
- `format` defaults to `"plain"` if not specified
- The `value` shape MUST match the declared `format`
- The renderer is responsible for interpreting the format and rendering it
- The editor UI provides an appropriate editing experience for the format
  (e.g., a Slate editor for `"slate"`, a Markdown editor for `"markdown"`)
- `allowedFormats` constrains which inline formatting options the editor exposes,
  regardless of the storage format
- A theme built for one format (e.g., `"slate"`) may not be directly compatible with
  a platform that uses a different format (e.g., `"prosemirror"`) — platforms should
  document which formats their renderer supports

### `datasource`

A reference to a dynamic data source (collection, product list, etc.). The datasource
setting is the bridge between JTOX templates and live platform data.

```json
{
  "type": "datasource",
  "intent": "data",
  "label": "Product Collection",
  "value": null,
  "sourceType": "collection",
  "entityTypes": ["product"]
}
```

| Property | Required | Type | Description |
|---|---|---|---|
| `sourceType` | Yes | string | `"collection"`, `"single"`, `"query"` |
| `entityTypes` | No | array | Allowed entity types: `"product"`, `"content"`, `"custom"` |

#### How Data Flows: Setting → Editor → Renderer → Template

**Step 1: Theme author declares the setting** in the section's `settings.json`:

```json
{
  "product_collection": {
    "type": "datasource",
    "intent": "data",
    "label": "Product Collection",
    "description": "Choose which products to display in this section",
    "value": null,
    "sourceType": "collection",
    "entityTypes": ["product"]
  }
}
```

**Step 2: Business owner configures it** in the editor UI. They see a collection picker,
select "Summer Sale", and the setting value becomes:

```json
{
  "value": {
    "collectionId": "uuid-of-summer-sale",
    "collectionName": "Summer Sale",
    "entityType": "product"
  }
}
```

**Step 3: Renderer fetches data** at render time. The renderer sees `sourceType: "collection"`
and fetches the collection data from the platform's data API. The fetched items are injected
into the section's `props` scope under a key matching the setting key:

```
settings.product_collection.value.collectionId = "uuid-of-summer-sale"
  → Renderer fetches: GET /api/collections/uuid-of-summer-sale/items
  → Response: [{ id, title, price, image_url, ... }, ...]
  → Injected as: props.product_collection = [array of items]
```

**Step 4: Template binds to the data:**

```json
{
  "type": "each",
  "source": { "$bind": "props.product_collection" },
  "as": "product",
  "limit": { "$bind": "settings.max_items" },
  "children": [
    {
      "type": "component",
      "$ref": "/components/product-card",
      "props": { "product": { "$bind": "scope.product" } }
    }
  ]
}
```

#### Source Types

| `sourceType` | Editor UI | Value Shape | Data Injection |
|---|---|---|---|
| `"collection"` | Collection picker | `{ collectionId, collectionName, entityType }` | Array of collection items → `props.{settingKey}` |
| `"single"` | Entity picker (one item) | `{ entityId, entityType }` | Single entity object → `props.{settingKey}` |
| `"query"` | Filter builder UI | `{ entityType, filters, sort, limit }` | Query results array → `props.{settingKey}` |

#### Query Source Type

For `sourceType: "query"`, the business owner builds a dynamic query through the editor:

```json
{
  "featured_products": {
    "type": "datasource",
    "intent": "data",
    "label": "Featured Products",
    "value": {
      "entityType": "product",
      "filters": [
        { "field": "tag", "operator": "contains", "value": "featured" },
        { "field": "price", "operator": "gte", "value": 25.00 }
      ],
      "sort": { "field": "created_at", "direction": "desc" },
      "limit": 12
    },
    "sourceType": "query",
    "entityTypes": ["product"]
  }
}
```

The renderer evaluates the query against the platform's data layer and injects results
into `props.featured_products`.

#### Entity Types

| Entity Type | Typical Fields | Example |
|---|---|---|
| `"product"` | id, title, price, images, variants, tags | E-commerce product |
| `"content"` | id, title, body, author, published_at, tags | Blog post, article |
| `"custom"` | Platform-defined | Custom data types |

The JTOX spec defines the `datasource` setting contract — the source types, value shapes,
and data injection convention (`props.{settingKey}`). The actual entity schemas, data
fetching mechanisms, and query evaluation are platform-specific.

### `list`

A repeatable group of settings. Each item in the list follows the same schema, and the
business owner can add, remove, and reorder items through the editor UI.

```json
{
  "slides": {
    "type": "list",
    "intent": "data",
    "label": "Slides",
    "value": [],
    "min": 1,
    "max": 10,
    "itemSchema": {
      "title": {
        "type": "text",
        "intent": "data",
        "label": "Slide Title",
        "value": "Slide"
      },
      "image": {
        "type": "image",
        "intent": "data",
        "label": "Slide Image",
        "value": null
      },
      "link": {
        "type": "url",
        "intent": "data",
        "label": "Slide Link",
        "value": ""
      }
    }
  }
}
```

| Property | Required | Type | Description |
|---|---|---|---|
| `itemSchema` | Yes | object | A settings schema defining the fields for each item |
| `min` | No | number | Minimum number of items (default: `0`) |
| `max` | No | number | Maximum number of items (default: no limit) |

#### How It Works

The `itemSchema` is a standard settings schema — the same format used for section or
component settings. Each key in `itemSchema` defines a field that every list item has.
The `value` in each `itemSchema` field is the default value for new items.

At runtime, `settings.{listKey}` is an array of objects. Each object has keys matching
the `itemSchema`:

```json
[
  { "title": "Summer Sale", "image": "/img/summer.jpg", "link": "/sale" },
  { "title": "New Arrivals", "image": "/img/new.jpg", "link": "/new" },
  { "title": "Best Sellers", "image": "/img/best.jpg", "link": "/best" }
]
```

#### Editor Behavior

The editor UI reads `itemSchema` and presents:
- An "Add Item" button (disabled when `max` is reached)
- A "Remove" button per item (disabled when `min` is reached)
- Drag handles for reordering items
- A collapsible group of fields per item, based on `itemSchema`

#### Template Usage

Templates iterate over the list using `each` and render blocks or elements per item:

```json
{
  "type": "each",
  "source": { "$bind": "settings.slides" },
  "as": "slide",
  "children": [
    {
      "type": "block",
      "$ref": "/blocks/slide-card",
      "props": {
        "title": { "$bind": "scope.slide.title" },
        "image": { "$bind": "scope.slide.image" },
        "link": { "$bind": "scope.slide.link" }
      }
    }
  ]
}
```

This uses only existing spec patterns — `each` for iteration, `$ref` for composition,
`$bind` for data binding. No new template concepts are needed.

#### Common Use Cases

| Use Case | List Setting | Block/Component |
|---|---|---|
| Image slider | Slides with title, image, link | `/blocks/slide-card` |
| Tabs | Tabs with label, content | `/blocks/tab-panel` |
| Accordion / FAQ | Items with question, answer | `/blocks/accordion-item` |
| Feature grid | Features with icon, title, description | `/blocks/feature-card` |
| Testimonials | Quotes with text, author, avatar | `/blocks/testimonial` |
| Navigation links | Links with label, url, icon | `/blocks/nav-link` |

#### Nesting Rules

- `itemSchema` supports all setting types EXCEPT `list` (no nested lists)
- `itemSchema` supports `datasource` — each list item can reference a data source
- The `intent` on the `list` setting itself MUST be `"data"` (lists are always data)

#### Validation

At authoring time:
- `itemSchema` must be a valid settings schema
- `min` must be ≤ `max` (if both specified)
- Default `value` must be an array
- Each item in the default `value` must conform to `itemSchema`

At configuration time:
- Array length must satisfy `min` and `max` constraints
- Each item must conform to `itemSchema` field types and constraints

---

## 4. Setting Intent

Every setting declares its **intent** — whether it affects visual presentation or data/behavior.

| Intent | Meaning | Example |
|---|---|---|
| `"style"` | Affects visual presentation | Colors, fonts, spacing, borders |
| `"data"` | Affects content or behavior | Titles, URLs, item counts, toggles |

### Why Intent Matters

The intent determines how the renderer applies the setting:

**Style intent:** The renderer translates the value into a platform-specific style mechanism.
On web, this typically means CSS custom properties. On native, it means style attributes.
The JTOX template does NOT reference style settings directly — the style layer does.

**Data intent:** The value is available in the `settings` binding scope. Templates reference
it via `{ "$bind": "settings.key" }`.

```
Style setting flow:
  Setting (intent: style) → Renderer → CSS variable / style attribute → Style definition uses it

Data setting flow:
  Setting (intent: data) → settings scope → $bind in template → rendered output
```

---

## 5. Setting Groups

Settings can be organized into groups for the editor UI:

```json
{
  "title": {
    "type": "text",
    "intent": "data",
    "label": "Title",
    "value": "Products",
    "group": "content"
  },
  "subtitle": {
    "type": "text",
    "intent": "data",
    "label": "Subtitle",
    "value": "",
    "group": "content"
  },
  "background_color": {
    "type": "color",
    "intent": "style",
    "label": "Background",
    "value": "#fff",
    "group": "appearance"
  },
  "max_items": {
    "type": "number",
    "intent": "data",
    "label": "Max Items",
    "value": 8,
    "group": "behavior"
  }
}
```

Group definitions are part of the theme metadata (see [Themes](08-themes.md)), not the
settings schema itself. The schema just references group identifiers.

---

## 6. Settings Validation

Settings are validated at two points:

### At Authoring Time (Theme Developer)
- All required properties present (`type`, `intent`, `label`, `value`)
- Type-specific constraints valid (`options` for `select`, `min/max` for `number`)
- Default `value` matches the declared `type`

### At Configuration Time (Site Owner)
- Value matches the declared `type`
- Value satisfies constraints (`min`, `max`, `maxLength`, `pattern`, `options`)
- Required settings have non-null values

---

## 7. Theme-Level Settings

In addition to per-section/component settings, themes have global settings that apply
across all templates:

```json
{
  "colors": {
    "primary": { "type": "color", "intent": "style", "label": "Primary Color", "value": "#3b82f6" },
    "secondary": { "type": "color", "intent": "style", "label": "Secondary Color", "value": "#64748b" },
    "background": { "type": "color", "intent": "style", "label": "Background", "value": "#ffffff" },
    "text": { "type": "color", "intent": "style", "label": "Text Color", "value": "#1e293b" }
  },
  "typography": {
    "heading_font": { "type": "select", "intent": "style", "label": "Heading Font", "value": "Inter", "options": [...] },
    "body_font": { "type": "select", "intent": "style", "label": "Body Font", "value": "Inter", "options": [...] },
    "base_size": { "type": "range", "intent": "style", "label": "Base Font Size", "value": 16, "min": 12, "max": 24, "unit": "px" }
  },
  "spacing": {
    "section_padding": { "type": "range", "intent": "style", "label": "Section Padding", "value": 64, "min": 16, "max": 128, "unit": "px" }
  }
}
```

Theme-level settings are accessed via the `theme` binding scope:

```json
{ "$bind": "theme.colors.primary" }
```
