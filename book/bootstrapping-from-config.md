# Scaling Service Pipelines Through Configuration

In this chapter we are going to talk about what it take to bring your application development to the next level from platform point of view by bootstrapping pipelines out of configuration.

The framework already provides a lot of flexibility by decoupling a data flow into specific stages that can be developed and tested separately and used to compose complex and reliable data flows.

But it is still not perfect if one takes a position of platform team that needs to service 100+ applications, especially when it comes to providing fixes or other maintenance tasks to reflect current network infrastructure.

That’s where pipeline configuration comes to rescue. The configuration is much easier to inject to any application as well as update it remotely without changes to the application code. That’s where we eliminate the involvement of application teams and they would definitely appreciate this if they knew the effort it would take them to update this manually every time we change something. This approach greatly simplifies the life of the platform team.

This is one if not the most important component of the framework.

The idea is to have a base handlers and their properties defined by the platform teams and allow a way to tweak or add custom handlers in the specific places of the pipeline for application teams.

We have developed a [trooba-bootstrap](https://github.com/trooba/trooba-bootstrap) component that we are going to review here.

The component tries to abstract from the configuration lifecycle and assumes only two phases: initial bootstrap and update phase.

During initial bootstrap application developer would read config from the platform application repository or local application configuration and use it to create a pipe provider.

The update phase is optional and used when the configuration used for the pipes is detected to have changed.

A platform developer can define his own bootstrap module or layer on top of the existing one that understands the platform specific configuration infrastructure.

## Configuration

### Profiles

Platform team needs to define a common configuration metadata for the handlers used in the application pipelines and allow extending from it. The configuration metadata is set with some default configuration for every handler if needed.

As platform, we also would like define more than one profile as they may be different in some parts depending on requirements, for example on the protocol being used.

One profile can be defined as default, so that if a pipeline configuration does not specify any specific profile, the default one will be used to extend from. We would like to encourage a developer to use profiles and that’s why the framework does not allow a pipeline configuration without a profile, though one can define an empty profile and extend from it.

We may have service endpoint pipelines and service invocation pipelines. They are different in position of transport in the pipeline. The transport is a connector to the external world, where at the service endpoint it starts the pipeline while on the service invocation pipeline the transport sits at the end of the pipeline.

### Execution order

To make execution order manageable we would like to introduce order priority that defines a position of the handlers in the pipeline. This approach works really well in kraken-js framework that we use for our frontend applications. It is the simplest approach though it is possible to create a more complex based, for example, on a dependency tree, but it has some limitations and usually more difficult to resolve circular dependencies for application developers.

### Pipe configuration

This type of configuration is used by application teams to define their specific pipelines by extending from existing profiles. The application teams may modify existing profiles by providing more specific details to some handlers. For example, one can provide some connection information like hostname and port to the transport handler.

## Usage

### Bootstrap API

```js
const provider = require('trooba-bootstrap')(profiles, clients);
// Update route: if handler is already there, it will override it
// This can be used to listen to config changes and updating it on the fly
Provider.updateProfiles(profiles);
// if client is already there, it will override it
// This also can be used to listen to config changes and updating it on the fly
Provider.updatePipes(clients);
// Create a client and make a call
Provider.createClient('my-service-rest-client');
    .create({ 'some': 'context' })
    .request({foo:'bar'}, console.log);
```

#### Profile Configurations

```js
// define profiles for easy reference in pipeline definitions
const profiles = {
    // default profile to be used if no profile specified
    "default": {
        "trace": {
            // Priority defines execution order
            "priority": 10,
            "module": "trooba-opentrace"
        },
        "circuit": {
            "priority": 20,
            // module that provide circuit breaker functionality
            "module": "trooba-hystrix-handler",
            // default configuration
            "config": {
                "timeout": 0,
                "circuitBreakerErrorThresholdPercentage": 50
            }
        },
        "retry": {
            "priority": 30,
            "module": "trooba-retry" // TBD (to be defined/implemented)
        },
        "http-transport": {
            "transport": true,
            "module": "trooba-http-transport"
        }
    },
    // profile for soap service pipelines
    "soap": {
        "trace": {
            // Priority defines execution order
            "priority": 10,
            "module": "trooba-opentrace"
        },
        "circuit": {
            "priority": 20,
            // module that provide circuit breaker functionality
            "module": "trooba-hystrix-handler",
            // default configuration
            "config": {
                "timeout": 0,
                "circuitBreakerErrorThresholdPercentage": 70
            }
        },
        "retry": {
            "priority": 30,
            "module": "trooba-retry" // TBD (to be defined/implemented)
        },
        // We added new handler that adopts XML request to http request
        "soap": {
            "priority": 40,
            "module": "trooba-soap" // TBD
        }
        "http-transport": {
            "transport": true,
            "module": "trooba-http-transport"
        }
    }
};
```

#### Defining service pipelines

Service invocation pipe configuration extending from default profile

```js
const clients = {
    // id of the pipe
    "my-service-rest-client": {
        // since no profile given, it will use default profile
        // "$profile": "default",
        // custom configuration for handlers and pipe overall
        "circuit": {
            "config": {
                // modify circuit breaker property from 50 to 70
                "circuitBreakerErrorThresholdPercentage": 70
            }
        },
        "retry": {
            // change priority
            "priority": 35,
            "config": {
                // set retry to 3
                "retry": 3
            }
        },
        // provide transport service configuration
        // one can also provide this via pipe.context if transport understands it
        "http-transport": {
            "config": {
                "context": "my-app-http-context",
                "hostname": "localhost",
                "port": 8000,
                "protocol": "http:",
                "path": "/view/item",
                "socketTimeout": 2000
            }
        }
    },
    "my-service-soap-client": {
        // uses soap profile
        "$profile": "soap",
        // provide custom configuration for transport
        "http-transport": {
            "config": {
                "context": "my-app-http-context",
                "hostname": "localhost",
                "port": 8000,
                "protocol": "http:",
                "path": "/view/item",
                "socketTimeout": 2000
            }
        }
    },
    // we can extend existing profile for the service
    "my-service-extend-client": {
        // define custom pipeline
        "$profile": "soap",
        // adding custom metrics handler
        "metrics": {
            // want to go as close as possible to the transport in the pipeline
            "priority": 1000,
            "module": "metrics-module"
        },
        // configure transport
        "http-transport": {
            "config": {
                "context": "my-app-http-context",
                "hostname": "localhost",
                "port": 8000,
                "protocol": "http:"
            }
        }
    }
};
```

We hope this gives the reader a basic idea on how one can bootstrap the pipelines and encourage you to take a look at more examples provided at https://github.com/trooba/trooba-bootstrap

Now we are ready to jump into implementing some of the handlers for the trooba framework in the following chapters.
