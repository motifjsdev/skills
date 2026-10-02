# Virtualization (large lists)

Use `Virtualization<T>` for thousands of rows whose heights are fixed or computable in advance, or for
paged/infinite data. For < ~100 rows or heights known only after rendering use a normal `.map`.

```tsx
import { Component, reactive, Virtualization } from '@motifx/core';

interface Todo { id: number; title: string; completed: boolean }

export default class TodoList extends Component<HTMLDivElement> {
  state = reactive({ todos: [] as Todo[] });

  async onConfig() {
    this.state.todos = await this.doWork(fetch('/api/todos').then(r => r.json()));
  }

  view() {
    return <Virtualization<Todo>
      x-wait={() => this.state.todos.length === 0}
      itemHeight={48}
      pageSize={40}
      dataRequest={async ({ page, pageSize }) => {
        const start = page * pageSize, end = start + pageSize;
        return { items: this.state.todos.slice(start, end), totalCount: this.state.todos.length, hasMore: end < this.state.todos.length };
      }}
      itemTemplate={(todo, i) => (                 // called once per row; the component is reused while scrolling
        <div class="row">
          <input type="checkbox" checked={() => todo.completed} onchange={(e) => todo.completed = e.target.checked} />
          <span>{todo.title}</span>
        </div>
      )}
      mainTemplate={(content, st) => (
        <div class="list">
          <header>{() => st.data.length} loaded</header>
          <div class="scroll"><content></content></div>      {/* rows go here */}
        </div>
      )}
      loadingTemplate={(st) => <div class="spinner" />}
      emptyTemplate={() => <div>No rows</div>}
      errorTemplate={(err) => <div class="error">{err.message}</div>}
      autoLoad autoLoadThreshold={200}
    />;
  }
}
```

## Props (`VirtualizationProps<T>`)

| Prop | Type | Notes |
|---|---|---|
| `dataRequest` | `({ page, pageSize, scrollTop }) => Promise<{ items, totalCount, hasMore }>` | **required**; `page` is 0-based; `totalCount` only fills `state.totalCount` (the scrollbar and spacers use the loaded data); `hasMore: false` stops further page requests |
| `itemTemplate` | `(item, index) => JSX` | **required** |
| `itemHeight` | `number \| (item, index) => number` | **required**; a number for fixed rows, a function for known per-row heights (offsets are summed once per data change, lookup is a binary search) |
| `pageSize` | `number` | default 50 |
| `mainTemplate` | `(content, state) => JSX` | wrap the list; place `<content></content>` or `{content}` where rows go (see the placement rule below) |
| `loadingTemplate`, `emptyTemplate`, `errorTemplate` | | `errorTemplate(error)` shows failures of the first load, `refresh()` and `autoRefresh` reloads (without it they are reported as `MJX207`); a failing next-page request (`autoLoad`) is reported as `MJX207` and stored in `getState().error`, the loaded rows stay, no error view is shown, it is retried on the next scroll and a successful load clears `error` |
| `filter` | `(items: T[]) => T[]` | applied to the loaded data; spacers/indices use the filtered length |
| `overscan` | `number` | extra rows rendered above **and** below the viewport; default = visible row count |
| `cacheSize` | `number` | max off-screen row components kept alive for reuse (LRU); default `max(200, 3 x window)` |
| `overscanPages`, `pageBuffer`, `renderMode` | | reserved; `'page'` mode is not implemented |
| `autoLoad`, `autoLoadThreshold` | | load next page when scrolled within `threshold` px (default 100) of the **loaded** data's bottom, and also when the rendered window comes within `overscan` rows of the loaded end (e.g. the first page does not fill the viewport); `autoLoad` defaults to on. With a fixed `itemHeight` the bottom is the **unfiltered** `data.length x itemHeight`, while the spacers use the filtered length; with a `filter` that hides rows the threshold check measures past the real content end and the window check (filtered length) is what loads the next page. With a height function both use the filtered rows |
| `autoRefresh` | `boolean` | default on: when reactive data read synchronously inside `dataRequest` changes, the loaded range (page 0 to the current page) is requested again in one call and the row components are rebuilt; `false` → reload only through `refresh()`. Reads after an `await` in an inline `dataRequest` are tracked too (the compiler instruments it). The watch is set up by the first load and `refresh()`; `setData()` stops it. More than 30 reloads within one second stop the watch (a `dataRequest` that writes the data it reads); dev mode warns `MJX206` |
| `watch` | `() => unknown` | extra dependencies for `autoRefresh`: reactive fields read here also trigger a reload (fallback when the reads cannot be seen). A throw is reported as `MJX205` and the request still runs |
| `className`, `style` | | container styling |

