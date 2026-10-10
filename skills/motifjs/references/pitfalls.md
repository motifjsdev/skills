# Pitfalls and known quirks

## React/Vue habits that break MotifJS code

| Wrong | Right | Why |
|---|---|---|
| `const [n, setN] = useState(0)` in a class | `state = reactive({ n: 0 })` | there is no `useState`; hooks are not the model |
| `useEffect(() => …, [])` | `onMounted()` / `this.bindings.watch()` | no render cycle; `onBuilt` may run before the element is attached |
| `onClick`, `className`, `htmlFor` | `onclick`, `class`, `for` | DOM names |
| `<div>{count}</div>` where `count` is a local | `<div>{state.count}</div>` or `{() => …}` | locals are frozen at build |
| `const { items } = this.state` then `{items.map()}` | `{this.state.items.map()}` | destructuring drops the proxy |
| `ref={(el: HTMLDivElement) => …}` | `ref={(c) => this.box = c}` then `c.element` | refs are components (on a plain DOM tag, the `Component` wrapping the element), never raw DOM nodes |
| `<a onclick:once:prevent={…}>` | one modifier per JSX attribute, or `this.motif.on('click:once:prevent', fn)` | JSX attribute names allow a single `:`; a chain is a parse error |
| `this.motif.setDisposable(this.context.on(…))` | just `this.context.on(…)` | subscriptions through `this.context` are already tied to the component |
| `key={index}` | `key={item.id}` | rows are matched by the item object, not by key; an index does not identify the item, so the duplicate-key check (`MJX202`) tells you nothing |
| `string[]` lists | `{ id, text }[]` | primitives lack identity |
| Returning arrays of JSX from helpers each render | Build once; mutate reactive data | there is no re-render |
| Cleanup in `componentWillUnmount` only | `this.motif.setDisposable(fn)` in the same place you create the resource | resources registered anywhere are disposed |
| `await nextTick(); expect(dom)` after `controls.add` | not needed; insertion is synchronous | but reactive *updates* flush on a microtask |
| `<Outlet/>`, `<Link to>` | `<RouterView/>`, `<a rel="router">` / `<RouterLink to el="a">` | |
| Forwarding `class`/`id`/`aria-*` from `this.props` to the root by hand | nothing — `<Card class="x" id="y"/>` falls through to the root automatically | only common attributes fall through (class/style/id/tabindex/role/aria-*/data-*); `title`, `disabled` and data props stay in `this.props`; `<div {...props}/>` on a DOM tag applies every key |

## Compiler quirks

1. **Block-arrow + local + conditional in child position**: `{() => { const x = …; return c ? <A/> : <B/>; }}` throws
   `ReferenceError: x is not defined` at runtime. Use expression-bodied arrows or a method.
2. **`class X extends Component` without a generic and without `super('tag')`** becomes a fragment;
   attributes/events on the component tag itself are lost. Declare `Component<HTMLDivElement>`.
3. Files with `//useReact` or `//useVue` are skipped; make sure no stray marker exists.
4. Only `.jsx .tsx .aio .mjsx .mtsx` are transformed; JSX inside `.ts` is not.
5. `@Injectable` is a standard decorator; the Vite plugin lowers it in the JSX files it compiles and in
   `.ts`/`.mts`/`.cts`/`.js`/`.mjs`/`.cjs` files, not under `node_modules` or in `.d.ts` files; a `.ts`/`.js`
   file whose nearest tsconfig sets `experimentalDecorators: true` is left to esbuild's legacy transform. Only the
   registration is runtime: the decorator records the class when the module is evaluated and
   `builder.build()` registers it (see di.md).
6. **`childs` is a reserved name in child position.** `{this.childs}`, `{props.childs}`, `{childs}`
   compile to `controls.add(...)` (content slot), never to a text binding — the same is true for
   `{[compA, compB]}`. `{() => this.childs}` goes through a `Frame` and is the form to use when the
   content must swap later. Only an exact `childs` match is special: `{this.childsCount}` and
   `{this['childs']}` stay text bindings.
