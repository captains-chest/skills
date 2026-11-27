# code-comments

Bans code comments by default. Code carries its own meaning through names, small functions, types, and tests; a comment stays only when it carries a why the code cannot express.

## What it covers

- Moving a comment's meaning into code before writing it
- The comments that earn their place: workarounds with an upstream link and removal condition, non-obvious constraints, tool directives with a reason, license headers, ecosystem doc comments, and TODOs tied to both a tracked issue and that exact spot in the code
- What goes: restated code, commented-out code, author and changelog notes, section dividers, untracked TODOs
- Handling existing comments in the lines you touch, without sweeping unrelated code

`SKILL.md` is the authoritative source for comment rules; `project-code-standards` defers to it.

## Install

```bash
npx skills@latest add captains-chest/skills \
  --skill code-comments -y
```
