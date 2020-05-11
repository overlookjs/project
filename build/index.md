# Overlook framework roadmap

# Build

## Purpose

An Overlook app can be run by loading it, running `.init()` and then `[START]()`.

The advantage of building an app is that the init phase can be completed at build time and then an optimized build of the app written to a build folder which does not require initialization - it can be started immediately.

The built version:

* Does not require `.init()` to be called on it - so any time-consuming operations in init phase (e.g. route ordering) can be skipped when running app.
* Contains only executable Javascript - Typescript etc would be transpiled during build.
* ESM code transpiled to CJS (TBC).
* Contains client-side bundles pre-built.
* May contain other artefacts to make the app run faster in deployment - e.g. gzipped versions of static files.
* Loaded by `require()`ing the root route file. App started with `root[START]()`.

## Implementation

Build is created in a separate directory to source.

All routes are written to disc as `.js` files. This includes routes which are created dynamically in init phase. The difficulty is that live code must be serialized back to Javascript text.

* Where route was originally loaded from disc, the original file is copied.
* Where route created dynamically, a file will be created (see [virtual routes](../files/virtualRoutes.md))

Properties which have been added to the route in init phase are added in static code in the route file.

Parent and children are added by requiring their (sometimes newly created) route files.

This original code (assuming the route has parent route `index` and a child route called `edit`):

```js
const Route = require('@overlook/route'),,
  { INIT_ROUTE } = Route;
const pathPlugin = require('@overlook/plugin-path'),
  { PATH_PART } = pathPlugin;

const PathRoute = Route.extend( pathPlugin );

const LLAMAS = Symbol();

class MyRoute extends PathRoute {
  async [INIT_ROUTE]() {
    await super[INIT_ROUTE]();
    this[PATH_PART] = ':id';
    this.donkeys = 'nine';
    this[LLAMAS] = 'ten';
  }

  async handle( req ) {
    const llamas = this[LLAMAS];
    // ...
  }
}

module.exports = new MyRoute( { name: 'view' } );
```

Would be built as:

```js
const Route = require('@overlook/route'),,
  { INIT_ROUTE } = Route;
const pathPlugin = require('@overlook/plugin-path'),
  { PATH_PART } = pathPlugin;

const PathRoute = Route.extend( pathPlugin );

const LLAMAS = Symbol('LLAMAS');

class MyRoute extends PathRoute {
  // NB INIT_ROUTE method has been stripped out
  async handle( req ) {
    const llamas = this[LLAMAS];
    // ...
  }
}

module.exports = new MyRoute( { name: 'view' } );

module.exports[
  require('@overlook/symbol-store')['@overlook/plugin-path'].PATH_PART
] = ':id';
module.exports.donkeys = 'nine';
module.exports[LLAMAS] = 'ten';
module.exports.parent = require('./index.js');
module.exports.children = [
  require('./edit.js')
];
```

### User-defined Symbols

User-defined Symbols (i.e. symbols not provided by plugins) need special treatment. Parsing files to locate where symbols are created would not be possible in all cases (the code that creates them may be conditional or in another file).

Would need to do one of:

* Restrict Symbols to being defined in same file they are used, in top scope, and being named. The symbol definition can then be located by parsing the file and getting the variable name it's assigned to.
* If symbols are imported from another file, insist that the name of the var that symbol is assigned to is same as the Symbols's name.
* Make user create symbols with an Overlook-provided `createSymbol()` function. That function would keep a store of all symbols created in init and then recreate that store in a `symbols.js` file which would reference the symbols. Tricky part is ensuring that symbols are created in same order in built app as in source app. This is particularly hard as original loading of app is async so symbols may be created in non-deterministic order.
* Transpile files to replace all `Symbol()` expressions with `require('../symbols.js)['unique name']`. `symbols.js` is a file created in build. I think this would only work if the transpiling was done on source before the app is initialized = slower initialization = not good.
* Outlaw user-defined symbols! At least the user controls the whole namespace, so could make sure there's no property name conflicts.

Symbols in user-defined plugins are also problematic. A simpler solution than the above is to mandate all plugins being named, and their symbols defined in `new Plugin()`, so the global symbol store can be used to access them. Local plugins could be named with a special prefix (e.g. `~`) to prevent namespace conflicts with plugins published as NPM modules.

## Restrictions

All routes in an app which is going to built can only define methods statically (i.e. in the original route file, or methods added to Route class prototype by plugins). Init methods cannot add methods on the route dynamically. Reason is these dynamically-created functions cannot be serialized back to Javascript code due to not being able to identify out-of-scope variables or closures.

## Unanswered questions

### External files

What should happen with files imported from outside `routes` folder?

Modules in `node_modules` should probably be left where they are.

Files in project outside `routes` folder could be included in build, or left where they are. e.g. if `src/plugins/myPlugin.js` is required from within a route file `src/routes/index.js`, it could be:

1. left out of build and the `require()` statement in built route file amended to `require('../src/plugins/myPlugin.js')
2. included in build folder as `build/plugins/myPlugin.js`

(1) would require parsing the Route's code and adjusting `require()` paths.
(2) would require parsing not just the files which are directly `require()`ed, but also those files in turn in case they `require()` other files with relative paths in turn, which would also need to be moved in the build folder.

(2) is preferable, as it avoids having to deploy the `src` folder along with `build` folder. But it's pretty complicated. Maybe could use Webpack's ability to create a dependency graph, if that functionality is available independently of creating a whole Webpack build.
