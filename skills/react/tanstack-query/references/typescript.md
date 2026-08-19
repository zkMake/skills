# TypeScript

## Inference over generics

`useQuery` has four generics — `TQueryFnData`, `TError`, `TData`, `TQueryKey` — and TypeScript has no partial application: supply one and you supply all four. Supply none.

A generic earns its place only when it appears **at least twice**. `useQuery<Todo>()` appears once, in the return position, which makes it an unchecked assertion wearing generic clothing. Type the fetcher instead:

```ts
const fetchTodo = async (id: number): Promise<Todo> => {
  const response = await fetch(`/api/todos/${id}`)
  if (!response.ok) throw new Error(`HTTP ${response.status}`)
  return response.json()
}

const query = useQuery({ queryKey: ['todos', 'detail', id], queryFn: () => fetchTodo(id) })
// query.data: Todo | undefined
```

`select` then narrows `TData` on its own — the classic "I added `select` and everything broke" is the partial-generic trap.

## Typed cache access

`queryOptions()` tags the `queryKey` with a `DataTag`, so client methods infer without generics:

```ts
const options = queryOptions({ queryKey: ['todos', 'detail', id], queryFn: () => fetchTodo(id) })

queryClient.getQueryData(options.queryKey)        // Todo | undefined
queryClient.setQueryData(options.queryKey, (old) => old && { ...old, done: true }) // old: Todo | undefined
```

Passing a raw array key gives `unknown`. This is the strongest single argument for routing every key through `queryOptions`.

## Narrowing

Keep the query object intact rather than destructuring at the top; status checks then narrow `data`:

```ts
const todos = useQuery(todoQueries.list(filters))
if (todos.isSuccess) {
  todos.data // Todo[], not Todo[] | undefined
}
```

`enabled: false` does **not** narrow — the compiler cannot know the guard held. Use `skipToken` in the `queryFn` position, which narrows and disables in one move.

## Error type

`error` is `Error` by default. Register a global error type once instead of casting at every call site:

```ts
declare module '@tanstack/react-query' {
  interface Register {
    defaultError: AxiosError
  }
}
```

Without registration, narrow with `instanceof` rather than asserting.

## Runtime validation

Types describe the response you *hope* for; the network is a trust boundary. Parse at the edge and the mismatch surfaces as an ordinary query error, at the source, instead of as a crash three components away:

```ts
const todoSchema = z.object({ id: z.number(), name: z.string(), done: z.boolean() })

const fetchTodo = async (id: number) => {
  const response = await fetch(`/api/todos/${id}`)
  if (!response.ok) throw new Error(`HTTP ${response.status}`)
  return todoSchema.parse(await response.json())
}
```

Inference flows out of `parse`, so no annotation is needed. Parsing costs runtime — apply it to endpoints whose contract you do not control, not to every call. In a monorepo, prefer end-to-end typing (tRPC, a shared schema package) over per-endpoint parsing.

## Infinite queries

`infiniteQueryOptions` plus `initialPageParam` types `pageParam` for you; without them it falls back to `any`.
