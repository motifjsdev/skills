# Routing

## Define routes

```ts
// config/routes.ts
import type { RouteItem } from '@motifx/core';

export const routes: RouteItem[] = [
  {
    path: '/',                                             // layout (has childs)
    control: () => import('../layouts/MainLayout'),
    childs: [
      { path: '/',                      control: () => import('../pages/Home') },      // default child
      { path: '/docs/{page?:overview}', control: () => import('../pages/Docs') },
      { path: '/orders/{id}',           name: 'order', control: () => import('../pages/Order'),
        meta: { requiresAuth: true },
        onLeave: ({ from, to }) => hasUnsaved() ? { cancel: true, reason: 'Unsaved' } : undefined,
        onUpdate: ({ to }) => reload(to.params.id) },
      { path: '/old', redirect: '/docs' },
    ],
  },
];
```

`RouteItem` fields: `path`, `control` (component class, instance, `() => Component`, an Options API
factory `() => ({ el, view })`, `() => import(...)`, or a Promise; required unless the route has `redirect`), `childs`, `name`,
`meta`, `extend`, `keepAlive`, `redirect`, `alias`, `validate(e) => boolean`, `onShow(component)`,
`onEntering(ctx)`, `onEnter(ctx)`, `onLeave(ctx)`, `onUpdate(ctx)`. `RouteItem` is a type that
requires `control` or `redirect` (a route with neither is a TS error). `meta` and `extend` are merged
along the matched chain (a layout's `meta.requiresAuth` is seen by its children).
A lazily imported page must be the module's `export default`; for a named export map it:
`() => import('../pages/Home').then(m => m.HomePage)`. A module without `default` (or another non-component object)
throws `MJX127`, reported as the `cause` of `MJX304`, and `fallbacks.error` is shown.
`extend.targetOutlet: 'name'` mounts the route into the named `<RouterView name="name">` instead of
`default`. `validate(e)` runs for every route of a candidate chain (layouts included) and receives
`{ uri, key, routes, params }` (`key`: the route's `name`, else its full path; `params`: the matched
path and query params); returning `false` makes the route not match (another route or `notFound` wins).

Hook order: the current route's `onLeave` → app guards (`useGuard`) → `onUpdate` (same route, params
changed only) → `onEntering` (page not created, not in the DOM yet) → page built and mounted →
`onShow` → `onEnter` → `onRouterChanged`. A parameter change on the same route rebuilds the
page (that is the router's job); `keepAlive: true` keeps the instance and only the hooks run. Do not route UI
state such as list filters unless rebuilding the page on each change is what you want.
Data prefetch in `onEntering` covers both a new route and a param change. `onLeave` also runs on a
param change of the same route. `onLeave` cancels the navigation by returning `false`
(`reason: 'onLeave'`) or `{ cancel: true, reason? }` (`reason` defaults to `'onLeave'`); any other result
lets it continue.

Errors: a throwing (or rejecting) `onLeave` / `onEntering` / `onUpdate` / `onEnter`, and a throwing
(or rejecting) `onShow`, is reported as `MotifError` `MJX306` and the navigation continues. A route control that
fails to construct or load reports `MJX304` and renders `fallbacks.error`.

`useRouter({ hooks: { onEntering, onEnter, onLeave, onUpdate } })` sets app-wide hooks; a route's own
hook replaces the global one (the global hook runs only for routes without their own).

Paths: layouts and default children use `'/'`; every child path starts with `/` (the dev linter
warns otherwise). At the same depth a static segment wins over a parameter (`/orders` beats
`/{slug}`), so a `/{slug}` sibling does not swallow real routes.

### `keepAlive`

`keepAlive: true` keeps the component instance across navigations: on leave it is **detached** from
its `RouterView` and cached (`onDeactivated(sender, e)` fires), on return the same instance is
re-attached in the same outlet position (`onActivated` fires; both also reach the page's visible
child components) — state, scroll, timers survive. Without
the flag a new instance is built on every visit. Cached instances are released by
`app.router.evict('routeName')` / `app.router.evict(routeItem)` / `app.router.evict()` (all; the instance
currently on screen is skipped) or `app.dispose()`. An unknown route name rejects with `MJX302`.

Query the route table without navigating: `app.router.resolve(uri)` (match result: `ok`, `uri`, `fullPath`,
`route`, `chain`, `params`, `meta`, `extend`, `aliasOf`), `app.router.href(name, params?)` (path of a named
route, e.g. for `<RouterLink to>`; unknown name throws `MJX302`; `0` and `false` values are written,
`undefined` / `null` / `''` fall back to the pattern default, and a param with no value and no default
drops its segment), and `app.router.routes` (every route in definition order, aliases excluded, as
`{ fullPath, name, meta, route, chain }`, for menus).

### `redirect` and `alias`

```ts
const routes: RouteItem[] = [
  { path: '/old-profile/{id}', redirect: '/users/{id}' },                        // placeholders from matched params
  { path: '/find/{q}', redirect: (to) => '/search?q=' + encodeURIComponent(to.params.q) },
  {
    path: '/users', control: UsersLayout, alias: '/people', childs: [          // /people/5 → UserDetail
      { path: '/', control: UserList },
      { path: '/{id}', control: UserDetail, alias: '/profile/{id}' },           // → /users/profile/{id}
    ],
  },
];
```

`redirect` (string with `{param}` placeholders, or `(to: { path, params, meta }) => string`):
- Applied right after matching, **before** `onLeave` and `useGuard`; guards and hooks only see the
  target, the source component is never created. `to.redirectedFrom` (guard / `onLeave` context) and
  `redirectedFrom` in the `onRouterChanged` payload hold the original address.
- String form carries the source `?query` and `#hash` over unless the target has its own; a function
  result is used as is.
- Chains are followed (`/a → /b → /c`). More than 10 hops or a loop stops: `fallbacks.error` renders at
  the requested address with `params.error` set to a `MotifError` `MJX303`, and dev mode warns `MJX303`
  (`Redirect loop or too many redirects: /a → /b → /a`).
- History gets only the target; if the address bar already shows the source (initial load, back/forward)
  the entry is replaced, so Back never bounces into the redirect.
- Only the matched **leaf** redirects. A layout with a `'/'` default child: at the layout path the leaf
  is that child, so a `redirect` on the layout never fires — put it on the default child instead.

`alias` (string or array):
- Opens the same route under another path; the address bar keeps the alias; params come from the alias
  pattern. Joined to the parent path exactly like `path`; a layout alias also opens its children.
- Canonical path ↔ alias is the **same route**: the component is kept, `onUpdate` runs only when params
  change, `keepAlive` cache is shared.
- A canonical path beats another route's alias on the same pattern. Dev linter warns on alias equal to
  its own path, duplicate aliases, child alias without leading `/`.
- `app.router.fullPath` is the matched pattern, `app.router.aliasOf` the canonical one (`null` if not
  an alias). `navigateByName` always builds the canonical path. `RouterLink` classes compare URLs, so
  `to="/users"` is not active on `/people`.

### Path parameters

| Syntax | Meaning |
|---|---|
| `{id}` | required |
| `{id:1}` | required; `1` is used only when `href` / `navigateByName` builds the path without a value (`/h` does not match `/h/{id:1}`) |
| `{id?}` | optional |
| `{id?:5}` | optional with default (`params.id` is `'5'` when absent) |

`href`, `navigateByName` and string `redirect` targets fill `{…}` placeholders.

`params` also holds the query string. Query values are strings, exactly as written in the address (after
URL decoding), just like path params: `/users/5?page=2` → `{ id: '5', page: '2' }`. Nothing is converted:
`true`, `null`, `42`, `2025-10-27` and `{"a":1}` all stay text, so convert where a number, boolean or date
is needed (`Number(params.page)`).

A repeated key becomes an array of strings (`?t=a&t=b` → `['a', 'b']`). A path param wins over a query
value of the same name; `#…` never reaches `params`.

## Enable the router

```ts
app.useRouter({
  routes,
  mode: 'history',            // 'history' | 'hash' | 'file' | 'shell'
  fallbacks: { notFound: () => import('./pages/NotFound'), error: () => import('./pages/Error') },
  scrollMemory: true,         // remember/restore scroll position per path (default: off)
  stack: { retain: true },    // mobile-style page stack (default: off) — see below
});
app.run('#app');              // no root → default RouterView is created
```
`app.useRouter(routes)` (plain array) also works. `scrollMemory: true` and `stack: true` turn the
feature on with every option at its default.

Modes: `'history'` (default; path + query of the address bar, `pushState`/`replaceState`);
`'hash'` and `'file'` behave the same (route after `#`); without `mode`, a page opened from a `file:` URL
(Electron) uses the hash behaviour — `'history'` is not usable under `file:` (the path is a file path);
`'shell'` never touches the address bar or history, always starts at `/`, ignores the page's own
query, and ignores `stack`; it installs no `popstate`/`hashchange` listener, so browser back/forward or
a `popstate` event never moves the router.

`fallbacks.notFound` accepts the same forms as `control`. When no route matches, the fallback is
rendered **inside the deepest layout whose path prefix matches the URL** (the shell stays), the
address bar is updated to the requested URL and `app.router.ok` is `false` (`params.path` holds the
URL). Without a fallback a built-in "404 - Not Found" panel is shown. `fallbacks.error` is used the
same way when a route control fails to load or throws (`params.error` holds the error; `MJX304` is
reported).

## Scroll memory — `scrollMemory`

Off by default. When enabled the router stores the scroll position of every visited path and puts it
back when you return there (back/forward, link click, page reload — the positions live in
`sessionStorage`). A hash in the URL wins over the stored position; a path seen for the first time
starts at the top; a per-navigation `scroll` option (`app.router.navigate('/x', { scroll: 'top' })`)
overrides the memory.

```ts
app.useRouter({
  routes,
  scrollMemory: {
    top: true,        // first visit starts at the top (default)
    anchor: true,     // an anchor (`#id`) wins over the stored position (default)
    settleMs: 1200,   // how long to keep re-trying while content is still rendering
    persist: true,    // keep positions in sessionStorage (default)
    limit: 60,        // how many paths to remember
    key: (path) => (path === '/docs' ? '/docs/giris' : path),   // fold aliases into one key
    container: '#content',   // remember an inner scroller instead of the window (selector | element | () => element)
  },
});
```
`container` is resolved on every navigation (it may not exist yet at startup); when it is not found the
window is used.

Restoring is retried every frame until the target is reachable (async content grows the document
later) or `settleMs` passes; user input (wheel/touch/key/mouse down) aborts it. Nothing listens to
`scroll`: the position is read once per navigation, after the guards pass and right before the URL is
applied and the content is swapped, so the leaving page's position is still real (a cancelled
navigation reads nothing). When enabled, `history.scrollRestoration` is set to `'manual'`.

The address-bar hash is read as an anchor only in `history` mode. In `hash`/`file` (and `shell`) mode
only the route's own anchor (`/page#section`) is used, since the address hash is the route; scroll
memory works there with the default `anchor` option.

