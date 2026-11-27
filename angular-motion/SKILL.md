---
name: angular-motion
description: Chooses compositor-only motion for Angular UI on the first pass, without profiling. Use when creating or editing Angular components, templates, or styles that animate, including transitions, enter/leave effects, expand/collapse, dialogs, drawers, toasts, popovers, route transitions, or hover and scroll effects.
---

# Angular Motion

Angular is property-agnostic: `animate.enter` / `animate.leave` only add classes, wait for `animationend` / `transitionend`, then drop the classes or the node. The browser runs the animation, so the CSS you write decides whether it stays on the **compositor** (GPU, off the main thread) or re-runs layout and paint every frame.

For API mechanics (`AnimationCallbackEvent`, `MAX_ANIMATION_TIMEOUT`, testing), use the `angular-developer` skill. This skill decides *which* mechanism and *which* properties.

## 1. Pick the mechanism

Take the first row that fits each motion in the change.

| Motion | Mechanism |
| --- | --- |
| Hover, focus, or state class on an element that stays in the DOM | CSS `transition` on a `[class.x]` / `:hover` change |
| Element appears via `@if` / `@for` | `animate.enter` with keyframes, or a `transition` plus `@starting-style` for the *from* state |
| Element must finish an exit before removal | `animate.leave`: Angular keeps the node until the animation ends |
| Native `<dialog>` / `[popover]` | `@starting-style` + `transition-behavior: allow-discrete` on `display` and `overlay`. Firefox snaps the exit (no `display` transitions); that is the accepted fallback. When the exit must animate everywhere, render the dialog with `@if` and use `animate.leave` |
| Route change, shared-element morph | `withViewTransitions()` on the router; `::view-transition-*` rules in **global** styles (view encapsulation breaks the pseudo-element selectors) |
| Scroll-linked effect | `animation-timeline: scroll()` / `view()` inside `@supports (animation-timeline: scroll())` |
| List stagger | `animation-delay: calc(var(--i) * 40ms)` with `[style.--i]="$index"` in the `@for` |
| Expand/collapse to content height | See **Height** below |
| FLIP reorder, drag physics, spring, or interruptible gesture | WAAPI (`element.animate`) from `(animate.enter)` / `(animate.leave)`, calling `animationComplete()` when it finishes. A JS library only when WAAPI can't express it |

Motion runs on `animate.enter`, `animate.leave`, and native CSS. `@angular/animations` (deprecated since v20.2), `provideAnimations*()`, and JS animation libraries come in only for the choreography the last row names, with the reason stated in the summary.

`animate.leave` runs only within the same component template: a child component's own leave animations are skipped when its parent is removed. Put `animate.leave` on the child's host element in the parent template.

## 2. Write compositor-only CSS

- Move with `transform` or the individual `translate`, `scale`, `rotate` properties; fade with `opacity`. This is the **compositor-only** set every engine honours.
- `filter` composites in Chromium only, and blurs cost raster time; use it as garnish, not the main motion.
- Layout and paint properties stay static: `width`, `height`, `top`/`left`/`right`/`bottom`, `margin`, `padding`, `border-width`, `font-size`, `box-shadow`. Fake a size change with `scale`, a position change with `translate`, a shadow change by fading a pseudo-element's `opacity`.
- Leave `will-change` off by default: running animations already get their own layer. Add it to one specific element only when a hover- or script-triggered start visibly stutters.
- Every animated selector gets a `@media (prefers-reduced-motion: reduce)` rule that cuts the duration to `1ms`. Keep it above `0` so `animationend` / `transitionend` still fire and Angular removes the classes or node.

### Height

Content-height expand/collapse is the one sanctioned **layout** animation, and every variant (`grid-template-rows`, `interpolate-size`, `calc-size()`) re-runs layout each frame. In order:

1. Sidestep it: an overlay, drawer, or menu that floats above content uses `translate` / `scale` + `opacity` instead.
2. Cross-browser: wrap the content in a `display: grid` container, transition `grid-template-rows` from `0fr` to `1fr`, and give the child `overflow: hidden` and `min-height: 0`. Keep the animated subtree small.
3. Progressive enhancement for a small one-off `height: auto`: `interpolate-size: allow-keywords` (Chromium only; elsewhere it snaps).

### Example

```html
@if (open()) {
  <section class="panel" animate.enter="panel-enter" animate.leave="panel-leave">…</section>
}
```

```scss
.panel-enter { animation: panel-in 200ms ease-out; }
.panel-leave { animation: panel-out 150ms ease-in forwards; }

@keyframes panel-in { from { opacity: 0; translate: 0 -8px; } }
@keyframes panel-out { to { opacity: 0; translate: 0 -8px; } }

@media (prefers-reduced-motion: reduce) {
  .panel-enter, .panel-leave { animation-duration: 1ms; }
}
```

The element's base styles are the end state; the keyframes describe only the *from* (enter) or *to* (leave).

## 3. Self-check

The change is done when every animation, transition, and keyframe it adds or edits passes all of these:

- [ ] Each motion uses the mechanism of its first matching row in step 1.
- [ ] Animated properties are limited to `transform`, `translate`, `scale`, `rotate`, and `opacity` (plus garnish `filter`), or a **Height** case that followed its order.
- [ ] Every animated selector has a reduced-motion rule with a non-zero duration.
- [ ] `will-change` appears only on a named element with a stated reason.
- [ ] Any new `@angular/animations` import, `provideAnimations*()` call, or animation library has its choreography reason in the summary.
- [ ] `::view-transition-*` rules live in global styles.
