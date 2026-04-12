# JTOX Bindings

Version: 1.0.0-draft

---

## 1. Overview

Bindings connect JTOX templates to dynamic data. A binding is a `$bind` object that
resolves to a value at render time. Bindings are the ONLY mechanism for dynamic values
in JTOX — there is no string interpolation, no expression language, no inline code.

```json
{ "$bind": "settings.title" }
```

A `$bind` object can appear anywhere a literal value can appear:
- As a text node's `content`
- As an attribute value
- As a control flow `source` or `test`
- As a prop value passed to a component

---

## 2. The `$bind` Object

### Minimal Form

```json
{ "$bind": "scope.path.to.value" }
```

### Full Form

```json
{
  "$bind": "scope.path.to.value",
  "type": "string",
  "default": "Untitled",
  "transform": "uppercase"
}
```

### Properties

| Property | Required | Type | Description |
|---|---|---|---|
| `$bind` | Yes | string | Dot-notation path with scope prefix |
| `type` | No | string | Expected value type: `string`, `number`, `boolean`, `array`, `object` |
| `default` | No | any | Fallback value if the path resolves to `null` or `undefined` |
| `transform` | No | string \| object \| array | Transform(s) to apply to the resolved value (see §5) |

---

## 3. Binding Scopes

Every `$bind` path starts with a **scope prefix** that tells the renderer where to look
for the value.

| Scope | Prefix | Description | Available In |
|---|---|---|---|
| Settings | `settings.*` | The current template's settings values | Sections, components, blocks, overlays |
| Props | `props.*` | Data passed from the parent template or `ui.open` data | Components, blocks, overlays |
| Scope | `scope.*` | Loop variables from `each` nodes | Children of `each` nodes |
| Theme | `theme.*` | Global theme settings | All templates |
| Context | `context.*` | Renderer-provided context (page URL, locale, etc.) | All templates |
| State | `state.*` | Local UI state declared on ancestor nodes | Descendants of state-declaring nodes |
| Self | `self.*` | Element's runtime value | Action payloads with `change`/`blur` triggers |
| Computed | `computed.*` | Script-derived reactive values (see [Scripts](12-scripts.md)) | Templates with scripts |

### Settings Scope

Resolves against the settings schema of the current section, component, or block.

```json
// In a section with settings: { "title": { "type": "text", "value": "Welcome" } }
{ "$bind": "settings.title" }
// Resolves to: "Welcome"
```

### Props Scope

Resolves against data passed from a parent template via the `props` property on a
reference node.

```json
// Parent passes: "props": { "product": { "$bind": "scope.item" } }
// Inside the component:
{ "$bind": "props.product.title" }
// Resolves to the product's title
```

### Scope Scope

Resolves against variables created by `each` control flow nodes.

```json
{
  "type": "each",
  "source": { "$bind": "props.products" },
  "as": "product",
  "children": [
    {
      "type": "text",
      "content": { "$bind": "scope.product.title" }
    },
    {
      "type": "text",
      "content": { "$bind": "scope.$index" }
    }
  ]
}
```

| Variable | Description |
|---|---|
| `scope.{as}` | The current iteration item |
| `scope.$index` | Zero-based iteration index |
| `scope.$first` | `true` if this is the first item |
| `scope.$last` | `true` if this is the last item |
| `scope.$count` | Total number of items in the source |
| `scope.$even` | `true` when `$index` is even |
| `scope.$odd` | `true` when `$index` is odd |

### Theme Scope

Resolves against global theme settings (colors, fonts, spacing, etc.).

```json
{ "$bind": "theme.colors.primary" }
```

Theme settings are defined in the theme's global settings file and are available to all
templates in the theme.

### Context Scope

Resolves against renderer-provided context. This is the escape hatch for platform-specific
data that doesn't fit into settings or props.

```json
{ "$bind": "context.page.url" }
{ "$bind": "context.locale" }
{ "$bind": "context.user.authenticated" }
```

**Context is renderer-defined.** The JTOX spec does not prescribe what's in context.
Renderers SHOULD document their context shape.

### State Scope

Resolves against local UI state declared on ancestor nodes via the `state` property.

```json
{ "$bind": "state.quantity" }
{ "$bind": "state.expanded" }
```

