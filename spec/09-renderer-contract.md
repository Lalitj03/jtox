# JTOX Renderer Contract

Version: 1.0.0-draft

---

## 1. Overview

A JTOX renderer is any system that takes a JTOX document and produces visible output.
This document defines the contract that renderers must fulfill.

JTOX does not prescribe renderer implementation. A renderer can be:
- A React component that interprets JTOX at runtime
- A Vue plugin that maps JTOX nodes to Vue components
- A Go template engine that produces HTML
- A Python script that compiles JTOX to static HTML
- A native mobile renderer that produces SwiftUI or Jetpack Compose views

All renderers must fulfill the same contract.

---

## 2. Renderer Requirements

### MUST (Required)

A conforming renderer MUST:

1. Accept any valid JTOX document for its declared spec version
2. Render all node types defined in the spec version (including `overlay` in 1.1+)
3. Resolve `$bind` objects according to the binding rules
4. Support all eight binding scopes (settings, props, scope, theme, context, state, self, computed)
5. Process `each` nodes by iterating over the source data
6. Process `conditional` nodes by evaluating the test condition
7. Resolve `$ref` paths to referenced templates
8. Pass `props` from reference nodes to referenced templates
9. Apply `styles` identifiers to rendered elements (mapping to platform styles)
10. Handle missing bindings gracefully (return null/default, never crash)
11. Handle missing `$ref` targets gracefully (skip, never crash)
12. Declare which JTOX spec version(s) it supports

### SHOULD (Recommended)

A conforming renderer SHOULD:

