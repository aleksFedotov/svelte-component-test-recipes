# Svelte Component Test Recipes (WIP)

Svelte component test recipes using Vitest & Testing Library with TypeScript

---
## Table of Contents
- [Svelte Component Test Recipes (WIP)](#svelte-component-test-recipes-wip)
  - [Table of Contents](#table-of-contents)
  - [Setup](#setup)
  - [Testing component props](#testing-component-props)
    - [Get your component props type](#get-your-component-props-type)
  - [Testing component events](#testing-component-events)
  - [Testing the `bind:` directive (two-way binding)](#testing-the-bind-directive-two-way-binding)
    - [Caveats ⚠️](#caveats-️)
  - [Testing the `use:` directive (Svelte Actions)](#testing-the-use-directive-svelte-actions)
  - [Testing slots](#testing-slots)
  - [Testing the Context API](#testing-the-context-api)
  - [Testing components which use SvelteKit runtime modules (`$app/*`)](#testing-components-which-use-sveltekit-runtime-modules-app)
  - [Testing data fetching components using `msw`](#testing-data-fetching-components-using-msw)


In this repo, we'll use `vitest`, `@testing-library/svelte`, and `svelte-htm` to test Svelte components that seemed to be hard to test. Such as **two-way bindings**, **name slots**, **Context API**, ...etc.

Feel free to open an issue or send a PR to add more test recipes. 😉

## Setup

Let's install libraries in your SvelteKit project.

```bash
# Minimal setup
npm install -D vitest c8 @testing-library/svelte jsdom
# Companion libraries for Testing Library
npm install -D @testing-library/jest-dom  @testing-library/dom @testing-library/user-event @types/testing-library__jest-dom
# Test harness libraries
npm install -D svelte-htm svelte-fragment-component patch-package

```

`Vitest` can read your configuration in your existing `vite.config.(js|ts)` file, and here is my setup:

```ts
// vite.config.ts
import { sveltekit } from '@sveltejs/kit/vite';
import type { UserConfig } from 'vite';
import { configDefaults, type UserConfig as VitestConfig } from 'vitest/config';

const config: UserConfig & { test: VitestConfig['test'] } = {
  plugins: [sveltekit()],
  define: {
    // Eliminate in-source test code
    'import.meta.vitest': 'undefined'
  },
  test: {
    // jest like globals
    globals: true,
    environment: 'jsdom',
    // in-source testing
    includeSource: ['src/**/*.{js,ts,svelte}'],
    // Add @testing-library/jest-dom matchers & mocks of SvelteKit modules
    setupFiles: ['./setupTest.ts'],
    // Exclude files in c8
    coverage: {
      exclude: ['setupTest.ts']
    },
    deps: {
      // Put Svelte component here, e.g., inline: [/svelte-multiselect/, /msw/]
      inline: []
    },
    // Exclude playwright tests folder
    exclude: [...configDefaults.exclude, 'tests']
  }
};

export default config;
```

You may notice that there is a `setupTest.ts` file. We can add `@testing-library/jest-dom` matchers & mocks of SvelteKit there:

```ts
// setupTest.ts
/* eslint-disable @typescript-eslint/no-empty-function */
import matchers from '@testing-library/jest-dom/matchers';
import { expect, vi } from 'vitest';
import type { Navigation, Page } from '@sveltejs/kit';
import { readable } from 'svelte/store';
import * as environment from '$app/environment';
import * as navigation from '$app/navigation';
import * as stores from '$app/stores';

// Add custom jest matchers
expect.extend(matchers);

// Mock SvelteKit runtime module $app/environment
vi.mock('$app/environment', (): typeof environment => ({
  browser: false,
  dev: true,
  prerendering: false
}));

// Mock SvelteKit runtime module $app/navigation
vi.mock('$app/navigation', (): typeof navigation => ({
  afterNavigate: () => {},
  beforeNavigate: () => {},
  disableScrollHandling: () => {},
  goto: () => Promise.resolve(),
  invalidate: () => Promise.resolve(),
  invalidateAll: () => Promise.resolve(),
  prefetch: () => Promise.resolve(),
  prefetchRoutes: () => Promise.resolve()
}));

// Mock SvelteKit runtime module $app/stores
vi.mock('$app/stores', (): typeof stores => {
  const getStores: typeof stores.getStores = () => {
    const navigating = readable<Navigation | null>(null);
    const page = readable<Page>({
      url: new URL('http://localhost'),
      params: {},
      routeId: null,
      status: 200,
      error: null,
      data: {}
    });
    const updated = { subscribe: readable(false).subscribe, check: () => false };

    return { navigating, page, updated };
  };

  const page: typeof stores.page = {
    subscribe(fn) {
      return getStores().page.subscribe(fn);
    }
  };
  const navigating: typeof stores.navigating = {
    subscribe(fn) {
      return getStores().navigating.subscribe(fn);
    }
  };
  const updated: typeof stores.updated = {
    subscribe(fn) {
      return getStores().updated.subscribe(fn);
    },
    check: () => false
  };

  return {
    getStores,
    navigating,
    page,
    updated
  };
});

```
> The `@testing-library/jest-dom` library provides a set of custom jest matchers that you can use to extend vitest. These will make your tests more declarative and clear to read and maintain. You can also check [Common mistakes with React Testing Library #Using the wrong-assertion](https://kentcdodds.com/blog/common-mistakes-with-react-testing-library#using-the-wrong-assertion).

> SvelteKit runtime modules like `$app/stores` and `$app/navigation` are not set until SvelteKit's start function is called, which means you won't have them in a test environment because tests are isolated. In the context of unit testing, any small gaps in functionality can be resolved by simply mocking that module.

OK! The setup is ready. Let's start with a simple component test.

## Testing component props

Here's our svelte component:

`$lib/props/DefaultProps.svelte`
```svelte
<script>
  export let answer = 'a mystery';
</script>

<p>The answer is {answer}</p>
```

It's a simple component with only one prop `answer` with a default value.

Let's see how to test the default value and pass a new one:

```ts
// $lib/props/DefaultProps.test.ts
import { render, screen } from '@testing-library/svelte';
import DefaultProps from './DefaultProps.svelte';

it("doesn't pass prop", () => {
  render(DefaultProps);
  expect(screen.queryByText('The answer is a mystery')).toBeInTheDocument();
});

it('set and update prop', async () => {
  // Pass your prop to the render function
  const { component } = render(DefaultProps, { answer: 'I dunno' });

  expect(screen.queryByText('The answer is I dunno')).toBeInTheDocument();

  // Update prop using Svelte's Client-side component API
  await component.$set({ answer: 'another mystery' });
  expect(screen.queryByText('The answer is another mystery')).toBeInTheDocument();
});
```

### Get your component props type

If you're using TypeScript, a recent release of `@testing-library/svelte`
had improved props typing for `render` function:

![demo-props-test](./static/demo-props-test.gif)

Sometimes you may want to predefined your props before passing. We can use Svelte's native utility type `ComponentProps`. `ComponentProps` takes in a Svelte component type and gives you a type corresponding to the component’s props.

```ts
// $lib/props/DefaultProps.test.ts
import type { ComponentProps } from 'svelte';
import { render, screen } from '@testing-library/svelte';
import DefaultProps from './DefaultProps.svelte';

it('Pass predefined prop to the component', () => {
  const prop: ComponentProps<DefaultProps> = { answer: 'TypeScript!' };

  render(DefaultProps, prop);

  expect(screen.getByText('The answer is TypeScript!'));
});
```

> Here is an excellent post from Andrew Lester: [Typing Components in Svelte](https://www.viget.com/articles/typing-components-in-svelte/). Highly recommended.

## Testing component events

The component we're going to test has a button that'll dispatch a custom event `message` when you click on it. It's the component from [svelte.dev/tutorials/component-events](https://svelte.dev/tutorial/component-events).

`$lib/events/ComponentEvent.svelte`
```svelte
<script>
  import { createEventDispatcher } from 'svelte';

  const dispatch = createEventDispatcher();

  function sayHello() {
    dispatch('message', {
      text: 'Hello!'
    });
  }
</script>

<button on:click={sayHello}> Click to say hello </button>
```

To test component events, we use a combination of vitest utility function [`vi.fn`](https://vitest.dev/api/#vi-fn) and the Svelte client-side component API [`component.$on`](https://svelte.dev/docs#run-time-client-side-component-api-$on). We also use [`@testing-library/user-event`](https://github.com/testing-library/user-event) instead of the built-in `fireEvent` to simulate user interactions.

> `user-event` applies workarounds and mocks the UI layer to simulate user interactions like they would happen in the browser. Check [Common mistakes with React Testing Library  #Not using @testing-library/user-event](https://kentcdodds.com/blog/common-mistakes-with-react-testing-library#not-using-testing-libraryuser-event).

```ts
// $lib/events/ComponentEvent.test.ts
import { render, screen } from '@testing-library/svelte';
import ComponentEvent from './ComponentEvent.svelte';
import userEvent from '@testing-library/user-event';

it('Test ComponentEvent component', async () => {
  const user = userEvent.setup();

  const { component } = render(ComponentEvent);

  // Mock function
  let text = '';
  const mock = vi.fn((event) => (text = event.detail.text));
  component.$on('message', mock);

  const button = screen.getByRole('button');
  await user.click(button);

  expect(mock).toHaveBeenCalled();
  expect(text).toBe('Hello!');
});
```

We first create a mock function and pass it to the `component.$on`, so we can monitor it whenever the component dispatch a `message` event.

## Testing the `bind:` directive (two-way binding)

We use `Keypad.svelte` from [svelte.dev/tutorial/component-bindings](https://svelte.dev/tutorial/component-bindings):

`$lib/bindings/Keypad.svelte`
```svelte
<script lang="ts">
  import { createEventDispatcher } from 'svelte';

  export let value = '';

  const dispatch = createEventDispatcher();

  const select = (num: number) => () => (value += num);
  const clear = () => (value = '');
  const submit = () => dispatch('submit');
</script>

<div class="keypad">
  <button on:click={select(1)}>1</button>
  <button on:click={select(2)}>2</button>
  <button on:click={select(3)}>3</button>
  <button on:click={select(4)}>4</button>
  <button on:click={select(5)}>5</button>
  <button on:click={select(6)}>6</button>
  <button on:click={select(7)}>7</button>
  <button on:click={select(8)}>8</button>
  <button on:click={select(9)}>9</button>

  <button disabled={!value} on:click={clear}>clear</button>
  <button on:click={select(0)}>0</button>
  <button disabled={!value} on:click={submit}>submit</button>
</div>

<style>
  .keypad {
    display: grid;
    grid-template-columns: repeat(3, 5em);
    grid-template-rows: repeat(4, 3em);
    grid-gap: 0.5em;
  }

  button {
    margin: 0;
  }
</style>
```

There is no programmatic interface to test `bind:`, `use:`, slots, and Context API. Instead of creating a dummy svelte component (e.g. `TestHarness.svelte`) to test your component, we can use [`svelte-htm`](https://github.com/kenoxa/svelte-htm) to simplify your testing code.

---

### Caveats ⚠️

This repo use [`patch-package`](https://github.com/ds300/patch-package) to work around the issue when you pass `svelte-htm` inline component in a `render` function: [New component root property may throw errors #6584](https://github.com/sveltejs/svelte/issues/6584)

Here are the steps:

1. Change the source code in the `node_modules`:

```diff
// /node_modules/svelte/internal/index.mjs
- root: options.target || parent_component.$$.root
+ root: options.target || parent_component?.$$.root
```

2. Run `npx patch-package svelte`
3. Add `"postinstall": "patch-package"` script in your `package.json`

---

Assume we use `Keypad.svelte` in one of the `+page.svelte`:

```svelte
<Keypad bind:value={} on:submit={} />
```

To test the two-way binding of `value`, we use a writeable store as property value:

```ts
// $lib/bindings/Keypad.test.ts
import { get, writable } from 'svelte/store';
import { render, screen } from '@testing-library/svelte';
import html from 'svelte-htm';
import Keypad from './Keypad.svelte';
import userEvent from '@testing-library/user-event';

describe('Test Keypad component', async () => {
  const user = userEvent.setup();

  it('Test two-way binding', async () => {
    // Create a writable store to enable reactivity
    const pin = writable('');
    const mock = vi.fn();

    render(html`<${Keypad} bind:value=${pin} on:submit=${mock} />`);

    // Get Keypad button "1"
    const button1 = screen.getByText('1');
    await user.click(button1);
    await user.click(button1);
    expect(get(pin)).toBe('11');

    const submitButton = screen.getByText('submit');
    await user.click(submitButton);
    expect(mock).toHaveBeenCalled();

    const clearButton = screen.getByText('clear');
    await user.click(clearButton);
    expect(get(pin)).toBe('');
  });
});

```

## Testing the `use:` directive (Svelte Actions)

We use the example from [svelte.dev/tutorial/actions](https://svelte.dev/tutorial/actions):

```ts
// $lib/actions/clickOutside.ts
export function clickOutside(node: HTMLElement) {
  const handleClick = (event: Event) => {
    if (!node.contains(event.target as Node)) {
      node.dispatchEvent(new CustomEvent('outclick'));
    }
  };

  document.addEventListener('click', handleClick, true);

  return {
    destroy() {
      document.removeEventListener('click', handleClick, true);
    }
  };
}
```

Then use it in `src/routes/actions/+page.svelte`:

```svelte
<script lang="ts">
  import { clickOutside } from '$lib/actions/clickOutside';

  let showModal = true;
</script>

<button on:click={() => (showModal = true)}>Show Modal</button>
{#if showModal}
  <div class="box" use:clickOutside on:outclick={() => (showModal = false)}>Click outside me!</div>
{/if}
```

To test svelte actions, we actually test the function but not the component which use the function. Be careful that we need to use `use:action=${/** yourActionFunction *}` in `svelte-htm`'s tagged templates:

```ts
import { render, screen } from '@testing-library/svelte';
import html from 'svelte-htm';
import userEvent from '@testing-library/user-event';
import { clickOutside } from './clickOutside';

it('Test clickOuside svelte action', async () => {
  const user = userEvent.setup();
  const mock = vi.fn();

  render(html`
    <button>Outside the button</button>
    <button
      use:action=${clickOutside /** or (node) => yourAction(node, params) */}
      on:outclick=${mock}
    >
      Click outside me!
    </button>
  `);

  const button = screen.getByText('Outside the button');
  await user.click(button);
  expect(mock).toHaveBeenCalled();
});
```

## Testing slots

## Testing the Context API

## Testing components which use SvelteKit runtime modules (`$app/*`)

## Testing data fetching components using `msw`
