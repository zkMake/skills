# TanStack Query integration

The router knows **when** to fetch; Query decides **what is cached and how fresh it is**. Let each do only its job.

## Wiring

```tsx
const queryClient = new QueryClient()

const router = createRouter({
  routeTree,
  context: { queryClient },
  defaultPreload: 'intent',
  defaultPreloadStaleTime: 0,   // hand freshness to Query
  Wrap: ({ children }) => (
    <QueryClientProvider client={queryClient}>{children}</QueryClientProvider>
  ),
})
```

The **same** `queryClient` instance must reach both the router context and the provider. `defaultPreloadStaleTime: 0` disables the router's own freshness logic so two caches never disagree; the router still deduplicates in-flight loader work.

Type it with `createRootRouteWithContext<{ queryClient: QueryClient }>()()`.

## Loader + component

```ts
export const Route = createFileRoute('/dashboards/$dashboardId')({
  loader: ({ context, params }) =>
    context.queryClient.ensureQueryData(dashboardQueryOptions(params.dashboardId)),
  component: Dashboard,
})

function Dashboard() {
  const { dashboardId } = Route.useParams()
  const { data } = useSuspenseQuery(dashboardQueryOptions(dashboardId))
}
```

- The loader only **triggers** the query; the component's hook is what reads it. One request, started as early as possible.
- **Always call the hook.** Reading `useLoaderData()` instead leaves the query without an observer, so it never refetches on focus or reconnect and ignores `invalidateQueries`.
- `useSuspenseQuery` is the natural fit — every route already has its own pending and error boundaries, so `pendingComponent`/`errorComponent` handle the states for free.
- `await`ing `ensureQueryData` blocks the transition until data is ready; dropping the `await` (or using `prefetchQuery`) transitions immediately and lets the component suspend. That choice is per-route: block the primary data, stream the secondary.

## Keep loader and component in sync

The failure mode that grows with the app: the component's query options gain a parameter (a new search param, a filter) and the loader's do not. The loader then prefetches under the wrong key, the component suspends again on the right one — a waterfall plus a wasted request.

Build the options **once, in route context**, and read them from both sides:

```ts
export const Route = createFileRoute('/dashboards/$dashboardId')({
  validateSearch: dashboardSearchSchema,
  loaderDeps: ({ search: { asOf } }) => ({ asOf }),
  context: ({ params, deps }) => ({
    dashboardQueryOptions: dashboardQueryOptions(params.dashboardId, deps),
  }),
  loader: ({ context }) => context.queryClient.ensureQueryData(context.dashboardQueryOptions),
  component: Dashboard,
})

function Dashboard() {
  const { dashboardQueryOptions } = Route.useRouteContext()
  const { data } = useSuspenseQuery(dashboardQueryOptions)
}
```

Now there is one definition and one key. Because context is inherited, a child route's component can consume a parent's options the same way — and a change to the component's data needs is a change to the route file, where the prefetch is right there to update.

## Invalidation across the two systems

- Data changes → `queryClient.invalidateQueries(...)`. The observing components refetch; no navigation involved.
- Route-level state changes (auth, permissions, anything read in `beforeLoad`) → `router.invalidate()`.
- After a mutation that should block the next transition, `await` the invalidation in the mutation's `onSuccess`; leave it un-awaited to navigate immediately and let the data update underneath.

## SSR and streaming (TanStack Start)

```ts
setupRouterSsrQueryIntegration({ router, queryClient })
```

This dehydrates the query cache into the SSR payload and hydrates it on the client automatically — no manual `dehydrate`/`HydrationBoundary` wiring. Without it, do it by hand on the router:

```ts
createRouter({
  routeTree,
  context: { queryClient },
  dehydrate: () => ({ queryClientState: dehydrate(queryClient) }),
  hydrate: (dehydrated) => hydrate(queryClient, dehydrated.queryClientState),
})
```

Set a non-zero `staleTime` on the server's `QueryClient` (60s is a common choice), or the client refetches everything immediately after hydration. With `useSuspenseQuery`, streaming SSR flushes each route's markup as its data resolves.
