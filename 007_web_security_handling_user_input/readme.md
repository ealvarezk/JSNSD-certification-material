# Avoiding Parameter Pollution Attacks
The parameter pollution exploits a bug that's often created by developers when handling query string parameters. Even when we know about the potential for the bug, it can still be easy to forget. The main aim of such an attack is to cause a service to either crash to slow down by generating an exception in the service. In cases where a crash occurs, it will be because of an unhandled exception. In cases where a slow down occurs it can be caused by generating an exception that's generically handled and the error handling overhead (for instance, stack generation for a new error object on every request) and then sending many requests to the server. Both of these are forms of Denial of Service attacks, we cover mitigating such attacks (in a limited fashion) in the next chapter.

# Route Validation
The recommended approach to route validation in Fastify is using the `schema` option which can be passed when declaring routes. Fastify supports the `JSONSchema` format for declaring the rules for incoming (and also outgoing) data.

The `schema` option supports `body`, `query`, `params`, `headers` and `response` as schemas that can be declared for these areas of input (or output in the case of response).

## body
Let's take a look at the first POST route:
```js
  fastify.post('/', async (request, reply) => {
    const { data } = request.body
    const id = uid()
    await create(id, data)
    reply.code(201)
    return { id }
  })
```
In the route handler we can observe that a data property is expected on request.body. Let's make a schema for the POST body that enforces that shape by modifying the POST route to the following:
```js
 fastify.post('/', {
    schema: {
      body: {
        type: 'object',
        required: ['data'],
        additionalProperties: false,
        properties: {
          data: {
            type: 'object',
            required: ['brand', 'color'],
            additionalProperties: false,
            properties: {
              brand: {type: 'string'},
              color: {type: 'string'}
            }
          }
        }
      }
    }
  }, async (request, reply) => {
    const { data } = request.body
    const id = uid()
    await create(id, data)
    reply.code(201)
    return { id }
  })
```

If we change the payload to something wrong we should get a 400 Bad Request response.

## params
We can apply validation to route parameters with the schema.params option.

Let's take a look at the first POST route:
```js
fastify.post('/:id/update', async (request, reply) => {
    const { id } = request.params
    const { data } = request.body
    try {
      await update(id, data)
      reply.code(204)
    } catch (err) {
      if (err.message === 'not found') throw notFound()
      throw err
    }
  })
```
Let's make a schema for the POST params that enforces that shape by modifying the POST route to the following:
```js
fastify.post('/:id/update', {
  schema: {
    params: {
      id: {
        type: 'integer'
      }
    }
  }
}, async (request, reply) => {
  const { id } = request.params
  const { data } = request.body
  try {
    await update(id, data)
    reply.code(204)
  } catch (err) {
    if (err.message === 'not found') throw notFound()
    throw err
  }
})
```

## response
Finally there's one more thing we can validate: the response. At first this can seem like an odd thing to do. However, in many enterprise architectures databases can be shared, in that multiple services may read and write to the same data storage. This means when retrieving data from a remote source, we cannot entirely trust that data even if it is internal. What if another service hasn't validated input? We don't want to send malicious state to the user.

Let's take a look at the first GET route:
```js
fastify.get('/:id', async (request, reply) => {
  const { id } = request.params
  try {
    return await read(id)
  } catch (err) {
    if (err.message === 'not found') throw notFound()
    throw err
  }
})
```
Let's make a schema for the GET reponse that enforces that shape by modifying the GET route to the following:
```js
fastify.get('/:id', {
  schema: {
    params: paramsSchema,
    response: {
      200: {
        type: 'object',
        required: ['brand', 'color'],
        additionalProperties: false,
        properties: {
          brand: {type: 'string'},
          color: {type: 'string'}
        }
      }
    }
  }
}, async (request, reply) => {
  const { id } = request.params
  try {
    return await read(id)
  } catch (err) {
    if (err.message === 'not found') throw notFound()
    throw err
  }
})
```
Note that the `schema.response` option differs slightly to others, in that we also need to specify the response code. This is because routes can respond with different response codes that send different data. If we wanted to apply a schema to all response codes from 200 to 299 we could set a key called `2xx` on the `schema.response` object. Since our route should only respond with a 201 (unless there's an error) we specified just use a key named 201.