7. **`{a ?? b}` / `{a || b}`** compile to `bindings.method(() => a ?? b)`: left when present, otherwise
   right (the right side may be JSX); reactive. `{a && <X/>}` is `bindings.when`.
8. **`{this.method(x)}` (member call)** goes through `bindings.method`: a returned component is
   inserted (in a `Frame`, re-evaluated when reactive reads change), a string/number becomes reactive
   text; a boolean is text too (`false` prints `false`). The function runs once per change, and the result may switch between text and a component
   in place; `null` in place of a component empties the spot. `{helper(x)}` (plain identifier call) is inserted once, statically.
9. **`.map` on a non-identifier source** — `{[X].map(...)}`, `{(a ? xs : ys).map(...)}` — works
   (the source is wrapped in a getter). Named sources get a null-guard chain.
10. **SVG namespace**: every SVG tag (`g`, `path`, `circle`, …) is created with `createElementNS`,
   also when returned from a separate helper (`const icon = () => <g><path/></g>`). Tags that exist
   in both HTML and SVG (`a`, `title`, `style`, `script`) are SVG only under an `<svg>` ancestor in
   the same JSX tree; `<foreignObject>` switches back to HTML.
11. **Lowercase tag = local variable of the same name.** If a lowercase JSX tag matches a local
   variable/parameter in scope (`(content) => <div><content></content></div>`), the compiler inserts
   that variable as a component instead of creating an element. This is how `Virtualization`'s
   `mainTemplate` slot works; `{content}` there would be a text binding (`[object Object]`). Avoid
   naming locals after real HTML tags (`const div = ...`, `const input = ...`) inside components that
   render those tags.
12. **`x-style` equals `style`** (compiles to `sender.style(...)`, getter live). A style getter
   re-applies on change and clears keys it stops returning.
13. **`on:name` / `on-name` / `on_name` handlers follow the `motif.on` arity rule**: the handler is
   passed as is, so `fn.length <= 1` → `fn(event)`, otherwise `fn(sender, event)`. `fn.length` stops
   at the first default-valued or rest parameter: `(...a) => …` counts 0 and receives only the event.
   Only the prefix is stripped from the name: `on-my-event` → `my-event`, `on_my_event` → `my_event`.
14. **`ref={this.x}` / `ref={name}` is resolved by the compiler.** A method, a function-valued field
   or a local function is called with the component; a declared plain field or an uninitialised variable
   is assigned. When the compiler cannot tell (undeclared or inherited member, import, parameter,
   `props.x`, function-typed field, getter) the runtime calls it if it holds a function, else assigns.

## Runtime quirks

- `const { params } = useNavigation()` keeps that moment's value; read `nav.params.id` inside a getter for live params. A hidden `keepAlive` page also re-runs route bindings with other routes' values.
- `app.onRouterChanged` is the only after-navigation hook. It fires on every navigation including the first
  and 404s, with `{ uri, params, meta, route, ok, initial, redirectedFrom?, direction, state }`; check `initial` to skip the first load.
- `redirect` fires only on the matched **leaf**: on a layout that has a `'/'` default child it never fires
  (the child is the leaf) — put the redirect on the default child.
- `params` never contains the `#fragment`, but it **does** merge query-string values with path params.
  Query values are strings like path params (`/users/5?page=2` → `{ id: '5', page: '2' }`), so
  `params.page === 2` is always false: convert explicitly (`Number(params.page)`). `true`, `null`, dates
  and JSON stay text too; a repeated key is an array of strings.
  If a query key has the same name as a path param, the path param wins. Query values come only from the
  navigated address: in `history` mode no `?` means no query params (the previous page's query never
  leaks); in `hash`/`file` mode the page query before `#` (`/?x=1#/a`) is used for routes without their own.
- A layout route must render `<RouterView/>` (and every named `<RouterView name>` its children target
  through `extend.targetOutlet`). Without it the child page is never mounted; dev mode warns `MJX301`
  (`RouterView outlet '…' was not built within 3000 ms`) — not an exception.
