# felt — sharing one stylesheet across React and web components

A small, dependency-light demo that answers a specific question:

> If I'm building a component library in **React** and in **web components** — and the web
> components can render in both **light DOM** and **shadow DOM** — how do I ship a **single
> stylesheet** that styles all three?

It renders the same card (a flex column whose body uses `flex-grow: 1` to push the footer to
the bottom) three ways, side by side, all styled by one shared "felt" stylesheet:

1. **React** — a native React component (not a web-component wrapper)
2. **Web component · shadow DOM**
3. **Web component · light DOM**

The React components are native React; the web component is a real custom element
(`<felt-card-element>`) with a `shadow` attribute that toggles shadow vs. light DOM.

## Running it

It's a static page with no build step. Either:

```bash
# just open the file
open index.html

# …or serve it (nicer, avoids any file:// quirks)
npx serve .
# then visit the printed URL
```

React is loaded from a CDN (unpkg), so the React column needs internet access. The two
web-component columns work offline.

## What you can toggle

The page has two independent toggles. All four combinations render the identical card.

### 1. Stylesheet strategy — how the CSS is *authored*

The hard part of "one stylesheet for all three" is the shadow boundary: styles don't cross
it, and a class selector can never match a custom-element **host** from inside its own shadow
root. There are two ways to make one stylesheet work anyway.

- **Strategy A — inner element + `display: contents`**
  The web component renders a real `.felt-card` element (inside its shadow root, or as a
  light-DOM child). The custom-element host is set to `display: contents`, so it drops out of
  the box tree and the inner `.felt-card` becomes the flex/grid item — structurally identical
  to React's `<div class="felt-card">`. The stylesheet targets **plain classes only**; no
  `:host()` required. See [`felt-strategy-a.css`](./felt-strategy-a.css).

- **Strategy B — `:host(.felt-card), .felt-card`**
  The custom element host **is** the card box: it carries the class and holds the content
  directly, with no wrapper and no `display: contents`. Light DOM matches the plain
  `.felt-card` selector; inside a shadow root the same rules reach the host via
  `:host(.felt-card)`. See [`felt-strategy-b.css`](./felt-strategy-b.css).

The page shows a live pros/cons comparison for each. In short:

| | Strategy A (`display: contents`) | Strategy B (`:host`) |
|---|---|---|
| Selectors | plain classes only | `.felt-card` **and** `:host(.felt-card)` per host rule |
| Specificity | uniform (10) everywhere | split: `.felt-card` = 10 vs `:host(.felt-card)` = 20 |
| Extra DOM | inner wrapper element | none — host is the card |
| Consumer styling the outer element | ❌ host has no box (`margin`/`width`/`grid-column` on `<felt-card-element>` are ignored) | ✅ host is a real box |
| Authoring cost | low, no tooling | needs the twin selector (or a build step) |

> **Tip for Strategy B:** wrap host rules in `:where(.felt-card, :host(.felt-card))` to flatten
> the 10-vs-20 specificity difference, and never use a *bare* `:host` in a shared/bundled sheet
> (it would match every component's host and collide) — always scope it as `:host(.felt-card)`.

### 2. Delivery — how the CSS is *delivered*

- **`<link>` in head + inline `<style>` in shadow** (the typical setup)
  A `<link rel="stylesheet">` in `<head>` styles the light DOM (React + the light-DOM web
  component); each shadow-DOM card gets its own inline `<style>` injected into its shadow root.
  No JS plumbing, but the CSS is parsed once per shadow root and a brief FOUC is possible.

- **Adopted stylesheets** (shared object)
  A single constructed `CSSStyleSheet` is adopted into `document.adoptedStyleSheets` **and**
  every shadow root — one shared, parsed-once object across all three contexts. Cheapest at
  runtime; requires a little JS to wire up. Falls back to `<link>`/`<style>` automatically if
  the browser lacks support.

## The shadow-boundary demonstration

Each card body carries a note explaining why its text may be red. The root page's `<style>`
contains an **app-level** rule that is deliberately *not* part of the shared felt stylesheet:

```css
/* in the page's <head> — NOT the shared stylesheet */
.felt-card__body { color: red; }
```

This global app style reaches the **React** and **light-DOM** card bodies (they live in the
document's style scope) but **cannot cross the shadow boundary**, so the **shadow-DOM** card's
body stays the default color. It's the same boundary regardless of which delivery mechanism is
selected — the shadow DOM does the blocking, not the delivery.

Two things *do* cross the boundary and are worth leaning on in a real system: **CSS custom
properties** (define tokens at `:root`, values inherit into shadow DOM) and inherited
properties like `color`/`font`.

## Files

| File | Purpose |
|---|---|
| [`index.html`](./index.html) | The whole demo: page chrome, the React component, the `<felt-card-element>` custom element, and the toggle wiring. |
| [`felt-strategy-a.css`](./felt-strategy-a.css) | Shared stylesheet, Strategy A (backs the `<link>` delivery). |
| [`felt-strategy-b.css`](./felt-strategy-b.css) | Shared stylesheet, Strategy B (backs the `<link>` delivery). |

The class names use a `felt` design-system prefix (`.felt-card`, `.felt-card__header`,
`.felt-card__body`, `.felt-card__footer`) — flat, low-specificity, BEM-style classes, which is
exactly what makes a stylesheet portable across the shadow boundary. Modifier classes are left
out to keep the demo focused.

## Takeaways

- A single document-scoped stylesheet covers React and light-DOM web components for free; the
  shadow boundary is the only part that needs a deliberate mechanism.
- Author self-contained, low-specificity class selectors so the same rules work in every
  context; keep design tokens as CSS custom properties so they inherit through shadow DOM.
- **Strategy A** keeps the CSS dead simple (plain classes) at the cost of a transparent host
  you can't style directly. **Strategy B** keeps the host stylable at the cost of duplicated,
  differently-specific selectors. Pick based on whether consumers need to style the outer
  element.
