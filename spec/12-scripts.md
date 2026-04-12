# JTOX Scripts

Version: 1.1.0-draft

---

## 1. Overview

JTOX templates are declarative — they describe structure and bind to data. The action
system (see [Actions & State](11-actions.md)) handles simple interactive patterns like
toggles, counters, and tabs without any code. But some interactions require logic that
declarative templates cannot express:

- Computing derived values from data (filtering a list based on a selection)
- Responding to user interactions with conditional logic (if X changed, reset Y)
- Performing DOM operations (scrolling an element to a position)
- Managing asynchronous workflows (debounced input, data fetching)

Scripts are the behavior layer. A script is a JavaScript file that lives alongside a
template and provides computed values and event handlers. The template remains pure JSON.
The script provides the logic that the template references by name.

```
template.json  — structure and intent (JTOX, declarative)
settings.json  — configuration schema (JSON)
styles.css     — presentation (CSS)
script.js      — behavior (JavaScript)    ← this chapter
```

Four concerns, four files. Each is optional. A section with no interactivity beyond
`state.*` actions does not need a script.


---

## 2. The Script Contract

A script file MUST export a single default function. The renderer calls this function
once per template instance, passing a context object. The function returns an API object
that the renderer wires to the template.

### 2.1 Function Signature

```javascript
export default function MySection({ props, state, actions, refs, context, settings }) {
    // ... compute values, define handlers ...

    return {
        computed: { /* named functions that return derived values */ },
        handlers: { /* named functions that handle user interactions */ },
        onMount()   { /* called after DOM is ready, client-side only */ },
        onUnmount() { /* called before removal, client-side only */ }
    };
}
```

The function name is optional but recommended for debugging. It has no semantic meaning.

### 2.2 Context Object

The renderer passes a context object with these properties:

| Property | Type | Description |
|---|---|---|
| `props` | object | Data passed from the parent or injected by the renderer |
| `state` | object | Current values of the template's declared `state` variables (read-only snapshot) |
| `actions` | object | Functions to modify state and trigger platform behavior (see §2.3) |
| `refs` | object | DOM element references declared in the template (see §4) |
| `context` | object | Global platform context available to all templates |
| `settings` | object | The template's resolved settings values |

All properties except `actions` are read-only. To modify state, use `actions.state.*`.
To trigger platform behavior, use the appropriate action namespace.

### 2.3 The `actions` Object

The `actions` object provides the same action namespaces available in templates, but as
callable functions instead of declarative attributes:

```javascript
// State actions (same as state.* in templates)
actions.state.set('selectedItem', 'abc');
actions.state.toggle('menuOpen');
actions.state.increment('quantity', { max: 99 });
actions.state.decrement('quantity', { min: 1 });

// UI actions (same as ui.* in templates)
actions.ui.open('sidebar');
actions.ui.close('sidebar');
actions.ui.toggle('sidebar');

// Platform-specific actions (renderer-defined)
// e.g., actions.cart.add({ ... }), actions.navigate({ href: '/' })
```

The JTOX spec defines `state.*`, `ui.*`, `form.*`, and `navigate`. Platforms add their
own namespaces. See [Actions & State §3](11-actions.md) for the standard set.

### 2.4 Return Object

The function returns an object with these optional properties:

| Property | Type | Description |
|---|---|---|
| `computed` | `{ [name]: () => value }` | Named functions that return derived values. Referenced in templates via `computed.*` bindings. |
| `handlers` | `{ [name]: (...args) => void }` | Named functions that handle user interactions. Referenced in templates via `handler.*` actions. |
| `onMount` | `() => void` | Called once after the template instance is rendered and DOM refs are available. Client-side only. |
| `onUnmount` | `() => void` | Called once before the template instance is removed from the DOM. Client-side only. |

All properties are optional. A script that only provides computed values can omit
`handlers` and lifecycle hooks. A script that only sets up a third-party library in
`onMount` can omit `computed` and `handlers`.


---

## 3. Computed Values

Computed values are functions that derive data from props, state, settings, or context.
They are the primary mechanism for logic that declarative templates cannot express.

### 3.1 Defining Computed Values

