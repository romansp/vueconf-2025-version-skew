---
title: Handle frontend version skew with Vite and Vue Router
---

# Handle frontend version skew with Vite and Vue Router

## Raman Paulau
## Lead Software Engineer at [Axure](https://axure.com)

---

# Hello! My name is Raman 👋
Let's talk about deployments 

* Our flagship product Axure RP for creating complex interactive wireframes and prototypes without coding.
* We also provide companion web application called Axure Cloud which allows to share and collaborate on your prototypes in browser.
* In 2018 we rewrote front-end on Vue 2 with Vue CLI 
* And in late 2024 we fully migrated to Vue 3 with Vite.

<!--
Hi everyone! My name is Raman Paulau. Let's talk about deployments.

I'd like to share what I find to be a very elegant solution to how we handle an issue called frontend version skew in our web applications. But before we dive into what frontend version skew is, let me setup some context.

I work for company Axure Software. You may have heard or even used our flagship product Axure RP which allows to create complex interactive prototypes and wireframes without coding. Together with RP we also provide a companion web app Axure Cloud where you can share your prototypes and collaborate with your team in browser.

We rewrote Axure Cloud front-end in 2018 on Vue 2 with Vue CLI. And in late 2024 we fully migrated to Vue 3 with Vite.
-->

---
layout: default
---

# Global error handling

### AppErrorBoundary.vue
```vue
<script lang="ts">
export default Vue.extend({
  created() {
    window.addEventListener("unhandledrejection", this.handleUnhandledRejection);
    window.onerror = this.handleWindowError;
    Vue.config.errorHandler = this.handleVueError;
  },
  // ...
});
<script>
```

### App.vue
```vue
<template>
  <AppErrorBoundary>
    <!-- ... -->
  </AppErrorBoundary>
</template>
```

<!-- So before we migrated Axure Cloud to Vite weaknesses of the app was global error handling.

At the root level of the app we used `<AppErrorBoundary />` component which would capture all unhandled promise rejections, window errors and Vue component errors.
-->

---
layout: image
image: /image.png
backgroundSize: 80%
---

<!-- 
This worked fairly well but if captured error can't be handled gracefully we would consider that app has crashed, will fallback to full-screen error message and ask user to refresh the page. 

And this isn't a great experience. 

We started looking into what causes these unknown errors and identified the reason: these were chunk load errors. These were triggered due to chunk files being missing on the server. With migration to Vite and Vue 3 we decided to find a robust solution to that.
-->

---
hide: false
---
# Frontend version skew

1. New build is deployed while currently connected clients still have old website loaded.

<v-clicks>

2. Assets from the previous deployment are deleted.
3. Client tries to access chunk via router navigation and async import. 
</v-clicks>