1. Support the `tag` override for web renderers
2. Support all standard transforms on `$bind` objects (see [Bindings §5](03-bindings.md#5-transforms) for the full set including parameterized and chained transforms)
3. Implement the standard action namespaces: `navigate`, `state.*`, `ui.*`, `form.*`, `handler.*` (see [Actions & State](11-actions.md), [Scripts](12-scripts.md))
4. Log warnings for unresolved bindings and references in development mode
5. Enforce section `placement` constraints
6. Validate documents before rendering (at least structural validation)
7. Support the `empty` property on `each` nodes
8. Support the `fallback` property on `conditional` nodes
9. Support the `default` property on `$bind` objects
10. Load and execute script modules for templates that have them (see [Scripts](12-scripts.md))
11. Manage overlay visibility state and respond to `ui.open`/`ui.close`/`ui.toggle` (see [Overlays](13-overlays.md))

### MAY (Optional)

A conforming renderer MAY:

1. Define additional action identifiers beyond the standard set
2. Define additional transforms beyond the standard set (using the same `{ "name", "args" }` convention)
3. Define additional context variables beyond the standard set
4. Implement rendering optimizations (memoization, virtualization, etc.)
5. Support additional node types as extensions (must not conflict with standard types)
6. Implement a compilation mode (JTOX → static output at build time)

---

## 3. Rendering Pipeline

Every renderer follows this conceptual pipeline (implementation may vary):

```
1. LOAD        Load the JTOX document (parse JSON)
2. VALIDATE    Validate structure and types (optional but recommended)
3. RESOLVE     Resolve $ref references and load scripts
4. SCRIPT      Call script functions, register computed values and handlers
5. BIND        Resolve $bind objects against their scopes (including computed.*)
6. CONTROL     Process each/conditional nodes
7. STYLE       Map style identifiers to platform styles
8. RENDER      Produce visible output
9. MOUNT       Call onMount, populate refs (client-side only)
```

### Step Details

**LOAD:** Parse the JSON. Verify `$jtox` version is supported. If not, reject with
a clear error message.

**VALIDATE:** Check structural validity. Are required fields present? Are types correct?
This step is optional at render time but MUST be available as a standalone operation.

**RESOLVE:** Walk the tree. For each `component` or `block` node, load the referenced
template. Inject `props` into the referenced template's scope. Detect circular references.
For templates with a `script.js` file, load the script module.

**SCRIPT:** For each template instance with a script, call the script's default export
function with the context object (`props`, `state`, `actions`, `refs`, `context`,
`settings`). Register the returned `computed` functions and `handlers`. See
[Scripts](12-scripts.md) for the full contract.

**BIND:** For each `$bind` object, resolve the path against the appropriate scope:
- `settings.*` → current template's settings values
- `props.*` → data passed from parent (or `ui.open` data for overlays)
- `scope.*` → loop variables from ancestor `each` nodes
- `theme.*` → global theme settings
- `context.*` → renderer-provided context
- `state.*` → local UI state from ancestor state declarations
- `self.*` → element runtime value (change/blur triggers only)
- `computed.*` → script-derived reactive values
- `self.*` → element runtime value (change/blur triggers only)

Apply `default` if the path resolves to null. Apply `transform` if specified — if
`transform` is an array, execute each transform left to right, piping the output of
each into the next.

**CONTROL:** For `each` nodes, iterate over the resolved `source`. Create scope variables
(`scope.{as}`, `scope.$index`, etc.) for each iteration. For `conditional` nodes, evaluate
the `test` binding and render `children` or `fallback` accordingly.

**STYLE:** Map style identifiers to platform-specific styles. On web, this typically means
CSS class names. The mapping mechanism is renderer-specific.

**RENDER:** Produce the final output. On web: DOM elements or HTML string. On native:
platform views. The output format is renderer-specific.

---

## 4. Error Handling

Renderers MUST be resilient. A JTOX document with missing data, broken references, or
invalid bindings should degrade gracefully, not crash.

| Error | Behavior |
|---|---|
| Missing `$bind` path | Return `default` value, or `null` if no default |
| Missing `$ref` target | Skip the node (render nothing) |
| Invalid `type` value | Skip the node, log warning |
| `each` source is not an array | Treat as empty array, render `empty` content |
| `conditional` test is null | Treat as falsy, render `fallback` |
| Circular `$ref` | Stop resolution, log error, render nothing for the circular node |
| Unknown `transform` | Skip the transform, return the raw value |
| Unknown `action` | Ignore the action (button renders but does nothing) |

---

## 5. Rendering Strategies

JTOX supports multiple rendering strategies. The document is the same; the strategy
determines when and where rendering happens.

### Runtime Interpretation (Client-Side)

```
Browser loads JTOX JSON → JavaScript renderer walks tree → React/Vue/DOM elements
```

Best for: Design editors, live preview, interactive authoring tools.
Tradeoff: Every visitor pays the interpretation cost.

### Server-Side Rendering (SSR)

```
Server loads JTOX JSON → Server-side renderer walks tree → HTML string → sent to browser
```

Best for: Customer-facing pages, SEO, fast first paint.
Tradeoff: Server CPU per request (mitigated by caching/ISR).

### Static Compilation (Build-Time)

```
Build step loads JTOX JSON → Compiler resolves static bindings → outputs optimized
templates → Server fills dynamic bindings at request time
```

Best for: High-traffic pages, maximum performance, CDN-friendly.
Tradeoff: Requires a compilation pipeline. Changes require re-publish.

### Hybrid

Most real-world implementations use a hybrid:
- Static bindings (`settings.*`, `theme.*`) resolved at build/publish time
- Dynamic bindings (`props.*`, `context.*`) resolved at request/render time
- Interactive elements hydrated on the client

---

## 6. Renderer Metadata

Renderers SHOULD expose metadata about their capabilities:

```json
{
  "name": "jtox-react",
  "version": "1.0.0",
  "jtoxVersions": ["1.0"],
  "platform": "web",
  "strategy": "runtime",
  "extensions": {
    "additionalActions": ["add-to-cart", "wishlist-toggle"],
    "additionalTransforms": ["slugify", "initials"],
    "additionalContext": ["cart", "wishlist", "recentlyViewed"]
  }
}
```

This metadata helps theme authors understand what a renderer supports beyond the
standard spec.
