# JTOX Actions & State

Version: 1.0.0-draft

---

## 1. Overview

JTOX is declarative — templates describe structure and intent, not behavior. The action
system is how templates express interactive intent without prescribing implementation.

An action says "the user wants to do this." The renderer decides how to make it happen.

This document defines:
- The action model (how templates declare interactive intent)
- Namespaced action identifiers (standard and platform-specific)
- Local component state (client-side UI state managed by the renderer)
- The `state` binding scope

---

## 2. The Action Model

Interactive elements declare actions via `attributes`:

```json
{
  "type": "button",
  "attributes": {
    "action": "navigate",
    "payload": { "href": "/products" }
  },
  "children": [{ "type": "text", "content": "Shop Now" }]
}
```

| Property | Required | Type | Description |
|---|---|---|---|
| `action` | Yes | string | Action identifier (dot-notation namespace) |
| `payload` | No | object | Data associated with the action. Values can be literals or `$bind` objects. |
| `trigger` | No | string | When the action fires. Defaults to `"click"`. See Section 2.1. |

The renderer maps action identifiers to platform-specific behavior. A React renderer
might dispatch to a state manager. A server-side renderer might generate a form submission.
JTOX defines the intent; the renderer defines the mechanism.

### 2.1 Action Triggers

By default, actions fire on `click` (user clicks or taps the element). The `trigger`
property overrides this with a different semantic event:

```json
{
  "type": "select",
  "attributes": {
    "action": "state.set",
    "payload": { "key": "selectedSize", "value": { "$bind": "self.value" } },
    "trigger": "change",
    "options": [
      { "label": "Small", "value": "s" },
      { "label": "Medium", "value": "m" },
      { "label": "Large", "value": "l" }
    ]
  }
}
```

#### Standard Triggers

| Trigger | Meaning | Common Elements | Renderer Mapping (Web) |
|---|---|---|---|
| `"click"` | User clicks or taps (default) | button, link, container, image | `onClick` |
| `"change"` | Element value changes | select, input | `onChange` |
| `"submit"` | Form is submitted | form | `onSubmit` |
| `"mount"` | Element enters the page | any | `useEffect` on mount |
| `"hover"` | User hovers over element | any | `onMouseEnter` |
| `"blur"` | Element loses focus | input, select, textarea | `onBlur` |

Triggers are semantic — they describe the user intent, not the DOM event. A `"change"`
trigger on a `select` means "the user picked a different option." The renderer maps it
to the appropriate platform event (`onChange` in React, `change` event in vanilla JS,
value observer in native mobile).

#### Rules

- If `trigger` is omitted, the action fires on `"click"`
- If `trigger` is an unknown value, the renderer SHOULD ignore the action and log a warning
- `"mount"` fires once when the element first renders — useful for analytics tracking,
  lazy data loading, or initial state setup
- `"hover"` is a hint — touch devices may not support hover. Renderers SHOULD provide
  a fallback (e.g., treat as click on mobile) or ignore gracefully
- `"submit"` on a `form` element prevents the default browser form submission and
  fires the action instead

#### The `self` Binding

When `trigger` is `"change"`, the element's current value is available via `self.value`
in the payload. This is a special binding that resolves to the element's runtime value:

```json
{
  "type": "input",
  "attributes": {
    "inputType": "search",
    "placeholder": "Search products...",
    "action": "state.set",
    "payload": { "key": "searchQuery", "value": { "$bind": "self.value" } },
    "trigger": "change"
  }
}
```

`self.value` is only meaningful with `"change"` and `"blur"` triggers on form elements
(`input`, `select`, `textarea`). On other elements or triggers, `self.value` resolves
to `null`.

#### Examples

**Variant selector that updates displayed price:**

```json
{
  "type": "select",
  "attributes": {
    "action": "state.set",
    "payload": { "key": "selectedVariant", "value": { "$bind": "self.value" } },
    "trigger": "change",
    "options": { "$bind": "props.product.variants" }
  }
}
```

**Analytics tracking on section view:**

```json
{
  "type": "container",
  "styles": ["hero-section"],
  "attributes": {
    "action": "analytics.track",
    "payload": { "event": "section_view", "section": "hero" },
    "trigger": "mount"
  },
  "children": [...]
}
```

**Image preview on hover:**

