# Rendering Bugs: Spinner Output Appearing Under Wrong Package Heading

## Symptom

When running `ultra` in a monorepo with multiple packages, the final output could show
the output of one package (e.g. `lib-themes`) under the heading of a different package
(e.g. `ui-traffic`). The affected package heading could appear twice in the terminal.

The bug was reproducible when using a git alias like `git st .` (where `st` expands to
`status --short --branch`) but not with `git status .`. It occurred both in parallel and
`--serial` mode.

## Root Cause: Two Cooperating Bugs

### Bug 1 — Full render overwrites partial-render positions (`spinner.ts:_stop`)

The spinner system re-renders the terminal every ~120 ms during execution. To fit the
terminal height, each spinner's output is truncated: only the **first line** (heading)
and the **last N−1 lines** of output are shown per spinner (`limitLines`, `spinner.ts`).

When all processes finish, `_stop()` calls `render(full=true)`, which shows **all**
output lines for every spinner. This full render calls `terminal.update()` with the
complete line list.

`terminal.update()` performs an **in-place diff** against `this.lines` — the line list
from the previous partial render:

1. Move cursor up to the start of the previous partial render.
2. Overwrite the first `this.lines.length` lines.
3. Append any remaining new lines below.

**The problem:** if a spinner's full output is longer than what `limitLines` showed in
the last partial render, the full render has more lines than `this.lines`. The extra
lines push all subsequent spinners downward. The overwrite phase writes those extra
lines into the positions that previously held the headings of later packages, destroying
them. The later packages are then appended at the bottom, appearing a second time.

**Why `git st` triggers it but `git status` does not:**  
`git st` (with short-format flags like `--short --branch`) produces one line per
changed file plus a header — more total lines than the verbose `git status` output in a
typical repo with several modified files. Whether the bug manifests depends on whether
the spinner's full output exceeds the `limitLines` count for the given terminal height.
The specific packages affected (which is "victim") depend on:

- workspace package order (alphabetical / workspace definition)
- number of output lines produced per package for the given command
- terminal height

**Fix (`spinner.ts:_stop`):** call `terminal.reset()` before `render(true)`. `reset()`
moves the cursor back to the top of the partial render and clears the screen from there,
so the subsequent full render starts with an empty `this.lines = []`. `terminal.update()`
then simply appends all lines without any positional mismatch.

### Bug 2 — `diff()` writes old-line suffix instead of new-line suffix (`terminal.ts`)

`terminal.update()` calls `this.diff(newLine, oldLine)` to find a common prefix/suffix
and only rewrite the changed middle segment (an optimisation to reduce terminal writes).

```typescript
// call site
const diff = this.diff(line, this.lines[l])  // from=new, to=old
```

`diff()` correctly identifies the common prefix length (`left`) but then returned:

```typescript
str: to.slice(left),   // BUG: to = old line
```

It should return the **new** line's suffix:

```typescript
str: from.slice(left), // from = new line  ✓
```

When `diff()` fired (same-length lines with a common prefix), the terminal received the
old line's tail instead of the new one. Combined with Bug 1, this produced hybrid lines:
the new prefix from the full render followed by the old suffix from the partial render —
which is how a later package's heading text (e.g. `ui-traffic at …`) could appear
appended to content that belonged to an earlier package.

A `// FIX:` comment in the original source already pointed at this line but contained
an incorrect suggested replacement (`to.slice(left, -right + 1)` still used `to`).

**Fix (`terminal.ts:diff`):** change `str: to.slice(left)` to `str: from.slice(left)`.

## Files Changed

| File | Change |
|---|---|
| `src/terminal.ts` | Fix `diff()` return value; add `reset()` method |
| `src/spinner.ts` | Call `this.terminal.reset()` before `render(true)` in `_stop()` |
