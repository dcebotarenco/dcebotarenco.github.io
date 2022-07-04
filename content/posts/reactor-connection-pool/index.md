---
title: "🍃 Spring WebClient & Connection Pool"
date: 2022-06-29T09:37:22+03:00
draft: false
---


# 🧑‍🎓 tltr
[Configure](https://projectreactor.io/docs/netty/release/reference/index.html#connection-pool-timeout) `evictInBackground`, `maxIdleTime` and `maxLifeTime` to clear connections from the connection pool or [retry](https://projectreactor.io/docs/core/release/reference/#_retrying) the call or use [RestTemplate](https://docs.spring.io/spring-framework/docs/current/reference/html/web.html#webmvc-resttemplate)

# 🐛 Issue
At one of my clients, it was decided no more `RestTemplate`, all move to 🍃 [*Spring WebClient*](https://docs.spring.io/spring-boot/docs/2.0.x/reference/html/boot-features-webclient.html). I consider this project very interesting, but it comes with a different mindset which developers do not pay attention to:
1. Non-blocking
2. Event driven
3. Connection pool

Back to the client. \
After each deployment, in a matter of days, we were getting - the connection reset by the peer. The server on the other side was resetting the connection. Our connection is not valid anymore. \
Here is the exception example:

```java
reactor.core.Exceptions$ReactiveException: io.netty.channel.unix.Errors$NativeIoException: readAddress(..) failed: Connection reset by peer
        at reactor.core.Exceptions.propagate(Exceptions.java:392)
        at reactor.core.publisher.BlockingSingleSubscriber.blockingGet(BlockingSingleSubscriber.java:97)
        at reactor.core.publisher.Mono.block(Mono.java:1706)
```

### Logs
IPs are cleared.
```
./catalina.out-20211223.gz:2021-12-22 19:04:34.458 DEBUG --- [or-http-epoll-3] reactor.netty.http.client.HttpClient     : [30857d36] REGISTERED
./catalina.out-20211223.gz:2021-12-22 19:04:34.460 DEBUG --- [or-http-epoll-3] reactor.netty.http.client.HttpClient     : [30857d36] CONNECT: rest.apisandbox.server.com/1.1.1.1:443

...time...

./catalina.out-20211224.gz:2021-12-23 08:26:16.138 DEBUG --- [or-http-epoll-3] reactor.netty.http.client.HttpClient     : [30857d36-92, L:/0.0.0.0:55868 - R:rest.apisandbox.server.com/1.1.1.1:443] READ COMPLETE
./catalina.out-20211224.gz:2021-12-23 08:26:16.138 DEBUG --- [or-http-epoll-3] reactor.netty.http.client.HttpClient     : [30857d36-92, L:/0.0.0.0:55868 - R:rest.apisandbox.server.com/1.1.1.1:443] EXCEPTION: io.netty.channel.unix.Errors$NativeIoException: readAddress(..) failed: Connection reset by peer
./catalina.out-20211224.gz:2021-12-23 08:26:16.139  WARN --- [or-http-epoll-3] r.netty.http.client.HttpClientConnect    : [30857d36-92, L:/0.0.0.0:55868 - R:rest.apisandbox.server.com/1.1.1.1:443] The connection observed an error, the request cannot be retried as the headers/body were sent
./catalina.out-20211224.gz:2021-12-23 08:26:16.139 DEBUG --- [or-http-epoll-3] reactor.netty.http.client.HttpClient     : [30857d36-92, L:/0.0.0.0:55868 ! R:rest.apisandbox.server.com/1.1.1.1:443] USER_EVENT: SslCloseCompletionEvent(java.nio.channels.ClosedChannelException)
./catalina.out-20211224.gz:2021-12-23 08:26:16.139 DEBUG --- [or-http-epoll-3] reactor.netty.http.client.HttpClient     : [30857d36-92, L:/0.0.0.0:55868 ! R:rest.apisandbox.server.com/1.1.1.1:443] INACTIVE
./catalina.out-20211224.gz:2021-12-23 08:26:16.139 DEBUG --- [or-http-epoll-3] reactor.netty.http.client.HttpClient     : [30857d36-92, L:/0.0.0.0:55868 ! R:rest.apisandbox.server.com/1.1.1.1:443] UNREGISTERED
```

# 🕵️‍♂️ Investigation
We can confirm the connection is broken based on the symbols between the IPs:
```
//Good connection
[30857d36-92, L:/0.0.0.0:55868 - R:rest.apisandbox.server.com/1.1.1.1:443]
//Bad connection
[30857d36-92, L:/0.0.0.0:55868 ! R:rest.apisandbox.server.com/1.1.1.1:443]
```
#### The legend
- `-` means the connection is OK
- `!` means the connection is broken
- `30857d36` the connection id
- `92` number of re-usages of the connection

### Example
```
2021-12-23 08:26:23.069 DEBUG --- [or-http-epoll-2] reactor.netty.http.client.HttpClient     : [338e870a-97, L:/0.0.0.0:55864 ! R:rest.apisandbox.server.com/1.1.1.1:443] USER_EVENT: SslCloseCompletionEvent(java.nio.channels.ClosedChannelException)
2021-12-23 08:26:23.069 DEBUG --- [or-http-epoll-2] reactor.netty.http.client.HttpClient     : [338e870a-97, L:/0.0.0.0:55864 ! R:rest.apisandbox.server.com/1.1.1.1:443] INACTIVE
2021-12-23 08:26:23.069 DEBUG --- [or-http-epoll-2] reactor.netty.http.client.HttpClient     : [338e870a-97, L:/0.0.0.0:55864 ! R:rest.apisandbox.server.com/1.1.1.1:443] UNREGISTERED
```

### The problem

The connection was registered/created on: `2021-12-22 19:04:34.458` and it was invalidated on `2021-12-23 08:26:16.139`. 
It means it stayed in the connection pool and was re-used for almost 13 hours. At a certain moment, the server, on the other end, reset the connection.
Now we have to throw the connection out from the pool. 

A pick in the [documentation](https://projectreactor.io/docs/netty/release/reference/index.html#connection-pool-timeout), we see some properties:
| Property      | Description |
| ----------- | ----------- |
| `maxIdleTime` | The time after which the channel is eligible to be closed when idle (resolution: ms). Default: max idle time is not specified       |
| `maxLifeTime` | The total lifetime after which the channel is eligible to be closed (resolution: ms). Default: max lifetime is not specified      |
| `evictInBackground` | When this option is enabled, each connection pool regularly checks for connections that are eligible for removal according to eviction criteria like maxIdleTime. By default, this background eviction is disabled      |



### ✅ Solution

We have to tell the reactor what connections are invalid. We can do that by setting up a `ConnectionProvider` with some properties. By setting the `evictInBackground` in combination with `maxIdleTime` we tell the reactor to clear connections idle for 30 seconds on every 120 seconds. Bellow an example.

We can also consider a retry mechanism when the connection is invalid and create a new connection or finally get back to good old `RestTemplate`, which is blocking and uses one connection at the time

Simplified for bravery.
```java
@Configuration
class WebClientConfiguration {

    private static final Logger log = LoggerFactory.getLogger(WebClientConfiguration.class);

    @Bean
    WebClient yourWebClient(Environment config) {
        ConnectionProvider connectionProvider = ConnectionProvider.builder("http-connection-pool")
            .maxConnections(50) //a number of connections
            .maxIdleTime(Duration.ofSeconds(30))) //max time to stay in the pool in idle state
            .maxLifeTime(Duration.ofHours(3))) //max lifetime 3 hours then closed
            .pendingAcquireTimeout(Duration.ofSeconds(60))) //try to aquire for 60 seconds or else fail
            .evictInBackground(Duration.ofSeconds(120))) //regularly checks for connections each 120 seconds
            .metrics(true) //for actuator
            .build();

        HttpClient httpClient = HttpClient.create(connectionProvider)
            .responseTimeout(Duration.ofSeconds(10))) //close connection, when no data was read within 10 seconds after the socket was initialized.
            .compress(true)
            .wiretap(HttpClient.class.getName(), LogLevel.DEBUG, AdvancedByteBufFormat.SIMPLE) //print wiretap logs only when HttpClient logger is on DEBUG
            .metrics(true, Function.identity());

        return WebClient.builder()
            .clientConnector(new ReactorClientHttpConnector(httpClient))
            .baseUrl('baseUrl')
            .build();
    }
}

```
### Optional settings

`maxLifeTime` is just a precaution if there is an exchange of data for 3 hours. \
`pendingAcquireTimeout` is how much time to try to acquire a connection - the default is 45 seconds. \
`metrics` this is handly for monitoring the throughput and connection pool details. \
`responseTimeout` this one is about how much time you have to wait for data after the connection was initialiazed


That's pretty much it. \
You just have to monitor the throughput, see if a tweak of settings is needed. The reactor connection pool is pretty the same as a JDBC connection pool.


Btw, the same strategy can be applied for the famous prematurely closed exception cases. \
We could also improve the logging by printing the exception and the connection id on the exception stack trace using `ClientRequest.logPrefix()`.

### 📚 Read more

[The documentation](https://projectreactor.io/docs/netty/release/reference/index.html#connection-pool-timeout) \
[How to Avoid Common Mistakes When Using Reactor Netty](https://www.youtube.com/watch?v=LLSln1_JAMY)

