# Implementing a RESTful JSON GET

## fastify-sensible

The fastify-sensible plugin adds some useful "sane defaults" to Fastify, including the convenience functions for HTTP status codes and messages. 

## HTTP Status codes
### 1×× Informational
100 Continue

101 Switching Protocols

102 Processing

### 2×× Success

200 OK

201 Created

202 Accepted

203 Non-authoritative Information

204 No Content

205 Reset Content

206 Partial Content

207 Multi-Status

208 Already Reported

226 IM Used

### 3×× Redirection

300 Multiple Choices

301 Moved Permanently

302 Found

303 See Other

304 Not Modified

305 Use Proxy

307 Temporary Redirect

308 Permanent Redirect

### 4×× Client Error

400 Bad Request

401 Unauthorized

402 Payment Required

403 Forbidden

404 Not Found

405 Method Not Allowed

406 Not Acceptable

407 Proxy Authentication Required

408 Request Timeout

409 Conflict

410 Gone

411 Length Required

412 Precondition Failed

413 Payload Too Large

414 Request-URI Too Long

415 Unsupported Media Type

416 Requested Range Not Satisfiable

417 Expectation Failed

418 I'm a teapot

421 Misdirected Request

422 Unprocessable Entity

423 Locked

424 Failed Dependency

426 Upgrade Required

428 Precondition Required

429 Too Many Requests

431 Request Header Fields Too Large

444 Connection Closed Without Response

451 Unavailable For Legal Reasons

499 Client Closed Request

### 5×× Server Error

500 Internal Server Error

501 Not Implemented

502 Bad Gateway

503 Service Unavailable

504 Gateway Timeout

505 HTTP Version Not Supported

506 Variant Also Negotiates

507 Insufficient Storage

508 Loop Detected

510 Not Extended

511 Network Authentication Required

599 Network Connect Timeout Error


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
