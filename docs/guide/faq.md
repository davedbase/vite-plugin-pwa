---
title: FAQ | Guide
---

# FAQ

## IDE errors 'Cannot find module' (ts2307)

If your TypeScript build step or IDE complain about not being able to find modules or type definitions on imports, 
add the following to the `compilerOptions.types` array of your `tsconfig.json`:

```json
{
  "compilerOptions": {
    "types": [
      "vite-plugin-pwa/client"
    ]
  }
}
```

## Service Worker Registration Errors

You can handle Service Worker registration errors if you want to notify the user with following code on your `main.ts` 
or `main.js`:

```ts
import { registerSW } from 'virtual:pwa-register'

const updateSW = registerSW({
  onRegisterError(error) {}
})
```

and then inside `onRegisterError`, just notify the user that there was an error registering the service worker.

## Missing assets from SW precache manifest

If you find any assets are missing from the service worker's precache manifest, you should check if they exceed the
`maximumFileSizeToCacheInBytes`, the default value is **2 MiB**.

You can increase the value to your needs, for example to allow assets up to **3 MiB**:
- when using `generateSW` strategy:
```ts
workbox: {
  maximumFileSizeToCacheInBytes: 3000000
}
```
- when using `injectManifest` strategy:
```ts
injectManifest: {
  maximumFileSizeToCacheInBytes: 3000000
}
```

## `navigator / window` is `undefined`

If you are getting `navigator is undefined` or `window is undefined` errors when building your application, you have 
configured your application in an `SSR / SSG` environment.

The error could be due to using this plugin or another library not aware of `SSR / SSG`:  your code will be called on 
the client but also on the server side on build process, so when building the application your server logic will be 
invoked, and there is no `navigator / window` on the server, it is `undefined`.

### Third party libraries

If the cause of the error is a third party library that is not aware of the `SSR / SSG` environment, the way to work 
around the error is to import it with a dynamic import when `window` is defined:

```ts
if (typeof window !== 'undefined')
  import('./library-not-ssr-ssg-aware')  
```

Alternatively, if your framework supports component `onMount / onMounted` lifecycle hook, you can import the third 
party library on the callback, since the frameworks should call this lifecycle hook only on client side, you should
check your framework documentation.

### Vite PWA Virtual Module

If the cause of the error is the virtual module of this plugin, you can work around this problem following 
[SSR/SSG: Prompt for update](/guide/prompt-for-update.html#ssr-ssg) or
[SSR/SSG: Automatic reload](/guide/auto-update.html#ssr-ssg) entries.

If you are using `autoUpdate` strategy and a `router` with `isReady` support (that is, the router allow register a callback
to be called when the current component route finish loading), you can delay the service worker registration to be on the 
router callback.

For example, using `vue-router`, you can register the service worker for `autoUpdate` strategy using this code:

```ts
import type { Router } from 'vue-router'
export const registerPWA = (router: Router) => {
  router.isReady(async() => {
    const { registerSW } = await import('virtual:pwa-register')
    registerSW({ immediate: true })
  })
}
```

You can see an example for `autoUpdate` strategy on a `SSR / SSG` environment ([vite-ssg](https://github.com/antfu/vite-ssg) <outbound-link />)
on [Vitesse Template](https://github.com/antfu/vitesse/blob/main/src/modules/pwa.ts) <outbound-link />. 

If you are using `prompt` strategy, you will need to load the `ReloadPrompt` component using dynamic import with async fashion,
for example, using `vue 3`:

```vue
// src/App.vue
<script setup lang='ts'>
import { defineAsyncComponent } from 'vue'
const ClientReloadPrompt = typeof 'window' !== undefined 
  ? defineAsyncComponent(() => import('./ReloadPrompt.vue'))
  : null
</script>
<template>
  <router-view/>
  <template v-if='ClientReloadPrompt'>
    <ClientReloadPrompt />
  </template>
</template>
```

or using `svelte`:

```html
<!-- App.svelte -->
<script>
  import { onMount } from 'svelte';
  let ClientReloadPrompt;
  onMount(async () => {
    typeof 'window' !== undefined && (ClientReloadPrompt = await import('$lib/ReloadPrompt.svelte')).default)
  })
</script>
...
{#if ClientReloadPrompt}
<svelte:component this={ClientReloadPrompt}/>
{/if}
```

You can check your `SSR / SSG` environment to see if it provides some way to register components only on client side.
Following with `vite-ssg` on `Vitesse Template`, it provides `ClientOnly` functional component, that will prevent 
registering components on server side, and so you can use the original code but enclosing `ReloadPrompt` component with 
it:

```vue
// src/App.vue
<template>
  ...
  <ClientOnly>
    <ReloadPrompt />
  </ClientOnly>
</template>
```
