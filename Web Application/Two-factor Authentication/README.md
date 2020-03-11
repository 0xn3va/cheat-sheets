# Two-factor Authentication

This section describes the possible vulnerabilities of two-factor authentication, which is implemented as a confirmation
 of user actions with a One-Time Password (OTP).

## Rate Limit

A rate limit is used to bandwidth throttling and / or a limit on the number of attempts when checking OTP. If the lack
 of rate limit, it becomes possible to bruteforce OTP during lifetime of session and / or OTP.

If the established restrictions are violated, the web application may return `429 Too Many Requests` or `200 OK` with an
 error in the response body. Also server may not change the HTTP code and response body, thus creating the appearance of
 a lack of rate limit. In order to verify this behavior, it is enough to enter a valid OTP, thus simulating the final
 result of the bruteforce.

### Bandwidth Throttling after Reaching a Certain Rate

Sometimes the rate limit algorithm can only respond to how many requests were made in a certain period of time. And if
 you try to bruteforce OTP using n-threads, rate limit algorithm will throttle the bandwidth. To avoid this behavior,
 you should implement bruteforce using a single thread and possibly even with a delay between requests of 1 or more
 seconds.

If the OTP does not have an expiration date, you will have a lot of time to bruteforce. But if an expiration date is
 present, the success of the attack is less, however, there is still a chance of a potential exploitation of this
 vulnerability.

### The OTP does not Change

If during a certain period of time the client receives the same OTP, in some cases there is the possibility of a
 successful bruteforce attack.

For example, a service uses numbers from `000` to `999` as OTP with an expiration time of 10 minutes. Assume that
 resending the OTP will send the user the same value for the lifetime. But the OTP resend request has a rate limit,
 which limits the number of attempts per request token. In this situation, you can generate a new token, substitute
 it in a request for resending OTP and successfully implement a bruteforce attack.

Im similar situation, you should try to bruteforce even some of the possible values, because there is no zero probability
 that the generated OTP is in the list of iterated values, and getting into the right combination confirms the vulnerability.

Also, do not disregard the situation when OTP is a long number. Attackers can find out from the client the first or last
 (for example, through social engineering) n-digits and thereby reduce the set of possible values.

### Reset Rate Limit when Updating OTP

If the OTP check has a rate limit that is reset after resending, you can successfully implement a bruteforce attack.

Example of reports:

- [Insecure 2FA/authentication implementation creates a brute force vulnerability](https://hackerone.com/reports/149598)
- [Authorization bypass using login by phone option+horizontal escalation possible on Grab Android App](https://hackerone.com/reports/205000)

### Bypassing Rate Limit by changing the IP

Many rate limit algorithms rely on the client IP address when deciding to block a request. If you change the IP address
 or make the server think that the request came from another IP, then you will successfully bypass this restriction.

#### Change IP Address

You can use a proxy or VPN to change the IP address. But to bypass the rate limit, you need to send requests from
 different IP addresses, and for this you need to automate the process of changing IP addresses. There are several ways
 to do this:

- `IP Rotator` extension for Burp Suite. `IP Rotator` allows you to bruteforce from different IP addresses without `42x`
 errors and disconnections. But if the web application is behind the CloudFlare firewall, all requests will be blocked,
 because `IP Rotator` sends requests using AWS IP addresses. In order to avoid this you need to find out the original IP.
- Ready-made or self-written script. But you need to have a large number of proxy servers or VPNs.

#### Using the X-Forwarded-For Header

If the server relies on the contents of the `X-Forwarded-For` header to determine the IP address of the client, you can
 try using this header to spoof the IP. To do this, just add the `X-Forwarded-For` header to the OTP check request,
 for example:

```
POST /api/v1/otp/check HTTP/1.1
Host: foo.com
Content-Type: application/x-www-form-urlencoded
Content-Length: 9
X-Forwarded-For: <desired-ip>

otp=12345
```

### Rate Limit in Account

Various functions in user account may require 2FA, for example, changing the email address, password, etc.
 The implementation of rate limit algorithm in user account may differ from the implementation of rate limit algorithm
 during authentication.

## Bypass by the substitution of part of the request

If during the OTP check the server additionally uses some parameter, try to send the value from the request to another
 account. For example, this parameter could be cookie, form or user ID. If you use the parameter value for **Account-1**
 (on this account you need to bypass 2FA) in the OTP verification request for **Account-2** (belongs to you), you can
 bypass 2FA for **Account-1**.

It’s also worth trying to use the valid OTP value for **Account-2** as the OTP value for **Account-1**, and if the server
 does not verify that the OTP belongs to account, you will bypass 2FA for **Account-1**.

## Bypass with "remember me"

Many applications with 2FA have the "remember me" functionality. This is needed so that the server "remembers" the user
 and does not require 2FA at the next sing-ins. In such cases, it is necessary to determine how the server remembers
 the user. The server can set cookies, store the value in session or local storage, or remember the IP address.

### Cookie Setting

The cookie value must be unguessable and the `HttpOnly` attribute must be set on the cookie so that it cannot be stolen 
 with XSS.

### IP Address Remembering

To replace the IP address, you can use the `X-Forwarded-For` header, if the server relies on this header to determine
 the client's IP address.

Using `X-Forwarded-For`, you can also bypass the `IP address whitelist` functionality. This is sometimes used together
 with 2FA as an additional account protection or can be used as a whitelist of IP addresses, from which server does not
 require 2FA. Thus, in some cases, you can bypass 2FA by other protection mechanisms.

## Improper Access Control to 2FA Page

Sometimes 2FA is a separate page to which the user gets to the URL with the parameters. If such a page can be accessed
 without cookies or with a different cookie than the one used to generate the URL, then it is necessary to check the
 following cases:
- URL expiration date,
- Search engine URL indexing.

If the URL is indexed by search engines and has a long expiration date, this avoids entering a credentials and, gain
 access to any account (if there is a 2FA bypass).

## Information Disclosure on 2FA Page

It is necessary to check whether initially unknown data are disclosed by requests, for example to API, for which you
 have enough rights at the 2FA stage.

## Ignoring 2FA

Sometimes 2FA can be ignored when performing some actions that lead to automatic login to the account.

### Password Recovery

Many services automatically log into account after the password recovery process is completed, and sometimes access to
 account is provided without 2FA.

### Sign in with a Social Network Account

If a user’s account can be linked to an account on the social network for quick login, then when entering the account
 through the social network 2FA can be ignored.

### Old API Version

If developers add an intermediate version of the application to a domain or subdomain to test certain functions,
 try signing in with your credentials, sometimes 2FA may be ignored.

It is also worth checking support for older versions of the API, which may not have 2FA or the rate-limit algorithm.

Example of reports:

- [I figured out a way to hack any of Facebook’s 2 billion accounts, and they paid me a $15,000 bounty...](https://medium.com/@iSecMax/two-factor-authentication-security-testing-and-possible-bypasses-f65650412b35)

### Cross Platform Applications

The implementation of 2FA in the mobile or desktop version of the application may differ from the web version.
 2FA may be weaker or completely absent.

### Disabling 2FA

If disabling 2FA does not require additional confirmation (OTP from google authenticator or SMS), there is a possibility
 of disabling 2FA with CSRF, XSS or Clickjacking attacks.

## Enabling 2FA does not End Previously Created Sessions

> This is the best practice to follow.

Enabling 2FA, like changing a password, should complete previously created concurrent sessions.

## Improper Access Control to the Backup Codes

Reserve codes are generated immediately after enabling 2FA and are available on a single request. For each subsequent
 request, codes can be generated by a new one or remain unchanged.

If there is a vulnerability that allows you to steal backup codes from a response to a request to the backup code display
 endpoint, then you can bypass 2FA.

## References

- [Two-factor authentication security testing and possible bypasses](https://medium.com/@iSecMax/two-factor-authentication-security-testing-and-possible-bypasses-f65650412b35)