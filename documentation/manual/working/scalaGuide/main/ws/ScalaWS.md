<!--- Copyright (C) 2009-2016 Typesafe Inc. <http://www.typesafe.com> -->
# The Play WS API

Sometimes we would like to call other HTTP services from within a Play application. Play supports this via its [WS library](api/scala/play/api/libs/ws/package.html), which provides a way to make asynchronous HTTP calls.

There are two important parts to using the WS API: making a request, and processing the response.  We'll discuss how to make both GET and POST HTTP requests first, and then show how to process the response from WS.  Finally, we'll discuss some common use cases.

## Making a Request

To use WS, first add `ws` to your `build.sbt` file:

```scala
libraryDependencies ++= Seq(
  ws
)
```

Now any controller or component that wants to use WS will have to declare a dependency on the `WSClient`:

@[dependency](code/ScalaWSSpec.scala)

We've called the `WSClient` instance `ws`, all the following examples will assume this name.

To build an HTTP request, you start with `ws.url()` to specify the URL.

@[simple-holder](code/ScalaWSSpec.scala)

This returns a [WSRequest](api/scala/play/api/libs/ws/WSRequest.html) that you can use to specify various HTTP options, such as setting headers.  You can chain calls together to construct complex requests.

@[complex-holder](code/ScalaWSSpec.scala)

You end by calling a method corresponding to the HTTP method you want to use.  This ends the chain, and uses all the options defined on the built request in the `WSRequest`.

@[holder-get](code/ScalaWSSpec.scala)

This returns a `Future[WSResponse]` where the [Response](api/scala/play/api/libs/ws/WSResponse.html) contains the data returned from the server.

### Request with authentication

If you need to use HTTP authentication, you can specify it in the builder, using a username, password, and an [AuthScheme](api/scala/play/api/libs/ws/WSAuthScheme.html).  Valid case objects for the AuthScheme are `BASIC`, `DIGEST`, `KERBEROS`, `NONE`, `NTLM`, and `SPNEGO`.

@[auth-request](code/ScalaWSSpec.scala)

### Request with follow redirects

If an HTTP call results in a 302 or a 301 redirect, you can automatically follow the redirect without having to make another call.

@[redirects](code/ScalaWSSpec.scala)

### Request with query parameters

Parameters can be specified as a series of key/value tuples.

@[query-string](code/ScalaWSSpec.scala)

### Request with additional headers

Headers can be specified as a series of key/value tuples.

@[headers](code/ScalaWSSpec.scala)

If you are sending plain text in a particular format, you may want to define the content type explicitly.

@[content-type](code/ScalaWSSpec.scala)

### Request with virtual host

A virtual host can be specified as a string.

@[virtual-host](code/ScalaWSSpec.scala)

### Request with timeout

If you wish to specify a request timeout, you can use `withRequestTimeout` to set a value. An infinite timeout can be set by passing `Duration.Inf`.

@[request-timeout](code/ScalaWSSpec.scala)

### Submitting form data

To post url-form-encoded data a `Map[String, Seq[String]]` needs to be passed into `post`.

@[url-encoded](code/ScalaWSSpec.scala)

### Submitting JSON data

The easiest way to post JSON data is to use the [[JSON|ScalaJson]] library.

@[scalaws-post-json](code/ScalaWSSpec.scala)

### Submitting XML data

The easiest way to post XML data is to use XML literals.  XML literals are convenient, but not very fast.  For efficiency, consider using an XML view template, or a JAXB library.

@[scalaws-post-xml](code/ScalaWSSpec.scala)

### Streaming data

It's also possible to stream data.

