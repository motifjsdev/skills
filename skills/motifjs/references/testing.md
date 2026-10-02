# Testing MotifJS components

The framework's own suite uses **Jest + jsdom + ts-jest**, constructing components without JSX so
no compiler step is needed. Mirror that for unit tests; when you need JSX, compile through the
`@motifx/compiler` Vite plugin (Vitest, or Vite + a browser).

## Jest config (jsdom)

```js
// jest.config.cjs
module.exports = {
  testEnvironment: 'jsdom',
  transform: { '^.+\\.tsx?$': ['ts-jest', { tsconfig: { jsx: 'preserve' } }] },
  moduleNameMapper: { '^@motifx/core$': '<rootDir>/node_modules/@motifx/core/dist/index.cjs',
                      '^@motifx/core/devtools$': '<rootDir>/node_modules/@motifx/core/dist/devtools.cjs' },
};
```
In an application, map `@motifx/core` to the installed build `node_modules/@motifx/core/dist/index.cjs`
(and `@motifx/core/devtools` to `dist/devtools.cjs`) exactly as above; no build step is needed.

Working on the framework itself: the monorepo's `packages/tests` mapper points at
`packages/motifjs/dist/index.cjs`, so build `@motifx/core` first (`npm run build --workspace=@motifx/core`,
then `npm test --workspace=motifjs-tests`).

## Pattern

```ts
import { Component, ComponentBase, reactive } from '@motifx/core';
const nextTick = () => Promise.resolve();

test('text binding updates', async () => {
  const data = reactive({ message: 'Hello' });
  const c = new Component('div', {
    initializeComponent: (s: ComponentBase) => { s.bindings.text(() => data.message); },
  });
  c.build();
  document.body.appendChild(c.element as Node);
  await nextTick();
  expect(c.element.textContent).toBe('Hello');

  data.message = 'World';
  await nextTick();
  expect(c.element.textContent).toBe('World');

  await c.dispose();
  expect(c.isDisposed).toBe(true);
});
```

- `component.build()` runs config → initializeComponent → view → built.
- Reactive updates flush on a microtask: `await Promise.resolve()` before asserting.
- Animations: jsdom lacks WAAPI; MotifJS shims it, but await `motif.show()/motif.hide()/dispose()` promises.
- Testing a class with `view()`: instantiate, `build()`, append `element`, assert on
  `element.querySelector(...)`; bindings created by the compiler behave identically to manual ones.
- Events: `element.dispatchEvent(new Event('input', { bubbles: true }))` after setting `.value`.
- Errors in hooks, event handlers and effects do not throw into the test: they are reported. Capture
  them with `const off = errorHandler.addListener((e) => errors.push(e))` and assert on `e.code`
  (`'MJX122'` hook, `ref` or `x:` listener, `'MJX123'` event handler or `app.on`/`onRouterChanged` listener, `'MJX208'` effect) and `e.cause`; call `off()` afterwards.
- One `Application` per page: call `app.dispose()` at the end of a test before building another
  (it also resets the address to `/` with `history.replaceState`).
- Memory tests: create+dispose inside an inner function, hold a `WeakRef`, call `global.gc()` (run
  Jest with `node --expose-gc`).

## Full-app smoke in jsdom

Load the built Vite bundle via `vite.createServer().ssrLoadModule` or run the real browser with
Playwright; `Application` needs `document` and (for `history` mode) `window.history`.
