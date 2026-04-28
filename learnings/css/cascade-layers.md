# CSS Cascade Layers (`@layer`) and `!important`

> Inside `@layer`, the precedence of `!important` declarations is **inverted** — earlier layers beat later ones, and **unlayered `!important` loses to any layered `!important`**. This is the opposite of how non-important rules work.

## Background — the cascade

When two CSS rules target the same property on the same element, the browser picks a winner using these tiebreakers, in order:

1. **Origin & importance** — author/user/UA + whether `!important` is set.
2. **Cascade layers** — which `@layer` the rule belongs to.
3. **Specificity** — `(inline, IDs, classes/attrs, elements)` score.
4. **Source order** — later in the document wins.

Each step is only consulted if the previous one tied.

### Specificity quick reference

| Selector | Score |
|---|---|
| `img` | `0,0,0,1` |
| `.logo` | `0,0,1,0` |
| `#section img` | `0,1,0,1` |
| `#section img[alt*="logo"]` | `0,1,1,1` |
| inline `style=""` | `1,0,0,0` |

### `!important` in plain CSS (no layers)

For one element, the winning property comes from this stack (top wins):

1. Important author rule (most specific, then later)
2. Normal inline style
3. Normal author rule
4. Browser default

Inline styles **do not** beat `!important` from a stylesheet — common surprise.

## Cascade layers

`@layer` lets you group rules into named buckets. The browser treats layer membership as a tiebreaker that sits *above* specificity:

```css
@layer base, utilities, overrides;

@layer utilities {
  .text-red { color: red !important; }
}
```

### The non-obvious rule

For **normal** declarations:
- Unlayered styles win over all layers.
- Among layers, later layers win.

For **`!important`** declarations the order **flips**:
- Earlier layers win over later layers.
- **Unlayered `!important` loses to every layered `!important`.**

```
TOP (wins) for !important
  @layer base                         ← earliest
  @layer utilities
  @layer overrides                    ← latest
  unlayered !important                ← loses to ALL layers
  inline style="" (no !important)
  normal author rules
BOTTOM (loses)
```

The flip exists so a base library can mark something `!important` inside a layer and downstream code can override it without knowing the library's layer order — but it's a footgun if you don't know about it.

## Real-world example: care_fe print logo bug

[`src/style/index.css`](https://github.com/ohcnetwork/care_fe/blob/develop/src/style/index.css) contains, inside `@layer utilities`:

```css
@media print {
  #section-to-print img[alt*="logo" i] {
    width: auto !important;
    height: auto !important;
    max-width: 100px !important;
    max-height: 100px !important;
  }
}
```

A facility-configurable logo passed inline `style="width: 20px; height: 20px"`. The on-screen preview honored it (no `@media print` rule applies on screen). The printed PDF did not — `!important` beat the inline styles.

### Attempt 1 — add `!important` override (failed)

Injected a `<style>` block at component render time:

```css
@media print {
  #section-to-print img[data-print-logo="true"] {
    width: 20px !important;
    max-width: none !important;
    height: 20px !important;
    max-height: none !important;
  }
}
```

Equal specificity, later in source, both `!important`. *Should* have won. Didn't, because the global rule was inside `@layer utilities` and mine was unlayered → mine silently lost the layer-level tiebreaker.

### Attempt 2 — wrap override in same layer (worked)

```css
@layer utilities {
  @media print {
    #section-to-print img[data-print-logo="true"] {
      width: 20px !important;
      max-width: none !important;
      height: 20px !important;
      max-height: none !important;
    }
  }
}
```

Both rules now in `@layer utilities` → layer tiebreaker is a tie → specificity tie → source-order decides → my rule (rendered into `<body>` after the head's `<link>`) wins.

### Attempt 3 — sidestep the selector (best fix)

The cleanest fix was to make the configured logo's `alt` not contain "logo":

```tsx
<img alt={facility.name} ... />   // instead of alt="Logo"
```

The global selector `[alt*="logo" i]` no longer matches → no competing `!important` → inline styles just work. Same trick the existing `headerImage` (`alt="Custom Header"`) was already using.

## Mental model: debugging "my style won't apply" in print

Walk this checklist in order:

1. Is `@media print` actually active? (DevTools → Rendering panel → "Emulate CSS media type: print")
2. What other rules target this element? (Inspect → Computed → click property → see strikethroughs)
3. Is the loser `!important`? You need `!important` (or higher specificity) to fight it.
4. Is `max-*` clamping you? `width: 20px` is bounded by `max-width` — reset with `max-width: none`.
5. Is the winning rule inside `@layer X`? Wrap your override in the same (or earlier) layer.
6. Inline style trying to beat `!important`? It can't — promote to a stylesheet.
7. **Best fix is often: don't match the offending selector at all.**

## References

- [MDN — Cascade layers](https://developer.mozilla.org/en-US/docs/Web/CSS/@layer)
- [MDN — Specificity](https://developer.mozilla.org/en-US/docs/Web/CSS/Specificity)
- Bug context: care_fe `PrintPreview.tsx` print logo sizing, April 2026
