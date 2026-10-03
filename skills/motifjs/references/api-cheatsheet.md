# API cheat-sheet (`import { ... } from '@motifx/core'`)

Ground truth: `node_modules/@motifx/core/dist/index.d.ts` (value + type exports), `dist/jsx-runtime.d.ts`
(JSX types) and `dist/devtools.d.ts` (`@motifx/core/devtools` subpath).

## Application

```ts
Application.CreateBuilder(): ApplicationBuilder      // .services: ServiceCollection; .build(): Application; .rebuild()
Application.main: Application                        // global instance
app.run(host: string | HTMLElement | Node, root?: ComponentBase): void   // no root → default <RouterView name="default"/>
app.useRouter(routes | { routes, mode?, fallbacks?, hooks?, scrollMemory?, stack? }): Application
app.useGuard((ctx: { to, from }, next: (to?: string | false) => void) => void): Application
app.use((ctx: RouteResolveContext, next) => any): Application        // ctx: { uri, context, rewritePath(uri) }
app.navigate(uri, options?: NavigationOptions): Promise<any>         // options are forwarded to the router; resolves to the navigation result, e.g. { ok: false, cancelled: true, reason: 'guard' }
app.navigateByName(name, params?, options?: NavigationOptions): Promise<any>
app.router.navigate(uri, options?: NavigationOptions): Promise<void>  // { replace?, state?, force?, scroll? }
app.router.params / .route / .uri / .ok / .meta / .extend / .chain   // one object per app; read-only, reactive
app.router.direction   // 'initial' | 'push' | 'replace' | 'back' | 'forward' | 'traverse'
app.router.state       // the `state` given to navigate() for the current history entry; comes back on back/forward and reload
app.router.stack       // only with useRouter({ stack }): [{ index, uri, current, retained }]
app.router.fullPath / app.router.aliasOf             // matched pattern / canonical pattern when matched via alias (else null)
app.router.evict(nameOrRoute?): Promise<void>      // release cached keepAlive instances (no argument → all); unknown name rejects with MJX302
app.router.resolve(uri): ResolveResult                // match without navigating: { ok, uri, fullPath, route, chain, params, meta, extend, aliasOf }
app.router.href(name, params?): string               // path of a named route (throws MJX302 for an unknown name)
app.router.routes: RouteInfo[]                        // route table in definition order: { fullPath, name, meta, route, chain }
app.on(event, handler) → unsubscribe fn; app.off(event, handler); app.fire(event, args?)
app.onRouterChanged((e: RouterNavigatedEventArgs) => …) → unsubscribe fn   // { uri, params, meta, route, ok, initial, redirectedFrom?, direction, state }; every navigation incl. initial + 404
app.onLifecycle(({ state, visible, online }) => …) → unsubscribe fn   // state: 'visible'|'hidden'|'frozen'|'resumed'|'restored'|'online'|'offline'; listeners installed lazily, removed by app.dispose(); never tied to a component, also not through this.context: call the unsubscribe yourself
app.isVisible; app.isOnline
app.useDevelopment(bool = true); app.useLogging(bool = true); app.useReactiveMonitor({ enabled, threshold, name })
app.isDevelopmentModeEnabled; app.restartRouter() (builds a fresh router from the latest useRouter config and restarts at the current address/shown page: routes, keepAlive cache, stack and scroll memory reset; page rebuilt; onRouterChanged fires with initial: true; no-op before run)
app.provider: ServiceProvider; app.getAppShell(): Component; app.dispose()   // disposes router, lifecycle listeners and provider; resets the address to '/' with history.replaceState
app.insert(c) / app.attach(c)  // add to the app shell; app.remove(c) (dispose) / app.detach(c) (no dispose); app.appendToMainHost(node)
useNavigation(): Router   // the same object as app.router; read nav.params.id live, destructuring keeps that moment's value
useApplication(): { application, services, router, attach(...components) }   // application = Application.main
```

