# JTOX Overlays

Version: 1.1.0-draft

---

## 1. Overview

Overlays are UI surfaces that appear on top of the main page content in response to a
user action. They are temporary — the user interacts with them and dismisses them to
return to the main content.

Common overlay patterns include drawers, modals, fullscreen panels, toasts, and bottom
bars. The visual presentation is controlled entirely by CSS — the JTOX spec defines
the structure and visibility model, not the appearance.

Overlays exist because some interactive surfaces need to be:
- **Cross-section** — triggered from any section on any page
- **Independently themed** — their own template, settings, and styles
- **Top-layer** — rendered above all page content, outside any stacking context
- **Data-receiving** — able to accept data from the trigger action

Simple show/hide patterns (accordions, dropdowns, tabs, mobile menus within a section)
do NOT need overlays. Use `state.*` actions with `conditional` nodes instead. Overlays
are for surfaces that span across sections.


---

## 2. The `overlay` Node Type

An overlay is a root node type, like `section`, `page`, `header`, `footer`, and `layout`.
It is defined as a standalone JTOX document.

```json
{
  "$jtox": "1.0",
  "type": "overlay",
  "meta": {
    "id": "image-viewer",
    "name": "Image Viewer"
  },
  "styles": ["overlay-fullscreen"],
  "attributes": {
    "role": "dialog",
    "aria-label": "Image Viewer"
  },
  "children": [
    {
      "type": "container",
      "styles": ["overlay-backdrop"],
      "attributes": {
        "action": "ui.close",
        "payload": { "target": "image-viewer" }
      }
    },
    {
      "type": "container",
      "styles": ["overlay-content"],
      "children": [
        {
          "type": "image",
          "attributes": {
            "src": { "$bind": "props.src" },
            "alt": { "$bind": "props.alt" }
          }
        },
        {
          "type": "button",
          "styles": ["overlay-close"],
          "attributes": {
            "action": "ui.close",
            "payload": { "target": "image-viewer" }
          },
          "children": [
            { "type": "icon", "attributes": { "name": "x", "size": 24 } }
          ]
        }
      ]
    }
  ]
}
```

### Properties

| Property | Required | Type | Description |
|---|---|---|---|
| `type` | Yes | `"overlay"` | Node type identifier |
| `meta.id` | Yes | string | Unique identifier — the `ui.open` target |
| `meta.name` | No | string | Human-readable name for tooling |
| `styles` | No | array | Style identifiers for the overlay container |
| `attributes` | No | object | Accessibility and other attributes |
| `children` | Yes | array | The overlay's content |
| `state` | No | object | Local state declarations (same as sections) |

### Key Differences from Other Root Types

| | Section | Page / Header / Footer | Overlay |
|---|---|---|---|
| `meta.id` | Optional | Optional | **Required** |
| `placement` | Yes | N/A | N/A |
| `allowedSections` | N/A | Yes | **No** |
| Visible by default | Yes | Yes | **No** |
| Triggered by | Always rendered | Always rendered | **`ui.open` action** |
| Contains sections | N/A | Yes (via slot) | **No** (direct content only) |
| Supports scripts | Yes | No | **Yes** |
| Supports settings | Yes | No | **Yes** |


---

## 3. Connection to `ui.*` Actions

Overlays are controlled by the `ui.*` actions defined in [Actions & State §3.3](11-actions.md).
The connection is through `meta.id`:

```json
// Overlay declares its identity
{ "type": "overlay", "meta": { "id": "image-viewer" }, ... }

// Any element in any section can trigger it
{ "action": "ui.open", "payload": { "target": "image-viewer" } }
```

The `target` in `ui.open`, `ui.close`, and `ui.toggle` MUST match an overlay's `meta.id`.
If no overlay with the given `meta.id` exists, the renderer SHOULD log a warning and
take no action.

### 3.1 `ui.open` with Data

`ui.open` accepts an optional `data` property in its payload. The renderer passes this
data to the overlay as `props`:

```json
{
  "type": "image",
  "attributes": {
    "src": { "$bind": "scope.image.thumbnail" },
    "alt": { "$bind": "scope.image.alt" },
    "action": "ui.open",
    "payload": {
      "target": "image-viewer",
      "data": {
        "src": { "$bind": "scope.image.fullsize" },
        "alt": { "$bind": "scope.image.alt" }
      }
    }
  }
}
```