`VirtualizationState<T>`: `{ isLoading, isInitialized, currentPage, totalCount, hasMore, error?, data }`.

Public fields: `wrapper` is the scroll container (its `scrollTop` drives the window). Without
`mainTemplate` it is the `Virtualization` element itself (`overflow-y: auto; height: 100%; position: relative`);
with `mainTemplate` it is the nearest ancestor of `content` that owns a real element, found when `content`
is built, and it receives `overflow-y: auto; position: relative` and the scroll listener, so give that
element a height. `contentWrapper` is the `content` component handed to `mainTemplate` (spacers + rows).

A `ResizeObserver` on the scroll container re-renders the visible rows when its height changes
(rotation, on-screen keyboard); it is skipped when `ResizeObserver` is missing and disconnected on
dispose. Heights that are only known after measuring (free-text feeds) are not supported.

## How it renders (what to expect in the DOM)

- The DOM holds only the window `[first, last)` = `[anchor - overscan, anchor + visible + overscan)`
  where `anchor = floor(scrollTop / itemHeight)`. Rows are plain siblings between two spacer
  `div`s (`.motif-virtualization-top-spacer` / `-bottom-spacer`); spacer heights are never negative
  and `top + rows + bottom = filteredLength x itemHeight`.
- A row leaving the window is **detached, not disposed**: the same component (its bindings, input
  state, `reactive(row)` proxy) comes back when scrolled into view. No `<!--h-->` placeholders
  accumulate. Off-screen rows beyond `cacheSize` are disposed oldest-first.
- Every scroll event re-renders the window (also when `hasMore === false`); only the difference to
  the previous window is touched, so cost is O(window), not O(data).
- Rows are cached by item object identity (by index for primitive items); the same object may appear
  more than once in the data and every occurrence gets its own row. An `autoRefresh` reload,
  `refresh()` and `setData()` dispose every cached row.
- The first render waits for layout: if `clientHeight` is 0 (not yet in `document`) it is retried
  in `onMounted`. In jsdom tests fake `clientHeight`/`scrollTop` with `Object.defineProperty` and
  dispatch `new Event('scroll')` (see `packages/tests/src/components/virtualization.test.ts`).
- `refresh()` reloads page 0 through `dataRequest` and disposes every cached row (returns a
  `Promise`); `setData(items)` replaces the data without a request (`hasMore` becomes `false`);
  `scrollToIndex(i)` scrolls and renders synchronously; `getState()` returns a shallow copy of the
  state.
- A row that leaves the window receives `onDeactivated`; a cached row coming back receives
  `onActivated`.
- Paged mode: the scrollbar represents **loaded** pages only (the loaded rows after `filter` x row height), giving an
  infinite-scroll feel; it grows as pages arrive.

## Placement rule for `content`

Both forms place the rows component inside `mainTemplate`:

- `<content></content>`: the compiler turns **any** lowercase tag name that is not an HTML/SVG tag name
  into a variable reference, whether or not such a variable exists (an undeclared name fails at runtime),
  and a component instance is placed as is. Trap: a parameter named like an HTML tag (`header`, `main`,
  `section`, `slot`) produces that DOM element instead; keep the name `content` or another non-HTML name.
  A name with `-` stays a custom element.
- `{content}`: a bare identifier or member expression child is checked once, when the child is created: if
  it holds a component (or a non-empty array of components) it is placed; any other value becomes a
  text binding.
