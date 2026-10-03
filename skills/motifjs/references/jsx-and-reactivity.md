# JSX, reactivity and bindings

## Reactive primitives (`@motifx/core` exports)

```ts
import { reactive, useModel, createSignal, createComputed, effect, untracked, deepClone, clearModel } from '@motifx/core';
```

| API | Notes |
|---|---|
| `reactive(obj)` / `useModel(obj)` | Deep proxy. Read = tracked, write = triggers. Nested objects and arrays (`push`, `splice`, sort, index assignment) are tracked. A function-valued field is returned as the function itself; a binding that ends up with a function value calls it (unbound, so no `this`) — `{state.fn}`, `bindings.add(prop, state, 'fn')`, attribute getters — but your own code must call it: `() => state.fn() + '!'`. |
| `createSignal(v)` / `new Signal(v, equals?)` / `Signal.create(v, equals?)` | `.value` get/set, `.peek()`, `.update(fn)`, `.mutate(fn)`, `.notify()`, `.asReadonly()`, `.dispose()`. |
| `createComputed(fn)` | Eager, cached derived value: computed at creation and recomputed on dependency change even when unread; `.value`, `.peek()`, `.dispose()`. |
| `createLazyComputed(fn)` | Lazy derived value: nothing is computed until `.value` is read; a dependency change only marks it dirty (synchronously) and wakes its readers; `.value`, `.peek()`, `.isDirty`, `.dispose()`. |
| `effect(fn, onValueChanged?)` | Runs immediately and again on every change of what it read. `onValueChanged(value)` receives `fn`'s return value after every run, outside `fn`'s tracking. Returns `stop()`. Inside components prefer `this.bindings.watch(fn)`. |
| `untracked(fn)` | Reads inside are not tracked by the enclosing effect; returns `fn`'s value. Nested `effect()` calls still track normally. |
| `deepClone(v)`, `clearModel(m)` | Utilities. |
| `asyncTracking` | `capture()`/`suspend()`/`resume()`/`end()`; emitted by the compiler around `await` in a `Virtualization` `dataRequest` so reads after `await` stay tracked. Not needed in app code. |

Effects flush on a microtask in rounds. An effect that runs more than 50 times in one flush (it writes
what it reads, e.g. `state.n++`) is skipped for the rest of that flush and raises the `MJX203` dev warning;
wrap the write in `untracked(...)`.

Module-level stores are idiomatic:

```ts
// store/session.ts
export const session = reactive({ user: null as User | null, theme: 'light' });
```

## Expression forms inside JSX

| You write | Behaviour |
|---|---|
| `{"text"}` | Static. |
| `{localVar}` | Guarded getter `bindings.add('textContent', () => …)`: a string/number local is written once (nothing reactive is read); a local holding a function is called inside the binding and stays live; `null`/`undefined` gives empty text; a component (or array of them) is placed with `controls.add`. |
| `{fn()}` | Plain call: `controls.add(fn())`, evaluated once; the one-shot form. |
| `{state.field}` | One-way binding to that field (`bindings.add('textContent', state, 'field')`). |
| `{() => expr}` | Reactive getter; re-evaluated when any reactive read inside changes. Most flexible. |
| `` {`Count: ${state.n}`} `` | Template literal is wrapped in a getter → reactive. |
| `{cond && <X/>}` | Conditional (bindings.when) — **constructs and disposes on every toggle**; rebuilt only when the condition value changes. Prefer `x-wait` (see below). |
| `{a ?? b}` / `{a \|\| b}` | Left when present (`??`: not null/undefined; `\|\|`: truthy), else right; reactive; right may be JSX (bindings.method). |
| `{this.method(x)}` | Member call: a returned component is inserted (Frame, re-evaluated on reactive change), a string becomes reactive text (bindings.method). |
| `{cond ? <A/> : <B/>}` | Two-branch (bindings.ternary), nested ternaries allowed; a branch is rebuilt only when its condition value changes. |
| `{arr.map(item => <li key={item.id}/>)}` | List (bindings.list); rows matched by the item object. `.filter().map()`, `[x].map()` and `(c ? xs : ys).map()` work. |
| `{this.childs}` / `{[compA, compB]}` | Insert component instances. |
| `{someComponentInstance}` | Insert. |

A getter that returns JSX (`{() => cond ? <A/> : <B/>}`) is also supported and behaves like a
reactive single-content slot.

## Attributes