- `fallbacks.notFound` / `fallbacks.error`: the fallback renders inside the deepest layout whose
  path prefix matches the URL (the shell stays), the address bar is updated, `app.router.ok === false`.
- `keepAlive: true` keeps the instance: on leave it is detached from the outlet and cached
  (`onDeactivated`), on return it is re-attached in the same outlet position (`onActivated`); both
  propagate to the page's visible subtree.
  Cached instances live until `app.router.evict(nameOrRoute)` (no argument → all) or `app.dispose()`.
- `class.add(...)` works on SVG elements (`<svg class="ring">`).
- `x:mounted` / `onMounted` observers are released on dispose even if the element never attaches.
- A hidden or waiting component (list rows included) is replaced by a `<!--h-->` trace, so
  `element.parentNode` is `null` while hidden; with `hideStrategy: 'detach'` no trace is left either. `hideStrategy` is set through the `options` prop (`options={{ hideStrategy }}`) or `motif.options.hideStrategy` in code.
- `Virtualization` rows that scroll out of view are detached (not disposed) and kept in an LRU
  cache (`cacheSize`); `refresh()`, `setData()` and an `autoRefresh` reload dispose them all. Do not
  hold external references to row components across a reload. In jsdom, fake `clientHeight`/`scrollTop` or nothing renders.
- On a **plain DOM tag** any `on*` prop that is not a lifecycle name (`<div {...props}/>` with `onClick`,
  `new Component('div', { onclick })`) is registered as a DOM listener for the suffix event. On a
  component tag non-lifecycle `on*` props stay in `this.props` (the compiler wires known DOM event
  names written inline, see below).
- **The compiler warns you.** `@motifx/compiler` emits non-fatal build warnings with file, line and a
  code frame: `MJX001` (block-bodied arrow declaring locals then returning a conditional →
  `ReferenceError`), `MJX002` (camelCase DOM event name on a component tag — `<Comp onChange>` binds
  a DOM listener, not a callback prop; lowercase `<Button onclick>` is the deliberate spelling and is
  never warned), `MJX003` (`.map()` item without `key`; rows are matched by the item object, so the key only feeds the
  `MJX202` duplicate check), `MJX004` (`some`/`every`/`find` on a `this.`
  rooted array inside a reactive getter loses dependencies), `MJX007` (unknown `x-*` name; it is passed on
  as an `on<name>` prop), `MJX015` (a class component override of `build`/`dispose`/`style`/… that
  can finish without reaching `super`; checked in `.ts`/`.js` files too). Disable with
  `compiler({ diagnostics: false })`. A separate type-aware check, `npx motif-lint`,
  reports `MJX005`: a ternary on a component prop whose declared type is not `Bind<T>` (the compiler
  cannot see prop types; TypeScript does not see the getter wrapping). A hit is a real contract
  violation.
- **Some JSX fails the build.** `x-reload`, `x-bind`, `x-effect`, `x-focus`, `x-interrupt`, `x-to`,
  `x-list`, `x-loop` are not directives (`MJX006`); use `x-wait`/`x-display`, `{items.map(i => <X key={i.id}/>)}`,
  `effect(...)` or `ref` + `focus()` instead. A `function () {}` child is `MJX008` (write `{() => …}`).
  A string/boolean event handler (`onclick="go()"`) is `MJX009`; invalid lifecycle/directive values and
  unsupported tag or child forms are `MJX010`–`MJX013`.
  The thrown error carries the code in `error.code`.
- **Runtime messages are coded too.** Thrown framework errors are `MotifError` (`code`, message
  `[motifjs] MJX302: …`); errors the framework catches and reports (component and route hooks incl. rejected async hooks `MJX122`/`MJX306`, guards `MJX306` which cancel the navigation, event handlers and application event listeners incl. `onRouterChanged` `MJX123`, effects `MJX208`, dispose and data-load failures)
  reach `errorHandler.addListener(fn)` as `MotifError` with the original error in `cause`; warnings print
  only in development. These coded reports do not go to `setUnexpectedErrorHandler`.
