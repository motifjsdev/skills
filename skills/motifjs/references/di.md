# Dependency injection

.NET-style service collection built into `Application`.

## Lifetimes

| Lifetime | Instance |
|---|---|
| `singleton` | one per application |
| `scoped` | one per navigation: everything on screen in that navigation (layout, page, children, `FromService`, singleton deps) sees the same instance; without a router, one instance on the root provider |
| `transient` | new on every resolve (default) |

### `scoped` in detail

Resolving a `scoped` service (`getService`, `inject`, `deps`, `FromService`) returns a **handle**, not the
instance. It behaves like the service (fields, methods, `#private`, `instanceof`) and forwards every access
to the right navigation's instance:

- A layout that stays mounted moves to the new navigation's instance automatically; a binding such as
  `{() => this.cart.items.length}` re-runs. The previous navigation's instance is disposed.
- A page that left stays bound to its own navigation's instance, so work that finishes after an `await`
  (through a handle captured before it) never writes into the new page's service. A param change
  (`/p/1` → `/p/2`) and back/forward are new navigations.
- A cached `keepAlive` page (and a page waiting in the mobile stack) keeps its navigation's instance alive
  and moves to the current one when shown again. State that must survive navigations belongs in a
  `singleton`.
- A failed navigation (guard cancel, page constructor error) keeps the current instance.
- `app.provider.createScope()` is outside this: it returns real instances that live only in that scope.

Handles are one object per component; compare services by content, not with `===` across components.

## Register

```ts
const builder = Application.CreateBuilder();
builder.services.addSingleton(LoggerService, LoggerService);          // class
builder.services.addSingleton(Config, { apiUrl: '/api' });            // value
builder.services.addSingleton(Db, { useFactory: (config: Config) => new Db(config), deps: [Config] });
// deps → factory is called (…deps, provider); without deps it gets (provider). Promise / async factory → resolve with `await app.provider.getAsync(T)` (`get` throws MJX402)
builder.services.addScoped(AuthService, AuthService);
builder.services.addTransient(RequestId, RequestId);
builder.services.tryAddSingleton(T, T);   // boolean: only if absent
builder.services.replace(T, Impl, 'singleton'); builder.services.remove(T); builder.services.has(T);
const app = builder.build();              // builds provider + auto-registers @Injectable classes
```
Only `useClass` is constructed with `new`. A `useValue` and a factory's result are returned as is, and
`get`/`getAsync` return the same value — to provide a function or a class itself, use `{ useValue: fn }`
or a factory that returns it; a factory that wants an instance calls `new` itself. A bare function passed
to `add*` is treated as a class: a non-constructible one (arrow, `async`, method) is a type error and throws
`MJX413` at registration — use `{ useValue: fn }` or `{ useFactory: fn }`.

Constructor/factory dependencies can also live on the class or factory itself as a static array,
`static inject = [...]` or `static dependencies = [...]`; a descriptor's own `deps` wins over both.

```ts
class OrderService {
  static inject = [Db, LoggerService];                  // constructor params, in order
  constructor(private db: Db, private log: LoggerService) {}
}
builder.services.addSingleton(OrderService, OrderService);

const makeCache = (config: Config) => new Cache(config.apiUrl);
makeCache.dependencies = [Config];                    // factory gets (...deps, provider)
builder.services.addSingleton(Cache, { useFactory: makeCache });
```

The router resolves a route component from the provider when it is registered (`add*` or
`@Injectable`) and constructs it directly otherwise. A registered component that cannot be resolved
(missing dependency, cycle, constructor error) fails the navigation with `MJX304` (the original error is
its `cause`) and shows `fallbacks.error`.

## `@Injectable` and `inject()`

`@Injectable` is a standard (TC39) class decorator: no `experimentalDecorators` / `emitDecoratorMetadata`
in tsconfig, no `esbuild.target` in vite.config. `@motifx/compiler` lowers decorators in `.ts`/`.mts`/`.cts`/`.js`/`.mjs`/`.cjs`
files as well as the JSX files it compiles, so dev and build behave the same; files under `node_modules` and `.d.ts` files are
not lowered, and a `.ts`/`.js` file whose nearest tsconfig sets `experimentalDecorators: true` is left to esbuild's legacy
transform, where `@Injectable` also works. Parameter decorators (`constructor(@X() dep)`) do not compile.

