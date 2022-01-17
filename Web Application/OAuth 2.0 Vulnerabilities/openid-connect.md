# OpenID Connect

[OpenID Connect](https://openid.net/specs/openid-connect-core-1_0.html) is a simple identity layer on top of the OAuth 2.0 protocol. It enables clients to verify the identity of the end-user based on the authentication performed by an authorization server, as well as to obtain basic profile information about the end-user in an interoperable and REST-like manner.

OpenID Connect implements authentication as an extension to the OAuth 2.0 authorization process. Use of this extension is requested by clients by including the `openid` scope value in the authorization request. Information about the authentication performed is returned in a JSON Web Token (JWT) called an `ID token`.

## OpenID Connect roles

In any OpenID Connect flow we can identify the following roles:
- `End-User` - human participant.
- `Relying Party (client)` - OAuth 2.0 client application requiring end-yser authentication and claims from an OpenID Provider.
- `OpenID Provider` - OAuth 2.0 authorization server that is capable of authenticating the end-user and providing claims to a Relying Party about the authentication event and the end-user.

## Protocol flow

The OpenID Connect protocol, in abstract, has the following flow:

![](img/openid-connect-flow.png)

1. The Replying Party (client) sends a request to the OpenID Provider.
2. The OpenID Provider authenticates the end-user and obtains authorization.
3. The OpenID Provider responds with an ID token and usually an access token.
4. The Replying Party (client) can send a request with the access token to the UserInfo endpoint.
5. The UserInfo endpoint returns claims about the end-user.

## Authentication

OpenID Connect performs authentication to log in the end-user or to determine that the end-user is already logged in. OpenID Connect returns the result of the authentication performed by the server to the client in a secure manner so that the client can rely on it. For this reason, the client is called Relying Party in this case.

Authentication can follow one of three paths:
- [Authorization Code Flow](https://openid.net/specs/openid-connect-core-1_0.html#CodeFlowAuth)
- [Implicit Flow](https://openid.net/specs/openid-connect-core-1_0.html#ImplicitFlowAuth)
- [Hybrid Flow](https://openid.net/specs/openid-connect-core-1_0.html#HybridFlowAuth)

## ID Token

The primary extension that OpenID Connect makes to OAuth 2.0 to enable end-users to be authenticated is the ID token data structure. The ID token is a security token that contains claims about the authentication of an end-user by an authorization server when using a client, and potentially other requested claims. The ID token is represented as a JSON Web Token (JWT) signed with a JSON Web Signature (JWS).

The following claims are used within the ID token for all OAuth 2.0 flows used by OpenID Connect (ID tokens may also contain other claims):

| Claim | Optionality | Description |
| --- | --- | --- |
| iss | Required | Issuer Identifier for the Issuer of the response. The iss value is a case sensitive URL using the `https` scheme that contains scheme, host, and optionally, port number and path components and no query or fragment components. |
| sub | Required | Subject Identifier. The sub value is a case sensitive ASCII string with a length not exceed 255 characters. A locally unique and never reassigned identifier within the Issuer for the end-user, which is intended to be consumed by the client. |
| aud | Required | Audience(s) that this ID token is intended for. The aud value is an array of case sensitive strings (or a single case sensitive string when there is one audience). It contains the OAuth 2.0 `client_id` of the Relying Party and may also contain identifiers for other audiences.  |
| exp | Required | Expiration time. |
| iat | Required | Time at which the JWT was issued. |
| auth_time | | Time when the end-user authentication occurred. |
| nonce | | String value used to associate a client session with an ID token, and to mitigate replay attacks. The value is passed through unmodified from the authentication request to the ID token. |
| acr | Optional | Authentication Context Class Reference, see [here](https://openid.net/specs/openid-connect-core-1_0.html#IDToken). |
| amr | Optional | Authentication Methods References. JSON array of strings that are identifiers for authentication methods used in the authentication. For instance, values might indicate that both password and OTP authentication methods were used. |
| azp | Optional | Authorized party - the party to which the ID token was issued, see [here](https://openid.net/specs/openid-connect-core-1_0.html#IDToken). |

### Authorization code flow

When using the authorization code flow, these additional requirements for the following ID token claims apply:

| Claim | Optionality | Description |
| --- | --- | --- |
| at_hash | Optional | Access token hash value. |

### Implicit flow

When using the implicit flow, these additional requirements for the following ID token claims apply:

| Claim | Optionality | Description |
| --- | --- | --- |
| nonce | Required | Use of the nonce claim is required for this flow. |
| at_hash | Required | Access token hash value; it may not be used when no access token is issued, which is the case for the `response_type=id_token`. |

### Hybrid flow

When using the hybrid flow, these additional requirements for the following ID token claims apply:

| Claim | Optionality | Description |
| --- | --- | --- |
| nonce | Required | Use of the nonce claim is required for this flow. |
| at_hash | Required | Access token hash value; it may not be used when no access token is issued. |
| c_hash | | Code hash value; it may not be used when no code is issued. |

## OpenID Connect endpoints

The OpenID Connect utilizes three endpoints:
- Authorization endpoint
- Token endpoint
- UserInfo endpoint

### Authorization endpoint

The authorization endpoint performs authentication of the end-user. This is done by sending the user agent to the authorization server's authorization endpoint for authentication and authorization, using request parameters defined by OAuth 2.0 and additional parameters and parameter values defined by OpenID Connect.

{% hint style="info" %}
The token endpoint is used with every authorization flow.
{% endhint %}

OpenID Connect uses the following OAuth 2.0 request parameters:

| Claim | Optionality | Description |
| --- | --- | --- |
| scope | Required | The scope of the access request. OpenID Connect requests must contain the `openid` scope value. |
| response_type | Required | OAuth 2.0 response type value that determines the authorization processing flow to be used, including what parameters are returned from the endpoints used. |
| client_id | Required | OAuth 2.0 client identifier valid at the authorization server. |
| redirect_uri | Required | Redirection URI to which the response will be sent. |
| state | Recommended | Opaque value used to maintain state between the request and the callback. Typically, Cross-Site Request Forgery mitigation is done by cryptographically binding the value of this parameter with a browser cookie. |

OpenID Connect also uses the additional request parameters:

| Claim | Optionality | Description |
| --- | --- | --- |
| nonce | Optional | String value used to associate a client session with an ID token, and to mitigate replay attacks. |
| display | Optional | ASCII string value that specifies how the authorization server displays the authentication and consent user interface pages to the end-user, see [here](https://openid.net/specs/openid-connect-core-1_0.html#AuthRequest). |
| prompt | Optional | Space delimited, case sensitive list of ASCII string values that specifies whether the authorization server prompts the end-user for reauthentication and consent, see [here](https://openid.net/specs/openid-connect-core-1_0.html#AuthRequest). |
| max_age | Optional | Maximum Authentication Age. Specifies the allowable elapsed time in seconds since the last time the end-user was actively authenticated by the OpenID Provider. |
| ui_locales | Optional | End-user's preferred languages and scripts for the user interface, see [here](https://openid.net/specs/openid-connect-core-1_0.html#AuthRequest). |
| id_token_hint | Optional | ID token previously issued by the authorization server being passed as a hint about the end-user's current or past authenticated session with the client, see [here](https://openid.net/specs/openid-connect-core-1_0.html#AuthRequest). |
| login_hint | Optional | Hint to the authorization server about the login identifier the end-user might use to log in, see [here](https://openid.net/specs/openid-connect-core-1_0.html#AuthRequest). |
| acr_values | Optional | Requested authentication context class reference values, see [here](https://openid.net/specs/openid-connect-core-1_0.html#AuthRequest). |

#### How response_type works

The `response_type` request parameter is used to inform the authorization server which flow type to use:
- `code` - value to use authorization code flow.
- `token` or `id_token token` - values to use implicit flow.
- `code id_token`, `code token` or `code id_token token` - values to use hybrid flow.

#### How nonce works

The `nonce` claim is used to associate a client session with an ID token, and to mitigate replay attacks. The value is passed through unmodified from the authentication request to the ID token. If ID token contains the `nonce` claim, clients verify that this value is equal to the value of the `nonce` parameter sent in the authentication request. In general, it works according to the following flow:
1. The client generates a secure random value and stores it as is, in clear text, in a HttpOnly session cookie.
2. The client use a cryptographic hash of the value as the nonce parameter and sends this as an authentication request parameter.
3. When ID token contains the nonce claim, the client pulls and removes the random value from persistent storage (in this case from cookie), hashes this value, and compares with the nonce claim in the ID token. If they do not match, the client refuses to establish identity.

See also [Nonce Implementation Notes](https://openid.net/specs/openid-connect-core-1_0.html#NonceNotes).

### Token endpoint

The token endpoint is used by relying party (client) to obtain an access token, an ID token, and optionally a refresh token. The token endpoint requires relying party (client) authentication and support several methods, details you can find [here](https://openid.net/specs/openid-connect-core-1_0.html#ClientAuthentication).

{% hint style="info" %}
The token endpoint is used by the authorization code and the hybrid flow types. 
{% endhint %}

When using the hybrid flow, the contents of an ID token returned from the Token Endpoint are the same as for an ID token returned from the authorization endpoint.

### UserInfo endpoint

The UserInfo endpoint is an OAuth 2.0 protected resource that returns claims about the authenticated end-user. To obtain the requested claims about the end-user, the client makes a request to the UserInfo endpoint using an access token obtained through OpenID connect authentication. These claims are normally represented by a JSON object that contains a collection of name and value pairs for the claims. You can see a standard set of basic profile claims [here](https://openid.net/specs/openid-connect-core-1_0.html#claims).

## Authorization Code Flow

The [authorization code flow](https://openid.net/specs/openid-connect-core-1_0.html#CodeFlowAuth) returns an authorization code to the client, which can then exchange it for an ID token and an access token directly. 

### How it works

![](img/openid-auth-code-flow.png)

1. Client prepares an authentication request containing the desired request parameters and sends the request to the authorization server (see also [how the authorization server validates the authentication request](https://openid.net/specs/openid-connect-core-1_0.html#AuthRequestValidation)). The request may look like this:

    ```http
    https://auth-server.com/auth?
        response_type=code&
        scope=openid%20profile%20email&
        client_id=s6BhdRkqt3&
        state=af0ifjsldkj&
        redirect_uri=https://application-website.com/callback
    ```

2. Authorization server [authenticates the end-user](https://openid.net/specs/openid-connect-core-1_0.html#Authenticates) and [obtains end-user consent/authorization](https://openid.net/specs/openid-connect-core-1_0.html#Consent).
3. Authorization server sends the end-user back to the client with an authorization code:

    ```http
    https://application-website.com/callback?
        code=SplxlOBeZQQYbYS6WxSbIA&
        state=af0ifjsldkj
    ```

4. Client requests a response using the authorization code at the token endpoint (see also additional information about [client authentication](https://openid.net/specs/openid-connect-core-1_0.html#ClientAuthentication) and [how the authorization server validates the token request](https://openid.net/specs/openid-connect-core-1_0.html#TokenRequestValidation)). The request may look like this:

    ```http
    POST /token HTTP/1.1
    Host: auth-server.com
    Content-Type: application/json
    Content-Length: 185

    {
        "client_id": "s6BhdRkqt3",
        "client_secret": "gX1fBat3bV",
        "grant_type": "authorization_code",
        "code": "SplxlOBeZQQYbYS6WxSbIA",
        "redirect_uri": "https://application-website.com/callback"
    }
    ```

5. Client receives a response that contains an ID token and access token in the response body, [validates the response](https://openid.net/specs/openid-connect-core-1_0.html#TokenResponseValidation) and retrieves the end-user's subject identifier:

    ```http
    HTTP/1.1 200 OK
    Content-Type: application/json
    Content-Length: 475
    Cache-Control: no-store
    Pragma: no-cache

    {
        "access_token": "SlAV32hkKG",
        "token_type": "Bearer",
        "refresh_token": "8xLOxBtZp8",
        "expires_in": 3600,
        "id_token": "eyJhbGciOiJSUzI1NiIsImtpZCI6IjFlOWdkazcifQ.
                    eyJpc3MiOiJodHRwczovL2F1dGgtc2VydmVyLmNvbSIs
                    InN1YiI6IjI0ODI4OTc2MTAwMSIsImF1ZCI6InM2Qmhk
                    UmtxdDMiLCJleHAiOjEzMTEyODE5NzAsImlhdCI6MTMx
                    MTI4MDk3MH0.
                    R06v-GNzsQiT8bc4DbkCm6laQ37fHcbLST4Pf52Yr6Jn
                    36DCP-TaWY6cyko6SyqmfPqqEJ3M9x5AtievsvM-CEGt
                    ggBP7-PeuicHS16199b6syKyQFfkUT8jFj_NcdVc-VL_
                    Zj6g_bgcjRpB1KLgKqqvA3hDBWhI2NHYeCI7eRY"
    }
    ```

## Implicit Flow

The [implicit flow](https://openid.net/specs/openid-connect-core-1_0.html#ImplicitFlowAuth) returns all tokens to clients from the authorization endpoint and the token endpoint is not used.

### How it works

![](img/openid-implicit-flow.png)

1. Client prepares an authentication request containing the desired request parameters and sends the request to the authorization server (see also [how the authorization server validates the authentication request](https://openid.net/specs/openid-connect-core-1_0.html#ImplicitValidation)). The request may look like this:

    ```http
    https://auth-server.com/auth?
        response_type=id_token%20token
        client_id=s6BhdRkqt3&
        redirect_uri=https://application-website.com/callback&
        scope=openid%20profile&
        state=af0ifjsldkj&
        nonce=n-0S6_WzA2Mj
    ```

2. Authorization server [authenticates the end-user](https://openid.net/specs/openid-connect-core-1_0.html#ImplicitAuthenticates) and [obtains end-user consent/authorization](https://openid.net/specs/openid-connect-core-1_0.html#ImplicitConsent).
3. Authorization server sends the end-user back to the client with an ID token and, if requested, an access token; client validates the authentication response and retrieves the end-user's subject identifier:

    ```http
    https://application-website.com/callback?
        access_token=SlAV32hkKG&
        token_type=bearer&
        id_token=eyJ0 ... NiJ9.eyJ1c ... I6IjIifX0.DeWt4Qu ... ZXso&
        expires_in=3600&
        state=af0ifjsldkj
    ```

## Hybrid Flow

The [hybrid flow](https://openid.net/specs/openid-connect-core-1_0.html#HybridFlowAuth) returns some tokens from the authorization endpoint and others from the token endpoint.

### How it works

![](img/openid-hybrid-flow.png)

1. Client prepares an authentication request containing the desired request parameters and sends the request to the authorization server (see also [how the authorization server validates the authentication request](https://openid.net/specs/openid-connect-core-1_0.html#HybridValidation)). The request may look like this:

    ```http
    https://auth-server.com/auth?
        response_type=code%20id_token&
        client_id=s6BhdRkqt3&
        redirect_uri=https://application-website.com/callback&
        scope=openid%20profile%20email&
        nonce=n-0S6_WzA2Mj&
        state=af0ifjsldkj
    ```

2. Authorization server [authenticates the end-user](https://openid.net/specs/openid-connect-core-1_0.html#HybridAuthenticates) and [obtains end-user consent/authorization](https://openid.net/specs/openid-connect-core-1_0.html#HybridConsent).
3. Authorization server sends the end-user back to the client with an authorization code and, depending on the response type, one or more additional parameters:

    ```http
    https://application-website.com/callback?
        code=SplxlOBeZQQYbYS6WxSbIA&
        id_token=eyJ0 ... NiJ9.eyJ1c ... I6IjIifX0.DeWt4Qu ... ZXso&
        state=af0ifjsldkj
    ```

4. Client requests a response using the authorization code at the token endpoint (see also additional information about [client authentication](https://openid.net/specs/openid-connect-core-1_0.html#ClientAuthentication) and [how the authorization server validates the token request](https://openid.net/specs/openid-connect-core-1_0.html#HybridTokenRequestValidation)). The request may look like this:

    ```http
    POST /token HTTP/1.1
    Host: auth-server.com
    Content-Type: application/json
    Content-Length: 185

    {
        "client_id": "s6BhdRkqt3",
        "client_secret": "gX1fBat3bV",
        "grant_type": "authorization_code",
        "code": "SplxlOBeZQQYbYS6WxSbIA",
        "redirect_uri": "https://application-website.com/callback"
    }
    ```

5. Client receives a response that contains an ID token and access token in the response body, [validates the response](https://openid.net/specs/openid-connect-core-1_0.html#HybridTokenResponseValidation) and retrieves the end-user's subject identifier.

    ```http
    HTTP/1.1 200 OK
    Content-Type: application/json
    Content-Length: 475
    Cache-Control: no-store
    Pragma: no-cache

    {
        "access_token": "SlAV32hkKG",
        "token_type": "Bearer",
        "refresh_token": "8xLOxBtZp8",
        "expires_in": 3600,
        "id_token": "eyJ0 ... NiJ9.eyJ1c ... I6IjIifX0.DeWt4Qu ... ZXso"
    }
    ```

# Security issues in the OAuth2.0 protocol

{% embed url="https://0xn3va.gitbook.io/cheat-sheets/web-application/oauth-2.0-vulnerabilities" %}

# Security issues with JSON Web Token

{% embed url="https://0xn3va.gitbook.io/cheat-sheets/web-application/json-web-token-vulnerabilities" %}

# Security issues in the relying party

## Improper handling of nonce claim

If the `nonce` claim is:
- missing,
- a static value that never changes,
- present but not validated,
- available to an attacker in cleartext on the client side,

the OpenID Connect flow is likely to be vulnerable to replay attack. To exploit this, you need obtain the authentication response and use this to establish identity.

# Security issues in the OpenID provider

{% hint style="info" %}
Try to discover the `.well-known/openid-configuration` endpoint, according to the [specification](https://openid.net/specs/openid-connect-discovery-1_0.html#ProviderConfig) it contains useful information about different server configuration values.
{% endhint %}

## Abusing WebFinger service

In order for an OpenID Connect relying party to utilize OpenID Connect services for an end-user, the replying party needs to know where the OpenID provider is. OpenID Connect uses [WebFinger](https://tools.ietf.org/html/rfc7033) to locate the OpenID provider for an end-user. Details you can find in the [specification](https://openid.net/specs/openid-connect-discovery-1_0.html).

The WebFinger service is available at the `.well-known/webfinger` endpoint and allow you to get information about the users and resources used on the server. It can be used to validate that the "anonymous" user has an account on the server:

```http
https://openid-server.com/.well-known/webfinger?
    resource=http://x/anonymous&
    rel=http://openid.net/specs/connect/1.0/issuer
```

```http
HTTP/1.1 200
Content-Type: application/json
Content-Length: 132

{
    "subject": "http://x/anonymous",
    "links": [
        "rel": "http://openid.net/specs/connect/1.0/issuer",
        "href": "https://auth-server.com/auth"
    ]
}
```

The `rel` parameter should have a static value of `http://openid.net/specs/connect/1.0/issuer`.

The `resource` should contain a valid URL in one of the following forms:
- `http(s)://host/user`
- `acct://user@host`

## UserInfo endpoint: CORS misconfiguration

The UserInfo endpoint should support the use of Cross-origin resource sharing according to the [specification](https://openid.net/specs/openid-connect-core-1_0.html#UserInfo).

{% embed url="https://0xn3va.gitbook.io/cheat-sheets/web-application/cors-misconfiguration" %}

## SSRF via dynamic client registration

The [OpenID Connect Dynamic Client Registration](https://openid.net/specs/openid-connect-registration-1_0.html) specification defines a standardized way of allowing client applications to register with the OpenID provider. If dynamic client registration is supported, the client application can register itself by sending a `POST` request to a `/register` endpoint. The registration request may look like this:

```http
POST /connect/register HTTP/1.1
Host: auth-server.com
Content-Type: application/json
Authorization: Bearer eyJhbGciOiJSUzI1NiJ9.eyJ...

{
    "application_type": "web",
    "redirect_uris":["https://application-website.com/callback"],
    "client_name": "My Example",
    "logo_uri": "https://application-website.com/logo.png",
    "subject_type": "pairwise",
    "sector_identifier_uri": "https://other-application-website.com/file_of_redirect_uris.json",
    "token_endpoint_auth_method": "client_secret_basic",
    "jwks_uri": "https://application-website.com/my_public_keys.jwks",
    "userinfo_encrypted_response_alg": "RSA1_5",
    "userinfo_encrypted_response_enc": "A128CBC-HS256",
    "contacts": ["mail@application-website.com"],
    "request_uris":["https://application-website.com/rf.txt"]
}
```

Some of these values are passed in via URL links and can be potentially vulnerable to SSRF.

{% hint style="info" %}
Most servers do not resolve these URLs immediately when they receive a registration request. Instead, they just save these parameters and use them later during the OAuth authorization flow.
{% endhint %}

The following parameters are particularly interesting for SSRF attacks:
- `logo_uri` - URL that references a logo for the client application. After you register a client, you can try to call the OAuth authorization endpoint using your new "client_id". After the login, the server will ask you to approve the request and may display the image from the "logo_uri". If the server fetches the image by itself, the SSRF should be triggered by this step. Alternatively, the server may just include the logo via a client-side `&lt;img&gt;` tag, which can lead to XSS.
- `jwks_uri` - URL for the client's JSON Web Key Set document. This key set is needed on the server for validating signed requests made to the token endpoint when using JWTs for client authentication. In order to test for SSRF in this parameter, register a new client application with a malicious "jwks_uri", perform the authorization process to obtain an authorization code for any user, and then fetch the token endpoint with body as in the request below. If vulnerable, the server should perform a server-to-server HTTP request to the supplied "jwks_uri" because it needs this key to check the validity of the "client_assertion" parameter in your request. This will probably only be a blind SSRF vulnerability though, as the server expects a proper JSON response.

    ```http
    POST /token HTTP/1.1
    Host: auth-server.com
    Content-Type: application/json

    {
        "client_assertion_type": "urn:ietf:params:oauth:client-assertion-type:jwt-bearer",
        "client_assertion": "eyJhbGci...",
        "grant_type": "authorization_code",
        "code": "SplxlOBeZQQYbYS6WxSbIA"
    }    
    ```

- `sector_identifier_uri` - This URL references a file with a single JSON array of `redirect_uri` values. If supported, the server may fetch this value as soon as you submit the dynamic registration request. If this is not fetched immediately, try to perform authorization for this client on the server. As it needs to know the `redirect_uris` in order to complete the authorization flow, this will force the server to make a request to your malicious `sector_identifier_uri`.
- `request_uris` - An array of the allowed `request_uris` for this client. The `request_uri` parameter may be supported on the authorization endpoint to provide a URL that contains a JWT with the request information, see also [here](https://openid.net/specs/openid-connect-core-1_0.html#rfc.section.6.2). To exploit SSRF on the authorization endpoint, simply use `request_uri`:
    
    ```http
    https://auth-server.com/auth?
        response_type=code%20id_token&
        client_id=s6BhdRkqt3&
        request_uri=https://attacker-website.com/request.jwt
    ```

The following parameters also contain URLs, but are not normally used for issuing server-to-server requests. They are instead used for client-side redirection/referencing:
- `redirect_uri` - URLs that are used for redirecting clients after the authorization
- `client_uri` - URL of the home page of the client application
- `policy_uri` - URL that the relying party client application provides so that the end user can read about how their profile data will be used.
- `tos_uri` - URL that the relying party client provides so that the end user can read about the relying party's terms of service.
- `initiate_login_uri` - URI using the https scheme that a third party can use to initiate a login by the relying party. Also should be used for client-side redirection.

All parameters and their definitions you can find [here](https://openid.net/specs/openid-connect-registration-1_0.html#ClientMetadata).

## SSRF via request_uri parameter

The [request_uri](https://openid.net/specs/openid-connect-core-1_0.html#UseRequestUri) parameter is fetched by the server at the start of the authorization process (do not confuse this parameter with `redirect_uri`).

Many servers do not allow arbitrary `request_uri` values: they only allow whitelisted URLs that were pre-registered during the client registration process. But even if dynamic client registration is not enabled, or it requires authentication, you can try to bypass a validation and perform SSRF on the authorization endpoint by using `request_uri`:

```http
https://auth-server.com/auth?
    response_type=code%20id_token&
    client_id=s6BhdRkqt3&
    request_uri=https://attacker-website.com/request.jwt
```

# References

- [OpenID Connect Core 1.0](https://openid.net/specs/openid-connect-core-1_0.html)
- [PortSwigger Articles: Hidden OAuth attack vectors](https://portswigger.net/research/hidden-oauth-attack-vectors)
