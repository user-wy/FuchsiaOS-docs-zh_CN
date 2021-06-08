# Implement a FIDL client

## Prerequisites

This tutorial builds on the [FIDL server][server-tut] tutorial. For the
full set of FIDL tutorials, refer to the [overview][overview].

## Overview

This tutorial implements a client for a FIDL protocol and runs it
against the server created in the [previous tutorial][server-tut]. The client in this
tutorial is asynchronous. There is an [alternate tutorial][sync-client] for
synchronous clients.

If you want to write the code yourself, delete the following directories:

```
rm -r examples/fidl/hlcpp/client/*
```

## Create a stub component

Note: If necessary, refer back to the [previous tutorial][server-tut-component].

1. Set up a hello world component in `examples/fidl/hlcpp/client`.
   You can name the component `echo-client`, and give the package a name of
   `echo-hlcpp-client`.

1. Once you have created your component, ensure that the following works:

   ```
   fx set core.x64 --with //examples/fidl/hlcpp/client
   ```

1. Build the Fuchsia image:

   ```
   fx build
   ```

1. In a separate terminal, run:

   ```
   fx serve
   ```

1. In a separate terminal, run:

   ```
   fx shell run fuchsia-pkg://fuchsia.com/echo-hlcpp-client#meta/echo-client.cmx
   ```

## Edit GN dependencies

1. Add the following dependencies:

   ```gn
   {%includecode gerrit_repo="fuchsia/fuchsia" gerrit_path="examples/fidl/hlcpp/client/BUILD.gn" region_tag="deps" %}
   ```

1. Then, include them in `main.cc`:

   ```cpp
   {%includecode gerrit_repo="fuchsia/fuchsia" gerrit_path="examples/fidl/hlcpp/client/main.cc" region_tag="includes" %}
   ```

   The reason for including these dependencies is explained in the
   [server tutorial][server-tut-deps].

## Edit component manifest

1. Include the `Echo` protocol in the client component's sandbox by
   editing the component manifest in `client.cmx`.

   ```cmx
   {%includecode gerrit_repo="fuchsia/fuchsia" gerrit_path="examples/fidl/hlcpp/client/client.cmx" %}
   ```

## Connect to the server {#main}

The steps in this section explain how to add code to the `main()` function
that connects the client to the server and makes requests to it.

### Initialize the event loop

As in the server, the code first sets up an async loop so that the client can
listen for incoming responses from the server without blocking.

```cpp
{%includecode gerrit_repo="fuchsia/fuchsia" gerrit_path="examples/fidl/hlcpp/client/main.cc" region_tag="main" highlight="2,28" %}
```

### Initialize a proxy class {#proxy}

In the context of FIDL, proxy designates the code
generated by the FIDL bindings that enables users to make
remote procedure calls to the server. In HLCPP, the proxy takes the form
of a class with methods corresponding to each FIDL protocol method.

The code then creates a proxy class for the `Echo` protocol, and connects it
to the server.

```cpp
{%includecode gerrit_repo="fuchsia/fuchsia" gerrit_path="examples/fidl/hlcpp/client/main.cc" region_tag="main" highlight="4,5,6" %}
```

* [`fuchsia::examples::EchoPtr`][proxy] is an alias for
  `fidl::InterfaceRequest<fuchsia::examples::Echo>` generated by the bindings.
* Analogous to the `fidl::Binding<fuchsia::examples::Echo>` used in the server,
  `fidl::InterfaceRequest<fuchsia::examples::Echo>` is parameterized by a FIDL
  protocol and a channel it will proxy requests over the channel, and listen for
  incoming responses and events.
* The code calls `EchoPtr::NewRequest()`, which creates a channel,
  binds the class to one end of the channel, and returns the other end of the
  channel.
* The returned end of the channel is passed to `sys::ServiceDirectory::Connect()`.
  * Analogous to the call to `context->out()->AddPublicService()` on the server
    side, `Connect` has an implicit second parameter here, which is the protocol
    name (`"fuchsia.examples.Echo"`). This is where the input to the handler
    defined in the [previous tutorial][server-tut-handler] comes from: the
    client passes it in to `Connect`, which then passes it to the handler.