```tsx
<input type="text" placeholder="Search" />         // static
<a href={state.url} />                              // reactive field
<img src={() => state.avatar} />                    // reactive getter
<button disabled={() => state.busy}>Go</button>
<input value={() => state.q} oninput={(e) => state.q = e.target.value} />
<label for="email" />                               // `for` is fine (not htmlFor)
```

Rule: **a function value = reactive binding; anything else = set once.** On a plain DOM tag the
compiler wraps an expression that reads a variable (`a + b`, `!state.open`, `state.url`) in a getter,
so it stays live; a plain call (`data-n={fmt(x)}`) is evaluated once; a bare identifier (`attr={v}`) is
passed as is (live only when it holds a function); literal values (numbers, negative
numbers, `true`/`false`, `null`, arrays, `new X()`) are passed to `attr.add` as they are.

Attribute values: `false`, `null` and `undefined` remove the attribute, `true` and a valueless attribute
(`<button disabled />`) write `""`, strings and numbers are written as they are (`tabindex={0}` →
`"0"`). On `aria-*` and `data-*` a boolean is written as text: `aria-expanded={false}` →
`aria-expanded="false"`, `data-on={true}` → `"true"`; `null`/`undefined` still remove it. `attr.add({ title: null, role: 'tab' })` removes `title` and still writes `role`.

