# Errors, status, and offline

## Rendering order: data first

```tsx
if (todos.data) return <TodoList todos={todos.data} />
if (todos.error) return <ErrorMessage error={todos.error} />
return <Skeleton />
```

Checking `isPending` → `isError` → `data` looks tidier but blanks the screen whenever a *background* refetch fails, even though good stale data is sitting right there. A query can hold `status: 'error'` and populated `data` simultaneously — that is stale-while-revalidate working as intended.

Surface the background failure without unmounting: a toast, or an inline banner driven by `isError && data !== undefined`.

`isLoading` is `isPending && isFetching` — the first-ever load. `isPending` alone means "no data yet", which includes a disabled query that has never run.

## The queryFn must reject

```ts
queryFn: async () => {
  const response = await fetch(`/api/todos`)
  if (!response.ok) throw new Error(`HTTP ${response.status}`)
  return response.json()
}
```

`fetch` resolves on 4xx/5xx. axios and ky reject by default. Any `catch` that only logs must re-throw, or the query lands in `success` with `undefined` data.

## Error boundaries

`throwOnError` re-throws into the nearest boundary instead of returning an `error`:

```ts
useQuery({ ...todoQueries.list(filters), throwOnError: (error) => error.response?.status >= 500 })
```

The predicate form is the useful one: 5xx goes to the boundary, 4xx stays inline for the component to render. Query only throws when there is no data to show, so boundaries never replace a working screen because of a failed refetch.

Reset from the boundary with `useQueryErrorResetBoundary` + `react-error-boundary`:

```tsx
const { reset } = useQueryErrorResetBoundary()
<ErrorBoundary onReset={reset} fallbackRender={({ resetErrorBoundary }) => (
  <button onClick={resetErrorBoundary}>Retry</button>
)}>
```

## Global handling

Per-query `onSuccess`/`onError`/`onSettled` were removed from `useQuery` in v5 by design — they fired once per component, so a hook used twice produced two toasts, and they skipped on cache hits. Handle it once in the `QueryCache`:

```ts
const queryClient = new QueryClient({
  queryCache: new QueryCache({
    onError: (error, query) => {
      if (query.state.data !== undefined) {
        toast.error(`${query.meta?.errorMessage ?? 'Something went wrong'}: ${error.message}`)
      }
    },
  }),
})
```

The `data !== undefined` guard limits toasts to background failures — a first-load failure is already visible in the UI. `meta` carries the per-query message without a per-query callback.

For per-component side effects, use `useEffect` on the stable `data`/`error` references.

## Retries

3 attempts with exponential backoff by default, so an error surfaces several seconds after the request actually failed. Turn it off for mutations you want to fail fast, and always off in tests. `retry: (failureCount, error) => error.status !== 404` skips retries for errors that cannot succeed.

## Offline and networkMode

`networkMode` decides what happens with no connection:

- `'online'` (default) — the query goes `fetchStatus: 'paused'` and fires when the connection returns. `status` stays whatever it was.
- `'always'` — ignores connectivity entirely. Correct when the `queryFn` is not a network call (async state manager use).
- `'offlineFirst'` — fires once, then pauses retries. Correct behind a service worker or HTTP cache that may answer offline.

Render `fetchStatus === 'paused'` as "waiting for network", not as loading. `onlineManager.setOnline()` lets a React Native / custom connectivity source drive it.
