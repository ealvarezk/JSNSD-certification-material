# Creating a Web Server with Fastify
In Node.js binary data is handled with the Buffer constructor. The Buffer constructor is a global, so there's no need to require any core module in order to use the Node core Buffer API.
## Bootstrap a Fastify project
```
npm init fastify
npm install
```
## Scripts
In the scripts field of the package.json we can see a `start` field and a `dev` field. The fastify `start` command automatically starts the server. Notice we have no defined port, this is because fastify start defaults to port 3000, it could be configured with the -p flag (lowercase) if desired.

In the `dev` script there are two additional flags: -w and -P (uppercase). The -P flag means "prettify the log output", which would otherwise be newline delimited JSON logs. The -w flag means “watch and reload the project as we work on it”, so we can go ahead and run the following to start our server:
```
npm run dev
```

## Routes
A route file looks like this:

```js
module.exports = async function (fastify, opts) {
  fastify.get('/', async function (request, reply) {
    return { test: true }
  })
}
```
The file exports an async function that accepts the fastify instance and an options argument. The routes/root.js file exports a Fastify plugin. A Fastify plugin is a function that takes the server instance (fastify).

Within the plugin function, fastify.get is called. This registers an HTTP GET route. The first argument is a string containing a forward slash (/), indicating that the route being registered is the root route (/). All HTTP verbs can be called as methods on the fastify instance (e.g. fastify.post, fastify.put and so on).

Whatever is returned from the function or async function is automatically processed and sent as the content of the HTTP response.

Alternatively the `reply.send` method can be used (e.g. `reply.send({root: true})`). This can be useful when working with nested callback APIs.

Since an object is returned, Fastify converts it to a JSON payload before sending it as a response.

## Block HTTP method
```js
const path = require('path')
const AutoLoad = require('fastify-autoload')

module.exports = async function (fastify, opts) {

  fastify.register(AutoLoad, {
    dir: path.join(__dirname, 'plugins'),
    options: Object.assign({}, opts)
  })

  fastify.register(AutoLoad, {
    dir: path.join(__dirname, 'routes'),
    options: Object.assign({}, opts)
  })

  fastify.setNotFoundHandler((request, reply) => {
    if (request.method !== 'GET') {
      reply.status(405)
      return 'Method Not Allowed\n'
    }
    return 'Not Found\n'
  })

}
```
Here we have the fastify.setNotFoundHandler method call, this method accepts a function with the same criteria as the route handler function passed to fastify.get (and fastify.post, fastify.put and so on). In our case, we use a normal function, inspect the HTTP method and if it is not GET we set the HTTP status code to 405 and then return the associated message (Method Not Allowed). Otherwise we return the 404 message (Not Found). Fastify will call this function and use its output in cases where a route cannot be found (which includes routes that haven't been registered with the requested HTTP verb).