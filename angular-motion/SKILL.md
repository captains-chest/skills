---
name: angular-motion
description: Chooses compositor-only motion for Angular UI on the first pass, without profiling. Use when creating or editing Angular components, templates, or styles that animate, including transitions, enter/leave effects, expand/collapse, dialogs, drawers, toasts, popovers, route transitions, or hover and scroll effects.
---

# Angular Motion

Angular is property-agnostic: `animate.enter` / `animate.leave` only add classes, wait for `animationend` / `transitionend`, then drop the classes or the node. The browser runs the animation, so the CSS you write decides whether it stays on the **compositor** (GPU, off the main thread) or re-runs layout and paint every frame.

For API mechanics (`AnimationCallbackEvent`, `MAX_ANIMATION_TIMEOUT`, testing), use the `angular-developer` skill. This skill decides *which* mechanism and *which* properties.

## 1. Pick the mechanism

Take the most specific row that fits each motion in the change. List stagger adds timing to an enter/leave row; a content-height change always follows **Height**.

| Motion | Mechanism |
| --- | --- |
| Hover, focus, or state class on an element that stays in the DOM | CSS `transition` on a `[class.x]` / `:hover` change |
| Element appears via `@if` / `@for` | `animate.enter` with keyframes, or a `transition` plus `@starting-style` for the *from* state |
| Element must finish an exit before removal | `animate.leave`: Angular keeps the node until the animation ends |
| Native `<dialog>` / `[popover]` | `@starting-style` + `transition-behavior: allow-discrete` on `display` and `overlay`. Where discrete transitions are unsupported (Firefox) the exit snaps; accept that. When the exit must animate everywhere, remove the dialog via `@if` + `animate.leave` while it is still open, intercepting `cancel` and close paths (Escape, `close()`, `method="dialog"` submit), since native closing hides it before any exit runs |
| Route change, shared-element morph | `withViewTransitions()` on the router; `::view-transition-*` rules in **global** styles (view encapsulation breaks the pseudo-element selectors) |
| Scroll-linked effect | `animation-timeline: scroll()` / `view()` inside `@supports (animation-timeline: scroll())` |
| List stagger | `animation-delay: calc(var(--i) * 40ms)` with `[style.--i]="$index"` in the `@for`; entering keyframes get `animation-fill-mode: backwards` so items hold the *from* state through the delay |
| Expand/collapse to content height | See **Height** below |
| FLIP reorder, drag physics, spring, or interruptible gesture | WAAPI (`element.animate`) started from the reorder or gesture event; enter/leave hooks don't fire for these |
| Imperative choreography on DOM entry or exit | WAAPI from `(animate.enter)` / `(animate.leave)`; a leave handler calls `animationComplete()` when it finishes |

Motion runs on native CSS, `animate.enter` / `animate.leave`, and WAAPI. `@angular/animations` (deprecated since v20.2) and `provideAnimations*()` stay out of new code; mixing them with `animate.*` in one component also breaks cleanup. A JS animation library comes in only for a capability WAAPI can't express, with that gap stated in the summary.

`animate.leave` runs only within the same component template: a child component's own leave animations are skipped when its parent is removed. Put `animate.leave` on the child's host element in the parent template.

## 2. Write compositor-only CSS

- Move with `transform` or the individual `translate`, `scale`, `rotate` properties; fade with `opacity`. This is the **compositor-only** set every engine honours.
- `filter` is not a portable compositor-only guarantee, and blur can be expensive; keep it optional garnish, not the main motion.
- Layout and paint properties stay static: `width`, `height`, `top`/`left`/`right`/`bottom`, `margin`, `padding`, `border-width`, `font-size`, `box-shadow`. Fake a size change with `scale`, a position change with `translate`, a shadow change by fading a pseudo-element's `opacity`.
- Leave `will-change` off by default: running animations already get their own layer. Add it to one specific element only when a hover- or script-triggered start visibly stutters.

### Reduced motion

Every motion gets a `prefers-reduced-motion: reduce` branch, chosen by mechanism:

- One-shot lifecycle CSS (`animate.*`, transitions): `1ms` duration, `0s` delay, one iteration. Keep the duration above `0` so `animationend` / `transitionend` still fire and Angular removes the classes or node.
- Decorative looping or scroll-driven motion: `animation: none`, showing the static state.
- Router view transitions: in `withViewTransitions({ onViewTransitionCreated })`, call `transition.skipTransition()` when `matchMedia('(prefers-reduced-motion: reduce)')` matches.
- WAAPI handlers: apply the end state and call `animationComplete()` immediately.

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
.panel-enter { animation: panel-in 200ms ease-out backwards; }
.panel-leave { animation: panel-out 150ms ease-in forwards; }

@keyframes panel-in { from { opacity: 0; translate: 0 -8px; } }
@keyframes panel-out { to { opacity: 0; translate: 0 -8px; } }

@media (prefers-reduced-motion: reduce) {
  .panel-enter, .panel-leave {
    animation-duration: 1ms;
    animation-delay: 0s;
    animation-iteration-count: 1;
  }
}
```

The element's base styles are the end state; the keyframes describe only the *from* (enter) or *to* (leave).

## 3. Self-check

The change is done when every animation, transition, and keyframe it adds or edits passes all of these:

- [ ] Each motion uses its most specific matching row in step 1; stagger only adds timing, and content-height changes follow **Height**.
- [ ] Interpolated properties are `transform`, `translate`, `scale`, `rotate`, and `opacity` (plus garnish `filter`), or a **Height** case that followed its order. `display` and `overlay` appear only as discrete bookkeeping.
- [ ] Reduced motion is handled per mechanism used, as in **Reduced motion**.
- [ ] `will-change` appears only on a named element with a stated reason.
- [ ] No new `@angular/animations` imports or `provideAnimations*()` providers; any animation library has its capability gap stated in the summary.
- [ ] `::view-transition-*` rules live in global styles.
