# Migrating from Express to Fastify

This guide helps you migrate an existing [Express](https://expressjs.com/)
application to Fastify. It maps common Express patterns to their Fastify
equivalents and highlights the differences to keep in mind along the way.

Fastify is not a drop-in replacement for Express. The two frameworks share
similar concepts, such as routes, request and reply objects, and a plugin
system, but they differ in important ways:

* Fastify encourages `async`/`await` handlers and treats a returned value as
  the response payload.
* Fastify does not use Express-style middleware out of the box. It provides
  [hooks](../Reference/Hooks.md) and [decorators](../Reference/Decorators.md)
  instead, though Express middleware can still be used through a plugin.
* Fastify has a powerful [plugin](../Reference/Plugins.md) system with
  encapsulation, which replaces the Express router for structuring an
  application.
* Fastify validates and serializes data using [JSON
  Schema](../Reference/Validation-and-Serialization.md), which improves both
  safety and performance.

Read this guide top to bottom for a full migration, or jump to the pattern you
need using the table of contents below.

## Table of contents

* [Installation](#installation)
* [Creating the server](#creating-the-server)
* [Routing](#routing)
* [Request and reply objects](#request-and-reply-objects)
* [Sending responses](#sending-responses)
* [Middleware](#middleware)
* [Body parsing](#body-parsing)
* [Structuring an application](#structuring-an-application)
* [Validation](#validation)
* [Error handling](#error-handling)
* [Not found (404) handling](#not-found-404-handling)
* [Serving static files](#serving-static-files)
* [Templates and views](#templates-and-views)
* [Common Express libraries and their Fastify equivalents](#common-express-libraries-and-their-fastify-equivalents)
* [Next steps](#next-steps)

## Installation

Install Fastify with your package manager of choice:

```bash
npm install fastify
```

Fastify requires an actively supported version of Node.js. See the
[LTS](../Reference/LTS.md) document for the current support policy.

## Creating the server

In Express, the application object is both the server and the router. In
Fastify, the `fastify` factory function returns an instance that is registered
against a Node.js HTTP server when you call `listen`.

Express:

```js
const express = require('express')
const app = express()

app.get('/', (req, res) => {
  res.send('Hello world')
})

app.listen(3000, () => {
  console.log('Server listening on port 3000')
})
```

Fastify:

```js
const Fastify = require('fastify')
const fastify = Fastify({ logger: true })

fastify.get('/', async (request, reply) => {
  return 'Hello world'
})

fastify.listen({ port: 3000 }, (err, address) => {
  if (err) {
    fastify.log.error(err)
    process.exit(1)
  }
})
```

Note the differences:

* `listen` takes an options object (`{ port, host }`) rather than positional
  arguments.
* Fastify ships with a built-in [logger](../Reference/Logging.md) powered by
  [Pino](https://getpino.io/). Enable it with `{ logger: true }` instead of
  adding a separate logging middleware.
* A handler may return a value or a promise. The returned value becomes the
  response body, so an explicit `reply.send()` is not required.

`listen` also returns a promise, so the server can be started with
`async`/`await`:

```js
const start = async () => {
  try {
    await fastify.listen({ port: 3000 })
  } catch (err) {
    fastify.log.error(err)
    process.exit(1)
  }
}
start()
```

## Routing

Fastify supports the same HTTP method shorthands as Express, so most route
definitions look familiar.

Express:

```js
app.get('/users/:id', handler)
app.post('/users', handler)
app.put('/users/:id', handler)
app.delete('/users/:id', handler)
```

Fastify:

```js
fastify.get('/users/:id', handler)
fastify.post('/users', handler)
fastify.put('/users/:id', handler)
fastify.delete('/users/:id', handler)
```

Fastify also provides a full `route` method that accepts a single options
object, which is the recommended form when adding validation or hooks:

```js
fastify.route({
  method: 'GET',
  url: '/users/:id',
  handler: async (request, reply) => {
    return getUser(request.params.id)
  }
})
```

### Route parameters and wildcards

Named parameters use the same `:param` syntax in both frameworks. Wildcards
differ: Express uses `*`, while Fastify uses a named parameter with a `*`
placeholder.

Express:

```js
app.get('/files/*', (req, res) => {
  res.send(req.params[0])
})
```

Fastify:

```js
fastify.get('/files/*', async (request, reply) => {
  return request.params['*']
})
```

Fastify uses [find-my-way](https://github.com/delvedor/find-my-way) for
routing, which is significantly faster than the Express router. See the
[Routes](../Reference/Routes.md) reference for the full set of options.

## Request and reply objects

Fastify wraps the Node.js `req` and `res` objects in its own
[Request](../Reference/Request.md) and [Reply](../Reference/Reply.md)
instances. The most commonly used properties map directly.

| Express                | Fastify                     |
| ---------------------- | --------------------------- |
| `req.params`           | `request.params`            |
| `req.query`            | `request.query`             |
| `req.body`             | `request.body`              |
| `req.headers`          | `request.headers`           |
| `req.params.id`        | `request.params.id`         |
| `req.ip`               | `request.ip`                |
| `req.log`              | `request.log`               |
| `res.send(payload)`    | `reply.send(payload)`       |
| `res.status(code)`     | `reply.code(code)`          |
| `res.json(payload)`    | `reply.send(payload)`       |
| `res.set(name, value)` | `reply.header(name, value)` |
| `res.redirect(url)`    | `reply.redirect(url)`       |

The underlying Node.js objects remain available as `request.raw` and
`reply.raw` when low-level access is needed.

## Sending responses

Express relies on `res.send`, `res.json`, and `res.status`. In Fastify, a
handler can return the payload directly, which is the idiomatic approach.

Express:

```js
app.get('/user', (req, res) => {
  res.status(200).json({ name: 'Jane' })
})
```

Fastify:

```js
fastify.get('/user', async (request, reply) => {
  reply.code(200)
  return { name: 'Jane' }
})
```

An object returned from a handler is serialized to JSON automatically, with the
`content-type` header set to `application/json`. Returning a string sends a
`text/plain` response.

When mixing `reply.send()` with `async` handlers, do not both call
`reply.send()` and return a value from the same handler. Either return the
payload or call `reply.send()`, but not both. To send a payload from within a
callback while using an `async` handler, return `reply` to signal that the
response is handled manually:

```js
fastify.get('/user', async (request, reply) => {
  setImmediate(() => {
    reply.send({ name: 'Jane' })
  })
  return reply
})
```

## Middleware

Express middleware is the most significant conceptual change. Fastify does not
enable Express-style middleware by default. It offers three approaches,
depending on your needs.

### Prefer hooks

Most middleware exists to run code at a specific point in the request
lifecycle, such as authentication or logging. In Fastify, use
[hooks](../Reference/Hooks.md) for this. The `preHandler` hook is the closest
equivalent to route-level middleware.

Express:

```js
function authenticate (req, res, next) {
  if (!req.headers.authorization) {
    return res.status(401).send('Unauthorized')
  }
  next()
}

app.get('/private', authenticate, handler)
```

Fastify:

```js
async function authenticate (request, reply) {
  if (!request.headers.authorization) {
    reply.code(401)
    throw new Error('Unauthorized')
  }
}

fastify.get('/private', { preHandler: authenticate }, handler)
```

Hooks can be registered for a single route, as above, or for every route in
the current scope with `fastify.addHook('preHandler', authenticate)`. See
[Structuring an application](#structuring-an-application) for how scope works.

### Use Express-style middleware through a plugin

To reuse existing Express middleware, register
[`@fastify/middie`](https://github.com/fastify/middie) or
[`@fastify/express`](https://github.com/fastify/fastify-express). Middie is
lighter and supports simple Express-style middleware, while
`@fastify/express` provides broader Express compatibility.

```js
await fastify.register(require('@fastify/middie'))
fastify.use(require('cors')())
```

For more details, see the [Middleware](../Reference/Middleware.md) reference.

### Prefer native plugins

Many popular Express middleware packages have first-party Fastify equivalents
that are faster and integrate with the plugin system. See the
[table below](#common-express-libraries-and-their-fastify-equivalents) for the
most common ones.

## Body parsing

Express requires `express.json()` or the `body-parser` package to parse
request bodies. Fastify parses `application/json` and
`text/plain` content types out of the box, so no extra setup is needed.

Express:

```js
const express = require('express')
const app = express()

app.use(express.json())

app.post('/user', (req, res) => {
  res.send(req.body)
})
```

Fastify:

```js
const fastify = require('fastify')()

fastify.post('/user', async (request, reply) => {
  return request.body
})
```

To parse other content types, such as `application/x-www-form-urlencoded` or
`multipart/form-data`, register a plugin such as
[`@fastify/formbody`](https://github.com/fastify/fastify-formbody) or
[`@fastify/multipart`](https://github.com/fastify/fastify-multipart), or add a
custom parser with
[`addContentTypeParser`](../Reference/ContentTypeParser.md).

## Structuring an application

Express applications are typically split using `express.Router`. Fastify uses
[plugins](../Reference/Plugins.md) for the same purpose. Every plugin creates
an encapsulated context: hooks, decorators, and other plugins registered inside
it do not leak to the parent scope. This encapsulation is a core Fastify
feature and has no direct Express equivalent.

Express router:

```js
// routes/users.js
const router = require('express').Router()

router.get('/', listUsers)
router.get('/:id', getUser)

module.exports = router

// app.js
app.use('/users', require('./routes/users'))
```

Fastify plugin:

```js
// routes/users.js
async function userRoutes (fastify, options) {
  fastify.get('/', listUsers)
  fastify.get('/:id', getUser)
}

module.exports = userRoutes

// app.js
fastify.register(require('./routes/users'), { prefix: '/users' })
```

The `prefix` option replaces the path passed to `app.use`. Routes defined
inside the plugin are automatically mounted under it.

To share functionality across plugins, such as a database connection, use a
[decorator](../Reference/Decorators.md) instead of attaching properties to the
application object. Wrap the decorator in
[`fastify-plugin`](https://github.com/fastify/fastify-plugin) to break out of
the encapsulation and make it available application-wide:

```js
const fp = require('fastify-plugin')

module.exports = fp(async function (fastify, options) {
  const db = await connectToDatabase()
  fastify.decorate('db', db)
})
```

The decorated value is then available as `fastify.db` and `request.server.db`
in any scope.

## Validation

Express applications often validate input manually or with a library such as
Joi inside each handler. Fastify validates requests declaratively with [JSON
Schema](../Reference/Validation-and-Serialization.md), attached to the route.
Invalid requests are rejected before the handler runs.

Express with manual validation:

```js
app.post('/user', (req, res) => {
  const { name, age } = req.body
  if (typeof name !== 'string' || typeof age !== 'number') {
    return res.status(400).send('Invalid input')
  }
  res.send({ name, age })
})
```

Fastify with a schema:

```js
const schema = {
  body: {
    type: 'object',
    required: ['name', 'age'],
    properties: {
      name: { type: 'string' },
      age: { type: 'number' }
    }
  }
}

fastify.post('/user', { schema }, async (request, reply) => {
  return request.body
})
```

The same mechanism validates `params`, `querystring`, and `headers`. A response
schema additionally speeds up serialization and filters out properties that are
not declared. See
[Validation and Serialization](../Reference/Validation-and-Serialization.md)
for the details.

## Error handling

Express uses error-handling middleware with a four-argument signature. Fastify
provides a single error handler per scope, set with `setErrorHandler`. In an
`async` handler, throwing an error routes it to the handler automatically.

Express:

```js
app.use((err, req, res, next) => {
  console.error(err)
  res.status(500).send('Something broke')
})
```

Fastify:

```js
fastify.setErrorHandler((error, request, reply) => {
  request.log.error(error)
  reply.status(500).send('Something broke')
})
```

Errors thrown inside a handler, hook, or validation step are passed to the
error handler. To control the status code, attach a `statusCode` property to
the error or use [`@fastify/sensible`](https://github.com/fastify/fastify-sensible)
for convenient HTTP error helpers such as `httpErrors.notFound()`.

## Not found (404) handling

In Express, a 404 is handled by a middleware added after all routes. Fastify
provides a dedicated `setNotFoundHandler`.

Express:

```js
app.use((req, res) => {
  res.status(404).send('Not found')
})
```

Fastify:

```js
fastify.setNotFoundHandler((request, reply) => {
  reply.status(404).send('Not found')
})
```

Like other handlers, the not-found handler is scoped, so a plugin can define
its own.

## Serving static files

Express uses the built-in `express.static` middleware. Fastify uses the
[`@fastify/static`](https://github.com/fastify/fastify-static) plugin.

Express:

```js
app.use(express.static('public'))
```

Fastify:

```js
const path = require('node:path')

fastify.register(require('@fastify/static'), {
  root: path.join(__dirname, 'public'),
  prefix: '/public/'
})
```

## Templates and views

Express integrates template engines through `app.set('view engine', ...)` and
`res.render`. Fastify provides the same capability through the
[`@fastify/view`](https://github.com/fastify/point-of-view) plugin.

Express:

```js
app.set('view engine', 'ejs')

app.get('/', (req, res) => {
  res.render('index', { text: 'world' })
})
```

Fastify:

```js
fastify.register(require('@fastify/view'), {
  engine: {
    ejs: require('ejs')
  }
})

fastify.get('/', (request, reply) => {
  return reply.view('/templates/index.ejs', { text: 'world' })
})
```

## Common Express libraries and their Fastify equivalents

Many widely used Express middleware packages have first-party Fastify plugins.
Prefer these over the Express versions, as they are optimized for Fastify and
integrate with its plugin system.

| Express package | Fastify equivalent |
| --------------- | ------------------ |
| `cors`          | [`@fastify/cors`](https://github.com/fastify/fastify-cors) |
| `helmet`        | [`@fastify/helmet`](https://github.com/fastify/fastify-helmet) |
| `serve-static`  | [`@fastify/static`](https://github.com/fastify/fastify-static) |
| `cookie-parser` | [`@fastify/cookie`](https://github.com/fastify/fastify-cookie) |
| `express-session` | [`@fastify/session`](https://github.com/fastify/session) |
| `body-parser` (urlencoded) | [`@fastify/formbody`](https://github.com/fastify/fastify-formbody) |
| `multer`        | [`@fastify/multipart`](https://github.com/fastify/fastify-multipart) |
| `express-rate-limit` | [`@fastify/rate-limit`](https://github.com/fastify/fastify-rate-limit) |
| `passport`      | [`@fastify/passport`](https://github.com/fastify/fastify-passport) |
| `compression`   | [`@fastify/compress`](https://github.com/fastify/fastify-compress) |
| `morgan`        | Built-in [Pino](https://getpino.io/) logger (`{ logger: true }`) |

For a broader list of plugins, see the [Ecosystem](./Ecosystem.md) guide.

## Next steps

* Read the [Getting Started](./Getting-Started.md) guide for a hands-on
  introduction to Fastify.
* Learn how the [Plugins](./Plugins-Guide.md) system and encapsulation work.
* Explore [Hooks](../Reference/Hooks.md) to replace Express middleware.
* Review [Validation and
  Serialization](../Reference/Validation-and-Serialization.md) to take full
  advantage of JSON Schema.
