# SkillCatalog UI Style Specification

This document describes the light, mist-blue and indigo/cyan interface currently
used in SkillCatalog. It is intended as a portable visual implementation
reference for another product or workplace. The palette is a product design
choice, not an official brand specification.

## 1. Design intent

- Create a calm, focused workspace for browsing and authoring reusable skills.
- Use a light, cool-neutral canvas with white content surfaces.
- Use indigo for primary actions, selected navigation, and key emphasis.
- Use cyan sparingly for secondary emphasis, data visualization, and small AI
  motifs.
- Keep contrast, content hierarchy, and usability ahead of decorative effects.
- Keep page-canvas treatments separate from sidebar, header, card, and control
  surfaces.

## 2. Color tokens

Use semantic tokens rather than repeating raw colors throughout a new
implementation. These values represent the current SkillCatalog styling.

| Token | Value | Intended use |
|---|---|---|
| `--canvas` | `#F1F5F9` | Overall page background base |
| `--paper` | `#F7F8FC` | Light page/paper surface |
| `--paper-deep` | `#EEF1F8` | Slightly deeper tinted surface |
| `--surface` | `#F0F2F8` | Secondary surface |
| `--surface-raised` | `#FFFFFF` | Cards and elevated content |
| `--surface-hover` | `#F0F3FB` | Subtle hovered surface |
| `--border` | `#E0E5EF` | Default border and divider |
| `--text-primary` | `#202541` | Headings and primary text |
| `--text-secondary` | `#56617A` | Body copy and supporting text |
| `--text-muted` | `#727D96` | Secondary labels and metadata |
| `--accent-violet` | `#554BC2` | Indigo emphasis, active states |
| `--accent-cyan` | `#1285A3` | Cyan highlights and secondary accent |
| `--green` | `#3158BD` | Legacy semantic accent token; visually used as blue/indigo |
| `--green-dark` | `#4F46B8` | Darker primary action token |
| `--green-pale` | `#E9EAFF` | Pale indigo selected/icon surface |
| `--orange` | `#1285A3` | Legacy token currently mapped to cyan |
| `--white` | `#FFFFFF` | White surface token |

### Page background treatment

The canvas is not a flat fill. The current implementation uses a low-contrast,
cool gradient with large, blurred indigo and cyan radial washes. Keep these
effects subtle enough that text and white cards remain the focus.

```css
body {
  background:
    radial-gradient(
      ellipse 56rem 38rem at 88% 2%,
      rgba(85, 75, 194, 0.035),
      transparent 72%
    ),
    radial-gradient(
      ellipse 48rem 34rem at 7% 38%,
      rgba(18, 133, 163, 0.03),
      transparent 74%
    ),
    linear-gradient(
      145deg,
      #edf3f8 0%,
      #f4f7fa 52%,
      #eef4f8 100%
    );
}
```

The `--canvas` token is `#F1F5F9`. The gradient provides a slightly lighter,
softly varied page treatment. Do not apply the page gradient to cards or the
sidebar.

### Surface and accent guidance

- Use white for cards, editors, and other primary content panels.
- Use very pale blue-gray or lavender-gray for navigation and control groups.
- Use thin cool-gray borders; avoid heavy outlines or high-contrast shadows.
- Use indigo for selected navigation, active filters, links, and prominent
  actions.
- Use cyan as a complementary accent, not as a second competing primary.
- Reserve green, amber, and red status colors for clear success, pending, and
  error meanings.

## 3. Typography

| Role | Typeface | Guidance |
|---|---|---|
| Main interface | DM Sans, with Segoe UI/system sans-serif fallback | Use for body copy, headings, buttons, labels |
| Metadata and technical content | DM Mono, with Consolas/monospace fallback | Use for small labels, keyboard hints, counts, and skill source |
| Display/serif token | DM Sans in the current theme | Keep consistent with the sans-serif interface |

Suggested hierarchy based on the current interface:

- Hero heading: 46 px / 1.13 line-height, weight 500, tight letter spacing.
- Page heading: about 36 px; reduce to 32 px on narrow screens.
- Section heading: around 22–24 px.
- Card heading: around 14–16 px, weight 600.
- Body copy: 12–14 px with 1.6–1.75 line-height.
- Labels and metadata: 9–11 px; use monospaced text selectively.

Use `font-synthesis: none`, `text-rendering: optimizeLegibility`, and
`-webkit-font-smoothing: antialiased` where appropriate. If external font
delivery is not permitted, use the listed system fallbacks.

## 4. Layout and spacing

- Desktop sidebar: fixed left rail, 244 px wide; reduce to about 218 px at
  tablet widths.
- Main content: centered, maximum width about 1600 px.
- Standard page content: maximum width about 1240 px, with 42 px horizontal
  padding on desktop.
- Sticky top bar: 68 px high, translucent light surface, subtle lower border,
  and backdrop blur.
- Hero panel: two-column composition on wide screens, with text and a restrained
  abstract motif; use a 17 px corner radius.
- Content cards: generally 11–12 px corner radius, white background, thin
  border, and restrained hover shadow.