## Layouts and `RouterView`

```tsx
import { RouterView } from '@motifx/core';

export default function MainLayout() {
  return <div class="layout">
    <header>…</header>
    <main><RouterView /></main>            {/* child routes render here */}
    <aside><RouterView name="sidebar" /></aside>
  </div>;
}
```
A route renders into the parent's `default` outlet unless its `extend.targetOutlet` names another one;
each route renders into exactly one outlet.

A layout (a route with `childs`) must render a `<RouterView/>` for every outlet its children target.
Until that outlet is built the child route is not mounted; after 3000 ms without it, dev mode warns
`MJX301` (`RouterView outlet 'default' was not built within 3000 ms, so the route chain cannot be
mounted.`).

The router opens one DI scope per navigation; route components it builds resolve constructor deps,
`inject()` and `getService()` from it, so a `scoped` service is one instance per navigation, shared by
everything on screen in it (see `di.md`).

## Links

```tsx
<a rel="router" href="/docs">Docs</a>               // intercepted, no reload

import { RouterLink } from '@motifx/core';
<RouterLink to="/docs" el="a" activeClass="active" exactClass="exact">Docs</RouterLink>
```
`<a>` interception also works with `data-router-link`; clicks with Ctrl/Meta/Shift/Alt, links whose
`target` is not `_self`, and (in `history` mode) other-origin URLs are left to the browser.

