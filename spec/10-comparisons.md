# JTOX Comparisons

Version: 1.0.0-draft

---

## 1. Why JTOX Exists

JTOX fills a specific gap: a JSON-based, platform-agnostic UI description format designed
for theming systems where templates are authored by one party (theme developers) and
rendered by another (a platform).

Existing solutions either assume a specific rendering target, require a custom parser,
or conflate structure with behavior.

---

## 2. Comparison Matrix

| Feature | JTOX | Liquid (Shopify) | Handlebars | MDX | JSON Schema Forms | Block Protocol |
|---|---|---|---|---|---|---|
| Format | JSON | Custom syntax | Custom syntax | Markdown + JSX | JSON | JSON |
| Parser required | No (standard JSON) | Yes | Yes | Yes | No | No |
| Platform agnostic | Yes | Web only | Web-focused | React-focused | Yes | Yes |
| Composition ($ref) | Yes | Partial (snippets) | Partial (partials) | Yes (imports) | No | Yes |
| Data binding | $bind objects | {{ variable }} | {{ variable }} | JS expressions | JSON pointers | Entity props |
| Control flow | each, conditional | for, if, unless | #each, #if | JS expressions | Conditional schemas | No |
| Style system | Identifiers (decoupled) | CSS classes (coupled) | None | CSS-in-JS | None | None |
| Settings/config | Built-in schema | settings_schema.json | None | Props | JSON Schema | Entity types |
| Validation | JSON Schema | Liquid linter | None built-in | TypeScript | JSON Schema | JSON Schema |
| Designed for theming | Yes | Yes | No | No | No | No |

---

## 3. Detailed Comparisons

### vs Shopify Liquid

Liquid is the closest analog — it's a theming language for an e-commerce platform.

**What Liquid does well:**
- Mature ecosystem with extensive documentation
- Server-side rendering by default (great performance)
- Rich filter system (date formatting, currency, string manipulation)
- Section/block architecture similar to JTOX sections/components

**Where JTOX differs:**
- Liquid is a text-based template language that requires a custom parser. JTOX is JSON —
  any language can parse it without a Liquid interpreter.
- Liquid mixes structure and presentation (`<div class="product-card">{{ product.title }}</div>`).
  JTOX separates them (structure in JSON, styles in external files).
- Liquid is web-only. JTOX is platform-agnostic.
- Liquid's `{{ }}` syntax is string interpolation. JTOX's `$bind` is a typed, validatable object.
- Liquid has no formal composition model. Snippets are include-based, not reference-based.
  JTOX's `$ref` with props passing is more structured.

**When to use Liquid instead:** If you're building on Shopify or need a mature, battle-tested
template language for web-only rendering.

### vs Handlebars / Mustache

Handlebars is a logic-less template language popular in JavaScript ecosystems.

**Where JTOX differs:**
- Handlebars is string-based (`{{#each products}}...{{/each}}`). JTOX is structured JSON.
- Handlebars has no type system. JTOX bindings can declare expected types.
- Handlebars has no built-in settings/configuration schema. JTOX has a complete settings system.
- Handlebars partials are simpler than JTOX's `$ref` + props composition.
- Handlebars is primarily a web technology. JTOX is platform-agnostic.

### vs MDX / JSX

MDX combines Markdown with JSX for component-based content.

**Where JTOX differs:**
- MDX requires a JavaScript runtime and React (or compatible framework). JTOX is pure JSON.
- MDX allows arbitrary JavaScript expressions. JTOX is intentionally limited to bindings
  and control flow — no arbitrary code execution.
- MDX is author-friendly for developers. JTOX is designed for visual editors and non-technical
  theme customization.
- MDX has no settings schema. JTOX's settings system generates editor UIs automatically.

### vs Block Protocol

The Block Protocol defines a standard for interactive blocks that can be embedded anywhere.

**Where JTOX differs:**
- Block Protocol blocks are self-contained applications (with their own logic and rendering).
  JTOX nodes are declarative descriptions rendered by an external renderer.
- Block Protocol is about interoperability between block implementations. JTOX is about
  describing UI structure for a theming system.
- Block Protocol blocks can fetch their own data. JTOX templates receive data from the
  rendering pipeline — they never fetch data themselves.

### vs JSON Schema Forms (react-jsonschema-form, etc.)

JSON Schema Forms generate form UIs from JSON Schema definitions.

**Where JTOX differs:**
- JSON Schema Forms are specifically for forms. JTOX describes any UI.
- JSON Schema Forms generate UI from data schemas. JTOX describes UI structure directly.
- JTOX's settings system is conceptually similar to JSON Schema Forms — both generate
  editor UIs from schemas. But JTOX settings are one part of a larger system.

---

## 4. The JTOX Sweet Spot

JTOX is designed for a specific use case: **platform theming systems** where:

1. Theme developers create templates
2. Site owners customize templates via settings (no code)
3. A platform renders templates with live data
4. Templates must work across different rendering contexts (editor, preview, production)
5. Templates must be validatable without rendering them
6. The platform may change its rendering technology without changing themes

If your use case matches these criteria, JTOX is a good fit. If you need a general-purpose
template language (Liquid, Handlebars), a component framework (React, Vue), or a content
format (MDX, Markdown), those tools are better suited.