Inside the overlay template, the data is available via `props.*`:

```json
{ "$bind": "props.src" }
{ "$bind": "props.alt" }
```

Values in `data` can be literals or `$bind` objects — they are resolved at the time
the action fires, before being passed to the overlay.

### 3.2 Updated `ui.open` Payload

| Property | Required | Type | Description |
|---|---|---|---|
| `target` | Yes | string | The `meta.id` of the overlay to open |
| `data` | No | object | Data to pass to the overlay as `props`. Values can be literals or `$bind` objects. |

`ui.close` and `ui.toggle` payloads remain unchanged (`{ target }` only).


---

## 4. Visibility Model

Overlays are hidden by default. The renderer manages their visibility state.

### 4.1 States

An overlay is in one of two states: **closed** (default) or **open**.

- `ui.open` transitions from closed → open
- `ui.close` transitions from open → closed
- `ui.toggle` switches between the two states

### 4.2 Rendering Strategy

The spec does not prescribe how the renderer implements visibility. Options include:

- Pre-rendering the overlay in the DOM but hidden, then toggling visibility
- Lazy-rendering the overlay only when `ui.open` is called
- Using the native `<dialog>` element's `showModal()` / `close()` methods
- Using the Popover API

The renderer chooses the strategy. The spec requires only that:
- Closed overlays are not visible to the user
- Open overlays are visible and interactive
- The transition between states is immediate or animated (renderer's choice)

### 4.3 Multiple Overlays

Multiple overlays MAY be open simultaneously. A notification toast and a drawer can
coexist. The renderer manages stacking order.

If the platform wants to enforce "only one modal at a time," that is a platform
constraint, not a JTOX spec constraint.

---

## 5. Data Flow

Overlays receive data from three sources:

| Source | Scope | Description |
|---|---|---|
| Settings | `settings.*` | The overlay's own settings (from `settings.json`) |
| Context | `context.*` | Global platform context (same as all templates) |
| Action data | `props.*` | Data passed via `ui.open` `payload.data` |

### 5.1 Settings

Overlays can have their own `settings.json`, just like sections. The business owner
configures these in the editor.

```json
{ "$bind": "settings.drawer_title" }
```

### 5.2 Context

Global context is available to overlays, same as any template:

```json
{ "$bind": "context.user.name" }
{ "$bind": "context.navigation.main" }
```

### 5.3 Action Data (Props)

Data passed via `ui.open` `payload.data` is available as `props.*`:

```json
// Trigger
{ "action": "ui.open", "payload": { "target": "detail-view", "data": { "itemId": "abc" } } }

// Overlay template
{ "$bind": "props.itemId" }
```

If the overlay is opened without `data`, `props` is an empty object. Overlay templates
SHOULD use `default` values on `$bind` objects that reference `props` to handle this
gracefully.

### 5.4 No Parent Props

Unlike components (which receive `props` from a parent `$ref`), overlays do not have
a parent in the composition hierarchy. Their only source of `props` is `ui.open` data.


---

## 6. Composition

### 6.1 Position in the Hierarchy

Overlays are peers of header, page, and footer in the layout:

```
Layout
  ├── Header (region → sections)
  ├── Page (region → sections)
  ├── Footer (region → sections)
  └── Overlays (hidden until opened)
      ├── overlay "image-viewer"
      ├── overlay "search-panel"
      └── overlay "notification"
```

### 6.2 Layout Slot

The layout template MAY include an `overlays` slot:

```json
{
  "type": "layout",
  "children": [
    { "type": "slot", "name": "header" },
    { "type": "slot", "name": "page" },
    { "type": "slot", "name": "footer" },
    { "type": "slot", "name": "overlays" }
  ]
}
```

The rendering pipeline fills the `overlays` slot with all overlay templates. If the
layout does not include an `overlays` slot, the renderer SHOULD still render overlays
(appending them to the layout output).

### 6.3 No Section Composability

Overlays contain direct content — elements, components, and blocks. They do NOT
support `allowedSections` or `defaultSections`. Overlays are self-contained.

If a future use case requires section composability inside overlays, it can be added
as a minor version extension.

### 6.4 Theme Directory

Overlays follow the same four-file pattern as sections and components:

```
{theme-name}/
├── overlays/
│   └── {overlay-id}/
│       ├── template.json     (required)
│       ├── settings.json     (optional)
│       ├── styles.css        (optional)
│       └── script.js         (optional)
```

---

## 7. Accessibility

Overlays have specific accessibility requirements because they appear on top of page
content and may block interaction with the underlying page.

### SHOULD

1. Overlays that block interaction with the page SHOULD declare
   `attributes.role = "dialog"` and `attributes["aria-modal"] = "true"`
2. All overlays SHOULD have `attributes["aria-label"]` or
   `attributes["aria-labelledby"]` describing their purpose
3. When an overlay opens, the renderer SHOULD move focus to the overlay
4. When an overlay closes, the renderer SHOULD return focus to the trigger element
5. Modal overlays SHOULD trap focus (Tab cycles within the overlay)
6. The Escape key SHOULD close the topmost open overlay

These are SHOULD, not MUST, because:
- The spec does not prescribe rendering behavior (P1)
- Accessibility implementation varies by platform
- On web, the `<dialog>` element handles most of these automatically

The overlay template declares the accessibility attributes. The renderer implements
the behavior:

```json
{
  "type": "overlay",
  "attributes": {
    "role": "dialog",
    "aria-label": "Image Viewer",
    "aria-modal": "true"
  }
}
```


---

## 8. When to Use Overlays vs State + Conditional

Not every show/hide pattern needs an overlay. Most don't.

| Pattern | Approach | Why |
|---|---|---|
| Accordion panel | `state.toggle` + `conditional` | Same section, no cross-section need |
| Dropdown submenu | `state.set` + `conditional` | Same section, hover/click within nav |
| Mobile menu (in header) | `state.toggle` + `conditional` | Same section, CSS handles positioning |
| Tab panel | `state.set` + `conditional` | Same section, content switching |
| Tooltip | `state.set` + `conditional` | Same element, hover-triggered |
| Drawer triggered from multiple sections | **Overlay** | Cross-section trigger |
| Image viewer (any image, any page) | **Overlay** | Cross-section, needs data passing |
| Search panel (triggered from header) | Either | Overlay if independently themed |
| Quick view modal (from a list) | **Overlay** | Cross-section, needs data passing |
| Notification toast (after an action) | **Overlay** | Triggered by platform actions, cross-section |

**Rule of thumb:** If the trigger and the content are in the same section, use
`state.*` + `conditional`. If the trigger is in one section and the content needs to
be independently themed or triggered from multiple places, use an overlay.

---

## 9. Complete Example: Data-Receiving Overlay

An image viewer overlay that receives the image URL from the trigger:

### Trigger (in any section)

```json
{
  "type": "each",
  "source": { "$bind": "props.images" },
  "as": "image",
  "children": [
    {
      "type": "image",
      "styles": ["thumbnail"],
      "attributes": {
        "src": { "$bind": "scope.image.thumbnail" },
        "alt": { "$bind": "scope.image.alt" },
        "action": "ui.open",
        "payload": {
          "target": "image-viewer",
          "data": {
            "src": { "$bind": "scope.image.fullsize" },
            "alt": { "$bind": "scope.image.alt" }
          }
        }
      }
    }
  ]
}
```

### Overlay Template

```json
{
  "$jtox": "1.0",
  "type": "overlay",
  "meta": { "id": "image-viewer", "name": "Image Viewer" },
  "styles": ["overlay-fullscreen"],
  "attributes": {
    "role": "dialog",
    "aria-label": "Image Viewer",
    "aria-modal": "true"
  },
  "children": [
    {
      "type": "container",
      "styles": ["viewer-backdrop"],
      "attributes": {
        "action": "ui.close",
        "payload": { "target": "image-viewer" }
      }
    },
    {
      "type": "container",
      "styles": ["viewer-content"],
      "children": [
        {
          "type": "image",
          "styles": ["viewer-image"],
          "attributes": {
            "src": { "$bind": "props.src" },
            "alt": { "$bind": "props.alt", "default": "Image" }
          }
        },
        {
          "type": "button",
          "styles": ["viewer-close"],
          "attributes": {
            "action": "ui.close",
            "payload": { "target": "image-viewer" }
          },
          "children": [
            { "type": "icon", "attributes": { "name": "x", "size": 24 } }
          ]
        }
      ]
    }
  ]
}
```

### Interaction Flow

```
1. User clicks a thumbnail in any section on any page
2. Renderer resolves $bind objects in payload.data:
   data = { src: "https://cdn.example.com/full/photo.jpg", alt: "Beach sunset" }
3. Renderer opens the "image-viewer" overlay
4. Overlay receives data as props: props.src, props.alt
5. Overlay renders with the full-size image
6. User clicks backdrop or close button → ui.close fires
7. Overlay closes, focus returns to the thumbnail
```

---

## 10. SSR Considerations

On the server:
- Overlays are rendered in their closed (hidden) state
- `props` from `ui.open` data is not available (no action has fired)
- Overlay templates SHOULD render gracefully with empty `props` (use `default` values)
- The overlay DOM is present in the HTML but hidden (for instant show on client)

On the client:
- Overlays become interactive after hydration
- `ui.open` populates `props` and makes the overlay visible
- Scripts (if present) activate — `onMount` is called when the overlay opens

---

## 11. Renderer Requirements

### MUST

1. Support the `overlay` node type as a root node
2. Render overlays as hidden by default
3. Show overlays when `ui.open` fires with a matching `target`
4. Hide overlays when `ui.close` fires with a matching `target`
5. Toggle overlay visibility when `ui.toggle` fires with a matching `target`
6. Pass `payload.data` from `ui.open` to the overlay as `props`
7. Handle `ui.open` with a non-existent `target` gracefully (log warning, no crash)

### SHOULD

1. Render overlays in the top layer (above all page content, outside stacking contexts)
2. Implement focus management (move focus to overlay on open, return on close)
3. Close the topmost overlay on Escape key press
4. Support multiple overlays open simultaneously
5. Support the `overlays` slot in layout templates
6. Pre-render overlay DOM on the server for instant client-side show

### MAY

1. Implement scroll locking when a modal overlay is open
2. Implement backdrop click-to-close behavior based on template structure
3. Animate overlay transitions (CSS-driven)
4. Lazy-render overlays (create DOM only on first `ui.open`) instead of pre-rendering

---

## 12. Design Decisions

### DD-016: Overlay as a new root node type

**Decision:** Overlays are a distinct root node type, not a property on sections.

**Rationale:**
- Overlays are fundamentally different from sections: hidden by default, action-triggered,
  top-layer rendered, with dialog accessibility requirements
- A distinct type makes these differences explicit in the node type system
- Overlays have their own templates, settings, styles, and scripts — same four-file
  pattern as sections but with different semantics
- Sections have `placement` (PAGE, HEADER, FOOTER) — overlays don't fit this model

**Tradeoff:** One more root node type to learn. But the concept is intuitive — "an
overlay is a hidden surface that appears when triggered."

### DD-017: `payload.data` on `ui.open`

**Decision:** `ui.open` accepts optional `data` that becomes the overlay's `props`.

**Rationale:**
- Many overlays need data from the trigger context (which image to show, which item
  to preview)
- Without data passing, overlays can only show static content or global context
- The pattern mirrors how components receive `props` from parent `$ref` nodes
- `$bind` objects in `data` are resolved at trigger time — the overlay receives
  concrete values, not binding expressions

**Tradeoff:** Adds complexity to the `ui.open` payload format. But the alternative
(every data-receiving overlay needs a script that reads from a global store) is worse.

### DD-018: No section composability inside overlays

**Decision:** Overlays contain direct content only. No `allowedSections` or
`defaultSections`.

**Rationale:**
- Overlays are typically self-contained (a viewer, a drawer, a panel)
- Section composability inside overlays adds significant complexity to the editor UI
  and rendering pipeline
- The business owner customizes overlays through settings, not by adding/removing sections
- If needed, section composability can be added in a future minor version

**Tradeoff:** An overlay that needs complex, configurable content must use components
and settings rather than section composition. This is sufficient for all common overlay
patterns.