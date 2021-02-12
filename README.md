# Edge Launcher [![Linux Build Status](https://img.shields.io/travis/cezaraugusto/edge-launcher/master.svg)](https://travis-ci.org/cezaraugusto/edge-launcher) [![Windows Build Status](https://img.shields.io/appveyor/ci/paulirish/edge-launcher/master.svg)](https://ci.appveyor.com/project/paulirish/edge-launcher/branch/master) [![NPM edge-launcher package](https://img.shields.io/npm/v/edge-launcher.svg)](https://npmjs.org/package/edge-launcher)


<img src="https://user-images.githubusercontent.com/39191/29847271-a7ba82f8-8ccf-11e7-8d54-eb88fdf0b6d0.png" align=right height=200>

Launch Google Edge with ease from node.

* [Disables many Edge services](https://github.com/cezaraugusto/edge-launcher/blob/master/src/flags.ts) that add noise to automated scenarios
* Opens up the browser's `remote-debugging-port` on an available port
* Automagically locates a Edge binary to launch
* Uses a fresh Edge profile for each launch, and cleans itself up on `kill()`
* Binds `Ctrl-C` (by default) to terminate the Edge process
* Exposes a small set of [options](#api) for configurability over these details

Once launched, interacting with the browser must be done over the [devtools protocol](https://chromedevtools.github.io/devtools-protocol/), typically via [edge-remote-interface](https://github.com/cyrus-and/edge-remote-interface/). For many cases [Puppeteer](https://github.com/cezaraugusto/puppeteer) is recommended, though it has its own edge launching mechanism.

### Installing

```sh
yarn add edge-launcher

# or with npm:
npm install edge-launcher
```


## API

### `.launch([opts])`

#### Launch options

```js
{
  // (optional) remote debugging port number to use. If provided port is already busy, launch() will reject
  // Default: an available port is autoselected
  port: number;

  // (optional) Additional flags to pass to Edge, for example: ['--headless', '--disable-gpu']
  // See: https://github.com/cezaraugusto/edge-launcher/blob/master/docs/edge-flags-for-tools.md
  // Do note, many flags are set by default: https://github.com/cezaraugusto/edge-launcher/blob/master/src/flags.ts
  chromeFlags: Array<string>;

  // (optional) Close the Edge process on `Ctrl-C`
  // Default: true
  handleSIGINT: boolean;

  // (optional) Explicit path of intended Edge binary
  // * If this `chromePath` option is defined, it will be used.
  // * Otherwise, the `CHROME_PATH` env variable will be used if set. (`LIGHTHOUSE_CHROMIUM_PATH` is deprecated)
  // * Otherwise, a detected Edge Canary will be used if found
  // * Otherwise, a detected Edge (stable) will be used
  chromePath: string;

  // (optional) Edge profile path to use, if set to `false` then the default profile will be used.
  // By default, a fresh Edge profile will be created
  userDataDir: string | boolean;

  // (optional) Starting URL to open the browser with
  // Default: `about:blank`
  startingUrl: string;

  // (optional) Logging level
  // Default: 'silent'
  logLevel: 'verbose'|'info'|'error'|'silent';

  // (optional) Flags specific in [flags.ts](src/flags.ts) will not be included.
  // Typically used with the defaultFlags() method and chromeFlags option.
  // Default: false
  ignoreDefaultFlags: boolean;

  // (optional) Interval in ms, which defines how often launcher checks browser port to be ready.
  // Default: 500
  connectionPollInterval: number;

  // (optional) A number of retries, before browser launch considered unsuccessful.
  // Default: 50
  maxConnectionRetries: number;

  // (optional) A dict of environmental key value pairs to pass to the spawned edge process.
  envVars: {[key: string]: string};
};
```

#### Launched edge interface

#### `.launch().then(edge => ...`

```js
// The remote debugging port exposed by the launched edge
edge.port: number;

// Method to kill Edge (and cleanup the profile folder)
edge.kill: () => Promise<void>;

// The process id
edge.pid: number;

// The childProcess object for the launched Edge
edge.process: childProcess
```

### `ChromeLauncher.Launcher.defaultFlags()`

Returns an `Array<string>` of the default [flags](docs/edge-flags-for-tools.md) Edge is launched with. Typically used along with the `ignoreDefaultFlags` and `chromeFlags` options.

Note: This array will exclude the following flags: `--remote-debugging-port` `--disable-setuid-sandbox` `--user-data-dir`.

### `ChromeLauncher.Launcher.getInstallations()`

Returns an `Array<string>` of paths to available Edge installations. When `chromePath` is not provided to `.launch()`, the first installation returned from this method is used instead.

Note: This method performs synchronous I/O operations.

### `.killAll()`

Attempts to kill all Edge instances created with [`.launch([opts])`](#launchopts). Returns a Promise that resolves to an array of errors that occurred while killing instances. If all instances were killed successfully, the array will be empty.

```js
const ChromeLauncher = require('edge-launcher');

async function cleanup() {
  await ChromeLauncher.killAll();
}
```

## Examples

#### Launching edge:

```js
const ChromeLauncher = require('edge-launcher');

ChromeLauncher.launch({
  startingUrl: 'https://google.com'
}).then(edge => {
  console.log(`Edge debugging port running on ${edge.port}`);
});
```


#### Launching headless edge:

```js
const ChromeLauncher = require('edge-launcher');

ChromeLauncher.launch({
  startingUrl: 'https://google.com',
  chromeFlags: ['--headless', '--disable-gpu']
}).then(edge => {
  console.log(`Edge debugging port running on ${edge.port}`);
});
```

#### Launching with support for extensions and audio:

```js
const ChromeLauncher = require('edge-launcher');

const newFlags = ChromeLauncher.Launcher.defaultFlags().filter(flag => flag !== '--disable-extensions' && flag !== '--mute-audio');

ChromeLauncher.launch({
  ignoreDefaultFlags: true,
  chromeFlags: newFlags,
}).then(edge => { ... });
```

### Continuous Integration

In a CI environment like Travis, Edge may not be installed. If you want to use `edge-launcher`, Travis can [install Edge at run time with an addon](https://docs.travis-ci.com/user/edge).  Alternatively, you can also install Edge using the [`download-edge.sh`](https://raw.githubusercontent.com/cezaraugusto/edge-launcher/v0.8.0/scripts/download-edge.sh) script.

Then in `.travis.yml`, use it like so:

```yaml
language: node_js
install:
  - yarn install
before_script:
  - export DISPLAY=:99.0
  - export CHROME_PATH="$(pwd)/edge-linux/edge"
  - sh -e /etc/init.d/xvfb start
  - sleep 3 # wait for xvfb to boot

addons:
  edge: stable
```

### Acknowledgements

This project started as a fork of, and is inspired by https://github.com/cezaraugusto/edge-launcher which is released under the Apache-2.0 License and is copyright of Google Inc.