```javascript
export default function FilterableList({ props, state }) {
    const filteredItems = () => {
        if (!state.filterText) return props.items;
        const query = state.filterText.toLowerCase();
        return props.items.filter(item =>
            item.title.toLowerCase().includes(query)
        );
    };

    const resultCount = () => filteredItems().length;

    const hasResults = () => resultCount() > 0;

    return {
        computed: { filteredItems, resultCount, hasResults },
        handlers: {}
    };
}
```

### 3.2 Using Computed Values in Templates

Computed values are accessed via the `computed` binding scope:

```json
{
  "type": "each",
  "source": { "$bind": "computed.filteredItems" },
  "as": "item",
  "children": [
    {
      "type": "text",
      "content": { "$bind": "scope.item.title" }
    }
  ],
  "empty": [
    { "type": "text", "content": "No results found." }
  ]
}
```

```json
{
  "type": "text",
  "content": {
    "$bind": "computed.resultCount",
    "transform": "toString"
  }
}
```

Computed values support transforms, defaults, and type hints — the same `$bind` features
available on any other binding scope.

### 3.3 Reactivity

Computed values are reactive. The renderer tracks which data each computed function reads
and re-evaluates the function when that data changes.

```
1. User types "shirt" → handler calls actions.state.set('filterText', 'shirt')
2. Renderer detects state.filterText changed
3. Renderer re-calls filteredItems() — it reads state.filterText, so it's a dependency
4. filteredItems() returns the filtered array
5. Renderer re-renders the each loop with the new array
6. resultCount() and hasResults() also re-evaluate (they call filteredItems())
```

The renderer implements dependency tracking. The spec does not prescribe the mechanism —
it could be proxy-based observation, explicit dependency declaration, or re-evaluation
on every state change. The result must be: when data changes, computed values that depend
on it produce updated results, and template nodes that bind to those computed values
re-render.

### 3.4 Computed Values Are Synchronous

Computed values MUST be synchronous functions. They return a value, not a Promise.

This is intentional. Computed values are called during the render cycle — the renderer
needs the value immediately to produce output. Asynchronous data fetching belongs in
handlers (see §5.5 for the async pattern).

### 3.5 Computed Values Should Be Pure

A computed function SHOULD be a pure function of its inputs — given the same `props`,
`state`, `settings`, and `context`, it should return the same result. It SHOULD NOT:

- Modify state (use handlers for that)
- Trigger side effects (use `onMount` or handlers)
- Access the DOM (use `refs` in handlers or `onMount`)

Purity is a recommendation, not an enforcement. The renderer does not verify purity.
But impure computed values may cause unexpected re-renders or stale data.

### 3.6 Computed Values as Data Shapers

Computed values can build structured data that the template iterates over. This is the
recommended pattern when the template needs a data shape that doesn't exist in the raw
props — the script transforms the data, the template consumes the result.

```javascript
// Script builds a structure the template can iterate
const optionGroups = () => {
    return props.attributeNames.map(name => ({
        name,
        label: name.charAt(0).toUpperCase() + name.slice(1),
        options: getOptionsForAttribute(name),
        selected: state.selections[name] || ''
    }));
};
```

```json
{
  "type": "each",
  "source": { "$bind": "computed.optionGroups" },
  "as": "group",
  "children": [
    {
      "type": "label",
      "children": [{ "type": "text", "content": { "$bind": "scope.group.label" } }]
    },
    {
      "type": "each",
      "source": { "$bind": "scope.group.options" },
      "as": "option",
      "children": [
        {
          "type": "button",
          "attributes": {
            "action": "handler.onOptionSelect",
            "payload": { "args": [{ "$bind": "scope.group.name" }, { "$bind": "scope.option" }] }
          },
          "children": [{ "type": "text", "content": { "$bind": "scope.option" } }]
        }
      ]
    }
  ]
}
```

The script does the heavy lifting — the template stays simple with standard `each`
loops and `$bind` paths. No parameterized computed values needed.


---

## 4. Refs

Refs are named references to rendered DOM elements. They allow scripts to perform
operations that require direct element access — scrolling, measuring, focusing, or
initializing third-party libraries.

### 4.1 Declaring Refs in Templates

A template declares a ref by adding a `ref` attribute to any element node:

```json
{
  "type": "container",
  "styles": ["carousel"],
  "attributes": { "ref": "carousel" },
  "children": [...]
}
```

The `ref` value is a string identifier. It MUST be unique within the template.

