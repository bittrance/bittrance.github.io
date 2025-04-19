---
title: Zero downtime shutdown for REST APIs
date: 2025-04-13T13:47:00+02:00
draft: false
summary:
  How to ensure that your Kubernetes deployment rolling restart lose zero requests.
authors:
  - Anders Qvist
tags:
  - zero-downtime
  - rest
  - http
  - design
keywords:
  - zero-downtime
  - rest
  - http
  - design
---
I fell into a rabbit hole the other day. I had thought Kubernetes was this fantastic thing that magically fixed all my graceful shutdown problems, making it trivial to achieve zero downtime deploys and node replacements. I would make sure to:

 1. close my listener socket,
 1. run all in-flight requests to completion,
 1. shut down supporting services,
 1. exit cleanly,

and Kubernetes would do the rest, no?

Alas, [distributed systems are hard](https://thenewstack.io/distributed-systems-hard/) to make predictable. The creators of Kubernetes know this better than most. Thus, Kuberntees de-registers the pod from the service (i.e. removes the Endpoint reaource) and sends it SIGTERM at the same time, making no effort to synchronize these events. Because of this design decision, there is a chance that a quick process may close its listener socket while the service still sends new connections to it, causing "connection refused" or similar errors for the client. Not ok.

![Shutdown race](/graceful-shutdown-for-rest-apis/shutdown-race.png "Shutdown race")

The conventional way to solve this problem is to use a pre-stop lifecycle hook to ask Kubernetes to delay SIGTERM a bit. This pattern was canonicalized in version 1.30 through [KEP 3960](https://github.com/kubernetes/enhancements/blob/master/keps/sig-node/3960-pod-lifecycle-sleep-action/README.md). You can add something like this to your pod spec:

```yaml
spec:
  containers:
  - name: api
    image: bittrance/hello-rest:0.3.0
    lifecycle:
      preStop:
        sleep:
          seconds: 5
```

Now Kubernetes gets some time to remove all the endpoints across all nodes and the pod should get no new traffic. Given that I follow the flow above, there should be zero downtime and ice cream for everyone?

![Pre-stop sleep](/graceful-shutdown-for-rest-apis/prestop-sleep.png "Pre-stop sleep")

Well, there is this rather oblique passage in the [Kubernetes pods docs](https://kubernetes.io/docs/concepts/workloads/pods/pod-lifecycle/):

> Some applications need to go beyond finishing open connections and need more graceful termination, for example, session draining and completion.

As it turns out, the de-registration race is not the main source of deployment-related errors in our clusters. Closing the window between endpoint deletion and SIGTERM is sufficient when each request arrives in a fresh TCP connection. However, if you live in microservice-land and use HTTP/1.1 REST APIs, you are probably using [persistent HTTP connections](https://datatracker.ietf.org/doc/html/rfc7230#section-6.3) when you are communicating between services. Modern HTTP clients typically default to "keep-alive" connections. They maintain the actual TCP connection behind the scene and only talk with developers in terms of requests. Here is a simple demonstration using [curl](https://curl.se); despite making four requests, [tcpdump](https://tcpdump.org) only sees a single SYN packet, that is, only one connection initiation:

![connection reuse](/graceful-shutdown-for-rest-apis/connection-reuse.png "Connection reuse")

This means that what happens to the listenting socket is not the whole story of graceful shutdown. You also need to gracefully shut down the persistent connections in a manner that minimize disruption. In the extreme scenario, where your REST API clients are other internal microservices using pools of persistent TCP connections, those clients may create their connections on startup and re-use across their full life cycle.

## The theory, or how to say goodbye?

There are two methods you can use to address persistent connections. I call them "immediate hang-up" and "one more message".

### Immediate hangup

With *immediate hangup*, you close all persistent connections as soon as they are idle. In many scenarios they will be idle most of the time, so this will be quick. Should a connection have an in-flight request, you wait until it is done and then immediately hang up. (This is effectively the "one more message" method.)

![immediate hangup](/graceful-shutdown-for-rest-apis/immediate-hangup.png "immediate hangup")

The benefit of this method is that it is fast; shutdown is very likely to be done before your p(95) latency has passed. The drawback is that since you "abruptly" close the connection, there is a small chance that there is a request coming down the wire that the backend has not yet seen. In a Kubernetes cluster, nodes may be in different data centers or availability zones, so the one-way latency is probably <1 ms, but if the client is a mobile app, this window may be many ms long.

This method makes sense when you have many persistent connections with few requests each, since that means the chance for an inbound request is small.

### One more message

The *one more message* method waits for one more request to arrive on the persistent connection. When responding, it sets the HTTP header `Connection: close` on the response. This indicates that the connection will be closed and cannot be used anymore. This behavior is specified in the [HTTP/1.1 RFC](https://www.rfc-editor.org/rfc/rfc9112.html#name-tear-down):

> A client that receives a "close" connection option MUST cease sending requests on that connection and close the connection after reading the response message containing the "close" connection option.

![connection close](/graceful-shutdown-for-rest-apis/connection-close.png "Connection close")

The upside of this method is that all connections that get a request will be closed with no risk of losing requests. The downside is that the shutdown becomes protracted. As long as there are persistent connections left, a request may show up and further delay shutdown. We could of course address this downside by setting a deadline beyond which we close all idle connections (i.e. use the "immediate hang-up" method), risking a few failed requests.

This method makes sense when your request volume is large. In particular, if you multiplex requests (a k a "layer 7" or "application" load balancer) across your backends, the request flood will be spread evenly across the connections and trigger their closure in short order.

It is also worth noting that this method can be implemented as a middleware (assuming we can request to close the current connection). The *immediate shutdown* method requires access to HTTP server internals in order to inspect persistent connections.

## The practice, or how can this *not* be default?

So much for workable shutdown strategies for persistent connections. What do existing REST API frameworks actually do?

| REST framework | Possible shutdown behavior | How to get it |
| ---- | ---- | ---- |
| Axum (Rust) | immediate shutdown | `axum::serve(...).with_graceful_shutdown(...)` |
| Express (node.js) | ... | ... |
| Flask/Gunicorn (Python) | immediate shutdown | `gunicorn --graceful-timeout <sec>` |
| Gin (Golang) | ... | ... |
| Spring boot/Tomcat (Java) | one more request | Switch to Jetty |
| quarkus (Java) | ... | ... |

### Axum (Rust)

The Rust ecosystem is blessed with a set of low-level HTTP libraries of outstanding quality. They make implementing REST API frameworks relatively straight-forward. The premiere framework is called [Axum](https://docs.rs/axum/latest/axum/). A trivial REST API with Axum 0.8 might look something like this (using async closures that were stabilized in 1.85):

```rs
let app = Router::new()
    .route(
        "/",
        get(async move || {
            sleep(request_delay).await;
            "Hello, World!"
        }),
    );
let listener = TcpListener::bind("0.0.0.0:8080").await.unwrap();
axum::serve(listener, app)
    .with_graceful_shutdown(shutdown_signal())
    .await
    .unwrap();
```

### Express (node.js)

The node.js ecosystem has many REST frameworks, but almost all of them utilize the [http.Server](https://nodejs.org/api/http.html#class-httpserver) which controls connections, so this analysis holds for almost all node.js REST frameworks.

The method `http.Server.close()` can be called to close a server gracefully. However, this method waits for all *connections* to be closed by the client, so a busy persistent connection will never be closed.

The only simple method I have found that gives a good shutdown behavior for persistent connections is the [terminus](https://github.com/godaddy/terminus) library. With terminus, the following code results in the *immediate shutdown* behavior:

```js
const app = express()
app.get('/', function (req, res) {
  setTimeout(() => res.send('Hello World\n'), request_delay);
})

const server = app.listen(8080);

createTerminus(server, {
  signals: ['SIGTERM', 'SIGINT'],
  useExit0: true,
  timeout: 10000,
});
```

### Flask/Gunicorn (Python)

Flask is the main REST framework in the Python ecosystem.

```py
app = Flask(__name__)
@app.route("/")
def hello_world():
    time.sleep(request_delay)
    return "Hello, World!"
```

Flask does not do graceful shutdown by itself. However, that does not matter so much; because of Python's memory-intensive and single-threaded nature, production deployments of Flask workloads typically use a "pre-fork" helper. Since uwsgi went into maintenance mode, [gunicorn](https://docs.gunicorn.org/en/stable/) is the go-to solution in this space. 

```shell
gunicorn --workers=4 --bind=127.0.0.1:8080 --graceful-timeout=10 'app:app'
```

### Gin (Golang)

Most web frameworks use [http.Server](https://pkg.go.dev/net/http#Server) from the Go standard library. I chose Gin for this exercise, but this section speaks for most Golang web framework. http.Server has a method `Shutdown()` which performs graceful shutdown. The core implementation of a REST service looks something like this:

```go
router := gin.Default()
router.GET("/", func(c *gin.Context) {
  time.Sleep(time.Duration(request_delay * float64(time.Second)))
  c.String(http.StatusOK, "Hello world!")
})

server := &http.Server{
  Addr:    ":8080",
  Handler: router,
  // You probably also want to set various timeout values
}
```

Shutdown works by first closing the listener socket, then closing all idle connections, and then waiting for connections to return to idle and then shut down. In Golang, the convention is to implement this sort of logic without leaning on dependencies. The `Shutdown()` documentation has an [example](ihttps://pkg.go.dev/net/http#example-Server.Shutdown) we can use. However, it is a bit simplistic:

- in production, the procees will probably be terminated with `SIGTERM` rather than ctrl+c, and
- we do not want to wait forever for shutdown to occur.

A production-ready version might look something like this:

```go
shutdownComplete := make(chan struct{})
go func() {
  quit := make(chan os.Signal, 1)
  signal.Notify(quit, syscall.SIGINT, syscall.SIGTERM)
  <-quit

  grace_period := time.Duration(request_delay * 2.0 * float64(time.Second))
  ctx, cancel := context.WithTimeout(context.Background(), grace_period)
  defer cancel()
  if err := server.Shutdown(ctx); err != nil {
    log.Printf("HTTP server Shutdown: %v", err)
  }
  close(shutdownComplete)
}()

if err := server.ListenAndServe(); err != nil && err != http.ErrServerClosed {
  log.Fatalf("listen: %s\n", err)
}
<-shutdownComplete
```

This results in the *immediate shutdown* behavior.

### Spring Boot (Java)

It turns out that of the available HTTP servers that Spring Boot includes support for, [only Jetty provides a reasonable behavior](https://github.com/spring-projects/spring-boot/issues/40108) for persistent connections. 

 - **Tomcat** will keep persistent connections open while running in-flight requests, but will close the connection on inbound traffic, resulting in EOF errors on the client. I have failed find an explanation for this non-sensical behavior. This behavior is [hard-coded](https://github.com/apache/tomcat/blob/e770b48dd0064f7f21d67278e73f5c1ac30460bf/java/org/apache/tomcat/util/net/Acceptor.java#L152).

 - **Undertow** will keep persistent connections open while running in-flight requests, but will respond to requests with 503 and keep the connection open. Presumably, this behavior is motivated by using Undertow as an app container, where apps can be added and removed. This behavior is [hard-coded](https://github.com/undertow-io/undertow/blob/85cb914965b24bcd3e3bd6075597ac1d031fa94e/core/src/main/java/io/undertow/server/handlers/GracefulShutdownHandler.java#L89).

 - **Jetty** will use the *one more request* strategy as outlined above.

Spring Boot services can also be written in "reactive" style. They are mostly using Netty, which I have not explored during this investigation.

### Quarkus (Java)

Quarkus is the newish kid on the block in the Java world. Given the sorry state of graceful shutdown in the Spring Boot ecosystem, I wanted to explore whether Quarkus could be an alternative.

Indeed, Quarkus docs has a section on [graceful shutdown](https://quarkus.io/version/3.15/guides/lifecycle#graceful-shutdown). By setting the `graceful.shutdown.timeout` in your `application.properties` Quarkus should do graceful shutdown. Alas, Quarkus replicates Undertow's problematic behavior of keeping idle sockets open during shutdown and [returning status code 503](https://github.com/quarkusio/quarkus/blob/006177281dbc82b3cb67547b153b78f5172f9631/extensions/vertx-http/runtime/src/main/java/io/quarkus/vertx/http/runtime/filters/GracefulShutdownFilter.java#L44) to any late-arriving requests. Unlike Undertow, it at least closes the connection at this point, limiting the number of failures you get.

## Conclusions

The first, obvious conclusion, is that no ecosystem defaults to graceful shutdown. Indeed node.js, the world's largest programming ecosystem, does not even provide a mechanism to perform graceful shutdown for persistent connections through its standard library. Even some ecosystems that ostensibly provide graceful shutdown, such as Spring Boot, does it badly. Only the newer kids on the block (Axum, Golang) provides simple methods to ensure a fully graceful shutdown behavior.

The other conclusion is that getting graceful shutdown completely right is hard. For example, even the better implementations investigated above will shut down a connection that is in the process of receiving HTTP headers, thus resulting in the occasional EOF error to the client on backend shutdown. Only when the headers are completely received will the request be considered as an in-flight request.

If your ecosystem does not provide good support for graceful shutdown on HTTP/1.1, you may be able to work around this by switching to HTTP/2 where [shutdown behavior is strictly defined](https://datatracker.ietf.org/doc/html/rfc9113#name-goaway).

Clearly, writing highly available services remains niche in the minds of many framework developers.
