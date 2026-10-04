# Styling and transitions

## Classes and inline styles

```tsx
<div class="card active" />
<div class={() => ['card', state.active && 'active']} />
<div class={() => ({ card: true, active: state.active })} />
<div style={{ width: '33%' }} />
<div style={() => ({ opacity: state.visible ? 1 : 0 })} />
```

Imperative (ref-counted, reactive-aware):

```ts
this.class.add('a', 'b');
this.class.add(() => state.on ? 'on' : 'off');
this.class.remove('a');
this.class.remove('**');          // remove everything incl. reactive contributors
// `class` / class.add work on SVG elements too (<svg class="ring">, <path class={() => ...}>).
this.style({ color: 'red' }); this.style('color:red'); this.style(() => ({ opacity: s.o }));
this.attr.add({ title: 'Hi', 'aria-label': 'x' });
```

CSS can be embedded: `<style>{`.panel { padding: 1rem }`}</style>` inside `view()` (the rules are
not scoped; they apply to the whole document).
Tailwind/Bootstrap/plain CSS all work — MotifJS does not touch CSS.

## Transitions

Enter runs when a component is inserted/shown; leave runs before it is removed/hidden.

### CSS class transitions

```tsx
<div transition="fade">…</div>
```
Applies `fade-enter-from → fade-enter-active → fade-enter-to` and
`fade-leave-from → fade-leave-active → fade-leave-to`. Write the CSS yourself:

```css
.fade-enter-active, .fade-leave-active { transition: opacity .3s ease; }
.fade-enter-from, .fade-leave-to { opacity: 0; }
```

Custom names/duration via object form (`TransitionProps`):

```tsx
<div transition={{ name: 'slide', duration: { enter: 300, leave: 200 },
                   enterFromClass: 'slide-in-start', enterActiveClass: 'slide-in-active', enterToClass: 'slide-in-end',
                   leaveFromClass: 'slide-out-start', leaveActiveClass: 'slide-out-active', leaveToClass: 'slide-out-end' }} />
```
Fields: `name`, `type` ('transition' | 'animation': which end event to wait for), `css`, `duration` (`number` or `{ enter, leave }`), `enter*Class`, `leave*Class`, `appear*Class`.
`css: false` disables the class-based transition: no transition classes are added and enter/leave
finish at once. Without `name` the class prefix is `motif`
(`motif-enter-from`, …). Without `duration` the wait comes from the element's computed CSS
`transition`/`animation` duration; a zero duration finishes at once.
`appear*Class` is used only for the first entry (the first time the component enters the DOM visible;
for a component that starts hidden that is its first `motif.show()`) and falls back to `enter*Class`;
later entries always use `enter*Class`.
`transition` writes no HTML attribute. On a plain tag (`<div transition="fade">`), a class component tag
and a function component tag (applied to the returned root; the tag wins over a `transition` written on
the root inside the function) it sets the component's transition; it is applied the same way when it
arrives through a spread or inside `runover`. A getter — which is what a ternary on the tag compiles to
(`transition={state.fast ? 'quick' : 'slow'}`) — is followed for the component's lifetime, and a getter
that returns `null` or `''` clears the transition.

### WAAPI keyframes

```tsx
const fx = (s: ComponentBase) => {
  s.motif.options.transition.in({ keyframes: [{ opacity: 0, transform: 'translateX(-40px)' }, { opacity: 1, transform: 'none' }],
                                  options: { duration: 300, easing: 'ease-out', fill: 'forwards' } });
  s.motif.options.transition.out({ keyframes: [{ opacity: 1 }, { opacity: 0 }], options: 200 });
};
<div onconfig={fx}>Animated</div>
```
WAAPI keyframes take precedence over CSS class transitions when both exist.

Imperative, outside enter/leave: `motif.options.transition.run(keyframes, options?, done?)` calls
`motif.stopAnimations()` (not awaited), then `element.animate(keyframes, options)`, tracks the animation and returns
it; `done` runs on `finish` (not on cancel). Where `element.animate` is missing (jsdom) `done` runs at
once.

```ts
this.motif.options.transition.run([{ transform: 'scale(1)' }, { transform: 'scale(1.1)' }, { transform: 'scale(1)' }], 200);
```

### Enter/leave order (`mode`)

`'concurrent'` (default): a child added to a container enters at once while a sibling's leave is still running.
`'out-in'`: the added child is inserted (and enters) only after the running leaves in that container finish.
`'in-out'`: the added child is inserted and enters at once; the leaving child stays in place and plays its leave
after that enter finishes (no entering sibling with an enter transition means the leave plays immediately).

```ts
app.useTransitions({ mode: 'out-in' });                 // app-wide default, no router needed
<div transition={{ mode: 'out-in' }}>{...}</div>        // per container; sets only the mode, no animation on the container
this.motif.options.transition.mode = 'concurrent';      // per container, imperative; container beats app default
```

Applies to everything inserted through `controls.add` (when/ternary branches, list rows, manual adds) and to
`motif.show()` waiting for a sibling's `motif.hide()`. A child removed or hidden again while waiting is never
inserted; no running leave means no wait; a cancelled leave releases the waiting child. In `in-out` the
`dispose()` / `motif.hide()` promise of the leaving child resolves when its delayed leave finishes; a cancelled
enter releases the waiting leave.

### Control

```ts
this.motif.options.transition.skipNextLeave = true; await this.motif.hide();   // skip once (flag resets; a fragment root's visible children also hide without animation)
await this.motif.stopAnimations();   // finishes the running CSS class transition at once; waits for tracked WAAPI animations to finish (does not cancel them), then forgets them
await this.dispose({ skipLeaveTransition: true });
```

A route page's transitions also play on navigation (leave finishes, then the next page enters). With
`useRouter({ stack })` and no `stack.animation` they are the stack transition, and the page element carries
`data-nav-direction` while it animates (see routing.md).
