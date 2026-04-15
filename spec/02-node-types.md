# JTOX Node Types

Version: 1.1.0-draft

---

## 1. Overview

Every object in a JTOX tree is a **node**. Every node MUST have a `type` field. Nodes are
organized into five categories:

| Category | Purpose | Types |
|---|---|---|
| **Element** | Visible UI elements | `container`, `heading`, `paragraph`, `span`, `link`, `button`, `image`, `video`, `icon`, `list`, `listItem`, `table`, `tableRow`, `tableCell`, `form`, `input`, `select`, `textarea`, `label`, `nav`, `main`, `aside`, `article`, `figure`, `figCaption`, `separator`, `richtext` |
| **Text** | Text content with formatting | `text` |
| **Reference** | Includes another template | `component` |
| **Structural** | Named regions within a parent | `block` |
| **Control** | Rendering flow control | `each`, `conditional`, `slot` |
| **Root** | Top-level structural units | `section`, `page`, `header`, `footer`, `layout`, `overlay`, `fragment` |

A renderer MUST support all node types in the spec version it claims to implement.

---

## 2. Common Node Properties

These properties are available on ALL node types (except where noted):

```json
{
  "type": "string (required)",
  "meta": {
    "id": "string — unique identifier within the document",
    "name": "string — human-readable name (for tooling/editors)",
    "description": "string — purpose description (for tooling/editors)"
  },
  "styles": ["array of style identifiers"],
  "attributes": { "key-value pairs — platform-specific attributes" },
  "children": ["array of child nodes"]
}
```

| Property | Required | Description |
|---|---|---|
| `type` | Yes | The node type. Must be one of the defined types. |
| `meta` | No | Metadata for tooling. Renderers MAY ignore this. |
| `styles` | No | Array of style identifier strings. See [Styles](04-styles.md). |
| `attributes` | No | Key-value pairs passed to the rendered element. Values can be literals or `$bind` objects. |
| `children` | No | Ordered array of child nodes. Not all types accept children (see per-type docs). |

---

## 3. Element Nodes

Element nodes render visible UI elements. They map to platform-specific elements at render
time (e.g., `container` → `<div>` on web, `View` on React Native).

### 3.1 Structural Elements

#### `container`

A generic grouping element. The most common structural node.

```json
{
  "type": "container",
  "styles": ["card", "elevated"],
  "children": [...]
}
```

**Web mapping:** `<div>` (default), or `<section>`, `<article>`, etc. via `tag` override.
**Accepts children:** Yes.

#### `nav`

A navigation region.

```json
{
  "type": "nav",
  "styles": ["main-nav"],
  "children": [...]
}
```

**Web mapping:** `<nav>`
**Accepts children:** Yes.

#### `main`

The primary content area of a page.

```json
{
  "type": "main",
  "children": [...]
}
```

**Web mapping:** `<main>`
**Accepts children:** Yes.

#### `aside`

Supplementary content (sidebars, callouts).

```json
{
  "type": "aside",
  "children": [...]
}
```

**Web mapping:** `<aside>`
**Accepts children:** Yes.

#### `article`

A self-contained piece of content.

```json
{
  "type": "article",
  "children": [...]
}
```

**Web mapping:** `<article>`
**Accepts children:** Yes.

#### `figure` / `figCaption`

A figure with optional caption.

```json
{
  "type": "figure",
  "children": [
    { "type": "image", "attributes": { "src": "photo.jpg", "alt": "A photo" } },
    { "type": "figCaption", "children": [{ "type": "text", "content": "Photo caption" }] }
  ]
}
```

**Web mapping:** `<figure>` / `<figcaption>`
**Accepts children:** Yes.

#### `separator`

A thematic break or visual divider.

```json
{ "type": "separator" }
```

**Web mapping:** `<hr>`
**Accepts children:** No.

#### `fragment`

An invisible grouping node. Renders its children without a wrapping element.

```json
{
  "type": "fragment",
  "children": [...]
}
```

**Web mapping:** React Fragment / no wrapper.
**Accepts children:** Yes.

### 3.2 Text Elements

#### `heading`

A heading with a level (1-6).

```json
{
  "type": "heading",
  "level": 2,
  "children": [
    { "type": "text", "content": "About Us" }
  ]
}
```