### 4.2 Accessing Refs in Scripts

Refs are available in the context object as `refs.{name}`:

```javascript
export default function ImageGallery({ refs }) {
    const scrollToSlide = (index) => {
        refs.carousel?.scrollTo({
            left: index * refs.carousel.offsetWidth,
            behavior: 'smooth'
        });
    };

    return {
        computed: {},
        handlers: { scrollToSlide }
    };
}
```

### 4.3 Ref Lifecycle

- During SSR: all refs are `null` (no DOM on the server)
- After client-side mount: refs are populated with DOM element references
- In `onMount`: refs are available (DOM is ready)
- In computed values: refs SHOULD NOT be accessed (computed values run during render,
  before DOM updates are flushed)

### 4.4 Refs Are a Convenience, Not a Requirement

Scripts have full JavaScript access. Direct DOM queries also work:

```javascript
// Using refs (recommended)
refs.carousel?.scrollTo({ left: 0 });

// Using DOM API (also works)
document.querySelector('.carousel')?.scrollTo({ left: 0 });
```

Refs are recommended because they are:
- SSR-safe (null on server — no crash)
- Scoped to the template instance (no accidental cross-template access)
- Stable across style scoping changes (ref names don't change, class names may)

The spec does not restrict DOM access. Script authors may use any browser API.


---

## 5. Handlers

Handlers are functions that respond to user interactions. They are the imperative
counterpart to declarative actions — where `state.set` handles simple value changes,
handlers handle complex logic that involves multiple state updates, conditional behavior,
or side effects.

### 5.1 Defining Handlers

```javascript
export default function OptionPicker({ props, state, actions }) {
    const onSelect = (groupName, value) => {
        const newSelections = { ...state.selections, [groupName]: value };
        actions.state.set('selections', newSelections);

        // If the current selection in another group is no longer valid, clear it
        const valid = isValidCombination(newSelections);
        if (!valid) {
            actions.state.set('selections', { [groupName]: value });
        }
    };

    return {
        computed: {},
        handlers: { onSelect }
    };
}
```

### 5.2 Calling Handlers from Templates

Handlers are invoked via the `handler` action namespace:

```json
{
  "type": "button",
  "attributes": {
    "action": "handler.onSelect",
    "payload": { "args": [{ "$bind": "scope.group.name" }, { "$bind": "scope.option" }] }
  },
  "children": [
    { "type": "text", "content": { "$bind": "scope.option" } }
  ]
}
```

The `payload.args` array is passed as positional arguments to the handler function.
Values in `args` can be literals or `$bind` objects — they are resolved before the
handler is called.

```
Template: action "handler.onSelect", payload { args: ["color", "red"] }
    ↓
Renderer resolves $bind objects in args
    ↓
Renderer calls: handlers.onSelect("color", "red")
```

### 5.3 Handler Action Format

| Property | Required | Type | Description |
|---|---|---|---|
| `action` | Yes | string | `"handler.{name}"` — the handler function name |
| `payload` | No | object | `{ "args": [...] }` — arguments to pass to the handler |
| `trigger` | No | string | When the handler fires. Defaults to `"click"`. Same triggers as standard actions. |

If `payload` is omitted or `payload.args` is omitted, the handler is called with no
arguments.

### 5.4 Handlers vs Declarative Actions

Use declarative actions (`state.set`, `state.toggle`, etc.) when the interaction is
simple — a single state change with no conditional logic. Use handlers when you need:

- Multiple state changes in sequence
- Conditional logic (if X changed, maybe reset Y)
- Data lookups (find an item by criteria)
- Side effects (scroll, focus, fetch)
- Complex computations before updating state

```
Simple: state.toggle → use declarative action in template
Complex: select option → validate combination → maybe reset → update display → use handler
```

Handlers and declarative actions coexist. A template can use `state.toggle` for an
accordion and `handler.onSelect` for complex selection logic — in the same section.

### 5.5 Async Handlers

Handlers can be async. This is the pattern for data fetching:

```javascript
export default function SearchResults({ state, actions }) {
    let debounceTimer = null;

    const onSearchInput = (query) => {
        actions.state.set('query', query);
        clearTimeout(debounceTimer);
        debounceTimer = setTimeout(() => performSearch(query), 300);
    };

    const performSearch = async (query) => {
        if (query.length < 2) {
            actions.state.set('results', []);
            return;
        }
        actions.state.set('loading', true);
        try {
            const response = await fetch(`/api/search?q=${encodeURIComponent(query)}`);
            const data = await response.json();
            actions.state.set('results', data.results);
        } catch (error) {
            actions.state.set('error', error.message);
        } finally {
            actions.state.set('loading', false);
        }
    };

    return {
        computed: {},
        handlers: { onSearchInput },
        onUnmount() { clearTimeout(debounceTimer); }
    };
}
```

The template uses `state.loading`, `state.results`, and `state.error` to show the
appropriate UI. Async state is managed explicitly via `state.*` — the developer
controls when loading starts, when results arrive, and when errors occur. No special
async binding syntax.


---

## 6. Lifecycle Hooks

Lifecycle hooks run at specific points in a template instance's lifetime.

### 6.1 `onMount`

Called once after the template instance is rendered and DOM refs are available.
Client-side only — never called during SSR.

Use `onMount` for:
- Initializing third-party libraries that need a DOM element
- Adding event listeners not expressible via JTOX triggers
- Starting animations or observers
- Any setup that requires the DOM

```javascript
import SomeLibrary from 'some-carousel-library';

export default function ImageCarousel({ refs }) {
    let instance = null;

    return {
        computed: {},
        handlers: {},
        onMount() {
            instance = new SomeLibrary(refs.carousel, {
                slidesPerView: 1,
                pagination: { el: refs.pagination }
            });
        },
        onUnmount() {
            instance?.destroy();
            instance = null;
        }
    };
}
```

### 6.2 `onUnmount`

Called once before the template instance is removed from the DOM. Client-side only.

Use `onUnmount` for:
- Destroying third-party library instances
- Removing event listeners
- Clearing timers (`clearTimeout`, `clearInterval`)
- Cancelling pending network requests
- Any cleanup to prevent memory leaks

**Rule:** Every resource acquired in `onMount` or handlers SHOULD be released in
`onUnmount`. Timers, observers, event listeners, and library instances that are not
cleaned up will leak memory.

### 6.3 No `onPropsChange` Hook

Computed values automatically re-evaluate when their dependencies (props, state, context)
change. There is no explicit `onPropsChange` hook.

If a developer needs to reset state when props change (e.g., clear selections when
navigating to a different entity), the renderer may re-instantiate the script when the
primary data changes. This is a renderer implementation detail. The spec does not
prescribe when the script function is re-called — only that computed values reflect
current data.


---

## 7. Which Templates Support Scripts

Not every template type supports scripts. Scripts are for templates that render
interactive content.

| Template Type | Supports `script.js` | Rationale |
|---|---|---|
| Section | ✅ Yes | Primary interactive unit. Selection UIs, forms, search, filtering. |
| Component | ✅ Yes | Reusable interactive patterns. Carousels, search inputs, date pickers. |
| Overlay | ✅ Yes | Self-contained interactive units. Drawers, modals, menus. |
| Block | ❌ No | Simple content display. Promote to component if interactivity is needed. |
| Page | ❌ No | Section list with governance rules. No content, no behavior. |
| Layout | ❌ No | Pure structure (slots). No content, no data. |
| Header / Footer | ❌ No | Regions that contain sections. The sections inside them have the scripts. |

### Theme Utility Modules

Themes can include shared JavaScript modules in a `utils/` directory:

```
{theme-name}/
├── utils/
│   ├── helpers.js
│   └── validation.js
├── sections/
│   └── my-section/
│       ├── template.json
│       └── script.js       # imports from ../../utils/helpers.js
```

Utility modules are importable by any script in the theme:

```javascript
import { validateEmail } from '../../utils/validation.js';
```

Utility modules do NOT export a default function and are NOT called by the renderer.
They are standard JavaScript modules imported by scripts.

---

## 8. Callback Props

When a section uses a component via `$ref`, the section may need to know about
interactions inside the component. The pattern is callback props — the section passes
handler references as props, and the component's template invokes them.

### 8.1 Section Passes Handler as Prop

```json
{
  "type": "component",
  "$ref": "/components/option-picker",
  "props": {
    "options": { "$bind": "computed.availableOptions" },
    "selected": { "$bind": "state.selectedOption" },
    "onSelect": "handler.onOptionSelect"
  }
}
```

### 8.2 Component Template Invokes the Callback

```json
{
  "$jtox": "1.0",
  "type": "fragment",
  "children": [
    {
      "type": "each",
      "source": { "$bind": "props.options" },
      "as": "option",
      "children": [
        {
          "type": "button",
          "attributes": {
            "action": { "$bind": "props.onSelect" },
            "payload": { "args": [{ "$bind": "scope.option" }] }
          },
          "children": [
            { "type": "text", "content": { "$bind": "scope.option" } }
          ]
        }
      ]
    }
  ]
}
```

When the user clicks an option, the renderer resolves `props.onSelect` to
`"handler.onOptionSelect"` and calls the section's handler with the option value.

### 8.3 When Components Need Their Own Scripts

Most components are presentational — they receive data and callbacks via props. A
component needs its own `script.js` only when it has self-contained interactive logic
independent of the parent:

- A carousel with keyboard navigation and slide tracking
- A search input with debounced typing and local suggestion state
- A date picker with calendar navigation logic

When a component has its own script, communication with the parent still happens
through callback props:

```javascript
// components/search-input/script.js
export default function SearchInput({ state, actions, props }) {
    let timer = null;

    const onInput = (value) => {
        actions.state.set('query', value);
        clearTimeout(timer);
        timer = setTimeout(() => {
            if (props.onSearch) props.onSearch(value);
        }, 300);
    };

    return {
        computed: {},
        handlers: { onInput },
        onUnmount() { clearTimeout(timer); }
    };
}
```


---

## 9. SSR Considerations

Scripts run in both server and client environments. The behavior differs.

### 9.1 Server-Side Rendering

On the server:

1. The script's default function IS called (computed values are needed for initial HTML)
2. `state` values are their declared defaults
3. `refs` is an empty object (no DOM on the server)
4. `computed` values are evaluated with default state → produce initial render values
5. `handlers` are registered but never invoked (no user interaction on the server)
6. `onMount` is NOT called (no DOM)
7. `onUnmount` is NOT called

The server-rendered output reflects the initial state. This is the same behavior as
`state.*` bindings without scripts (see [Actions & State §9](11-actions.md)).

### 9.2 Client-Side Hydration

On the client:

1. The script's default function is called with live state
2. `refs` are populated with DOM element references
3. `computed` values become reactive — they re-evaluate when dependencies change
4. `handlers` become active — user interactions invoke them
5. `onMount` is called (DOM is ready)

### 9.3 Browser API Access

Scripts have full access to browser APIs (`document`, `window`, `fetch`, `setTimeout`,
etc.). However, these APIs do not exist on the server. Scripts that access browser APIs
outside of `onMount` or handlers MUST guard against server execution:

```javascript
// ❌ Crashes on server
const width = window.innerWidth;

// ✅ Safe — only runs on client
const onMount = () => {
    const width = window.innerWidth;
};

// ✅ Safe — guard with typeof check
const getWidth = () => {
    if (typeof window === 'undefined') return 0;
    return window.innerWidth;
};
```

**Rule of thumb:** Computed values should be pure functions of `props`, `state`,
`settings`, and `context` — no browser APIs. Browser APIs belong in `onMount`,
`onUnmount`, and handlers, which only run on the client.

---

## 10. External Dependencies

Scripts MAY import external JavaScript libraries. Themes that use external dependencies
include a standard `package.json` at the theme root:

```
{theme-name}/
├── package.json
├── theme.json
├── sections/
│   └── my-section/
│       └── script.js       ← import SomeLib from 'some-lib'
└── utils/
```

```json
{
  "name": "my-theme",
  "private": true,
  "dependencies": {
    "some-carousel-lib": "^3.0.0"
  }
}
```

Standard package manager commands (`npm install`, `yarn`, `pnpm`) resolve dependencies.
Scripts import using standard ES module syntax:

```javascript
import SomeLib from 'some-carousel-lib';
```

Themes without external dependencies do not need a `package.json`. Most themes will
fall into this category — the built-in `state.*` actions, transforms, and computed
values handle the majority of use cases.

### 10.1 Bundle Considerations

External libraries are included in the theme's JavaScript bundle. Theme authors should
be mindful of bundle size:

- Import only what you need (tree-shakeable ESM imports)
- Consider whether a small utility in `utils/` could replace a library
- Platforms MAY enforce bundle size limits as part of theme submission requirements

### 10.2 What NOT to Import

Some categories of libraries conflict with the JTOX rendering model:

| Category | Why Not |
|---|---|
| UI frameworks (React, Vue, Svelte) | The renderer IS the framework — themes don't bring their own |
| State management (Redux, MobX) | JTOX `state.*` and computed values handle state |
| Routing libraries | The platform handles routing |
| CSS-in-JS (styled-components) | Themes use CSS files, not JS-generated styles |

These are guidelines, not enforced restrictions. But importing a UI framework into a
script would conflict with the renderer's own framework and cause unpredictable behavior.


---

## 11. Complete Example: Filterable List with Selection

This example shows a section with a search filter, a selectable list, and a detail
display — demonstrating computed values, handlers, refs, and state working together.

### 11.1 Script

```javascript
// sections/filterable-list/script.js

export default function FilterableList({ props, state, actions, refs }) {

    // --- Computed values ---

    const filteredItems = () => {
        let items = props.items;
        if (state.filterText) {
            const query = state.filterText.toLowerCase();
            items = items.filter(item =>
                item.title.toLowerCase().includes(query)
            );
        }
        if (state.activeCategory) {
            items = items.filter(item => item.category === state.activeCategory);
        }
        return items;
    };

    const categories = () => {
        return [...new Set(props.items.map(item => item.category))];
    };

    const selectedItem = () => {
        if (!state.selectedId) return null;
        return props.items.find(item => item.id === state.selectedId) || null;
    };

    const resultSummary = () => {
        const total = props.items.length;
        const shown = filteredItems().length;
        if (shown === total) return `${total} items`;
        return `${shown} of ${total} items`;
    };

    // --- Handlers ---

    const onFilterInput = (text) => {
        actions.state.set('filterText', text);
    };

    const onCategorySelect = (category) => {
        // Toggle: clicking the active category clears it
        if (state.activeCategory === category) {
            actions.state.set('activeCategory', '');
        } else {
            actions.state.set('activeCategory', category);
        }
    };

    const onItemSelect = (id) => {
        actions.state.set('selectedId', id);
        // Scroll the detail panel into view
        refs.detailPanel?.scrollIntoView({ behavior: 'smooth' });
    };

    return {
        computed: { filteredItems, categories, selectedItem, resultSummary },
        handlers: { onFilterInput, onCategorySelect, onItemSelect }
    };
}
```

### 11.2 Template

```json
{
  "$jtox": "1.0",
  "type": "section",
  "meta": { "id": "filterable-list", "name": "Filterable List" },
  "state": {
    "filterText": { "type": "string", "default": "" },
    "activeCategory": { "type": "string", "default": "" },
    "selectedId": { "type": "string", "default": "" }
  },
  "children": [
    {
      "type": "container",
      "styles": ["list-controls"],
      "children": [
        {
          "type": "input",
          "attributes": {
            "inputType": "search",
            "placeholder": "Filter...",
            "action": "handler.onFilterInput",
            "payload": { "args": [{ "$bind": "self.value" }] },
            "trigger": "change"
          }
        },
        {
          "type": "each",
          "source": { "$bind": "computed.categories" },
          "as": "cat",
          "children": [
            {
              "type": "button",
              "styles": ["category-pill"],
              "attributes": {
                "action": "handler.onCategorySelect",
                "payload": { "args": [{ "$bind": "scope.cat" }] }
              },
              "children": [
                { "type": "text", "content": { "$bind": "scope.cat" } }
              ]
            }
          ]
        },
        {
          "type": "span",
          "styles": ["result-count"],
          "children": [
            { "type": "text", "content": { "$bind": "computed.resultSummary" } }
          ]
        }
      ]
    },
    {
      "type": "each",
      "source": { "$bind": "computed.filteredItems" },
      "as": "item",
      "children": [
        {
          "type": "button",
          "styles": ["list-item"],
          "attributes": {
            "action": "handler.onItemSelect",
            "payload": { "args": [{ "$bind": "scope.item.id" }] }
          },
          "children": [
            { "type": "text", "content": { "$bind": "scope.item.title" } }
          ]
        }
      ],
      "empty": [
        { "type": "text", "content": "No items match your filter." }
      ]
    },
    {
      "type": "conditional",
      "test": { "$bind": "computed.selectedItem" },
      "children": [
        {
          "type": "container",
          "styles": ["detail-panel"],
          "attributes": { "ref": "detailPanel" },
          "children": [
            {
              "type": "heading",
              "level": 2,
              "children": [
                { "type": "text", "content": { "$bind": "computed.selectedItem.title" } }
              ]
            },
            {
              "type": "paragraph",
              "children": [
                { "type": "text", "content": { "$bind": "computed.selectedItem.description" } }
              ]
            }
          ]
        }
      ]
    }
  ]
}
```

### 11.3 Interaction Flow

```
1. Page loads → SSR renders with default state
   → filterText = "", activeCategory = "", selectedId = ""
   → filteredItems() returns all items, categories() returns all categories
   → No item selected, detail panel hidden

2. Client hydrates → script called, computed values become reactive

3. User types "widget" → handler.onFilterInput("widget")
   → state.filterText = "widget"
   → filteredItems() re-evaluates → returns items matching "widget"
   → resultSummary() re-evaluates → "3 of 12 items"
   → List re-renders with filtered results

4. User clicks "Electronics" category → handler.onCategorySelect("Electronics")
   → state.activeCategory = "Electronics"
   → filteredItems() re-evaluates → items matching "widget" AND "Electronics"
   → resultSummary() updates

5. User clicks an item → handler.onItemSelect("item-42")
   → state.selectedId = "item-42"
   → selectedItem() re-evaluates → returns the item object
   → Detail panel appears (conditional test becomes truthy)
   → refs.detailPanel.scrollIntoView() scrolls it into view
```


---

## 12. Renderer Requirements for Scripts

### MUST

1. Load script modules and call the default export function per template instance
2. Pass the context object (`props`, `state`, `actions`, `refs`, `context`, `settings`)
3. Wire `computed.*` bindings to the returned computed functions
4. Wire `handler.*` actions to the returned handler functions
5. Re-evaluate computed values when their dependencies change
6. Re-render template nodes that bind to changed computed values
7. Call `onMount` after client-side rendering (not during SSR)
8. Call `onUnmount` before removing the template instance from the DOM
9. Populate `refs` with DOM element references after mount
10. Handle missing scripts gracefully — a template that references `computed.*` or
    `handler.*` without a script file SHOULD log a warning and render with null values

### SHOULD

1. Implement dependency tracking for computed values (only re-evaluate when
   dependencies actually change, not on every state change)
2. Memoize computed values within a render cycle (if a computed value is bound in
   two places, call the function once)
3. Batch state updates from handlers (multiple `actions.state.set` calls in one
   handler should trigger one re-render, not multiple)
4. Log warnings when a template references a `computed` or `handler` name that
   the script does not export
5. Support the `ref` attribute on any element node type

### MAY

1. Re-call the script function when props change significantly to reset closures
2. Implement hot module replacement for scripts during development
3. Provide TypeScript type definitions for the context object
4. Implement computed value caching across render cycles

---

## 13. Validation

The validator can check the template-to-script contract without executing the script:

### Level 1: Structural

- If the template contains `computed.*` bindings or `handler.*` actions, does a
  `script.js` file exist in the same directory?
- Does the script file export a default function? (static analysis of the export)

### Level 2: Contract

- Do `computed.*` binding names in the template match exported computed property names?
- Do `handler.*` action names in the template match exported handler property names?
- Are there computed values or handlers in the script that the template never references?
  (warning — unused exports)

### Level 3: Quality (Optional Warnings)

- Does the script have an `onUnmount` if it uses `setTimeout`, `setInterval`, or
  `addEventListener` in `onMount`? (potential memory leak)
- Does the script access `document` or `window` outside of `onMount`/handlers?
  (potential SSR crash)
- Does the script import large libraries? (bundle size warning)

Levels 1-2 are recommended for all validators. Level 3 is optional and advisory.

---

## 14. State Type Extension

Scripts introduce a need for complex state values that the original `state` system
(see [Actions & State §4](11-actions.md)) did not anticipate. The original state types
are `"string"`, `"number"`, and `"boolean"` — sufficient for toggles and counters but
not for script-managed state like a selections map or a search results array.

### 14.1 Extended State Types

When a template has an associated script, the following additional state types are
available:

| Type | Default Value | Description |
|---|---|---|
| `"object"` | `{}` | A key-value map. Used for dynamic structures like selection state. |
| `"array"` | `[]` | An ordered list. Used for search results, filtered items, etc. |

These types are ONLY available when the template has a `script.js` file. Templates
without scripts continue to use the original three types (`string`, `number`, `boolean`).

**Rationale:** Object and array state values are managed by script handlers, not by
declarative `state.*` actions. A `state.set('selections', { a: '1', b: '2' })` call
from a handler is clear and intentional. Allowing `state.set` on objects from
declarative template actions would be confusing — what does `state.toggle` mean on an
object? By restricting these types to scripted templates, the declarative path stays
simple and the scripted path stays powerful.

### 14.2 Declarative Actions on Extended Types

| Action | `"object"` | `"array"` |
|---|---|---|
| `state.set` | ✅ Replaces the entire object | ✅ Replaces the entire array |
| `state.toggle` | ❌ Not applicable | ❌ Not applicable |
| `state.increment` | ❌ Not applicable | ❌ Not applicable |
| `state.decrement` | ❌ Not applicable | ❌ Not applicable |

Only `state.set` works on object and array types. For fine-grained mutations, the
handler builds the new value and calls `state.set` with the complete replacement:

```javascript
// Update a key in an object
const selections = { ...state.selections, color: 'Blue' };
actions.state.set('selections', selections);

// Add an item to an array
const items = [...state.items, newItem];
actions.state.set('items', items);
```

This immutable update pattern ensures the renderer can detect changes by reference
comparison.

---

## 15. Design Decisions

### DD-012: Scripts as separate files, not embedded in JSON

**Decision:** Behavior lives in `script.js`, not inside `template.json`.

**Rationale:**
- JSON cannot contain executable code without string-encoding it
- Separate files maintain the separation of concerns (structure / style / behavior)
- JavaScript tooling (linters, formatters, type checkers) works on `.js` files
- The pattern mirrors CSS — styles are in `styles.css`, not embedded in JSON
- Each file can be edited, tested, and versioned independently

**Tradeoff:** Two files to coordinate (template references script exports by name).
The validator checks this contract to catch mismatches early.

### DD-013: Computed values as functions, not reactive declarations

**Decision:** Computed values are plain functions that the renderer calls, not
reactive primitives declared in a special syntax.

**Rationale:**
- Functions are the simplest JavaScript construct — every developer knows them
- The renderer controls reactivity (when to call, when to cache, when to re-evaluate)
- No dependency on a specific reactivity library (MobX, Vue reactivity, Solid signals)
- Functions are testable — pass mock data, assert on return value
- The spec stays renderer-agnostic (P1)

**Tradeoff:** The renderer must implement dependency tracking. This is a renderer
implementation concern, not a theme developer concern.

### DD-014: Full JavaScript access (no sandbox)

**Decision:** Scripts have unrestricted access to JavaScript APIs, DOM, and external
libraries. No sandbox, no forbidden API list.

**Rationale:**
- The script contract (`{ computed, handlers, onMount, onUnmount }`) is the
  abstraction boundary — it defines what the renderer consumes
- Sandboxing adds significant renderer complexity for marginal safety gains
- CSS already has unrestricted access to presentation — JS gets the same trust level
- Quality control happens at theme review, not at runtime
- The contract is designed to be sandboxable in a future spec version if needed —
  the function signature becomes the sandbox API surface

**Tradeoff:** A careless script can break the page. Platforms mitigate this through
theme review processes and content security policies. The contract is forward-compatible
with sandboxing.

### DD-015: No async computed values

**Decision:** Computed values must be synchronous. Async data fetching uses handlers
that update state explicitly.

**Rationale:**
- Computed values are called during render — the renderer needs values immediately
- Async computed would require `$loading`/`$value`/`$error` sub-bindings (3 new concepts)
- Explicit async via handlers + state uses only existing concepts (`state.*`)
- The developer has full control over loading/error UX
- Simpler mental model: computed = derive, handler = do things

**Tradeoff:** Async patterns are more verbose (3 state variables instead of 1 async
computed). If this proves too painful in practice, async computed can be added in a
future minor version as a convenience layer on top of the existing model.