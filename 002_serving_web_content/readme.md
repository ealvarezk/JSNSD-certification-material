# Serving Static Content
We need to add a new Fastify plugin that will handle static content for us.

```
npm install --save-dev fastify-static
```
We've deliberately installed fastify-static as a development dependency. It's generally bad practice to use Node.js for static file hosting in production.

We need to register and configure fastify-static but not in production. Let's make our app.js look as follows:

```js
const path = require('path')
const AutoLoad = require('fastify-autoload')

const dev = process.env.NODE_ENV !== 'production'

const fastifyStatic = dev && require('fastify-static')

module.exports = async function (fastify, opts) {
  if (dev) {
    fastify.register(fastifyStatic, {
      root: path.join(__dirname, 'public')
    })
  }
  ...
}
```

The fastify-static module also decorates the reply object with sendFile method. We can use this to create a route that manually responds with the contents of hello.html if we wanted to alias /hello.html to /hello.

Let's write the following code into `routes/hello/index.js`:

```js
'use strict'

module.exports = async (fastify, opts) => {
  fastify.get('/', async (request, reply) => {
    return reply.sendFile('hello.html')
  })
}
```

## Using Templates
Run the following command in order to install a template engine and Fastify's view rendering plugin:
```
npm install point-of-view handlebars
```

We can set up and configure view rendering, we need to modify our app.js file to look as follows:

```js
const path = require('path')
const AutoLoad = require('fastify-autoload')

const pointOfView = require('point-of-view')
const handlebars = require('handlebars')

module.exports = async function (fastify, opts) {

  fastify.register(pointOfView, {
    engine: { handlebars },
    root: path.join(__dirname, 'views'),
    layout: 'layout.hbs'
  })
  
  ...

}
```

The `views/layout.hbs` file should contain the following:
```hbs
<html>
  <head>
    <style>
     body { background: #333; margin: 1.25rem }
     h1 { color: #EEE; font-family: sans-serif }
     a { color: yellow; font-size: 2rem; font-family: sans-serif }
    </style>
  </head>
  <body>
    {{{ body }}}
  </body>
</html>
```

The `views/index.hbs` file should contain the following:
```hbs
<a href='/hello'>Hello</a><br>
<a href='/hello?greeting=Ahoy'>Ahoy</a>
```

The `views/hello.hbs` file should contain the following:
```hbs
<h1>{{ greeting }} World</h1>
```

The `routes/root.js` file should contain the following:

```js
module.exports = async (fastify, opts) => {
  fastify.get('/', async (request, reply) => {
    return reply.view('index.hbs')
  })
}
```

The `routes/hello/index.js` file should contain the following:

```js
module.exports = async (fastify, opts) => {
  fastify.get('/', async (request, reply) => {
    const { greeting = 'Hello '} = request.query
    return reply.view(`hello.hbs`, { greeting })
  })
}
```


## Streaming Content
The HTTP specification has a header called Transfer-Encoding which can be set to chunked. This means that chunks of data can be sent over HTTP and in many cases browser-clients can begin parsing immediately. Node.js Streams also allow for chunked reading, processing and writing of data. This affinity between Node.js Streams means we can serve content in a highly efficient way: instead of waiting for the server to prepare and process all data and then sending the response, the client can begin parsing some HTML (or theoretically any structured data) we've sent before the server has even finished preparing it for sending.

The contents of `routes/articles/index.js` should be as follows:

```js
const hnLatestStream = require('hn-latest-stream')

module.exports = async (fastify, opts) => {
  fastify.get('/', async (request, reply) => {
    const { amount = 10, type = 'html' } = request.query

    if (type === 'html') reply.type('text/html')
    if (type === 'json') reply.type('application/json')
    return hnLatestStream(amount, type) // hnLatestStream package that provides a stream of Hacker News content
  })
}
```