State is scoped to the declaring node and its descendants. See [Actions & State](11-actions.md)
for the full state system, including declaration, scoping rules, and state actions.

| Variable | Description |
|---|---|
| `state.{key}` | The current value of the state variable |

State bindings are dynamic — they update when state actions modify the value. The renderer
is responsible for re-rendering affected nodes when state changes.

### Self Scope

A special scope available only in action payloads with `"change"` or `"blur"` triggers.
Resolves to the element's current runtime value.

```json
{ "$bind": "self.value" }
```

| Variable | Description |
|---|---|
| `self.value` | The current value of the form element (input, select, textarea) |

`self.value` is only meaningful on form elements (`input`, `select`, `textarea`) with
`"change"` or `"blur"` triggers. On other elements or triggers, it resolves to `null`.
See [Actions & State](11-actions.md) for details.

---

## 4. Path Resolution

Paths use dot notation to traverse nested objects:

```
settings.hero.title          → settings["hero"]["title"]
props.product.price.amount   → props["product"]["price"]["amount"]
scope.item.media[0].url      → scope["item"]["media"][0]["url"]
```

### Array Access

Array elements are accessed with bracket notation:

```json
{ "$bind": "props.product.images[0].url" }
```

### Resolution Rules

1. Split the path by `.` (dots) and `[n]` (brackets)
2. Start at the scope root (settings, props, scope, theme, context, state, or self)
3. Traverse each segment
4. If any segment resolves to `null` or `undefined`:
   - If `default` is specified, return the default value
   - Otherwise, return `null`
5. Apply `transform` if specified
6. Return the resolved value

**A binding MUST NOT throw an error on a missing path.** Missing paths resolve to `null`
(or the default value). This is critical for resilient rendering — a missing data field
should not crash the page.

---

## 5. Transforms

Transforms are named, stateless functions that modify a resolved binding value. Transforms
can accept named arguments and can be chained.

### 5.1 Transform Syntax

Three forms are supported. All are equivalent where applicable.

**String shorthand** — for zero-argument transforms:

```json
{ "$bind": "settings.title", "transform": "uppercase" }
```

**Object form** — for transforms with arguments:

```json
{
  "$bind": "settings.description",
  "transform": { "name": "truncate", "args": { "length": 120, "ellipsis": "..." } }
}
```

**Array form** — for chaining multiple transforms (applied left to right):

```json
{
  "$bind": "settings.description",
  "transform": [
    { "name": "truncate", "args": { "length": 120 } },
    { "name": "uppercase" }
  ]
}
```

Mixing forms inside an array is allowed:

```json
"transform": ["trim", { "name": "truncate", "args": { "length": 100 } }, "uppercase"]
```

### 5.2 Transform Object

| Property | Required | Type   | Description                          |
|----------|----------|--------|--------------------------------------|
| `name`   | Yes      | string | The transform identifier             |
| `args`   | No       | object | Named arguments (keys depend on the transform) |

Arguments are always named (never positional) to stay JSON-idiomatic and explicit (P3).

### 5.3 Chaining Rules

When `transform` is an array, transforms execute left to right. The output of each
transform becomes the input of the next.

```
value → transform[0] → transform[1] → ... → final value
```

If any transform in the chain receives an incompatible input type, it MUST skip (return
its input unchanged) rather than throw an error.

### 5.4 Standard Transforms

Renderers SHOULD support the standard transforms below. Renderers MAY define additional
transforms.

#### String Transforms

