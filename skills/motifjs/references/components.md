# Components

Every component derives from `ComponentBase<TElement, TProps>`; you use the concrete `Component`.
A component = one root DOM node (`element`) + a live child collection (`controls`) + bindings.

## Class component (pages and views bound to a model)

```tsx
import { Component, ComponentBase, EventArgs, reactive } from '@motifx/core';

interface UserCardProps { userId: number; highlighted?: boolean }

export default class UserCard extends Component<HTMLDivElement, UserCardProps> {
  state = reactive({ user: null as null | { name: string }, loading: true });

  // Optional. Omit it entirely and the compiler-injected `div` is used.
  constructor(props: UserCardProps) { super('div', props); }

  async onConfig(sender: ComponentBase, e: EventArgs) {     // props ready, not in DOM
    this.state.user = await this.doWork(fetchUser(this.props.userId));
    this.state.loading = false;
  }

  onMounted() { this.element.focus(); }                      // element attached to document (onBuilt may run detached)

  view() {
    return (
      <div class={() => this.props.highlighted ? 'card active' : 'card'}>
        {this.state.loading ? <span>Loading…</span> : <b>{() => this.state.user?.name}</b>}
      </div>
    );
  }
}
```

### Constructor forms (`Component`)

| Call | Root element |
|---|---|
| `super('div')` / `super('div', props)` | `<div>` |
| `super(existingNode, props)` | wraps an existing DOM node |
| `super(props)` / `super()` on `class X extends Component<HTMLDivElement>` | element from the declared generic (`static elementTag`, injected by the compiler; can also be written by hand: `static elementTag = 'section'`) |
| `super(props)` / `super()` with no declared element type | **fragment** (comment marker; children are inserted between markers; no attributes/events on the root) |
| `super(['a','b'])` | fragment holding one child component per array entry |

Use SVG by declaring `Component<SVGSVGElement>` (namespace injected) or passing `{ options: { isSvg: true } }`.

To narrow `props` in a subclass of an existing component, write `declare props: DialogProps;` (type only, no
code). Never `props: DialogProps;` or `props!: DialogProps;`: that creates a new field that overwrites the base
class's `props` with `undefined`. `declare` and `abstract` fields compile to nothing.

### `view()` vs imperative `controls`

- `view()` is called once during `build()`; its return value becomes the children.
- `this.controls.add(...)` can be called any time; when the component is already in the DOM the child is inserted immediately.
  A child that has another parent is detached from it first.

```ts
this.controls.add(child, child2);  // append; arrays are flattened
this.controls.add('Total: ', 42);  // strings, numbers, booleans → text nodes (a single `add(5)` appends the text "5")
this.controls.add(() => <b>x</b>); // a factory is called with no arguments; a component class is constructed without props
this.controls.add(0, child);       // number first + at least one child → insert at that index
this.controls.insert(2, a, b);     // same as add(2, a, b)
this.controls.move(child, before); // reorder inside this collection: put `child` before `before` (end when omitted/null); DOM moved, nothing rebuilt
this.controls.moveToIndex(child, i); // move before the item currently at index i (end when i >= length)
this.controls.remove(child);       // remove + dispose
this.controls.clear();             // dispose all children
await this.controls.clearAsync();  // dispose all children and wait until every dispose finished
this.controls.detach(child);       // plays the leave (root element, or the element children of a fragment root), then removes from DOM; NOT disposed; returns a Promise; parent gets `controlremoved` after removal
this.controls.silentDetach(child); // remove from DOM, do NOT dispose, no `controlremoved` notification
this.controls.silentUnlink(child); // unlink from the collection only; DOM untouched
this.controls.items; this.controls.length; this.controls.forEach(fn); this.controls.map(fn);
```

### Props and children

