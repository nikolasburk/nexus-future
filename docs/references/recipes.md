## Use Prisma

1. Install the prisma plugin

   ```cli
   npm install nexus-plugin-prisma
   ```

1. Add a `schema.prisma` file. Add a datasource. Here we're working with SQLite. Add photon.

   ```diff
   +++ prisma/schema.prisma
   +
   +  datasource db {
   +    provider = "sqlite"
   +    url      = "file:dev.db"
   +  }
   +
   +  generator photonjs {
   +    provider = "photonjs"
   +  }
   ```

1. Initialize your database

   ```cli
   npx nexus db init
   ```

1. Done. Now your app has:

   1. Functioning `db` command

      ```cli
      nexus db
      ```

   1. `nexus-prisma` Nexus plugin allowing e.g.:

      ```diff
      +++ src/schema.ts
        objectType({
          name: 'User',
          definition(t) {
      -     t.id('id)
      -     t.string('name')
      +     t.model.id()
      +     t.model.name()
          },
        })
      ```

   1. An instance of the generated Photon.JS client is a added to context under `photon` property, allowing e.g.:

      ```diff
      +++ src/schema.ts
        queryType({
          definition(t) {
            t.list.field('users', {
              type: 'User',
      -       resolve() {
      -         return [{ id: '1643', name: 'newton' }]
      +       resolve(_root, _args, ctx) {
      +         return ctx.photon.users.findMany()
              },
            })
          },
        })
      ```

   1. The TypeScript types representing your Prisma models are registered as a Nexus data source. In short this enables proper typing of `parent` parameters in your resolves. They reflect the data of the correspondingly named Prisma model.

## Create a Consumable Plugin

1. Scaffold Your plugin project

   ```cli
   npx nexus-future create plugin
   ```

2. Publish it

   ```cli
   yarn publish
   ```

## Local PostgreSQL

The reccommended way to run postgres locally is with docker, because it is easy flexible and reliable.

1. Start a postgres server for your app:

   ```cli
   docker run --detach --publish 5432:5432 --name 'postgres' postgres
   ```

2. Now you can use a connection URL like:

   ```
   postgresql://postgres:postgres@localhost:5432/myapp
   ```

If you don't want to use a docker, here are some links to alternative approaches:

- [With Homebrew](https://wiki.postgresql.org/wiki/Homebrew)

## Go to proudction

1. Add a build script

   ```diff
   +++ package.json
   + "build": "nexus build"
   ```

2. Add a start script

   ```diff
   +++ package.json
   + "start": "node node_modules/.build"
   ```

3. In many cases this will be enough. Many deployment platforms will call into these scripts by default. You can customize where `build` outputs to if your deployment platform requires it. There are built in guides for `zeit` and `heroku` which will check your project is prepared for deployment to those respective platforms. Take advantage of them if applicable:

   ```diff
   +++ package.json
   + "build": "nexus build --deployment now"
   ```

   ```diff
   +++ package.json
   + "build": "nexus build --deployment heroku"
   ```

## Prisma + Heroku + PostgreSQL

1. Confirm the name of the environment variable that Heroku will inject into your app at runtime for the database connection URL. In a simple setup, with a single attached atabase, it is `DATABASE_URL`.
1. Update your Prisma Schema file to get the database connection URL from an environment variable of the same name as in step 1. Example:

   ```diff
   --- prisma/schema.prisma
   +++ prisma/schema.prisma
     datasource postgresql {
       provider = "postgresql"
   -   url      = "postgresql://<user>:<pass>@localhost:5432/<db-name>"
   +   url      = env("DATABASE_URL")
     }
   ```

1. Update your local development environment to pass the local development database connection URL via an environment variable of the same name as in step 1. Example with [direnv](https://direnv.net/):

   1. Install `direnv`

      ```cli
      brew install direnv
      ```

   1. Hook `direnv` into your shell ([instructions](https://direnv.net/docs/hook.html))
   1. Setup an `.envrc` file inside your project

      ```diff
      +++ .envrc
      + DATABASE_URL="postgresql://postgres:postgres@localhost:5432/myapp"
      ```

   1. Approve the `.envrc` file (one time, every time the envrc file changes).
      ```cli
      direnv allow .
      ```
   1. Done. Now when you work within your project with a shell, all your commands will be run with access to the environment variables defined in your `.envrc` file. The magic of `direnv` is that these environment variables are automatically exported to and removed from your environment based on you being within your prject directory or not.

## Integrate `createTestContext` with `jest`

1. Wrap `createTestContext` so that it is integrated with the `jest` test suite lifecycle hooks:

   ```ts
   // tests/__helpers.ts
   import { createTestContext, TestContext } from 'nexus-future/testing'

   export function createTestContext(): TestContext {
     let ctx: TestContext

     beforeAll(async () => {
       ctx = await createTestContext()
       await ctx.app.server.start()
     })

     afterAll(async () => {
       await ctx.app.server.stop()
     })

     return ctx
   }
   ```

1. Import your wrapped version into all test suites needing it:

   ```ts
   // tests/foo.spec.ts
   import { createTestContext } from './__helpers'

   const ctx = createTestContext()

   it('foo', () => {
     // use `ctx` in here
   })
   ```

   Note that `ctx` is not usable outside of `jest` blocks (`it` `before` `after` `...`). If you try to you'll find it to be `undefined`.

   ```ts
   import { createTestContext } from './__helpers'

   const { app } = createTestContext() // Error!
   ```