Attribute names are only the own keys of the object given to `attr.add`; a value is never traversed.
An object or array value is written as one attribute value, converted to text: `title={obj}` →
`title="[object Object]"`, `data-ids={[1, 2]}` → `data-ids="1,2"`, and a getter returning an object or
array writes the same text. On a method-like prop the value is the argument instead (see
[Method-like props](#method-like-props)).

**Security:** attributes written on the tag and `attr.add` values are written without filtering. When a
URL attribute (`href`, `src`, `action`, `formaction`) is bound to user data, a `javascript:` value runs
code on click, so validate the scheme first, e.g.
`href={() => /^(https?:|mailto:|\/|#|\.)/i.test(u.trim()) ? u : '#'}`. `attr.add({ innerHTML })` writes
HTML as is, like `x-html`. Only a spread drops `innerHTML` and `javascript:` URLs on its own.

**One deliberate exception: a ternary in attribute/prop position is always wrapped lazily** —
`mode={x ? 'a' : 'b'}` compiles to `mode: () => x ? 'a' : 'b'`, on DOM tags and component tags alike,
so the expression stays reactive instead of freezing at `view()` time. The contract for the receiving
component is to declare that prop as `Bind<T>` and read it with `read()` / `toGetter()` (all three exported from `@motifx/core`); a component
that declares a plain type and passes the prop straight through gets a function at runtime (e.g. an
`Icon` that looks up `name` directly receives `() => …` from `<Icon name={ok ? 'check' : 'error'}/>`).
A valueless attribute on a component tag (`<Comp flag />`) passes `flag: true`. `npx motif-lint`
flags a ternary on a prop whose declared type is not `Bind<T>` (`MJX005`).

**When in doubt, do not guess the lowering — print it.** `npx motif-explain file.tsx` lists every
JSX expression with the real generated call (`sender.bindings.add("textContent", state, "ad")`,
`sender.bindings.when(…)`, `sender.attr.add({ title: () => … })`…), its reactivity class and its
dependency surface. JSX here is shorthand for the binding API; the compiler is a lowering pass, and
`explain` is its `-S` flag.

### Children: conditionals and prebuilt components

**Prefer the `x-wait` directive over `{cond && <X/>}`.** `<X x-wait={() => !cond} />` waits while the
function returns `true`: if it is `true` initially the element is never built (subtree included), and
toggling it later only hides/shows — the instance and its state survive. `{cond && <X/>}` instead
builds a fresh instance on every show and disposes it on every hide. Use `&&` only when that dispose
is what you want. Full comparison in `references/conditionals-lists-frames.md`.

`&&` and ternary branches are rebuilt only when the condition **value** changes (`Object.is`; for
`&&` two falsy values count as the same); while it stays the same the branch keeps its component
instance and state. A component prop written as a plain expression inside a branch
(`<Badge count={state.n}/>`) is therefore a snapshot taken when the branch was built, exactly as
outside a conditional. Pass a getter (`count={() => state.n}`) and read it on the receiving side with
the `Bind<T>` / `read()` / `toGetter()` contract to keep it live.

`{cond && <X/>}` uses **JS truthiness** — `false`, `null`, `undefined`, `0`, `''` render nothing — both
inside a real element and at a fragment (`<>…</>`) root.

**Component-valued props need no special syntax.** `{props.headerTemplate}`, `{this.props.tpl}`,
`{parts}`, `{one}` holding a component (or an array of them) are placed into `controls` instead of
being stringified; non-component values keep the reactive text binding. So a slot prop and a data prop
are written the same way:

```tsx
<div>{props.headerTemplate}{props.title}</div>   // component slot + reactive text
<Card headerTemplate={<h1>Title</h1>} title="Sub" />
```

The decision is made **once, when the parent's `initializeComponent` places it**: the placed component's own bindings stay live,
but reassigning the prop later does not swap the DOM. **If the template can change (starts `null`,
gets swapped at runtime), write a getter** — `{() => props.headerTemplate}` — which creates a `Frame`
slot and renavigates on every change. If a component reaches a text binding anyway, nothing is written
and an `MJX204` dev warning names the getter fix.

### Spread `{...props}`

On a **plain DOM tag** the spread object is applied as if the attributes were written inline:
`class`/`className` → `class.add` (merged with an inline `class`), `style` → `style()`, `value`/`checked`/
`selected` → property binding, `on*` (`onclick`, `onClick`, `oninput:once`) → DOM listener, everything else
→ attribute. Getters stay live; functions that take parameters (`renderItem: (x) => …`) are skipped.
Inline attributes are applied after the spread, so `<div {...p} id="fixed"/>` keeps `id="fixed"`.
A spread never sets `innerHTML` (dev warning `MJX124`) and never writes a `javascript:` URL to `href`/`src`/
`action`/`formaction`/`xlink:href` (dev warning `MJX125`; a getter's current value is removed). Use `x-html` for
trusted HTML and write an intended `javascript:` link inline on the tag; inline attributes are not filtered.

On a **component tag** (`<Card {...props}/>`, `<Card class="x" id="y"/>`) only the **common attributes**
fall through to the component's root element (Vue-style attribute fallthrough): `class`/`className` (merged
with the component's own classes), `style`, `id`, `tabindex`, `role`, `aria-*`, `data-*`. Getters
stay live. Data/callback props (`items`, `label`, `onSave`, `disabled`) are never written to the root, and
neither is `title` — deliberately, since it is a very common data prop name (`<Card title="…"/>`).
Everything, including the fallen-through keys, remains readable in `this.props`. The component's own
`onConfigured`/`initializeComponent` attributes win for the same attribute (fallthrough is applied in the constructor).
Function components behave the same: the tag's common attributes land on the returned root (no double
application when the root is `<div {...props}/>`). A fragment-rooted component silently ignores them.

### `class` / `className`

String, array, object map or getter: `class={() => ({ active: state.on, big: size > 2 })}`.

### `style`

String, object (`{ width: '33%' }`) or getter returning either. A getter is **live**: keys it stops
returning are cleared; a returned string replaces the whole inline style. `this.style(fn)` keeps one
watcher — calling it again stops the previous one.

### Method-like props

On a **plain DOM tag**, props named after element methods call the method instead of writing an
attribute. A getter or reactive field re-applies on every change (`showModal={() => state.open}`);
the call runs one microtask later. On a component tag these names are ordinary props.

- `focus`: only strict `false` → `blur()`; every other value (`true`, `undefined`, `null`, `0`, `''`) → `focus()` / `focus(value)`
  (`focus={{ preventScroll: true }}`). `focus={() => f.editing}` with `editing` starting `undefined`
  therefore calls `focus()` on the first run; start the field at `false`.
- `show`, `showModal` / `showPopover`: truthy → call, `false` → `close()` / `hidePopover()`.
- `togglePopover`: boolean → `togglePopover(bool)`, other truthy → `togglePopover()`.
- `play` (falsy → `pause()`), `select` (falsy → clears the selection), `requestFullscreen` /
  `requestPointerLock` (falsy → exit): truthy → call.
- `close`, `hidePopover`, `requestSubmit`, `checkValidity`, `reportValidity`, `showPicker`, `load`:
  `true` → `method()`, other truthy → `method(value)` (`close={() => 'cancel'}`); falsy → nothing.
- `blur`, `click`, `pause`, `submit`, `reset`, `exitFullscreen`, `exitPointerLock`,
  `requestPictureInPicture`, `exitPictureInPicture`: truthy → call; falsy → nothing.
- `scrollIntoView`: `true`/`null`/`undefined` → `scrollIntoView()`, anything else (including `false`) → `scrollIntoView(value)`.
- `scrollTo`, `scrollBy`: only an object or array → `method(value)`.
- `setSelectionRange`, `setRangeText`, `setPointerCapture`, `releasePointerCapture`, `fastSeek`:
  any value except `null`/`undefined`/`false` → `method(value)`.

An array value is spread into arguments (`setSelectionRange={[0, 5]}` or `setSelectionRange={() => [0, 5]}`
→ `setSelectionRange(0, 5)`); an object value is passed as one argument (`scrollTo={{ top: 0 }}`).
Literal values call the method too: `showModal={true}`, `focus={true}`, and a valueless `<dialog show />`
counts as `true`. If the element has no such
method, the value is written as a plain attribute.

### Special `x-` props

| Prop | Effect |
|---|---|
| `ref={(s) => ...}` / `ref={this.x}` / `x-ref` | Capture the tag's component instance (callback, or a member/name that is called when it is a function and assigned otherwise; every tag, exactly once). `ref` and `x-ref` on one tag both run, in source order. Applied only to the component it was given to (a function component's returned root); never present in `props` / `this.props`, so `{...props}` does not forward it. Use a separately named prop (`inputRef`) for an inner element. |
| `x-html={() => html}` | Reactive `innerHTML` (unsafe with untrusted input). |
| `x-text`, `x-value`, `x-watch`, `x-model` | Compile to `sender.bindings.text(…)` / `.value(…)` / `.watch(…)` / `.model(…)`. `x-model={() => s.q}` (or `x-model={s.q}`) becomes `bindings.model(() => s.q, v => { s.q = v })`: two-way, and the owner object is read again on every write. A value that is not a member access binds one-way (events-and-forms.md). |
| `x-inject={v}` | Passed as the prop `inject` (no attribute on a plain DOM tag). |
| `x-foo` (unknown) | Passed as the prop `onfoo`; compiler warning `MJX007`. `x-bind`, `x-focus`, `x-list`, `x-loop`, `x-effect`, `x-reload`, `x-interrupt`, `x-to` are compile errors (`MJX006`). |
| `x-wait={() => bool}` | While `true`, the component is held back from the DOM. |
| `x-display={() => bool}` | Show/hide (uses `hideStrategy`). |
| `x-style={...}` | Same as `style` (compiles to `sender.style(...)`; getter live). |
| `x-config`, `x-configured`, `x-building`, `x-built`, `x-mounted`, `x-activated`, `x-deactivated`, `x-disposing`, `x-disposed`, `x-initializing`, `x-initialized`, `x-visibilitychanged` (or `x:config`…, `onconfig`, `onbuilt`, `onmounted`, …) | Lifecycle hooks as props. Several spellings of one hook on the same tag (`onbuilt` + `x-built`, case-insensitive) are merged: all of them run once, in source order. On a function component tag every spelling reaches the returned root; `initializing`/`initialized` fire right after the root is constructed, in the class tag order (`ref` → `initializing` → `initialized` → `config`). |
| `key` | Identifies a list item; duplicates raise the `MJX202` dev warning. Rows are matched by the item object, so `key` does not change DOM reuse. |
| `transition="fade"` or `transition={{...}}` | Enter/leave CSS classes (styling-and-transitions.md). No attribute is written, also on plain DOM tags. |
| `options={{ hideStrategy, disableDisposal }}` | Copied into the tag's `motif.options` (plain DOM tags and component tags); no attribute is written. |
| `initializeComponent={(s) => ...}` | Called once with the tag's component as `sender`, during build right before `view()`; good place for `s.bindings.*` calls. Same meaning and timing on plain DOM tags, class component tags and function component tags (the returned root). The compiler emits its own `initializeComponent` for the tag's attributes, events, directives and children (inside `runover` on component tags); both run: the component's method, then the tag's, then the compiled one. With JSX you rarely write it; it is mainly for building components without JSX. |

## The bindings API (`this.bindings` / `sender.bindings`)

The compiler generates these; you can call them directly in `initializeComponent`/`onconfig` or in tests.

```ts
add(prop, source)                        // bind DOM property `prop` to reactive object/getter
add(prop, source, member)                // bind to source[member]
add(prop, source, member, formatString)
add(prop, source, member, formatString, formatInfo)   // formatInfo: { locale?, currency? }
add(binding)                             // an object with `propertyName` is registered as is
text(() => string)                       // textContent
value(() => any)                         // value
html(() => string)                       // innerHTML
when(() => cond, () => component)        // conditional; rebuilt when the cond value changes (Object.is, falsy values equal)
ternary(() => cond, frame => frame.navigate(A), frame => frame.navigate(B))   // branch rebuilt when the cond value changes
list(() => items, (item, i) => component) // rows matched by item object identity, not by key
loop(...)                                // alias of list
switchCase(() => key, { a: frame => ..., b: frame => ... }, frame => defaultCase)
method(() => any)                        // string/number/boolean/bigint → text; component → Frame slot;
                                         // null/undefined in slot mode empties it (old component disposed)
watch(() => void)                        // side-effect, auto-stopped on dispose
model(getter, setter)                    // two-way through functions; what `x-model` compiles to
model(source, member, format?, formatInfo?) // two-way (checked/value/src chosen by element type)
model(binding)                           // any single object argument is taken as an IBaseBinding;
                                         // a single function argument binds one-way (no write-back)
wait(() => bool); display(() => bool)
remove(binding); clear()                 // deactivate registered bindings
activateAll(); deactivateAll()           // deactivateAll also empties the registry
reActivateAll()                          // see "Non-reactive data"
items                                    // copy of the registered bindings
```

`add`, `html`, `list`/`loop`, `model`, `wait`, `display` register a binding and return it; `text` and
`value` register one and return nothing. `clear` (and `remove` with the returned binding) stops them. `when`, `ternary` and `switchCase` return `null`, `method` and `watch`
return nothing; none of them is registered, so `remove`/`clear` cannot stop them. All five stop when the
component is disposed; the inner condition of a nested ternary also stops when the outer branch switches away.

A `Binding` (returned by `add`/`model`) has `converter` (applied to the value read, after
`formatString`), `converterBack` (applied to the DOM value before `model` writes it back), `setter`
(the write function of the getter/setter form) and `formatInfo`; set them on the returned binding before
the component is built. `updateMode` is not read by any binding.

`formatString` codes: on numbers `C` → currency, `P` → percent, `N`/`N2` → fixed decimals (default 2);
on `Date` values `d` → `toLocaleDateString`, `t` → `toLocaleTimeString`; any other value is written
with `toString()`. `null`/`undefined` are not formatted. Locale and currency come from the binding's
`formatInfo` (`{ locale?: string | string[], currency?: string }`):
`bindings.add('textContent', s, 'price', 'C', { locale: 'en-US', currency: 'USD' })` → `$1,234.50`.
Without `locale` the runtime's default locale applies (the `Intl` default); without `currency`, `C`
writes the amount with two decimals and no currency symbol.

Example without JSX (as used in the test-suite):

```ts
const data = reactive({ message: 'Hello' });
const c = new Component('div', { initializeComponent: (s) => { s.bindings.text(() => data.message); } });
c.build();
document.body.appendChild(c.element);
data.message = 'World'; // textContent updates after a microtask
```

## Non-reactive data

Prefer reactive data (`reactive(obj)`). For plain objects mutated in place, `this.setState()` refreshes
the subtree: it calls `setState()` on each child, then `bindings.reActivateAll()`, which re-runs every
registered binding of the component (text, attribute, class and style getters, `model`, `wait`/`display`)
and makes each list dispose and rebuild all its rows (`ListBinding.reActivate`). `reState()` does the
same with this component first, then its children. `when`, `ternary`, `switchCase`, `method` and `watch`
are not registered bindings and are not re-run. A binding that throws while refreshing is reported as
`MJX208` and the remaining ones still run.

## Gotchas

- Reading a reactive field **outside** a getter in `view()` captures a snapshot: `const n = this.state.count; <p>{n}</p>` never updates.
- Destructuring a reactive object (`const { count } = state`) loses reactivity.
- Array mutators (`push`, `pop`, `shift`, `unshift`, `splice`, `reverse`) do not track their own internal
  reads, so `state.items.push(x)` inside an effect does not subscribe the effect to the array (an explicit
  read such as `state.items.length` still does).
- `createComputed` recomputes on the next microtask, so a synchronous read right after a write may see
  the previous value; `createLazyComputed` is marked dirty synchronously and recomputes on read.
- Replacing a whole array (`state.items = newArr`) is fine: rows are matched by item object identity,
  so objects that are in both arrays keep their rows; fresh objects (e.g. a refetched list) get new rows
  even when their `key`/`id` is the same.
- An effect or binding getter that throws is reported as a `MotifError` with code `MJX208` (original
  error in `cause`) to the console and `errorHandler.addListener(fn)` listeners; the other effects in
  the flush keep running. A throwing (or rejected async) event handler is reported as `MJX123`.
- Updates are flushed on a microtask; in tests `await Promise.resolve()` (the local `nextTick()` helper in testing.md; `@motifx/core` exports none) before asserting.
