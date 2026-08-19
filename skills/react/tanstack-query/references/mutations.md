# Mutations

Mutations are **imperative** — nothing fires them but a call to `mutate`. Queries are declarative; do not try to make mutations behave like queries.

## Shape

```ts
const { mutate, isPending, variables, error } = useMutation({
  mutationFn: (todo: NewTodo) => createTodo(todo),
  onSuccess: () => queryClient.invalidateQueries({ queryKey: todoQueries.lists() }),
})
```

- `mutationFn` takes exactly **one** argument. Pass an object for multiple values: `mutate({ title, body })`.
- Prefer `mutate` with callbacks over `mutateAsync`. `mutate` cannot reject; `mutateAsync` hands you an unhandled rejection unless you `catch`. Use `mutateAsync` only when chaining dependent mutations.
- Disable the submit control on `isPending` to block double submits.

## Where callbacks belong

| Callback site | Owns | Fires |
| --- | --- | --- |
| `useMutation({ onSuccess })` | query logic — invalidation, cache writes | always, even if the component unmounted |
| `mutate(vars, { onSuccess })` | UI logic — redirect, toast, close dialog | only while the component is mounted |

Splitting this way keeps the custom mutation hook reusable across call sites.

## Return the invalidation promise

```ts
onSuccess: () => queryClient.invalidateQueries({ queryKey: todoQueries.lists() })
```

Returning the promise keeps the mutation `isPending` until the refetch lands, so the button stays disabled through the whole round trip. Omit the `return` for fire-and-forget.

## Invalidation targeting

Partial prefix matching is the default: `{ queryKey: ['todos'] }` hits `['todos','list',{...}]` and `['todos','detail',1]`.

```ts
queryClient.invalidateQueries({ queryKey: ['todos'], exact: true })
queryClient.invalidateQueries({ predicate: (q) => q.queryKey[0] === 'todos' && q.state.dataUpdatedAt < cutoff })
queryClient.invalidateQueries({ queryKey: ['todos'] }, { cancelRefetch: false }) // adopt an in-flight refetch
```

`refetchType` decides what refetches immediately: `'active'` (default — observed queries), `'inactive'`, `'all'`, `'none'` (mark stale only).

## Global automatic invalidation

Invalidating everything after every mutation is a defensible default — most apps refetch far less than they should. Do it once in the `MutationCache`:

```ts
const queryClient = new QueryClient({
  mutationCache: new MutationCache({
    onSuccess: (_data, _vars, _ctx, mutation) => {
      const invalidates = mutation.meta?.invalidates as Array<QueryKey> | undefined
      queryClient.invalidateQueries(
        invalidates
          ? { predicate: (q) => invalidates.some((key) => matchQuery({ queryKey: key }, q)) }
          : undefined,
      )
    },
  }),
})

useMutation({ mutationFn: updateLabel, meta: { invalidates: [['issues'], ['labels']] } })
```

Exclude genuinely static queries with a predicate on `staleTime: Infinity`. Layer a local `onSuccess` that *awaits* only the keys the UI must see fresh before the mutation settles.

## Direct cache writes

Use `setQueryData` when the mutation response is the new entity:

```ts
onSuccess: (updatedTodo) => {
  queryClient.setQueryData(todoQueries.detail(updatedTodo.id).queryKey, updatedTodo)
  queryClient.invalidateQueries({ queryKey: todoQueries.lists() })
}
```

Hand-updating lists means reimplementing server sorting, filtering, and pagination on the client. Invalidate lists; write only detail entries.

## Optimistic updates

**Via the UI (preferred when one place shows the result).** No cache surgery, no rollback:

```tsx
{todos.map((t) => <Row key={t.id} todo={t} />)}
{isPending && <Row todo={variables} pending />}
{isError && <RetryRow onRetry={() => mutate(variables)} />}
```

**Via the cache (when several places must reflect it).** Cancel, snapshot, patch, roll back, settle:

```ts
useMutation({
  mutationFn: updateTodo,
  onMutate: async (newTodo) => {
    await queryClient.cancelQueries({ queryKey: todoQueries.detail(newTodo.id).queryKey })
    const previous = queryClient.getQueryData(todoQueries.detail(newTodo.id).queryKey)
    queryClient.setQueryData(todoQueries.detail(newTodo.id).queryKey, newTodo)
    return { previous }
  },
  onError: (_err, newTodo, context) => {
    queryClient.setQueryData(todoQueries.detail(newTodo.id).queryKey, context?.previous)
  },
  onSettled: (_data, _err, newTodo) => {
    queryClient.invalidateQueries({ queryKey: todoQueries.detail(newTodo.id).queryKey })
  },
})
```

`cancelQueries` is mandatory — an in-flight refetch would otherwise land after the patch and overwrite it. Recent v5 releases also pass a `context` object carrying `context.client`, so the closed-over `queryClient` can be dropped; the closure form above works on every v5.

Reserve optimistic updates for mutations that rarely fail and stay on screen. Optimistically updating and then immediately redirecting or closing a dialog makes the rollback invisible.

## Concurrent mutations

Two toggles in flight against the same entity: the first one's `onSettled` invalidation refetches and reverts the second one's optimistic patch. Gate the invalidation on being the last mutation standing:

```ts
useMutation({
  mutationFn: api.toggleIsActive,
  mutationKey: ['items'],
  onMutate: async () => { /* cancel + patch as above */ },
  onSettled: () => {
    if (queryClient.isMutating({ mutationKey: ['items'] }) === 1) {
      queryClient.invalidateQueries({ queryKey: itemQueries.detail(id).queryKey })
    }
  },
})
```

The count is `1` because the settling mutation still counts itself.

## Reading mutation state elsewhere

`useMutationState({ filters: { mutationKey: ['items'], status: 'pending' }, select: (m) => m.state.variables })` surfaces in-flight variables to components that did not call `mutate`.

## Forms

Server state is the form's **initial** value; once the user types, client state owns the field.

- Set `staleTime: Infinity` on the query backing the form so a background refetch cannot stomp on typed input.
- `defaultValues` cannot be `undefined` and hooks cannot be conditional — split the form into a child component rendered only once `data` exists.
- To keep background updates alive, control the fields and derive each value: user-changed → client state, otherwise → server state. After a successful mutation, reset the client state to `undefined` and invalidate, so the server value is picked up again.