An important point to note here is that this code assumes that `/svc` already
contains an instance of the `Echo` protocol. This is not the case by default
because of the sandboxing provided by the component framework.

### Set an error handler

Finally, the code sets an error handler for the proxy:

```cpp
{%includecode gerrit_repo="fuchsia/fuchsia" gerrit_path="examples/fidl/hlcpp/client/main.cc" region_tag="main" highlight="8,9,10" %}
```

### Send requests to the server

The code makes two requests to the server:

* An `EchoString` request
* A `SendString` request

```cpp
{%includecode gerrit_repo="fuchsia/fuchsia" gerrit_path="examples/fidl/hlcpp/client/main.cc" region_tag="main" highlight="14,15,16,17,18,19,20" %}
```

For `EchoString` the code passes in a callback to handle the response.
`SendString` does not require such a callback because the method does not
have any response.

### Set an event handler

The code then sets a handler for any incoming `OnString` events:

```cpp
{%includecode gerrit_repo="fuchsia/fuchsia" gerrit_path="examples/fidl/hlcpp/client/main.cc" region_tag="main" highlight="21,22,23,24,25,26" %}
```

### Terminate the event loop

The code waits to receive both a response to the `EchoString` method as well as an
`OnString` event (which in the [current implementation][server-tut-impl] is sent after receiving a
`SendString` request) before quitting from the loop. The code returns a successful exit
code only if it receives both a response and an event:

```cpp
{%includecode gerrit_repo="fuchsia/fuchsia" gerrit_path="examples/fidl/hlcpp/client/main.cc" region_tag="main" highlight="13,17,18,23,24,29" %}
```

## Run the client

If you run the client directly, the error handler gets called because the
client does not automatically get the `Echo` protocol provided in its
sandbox (in `/svc`). To get this to work, a launcher tool is provided
that launches the server, creates a new [`Environment`][environment] for
the client that provides the server's protocol, then launches the client in it.

1. Configure your GN build as follows:

    ```
    fx set core.x64 --with //examples/fidl/hlcpp/server --with //examples/fidl/hlcpp/client --with //examples/fidl/test:echo-launcher
    ```

2. Build the Fuchsia image:

   ```
   fx build
   ```

3. Run the launcher by passing it the client URL, the server URL, and
   the protocol that the server provides to the client:

    ```
    fx shell run fuchsia-pkg://fuchsia.com/echo-launcher#meta/launcher.cmx fuchsia-pkg://fuchsia.com/echo-hlcpp-client#meta/echo-client.cmx fuchsia-pkg://fuchsia.com/echo-hlcpp-server#meta/echo-server.cmx fuchsia.examples.Echo
    ```

You should see the client print output in the QEMU console (or using `fx log`).

```
[117659.968] 754089:754091> Running echo server
[117659.978] 754194:754196> Got event hi
[117659.978] 754194:754196> Got response hello
```

<!-- xrefs -->
[server-tut]: /docs/development/languages/fidl/tutorials/hlcpp/basics/server.md
[server-tut-component]: /docs/development/languages/fidl/tutorials/hlcpp/basics/server.md#component
[server-tut-impl]: /docs/development/languages/fidl/tutorials/hlcpp/basics/server.md#impl
[server-tut-deps]: /docs/development/languages/fidl/tutorials/hlcpp/basics/server.md#dependencies
[server-tut-handler]: /docs/development/languages/fidl/tutorials/hlcpp/basics/server.md#handler
[sync-client]: /docs/development/languages/fidl/tutorials/hlcpp/basics/sync_client.md
[proxy]: /docs/reference/fidl/bindings/hlcpp-bindings.md#protocols-client
[overview]: /docs/development/languages/fidl/tutorials/overview.md
[environment]: /docs/concepts/components/v2/environments.md