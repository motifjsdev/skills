# Thinking in Motif

This file is about design, not API. Code that uses every MotifJS API correctly can still be
structured like a re-rendering framework: services carried down as props, callbacks carried up,
a phase flag that swaps whole subtrees, a disposed check after every `await`. Such code compiles,
lints clean and passes tests, and it is still foreign to the runtime it runs on. Read this before
deciding the file layout of a feature.

## 1. A component is an object

A component is a long-lived object that owns real DOM nodes and has an identity. This is true for
all three forms: a class, a function returning JSX and an Options API object all become the same
kind of object. The form is a writing choice, not a different model.

`view()` hands the component its template once. From then on the template is in the component's
hands: the reactive bindings and the list slots are wired, and everything that happens later is
reactivity writing into existing nodes or `controls` inserting and removing children. There is no
render cycle to keep pure, no memoisation to protect and no "runs again" to design around.
Computing something inside `view()` is allowed; it is just never necessary, because the rules live
in the model and in getters.

## 2. Model and view are separate files

The default layout of a feature has two layers, and a third when the page earns it.

- **Domain services** hold reactive state and the operations on it: `AuthService`, `ArticlesService`,
  `CommentsService`. They are plain classes registered in DI as `singleton` and they know nothing
  about components. Everything that fetches, validates, caches, retries or decides lives here, split
  by domain the way a .NET or XAML application splits its business layer.
- **Views** are components. A view binds to reactive objects and calls operations. It holds only UI
  state that belongs to the screen itself (an open panel, a draft, a drag target). It carries no
  `fetch`, no `try/catch` around business work and no business rule.
- **A page model** is added when a page's own state outgrows a handful of fields or when it
  coordinates several services. It is a class registered as `scoped`: one instance per navigation,
  shared by the page and every component under it (`references/di.md`). Its methods are the page's
  operations; its reactive `state` is what the page binds to.

Write the model first, then the view that binds to it.

This split is natural in MotifJS for a reason that does not exist in re-rendering frameworks: the
view's bindings are wired once to objects, so the thing a view binds to must be a persistent object
anyway. Once the model is that object, "rebuild the view or fill it in place" becomes a free choice.
The view can be torn down and built again while the model and its data stay where they are.

```text
src/
  services/
    auth.ts             AuthService: session state + login/logout (singleton)
    articles.ts         ArticlesService: article operations (singleton)
    comments.ts         CommentsService: comment operations (singleton)
  pages/article/
    article-page.model.ts   ArticlePageModel: this page's state + operations (scoped)
    ArticlePage.tsx         the view (class component)
    CommentCard.tsx         a part used by the view (function component)
  parts/
    FavoriteButton.tsx      reusable part (function component)
```

## 3. Communication is chosen by scope

Each way of passing something has a scope it belongs to. Picking the tool by habit instead of by
scope is what produces three levels of `auth={this.auth}`.

| What is passed | Tool | Example |
|---|---|---|
| What belongs to the parent/child relation: which item this instance represents, which variant it shows, which slot content it hosts | prop | `<FavoriteButton article={item} variant="compact" />` |
| An app-wide concern the parent should not care about: session, API, settings, the page model | DI, resolved by the component that needs it | `auth = this.getService<AuthService>(AuthService)!` or `FromService(AuthService)` |
| Live data read and written by several components | the same reactive object, handed once | `article` shared by the banner, the bottom meta and every preview |
| A notification between parts far apart in the tree | `this.context.fire` / `this.context.on` | `context.fire('article:deleted', slug)` |
| A choice made inside a small reusable part | callback prop | `<TagInput onAdd={(tag) => model.addTag(tag)} />` |

Two consequences follow:

- Nothing app-wide travels through props. A `FavoriteButton` that needs the session and the article
  operations resolves both from DI; only `article` and `variant` arrive as props.
- Business operations are called on the model, not reported upward. A comment card that deletes a
  comment calls `model.removeComment(comment)`; it does not call `props.onRemoveComment(comment)` so
  that the page can call the service. A callback prop is for a part that is reused in many places and
  has no idea what its choice means.

