# Accessibility (a11y): L4 Engineering Guide

## Part 1: WCAG & Accessibility Standards

### 1.1 WCAG 2.1 Compliance Levels

**Three levels of conformance:**

- **Level A (Minimum)**: Satisfies basic accessibility needs
- **Level AA (Recommended)**: Addresses most accessibility concerns
- **Level AAA (Enhanced)**: Highest level of accessibility

Most enterprises target **Level AA** as the standard.

### 1.2 Four Core Principles (POUR)

1. **Perceivable**: Information/UI must be perceivable to all users
2. **Operable**: Components must be navigable via keyboard
3. **Understandable**: Content and navigation must be clear
4. **Robust**: Content must be compatible with assistive technologies

---

## Part 2: Keyboard Navigation & Focus Management

### 2.1 Keyboard Navigation: The Foundation of Accessibility

**Core principle**: All functionality must be accessible via keyboard alone. Some users cannot use a mouse (motor disabilities, assistive technology restrictions). Keyboard navigation means:

1. **Tab order follows HTML source order** (or explicit `tabindex="0"` override)
2. **Enter/Space activate buttons**
3. **Arrow keys navigate select elements**
4. **Escape closes modals**

**Why tabindex matters**: Natural HTML elements (`<button>`, `<a>`, `<input>`) are keyboard accessible by default. Custom elements (divs, spans) are not. Setting `tabindex="0"` makes div focusable but does NOT add keyboard handlers—you must add `@keydown` listeners manually.

**Common mistake**: Using positive tabindex values (e.g., `tabindex="5"`) creates confusing tab order. Tab order should match visual left-to-right, top-to-bottom reading order. Use `tabindex="0"` (natural order) or `-1` (programmatic focus only, not in tab order).

**Focus trap in modals**: When modal opens, focus must be trapped within modal (Tab cycles through modal elements, not background page). On close, focus returns to element that triggered modal. This requires JavaScript to track focusable elements and prevent Tab from escaping.

---

## Part 3: Color Contrast & WCAG Formulas

### 3.1 Color Contrast: WCAG Formula and Calculation

**Contrast Ratio**: Measures readability for visually impaired users. Calculated from relative luminance:

$$\text{Contrast Ratio} = \frac{L_{\text{lighter}} + 0.05}{L_{\text{darker}} + 0.05}$$

Where **relative luminance** for sRGB is:

$$L = 0.2126 \cdot R_{\text{linear}} + 0.7152 \cdot G_{\text{linear}} + 0.0722 \cdot B_{\text{linear}}$$

And each RGB channel applies gamma correction:

$$C_{\text{linear}} = \begin{cases} \frac{C_{\text{sRGB}}}{12.92} & \text{if } C_{\text{sRGB}} \leq 0.03928 \\ \left(\frac{C_{\text{sRGB}} + 0.055}{1.055}\right)^{2.4} & \text{otherwise} \end{cases}$$

**WCAG Compliance Thresholds:**

- **Normal text**: 4.5:1 (AA), 7:1 (AAA)
- **Large text** (18pt+ or 14pt bold): 3:1 (AA), 4.5:1 (AAA)

**Real-world example**: Blue #0066cc on white #ffffff has contrast ratio ~5.6:1 (passes AA for normal text, fails AAA). Gray #999999 on white has ~4.5:1 (barely passes AA for normal text).

**Color blindness**: 8% of males and 0.5% of females have color blindness. Red-green blindness is most common (protanopia, deuteranopia). Design principle: **Never use color alone to convey information**. Combine with icons, text, or borders.

---

## Part 4: ARIA (Accessible Rich Internet Applications)

### 4.1 ARIA: When and Why

**ARIA Principle**: Add semantic information to non-semantic elements. ARIA doesn't add functionality—it adds labels for assistive technology.

**Three types of ARIA attributes:**

1. **Roles**: Define what element is (`role="button"`, `role="dialog"`, `role="alert"`)
2. **States**: Current condition (`aria-pressed="true"`, `aria-expanded="false"`, `aria-invalid="true"`)
3. **Properties**: Relationships/labels (`aria-label="Close"`, `aria-describedby="help-text"`, `aria-controls="panel"`)

