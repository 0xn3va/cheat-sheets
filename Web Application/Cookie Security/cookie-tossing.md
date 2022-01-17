If you control a subdomain or you find a XSS in a subdomain, you can set a cookie that will be used in a domain and their subdomains. This can lead to the following attack vectors:
- Setting an attacker cookie for a victim and collecting sensitive data that a victim will add when using an attacker's account
- Fixate a cookie if cookie is not changed after login
- If a cookie sets an initial value you can set a known value and abuse it. For instance, a cookie sets a CSRF token of a session in Flask and this value is maintained after login, therefore, you can use known value to perform a CSRF
- Perform [Cookie bomb](/Web%20Application/Cookie%20Security/cookie-bomb.md) attack

Cookie tossing is possible even a cookie is already set, since when a browser receives two cookies with the same name partially affecting the same scope (domain, subdomains and path), the browser will send both cookies when valid for request. If an application uses only the first cookie, you can force it to use your cookit by adding the `Path` attribute with longer path, check [Cookie Security: Cookie-list sorting](/Web%20Application/Cookie%20Security/README.md#cookie-list-sorting).

If an application does not accept requests with cookies with the same name and different values, you can try the following tricks:
- Overflow a legit cookie with attacker's one, check [Cookie Jar Overflow](/Web%20Application/Cookie%20Security/cookie-jar-overflow.md)
- Change a cookie name: use URL encoding, use different letter-case, add extra symbols, such as `%00`, `%20`, `%09`, etc.

# References

- [HackTricks: Cookie Tossing](https://book.hacktricks.xyz/pentesting-web/hacking-with-cookies/cookie-tossing)
