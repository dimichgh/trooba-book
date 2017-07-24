# Pipeline Lifecycle

Here we will explore two main use cases where pipelines are used: service and client.
Based on the type of the pipeline, the lifecycles can be different.

Other use-cases are possible like pub/sub with one way transmission or transformer pipeline that have no service flow and simply used to transform the data.

The differences are defined by transport, its position in the pipeline (at the start or end of the pipe) and execution order of handlers.

## The service invocation lifecycle (Client API)

The service invocation is always initiated by application code which is already running in the specific context that can be injected into the pipeline context to make sure the context is propagated to the downstream services.

* Pipe construction
    * Transport or some other handler can inject custom API that one then use after construction
* Execution
    * Context injection
    * Request phase
        * Initialization phase
            * Request hook: wait for request event
            * Response hook, wait for response event
            * Error hook: wait for error event
        * Execution phase
            * On request event
            * Continue or throw error
    * Transport phase
        * Initialization phase
            * Request hook
        * Execution phase
            * On request event
            * Response initiation or throw error
    * Response phase (optional in pub/sub use-case)
        * Pipe handler execution in reverse order
            * Hook: on response event
            * Hook: on error event - continue or retry
    * Completion
        * Should be marked by either response or some other event in case of streams (data === undefined)

## The service endpoint lifecycle (Service API)

The service endpoint is any http/soap/grpc/websocket/etc service that accepts requests from the external world and executes some logic based on request type, path and other request parameters. The most popular one is express or hapi frameworks that provide this type of functionality.

* Pipe construction
    * Transport initialization
    * Service API injection, generic API (listen, close)
* Application creation, using service API
    * Execution
        * Request phase
            * Route selection
            * Pipe construction based on route, unless already created
            * Pipe context injection
            * Pipe execution for given request
    * Shutdown phase
        * Graceful
        * Forced termination
    * Closed

The main cycle will always start from the client code initiating the call with injected context for service calls or an external request coming to the service from the outside world.

The request phase (direction) will always be present, while response phase is optional in case of pub/sub use-case.

Request and response phases can be repeated partially in case a specific handlers, for example, ‘retry’ handler decides to do so based on the error received from the last response.

The framework does not allow registering more than one hook for every type of the event in the same pipe point/handler. We would like to avoid splitting the flow that may be caused by multiple listeners hooking the the same event in the same pipe point. This simplifies the flow. In case it detects existing hook, it will throw Error.