```json
{
  "type": "image",
  "attributes": {
    "src": { "$bind": "scope.product.thumbnail" },
    "alt": { "$bind": "scope.product.title" },
    "action": "state.set",
    "payload": { "key": "previewImage", "value": { "$bind": "scope.product.image_full" } },
    "trigger": "hover"
  }
}
```

### Which Elements Support Actions

Actions can be declared on any element via `attributes.action`. In practice, actions
are most common on:
- `button` — click actions
- `link` — navigation (though `link` also has `href` for simple navigation)
- `form` — form submission (`trigger: "submit"`)
- `input`, `select`, `textarea` — value change actions (`trigger: "change"`)
- `container`, `image` — mount/hover actions

---

## 3. Action Namespaces

Actions use dot-notation namespaces to organize intent by domain. The JTOX spec defines
standard namespaces. Platforms add their own.

### 3.1 Navigation

| Action | Payload | Description |
|---|---|---|
| `navigate` | `{ href, target? }` | Navigate to a URL |

```json
{
  "type": "button",
  "attributes": {
    "action": "navigate",
    "payload": {
      "href": { "$bind": "props.product.url" },
      "target": "_blank"
    }
  }
}
```

`target` is optional. Values follow web conventions: `_self` (default), `_blank`.
Non-web renderers interpret `target` as appropriate for their platform.

### 3.2 State Actions

State actions modify local component state (see Section 4).

| Action | Payload | Description |
|---|---|---|
| `state.set` | `{ key, value }` | Set a state variable to a specific value |
| `state.toggle` | `{ key }` | Toggle a boolean state variable |
| `state.increment` | `{ key, min?, max?, step? }` | Increment a numeric state variable |
| `state.decrement` | `{ key, min?, max?, step? }` | Decrement a numeric state variable |

```json
{
  "type": "button",
  "attributes": {
    "action": "state.increment",
    "payload": { "key": "quantity", "max": 99, "step": 1 }
  },
  "children": [{ "type": "text", "content": "+" }]
}
```

Rules:
- `key` MUST reference a state variable declared on an ancestor node
- `step` defaults to `1` for increment/decrement
- If `min` or `max` is specified, the value is clamped (not wrapped)
- `state.toggle` only works on boolean state variables
- `state.set` can set any type — the `value` in the payload becomes the new state value

### 3.3 UI Actions

UI actions control visibility of overlays, drawers, modals, and similar UI elements.

| Action | Payload | Description |
|---|---|---|
| `ui.open` | `{ target, data? }` | Open a named UI element |
| `ui.close` | `{ target }` | Close a named UI element |
| `ui.toggle` | `{ target }` | Toggle a named UI element |

```json
{
  "type": "button",
  "attributes": {
    "action": "ui.open",
    "payload": { "target": "mobile-menu" }
  },
  "children": [{ "type": "text", "content": "Menu" }]
}
```

`target` is a string identifier that matches an overlay's `meta.id` (see
[Overlays](13-overlays.md)). The renderer maps it to the corresponding overlay and
manages its visibility.

`data` is an optional object passed to the overlay as `props`. Values can be literals
or `$bind` objects — they are resolved when the action fires. See
[Overlays §3.1](13-overlays.md) for details.

### 3.4 Form Actions

| Action | Payload | Description |
|---|---|---|
| `form.submit` | `{ formId? }` | Submit a form |
| `form.reset` | `{ formId? }` | Reset form fields to defaults |

```json
{
  "type": "button",
  "attributes": {
    "action": "form.submit",
    "payload": { "formId": "contact-form" }
  },
  "children": [{ "type": "text", "content": "Send Message" }]
}
```

If `formId` is omitted, the action targets the nearest ancestor `form` node.

### 3.5 Platform-Specific Actions

Platforms define their own action namespaces for domain-specific behavior. These are
NOT part of the JTOX spec — they are renderer extensions.

Examples of platform-specific namespaces:

| Namespace | Example Actions | Platform |
|---|---|---|
| `cart.*` | `cart.add`, `cart.remove`, `cart.update` | E-commerce platforms |
| `wishlist.*` | `wishlist.add`, `wishlist.remove` | E-commerce platforms |
| `auth.*` | `auth.login`, `auth.logout` | Platforms with authentication |
| `analytics.*` | `analytics.track` | Platforms with analytics |

```json
{
  "type": "button",
  "attributes": {
    "action": "cart.add",
    "payload": {
      "productId": { "$bind": "props.product.id" },
      "variantId": { "$bind": "props.selectedVariant.id" },
      "quantity": { "$bind": "state.quantity" }
    }
  },
  "styles": ["primary"],
  "children": [{ "type": "text", "content": "Add to Cart" }]
}
```

