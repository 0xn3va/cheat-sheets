# Overview

The [&lt;iframe&gt;](https://developer.mozilla.org/en-US/docs/Web/HTML/Element/iframe) tag is used to embed an HTML document in another HTML document. If the source of the inserted document is located on another origin, the same origin policy will block any access to the content of the other document for both of them.

# Open redirect

Child documents can view and set location property for parents, even if cross-origin `top.window.location`.

For example, if `vulnerable-website.com` contains the following `iframe`:

```html
<iframe src=//malicious-website.com/toplevel.html></iframe>
```

where `https://malicious-website.com/toplevel.html` is:

```html
<html><head></head><body><script>top.window.location = "https://malicious-website.com/pwned.html"</script></body></html>
```

when the `iframe` is loaded, the parent will be redirected to the `https://malware-website.com/pwned.html` page, even if the child document is loaded from a different origin. In this case, the same origin policy will be bypassed because the `iframe` is not being "sandboxed", check out the [sandbox](https://developer.mozilla.org/en-US/docs/Web/HTML/Element/iframe) iframe attribute.

References:
- [&lt;meta&gt; and &lt;iframe&gt; tags chained to SSRF](https://medium.com/@know.0nix/hunting-good-bugs-with-only-html-d8fd40d17b38)
