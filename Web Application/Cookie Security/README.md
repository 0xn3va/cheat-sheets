# Cookie attributes

`Ultimate` cookie:

```http
Set-Cookie: __Host-SessionID=3h93...;Path=/;Secure;HttpOnly;SameSite=Strict
```

## Secure attribute

The `Secure` attribute indicates that the cookie is sent to a server only when a request is made with the `https:` scheme (except on localhost). It only protects the **confidentiality** of a cookie against MitM attackers - there is no integrity protection. Therefore, cookies with this attribute can still be modified either with access to the client's hard disk or from JavaScript.

Insecure sites `http:` can't set cookies with the `Secure` attribute (since Chrome 52 and Firefox 52). For Firefox, the `https:` requirements are ignored when the `Secure` attribute is set by localhost (since Firefox 75).

## HttpOnly attribute

Cookies marked with the `HttpOnly` attribute are not accessible from JavaScript. The `HttpOnly` attribute only protects the **confidentiality** of a cookie. `HttpOnly` cookies can be replaced by overflowing the cookie jar from JavaScript.

## Path attribute

The `Path` attribute indicates the path that must exist in the requested URL for the browser to send the Cookie header. It can be used to prevent unauthorized access to cookies from other applications on the same host.

The forward slash `/` character is interpreted as a directory separator, and subdirectories are matched as well. For example, for `Path=/docs`:

- The request paths `/docs`, `/docs/`, `/docs/Web/`, and `/docs/Web/HTTP` will all match.
- The request paths `/`, `/docsets`, `/fr/docs` will not match.

### Cookie scope vs Same-Origin Policy

![](/Web%20Application/Cookie%20Security/img/scope-sop-cookie.png)

### Isolation two different applications on shared host

![](/Web%20Application/Cookie%20Security/img/isolation-sop-cookie.png)

## Domain attribute

The `Domain` attribute defines the host to which the cookie will be sent.

- If the `Domain` attribute unspecified, it defaults to the host of the current document location, excluding subdomains
    - IE will always send to subdomains regardless
- If the `Domain` attribute is specified, cookies will be sent to that domain and **all its subdomains**

## Expires attribute

The `Expires` attribute indicates the maximum lifetime of the cookie as an HTTP-date timestamp.

- If the `Expires` attribute unspecified, cookie lifetime is equal to session lifetime
    - It is up to the browser to decide when the session ends
- `Non-persistent` session cookies may actually be persisted to survive browser restart

![](/Web%20Application/Cookie%20Security/img/cookie-survive.png)

References:
- [MDN Web Docs - Document.cookie](https://developer.mozilla.org/en-US/docs/Web/API/document/cookie)

## Max-Age attribute

The `Max-Age` attribute indicates the number of seconds until the cookie expires. If both `Expires` and `Max-Age` are set, `Max-Age` has precedence.

## SameSite

The `SameSite` attribute prevents the browser from sending cookies along with cross-site requests. The `SameSite` attribute can have one of two values (case-insensitive):

- `Strict`, means that the browser sends the cookie only for same-site requests, that is, requests originating from the same site that set the cookie. If a request originates from a URL different from the current one, no cookies with the `SameSite=Strict` attribute are sent.
- `Lax`, means that the cookie is not sent on cross-site requests, such as on requests to load images or frames, but is sent when a user is navigating to the origin site from an external site using `safe` HTTP methods (for example, when following a link). This is the default behavior if the SameSite attribute is not specified. The `safe` methods: `GET, HEAD, OPTIONS` and `TRACE`.
- `None`, means that the browser sends the cookie with both cross-site and same-site requests. The `Secure` attribute must also be set when setting this value, like so `SameSite=None; Secure`.

{% hint style="info" %}
The cookies without the `SameSite` in Chrome are still treated as `None` during the first 2 minutes and then as `Lax`, check out [Bypass SameSite Cookies Default to Lax and get CSRF](https://medium.com/@renwa/bypass-samesite-cookies-default-to-lax-and-get-csrf-343ba09b9f2b)
{% endhint %}

{% hint style="info" %}
Keep in mind that same-site and cross-site requests are not the same thing. The SameSite cookie attribute is only concerned with cross-site requests. It does not affect cross-origin requests that refer to the same-site, check out [The great SameSite confusion](https://jub0bs.com/posts/2021-01-29-great-samesite-confusion/)
{% endhint %}

References:
- [MDN Web Docs - Set-cookie: Browser compatibility](https://developer.mozilla.org/en-US/docs/Web/HTTP/Headers/Set-Cookie#Browser_compatibility)

# Cookie prefix

The cookie prefix allows you to pass metadata about a cookie and notify a client that certain attributes have been set. The following prefixes are supported:

- `__Secure-` tells the browser that the `Secure` attribute is required.
- `__Host-` tells the browser that the `Path=/` and `Secure` attributes are required, and at the same time that the `Domain` attribute should not be present (and therefore, can not be sent to subdomains).

References:
- [MDN Web Docs - Set-cookie: Browser compatibility](https://developer.mozilla.org/en-US/docs/Web/HTTP/Headers/Set-Cookie#Browser_compatibility)

# Cookie-list sorting

The [RFC6265](https://tools.ietf.org/html/rfc6265#section-5.4) standard defines the order of cookies:

```
2.  The user agent SHOULD sort the cookie-list in the following order:

    *  Cookies with longer paths are listed before cookies with shorter paths.

    *  Among cookies that have equal-length path fields, cookies with earlier creation-times are listed before cookies with later creation-times.
```

Therefore, if a vulnerable application uses the first cookie, you can force it to use your cookie by adding the `Path` attribute with a longer path. 

# References

- [MDN Web Docs - Set-cookie](https://developer.mozilla.org/en-US/docs/Web/HTTP/Headers/Set-Cookie)
