# Overlook framework roadmap

# React

Plan for implementation of React support:

## Plugins

1. `plugin-react`
2. `plugin-react-root`
3. `plugin-react-router`
4. `plugin-bundle`
5. `plugin-virtual-fs`
6. `plugin-static-dir`
7. `plugin-load-react`

### `plugin-react`

Used for every React route.

Exports `React` at `@overlook/plugin-react/react`.

Augments Route with `[REACT_FILE]` property. This prop should contain path to file to use as page for React.

If `[FILES].jsx` exists, that path is used as `[REACT_FILE]`.

If route does not also use `plugin-react-root`, locates closest route above (or self) using `plugin-react-router`. Call `[REGISTER_ROUTE]()` with `[REACT_FILE]` and `[URL_PATH]`.

### `plugin-react-root`

Used for root route in React app.

Exports `ReactDOM` at `@overlook/plugin-react-root/react-dom` + `ReactDOMServer` at `@overlook/plugin-react-root/react-dom/server`.

1. Passes `[REACT_FILE]` to `plugin-bundle` as entry point using `[REGISTER_ENTRY]()`.
2. If HTML file does not already exist, creates one and registers it as a virtual file (using bundle path returned from `plugin-webpack`).
3. Provides `.handle()` / `[HANDLE_ROUTE]()` method to serve HTML file.

### `plugin-react-router`

Provides `[REGISTER_ROUTE]()` method to receive paths to nested routes. NB `plugin-react` will call this method on own route too.

In `[INIT_CHILDREN]()`:

* Create a router file and save as virtual file with extension `.router.jsx`.
* Record router file's path as `[REACT_FILE]`.
* If route does not also use `plugin-react-root`, call closest route above using `plugin-react-router`. Call `[REGISTER_ROUTE]()` with `[REACT_FILE]` and `[URL_PATH]`.

### `plugin-bundle`

Provides `[REGISTER_ENTRY]()` method:

* Receives path to entry point
* Returns file path for bundle file

**TODO:** If bundle filename includes hash, it cannot be known ahead of compilation. How to deal with this?

In `[INIT_CHILDREN]()`:

* Bundle application with Webpack.
* Inject virtual files into Webpack as if they're real files.
* Save all output files as virtual files in `public` directory (or does it pass them direct to route which uses `plugin-static-dir`?)

### `plugin-virtual-fs`

Instantiates virtual file system on top of real `src` dir.

**TODO:** How to implement?

### `plugin-static-dir`

Could alternatively be called `plugin-public`.

* Serve files in a static directory.
* Add those files to build output.
* Include virtual files which have been added to the dir.

**TODO:** How to implement?

### `plugin-load-react`

Loader plugin. If route has a `.jsx` file but no `.js` file, create Route using `plugin-react`.

## Other changes required

`plugin-path` needs to record `[URL_PATH]` containing URL path. Or `plugin-react` needs to extend `[INIT_ROUTE]()` to create `[URL_PATH]`.
