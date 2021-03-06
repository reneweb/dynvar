# dvar

dvar is a configuration management library.
It currently supports the following to provide the configuration:

- env (reading from process.env)
- file (reading from file)
- provided (provided in the code as json)
- http (getting from url)

* * *

### Quick Links
* [Install](#install)
* [Usage](#usage)
  * [Quick start](#quick-start)
  * [Multiple config's](#multiple-configs)
* [Options](#options)
* [Events](#events)
* [Types](#types)
  * [Provided](#provided)
  * [File](#file)
  * [Env](#env)
  * [Http](#http)
* [Environment specific config](#environment-specific-config)
* [Dynamic reloading](#dynamic-reloading)
* [Override](#override)
* [Extensions](#extensions)

### Install

`npm install dvar --save`

### Usage

#### Quick start
```javascript
const dvar = require('dvar')
dvar.configure([
    {type: 'provided', variables: {test: 123}}
  ], (err, res) => {
    console.log(res.get('test')) //<- prints 123
    console.log(res.all()['test']) //<- prints 123
})
```

#### Multiple config's

dvar takes an array of config's which makes it possible to provide config from multiple sources.
```javascript
const dvar = require('dvar')

process.env.testEnv = 456

dvar.configure([
  {type: 'provided', variables: {test: 123}},
  {type: 'env'},
  {type: 'provided', variables: {anotherOne: 'someString'}}
], (err, res) => {
  console.log(res.get('test')) //<- prints 123
  console.log(res.get('testEnv')) //<- prints 456
  console.log(res.get('anotherOne')) //<- prints 'someString'
})
```

When the same variable is provided in different sources it will be overridden based on the array order (higher index will override lower).
```javascript
const dvar = require('dvar')

dvar.configure([
  {type: 'provided', variables: {override: 'firstString'}},
  {type: 'provided', variables: {override: 'secondString'}}
], (err, res) => {
  console.log(res.get('override')) //<- prints 'secondString'
})
```

#### Options
It is possible to provide a options object as the second parameter to the `config` function. If not provided defaults will be applied.
The currently supported options are:
- replaceKeys: Takes a function and will apply it for every key, replacing it with a new key. The key will be passed as a parameter to the function and the new key must be returned.
- replaceValues: Same as replaceKeys for values.
- env: The current environment as a string or a function that returns the environment. Only used when providing config specific for certain environments as described [here](#environment-specific-config)
- dynamic: Setting this option to a truthy value will enable dynamic reloading of the configuration. See more [here](#dynamic-reloading)

Example:

```javascript
{
  replaceKeys: key => key.replace('-D', ''),
  replaceValues: values => String(value),
}
```

#### Events
The following events are emitted:
- `loaded` - Called when the config is finished loading. Passes the config values as a parameter.
- `error` - Called when there is an error reading from a provider. Passes the error as a parameter.
- `updated` - Called when the config is updated (i.e. when the dynamic reload interval passed). Passes the new config values as a parameter. This event is only fired when using dynamic reloading and will be called every time the config is reloaded no matter if the config actually changed from the reload.

Example `loaded` event:
```javascript
const dvar = require('dvar')
dvar.once('loaded', values => {
  console.log(values.test) //<- prints 123
})
dvar.configure([{type: 'provided', variables: {test: 123}}])
```

Example `error` event:
```javascript
const dvar = require('dvar')
dvar.once('error', err => {
  //...
})
dvar.configure([{type: 'file', path: 'does-not-exist'}])
```

Example `updated` event:
```javascript
const dvar = require('dvar')
const fs = require('fs')

fs.writeFileSync('./testFile', 'testKey=testValue')

dvar.once('updated', values => {
  console.log(values.testKey)  //<- prints dynamicallyChangedtestValue
})

dvar.configure([
  {type: 'file', format: 'property', path: './testFile'}
], {
  dynamic: {
    interval: 10
  }
}, (err, res) => {
  fs.writeFileSync('./testFile', 'testKey=dynamicallyChangedtestValue')
})
```

#### Types

##### Provided
```javascript
const dvar = require('dvar')

dvar.configure([
  {type: 'provided', variables: {test: 123}}
], (err, res) => {
  console.log(res.get('test')) //<- prints 123
})
```

##### File
Currently there are two formats supported:
- json
- property (`key=value` separated by new line)

```javascript
const dvar = require('dvar')
const fs = require('fs')

fs.writeFileSync('./app.env', `testKey=testValue`)

dvar.configure([
  {type: 'file', format: 'property', path: './app.env'}
], (err, res) => {
  console.log(res.get('testKey')) //<- prints 'testValue'
})
```

##### Env
```javascript
const dvar = require('dvar')

process.env.testEnv = 456

dvar.configure([
  {type: 'env'}
], (err, res) => {
  console.log(res.get('testEnv')) //<- prints 456
})
```

##### HTTP
Currently there are two supported formats, that can be passed in the http response:
- json
- property (`key=value` separated by new line)

```javascript
const dvar = require('dvar')
const http = require('http')

http.createServer(function(request, response) {
  response.end(`testKey=testValue`)
}).listen(8080)

dvar.configure([
  {type: 'http', format: 'property', url: 'http://localhost:8080/test'}
], (err, res) => {
  console.log(res.get('testKey')) //<- prints 'testValue'
})
```

#### Environment specific config

Instead of providing an array to the `config` function, it is also possible to pass an object to it. The key(s) should be the environment for which the config is and the value the actual config array. Further, it is possible to provide an option `env` that is used to determine the current environment that should be used. It can be either a string or a function that returns a string with the environment.

```javascript
dvar.configure({
  staging: [{type: 'provided', variables: {staging: 123}}],
  production: [{type: 'provided', variables: {prod: 123}}]
}, {
  env: () => {
    return 'staging'
  }
}, (err, res) => {
  console.log(res.get('staging')) //<- prints 123
})
```

#### Dynamic reloading

Dynamic reloading of the configuration can be enabled by setting the `dynamic` option to a truthy value. By default it will poll for configuration changes every minute. When setting the `dynamic` field to a object, an `interval` option (in ms) can be passed to it.

```javascript
const dvar = require('dvar')
const fs = require('fs')

fs.writeFileSync('./testFile', 'testKey=testValue')

dvar.configure([
  {type: 'file', format: 'property', path: './testFile'}
], {
  dynamic: {
    interval: 10 //in ms
  }
}, (err, res) => {
  console.log(res.get('testKey')) //<- prints testValue

  fs.writeFileSync('./testFile', 'testKey=dynamicallyChangedTestValue')

  setTimeout(() => {
    console.log(res.get('testKey')) //<- prints dynamicallyChangedTestValue
  }, 12)
})
```

#### Override
Override is a function that takes a config the same way as `configure`. When using that function all subsequent calls to `configure` will use the configuration defined in the `override` function instead of it's own. This is useful for testing, when a specific test configuration is needed. Calls to `configure` before the `override` function is called won't be affected.
```javascript
const dvar = require('dvar')

dvar.override([
  {type: 'provided', variables: {test: 123}}
], (err, res) => {
  console.log(res.get('test')) //<- prints 123

  dvar.configure([
    {type: 'provided', variables: {test: 456}}
  ], (err, res) => {
    console.log(res.get('test')) //<- prints 123
  })
})
```

To remove a override config there is a function `removeOverride`. This will cause all subsequent calls to `configure` to use their own configuration instead of the one provided by `override`. The `configure` calls done before the `removeOverride` will still use the override configuration though.

#### Extensions
Additional providers can be added using extensions.

```javascript
const dvar = require('dvar')

dvar.addExtension('myExtension', {init: (config, cb) => {
  const readF = cb => cb(null, {test: 123})
  cb(null, {read: readF})
}}).configure([
  {type: 'myExtension'}
], (err, res) => {
  console.log(res.get('test')) //<- prints 123
})
```

As you can see in the example the `addExtension` function takes the name of the extension and the actual extension / provider.
The extension must have a `init` function that takes the config (As defined in the `configure` call) and a callback function. On success the callback function must return another function `read` that will read the configuration.