The link is an `<a>` element; pass `el` (a tag name or an existing `Node`) for another element.
`RouterLink` props: `to`, `el`, `activeClass`, `exactClass`,
`onActive/offActive`, `onExact/offExact`, `showHref` (`false` leaves `href` out; a click still
navigates), `target` (written as an attribute; any value other than `'_self'` leaves the click to the
browser, no router navigation), `text` (link text when the link has no children; children win).
`bypass` (`true` leaves the click to the browser as a plain link, no router navigation). A click with Ctrl/Cmd/Shift/Alt is left to the
browser; any other click calls `router.navigate(to)`. `activeClass`/`onActive` apply when the current path starts with `to`
segment-wise (`/users` is active on `/users/5`; `to="/"` only on `/`); `exactClass`/`onExact` only on
an exact match (query and `#` ignored).

## Programmatic navigation

```ts
this.context.navigate('/docs/routing');
await app.navigate('/docs', { replace: true });              // NavigationOptions: replace, state, force, scroll
await app.router.navigate('/docs', { replace: true });       // same options type
await app.navigateByName('order', { id: 42 });

import { useNavigation } from '@motifx/core';
const nav = useNavigation();      // the same object as app.router
nav.params.id;
```

`force: true` re-runs middleware, guards and `onRouterChanged` for the address already shown (otherwise
that navigation is skipped); the page is not rebuilt. `scroll`: `'top' | 'smooth' | 'instant' |
{ top, left, behavior }`, applied after the navigation, overrides `scrollMemory`.

