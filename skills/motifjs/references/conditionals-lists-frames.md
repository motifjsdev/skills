# Conditionals, lists, frames and slots

## `x-wait` — the primary way to show/hide (prefer this)

```tsx
<p x-wait={() => !state.loggedIn}>Welcome back</p>     // waits WHILE the fn returns true
<p x-display={() => state.loggedIn}>Welcome back</p>   // exact inverse, same machinery
```
Compiled to `bindings.wait` / `bindings.display`. Works on plain DOM tags, class components and
function components alike.

Measured behaviour (`packages/tests/src/components/x-wait-semantics.test.ts`):

- **Lazy build.** If the condition is `true` on first evaluation the element is never `build()`ed —
  neither it nor its subtree enters the DOM, a placeholder is left instead. `onbuilding`/`onbuilt`/
  `onmounted` do not fire until it first becomes visible. A panel that starts closed costs nothing.
- **Instance preserved.** After the first build, toggling only hides/shows. The component is **not**
  disposed, so scroll position, form input and third-party plugin state survive. `ondisposing`/
  `ondisposed` do not fire.
- **Both are synchronous on first render.** The first content navigated into an empty `Frame`
  lands during `build()`, so `&&`, ternary and a JSX-returning `{this.part()}` are in the DOM when
  `build()` returns — same as the directive. Later branch *changes* are async (the outgoing content
  is awaited-disposed first; latest navigation wins).

Pair it with `options={{ hideStrategy: 'detach' }}` when the hidden subtree should leave the DOM entirely.

## `&&` — only when you actually want construct/dispose

```tsx
{state.loggedIn && <p>Welcome back</p>}
```
Compiled to `bindings.when`. A **new instance is created** every time the condition becomes truthy
and **disposed** every time it becomes falsy. The branch is rebuilt only when the condition **value**
changes (`Object.is`; two falsy values count as the same): `{state.n > 0 && <X/>}` keeps the same `X`
while `n` goes 1 → 2 → 3, and `{state.user && <X/>}` rebuilds when `state.user` is replaced by
another object.

Choose it only when that churn is the point: a third-party plugin that must really be destroyed, or
content too heavy to keep in memory while hidden. Otherwise `x-wait` is cheaper and more predictable.

| Situation | Use |
|---|---|
| Show/hide, state must survive | `x-wait` / `x-display` |
| Starts hidden, heavy content | `x-wait` (never built) |
| Must genuinely dispose on hide | `{cond && <X/>}` |
| One of two branches | ternary |
| Only text changes | getter |

## Ternary

```tsx
{state.loading ? <Spinner /> : <Content data={state.data} />}
{state.level > 2 ? <Gold/> : state.level > 1 ? <Silver/> : <Bronze/>}   // nested OK
```
Compiled to `bindings.ternary`; each branch receives a `Frame` and `navigate()`s into it. The old
branch is disposed **without a leave transition** (`skipLeaveTransition: true`; running animations on it
are cancelled); the same holds for `&&`, `{this.method()}`, `switch` swaps and every `frame.navigate`.
Animate a swap with `x-display`/`x-wait` on kept instances instead. A branch is rebuilt only when its condition
value changes (`Object.is`); in a nested ternary the inner branch is kept while the outer value stays
the same.

**Props inside a branch are snapshots.** The branch is built once and kept while the condition value
stays the same, so a component prop written as a plain expression (`<Content data={state.data} />`,
`<Badge count={state.n} />`) holds the value from when the branch was built, exactly as outside a
conditional. Pass a getter and read it with the `Bind<T>` contract to keep it live:

```tsx
<Badge count={() => state.n} />
// Badge: interface Props { count: Bind<number> }  …  <span>{() => read(this.props.count)}</span>
```

For plain text use a getter instead: `<span>{() => state.done ? '✓' : '✗'}</span>`.

### Compiler restriction (important)

Inside a JSX child position, **do not** write a block-bodied arrow that declares locals and then
returns a conditional:

```tsx
{() => { const label = compute(state.x); return state.ok ? <A t={label}/> : <B/>; }}  // ✗ ReferenceError: label
```
The compiler lifts the conditional out of the closure. Use an expression-bodied arrow, move the
logic into a named method, or compute inside the branch:

```tsx
{() => state.ok ? <A t={compute(state.x)}/> : <B/>}                                   // ✓
{this.renderStatus()}   // where renderStatus() returns bindings-free JSX or a component  // ✓
```

## Lists with `.map`

```tsx
const state = reactive({ todos: [{ id: 1, title: 'Buy milk', done: false }] });

<ul>
  {state.todos.map(todo => (
    <li key={todo.id} class={() => todo.done ? 'done' : ''}>
      <input type="checkbox" checked={() => todo.done} onchange={(e) => todo.done = e.target.checked} />
      {todo.title}
      <button onclick={() => state.todos.splice(state.todos.indexOf(todo), 1)}>×</button>
    </li>
  ))}
</ul>
```

- Compiled to `bindings.list(() => state.todos, (todo) => ...)`. Add/remove/reorder patch the
  DOM minimally (identity diff with longest-increasing-subsequence moves).