- Props are whatever JSX attributes you pass: `<UserCard userId={42} highlighted />`.
- Lifecycle props (`onconfig`, `onbuilt`, `onmounted`, … and their `x-` forms) and `initializeComponent` are removed from
  `props` and registered as hooks. A function `ref` (JSX always passes `ref` as a function) is applied to the component and
  removed from `props`; an object `ref` passed without JSX stays as `props.ref` (see Refs). On a component tag, a known DOM event name (`onclick`, `onChange`,
  `oninput:once`) compiles to a listener on the root element (`motif.on`) and never reaches `props`.
  Every other `on*` prop (`onSave`, `onValueChange`) is an ordinary callback prop in `this.props`.
- `options={{ hideStrategy, disableDisposal }}` is copied into `this.motif.options`; other keys of
  `options` (apart from `isSvg`, read when the element is created) are ignored, so do not use
  `options` as a data prop name.
- JSX children are delivered as `this.childs` (`ComponentBase[]`). A component that wants to
  render them must add them: `this.controls.add(...this.childs)` or `{this.childs}` in `view()`. When the
  constructor gets no element (fragment root, or a declared element from `Component<HTMLDivElement>` /
  `static elementTag`), the children are appended to the root automatically, unless the class's JSX places
  `{this.childs}` / `{() => this.childs}`: then they are built only in that slot, so a slot behind
  `x-wait` / `x-display` or inside a nested element waits with it (the compiler marks such a class with
  `static _placesChilds`). With `super('div', props)` nothing is appended for you. Children that are never
  placed are disposed with the component that received them.

```tsx
class Panel extends Component<HTMLDivElement, { title: string }> {
  view() {
    return <div class="panel">
      <h3>{this.props.title}</h3>
      <div class="body">{this.childs}</div>
    </div>;
  }
}
<Panel title="Settings"><p>content</p></Panel>
```

## Function component

```tsx
export function Todo(props: { text: string; done?: boolean }) {
  const s = reactive({ done: !!props.done });
  return <li class={() => s.done ? 'done' : ''} onclick={() => s.done = !s.done}>{props.text}</li>;
}
```

- Called once with `props`; the returned JSX becomes the component (wrapped in a fragment when
  multiple roots). Closure variables act as state when they are `reactive()`.
- Lifecycle hooks are attached as props on the root element: `onconfig`, `onbuilt`, `ondisposing`, …
  Hooks, `x-wait`/`x-display`, `initializeComponent`, `ref` and `options` written on the function
  component's tag (`<Panel onmounted={…} />`) are applied to the returned root.
- `transition` on a function component's tag is applied to the returned root as well (a name, an object
  or a getter); the tag wins over a `transition` written on the root inside the function.
  A class component tag applies `transition` itself.
- `FNComponent(viewFn)` wraps a view function into a factory explicitly (rarely needed; JSX does it).

## Options-API object

`{ el: 'div', data: reactive({...}), view() {...}, ctor?(props) {...}, ...anything }` — every key except
`el` and `ctor` is copied onto a component whose root is `el`. `this` inside `view()` is the component.
`ctor(props)` runs once right after creation (before `build()`), with `this` = the component; a throw
is reported as `MJX122` and the component is still created.
A function returning such an object works wherever a component goes: a JSX tag, a route `control`,
a `Lazy` `default`, and `new Component(Factory, props)`. A route calls it as `fn(app)` and its `ctor`
receives `undefined`.

## Members you will use

