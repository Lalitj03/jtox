# JTOX — A Declarative UI Description Language

JTOX is a JSON-based language for describing user interface trees. It is platform-agnostic,
framework-agnostic, and designed to be interpreted or compiled by any rendering target.

JTOX describes **what** appears on screen — structure, content, data bindings, and style
references — without prescribing **how** it is rendered. A React renderer, a Vue renderer,
a Go HTML template engine, or a native mobile renderer can all consume the same JTOX document.

## Specification Version

**Current:** `1.0.0-draft`

## Documentation

| Document | Purpose |
|---|---|
| [Principles](spec/01-principles.md) | Design philosophy and non-negotiable rules |
| [Node Types](spec/02-node-types.md) | The complete node type system |
| [Bindings](spec/03-bindings.md) | Data binding, transforms, and expression system |
| [Styles](spec/04-styles.md) | Style system and theming contract |
| [Composition](spec/05-composition.md) | References, templates, and composition model |
| [Control Flow](spec/06-control-flow.md) | Iteration, conditionals, and slots |
| [Settings](spec/07-settings.md) | Configuration and settings schema |
| [Themes](spec/08-themes.md) | Theme file structure and packaging |
| [Renderer Contract](spec/09-renderer-contract.md) | Requirements for conforming renderers |
| [Comparisons](spec/10-comparisons.md) | How JTOX compares to Liquid, Handlebars, MDX, etc. |
| [Actions & State](spec/11-actions.md) | Interactivity, actions, triggers, and local state |
| [Examples](examples/) | Complete working examples |
| [Schema](schema/) | JSON Schema definitions for validation |

## Quick Example

```json
{
  "$jtox": "1.0",
  "type": "section",
  "meta": { "id": "hero-banner", "name": "Hero Banner" },
  "styles": ["hero", "full-width"],
  "children": [
    {
      "type": "heading",
      "level": 1,
      "children": [
        { "type": "text", "content": { "$bind": "settings.title" } }
      ]
    },
    {
      "type": "paragraph",
      "children": [
        { "type": "text", "content": { "$bind": "settings.subtitle" } }
      ]
    },
    {
      "type": "link",
      "attributes": { "href": { "$bind": "settings.cta_url" } },
      "styles": ["button", "primary"],
      "children": [
        { "type": "text", "content": { "$bind": "settings.cta_text" } }
      ]
    }
  ]
}
```

## Design Goals

1. **Platform agnostic.** JTOX is JSON. Any language that can parse JSON can work with JTOX.
2. **Renderer agnostic.** The spec defines structure and semantics. Renderers decide implementation.
3. **Declarative.** JTOX describes UI state, not UI transitions. No imperative logic.
4. **Composable.** Templates reference other templates via `$ref`. Build complex UIs from simple parts.
5. **Validatable.** Every JTOX document can be validated against the JSON Schema definitions.
6. **Versionable.** Every document declares its spec version. Renderers handle compatibility.

## What JTOX Is Not

- Not a programming language. No functions, no variables, no loops (iteration is a node type).
- Not a styling language. JTOX references styles; it does not define them.
- Not a framework. JTOX is a data format. Frameworks consume it.
- Not HTML-in-JSON. JTOX uses semantic element types, not HTML tags.

## License

MIT
