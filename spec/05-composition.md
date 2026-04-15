# JTOX Composition

Version: 1.0.0-draft

---

## 1. Overview

Composition is how simple JTOX templates combine into complex UIs. A section references
components. A page arranges sections. A layout wraps pages with headers and footers. Each
level is a standalone JTOX document that references others via `$ref`.

The composition model has three mechanisms:
1. **References** (`$ref`) — include another template
2. **Slots** — placeholder points that parent templates fill
3. **Section Rules** — governance constraints on which sections can appear where

---

## 2. References (`$ref`)

A reference node includes another JTOX document by path.

```json
{
  "type": "component",
  "$ref": "/components/product-card",
  "props": {
    "product": { "$bind": "scope.item" }
  }
}
```

### `$ref` Path Format

Paths are **theme-relative** and use forward slashes:

```
/components/{component-id}
/sections/{section-id}
/headers/{header-id}
/footers/{footer-id}
/pages/{page-id}
/layouts/{layout-id}
/overlays/{overlay-id}
```

| Path Segment | Resolves To |
|---|---|
| `/components/product-card` | `theme/components/product-card/template.json` |
| `/sections/hero-banner` | `theme/sections/hero-banner/template.json` |
| `/overlays/image-viewer` | `theme/overlays/image-viewer/template.json` |

Note: Blocks are NOT referenced via `$ref`. Blocks are inline structural nodes
within their parent template. See [Node Types §5a](02-node-types.md).

### Resolution Rules

1. The `$ref` path is relative to the theme root
2. The renderer resolves the path to a JTOX document
3. The referenced document is rendered in place of the reference node
4. Props from the reference node are available in the referenced document's `props` scope
5. If the path cannot be resolved, the renderer SHOULD:
   - Log a warning
   - Render nothing (skip the node)
   - NOT crash

### Props Passing

When a reference node includes `props`, those values become available in the referenced
document's `props` binding scope:

```json
// Parent: section template
{
  "type": "each",
  "source": { "$bind": "props.products" },
  "as": "item",
  "children": [
    {
      "type": "component",
      "$ref": "/components/product-card",
      "props": {
        "product": { "$bind": "scope.item" },
        "showPrice": true,
        "currency": { "$bind": "settings.currency" }
      }
    }
  ]
}

// Child: /components/product-card/template.json
{
  "$jtox": "1.0",
  "type": "fragment",
  "children": [
    {
      "type": "heading",
      "level": 3,
      "children": [{ "type": "text", "content": { "$bind": "props.product.title" } }]
    },
    {
      "type": "conditional",
      "test": { "$bind": "props.showPrice" },
      "children": [
        {
          "type": "text",
          "content": { "$bind": "props.product.price" }
        }
      ]
    }
  ]
}
```

### Circular Reference Prevention

Circular references (`A → B → A`) are invalid. Validators MUST detect and reject them.
Renderers SHOULD enforce a maximum reference depth (recommended: 10 levels).

---

## 3. Slots

Slots are named placeholders in a template that parent templates fill with content.

### Defining a Slot

```json
{
  "$jtox": "1.1",
  "type": "layout",
  "children": [
    { "type": "slot", "name": "page" },
    { "type": "slot", "name": "overlays" }
  ]
}
```

### Filling a Slot

The system that assembles the final page fills slots by name. This is NOT done in JTOX
documents themselves — it's done by the rendering pipeline.

### Layout Composition: `$ref` + Slots

Layouts use a combination of `$ref` (for header/footer) and slots (for page content
and overlays). The layout directly references which header and footer to use, while
the page content is injected via a slot:

```json
{
  "$jtox": "1.1",
  "type": "layout",
  "children": [
    { "type": "header", "$ref": "/headers/default" },
    { "type": "slot", "name": "page" },
    { "type": "footer", "$ref": "/footers/default" },
    { "type": "slot", "name": "overlays" }
  ]
}
```

The rendering pipeline:
```
1. Load layout template
2. If layout has header $ref → resolve and render header sections
   If no header $ref → no header rendered
3. Render page sections → fill layout's "page" slot
4. If layout has footer $ref → resolve and render footer sections
   If no footer $ref → no footer rendered
5. Render overlays → fill layout's "overlays" slot
6. Render the assembled tree
```

If the layout uses slots instead of `$ref` (backward compatibility), the pipeline
falls back to `theme.json`'s `defaultHeader`/`defaultFooter`.

### Default Slot Content

A slot MAY have default children that render when the slot is not filled:

```json
{
  "type": "slot",
  "name": "sidebar",
  "children": [
    {
      "type": "paragraph",
      "children": [{ "type": "text", "content": "No sidebar content" }]
    }
  ]
}
```

### Slot Rules

1. Slot names MUST be unique within a document
2. Slot names use the pattern: `[a-z][a-z0-9-]*`
3. Unfilled slots with no default content render nothing
4. Slots cannot be nested (a slot inside a slot is invalid)