Inside a component `this.context` is the application seen through a component-scoped view:
`this.context.on(...)` and `this.context.onRouterChanged(...)` subscriptions are removed
automatically when the component is disposed; `this.context.onLifecycle(...)` is not (keep its
unsubscribe and pass it to `this.motif.setDisposable`). Calling `on` on `Application.main`, a captured
`app` or `useApplication().application` is not tied to any component.

## Components

```ts
class Component<TElement = any, TProps extends object = any> extends ComponentBase<TElement, TProps>
  static elementTag?: string; static elementNamespace?: string
  constructor(elementOrProps?: TElement | string | IBaseProp<TProps>, props?: IBaseProp<TProps>)
FNComponent(view: (props) => JSX): (props) => FragmentNode
FragmentNode, Frame, Transport, TransportTo, ContentBody, ContentBlock, RouterView, RouterLink, Virtualization<T>
Lazy({ caller: () => Promise<T>, options?: LazyOptions<T> }): Component      // a function, used as <Lazy …/>
motifComponent(el, props?), motifFragment(props?), motifCompiled(contract)   // what the compiler emits (_mc / _mf / _mv); MJX121 on a contract mismatch (dev)

// ComponentBase members
element, props, childs, parent, context, controls, bindings, class, attr, motif
view?(): any; initializeComponent?(sender): void; oninitializeComponent?(sender, e): void
onInitializing/onInitialized/onConfig/onConfigured/onBuilding/onBuilt/onMounted/onVisibilityChanged/onActivated/onDeactivated/onDisposing/onDisposed(sender, e)
onRefCreated?(sender)   // after each ref={this.x} / ref={name} in this class's view(); not for callback refs
build(); setState(); reState(); setText(s); style(v)
isVisible; isWait (get/set)
dispose(opts?: { deep?, skipLeaveTransition? }): Promise<void>; disposeAsync(opts?)
using(promise, onfulfilled?, onrejected?)   // callbacks skipped once disposed
doWork(promise): Promise<T>                 // resolves to an Error instance (does not reject) if disposed meanwhile
getService<T>(token): T | null; serviceProvider; useModel(obj)
$(selector): { fromDom(), fromComponent() }; siblings.{all,next,prev,nextAll,prevAll}()
isBuilt, isInitialized (false in onInitializing, true from onInitialized on), isConfigured, isDisposed, isPainted

// component.motif (ComponentMotif): framework operations; a subclass may define its own show/on/clear/options
motif.on(event, cb, domEvent = true): Promise<this>; motif.off(event, cb): Promise<this>; motif.trigger(event, args): Promise<this>; motif.addHandler(event, handler)
motif.show(): Promise; motif.hide(): Promise; motif.toggle()
motif.clear(): Promise; motif.register(d: IDisposable); motif.setDisposable(fn); motif.stopAnimations(): Promise<void>
motif.options                                          // transition, hideStrategy, disableDisposal
```

`Frame`: `navigate(component | component[], keepOldControl = false)`, `navigateLazy(caller, options?, keepOldControl?)`,
`flush()`, `current`, `isBusy`, `motif.clear()`.

`Transport` props `{ name, mode?: 'replace' | 'merge' }` (+ `clearSlot()`); `TransportTo` props `{ name }`.
`ContentBody` props `{ name }`; `ContentBlock` props `{ target }`.
`Transporter.transport(child, newParent, options?)`, `transportMany(children, newParent, options?)`,
`createSlot(name, parent?)` — the low-level mover behind `Transport` (`TransportOptions { index?, keepState?, owner? }`).
`transport` unlinks the child from its old parent and adds it to the new one; the child stays alive (not disposed).

## ControlCollection (`component.controls`)

`add(...c)` / `add(index, ...c)`, `insert(index, ...c)`, `remove(c)` (dispose), `clear()` / `clearAsync()` (dispose all),
`detach(c): Promise<void>` (leave animation, no dispose), `silentDetach(c)`, `silentUnlink(c)`,
`move(c, before?)`, `moveToIndex(c, index)`, `forEach(fn)`, `map(fn)`, `items`, `length`, `onAdd`, `onRemove`.

