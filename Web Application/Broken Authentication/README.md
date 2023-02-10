# Adding or changing email addresses

If the operations of adding a new email address or changing an existing one do not require password confirmation, session hijacking, XSS or CSRF can lead to account takeover.

# Amazon Cognito misusing

{% embed url="https://0xn3va.gitbook.io/cheat-sheets/cloud/aws/amazon-cognito" %}

# Email confirmation

## Binding an email using a confirmation link

Try to follow a confirmation link for account `A` within the session of account `B` within an email confirmation flow. If an application is vulnerable, it will link the verified email to account `B`. In this case, the attack flow may look like:

1. An attacker links `attacker@website.com` to their account.
1. An attacker sends a confirmation link to a victim.
1. A victim follows the link from an email while logged into an application.
1. An application links `attacker@website.com` to a victim.

References:
- [Writeup: Watch out the links : Account takeover!](https://akashhamal0x01.medium.com/watch-out-the-links-account-takeover-32b9315390a7)

## Confirmation of multiple emails

A method for adding new emails can accept several email address in a single request. However, the method can create a one-time token based on only one email, but send an email with a recovery link to all passed emails at once. For instance, if an application is vulnerable to such an attack, on the following request:

```http
PUT /user/profile HTTP/1.1
Host: vulnerable-website.com
Content-Type: application/json
Content-Length: 57

{"email": ["victim@website.com", "attacker@website.com"]}
```

an application sends the same confirmation link to two email addresses.

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

Another similar case is if there is a business logic vulnerability when confirmation link is sent to a wrong email. For example, if an application sends a confirmation link to an already added, main email, instead of an unconfirmed one.

References:
- [Report: Email Confirmation Bypass in myshop.myshopify.com that Leads to Full Privilege Escalation to Any Shop Owner by Taking Advantage of the Shopify SSO](https://hackerone.com/reports/791775)

## Skipping the confirmation process

If an application allows you to create users without an email confirmation process, you can try to abuse that to get a new user with a pre-confirmed email. This is possible at least in the following cases:

- An application allows creation of bot users that have pre-defined confirmed emails.
- SCIM provisioning functions.
- OAuth authentication via a vulnerable service that allows using unconfirmed accounts for authentication.

References:
- [Report: Bypass Email Verification -- Able to Access Internal Gitlab Services that use Login with Gitlab and Perform Check on email domain](https://gitlab.com/gitlab-org/gitlab/-/issues/11643)

## Using unconfirmed emails

If an application allows using unconfirmed emails, try to abuse this behavior in flows where an application or other applications/systems rely on an email address. For example:

- If there are applications/systems that blindly trust the data of a vulnerable application, you can try to abuse this trust. For example, if an application can be used as an OAuth authorization server, try using an account with an unconfirmed email to authenticate to a third-party application using OAuth, you can find more details at [OAuth 2.0 Vulnerabilities: Abusing accounts with unconfirmed email](/Web%20Application/OAuth%202.0%20Vulnerabilities/README.md#abusing-accounts-with-unconfirmed-email).
- Try using unconfirmed emails to preoccupy emails that may be used by other users or used internally by an application. In this case, you can block some application functions for all or specific users.
- If some functions must not be available for users who have not confirmed their email, check that these functions are really not available for them. Additionally, make sure the REST and GraphQL APIs also abide by the same policy.

## Using OTP for email confirmation

{% embed url="https://0xn3va.gitbook.io/cheat-sheets/web-application/broken-authentication#phone-and-otp-authentication" %}

## Weak confirmation token

Confirmation token may be generated using a vulnerable generation algorithm, which may lead to the possibility of predicting the generated values. If you manage to predict tokens you will be able to generate valid confirmation tokens for any emails.

{% embed url="https://0xn3va.gitbook.io/cheat-sheets/web-application/weak-random-generation" %}

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

Successful password recovery should end previously created sessions. If this does not happen, an victim will not have mechanisms for managing the security of their account. As a result, an attacker will be able to maintain an active session for an extended period of time.

## Token leakage via Referer header

The [Referer](https://developer.mozilla.org/en-US/docs/Web/HTTP/Headers/Referer) HTTP request header contains an absolute or partial address of the page that makes the request. The `Referer` header allows a server to identify a page where people are visiting it from. This data is used for analytics, logging, optimized caching, and more.

The Referer header can contain an `origin`, `path`, and `querystring`, and may not contain URL fragments (i.e. `#section`) or `username:password` information. The request's referrer policy defines the data that can be included, see [Referrer-Policy](https://developer.mozilla.org/en-US/docs/Web/HTTP/Headers/Referrer-Policy).

A page with a password recovery, a link to which an application sends by email, can load third-party scripts, for instance, analytics, or contain links to third-party resources, for instance, social networks. If the password recovery link is passed into the `Referer` header when requesting these resources, an one-time token is leaked to a third-party resource, since it is passed to the `querystring`.

References:
- [Report: Cross Domain leakage of sensitive information - Leading to Account Takeover at Instagram Brand](https://hackerone.com/reports/209352)
- [Report: [Cross Domain Referrer Leakage] Password Reset Token Leaking to Third party Sites.](https://hackerone.com/reports/265740)
- [Report: Referer Referer Header Leakage in language changer may lead to FB token theft](https://hackerone.com/reports/870062)

## Weak recovery token

Recovery token may be generated using a vulnerable generation algorithm, which may lead to the possibility of predicting the generated values. If you manage to predict tokens you will be able to generate valid recovery tokens for any accounts.

{% embed url="https://0xn3va.gitbook.io/cheat-sheets/web-application/weak-random-generation" %}

# Phone and OTP authentication

## OTP resend

If an application does not set rate limits on resending OTP and resending OTP resets the set rate limits on attempts to enter OTP, an application is vulnerable to OTP bruteforce.

References:
- [Report: Authorization bypass using login by phone option+horizontal escalation possible on Grab Android App](https://hackerone.com/reports/205000)

## OTP reusing

If an application allows using of old OTPs that have been used or re-generated, any leakage of OTPs (for example, to logs or third-party services) will compromise the functionality that relies on OTPs.

## Session fixation

An application can implement the authentication flow as the following one:

1. An application calls `/SessionCreate` endpoint with a mobile phone number of a user
2. A backend creates a session for a user and returns a session token, but no operations with this session are possible until the verification is complete
3. An SMS message is sent to a user with a verification code
4. An application calls `/SessionVerify` endpoint with both the session token and the verification code received by SMS
5. Once this request is successfully completed, the session token becomes valid and the user is now logged in

If subsequent calls to `/SessionCreate` return the same session token as the first one until a call to `/SessionVerify`, you can use `/SessionCreate` endpoint to fecth a session token, that will valid after victim's authentication.

References:
- [Report: Account Takeover via SMS Authentication Flow](https://hackerone.com/reports/1245762)

## Spoofing part of an authentication request

If during the OTP check an application additionally uses some parameters, try to use a value from a request for a different account. This parameter can be a cookie, header, or request parameter. If you use the parameter value for a victim in the OTP check request for an attacker, you can get into a victim account or bypass two-factor authentication.

Additionally, try to re-use the valid OTP value for an attacker as the OTP value for a victim. If an application does not verify that the OTP belongs to an attacker, you will be able to get into a victrim account or bypass two-factor authentication.

## Short and long-lived OTP

An application can use short (less 5 digits) and/or long-lived OTPs. Such implementation leaves many ways to bruteforce, especially if there are no rate limits, or they are weak or can be bypassed.

# Rate limits

{% embed url="https://0xn3va.gitbook.io/cheat-sheets/web-application/improper-rate-limits" %}

# Third-party sign-in or sign-up

Applications that implement third party sign-in or sign-up often identify users based on the attached email address that the third party application sends. In such cases, you can try sign-in or sign-up with two different third-party applications that are linked to the same email address. Sometimes these two accounts can be joined and you will have access to the victim's account by sign-in or sign-up with your third-party application account.

For example, suppose a victim sign-in or sign-up to `vulnerable-website.com` with `third-party-app1.com`, which is linked to `victim@website.com`. To exploit this, try the following chain:

1. Create an account at `third-party-app2.com` and enter the `victim@website.com` email address (ideal if email confirmation is not required)
2. Try sign-in or sign-up at `vulnerable-website.com` through `third-party-app2.com`
3. If the application is vulnerable, you will have access to the victim's account

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
