
# Guide

  This guide covers Koa topics that are not directly API related, such as best practices for writing middleware and application structure suggestions. In these examples we use async functions as middleware - you can also use commonFunction or generatorFunction which will be a little different.

## Table of Contents

- [Guide](#guide)
  - [Table of Contents](#table-of-contents)
  - [Writing Middleware](#writing-middleware)
  - [Middleware Best Practices](#middleware-best-practices)
    - [Middleware options](#middleware-options)
    - [Named middleware](#named-middleware)
    - [Combining multiple middleware with koa-compose](#combining-multiple-middleware-with-koa-compose)
    - [Response Middleware](#response-middleware)
  - [Async operations](#async-operations)
  - [Debugging Koa](#debugging-koa)
  - [HTTP2](#http2)
  - [Server-Side Events](#server-side-events)

## Writing Middleware

  Koa middleware are simple functions with the signature `(ctx, next)` (i.e. a `MiddlewareFunction`). When
  the middleware is run, it must manually invoke `next()` to run the "downstream" middleware.

  For example if you wanted to track how long it takes for a request to propagate through Koa by adding an
  `X-Response-Time` header field the middleware would look like the following:

```js
async function responseTime(ctx, next) {
  const start = Date.now();
  await next();
  const ms = Date.now() - start;
  ctx.set('X-Response-Time', `${ms}ms`);
}

app.use(responseTime);
```

  If you're a front-end developer you can think any code before `next();` as the "capture" phase,
  while any code after is the "bubble" phase. This crude gif illustrates how async function allow us
  to properly utilize stack flow to implement request and response flows:

![Koa middleware](/docs/middleware.gif)

   1. Create a date to track response time
   2. Await control to the next middleware
   3. Create another date to track duration
   4. Await control to the next middleware
   5. Set the response body to "Hello World"
   6. Calculate duration time
   7. Output log line
   8. Calculate response time
   9. Set `X-Response-Time` header field
   10. Hand off to Koa to handle the response

 Next we'll look at the best practices for creating Koa middleware.

## Middleware Best Practices

  This section covers middleware authoring best practices, such as middleware
  accepting options, named middleware for debugging, among others.

### Middleware options

  When creating a public middleware, it's useful to conform to the convention of
  wrapping the middleware in a function that accepts options, allowing users to
  extend functionality. Even if your middleware accepts _no_ options, this is still
  a good idea to keep things uniform.

  Here our contrived `logger` middleware accepts a `format` string for customization,
  and returns the middleware itself:

```js
function logger(format) {
  format = format || ':method ":url"';

  return async function (ctx, next) {
    const str = format
      .replace(':method', ctx.method)
      .replace(':url', ctx.url);

    console.log(str);

    await next();
  };
}

app.use(logger());
app.use(logger(':method :url'));
```

### Named middleware

  Naming middleware is optional, however it's useful for debugging purposes to assign a name.

```js
function logger(format) {
  return async function logger(ctx, next) {

  };
}
```

### Combining multiple middleware with koa-compose

  Sometimes you want to "compose" multiple middleware into a single middleware for easy re-use or exporting. You can use [koa-compose](https://github.com/koajs/compose)

```js
const compose = require('koa-compose');

async function random(ctx, next) {
  if ('/random' == ctx.path) {
    ctx.body = Math.floor(Math.random() * 10);
  } else {
    await next();
  }
};

async function backwards(ctx, next) {
  if ('/backwards' == ctx.path) {
    ctx.body = 'sdrawkcab';
  } else {
    await next();
  }
}

async function pi(ctx, next) {
  if ('/pi' == ctx.path) {
    ctx.body = String(Math.PI);
  } else {
    await next();
  }
}

const all = compose([random, backwards, pi]);

app.use(all);
```

### Response Middleware

  Middleware that decide to respond to a request and wish to bypass downstream middleware may
  simply omit `next()`. Typically this will be in routing middleware, but this can be performed by
  any. For example the following will respond with "two", however all three are executed, giving the
  downstream "three" middleware a chance to manipulate the response.

```js
app.use(async function (ctx, next) {
  console.log('>> one');
  await next();
  console.log('<< one');
});

app.use(async function (ctx, next) {
  console.log('>> two');
  ctx.body = 'two';
  await next();
  console.log('<< two');
});

app.use(async function (ctx, next) {
  console.log('>> three');
  await next();
  console.log('<< three');
});
```

  The following configuration omits `next()` in the second middleware, and will still respond
  with "two", however the third (and any other downstream middleware) will be ignored:

```js
app.use(async function (ctx, next) {
  console.log('>> one');
  await next();
  console.log('<< one');
});

app.use(async function (ctx, next) {
  console.log('>> two');
  ctx.body = 'two';
  console.log('<< two');
});

app.use(async function (ctx, next) {
  console.log('>> three');
  await next();
  console.log('<< three');
});
```

  When the furthest downstream middleware executes `next();`, it's really yielding to a noop
  function, allowing the middleware to compose correctly anywhere in the stack.

## Async operations

  Async function and promise forms Koa's foundation, allowing
  you to write non-blocking sequential code. For example this middleware reads the filenames from `./docs`,
  and then reads the contents of each markdown file in parallel before assigning the body to the joint result.


```js
const fs = require('fs').promises;

app.use(async function (ctx, next) {
  const paths = await fs.readdir('docs');
  const files = await Promise.all(paths.map(path => fs.readFile(`docs/${path}`, 'utf8')));

  ctx.type = 'markdown';
  ctx.body = files.join('');
});
```

## Debugging Koa

  Koa, along with many of the libraries it's built, can be debugged using Node.js's built-in util.debuglog, which provides simple conditional logging via the __NODE_DEBUG__ environment variable.

  For example
  to see all Koa-specific debugging information just pass `NODE_DEBUG=koa*` and upon boot you'll see the list of middleware used, among other things.

```
$ NODE_DEBUG=koa* node --harmony examples/simple
  koa:application use responseTime +0ms
  koa:application use logger +4ms
  koa:application use contentLength +0ms
  koa:application use notfound +0ms
  koa:application use response +0ms
  koa:application listen +0ms
```

  Since JavaScript does not allow defining function names at
  runtime, you can also set a middleware's name as `._name`.
  This is useful when you don't have control of a middleware's name.
  For example:

```js
const path = require('node:path');
const serve = require('koa-static');

const publicFiles = serve(path.join(__dirname, 'public'));
publicFiles._name = 'static /public';

app.use(publicFiles);
```

  Now, instead of just seeing "serve" when debugging, you will see:

```
  koa:application use static /public +0ms
```

This lets you use Koa’s internal debug logs without any third-party dependencies by relying on node:util's built-in debuglog.

## HTTP2

Example of setting up an HTTP2 server with Koa using the HTTP compatibility layer:

```js
import Koa from 'koa'
import http2 from 'node:http2'
import fs from 'node:fs'

const app = new Koa();

const onRequestHandler = app.callback();
const serverOptions = {
  key: fs.readFileSync('key.pem'),
  cert: fs.readFileSync('cert.pem')
}

const server = http2.createSecureServer(serverOptions, onRequestHandler);
server.listen(3000);
```

## Server-Side Events

An example of using server-side events with Koa:

```js
import { PassThrough } from 'node:stream'
import OpenAI from 'openai'
import Koa from 'koa'

const openai = new OpenAI({
  apiKey: process.env.OPENAI_API_KEY,
})

const app = new Koa()

app.use(async (ctx, next) => { // TODO: use your favorite routing library
  const { message } = await ctx.request.json() // this example uses koa-body-parsers

  const stream = await openai.chat.completions.create({
    model: 'gpt-4o-mini',
    messages: [{ role: 'user', content: message }],
    stream: true,
  })

  ctx.response.type = 'text/event-stream'
  const body = ctx.body = new PassThrough()

  // streaming needs to be handled in a separate event loop
  ;(async () => {
    for await (const chunk of stream) {
      const content = chunk.choices[0]?.delta?.content
      if (!content) continue

      body.write(`data: ${JSON.stringify({
        delta: {
          content: chunk.choices[0].delta.content || ''
        }
      })}\n\n`)
    }
    body.end()
  })().catch((err) => app.emit('error', err))
})

app.listen(3000)
```
