# Search params as state

TanStack Router treats the query string as a first-class, typed, validated state container — JSON-serialisable, not just strings. Filters, pagination, sort order, the open tab, the selected row: all belong here, because URL state survives reloads, back/forward, and being pasted into Slack.

## validateSearch

```ts
import { z } from 'zod'

const productSearchSchema = z.object({
  page: z.number().catch(1).default(1),
  filter: z.string().catch('').default(''),
  sort: z.enum(['newest', 'oldest', 'price']).catch('newest').default('newest'),
})

export const Route = createFileRoute('/shop/products/')({
  validateSearch: productSearchSchema,
})
```

- Zod v4, Valibot, ArkType and anything Standard Schema-compatible plug in directly. For Zod v3 wrap in `zodValidator` from `@tanstack/zod-adapter` so input and output types stay distinct.
- **Always give every key a default and a fallback** (`.catch()` / `fallback()`). Users edit URLs; a throwing validator turns a typo into an error boundary.
- A plain function works too: `validateSearch: (search: Record<string, unknown>): ProductSearch => ({ page: Number(search.page ?? 1) })`.
- Search validation is inherited: a child route sees its own params merged with every ancestor's.

## Reading

```ts
const { page, filter, sort } = Route.useSearch()

// outside the route file
const search = useSearch({ from: '/shop/products/' })
const routeApi = getRouteApi('/shop/products/')
const page = routeApi.useSearch({ select: (s) => s.page })
```

`select` is the render-optimisation lever: results are structurally shared, so a component selecting `s.page` does not re-render when a *sibling* route pushes an unrelated param into the URL. Use it on any search read inside a frequently-rendered component.

## Writing

Always update through an updater function so unrelated params survive:

```tsx
<Link from={Route.fullPath} search={(prev) => ({ ...prev, page: prev.page + 1 })}>Next</Link>
```

```ts
const navigate = useNavigate({ from: Route.fullPath })
navigate({ search: (prev) => ({ ...prev, filter, page: 1 }), replace: true })
```

Use `replace: true` for high-frequency updates (a search-as-you-type box) so the back button does not walk through every keystroke.

## Search middlewares

Declared on the route, applied to every link generated from it.

```ts
import { retainSearchParams, stripSearchParams } from '@tanstack/react-router'

export const Route = createRootRoute({
  validateSearch: rootSearchSchema,
  search: {
    middlewares: [
      retainSearchParams(['workspaceId', 'theme']),   // carry these across every navigation
      stripSearchParams({ page: 1, sort: 'newest' }), // omit values equal to the default
    ],
  },
})
```

`retainSearchParams(true)` retains everything. Middlewares chain in order and run on link generation, so they keep URLs short without every call site spreading `prev`.

## Serialisation

The default serialiser handles nested objects and arrays via JSON. Override with `parseSearch`/`stringifySearch` on the router (e.g. `JSON.parse`/`JSON.stringify` with a custom encoder, or `@tanstack/router` + `jsurl2`) when URLs need to be compact or must interop with an existing format.
