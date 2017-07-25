# Trooba API

In this chapter we are going to explore Trooba API and build some generic pipelines. Trooba is generic enough and does not impose any restrictions on how it should be used. It does provide a minimal set of artifacts and defines life cycle, a basic request/response and streaming support to demonstrate how it can be used, but it does not assume any structure for the request/response or data objects.

The pipeline lifecycle has three main stages

* Initialization, where one can hook to a different events.
* Initiation and execution of “request” flow, which is not an actual request assumed here, but more a direction in which events flow, from the initiator towards the target.
* Initiation and execution of “response” flow, a direction from the target towards the initiator.

One can view Trooba runtime as a message bus. Trooba defines the directions (request, response); which is used to decide where the messages should go. Every handler becomes a pipe point where the messages can originate, go through, get transformed or change the direction of the flow can change.

The understanding of the above is important when one wants to extend the existing flows or go further and create a new use-case.

Trooba defines two types of handlers, but does not limit to define more. The types are

* Generic is handler that performs some specific tasks and passes control to the next in the pipeline or can reverse the flow by throwing the error or starting the response. For example, oauth handler can get a security token form some token minting service and attach it to the request for the services that require security token.
* Transport is almost the same as a handler, but it usually connects the pipe to the external component through some protocol like http, TCP or more high level as gRPC.

## Pipeline API

### Building a pipe

```js
const Trooba = require('trooba');
// build the shared pipe object
const pipe = Trooba
  .use('handler-1')
  .use('handler-2')
  .use('transport')
  .build();
// make a call
pipe.create(context).request(request, (err, response) => {
  // handle err or response here
});
```

### Defining a handler

```js
module.exports = function handler(pipe, config) {
  pipe.on('request', (request, next) => { // optional
    // modify request here
    next(); // or next({foo:'bar'})
  });
  pipe.on('error', (err, next) => { // optional
    // do something with error case
    next(); // next(new Error('Boom'))
  });
  pipe.on('response', (response, next) => { // optional
    // modify response here
    next(); // you can provide a new one next({statusCode:404})
  });
};
```

### Defining an http transport

```js
module.exports = function transport(pipe, config) {
    pipe.on('request', request => {
        Wreck.request(request.method,
          config.url, request, (err, response) => {
            if (err) return pipe.throw(err);
            // read response
            Wreck.read(response, (err, body) => {
                if (err) {
                    return pipe.throw(err);
                }
                response.body = body;
                pipe.respond(response);
            });
        });
    });
}
```

Now, let’s have some fun and build a slot machine pipe:

```js
'use strict';

const Trooba = require('trooba');

const symbols = ['🍊', '🍉', '🍈', '🍇', '🍆', '🍅', '🍄'];
function spinReel(pipe) {
  pipe.once('request', (request, next) => {
    const position = Math.round(Math.random() * (symbols.length - 1));
    request.push(symbols[position]);
    next();
  });
}

function validate(pipe) {
  pipe.once('request', (request, next) => {
    const line = request.join(' ');
    if (line === [request[0], request[0], request[0]].join(' ')) {
      // you won!
      return pipe.respond('You won! ' + line);
    }
    pipe.respond('You lost! ' + line);
  });
}

// create a pipeline
const sharablePipe = Trooba
  .use(spinReel) // reel 1
  .use(spinReel) // reel 2
  .use(spinReel) // reel 3
  .use(validate)
  .build();

// create a generic client and inject context
const client = sharablePipe.create();

// make a request
client.request([], console.log);
```

```
null 'You lost! 🍉 🍊 🍆'
$ node trooba-slot-machine.js
null 'You lost! 🍆 🍈 🍄'
$ node trooba-slot-machine.js
null 'You lost! 🍆 🍆 🍉'
$ node trooba-slot-machine.js
null 'You won! 🍈 🍈 🍈'
```

Here’s a more typical example of service invocation pipeline:

```js
const Trooba = require('trooba');

const sharablePipe = Trooba
  .use('circuit')
  .use('trace')
  .use('http')
  .build();
// At this moment sharablePipe can be re-used for different requests

// create a generic client and inject context
const client = sharablePipe.create({
  'foo': 'bar',
  'etc': '...'
});

// make a request
client.request('hello', (err, response) => {
  // handle a response
  console.log(err, response);
});
```

In the next chapter we will explore how we can expand the use of pipeline framework.