- A lifecycle listener added in code with `this.motif.on('x:built', fn)` (likewise `x:configured`,
  `x:disposed`, `x:mounted` and the other `x:` names) that throws is reported as `MJX122`
  (`The component x:built hook threw.`), the same as the hook written as a method or a JSX prop. A
  throwing `ref` callback on a plain tag or a class component tag is `MJX122` too; the component is
  still created.
- **Not sure what a `{…}` compiles to? Ask the compiler:** `npx motif-explain file.tsx` prints,
  per expression, the real lowered call, its reactivity class (`LIVE`/`STATIC`/`ONCE`/`RECEIVER`/`RUNTIME`) and its dependency surface. Two facts it makes visible that people miss:
  `attr={f(x)}` on a DOM tag is evaluated ONCE (write `attr={() => f(x)}` to keep it live), and
  `prop={object}` on a component passes the value once (a reactive proxy stays live only if the
  receiver reads its fields inside a getter).
- **A ternary in attribute/prop position is ALWAYS wrapped lazily** (`mode={x ? 'a' : 'b'}` →
  `mode: () => x ? 'a' : 'b'`), on DOM tags and component tags alike. This is deliberate: without it
  the expression would be evaluated once during `view()` and freeze. The contract is that the
  receiving component declares the prop as `Bind<T>` and reads it through `read()` / `toGetter()`.
  A component that declares a plain type and uses the prop directly is the thing that is wrong (an
  `Icon` reading `name` directly gets `() => …` and draws its fallback glyph); fix the receiver, not
  the call site. Every other expression form passes by value.
- A valueless attribute on a component tag (`<Comp flag />`) passes `flag: true`. Member-expression
  tags (`<Foo.Bar/>`) compile like any other component tag.
