# Queries

## queryOptions is the abstraction

Not a custom hook. A custom hook only runs in components; a `queryOptions` factory runs in route loaders, event handlers, servers, and tests too, and stays open to `useSuspenseQuery`/`useQueries` at the call site.

```ts
import { queryOptions } from '@tanstack/react-query'

const fetchTodo = async (id: number): Promise<Todo> => {
  const response = await fetch(`/api/todos/${id}`)
  if (!response.ok) throw new Error(`Failed to load todo ${id}`)
  return response.json()
}

export const todoQueries = {
  all: () => ['todos'] as const,
  lists: () => [...todoQueries.all(), 'list'] as const,
  list: (filters: Filters) =>
    queryOptions({
      queryKey: [...todoQueries.lists(), filters] as const,
      queryFn: () => fetchTodos(filters),
      staleTime: 30_000,
    }),
  details: () => [...todoQueries.all(), 'detail'] as const,
  detail: (id: number) =>
    queryOptions({
      queryKey: [...todoQueries.details(), id] as const,
      queryFn: () => fetchTodo(id),
    }),
}
```

Consume it everywhere, composing extra options at the call site:

```ts
useQuery(todoQueries.detail(id))
useSuspenseQuery({ ...todoQueries.detail(id), select: (todo) => todo.title })
useQueries({ queries: ids.map((id) => todoQueries.detail(id)) })
queryClient.prefetchQuery(todoQueries.list(filters))
queryClient.getQueryData(todoQueries.detail(id).queryKey) // typed Todo | undefined
queryClient.invalidateQueries({ queryKey: todoQueries.lists() })
```

The best abstractions are not configurable — keep the factory to the shared config and spread per-call extras (`select`, `throwOnError`, `staleTime`) at the usage site. Use `infiniteQueryOptions` for infinite queries.

## Keys

- Array keys only, even for a single segment: `['todos']`.
- Generic → specific, so `['todos']` invalidates everything and `['todos','list']` only lists.
- Keys hash deterministically; object member order does not matter, but types do — `['item', '1']` and `['item', 1]` are different entries.
- Never share a key between `useQuery` and `useInfiniteQuery`; the cached shapes differ.
- Colocate keys with their queries in the feature folder, not in a global `queryKeys.ts`.

## Query function context

Read parameters from the context rather than closing over them, so the key and the fetch cannot drift:

```ts
const fetchTodos = async ({
  queryKey: [, , filters],
  signal,
}: QueryFunctionContext<ReturnType<typeof todoQueries.list>['queryKey']>) => {
  const response = await fetch(`/api/todos?${new URLSearchParams(filters)}`, { signal })
  if (!response.ok) throw new Error('Failed to load todos')
  return response.json()
}
```

Always forward `signal` to `fetch`/`axios` so Query can cancel superseded requests. Use `meta` for static per-query data (e.g. an error message for a global handler) — it does not affect the key.

## Transforming data

Ranked by preference:

1. **Backend** returns the shape the UI needs.
2. **`select`** — runs per observer, only when `data` exists, and its result is structurally shared. This is the render-optimisation lever: a component that selects `data.length` re-renders only when the length changes.
3. **In the `queryFn`** — transformed data lands in the cache, so DevTools no longer show the server payload and the original shape is unrecoverable.
4. **In the render function** — needs `useMemo` with `[query.data]` (not `[query]`) as deps.

`select` re-runs when `data` changes *or* the function identity changes. Hoist it out of the component when it closes over nothing; wrap in `useCallback` when it does; memoize with `fast-memoize` when it is expensive and shared across many observers.

```ts
const selectTitles = (todos: Array<Todo>) => todos.map((t) => t.title)

const useTodoTitles = (filters: Filters) =>
  useQuery({ ...todoQueries.list(filters), select: selectTitles })
```

A reusable factory that accepts a selector keeps inference:

```ts
const todoListOptions = <TData = Array<Todo>>(
  filters: Filters,
  select?: (data: Array<Todo>) => TData,
) => queryOptions({ queryKey: [...], queryFn: ..., select })
```

## Render optimisation

- Structural sharing already keeps unchanged sub-objects referentially stable — combined with `select`, most components re-render only when their slice changes.
- Correctness beats render count. Re-renders keep the UI truthful; only optimise a measured problem.
- `notifyOnChangeProps: 'tracked'` (the default in v5) tracks which fields the render actually read. Object rest destructuring (`const { data, ...rest } = query`) subscribes to everything and defeats it.
- `structuralSharing: false` is worth it only for very large payloads where the diff itself is the bottleneck.