Renderers MUST handle unknown actions gracefully — the element renders but the action
does nothing. Renderers SHOULD log a warning for unknown actions in development mode.

Renderers SHOULD document their supported platform-specific actions in their renderer
metadata (see 09-renderer-contract.md, Section 6).

### 3.6 Script Handlers

Templates with scripts can define custom handler functions. These are invoked via the
`handler` action namespace:

```json
{
  "type": "button",
  "attributes": {
    "action": "handler.onSelect",
    "payload": { "args": ["option-a"] }
  },
  "children": [{ "type": "text", "content": "Select A" }]
}
```

The `payload.args` array is passed as positional arguments to the handler function.
See [Scripts §5](12-scripts.md) for the full handler specification.

---

## 4. Local Component State

State is client-side UI state managed by the renderer. It enables interactive patterns
like quantity selectors, tabs, accordions, and toggles — without custom JavaScript.

### 4.1 Declaring State

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

| Property | Required | Type | Description |
|---|---|---|---|
| `type` | Yes | string | `"string"`, `"number"`, `"boolean"` |
| `default` | Yes | any | Initial value (must match declared type) |

### 4.2 State Scope

State variables are available via the `state` binding scope:

```json
{ "$bind": "state.quantity" }
{ "$bind": "state.expanded" }
```

State is scoped to the declaring node and all its descendants. A child node can read
and modify state declared by any ancestor. Sibling nodes that are not descendants of
the declaring node cannot access the state.

```
container (declares state.quantity)
  ├── button (can read/modify state.quantity) ✓
  ├── text (can read state.quantity) ✓
  └── container
        └── text (can read state.quantity) ✓

sibling-container (CANNOT access state.quantity) ✗
```

### 4.3 State Shadowing

If a descendant declares a state variable with the same name as an ancestor's state,
the descendant's state shadows the ancestor's within its subtree:

```json
{
  "type": "container",
  "state": { "active": { "type": "boolean", "default": false } },
  "children": [
    {
      "type": "text",
      "content": { "$bind": "state.active" }
    },
    {
      "type": "container",
      "state": { "active": { "type": "boolean", "default": true } },
      "children": [
        {
          "type": "text",
          "content": { "$bind": "state.active" }
        }
      ]
    }
  ]
}
```

The outer text reads the outer `active` (false). The inner text reads the inner
`active` (true). This follows standard lexical scoping rules.

### 4.4 State Types

| Type | Default Value | Valid `state.*` Actions |
|---|---|---|
| `"string"` | `""` | `state.set` |
| `"number"` | `0` | `state.set`, `state.increment`, `state.decrement` |
| `"boolean"` | `false` | `state.set`, `state.toggle` |

State types are intentionally limited. State is for simple UI toggles and counters,
not for complex data management. Complex state belongs in the platform's data layer
(props, context) or in scripted components (platform-specific).

---

## 5. Complete Example: Quantity Selector with Add to Cart

This example combines state, state actions, and a platform-specific action:

```json
{
  "type": "container",
  "styles": ["product-actions"],
  "state": {
    "quantity": { "type": "number", "default": 1 }
  },
  "children": [
    {
      "type": "container",
      "styles": ["quantity-selector"],
      "children": [
        {
          "type": "button",
          "styles": ["qty-btn"],
          "attributes": {
            "action": "state.decrement",
            "payload": { "key": "quantity", "min": 1 }
          },
          "children": [{ "type": "text", "content": "−" }]
        },
        {
          "type": "text",
          "content": { "$bind": "state.quantity" }
        },
        {
          "type": "button",
          "styles": ["qty-btn"],
          "attributes": {
            "action": "state.increment",
            "payload": { "key": "quantity", "max": 99 }
          },
          "children": [{ "type": "text", "content": "+" }]
        }
      ]
    },
    {
      "type": "button",
      "styles": ["primary", "add-to-cart"],
      "attributes": {
        "action": "cart.add",
        "payload": {
          "productId": { "$bind": "props.product.id" },
          "quantity": { "$bind": "state.quantity" }
        }
      },
      "children": [{ "type": "text", "content": "Add to Cart" }]
    }
  ]
}
```

### Interaction Flow

