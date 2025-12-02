## Dynamic Route Matching

A **dynamic segment** in the URL path (e.g. `/users/:id`) allows multiple URLs that share the same pattern to be matched by a single route and rendered with the same component. In other words, all paths like `/users/1`, `/users/42`, or `/users/john` will be handled by the same route definition, with the dynamic value exposed as a route parameter.

Any variable that starts with `:` (a colon) is treated as a **parameter**. Once the route is matched, the parameter values are available on `route.params`. With **Vue 3** and `vue-router@4`, when you use the Composition API (including `<script setup>`), you typically access the current route via the `useRoute` composable instead of relying on `this.$route` in the Vue 3 Options API:

```typescript
import { useRoute } from "vue-router";

const route = useRoute();

console.log(route.params.id);
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
  { path: "/:pathMatch(.*)*", name: "NotFound", component: NotFound },
  // will match anything starting with `/user-` and put it under `route.params.afterUser`
  { path: "/user-:afterUser(.*)", component: UserGeneric },
];
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
    { path: "/users/:id", sensitive: true },
    // will match /users, /Users, and /users/42 but not /users/ or /users/42/
    { path: "/users/:id?" },
  ],
  strict: true, // applies to all routes
});
```

An **optional parameter** ends with `?` and cannot be repeatable.

You can use [paths.esm.dev](https://paths.esm.dev/) to visualize and debug your routes.

When using custom regex, make sure to avoid slow patterns. For example, `.*` matches any character and can lead to serious performance issues if it is combined with a repeatable modifier `*` or `+` and followed by additional segments (for example, `/:pathMatch(.*)*/something-at-the-end`). In practice, use these "match everything" params only **at the very end of the URL**. If you need them in the middle of the path, **do not make them repeatable**.

## Named Routes

Each route can optionally be given a name, and you can pass this name to `<router-link>` instead of the raw path. And the name must be **unique**, otherwise only the last one works. This has several advantages:

- **No hardcoded URLs.** If you hardcode a URL like `<router-link :to="{ path: '/1/2/3' }">User profile</router-link>` in multiple places, you’ll have to update every occurrence manually whenever that route changes.
- **Fewer typos in URLs.** Using named routes instead of literal strings helps avoid subtle mistakes when writing paths by hand.
- **Automatic parameter encoding.** If you construct URLs like `'/root/' + keyword`, you have to remember to wrap `keyword` in `encodeURIComponent` yourself. With `{ name: 'xxx', params: { ... } }`, you simply pass an object and let Vue Router handle the encoding for you.

## Nested Routes

```plain text
/user/johnny/profile                   /user/johnny/posts
┌──────────────────┐                  ┌──────────────────┐
│ User             │                  │ User             │
│ ┌──────────────┐ │                  │ ┌──────────────┐ │
│ │ Profile      │ │  ────────────>   │ │ Posts        │ │
│ │              │ │                  │ │              │ │
│ └──────────────┘ │                  │ └──────────────┘ │
└──────────────────┘                  └──────────────────┘
```

Some applications have UIs composed of components nested several levels deep. In these cases, it’s very common for different segments of the URL to map directly onto this tree of nested components. In the example above, the `User` component is rendered in `App.vue`, and then `Posts` or `Profile` is rendered inside `User.vue`. The `<router-view>` in `App.vue` is the top-level outlet that renders the component matched by a top-level route. Likewise, any rendered component can contain its own nested `<router-view>`. The `<router-view>` in `User.vue` renders whichever child component matches the nested route defined under `User` in the route table.

```typescript
// App.vue
<template>
  <router-view />
</template>

// User.vue
<template>
  <div class="user">
    <h2>User {{ $route.params.id }}</h2>
    <router-view />
  </div>
</template>

// router index
const routes = [
  {
    path: '/user/:id',
    component: User,
    children: [
      {
        // UserProfile will be rendered inside User's <router-view>
        // when /user/:id/profile is matched
        path: 'profile',
        component: UserProfile,
      },
      {
        // UserPosts will be rendered inside User's <router-view>
        // when /user/:id/posts is matched
        path: 'posts',
        component: UserPosts,
      },
    ],
  },
]
```

As you can see, the `children` option is simply another array of routes, just like `routes` itself, so you can keep nesting views as deeply as you need. In the example above, visiting `/user/eduardo` on its own won’t render anything inside the `<router-view>` of `User.vue`. If you do want something to appear there for that URL, you can provide an empty nested path such as `{ path: '', component: UserHome }`.

```typescript
const routes = [
  {
    path: "/user/:id",
    component: User,
    // notice how only the child route has a name
    children: [{ path: "", name: "user", component: UserHome }],
  },
];
```

However, with this configuration, visiting `/user/:id` will always render the nested `UserHome` route. Sometimes you may want to display only the `User` component without any nested content. In that case, you can also give the parent route its own name, for example `user-parent`. If you first navigate using `{ name: 'user-parent', params: { id: 123 } }`, only `User` is rendered and `UserHome` is skipped. After a full page reload, though, `UserHome` will appear again Because on initial load Vue Router resolves the route by path rather than by name.

Note that a parent route does not need to define its own component; it can simply be used to group routes that share the same prefix and to apply common metadata and navigation guards.
