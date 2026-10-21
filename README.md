# Jettyx

Jettyx is a declarative HTTP client for Java 17 and newer using
`java.net.http.HttpClient`. The core has no external dependencies. HTTP/1.1
and HTTP/2 are built in; HTTP/3 is discontinued for this Java 17 API.

Jettyx provides synchronous and `CompletableFuture` APIs, annotated requests,
authentication, custom body mappers, request deadlines and bounded response bodies.

## Installation

```groovy
implementation 'org.exploit:jettyx:0.2.0'

// JSON support
implementation 'org.exploit:jettyx-jackson:0.2.0'
```

```xml
<dependency>
    <groupId>org.exploit</groupId>
    <artifactId>jettyx</artifactId>
    <version>0.2.0</version>
</dependency>
```

The optional JSON module uses Jackson 2.22.4.
The core handles UTF-8 strings, primitive and boxed scalars, `byte[]` and void
responses without additional modules.

## Usage

```java
import org.exploit.jettyx.Jettyx;
import org.exploit.jettyx.annotation.Body;
import org.exploit.jettyx.annotation.HttpRequest;
import org.exploit.jettyx.annotation.Path;
import org.exploit.jettyx.annotation.Query;
import org.exploit.jettyx.auth.BearerAuth;
import org.exploit.jettyx.jackson.JacksonHttpMapper;
import org.exploit.jettyx.model.HttpResponse;

import java.net.http.HttpClient;
import java.util.concurrent.CompletableFuture;

import static org.exploit.jettyx.constant.HttpMethod.POST;

record User(String name) {}

interface UsersApi {
    @HttpRequest(path = "/users/{id}")
    User get(@Path("id") String id, @Query("expand") String expand);

    @HttpRequest(method = POST, path = "/users")
    CompletableFuture<HttpResponse<User>> create(@Body User user);
}

try (var jettyx = Jettyx.newBuilder()
        .addHttpMapper(JacksonHttpMapper.create())
        .version(HttpClient.Version.HTTP_2)
        .timeoutSeconds(30)
        .maxBufferSize(8 * 1024 * 1024)
        .build()) {
    var users = jettyx.newApiClient("https://api.example.com", new BearerAuth("token"))
            .create(UsersApi.class);

    var user = users.get("123", null);
    var response = users.create(new User("Alice")).join();
}
```

The JDK prefers HTTP/2 by default and negotiates HTTP/1.1 when needed. To force
HTTP/1.1, use `.version(HttpClient.Version.HTTP_1_1)`. Configure native settings
with `.customize(builder -> builder.followRedirects(HttpClient.Redirect.NORMAL))`,
`.sslContext(sslContext)` or `.executor(executor)`. Redirects are disabled by default.

An API is an interface. HTTP methods require `@HttpRequest`; each parameter needs
exactly one Jettyx parameter annotation. Default interface methods work normally.
Use parameterized `CompletableFuture<T>` and `HttpResponse<T>` wrappers;
parameterized body types such as `List<User>` are preserved for mappers.
Response types must not contain unresolved type variables such as `T` or `List<T>`.

## Requests and authentication

| Annotation | Purpose |
| --- | --- |
| `@HttpRequest` | Method and path; defaults to GET and an empty path |
| `@Path("id")` | Required path value, encoded as one path segment |
| `@Query("page")` | Query value; null values are omitted |
| `@QueryMap` | Map of query values; null values are omitted |
| `@Header("X-Trace")` | Header value; null values are omitted |
| `@HeaderMap` | Map of non-null header names and values |
| `@Body` | One body; null means an empty body |
| `@ContentType("application/json")` | Method annotation overriding mapper content type |
| `@Auth` | Per-request authorization; null uses the client's default |

Base URLs must be absolute HTTP or HTTPS URLs without credentials, a query or a
fragment. Base path prefixes are retained. Use query annotations instead of
embedding a query in the request path. Path and query values are encoded separately.

`BasicAuth`, `BearerAuth`, `HeaderAuth`, `QueryAuth` and `NoAuth` are built in.
Authorization replaces conflicting request headers or query parameters.
Pass `new NoAuth()` through `@Auth` to disable default authentication for one call.
Custom implementations receive `HttpRequest.Builder` or `UrlBuilder`.

Request bodies can also be native `HttpRequest.BodyPublisher` instances. For other
types, implement `HttpMapper` and register it with `addHttpMapper`. Custom mappers
are checked in registration order before built-in scalar and byte-array mappers.

## Responses, limits and lifecycle

Return `T` for a decoded body, or `HttpResponse<T>` for status, JDK `HttpHeaders`,
the decoded body and the raw JDK `HttpResponse<byte[]>`. Both forms can be wrapped
in `CompletableFuture`. HTTP error statuses are returned normally.

For wrapped responses, a mapping failure leaves `body()` null and stores the error
in `exception()` while preserving the original response. Unwrapped mapping failures
are thrown or complete the future exceptionally. Transport failures always fail the
call. Primitive `void` and boxed `Void` discard the body without deserialization.

The timeout covers the complete HTTP exchange, including reading the response body,
for both synchronous and asynchronous calls. The response limit applies while bytes
arrive, including chunked responses. Cancelling an asynchronous future cancels the
underlying exchange. Interrupted synchronous calls restore the thread's interrupt flag.

Closing Jettyx cancels pending requests and rejects new HTTP calls.
Application-provided executors remain owned by the application.

Rate limiting uses a callback invoked once before each HTTP exchange. For example,
an application using Bucket4j can pass:

```java
var users = jettyx.newApiClient("https://api.example.com", new BearerAuth("token"),
        () -> bucket.asBlocking().consume(1)).create(UsersApi.class);
```

The callback may block and throw `InterruptedException`. Async calls acquire their
permit before returning the future.

## Migrating from 0.1.x

- Remove `jettyx-http2` and `jettyx-http3`; HTTP/2 is built into core and HTTP/3 is discontinued.
- Replace Jetty's `HttpMethod` import with `org.exploit.jettyx.constant.HttpMethod`.
- Replace `enableVersion(...)` with `version(HttpClient.Version.HTTP_2)` or `HTTP_1_1`.
- Replace `sslContextFactory(...)` with `sslContext(SSLContext)` and update customizers
  to accept JDK `HttpClient.Builder`.
- Update custom authorization to use JDK `HttpRequest.Builder`, raw body publishers
  to use JDK `BodyPublisher`, and response access to use JDK headers and responses.
- Replace Bucket4j arguments with a `RateLimiter` callback, for example
  `() -> bucket.asBlocking().consume(1)`.

## License

[BSD 2-Clause](LICENSE.md).
