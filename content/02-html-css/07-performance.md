# Performance Optimization: L4 Engineering Guide

## Part 1: Critical Rendering Path

### 1.1 Rendering Pipeline: The Sequential Cost of Each Stage

Browser rendering follows a strict sequential pipeline:

$$\text{HTML Parse} \to \text{CSSOM Build} \to \text{Render Tree} \to \text{Layout} \to \text{Paint} \to \text{Composite}$$

**Computational Complexity at Each Stage:**

1. **HTML Parsing**: $O(n)$ where $n$ = total elements. Streaming HTML parser allows progressive rendering—don't wait for full HTML before rendering.
2. **CSSOM Construction**: $O(m)$ where $m$ = CSS rules. Selector complexity increases cost; descendant selectors (`div p span`) are more expensive than class selectors.
3. **Render Tree**: $O(n \times m)$ worst case—match each element to applicable CSS rules. This is why selector efficiency matters at scale.
4. **Layout**: $O(n)$ to $O(n^2)$ depending on layout engine. Flexbox/Grid are more efficient than float-based layouts; deeply nested layouts increase cost.
5. **Paint**: $O(n)$ proportional to elements, but heavily optimizable with `will-change` and `content-visibility`.
6. **Composite**: $O(L)$ where $L$ = number of composite layers. GPU-intensive but fixed cost per frame.

**Critical insight**: Each stage blocks downstream stages. Slow CSSOM construction delays layout. Slow layout delays paint. Optimizing early stages (HTML/CSS) has multiplicative benefits.

### 1.2 Reflow vs Repaint: The Cost of Layout Changes

**Reflow (Layout Recalculation)**: Browser recalculates element dimensions, positions, and offsets. Triggered by properties that affect layout:
- Dimensions: `width`, `height`, `padding`, `margin`, `border`
- Position: `top`, `left`, `right`, `bottom`, `position`
- Display: `display`, `float`, `flex`, `grid`

Reflow is expensive because changes cascade: if parent width changes, children must recalculate. Affects all dependent elements ($O(n)$ complexity).

**Repaint (Pixel Update)**: Browser updates pixels without recalculating layout. Triggered by visual properties:
- Colors: `color`, `background-color`, `border-color`
- Effects: `box-shadow`, `text-shadow`, `outline`

Repaint is cheaper than reflow (no layout math) but still blocks rendering.

**Layout Thrashing**: Alternating DOM reads and writes (e.g., read `offsetWidth`, write `style.width`) causes multiple reflows. Solution: batch all reads, then all writes.

**Example**:
```
❌ Thrashing: For each item, read width, write new width (N reflows)
✅ Optimal: Read all widths, write all widths (1 reflow)
```

---

## Part 2: Critical Rendering Path Optimization

### 2.1 Render-Blocking Resources: The Critical Path

Render-blocking resources delay initial page rendering because browser waits for them before painting:

- **Render-blocking CSS**: `<link rel="stylesheet">` in `<head>` blocks rendering until downloaded and parsed. This is intentional (prevent FOUC—flash of unstyled content).
- **Render-blocking JavaScript**: `<script>` in `<head>` blocks HTML parsing and rendering. Parser halts until script downloads and executes.

**Non-blocking alternatives**:
- `defer`: Script downloads in parallel; executes after HTML parsing completes
- `async`: Script downloads in parallel; executes immediately when ready (may run during HTML parsing)
- Media queries on CSS: `media="print"` doesn't block rendering for screen users

**Critical rendering path optimization**: Minimize render-blocking resources for above-the-fold content. Inline critical CSS (~14 KB) in `<head>`; defer non-critical CSS for below-fold content.

---

## Part 3: Core Web Vitals

### 3.1 Core Web Vitals: Three Metrics That Matter

Google's Core Web Vitals measure real user experience:

**Largest Contentful Paint (LCP)**: Measures when the largest visible content element (typically an image or text block) finishes rendering.
- Target: < 2.5s (Good)
- Metric range: 0-4s (>4s is Poor)
- Optimization: Remove render-blocking CSS, lazy-load images, optimize server response time

**Interaction to Next Paint (INP)**: Measures delay from user input (click, tap) to visual response.
- Target: < 200ms (Good)
- Metric range: 0-500ms (>500ms is Poor)
- Optimization: Break long JavaScript tasks, defer non-critical work, use `scheduler.yield()`

**Cumulative Layout Shift (CLS)**: Measures unexpected layout changes (e.g., ads loading and pushing content down).
- Target: < 0.1 (Good, near-imperceptible)
- Metric range: 0-1 (>0.25 is Poor)
- Optimization: Reserve space for dynamic content with `aspect-ratio`, avoid inserting content above fold, use `font-display: swap`

