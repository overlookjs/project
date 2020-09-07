# Overlook framework roadmap

# Files

## File objects

Files are represented by `File` class instances.

`File` is a class exported by [@overlook/plugin-fs](https://www.npmjs.com/package/@overlook/plugin-fs).

A file has two properties `path` and `content`. The `File` class constructor will take either:

* `path` only, for a file that already exists.
* `path` and `content` for a virtual file.

The `[FILES]` object populated by [@overlook/plugin-load](https://www.npmjs.com/package/@overlook/plugin-load) will contain `File` objects.

## Virtual files

When creating a virtual file with `[WRITE_FILE]()`, the path may be:

* Omitted - [@overlook/plugin-fs](https://www.npmjs.com/package/@overlook/plugin-fs) will create a path.
* Provided as absolute path - actual path will correspond as closely as as possible to provided path.
* Provided as relative path - actual path will be in same dir as the route file.

`[WRITE_FILE]()` will ensure the path does not clash with an existing file on disc, or existing virtual file, so all paths are unique. The directory will always be the same as what's requested, but the filename may be different (e.g. `index.html` -> `index2.html`).

So files should be created, and the path then obtained from `file.path`.

But in general methods which work with files should take `File` objects rather than paths.

The highest route (usually router root) which uses [@overlook/plugin-fs](https://www.npmjs.com/package/@overlook/plugin-fs) will need to keep a record of all virtual files created so that paths assigned to them can be ensured to be unique.

**TODO** How will relative paths be resolved for routes which have been created programmatically and therefore have no `[DIR_PATH]` defined?

## Building files

When building the app, the [@overlook/plugin-build](https://www.npmjs.com/package/@overlook/plugin-build) will alter the `.path` property of all `File` objects to be a relative path, relative to build dir root. This way source paths are never left visible in the build.

If files need to be located in a specific position relative to each other, `[BUILD_FILE]()` will take a path (relative to build root) which will be maintained. In this way client-side files which `require` / `import` each other will not have their relative positions changed, so `require('./foo.js')` will continue to be valid.

`[BUILD_FILE]()` should throw an error if there's an attempt to write two files to the same build path.

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

`[BUILD_FILE]()` method would be provided by [@overlook/plugin-build](https://www.npmjs.com/package/@overlook/plugin-build). It should be called during init to signal that this file needs to be included in the build.

`[READ_FILE]()` method would be provided by [@overlook/plugin-fs](https://www.npmjs.com/package/@overlook/plugin-fs)). It would read a file's contents. `fs.readFile()` could alternatively be used, but wouldn't work for [virtual files](#virtual-server-side-associate-files).

API something like:

```js
const Route = require('@overlook/route'),
  { INIT_ROUTE } = Route;
const buildPlugin = require('@overlook/plugin-build'),
  { BUILD_FILE } = buildPlugin;
const { READ_FILE } = require('@overlook/plugin-fs');
const { FILES } = require('@overlook/plugin-load');
const { RES } = require('@overlook/plugin-request');

const BuildFsRoute = Route.extend( buildPlugin );
// NB `plugin-build` extends `plugin-fs`

const HTML_FILE = Symbol('HTML_FILE');

class MyRoute extends BuildFsRoute {
  async [INIT_ROUTE]() {
    await super[INIT_ROUTE]();

    const htmlFile = this[FILES].html;
    this[HTML_FILE] = htmlFile;
    this[BUILD_FILE]( htmlFile );
  }

  async handle(req) {
    // Thanks to `[BUILD_FILE]()` call,
    // this file will be present in the build.
    const contents = await this[READ_FILE]( this[HTML_FILE] );
    req[RES].end( contents );
  }
}
```

### Virtual server-side associate files

Same as above, but file doesn't exist in source. It is created dynamically during init.

This requires a virtual file system, which should be implemented by [@overlook/plugin-fs](https://www.npmjs.com/package/@overlook/plugin-fs). NB Could be used even if app is not being built e.g. if files need to be created to be later `require()`ed / `import()`ed (e.g. React components).

`[WRITE_FILE]()` is used to write a file to virtual file system.

`[WRITE_FILE]()` could be sync as file is not actually written to disc, just held in memory, but better to make it async to allow for future extension (e.g. actually writing files to a temp folder which can be cached across runs).

[@overlook/plugin-fs](https://www.npmjs.com/package/@overlook/plugin-fs)'s `[READ_FILE]()` method would read from virtual file system.

```js
const BuildFsVirtualRoute = Route.extend( buildPlugin );

class MyRoute extends BuildFsVirtualRoute {
  async [INIT_ROUTE]() {
    await super[INIT_ROUTE]();
    const htmlFile = await this[WRITE_FILE]( {
      content: '<html><body>Hello!</body></html>',
      path: './foo.html'
      // or ext: 'html'
    } );
    this[HTML_FILE] = htmlFile;
    this[BUILD_FILE]( htmlFile );
  }

  async handle(req) {
    const contents = await this[READ_FILE]( this[HTML_FILE] );
    req[RES].end( contents );
  }
}
```

Perhaps a method `[WRITE_BUILD_FILE]()` method could be provided as a shortcut for calling `[WRITE_FILE]()` followed by `[BUILD_FILE]()`.

### Server-side imported files

Would need an async `[IMPORT_FILE]()` method in [@overlook/plugin-fs](https://www.npmjs.com/package/@overlook/plugin-fs).

TBC how to implement this. Perhaps hooking into Node's private API to require files?

The loaded JS would then be written to a property on the Route, so it gets included in the build.

This would be required for React server-side rendering.

### Client-side files

Same as [server-side associate files](#server-side-associate-files) and [virtual files](#virtual-server-side-associate-files), except `[BUILD_FILE]()` would not be called, as it doesn't need to be in build. Instead, whatever is creating the client-side build (e.g. Webpack) would create the client-side build and write it as virtual files to a public folder and *it* would call `[BUILD_FILE]()` to include the client-side bundle in the build.