- **Row identity is the item object** (the raw object behind the proxy), not `key`: an object present
  in the old and new array keeps its row (moved if needed); a removed object's row is disposed; fresh
  objects (a refetched list) get new rows even with the same `id`. The same object may appear more
  than once (`push(items[0])`, `splice(i, 0, items[j])`): each occurrence gets its own row, all
  showing that object. Array methods never copy items; write `{ ...item }` for an independent copy.
  Primitive items (`string[]`) keep
  their row when the same value stays at the same index, or anywhere when the render function takes
  no `index` parameter.
- **`key`**: still give a stable unique value; it identifies items. It does not take part in matching
  and does not change DOM reuse; a duplicate key raises the `MJX202` dev warning, and a `.map` without
  `key` raises the compiler warning `MJX003`.
- Items are reactive: mutating `todo.done` updates only that row.
- `{state.todos.filter(t => !t.done).map(...)}` is reactive when the filter reads reactive fields.
- `items.map(function (item) { … }, ctx)`: the second `map` argument is honoured as `this`.
- The render function must return a component (instance, class or factory); anything else throws
  `MJX201`.

## `switch`-like rendering

```ts
this.bindings.switchCase(
  () => state.status,
  {
    loading: (frame) => frame.navigate(<Spinner />),
    ready:   (frame) => frame.navigate(<Content />),
    error:   (frame) => frame.navigate(<ErrorView />),
  },
  (frame) => frame.navigate(<Empty />)   // default, optional
);
```

In JSX write a `switch` inside a block-bodied getter; the compiler turns the `case` branches into
`switchCase`:

```tsx
<div>
  {() => {
    switch (state.status) {
      case 'loading': return <Spinner />;
      case 'ready':   return <Content />;
      default:        return <Empty />;
    }
  }}
</div>
```
`case` labels must be string, number or boolean **literals** (other cases are skipped). Each `case`
contributes only the argument of its own `return`; a branch without `return` places empty content, so
fall-through (`case 'a': case 'b': return <X/>;`) leaves `'a'` empty — repeat the `return` per case.
Statements before the `switch` in the block body, and statements before a `return` inside a case, are
dropped.

## `Frame` — a slot you navigate programmatically

```tsx
import { Frame } from '@motifx/core';

class Shell extends Component<HTMLDivElement> {
  frame = new Frame();
  view() { return <div><nav/>{this.frame}</div>; }
  async showSettings() { await this.frame.navigate(<Settings />); }   // disposes previous content
  async showLazy() { await this.frame.navigateLazy(() => import('./Reports'), { Loaderview: new Spinner() }); }
}
```

`frame.navigate(component | component[], keepOldControl = false)`, `frame.flush()`, `frame.motif.clear()`,
`frame.current`, `frame.isBusy`. Navigation is serialised and "latest wins".

- Navigating to the component that is already `frame.current` (or to `null`) does nothing.
- A `Promise` (e.g. `import('./Reports')`) is also accepted at runtime and wrapped in `Lazy`; the
  parameter is typed as a component, so cast it. `navigateLazy` is the typed form.
- `keepOldControl = true` keeps the previous content instead of disposing it.
- `frame.flush()` disposes the current content (no leave transition) without navigating;
  `frame.current` still references the disposed content.

## `Lazy` — dynamic import as a component

```tsx
import { Lazy } from '@motifx/core';

<Lazy caller={() => import('./HeavyChart')}
      options={{ Loaderview: new Spinner(), Fallbackview: new ErrorBox(), minDelayMs: 200, timeoutMs: 10000 }} />
```
`LazyOptions`: `Loaderview`, `Placeholderview`, `Fallbackview`, `minDelayMs`, `timeoutMs`, `signal`,
`onError`, `mapResult` (default: `module.default` when it is a function, otherwise the whole resolved
value), `retry` (count or `{ count, delayMs?, whenOnline? }`),
`onRetry(attempt, error)`.

## `Transport` / `TransportTo` — portal-style slots

```tsx
// somewhere in the layout
<Transport name="toolbar" mode="replace" />      // or mode="merge"

// in any page
<TransportTo name="toolbar"><button>Save</button></TransportTo>
```
Children of `TransportTo` are moved into the `Transport` slot with the same name and removed (and
disposed) when the `TransportTo` is disposed; their bindings keep working against the sender's state.
Order does not matter: a `TransportTo` built before its slot delivers when the slot registers.
`TransportTo` itself leaves an empty `<div>` in place. Children added later to the `TransportTo`'s
`controls` go straight to the slot, stay alive there and are disposed with the sender like the initial
children. When the slot is disposed its transported content is disposed too.
Several senders on one slot: `mode="replace"` (default) keeps only the most recently built sender's
content (the previous content is disposed and does not come back), `mode="merge"` appends every
sender's content in build order and removes each sender's own content when it closes. Needs a
running `Application` (the pair talks over application events).

`transport.clearSlot()` removes and disposes everything currently in a `Transport` slot.

`ContentBody name="x"` / `ContentBlock target="x"` is a simpler slot pair without modes; prefer
`Transport`.

## Showing/hiding without recreating

```tsx
<div x-display={() => state.open} options={{ hideStrategy: 'detach' }}>Heavy panel</div>
```
`x-display` toggles visibility (with transitions) instead of disposing; use `&&` when you want
the subtree destroyed. `hideStrategy` is given through the `options` prop (or set on `motif.options` in code): `'placeholder'`
(comment node stays in place), `'detach'` (node removed), `'auto'` (default: `'detach'` for list
rows, `'placeholder'` otherwise). On a plain DOM tag `options` writes no attribute.
