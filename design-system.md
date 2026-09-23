# Ugarit Design System Architecture

The Ugarit Design System is a modern, token-driven, CSS-first architecture composed of two primary layers: **Primitives** and **Semantic Tokens**, integrated seamlessly with Tailwind CSS v4 via `@theme inline`.

---

## 1. Directory Structure

```text
resources/css/
├── primitives/                 # Core design tokens (raw values & scales)
│   ├── color/                  # 10 HSL color palettes + ink/paper/shadow
│   ├── breakpoint.css          # Fluid viewport media query boundaries
│   ├── type.css                # 17 fluid typography roles with clamp()
│   ├── space.css               # Fluid spatial rhythm
│   ├── size.css                # Fluid dimensions for components
│   ├── radius.css              # Fluid border radius scale
│   ├── border.css              # Stroke widths and styles
│   ├── opacity.css             # Standardized opacity levels
│   ├── motion.css              # Transition durations, easings & animations
│   ├── elevation.css           # Box-shadow layers
│   ├── blur.css                # Backdrop and filter blurs
│   ├── layout.css              # Z-index and layout containers
│   ├── theme.css               # Tailwind v4 @theme inline primitives bridge
│   └── index.css               # Primitives entry manifest
├── semantic/                   # Contextual design tokens (purpose-driven)
│   ├── foundation/             # Mode-aware bases (light.css, dark.css)
│   ├── color/                  # Surface, text, stroke, action, status, icon, overlay
│   ├── type/                   # 17 semantic typography utility classes
│   ├── theme.css               # Tailwind v4 @theme inline semantic bridge
│   └── index.css               # Semantic entry manifest
└── app.css                     # Standard 21-line application bridge
```

---

## 2. Primitives Layer

### Fluid Typography Scale (17 Roles)
Instead of static pixel sizes, all typography roles use CSS `clamp(...)` to scale smoothly across viewports:
- **Display & Headings:** `hero`, `display`, `title-lg`, `title-md`, `title-sm`, `heading-xl`, `heading-lg`, `heading-md`, `heading-sm`, `heading-xs`.
- **Body & Content:** `body-xl`, `body-lg`, `body-md`, `body-sm`.
- **UI Elements:** `label-lg`, `label-md`, `label-sm`, `caption-lg`, `caption-sm`, `overline`.

### Fluid Radius Scale
- `sharp` (0)
- `hairline`
- `subtle`
- `soft` (Standard control radius default: `--radius: var(--radius-soft);`)
- `rounded`
- `pill`
- `circle`

### Color Palettes
10 curated palettes with 11 steps each (50, 100, 200, 300, 400, 500, 600, 700, 800, 900, 950):
- **Brand:** `purple`, `lilac`
- **Functional:** `blue` (info), `emerald` (success), `amber` (warning), `rose` (danger), `cyan` (neutral-cool)
- **Neutrals:** `slate`, `neutral`, `slate-neutral`
- **Substrates:** `ink.css`, `paper.css`, `shadow.css`

---

## 3. Semantic Layer & Chart Support

The semantic layer abstracts color into contextual intents:
- **Surfaces:** `--surface-page`, `--surface-card`, `--surface-elevated`, `--surface-overlay`.
- **Text:** `--text-primary`, `--text-secondary`, `--text-muted`, `--text-disabled`, `--text-on-action`.
- **Actions:** `--action-primary`, `--action-hover`, `--action-active`, `--action-disabled`.
- **Charts:** 8 dedicated high-contrast chart tokens (`--chart-1` through `--chart-8`) for dashboard visualization across light and dark modes.

---

## 4. Tailwind CSS v4 Integration

All tokens are mapped to Tailwind utilities inside `primitives/theme.css` and `semantic/theme.css` using `@theme inline`:
```css
@theme inline {
  --color-primary: var(--primary);
  --color-surface: var(--surface-page);
  --radius-soft: var(--radius-soft);
  --font-hero: var(--text-hero-size);
}
```
This enables classes like `bg-surface`, `text-primary`, `rounded-soft`, and `p-space-md` with zero build-time configuration.
