# nextjs-binds


**⚠️ THIS PROJECT IS IN EARLY DEVELOPMENT AND IS NOT STABLE YET ⚠️**

This is minimal compatibility layer for effector + Next.js - it only provides one special `EffectorNext` provider component, which allows to fully leverage effector's Fork API, while handling some *special* parts of Next.js SSR and SSG flow.

So far there are no plans to extend the API, e.g., towards better DX - there are already packages like [`nextjs-effector`](https://github.com/risenforces/nextjs-effector).
This package aims only at technical nuances.

## Installation

### Yarn

```bash
yarn add effector effector-react <package-name-will-be-added-later>
```

### NPM

```bash
npm install effector effector-react <package-name-will-be-added-later>
```

### PNPM

```bash
pnpm add effector effector-react <package-name-will-be-added-later>
```

## Usage

### SIDs

To serialize and transfer state of effector stores between the network boundaries all stores must have an Stable IDentifier - sid.

Sid's are added automatically via either built-in babel plugin or our expiremental SWC plugin.

#### Babel-plugin

```json
{
  "presets": ["next/babel"],
  "plugins": ["effector/babel-plugin"]
}
```

[See the docs](https://effector.dev/docs/api/effector/babel-plugin/#usage)

#### SWC Plugin

[See effector SWC plugin documentation](https://github.com/effector/swc-plugin)

### Pages directory (Next.js Stable)

1. Start your computations in server handlers using Fork API

```tsx
export const getStaticProps = async () => {
  const scope = fork();

  await allSettled(pageStarted, { scope, params });

  return {
    props: {
      // notice serialized effector's scope here!
      values: serialize(scope),
    },
  };
};
```

2. Add provider to the `pages/_app.tsx` and provide it with server-side `values`

```tsx
export default function App({ Component, pageProps }: AppProps) {
  return (
    <main>
      <EffectorNext values={pageProps.values}>
        <Layout>
          <Component />
        </Layout>
      </EffectorNext>
    </main>
  );
}
```

You're all set. Just use effector's units anywhere in components code via `useUnit` from `effector-react`.


## App directory (Next.js Beta)

In the `app` directory there are no separate server handlers - because of it steps are a bit different.

1. New `app` directory considers all components as Server Components by default.

Because of that `EffectorNext` provider won't work as it is, as it uses client-only `createContext` API internally - you will immideatly get an compile error in Next.js

The official way to handle this - [is to re-export such components as modules with 'use client' directive](https://beta.nextjs.org/docs/rendering/server-and-client-components#third-party-packages).

To do so, create `effector-provider.tsx` file at the top level of your `app` directory and copy-paste following code from snippet there:

```tsx
// app/effector-provider.tsx
'use client';

import type { ComponentProps } from 'react';
import { EffectorNext } from '#/lib/effector-next';

export function EffectorAppNext({
  values,
  children,
}: ComponentProps<typeof EffectorNext>) {
  return <EffectorNext values={values}>{children}</EffectorNext>;
}

```

You should use this version of provider in the `app` directory from now on.

2. To use client components with effector units anywhere in the tree - add `EffectorAppNext` provider (which was created at previous step) at your [Root Layout](https://beta.nextjs.org/docs/routing/pages-and-layouts#root-layout-required)

If you are using [multiple Root Layouts](https://beta.nextjs.org/docs/routing/defining-routes#example-creating-multiple-root-layouts) - each one of them should also have the `EffectorAppNext` provider.

```tsx
// app/layout.tsx
import { EffectorAppNext } from "project-root/app/effector-provider"

export function function RootLayout({
  children,
}: {
  children: React.ReactNode;
}) {
  return (
    <html lang="en">
      <body>
        <EffectorNext>
          // rest of the components tree
        </EffectorNext>
     </html>
  )
}
```

3. Server computations work in a similiar to `pages` way, but inside Server Components of the `app` pages.

In this case you also will need to add the `EffectorAppNext` provider to the tree of this Server Component + provide it with serialized scope.

```tsx
// app/some-path/page.tsx
import { EffectorAppNext } from "project-root/app/effector-provider"

export default async function Page() {
  const scope = fork();

  await allSettled(pageStarted, { scope, params });

  const values = serialize(scope);

  return (
    <EffectorAppNext values={values}>
     // rest of the components tree
    </EffectorAppNext>
 )
}
```
This will both automatically render this subtree with effector's state and also will automatically "hydrate" client scope with new values.

You're all set. Just use effector's units anywhere in components code via `useUnit` from `effector-react`.

## Contributing

There is some stuff to do.