For example, imagine you have executed a database query that is returning a large image, and you would like to forward that data to a different endpoint for further processing. Ideally, if you can send the data as you receive it from the database, you will reduce latency and also avoid problems resulting from loading in memory a large set of data. If your database access library supports [Reactive Streams](http://www.reactive-streams.org/) (for instance, [Slick](http://slick.typesafe.com/) does), here is an example showing how you could implement the described behavior:

@[scalaws-stream-request](code/ScalaWSSpec.scala)

The `largeImageFromDB` in the code snippet above is an Akka Streams `Source[ByteString, _]`.

## Processing the Response

Working with the [Response](api/scala/play/api/libs/ws/WSResponse.html) is easily done by mapping inside the [Future](http://www.scala-lang.org/api/current/index.html#scala.concurrent.Future).

The examples given below have some common dependencies that will be shown once here for brevity.

Whenever an operation is done on a `Future`, an implicit execution context must be available - this declares which thread pool the callback to the future should run in.  You can inject the default Play execution context in your DI-ed class by declaring an additional dependency to `ExecutionContext` in the class' constructor:

@[scalaws-context-injected](code/ScalaWSSpec.scala)

If you are not using DI, you can still access the default Play execution context:

@[scalaws-context](code/ScalaWSSpec.scala)

The examples also use the folowing case class for serialization / deserialization:

@[scalaws-person](code/ScalaWSSpec.scala)

### Processing a response as JSON

You can process the response as a [JSON object](api/scala/play/api/libs/json/JsValue.html) by calling `response.json`.

@[scalaws-process-json](code/ScalaWSSpec.scala)

The JSON library has a [[useful feature|ScalaJsonCombinators]] that will map an implicit [`Reads[T]`](api/scala/play/api/libs/json/Reads.html) directly to a class:

@[scalaws-process-json-with-implicit](code/ScalaWSSpec.scala)

### Processing a response as XML

You can process the response as an [XML literal](http://www.scala-lang.org/api/current/index.html#scala.xml.NodeSeq) by calling `response.xml`.

@[scalaws-process-xml](code/ScalaWSSpec.scala)

### Processing large responses

Calling `get()`, `post()` or `execute()` will cause the body of the response to be loaded into memory before the response is made available.  When you are downloading a large, multi-gigabyte file, this may result in unwelcomed garbage collection or even out of memory errors.

`WS` lets you consume the response's body incrementally by using an Akka Streams `Sink`.  The `stream()` method on `WSRequest` returns a `Future[StreamedResponse]`. A `StreamedResponse` is a simple container holding together the response's headers and body.

Here is a trivial example that uses a folding `Sink` to count the number of bytes returned by the response:

@[stream-count-bytes](code/ScalaWSSpec.scala)

Alternatively, you could also stream the body out to another location. For example, a file:

@[stream-to-file](code/ScalaWSSpec.scala)

Another common destination for response bodies is to stream them back from a controller's `Action`:

@[stream-to-result](code/ScalaWSSpec.scala)

As you may have noticed, before calling `stream()` we need to set the HTTP method to use by calling `withMethod` on the request. Here follows another example that uses `PUT` instead of `GET`:

@[stream-put](code/ScalaWSSpec.scala)

Of course, you can use any other valid HTTP verb.

## Common Patterns and Use Cases

### Chaining WS calls

Using for comprehensions is a good way to chain WS calls in a trusted environment.  You should use for comprehensions together with [Future.recover](http://www.scala-lang.org/api/current/index.html#scala.concurrent.Future) to handle possible failure.

@[scalaws-forcomprehension](code/ScalaWSSpec.scala)

### Using in a controller

When making a request from a controller, you can map the response to a `Future[Result]`. This can be used in combination with Play's `Action.async` action builder, as described in [[Handling Asynchronous Results|ScalaAsync]].

@[async-result](code/ScalaWSSpec.scala)

## Using WSClient

WSClient is a wrapper around the underlying AsyncHttpClient.  It is useful for defining multiple clients with different profiles, or using a mock.

You can define a WS client directly from code without having it injected by WS, and then use it implicitly with `WS.clientUrl()`:

@[implicit-client](code/ScalaWSSpec.scala)

> NOTE: if you instantiate a AhcWSClient object, it does not use the WS module lifecycle, and so will not be automatically closed in `Application.onStop`. Instead, the client must be manually shutdown using `client.close()` when processing has completed. This will release the underlying ThreadPoolExecutor used by AsyncHttpClient. Failure to close the client may result in out of memory exceptions (especially if you are reloading an application frequently in development mode).

or directly:

@[direct-client](code/ScalaWSSpec.scala)

Or use a magnet pattern to match up certain clients automatically:

@[pair-magnet](code/ScalaWSSpec.scala)

By default, configuration happens in `application.conf`, but you can also set up the builder directly from configuration:

@[programmatic-config](code/ScalaWSSpec.scala)

You can also get access to the underlying [AsyncHttpClient](http://static.javadoc.io/org.asynchttpclient/async-http-client/2.0.0-alpha27/org/asynchttpclient/AsyncHttpClient.html).

@[underlying](code/ScalaWSSpec.scala)

This is important in a couple of cases.  WS has a couple of limitations that require access to the client:

* `WS` does not support multi part form upload directly.  You can use the underlying client with [RequestBuilder.addBodyPart](http://static.javadoc.io/org.asynchttpclient/async-http-client/2.0.0-alpha27/org/asynchttpclient/RequestBuilderBase.html#addBodyPart-org.asynchttpclient.request.body.multipart.Part-).
* `WS` does not support streaming body upload.  In this case, you should use the `FeedableBodyGenerator` provided by AsyncHttpClient.

## Configuring WS

Use the following properties in `application.conf` to configure the WS client:

* `play.ws.followRedirects`: Configures the client to follow 301 and 302 redirects *(default is **true**)*.
* `play.ws.useProxyProperties`: To use the system http proxy settings(http.proxyHost, http.proxyPort) *(default is **true**)*.
* `play.ws.useragent`: To configure the User-Agent header field.
* `play.ws.compressionEnabled`: Set it to true to use gzip/deflater encoding *(default is **false**)*.

### Configuring WS with SSL

To configure WS for use with HTTP over SSL/TLS (HTTPS), please see [[Configuring WS SSL|WsSSL]].

### Configuring Timeouts

There are 3 different timeouts in WS. Reaching a timeout causes the WS request to interrupt.

* `play.ws.timeout.connection`: The maximum time to wait when connecting to the remote host *(default is **120 seconds**)*.
* `play.ws.timeout.idle`: The maximum time the request can stay idle (connection is established but waiting for more data) *(default is **120 seconds**)*.
* `play.ws.timeout.request`: The total time you accept a request to take (it will be interrupted even if the remote host is still sending data) *(default is **120 seconds**)*.

The request timeout can be overridden for a specific connection with `withRequestTimeout()` (see "Making a Request" section).

### Configuring AsyncHttpClientConfig

The following advanced settings can be configured on the underlying AsyncHttpClientConfig.
Please refer to the [AsyncHttpClientConfig Documentation](http://static.javadoc.io/org.asynchttpclient/async-http-client/2.0.0-alpha27/org/asynchttpclient/DefaultAsyncHttpClientConfig.Builder.html) for more information.

* `play.ws.ahc.allowPoolingConnection`
* `play.ws.ahc.allowSslConnectionPool`
* `play.ws.ahc.ioThreadMultiplier`
* `play.ws.ahc.maxConnectionsPerHost`
* `play.ws.ahc.maxConnectionsTotal`
* `play.ws.ahc.maxConnectionLifeTime`
* `play.ws.ahc.idleConnectionInPoolTimeout`
* `play.ws.ahc.webSocketIdleTimeout`
* `play.ws.ahc.maxNumberOfRedirects`
* `play.ws.ahc.maxRequestRetry`
* `play.ws.ahc.disableUrlEncoding`