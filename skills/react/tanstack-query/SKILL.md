---
name: tanstack-query
description: TanStack Query (@tanstack/react-query) usage and best practices — query keys, queryOptions, staleTime, mutations, invalidation, optimistic updates, select and render performance, error handling, suspense, SSR hydration, and testing. Use when writing or reviewing code that imports @tanstack/react-query, useQuery, useMutation, useSuspenseQuery, useInfiniteQuery, queryOptions or QueryClient; when caching, refetching, or stale-data behaviour is wrong; or when the user says "react query" or "tanstack query".
---

# TanStack Query

Distilled from TkDodo's blog and the v5 docs. Targets `@tanstack/react-query` v5.

## Mental model

Query is an **async state manager**, not a data fetching library. The cache holds **server state** — data the app borrows and re-synchronises. Client state (form drafts, selections, UI toggles) lives in `useState`/Zustand/a store, never in the query cache.

Three objects explain every behaviour:

- **Query** — one cache entry, keyed by the hashed `queryKey`. Owns data, status, retries, dedupe.
- **Observer** — created per `useQuery` call. Subscribes to a Query, runs `select`, decides which renders happen.
- **QueryCache** — the in-memory map of key → Query, held by the `QueryClient`.

A Query with zero observers is **inactive**: still cached, but not refetched on focus/reconnect/invalidate, and garbage-collected after `gcTime`. This is why data must be read through a hook, not lifted out of the cache imperatively.

Queries are **declarative**. Parameters live in the `queryKey`, never in a `refetch()` argument — change the key and Query fetches.

## Defaults (v5)

| Behaviour | Default |
| --- | --- |
| `staleTime` | `0` — data is stale the moment it arrives |
| `gcTime` | 5 minutes after a query goes inactive |
| `retry` | 3 attempts, exponential backoff |
| `refetchOnMount` / `refetchOnWindowFocus` / `refetchOnReconnect` | `true`, for stale data only |
| `structuralSharing` | `true` — unchanged sub-objects keep referential identity |
| `networkMode` | `'online'` — offline queries go `fetchStatus: 'paused'` |

`status` (`pending`/`error`/`success`) answers *what data do I have*. `fetchStatus` (`fetching`/`paused`/`idle`) answers *is the queryFn running*. They are independent.

## Rules that apply to every task

1. **Set `staleTime` deliberately.** With the default `0`, every mount and every window focus refetches. Pick a real number per query (20s+ is a sane floor for dedupe); use `Infinity` when a WebSocket or a mutation owns freshness. Tune `staleTime`, not `gcTime`.
2. **Define each query as a `queryOptions()` factory**, colocated with its fetcher, not as a bare hook. One definition feeds `useQuery`, `useSuspenseQuery`, `useQueries`, `prefetchQuery`, `ensureQueryData`, `getQueryData`, `setQueryData` — and tags the key with its data type, so cache reads are typed with no generics.
3. **Structure keys generic → specific**: `['todos']` → `['todos','list',filters]` → `['todos','detail',id]`. Invalidation then targets any level.
4. **Every value the `queryFn` reads belongs in the `queryKey`.** Treat the key as the dependency array.
5. **Type the fetcher, not the hook.** `const fetchTodo = (id: number): Promise<Todo> => ...`, then let inference flow. Explicit generics on `useQuery` are type assertions in disguise.
6. **The `queryFn` must return a rejected promise on failure.** `fetch` resolves on 4xx/5xx — check `response.ok` and throw. Any `catch` that logs must re-throw.
7. **Keep server state out of `useState`.** Copying `data` into local state opts that component out of every background update. Derive instead.
8. **Prefer `invalidateQueries` over hand-writing cache updates** after a mutation. Reach for `setQueryData` only when the response already contains the exact new entity.
9. **Check `data` before `error`** when rendering, so a failed background refetch does not blank out good stale data.
10. **One `QueryClient` instance**, created outside the component tree (or in `useState`), reached via `useQueryClient()` rather than a module import.

## Reference

Load the file for the branch you are on; apply every rule in it.

| File | Load when |
| --- | --- |
| `references/queries.md` | Writing or fixing a read: keys, `queryOptions`, `select`, render performance, `enabled`/`skipToken`, dependent queries, `initialData` vs `placeholderData`, infinite queries, seeding, WebSockets |
| `references/mutations.md` | Writing or fixing a write: `useMutation`, invalidation, optimistic updates, concurrent mutations, forms |
| `references/errors.md` | Error handling, error boundaries, toasts, retries, offline/`networkMode`, status rendering |
| `references/typescript.md` | Typing queries, `select` typing, error types, runtime schema validation |
| `references/integration.md` | Suspense, SSR/hydration, prefetching, router loaders, request waterfalls |
| `references/testing.md` | Writing tests that render components using queries |

## Before finishing

Confirm every rule above, plus every rule in each reference file you loaded, is satisfied by the code you wrote — naming each rule you deliberately broke and why.