Public methods on a component belong to library components, where consumers need an outward
contract. Inside an application a component exposes nothing; whoever needs it to change writes the
state it is bound to. That is both less code and safer.

## 4. Identity drives the data model

Rows in a list follow the item object, and every binding follows the object it read. The data model
is therefore designed around identity from the start:

- One reactive object per entity for the life of the screen. The article the banner shows, the
  favorite button flips and the preview lists is one object. When the server answers, its fields are
  written into that object (`Object.assign(state.article, data.article)` or field by field); the
  object is not replaced.
- Lists change through the array: `push`, `splice`, `unshift`, assignment to one index. A copied array
  (`[...items, x]`) is a new array and rebuilds every row.
- Replacing an object or an array is allowed and sometimes right: it means "everything bound to this is
  rebuilt". It is a decision, not the default way to update.
- A reactive object holds plain data. Functions and `RegExp` instances stay outside it.

## 5. Rebuild or fill in place

Both are correct, each in its place, and the choice is made per page when the feature is designed.

- **The same thing is changing state**: the profile's data arrives late, the feed moves to page 2, the
  user signs in. The tree is built once with every part present and the parts are shown or hidden with
  `x-display` (hidden, still built) or `x-wait` (not built until first shown, then kept). The data
  fills existing objects.
- **It is a different thing**: another article, a tree whose shape depends on the data that arrives,
  a page whose content depends on the route parameter. Nothing of the old instance should survive, so
  it is rebuilt: a ternary or `&&` branch, a `Frame`, or a route without `keepAlive`.

A page that keeps its instance across parameter changes is declared `keepAlive: true` and reads the
parameters reactively or through a service; a page that should start fresh per parameter is left to
be rebuilt. Neither is a workaround.

## 6. Component boundaries

Components are not split by size and never down to atoms. A `<div>` or a `<label>` is not a component.
A part becomes its own component when:

- it has state and a lifetime of its own (a user input with its validation, an editor panel),
- it is used in more than one place (a label used by five forms is a component; the one used once is
  markup inside its form),
- it is a list row.

Three lines or three thousand lines are both fine. Pages and views bound to a model are classes
(lifecycle hooks and `getService` are at hand); small parts that take props and draw are functions;
the Options API carries object-based code that already has `data` and methods.

## 7. `controls` and the imperative side

Everything bound to a model is written in JSX. `this.controls.add`, `insert`, `move` and `remove`
are the tool for content that has no state to bind to: a list-looking area whose entries are not
backed by a model, nodes produced by a third-party library, a panel inserted because a plug-in
asked for it. Inserting that way is first-class and immediate; it is not a smell.

## 8. Lifetime is yours

A component is disposed when its parent disposes it, and everything it created through JSX,
`this.bindings` and `this.context.on` goes with it. What you opened yourself (timers, sockets,
observers, `effect` stops, third-party instances) is registered with `this.motif.setDisposable` at
the place you opened it. Work that finishes after an `await` goes through `this.using(promise, cb)`
or `await this.doWork(promise)`, or lives in a scoped model that is bound to its own navigation. A
hand-written `if (this.isDisposed) return` after every `await` is the sign that the work is in the
wrong layer.

## 9. One page, written twice

The page shows an article with its comments; the visitor can delete a comment they wrote. Both
versions use only valid MotifJS. The first is structured by habit, the second by the rules above.

### By habit

