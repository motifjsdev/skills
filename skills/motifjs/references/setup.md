# Setup and build

## Packages

| Package | Role |
|---|---|
| `@motifx/core` | Runtime. ESM + CJS + UMD builds, bundled types at `dist/index.d.ts` (`dist/index.d.cts` for `require`). Exposes `./devtools` (`debugGetDeps`, `debugGetDepMap`), `./jsx-runtime` and `./jsx-dev-runtime` (types only; JSX is compiled by @motifx/compiler). |
| `@motifx/compiler` | Compiler plugin + `motif-lint` / `motif-explain` CLIs. Default export: the Vite plugin function (`compiler()`). Named exports: `Compiler`, `explain`, `printDiagnostics`, `printExplanations`, `formatExplanations`, `lintProject`, `lintProgram`, and the types `MotifVitePluginOptions`, `MotifDiagnostic`, `MotifExplanation`, `ExplainSite`, `ExplainReactivity`, `LintOptions` (see below). It depends on its own TypeScript (`>=5 <7`) for the type-aware lint, so it works whatever TypeScript version the project uses. |

Programmatic API of `@motifx/compiler`:

| Export | Shape |
|---|---|
| `new Compiler()` | `start(code, filename)` / `motifCompile(code, filename)` (same call) → Babel result or `null`; afterwards `compiler.diagnostics: MotifDiagnostic[]`. `explain(code, filename)` also fills `compiler.explanations`. `lowerDecorators(code, filename)` lowers only decorators |
| `MotifDiagnostic` | `{ code, message, file, line: number \| null, frame? }` |
| `MotifExplanation` | `{ file, line, column, site: ExplainSite, shape, source, lowered, reactive: ExplainReactivity, deps, note? }` |
| `ExplainSite` | `'child' \| 'element' \| 'attr' \| 'prop' \| 'event' \| 'directive' \| 'component-event' \| 'key'` |
| `ExplainReactivity` | `'live' \| 'static' \| 'once' \| 'receiver' \| 'runtime' \| 'n/a'` |
| `LintOptions` | `{ project?: string, files?: string[] }` for `lintProject(options?)` → `MotifDiagnostic[]`; an unreadable tsconfig throws `MJX014` |

Install: `npm i @motifx/core` and `npm i -D @motifx/compiler vite typescript`. Keep both on the same major version: the
compiled output depends on a runtime surface (the compiler contract) that changes only in a major version; a
mismatch warns `MJX121` in development.

## Vite

```ts
import { defineConfig } from 'vite';
import compiler from '@motifx/compiler';

export default defineConfig({
  plugins: [compiler()],   // enforce: 'pre'; transforms .jsx .tsx .aio .mjsx .mtsx (node_modules included); lowers decorators in .ts .mts .cts .js .mjs .cjs (not in node_modules, .d.ts, or under a tsconfig with experimentalDecorators); skips ?raw/?url/?worker/?sharedworker/?inline and virtual modules
  esbuild: { jsx: 'preserve' },       // esbuild must NOT touch JSX
  resolve: { extensions: ['.tsx', '.ts', '.jsx', '.js'] },
  server: { port: 3000 },
});
```

Rollup: pass the same `compiler()` to a plain Rollup build; there is no separate Rollup or Babel plugin.

### Per-file opt-out

A file containing the comment `//useReact` or `//useVue` is skipped by the plugin (useful when a
single file must compile with another JSX runtime).

### Plugin options

```ts
compiler({
  diagnostics: false,        // silence the MJX001–MJX004 and MJX007 compile-time warnings (default: on)
  explain: 'pages/Home',     // print "what did this expression compile to" for matching files
})                           //   true = every file; string = path includes; RegExp = test(id)
```

## CLI tools (shipped with `@motifx/compiler`, in `node_modules/.bin`)

**`motif-explain`** — the compiler's `-S` flag. JSX in MotifJS is shorthand for the binding API
(`bindings.add`, `bindings.when`, `controls.add`…); the compiler is a *lowering* pass. This prints,
for every JSX expression in a file, the REAL lowered call (taken from generated code, not guessed),
its reactivity class and its dependency surface. Use it whenever you are unsure what a shape does.

```sh
npx motif-explain src/pages/Home.tsx              # readable dump, source order
npx motif-explain src/pages/Home.tsx --site prop  # only component props (child|element|attr|prop|event|component-event|directive|key)
npx motif-explain src/pages/Home.tsx --json --code
```

Reactivity classes: `LIVE` (live effect), `STATIC` (literal such as a string, number, negative number, boolean or `null`; no binding), `ONCE` (evaluated once
in `initializeComponent` — `attr={f(x)}`, `prop={object}`), `RECEIVER` (passed as a function; liveness depends
on the receiver's `Bind<T>` contract), `RUNTIME` (`attr={variable}`: function → live, value →
once). Programmatic: `import { explain } from '@motifx/compiler'` → `MotifExplanation[]`.

