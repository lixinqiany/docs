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

## Programmatic Navigation

To navigate programmatically, you first obtain the `router` instance via `useRouter()`. Calling `router.push()` moves to a different URL and also pushes a new entry onto the history stack, so the user can click the browser’s Back button to return to the previous page. The declarative equivalent is `<router-link :to="...">`, which simply calls `router.push()` under the hood. One important subtlety is that when you provide a `path`, any `params` are ignored (unlike `query`), so you must either navigate by route `name` or construct the full path yourself with all required parameters.

```ts
// literal string path
router.push("/users/eduardo");

// object with path
router.push({ path: "/users/eduardo" });

// named route with params to let the router build the url
router.push({ name: "user", params: { username: "eduardo" } });

// with query, resulting in /register?plan=private
router.push({ path: "/register", query: { plan: "private" } });

// with hash, resulting in /about#team
router.push({ path: "/about", hash: "#team" });
```

When specifying `params`, make sure each value is a string or number (or an array of these for repeatable params); any other type (such as objects or booleans) will be automatically stringified. For optional params, you can pass an empty string (`""`) or `null` to remove that param from the URL. `router.push` and all the other navigation methods return a Promise, which lets you wait for navigation to finish and check whether it succeeded or failed.

`router.replace` works like `router.push()`, with one key difference: instead of adding a new entry to the history stack, it replaces the current entry. The declarative equivalent is `<router-link :to="..." replace>`.

To move through the history stack, you can call `router.back()` to go back one step without using the browser’s Back button, or `router.forward()` to go forward one step. More generally, `router.go(n)` lets you jump `n` steps: a positive `n` behaves like calling `forward` multiple times, and a negative `n` behaves like calling `back` multiple times.

## Named Views

Sometimes you need to display multiple views **at the same time** instead of nesting them, e.g. creating a layout with a sidebar view and a main view. This is where named views come in handy. Instead of having one single outlet in your view, you can have multiple and give each of them a name. A router-view without a name will be given default as its name.

```typescript
<router-view class="view left-sidebar" name="LeftSidebar" />
<router-view class="view main-content" />
<router-view class="view right-sidebar" name="RightSidebar" />
```

A view is rendered by using a component, therefore multiple views require multiple components for the same route. Make sure to use the components (with an s) option:

```typescript
const router = createRouter({
  history: createWebHashHistory(),
  routes: [
    {
      path: "/",
      components: {
        default: Home,
        // short for LeftSidebar: LeftSidebar
        LeftSidebar,
        // they match the `name` attribute on `<router-view>`
        RightSidebar,
      },
    },
  ],
});
```

## Redirect and Alias

Redirecting is also done in the routes configuration. For example, redirect from `/home` to `/` via `const routes = [{ path: '/home', redirect: '/' }]` or another named route `const routes = [{ path: '/home', redirect: { name: 'homepage' } }]`. You even can use a function for dynamic routing:

```ts
const routes = [
  {
    // /search/screens -> /search?q=screens
    path: "/search/:searchText",
    redirect: (to) => {
      // the function receives the target route as the argument
      // we return a redirect path/location here.
      return { path: "/search", query: { q: to.params.searchText } };
    },
  },
  {
    path: "/search",
    // ...
  },
];
```

It's also possible to redirect to a relative location:

```ts
const routes = [
  {
    // will always redirect /users/123/posts to /users/123/profile
    path: "/users/:id/posts",
    redirect: (to) => {
      // the function receives the target route as the argument
      return to.path.replace(/posts$/, "profile");
    },
  },
];
```

A redirect means when the user visits `/home`, the URL will be replaced by` /`, and then matched as `/`. But what is an alias? An alias of `/` as `/home` means when the user visits` /home`, the URL remains `/home`, but it will be matched as if the user is visiting `/`.The above can be expressed in the route configuration as `const routes = [{ path: '/', component: Homepage, alias: '/home' }]`.

## Passing Props to Route Components

Using `useRoute()` in your component creates a tight coupling with the route which limits the flexibility of the component as it can only be used on certain URLs. While this is not necessarily a bad thing, we can decouple this behavior with a props option. For example, we can set `const routes = [{ path: '/user/:id', component: User, props: true }]`to configure the route to pass the id param as a prop.

```ts
<!-- User.vue -->
<script setup>
defineProps({
  id: String
})
</script>

<template>
  <div>
    User {{ id }}
  </div>
</template>
```

When props is set to true, the **`route.params`** will be set as the component props. For routes with named views, you have to define the props option for each named view.

```ts
const routes = [
  {
    path: "/user/:id",
    components: { default: User, sidebar: Sidebar },
    props: { default: true, sidebar: false },
  },
];
```

When props is an **object**, this will be set as the component props as-is. Useful for when the props are static.

```ts
const routes = [
  {
    path: "/promotion/from-newsletter",
    component: Promotion,
    props: { newsletterPopup: false },
  },
];
```

You can also create a function that returns props. This allows you to cast parameters into other types, combine static values with route-based values, etc.

```ts
const routes = [
  {
    path: "/search",
    component: SearchUser,
    props: (route) => ({ query: route.query.q }),
  },
];
```

## Different History modes

### HTML5

The HTML5 mode is created with createWebHistory() and is the recommended mode:

```ts
import { createRouter, createWebHistory } from "vue-router";

const router = createRouter({
  history: createWebHistory(),
  routes: [
    //...
  ],
});
```

When using createWebHistory(), the URL will look "normal," e.g. https://example.com/user/id.

Here comes a problem, though: Since our app is a single page client side app, without a proper server configuration, the users will get a 404 error if they access https://example.com/user/id directly in their browser. Now that's ugly. Not to worry: To fix the issue, we can setup nginx as follows:

```bash
location / {
  try_files $uri $uri/ /index.html;
}
```

### Hash Mode

The hash history mode is created with createWebHashHistory():

```ts
import { createRouter, createWebHashHistory } from "vue-router";

const router = createRouter({
  history: createWebHashHistory(),
  routes: [
    //...
  ],
});
```

It uses a hash character (#) before the actual URL that is internally passed. Because this section of the URL is never sent to the server, it doesn't require any special treatment on the server level. It does however have a bad impact in SEO. If that's a concern for you, use the HTML5 history mode. The path behind `#` won't be seen by nginx, so only `/` will be matched.

## Active links

### When are links active?

A RouterLink is considered to be active if:

- It matches the same route record (i.e. configured route) as the current location.
- It has the same values for the params as the current location.

If you're using nested routes, any links to ancestor routes will also be considered active if the relevant params match.Other route properties, such as the query, are not taken into account.

The path doesn't necessarily need to be a perfect match. For example, using an alias would still be considered a match, so long as it resolves to the same route record and params. If a route has a redirect, it won't be followed when checking whether a link is active.

### Exact active links

An exact match does not include ancestor routes. Let's imagine we have the following routes:

```ts
const routes = [
  {
    path: "/user/:username",
    component: User,
    children: [
      {
        path: "role/:roleId",
        component: Role,
      },
    ],
  },
];
```

Now consider these two links:

- `<RouterLink to="/user/erina">`
- `<RouterLink to="/user/erina/role/admin">`

If the current location path is `/user/erina/role/admin`, both links are considered **active**, so the `router-link-active` class will be applied to each of them. However, only the second link is an **exact** match for the current URL, so only that link will receive the `router-link-exact-active` class.
