# Overlook framework roadmap

# Virtual routes

Virtual routes are:

* Created in loading ([@overlook/plugin-load](https://www.npmjs.com/package/@overlook/plugin-load) / [@overlook/load-routes](https://www.npmjs.com/package/@overlook/load-routes))
* Created programmatically in other routes (`this.attachChild( route )`)

## The problem

See discussion of the purpose of building an app [here](../build/index.md).

When building the app, all routes must be turned into Javascript files which can be required i.e. live code must be serialized as a Javascript text file.

This is problematic when the code includes functions - function scope etc cannot be "dehydrated" back to Javscript text.

## TLDR

* [Solution 6/7](#implementation-of-6-or-7) is best API but quite complex to implement.
* [Solution 5](#implementation-of-5) is good balance of decent API and simple(ish) implementation.

Either:

1. implement [Solution 5](#implementation-of-5) to start with or...
2. go for [Solution 6/7](#implementation-of-6-or-7) straight away.

## Possible APIs

Virtual route files could be created by any of following APIs:

(in ascending order of niceness)

1. providing code for a file which can be required to instantiate the Route (`'module.exports = ...'`).
2. providing a path to load the Route instance from (`'/path/to/file.js'`).
3. providing a path to load the Route class from + props (`'/path/to/file.js', { props }`).
4. providing paths to load plugins from + props (`['@overlook/plugin-load', '@overlook/plugin-path'], { props }`)
5. define method on loading/creating route which returns a Route instance, then provide method name to `.createRoute()`. The built file `require()`s the loading/creating route, calls the method and exports the result.
6. specifying a Route class instance (`new Route( { props } )`).
7. specifying a Route class (`Route`).

## Implementations

### Implementation of (1)

Not great API to use.

Implementation: Would need to wrap the code in `function(require, module, exports, __filename, __dirname) { ... }` and provide a mock `require` function.

**CONCLUSION:** Terrible API and tricky to implement. Skip this.

### Implementation of (2) or (3)

(3) is more user-friendly than (2).

* Users would need to create files which export Route classes and point to them. This will work fine. They could possibly export the class as an ESM named export from the same file where it's used and then point to that export of itself (!)
* Plugins would need to similarly include files in their exports and point to them e.g. `'@overlook/plugin-load-html/Route.js'`.
* Properties which are keyed by symbols would need to be recreated by using the Symbol's description which specifies the plugin to get that symbol from (not applicable with solution (2)).

The problem is with plugins which are used as dependencies of another plugin. e.g. Plugin `plugin-Y` extends `plugin-X`. `plugin-Y` is the one the user imports, so is in the base `node_modules` folder. `plugin-X` is a dependency of `plugin-Y`, so may not be in base `node_modules`, it may be in `node_modules/plugin-Y/node_modules`. So if `plugin-X` creates a route, it cannot point to `plugin-X/Route.js` as the path to load the Route class from. From the application's point of view, it cannot load from `plugin-X/Route.js`.

NB A plugin cannot know internally whether it has been used by user directly, or by another plugin, so it cannot help Overlook to determine the import path.

The same problem occurs with symbols which come from a plugin which is not a direct dependency. But it's less acute as symbols are the same for every version of a plugin.

Possible solutions:

* Plugins provide an absolute path to the Route file class. Overlook then creates route files containing e.g. `const Route = require('../../node_modules/plugin-Y/node_modules/plugin-X/Route.js'); module.exports = new Route( { props} );`. NB absolute path is converted to relative path, so path where the build is deployed can differ from path on the build machine.

* Same, except Overlook would track which plugins depend on others, and so could convert the absolute path to an array ['plugin-Y', 'plugin-X', 'Route.js'] which can be used to load the file relative to the plugin which it's provided by.

* Plugins are provided a list of the "dependency path" using them e.g. `plugin-X` is passed `['plugin-Y']`. The plugin then passes that back out when declaring where to find the file for a Route class. This makes writing plugins more complex, and probably the previous solution would be equally easy to implement once dependency tracking is implemented.

* All plugins must declare their dependencies as peer dependencies. This would flatten the dependency tree so resolution is easy, but (a) is bad for user having to manage a bunch of extra dependencies, and (b) prevents different routes using conflicting versions of plugins.

The first of these solutions is the easiest to implement. The downside is that the built app will only work if the structure of `node_modules` in deployment is the same as it was at build time. Assuming that a lock file is used, however, this would be the case. The restriction is that dependencies cannot be updated without rebuilding the app. That's probably acceptable.

The problem with symbols can be solved by:

* Import plugin-defined symbols from global symbol store ([@overlook/symbol-store](https://www.npmjs.com/package/@overlook/symbol-store)), based on symbol descriptions (e.g. symbol with description `@overlook/plugin-path.PATH_PART` can be imported with `require('@overlook/symbol-store')['@overlook/plugin-path'].PATH_PART`).
* User-defined symbols could be defined in a `symbols.js` file created during build which creates symbols to represent all user-defined symbols. The built files then import symbols from this file.

**CONCLUSION:** Go with 1st solution. Improve the implementation later if it's problematic.

### Implementation of (4)

Easier to use than (2) or (3) for users, as they don't need to create a file just for a Route class. It can be defined dynamically by listing paths of the plugins to use + props to add to route.

Downside is custom classes cannot be created inline - they must be composed of plugins which are in separate files.

Paths could be:

* absolute - `/full/path/to/app/src/my-plugin.js`
* relative - `../../myPlugin.js`
* relative to app root - `~/src/my-plugin.js` (requires support)
* module require path - `@overlook/plugin-load`

e.g. `['@overlook/plugin-load', '@overlook/plugin-path', '../../my-plugin.js'], { foo: 'bar' }`

This can be converted in build to:

```js
const Route = require('@overlook/route')
  .extend(require('@overlook/plugin-load'))
  .extend(require('@overlook/plugin-path'))
  .extend(require('../../../src/my-plugin.js'));
module.exports = new Route( { foo: 'bar' } );
```

Within plugins, it's trickier. The same problems discussed above of plugins which are dependencies of other plugins raises its head.

Possible solution:

* Create a global plugin store.
* Every time `Route.extend()` is called, the plugin is added to this store.
* The store is an object. Plugins would be stored as `require('@overlook/plugin-store')[plugin.name][plugin.version] = plugin`.
* Any plugin can then imported from this store.
* Plugins define the plugins to be used with *absolute* paths only - `require.resolve('@overlook/plugin-path')`. Overlook `require()`s that file and gets plugin name and version.
* It's possible that in dev what were two different instances of a plugin (`node_modules/plugin-Y/node_modules/plugin-X` and `node_modules/plugin-Z/node_modules/plugin-X`) will end up pointing to the same instance in build. But plugins should not keep any state inside themselves, so it shouldn't matter. Same as first solution, plugins provide an absolute path to the Route class file and Overlook will determine plugin name and version from that, by finding the `package.json` file above the path provided.

Overlook can than build the route file as:

```js
const Route = require('@overlook/route')
  .extend(require('@overlook/plugin-store')['@overlook/plugin-path']['0.8.0']);
module.exports = new Route( { foo: 'bar' } );
```

NB Ideally would reduce the need for 2 store packages by combining symbols and plugins stores into one package `@overlook/store` which exports an object with `.symbols` and `.plugins` properties.

**CONCLUSION:** This sounds quite do-able. Could implement next after (2) / (3).

### Implementation of (5)

The advantage of this implementation is there is no need to statically refer to other files by path. It's pretty simple, even in the cases where creating/loading a route is done inside plugins.

The disadvantage is that it binds routes together. The built files for the virtual route need to `require()` the route that created it in order to call the method to create itself.

How it would work:

```js
// @overlook/plugin-load-html plugin
// Creates a child route to serve a static HTML file
const Plugin = require('@overlook/plugin');
const Route = require('@overlook/route'),
  { INIT_ROUTE } = Route;
const { CREATE_CHILD } = require('@overlook/plugin-build');
const staticPlugin = require('@overlook/static'),
  { STATIC_FILE_PATH } = staticPlugin;
const pkg = require('./package.json');

const loadHtmlPlugin = new Plugin(
  pkg,
  { symbols: ['CREATE_HTML_ROUTE'] },
  extend
);
module.exports = loadHtmlPlugin;
const { CREATE_HTML_ROUTE } = loadHtmlPlugin;

const StaticRoute = Route.extend( staticPlugin );

function extend( Route ) {
  return class extends Route {
    async [INIT_ROUTE]() {
      await super[INIT_ROUTE]();

      this[CREATE_CHILD](
        CREATE_HTML_ROUTE,
        { name: 'foo', path: './foo.html' }
      );
    }

    [CREATE_HTML_ROUTE]( { name, path } ) {
      return new StaticRoute( { name, [STATIC_FILE_PATH]: path } );
    }
  };
}
```

`[CREATE_CHILD]()` is a method which would need to be implemented in [@overlook/plugin-build](https://www.npmjs.com/package/@overlook/plugin-build). It would look something like:

```js
// [CREATOR_ROUTE], [CREATOR_METHOD], [CREATOR_PARAMS]
// are defined in @overlook/plugin-build
[CREATE_CHILD]( method, params, isDir ) {
  const child = this[method](params);
  child[CREATOR_ROUTE] = this;
  child[CREATOR_METHOD] = method;
  child[CREATOR_PARAMS] = params;
  // TODO Record `isDir` somehow - if true,
  // would create file as an index route in new directory
  this.attachChild(child);
  return this;
}
```

The `[CREATOR_METHOD]` and `[CREATOR_PARAMS]` properties saved on the created route are serializable, and `[CREATOR_ROUTE]` could be converted to a relative path. So when building the created route, they can all be transformed into Javascript text in the built file.

This created route would be built as this file:

```js
const { CREATE_HTML_ROUTE } = require('@overlook/symbol-store')['@overlook/plugin-load-html'];
const parent = require('./index.js');
module.exports = parent[CREATE_HTTP_FILE]( { name: 'foo', path: './foo.html' } );
```

Could also be implemented in loaders by returning `{ method: ..., props: { ... } }` from `[IDENTIFY_ROUTE_FILE]()`. [@overlook/plugin-load](https://www.npmjs.com/package/@overlook/plugin-load) would need to extend [@overlook/plugin-build](https://www.npmjs.com/package/@overlook/plugin-build).

[@overlook/load-routes](https://www.npmjs.com/package/@overlook/load-routes) would need to take a path (absolute path or module path) to loader Route rather than a class so it can be built too. None of the problems with locating deep dependency plugins would apply as [@overlook/load-routes](https://www.npmjs.com/package/@overlook/load-routes) is only used by user, not within plugins.

**CONCLUSION:** This is pretty simple to implement and has an better API than all previous options. Implement this first.

### Implementation of (6) or (7)

(6) and (7) are the nicest API for creating routes, but then the problem is how to serialize that to a file. Would require Route classes to include some properties from which it can be deduced what paths the plugins which created that class are at.

To a degree, this exists already. Route classes have a property `[EXTENSIONS]` which lists all the plugins used to create a Route subclass.

However, what's not apparent from `[EXTENSIONS]` is what plugins were applied within other plugins (i.e. nested dependencies). Would need to alter `Route.extend()` to track further calls to `.extend()` within `extend()` function (i.e. applying a nested dependency plugin) and record this on the Route, to be able to make it work.

It would *not* support all ad hoc Route class extension (`class extends MyRoute { ... }`) in user code, as this code could not be imported from any file.

However, could have limited support for ad hoc extensions, by calling `.toString()` on `Route` or `route.constructor` to get the Javascript code. Restrictions would be:

1. Class extension cannot reference external scope (i.e. functions or vars defined outside the class definition)
2. Symbols used would need to be from plugins used by the Route class, or named so they are trackable (see [here](../build/index.md#user-defined-symbols])).

A fuller implementation could do a bit better than this by parsing the code of the file the custom class extension is defined to locate vars in scope and then extract this code with tree-shaking to remove everything else.

**CONCLUSION:** Nice API but trickier to implement.
