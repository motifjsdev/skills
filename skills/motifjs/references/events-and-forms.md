# Events and forms

## DOM events in JSX

```tsx
<button onclick={() => save()}>Save</button>
<input oninput={(e) => state.q = (e.target as HTMLInputElement).value} />
<form onsubmit={(sender, e) => { e.preventDefault(); submit(); }}>…</form>
<a onclick={(s, e) => { s.context.navigate('/home'); return false; }}>Home</a>
```

- Names are lowercase DOM names (`onclick`, `onchange`, `onkeydown`, `onpointerdown`). On a plain tag any
  DOM event binds this way, and a camelCase spelling (`onClick`) binds the same event.
- On a **component tag**, a prop named like a DOM event (`onchange`/`onChange`, `onCopy`, `onResize`,
  `onSelect`, `onReset`, `onContextMenu` …) is bound as a DOM listener on the component's root element and
  never reaches `this.props` (the compiler warns `MJX002` for the camelCase spelling). Name callback props
  so they do not collide: `onValueChange`, `onConfirm`.
- A handler that throws, or an async handler whose promise rejects, is reported as `MotifError` `MJX123`
  (`The '<event>' event handler threw.`, original error in `cause`) through `errorHandler`: logged to the
  console in production too (`app.useLogging(false)` turns it off) and delivered to
  `errorHandler.addListener(fn)`. Other listeners of the event still run.
- Handler arity decides the signature: 1 param → `(event)`, 2 params → `(sender, event)` where
  `sender` is the `ComponentBase` (`sender.element`, `sender.context`, `sender.props`).
- Returning `false` or `{ cancel: true }` → `preventDefault()` + `stopPropagation()`. These (and the
  `:prevent` / `:stop` modifiers) are called only when the event object has them: with
  `motif.trigger('name', { … })` and a plain data object nothing is called and no error is raised.
- Listeners are removed automatically on dispose.
- Arity is `fn.length`, which stops at the first default-valued or rest parameter: `(...a) => …`
  counts 0 and receives only the event.

### `on:name` / `on-name` / `on_name`

This spelling binds the handler as is to the tag's component: `<X on:save={fn}/>` is
`sender.motif.on('save', fn)`. Only the prefix is stripped: `on-my-event` → `my-event`,
`on_my_event` → `my_event`. It works for DOM events and for custom events fired with
`motif.trigger`, with the same arity rule (1 param → `fn(event)`, 2+ → `fn(sender, event)`).

```tsx
<Editor on:save={(e) => console.log(e.id)} />      // Editor: this.motif.trigger('save', { id: 7 })
<button on-click={(s, e) => s.motif.hide()}>Close</button>
```

### Modifiers (append with `:`)

`once`, `prevent`, `stop`, `capture`, `passive`, `self` (only when `event.target === element`),
`trusted` (only `isTrusted`). A JSX attribute takes one modifier (`onclick:once`, `onscroll:passive`; JSX syntax allows a single `:`);
chains such as `click:once:prevent` are written with the imperative API below.

## Imperative event API

The imperative API lives on the `this.motif` namespace (a subclass may define its own `on`/`off`/`trigger` methods).

```ts
this.motif.on('click', (s, e) => …);         // returns Promise<this>; auto-removed on dispose
this.motif.on('click:once:prevent', handler);
this.motif.on('x:mounted', () => …);         // once per subscription, when the element reaches document (immediately if already there); NOT again on re-attach — use x:activated; motif.off('x:mounted', fn) cancels a pending one
this.motif.on('x:built', (sender, e) => …); // lifecycle events via the x: prefix: initializing, initialized, config, configured, building, built, mounted, visibilityChanged, activated, deactivated, disposing, disposed; motif.off(name, fn) removes every subscription of fn; a throwing x: listener (x:mounted included) is reported as MJX122 ("The component x:built hook threw.")
this.motif.on('custom-thing', handler, false); // 3rd arg false = internal event only (no addEventListener)
this.motif.trigger('custom-thing', { data: 1 }); // fire internal/custom event
this.motif.off('click', handler);
this.motif.addHandler('click', handler);     // same as motif.on(event, handler)
this.motif.toggle();                          // hide when visible, show otherwise
```

- Handlers are stored under the full lowercased event string, modifiers included. `trigger('save')` does
  not reach a handler registered as `'save:once'`, and `off('click', fn)` does not remove
  `on('click:once', fn)`; pass the same string to `off`.
- `trigger(name, data)` calls the stored handlers directly (DOM listeners registered through `on` for
  that name included); it dispatches no DOM event.
- `off` for a DOM/custom event removes only the first record matching `fn`; a handler registered twice
  needs two `off` calls. For `x:` lifecycle names `off` removes every subscription of `fn`.

## Application-wide event bus

