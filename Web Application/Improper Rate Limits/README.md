# Rate limits overview

A rate limit is used to bandwidth throttling and/or a limit on the number of attempts, for instance, when checking passwords or OTP. If the lack of rate limit, it becomes possible to send an arbitrary number of requests.

If the established restrictions are violated, an application may return `429 Too Many Requests` or `200 OK` with an error in the response body. However, an application may not change the HTTP code and response body, thus creating the appearance of a lack of rate limit. In some cases you can verify this behavior by entering a valid value, for instance, valid password or OTP, thus simulating the final result of bruteforce.

# Bandwidth throttling after reaching a certain rate

Rate limit algorithms can only respond to how many requests were made in a certain period of time. If you try to send requests using n-threads, rate limit algorithm will throttle the bandwidth. To avoid this behavior, you should send requests using a single thread and possibly even with a delay between requests.

# Changing IP address

Many rate limit algorithms rely on a client IP address when deciding to block a request. If you change the IP address or make a server think that the request came from another IP, you will successfully bypass this restriction.

## Override original IP

If an application relies on the value of HTTP headers to determine an original IP address of a client, you can try to override IP address with the following headers:

{% embed url="https://gist.github.com/0xn3va/c27cdcee6a7e84300165d9ec25a3d2b4" %}

For instance, you can add the `X-Forwarded-For` header to the OTP check request:

```http
POST /api/v1/otp/check HTTP/1.1
Host: vulnerable-website.com
Content-Type: application/x-www-form-urlencoded
Content-Length: 9
X-Forwarded-For: <desired-ip>

otp=12345
```

## Using proxy or VPN

You can use a proxy or VPN to change an IP address. To automate the change of IP addresses, use the following methods:

- [IPRotate_Burp_Extension](https://github.com/PortSwigger/ip-rotate). Extension for Burp Suite which uses AWS API Gateway to rotate your IP on every request.
- [requests-ip-rotator](https://github.com/Ge0rg3/requests-ip-rotator). A Python library to utilize AWS API Gateway's large IP pool as a proxy to generate pseudo-infinite IPs for web scraping and brute forcing.
- A ready-made or self-written script with a large number of proxy servers or VPNs.

{% hint style="info" %}
If an application is behind a CloudFlare firewall, all requests sent from AWS IPs will be blocked. To avoid this, you need to discover an original IP of an application.
{% endhint %}

# Changing path

Try to change an endpoint path to bypass rate limits. Use different case or add extra symbols, such as `%00`, `%09`, `%0a`, `%0c`, `%20`, etc. For instance, if a bare endpoint is `/api/v4/endpoint`, try the following endpoints:

- `/api/v4/endpoint`
- `/api/v4/Endpoint`
- `/api/v4/EndPoint`
- `/api/v4/endpoint%00`
- `/api/v4/%0aendpoint`
- `/api/v4/endpoint%09`
- `/api/v4/%20endpoint`
- ...

Also, you can try to add extra parameters, for example, `/api/v4/endpoint?some_param=1`

# Extra characters in parameters

Try to add extra symbols, such as `%00`, `%09`, `%0a`, `%0c`, `%20`, etc. to params. For instance, if requesting a code to an email have only n tries, use `name@website.com%00` after exceeding n attempts, then `name@website.com%20`, and etc.

# Multiple values in request

When brute-force, try passing several values in one request at once, for instance:

```json
{
    "phone": "+17342239011",
    "code": [
        "123456",
        "654321",
        "133713",
        // ...
        "331337"
    ]
}
```

or

```http
phone=+17342239011&code[]=123456&code[]=654321&...&code[]=331337
```

An application can correctly process such requests and interpret this as one attempt, which will significantly expand the number of possible attempts.

# Race condition

{% embed url="https://0xn3va.gitbook.io/cheat-sheets/web-application/race-condition" %}

# Rate limits after authentication

An application can implement different rate limits for authenticated and unauthenticated users. For instance, after authentication, there may be no rate limits on attempts to change a password or disable two-factor authentication.

# Reset a rate limit

An application may incorrectly implement rate limits or have a logical vulnerability that allows a user to reset the limits.

For example, an application may reset limits on attempts to send OTP when you resend a new OTP, so you can reset rate limits before each OTP check attempt and a rate limit will be never reached. Or an application may store the remaining number of attempts in a cookie, and you can reset the limit by spoofing the cookie.

# Send requests to different instances of an application

Often, an application backend is deployed on multiple instances that run in parallel. Accordingly, when a client sends a request to the backend, a balancer ties a user's session to a specific instance. This can be accomplished by setting a custom HTTP header or cookie with an instance ID. If rate limit values are not synchronized between instances, you can increase the number of available attempts by sending requests to different instances. In order to find out existing IDs, try to send requests to the backend from different IPs and without the header or cookie. A balancer will assign a random instance and return it in the header or cookie for such requests.

# References

- [Two-factor authentication security testing and possible bypasses](https://medium.com/@iSecMax/two-factor-authentication-security-testing-and-possible-bypasses-f65650412b35)
