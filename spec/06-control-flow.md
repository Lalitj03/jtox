# JTOX Control Flow

Version: 1.0.0-draft

---

## 1. Overview

Control flow nodes affect rendering logic without producing visible elements. They are
the ONLY mechanism for dynamic rendering behavior in JTOX. There is no expression
language, no inline conditionals, no ternary operators.

JTOX has three control flow nodes:

| Node | Purpose |
|---|---|
| `each` | Iterate over a data source |
| `conditional` | Render based on a condition |
| `slot` | Placeholder for content injection (see [Composition](05-composition.md)) |

Control flow nodes are intentionally minimal. Complex logic belongs in the data layer
(the API that provides data to the renderer), not in the template.

---

## 2. `each` — Iteration

Iterates over an array and renders its children for each item.

### Basic Usage

```json
{
  "type": "each",
  "source": { "$bind": "props.products" },
  "as": "product",
  "children": [
    {
      "type": "container",
      "styles": ["product-card"],
      "children": [
        { "type": "text", "content": { "$bind": "scope.product.title" } }
      ]
    }
  ]
}
```


### Properties

| Property | Required | Type | Description |
|---|---|---|---|
| `source` | Yes | `$bind` object or array literal | The data to iterate over |
| `as` | Yes | string | Variable name for the current item |
| `limit` | No | number or `$bind` | Maximum items to render |
| `offset` | No | number or `$bind` | Items to skip from the start |
| `empty` | No | array of nodes | Content to render when source is empty |

### Scope Variables

Inside an `each` node, the following variables are available via the `scope` binding prefix:

| Variable | Type | Description |
|---|---|---|
| `scope.{as}` | any | The current iteration item |
| `scope.$index` | number | Zero-based iteration index |
| `scope.$first` | boolean | `true` for the first item |
| `scope.$last` | boolean | `true` for the last item |
| `scope.$count` | number | Total number of items in the source |
| `scope.$even` | boolean | `true` when `$index` is even |
| `scope.$odd` | boolean | `true` when `$index` is odd |

### With Limit and Offset

```json
{
  "type": "each",
  "source": { "$bind": "props.products" },
  "as": "product",
  "limit": { "$bind": "settings.maxItems" },
  "offset": 0,
  "children": [...]
}
```

`limit` and `offset` are applied before rendering. If `source` has 20 items, `offset: 5`,
`limit: 10` renders items 5-14. The `$count` scope variable reflects the original source
length (20), not the limited count.

### Empty State

```json
{
  "type": "each",
  "source": { "$bind": "props.products" },
  "as": "product",
  "children": [
    { "type": "component", "$ref": "/components/product-card", "props": { "product": { "$bind": "scope.product" } } }
  ],
  "empty": [
    {
      "type": "container",
      "styles": ["empty-state"],
      "children": [
        { "type": "text", "content": "No products found." }
      ]
    }
  ]
}
```

The `empty` content renders when `source` resolves to an empty array, `null`, or `undefined`.

### Nested Iteration

`each` nodes can be nested. Inner loops access their own `scope` variables. To access an
outer loop's variables, the outer `as` name is preserved in scope:

```json
{
  "type": "each",
  "source": { "$bind": "props.categories" },
  "as": "category",
  "children": [
    {
      "type": "heading",
      "level": 2,
      "children": [{ "type": "text", "content": { "$bind": "scope.category.name" } }]
    },
    {
      "type": "each",
      "source": { "$bind": "scope.category.products" },
      "as": "product",
      "children": [
        {
          "type": "text",
          "content": { "$bind": "scope.product.title" }
        }
      ]
    }
  ]
}
```

**Scope shadowing:** If an inner `each` uses the same `as` name as an outer `each`, the
inner value shadows the outer. This is valid but discouraged — use distinct names.

### Rendering Rules

1. If `source` is not an array, treat it as an empty array (render `empty` content)
2. If `source` is `null` or `undefined`, render `empty` content
3. If `limit` is 0 or negative, render nothing (not even `empty`)
4. `children` is rendered once per item. All children share the same scope.
5. The renderer MUST NOT modify the source array

---

## 3. `conditional` — Conditional Rendering

Renders children only when a condition is truthy.

### Basic Usage

