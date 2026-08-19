# Routing, file conventions, and setup

## Setup

```ts
// vite.config.ts — router plugin BEFORE the framework plugin
import { tanstackRouter } from '@tanstack/router-plugin/vite'

export default defineConfig({
  plugins: [tanstackRouter({ target: 'react', autoCodeSplitting: true }), react()],
})
```

```tsx
const router = createRouter({
  routeTree,
  context: { queryClient },
  defaultPreload: 'intent',
  defaultPreloadStaleTime: 0,        // when an external cache owns freshness
  defaultPendingComponent: Spinner,
  defaultErrorComponent: ErrorScreen,
  defaultNotFoundComponent: NotFound,
  scrollRestoration: true,
})

declare module '@tanstack/react-router' {
  interface Register { router: typeof router }
}
```

`routeTree.gen.ts` is generated — commit it, never edit it, and re-run the dev server if it looks stale.

## File naming

Routes live in `src/routes/`. Nesting is expressed by directories, by `.` in the filename, or by both — pick one style per area and stay consistent.

| Pattern | Meaning |
| --- | --- |
| `__root.tsx` | Root layout, always rendered, cannot be code-split |
| `posts.tsx` | Layout route at `/posts`, renders `<Outlet />` for children |
| `posts.index.tsx` / `posts/index.tsx` | Exact match `/posts` |
| `posts.$postId.tsx` | Dynamic segment → `params.postId` |
| `posts.{-$category}.tsx` | Optional param — matches `/posts` and `/posts/tech` |
| `files.$.tsx` | Splat — remaining path in `params._splat` |
| `_authenticated.tsx` | Pathless layout: wraps children, adds nothing to the URL |
| `_authenticated.dashboard.tsx` | Child of that pathless layout |
| `posts_.$postId.edit.tsx` | Trailing `_` un-nests from `posts.tsx` (full-screen editor, no parent layout) |
| `(marketing)/about.tsx` | Route group — organises files only, invisible in the URL |
| `-components/PostCard.tsx` | Leading `-` excludes the file from route generation; colocate helpers here |

Every route file exports `Route`:

```tsx
export const Route = createFileRoute('/posts/$postId')({
  loader: ({ params }) => fetchPost(params.postId),
  component: PostComponent,
})

function PostComponent() {
  const { postId } = Route.useParams()
  const post = Route.useLoaderData()
}
```

## Layouts vs pathless layouts vs groups

- **Layout route** (`posts.tsx`) — shares UI *and* contributes `/posts` to the URL.
- **Pathless layout** (`_authenticated.tsx`) — shares UI, `beforeLoad` guards, and context without touching the URL. This is where auth checks belong.
- **Route group** (`(app)/`) — pure file organisation; no component, no context, no URL segment.

Every layout must render `<Outlet />` or its children never appear.

## Code splitting

With `autoCodeSplitting: true` the plugin splits each route automatically. Two tiers decide what moves:

- **Critical**, stays in the main bundle: `path`, `params` parsing, `validateSearch`, `beforeLoad`, `loader`, `loaderDeps`, `context`, `staticData`, `head`.
- **Non-critical**, lazy-loaded: `component`, `pendingComponent`, `errorComponent`, `notFoundComponent`.

Without the plugin, split by hand: keep critical options in `posts.$postId.tsx` via `createFileRoute`, and move the components into `posts.$postId.lazy.tsx` via `createLazyFileRoute`. If nothing critical remains, delete the non-lazy file — a virtual route is generated to anchor the lazy one.

`__root.tsx` can never be split. Splitting loaders is possible but usually a loss: you pay a chunk fetch *before* the data fetch can start.