**Critical rule**: Use semantic HTML first. `<button>` automatically has `role="button"` and keyboard handlers. Only use ARIA when no semantic element exists (e.g., custom tab widget).

**Common ARIA misuse**: Adding `aria-label` to a button without making it keyboard accessible doesn't help—screen readers announce it's a button, but keyboard users can't tab to it or activate with Enter/Space.

**Live regions** (`aria-live="polite"` or `aria-live="assertive"`): Screen readers announce dynamically inserted content. Use `aria-atomic="true"` to announce entire region, not just changes. `polite` waits for user to pause; `assertive` interrupts immediately (for errors).

---

## Part 5: Accessible Forms & Validation

### 5.1 Form Accessibility: Labels, Descriptions, and Validation

**Labels are critical**: Every form input requires an associated `<label>`. Label-input association (via `for` attribute) ensures:
1. Screen reader announces label when input focused
2. Clicking label focuses input (enlarged touch target on mobile)
3. Automatic form validation messages announced in context

**Field descriptions** use `aria-describedby` to link input to helper text (e.g., "Password must be 8+ chars"). This is announced after label, providing context without cluttering the label.

**Fieldset & legend** group related inputs (e.g., radio buttons for notification preferences). Legend is announced once per group, preventing repetition.

**Validation strategy**: On form submission, find first invalid input, focus it, display error message in `aria-describedby` element with `role="alert"`. Never just change color—color-blind users miss errors. Use icon + text + color.

---

## Part 6: Motion & Vestibular Disorders

### 6.1 Motion and Vestibular Disorders

**Vestibular disorders** affect balance and spatial orientation. Parallax scrolling, rapid animations, and flashing content trigger vertigo and nausea in affected users.

**`prefers-reduced-motion` media query**: Operating systems expose a user preference for reduced motion. Developers must respect it by:
1. Disabling parallax effects
2. Reducing animation duration to near-instant (0.01ms)
3. Removing animated GIFs or providing static alternatives

**Detection**: Query via CSS media query or JavaScript: `window.matchMedia('(prefers-reduced-motion: reduce)').matches`

**Best practice**: Make animations opt-in for users who want them, not opt-out. Default to no animation; enable only if user hasn't set `prefers-reduced-motion: reduce`.

---

## Interview Questions

**Q1: What's the minimum contrast ratio for AA compliance text?**

4.5:1 for normal text, 3:1 for large text (18pt+ or 14pt bold). Calculated using relative luminance: $\frac{L_{lighter} + 0.05}{L_{darker} + 0.05}$

**Q2: A form field has validation error. Make it accessible without red color alone.**

Use `aria-invalid="true"`, `aria-describedby` pointing to error text, add visual indicator (icon, border), and ensure contrast meets 4.5:1. Never rely on color alone.

**Q3: When should you use ARIA vs semantic HTML?**

Use semantic HTML first (`<button>`, `<nav>`, `<form>`). Only use ARIA when no semantic element exists (custom components). ARIA doesn't add keyboard functionality—it just labels.

**Q4: A user with motion sensitivity visits a parallax scroll website. How do you adapt?**

Check `prefers-reduced-motion: reduce` media query. If set, disable parallax and animations (or set to instant). Use `@media (prefers-reduced-motion: reduce)` to set `animation-duration: 0.01ms` and remove animations.

---

## Key Takeaways

1. **Native semantic elements are keyboard accessible by default** - Use `<button>`, `<a>`, `<input>` before ARIA
2. **Contrast ratio 4.5:1 minimum for AA compliance** - Use $L = 0.2126R + 0.7152G + 0.0722B$ formula
3. **Never use color alone** - Combine with icons, text, borders
4. **Focus management critical for keyboard users** - Trap focus in modals, provide skip links
5. **prefers-reduced-motion must be respected** - Disable parallax, animations for sensitive users
6. **ARIA labels, not functionality** - ARIA doesn't add keyboard support
7. **Test with screen readers** - NVDA (Windows), JAWS, VoiceOver (Mac)

