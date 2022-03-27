# Abusing "remember me" function

An application can implement the "remember me" function so that the next authentication from the same device does not require to enter the second factor. To implement this functionality, an application can save a long-lived token to cookies and/or remember an IP address.

In case it is a long-lived token, you can check the following points:

1. Token is randomly-generated
2. Token is different for each user and session (device)
3. Token is stored secure and not accessable from JavaScript, check [Cookie Security](/Web%20Application/Cookie%20Security/README.md)
4. Check if it is possible to revoke remembered devices remotely

If an application remember an IP, check if it possible to spoof an IP using HTTP headers, check [Abusing IP whitelists](#abusing-ip-whitelists)

# Abusing IP whitelists

An application can "memorizes" the IP address from which a user is authenticated and does not ask for the second factor at further logins. If an application relies on HTTP headers to determine a client's IP address, try to spoof the IP address using the following headers:

{% embed url="https://gist.github.com/0xn3va/c27cdcee6a7e84300165d9ec25a3d2b4" %}

# Common cases when using OTP

{% embed url="https://0xn3va.gitbook.io/cheat-sheets/web-application/broken-authentication#phone-and-otp-authentication" %}

# Enabling 2FA does not end previously created sessions

Enabling two-factor authentication must revoke all existing sessions. If it does not happen, a victim will not be able to manage the security of their account. As a result, an attacker can keep an active session created before a second factor was enabling for an extended period of time.

# Ignoring 2FA

An application can ignore two-factor authentication when performing actions that can lead to automatic login to an user account.

## Abuse of half-authenticated sessions

An application can issue a session token with limited access after providing credentials. Try to use this session token in an un-enrollment request to disable 2FA (or access API endpoints which require authentication). You can check it with the next steps:

1. Submit credentials to an application
2. Catch a session token from the response
3. Stop an authentication on the 2FA step
4. Use the token in a un-enrollment request (or an API request)
5. Login into account without 2FA requirement (access sensitive data / perform changes)

References:
- [Writeup: Bypassing Box's Time-based One-Time Password MFA](https://www.varonis.com/blog/box-mfa-bypass-totp/)

## Cross platform applications

The implementation of two-factor authentication for mobile or desktop version of an application may differ from a web one. Two-factor authentication may be weaker or completely absent.

## Disabling 2FA does not require confirmation

If an application does not require an additional confirmation (current password, current OTP, code from SMS, etc.) when disabling two-factor authentication, this provides an option to disable two-factor authentication via CSRF, XSS or Clickjacking attacks.

## Password recovery / email confirmation

An application can automatically log into an account without requiring a second factor after the password recovery / email confirmation process is completed. For instance, you can check it for the password recovery with the following steps:

1. Log into an account.
2. Enable two-factor authentication.
3. Go through password recovery.
4. If after changing your password you got into your account without a second factor, an application is vulnerable.

## Sign in with a third-party account

An application can ignore two-factor authentication when logging in through a third-party account, for instance, social networks, Github, Google, etc. You can check it with the following steps:

1. Log into an account.
2. Enable two-factor authentication.
3. Link a third-party account.
4. If logging in through a third-party account does not require a second factor, an application is vulnerable.

## Using staging or old API

If a staging version of an application is publicly available, try logging in with your credentials. In such environments, two-factor authentication can be disabled. Non-production environments can be located on separate domains or subdomains of the main domain, for instance:

- `beta.website.com`
- `stage.website.com`
- `beta-website.com`
- `stagewebsite.com`
- etc.

Additionally check if older API versions are available, they may not require a second factor or not have rate-limit algorithms. If an application supports API versioning, the version can be passed in the URL or HTTP headers, for instance:

- `https://api.website.com/v1/users`
- `https://website.com/api/v1/users`
- `Accept: application/test.data.v1.12+json`
- etc.

References:
- [Writeup: I figured out a way to hack any of Facebook’s 2 billion accounts, and they paid me a $15,000 bounty...](https://www.freecodecamp.org/news/responsible-disclosure-how-i-could-have-hacked-all-facebook-accounts-f47c0252ae4d/)

# Improper access control to 2FA page

With two-factor authentication, an application can direct a user to a special page using a URL with predefined parameters. If this page can be accessed without cookies or using cookies from another session, you can check the following cases:
- URL generation
- URL expiration date
- Search engine URL indexing

If the URL is guessed, indexed by search engines, has a long expiration date, it can be used to sign in to accounts without entering credentials. Therefore, if there is a way to bypass two-factor authentication, you can access other users account.

# Improper access control to the backup codes

Backup codes are generated immediately after a second factor is enabled and are available only after enabling. For each subsequent request, the codes can be generated anew or remain unchanged.

Try to find a vulnerability that could steal backup codes from a response to a request to backup code display endpoint.

# Improper rate limits

{% embed url="https://0xn3va.gitbook.io/cheat-sheets/web-application/improper-rate-limits" %}

# Time-based 2FA

## Long-lived codes

Applications that use time-based codes typically maintain a time interval during which the codes are valid, so people can type them slowly. Since new codes are generated in steps of `n` seconds (generally `n` is 30 - 60 seconds), multiple codes are valid at the same time. However, each code in the allowable window is a valid code for the user's account. If the codes are valid for an hour, many codes are valid at the same time within this window. Therefore, it is much easier to guess the valid code, because there are more than one valid code in the possible set of codes.

For example, if the codes are generated in steps of 30 seconds and a common validity range here is about 5 minutes in either direction from the time as set on the server, the allowable window is `[current_time - 5 mins, current_time + 5mins]` and the number of valid codes at the same time is `10 mins * 60 sec / 30 sec = 20 codes`.

References:
- [Report: Insecure 2FA/authentication implementation creates a brute force vulnerability](https://hackerone.com/reports/149598)

## Using old codes

An application can revoke the current code and does not store the most recent code's timestamp for rejecting codes that are older than the current one. As a result, an attacker is able to use a code that is older that the one, that the user chooses to enter. You can check it with the following steps:

1. Open two different browsers and go to log in
2. Get to the 2FA screen in both browsers
3. Obtain a valid 2FA code, and write it down, do not use it yet
4. Wait for the next valid 2FA code, and log in using it in one browser
5. In the other browser, try to log in with the old 2FA code that was written down
6. If you can log in using both sessions, the application is vulnerable

# Mixing 2FA modes

An application can provide multiple 2FA modes, such as SMS- and TOTP-based 2FA processes. If an application uses a per-user unique parameter other than a half-authenticated session cookie to refer to the desired 2FA mode, you can try to complete an authentication process with your 2FA mode that is not used by a victim. An example of an attack flow might contains the next steps:

1. Victim set up SMS-based 2FA
2. Attacker enrolls in 2FA using an authenticator app (TOTP-based 2FA) and saves their unique parameter, for instance `factor_id`
3. Attacker enters a victim's email address and password
4. If the password is correct, an attacker's browser is sent a new authentication cookie and redirects to `/mfa/sms/verification`
5. Attacker does not follow the redirect to the SMS verification form. Instead, attacker sends their `factor_id` and code from the authenticator app with victim's half-authenticated cookie to TOTP verification endpoint: `/mfa/totp/verification` endpoint.

References:
- [Writeup: Mixed Messages: Busting Box’s MFA Methods](https://www.varonis.com/blog/box-mfa-bypass-sms)

# Session persistence to the 2FA flow

Typically, an authentication flow initializes a session after creds have been passed and waits for a second factor to complete the flow. Try to change password/disable 2FA in this step and complete the authentication flow. If the session does not time out or you can renew it yourself (for example by changing the authentication way) and an app allows you to login, you can still access the user account even after changing password/disabling 2FA.

An example of an attack might look as follows:

1. An attacker gains access to a victim's account that does not have 2FA.
2. The attacker enables 2FA for the victim's account.
3. In another browser, the attacker stops after passing creds and waits indefinitely at this step.
4. The attacker disables 2FA for the victim's account.
5. The victim regains access to the account and changes password/resets sessions.
6. The attacker completes the flow with 2FA and gain access to the account again.

References:
- [Writeup: How I abused 2FA to maintain persistence after a password change (Google, Microsoft, Instagram, Cloudflare, etc)](https://medium.com/@lukeberner/how-i-abused-2fa-to-maintain-persistence-after-a-password-change-google-microsoft-instagram-7e3f455b71a1)

# Weak shared secret

Most code-based 2FA systems rely on a shared secret. If this secret is not actually cryptographically secure, or is very short, an attacker may be able to guess it. As a result, an attacker will be able to generate valid codes.

If you have source code access, ensure that the secret is being generated with cryptographically strong generator. Most programming languages' default random number generators are [predictable](https://jazzy.id.au/2010/09/20/cracking_random_number_generators_part_1.html), meaning an attacker might be able to figure out which secrets were assigned to which users. Additionally, verify that a shared secret is [20 bytes long at least](https://datatracker.ietf.org/doc/html/rfc6238#section-5.1), because shorter secrets should be avoided for security purposes.

If you can review the source code, try to check the secret for lenght and apparent randomness (scan a QR code with a barcode scanner to retrive the secret). Generate several secrets for the same or different accounts and try to answer the following questions:

- Is each secret different?
- Does any part of the secret appear to be sequential?
- Is any part of the secret based on the account information?
- Otherwise insecure?

# References

- [Two-factor authentication security testing and possible bypasses](https://medium.com/@iSecMax/two-factor-authentication-security-testing-and-possible-bypasses-f65650412b35)
- [NCC Group Research: Testing Two-Factor Authentication](https://research.nccgroup.com/2021/06/10/testing-two-factor-authentication/)
- [List of websites and whether or not they support 2FA](https://twofactorauth.org/)