| Member | Purpose |
|---|---|
| `element` | Real DOM node; available before mount. |
| `props`, `childs` | Inputs. |
| `controls` | Child collection (see above). |
| `context` | The `Application` (walks up the parent chain, falls back to `Application.main`), seen through a component-scoped view: `context.on(...)` / `context.onRouterChanged(...)` subscriptions are removed when the component is disposed. Every other member is the application's own: `context.onLifecycle(...)` is **not** removed; register its returned unsubscribe with `this.motif.setDisposable(...)`. |
| `parent` | Parent component or `null`. |
| `bindings` | `BindingCollection` (see jsx-and-reactivity.md). |
| `class.add(...)`, `class.remove(...)`, `class.has(name)` | Ref-counted class management; accepts strings or getters. `class.remove('**')` clears everything. `has` reads the element's `classList`. |
| `attr.add({...})`, `attr.remove(key)`, `attr.has(key)`, `attr.get(key)` | Set attributes; `remove` also stops a reactive attribute getter; `has` reads the element; `get` returns the element's value, else the last value given to `add`, else `null`. |
| `style(v)` | String, object, or getter → inline style. |
| `setText(s)` | textContent. |
| `motif` | Namespace (`ComponentMotif`) for the framework operations below. A subclass may define its own `show`, `on`, `clear`, `options`… without breaking `x-display`, transitions, list keys or dispose; the framework only calls `this.motif.*`. Members: `show`, `hide`, `toggle`, `on`, `off`, `trigger`, `addHandler`, `clear`, `register`, `setDisposable`, `stopAnimations`, `options`. |
| `motif.on/off/trigger/addHandler` | Events (events-and-forms.md). |
| `motif.show()/hide()/toggle()`, `isVisible`, `isWait` | Visibility (lifecycle-and-dispose.md). |
| `motif.options.hideStrategy` | `'placeholder'`, `'detach'` or `'auto'` (default, behaves as `'placeholder'`). Give it in JSX through the `options` prop (`options={{ hideStrategy: 'detach' }}`) or assign `this.motif.options.hideStrategy` in code. A bare `hideStrategy="detach"` attribute is not special. |
| `motif.options.display` | Getter returns `isVisible`; assigning `true`/`false` calls `motif.show()`/`motif.hide()`. |
| `motif.options.hasEvent(name)` | `true` when a `motif.on(name, …)` listener is registered (names are stored lowercased). |
| `motif.options.enableRouterClassing = { to, path, activeClass?, exactClass?, onActive?, offActive?, onExact?, offExact? }` | Setter; toggles the classes on the root on every route change (subscription auto-disposed). `to`: `'all'`, `'active'` (segment-prefix match, exact included), `'exact'` or `'none'`. A failure is reported as `MJX105`. |
| `motif.options.transition` | Enter/leave animations (styling-and-transitions.md). |
| `setState()` / `reState()` | Re-run the registered bindings of this subtree (text, attribute, class and style getters, `model`; lists rebuild their rows). Only needed for non-reactive data; `setState` runs the children first, `reState` this component first. |
| `getService(Token)` | DI resolve via nearest provider; `null` if missing. |
| `useModel(obj)` | Same as `reactive(obj)`. |
| `motif.register(disposable)` / `motif.setDisposable(fn)` | Cleanup on dispose. |
| `using(promise, cb)` / `doWork(promise)` | Dispose-safe async. |
| `dispose(opts?)`, `disposeAsync(opts?)`, `motif.clear()` | Teardown (`motif.clear()` disposes the children only and does nothing when `motif.options.disableDisposal` is set). `IDisposeOptions`: `{ skipLeaveTransition?, deep? }` (lifecycle-and-dispose.md). |
| `onElementCreating()` | Optional class method (or `onElementCreating` prop) that returns the root element; it replaces the element given to the constructor. Runs first in the base constructor: `this.props` and subclass fields are not set yet. |
| `$(selector)` → `{ fromDom(), fromComponent() }` | Query descendants. |
| `siblings.next()/prev()/all()/nextAll()/prevAll()` | Sibling components. |
| Flags | `isBuilt`, `isInitialized` (`false` in `onInitializing`, `true` from `onInitialized` on), `isConfigured`, `isDisposed`, `isVisible`, `isWait`. |
| `isWait = bool` | Setter: on a built component `true` → `motif.hide()`, `false` → `motif.show()`; before build, `false` builds it into an already built parent. |

## Refs

```tsx
class Form extends Component<HTMLFormElement> {
  input!: Component<HTMLInputElement>;
  view() {
    return <form onsubmit:prevent={() => this.input.element.focus()}>
      <input ref={this.input} />
    </form>;
  }
}
```

`ref` receives the tag's **component** (on a plain DOM tag, the `Component` wrapping the element); use
`.element` for the node. It is called while the component is constructed (before `build()`) and is never
written to the DOM. `props`, the common attributes and `options` are already applied when it runs; on a class
component the subclass's own fields are not yet initialized (JavaScript sets them after the base constructor
returns), so use a ref to store the component (`ref={(b) => (this.okButton = b)}`), not to write its fields. Two forms:

- `ref={(c) => …}` (callback) and `ref={this.x}` (called if the target is a function, assigned otherwise) work on plain DOM tags and component
  tags alike; on a function component tag the returned root is passed.
- `x-ref` / `x:ref` are the same as `ref`. It is called exactly once per tag; when `ref` and `x-ref`
  are both written on one tag, both run once, in source order.
- A JSX `ref` is not a prop: it runs only on the component it was given to (a class component itself, a
  function component's returned root). It is absent from the `props` a function component receives and
  from `this.props`, so spreading `{...props}` / `{...this.props}` never passes it to an inner component.
  Without JSX, `new Card({ ref })` (or `{ runover: { ref } }`) applies it to that component the same way.
  Rule: a function ref is called with the component and removed from props; an object ref is
  assigned the component in place (`props.ref` / `runover.ref`), e.g. `const p = { ref: {} }; const c = new Card(p); p.ref === c`.
- To expose an inner element, use a separately named prop and pass it to the inner tag's `ref`
  (`ref={props.inputRef}` or `ref={(c) => props.inputRef?.(c)}`):

```tsx
function SearchBox(props: { inputRef?: (c: Component) => void }) {
  return <div class="search"><input ref={(c) => props.inputRef?.(c)} /></div>;
}
// <SearchBox ref={this.box} inputRef={(c) => this.search = c} />  → box: the div, search: the input
```

`onRefCreated(sender)`: when a class defines it, it runs right after each `ref={this.x}` / `ref={name}` in that
class's `view()` is applied (the field is already assigned, the method already called); `sender` is the ref'd
tag's component. It does not run for callback refs (`ref={(c) => …}`). It is a member of the class whose
`view()` holds the JSX, not of the ref'd component. It runs for JSX in the class's methods and fields (arrow
functions included), not in function components, module-level JSX or `function` expressions inside a method.

## Communicating between components

- Parent → child: props (static) or shared `reactive()` objects (live).
- Child → parent: callback props. `onSave={(v) => ...}` stays in `this.props.onSave` (call it as
  `this.props.onSave?.(v)`). Pick names that are not DOM event names: `onChange`, `onInput`, `onSelect`,
  `onToggle`, `onResize`… on a component tag become root-element DOM listeners and never reach `this.props`
  (compiler warning `MJX002`), or call the parent through `this.parent`.
- Anywhere ↔ anywhere: `this.context.on('evt', h)` / `this.context.fire('evt', payload)` (a subscription
  made through `this.context` is removed when the component is disposed), or a singleton service /
  module-level `reactive()` store.
- Through the tree: `this.parent`, `this.controls.items`, `this.siblings`, `this.$(selector).fromComponent()`
  and `ref` reach any component; its `element`, `props`, public fields and methods are used directly.

## Navigating the component tree

Components form a live object tree that can be walked in every direction:

```tsx
this.parent;                         // parent component; null at the root
this.controls.items;                 // children
this.siblings.next();                // next sibling (undefined at the end)
this.siblings.prev();                // previous sibling (undefined at the start)
this.siblings.all();                 // the parent's children, this one included
this.siblings.nextAll();             // the siblings after this one
this.siblings.prevAll();             // the siblings before this one
this.$('.tab').fromComponent();      // matching components below: ComponentBase[]
this.$('input').fromDom();           // matching DOM nodes inside the element: NodeList

<button class="tab" onclick={(sender) => {
  sender.siblings.all()?.forEach((t) => t.class.remove('active'));
  sender.class.add('active');
}}>A</button>
```

- `siblings` reads the parent's `controls.items`; without a parent every method returns `undefined`.
- `fromComponent()` walks the component tree depth-first from this component (it is included when it
  matches) and matches each component's element against the selector. A component whose root is a
  fragment starts from the nearest ancestor that owns a real element. `fromDom()` runs
  `querySelectorAll` inside that element; a disposed component returns `null` / `[]`.
- A plain tag in JSX is a component too: `sender` in an event handler is that tag's component.
