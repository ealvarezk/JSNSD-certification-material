# Implementing a RESTful JSON GET

## fastify-sensible

The fastify-sensible plugin adds some useful "sane defaults" to Fastify, including the convenience functions for HTTP status codes and messages. 

## Example

Let's modify `routes/bicycle/index.js` to the following:
```js
'use strict'

const { bicycle } = require('../../model')

module.exports = async (fastify, opts) => {
  fastify.get('/:id', async (request, reply) => {
    const { id } = request.params
    bicycle.read(id, (err, result) => {
      if (err) {
        if (err.message === 'not found') reply.notFound()
        else reply.send(err)
      } else reply.send(result)
    })
    await reply
  })
}
```

If there is an error, the err object will be populated. If the message is `'not found'` then `reply.notFound` is called. This is a method added by the `fastify-sensible` plugin, it sets the response status code to 404 and generates some JSON output describing the Not Found error. If the error is something else, this is assumed to be a server error, and the `err` object is passed to `reply.send`. This causes Fastify to generate a 500 response and output the error message.

The result argument would hold a JavaScript object, so Fastify would know to set the response `Content-Type` header to `application/json`.

### Using promisify
```js
const { promisify } = require('util')
const { bicycle } = require('../../model')
const read = promisify(bicycle.read)

module.exports = async (fastify, opts) => {
  const { notFound } = fastify.httpErrors

  fastify.get('/:id', async (request, reply) => {
    const { id } = request.params
    try {
      return await read(id)
    } catch (err) {
      if (err.message === 'not found') throw notFound()
      throw err
    }
  })
}
```
Note that to generate a 404 Not Found HTTP Status we throw `fastify.httpErrors.notFound` instead of using `reply.notFound`. We also throw the caught `err` instead of passing `err` to `reply.send`. This is extremely useful as any unexpected throw in an async route handler will result in 500 stratus.
