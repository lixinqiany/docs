## Dynamic Route Matching

A **dynamic segment** in the URL path (e.g. `/users/:id`) allows multiple URLs that share the same pattern to be matched by a single route and rendered with the same component. In other words, all paths like `/users/1`, `/users/42`, or `/users/john` will be handled by the same route definition, with the dynamic value exposed as a route parameter.

Any variable that starts with `:` (a colon) is treated as a **parameter**. Once the route is matched, the parameter values are available on `route.params`. With **Vue 3** and `vue-router@4`, when you use the Composition API (including `<script setup>`), you typically access the current route via the `useRoute` composable instead of relying on `this.$route` in the Vue 3 Options API:

```typescript
import { useRoute } from 'vue-router'

const route = useRoute()

console.log(route.params.id)
```

Multiple parameters can exist in a single route definition (for example, `/users/:username/posts/:postId`).

**One important thing** to note when using routes with params is that when the user navigates from `/users/johnny` to `/users/jolyne`, the same component instance will be reused. Since both routes render the same component and match the same route record, this is more efficient than destroying the old instance and creating a new one. However, it also means that some lifecycle hooks (such as `created` or `mounted` in the Composition API) will not run again when only the route params change.

Therefore, we need to manually watch something on the `route` object for reactions to changes:
```typescript
<script setup>
import { watch } from 'vue'
import { useRoute } from 'vue-router'

const route = useRoute()

watch(
  () => route.params.id,
  (newId, oldId) => {
    // react to route changes...
  }
)
</script>
```

**Catch-all routes** can use a custom parameter regexp by placing the regexp in parentheses immediately after the parameter name. The regexp defines how that segment of the URL should be matched; only URL segments satisfying the regexp will match the route. For example, `(.*)*` can be used to match any path at any nesting level and will place the matched segments into `route.params.pathMatch` (typically as an array). In contrast, `user-:afterUser(.*)` indicates that this segment must start with `user-`, and the remaining part of the segment must satisfy the `(.*)` regexp.
```typescript
const routes = [
  // will match everything and put it under `route.params.pathMatch`
  { path: '/:pathMatch(.*)*', name: 'NotFound', component: NotFound },
  // will match anything starting with `/user-` and put it under `route.params.afterUser`
  { path: '/user-:afterUser(.*)', component: UserGeneric },
]
```

## Routes' Matching Syntax  

When defining a param like `:userId`, Vue Router implicitly uses the regex `([^/]+)` (at least one character that is not a slash `/`) to extract params from URLs. This makes it hard to distinguish `/:orderId` from `/:productName` because they share the same implicit matching pattern. The easiest solution is to add a static segment that differentiates them (for example, `/order/:orderId` and `/product/:productName`). Sometimes we do not want extra static segments, in which case we can apply different regexes directly to the params (for example, `/:orderId(\\d+)` and `/:productName`).

When we expect **repeatable params** like `/first/second/third`, we can use `*` (0 or more) and `+` (1 or more) to mark a param as repeatable (for example, `/:chapters+` matches `/one`, `/one/two`, `/one/two/three`, etc.), and `route.params.chapters` will be an array instead of a string. These modifiers can also be combined with a custom regexp by placing them after the closing parenthesis.

One more important thing is that, by default, routes are **case-insensitive** and match paths **with or without a trailing slash**. For example, a route `/users` matches `/users`, `/users/`, and even `/Users/`. This behavior can be configured with the `strict` and `sensitive` options, which can be set both at the router level and per route:
```typescript
const router = createRouter({
  history: createWebHistory(),
  routes: [
    // will match /users/posva but not:
    // - /users/posva/ because of strict: true
    // - /Users/posva because of sensitive: true
    { path: '/users/:id', sensitive: true },
    // will match /users, /Users, and /users/42 but not /users/ or /users/42/
    { path: '/users/:id?' },
  ],
  strict: true, // applies to all routes
})
```

An **optional parameter** ends with `?` and cannot be repeatable.

You can use [paths.esm.dev](https://paths.esm.dev/) to visualize and debug your routes.

When using custom regex, make sure to avoid slow patterns. For example, `.*` matches any character and can lead to serious performance issues if it is combined with a repeatable modifier `*` or `+` and followed by additional segments (for example, `/:pathMatch(.*)*/something-at-the-end`). In practice, use these "match everything" params only **at the very end of the URL**. If you need them in the middle of the path, **do not make them repeatable**.