```ts
const off = this.context.on('languageChanged', (payload) => …);   // returns unsubscribe fn
this.context.fire('languageChanged', { lang: 'en' });
this.context.off('languageChanged', handler);
this.context.onRouterChanged((ctx) => …);                          // every navigation, incl. initial + 404
off();                                                             // optional: leave early
```

`this.context` is a component-scoped view of the application: subscriptions opened through its `on`
and `onRouterChanged` are removed automatically when the component is disposed, so nothing has to be
registered. Subscriptions opened on `Application.main`, a captured `app` or
`useApplication().application` are not tied to a component; register their unsubscribe with
`this.motif.setDisposable(off)` or call it yourself.

A listener that throws (`app.on` / `app.fire`, `onRouterChanged` included) is reported as `MJX123`
(`The 'motifjs-router-navigated' event handler threw.` for `onRouterChanged`); the other listeners
still run and the navigation completes.

## Forms

### One-way + event (explicit, preferred when validating/transforming)

```tsx
const f = reactive({ email: '', agree: false, lang: 'en', note: '' });

<form onsubmit:prevent={() => submit(f)}>
  <input type="email" value={() => f.email} oninput={(e) => f.email = e.target.value} />
  <input type="checkbox" checked={() => f.agree} onchange={(e) => f.agree = e.target.checked} />
  <select value={() => f.lang} onchange={(e) => f.lang = e.target.value}>
    <option value="en">English</option><option value="tr">Türkçe</option>
  </select>
  <textarea value={() => f.note} oninput={(e) => f.note = e.target.value} />
  <button type="submit" disabled={() => !f.agree}>Send</button>
</form>
```

### Two-way with `x-model` / `bindings.model`

```tsx
<input type="text"     x-model={() => f.email} />
<input type="checkbox" x-model={() => f.agree} />
<input type="text"     onconfig={(s) => s.bindings.model(f, 'email')} />
```
`x-model` takes a getter (or a bare member expression, `x-model={f.email}`) and compiles to
`bindings.model(getter, setter)`. The setter reads the owner again on every write (`f` in `f.email`,
`row` in `row.name`, `s.form` in `s.form.name`), so the binding keeps working when the owner object is
replaced (`s.form = {…}`) and when list rows are reordered. The write is skipped while the owner is
`null`/`undefined` (`x-model={() => s.current?.name}`) and when the member holds a function (a getter
prop is never overwritten). A value that is not a member access (`() => a + b`, `() => fn(x)`, a plain
identifier, a `function` expression, a block body with more than one statement) gets no setter and binds
one-way; `motif-explain` reports which case applies.

`model` picks the DOM property by element type: checkbox/radio → `checked` (boolean); number/range →
number (`null` when empty); other inputs, select, textarea → `value`; img/audio/video → `src` (one-way,
nothing is written back). Signatures: `model(getter, setter)`, `model(source, member)`,
`model(source, member, formatString, formatInfo?)`, `model(binding)`; `member` may be a dotted path
(`'address.city'`). `model(source, member)` keeps the `source` object it was called with, so it goes on
writing to that object after a parent replaces it; use `x-model` or the getter/setter form when the
owner can change. A single object argument is taken as an `IBaseBinding`, and a single function binds
one-way. For inputs the property choice is re-evaluated on every read/write, not once at bind time —
when `onconfig` runs, the `type` attribute may not be applied yet.
`bindings.add('checked', f, 'agree')` binds one-way to an explicit property.

`checked`, `selected`, `muted`, `indeterminate` and `srcObject` written as attributes are set as element
properties (`!!value`; `srcObject` gets the value or `null`), not as HTML attributes:
`<input type="checkbox" indeterminate={() => f.partial} />`.
`type="number"`/`"range"` write **numbers** back (an emptied number input writes `null`); every other
`value`-bound input writes the raw string, date types included (`'2024-01-31'`).

Element methods written as JSX props call the method instead of writing an attribute:
`<form requestSubmit={() => f.send}>`, `<input focus={() => f.editing} />` (only strict `false` →
`blur()`; `undefined`/`null`/`0`/`''` call `focus()`, so a field starting `undefined` focuses on mount —
start it at `false`),
`<input setSelectionRange={[0, 5]} />`, `<dialog showModal={true} />`; `reportValidity`, `checkValidity`, `select`, `reset`, `submit`
belong to the same group (full list in jsx-and-reactivity.md).

### Validation pattern

```tsx
const f = reactive({
  email: '',
  get valid() { return /.+@.+\..+/.test(f.email); },
});
<small class="error" x-wait={() => f.valid}>Invalid e-mail</small>
<button disabled={() => !f.valid}>Save</button>
```

`e.target` is typed loosely; cast when `strict` complains: `(e.target as HTMLInputElement).value`.
