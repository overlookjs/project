# Overlook framework roadmap

# React

Plan for implementation of React support:

## Plugins

1. `plugin-react`
2. `plugin-react-root`
3. `plugin-react-router`
4. `plugin-bundle`
5. `plugin-fs`
6. `plugin-static-dir`
7. `plugin-load-react`

### `plugin-react`

Used for every React route.

Exports `React` at `@overlook/plugin-react/react`.

Augments Route with `[REACT_FILE]` property. This prop should contain `File` object to use as page for React.

In `[INIT_ROUTE]()`:

* If `[FILES].jsx` exists, that file is used as `[REACT_FILE]` via a method `[GET_REACT_FILE]()`.
* If route does not also use `plugin-react-root`, locates closest route above (or self) using `plugin-react-router`. If found, call `[ADD_ROUTE]()` with `[REACT_FILE]` and `[URL_PATH]` on router route.

### `plugin-react-root`

Used for root route in React app.

Exports `ReactDOM` at `@overlook/plugin-react-root/react-dom` + `ReactDOMServer` at `@overlook/plugin-react-root/react-dom/server`.

In `[INIT_CHILDREN]()`:

* Call `[GET_REACT_ROOT_FILE]()`.
* Pass file to `plugin-bundle` as entry point using `[ADD_ENTRY]()`.

Purpose of `[GET_REACT_ROOT_FILE]()` rather than just using `[REACT_FILE]` is to allow this plugin and `plugin-react-router` to be stacked in any order.

In `[BUNDLE]()` (i.e. after bundling complete):

* Create virtual HTML file (using bundle path obtained from `plugin-bundle`'s `[BUNDLE_ENTRIES]`).
* Set HTML file as `[STATIC_FILE]`
* Add HTML file to `public` dir

Provide `.handle()` / `[HANDLE_ROUTE]()` method to serve HTML file.

### `plugin-react-router`

Provides `[ADD_ROUTE]()` method to receive files for nested routes. NB `plugin-react` will call this method on own route too.

In `[INIT_CHILDREN]()`:

* Call `[INIT_REACT_ROUTER]()`

In `[GET_REACT_ROOT_FILE]()`:

* Call `[INIT_REACT_ROUTER]()`
* Return `[REACT_FILE]`

`[INIT_REACT_ROUTER]()`:

* Call `[GET_REACT_ROUTER_FILE]()` if `[REACT_ROUTER_FILE]` not already set, and record result as `[REACT_ROUTER_FILE]`.
* If `[REACT_ROUTER_FILE]` still unset:
  * Call `[CREATE_REACT_ROUTER]()`.
  * Record router file as `[REACT_ROUTER_FILE]`.
  * Record router file as `[REACT_FILE]`.
* If route does not also use `plugin-react-root`, call closest route above using `plugin-react-router`. Call `[ADD_ROUTE]()` with `[REACT_FILE]` and `[URL_PATH]`.

`[CREATE_REACT_ROUTER]()`:

* Create a router file and save as virtual file with extension `.router.jsx`.
* Return `File` object.

`[GET_REACT_ROUTER_FILE]()`:

* Return `undefined` (intended to be overridden in plugins)

### `plugin-bundle`

`[ADD_ENTRY]()`:

* Receive file for entry point

In `INIT_CHILDREN()`:

* Call `[BUNDLE]()`

`[BUNDLE]()`:

* Bundle app with Webpack (NB set public path as `public`)
* Inject virtual files into Webpack as if they're real files.
* Save all output files as virtual files in `public` directory with paths prefixed with `static/` (via route which uses `plugin-static-dir`)
* Save mapping of entry point files to bundle files in `[BUNDLE_ENTRIES]`.

### `plugin-fs`

Instantiates virtual file system on top of real `src` dir.

Implemented [@overlook/plugin-fs](https://www.npmjs.com/package/@overlook/plugin-fs).

### `plugin-static-dir`

* Serve files in a static directory.
* Add those files to build output.
* Include virtual files which have been added to the dir.

Implemented [@overlook/plugin-static-dir](https://www.npmjs.com/package/@overlook/plugin-static-dir).

### `plugin-load-react`

Loader plugin. If route has a `.jsx` file but no `.js` / `.route.js` file, create Route using `plugin-react`.
