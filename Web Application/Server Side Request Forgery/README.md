# Server Side Request Forgery

Server Side Request Forgery is a vulnerability in which an attacker forces a server to perform requests on their behalf.

## Credentials Bruteforce

SSRF allows you to bruteforce credentials for resources that use Basic access authentication as an authentication
 mechanism. To do this, just create a following link:

```http request
http://login:password@site.com/path/
```

## Responses Anomaly

Sometimes you can rely on an anomaly of answers when using SSRF, if the response of request execution is not available
 to you. To do this, you need to access internal resources and measure the response time for each request. Response time
 is an indirect sign that may indicate the availability of a resource. Having sent a lot of requests, you need to search
 among them those for which the response time is different from all the others. This approach allows you to blindly
 bruteforce internal services, open ports, directories and files.

## Bypass IP Check

Often applications implement IP address verification, for example, that it is not loopback, IPv4 link local, private
 address (RFC1918), etc. If the mechanisms for checking and establishing a connection are independent and there is no
 caching of the DNS resolution response, you can bypass this by controlling the DNS resolution response. To do this,
 you can use the service `http://1u.ms`. This service allows you to return the IP address that is being checked at the
 first request for a domain name resolution and on a second request will return desired IP address.

For example, if two requests go one after another, for 5 seconds, with DNS resolution

```http request
make-1-1-1-1-rebind-127-0-0-1-rr.1u.ms
```

the first DNS query, the address `1.1.1.1` will be returned, and to the second` 127.0.0.1`.

## Port Scanning Using DNS

Many libraries try to access the resource by IP in the order that they are placed in DNS records. For example, if the
 DNS records look like this:

```http request
site.com 172.16.1.1
site.com 172.16.1.2
```

first there will be an attempted connect to `172.16.1.1`, and if problems arise, to `172.16.1.2`. This allows you to
 find out which ports are open and which are not.

For this you can also use the service `http://1u.ms`. For example, if you need to find available ports on `127.0.0.1`,
 you can use

```http request
make-127-0-0-1-and-123-123-123-123rr.1u.ms
```

this will allow you to change the port number to determine which port is available.

```http request
http://make-127-0-0-1-and-123-123-123-123rr.1u.ms:22 - запрос не пришел на 123.123.123.123:22
http://make-127-0-0-1-and-123-123-123-123rr.1u.ms:80 - запрос пришел на 123.123.123.123:80
http://make-127-0-0-1-and-123-123-123-123rr.1u.ms:6379 - запрос не пришел на 123.123.123.123:6379
http://make-127-0-0-1-and-123-123-123-123rr.1u.ms:8080 - запрос пришел на 123.123.123.123:8080
```

This shows that ports `22` and `6379` are open on `127.0.0.1` because there were no connection attempts for the IP
 address from the second DNS record.

> It is worth paying attention to what DNS server resolves names on the backend side. DNS server can use the built-in 
 round robin algorithm for resolving domain names and change the order of records

## References

- [Подделка серверных запросов, эксплуатация Blind SSRF](https://bo0om.ru/blind-ssrf)