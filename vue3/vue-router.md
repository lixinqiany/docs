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