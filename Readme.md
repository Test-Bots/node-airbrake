# Airbrake

[![Circle CI](https://circleci.com/gh/airbrake/node-airbrake.svg?style=shield)](https://circleci.com/gh/airbrake/node-airbrake)

<img src="http://f.cl.ly/items/1t0F40132D243k1D0d0g/nodejs.jpg" width=800px>

Node.js client for [airbrake.io](https://airbrake.io).

## Install

``` bash
npm install airbrake
```

## Basic usage

The common use case for this module is to catch all `'uncaughtException'`
events on the `process` object and send them to Airbrake:

``` javascript
var airbrake = require('airbrake').createClient("your project ID", "your api key");
airbrake.handleExceptions();

throw new Error('I am an uncaught exception');
```

Please note that the above will re-throw the exception after it has been
successfully delivered to Airbrake, causing your process to exit with status 1.

This can optionally be disabled by passing false to `handleExceptions`:

``` javascript
airbrake.handleExceptions(false);
```

You probably never want to use this, unless you fully understand
the problems with recovering from exceptions.

If you want more control over the delivery of your errors, you can also
manually submit errors to Airbrake.

``` javascript
var airbrake = require('airbrake').createClient("your project ID", "your api key");
var err = new Error('Something went terribly wrong');
airbrake.notify(err, function(err, url) {
  if (err) throw err;

  // Error has been delivered, url links to the error in airbrake
});
```
### Usage with Express

A custom error handler will need to be set for Express:

Express 4.X
```javascript
var airbrake = require('airbrake').createClient("your project ID", "your api key");
app.use(airbrake.expressHandler());
```

Express 3.X
``` javascript
var airbrake = require('airbrake').createClient("your project ID", "your api key");
app.use(app.router);
app.use(airbrake.expressHandler());
```

Express 2.X
``` javascript
var airbrake = require('airbrake').createClient("your project ID", "your api key");
app.error(airbrake.expressHandler());
```

## Screenshot

This screenshot shows an Airbrake error send from this module:

![screenshot](https://github.com/felixge/node-airbrake/raw/master/screenshot.png)

## Features

* Send chosen environment variables (whitelist or blacklist)
* Detect and fix circular references in error context information
* Support for all features of the [2.1 notification API][2.1api]
* Support for [long-stack-traces][]
* Optional auto-handler for `uncaughtException` events
* Provides notification url linking to Airbrake in `notify()` callback
* Timeout Airbrake requests after 30 seconds, you never know

[long-stack-traces]: https://github.com/tlrobinson/long-stack-traces

[2.1api]: http://help.airbrake.io/kb/api-2/notifier-api-version-21

## Adding context to errors

The `notify()` method automatically adds the following context information to
each delivered error:

* **error.class:** (`err.type` string if set, or `'Error'`)
* **error.message:** (`err.message` string)
* **error.backtrace:** (`err.stack` as parsed by [stack-trace][])
* **request.url:** (`err.url`, see `airbrake.url`);
* **request.component:** (`err.component` string if set);
* **request.action:** (`err.action` string if set);
* **request.cgi-data:** (`process.env`, merged all other properties of `err`)
* **request.params:** (`err.params` object if set)
* **request.session:** (`err.session` object if set)
* **server-environment.project-root:** (`airbrake.projectRoot` string if set)
* **server-environment.environment-name:** (`airbrake.env` string)
* **server-environment.app-version:** (`airbrake.appVersion string if set)

You can add additional context information by modifying the error properties
listed above:

``` javascript
var airbrake = require('airbrake').createClient("your project ID", "your api key");
var http = require('http');

http.createServer(function(req, res) {
  if (req.headers['X-Secret'] !== 'my secret') {
    var err = new Error('403 - Permission denied');
    req.writeHead(403);
    req.end(err.message);

    err.url = req.url;
    err.params = {ip: req.socket.remoteAddress};
    airbrake.notify(err);
  }
});
```

Unfortunately `uncaughtException` events cannot be traced back to particular
requests, so you should still try to handle errors where they occur.

[stack-trace]: https://github.com/felixge/node-stack-trace

## Tracking deployments

This client supports Airbrake's [deployment tracking][]:

``` javascript
var airbrake = require('airbrake').createClient("your project ID", "your api key");
var deployment = {
  rev: '98103a8fa850d5eaf3666e419d8a0a93e535b1b2',
  repo: 'git@github.com:felixge/node-airbrake.git',
};

airbrake.trackDeployment(deployment, function(err, params) {
  if (err) {
    throw err;
  }

  console.log('Tracked deployment of %s to %s', params.rev, params.env);
});
```

Check out the `airbrake.trackDeployment()` API docs below for a list of all
options.

[deployment tracking]: http://help.airbrake.io/kb/api-2/deploy-tracking

## API

### var airbrake = Airbrake.createClient(projectId, key, [env])

`Airbrake.createClient()` returns a new Airbrake instance.

Options
* `projectId` - Your application's Airbrake project ID.
* `key` - Your application's Airbrake API key.
* `env` - The name of the server environment this is running in.

### airbrake.projectId = null

Your application's Airbrake project ID.

### airbrake.key = null

Your application's Airbrake API key.

### airbrake.env = process.env.NODE_ENV || 'development'

The name of the server environment this is running in.

### airbrake.host = 'https://' + os.hostname()

The base url for errors. If `err.url` is not set, `airbrake.host` is used
instead. If `err.url` is a relative url starting with `'/'`, it is appended
to `airbrake.host`. If `err.url` is an absolute url, `airbrake.host` is ignored.

### airbrake.projectRoot = process.cwd()

The root directory of this project.

### airbrake.appVersion = null

The version of this app. Set to a semantic version number, or leave unset.

### airbrake.protocol = 'https'

The protocol to use.

### airbrake.developmentEnvironments = []

Do not post to Airbrake when running in these environments.

### airbrake.timeout = 30 * 1000

The timeout after which to give up trying to notify Airbrake in ms.

### airbrake.proxy = null

The HTTP/HTTPS proxy to use when making requests.

### airbrake.requestOptions = {}

Additional request options that are merged with the default set of options that are passed to `request` during `notify()` and `trackDeployment()`.

### airbrake.whiteListKeys = []

Names of environment variables to send.

### airbrake.blackListKeys = []

Names of environment variables to filter out.

### airbrake.handleExceptions()

Registers a `process.on('uncaughtException')` listener. When an uncaught
exception occurs, the error is sent to Airbrake, and then re-thrown to
kill the process.

### airbrake.expressHandler(disableUncaughtException)

A custom error handler that is used with Express. Integrate with Express
middleware using `app.use()`.

Options:
* `disableUncaughtException`: Disables re-throwing and killing process on uncaught exception.

### airbrake.notify(err, [cb])

Sends the given `err` to airbrake.

The callback parameter receives two arguments, `err, url`. `err` is set if
the delivery to Airbrake failed.

If no `cb` is given, and the delivery fails, an `error` event is emitted. If
there is no listener for this event, node will kill the process as well. This
is done to avoid silent error delivery failure.

### airbrake.trackDeployment([params, [cb]])

Notifies Airbrake about a deployment. `params` is an object with the following
options:

* `env:` The environment being deployed, defaults to `airbrake.env`.
* `user:` The user doing the deployment, defaults to `process.env.USER`.
* `repo:` The github url of this repository. Defaults to `''`.
* `rev:` The revision of this deployment. Defaults to `''`.

## Contribute

Besides bug fixes, we're happy to accept patches for:

* Automatically parsing `repo` and `rev` from the local git repository when
  calling `airbrake.trackDeployment()`. This can be done via `exec()`, but must
  not be done when specifying `repo` / `rev` by hand, or if they are set to
  `false`.

If you have other feature ideas, please open an issue first, so we can discuss
it.

## Contributors

Originally created by [Felix Geisendörfer](https://github.com/felixge).

See all [contributors](https://github.com/airbrake/node-airbrake/graphs/contributors).

## License

MIT
