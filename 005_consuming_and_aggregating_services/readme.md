# Consuming and Aggregating Services
## Fetching Data

Let's modify the `routes/root.js` file to the following:

```js
'use strict'
const got = require('got')

const {
  BICYCLE_SERVICE_PORT = 4000
} = process.env

const bicycleSrv = `http://localhost:${BICYCLE_SERVICE_PORT}`

module.exports = async function (fastify, opts) {
  fastify.get('/:id', async function (request, reply) {
    const { id } = request.params
    const bicycle = await got(`${bicycleSrv}/${id}`).json()
    return bicycle
  })
}
```

In this case we've set a default port for the `BICYCLE_SERVICE_PORT` constant, this can be overridden by setting an environment variable named `BICYCLE_SERVICE_PORT` but since we've started our bicycle service on port 4000 there no need.

There is a wide selection of HTTP request libraries in the Node.js ecosystem. We'll use the `got` module because it's well-scoped, well-engineered and has API's that are compatible with async/await functions that we'll be using as route handlers.


## Combining Data
Let's modify the `routes/root.js` file to the following:

```js
'use strict'
const got = require('got')

const {
  BICYCLE_SERVICE_PORT = 4000, BRAND_SERVICE_PORT = 5000
} = process.env

const bicycleSrv = `http://localhost:${BICYCLE_SERVICE_PORT}`
const brandSrv = `http://localhost:${BRAND_SERVICE_PORT}`

module.exports = async function (fastify, opts) {
  fastify.get('/:id', async function (request, reply) {
    const { id } = request.params
    const [ bicycle, brand ] = await Promise.all([
      got(`${bicycleSrv}/${id}`).json(),
      got(`${brandSrv}/${id}`).json()
    ])
    return {
      id: bicycle.id,
      color: bicycle.color,
      brand: brand.name,
    }
  })
}
```


## Managing Status Codes
Let's modify the `routes/root.js` file to the following:

```js
'use strict'

const got = require('got')

const {
  BICYCLE_SERVICE_PORT = 4000, BRAND_SERVICE_PORT = 5000
} = process.env

const bicycleSrv = `http://localhost:${BICYCLE_SERVICE_PORT}`
const brandSrv = `http://localhost:${BRAND_SERVICE_PORT}`

module.exports = async function (fastify, opts) {
  const { httpErrors } = fastify
  fastify.get('/:id', async function (request, reply) {
    const { id } = request.params
    try {
      const [ bicycle, brand ] = await Promise.all([
        got(`${bicycleSrv}/${id}`).json(),
        got(`${brandSrv}/${id}`).json()
      ])
      return {
        id: bicycle.id,
        color: bicycle.color,
        brand: brand.name,
      }
    } catch (err) {
      if (!err.response) throw err
      if (err.response.statusCode === 404) {
        throw httpErrors.notFound()
      }
      if (err.response.statusCode === 400) {
        throw httpErrors.badRequest()
      }
      throw err
    }
  })
}
```