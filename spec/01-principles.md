# JTOX Principles

Version: 1.0.0-draft

---

## 1. The Seven Principles

These principles are non-negotiable. Every design decision in the JTOX specification must
satisfy all seven. When principles conflict, they are resolved in priority order (P1 highest).

### P1: Describe, Don't Prescribe

JTOX describes the **structure and intent** of a user interface. It never prescribes how
that interface is rendered, styled, or interacted with.

A JTOX document says "there is a heading with this text." It does not say "render an `<h1>`
tag" or "use a `Text` component with `fontSize: 32`." The renderer makes those decisions.

**Test:** Can a JTOX document be rendered by two completely different rendering engines
(e.g., React and Go HTML templates) without changing the document? If yes, the document
follows P1. If no, platform-specific assumptions have leaked in.

### P2: JSON Is the Only Dependency

JTOX is valid JSON. Period.

Any system that can parse JSON can parse JTOX. No special parser, no custom syntax, no
preprocessing step. A Python script, a Go service, a browser, a database JSON column — all
can store, transmit, query, and validate JTOX documents using standard JSON tooling.

**Implications:**
- No template string syntax inside values (no `{{variable}}` interpolation)
- No special characters that require escaping beyond JSON's own rules
- No comments (JSON doesn't support them; use `meta` fields instead)
- No trailing commas, no single quotes, no unquoted keys

### P3: Explicit Over Implicit

Every behavior in a JTOX document is explicitly declared. There are no defaults that change
meaning, no implicit type coercion, no magic resolution rules.

**Examples:**
- A binding is an object (`{ "$bind": "settings.title" }`), not a string that looks like
  a binding (`"{{settings.title}}"`)
- A style reference is an array of identifiers (`["hero", "primary"]`), not a class string
  (`"hero primary"`)
- A node's type is always present. There is no "default element type."
- A reference explicitly declares what it references (`{ "$ref": "/components/card" }`)

### P4: Composition Over Complexity

Complex UIs are built by composing simple, reusable templates — not by making individual
templates complex.

A section template references components. Components reference blocks. Blocks contain
elements. Each level is simple on its own. Complexity emerges from composition, not from
any single node.

**The composition hierarchy:**
```
Layout → Pages → Sections → Blocks / Components → Elements → Text
```

Each level can be authored, tested, and validated independently.

### P5: Data Model First

The JTOX node type system, binding system, and settings schema are the data model. Get
these right and renderers follow naturally. Get them wrong and no renderer can save you.

**Implications:**
- Node types are a closed set (new types require a spec version bump)
- Binding paths are typed and validatable at authoring time
- Settings declare their type, constraints, and intent — not just their value
- The schema is the contract between authors and renderers

### P6: Version Everything

Every JTOX document declares its spec version. Every theme declares its JTOX version.
Renderers declare which versions they support.

There is no "unversioned" JTOX. A document without `$jtox` is invalid.

**Version format:** Semantic versioning (`major.minor`).
- Major: breaking changes to node types, binding syntax, or composition rules
- Minor: additive changes (new node types, new attributes, new setting types)

A renderer that supports version `1.x` MUST be able to render any `1.0` through `1.x`
document. A `2.0` document MAY be incompatible with a `1.x` renderer.

### P7: Validate Before Render

A JTOX document should be fully validatable without a renderer. JSON Schema definitions
exist for every node type, every binding expression, every settings variable.

**Validation layers:**
1. **Structural:** Is this valid JSON? Does it have `$jtox` and `type`?
2. **Type:** Is the `type` value in the allowed set? Are required fields present?
3. **Binding:** Do `$bind` paths reference valid scopes? Are types compatible?
4. **Composition:** Do `$ref` paths resolve to existing templates?
5. **Theme:** Does the theme package include all referenced components, sections, styles?

Layers 1-3 can be validated with JSON Schema alone. Layers 4-5 require the full theme context.

---

## 2. What JTOX Governs

| JTOX Defines | JTOX Does NOT Define |
|---|---|
| Node types and their properties | How nodes are rendered to pixels |
| Data binding syntax and scopes | How data is fetched or stored |
| Style reference format | What styles look like (CSS, native, etc.) |
| Composition rules ($ref) | How references are resolved at runtime |
| Settings schema and types | How settings UI is presented to users |
| Control flow semantics (each, if) | How iteration/conditionals are optimized |
| Theme package structure | How themes are distributed or installed |
| Validation rules | How validation errors are reported |

---

## 3. Terminology

| Term | Definition |
|---|---|
| **Document** | A single JTOX JSON object describing a UI tree (a section, component, page, etc.) |
| **Node** | A single object in the JTOX tree. Every node has a `type`. |
| **Element** | A node that renders a visible UI element (heading, container, image, etc.) |
| **Text** | A leaf node containing text content with optional inline formatting |
| **Reference** | A node that includes another JTOX document via `$ref` |
| **Control** | A node that controls rendering flow (iteration, conditional) |
| **Root** | A structural node that defines a top-level unit (section, page, layout) |
| **Binding** | A `$bind` object that resolves to a value at render time |
| **Setting** | A configurable value declared in a settings schema |
| **Style** | A named visual treatment referenced by nodes |
| **Theme** | A package of JTOX documents, settings, and styles that define a complete UI |
| **Renderer** | Any system that interprets JTOX documents and produces visible output |
| **Author** | A person or tool that creates JTOX documents |

---

## 4. Design Decisions Log

Significant design decisions and their rationale.

### DD-001: `$bind` objects instead of string interpolation

**Decision:** Data bindings use `{ "$bind": "path" }` objects, not `{{path}}` strings.

**Rationale:**
- String interpolation requires regex parsing — fragile, language-specific, hard to validate
- `$bind` objects are standard JSON — any JSON Schema can validate them
- Objects can carry metadata (type hints, default values, transforms)
- No ambiguity: a `$bind` object is always a binding, a string is always a literal
- Tooling (editors, linters, validators) can find all bindings with a simple JSON query

**Tradeoff:** More verbose than `{{path}}`. A heading with a bound title is:
```json
{ "type": "heading", "level": 1, "children": [{ "type": "text", "content": { "$bind": "settings.title" } }] }
```
Instead of:
```json
{ "type": "h1", "text": "{{$settings.title}}" }
```
The verbosity is intentional — it eliminates ambiguity and enables tooling.

### DD-002: Semantic element types instead of HTML tags

**Decision:** Elements use semantic types (`heading`, `container`, `image`) not HTML tags
(`h1`, `div`, `img`).

**Rationale:**
- HTML tags are web-specific. Native platforms don't have `<div>` or `<span>`.
- Semantic types convey intent: "this is a heading" vs "this is an h1 tag."
- Renderers map semantic types to platform elements. Web: `heading` → `<h1>`. Native: `heading` → `Text` with heading style.
- HTML tags are available as an optional `tag` override for web-specific rendering.

**Tradeoff:** Theme authors familiar with HTML must learn the semantic vocabulary. The
vocabulary is small (< 25 types) and maps intuitively to HTML concepts.

### DD-003: Styles as identifiers, not definitions

**Decision:** Nodes reference styles by identifier (`"styles": ["hero", "primary"]`).
Style definitions live outside JTOX documents.

**Rationale:**
- Inline styles couple the document to a styling technology (CSS, StyleSheet, etc.)
- Style identifiers are platform-agnostic — the renderer maps them to whatever system it uses
- Separation enables style themes, variants, and overrides without changing JTOX documents
- Style definitions can be CSS files, JSON style objects, native stylesheets — JTOX doesn't care

**Tradeoff:** You can't look at a JTOX document and see what it looks like. You need the
style definitions alongside it. This is intentional — JTOX is structure, not presentation.