| Transform    | Args                                      | Description                        | Example                                      |
|-------------|-------------------------------------------|------------------------------------|----------------------------------------------|
| `uppercase`  | —                                         | Convert to uppercase               | `"hello"` → `"HELLO"`                        |
| `lowercase`  | —                                         | Convert to lowercase               | `"HELLO"` → `"hello"`                        |
| `capitalize` | —                                         | Capitalize first letter            | `"hello world"` → `"Hello world"`            |
| `titleCase`  | —                                         | Capitalize each word               | `"hello world"` → `"Hello World"`            |
| `trim`       | —                                         | Remove leading/trailing whitespace | `" hello "` → `"hello"`                      |
| `truncate`   | `length` (int), `ellipsis` (str, `"..."`) | Shorten to length including ellipsis | `"Ground control to Major Tom."` + `{length: 20}` → `"Ground control to..."` |
| `replace`    | `match` (str), `replacement` (str, `""`)  | Replace first occurrence           | `"hello world"` + `{match: "world", replacement: "JTOX"}` → `"hello JTOX"` |
| `replaceAll` | `match` (str), `replacement` (str, `""`)  | Replace all occurrences            | `"ha ha ha"` + `{match: "ha", replacement: "ho"}` → `"ho ho ho"` |
| `append`     | `value` (str)                             | Append string                      | `"hello"` + `{value: " world"}` → `"hello world"` |
| `prepend`    | `value` (str)                             | Prepend string                     | `"world"` + `{value: "hello "}` → `"hello world"` |
| `slice`      | `start` (int), `end` (int, optional)      | Extract substring                  | `"hello"` + `{start: 1, end: 4}` → `"ell"`  |
| `split`      | `separator` (str)                         | Split into array                   | `"a,b,c"` + `{separator: ","}` → `["a","b","c"]` |
| `padStart`   | `length` (int), `char` (str, `" "`)       | Pad from start                     | `"5"` + `{length: 3, char: "0"}` → `"005"`  |
| `padEnd`     | `length` (int), `char` (str, `" "`)       | Pad from end                       | `"5"` + `{length: 3, char: "0"}` → `"500"`  |
| `urlEncode`  | —                                         | Percent-encode for URLs            | `"hello world"` → `"hello%20world"`          |
| `escape`     | —                                         | HTML-entity encode                 | `"<b>hi</b>"` → `"&lt;b&gt;hi&lt;/b&gt;"`   |
| `stripHtml`  | —                                         | Remove HTML tags                   | `"<b>hi</b>"` → `"hi"`                      |

#### Number Transforms

| Transform  | Args                                        | Description                  | Example                                    |
|-----------|---------------------------------------------|------------------------------|--------------------------------------------|
| `round`    | —                                           | Round to nearest integer     | `3.7` → `4`                               |
| `floor`    | —                                           | Round down                   | `3.7` → `3`                               |
| `ceil`     | —                                           | Round up                     | `3.2` → `4`                               |
| `abs`      | —                                           | Absolute value               | `-5` → `5`                                |
| `toFixed`  | `digits` (int)                              | Fixed decimal places         | `3.14159` + `{digits: 2}` → `"3.14"`      |
| `money`    | `currency` (str, `"USD"`), `locale` (str, `"en"`) | Format as currency      | `19.99` + `{currency: "USD"}` → `"$19.99"` |

#### Array Transforms

| Transform  | Args                                  | Description                        | Example                                    |
|-----------|---------------------------------------|------------------------------------|--------------------------------------------|
| `first`    | —                                     | First element                      | `[1,2,3]` → `1`                           |
| `last`     | —                                     | Last element                       | `[1,2,3]` → `3`                           |
| `length`   | —                                     | Item count (also works on strings) | `[1,2,3]` → `3`                           |
| `sort`     | `key` (str, optional), `order` (str, `"asc"`) | Sort array                  | `[3,1,2]` + `{order: "asc"}` → `[1,2,3]` |
| `reverse`  | —                                     | Reverse order                      | `[1,2,3]` → `[3,2,1]`                     |
| `join`     | `separator` (str, `", "`)             | Join into string                   | `["a","b","c"]` + `{separator: " - "}` → `"a - b - c"` |
| `where`    | `key` (str), `value` (any)            | Filter objects by key/value        | `[{a:1},{a:2}]` + `{key: "a", value: 2}` → `[{a:2}]` |
| `map`      | `key` (str)                           | Pluck a key from each object       | `[{t:"A"},{t:"B"}]` + `{key: "t"}` → `["A","B"]` |
| `uniq`     | —                                     | Remove duplicates                  | `[1,2,2,3]` → `[1,2,3]`                   |
| `compact`  | —                                     | Remove null/undefined              | `[1,null,3]` → `[1,3]`                    |
| `flatten`  | —                                     | Flatten one level                  | `[[1,2],[3]]` → `[1,2,3]`                 |
| `size`     | —                                     | Alias for `length`                 | `[1,2,3]` → `3`                           |

