# Loaders, context, preloading, and boundaries

## The load sequence

On every URL change the router: matches routes and parses params + search → runs `beforeLoad` **serially, parent → child** → runs eligible `loader`s **in parallel** → renders.

`beforeLoad` is serial because each level's context feeds the next. Keep it cheap: guards, redirects, and context construction — not data fetching.

## loader and loaderDeps

```ts
export const Route = createFileRoute('/posts')({
  validateSearch: postSearchSchema,
  loaderDeps: ({ search: { offset, limit } }) => ({ offset, limit }),
  loader: ({ deps, params, context, abortController, preload, cause }) =>
    fetchPosts({ ...deps, signal: abortController.signal }),
})
```

- `loaderDeps` is the cache key for search-derived data. **Include only what the loader reads.** Returning the whole `search` object reloads the route whenever any unrelated param changes.
- Forward `abortController.signal` so superseded loads are cancelled.
- `cause` is `'enter' | 'stay' | 'preload'`; `preload` is a boolean — use them to fetch less aggressively on hover.
- Read the result with `Route.useLoaderData()`, or `getRouteApi('/posts').useLoaderData()` from a distant component.

## Router cache

The router has its own SWR cache, independent of any data library.

| Option | Default | Meaning |
| --- | --- | --- |
| `staleTime` | `0` | how long loader data is fresh on navigation |
| `defaultPreloadStaleTime` | 30s | freshness for preload-triggered loads |
| `gcTime` | 5 min | retention after the route unmounts |
| `shouldReload` | — | `false` = reload only on enter or deps change; also accepts a function |
| `staleReloadMode` | `'background'` | `'blocking'` waits for fresh data instead of rendering stale |

Set `defaultPreloadStaleTime: 0` when an external cache (TanStack Query) should be the single source of freshness.

`router.invalidate()` reloads matched routes and resets error boundaries.

## Context and inheritance

```ts
// __root.tsx
export const Route = createRootRouteWithContext<{ queryClient: QueryClient; auth: AuthState }>()({
  component: RootComponent,
})

// a child route adds to it
export const Route = createFileRoute('/dashboard/$dashboardId')({
  beforeLoad: ({ params }) => ({ dashboardId: params.dashboardId }),
  loader: ({ context }) => context.queryClient.ensureQueryData(dashboardQueryOptions(context.dashboardId)),
})
```

- Initial values are passed to `createRouter({ context: { queryClient, auth } })`.
- Only `beforeLoad` augments context today; the merged type flows to every descendant automatically.
- Read it in components with `Route.useRouteContext()` — optionally with `select` for a narrow subscription.
- This is dependency injection: pass the `queryClient`, an API client, a feature-flag reader, or breadcrumb metadata, rather than importing singletons into loaders. It also makes loaders trivially testable.
- `Wrap` on the router lets you mount providers (e.g. `QueryClientProvider`) around the whole tree while keeping the same instance in context.

## Preloading

```ts
createRouter({ routeTree, defaultPreload: 'intent', defaultPreloadDelay: 50 })
```

- `'intent'` — hover/touch-start, after `defaultPreloadDelay` (50ms), cancelled if the pointer leaves. The best default.
- `'viewport'` — IntersectionObserver.
- `'render'` — as soon as the `Link` mounts. Use sparingly.
- Per-link overrides: `<Link preload="viewport" preloadDelay={200} />`, or `preload={false}` to opt out.

Preloading fetches both the lazy route chunk and the loader data, which is why it removes almost all perceived navigation latency. A preload and the click that follows share the same in-flight loader work.

## Pending and error boundaries

Every route is wrapped in its own Suspense and error boundaries.

```ts
export const Route = createFileRoute('/posts')({
  loader: () => fetchPosts(),
  pendingComponent: () => <PostsSkeleton />,
  pendingMs: 1000,      // show pending UI only if loading exceeds this
  pendingMinMs: 500,    // once shown, keep it visible at least this long
  errorComponent: ({ error, reset }) => <ErrorPanel error={error} onRetry={reset} />,
  onCatch: ({ error }) => reportError(error),
})
```

`pendingMs` avoids a flash on fast loads; `pendingMinMs` avoids a flash once the spinner is up. The pending session runs on its own timer, decoupled from when the loader finishes — do not try to reproduce it with your own `setTimeout`.

Router-wide fallbacks: `defaultPendingComponent`, `defaultErrorComponent`, `defaultPendingMs`, `defaultPendingMinMs`.

## Authenticated routes

Guard once on a pathless layout, not per route:

```ts
// routes/_authenticated.tsx
export const Route = createFileRoute('/_authenticated')({
  beforeLoad: ({ context, location }) => {
    if (!context.auth.isAuthenticated) {
      throw redirect({ to: '/login', search: { redirect: location.href } })
    }
  },
})
```

Then `_authenticated.dashboard.tsx`, `_authenticated.settings.tsx`, … inherit the guard and any context the guard adds (e.g. returning `{ user }` from `beforeLoad` makes `context.user` non-nullable in every child).

Auth state that lives in React must be pushed into the router: put it in router context, and call `router.invalidate()` when it changes so guarded routes re-evaluate. To render a login form in place instead of redirecting, return `<Login />` from the layout's `component` rather than rendering `<Outlet />`.

## Router cache vs an external cache

The router cache is per-route, has no cross-route deduplication, no persistence, no mutations, and no optimistic updates. Once the app needs any of those, hand freshness to TanStack Query and keep the router as the *trigger* — see `references/query-integration.md`.
