# Coding standards

The rules a human or an agent holds in their head while writing and reviewing
code here. Rules a machine already enforces sit in
[Checked by machine, not by you](#checked-by-machine-not-by-you) at the bottom —
read that table once, then spend no review attention on it.

## Effect and configuration

Wherever possible, use Effect primitives like `FileSystem` over promises. This
is so that we can make use of DI and type-safe errors from Effect. However,
Effect should not leak out into the user-facing API.

Read every environment variable a run needs at its start, not at the moment of
use. A `Config.string(...)` inside a branch that runs rarely turns a missing
`.env` line into a failure that appears only when that branch first runs — a
video export that concats and normalizes for thirteen seconds, then fails
because nobody set `OVERLAY_RENDER_CACHE_DIRECTORY`, and does it again on every
retry. Resolve the config at the edge (the layer, or the command's entry point)
so a missing variable stops the process before any work starts, and let the
error name the variable.

## Function signatures

Optional parameters passed to functions should be scrutinised extremely
carefully. They are a huge source of bugs (by omission). Prioritise correctness
over backwards compatibility.

## Entities and their actions

### Every entity is right-clickable

Every entity the app renders — a Course, Section, Lesson, Video, Clip, Chapter,
Beat, Pitch, Deliverable — answers a right-click with a context menu. An entity
with actions and no right-click handler is an unfinished entity.

The right-click menu and the entity's **Actions menu** (the `Actions` dropdown,
or the `…` button on the entity itself) offer **the same set of actions**. They
are two doors into one list: an action added to one appears in the other, so
share the menu items between them rather than writing each list twice.

### Order the actions, and group the related ones

Both menus present that shared list in a deliberate order, most-reached action
first and the destructive ones (archive, delete) last. Related actions sit
together in a group — everything that moves the entity, everything that exports
it, everything that ends its life — with `DropdownMenuGroup` and a
`DropdownMenuSeparator` between groups, and a `DropdownMenuLabel` where the
group's name helps the reader. Adding an action means choosing the group it
belongs to, not appending to the end of the list.

### Context menu items carry an icon

Context menu items should always include a leading icon (from `lucide-react`),
matching the style of the surrounding items. When adding a new menu item, pick
an icon that conveys the action.

### Filters stay in sync with the entity

Filters must stay in sync with the shape of the data they filter. When a new
field is added to an entity that affects what something "is" (status, category,
state), every filter, count, and badge that surfaces that concept must be
updated to take the new field into account. Filters are part of the entity's
definition, not a one-time UI feature — drift between them and the data shape
produces silently-wrong results.

## React Router data flow

### Redirect from the action, not from an effect

When a fetcher action's sole job after success is to navigate, return
`redirect(...)` from the action instead of returning data and navigating from a
client-side `useEffect`. React Router handles fetcher redirects automatically.
The `useEffect` pattern is fragile: if any dep (e.g. an inline `onOpenChange`
prop) changes between renders, the effect re-fires and re-issues
`navigate(...)`, cancelling and restarting the in-flight navigation in a loop.

### Derive optimistic UI from `fetcher.formData`

For optimistic UI on fetcher mutations, derive the optimistic value from
`fetcher.formData` instead of mirroring it into `useState` + syncing back with
`useEffect`. When the fetcher is in-flight, `fetcher.formData.get("value")`
holds the pending value; when it settles, `formData` becomes `undefined` and the
component falls back to the revalidated loader data. Example:
`const optimistic = (fetcher.formData?.get("value") ?? loaderValue) as MyType;`.
This eliminates state-sync bugs and removes the need for `useEffect` entirely.

## Interface design

### Deep modules

Prefer deep modules: small interface, deep implementation. A few methods with
simple params hiding complex logic behind them.

Avoid shallow modules: large interface with many methods that just pass through
to thin implementation. When designing, ask: can I reduce the number of methods?
Can I simplify the parameters? Can I hide more complexity inside?

### Design for testability

1. **Accept dependencies, don't create them** — pass external dependencies in rather than constructing them internally.
2. **Return results, don't produce side effects** — a function that returns a value is easier to test than one that mutates state.
3. **Small surface area** — fewer methods = fewer tests needed, fewer params = simpler test setup.

## Testing

Tests verify behavior through public interfaces, not implementation details.
Code can change entirely; tests shouldn't break unless behavior changed.

Mock at **system boundaries** only — external APIs, time and randomness, and the
file system or a database when a real instance isn't practical. Everything
inside the boundary goes in real: never mock your own classes, modules or
internal collaborators. When something is hard to test without mocking an
internal, redesign the interface.

Writing, changing or reviewing a test — for the worked good and bad examples,
the red-flag list, the rule for Remotion renderer packages, and the
vertical-slice TDD loop, read
[`TESTING_STANDARDS.md`](./TESTING_STANDARDS.md).

## Checked by machine, not by you

These rules were here once. A check enforces each one now, so spend no review attention on them — `pnpm run check` runs the lot.

| Rule                                          | Check                              |
| --------------------------------------------- | ---------------------------------- |
| `localStorage` goes through `useLocalStorage` | `oxlint` (`no-restricted-globals`) |
| `import.meta.dirname` over CJS `__dirname`    | `scripts/check-no-dirname.sh`      |
| No test or utility files in `app/routes`      | `scripts/check-routes-folder.sh`   |
| Every env key documented in `.env.example`    | `scripts/check-env-example.sh`     |
| No file over 5,500 tokens                     | `scripts/check-file-tokens.sh`     |
| Deep-module import boundaries                 | `pnpm run lint:boundaries`         |

Oxlint's own `correctness` set runs advisory: its warnings are a standing backlog, cleared by hand in the PRs that touch each file.
