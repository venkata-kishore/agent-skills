---
name: code-comments
description: Use when writing, editing or reviewing comments inside code, in any language - deciding whether a comment is needed, where it goes and how to word it. Covers line and block comments only. Doc comments (Javadoc, TSDoc/JSDoc, Go doc comments) are out of scope and follow their own tool's conventions.
---

# Code comments

**Scope.** Line and block comments inside code. Doc comments are API documentation: write
them the way their tool expects (summary, parameter and return tags, exceptions) and do not
apply this skill to them.

## The test

A comment earns its place only if it tells the reader something the code cannot. If deleting
it loses nothing, do not write it. When code needs a comment to explain *what* it does, first
try a better name or an extracted function.

## Where a comment is required

- **Why, when it is not obvious:** a workaround, an outside constraint (protocol, spec, a bug in
  a dependency, with its link or ID), a choice that looks wrong but is deliberate.
- **Invariants and ordering** the code relies on but does not enforce: lock order, "must run
  before X", who owns a resource.
- **Concurrency reasoning:** why a race cannot happen, why a check is repeated.
- **Surprises:** a deliberately empty catch, an intentional fallthrough, where a magic number
  comes from and its unit.
- **Known limits:** a TODO with the reason and a ticket, not a bare "TODO: fix".

## Never

- Restating the line: `// the logger`, `// constructor`, `// increment counter`, `// return result`.
- Labels over obvious blocks: `// Populate pool`, `// Step 1: borrow`, a numbered label that
  repeats the test name below it. Tests included. A section banner that splits a long file into
  parts is not a label: keep those few and short.
- `// Do nothing` in an empty body. If the emptiness needs a reason, write the reason.
- Commented-out code, change history (`// v2:`, "it used to", "the old X was deleted"), or
  narration of the edit ("added null check"). Version control holds those. If old behaviour is
  why a test exists, state the rule the test protects, in the present tense.
- References the reader cannot follow, such as IDs from a document that is not in the repo and
  not linked.

## How to write one

- Put it directly above the code it explains, or at the end of the line for a short note on
  that line. One idea per comment.
- Plain present tense. Lead with the reason; name the consequence ("so X cannot happen",
  "otherwise Y").
- Match the file's existing comment style and density.
- When you change the code, update or delete its comment in the same edit.
- Existing restating comments in code you are changing: point them out, do not silently
  delete someone else's comments.

## Examples

```java
// Bad: restates the line
// the logger
private static final Logger log = LoggerFactory.getLogger(Pool.class);

// Good: says why
// Leaving without taking an object: pass the signal on so a wake-up meant for us is not lost.
available.signal();
```

```go
// Bad
// close the response body
defer resp.Body.Close()

// Good
// Drain before closing so the transport can reuse the connection.
io.Copy(io.Discard, resp.Body)
resp.Body.Close()
```

```typescript
// Bad
// set timeout to 29 seconds
const timeoutMs = 29_000;

// Good
// The gateway drops idle connections at 30 s; stay just under it.
const timeoutMs = 29_000;
```