## Disabling and dependent queries

`enabled: false` keeps `status: 'pending'` with `fetchStatus: 'idle'` — it does not narrow types. `skipToken` does both:

```ts
useQuery({
  queryKey: ['todos', 'detail', id],
  queryFn: id === undefined ? skipToken : () => fetchTodo(id),
})
```

Dependent queries are a request waterfall by construction. Prefer one endpoint that returns both; when a chain is unavoidable, keep it to one link.

## Seeding the cache

| Option | Level | Persisted | Refetches on mount | On refetch error |
| --- | --- | --- | --- | --- |
| `initialData` | cache | yes | only if stale per `staleTime` | keeps data, sets error |
| `placeholderData` | observer | no | always | `data` becomes `undefined` |

- `initialData` — real data you already hold, e.g. a detail entry pulled from a cached list. Pair with `initialDataUpdatedAt` set to the source query's `dataUpdatedAt`, or the entry is treated as fresh-right-now and never refetched.
- `placeholderData` — fake-it-till-you-make-it. Exposes `isPlaceholderData`. Use it whenever the placeholder is not the full detail shape.
- `placeholderData: keepPreviousData` holds the previous key's data during a key change, avoiding a hard loading state on pagination/filtering.
- **`initialData` + a non-zero `staleTime` is the usual "my queryFn never runs" bug.** Fix with `placeholderData`, or `initialDataUpdatedAt: 0`.

Other seeding routes: `queryClient.prefetchQuery(options)` before navigation; `setQueryData` per item after a list fetch (push seeding — beware those entries are inactive and get garbage-collected at `gcTime`).

## Infinite queries

```ts
const infiniteTodos = infiniteQueryOptions({
  queryKey: ['todos', 'infinite'],
  queryFn: ({ pageParam, signal }) => fetchTodoPage(pageParam, signal),
  initialPageParam: 0,
  getNextPageParam: (lastPage) => lastPage.nextCursor ?? undefined,
  maxPages: 5,
})
```

- `initialPageParam` is required in v5; `getNextPageParam` returning `undefined`/`null` sets `hasNextPage: false`.
- Data is `{ pages, pageParams }` — a linked list. A refetch re-runs the `queryFn` in a loop for **every** loaded page, so deep lists get expensive; cap with `maxPages`.
- Read `isFetchingNextPage` for the "load more" spinner, `isFetching` for the whole-list refresh.
- `getPreviousPageParam` + `fetchPreviousPage` give bi-directional lists.

## WebSockets and push updates

With a socket owning freshness, set `staleTime: Infinity` globally and let the socket drive:

```ts
socket.onmessage = (event) => {
  const { entity, id, payload } = JSON.parse(event.data)
  // coarse: let Query refetch, and only for queries someone is watching
  queryClient.invalidateQueries({ queryKey: [entity, id].filter(Boolean) })
  // fine: patch directly when messages carry the full delta
  queryClient.setQueriesData({ queryKey: [entity] }, (old) => patch(old, id, payload))
}
```

Invalidation is the default: it does nothing expensive for queries nobody is observing.

## Deriving client state from server state

Do not `useEffect` a stored selection back into range when the server list changes — derive the effective value on read:

```ts
const useSelectedUser = () => {
  const { data: users } = useQuery(userQueries.list())
  const { selectedUserId, setSelectedUserId } = useUserStore()
  const selectedId = users?.some((u) => u.id === selectedUserId) ? selectedUserId : null
  return [selectedId, setSelectedUserId] as const
}
```

The stored id survives a temporary disappearance and restores itself; the store is then only ever read through this hook.

## Guaranteeing data to a subtree

When a whole section needs a loaded entity, resolve the states once at a boundary and hand the non-nullable value down through context:

```tsx
export const CurrentUserProvider = ({ children }: { children: React.ReactNode }) => {
  const query = useQuery(userQueries.current())
  if (query.isPending) return <Skeleton />
  if (query.isError) return <ErrorMessage error={query.error} />
  return <CurrentUserContext.Provider value={query.data}>{children}</CurrentUserContext.Provider>
}

export const useCurrentUser = () => {
  const user = React.useContext(CurrentUserContext)
  if (!user) throw new Error('useCurrentUser must be used inside CurrentUserProvider')
  return user
}
```