```tsx
export class ArticlePage extends Component<HTMLDivElement> {
  private auth = this.getService<AuthService>(AuthService)!;
  private articles = this.getService<ArticlesService>(ArticlesService)!;
  private comments = this.getService<CommentsService>(CommentsService)!;
  private slug = String(useNavigation().params.slug ?? '');

  state = reactive({
    phase: 'loading' as 'loading' | 'ready' | 'error',
    article: null as Article | null,
    comments: [] as Comment[],
  });

  override onConfig() {
    void this.load();
  }

  private async load() {
    try {
      const data = await this.articles.get(this.slug);
      if (this.isDisposed) return;
      this.state.article = data.article;
      this.state.phase = 'ready';
      const list = await this.comments.list(this.slug);
      if (this.isDisposed) return;
      this.state.comments = list;
    } catch {
      if (!this.isDisposed) this.state.phase = 'error';
    }
  }

  private async removeComment(comment: Comment) {
    await this.comments.remove(this.slug, comment.id);
    if (this.isDisposed) return;
    this.state.comments.splice(this.state.comments.indexOf(comment), 1);
  }

  override view() {
    return (
      <div class="article-page">
        {this.state.phase === 'loading' ? (
          <p>Loading article...</p>
        ) : this.state.phase === 'error' ? (
          <p>This article could not be loaded.</p>
        ) : (
          <ArticleView
            article={this.state.article!}
            comments={this.state.comments}
            auth={this.auth}
            articles={this.articles}
            onRemoveComment={(c: Comment) => void this.removeComment(c)}
          />
        )}
      </div>
    );
  }
}

function ArticleView(props: {
  article: Article;
  comments: Comment[];
  auth: AuthService;
  articles: ArticlesService;
  onRemoveComment: (c: Comment) => void;
}) {
  return (
    <div>
      <h1>{props.article.title}</h1>
      <FavoriteButton article={props.article} auth={props.auth} articles={props.articles} />
      {props.comments.map((c) => (
        <CommentCard key={c.id} comment={c} auth={props.auth} onRemoveComment={props.onRemoveComment} />
      ))}
    </div>
  );
}
```

What is foreign here:

1. The view class is the business layer. Loading, error handling and comment removal live in the
   page; the services are only called from here.
2. `auth` and `articles` travel through `ArticleView` to `FavoriteButton` and `CommentCard`. The page
   carries what those parts should resolve themselves.
3. `CommentCard` reports a business operation to the page through `onRemoveComment`. The operation
   belongs to a model that the card can call.
4. The ternary rebuilds the whole article subtree when `phase` changes and takes `article` and
   `comments` as they are at that moment. The later `this.state.comments = list` replaces the array the
   subtree was bound to; nothing below notices.
5. `article: null` forces `!` and `?.` into every binding. An entity object that exists from the start
   removes that.
6. Every `await` is followed by a manual `isDisposed` check. The component has `using` and `doWork`,
   and a scoped model needs neither.

### In Motif

`article-page.model.ts`, the page's state and operations, one instance per navigation:

```ts
import { Injectable, inject, reactive } from '@motifx/core';
import { ArticlesService } from '../../services/articles';
import { CommentsService } from '../../services/comments';
import type { Article, Comment } from '../../services/types';

const emptyArticle = (): Article => ({
  slug: '', title: '', description: '', body: '', tagList: [],
  createdAt: '', updatedAt: '', favorited: false, favoritesCount: 0,
  author: { username: '', bio: null, image: null, following: false },
});

@Injectable({ lifetime: 'scoped' })
export class ArticlePageModel {
  private articles = inject(ArticlesService);
  private comments = inject(CommentsService);

  state = reactive({
    phase: 'loading' as 'loading' | 'ready' | 'error',
    article: emptyArticle(),
    comments: [] as Comment[],
    draft: '',
    posting: false,
  });

  async load(slug: string) {
    try {
      const data = await this.articles.get(slug);
      Object.assign(this.state.article, data.article);
      this.state.phase = 'ready';
      this.state.comments = (await this.comments.list(slug)).comments;
    } catch {
      this.state.phase = 'error';
    }
  }

  async postComment() {
    const body = this.state.draft.trim();
    if (!body || this.state.posting) return;
    this.state.posting = true;
    try {
      const comment = await this.comments.add(this.state.article.slug, body);
      this.state.comments.unshift(comment);
      this.state.draft = '';
    } finally {
      this.state.posting = false;
    }
  }

  async removeComment(comment: Comment) {
    await this.comments.remove(this.state.article.slug, comment.id);
    const index = this.state.comments.indexOf(comment);
    if (index >= 0) this.state.comments.splice(index, 1);
  }
}
```

`ArticlePage.tsx`, the view, built once with every part present:

```tsx
import { Component, useNavigation } from '@motifx/core';
import { FavoriteButton } from '../../parts/FavoriteButton';
import { renderMarkdown } from '../../lib/markdown';
import { ArticlePageModel } from './article-page.model';
import { CommentCard } from './CommentCard';

export class ArticlePage extends Component<HTMLDivElement> {
  model = this.getService<ArticlePageModel>(ArticlePageModel)!;

  override onConfig() {
    void this.model.load(String(useNavigation().params.slug ?? ''));
  }

  override view() {
    const s = this.model.state;
    const article = s.article;
    return (
      <div class="article-page">
        <p x-display={() => s.phase === 'loading'}>Loading article...</p>
        <p x-display={() => s.phase === 'error'}>This article could not be loaded.</p>
        <div x-display={() => s.phase === 'ready'}>
          <h1>{() => article.title}</h1>
          <FavoriteButton article={article} variant="full" />
          <div x-html={() => renderMarkdown(article.body)}></div>
          <form onsubmit:prevent={() => void this.model.postComment()}>
            <textarea x-model={() => s.draft}></textarea>
            <button type="submit" disabled={() => s.posting}>Post Comment</button>
          </form>
          {s.comments.map((c) => (
            <CommentCard key={c.id} comment={c} />
          ))}
        </div>
      </div>
    );
  }
}
```

`CommentCard.tsx`, a part that resolves what it needs and calls the model:

```tsx
import { FromService } from '@motifx/core';
import { AuthService } from '../../services/auth';
import type { Comment } from '../../services/types';
import { ArticlePageModel } from './article-page.model';

export function CommentCard(props: { comment: Comment }) {
  const auth = FromService(AuthService)!;
  const model = FromService(ArticlePageModel)!;
  const c = props.comment;
  return (
    <div class="card">
      <p class="card-text">{c.body}</p>
      <span class="comment-author">{c.author.username}</span>
      <i
        class="ion-trash-a"
        x-display={() => auth.state.user?.username === c.author.username}
        onclick={() => void model.removeComment(c)}
      ></i>
    </div>
  );
}
```

`FavoriteButton.tsx`, a reusable part: the article and the variant are props, the session and the
operations are not.

```tsx
import { FromService, type ComponentBase } from '@motifx/core';
import { ArticlesService } from '../services/articles';
import { AuthService } from '../services/auth';
import type { Article } from '../services/types';

export function FavoriteButton(props: { article: Article; variant: 'full' | 'compact' }) {
  const auth = FromService(AuthService)!;
  const articles = FromService(ArticlesService)!;
  const article = props.article;
  return (
    <button
      type="button"
      class={() => 'btn btn-sm ' + (article.favorited ? 'btn-primary' : 'btn-outline-primary')}
      onclick={(sender: ComponentBase, _e: Event) => {
        if (!auth.isAuthenticated) sender.context.navigate('/login');
        else void articles.toggleFavorite(article);
      }}
    >
      {() => (article.favorited ? 'Unfavorite' : 'Favorite')}
      <span x-display={() => props.variant === 'full'}> Article</span> ({() => article.favoritesCount})
    </button>
  );
}
```

What changed:

- The page has no business code. The model owns loading, posting and removal; the services own the
  HTTP work. The view binds and calls.
- Nothing app-wide is a prop. `CommentCard` and `FavoriteButton` resolve the session, the operations
  and the page model themselves; the page passes only the item each part represents.
- The tree is built once. The three phases are `x-display` siblings; `article` is one object that
  exists from the first paint and is filled when the server answers, so every binding under it stays
  live and no `!` is needed.
- No `isDisposed` checks. The model is scoped to its navigation; a response that arrives after the
  visitor left writes into that navigation's instance, never into the next page.
- The comment list is replaced once per load (a new thread is a different thing) and patched with
  `unshift`/`splice` afterwards.

## Before writing a feature

1. Which entities are on this screen? One reactive object each. Which service owns them?
2. Does this page need a model of its own, or does it bind to domain services directly?
3. For every child part: what does it represent (prop), how does it look (prop), what does it need from
   the application (DI)?
4. Which parts survive a hide (`x-display` / `x-wait`) and which are a different thing when the data
   changes (rebuild)?
5. Which resources will this component open, and where is the `setDisposable` for each?