---

## 4. The Composition Hierarchy

JTOX documents compose in a defined hierarchy:

```
Layout (outermost, renders <div>)
  ├── Header (renders <header>, collection of sections)
  │   ├── Section A (e.g., announcement-bar)
  │   └── Section B (e.g., main-nav)
  ├── Page (renders <main>, collection of sections)
  │   ├── Section 1 (placement: ["page"])
  │   ├── Section 2 (placement: ["page"])
  │   └── Section N
  ├── Footer (renders <footer>, collection of sections)
  │   ├── Section X (e.g., footer-links)
  │   └── Section Y (e.g., copyright-bar)
  └── Overlays (hidden until ui.open)
      ├── Overlay A (e.g., image-viewer)
      └── Overlay B (e.g., search-panel)

Section / Overlay
  ├── Elements (container, heading, paragraph, etc.)
  ├── Blocks (inline named regions — reorderable/toggleable by editors)
  ├── Component references ($ref to /components/*)
  └── Control flow (each, conditional)

Component
  ├── Elements
  ├── Blocks (inline named regions)
  ├── Text nodes
  └── Control flow

Block (inline, parent-scoped)
  ├── Elements
  ├── Text nodes
  └── Control flow
```

Header, page, and footer are all "regions" — structural containers that hold collections
of sections. They share the same properties (`allowedSections`, `defaultSections`,
`sectionRules`) and differ only in their semantic role and default HTML tag.

Overlays are peers of regions in the layout but differ in visibility — they are hidden
by default and shown via `ui.open` actions. See [Overlays](13-overlays.md).

### Nesting Rules

| Parent | Can Contain |
|---|---|
| Layout | Slots (filled with header, page, footer regions and overlays) |
| Header | Slots (filled with sections) |
| Page | Slots (filled with sections) |
| Footer | Slots (filled with sections) |
| Section | Elements, blocks (inline), components ($ref), control flow |
| Overlay | Elements, blocks (inline), components ($ref), control flow |
| Component | Elements, blocks (inline), text, control flow |
| Block | Elements, text, control flow |
| Element | Elements, text, control flow |
| Text | Nothing (leaf node) |

### What Cannot Reference What

| Source | Cannot Reference |
|---|---|
| Component | Sections, overlays, pages, layouts |
| Block | Components, sections, overlays, pages, layouts (blocks are inline, no $ref) |
| Section | Pages, layouts, overlays |
| Overlay | Pages, layouts, sections, other overlays |

These constraints prevent circular composition and maintain the hierarchy.

---

## 5. Section Placement

Sections declare where they can be placed via the `placement` property:

```json
{
  "$jtox": "1.0",
  "type": "section",
  "placement": ["page", "header"],
  "children": [...]
}
```

| Placement | Meaning |
|---|---|
| `"page"` | Can be placed in the main content area |
| `"header"` | Can be placed in the header slot |
| `"footer"` | Can be placed in the footer slot |

A section with `"placement": ["page", "header"]` can be used in either position.
A section with no `placement` property defaults to `["page"]`.

The rendering pipeline MUST enforce placement constraints — a footer-only section
cannot be placed in the page content area.

---

## 6. Section Rules (Page Governance)

Page templates can constrain which sections appear and how they combine.

### Allowed Sections

A whitelist of sections this page permits:

```json
{
  "$jtox": "1.0",
  "type": "page",
  "meta": { "id": "home" },
  "allowedSections": [
    "/sections/hero-banner",
    "/sections/product-grid",
    "/sections/testimonials",
    "/sections/newsletter-signup"
  ]
}
```

If `allowedSections` is present, only listed sections can be added to this page.
If absent, any section with `placement: ["page"]` is allowed.

### Section Combination Rules

```json
{
  "$jtox": "1.0",
  "type": "page",
  "meta": { "id": "blog" },
  "sectionRules": [
    {
      "rule": "oneOf",
      "sections": ["/sections/blog-grid", "/sections/blog-list"],
      "description": "Choose either grid or list layout, not both"
    },
    {
      "rule": "requires",
      "section": "/sections/blog-sidebar",
      "requires": "/sections/blog-list",
      "description": "Sidebar only works with list layout"
    },
    {
      "rule": "maxCount",
      "section": "/sections/hero-banner",
      "max": 1,
      "description": "Only one hero banner per page"
    }
  ]
}
```

| Rule | Meaning |
|---|---|
| `oneOf` | Only one of the listed sections can be active at a time |
| `requires` | Section A can only be used if section B is also present |
| `maxCount` | Maximum number of instances of this section on the page |
| `minCount` | Minimum number of instances (section is required) |

These rules are enforced by the editor UI (to guide the user) and validated at publish
time (to prevent invalid configurations).
