# JTOX Themes

Version: 1.0.0-draft

---

## 1. Overview

A JTOX theme is a package of templates, settings, and styles that together define a
complete UI. Themes are the distribution unit — they are what gets installed, customized,
and rendered.

A theme contains:
- JTOX templates (sections, components, blocks, pages, headers, footers, layouts)
- Settings schemas (per-template and theme-global)
- Style definitions (platform-specific, e.g., CSS files)
- Theme metadata (name, version, author, previews)

---

## 2. Theme Directory Structure

```
{theme-name}/
├── theme.json                 # Theme metadata and global settings
├── package.json               # External dependencies (optional)
├── components/                # Reusable UI components
│   └── {component-id}/
│       ├── template.json      # JTOX document
│       ├── settings.json      # Settings schema
│       ├── props.json         # Props interface definition (documentation)
│       ├── styles.css         # Platform-specific styles
│       └── script.js          # Behavior (optional, see Scripts)
├── sections/                  # Page sections
│   └── {section-id}/
│       ├── template.json
│       ├── settings.json
│       ├── styles.css
│       └── script.js          # Behavior (optional, see Scripts)
├── overlays/                  # Overlay templates (see Overlays)
│   └── {overlay-id}/
│       ├── template.json
│       ├── settings.json
│       ├── styles.css
│       └── script.js          # Behavior (optional)
├── pages/                     # Page templates
│   └── {page-id}/
│       └── template.json
├── headers/                   # Header templates
│   └── {header-id}/
│       ├── template.json
│       ├── settings.json
│       └── styles.css
├── footers/                   # Footer templates
│   └── {footer-id}/
│       ├── template.json
│       ├── settings.json
│       └── styles.css
├── layouts/                   # Layout templates (page shell)
│   └── {layout-id}/
│       └── template.json
├── utils/                     # Shared JavaScript modules (optional)
│   └── *.js
├── styles/                    # Global theme styles
│   └── base.css               # Reset, fonts, CSS custom properties
└── previews/                  # Preview images
    └── ...
```

### Naming Conventions

| Item | Pattern | Example |
|---|---|---|
| Theme name | `lowercase-with-hyphens` | `luminous`, `rise-starter` |
| Component ID | `lowercase-with-hyphens` | `product-card`, `hero-cta` |
| Block ID | `lowercase-with-hyphens` | `price`, `add-to-cart`, `title` |
| Section ID | `lowercase-with-hyphens` | `product-grid`, `hero-banner` |
| Overlay ID | `lowercase-with-hyphens` | `image-viewer`, `search-panel` |
| Page ID | `lowercase-with-hyphens` | `home`, `product`, `landing` |
| Header/Footer ID | `lowercase-with-hyphens` | `default`, `minimal` |
| Layout ID | `lowercase-with-hyphens` | `default`, `minimal`, `sidebar` |
| Setting key | `lowercase_with_underscores` | `background_color`, `max_items` |
| Style identifier | `lowercase-with-hyphens` | `card`, `elevated`, `primary` |

---

## 3. Theme Metadata (`theme.json`)

The root `theme.json` file defines the theme's identity and global configuration:

```json
{
  "$jtox": "1.0",
  "name": "luminous",
  "displayName": "Luminous",
  "version": "1.0.0",
  "description": "A clean, modern theme for online stores",
  "author": {
    "name": "Theme Studio",
    "url": "https://example.com/themes"
  },
  "license": "MIT",
  "preview": {
    "desktop": "previews/desktop.png",
    "mobile": "previews/mobile.png",
    "thumbnail": "previews/thumbnail.png"
  },
  "presets": [
    {
      "id": "default",
      "name": "Default",
      "description": "Clean and minimal",
      "settings": {},
      "preview": "previews/preset-default.png"
    },
    {
      "id": "dark",
      "name": "Dark Mode",
      "description": "Dark background with light text",
      "settings": {
        "colors.background": "#0f172a",
        "colors.text": "#f8fafc",
        "colors.primary": "#60a5fa"
      },
      "preview": "previews/preset-dark.png"
    }
  ],
  "settings": {
    "colors": {
      "primary": { "type": "color", "intent": "style", "label": "Primary Color", "value": "#3b82f6" },
      "secondary": { "type": "color", "intent": "style", "label": "Secondary Color", "value": "#64748b" },
      "background": { "type": "color", "intent": "style", "label": "Background", "value": "#ffffff" },
      "text": { "type": "color", "intent": "style", "label": "Text Color", "value": "#1e293b" }
    },
    "typography": {
      "heading_font": { "type": "select", "intent": "style", "label": "Heading Font", "value": "Inter", "options": [
        { "label": "Inter", "value": "Inter" },
        { "label": "Playfair Display", "value": "Playfair Display" },
        { "label": "Space Grotesk", "value": "Space Grotesk" }
      ]},
      "body_font": { "type": "select", "intent": "style", "label": "Body Font", "value": "Inter", "options": [
        { "label": "Inter", "value": "Inter" },
        { "label": "Source Sans Pro", "value": "Source Sans Pro" }
      ]}
    }
  },
  "settingGroups": [
    { "id": "colors", "label": "Colors", "icon": "palette" },
    { "id": "typography", "label": "Typography", "icon": "type" }
  ],
  "defaultLayout": "/layouts/default",
  "defaultHeader": "/headers/default",
  "defaultFooter": "/footers/default",
  "pages": [
    { "id": "home", "name": "Home", "path": "/pages/home" },
    { "id": "product", "name": "Product Detail", "path": "/pages/product" },
    { "id": "collection", "name": "Collection", "path": "/pages/collection" },
    { "id": "blog", "name": "Blog", "path": "/pages/blog" },
    { "id": "cart", "name": "Cart", "path": "/pages/cart" },
    { "id": "contact", "name": "Contact", "path": "/pages/contact" }
  ]
}
```

### Required Fields

| Field | Type | Description |
|---|---|---|
| `$jtox` | string | JTOX spec version this theme targets |
| `name` | string | Theme identifier (lowercase-with-hyphens) |
| `version` | string | Theme version (semver) |
| `settings` | object | Global theme settings schema |
| `defaultLayout` | string | `$ref` path to the default layout |
| `defaultHeader` | string | `$ref` path to the default header |
| `defaultFooter` | string | `$ref` path to the default footer |
| `pages` | array | Available page templates |

### Optional Fields

| Field | Type | Description |
|---|---|---|
| `displayName` | string | Human-readable theme name |
| `description` | string | Theme description |
| `author` | object | Author information |
| `license` | string | License identifier |
| `preview` | object | Preview image paths |
| `presets` | array | Pre-configured setting combinations |
| `settingGroups` | array | Group definitions for the editor UI |

---

## 4. Page Templates

A page template defines which sections appear on a page. Pages are lists of sections —
sections are the only root-level content nodes inside a page. Complex layouts (sidebars,
multi-column) are handled inside sections using components and blocks.

### Page Template Structure

```json
{
  "$jtox": "1.0",
  "type": "page",
  "meta": { "id": "home", "name": "Home Page" },
  "layout": "default",
  "allowedSections": [
    "/sections/hero-banner",
    "/sections/product-grid",
    "/sections/testimonials",
    "/sections/newsletter"
  ],
  "defaultSections": [
    { "ref": "/sections/hero-banner", "static": false },
    { "ref": "/sections/product-grid", "static": false },
    { "ref": "/sections/newsletter", "static": false }
  ]
}
```

| Property | Required | Type | Description |
|---|---|---|---|
| `layout` | No | string | Layout template ID to use. Defaults to theme's `defaultLayout`. |
| `allowedSections` | No | array | Section `$ref` paths that can be added to this page. If absent, all sections are allowed. |
| `defaultSections` | No | array | Sections pre-populated when a page is created from this template. |

### Static vs Dynamic Sections

Each entry in `defaultSections` can be marked as `static`:

- `"static": true` — Section is always present. Cannot be removed by the site owner.
  Used for core page functionality (e.g., the main product detail section on a product page).
- `"static": false` (default) — Section can be removed, reordered, or replaced by the
  site owner in the editor.

```json
"defaultSections": [
  { "ref": "/sections/main-product", "static": true },
  { "ref": "/sections/product-recommendations", "static": false },
  { "ref": "/sections/recently-viewed", "static": false }
]
```

### How Pages Render

When a page is rendered, the sections in the page are rendered in order inside the
page region's wrapper element (`<main>`). The layout template arranges the header,
page, and footer regions.

```
Layout (renders <div>)
  ├── Header region (renders <header>, contains header sections)
  ├── Page region (renders <main>, contains page sections)
  └── Footer region (renders <footer>, contains footer sections)
```

---

## 5. Layout Templates

A layout template defines the page shell — which header appears, which footer appears,
and how the page content is wrapped. Layouts are self-contained — they reference their
own header and footer templates via `$ref`, controlling the full page shell.

Layout styles are defined in the theme's `styles/base.css`, not in separate CSS files.

### Layout Template Structure

Layouts reference header and footer templates directly using `$ref`:

```json
{
  "$jtox": "1.1",
  "type": "layout",
  "meta": { "id": "default", "name": "Default Layout" },
  "styles": ["layout-default"],
  "children": [
    { "type": "header", "$ref": "/headers/default" },
    { "type": "slot", "name": "page" },
    { "type": "footer", "$ref": "/footers/default" },
    { "type": "slot", "name": "overlays" }
  ]
}
```

A minimal layout references a different header and omits the footer:

```json
{
  "$jtox": "1.1",
  "type": "layout",
  "meta": { "id": "minimal", "name": "Minimal Layout" },
  "styles": ["layout-minimal"],
  "children": [
    { "type": "header", "$ref": "/headers/minimal" },
    {
      "type": "main",
      "styles": ["content-area"],
      "children": [{ "type": "slot", "name": "page" }]
    },
    { "type": "slot", "name": "overlays" }
  ]
}
```

### Header/Footer Resolution

The layout is the authority for which header and footer appear:

- If the layout has a child with `type: "header"` and `$ref` → render that header
- If the layout has no header child → no header is rendered
- Same for footer
- `theme.json`'s `defaultHeader`/`defaultFooter` are only used for backward
  compatibility with legacy layouts that use slots instead of `$ref`

A blank layout with no header/footer refs produces a page with only content:

### Multiple Layouts

A theme can provide multiple layouts for different page types:

| Layout | Use Case | Header | Footer |
|---|---|---|---|
| `default` | Most pages | Full navigation | Full footer |
| `minimal` | Auth, checkout | Logo only | None |
| `full-width` | Landing pages | Full navigation | Full footer |

Page templates reference a layout by ID via the `layout` property. If omitted, the
theme's `defaultLayout` is used.

---

## 5a. Header & Footer Templates

Header and footer templates are region nodes — they define the structure of the header
and footer areas, including which sections they contain. They work identically to page
templates but render as `<header>` and `<footer>` respectively.

### Header Template

```json
{
  "$jtox": "1.0",
  "type": "header",
  "meta": { "id": "default", "name": "Default Header" },
  "styles": ["site-header", "header-sticky"],
  "allowedSections": ["/sections/announcement-bar", "/sections/main-nav"],
  "defaultSections": [
    { "ref": "/sections/main-nav", "static": true }
  ],
  "children": [
    { "type": "slot", "name": "sections" }
  ]
}
```

### Footer Template

```json
{
  "$jtox": "1.0",
  "type": "footer",
  "meta": { "id": "default", "name": "Default Footer" },
  "styles": ["site-footer"],
  "allowedSections": ["/sections/footer-links", "/sections/copyright-bar"],
  "defaultSections": [
    { "ref": "/sections/footer-links", "static": false },
    { "ref": "/sections/copyright-bar", "static": true }
  ],
  "children": [
    { "type": "slot", "name": "sections" }
  ]
}
```