```
1. Page loads → state.quantity = 1 (default)
2. User clicks "+" → state.increment { key: "quantity", max: 99 }
   → Renderer updates state.quantity to 2
   → Text node re-renders showing "2"
3. User clicks "+" again → state.quantity becomes 3
4. User clicks "Add to Cart" → cart.add { productId: "abc", quantity: 3 }
   → Renderer calls platform cart API
   → Platform handles the rest (API call, cart update, UI feedback)
```

---

## 6. Complete Example: Accordion

```json
{
  "type": "each",
  "source": { "$bind": "settings.items" },
  "as": "item",
  "children": [
    {
      "type": "container",
      "styles": ["accordion-item"],
      "state": {
        "open": { "type": "boolean", "default": false }
      },
      "children": [
        {
          "type": "button",
          "styles": ["accordion-header"],
          "attributes": {
            "action": "state.toggle",
            "payload": { "key": "open" }
          },
          "children": [
            { "type": "text", "content": { "$bind": "scope.item.title" } }
          ]
        },
        {
          "type": "conditional",
          "test": { "$bind": "state.open" },
          "children": [
            {
              "type": "container",
              "styles": ["accordion-body"],
              "children": [
                { "type": "text", "content": { "$bind": "scope.item.body" } }
              ]
            }
          ]
        }
      ]
    }
  ]
}
```

Each accordion item has its own `state.open` because the `state` declaration is inside
the `each` loop — each iteration creates a new state scope.

---

## 7. Complete Example: Tab Navigation

```json
{
  "type": "container",
  "styles": ["tabs"],
  "state": {
    "activeTab": { "type": "number", "default": 0 }
  },
  "children": [
    {
      "type": "container",
      "styles": ["tab-headers"],
      "children": [
        {
          "type": "each",
          "source": { "$bind": "settings.tabs" },
          "as": "tab",
          "children": [
            {
              "type": "button",
              "styles": ["tab-header"],
              "attributes": {
                "action": "state.set",
                "payload": {
                  "key": "activeTab",
                  "value": { "$bind": "scope.$index" }
                }
              },
              "children": [
                { "type": "text", "content": { "$bind": "scope.tab.label" } }
              ]
            }
          ]
        }
      ]
    },
    {
      "type": "each",
      "source": { "$bind": "settings.tabs" },
      "as": "tab",
      "children": [
        {
          "type": "conditional",
          "test": { "$bind": "scope.$index" },
          "operator": "eq",
          "value": { "$bind": "state.activeTab" },
          "children": [
            {
              "type": "container",
              "styles": ["tab-panel"],
              "children": [
                { "type": "text", "content": { "$bind": "scope.tab.content" } }
              ]
            }
          ]
        }
      ]
    }
  ]
}
```

---

## 8. Renderer Requirements for Actions & State

### MUST

1. Support the `state` property on element nodes
2. Maintain state values with correct scoping (declaring node + descendants)
3. Handle `state.set`, `state.toggle`, `state.increment`, `state.decrement` actions
4. Resolve `state.*` bindings against the nearest ancestor's state declaration
5. Re-render affected nodes when state changes
6. Handle unknown actions gracefully (ignore, don't crash)
7. Support the `trigger` property on action attributes
8. Default to `"click"` trigger when `trigger` is omitted
9. Resolve `self.value` bindings for `"change"` and `"blur"` triggers on form elements

### SHOULD

1. Support `navigate`, `ui.open`, `ui.close`, `ui.toggle`, `form.submit`, `form.reset`
2. Clamp `state.increment`/`state.decrement` to `min`/`max` when specified
3. Log warnings for actions targeting undeclared state keys
4. Document supported platform-specific action namespaces in renderer metadata
5. Support all standard triggers: `click`, `change`, `submit`, `mount`, `hover`, `blur`
6. Provide graceful fallback for `hover` on touch devices

### MAY

1. Define additional action namespaces (e.g., `cart.*`, `wishlist.*`)
2. Implement action chaining (multiple actions from one interaction)
3. Implement action feedback (loading states, success/error indicators)
4. Optimize state updates (batching, memoization)

---

## 9. SSR Considerations

State is a client-side concept. On the server:

- State variables resolve to their `default` values
- State actions are inert (buttons render but don't trigger actions)
- `conditional` nodes that test `state.*` evaluate against defaults

This means the server-rendered output shows the initial state. Client-side hydration
activates the state system and makes actions functional.

Theme authors should design templates so the default state produces a sensible initial
render. For example, an accordion with `state.open` defaulting to `false` renders all
panels collapsed — the server output is valid and usable.
