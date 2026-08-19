# Navigation

## Link

```tsx
<Link
  to="/posts/$postId"
  params={{ postId: post.id }}
  search={{ tab: 'comments' }}
  hash="top"
  replace={false}
  preload="intent"
  activeProps={{ className: 'font-bold' }}
  inactiveProps={{ className: 'text-muted' }}
  activeOptions={{ exact: true, includeSearch: false }}
>
  {post.title}
</Link>
```

- `to` is typed against the generated tree; params and required search params are enforced. An unknown path or a missing param is a compile error.
- `activeOptions.exact` stops a parent link from lighting up on child routes; `includeSearch` (default `true`) decides whether search params must match for the link to count as active.
- `search` accepts an **updater function**, which is how you preserve unrelated params: `search={(prev) => ({ ...prev, page: prev.page + 1 })}`.
- Children may be a render prop: `{({ isActive }) => ...}`.

## Relative navigation

```tsx
<Link from={Route.fullPath} to="../comments">Comments</Link>
<Link to="." search={(prev) => ({ ...prev, page: prev.page + 1 })}>Next</Link>
```

`from` anchors a relative `to` and types `prev` in the search updater. `"."` is the current route (the right choice inside a generic component that must not know where it is mounted); `".."` is the parent.

## useNavigate

```tsx
const navigate = useNavigate({ from: Route.fullPath })

await navigate({ to: '/posts/$postId', params: { postId: created.id }, replace: true })
navigate({ search: (prev) => ({ ...prev, page: prev.page + 1 }) })
```

Prefer `Link` for anything a user clicks — it renders a real anchor, supports middle-click and preloading, and is crawlable. Reserve `useNavigate` for post-submit and other imperative flows. `<Navigate to="..." />` navigates on mount.

Common `NavigateOptions`: `replace`, `resetScroll`, `hashScrollIntoView`, `viewTransition`, `ignoreBlocker`, `reloadDocument`.

## redirect

Throw, do not return:

```ts
beforeLoad: ({ location }) => {
  if (!isAuthenticated()) {
    throw redirect({ to: '/login', search: { redirect: location.href } })
  }
}
```

A redirect is control flow: it discards the in-flight navigation and starts a new one, so it never renders. Inside a `try/catch`, re-throw it explicitly — `if (isRedirect(error)) throw error` — or the catch block swallows the navigation.

## notFound

```ts
loader: async ({ params }) => {
  const post = await getPost(params.postId)
  if (!post) throw notFound()
  return post
}
```

- `notFoundComponent` on the route handles it; `defaultNotFoundComponent` on the router is the fallback.
- `throw notFound({ routeId: '/_authenticated' })` targets an ancestor's boundary; `rootRouteId` targets the root.
- `notFound({ data })` passes payload through to the component.
- `notFoundMode: 'fuzzy'` (default) uses the closest matching route with a `notFoundComponent`; `'root'` sends everything to the root.

## Blocking and scroll

`useBlocker({ shouldBlockFn, withResolver: true })` guards unsaved-changes flows; pass `ignoreBlocker: true` on the navigation that intentionally bypasses it. `scrollRestoration: true` on the router restores position per history entry; `resetScroll: false` on a navigation keeps the current position.
