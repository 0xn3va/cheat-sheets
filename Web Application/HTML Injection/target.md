# Overview

The [target](https://developer.mozilla.org/en-US/docs/Web/HTML/Element/a) attribute specifies where to display the linked URL as a name for the browsing context (a tab, window, or `<iframe>`). The following keywords have special meanings for where to load the URL:
- `_self` -  the current browsing context, by default.
- `_blank` - usually a new tab, but users can configure browsers to open a new window instead.
- `_parent` - the parent browsing context of the current one, if no parent, behaves as `_self`.
- `_top` - the topmost browsing context (the "highest" context that’s an ancestor of the current one), if no ancestors, behaves as `_self`.

# _blank

Using `target=_blank` allows the linked page to get partial access to the source page through the [window.opener](https://developer.mozilla.org/en-US/docs/Web/API/Window/opener) API. The newly opened tab can then change the `window.opener.location` to a phishing page or execute JavaScript on the opener page.

{% hint style="info" %}
In newer browser versions (e.g. Firefox 79+) setting target="_blank" on `<a>` elements implicitly provides the same `rel` behavior as setting `rel="noopener"`.
{% endhint %}

# References

- [Target="_blank" — the most underestimated vulnerability ever](https://medium.com/@jitbit/target-blank-the-most-underestimated-vulnerability-ever-96e328301f4c)
