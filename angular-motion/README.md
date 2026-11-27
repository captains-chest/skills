# angular-motion

Picks the right animation mechanism for Angular UI and keeps motion on compositor-only properties (`transform`, `translate`, `scale`, `rotate`, `opacity`), so animations are GPU-friendly on the first pass without profiling.

## What it covers

- Choosing between plain CSS transitions, `@starting-style`, `animate.enter` / `animate.leave`, native `<dialog>` / popover transitions, View Transitions for routes, scroll-driven animations, and WAAPI for real choreography
- Expand/collapse to content height, with honest layout trade-offs
- `prefers-reduced-motion`, `will-change` discipline, and keeping `@angular/animations` out of new code
- A self-check to run before finishing

`SKILL.md` is the authoritative source. API mechanics stay in the `angular-developer` skill.

## Install

```bash
npx skills@latest add captains-chest/skills \
  --skill angular-motion -y
```
