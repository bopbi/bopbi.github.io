# UI — Pico.css Conflict Patterns

Pico applies opinionated styles to HTML elements that clash with custom layouts. Know these before writing new templates.

## `<article>` as a card
Pico styles `<article>` with card appearance. If using `<article>` semantically without card look:
```css
article.my-class { background: none; box-shadow: none; padding: 0; margin-bottom: 0; border-radius: 0; }
```

## `article > header` and `article > footer` negative margins
Pico adds negative horizontal margins + card-section background to direct `<header>` and `<footer>` children of `<article>`. **Fix:** use `<div class="article-head">` and `<div class="article-footer">` instead.

## Form element bottom margins
Pico adds `margin-bottom: var(--pico-spacing)` (≈1rem) to `input`, `select`, and `textarea`. **Fix:**
```css
.my-form input:not([type="radio"]):not([type="checkbox"]):not([type="hidden"]):not([tabindex="-1"]),
.my-form textarea,
.my-form select {
    margin: 0 !important;
    height: auto !important;
}
```

## Label bottom margin
Pico adds ~6px to all `label` elements. Inside a flex column form-row with its own `gap`, this doubles the spacing. **Fix:** `label { margin-bottom: 0; }` scoped to the form container.

## Radio/checkbox labels get `width: fit-content`
Pico targets `label:has([type=checkbox],[type=radio])` at specificity `(0,1,1)`. A plain class selector `(0,1,0)` loses. **Fix:** use a parent+class selector `(0,2,0)`:
```css
.topic-row .topic-opt { width: auto; display: flex; }
```

## `summary[role=button]::after` chevron forced white
PicoCSS forces `::after` white on `summary[role="button"]`. On light backgrounds, add `filter: none !important; margin-left: auto` to the `::after` pseudo-element.

## `details summary::after` — hiding Pico's chevron and using your own indicator
PicoCSS adds a chevron to **all** `<summary>` elements via `details summary::after { float: right; background-image: var(--pico-icon-chevron); ... }`. When `<summary>` uses `display: flex` or `display: grid`, `float: right` pulls the chevron out of the flow.

**Preferred approach (used in `.reg-head`):** hide the Pico `::after` entirely and render your own indicator via `::before`:
```css
.region-stack .reg-head::after { display: none; }   /* kill Pico's chevron */
.reg-head::before {
    content: "▸";
    justify-self: start;
    font-family: var(--mono);
    font-size: 12px;
    color: var(--ink-3);
    display: inline-block;
    transition: transform .15s;
}
.reg[open] .reg-head::before { transform: rotate(90deg); }
```

Also cancel PicoCSS's `details[open] > summary { margin-bottom: var(--pico-spacing) }`:
```css
.reg[open] > .reg-head { margin-bottom: 0; }
```

And suppress accordion colour changes on open/focus:
```css
.reg-head { color: var(--ink); }
.reg[open] > .reg-head { color: var(--ink); }
.reg-head:focus, .reg-head:focus-visible { outline: none; color: var(--ink); }
```

## `section` default margin-bottom
Pico adds `section { margin-bottom: var(--pico-block-spacing-vertical) }` (~1rem). **Global reset** in `public.css`:
```css
section { margin-bottom: 0; }
```

## Pill search bar — input and button inside a flex container

Any pill-shaped search bar (`border-radius: 99px` container with a flex row of `input` + `button`) must fight three Pico overrides:

| Pico behaviour | Fix |
|---|---|
| `input { width: 100%; height: ~50px; }` | `flex: 1 1 0; width: 0; height: auto !important;` |
| `input { border: ...; box-shadow: ...; }` | `border: 0 !important; box-shadow: none !important;` |
| `button { width: 100%; }` | `flex: 0 0 auto; width: auto;` |

**Canonical CSS pattern:**
```css
.apex-search {
    display: flex; align-items: center; gap: 6px;
    background: #fff;
    border: 1px solid var(--rule);
    border-radius: 99px;
    padding: 5px 5px 5px 20px;
    transition: border-color .12s, box-shadow .12s;
}
.apex-search:focus-within {
    border-color: var(--ink);
    box-shadow: 0 2px 16px rgba(0,0,0,0.06);
}
.apex-search input[type="search"] {
    flex: 1 1 0;
    width: 0; min-width: 0;
    border: 0 !important; outline: 0;
    background: transparent;
    box-shadow: none !important;
    padding: 8px 0 !important; margin: 0 !important;
    height: auto !important;
    font-family: var(--sans); font-size: 14.5px; color: var(--ink);
}
.apex-search input[type="search"]::placeholder { color: var(--ink-3); }
.apex-search button {
    flex: 0 0 auto; width: auto;
    padding: 9px 18px;
    background: var(--ink); color: #fff;
    border-radius: 99px;
    font-size: 13.5px; font-weight: 600;
    border: 0; cursor: pointer; font-family: var(--sans); margin: 0;
}
.apex-search button:hover { background: var(--merah); }
```

Reference: `static/public.css` `.apex-search` block.

## `ul > li` list markers inside flex containers
Setting `list-style: none` only on `<ul>` is not enough when the `<ul>` uses `display: flex`. **Fix:**
```css
.my-flex-list li { list-style: none; }
```
