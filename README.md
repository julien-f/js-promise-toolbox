# promise-toolbox [![Build Status](https://travis-ci.org/JsCommunity/promise-toolbox.png?branch=master)](https://travis-ci.org/JsCommunity/promise-toolbox)

> Essential utils for promises.

**Features:**

- compatible with all promise implementations
- small (< 150 KB with all dependencies, < 5 KB with gzip)
- nice with ES2015 / ES2016 syntax

**Table of contents:**

- [Cancelation](#cancelation)
  - [Creation](#creation)
  - [Consumption](#consumption)
  - [Is cancel token?](#is-cancel-token)
  - [@cancelable decorator](#cancelable-decorator)
- [Resource management](#resource-management)
- [Functions](#functions)
  - [asyncFn(generator)](#asyncfngenerator)
  - [asyncFn.cancelable(generator)](#asyncfncancelablegenerator)
  - [defer()](#defer)
  - [fromCallback(cb => fn(arg1, ..., argn, cb))](#fromcallbackcb--fnarg1--argn-cb)
  - [fromEvent(emitter, event, [options]) => Promise](#fromeventemitter-event-options--promise)
  - [fromEvents(emitter, successEvents, errorEvents) => Promise](#fromeventsemitter-successevents-errorevents--promise)
  - [isPromise(value)](#ispromisevalue)
  - [promisify(fn, [ context ]) / promisifyAll(obj)](#promisifyfn--context---promisifyallobj)
  - [try(fn)](#tryfn)
  - [wrapApply(fn, args, [thisArg]) / wrapCall(fn, arg, [thisArg])](#wrapapplyfn-args-thisarg--wrapcallfn-arg-thisarg)
- [Pseudo-methods](#pseudo-methods)
  - [promises::all([ mapper ])](#promisesall-mapper-)
  - [promise::asCallback(cb)](#promiseascallbackcb)
  - [promise::catch(predicate, cb)](#promisecatchpredicate-cb)
  - [promise::delay(ms, [value])](#promisedelayms-value)
  - [collection::forEach(cb)](#collectionforeachcb)
  - [promise::ignoreErrors()](#promiseignoreerrors)
  - [promise::finally(cb)](#promisefinallycb)
  - [promise::reflect()](#promisereflect)
  - [promises::some(count)](#promisessomecount)
  - [promise::suppressUnhandledRejections()](#promisesuppressunhandledrejections)
  - [promise::tap(onResolved, onRejected)](#promisetaponresolved-onrejected)
  - [promise::tapCatch(onRejected)](#promisetapcatchonrejected)
  - [promise::timeout(ms, [cb])](#promisetimeoutms-cb-or-rejectionvalue)

### Node & [Browserify](http://browserify.org/)/[Webpack](https://webpack.js.org/)

Installation of the [npm package](https://npmjs.org/package/promise-toolbox):

```
> npm install --save promise-toolbox
```

### Browser

You can directly use the build provided at [unpkg.com](https://unpkg.com):

```html
<script src="https://unpkg.com/promise-toolbox@0.8/dist/umd.js"></script>
```

## Usage

### Promise support

If your environment may not natively support promises, you should use a polyfill such as [native-promise-only](https://github.com/getify/native-promise-only).

On Node, if you want to use a specific promise implementation,
[Bluebird](http://bluebirdjs.com/docs/why-bluebird.html) for instance
to have better performance, you can override the global Promise
variable:

```js
global.Promise = require('bluebird')
```

> Note that it should only be done at the application level, never in
> a library!

### Imports

You can either import all the tools directly:

```js
import * as PT from 'promise-toolbox'

console.log(PT.isPromise(value))
```

Or import individual tools from the main module:

```js
import { isPromise } from 'promise-toolbox'

console.log(isPromise(value))
```

Each tool is also exported with a `p` prefix to work around reserved keywords
and to help differentiate with other tools (like `lodash.map`):

```js
import { pCatch, pMap } from 'promise-toolbox'
```

If you are bundling your application (Browserify, Rollup, Webpack, etc.), you
can cherry-pick the tools directly:

```js
import isPromise from 'promise-toolbox/isPromise'
import pCatch from 'promise-toolbox/catch'
```

## API

### Cancelation

This library provides an implementation of `CancelToken` from the
[cancelable promises specification](https://tc39.github.io/proposal-cancelable-promises/).

A cancel token is an object which can be passed to asynchronous
functions to represent cancelation state.

```js
import { CancelToken } from 'promise-toolbox'
```

#### Creation

A cancel token is created by the initiator of the async work and its
cancelation state may be requested at any time.

```js
// Create a token which requests cancelation when a button is clicked.
const token = new CancelToken(cancel => {
  $('#some-button').on('click', () => cancel('button clicked'))
})
```

```js
const { cancel, token } = CancelToken.source()
```

A list of existing tokens can be passed to `source()` to make the created token
follow their cancelation:

```js
// `source.token` will be canceled (synchronously) as soon as `token1` or
// `token2` or token3` is, with the same reason.
const { cancel, token } = CancelToken.source([token1, token2, token3])
```

#### Consumption

The receiver of the token (the function doing the async work) can:

1. synchronously check whether cancelation has been requested
2. synchronously throw if cancelation has been requested
3. register a callback that will be executed if cancelation is requested
4. pass the token to subtasks

```js
// 1.
if (token.reason) {
  console.log('cancelation has been requested', token.reason.message)
}

// 2.
try {
  token.throwIfRequested()
} catch (reason) {
  console.log('cancelation has been requested', reason.message)
}

// 3.
token.promise.then(reason => {
  console.log('cancelation has been requested', reason.message)
})

// 4.
subtask(token)
```

#### Registering async handlers

> Asynchronous handlers are executed on token cancelation and the
> promise returned by the `cancel` function will wait for all handlers
> to settle.

```js
function httpRequest (cancelToken, opts) {
  const req = http.request(opts)
  req.end()
  cancelToken.addHandler(() => {
    req.abort()

    // waits for the socket to really close for the cancelation to be
    // complete
    return fromEvent(req, 'close')
  })
  return fromEvent(req, 'response')
}

const { cancel, token } = CancelToken.source()

httpRequest(token, {
  hostname: 'example.org',
}).then(response => {
  // do something with the response of the request
})

// wraps with Promise.resolve() because cancel only returns a promise
// if a handler has returned a promise
Promise.resolve(cancel()).then(() => {
  // the request has been properly canceled
})
```

#### Is cancel token?

```js
if (CancelToken.isCancelToken(value)) {
  console.log('value is a cancel token')
}
```

#### @cancelable decorator

> Make your async functions cancelable.

If the first argument passed to the cancelable function is not a
cancel token, a new one is created and injected and the returned
promise will have a `cancel()` method.

```js
import { cancelable, CancelToken } from 'promise-toolbox'

const asyncFunction = cancelable(async ($cancelToken, a, b) => {
  $cancelToken.promise.then(() => {
    // do stuff regarding the cancelation request.
  })

  // do other stuff.
})

// Either a cancel token is passed:
const source = CancelToken.source()
const promise1 = asyncFunction(source.token, 'foo', 'bar')
source.cancel('reason')

// Or the returned promise will have a cancel() method:
const promise2 = asyncFunction('foo', 'bar')
promise2.cancel('reason')
```

If the function is a method of a class or an object, you can use
`cancelable` as a decorator:

```js
class MyClass {
  @cancelable
  async asyncMethod ($cancelToken, a, b) {
    // ...
  }
}
```

### Resource management

> See [Bluebird documentation](http://bluebirdjs.com/docs/api/resource-management.html) for a good explanation.

```js
import { disposer, using } from 'promise-toolbox'

const getConnection = () =>
  // disposer() is used to associate a disposer to a resource
  //
  // The returned resource can only be used with using()
  db.connect()::disposer(connection =>
    connection.close()
  )

using(getConnection(), getConnection(), (connection1, connection2) => {
  // So something with connection1 and connection2
})).then(() => {
  // Both connections are now closed
})
```

### Functions

#### asyncFn(generator)

> Create an [async function](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Statements/async_function) from [a generator function](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Statements/function*)
>
> Similar to [`Bluebird.coroutine`](http://bluebirdjs.com/docs/api/promise.coroutine.html).

```js
import { asyncFn } from 'promise-toolbox'

const getUserName = asyncFn(function * (db, userId)) {
  const user = yield db.getRecord(userId)
  return user.name
})
```

#### asyncFn.cancelable(generator)

> Like [`asyncFn(generator)`](#asyncfngenerator) but the created async function supports [cancelation](#cancelation).
>
> Similar to [CAF](https://github.com/getify/CAF).

```js
import { asyncFn, CancelToken } from 'promise-toolbox'

const getUserName = asyncFn.cancelable(function * (cancelToken, db, userId)) {
  // this yield will throw if the cancelToken is activated
  const user = yield db.getRecord(userId)
  return user.name
})

const source = CancelToken.source()

getUserName(source.token, db, userId).then(
  name => {
    console.log('user name is', name)
  },
  error => {
    console.error(error)
  }
)

// only wait 5 seconds to fetch the user from the database
setTimeout(source.cancel, 5e3)
```

#### defer()

> Discouraged but sometimes necessary way to create a promise.

```js
import { defer } from 'promise-toolbox'

const { promise, resolve } = defer()

promise.then(value => {
  console.log(value)
})

resolve(3)
```

#### fromCallback(cb => fn(arg1, ..., argn, cb))

> Easiest and most efficient way to promisify a function call.

```js
import { fromCallback } from 'promise-toolbox'

fromCallback(cb => fs.readFile('foo.txt', cb)).then(content => {
  console.log(content)
})
```

#### fromEvent(emitter, event, [options]) => Promise

> Wait for one event. The first parameter of the emitted event is used
> to resolve/reject the promise.

```js
const promise = fromEvent(emitter, 'foo', {
  // whether the promise resolves to an array of all the event args
  // instead of simply the first arg
  array: false,

  // whether the error event can reject the promise
  ignoreErrors: false,

  // name of the error event
  error: 'error',
})

promise.then(
  value => {
    console.log('foo event was emitted with value', value)
  },
  reason => {
    console.error('an error has been emitted', reason)
  }
)
```

#### fromEvents(emitter, successEvents, errorEvents) => Promise

> Wait for one of multiple events. The array of all the parameters of
> the emitted event is used to resolve/reject the promise.
>
> The array also has an `event` property indicating which event has
> been emitted.

```js
fromEvents(emitter, ['foo', 'bar'], ['error1', 'error2']).then(
  values => {
    console.log('event %s have been emitted with values', values.event, values)
  },
  reasons => {
    console.error(
      'error event %s has been emitted with errors',
      reasons.event,
      reasons
    )
  }
)
```

#### isPromise(value)

```js
import { isPromise } from 'promise-toolbox'

if (isPromise(foo())) {
  console.log('foo() returns a promise')
}
```

#### promisify(fn, [ context ]) / promisifyAll(obj)

> Creates async functions taking node-style callbacks, create new ones
> returning promises.

```js
import fs from 'fs'
import { promisify, promisifyAll } from 'promise-toolbox'

// Promisify a single function.
//
// If possible, the function name is kept and the new length is set.
const readFile = promisify(fs.readFile)

// Or all functions (own or inherited) exposed on a object.
const fsPromise = promisifyAll(fs)

readFile(__filename).then(content => console.log(content))

fsPromise.readFile(__filename).then(content => console.log(content))
```

#### try(fn)

> Starts a chain of promises.

```js
import PromiseToolbox from 'promise-toolbox'

const getUserById = id =>
  PromiseToolbox.try(() => {
    if (typeof id !== 'number') {
      throw new Error('id must be a number')
    }
    return db.getUserById(id)
  })
```

> Note: similar to `Promise.resolve().then(fn)` but calls `fn()`
> synchronously.

#### wrapApply(fn, args, [thisArg]) / wrapCall(fn, arg, [thisArg])

> Wrap a call to a function to always return a promise.

```js
function getUserById (id) {
  if (typeof id !== 'number') {
    throw new TypeError('id must be a number')
  }
  return db.getUser(id)
}

wrapCall(getUserById, 'foo').catch(error => {
  // id must be a number
})
```

### Pseudo-methods

This function can be used as if they were methods, i.e. by passing the
promise (or promises) as the context.

This is extremely easy using [ES2016's bind syntax](https://github.com/zenparsing/es-function-bind).

```js
const promises = [Promise.resolve('foo'), Promise.resolve('bar')]

promises::all().then(values => {
  console.log(values)
})
// → [ 'foo', 'bar' ]
```

If you are still an older version of ECMAScript, fear not: simply pass
the promise (or promises) as the first argument of the `.call()`
method:

```js
const promises = [Promise.resolve('foo'), Promise.resolve('bar')]

all.call(promises).then(function(values) {
  console.log(values)
})
// → [ 'foo', 'bar' ]
```

#### promises::all([ mapper ])

> Waits for all promises of a collection to be resolved.
>
> Contrary to the standard `Promise.all()`, this function works also
> with objects.

```js
import { all } from 'promise-toolbox'

[
  Promise.resolve('foo'),
  Promise.resolve('bar')
]::all().then(value => {
  console.log(value)
  // → ['foo', 'bar']
})

{
  foo: Promise.resolve('foo'),
  bar: Promise.resolve('bar')
}::all().then(value => {
  console.log(value)
  // → {
  //   foo: 'foo',
  //   bar: 'bar'
  // }
})
```

#### promise::asCallback(cb)

> Register a node-style callback on this promise.

```js
import { asCallback } from 'promise-toolbox'

// This function can be used either with node-style callbacks or with
// promises.
function getDataFor (input, callback) {
  return dataFromDataBase(input)::asCallback(callback)
}
```

#### promise::catch(predicate, cb)

> Similar to `Promise#catch()` but:
>
> - support predicates
> - do not catch `ReferenceError`, `SyntaxError` or `TypeError` unless
>   they match a predicate because they are usually programmer errors
>   and should be handled separately.

```js
somePromise
  .then(() => {
    return a.b.c.d()
  })
  ::pCatch(TypeError, ReferenceError, reason => {
    // Will end up here on programmer error
  })
  ::pCatch(NetworkError, TimeoutError, reason => {
    // Will end up here on expected everyday network errors
  })
  ::pCatch(reason => {
    // Catch any unexpected errors
  })
```

#### promise::delay(ms, [value])

> Delays the resolution of a promise by `ms` milliseconds.
>
> Note: the rejection is not delayed.

```js
console.log(await Promise.resolve('500ms passed')::delay(500))
// → 500 ms passed
```

Also works with a value:

```js
console.log(await delay(500, '500ms passed'))
// → 500 ms passed
```

#### collection::forEach(cb)

> Iterates in order over a collection, or promise of collection, which
> contains a mix of promises and values, waiting for each call of cb
> to be resolved before the next one.

The returned promise will resolve to `undefined` when the iteration is
complete.

```js
['foo', Promise.resolve('bar')]::forEach(value => {
  console.log(value)

  // Wait for the promise to be resolve before the next item.
  return new Promise(resolve => setTimeout(resolve, 10))
})
// →
// foo
// bar
```

#### promise::ignoreErrors()

> Ignore (operational) errors for this promise.

```js
import { ignoreErrors } from 'promise-toolbox'

// will not emit an unhandled rejection error if the file does not
// exist
readFileAsync('foo.txt')
  .then(content => {
    console.log(content)
  })
  ::ignoreErrors()

// will emit an unhandled rejection error due to the typo
readFileAsync('foo.txt')
  .then(content => {
    console.lgo(content) // typo
  })
  ::ignoreErrors()
```

#### promise::finally(cb)

> Execute a handler regardless of the promise fate. Similar to the
> `finally` block in synchronous codes.
>
> The resolution value or rejection reason of the initial promise is
> forwarded unless the callback rejects.

```js
import { pFinally } from 'promise-toolbox'

function ajaxGetAsync (url) {
  return new Promise((resolve, reject) => {
    const xhr = new XMLHttpRequest()
    xhr.addEventListener('error', reject)
    xhr.addEventListener('load', resolve)
    xhr.open('GET', url)
    xhr.send(null)
  })::pFinally(() => {
    $('#ajax-loader-animation').hide()
  })
}
```

#### promise::reflect()

> Returns a promise which resolves to an objects which reflects the
> resolution of this promise.

```js
import { reflect } from 'promise-toolbox'

const inspection = await promise::reflect()

if (inspection.isFulfilled()) {
  console.log(inspection.value())
} else {
  console.error(inspection.reason())
}
```

#### promises::some(count)

> Waits for `count` promises in a collection to be resolved.

```js
import { some } from 'promise-toolbox'

const [first, seconds] = await [
  ping('ns1.example.org'),
  ping('ns2.example.org'),
  ping('ns3.example.org'),
  ping('ns4.example.org'),
]::some(2)
```

#### promise::suppressUnhandledRejections()

> Suppress unhandled rejections, needed when error handlers are attached
> asynchronously after the promise has rejected.
>
> Similar to [`Bluebird#suppressUnhandledRejections()`](http://bluebirdjs.com/docs/api/suppressunhandledrejections.html).

```js
const promise = getUser()::suppressUnhandledRejections()
$(document).on('ready', () => {
  promise.catch(error => {
     console.error('error while getting user', error)
  })
})
```

#### promise::tap(onResolved, onRejected)

> Like `.then()` but the original resolution/rejection is forwarded.
>
> Like `::finally()`, if the callback rejects, it takes over the
> original resolution/rejection.

```js
import { tap } from 'promise-toolbox'

// Contrary to .then(), using ::tap() does not change the resolution
// value.
const promise1 = Promise.resolve(42)::tap(value => {
  console.log(value)
})

// Like .then, the second param is used in case of rejection.
const promise2 = Promise.reject(42)::tap(null, reason => {
  console.error(reason)
})
```

#### promise::tapCatch(onRejected)

> Alias to [`promise:tap(null, onRejected)`](#promisetaponresolved-onrejected).

#### promise::timeout(ms, [cb or rejectionValue])

> Call a callback if the promise is still pending after `ms`
> milliseconds. Its resolution/rejection is forwarded.
>
> If the callback is omitted, the returned promise is rejected with a
> `TimeoutError`.

```js
import { timeout, TimeoutError } from 'promise-toolbox'

await doLongOperation()::timeout(100, () => {
  return doFallbackOperation()
})

await doLongOperation()::timeout(100)

await doLongOperation()::timeout(
  100,
  new Error('the long operation has failed')
)
```

> Note: `0` is a special value which disable the timeout, useful if the delay is
> configurable in your app.

## Development

```
# Install dependencies
> npm install

# Run the tests
> npm test

# Continuously compile
> npm run dev

# Continuously run the tests
> npm run dev-test

# Build for production
> npm run build
```

## Contributions

Contributions are _very_ welcomed, either on the documentation or on
the code.

You may:

- report any [issue](https://github.com/JsCommunity/promise-toolbox/issues)
  you've encountered;
- fork and create a pull request.

## License

ISC © [Julien Fontanet](https://github.com/julien-f)
