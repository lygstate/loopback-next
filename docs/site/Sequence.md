---
lang: en
title: 'Sequence'
keywords: LoopBack 4.0, LoopBack 4
sidebar: lb4_sidebar
permalink: /doc/en/lb4/Sequence.html
---

## What is a Sequence?

A `Sequence` is a stateless grouping of [Actions](#actions) that control how a
`Server` responds to requests.

The contract of a `Sequence` is simple: it must produce a response to a request.
Creating your own `Sequence` gives you full control over how your `Server`
instances handle requests and responses. The `DefaultSequence` looks like this:

<!--
  FIXME(kev): Should we be copying this logic into the docs directly?
  What if this code changes?
-->

```ts
class DefaultSequence {
  async handle(context: RequestContext) {
    try {
      const route = this.findRoute(context.request);
      const params = await this.parseParams(context.request, route);
      const result = await this.invoke(route, params);
      await this.send(context.response, result);
    } catch (error) {
      await this.reject(context, error);
    }
  }
}
```

## Elements

In the example above, `route`, `params`, and `result` are all Elements. When
building sequences, you use LoopBack Elements to respond to a request:

- [`FindRoute`](http://apidocs.loopback.io/@loopback%2fdocs/rest.html#FindRoute)
- [`Request`](http://apidocs.strongloop.com/loopback-next/) - (TBD) missing API
  docs link
- [`Response`](http://apidocs.strongloop.com/loopback-next/) - (TBD) missing API
  docs link
- [`OperationRetVal`](http://apidocs.loopback.io/@loopback%2fdocs/rest.html#OperationRetval)
- [`ParseParams`](http://apidocs.loopback.io/@loopback%2fdocs/rest.html#ParseParams)
- [`OpenAPISpec`](http://apidocs.loopback.io/@loopback%2fdocs/openapi-v3-types.html#OpenApiSpec)

## Actions

Actions are JavaScript functions that only accept or return `Elements`. Since
the input of one action (an Element) is the output of another action (Element)
you can easily compose them. Below is an example that uses several built-in
Actions:

```ts
class MySequence extends DefaultSequence {
  async handle(context: RequestContext) {
    // findRoute() produces an element
    const route = this.findRoute(context.request);
    // parseParams() uses the route element and produces the params element
    const params = await this.parseParams(context.request, route);
    // invoke() uses both the route and params elements to produce the result (OperationRetVal) element
    const result = await this.invoke(route, params);
    // send() uses the result element
    await this.send(context.response, result);
  }
}
```

## Custom Sequences

Most use cases can be accomplished with `DefaultSequence` or by slightly
customizing it. When an app is generated by the command `lb4 app`, a sequence
file extending `DefaultSequence` at `src/sequence.ts` is already generated and
bound for you so that you can easily customize it.

Here is an example where the application logs out a message before and after a
request is handled:

```ts
import {DefaultSequence, Request, Response} from '@loopback/rest';

class MySequence extends DefaultSequence {
  log(msg: string) {
    console.log(msg);
  }
  async handle(context: RequestContext) {
    this.log('before request');
    await super.handle(context);
    this.log('after request');
  }
}
```

In order for LoopBack to use your custom sequence, you must register it before
starting your `Application`:

```js
import {RestApplication} from '@loopback/rest';

const app = new RestApplication();
app.sequence(MySequencce);

app.start();
```

## Advanced topics

### Customizing Sequence Actions

There might be scenarios where the default sequence _ordering_ is not something
you want to change, but rather the individual actions that the sequence will
execute.

To do this, you'll need to override one or more of the sequence action bindings
used by the `RestServer`, under the `RestBindings.SequenceActions` constants.

As an example, we'll implement a custom sequence action to replace the default
"send" action. This action is responsible for returning the response from a
controller to the client making the request.

To do this, we'll register a custom send action by binding a
[Provider](http://apidocs.strongloop.com/@loopback%2fdocs/context.html#Provider)
to the `RestBindings.SequenceActions.SEND` key.

First, let's create our `CustomSendProvider` class, which will provide the send
function upon injection.

{% include code-caption.html content="/src/providers/custom-send.provider.ts" %}
**custom-send.provider.ts**

```ts
import {Send, Response} from '@loopback/rest';
import {Provider, BoundValue, inject} from '@loopback/context';
import {writeResultToResponse, RestBindings, Request} from '@loopback/rest';

// Note: This is an example class; we do not provide this for you.
import {Formatter} from '../utils';

export class CustomSendProvider implements Provider<Send> {
  // In this example, the injection key for formatter is simple
  constructor(
    @inject('utils.formatter') public formatter: Formatter,
    @inject(RestBindings.Http.REQUEST) public request: Request,
  ) {}

  value() {
    // Use the lambda syntax to preserve the "this" scope for future calls!
    return (response: Response, result: OperationRetval) => {
      this.action(response, result);
    };
  }
  /**
   * Use the mimeType given in the request's Accept header to convert
   * the response object!
   * @param response The response object used to reply to the  client.
   * @param result The result of the operation carried out by the controller's
   * handling function.
   */
  action(response: Response, result: OperationRetval) {
    if (result) {
      // Currently, the headers interface doesn't allow arbitrary string keys!
      const headers = (this.request.headers as any) || {};
      const header = headers.accept || 'application/json';
      const formattedResult = this.formatter.convertToMimeType(result, header);
      response.setHeader('Content-Type', header);
      response.end(formattedResult);
    } else {
      response.end();
    }
  }
}
```

Our custom provider will automatically read the `Accept` header from the request
context, and then transform the result object so that it matches the specified
MIME type.

Next, in our application class, we'll inject this provider on the
`RestBindings.SequenceActions.SEND` key.

{% include code-caption.html content="/src/application.ts" %}

```ts
import {RestApplication, RestBindings} from '@loopback/rest';
import {
  RepositoryMixin,
  Class,
  Repository,
  juggler,
} from '@loopback/repository';
import {CustomSendProvider} from './providers/custom-send.provider';
import {Formatter} from './utils';
import {BindingScope} from '@loopback/context';

export class YourApp extends RepositoryMixin(RestApplication) {
  constructor() {
    super();
    // Assume your controller setup and other items are in here as well.
    this.bind('utils.formatter')
      .toClass(Formatter)
      .inScope(BindingScope.SINGLETON);
    this.bind(RestBindings.SequenceActions.SEND).toProvider(CustomSendProvider);
  }
}
```

As a result, whenever the send action of the
[`DefaultSequence`](http://apidocs.strongloop.com/@loopback%2fdocs/rest.html#DefaultSequence)
is called, it will make use of your function instead! You can use this approach
to override any of the actions listed under the `RestBindings.SequenceActions`
namespace.

### Query string parameters

{% include content/tbd.html %}

How to get query string param values.

### Parsing Requests

Parsing and validating arguments from the request url, headers, and body. See
page [Parsing requests](Parsing-requests.md)

### Invoking controller methods

{% include content/tbd.html %}

- How to use `invoke()` in simple and advanced use cases.
- Explain what happens when you call `invoke()`
- Mention caching use case
- Can I call invoke multiple times?

### Writing the response

{% include content/tbd.html %}

- Must call `sendResponse()` exactly once
- Streams?

### Handling errors

There are many reasons why the application may not be able to handle an incoming
request:

- The requested endpoint (method + URL path) was not found.
- Parameters provided by the client were not valid.
- A backend database or a service cannot be reached.
- The response object cannot be converted to JSON because of cyclic
  dependencies.
- A programmer made a mistake and a `TypeError` is thrown by the runtime.
- And so on.

In the Sequence implementation described above, all errors are handled by a
single `catch` block at the end of the sequence, using the Sequence Action
called `reject`.

The default implementation of `reject` does the following steps:

- Call
  [strong-error-handler](https://github.com/strongloop/strong-error-handler) to
  send back an HTTP response describing the error.
- Log the error to `stderr` if the status code was 5xx (an internal server
  error, not a bad request).

To prevent the application from leaking sensitive information like filesystem
paths and server addresses, the error handler is configured to hide error
details.

- For 5xx errors, the output contains only the status code and the status name
  from the HTTP specification. For example:

  ```json
  {
    "error": {
      "statusCode": 500,
      "message": "Internal Server Error"
    }
  }
  ```

- For 4xx errors, the output contains the full error message (`error.message`)
  and the contents of the `details` property (`error.details`) that
  `ValidationError` typically uses to provide machine-readable details about
  validation problems. It also includes `error.code` to allow a machine-readable
  error code to be passed through which could be used, for example, for
  translation.

  ```json
  {
    "error": {
      "statusCode": 422,
      "name": "Unprocessable Entity",
      "message": "Missing required fields",
      "code": "MISSING_REQUIRED_FIELDS"
    }
  }
  ```

During development and testing, it may be useful to see all error details in the
HTTP responsed returned by the server. This behavior can be enabled by enabling
the `debug` flag in error-handler configuration as shown in the code example
below. See strong-error-handler
[docs](https://github.com/strongloop/strong-error-handler#options) for a list of
all available options.

```ts
app.bind(RestBindings.ERROR_WRITER_OPTIONS).to({debug: true});
```

An example error message when the debug mode is enabled:

```json
{
  "error": {
    "statusCode": 500,
    "name": "Error",
    "message": "ENOENT: no such file or directory, open '/etc/passwords'",
    "errno": -2,
    "syscall": "open",
    "code": "ENOENT",
    "path": "/etc/passwords",
    "stack": "Error: a test error message\n    at Object.openSync (fs.js:434:3)\n    at Object.readFileSync (fs.js:339:35)"
  }
}
```

### Keeping your Sequences

{% include content/tbd.html %}

- Try and use existing actions
- Implement your own version of built in actions
- Publish reusable actions to npm
