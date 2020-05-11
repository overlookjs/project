# Overlook framework roadmap

# Files

## Types of files

1. Route files
2. Virtual route files (routes created by routes)
3. Server-side associate files (files accessed during start/handle)
4. Virtual server-side associate files
5. Server-side imported files (files `require()`-ed / `import()`-ed during start/handle)
6. Client-side files (included in client-side build)

### Route files

Route files are loaded from file-system ([@overlook/plugin-load](https://www.npmjs.com/package/@overlook/plugin-load))

### Virtual route files

See [here](./virtualRoutes.md).

### Server-side associate files

If app is to be built, need to signal to Overlook that the file should be included in the build.

`[INCLUDE_FILE]()` method would be provided by [@overlook/plugin-build](https://www.npmjs.com/package/@overlook/plugin-build). It should be called during init to signal record that this file needs to be included in the build.

`[READ_FILE]()` method would be provided by [@overlook/plugin-fs](https://www.npmjs.com/package/@overlook/plugin-fs)). It would read a file's contents. `fs.readFile()` could alternatively be used, but wouldn't work for [virtual files](#virtual-server-side-associate-files).

API something like:

```js
const Route = require('@overlook/route'),
  { INIT_ROUTE } = Route;
const fsPlugin = require('@overlook/plugin-fs'),
  { READ_FILE } = fsPlugin;
const buildPlugin = require('@overlook/plugin-build'),
  { INCLUDE_FILE } = buildPlugin;
const { RES } = require('@overlook/plugin-request');

const BuildFsRoute = Route.extend( fsPlugin )
  .extend( buildPlugin );

class MyRoute extends BuildFsRoute {
  async [INIT_ROUTE]() {
    await super[INIT_ROUTE]();
    this[INCLUDE_FILE]( './foo.html' );
  }

  async handle(req) {
    // Thanks to `[INCLUDE_FILE]()` call,
    // this file will be present in the build.
    const contents = await this[READ_FILE]('./foo.html');
    req[RES].end( contents );
  }
}
```

### Virtual server-side associate files

Same as above, but file doesn't exist in source. It is created dynamically during init.

This requires a virtual file system. Should probably be a separate module from [@overlook/plugin-build](https://www.npmjs.com/package/@overlook/plugin-build)), as could be used even if app is not being built e.g. if files need to be created to be later `require()`ed / `import()`ed (e.g. React components).

`[WRITE_FILE]()` is used to write a file to virtual file system. NB method is sync as file is not actually written to disc, just held in memory.

[@overlook/plugin-fs](https://www.npmjs.com/package/@overlook/plugin-fs)'s `[READ_FILE]()` method would be extended in virtual fs plugin to read from virtual file system.

```js
const BuildFsVirtualRoute = Route.extend( fsPlugin )
  .extend( buildPlugin )
  .extend( fsPlugin );

class MyRoute extends BuildFsVirtualRoute {
  async [INIT_ROUTE]() {
    await super[INIT_ROUTE]();
    this[WRITE_FILE](
      './foo.html',
      '<html><body>Hello!</body></html>'
    );
    this[INCLUDE_FILE]( './foo.html' );
  }

  async handle(req) {
    const contents = await this[READ_FILE]('./foo.html');
    req[RES].end( contents );
  }
}
```

Perhaps a method `[WRITE_INCLUDE_FILE]()` method could be provided as a shortcut for calling `[WRITE_FILE]()` followed by `[INCLUDE_FILE]()`.

How virtual file system is implemented TBC.

### Server-side imported files

Would need an async `[IMPORT_FILE]()` method in [@overlook/plugin-fs](https://www.npmjs.com/package/@overlook/plugin-fs).

TBC how to implement this. Perhaps hooking into Node's private API to require files?

This would be required for React server-side rendering.

### Client-side files

Same as [server-side associate files](#server-side-associate-files) and [virtual files](#virtual-server-side-associate-files), except `[INCLUDE_FILE]()` would not be called, as it doesn't need to be in build. Instead, whatever is creating the client-side build (e.g. Webpack) would create the client-side build and write it as virtual files to a public folder and *it* would call `[INCLUDE_FILE]()` to include the client-side bundle in the build.

Need to think about how paths are handled when [@overlook/plugin-build](https://www.npmjs.com/package/@overlook/plugin-build) is not used. Are paths relative or absolute? Relative to what?