```json
{
  "type": "conditional",
  "test": { "$bind": "props.product.on_sale" },
  "children": [
    {
      "type": "span",
      "styles": ["sale-badge"],
      "children": [{ "type": "text", "content": "Sale!" }]
    }
  ]
}
```

### Properties

| Property | Required | Type | Description |
|---|---|---|---|
| `test` | Yes | `$bind` object | The value to evaluate |
| `operator` | No | string | Comparison operator (default: truthy check) |
| `value` | No | any | Right-hand value for comparison |
| `children` | Yes | array | Nodes to render when condition is true |
| `fallback` | No | array | Nodes to render when condition is false |

### Truthiness Rules

When no `operator` is specified, the `test` value is evaluated for truthiness:

| Value | Truthy? |
|---|---|
| `true` | Yes |
| `false` | No |
| `null` | No |
| `undefined` | No |
| `0` | No |
| `""` (empty string) | No |
| `[]` (empty array) | No |
| Any other value | Yes |

**Note:** Empty arrays are falsy. This is intentional — it allows `conditional` to check
if a collection has items without a separate length check.

### Comparison Operators

For more complex conditions, use `operator` and `value`:

```json
{
  "type": "conditional",
  "test": { "$bind": "props.product.stock" },
  "operator": "gt",
  "value": 0,
  "children": [
    { "type": "text", "content": "In Stock" }
  ],
  "fallback": [
    { "type": "text", "content": "Out of Stock" }
  ]
}
```

| Operator | Meaning | Example |
|---|---|---|
| `eq` | Equal (strict) | `stock eq 0` |
| `neq` | Not equal | `status neq "draft"` |
| `gt` | Greater than | `price gt 100` |
| `gte` | Greater than or equal | `rating gte 4` |
| `lt` | Less than | `stock lt 5` |
| `lte` | Less than or equal | `discount lte 50` |
| `contains` | Array contains value | `tags contains "featured"` |
| `startsWith` | String starts with | `title startsWith "New"` |
| `endsWith` | String ends with | `slug endsWith "-sale"` |
| `matches` | Regex match | `email matches ".*@example.com"` |
| `exists` | Value is not null/undefined | `image exists true` |
| `empty` | Value is empty ([], "", null) | `items empty true` |

### Fallback (Else)

```json
{
  "type": "conditional",
  "test": { "$bind": "context.user.authenticated" },
  "children": [
    { "type": "text", "content": { "$bind": "context.user.name" } }
  ],
  "fallback": [
    {
      "type": "link",
      "attributes": { "href": "/sign-in" },
      "children": [{ "type": "text", "content": "Sign In" }]
    }
  ]
}
```

### Nested Conditionals

Conditionals can be nested for multi-branch logic:

```json
{
  "type": "conditional",
  "test": { "$bind": "props.product.stock" },
  "operator": "gt",
  "value": 10,
  "children": [
    { "type": "text", "content": "In Stock" }
  ],
  "fallback": [
    {
      "type": "conditional",
      "test": { "$bind": "props.product.stock" },
      "operator": "gt",
      "value": 0,
      "children": [
        { "type": "text", "content": "Low Stock" }
      ],
      "fallback": [
        { "type": "text", "content": "Out of Stock" }
      ]
    }
  ]
}
```

**Recommendation:** Keep conditional nesting to 2-3 levels maximum. Deeply nested
conditionals are hard to read and maintain. If you need complex branching, consider
preparing the data in the API layer.

---

## 4. Control Flow and Performance

Control flow nodes have performance implications that renderers should be aware of:

### `each` Performance

- Large data sources (100+ items) should use `limit` to cap rendering
- Renderers SHOULD implement virtualization for large lists when possible
- The `$bind` source is evaluated once, not per-item

### `conditional` Performance

- The `test` binding is evaluated once per render cycle
- Falsy branches are NOT rendered (no hidden DOM elements)
- Renderers MAY cache conditional results within a render cycle

### Optimization Hints

JTOX does not define optimization strategies, but renderers are encouraged to:

1. Short-circuit `each` when `source` is empty (skip child tree entirely)
2. Memoize `$ref` resolution (same path = same document)
3. Pre-evaluate static bindings (`settings.*`, `theme.*`) at load time
4. Batch scope variable creation for `each` iterations
