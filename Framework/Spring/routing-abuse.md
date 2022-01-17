# Path traversal with /..;/

Spring Boot > 2.2.6 treats `https://website.com/allowed/..;/internal` same as `https://website.com/allowed/../internal`.

This can lead to inconsistency between Spring and middleware. For instance, if an application is deployed behind nginx, you can bypass restrictions on allowed paths. Assume nginx forward all request to `/allowed/` to an application and deny other requests. In this case, request to `/allowed/../internal` will be blocked, however, `/allowed/..;/internal` is not - nginx will pass it as is to an application and it will actually hit `/internal`.

References:
- [@0xsapra tweet](https://mobile.twitter.com/0xsapra/status/1468551562712682499)
