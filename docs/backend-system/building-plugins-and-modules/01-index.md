---
id: index
title: Building Backend Plugins and Modules
sidebar_label: Overview
# prettier-ignore
description: Building backend plugins and modules using the new backend system
---

> **DISCLAIMER: The new backend system is in alpha, and still under active development. While we have reviewed the interfaces carefully, they may still be iterated on before the stable release.**

> NOTE: If you have an existing backend and/or backend plugins that are not yet
> using the new backend system, see [migrating](./08-migrating.md).

## Overview

Backend [plugins](../architecture/04-plugins.md) and
[modules](../architecture/06-modules.md), sometimes collectively referred to as
backend _features_, are the building blocks that adopters add to their
[backends](../architecture/02-backends.md).

## Creating a new Plugin

This guide assumes that you already have a Backend project set up. Even if you only want to develop a single plugin for publishing, we still recommend that you do so in a standard Backstage monorepo project, as you often end up needing multiple packages. For instructions on how to set up a new project, see our [getting started](../../getting-started/index.md#prerequisites) documentation.

To create a Backend plugin, run `yarn new`, select `backend-plugin`, and fill out the rest of the prompts. This will create a new package at `plugins/<pluginId>-backend`, which will be the main entrypoint for your plugin.

## Plugins

A basic backend plugin might look as follows:

```ts
import {
  createBackendPlugin,
  coreServices,
} from '@backstage/backend-plugin-api';
import { createExampleRouter } from './router';

export const examplePlugin = createBackendPlugin({
  pluginId: 'example',
  register(env) {
    env.registerInit({
      deps: {
        // Declare dependencies to services that you want to consume
        logger: coreServices.logger,
        httpRouter: coreServices.httpRouter,
      },
      async init({
        // Requested service instances get injected as per above
        logger,
        httpRouter,
      }) {
        // Perform your initialization and access the services as needed
        const example = createExampleRouter(logger);
        logger.info('Hello from example plugin');
        httpRouter.use(example);
      },
    });
  },
});
```

When you depend on `plugin` scoped services, you'll receive an instance of them
that's specific to your plugin. In the example above, the logger might tag
messages with your plugin ID, and the HTTP router might prefix API routes with
your plugin ID, depending on the implementation used.

See [the article on naming patterns](../architecture/07-naming-patterns.md) for
details on how to best choose names/IDs for plugins and related backend system
items.

## Modules

Backend modules are used to extend [plugins](../architecture/04-plugins.md) with
additional features or change existing behavior. They must always be installed
in the same backend instance as the plugin that they extend, and may only extend
a single plugin. Modules interact with their target plugin using the [extension
points](./05-extension-points.md) registered by the plugin, while also being
able to depend on the [services](../architecture/03-services.md) of that plugin.
That last point is worth reiterating: injected `plugin` scoped services will be
the exact
same ones as the target plugin will receive later, i.e. they will be scoped
using the target `pluginId` of the module.

A module depends on the extension points exported by the target plugin's library
package, for example `@backstage/plugin-catalog-node`, and does not directly
declare a dependency on the plugin package itself. This is to avoid a direct
dependency and potentially cause duplicate installations of the plugin package,
while duplicate installations of library packages should always be supported.

The following is an example of how to create a module that adds a new processor
using the `catalogProcessingExtensionPoint`:

```ts
import { createBackendModule } from '@backstage/backend-plugin-api';
import { catalogProcessingExtensionPoint } from '@backstage/plugin-catalog-node';
import { MyCustomProcessor } from './MyCustomProcessor';

export const catalogModuleExampleCustomProcessor = createBackendModule({
  moduleId: 'exampleCustomProcessor',
  pluginId: 'catalog',
  register(env) {
    env.registerInit({
      deps: {
        catalog: catalogProcessingExtensionPoint,
        logger: coreServices.logger,
      },
      async init({ catalog }) {
        catalog.addProcessor(new MyCustomProcessor(logger));
      },
    });
  },
});
```

See [the article on naming patterns](../architecture/07-naming-patterns.md) for
details on how to best choose names/IDs for modules and related backend system
items.

Notice that we're placing the extension point we want to interact with in the
`deps` option, while also depending on the logger service at the same time. When
initializing modules we can depend on both extension points and services
interchangeably. You can also depend on multiple extension points at once, in
case the implementation of the module requires it.

It is typically best to keep modules slim and to each only add a single new
feature. It is often the case that it is better to create two separate modules
rather than one that provides both features. The one limitation here is that
modules can not interact with each other and need to be self contained.

### HTTP Handlers

Since modules have access to the same services as the plugin they extend, they
are also able to register their own HTTP handlers. For more information about
the HTTP service, see [core services](../core-services/01-index.md). When
registering HTTP handlers, it is important to try to avoid any future conflict
with the plugin itself, or other modules. A recommended naming pattern is to
register the handlers under the `/modules/<module-id>` path, where `<module-id>`
is the kebab-case ID of the module, for example
`/modules/example-custom-processor/v1/validators`. In a standard backend setup
the full path would then be
`<backendUrl>/api/catalog/modules/example-custom-processor/v1/validators`.

### Database Access

The same applies for modules that perform their own migrations and interact with
the database. They will run on the same logical database instance as the target
plugin, so care must be taken to choose table names that do not risk colliding
with those of the plugin. A recommended naming pattern is `<package
name>__<table name>`, for example the `@backstage/backend-tasks` package creates
tables named `backstage_backend_tasks__<table>`. If you use the default [`Knex`
migration facilities](https://knexjs.org/guide/migrations.html), you will also
want to make sure that it uses similarly prefixed migration state tables for its
internal bookkeeping needs, so they do not collide with the main ones used by
the plugin itself. You can do this as follows:

```ts
await knex.migrate.latest({
  directory: migrationsDir,
  tableName: 'backstage_backend_tasks__knex_migrations',
});
```

## Customization

There are several ways of configuring and customizing plugins and modules.

### Extension Points

Whenever you want to allow modules to configure your plugin dynamically, for
example in the way that the catalog backend lets catalog modules inject
additional entity providers, you can use the extension points mechanism. This is
described in detail with code examples in [the extension points architecture
article](../architecture/05-extension-points.md).

### Configuration

Your plugin or module can leverage the app configuration to configure its own
internal behavior. You do this by adding a dependency on `coreServices.config`
and reading from that. This pattern is a good fit especially for customization
that needs to be different across environments.

```ts
import { coreServices } from '@backstage/backend-plugin-api';

export const examplePlugin = createBackendPlugin({
  pluginId: 'example',
  register(env) {
    env.registerInit({
      deps: { config: coreServices.config },
      async init({ config }) {
        // Here you can read from the current config as you see fit, e.g.:
        const value = config.getOptionalString('example.value');
      },
    });
  },
});
```

Before adding custom configuration options, make sure to read [the configuration
docs](../../conf/index.md), in particular the section on [defining configuration
for your own plugins](../../conf/defining.md) which explains how to establish a
configuration schema for your specific plugin.

### Options

You'll have noted that the return values from `createBackendPlugin` and
`createBackendModule` are actually factory functions. These can be made to
accept options that shall be passed in at initialization time.

This pattern can be a good fit for fairly simple, static configuration values.

```ts
export interface ExampleOptions {
  silent?: boolean;
}

export const examplePlugin = createBackendPlugin(
  (options?: ExampleOptions) => ({
    pluginId: 'example',
    register(env) {
      env.registerInit({
        deps: {
          // Omitted dependencies but they remain the same as above
        },
        async init(
          {
            /* ... */
          },
        ) {
          // Here you can access the given options and act accordingly, e.g.:
          if (!options?.silent) {
            // ...
          }
        },
      });
    },
  }),
);
```

The return type from `createBackendPlugin` and `createBackendModule` will mimic
this, resulting in a factory function that accepts an optional options object.
You can also make it required to pass in options, by removing the optionality
(the question mark on the options) above.

```ts
backend.add(examplePlugin({ silent: true }));
```

Use this pattern sparingly. There is a big convenience benefit in allowing
people to easily install backend plugins without having to always pass in a
large number of options, and these options cannot easily be made dynamic based
on the environment etc.