## BindingCollection (`component.bindings`)

`add`, `text`, `value`, `html`, `when`, `ternary`, `list`, `loop`, `switchCase`, `method`, `watch`, `model`, `wait`, `display`, `remove`, `clear`, `items`. See jsx-and-reactivity.md.

## Reactivity

```ts
reactive<T>(m: T): T; useModel<T>(m: T): T
createSignal<T>(v): Signal<T>; new Signal(v, equals?); Signal.create(v, equals?); createComputed<T>(fn): Computed<T>; createLazyComputed<T>(fn): LazyComputed<T>
effect(fn, onValueChanged?): () => void   // onValueChanged(value) runs after every run of fn (the first included) with fn's return value, untracked
untracked(fn): T; deepClone(v); clearModel(m)
Signal: value, peek(), update(fn), mutate(fn), notify(), asReadonly(): ReadonlySignal<T>, dispose()
Computed: value, peek(), dispose(); LazyComputed: value, peek(), isDirty, dispose()
type Bind<T> = T | (() => T); toGetter(v: Bind<T>): () => T; read(v: Bind<T>): T
configureReactivityLeakMonitor({ enabled?, threshold?, name? })   // same switch as app.useReactiveMonitor
asyncTracking                                                    // used by compiler output (Virtualization dataRequest); not for app code
```

`import { debugGetDeps, debugGetDepMap } from '@motifx/core/devtools'` — dependency maps of reactive objects (tests/diagnostics).