#### Date Transforms

| Transform    | Args                          | Description              | Example                                         |
|-------------|-------------------------------|--------------------------|--------------------------------------------------|
| `dateFormat` | `format` (str, `"YYYY-MM-DD"`) | Format a date/timestamp  | `"2026-03-25T10:30:00Z"` + `{format: "MMM D, YYYY"}` → `"Mar 25, 2026"` |

#### Type Transforms

| Transform   | Args | Description              | Example              |
|------------|------|--------------------------|----------------------|
| `toString`  | —    | Convert to string        | `42` → `"42"`        |
| `toNumber`  | —    | Parse as number          | `"42"` → `42`        |
| `not`       | —    | Negate boolean           | `true` → `false`     |
| `json`      | —    | JSON string representation | `{a:1}` → `'{"a":1}'` |

### 5.5 Argument Defaults

When an argument has a default (shown in parentheses in the Args column), it may be
omitted. For example:

```json
// These are equivalent:
{ "name": "truncate", "args": { "length": 20 } }
{ "name": "truncate", "args": { "length": 20, "ellipsis": "..." } }
```

### 5.6 Backward Compatibility

The string shorthand (`"transform": "uppercase"`) is syntactic sugar for
`"transform": { "name": "uppercase" }`. Renderers MUST accept both forms.

All 14 transforms from JTOX 1.0.0-draft initial release remain valid and unchanged.
The expanded set is additive — no existing transforms were removed or had their behavior
altered.

---

## 6. Static vs Dynamic Bindings

| Binding Type | When Resolved | Example |
|---|---|---|
| **Static** | At publish/build time | `settings.*`, `theme.*` — values known when the page is published |
| **Dynamic** | At request/render time | `props.*`, `scope.*`, `context.*` — values depend on runtime data |
| **Client** | On the client after hydration | `state.*`, `self.*`, `computed.*` — values depend on user interaction or script logic |

This distinction matters for compilation strategies. A compiler can resolve all static
bindings at publish time and bake them into the output. Dynamic bindings remain as
placeholders that the runtime fills. Client bindings are activated after hydration.

**Renderers are not required to distinguish static from dynamic.** A runtime interpreter
resolves all bindings at render time. A compiler optimizes by pre-resolving static bindings.
The JTOX document is the same either way.

---

## 7. Binding Validation

Bindings can be validated at authoring time (before rendering):

### Level 1: Structural Validation
- Is the `$bind` value a string?
- Does it start with a valid scope prefix?
- Is the path syntactically valid (dots, brackets)?

### Level 2: Type Validation
- If `type` is specified, does the resolved value match?
- Does the binding target accept this type? (e.g., `image.src` expects a string)

### Level 3: Existence Validation
- Does the settings schema define the referenced key?
- Does the component's props interface include the referenced prop?
- Is the `scope` variable defined by an ancestor `each` node?

Levels 1-2 can be validated with JSON Schema. Level 3 requires the full theme context.

---

## 8. Examples

### Simple text binding
```json
{
  "type": "heading",
  "level": 1,
  "children": [
    { "type": "text", "content": { "$bind": "settings.title", "default": "Welcome" } }
  ]
}
```

### Image with bound source
```json
{
  "type": "image",
  "attributes": {
    "src": { "$bind": "props.product.image_url" },
    "alt": { "$bind": "props.product.title" }
  }
}
```

### Conditional with bound test
```json
{
  "type": "conditional",
  "test": { "$bind": "props.product.on_sale" },
  "children": [
    {
      "type": "container",
      "styles": ["sale-badge"],
      "children": [
        { "type": "text", "content": { "$bind": "props.product.sale_percentage", "transform": "round" } },
        { "type": "text", "content": "% OFF" }
      ]
    }
  ]
}
```

### Iteration with scope bindings
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
      "props": {
        "product": { "$bind": "scope.product" },
        "index": { "$bind": "scope.$index" },
        "isLast": { "$bind": "scope.$last" }
      }
    }
  ]
}
```

### Theme-level binding
```json
{
  "type": "container",
  "styles": ["footer"],
  "children": [
    {
      "type": "text",
      "content": { "$bind": "theme.brand.copyright_text", "default": "© 2026" }
    }
  ]
}
```
