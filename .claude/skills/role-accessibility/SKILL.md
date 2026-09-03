---
name: role-accessibility
description: Web accessibility standards targeting WCAG 2.2 AA - POUR principles, semantic HTML and landmarks, contrast ratios, focus and keyboard support, ARIA attributes, accessible forms, images and media, reduced motion, plus automated and manual testing. Use when building or reviewing UI, HTML/CSS/JSX markup, forms, modals, or running an a11y audit.
---

# Skill: Accessibility (a11y)

## Core principle

Accessibility is not an optional feature or a final checklist. It's a property of the design that, when ignored, excludes users — and in many contexts it's a legal requirement (ADA, WCAG 2.2, EN 301 549).

**Minimum acceptable level: WCAG 2.2 level AA.**

---

## The 4 principles (POUR)

| Principle | What it means |
|-----------|--------------|
| **Perceivable** | Information reaches the senses the user has available |
| **Operable** | The user can navigate and interact with what they have |
| **Understandable** | The content and the UI are predictable and comprehensible |
| **Robust** | It works with current and future assistive technology |

---

## Semantic structure — the most important part

```html
<!-- ❌ Div soup: no semantics -->
<div class="header">
  <div class="nav">
    <div onclick="go()">Home</div>
  </div>
</div>

<!-- ✅ Semantic HTML -->
<header>
  <nav aria-label="Main navigation">
    <a href="/">Home</a>
  </nav>
</header>
```

### Heading order
```html
<!-- ✅ Correct hierarchy — don't skip levels -->
<h1>Page title</h1>
  <h2>Section</h2>
    <h3>Subsection</h3>
```

### Mandatory landmarks
```html
<header>     <!-- site header -->
<nav>        <!-- navigation (aria-label when there are several) -->
<main>       <!-- main content — only one per page -->
<aside>      <!-- complementary content -->
<footer>     <!-- site footer -->
```

---

## Contrast — WCAG 2.2

| Text type | Minimum AA ratio | AAA ratio |
|---------------|----------------|-----------|
| Normal text (<18px / <14px bold) | 4.5:1 | 7:1 |
| Large text (≥18px / ≥14px bold) | 3:1 | 4.5:1 |
| UI components and icons | 3:1 | — |

Tools: [WebAIM Contrast Checker](https://webaim.org/resources/contrastchecker/), Figma Able plugin.

---

## Focus and keyboard

```css
/* ✅ Never remove the outline without replacing it */
:focus-visible {
  outline: 2px solid var(--color-action-primary);
  outline-offset: 2px;
  border-radius: var(--radius-sm);
}

/* ❌ This is an a11y bug */
:focus { outline: none; }
```

### Tab order
- The `Tab` order must follow the visual order
- Modals must trap focus (focus trap) while they're open
- When a modal closes, focus returns to the element that opened it

### Required keys
| Component | Required keys |
|-----------|------------------|
| Button | Enter, Space |
| Link | Enter |
| Checkbox | Space |
| Select/Listbox | ↑↓ to navigate, Enter to select |
| Modal | Esc to close |
| Accordion | Enter/Space to expand |

---

## ARIA — use it only when native HTML isn't enough

```html
<!-- ✅ Native HTML first -->
<button>Save</button>

<!-- ✅ ARIA when the element isn't semantically correct -->
<div role="button" tabindex="0" aria-pressed="false">
  Save
</div>

<!-- Most-used attributes -->
aria-label="description when there is no visible text"
aria-labelledby="id-of-the-element-that-provides-the-name"
aria-describedby="id-of-the-element-with-instructions"
aria-expanded="true|false"     <!-- accordions, dropdowns -->
aria-haspopup="true"           <!-- buttons that open menus -->
aria-live="polite|assertive"   <!-- regions with dynamic content -->
aria-hidden="true"             <!-- hide from screen readers -->
aria-disabled="true"           <!-- disabled (don't use disabled alone) -->
aria-invalid="true"            <!-- field with an error -->
aria-required="true"           <!-- required field -->
```

---

## Forms

```html
<!-- ✅ Always an explicit label -->
<label for="email">Email</label>
<input id="email" type="email" aria-describedby="email-hint email-error">
<p id="email-hint">Use your work email.</p>
<p id="email-error" role="alert" aria-live="assertive">
  <!-- Appears only when there's an error -->
  The email format is not valid.
</p>

<!-- ❌ Placeholder as the only label -->
<input type="email" placeholder="Email">
```

### Error messages
- Appear in real time (not only on submit)
- Describe WHAT is wrong and HOW to fix it
- `role="alert"` or `aria-live="assertive"` so screen readers announce them

---

## Images and media

```html
<!-- ✅ Informative image -->
<img src="chart.png" alt="Sales rose 40% in Q3 2025 compared to Q2">

<!-- ✅ Decorative image -->
<img src="decoration.png" alt="">

<!-- ✅ Icons with no visible text -->
<button aria-label="Close modal">
  <svg aria-hidden="true">...</svg>
</button>
```

### Videos
- Captions for spoken content
- Audio description if there's information that is visual only
- No autoplay with sound

---

## Reduced motion

```css
/* Respect the system preference */
@media (prefers-reduced-motion: reduce) {
  *, *::before, *::after {
    animation-duration: 0.01ms !important;
    transition-duration: 0.01ms !important;
  }
}
```

---

## Accessibility testing

### Automated (catches ~30% of issues)
- **axe DevTools** (Chrome extension) — runs on every PR
- **Lighthouse** — accessibility score as a CI gate

### Mandatory manual pass
```
1. Navigate the whole screen using only Tab
2. Turn on a screen reader: VoiceOver (Mac), NVDA (Win), TalkBack (Android)
3. Zoom to 200% — does the layout still work?
4. Disable CSS — does the content order make sense?
5. Test with keyboard only + mouse only + touch only
```

### In CI (GitHub Actions)
```yaml
- name: Accessibility audit
  run: npx axe-cli http://localhost:3000 --exit
```