| Property | Required | Type | Description |
|---|---|---|---|
| `level` | Yes | `1-6` | Heading level. 1 is the highest. |

**Web mapping:** `<h1>` through `<h6>` based on `level`.
**Accepts children:** Yes (text nodes and inline elements).

#### `paragraph`

A block of text.

```json
{
  "type": "paragraph",
  "children": [
    { "type": "text", "content": "This is a paragraph with " },
    { "type": "text", "content": "bold text", "format": { "bold": true } },
    { "type": "text", "content": "." }
  ]
}
```

**Web mapping:** `<p>`
**Accepts children:** Yes (text nodes and inline elements).

#### `span`

An inline container for text or inline elements.

```json
{
  "type": "span",
  "styles": ["highlight"],
  "children": [
    { "type": "text", "content": "highlighted text" }
  ]
}
```

**Web mapping:** `<span>`
**Accepts children:** Yes (text nodes and inline elements).

#### `label`

A label for a form element.

```json
{
  "type": "label",
  "attributes": { "for": "email-input" },
  "children": [{ "type": "text", "content": "Email" }]
}
```

**Web mapping:** `<label>`
**Accepts children:** Yes.

### 3.3 Interactive Elements

#### `link`

A hyperlink.

```json
{
  "type": "link",
  "attributes": {
    "href": "/products",
    "target": "_blank"
  },
  "children": [
    { "type": "text", "content": "View Products" }
  ]
}
```

| Property | Required | Description |
|---|---|---|
| `attributes.href` | Yes | The link destination. Can be a `$bind` object. |

**Web mapping:** `<a>`
**Accepts children:** Yes.

#### `button`

An interactive button.

```json
{
  "type": "button",
  "attributes": {
    "action": "add-to-cart",
    "payload": { "productId": { "$bind": "props.product.id" } }
  },
  "styles": ["primary", "large"],
  "children": [
    { "type": "text", "content": "Add to Cart" }
  ]
}
```