**Why these metrics**: They correlate with user satisfaction and conversion rates. Sites with good Core Web Vitals rank higher in Google Search.

---

## Part 4: Font Loading & Optimization

### 4.1 Font Loading Strategy: The font-display Property

**Font-display** controls how browsers render fonts while loading:

| Value | Block Period | Swap Period | Use Case |
|-------|---|---|---|
| `auto` | 0-3s | Indefinite | Browser-dependent behavior |
| `block` | 0-3s | Indefinite | Invisible text for 3s (poor UX) |
| `swap` | 0ms | Indefinite | Show fallback immediately; swap when ready (recommended) |
| `fallback` | 0-100ms | 3s | Show fallback; swap within 3s window if available |
| `optional` | 0-100ms | 0s | Show fallback; swap only if already cached (for repeat visits) |

**Best practice**: Use `swap` for brand-critical fonts (must appear correctly). Use `fallback` for body text (100ms delay acceptable). Avoid `block` (3s invisible text harms LCP).

**Trade-off**: `swap` causes FOIT (flash of incorrect text) when custom font loads, but prevents FOUT (flash of unstyled text) and improves perceived performance. Users see content immediately, then font updates when ready.

**Performance impact**: Font loading is a major LCP bottleneck. Subsetting fonts (Latin only) and using WOFF2 (60-80% smaller than TTF) are essential optimizations.

---

## Part 5: Advanced Optimization Techniques

### 5.1 Content-Visibility and Containment: Rendering Optimization Hints

**`content-visibility: auto`**: Skips rendering of offscreen content entirely. Browser defers rendering until element enters viewport. Provides massive performance improvement for long lists with 1000+ offscreen elements (50-80% rendering reduction).

**Trade-off**: Elements outside viewport have no height computed; you must use `contain` to reserve layout space, or scrollbar becomes inaccurate.

**`will-change`**: Hints to browser that property will change soon (e.g., `will-change: transform`). Browser may create GPU composite layer preemptively. Must be removed after animation completes (`will-change: auto`) to free GPU memory.

**`contain`**: Limits CSS layout/paint scope to subtree, reducing algorithmic complexity from $O(n)$ global to $O(m)$ local. Example: `contain: layout style paint` on `.card` means changing card styles doesn't affect outside page layout.

**Practical optimization**: Use `content-visibility: auto` on below-fold components (images, article cards). Use `contain` on independent sections (cards in a grid, sidebar).

**Browser support**: `content-visibility` is newer (90%+ support); `contain` is older (95%+ support).

---

## Interview Questions

**Q1: Explain the Critical Rendering Path. Why is render-blocking CSS bad?**

CRP: DOM → CSSOM → Render Tree → Layout → Paint → Composite. Render-blocking CSS prevents browser from painting until stylesheet downloads/parses. Solution: Inline critical CSS for above-the-fold, defer non-critical with media queries or async loading.

**Q2: A site has LCP of 3.2s. Main image loads at 1.8s, but DOM interactive is 2.2s. What's bottlenecking?**

LCP measures when largest content element finishes rendering, not just loading. If LCP is 3.2s but image loads at 1.8s, it's render-blocked by JavaScript or CSS. DOM Interactive at 2.2s indicates long parsing/compilation. Solution: Defer non-critical JS, move heavy computations to web workers.

**Q3: Compare will-change vs contain in performance.**

`will-change` proactively hints to browser that property will animate, potentially creating composite layer early. `contain` restricts layout/paint scope to subtree, reducing overall work from $O(n)$ to $O(m)$. `contain` is safer long-term; `will-change` increases memory if misused.

**Q4: Font loads too slowly, causing CLS. Which font-display and why?**

Use `swap` for brand fonts (absolutely must appear, swap immediately when custom font loads), or `fallback` for body text (can wait ~100ms, avoids swap if font is very slow). `block` causes invisible text for 3s (poor UX). `optional` means font might never swap on slow networks.

---

## Key Takeaways

1. **Critical Rendering Path is sequential** - Blocking one stage delays all downstream stages
2. **Render-blocking resources must be minimized** - Inline critical CSS, defer non-critical
3. **Web Vitals measure real user experience** - LCP < 2.5s, INP < 200ms, CLS < 0.1
4. **Animate transform/opacity only** - All other properties trigger reflow/repaint
5. **Content-visibility: auto enables 50-80% rendering improvements** - Use for long lists
6. **Font-display: swap prevents invisible text** - Show fallback immediately
7. **Contain reduces layout scope from $O(n)$ to $O(m)$** - Use on independent components