- Buttons: 40 px minimum height, 8 px corner radius, clear primary/secondary
  hierarchy.
- Inputs and selects: approximately 38–39 px high with a 6–7 px radius.
- Use a consistent spacing scale based on 4 px increments. Larger section gaps
  should be visibly greater than card-internal spacing.

Avoid making every surface a card. Use whitespace, alignment, and subtle
dividers to group related content.

## 5. Component styling

### Navigation

- Light tinted sidebar, separated from the page by a thin border.
- Muted default link color; hover uses a subtle tinted background.
- Active link uses indigo text, pale indigo fill, and a narrow indigo leading
  indicator.
- Keep labels and counts quieter than the primary navigation text.

### Buttons

- Primary: indigo-to-blue/cyan gradient with white text; modest shadow.
- Secondary: white fill, cool-gray border, dark muted text.
- Quiet: transparent surface, tinted hover fill.
- Hover: darken the accent slightly and raise by at most 1 px when appropriate.
- Disabled: reduce opacity and clearly communicate the unavailable state.

### Cards and panels

- White surface, cool-gray border, 11–12 px radius.
- Use light elevation on hover rather than a permanent heavy shadow.
- Keep card headings in primary text color and descriptions in secondary color.
- Tags use pale indigo fills and darker indigo text.

### Forms and controls

- White or near-white input backgrounds with thin cool-gray borders.
- On focus, use a clear indigo/cyan border and a soft 3 px focus halo.
- Keep labels above controls; use muted hint text below or beside them.
- Do not use color alone to communicate validation or status.

### Hero and AI/network motif

- Use an off-white, cool-tinted panel with a low-opacity indigo/cyan glow.
- Keep the abstract orbit/network illustration low-contrast and decorative.
- Avoid robot imagery, neon glow, and animation that competes with the content.

## 6. Responsive behavior

Use these breakpoints as reference points, adapting them to the target product's
content:

- At `1100px` and below, reduce page gutters and hero-panel spacing.
- At `850px` and below, narrow the sidebar, simplify hero artwork, and reduce
  multi-column content layouts.
- At `640px` and below:
  - Replace the fixed sidebar with an off-canvas navigation drawer.
  - Remove the desktop left margin from the main column.
  - Reduce top-bar and page padding.
  - Stack hero content and card grids into one column.
  - Stack detail panels and editor metadata fields.
  - Keep controls usable without horizontal scrolling.

## 7. Interaction, accessibility, and motion

- Provide a visible `:focus-visible` outline, 2 px wide, with a 3 px offset;
  use an indigo tone such as `#536FE0`.
- Maintain readable text/background contrast, especially for muted labels and
  small monospaced metadata.
- Give interactive controls clear hover, focus, disabled, and selected states.
- Pair semantic status colors with visible text or icons.
- Respect `prefers-reduced-motion`; disable or shorten transitions and
  animations for users who request reduced motion.
- Keep transitions short (roughly 150–200 ms) and avoid unnecessary movement.
- Ensure the layout remains usable with keyboard navigation and zoom.

## 8. Starter CSS tokens

The following is a compact starting point for reusing the theme. Extend the
tokens to match the target application's components rather than copying
SkillCatalog-specific selectors.

```css
:root {
  color-scheme: light;
  font-family: "DM Sans", "Segoe UI", sans-serif;
  --canvas: #f1f5f9;
  --surface: #f0f2f8;
  --surface-raised: #ffffff;
  --surface-hover: #f0f3fb;
  --border: #e0e5ef;
  --text-primary: #202541;
  --text-secondary: #56617a;
  --text-muted: #727d96;
  --accent-indigo: #554bc2;
  --accent-cyan: #1285a3;
  --focus-ring: #536fe0;
}

body {
  min-width: 320px;
  min-height: 100vh;
  color: var(--text-primary);
  background:
    radial-gradient(
      ellipse 56rem 38rem at 88% 2%,
      rgba(85, 75, 194, 0.035),
      transparent 72%
    ),
    radial-gradient(
      ellipse 48rem 34rem at 7% 38%,
      rgba(18, 133, 163, 0.03),
      transparent 74%
    ),
    linear-gradient(
      145deg,
      #edf3f8 0%,
      #f4f7fa 52%,
      #eef4f8 100%
    );
}

:focus-visible {
  outline: 2px solid var(--focus-ring);
  outline-offset: 3px;
}

@media (prefers-reduced-motion: reduce) {
  *,
  *::before,
  *::after {
    scroll-behavior: auto !important;
    animation-duration: 0.01ms !important;
    animation-iteration-count: 1 !important;
    transition-duration: 0.01ms !important;
  }
}
```

## 9. Application notes

- The style guide is framework-agnostic; translate the tokens to the target
  application's CSS variables, design tokens, or theme system.
- Treat the color tokens as a starting point and check contrast in the actual
  components and states where they are used.
- Keep content, product behavior, and data independent of this presentation
  layer.
- The current implementation is in `src/styles.css`; this document extracts
  its visual direction and principal values for reuse.
