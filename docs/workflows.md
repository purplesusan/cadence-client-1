# Workflows

A workflow is the implementation of coordination logic. The Cadence programming framework (the client
library) allows you to write the coordination logic as simple procedural code that uses standard Go
data modeling. The client library takes care of the communication between the worker service and the
Cadence service, and ensures state persistence between events even in case of worker failures. Furthermore,
any particular execution is not tied to a particular worker machine. Different steps of the coordination
logic execute on different worker instances, with the framework ensuring that the necessary state is
recreated on the worker executing the step.

To facilitate this operational model, both the Cadence programming framework and the managed service
impose some requrirements and restrictions on the implementation of the coordination logic. These
requirements and restrictions are outlined in the [Implementation](#Implementation) section.

## Overview

The sample workflow provided in the [README](../README.md) demonstrates a simple implementation of a
workflow that executes one activity. The workflow also passes the sole parameter that it receives as
part of its initialization as a parameter to the activity.

Let's take a look at each component of this workflow.

## Declaration

In the Cadence programming model, a workflow is implemented with a function. The function declaration
specifies the parameters the workflow accepts as well as any values it might return.

`func SimpleWorkflow(ctx workflow.Context, value string) error`

The first parameter to the function is `ctx workflow.Context`. This is a required parameter for all
workflow functions and is used by the Cadence client library to pass execution context. Virtually
all of the functions in the client library that are callable from workflow functions require this `ctx`
parameter. This context parameter is the same concept as the standard `context.Context` provided by Go.
The only difference between `workflow.Context` and `context.Context` is that the `Done()` function in
`workflow.Context` returns `workflow.Channel` instead the standard Go `chan`.

The second parameter is a custom workflow `string` parameter that can be used to pass data into the
workflow on start. A workflow can have one or more such parameters. All parameters to a workflow function
must be serializable, which means that params can’t be channels, functions, variadic, or unsafe pointers.

Since the function only declares error as the return value, the workflow does not return a value. The
error return value is used to indicate whether an error was encountered during execution and the workflow
should be terminated.

## Implementation

To support the synchronous and sequential programming model for the workflow implementation, there are
restrictions and requirements on how the workflow implementation must behave in order to guarantee
correctness. The requirements are that workflow execution must be:

* Deterministic
* Idempotent

To meet these requirements, your workflow code should follow these guidelines:

* Use `workflow.Context` everywhere.
* Don’t use range over `map`.
* Use `workflow.SideEffect` to call rand and similar nondeterministic functions like UUID generator.
* Use `workflow.Now` to get the current time. Use `workflow.NewTimer` or `workflow.Sleep` instead of standard Go functions.
* Don’t use native channel and select. Use `workflow.Channel` and `workflow.Selector`.
* Don’t use go func(...). Use `workflow.Go(func(...))`.
* Don’t use non-constant global variables as multiple instances of a workflow function can be executing in parallel.
* Don’t use any blocking functions besides belonging to `Channel`, `Selector` or `Future`.
* Don’t use any synchronization primitives as they can cause blockage and there is no possibility of races when running under dispatcher.
* Don't just change workflow code when there are open workflows. Always update code using `workflow.GetVersion`.
* Don’t perform any IO or service calls as they are not usually deterministic. Use activities for that.
* Don’t access configuration APIs directly from a workflow as changes in configuration will affect the workflow execution path. Either return configuration from an activity or use `workflow.SideEffect` to load it.

## Registration

In order to make the workflow visible to the worker process hosting it, the workflow needs to be registered via a call to `workflow.Register`.

```go
func init() {
	workflow.Register(SimpleWorkflow)
}
```
## Signals

Signals provide a fully asynchronous and durable mechanism for providing data to a running workflow.
When a signal is received for a running workflow, Cadence persists the event and the payload in the
workflow history. The workflow can then process the signal at any time afterwards without the risk of
losing the information. The workflow also has the option to stop execution by blocking on a signal
channel.

In the following example, the code uses `workflow.GetSignalChannel` to open a `workflow.Channel` for
the named signal. Then, it uses `workflow.Selector` to wait on that channel and process the payload
received with the signal.

```go
var signalVal string
signalChan := workflow.GetSignalChannel(ctx, signalName)

s := workflow.NewSelector(ctx)
s.AddReceive(signalChan, func(c workflow.Channel, more bool) {
        c.Receive(ctx, &signalVal)
        workflow.GetLogger(ctx).Info("Received signal!", zap.String("signal", signalName), zap.String("value", signalVal))
})
s.Select(ctx)

if len(signalVal) > 0 && signalVal != "SOME_VALUE" {
        return errors.New("signalVal")
}
```

## ContinueAsNew workflow completion

Workflows that need to rerun periodically could naively be implemented as a big **for loop** with a
**sleep** where the entire logic of the workflow is inside the body of the **for loop**. The problem
with this approach is that the history for that workflow will keep growing to a point where it reaches
the size maximum enforced by the service.

`ContinueAsNew` is the low-level construct that enables implementing such workflows without the risk
of failures down the road. The operation atomically completes the current execution and starts a new
execution of the workflow with the same workflow ID. The new execution will not carry over any history
from the old execution. To trigger this behavior, the workflow function should terminate by returning
the special `ContinueAsNewError` error:

```go
func SimpleWorkflow(workflow.Context ctx, value string) error {
    …
    return workflow.NewContinueAsNewError(ctx, SimpleWorkflow, value)
}
```
## Special Cadence client library functions and types

The Cadence client library provides a number of functions and types as alternatives to some native Go
functions and types. Using these replacement functions and types is necessary in order to ensure that
he workflow code execution is deterministic and repeatable within an execution context.

Coroutine related constructs:
* `workflow.Go`: This is a replacement for the the Go statement.
* `workflow.Channel`: This is a replacement for the native `chan` type. Cadence provides support for
both buffered and unbuffered channels.
* `workflow.Selector`: This is a replacement for the `select` statement.

Time related functions:
* `workflow.Now()`: This is a replacement for `time.Now()`.
* `workflow.Sleep()`: This is a replacement for `time.Sleep()`.

