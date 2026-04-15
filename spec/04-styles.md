# JTOX Styles

Version: 1.0.0-draft

---

## 1. Overview

JTOX separates structure from presentation. Nodes reference styles by **identifier** —
they never define visual properties inline. Style definitions live outside JTOX documents
in platform-specific files (CSS, style objects, native stylesheets, etc.).

This separation is a core principle (P1: Describe, Don't Prescribe). A JTOX document says
"this element uses the 'hero' and 'primary' styles." What those styles look like is the
renderer's concern.

---

## 2. Style References on Nodes

Any node MAY have a `styles` property — an array of style identifier strings:

```json
{
  "type": "container",
  "styles": ["card", "elevated", "rounded"],
  "children": [...]
}
```

### Rules

1. `styles` is always an array, even for a single style: `["hero"]`, not `"hero"`
2. Style identifiers are strings matching the pattern: `[a-z][a-z0-9-]*` (lowercase, hyphens)
3. Order matters — later styles override earlier ones when they conflict
4. An empty array `[]` is valid and means "no styles"
5. Style identifiers are **scoped to the current template** by default

### Scoped vs Global Styles

| Prefix | Scope | Example | Meaning |
|---|---|---|---|
| (none) | Local to the current template | `"card"` | Defined in this section/component's style file |
| `theme/` | Global theme style | `"theme/rounded-corner"` | Defined in the theme's global styles |

```json
{
  "type": "container",
  "styles": ["card", "theme/rounded-corner"],
  "children": [...]
}
```

The `theme/` prefix tells the renderer to look in the theme's global style definitions
instead of the current template's local styles.

---

## 3. Style Settings Bridge

Settings with `intent: "style"` create a bridge between the settings system and the style
layer. When a user changes a color setting, the renderer applies it visually.

**JTOX does not define how this bridge works.** It defines the contract:

1. A setting declares `"intent": "style"` (see [Settings](07-settings.md))
2. The renderer reads the setting value
3. The renderer applies it using its platform's mechanism

**Web example:** A setting `{ "intent": "style", "key": "bg_color", "type": "color", "value": "#fff" }`
becomes CSS custom property `--bg_color: #fff`. The section's CSS references `var(--bg_color)`.

**Native example:** The same setting becomes a style attribute `backgroundColor: "#fff"` on
the native view.

JTOX templates do NOT reference style settings directly. They reference style identifiers.
The style definitions (CSS, etc.) reference the settings. This keeps the JTOX document
clean and platform-agnostic.

```
JTOX template → references style "hero-bg"
Style definition (CSS) → .hero-bg { background-color: var(--bg_color); }
Settings → bg_color: #fff (intent: style)
Renderer → sets --bg_color: #fff as CSS custom property
```

---

## 4. Style File Convention

While JTOX does not define style file formats, it defines the **convention** for where
style files live in a theme package:

```
theme/
├── styles/                    # Global theme styles
│   ├── base.css               # Base/reset styles
│   ├── {preset-name}.css      # Named style presets
│   └── ...
├── sections/{id}/
│   └── styles.css             # Section-local styles
└── components/{id}/
    └── styles.css             # Component-local styles
```

The file format (`.css`, `.json`, `.styles`) is renderer-specific. The directory structure
and naming convention is part of the JTOX theme spec.

---

## 5. Style Identifier Resolution

When a renderer encounters `"styles": ["card", "theme/rounded"]`:

1. `"card"` → Look in the current template's local style file
2. `"theme/rounded"` → Look in the theme's global styles

If a style identifier is not found, the renderer SHOULD:
- Log a warning (in development)
- Silently ignore it (in production)
- NOT throw an error or crash

---

## 6. Responsive Styles

JTOX does not have a responsive/breakpoint system. Responsive behavior is handled entirely
in the style layer (CSS media queries, native layout constraints, etc.).

A JTOX document describes the structure. The styles make it responsive. This is intentional —
responsive behavior is deeply platform-specific and doesn't belong in a structural description.

---

## 7. Animation and Transitions

JTOX does not define animations or transitions. These are style-layer concerns.

A style identifier like `"fade-in"` might trigger a CSS animation. A native renderer might
map it to a platform animation. JTOX doesn't know or care — it just references the style.
