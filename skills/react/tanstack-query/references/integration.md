# Suspense, SSR, prefetching, and waterfalls

## Suspense

`useSuspenseQuery` returns `data` as always-defined and delegates pending/error to React boundaries.

- Disallowed options: `enabled` (nothing to conditionally suspend on), `placeholderData` (would replace the fallback), `throwOnError` (fixed, so `data` stays guaranteed).
- Two `useSuspenseQuery` calls in one component fetch **serially** — the first suspends before the second is reached. Use `useSuspenseQueries` for parallel independent reads.
- Errors reach the boundary only when there is no data to show; a failed background refetch keeps rendering the stale screen.
- Pair with `useQueryErrorResetBoundary` so the boundary's retry actually refetches.

## Request waterfalls

Causes, in the order they usually bite:

| Cause | Fix |
| --- | --- |
| Dependent queries (`enabled: !!user?.id`) | one endpoint returning both; otherwise keep the chain to one link |
| Several `useSuspenseQuery` in one component | `useSuspenseQueries` |
| Parent query gating a child that renders another query | hoist the child's query, or prefetch it in the parent |
| Lazy-loaded component that fetches on mount | prefetch in the route loader; split code below the fetch |
| Client-fetching data the server already had | prefetch + hydrate |

Prefetching is the general lever: start the request at the earliest moment you know the key, and let the component's hook adopt the in-flight promise.

## SSR and hydration

```tsx
// server
const queryClient = new QueryClient({ defaultOptions: { queries: { staleTime: 60_000 } } })
await queryClient.prefetchQuery(postQueries.list())

return (
  <HydrationBoundary state={dehydrate(queryClient)}>
    <Posts />
  </HydrationBoundary>
)
```

- A non-zero `staleTime` is required on the server, or the client refetches everything the instant it hydrates.
- Create a **fresh** `QueryClient` per request on the server; on the browser, create it once and reuse (a module singleton or `useState`), never per render.
- Treat server components as prefetch sites only. Rendering the same data in a server component *and* in a client component reading the same key desynchronises on revalidation.
- Stream by dehydrating pending queries and skipping the `await`:

```ts
new QueryClient({
  defaultOptions: {
    dehydrate: {
      shouldDehydrateQuery: (query) =>
        defaultShouldDehydrateQuery(query) || query.state.status === 'pending',
    },
  },
})
queryClient.prefetchQuery(postQueries.list()) // no await — streams as it resolves
```

Consume with `useSuspenseQuery` so the client suspends on the streamed promise.

## Router loaders

Applies to TanStack Router, React Router, and anything else with a loader. See the `tanstack-router` skill for the full TanStack Router integration.

```ts
loader: ({ context, params }) => context.queryClient.ensureQueryData(postQueries.detail(params.postId))
```

- `ensureQueryData` — return cached data or fetch it; `await` it to block the transition until data is ready.
- `prefetchQuery` without `await` — start the request early, let the component suspend. Use this when the route should transition immediately.
- **Always call a query hook in the component too.** Loader-only data has no observer, so it never refetches on focus/reconnect and ignores invalidation. Read via `useSuspenseQuery(sameOptions)`, not `useLoaderData()`.
- The `await` is the transition lever: awaiting invalidation after a mutation blocks navigation until fresh data lands; not awaiting shows stale data instantly and updates underneath.
