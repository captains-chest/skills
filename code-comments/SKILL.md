---
name: code-comments
description: Applies the default ban on code comments, TODOs, and suppression directives. Use when writing, editing, or reviewing code in any language.
---

# Code Comments

Code carries its own meaning. A comment earns its place only when it carries a **why** the code cannot express; every other comment is a liability that goes stale as soon as the code moves on without it.

## Express it in code first

Before writing a comment, move its meaning into the code:

- A name for the variable, function, or type that says what the comment would.
- A small extracted function whose name is the step's intent.
- A type, enum, or named constant that encodes the constraint.
- A test that pins the behaviour the comment would describe.

Write the comment only when none of these can carry the meaning.

## Comments that earn their place

- **Workaround**: code that exists only because something else is broken. Link the upstream issue or bug and state the removal condition.
- **Non-obvious why**: a constraint the code cannot show, such as a spec clause, regulation, measured performance trade-off, or external system quirk. Cite the source when one exists.
- **Directive**: `eslint-disable`, `@ts-expect-error`, `# noqa`, `#[allow(...)]` and their kin, scoped to the narrowest line, with the reason on the same line.
- **License header** the project requires.
- **Public API doc comment** where the ecosystem consumes it (TSDoc on a published library, docstrings, Rustdoc, Javadoc). Document the contract, not the implementation.
- **Tracked TODO**: `TODO(#123): ...`, only when the work must happen at this exact spot and a reader here needs to know it. Otherwise the work lives in the issue tracker alone; without an issue link, open the issue and drop the TODO.

Keep each one short, and phrase it so it stays true until its removal condition is met.

## Everything else goes

Restated code, commented-out code, author or changelog notes, section dividers, and untracked TODOs. Version control holds history and authorship; the issue tracker holds future work.

## Example

Before: the comments restate the code, and the real constraint goes unsaid.

```ts
// loop over the users
for (const user of users) {
  // check if active
  if (user.lastSeen > Date.now() - 2592000000) {
    await notify(user);
  }
}
```

After: names carry the meaning, and the one comment carries a workaround with its removal condition.

```ts
const activeWindowMs = 30 * 24 * 60 * 60 * 1000;
const isActive = (user: User) => user.lastSeen > Date.now() - activeWindowMs;

// One send per user: batch sends drop the reply-to header
// (https://github.com/acme/mailer/issues/412). Switch to sendBatch once fixed.
for (const user of users) {
  if (isActive(user)) {
    await notify(user);
  }
}
```

## Existing comments

When editing, apply these rules to the comments in the lines you touch: delete what is obsolete or redundant, and fix what has gone stale. Leave comments in untouched code alone; a comment sweep is its own change, made only on request.

Done when every comment in the change fits one of the kinds above and no comment in a touched line is stale.