`app.navigate` resolves to a result: the shown route's `ResolveResult` (`ok: false` for not-found and
error pages), `{ ok: true, skipped: true, uri }` when skipped, `{ ok: false, cancelled: true, reason }`
when a guard or `onLeave` cancels (`reason`: `'guard'`, `'onLeave'` or the `onLeave` reason), and
`{ ok: false, uri, cancelled: true, reason: 'middleware' }` when a middleware stops it. `app.router.navigate`
resolves to `undefined`.

`app.router` is one object for the whole application; `useNavigation()` and `useApplication().router`
return it. Its route fields (`params`, `route`, `uri`, `ok`, `meta`, `extend`, `fullPath`, `aliasOf`,
`chain`, `direction`, `state`, `stack`) are read-only and reactive: `<h1>{() => this.nav.params.id}</h1>`
re-runs on navigation, also in a layout that stays mounted.

- Read through the object: `const { params } = useNavigation()` or `const id = nav.params.id` keeps the
  value of that moment.
- The route changes after the old page is removed and before the new page mounts (before the stack
  transition), together with `scoped` services. The leaving page sees its own route in
  `onDeactivated`/`onDisposing`; a page the router constructs sees the target route in its constructor
  and field initializers. Guards and route hooks (`useGuard`, `onEntering`, `onUpdate`) run before the
  change: use their context (`to`, `params`). A cancelled navigation keeps the route.
- A cached `keepAlive` page keeps reading the current route while hidden, so its bindings re-run with
  other routes' values; do route-dependent work in `onActivated`.
- Before `useRouter()` the fields are empty (`params` `{}`, `uri` `''`, `route` `null`) and `navigate`
  throws `MJX309`.

`onRouterChanged` fires on **every** navigation including the initial one and 404s; its payload is
`{ uri, params, meta, route, ok, initial, redirectedFrom?, direction, state }` (`route` is `null` when
`ok` is `false`); check `initial` to skip the first load (for example in analytics).

## Guards, hooks, middleware (app level)

```ts
app.useGuard(({ to, from }, next) => {
  if (to.meta?.requiresAuth && !auth.isLoggedIn) next('/login');   // redirect
  else next();                                                     // continue; next(false) cancels
});
app.use(async (ctx, next) => { /* RouteResolveContext */ await next(); });
app.onRouterChanged(({ uri, initial }) => { if (!initial) analytics.page(uri); });   // after every navigation
```

Guards run on every navigation, the initial one included, in registration order; the first guard that
redirects or cancels ends the chain. A guard that never calls `next()` cancels. A guard that throws (or
rejects) cancels too: `MJX306` (`The guard hook threw.`) is reported and `app.navigate` resolves to
`{ ok: false, cancelled: true, reason: 'guard' }`; on back/forward the address is restored like any
other cancel.

Middleware (`app.use`) runs in registration order before redirects, `onLeave` and guards; `ctx` is
`{ uri, context (the Application), rewritePath(uri) }`. `rewritePath` changes the target (it becomes
the address shown); a middleware that does not call `next()` cancels. Middleware does **not** run for
the router's startup navigation (`run()` and `restartRouter()`); guards do.

A guard's `next('/login')` replaces the history entry when the address bar shows the blocked path
(initial load, back/forward), so Back does not loop into the same guard; from another page the target
is pushed normally. `app.navigate(uri, { replace: true })` and `app.router.navigate(uri, { replace: true })`
work in `hash`/`file` mode too; `app.navigate` forwards its options to the router.

**Cancelled back/forward restores the address.** The browser changes the URL before the router runs.
When a guard, `onLeave` or middleware cancels a back/forward navigation the router moves the history
back to the entry of the page still on screen (`history.go`), so URL and view never diverge. It keeps
an index in `history.state.__motifHistory`; code that calls `history.pushState` directly can skew it.

**Direction and entry state.** Every navigation carries `direction`
(`'initial' | 'push' | 'replace' | 'back' | 'forward' | 'traverse'`) in guards (`to.direction`),
`onRouterChanged` and `app.router.direction`. `navigate(uri, { state })` stores `state` on that history
entry; it is returned as `app.router.state` / `to.state` / event `state` again when the user comes back
with back/forward or reloads. Plain objects keep their fields (the router only adds its own key and
strips it on read); non-plain values (`Date`, `Map`) are stored untouched.

