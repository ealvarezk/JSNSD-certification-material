# Single-Route, Multi-Origin Proxy
There may be some circumstances where we need to send data from another service via our service. In these cases, we could actually use an HTTP request library like got as explored in the prior chapter. However, using a proxying library is a viable alternative that provides a more configuration-based approach vs the procedural approach of using a request library.

Let's install the `fastify-reply-from`.
```
npm install fastify-reply-from
```

We'll modify the `routes/route.js` file to the following:
```js
'use strict'

module.exports = async function (fastify, opts) {
  fastify.get('/', async function (request, reply) {
    const { url } = request.query
    try {
      new URL(url)
    } catch (err) {
      throw fastify.httpErrors.badRequest()
    }
    return reply.from(url)
  })
}
```

More advanced proxying scenarios involve rewriting some aspect of the response from the upstream service while it's replying to the client. To finish off this section let's make our proxy server uppercase all content that arrives from the upstream service before sending it on to the client.

Let's update the `routes/root.js` file to the following:
```js
'use strict'
const { Readable } = require('stream')
async function * upper (res) {
  for await (const chunk of res) {
    yield chunk.toString().toUpperCase()
  }
}
module.exports = async function (fastify, opts) {
  fastify.get('/', async function (request, reply) {
    const { url } = request.query
    try {
      new URL(url)
    } catch (err) {
      throw fastify.httpErrors.badRequest()
    }
    return reply.from(url, {
      onResponse (request, reply, res) {
        reply.send(Readable.from(upper(res)))
      }
    })
  })
}
```

# Single-Origin, Multi-Route Proxy
We can map every path (and indeed every HTTP method) made to our proxy service straight to the upstream service.

Let's install the `fastify-http-proxy`.
```
npm install fastify-http-proxy
```

We'll modify the `app.js` file to the following:
```js
'use strict'
const proxy = require('fastify-http-proxy')
module.exports = async function (fastify, opts) {
  fastify.register(proxy, {
    upstream: 'htt‌ps://news.ycombinator.com/'
  })
}
```

Imagine a nascent authentication approach which isn't yet supported in larger projects. We can use the `preHandler` option supported by `fastify-http-proxy` to implement custom authentication logic.

We'll modify the `app.js` file to the following:
```js
'use strict'

const proxy = require('fastify-http-proxy')
const sensible = require('fastify-sensible')
module.exports = async function (fastify, opts) {
  fastify.register(sensible)
  fastify.register(proxy, {
    upstream: 'https://news.ycombinator.com/',
    async preHandler(request, reply) {
      if (request.query.token !== 'abc') {
        throw fastify.httpErrors.unauthorized()
      }
    }
  })
}
```