Header and footer templates support the same `allowedSections`, `defaultSections`, and
`sectionRules` properties as page templates. The `styles` property controls the CSS
classes on the region's wrapper element — for example, `"header-sticky"` can apply
`position: sticky` via the theme's CSS.

---

## 6. Presets

Presets are pre-configured combinations of theme settings. They let site owners quickly
switch between visual variants without manually adjusting individual settings.

```json
{
  "id": "warm",
  "name": "Warm Tones",
  "description": "Earth tones with warm accents",
  "settings": {
    "colors.primary": "#d97706",
    "colors.secondary": "#92400e",
    "colors.background": "#fffbeb",
    "colors.text": "#451a03"
  },
  "preview": "previews/preset-warm.png"
}
```

### Preset Settings Format

Preset settings use dot-notation paths that map to the global settings schema:

```
"colors.primary" → settings.colors.primary
"typography.heading_font" → settings.typography.heading_font
```

A preset only needs to include settings that differ from the defaults. Unspecified
settings retain their default values.

---

## 7. Props Interface (`props.json`)

Components can declare their expected props in a `props.json` file:

```json
{
  "product": {
    "type": "object",
    "required": true,
    "description": "The product to display",
    "properties": {
      "title": { "type": "string" },
      "price": { "type": "number" },
      "image_url": { "type": "string" }
    }
  },
  "showPrice": {
    "type": "boolean",
    "required": false,
    "default": true,
    "description": "Whether to display the price"
  }
}
```

The props interface is a documentation and tooling file:
1. Tells section authors what data to pass to the component
2. Enables validators to check that parent templates pass required props
3. Can be used by development tools to generate dummy/preview data
4. Defines the compatibility contract for swappable components — all components
   that can fill the same role must accept the same props interface

Props are accessed in templates via `{ "$bind": "props.{key}" }`.

The `props.json` file is optional. Components without it rely on implicit prop
contracts defined by their `$bind` expressions.

Note: Blocks do NOT have `props.json`. Blocks are inline structural regions that
inherit their parent's binding scope — they access data via the parent's `props`,
`settings`, `state`, and `computed` scopes directly.

---

## 8. Theme Validation

A valid theme MUST satisfy:

### Structural Validation
- `theme.json` exists and is valid JSON
- All `$ref` paths in templates resolve to existing files
- No circular references
- All referenced style identifiers have corresponding style definitions
- All inline `block` nodes have unique `id` within their parent template
- All inline `block` nodes have `meta.name`

### Settings Validation
- All settings have required properties (`type`, `intent`, `label`, `value`)
- Default values match declared types
- Type-specific constraints are valid

### Composition Validation
- Section `placement` values are valid
- Page `allowedSections` reference existing sections
- Layout slots are filled by appropriate content types

### Cross-Reference Validation
- Components referenced by sections exist (`$ref` paths resolve)
- Style identifiers used in templates are defined in style files

---

## 9. Theme Versioning

Themes have their own version (in `theme.json`) separate from the JTOX spec version.

| Version | Meaning |
|---|---|
| Theme version (`version`) | The theme's release version. Follows semver. |
| JTOX version (`$jtox`) | The JTOX spec version the theme targets. |

A theme at version `2.3.0` targeting JTOX `1.0` means: this is the theme's third minor
release, and it uses JTOX spec version 1.0.

When the JTOX spec releases version `1.1` (additive changes), existing `1.0` themes
continue to work. When JTOX releases `2.0` (breaking changes), themes must be updated.

---

## 10. Theme Distribution

JTOX does not prescribe how themes are distributed. A theme is a directory of files.
How that directory is packaged, uploaded, installed, and managed is platform-specific.

Possible distribution mechanisms:
- ZIP archive uploaded to a theme store
- Git repository
- npm package
- API upload (files stored in database JSON fields)

The theme directory structure is the contract. The distribution mechanism is an
implementation detail.
