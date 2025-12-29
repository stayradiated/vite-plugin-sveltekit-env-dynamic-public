# vite-plugin-sveltekit-env-dynamic-public

A tiny Vite plugin that makes SvelteKit components using `$env/dynamic/public` render correctly in Storybook.

Storybook *claims* to support SvelteKit’s `$env/dynamic/public`, but in practice stories will throw an error when a component imports it (in both dev and static/build modes). This plugin fixes that by intercepting the import and replacing it with a virtual module that exports a static `env` object.

## The problem

This Svelte component is enough to trigger the issue in Storybook:

```svelte
<script lang="ts">
  import { env } from '$env/dynamic/public';

  const label = env.PUBLIC_BUTTON_LABEL ?? 'Hello World';
</script>

<button>{label}</button>
```

The error you’ll see looks like:

```
can't access property "env", globalThis.__sveltekit_dev is undefined

The component failed to render properly, likely due to a configuration issue in Storybook.

@http://localhost:6006/@id/__x00__virtual:env/dynamic/public:1:20
```

## What this plugin does

* Captures imports of `"$env/dynamic/public"`
* Replaces them with a virtual module that exports:

```ts
export const env = { /* your overrides */ }
```

By default, the exported `env` is `{}`.

## Why is this needed?

SvelteKit’s `$env/dynamic/public` is implemented via runtime behavior that depends on SvelteKit globals. Storybook doesn’t provide the same runtime, so importing the module crashes at evaluation time. This plugin avoids the crash by replacing the module with a stable, static implementation for Storybook.

## Caveats

* This is a **Storybook-only workaround**. Don’t add it to your actual SvelteKit app build.
* It only targets `"$env/dynamic/public"`. If you need additional SvelteKit shims, consider opening an issue or PR.

## Install

```bash
npm i -D vite-plugin-sveltekit-env-dynamic-public
```

```bash
pnpm add -D vite-plugin-sveltekit-env-dynamic-public
```

```bash
yarn add -D vite-plugin-sveltekit-env-dynamic-public
```

## Usage (Storybook + SvelteKit)

In `.storybook/main.ts`, add the plugin only for Storybook via a the `viteFinal` method:

```ts
import type { StorybookConfig } from '@storybook/sveltekit';
import { sveltekitEnvDynamicPublic } from 'vite-plugin-sveltekit-env-dynamic-public';

const config: StorybookConfig = {
  /*
   * ...your existing storybook config...
   */

  /*
   * Inject the `sveltekitEnvDynamicPublic` vite plugin
   * to ensure that components can safely reference "$env/dynamic/public"
   */
  async viteFinal(config) {
    return {
      ...config,
      plugins: [
        sveltekitEnvDynamicPublic(),
        ...(config.plugins ?? []),
      ],
    };
  },
};

export default config;
```

### Providing static env overrides

If your components read `env.PUBLIC_*` values, you can set them in Storybook:

```ts
sveltekitEnvDynamicPublic({
  env: {
    PUBLIC_BUTTON_LABEL: 'Custom Button Label',
    PUBLIC_FEATURE_ENABLED: true,
  },
}),
```

This makes the following work in stories:

```ts
import { env } from '$env/dynamic/public';

env.PUBLIC_BUTTON_LABEL; // "Custom Button Label"
env.PUBLIC_FEATURE_ENABLED; // true
```

## Options

### `env`

Type:

```ts
Record<string, string | number | boolean | null | undefined>
```

Notes:

* Values are serialized with `JSON.stringify()`
* `undefined` keys are dropped (standard JSON behavior)

## License

MIT