- **Do not shadow `ComponentBase` members.** Reserved on a class component: `build`, `dispose`,
  `disposeAsync`, `style`, `setText`, `setState`, `reState`, `using`, `doWork`, `getService`,
  `useModel`, `$`, `context`, `siblings`, `serviceProvider`, `isWait`, `element`, `props`,
  `controls`, `class`, `attr`, `bindings`, `motif`, `parent`, `childs` and the `is…` state flags
  (`isBuilt`, `isVisible` …). `view()`, the lifecycle hooks, `initializeComponent` and
  `onElementCreating` are meant to be written; an override of `build`/`dispose`/`style` … calls
  `super.<name>(...)`. TypeScript reports only clearly incompatible types (`style = 'red'`); it
  misses `any` fields, compatible-signature methods (`build() {}` leaves the component empty, a
  page's `dispose()` without `super` is never torn down on navigation, a parameterless `style()`
  drops the tag's `style`). The `is…` flags and `motif` are accessors, so redeclaring one as a
  field (`declare` included) is a TS2610 error. The compiler warns `MJX015` at build time for an
  override that skips `super` on any path (`if (x) return;`, one-branch `if`, loop, `catch`,
  callback), in `.ts`/`.js` files too; only `if (this.isBuilt|isDisposed|isWait) return;` in
  `build` and `if (this.isDisposed) return;` in `dispose` are exempt. In development mode `MJX128` reports, once per class,
  an override without `super`, an instance field hiding a method/accessor, and a replaced
  `controls`/`attr`/`class`/`bindings`/`motif`/`element`/`parent`; `props` and the flags are not
  checked. The framework operations `show`, `hide`, `toggle`, `on`, `off`, `trigger`, `addHandler`,
  `clear`, `register`, `setDisposable`, `stopAnimations` and `options` live on `this.motif`, so a
  subclass may use those names for its own members.
- **Reach for `x-wait`/`x-display` before `{cond && <X/>}`.** The directive keeps the instance and
  swaps a `<!--h-->` placeholder, and when the condition starts hidden the element is never built at
  all (subtree included, lifecycle hooks silent). `{cond && <X/>}` constructs a new instance on every
  show and disposes it on every hide — correct only when that churn is what you want. `&&` and
  ternary branches are rebuilt only when the condition value changes, so a plain-expression component
  prop inside a branch (`<Badge count={state.n}/>`) is a snapshot; pass `count={() => state.n}` and
  read it as `Bind<T>`. A single element
  whose class/text is a getter is the cheapest for mutually exclusive one-liners. Consecutive
  `x-display` siblings under one parent are safe.
- Directives and lifecycle props written on a **function component** tag (`<InfoBar x-display={...} />`,
  `onbuilt`, `x-mounted`, `x-initializing` and the other lifecycle spellings, `initializeComponent`, `ref`) are applied to the root the function returns — the function needs no
  `runover` prop and must not forward anything. Class components receive them in the constructor.
- **Short-circuit inside a reactive getter loses dependencies.** `arr.some(...)`, `every`, `a || b` stop
  reading once the answer is known, so the binding never subscribes to the rest. Loop over everything
  and accumulate instead.
- **`untracked(fn)`** reads without creating dependencies — use it when an effect
  writes to a store it also reads, or when a debounced async effect must consult sync state.
- **Template/slot props** (`<Card headerTemplate={<h1/>}/>` read as `{props.headerTemplate}`) work
  directly — no `{() => ...}` needed and no component-vs-data annotation. But the placement is decided
  once, in the parent's `initializeComponent`: a template that is `null` then and assigned later needs the getter form
  `{() => props.headerTemplate}` (a `Frame` slot); the direct form then writes nothing and warns
  (`MJX204`).
- A component callback prop whose name matches a **known DOM event** (`onChange`, `onInput`, `onSelect`,
  `onToggle`, `onResize`…) is consumed as that DOM listener and does **not** reach `this.props`. Name callbacks after
  domain events instead: `onValueChange`, `onAdd`, `onDismiss`.
- `Application.CreateBuilder()` may be called once per page; a second call while an app is running throws
  `MJX405` (it is allowed once `app.dispose()` has been called, even without `await`). Call `rebuild()` for
  hot-reload scenarios. `app.dispose()` disposes the services last, after the tree and its leave animations,
  so every `onDisposing` / `onDisposed` and code running during an animation still sees services and `context`.
- `this.context.on(...)` returns an unsubscribe function and is removed automatically when the component
  is disposed (so is `this.context.onRouterChanged`). `this.context.onLifecycle(...)`,
  `Application.main.on(...)` and a captured `app.on(...)` are not: keep and call (or
  `motif.setDisposable`) their unsubscribe.
- Raw `effect()` outside components never stops by itself.
- **An effect must not write a value it also reads.** `state.n++` inside an effect is a read *and* a
  write, so the effect retriggers itself. Wrap bookkeeping writes (counters, logs) in `untracked(...)`.
  The framework bounds this: an effect that runs 50 times in one flush is skipped for the rest of that
  flush and an `MJX203` dev warning names it; other queued effects are unaffected.
- **Chained state settles in the same microtask.** Effect A writing state that effect B reads is
  supported and order-independent; one `await tick()` is enough.
- An effect that throws does not abort the rest of the flush; the error is reported as `MJX208` and reaches
  `errorHandler.addListener(fn)` as `MotifError` (original error in `cause`).
- `scoped` services: one instance per navigation (everything the navigation puts on screen, constructor
  injection included, shares it; callers hold a handle that follows the current navigation); without a
  router there is one instance on the root provider, disposed with `app.provider.dispose()`. See di.md.
- SSR is unsupported by design. Importing `@motifx/core` works without a DOM, but creating any element
  throws `MJX101` when no `document` exists.

## Diagnosing "it does not update"

1. Is the value read **inside** a getter or directly as `state.field` in JSX? (`{label}` is static.)
2. Is the object actually a `reactive()` proxy (not a destructured copy or a class field assigned before proxying)?
3. Are the list items objects (rows follow the item object, not `key`)? Are you mutating the proxied array (`state.items.push`) rather than a stale reference?
4. Did an `untracked` or `peek()` wrap the read?
5. Was the component disposed (`isDisposed`) before the async result arrived? Use `doWork`/`using`.
