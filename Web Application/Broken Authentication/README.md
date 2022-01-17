# Adding or changing email addresses

If the operations of adding a new email address or changing an existing one do not require password confirmation, session hijacking, XSS or CSRF can lead to account takeover.

# Information disclosure

An application can return unknown data in response when performing an operation. These can be requested passwords, generated OTP, cookies with additional privileges, user data, detailed error messages, and etc. Check the response from the server for such data.

# OAuth 2.0

{% embed url="https://0xn3va.gitbook.io/cheat-sheets/web-application/oauth-2.0-vulnerabilities" %}

# Password recovery

## Host header poisoning

An applications can use `Host` header value for generating a password recovery link. If an application is vulnerable to `Host` header poisoning, you can affect the link and specify your domain name. As a result, if a victim follows the link from the email, a recovery token will leak to your domain.

There are several ways to implement `Host` header poisoning.

### Host header replacement

You can try to replace the `Host` header on your domain. Examples of vulnerable requests:

```http
POST /users/password HTTP/1.1
Host: attacker-website.com
Content-Type: application/json
Content-Length: 31

{"email": "victim@website.com"}
```

Or bypass host validation (see [SSRF: Broken parser](https://0xn3va.gitbook.io/cheat-sheets/web-application/server-side-request-forgery#broken-parser)):

```http
POST /users/password HTTP/1.1
Host: attacker-website.com/vulnerable-website.com
Content-Type: application/json
Content-Length: 31

{"email": "victim@website.com"}
```

References:

- [Write up: Password Reset link hijacking via Host Header Poisoning](https://hackerone.com/reports/226659)

### Override Host header

Components in a delivery chain, such as proxies, can pass an original host requested by the client using additional of HTTP headers:

{% embed url="https://gist.github.com/0xn3va/b8ffb5c43d8dc24cdfcfc98e1890d71f" %}

You can try to use these headers for identifying the original host requested by the client in the `Host` request header:

```http
POST /users/password HTTP/1.1
Host: vulnerable-website.com
x-forwarded-host: attacker-website.com
x-host: attacker-website.com
x-original-host: attacker-website.com
Content-Type: application/json
Content-Length: 31

{"email": "victim@website.com"}
```

References:

- [Write up: Account Takeover Through Password Reset Poisoning](https://medium.com/@vbharad/account-takeover-through-password-reset-poisoning-72989a8bb8ea)

### Several Host header

You can try to specify several `Host` headers. Example of vulnerable request:

```http
POST /users/password HTTP/1.1
Host: vulnerable-website.com
Host: attacker-website.com
Content-Type: application/json
Content-Length: 31

{"email": "victim@website.com"}
```

## Several email addresses

Password recovery method can accept multiple recovery emails in one request. However, the method can search for an account to create a one-time token based on only one email, but send an email with a recovery link to all passed emails at once. For instance, if an application is vulnerable to such an attack, on the following request:

```http
POST /users/password HTTP/1.1
Host: vulnerable-website.com
Content-Type: application/json
Content-Length: 57

{"email": ["victim@website.com", "attacker@website.com"]}
```

an application sends the same recovery link to two email addresses.

You can also try the following payloads:

```http
victim@website.com&attacker@website.com
victim@website.com,attacker@website.com
victim@website.com|attacker@website.com
victim@website.com%20attacker@website.com
victim@website.com%09attacker@website.com
victim@website.com%0a%0dcc:attacker@website.com
...
```

References:

- [Writeup: Readme.com Account Takeover #BugBounty #FullDisclosure #Fixed](https://medium.com/@0xankush/readme-com-account-takeover-bugbounty-fulldisclosure-a36ddbe915be)

## Lack of link between account and code

Try to request an one-time token for your account and use them to recover a victim's account:

```http
POST /users/password/recovery HTTP/1.1
Host: vulnerable-website.com
Content-Type: application/json
Content-Length: 72

{"email": "victim@website.com","code": "<code-for-an-attacker-account>"}
```

## Password recovery does not end previously created sessions

{% hint style="info" %}
This is the best practice to follow
{% endhint %}

Successful password recovery should end previously created sessions. If this does not happen, an victim will not have mechanisms for managing the security of their account. As a result, an attacker will be able to maintain an active session for an extended period of time.

## Token leakage via Referer header

The [Referer](https://developer.mozilla.org/en-US/docs/Web/HTTP/Headers/Referer) HTTP request header contains an absolute or partial address of the page that makes the request. The `Referer` header allows a server to identify a page where people are visiting it from. This data is used for analytics, logging, optimized caching, and more.

The Referer header can contain an `origin`, `path`, and `querystring`, and may not contain URL fragments (i.e. `#section`) or `username:password` information. The request's referrer policy defines the data that can be included, see [Referrer-Policy](https://developer.mozilla.org/en-US/docs/Web/HTTP/Headers/Referrer-Policy).

A page with a password recovery, a link to which an application sends by email, can load third-party scripts, for instance, analytics, or contain links to third-party resources, for instance, social networks. If the password recovery link is passed into the `Referer` header when requesting these resources, an one-time token is leaked to a third-party resource, since it is passed to the `querystring`.

References:
- [Report: Cross Domain leakage of sensitive information - Leading to Account Takeover at Instagram Brand](https://hackerone.com/reports/209352)
- [Report: [Cross Domain Referrer Leakage] Password Reset Token Leaking to Third party Sites.](https://hackerone.com/reports/265740)
- [Report: Referer Referer Header Leakage in language changer may lead to FB token theft](https://hackerone.com/reports/870062)

# Phone and OTP authentication

## OTP resend

If an application does not set rate limits on resending OTP and resending OTP resets the set rate limits on attempts to enter OTP, an application is vulnerable to OTP bruteforce.

References:
- [Report: Authorization bypass using login by phone option+horizontal escalation possible on Grab Android App](https://hackerone.com/reports/205000)

## Spoofing part of an authentication request

If during the OTP check an application additionally uses some parameters, try to use a value from a request for a different account. This parameter can be a cookie, a header, or a request parameter. If you use the parameter value for `victim` in the OTP check request for `attacker`, you can get into `victim` account or bypass two-factor authentication.

Additionally, try to use the valid OTP value for `attacker` as the OTP value for `victim`. If an application does not verify that the OTP belongs to the `attacker`, you will be able to get into `victrim` account or bypass two-factor authentication.

## Short and long-lived OTP

If an application use short (less 5 digits) and/or long-lived OTPs, such implementations leave many ways to bruteforce, especially if there are no rate limits, or they are weak or can be bypassed.

References:
- [Report: Insecure 2FA/authentication implementation creates a brute force vulnerability](https://hackerone.com/reports/149598)

# Rate limits

{% embed url="https://0xn3va.gitbook.io/cheat-sheets/web-application/improper-rate-limits" %}

# Third-party sign-in or sign-up

Applications that implement third party sign-in or sign-up often identify users based on the attached email address that the third party application sends. In such cases, you can try sign-in or sign-up with two different third-party applications that are linked to the same email address. Sometimes these two accounts can be joined and you will have access to the victim's account by sign-in or sign-up with your third-party application account.

For example, suppose a victim sign-in or sign-up to `vulnerable-website.com` with `third-party-app1.com`, which is linked to `victim@website.com`. To exploit this, you can try the following:

1. Create an account at `third-party-app2.com` and enter the `victim@website.com` email address (ideal if email confirmation is not required),
2. Try sign-in or sign-up at `vulnerable-website.com` through `third-party-app2.com`,
3. If the application is vulnerable, you will have access to the victim's account.

References:

- [Write up: account takeover https://teamplay.qiwi.com](https://hackerone.com/reports/439207)

# Username and password authentication

## Bruteforce

If an application does not set rate limits on login attempts, try to craft a dictionary and bruteforce a password.

One of the implementations of rate limits uses a username or email as an identifier to count attempts. Try to bypass the protection by using extra spaces or upper/lower case:

```http
email=" username@website.com"
email="username@website.com  "
email="Username@website.com"
email="USERNAME@website.com"
...
```

You can use the following links:
- [WordList Compendium](https://github.com/Dormidera/WordList-Compendium) - personal compilation of wordlists & dictionaries for everything; users, passwords, directories, files, vulnerabilities, fuzzing, injections, wordlists of tools, etc.
- [SecLists](https://github.com/danielmiessler/SecLists) - a collection of multiple types of lists used during security assessments.
- [PWDB - New generation of Password Mass-Analysis](https://github.com/ignis-sec/Pwdb-Public) - a collection of all the data extracted from 1 billion credential leaks from the Internet.
- [bopscrk](https://github.com/r3nt0n/bopscrk) - tool to generate smart and powerful wordlists.
- [BruteLoops](https://github.com/arch4ngel/BruteLoops) - protocol agnostic online password guessing API.

References:
- [Report: Bypass a fix for report #708013](https://hackerone.com/reports/1363672)

## Credential stuffing

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

## Default credentials

Try to login with default credentials into admin panels, third-party services, middleware, and etc.

You can use the following lists:
- [Default Credentials Cheat Sheet](https://github.com/ihebski/DefaultCreds-cheat-sheet)
- [SecLists Default Credentials](https://github.com/danielmiessler/SecLists/tree/master/Passwords/Default-Credentials)

## Lack of restrictions on password length

Most applications store hashes of user passwords and do not store passwords in clear text. Accordingly, an application hashes passed password and checks the hash against the one stored in a database when authenticating a user. Moreover, functions with sliding computational cost are often used as a hashing function, the computational complexity of which is much higher than common hash functions such as `sha256`. Therefore, if an application does not implement restrictions on the length of passwords, this can be used for DoS: hashing very long passwords can be resource intensive.

# References

- [HackTricks: Reset/Forgotten Password Bypass](https://book.hacktricks.xyz/pentesting-web/reset-password)
- [Two-factor authentication security testing and possible bypasses](https://medium.com/@iSecMax/two-factor-authentication-security-testing-and-possible-bypasses-f65650412b35)