Devtools flag: `?devtools=1` in the page URL or `window.__MOTIF_DEVTOOLS__ = true`, read when `app.useRouter()`
runs. It turns on dev warnings (except `MJX301`, which needs `useDevelopment(true)`) and the route linter, and
exposes `window.__motifDevBus` (`getWarnings()` → collected `{ code, message, details }`; `on(fn)` receives
`{ type: 'warning', payload }`). Ctrl+\` / Cmd+\` toggles a fixed overlay box at the bottom of the page (visible
from the start with `useDevelopment(true)`); the box renders no content, so read warnings from the console or
`getWarnings()`.

## DI

```ts
ServiceCollection: addSingleton/addScoped/addTransient(token, impl), tryAdd*(token, impl): boolean, replace(token, impl, lifetime?), remove(token), has(token), reset(), getDescriptor(token), buildServiceProvider()
ServiceProvider: get(token), getAsync(token), createScope(name?), dispose()
Injectable({ lifetime?, deps? }), inject(token) → instance (only during provider construction, else MJX409), FromService(token) → instance | null (dev warning MJX414 on failure); add*(token, fn) with a non-constructible fn → MJX413
type ServiceLifetime = 'transient' | 'singleton' | 'scoped'
```

## Routing

```ts
type RouteItem = { path, control?, childs?, name?, meta?, extend?, keepAlive?, redirect?: string | ((to: { path, params, meta }) => string), alias?: string | string[], validate?, onShow?, onEntering?, onEnter?, onLeave?, onUpdate? }   // control or redirect required
RouterView({ name? }); RouterLink({ to, el?: string | Node, activeClass?, exactClass?, onActive?, offActive?, onExact?, offExact?, showHref?, target?, text?, bypass? })   // renders <a> unless el is given; showHref:false omits href; bypass, a target other than '_self' and Ctrl/Cmd/Shift/Alt clicks are left to the browser; text = link text when there are no children
types: RouterOptions, ScrollMemoryOptions, NavigationOptions, NavigationGuard, NavigationGuardContext, RouterNavigatedEventArgs, RouteRedirect, RedirectTarget, RouterEvents, ResolveResult, RouteInfo
```

## Disposal

`IDisposable { dispose() }`, `Disposable`, `DisposableStore { add, delete, clear, dispose, isDisposed }`,
`disposableCore.toDisposable(fn)` / `toDisposable(fn)`, `disposableCore.disposableTracker` (live-instance
counting, see lifecycle-and-dispose.md).

## Collections and queries

`Query.from(iterable)` / `new Query(iterable)`: `where`, `select`, `orderBy`, `orderByDescending`, `distinctBy`,
`groupBy` (→ `Query<Group<TKey, T>>`, `Group { key, items }`) chain; `toArray`, `first`, `firstOrDefault`,
`any(pred?)`, `all(pred)`, `aggregate(seed, fn)` finish; a `Query` is iterable. Eager (each step builds an
array); the source is never mutated. Arrays have no LINQ methods of their own.
`List`, `Dictionary`, `LinkedList`, `NameValuePair`.

## Errors, utilities

`MotifError` (`Error` subclass with `code: MotifErrorCode`, an `MJX…` code; the framework throws it and passes it to `errorHandler` listeners; `MJX503` carries the individual dispose errors in `cause.errors`),
`errorHandler` (`ErrorHandler`: `addListener(fn) → unbind`, `setUnexpectedErrorHandler`, `safeCall`…),
`setUnexpectedErrorHandler(fn)` (receives uncoded errors caught by `safeCall`/`safeCallAsync` and by `Emitter` listeners, which also reach `addListener` as they are; the default rethrows in a `setTimeout`; coded reports such as `MJX122`/`MJX123`/`MJX208` never reach it),
`safeCall(fn, context, fallback?)`, `safeCallAsync`, `safeCallSilent`,
`Resilience.create.retry({...}).timeout({ timeoutMs }).circuitBreaker(...).bulkhead(...).rateLimiter(...).fallback({ fallback }).execute(fn)`,
`decorate(policy, fn)`, `dom.createElement/createElementNS/createComment/createTextNode`,
`NodeTypes`, `Emitter<T>` / `Event`, `preProcessing`.
Types: `EventArgs { cancel }`, `TransitionProps`, `IBaseProp<T>`, `LazyOptions`, `Virtualization*` types.

`Emitter<T>`: `fire(value)`, `event` (an `Event<T>`: `(listener, thisArgs?, disposables?) => IDisposable`),
`dispose()`. A service exposing an event:

```ts
class CartService {
  private readonly changed = new Emitter<Item[]>();
  readonly onChanged: Event<Item[]> = this.changed.event;
  add(item: Item) { this.items.push(item); this.changed.fire(this.items); }
}
this.motif.register(cart.onChanged(items => …));   // the IDisposable is released with the component
```

## MJX codes

`MotifError.code`; messages read `[motifjs] MJX…: …`. "Thrown" errors reach the caller; "reported" ones go
to `errorHandler.addListener` (original error in `cause`); "dev warning" prints only in development.

| Code | Kind | When |
|---|---|---|
| `MJX101` | thrown | creating an element with no `document` (SSR / plain Node) |
| `MJX108` | value | `doWork(p)` resolves to this `MotifError` (does not reject) when the component was disposed meanwhile |
| `MJX110` / `MJX111` | `Lazy` load error | `timeoutMs` elapsed / `signal` aborted; passed to `onError` (after a timeout `Fallbackview` is shown) |
| `MJX121` | dev warning | compiled code's compiler contract differs from the `@motifx/core` runtime |
| `MJX122` | reported | a component hook, `ref` callback or `x:` lifecycle listener threw (`The component onBuilt hook threw.`) |
| `MJX123` | reported | an event handler or application event listener threw (`The 'click' event handler threw.`) |
| `MJX124` | dev warning | a spread object on a DOM tag carried `innerHTML`; the key was ignored |
| `MJX125` | dev warning | a spread object on a DOM tag carried a `javascript:` URL for `href`/`src`/`action`/`formaction`/`xlink:href`; the value was ignored |
| `MJX201` | thrown | a list `renderFn` returned something that is not a component, class or factory |
| `MJX202` | dev warning | duplicate list `key` (rendering unaffected) |
| `MJX203` | dev warning | an effect ran 50 times in one flush; skipped for the rest of that flush |
| `MJX204` | dev warning | a component reached a text binding (template prop assigned after setup; write `{() => props.tpl}`) |
| `MJX205` | reported | `Virtualization` `watch` callback threw |
| `MJX206` | dev warning | `Virtualization` `autoRefresh` stopped (`dataRequest` keeps changing the data it reads) |
| `MJX207` | reported | `Virtualization` data load failed |
| `MJX208` | reported | an effect threw |
| `MJX301` | dev warning | a layout's `RouterView` outlet was not built within 3000 ms |
| `MJX302` | thrown | `href` / `evict` with an unknown route name |
| `MJX303` | dev warning | redirect loop or more than 10 redirects; `fallbacks.error` renders |
| `MJX304` / `MJX305` | reported | a route control failed to construct or load / the error fallback itself failed |
| `MJX306` | reported | a route hook, `onShow` or guard threw (a guard cancels the navigation) |
| `MJX309` | thrown | navigating before `useRouter()` |
| `MJX310`–`MJX316` | dev warning | route linter: empty path, child path/alias without `/`, child equals parent path, duplicate full path, alias equals own path, duplicate alias |
| `MJX401` | thrown | token not registered (and not `@Injectable`) |
| `MJX404` | thrown | cyclic service dependency |
| `MJX405` | thrown | second `Application.CreateBuilder()` before `app.dispose()` |
| `MJX406` | thrown | `app.run(selector)` found no host |
| `MJX409` | thrown | `inject()` outside a provider construction |
| `MJX413` | thrown | `add*(token, fn)` with a non-constructible function |
| `MJX414` | dev warning | `FromService` failed; it returns `null` |
| `MJX503` | thrown | several disposables threw while disposing a store (errors in `cause.errors`) |
| `MJX601` | thrown | `Query.first()` on an empty sequence |
| `MJX602`–`MJX605` | thrown | `Resilience`: circuit open / bulkhead queue full / rate limit exceeded / timeout (`TimeoutError`) |

Compiler (`@motifx/compiler`): warnings `MJX001` (block-bodied arrow in child position declares locals the
branches cannot see), `MJX002` (camelCase DOM event name on a component tag), `MJX003` (`.map()` item without
`key`), `MJX004` (`some`/`every`/`find`… inside a reactive getter), `MJX007` (unknown `x-*` directive, passed on
as `on<name>`); `MJX005` from `motif-lint` (ternary on a prop not typed `Bind<T>`); build errors `MJX006`
(unsupported directive), `MJX008` (`function () {}` as a JSX child), `MJX009` (string/boolean event handler),
`MJX010` (invalid lifecycle hook value), `MJX011` / `MJX012` (invalid directive value / object literal),
`MJX013` (unsupported JSX tag name or child); `MJX014` (`motif-lint` cannot read the tsconfig).

## JSX-only props

`key`, `ref` / `x-ref`, `x-html`, `x-wait`, `x-display`, `x-style`, `initializeComponent` (plain and component tags),
`transition`, `childs`, `on<event>[:modifier]` (one modifier per attribute), `on:name` / `on-name` / `on_name`
(handler passed as is to `motif.on(name, fn)`), `on<lifecycle>` / `x-<lifecycle>` / `x:<lifecycle>`
(`onmounted`, `onactivated`, `ondeactivated`… included; several spellings on one tag all run in source order), `options={{ hideStrategy, disableDisposal }}`,
method-like props that call the element method and write no attribute (`focus`, `blur`, `click`, `select`, `scrollIntoView`, `show`/`showModal`/`showPopover` (`true` opens, `false` closes), `close`, `togglePopover`, `requestSubmit`, `checkValidity`, `reportValidity`, `showPicker`, `load`, `setSelectionRange`/`setRangeText`/`setPointerCapture`/`releasePointerCapture`/`fastSeek` (value = argument, array = argument list), …; a getter re-calls on change). `hideStrategy` is given through the `options` prop
(`options={{ hideStrategy: 'detach' }}`) or set on `motif.options.hideStrategy` in code; a bare
`hideStrategy="…"` attribute is not special (it becomes a DOM attribute / ordinary prop).
