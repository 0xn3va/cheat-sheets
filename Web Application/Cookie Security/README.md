This section describes the attributes and methods that help improve cookie security.

# Cookie flags

- 'Ultimate' Cookie: `Set-Cookie: __Host-SessionID=3h93...;Path=/;Secure;HttpOnly;SameSite=Strict`

## Secure attribute

Cookies marked with the `Secure` attribute are only sent over encrypted HTTPS connections. The `Secure` attribute only protects the **confidentiality** of a cookie against MiTM attackers – there is no integrity protection.

![mitm-attack](/Web%20Application/Cookie%20Security/img/mitm-attack.png)

- Mallory can’t read `Secure` cookies,
- Mallory can still **write/change** `Secure` cookies.

## HttpOnly attribute

Cookies marked with the `HttpOnly` attribute are not accessible from JavaScript. The `HttpOnly` attribute only protects the **confidentiality** of a cookie, but `HttpOnly` cookies can be replaced by overflowing the cookie jar from JavaScript.

## Path attribute

The `Path` attribute limits the scope of a cookie to a specific path on the server and can therefore be used to prevent unauthorized access to it from other applications on the same host.

**Cookie scope vs Same-Origin Policy**

![cooke-scope](/Web%20Application/Cookie%20Security/img/scope-sop-cookie.png)

**Isolation two different applications on shared host**

![isolation-apps](/Web%20Application/Cookie%20Security/img/isolation-sop-cookie.png)

## Domain attribute

The `Domain` specifies allowed hosts to receive the cookie.

- If the `Domain` attribute unspecified, it defaults to the host of the current document location, excluding subdomains.
    - IE will always send to subdomains regardless.
- If the `Domain` attribute is specified, cookies will be sent to that domain and **all its subdomains**.

## Expires attribute

The `Expires` attribute allows you to set the maximum cookie lifetime.

- If the `Expires` attribute unspecified, cookie lifetime is equal to session lifetime.
    - It is up to the browser to decide when the session ends,
    - 'Non-persistent' session cookies may actually be persisted to survive browser restart.

[Document.cookie documentation](https://developer.mozilla.org/en-US/docs/Web/API/document/cookie)

![cookie-survive](/Web%20Application/Cookie%20Security/img/cookie-survive.png)

## SameSite

The `SameSite` attribute prevents the browser from sending cookies along with cross-site requests. The `SameSite` attribute can have one of two values (case-insensitive):
- `Strict`, if the URL of the constructed request is different from the URL of the current page, then `Strict` cookies will not be included in the request.
    - Possible problem: `Strict` cookies will not be sent when clicking on a link from another site.
    - Possible solution: Use two cookies. The first cookie, **without** `SameSite`, as a uniq user id, allows you to show username e.g. Second cookie, **with** `SameSite`, to make purchases, profile changes, and more.
- `Lax`, adds an exception allowing the send a cookies when navigating from an external URL, which uses "secure" HTTP methods, for example, when clicking on the link. The "secure" methods: `GET, HEAD, OPTIONS и TRACE`.

[Browser compatibility](https://developer.mozilla.org/en-US/docs/Web/HTTP/Headers/Set-Cookie#Browser_compatibility)

> Notice that from Chrome 80 (February 2019) the default behaviour of a cookie without a cookie SameSite attribute will be Lax, [link](https://www.troyhunt.com/promiscuous-cookies-and-their-impending-death-via-the-samesite-policy/).

> Notice that temporary, after applying this change, the cookies without a SameSite policy in Chrome will be treated as None during the first 2 minutes and then as Lax.

# Cookie prefix

The `Cookie Prefix` allows the send of cookie prefix information to ensure that certain attributes are present in a cookie request. Supported prefixes:
- `__Secure-`, tells the browser that the `Secure` attribute is required,
- `__Host-`,  tells the browser that the `Path = /` and `Secure` attributes are required, and at the same time that the `Domain` attribute should not be present.

[Browser compatibility](https://developer.mozilla.org/en-US/docs/Web/HTTP/Headers/Set-Cookie#Browser_compatibility)

# Cookie-list sorting

The [RFC6265](https://tools.ietf.org/html/rfc6265#section-5.4) standard defines the order of cookies:

```
2.  The user agent SHOULD sort the cookie-list in the following order:

    *  Cookies with longer paths are listed before cookies with shorter paths.

    *  Among cookies that have equal-length path fields, cookies with earlier creation-times are listed before cookies with later creation-times.
```

as a result, if the vulnerable application uses only the first cookie, you can force it to use your cookie by adding the `Path` attribute with longer path. 
