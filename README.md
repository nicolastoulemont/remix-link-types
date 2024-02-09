# Generate types to make type safe Links with Remix

![Type safe link gif](./assets/demo.gif)

## Disclaimer

This is an incomplete program that isn't made to pass the remix tests suit, it doesn't cover every (many) cases and can very much be improved.
It was made for demonstration purposes for a use case of writing types based on assets. It is not production ready and doesn't aim to be.

I'm sure that the remix team will at some point make their navigation related components and functions type safe in a much better way.

## How to use

Because we can't hook into remix itself, we have to run the type generation outside of remix, in user land.

We do so by :

1. Generating the types whenever the routes files or the remix config file change using a file watcher that we will run in parallel of our remix dev command.
2. Importing the generated types in our components.

### Type generation

#### Script

```ts
// write-links-type.ts
import chokidar from 'chokidar'
import path from 'path'
import fs from 'fs/promises'
import { generateLinkTypes } from '@nicolastoulemont/remix-link-types'

import remixConfig from './remix.config'
import { defineRoutes } from '@remix-run/dev/dist/config/routes'

const routesDirPath = path.join(__dirname, 'app/routes')

async function writeLinkTypes() {
  await fs.writeFile(
    // Or where ever you need to write the types
    path.join(__dirname, 'app/components/Link/link.types.ts'),
    await generateLinkTypes({
      routesDirPath,
      // Only needed if you define routes via the remix.config file
      routeManifest: remixConfig.routes(defineRoutes),
    })
  )
}

// Only need to watch the remix.config file if you define routes in it.
const watcher = chokidar.watch([routesDirPath, 'remix.config.js'])
watcher.on('all', function (event, path) {
  writeLinkTypes()
})
process.on('SIGINT', function () {
  watcher.close()
})
```

#### Output

The `generateLinkTypes` function will return a string with the following union types `RawRouteConfig` and `RouteConfig`. They're both unions of the following type:

```ts
// Route: backoffice.brands.$id.tsx
// will add the following type to the union
type Example = {
  path: '/backoffice/brands/{id}'
  params: {
    id: string
  }
}

// Route: backoffice.brands.tsx
// will add the following type to the union
type Example = {
  path: '/backoffice/brands'
}
```

Nothing fancy really.

- The `RouteConfig` excludes any potential type literal `/${string}` path from the generated union type. This type is generated when having a "catch all" route, but will also block typescript from providing actionable intellisense when present (since all other route paths can be included in that `/${string}` literal type).
- The `RawRouteConfig` is the 1 to 1 mapping of the routes to types, including `/${string}` type literal, if needed for some reason.

I recommend using the `RouteConfig` type because you probably won't want to link to your catch all route anyway.

### Link components

The links components are simple wrapper around the Remix links, where we replace the replaceable segments in the paths with the matching values in the params prop.

```tsx
/**
 * /Link <- folder
 *  index.tsx <- components file
 *  link.types.ts <- generated type file
 */

import {
  Link as RemixLink,
  NavLink as RemixNavLink,
  LinkProps,
  NavLinkProps,
} from '@remix-run/react'
import { type RouteConfig } from './link.types'
import { useMemo } from 'react'

type TypeSafeLinkProps = RouteConfig & Omit<LinkProps, 'to'>

export function TypeSafeLink({
  className,
  path,
  // @ts-expect-error Typescript doesn't allow us to use the params prop without knowing the path
  params,
  ...rest
}: TypeSafeLinkProps) {
  const to = useMemo(
    () =>
      path.replaceAll(/{.*}/g, (value) => {
        if (!params) {
          throw new Error('Missing params props')
        }
        const paramsValue = params[value.slice(1, -1)]
        if (!paramsValue) {
          throw new Error(`Missing params value: ${paramsValue}`)
        }
        return paramsValue
      }),
    [path, params]
  )

  return <RemixLink to={to} {...rest} />
}

type TypeSafeNavLinkProps = RouteConfig & Omit<NavLinkProps, 'to'>

export function TypeSafeNavLink({
  className,
  path,
  // @ts-expect-error Typescript doesn't allow us to use the params prop without knowing the path
  params,
  ...rest
}: TypeSafeNavLinkProps) {
  const to = useMemo(
    () =>
      path.replaceAll(/{.*}/g, (value) => {
        if (!params) {
          throw new Error('Missing params props')
        }
        const paramsValue = params[value.slice(1, -1)]
        if (!paramsValue) {
          throw new Error(`Missing params value: ${paramsValue}`)
        }
        return paramsValue
      }),
    [path, params]
  )

  return <RemixNavLink to={to} {...rest} />
}
```

### Redirect function

Same logic really

```ts
import { redirect } from '@remix-run/node'
import { RouteConfig } from '~/components/Link/link'

type TypeSafeRedirectParams = RouteConfig & { init?: number | ResponseInit }

export function typeSafeRedirect({
  path,
  // @ts-expect-error Typescript doesn't allow us to use the params prop without knowing the path
  params,
  init,
}: TypeSafeRedirectParams) {
  const url = path.replaceAll(/{.*}/g, (value) => {
    if (!params) {
      throw new Error('Missing params props')
    }
    const paramsValue = params[value.slice(1, -1)]
    if (!paramsValue) {
      throw new Error(`Missing params value: ${paramsValue}`)
    }
    return paramsValue
  })

  return redirect(url, init)
}
```