**`motif-lint`** — type-aware check the compiler cannot do (it never sees prop types). Builds the
TypeScript program from `tsconfig.json` and reports `MJX005`: a ternary written on a component prop
whose DECLARED type does not accept a function. The compiler always wraps an attribute ternary in
`() => …` (by design, for reactivity), so `<Icon name={ok ? 'a' : 'b'}/>` hands `Icon` a function
while TypeScript sees `IconName`. Silent on DOM tags, `Bind<T>`/`any`/`unknown`/function-typed props,
`key`/`x-*`/`on:*` names and generic `value: T` props. Exit code 1 on findings.

```sh
npx motif-lint                            # ./tsconfig.json
npx motif-lint -p tsconfig.json src/a.tsx # one project / specific files
npx motif-lint --json --no-fail           # CI-friendly
```

Add `"lint:jsx": "motif-lint"` to `package.json` scripts (the starter and ui-kit templates have it).

## tsconfig

```jsonc
{
  "compilerOptions": {
    "target": "ES2021",
    "module": "esnext",
    "moduleResolution": "bundler",
    "strict": true,
    "jsx": "preserve",              // TypeScript must not transform JSX
    "useDefineForClassFields": true,
    "lib": ["ESNext", "DOM", "DOM.Iterable"]
  },
  "include": ["./src/**/*"]
}
```

JSX types come from `@motifx/core` (`declare global { namespace JSX {...} }`), so `JSX.Element` is a
`ComponentBase`. Intrinsic elements accept every DOM property as `value | () => value`, plus any
extra key (index signature), so unknown attributes never fail type-checking.

## Host page

```html
<body>
  <div id="app"></div>
  <script type="module" src="/src/index.tsx"></script>
</body>
```

## Application bootstrap

```tsx
import { Application } from '@motifx/core';

const builder = Application.CreateBuilder();   // only one builder/app per page
builder.services.addSingleton(Api, Api);       // optional DI registrations
const app = builder.build();                   // also auto-registers @Injectable classes

app.useDevelopment(true);                      // MJX warnings, route linter, devtools (call before useRouter)
app.useLogging(true);                          // console error logging (on by default)
app.useReactiveMonitor({ enabled: true, threshold: 200 }); // dev-only leak warnings

app.useRouter({ routes, mode: 'history', fallbacks: { notFound: () => import('./pages/NotFound') } });

app.run('#app');                 // routed app: default <RouterView name="default"/> is mounted
// or
app.run('#app', new AppRoot());  // explicit root component
```

`Application.main` is the global instance; inside components use `this.context`.

## What the compiler does (so you can predict output)

- `<div>{state.name}</div>` → `sender.bindings.add("textContent", state, "name")`
- `<div>{() => expr}</div>` and template literals → `sender.bindings.method(fn)` (reactive text)
- `x-wait={() => cond}` → `sender.bindings.wait(() => cond)` (preferred for show/hide); `x-display` is its inverse
- `{cond && <X/>}` → `sender.bindings.when(() => cond, () => <X/>)` (rebuilds/disposes on each toggle, only when the condition value changes)
- `{cond ? <A/> : <B/>}` → `sender.bindings.ternary(() => cond, frame => ..., frame => ...)` (branch rebuilt only when the condition value changes)
- `{arr.map(fn)}` → `sender.bindings.list(() => arr, fn)` (rows matched by the item object; `key` is only checked for duplicates)
- `attr={() => v}` → reactive attribute binding; `attr={a + b}` / `attr={!s.open}` / `attr={s.x}` → wrapped in a getter (live);
  `attr={v}` → passed as is (live only when `v` is a function); literals (`0`, `-1`, `true`/`false`, `null`,
  arrays, `new X()`) → passed as values
- `class X extends Component<HTMLDivElement>` → injects `static elementTag = 'div'`

Attribute names starting with `x-` are normalised: `x-ref` → `ref`, lifecycle names `x-config` /
`x-built` / `x-mounted` … → `onconfig` / `onbuilt` / `onmounted` … (several spellings of one hook
on a tag become one ordered list: `onbuilt: [a, b]`), `on:name` / `on-name` / `on_name` →
`sender.motif.on("name", fn)` with the handler passed as is (only the prefix is stripped), `ref` / `x-ref`
→ one `ref` function (or an ordered list when both are written; the runtime takes it out of the props
before the component or the function sees them), the code generated for the tag → `initializeComponent`
(inside `runover` on component tags, merged with a user-written `initializeComponent` as `[user, compiled]`),
lifecycle spellings on component tags → `runover`, `x-wait` / `x-display` /
`x-html` / `x-style` → the corresponding bindings. `@Injectable` and other standard decorators are
lowered by the plugin (in the JSX files it compiles and in plain `.ts`/`.js` files, with the exceptions
listed in the Vite comment above); only the registration is runtime: `builder.build()` registers the
decorated classes.
