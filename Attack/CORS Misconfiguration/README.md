# CORS Misconfiguration

## Summary

- [Cross-Origin Resource Sharing (CORS)](#cross-origin-resource-sharing-cors)
    - [Overview](#overview)
    - [Simple requests](#simple-requests)
    - [Preflighted requests](#preflighted-requests)
    - [Requests with credentials](#requests-with-credentials)
    - [The HTTP response headers](#the-http-response-headers)
        - [Access-Control-Allow-Origin](#access-control-allow-origin)
        - [Access-Control-Expose-Headers](#access-control-expose-headers)
        - [Access-Control-Max-Age](#access-control-max-age)
        - [Access-Control-Allow-Credentials](#access-control-allow-credentials)
        - [Access-Control-Allow-Methods](#access-control-allow-methods)
        - [Access-Control-Allow-Headers](#access-control-request-headers)
    - [The HTTP request headers](#the-http-request-headers)
        - [Origin](#origin)
        - [Access-Control-Request-Method](#access-control-request-method)
        - [Access-Control-Request-Headers](#access-control-request-headers)
- [CORS Misconfiguration]()
    - [Generation the Access-Control-Allow-Origin header](#generation-the-access-control-allow-origin-header)
    - [The null origin](#the-null-origin)
    - [Breaking parsers](#breaking-parsers)
    - [Abusing CORS without credentials](#abusing-cors-without-credentials)
    - [Vary: Origin](#vary-origin)
        - [Client-Side cache poisoning](#client-side-cache-poisoning)
    - [Server-Side cache poisoning](#server-side-cache-poisoning)
- [References](#references)

## Cross-Origin Resource Sharing (CORS)

Useful references:
- [Same-origin policy](https://developer.mozilla.org/en-US/docs/Web/Security/Same-origin_policy),
- [CORS specification](https://www.w3.org/TR/cors/)

### Overview

The CORS standard works by adding new HTTP headers that let servers describe which origins are permitted to read that 
 information from a web browser. A web application executes a cross-origin HTTP request when it requests a resource that
 has a different origin (domain, protocol, or port) from its own.

For HTTP request methods that can cause side-effects on server data, the specification mandates that browsers preflight
 the request, soliciting supported methods from the server with the HTTP OPTIONS request method, and then, upon 
 approval from the server, sending the actual request. 

Servers can also inform clients whether credentials should be sent with requests.

CORS failures result in errors, but specifics about the error are not available to JavaScript.

### Simple requests

Simple requests don't trigger a CORS preflight. A simple request is one that meets **all** the following conditions:

- One of the allowed methods:
    - `GET`,
    - `HEAD`,
    - `POST`.
- The only allowed headers which can will be set manually:
    - `Accept`,
    - `Accept-Language`,
    - `Content-Language`,
    - `Content-Type`,
    - `DPR`,
    - `Downlink`,
    - `Save-Data`,
    - `Viewport-Width`,
    - `Width`.
- The only allowed values for the `Content-Type` header:
    - `application/x-www-form-urlencoded`,
    - `multipart/form-data`,
    - `text/plain`.
- No event listeners are registered on any `XMLHttpRequestUpload` object used in the request (these are accessed using 
 the [XMLHttpRequest.upload](https://developer.mozilla.org/en-US/docs/Web/API/XMLHttpRequest/upload) property).
- No [ReadableStream](https://developer.mozilla.org/en-US/docs/Web/API/ReadableStream) object is used in the request.

### Preflighted requests

Preflighted requests first send an HTTP request by the OPTIONS method to the resource on the other domain, to determine
 if the actual request is safe to send. 

### Requests with credentials

Credentialed requests allow to send HTTP cookies and HTTP Authentication information (by default browsers will not send
 credentials).

When responding to a credentialed request, the server must specify an origin in the value of the
 `Access-Control-Allow-Origin` header, instead of specifying the "*" wildcard.

> Note: Cookies set in CORS responses are subject to normal third-party cookie policies.

### The HTTP response headers

This section lists the HTTP response headers that servers send back for access control requests as defined by the
 Cross-Origin Resource Sharing specification.

#### Access-Control-Allow-Origin

```http request
Access-Control-Allow-Origin: <origin-or-null> | *
```

The `Access-Control-Allow-Origin` specifies either a single origin 
 ([or null](https://www.w3.org/TR/cors/#access-control-allow-origin-response-header)), which tells browsers to allow 
 that origin to access the resource; or else — for requests without credentials — the "*" wildcard, to tell browsers
 to allow any origin to access the resource.

If the server specifies a single origin rather than the "*" wildcard, then the server should also set `Origin` in the
 [Vary](https://developer.mozilla.org/en-US/docs/Web/HTTP/Headers/Vary) response header — to indicate to clients that
 server responses will differ based on the value of the `Origin` request header.

#### Access-Control-Expose-Headers

```http request
Access-Control-Expose-Headers: <header-name>[, <header-name>]*
```

The `Access-Control-Expose-Headers` header lets a server whitelist headers that browsers are allowed to access.

#### Access-Control-Max-Age

```http request
Access-Control-Max-Age: <delta-seconds>
```

The `delta-seconds` parameter indicates the number of seconds the results can be cached.

The `Access-Control-Max-Age` header indicates how long the results of a preflight request can be cached.

#### Access-Control-Allow-Credentials

```http request
Access-Control-Allow-Credentials: true
```

The `Access-Control-Allow-Credentials` header indicates whether or not the response to the request can be exposed when
 the `credentials` flag is true. When used as part of a response to a preflight request, this indicates whether or not
 the actual request can be made using credentials. Note that simple GET requests are not preflighted, and so if a 
 request is made for a resource with credentials, if this header is not returned with the resource, the response is 
 ignored by the browser and not returned to web content.

#### Access-Control-Allow-Methods

```http request
Access-Control-Allow-Methods: <method>[, <method>]*
```

The `Access-Control-Allow-Methods` header specifies the method or methods allowed when accessing the resource. 
 This is used in response to a preflight request.

#### Access-Control-Allow-Headers

```http request
Access-Control-Allow-Headers: <header-name>[, <header-name>]*
```

The `Access-Control-Allow-Headers` header is used in response to a preflight request to indicate which HTTP headers can
 be used when making the actual request.

### The HTTP request headers

This section lists headers that clients may use when issuing HTTP requests in order to make use of the cross-origin 
 sharing feature. Note that these headers are set for you when making invocations to servers. Developers using cross-site
 `XMLHttpRequest` capability do not have to set any cross-origin sharing request headers programmatically.

#### Origin

```http request
Origin: <origin>
```

The `Origin` header indicates the origin of the cross-site access request or preflight request. The `origin` parameter
 is a URI indicating the server from which the request initiated. It does not include any path information, but only
 the server name.

> Note: The origin value can be null, or a URI.

> Note: In any access control request, the `Origin` header is always sent.

#### Access-Control-Request-Method

```http request
Access-Control-Request-Method: <method>
```

The `Access-Control-Request-Method` is used when issuing a preflight request to let the server know what HTTP method 
 will be used when the actual request is made.

#### Access-Control-Request-Headers

```http request
Access-Control-Request-Headers: <field-name>[, <field-name>]*
```

The `Access-Control-Request-Headers` header is used when issuing a preflight request to let the server know what HTTP
 headers will be used when the actual request is made.

## CORS Misconfiguration

### Generation the Access-Control-Allow-Origin header

No browsers actually support a space-separated list of origins (the specification suggests this), like:

```http request
Access-Control-Allow-Origin: http://foo.com http://bar.net
```

Additionally, a wildcard won't work to trust all subdomains, like

```http request
Access-Control-Allow-Origin: *.bar.net
```

The only wildcard origin is `'*'`.

Since you cannot use wildcard in `Access-Control-Allow-Origin` when credentials flag
 is true ([click](#access-control-allow-origin)), many servers programmatically generate the `Access-Control-Allow-Origin`
 header based on the user-supplied `Origin` value. If you see a HTTP response with any `Access-Control-*` headers but 
 no origins declared, this is a strong indication that the server will generate the header based on your input. Other 
 servers will only send CORS headers if they receive a request containing the Origin header, making associated 
 vulnerabilities extremely easy to miss.

### The null origin

The specification mentions the null origin being triggered by redirects, and a few stackoverflow posts show that local
 HTML files also get it. Quite a few websites whitelist it, perhaps due to the association with local files, like:

```http request
GET /reader?url=zxcvbn.pdf
Host: docs.google.com
Origin: null

HTTP/1.1 200 OK
Acess-Control-Allow-Origin: null
Access-Control-Allow-Credentials: true
```

Any website can easily obtain the null origin using a sandboxed iframe:

```html
<iframe sandbox="allow-scripts allow-top-navigation allow-forms" 
src='data:text/html,<script>*cors stuff here*</script>’></iframe>
```

### Breaking parsers

Useful references:
- [Advanced CORS Exploitation Techniques](https://www.corben.io/advanced-cors-techniques/)

Most websites use basic string operations to verify the `Origin` header, but some parse it as a URL instead.
this allows to exploit the of browser's tolerance for unusual characters in domain names. 

In Safari, `http://foo.com%60.bar.net/` is a valid URL. If the CORS request originating from that URL will contain:

```http request
Origin: http://foo.com`.bar.net/
``` 

 and a site chooses to parse this header, it will potentially think that the hostname is `foo.com` and reflect it, 
 letting us exploit Safari users even though the site is using a whitelist of trusted hostnames.

Chrome and Firefox supported the `_` character in subdomains. This allow to use `_` instead of `` ` `` to exploit 
 Firefox and Chrome users.

### Abusing CORS without credentials

If the victim's network location functions as a kind of authentication, then you can use a victim’s browser as a proxy
 to bypass IP-based authentication and access intranet applications. In terms of impact this is similar to DNS rebinding,
 but much less fiddly to exploit.

### Vary: Origin

The CORS specification instructs developers specify the

```http request
Vary: Origin
```

HTTP header whenever `Access-Control-Allow-Origin` headers are dynamically generated.

That might sound pretty simple, but immense numbers of people forget, including the
 [W3C itself](https://lists.w3.org/Archives/Public/public-webappsec/2016Jun/0057.html). 

#### Client-Side cache poisoning

Say a web page reflects the contents of a custom header without encoding (reflected XSS in a custom HTTP header):

```http request
GET / HTTP/1.1
Host: example.com
X-User-id: <svg/onload=alert(1)>

HTTP/1.1 200 OK
Access-Control-Allow-Origin: *
Access-Control-Allow-Headers: X-User-id
Content-Type: text/html
...

Invalid user: <svg/onload=alert(1)>
```

Without CORS, this is impossible to exploit as there’s no way to make someone’s browser send the `X-User-id` header 
 cross-domain. 

With CORS, we can make them send this request. By itself, that's useless since the response containing our injected JS
 won't be rendered. However, if `Vary: Origin` hasn't been specified the response may be stored in the browser's cache
 and displayed directly when the browser navigates to the associated URL.

### Server-side cache poisoning

If an application reflects the `Origin` header without even checking it for illegal characters like `\r`, we effectively
 have a HTTP header injection vulnerability against IE/Edge users as IE and Edge view `\r (0x0d)` as a valid HTTP header
 terminator:

```http request
GET / HTTP/1.1
Origin: z[0x0d]Content-Type: text/html; charset=UTF-7
Internet Explorer sees the response as:

HTTP/1.1 200 OK
Access-Control-Allow-Origin: z
Content-Type: text/html; charset=UTF-7
```

This isn't directly exploitable because there's no way for an attacker to make someone's web browser send such a 
 malformed header, but we can manually craft this request and a server-side cache may save the response and serve it to
 other people. The current payload will change the page's character set to UTF-7, which is notoriously useful for 
 creating XSS vulnerabilities.

## References

- [Cross-Origin Resource Sharing (CORS)](https://developer.mozilla.org/en-US/docs/Web/HTTP/CORS)
- [Exploiting CORS misconfigurations for Bitcoins and bounties](https://portswigger.net/research/exploiting-cors-misconfigurations-for-bitcoins-and-bounties)
- [Tricky CORS Bypass in Yahoo! View](https://www.corben.io/tricky-CORS/)