**Note on interactivity:** JTOX does not define event handlers. The `action` attribute
declares the **intent** (what should happen). The renderer maps actions to platform-specific
event handling. See [Interactivity](#interactivity) below.

**Web mapping:** `<button>`
**Accepts children:** Yes.

### 3.4 Media Elements

#### `image`

A visual image.

```json
{
  "type": "image",
  "attributes": {
    "src": { "$bind": "props.product.image_url" },
    "alt": { "$bind": "props.product.title" },
    "width": 400,
    "height": 300
  }
}
```

| Property | Required | Description |
|---|---|---|
| `attributes.src` | Yes | Image source URL. Can be a `$bind` object. |
| `attributes.alt` | Yes | Alternative text for accessibility. Can be a `$bind` object. |

**Web mapping:** `<img>`
**Accepts children:** No.

#### `video`

A video element.

```json
{
  "type": "video",
  "attributes": {
    "src": "/videos/intro.mp4",
    "poster": "/images/intro-poster.jpg",
    "autoplay": false,
    "controls": true
  }
}
```

**Web mapping:** `<video>`
**Accepts children:** No (source elements are handled via attributes).

#### `icon`

A named icon reference.

```json
{
  "type": "icon",
  "attributes": {
    "name": "shopping-cart",
    "size": 24
  }
}
```

**Note:** Icon resolution is renderer-specific. The `name` is a semantic identifier.
The renderer maps it to an icon library (Lucide, Material Icons, SF Symbols, etc.).

**Accepts children:** No.

### 3.5 List Elements

#### `list`

An ordered or unordered list.

```json
{
  "type": "list",
  "attributes": { "ordered": false },
  "children": [
    { "type": "listItem", "children": [{ "type": "text", "content": "First item" }] },
    { "type": "listItem", "children": [{ "type": "text", "content": "Second item" }] }
  ]
}
```

**Web mapping:** `<ul>` or `<ol>` based on `ordered` attribute.
**Accepts children:** Yes (`listItem` nodes only).

#### `listItem`

A single item in a list.

**Web mapping:** `<li>`
**Accepts children:** Yes.

### 3.6 Table Elements

#### `table` / `tableRow` / `tableCell`

Tabular data.

```json
{
  "type": "table",
  "children": [
    {
      "type": "tableRow",
      "attributes": { "header": true },
      "children": [
        { "type": "tableCell", "children": [{ "type": "text", "content": "Product" }] },
        { "type": "tableCell", "children": [{ "type": "text", "content": "Price" }] }
      ]
    }
  ]
}
```

**Web mapping:** `<table>`, `<tr>`, `<td>` / `<th>` (based on `header` attribute on row).
**Accepts children:** Yes (table → tableRow → tableCell → any).

### 3.7 Form Elements

#### `form`

A form container.

```json
{
  "type": "form",
  "attributes": { "action": "submit-contact" },
  "children": [...]
}
```

**Web mapping:** `<form>`
**Accepts children:** Yes.

#### `input`

A text input field.

```json
{
  "type": "input",
  "attributes": {
    "inputType": "email",
    "placeholder": "Enter your email",
    "required": true
  }
}
```

| Property | Values | Default |
|---|---|---|
| `attributes.inputType` | `text`, `email`, `password`, `number`, `tel`, `url`, `search`, `date` | `text` |

**Web mapping:** `<input type="...">`
**Accepts children:** No.

#### `select`

A dropdown selection.

```json
{
  "type": "select",
  "attributes": {
    "options": [
      { "label": "Small", "value": "s" },
      { "label": "Medium", "value": "m" },
      { "label": "Large", "value": "l" }
    ]
  }
}
```

**Web mapping:** `<select>` with `<option>` children.
**Accepts children:** No (options are in attributes).

#### `textarea`

A multi-line text input.

```json
{
  "type": "textarea",
  "attributes": {
    "placeholder": "Your message...",
    "rows": 5
  }
}
```

**Web mapping:** `<textarea>`
**Accepts children:** No.

### 3.8 Richtext Element

#### `richtext`

A binding node that renders rich text content from a richtext setting. The template
declares where the content goes and how the wrapper is styled. The renderer reads the
bound setting value and generates the inner content — children are NOT declared in the
template.

```json
{
  "type": "richtext",
  "content": { "$bind": "settings.body" },
  "styles": ["article-content"]
}
```

| Property | Required | Type | Description |
|---|---|---|---|
| `content` | Yes | `$bind` object | Binds to a `richtext` setting. The renderer resolves the value and walks it to produce output. |
| `styles` | No | array | Style identifiers applied to the wrapper element. |

**How it works:**

1. The renderer resolves `content` via `$bind` to get the richtext setting value
2. The setting's `format` field (see [Settings](07-settings.md)) tells the renderer
   how to interpret the value (e.g., `"html"`, `"slate"`, `"jtox"`, `"markdown"`)
3. The renderer walks or converts the value into rendered output
4. The output is wrapped in an element with the declared `styles`

**Content model:** The structure of the richtext value (what nodes or markup are valid
inside it) is defined by the platform, not by JTOX. Different platforms may use different
formats and content models. The `format` field on the richtext setting is the contract
between the editor that writes the value and the renderer that reads it.

**Web mapping:** `<div>` wrapper (default). Inner HTML is produced by the renderer from
the bound value.
**Accepts children:** No (children are generated from the bound content, not declared).

---

## 4. Text Nodes

Text nodes are leaf nodes that contain text content. They cannot have children.

```json
{
  "type": "text",
  "content": "Hello, world!"
}
```

### Properties

| Property | Required | Type | Description |
|---|---|---|---|
| `content` | Yes | `string` or `$bind` object | The text content |
| `format` | No | Format object | Inline text formatting |

### Format Object

```json
{
  "format": {
    "bold": true,
    "italic": false,
    "underline": false,
    "strikethrough": false,
    "code": false,
    "superscript": false,
    "subscript": false
  }
}
```

All format properties are optional and default to `false`.

### Text with Link

```json
{
  "type": "text",
  "content": "Click here",
  "format": { "bold": true },
  "link": { "href": "/about", "target": "_blank" }
}
```

The `link` property wraps the text in a hyperlink. This is for inline links within
paragraphs. For standalone links, use the `link` element type.

### Bound Text

```json
{
  "type": "text",
  "content": { "$bind": "props.product.title" }
}
```

When `content` is a `$bind` object, the renderer resolves it at render time.

---

## 5. Reference Nodes

Reference nodes include another JTOX document by path. They are the composition mechanism.

### `component`

References a reusable UI component.

```json
{
  "type": "component",
  "$ref": "/components/product-card",
  "props": {
    "product": { "$bind": "props.item" }
  }
}
```

| Property | Required | Description |
|---|---|---|
| `$ref` | Yes | Path to the component template (theme-relative) |
| `props` | No | Data to pass to the component's scope |

Components are self-contained widgets with their own settings, styles, and optionally
scripts. They receive data via `props` from the parent section or component. Components
are reusable across sections — a `product-card` component can be used by any section
that displays products.

See [Composition](05-composition.md) for `$ref` resolution rules.

---

## 5a. Structural Nodes

Structural nodes define named regions within a parent template. They do not reference
external documents — their content is inline.

### `block`

A named structural region within a section or component template. Blocks group child
nodes under a unique `id` that external systems (editors, renderers) can use to identify,
reorder, or control visibility of template regions.

```json
{
  "type": "block",
  "id": "price",
  "meta": { "name": "Price Display" },
  "children": [
    {
      "type": "container",
      "styles": ["price-display"],
      "children": [
        { "type": "text", "content": { "$bind": "props.product.selling_price.unit_amount" } }
      ]
    }
  ]
}
```

| Property | Required | Description |
|---|---|---|
| `id` | Yes | Unique identifier within the parent template |
| `meta.name` | Yes | Human-readable name (for tooling/editors) |
| `children` | Yes | The block's content — standard JTOX nodes |

**Blocks vs Components:** Components are reusable across sections, have their own
settings/styles/scripts, and are referenced via `$ref` to external template files.
Blocks are inline structural regions within a parent template — they have no own
settings, no scripts, no separate files. They inherit their parent's binding scope
(`props`, `settings`, `state`, `computed`). Blocks are the unit of structural control
within a section or component.

**Scope:** Blocks do NOT create a new binding scope. `{ "$bind": "props.product.title" }`
inside a block resolves from the parent section or component's scope, not from any
block-specific scope.

**Nesting:** Blocks cannot contain other blocks. They contain elements, text, and
control flow nodes only. If a block needs a component, the component `$ref` should be
at the section level, not nested inside a block.

**Rendering:** A renderer walks block children the same way it walks any other children
array. The `id` and `meta.name` are metadata — the renderer MAY use them for
identification (e.g., injecting `data-block-id` attributes) but MUST render the
children regardless.

---

## 6. Control Flow Nodes

Control flow nodes affect rendering logic without producing visible elements themselves.

### `each`

Iterates over a data source and renders its template for each item.

```json
{
  "type": "each",
  "source": { "$bind": "props.products" },
  "as": "product",
  "limit": { "$bind": "settings.maxItems" },
  "children": [
    {
      "type": "component",
      "$ref": "/components/product-card",
      "props": { "product": { "$bind": "scope.product" } }
    }
  ]
}
```

| Property | Required | Type | Description |
|---|---|---|---|
| `source` | Yes | `$bind` object or array literal | The data to iterate over |
| `as` | Yes | string | Variable name for the current item (accessed via `scope.{as}`) |
| `limit` | No | number or `$bind` | Maximum items to render |
| `offset` | No | number or `$bind` | Items to skip from the start |
| `children` | Yes | array | Template to render for each item |

**Scope:** Inside an `each` node, `scope.{as}` refers to the current item, and
`scope.$index` refers to the zero-based iteration index.

See [Control Flow](06-control-flow.md) for details.

### `conditional`

Renders children only when a condition is met.

```json
{
  "type": "conditional",
  "test": { "$bind": "props.product.on_sale" },
  "children": [
    {
      "type": "span",
      "styles": ["sale-badge"],
      "children": [{ "type": "text", "content": "On Sale!" }]
    }
  ]
}
```

| Property | Required | Type | Description |
|---|---|---|---|
| `test` | Yes | `$bind` object | The value to test. Truthy = render, falsy = skip. |
| `children` | Yes | array | Nodes to render when test is truthy |
| `fallback` | No | array | Nodes to render when test is falsy |

See [Control Flow](06-control-flow.md) for truthiness rules and comparison operators.

### `slot`

A placeholder for dynamic content injection. Used in layouts and page templates.

```json
{
  "type": "slot",
  "name": "main-content"
}
```

| Property | Required | Description |
|---|---|---|
| `name` | Yes | Identifier for this slot. The parent template fills it. |
| `children` | No | Default content if the slot is not filled |

See [Composition](05-composition.md) for slot filling rules.

---

## 7. Root Nodes

Root nodes define top-level structural units. They are the entry points of JTOX documents.

### `section`

A self-contained page region (hero banner, product grid, testimonials, etc.).

```json
{
  "$jtox": "1.0",
  "type": "section",
  "meta": { "id": "product-grid", "name": "Product Grid" },
  "placement": ["page"],
  "styles": ["product-grid-section"],
  "children": [...]
}
```

| Property | Required | Description |
|---|---|---|
| `placement` | No | Array of allowed positions: `"page"`, `"header"`, `"footer"` |

**Web mapping:** `<section>`
**Accepts children:** Yes.

### `page`

A page region — a collection of sections that form the main content area. The rendering
pipeline fills the page's sections slot with the sections configured for this page.

```json
{
  "$jtox": "1.0",
  "type": "page",
  "meta": { "id": "home", "name": "Home Page" },
  "styles": ["content-area"],
  "allowedSections": ["/sections/hero", "/sections/product-grid", "/sections/testimonials"],
  "defaultSections": [
    { "ref": "/sections/hero", "static": false },
    { "ref": "/sections/product-grid", "static": false }
  ],
  "children": [
    { "type": "slot", "name": "sections" }
  ]
}
```

| Property | Required | Description |
|---|---|---|
| `allowedSections` | No | Whitelist of section `$ref` paths this page permits |
| `defaultSections` | No | Sections pre-populated when a page is created from this template |
| `sectionRules` | No | Constraints on section combinations (see [Composition](05-composition.md)) |

**Web mapping:** `<main>` (default). Override with `tag` property.
**Accepts children:** Yes (typically a single sections `slot`).

### `header`

A header region — a collection of sections that form the site header. Works identically
to `page` but renders as `<header>` and occupies the header slot in the layout.

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

| Property | Required | Description |
|---|---|---|
| `allowedSections` | No | Whitelist of section `$ref` paths this header permits |
| `defaultSections` | No | Sections pre-populated when a header is created from this template |
| `sectionRules` | No | Constraints on section combinations |

**Web mapping:** `<header>` (default). Override with `tag` property.
**Accepts children:** Yes (typically a single sections `slot`).

### `footer`

A footer region — a collection of sections that form the site footer. Works identically
to `page` and `header` but renders as `<footer>` and occupies the footer slot in the layout.

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

| Property | Required | Description |
|---|---|---|
| `allowedSections` | No | Whitelist of section `$ref` paths this footer permits |
| `defaultSections` | No | Sections pre-populated when a footer is created from this template |
| `sectionRules` | No | Constraints on section combinations |

**Web mapping:** `<footer>` (default). Override with `tag` property.
**Accepts children:** Yes (typically a single sections `slot`).

### Region Symmetry

`page`, `header`, and `footer` are all "regions" — structural containers that hold
collections of sections. They share the same properties and behavior:

| Property | page | header | footer |
|---|---|---|---|
| Default HTML tag | `<main>` | `<header>` | `<footer>` |
| `styles` | ✅ | ✅ | ✅ |
| `tag` override | ✅ | ✅ | ✅ |
| `allowedSections` | ✅ | ✅ | ✅ |
| `defaultSections` | ✅ | ✅ | ✅ |
| `sectionRules` | ✅ | ✅ | ✅ |
| Sections slot | ✅ | ✅ | ✅ |

The only difference is their semantic role and default HTML tag.

### `layout`

Wraps a page with header and footer. The outermost structural unit. A layout controls
the full page shell — which header appears, which footer appears, and how the page
content is wrapped.

Layouts reference their header and footer templates via `$ref`. This makes the layout
self-contained — it controls its own shell, similar to how Shopify layouts directly
include their header/footer markup.

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

A minimal layout can reference a different header and omit the footer entirely:

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

**Header/footer resolution:**
- If the layout has a child with `type: "header"` and `$ref` → render that header
- If the layout has no header child → no header is rendered
- Same for footer
- `theme.json`'s `defaultHeader`/`defaultFooter` are only used for backward
  compatibility with legacy layouts that use slots instead of `$ref`

The layout is the authority. A blank layout with no header/footer refs produces
a page with only content and overlays — no shell at all.

**Web mapping:** `<div>` (default). Override with `tag` property.
**Accepts children:** Yes — `$ref` nodes for header/footer, slots for page/overlays, structural wrappers.
```

### `overlay`

A hidden UI surface that appears on top of page content when triggered by a `ui.open`
action. Overlays are independently themed with their own templates, settings, styles,
and optionally scripts.

```json
{
  "$jtox": "1.0",
  "type": "overlay",
  "meta": { "id": "image-viewer", "name": "Image Viewer" },
  "styles": ["overlay-fullscreen"],
  "attributes": {
    "role": "dialog",
    "aria-label": "Image Viewer"
  },
  "children": [...]
}
```

| Property | Required | Description |
|---|---|---|
| `meta.id` | Yes | Unique identifier — the `ui.open` target. |

**Visible by default:** No. Hidden until `ui.open` is called with a matching `target`.
**Web mapping:** Renderer-specific (`<dialog>`, `<aside>`, `<div>`).
**Accepts children:** Yes (elements, components, blocks, control flow — no sections).

See [Overlays](13-overlays.md) for the full overlay specification.

---

## 8. The `tag` Override (Web Hint)

For web renderers, any element node MAY include a `tag` property to override the default
HTML tag mapping:

```json
{
  "type": "container",
  "tag": "section",
  "children": [...]
}
```

This renders as `<section>` instead of `<div>`. The `tag` property is a **renderer hint** —
non-web renderers SHOULD ignore it.

**Valid `tag` values:** Any valid HTML5 element tag name.

**Rule:** `tag` does not change the node's semantic type. A `container` with `tag: "section"`
is still a `container` for validation, binding scope, and composition purposes.

---

## 9. Interactivity Model

JTOX is declarative — it does not define event handlers, callbacks, or imperative logic.
Instead, interactive elements declare **actions** via attributes:

```json
{
  "type": "button",
  "attributes": {
    "action": "cart.add",
    "payload": {
      "productId": { "$bind": "props.product.id" },
      "quantity": { "$bind": "state.quantity" }
    }
  }
}
```

| Property | Description |
|---|---|
| `action` | A namespaced action identifier (e.g., `navigate`, `state.toggle`, `cart.add`) |
| `payload` | Data associated with the action (can contain `$bind` objects) |

The renderer is responsible for mapping action identifiers to actual behavior. A React
renderer might dispatch to a state manager. A server-side renderer might map it to a form
submission. JTOX defines the intent; the renderer defines the mechanism.

### Standard Action Namespaces

| Namespace | Actions | Description |
|---|---|---|
| `navigate` | `navigate` | URL navigation |
| `state.*` | `state.set`, `state.toggle`, `state.increment`, `state.decrement` | Local UI state |
| `ui.*` | `ui.open`, `ui.close`, `ui.toggle` | Overlay visibility control (see [Overlays](13-overlays.md)) |
| `form.*` | `form.submit`, `form.reset` | Form operations |
| `handler.*` | `handler.{name}` | Script-defined event handlers (see [Scripts](12-scripts.md)) |

See [Actions & State](11-actions.md) for the full action system, local component state,
and platform-specific action namespaces.

### Local Component State

Any element node can declare local state via the `state` property:

```json
{
  "type": "container",
  "state": {
    "quantity": { "type": "number", "default": 1 },
    "expanded": { "type": "boolean", "default": false }
  },
  "children": [...]
}
```

State variables are available via the `state` binding scope (`{ "$bind": "state.quantity" }`)
and are modified by `state.*` actions. State is scoped to the declaring node and its
descendants. See [Actions & State](11-actions.md) for details.

Renderers MAY define additional action namespaces for their platform (e.g., `cart.*` for
e-commerce). Unknown actions MUST be handled gracefully — the element renders but the
action does nothing.
