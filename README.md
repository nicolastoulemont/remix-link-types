# Generate types to make type safe Links with Remix

## Disclaimer

This is an incomplete program that isn't made (so far) to pass the remix tests suit. It won't cover every (many) cases.
and can very much be improved.

## How to use

Because we can't hook (at least so far) into remix itself, we have to run the type generation outside of remix, in user land.

We do so by :

1. Generating the types whenever the routes files or the remix config file change using a file watcher
2. Importing the generated types in our components.

### Type generation

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
const watcher = chokidar.watch(routesDirPath, 'remix.config.js')
watcher.on('all', function (event, path) {
  writeLinkTypes()
})
process.on('SIGINT', function () {
  watcher.close()
})
```

### Link components

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