```ts
import { Injectable, inject } from '@motifx/core';

@Injectable({ lifetime: 'singleton' })
export class LoggerService { log(m: string) { console.log(m); } }

@Injectable()                                   // default: transient
export class UserService {
  private logger = inject(LoggerService);       // typed from the token
  constructor() { this.logger.log('ready'); }   // usable in the constructor too
}

@Injectable({ deps: [Db, LoggerService] })      // constructor params, in order (not type-checked)
export class OrderService {
  constructor(private db: Db, private log: LoggerService) {}
}
```
`inject(token)` works only while the provider constructs the class (field initializers, constructor
body, or a `useFactory` call) and resolves from that provider, so a `scoped` dependency comes from the
requesting route scope. Anywhere else — a method, a callback, `new UserService()` by hand — it throws
`MJX409`. It is synchronous: async registrations need `deps` or `getAsync`. Components use
`this.getService(token)`, not `inject`. Use `deps` when a test should construct the class by hand.

`@Injectable({ lifetime, deps })`. The decorator runs when the module
is evaluated (in `.ts` and `.tsx` files alike) and the class is registered automatically at
`builder.build()` — no manual `add*` needed. A class whose module loads *after* `build()` (a service
imported only by a lazily routed page) is registered on first resolve instead, so `getService` works
there too. An explicit `add*`/`tryAdd*` registration always wins over the decorator.

## Other members

| Member | Meaning |
|---|---|
| `provider.enableAutoDisposeTransients(enable = true)` | transients resolved from a child scope (`createScope()`, a navigation scope) are disposed with that scope; scopes created afterwards inherit the flag; transients resolved from the root provider are not tracked |
| `provider.getAllServices()` | the `ServiceCollection` behind the provider |
| `services.getService(token)` | the `ServiceDescriptor` for `token` or `null` (same as `getDescriptor`; registers an unregistered `@Injectable` class on the way) |
| `services.canAutoRegister(token)` | `true` when `token` is an `@Injectable` class not registered yet |

## Errors

| Code | When |
|---|---|
| `MJX401` | `get` / `getAsync` / `inject()` (or a dependency of the service) asks for a token that is neither registered nor `@Injectable`: `Service not registered for token: X` |
| `MJX402` | `get` on a Promise / async factory; use `getAsync` |
| `MJX404` | a dependency cycle: `Cyclic dependency detected: A[useClass] -> B[useClass] -> A[useClass]` |
| `MJX405` | a second `Application.CreateBuilder()` before `app.dispose()`: `Only one ApplicationBuilder instance is allowed.` |
| `MJX406` | `app.run('#sel')` with a selector that matches nothing |
| `MJX407` / `MJX408` | dev warnings from `this.getService` (no provider / resolution failed); it returns `null` |
| `MJX409` | `inject()` outside a provider construction |
| `MJX413` | `add*(token, fn)` with a non-constructible function |
| `MJX414` | dev warning from `FromService` when resolution fails; it returns `null` |

## Resolve

```ts
class Dashboard extends Component<HTMLDivElement> {
  auth = this.getService(AuthService)!;      // nearest provider; null if unregistered or resolution fails (dev warning MJX407/MJX408)
}

import { FromService, useApplication } from '@motifx/core';
const logger = FromService(LoggerService);           // anywhere (app provider); null + dev warning MJX414 on failure
const { application, services, router, attach } = useApplication();
services.get(LoggerService);
Application.main.provider.get(LoggerService);
```

## Reactive service state

Services are plain classes; make their state reactive when the UI should follow it:

```ts
@Injectable({ lifetime: 'singleton' })
export class CartService {
  state = reactive({ items: [] as Item[] });
  get total() { return this.state.items.reduce((a, b) => a + b.price, 0); }
}
// <span>{() => cart.total}</span>
```
