---
name: role-ui
description: Visual interface craft - design tokens, typographic scale, semantic color and dark mode, mandatory component states, the 4pt spacing system, mobile-first breakpoints, and icon rules. Use when writing CSS or styling components, defining a token/theme system, building buttons/inputs/cards, or reviewing a screen's visual consistency and contrast.
---

# Skill: UI Design

## Base principle

The UI is the visual translation of the information architecture. If the UX is wrong, the UI cannot save it — but a bad UI can ruin good UX. The goal is for the user not to notice the design, because it simply works.

---

## Design tokens — the foundation of everything

Never magic values in the code. Everything comes from tokens.

```css
/* ✅ Semantic tokens (not atomic) */
--color-action-primary: #0066CC;
--color-action-primary-hover: #0052A3;
--color-feedback-error: #D93025;
--color-feedback-success: #1E7E34;
--color-surface-default: #FFFFFF;
--color-surface-subtle: #F5F5F5;

--spacing-xs: 4px;
--spacing-sm: 8px;
--spacing-md: 16px;
--spacing-lg: 24px;
--spacing-xl: 40px;

--radius-sm: 4px;
--radius-md: 8px;
--radius-full: 9999px;

--font-size-sm: 0.875rem;   /* 14px */
--font-size-base: 1rem;     /* 16px */
--font-size-lg: 1.125rem;   /* 18px */
--font-size-xl: 1.25rem;    /* 20px */
--font-size-2xl: 1.5rem;    /* 24px */

--shadow-sm: 0 1px 2px rgba(0,0,0,0.05);
--shadow-md: 0 4px 6px rgba(0,0,0,0.07);
--shadow-lg: 0 10px 15px rgba(0,0,0,0.1);
```

---

## Typography

### Scale
- A single font family for the UI (two at most: display + body)
- Modular scale: sizes have a mathematical relationship to each other
- Line-height: 1.5 for body, 1.2 for headings

### Legibility
```
✅ Column width: 60–75 characters per line (prose)
✅ Minimum contrast: 4.5:1 for normal text, 3:1 for large text
✅ Don't center long paragraphs
✅ Hierarchy: at most 3 visible levels per screen
```

---

## Color

### Semantic palette — not decorative
```
Primary:    the user's primary action
Secondary:  secondary / supporting action
Success:    confirmation, completed
Warning:    caution, needs attention
Error:      failure, blocking
Neutral:    text, borders, surfaces
```

### The color rule
- Color is never the only way to communicate state (critical for a11y)
- If you remove the color and the meaning is lost → the design fails
- Always pair it with an icon + text

### Dark mode
```css
/* Tokens with mode */
@media (prefers-color-scheme: dark) {
  :root {
    --color-surface-default: #121212;
    --color-surface-subtle: #1E1E1E;
    --color-text-primary: #E8E8E8;
    /* The action color may shift slightly */
  }
}
```

---

## Components — mandatory states

Every interactive component must have these states designed:

```
Default → Hover → Focus → Active → Disabled → Loading → Error
```

```
Input:   Default | Focus | Filled | Error | Disabled
Button:  Default | Hover | Focus | Active | Loading | Disabled
Card:    Default | Hover (if clickable) | Selected
```

---

## Spacing — 4pt system

Every margin, padding, and gap must be a multiple of 4.

```
4px  — minimum separation between related elements
8px  — separation within a component
16px — separation between components in the same group
24px — separation between sections
40px — separation between major blocks
```

---

## Responsive — mobile first

```css
/* Recommended breakpoints */
--bp-sm: 640px;   /* large phones */
--bp-md: 768px;   /* tablets */
--bp-lg: 1024px;  /* laptops */
--bp-xl: 1280px;  /* desktops */
```

### Rules
- Design first at 375px (iPhone SE), then scale up
- Minimum touch target: 44×44px
- Don't rely on hover for critical functionality (touch has no hover)
- Vertical stacks on mobile, horizontal on desktop

---

## Icons

```
✅ A unified system (Lucide, Heroicons, Phosphor — pick one)
✅ Consistent sizing: 16px, 20px, 24px
✅ Always with a visible label or aria-label
✅ Don't mix styles (outline vs filled)
✅ Status icons: always accompanied by color + text
```

---

## What NOT to do

```
❌ More than 2 fonts in the same product
❌ Colors that are not in the token system
❌ Inconsistent padding (e.g.: 13px, 17px, 22px)
❌ Animations longer than 300ms on frequent interactions
❌ Modals that open modals
❌ Horizontal scroll on mobile
❌ White text over an image with no overlay
❌ Placeholder as the only instruction in a form field
```