**Restart and dispose.** `app.restartRouter()` builds a fresh router and disposes the old one. It uses
the latest `useRouter` config when `useRouter` was called again after `run()` (such a config only takes
effect through `restartRouter()`), otherwise the current config. It restarts at the current address
(`shell` mode: the page the router shows, `/` if it never navigated); the route table is rebuilt, the
`keepAlive` cache and stack-kept pages are disposed, the page is rebuilt, no history entry is added, and
`onRouterChanged` fires with `initial: true`, `direction: 'initial'`. Before `run()` it does nothing.
`app.dispose()` also disposes the router and resets the address to `/` with `history.replaceState` only
(no hash write, no new history entry). `await app.dispose()` resolves once the pages, the `RouterView` and
the `run()` shell are disposed; the host element stays in the page and can host a new `run()`.

## Page stack — `stack`

Off by default: leaving a page disposes it. With `stack` the router behaves like a mobile navigation
stack — pushing keeps the previous page (state, form, inner scroll) and going back brings the **same
instance** back.

```ts
app.useRouter({
  routes, mode: 'history',
  stack: {
    retain: true,        // keep previous pages in the DOM, hidden (default false: kept in memory, detached)
    depth: 5,            // max previous pages kept (default 5; oldest is disposed)
    persist: false,      // write the stack's URIs to sessionStorage (app.router.stack survives reload)
    animation: 'slide',  // 'none' (default) | 'slide' | async ({ direction, entering, leaving }) => void
    duration: 300,
    swipeBack: true,     // drag from the left edge to go back (default off); { edge: 24, threshold: 0.35 }
  },
});
```
- Kept pages belong to the **history entry**, not the route: `/p/1 → /p/2 → back` returns `/p/1`'s own
  instance. Pushing the same route with other params creates a new instance (`onUpdate` still fires).
- push/forward keeps the leaving page; back disposes it (forward re-creates). `replace` does not keep.
  Pushing after going back disposes the stale forward pages. `keepAlive` routes keep their own cache.
- Hidden page: `display:none !important` + `inert` + `aria-hidden`; original inline style restored on
  return. `onDeactivated` / `onActivated` / route `onShow` fire. Inner scroll positions are restored.
- `animation` works with either `retain` value: with `retain: false` the leaving page stays in the DOM
  until the animation ends, then it is detached. Only the swipe-back underlay needs `retain: true`. Skipped under `prefers-reduced-motion`. A custom function receives both pages already
  layered; inline styles are restored afterwards. While it runs, the pages' own transitions do not.
- Without `animation`, the pages' own transitions (`motif.options.transition`) are the stack transition,
  in the same order as without a stack: the leaving page finishes its leave (then it is kept or disposed),
  then the entering page enters. A kept page coming back plays its enter again (either `retain`). During
  its transition the page element carries `data-nav-direction` (`initial` | `push` | `back` | `forward` | `replace` |
  `traverse`), removed when the transition ends: `[data-nav-direction="back"].page-enter-active { … }`.
  A swipe-back navigation plays no page transitions (the drag is the animation).
- `swipeBack`: touch starting within `edge` px of the left edge drags the page; past `threshold` of the
  width (or a quick flick) goes back via `history.back()`, otherwise it snaps back. A guard that cancels
  slides the page back. Only touches that start at the edge install the (non-passive) move listener.
- Disposing the router/app disposes every kept page. Ignored in `shell` mode.

## Lazy loading

`control: () => import('../pages/Reports')` splits the chunk and loads on first navigation.
`fallbacks.notFound/error` accept the same forms.

`Lazy` (component-level) accepts `retry: number | { count, delayMs = 500, whenOnline = true }` and
`onRetry(attempt, error)`: failed loads are retried (delay grows linearly), waiting for the `online`
event while offline. `onError` / `Fallbackview` apply only after the last attempt. Dispose or `signal`
abort stops retrying and removes listeners. Without `retry` there is a single attempt.

## Dev linter notes

With `useDevelopment(true)` called **before** `useRouter()` the router lints routes (the lint runs inside
`useRouter`). `MJX310` (empty path) and `MJX311` (child path without a leading `/`) fire for
`path: ''` / `'orders'`; use the warning-free form shown above (`'/'` for layouts and default children,
leading `/` on every child). Other lint codes: `MJX312` (child resolves to its parent's full path),
`MJX313` (two routes with the same full path), `MJX314` (child alias without leading `/`), `MJX315`
(alias equal to the route's own path), `MJX316` (duplicate alias).
