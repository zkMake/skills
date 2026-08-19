# Testing

## Mock the network, not the library

Use MSW (or the framework's equivalent) so tests exercise the real `queryFn`, real retries, and real error paths. Mocking `useQuery` itself tests nothing about the cache.

## A fresh QueryClient per test

```tsx
const createWrapper = () => {
  const queryClient = new QueryClient({
    defaultOptions: {
      queries: { retry: false, gcTime: Infinity },
      mutations: { retry: false },
    },
  })
  return ({ children }: { children: React.ReactNode }) => (
    <QueryClientProvider client={queryClient}>{children}</QueryClientProvider>
  )
}
```

- **Per test, not per file.** A shared client leaks cached data between tests and produces order-dependent failures.
- **`retry: false` is mandatory.** The default 3 retries with exponential backoff turn an error-path test into a timeout.
- `gcTime: Infinity` stops garbage collection from firing mid-test; keep the client scoped to the test so nothing leaks.

## Always await

```ts
const { result } = renderHook(() => useTodos(), { wrapper: createWrapper() })
await waitFor(() => expect(result.current.isSuccess).toBe(true))
expect(result.current.data).toEqual(mockTodos)
```

Query is async on every path, including cache hits in the same tick. Assert on rendered output with `findBy*` queries in component tests rather than on hook internals where you can.

## Overriding per-query behaviour

`queryClient.setQueryDefaults(['todos'], { retry: 5 })` inside the component code stays intact while the test client's global `retry: false` covers everything else — which is the reason to put retry config in `setQueryDefaults` rather than inline on `useQuery`.

## Error paths

Override the MSW handler for the single test, assert the error state, and confirm the retry count did not silently swallow it:

```ts
server.use(http.get('/api/todos', () => HttpResponse.json({ message: 'boom' }, { status: 500 })))
await waitFor(() => expect(result.current.isError).toBe(true))
```

Silence expected `console.error` noise per test rather than globally, so real errors still surface.
