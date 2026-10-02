# Phone-style apps and Electron / `file:` apps

MotifJS has no Electron- or Capacitor-specific API. What it offers for these targets is router
behaviour (`stack`, swipe-back, hash routing, `scrollMemory`, `keepAlive`), app lifecycle events and a
runtime that needs no `eval`. Full router reference: routing.md.

## Page stack (`useRouter({ stack })`)

```ts
app.useRouter({
  routes,
  stack: { retain: true, depth: 5, animation: 'slide', duration: 300, swipeBack: { edge: 24, threshold: 0.35 } },
});
```

| Option | Default | Notes |
|---|---|---|
| `stack` | off | `true` = all defaults. Leaving a page keeps its instance for back/forward instead of disposing it. Ignored in `mode: 'shell'`. |
| `retain` | `false` | `true`: kept pages stay in the DOM, hidden (`display:none !important` + `inert` + `aria-hidden`); `false`: detached, kept in memory. |
| `depth` | `5` | max kept previous pages; the oldest is disposed. |
| `persist` | `false` | write the stack's URIs to `sessionStorage`. |
| `animation` | `'none'` | `'slide'` or `async ({ direction, entering, leaving }) => void`; runs only for `push`/`back`/`forward`; skipped under `prefers-reduced-motion: reduce`. |
| `duration` | `300` | ms, for `'slide'` and the swipe settle (60 % of it). |
| `swipeBack` | off | `true` or `{ edge = 24, threshold = 0.35 }`. |

Swipe-back: only a single touch that starts within `edge` px of the left edge, on a page that has a
previous entry, installs a (non-passive) move listener. Once the finger has moved 8 px mostly to the
right the page follows it and scrolling is prevented; a vertical or leftward move lets the touch go.
Releasing past `threshold` of the page width, or a rightward flick faster than 0.5 px/ms, calls
`history.back()`; otherwise the page snaps back. With `retain: true` the previous page is shown
underneath while dragging. A swipe-back navigation plays no page transitions.

`data-nav-direction`: with `stack`, whenever the stack animation does not run (no `animation`, or a
`initial`/`replace`/`traverse` navigation), the page's own transition (`transition` /
`motif.options.transition`) is the stack transition, and while it runs the page element carries
`data-nav-direction` (`initial` | `push` | `back` | `forward` | `replace` | `traverse`):

```css
[data-nav-direction="back"].page-enter-active { animation: slide-from-left .25s; }
[data-nav-direction="push"].page-enter-active { animation: slide-from-right .25s; }
```

`keepAlive: true` on a route keeps that route's instance across navigations (with or without `stack`);
it is detached, not disposed, and receives `onDeactivated` / `onActivated` (routing.md, lifecycle-and-dispose.md).

## App lifecycle (`app.onLifecycle`)

```ts
const off = app.onLifecycle((e) => {
  if (e?.state === 'resumed' || e?.state === 'online') store.refresh();
});
```

| `state` | Source event |
|---|---|
| `'visible'` / `'hidden'` | `document` `visibilitychange` |
| `'frozen'` / `'resumed'` | `document` `freeze` / `resume` |
| `'restored'` | `window` `pageshow` with `persisted` (back/forward cache) |
| `'online'` / `'offline'` | `window` `online` / `offline` |

The handler receives `{ state, visible, online }`; `app.isVisible` and `app.isOnline` read the same
values at any time. The DOM listeners are installed on the first `onLifecycle` call and removed by
`app.dispose()`. The returned function unsubscribes. `onLifecycle` is **not** auto-disposed, not even
through `this.context`; in a component write
`this.motif.setDisposable(this.context.onLifecycle(fn))`.

## `file:` pages (Electron `loadFile`, packaged apps)

- Without `mode`, a page whose `location.protocol` is `file:` routes by the hash (`#/orders/5`), like
  `mode: 'hash'`; `'file'` is an explicit name for the same behaviour. Do not set `mode: 'history'` there:
  it reads `location.pathname`, which is the file path.
- `mode: 'shell'` never reads or writes the address bar or history and always starts at `/`; browser
  back/forward does not move it, and `stack` is ignored.
- The router stamps its own history entries (`history.state.__motifHistory`); when a back/forward
  navigation is cancelled (guard, `onLeave`, middleware) it moves the history back with `history.go()`
  so the address matches the page on screen. Calling `history.pushState` directly can skew that index.
- The runtime (`@motifx/core`) contains no `eval` / `new Function`, so a CSP without `'unsafe-eval'`
  works for MotifJS itself; JSX is compiled at build time. Third-party bundles have their own needs.

## Scroll and viewport

- `scrollMemory` (off by default) restores the scroll position per path on back/forward, link and
  reload, from `sessionStorage`; `container` targets an inner scroller instead of the window. When on,
  `history.scrollRestoration` is `'manual'`. Options in routing.md.
- `stack` restores the inner scroll positions of a kept page when it returns.
- `Virtualization` re-renders its window when its scroll container's height changes (rotation,
  on-screen keyboard) through a `ResizeObserver` (virtualization.md).
- `Lazy` `retry: { count, delayMs, whenOnline = true }` retries a failed load; while the browser is
  offline it waits for the `online` event first (routing.md).