<div v-click>
4. Network request fails. For example with 404 Not Found or MIME type checking error.

  <div class="text-red-400 p-4">
  Refused to execute script from 'https://app.axure.cloud/app/js/project-configure-plugins.38b908f7.js' because its MIME type ("text/html') is not executable, and strict MIME type checking is enabled.
  </div>
</div>

<v-clicks>

5. Vue Router throws an error and stops navigation.
</v-clicks>

<!-- 
This problem is referred to as frontend version skew or deployment skew. It happens when new build is deployed while currently connected clients still have old website loaded.

[click] Hosting provider deletes assets from the previous deployment.

[click] Client tries to access chunk via router navigation and async import.

[click] Network request fails [click] and Vue Router throws an error and stops navigation.
-->

---

# Code-split in Vite and chunks dependency map 

```ts
// router.ts
import { createRouter } from "vue-router";

const router = createRouter({
  routes: [
    { path: "/todos", component: () => import("./TodosList.vue") },
    { path: "/todos/:id", component: () => import("./TodosDetails.vue") },
    // ...more routes
  ],
});
```

### After Vite build
```js
// index-kx55ktaM.js
const __vite__mapDeps = [
  "assets/TodosList-Bnb4jiCt.js",
  "assets/TodoDetailsView-DZFCBlJz.js",
  // ...other chunks
];
```

<!--
It's often recommended to perform a code split at route level via async imports to improve initial load performance.

When code is split into chunks Vite will generate a dependency map of all async chunks and includes this map into the main chunk.

You may notice that by default Vite adds a hash to the chunk filename as well. This is needed for cache-busting and this hash is based on the content of the chunk. If the content of the chunk changes, the hash will change as well.
-->

---
hide: true
---
# Frontend version skew

1. New deployment fully replaces previous one
<v-clicks>

2. Chunks hashes in the new bundle may be different
3. Chunks referenced by the main chunk may no longer present on the server
4. Navigation to a route with no longer existing chunk will fail and Vue Router will produce an error

</v-clicks>

<!-- 
So to quickly recap
[click] New deployment fully replaces previous one.

[click] Chunks hashes in the new bundle may be different

[click] Chunks referenced by the main chunk may no longer present on the server.

[click] Navigation to a route with no longer existing chunk will fail and router will produce an error.
-->

---
layout: center
---

# Demo

---

# How can we fix that?

1. Catch async chunk loading errors.
<v-clicks>

2. Catch router errors with `router.onError()` hook. Verify that error is chunk loading error.
3. Build full URL of attempted to navigate route. Remember to include base URL as well.
4. Hard-reload app at that URL by setting `window.location.href`.
5. Page refresh will occur and the latest version of the app will be fetched from the server.
</v-clicks>

---
layout: image
image: /12084.png
backgroundSize: 60%
---

# `vite:preloadError` to the rescue

https://github.com/vitejs/vite/pull/12084

<!-- 
So what do we have at our disposal? 

Vite will actually dispatch a special event to notify about failed chunk loading just before re-throwing the import error further.

This event is called `vite:preloadError` and it was added to Vite in 2023 and adopted in Nuxt after that.

So if you're using Nuxt then your applications are already protected from this type of front-end version skew.
-->

---

# Combine `vite:preloadError` with Vue Router

<v-click>
````md magic-move
```ts
window.addEventListener("vite:preloadError", event => {
  // chunk has failed to load
  // call event.preventDefault() to prevent the error from being thrown
});
```
```ts
const chunkErrors = new Set<Error>();

window.addEventListener("vite:preloadError", event => {
  chunkErrors.add(event.payload);
});

```
```ts
const chunkErrors = new Set<Error>();

window.addEventListener("vite:preloadError", event => {
  chunkErrors.add(event.payload);
});

router.onError(error => {
   if (chunkErrors.has(error)) {
     // just saw this error in `vite:preloadError` above
   }
});
```
```ts
const chunkErrors = new Set<Error>();

window.addEventListener("vite:preloadError", event => {
  chunkErrors.add(event.payload);
});

router.onError((error, to) => {
   if (chunkErrors.has(error)) {
     reloadAppAtPath(to);
   }
});

function reloadAppAtPath(to: RouteLocation) {
  const path = joinURL(import.meta.env.VITE_APP_BASE_SUFFIX, to.fullPath);
  window.location.href = path;
}
```

```ts
import { useRouter } from "vue-router";

export function useChunkPreloadErrorHandling() {
  const chunkErrors = new Set<Error>();

  window.addEventListener("vite:preloadError", event => {
    chunkErrors.add(event.payload);
  });

  const router = useRouter();
  router.onError((error, to) => {
    if (chunkErrors.has(error)) {
      reloadAppAtPath(to);
   }
  });

  function reloadAppAtPath(to: RouteLocation) {
    const path = joinURL(import.meta.env.VITE_APP_BASE_SUFFIX, to.fullPath);
    window.location.href = path;
  }
}
```
````
</v-click>

<!--
But how can we add the same protection if we're not using Nuxt? It's actually quite simple.

[click] We start by setting up a listener to `vite:preloadError` event. This event is dispatched on global `window` object. Calling `preventDefault()` on it will prevent the error from being re-thrown but we actually want it to be thrown so it can be caught by Router.

[click] This event's payload field holds original import Error and will save this error by adding into a Set. This will help us to identify chunk import error later.

[click] Next let's register `router.onError` hook. Inside the hook we can check if error is present in the Set. 

[click] If it is, we can reload the app at the path that was attempted to be navigated. This will force the browser to reload the app and fetch the latest code and main chunk from the server.

[click]
And this all can be neatly combined into as single composable and referenced in your App.vue file.
-->

---
layout: center
---

# Back to demo

---
layout: default
---

# Thanks!
Links
* https://paulau.dev/vueconf-2025-version-skew
* https://paulau.dev/blog/handle-version-skew-after-new-deployment-with-vite-and-vue-router
* https://github.com/romansp
* https://bsky.app/profile/paulau.dev

<div class="px-4 my-4">
<img src="./qr.png" class="size-40" />
</div>