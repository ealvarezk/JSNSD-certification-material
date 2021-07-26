# Block an Attackers' IP Address
An attack can come from multiple machines, which tends to mean it can come from multiple IPs. However, once we know how to block one IP in a service we can block as many IPs as we like.

Fastify uses a plugin-based approach and provides an abstraction on top of the native `req` and `res` objects (request and reply) instead of adding directly to them as in Express. To get the IP address of a requesting client we use `request.ip`.

Fastify also provides a `Hooks API`, which allows us to intervene at various points of the request/response `life-cycle`.

To configure the server to block an IP, we could create a file named `deny.js` placed into the plugins folder. This will be automatically loaded.

If we wanted to block the IP 127.0.0.1 the `plugins/deny.js` file would look as follows:
```js
'use strict'

const fp = require('fastify-plugin')

module.exports = fp(async function (fastify, opts) {
  fastify.addHook('onRequest', async function (request) {
    if (request.ip === '127.0.0.1') {
      throw fastify.httpErrors.forbidden()
    }
  })
})
```

