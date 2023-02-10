# OAuth 2.0 Authorization Framework

[OAuth 2.0](https://tools.ietf.org/html/rfc6749) is a protocol that allows a user to grant a third-party web site or application access to the user's protected resources, without necessarily revealing their long-term credentials or even their identity.

OAuth 2.0 introduces an authorization layer and separates the role of the client from that of the resource owner. In OAuth 2.0, the client requests access to resources controlled by the resource owner and hosted by the resource server and is issued a different set of credentials than those of the resource owner. Instead of using the resource owner's credentials to access protected resources, the client obtains an `access token`.

Access tokens are issued to third-party clients by an authorization server with the approval of the resource owner. The client uses the access token to access the protected resources hosted by the resource server.

The permissions represented by the access token, in OAuth 2.0 terms are known as `scopes`.

{% hint style="info" %}
Scope is a mechanism that defines the specific actions applications can be allowed to do or information that they can request on a user's behalf.
{% endhint %}

## Access token

An [access token](https://tools.ietf.org/html/rfc6749#section-1.4) is a credential that can be used by an application to access an API. It informs the API that the bearer of the token has been authorized to access the API and perform specific actions specified by the scope that has been granted.

The access token can be in any format, but two popular options include opaque strings and JSON Web Tokens (JWT). They should be transmitted to the API as a Bearer credential in an HTTP Authorization header.

## Refresh token

A [refresh token](https://tools.ietf.org/html/rfc6749#section-1.5) is a special kind of token that can be used to obtain a renewed access token (see details [here](https://tools.ietf.org/html/rfc6749#section-6)). It is useful for renewing expiring access tokens without forcing the user to log in again. Using the refresh token, you can request a new access token at any time until the refresh token is blacklisted.

## OAuth 2.0 roles

In any OAuth 2.0 flow we can identify the following roles:
- `Resource Owner` - an entity that can grant access to a protected resource; typically this is the end-user.
- `Resource Server` - the server hosting the protected resources; this is the API you want to access.
- `Client` - an application requesting access to a protected resource on behalf of the resource owner.
- `Authorization Server` - the server that authenticates the resource owner, and issues access tokens after getting proper authorization.

## Protocol flow

OAuth 2.0 has many "flavors" (called authorization grant types) that you can use. For now we will have a more generic look into the flow.

![](img/oauth2-generic-flow.png)

1. The application (client) asks for authorization from the resource owner in order to access the resources.
2. Provided that the resource owner authorizes this access, the application receives an authorization grant. This is a credential representing the resource owner's authorization.
3. The application requests an access token by authenticating with the authorization server and giving the authorization grant.
4. Provided that the application is successfully authenticated and the authorization grant is valid, the authorization server issues an access token and sends it to the application.
5. The application requests access to the protected resource by the resource server, and authenticates by presenting the access token.
6. Provided that the access token is valid, the resource server serves the application's request.

## Authorization grant types

The [OAuth 2.0 Authorization Framework specification](https://tools.ietf.org/html/rfc6749) defines four flows to get an access token. These flows are called grant types. Deciding which one is suited for your case depends mostly on the type of your application.
- [Authorization Code Grant](https://tools.ietf.org/html/rfc6749#section-4.1) - used by web applications executing on a server. This is also used by mobile apps, using the [Proof Key for Code Exchange (PKCE)](https://tools.ietf.org/html/rfc7636) technique.
- [Implicit Grant](https://tools.ietf.org/html/rfc6749#section-4.2) - used by javascript-centric applications (single-page applications) executing on the user's browser.
- [Resource Owner Password Credentials Grant](https://tools.ietf.org/html/rfc6749#section-4.3) - used by trusted applications.
- [Client Credentials Grant](https://tools.ietf.org/html/rfc6749#section-4.4) - used for machine-to-machine communication.

The specification also provides an extensibility mechanism for defining additional types.

## OAuth 2.0 endpoints

The OAuth 2.0 utilizes two endpoints:
- Authorization endpoint
- Token endpoint

### Authorization endpoint

The [authorization endpoint](https://tools.ietf.org/html/rfc6749#section-3.1) is used to interact with the resource owner and get the authorization to access the protected resource.

{% hint style="info" %}
The authorization endpoint is used by the authorization code and the implicit grants. 
{% endhint %}

To better understand this, imagine that you want to log in to a service using your Google account. First, the service redirects you to Google in order to authenticate (if you are not already logged in) and then you will get a consent screen, where you will be asked to authorize the service to access some of your data (protected resources), for example, your email address and your list of contacts.

The request parameters of the authorization endpoint are:
- `response_type` - tells the authorization server which grant to execute.
- `client_id` - the id of the application that asks for authorization.
- `redirect_uri` - holds a URL; a successful response from this endpoint results in a redirect to this URL.
- `scope` - a space-delimited list of permissions that the application requires.
- `state` - is a parameter that allows you to restore the previous state of your application; the `state` parameter preserves some state object set by the client in the authorization request and makes it available to the client in the response.

#### How response_type works

The `response_type` request parameter is used to inform the authorization server which grant type to use:
- `code` - value to use authorization code grant.
- `token` - value to use implicit grant.
- `password` - value to use resource owner password credentials grant.
- `client_credentials` - value to use client credentials grant.

#### How state works

Here is a sequence diagram of the full authorization code grant with a `state` parameter. The client implements CSRF protection by checking that the `state` exists in the user's session when he comes back to get the `access_token`. The `state` parameter in this design is a key to a session attribute in the authenticated user's session with the client application.

![](img/how-state-works.png)

### Token endpoint

The [token endpoint](https://tools.ietf.org/html/rfc6749#section-3.2) is used by the client to obtain an [access token](https://tools.ietf.org/html/rfc6749#section-1.4) by presenting its authorization grant or refresh token. The token endpoint requires client authentication as described [here](https://tools.ietf.org/html/rfc6749#section-3.2.1).

{% hint style="info" %}
The token endpoint is used with every authorization grant except for the implicit grant type (since an access token is issued directly).
{% endhint %}

## Authorization Code Grant

Because regular web apps are server-side apps where the source code is not publicly exposed, they can use the [authorization code grant](https://tools.ietf.org/html/rfc6749#section-4.1), which exchanges an authorization code for a token.

Additional parameters:
- `code` - is the authorization code received from the authorization server.
- `grant_type` - is a parameter that explains what the grant type is, and which token is going to be returned.
- `client_secret` - is a secret known only to the application and the authorization server.

### How it works

![](img/auth-code-flow.png)

1. The user clicks `Login` within the regular web application.
2. The application redirects the user to the login and authorization prompt (see also [additional information about the authorization request](https://tools.ietf.org/html/rfc6749#section-4.1.1)). The request may look like this:

    ```http
    https://auth-server.com/auth?
        response_type=code&
        client_id=application_client_id&
        redirect_uri=https://application-website.com/callback&
        scope=read:email&
        state=kYlr93jbdhyguIVF73moq
    ```

3. The user authenticates using one of the configured login options and may see a consent page listing the permissions the authorization server will give to the regular web application. The consent page and the request may look like this:

    ![](img/authorize-example.png)

    ```http
    POST /auth HTTP/1.1
    Host: auth-server.com
    Content-Type: application/json
    Cookie: session_id=Ja2upepCYz5IhSdSIUn1lyi2Ylir3afn

    {
        "client_id": "application_client_id",
        "redirect_uri": "https://application-website.com/callback",
        "state": "kYlr93jbdhyguIVF73moq",
        "response_type": "code",
        "scope": "read:email"
    }
    ```

4. Once accepted, the authorization server will send a request back to the `redirect_uri` with the `code` and `state` parameters (see also [additional information about the authorization response](https://tools.ietf.org/html/rfc6749#section-4.1.2)):

    ```http
    https://application-website.com/callback?
        code=a68YhewbiYl93TR89hdjYwqP0&
        state=kYlr93jbdhyguIVF73moq
    ```

5. The application using the received `code` and its own `client_id` and `client_secret` will make a request for `access_token` (see also [additional information about the access token request](https://tools.ietf.org/html/rfc6749#section-4.1.3)):

    ```http
    POST /oauth/access_token HTTP/1.1
    Host: auth-server.com
    Contetn-Type: application/json
    Content-Length: 157

    {
        "client_id": "application_client_id",
        "client_secret": "application_client_secret",
        "grant_type": "authorization_code",
        "code": "a68YhewbiYl93TR89hdjYwqP0"
    }
    ```

6. Finally, the flow is complete and the application will make an API call to the resource server with `access_token` to access the user's data (see also [additional information about the access token response](https://tools.ietf.org/html/rfc6749#section-4.1.4)).

## Implicit Grant

The [implicit grant](https://tools.ietf.org/html/rfc6749#section-4.2) is much simpler. Rather than first obtaining an authorization code and then exchanging it for an access token, the client application receives the access code immediately after the user gives their consent.

### How it works

1. The user clicks `Login` within the regular web application
2. The application redirects the user to the login and authorization prompt (see also [additional information about the authorization request](https://tools.ietf.org/html/rfc6749#section-4.2.1)). The request may look like this:

    ```http
    https://auth-server.com/auth?
        response_type=token&
        client_id=application_client_id&
        redirect_uri=https://application-website.com/callback&
        scope=read:email&
        state=kYlr93jbdhyguIVF73moq
    ```

3. The user authenticates and decides whether to consent to the requested permissions or not. This process is exactly the same as for the Authorization Code Grant.
4. Once accepted, the authorization server will redirect the user's to the `redirect_uri` specified in the authorization request. However, instead of sending a query parameter containing an authorization code, it will send the access token and other token-specific data as a URL fragment (see also [additional information about the access token response](https://tools.ietf.org/html/rfc6749#section-4.2.2)):

    ```http
    https://application-website.com/callback?
        access_token=z0y9x8w7v6u5&
        token_type=Bearer&
        expires_in=5000&
        scope=email:read&
        state=kYlr93jbdhyguIVF73moq
    ```

5. Once the client application has successfully extracted the `access_token` from the URL fragment, it can use it to access the user's data.

# Security issues in the client application

## Improper implementation of the Implicit Grant

In the Implicit Grant, the `access_token` is sent from the authorization service to the client application via the user's browser as a URL fragment. The client application then accesses the token using javascript. The trouble is, if the application wants to maintain the session after the user closes the page, it needs to store the current user data (normally a `user_id` and the `access_token`) somewhere.

To solve this problem, the client application will often submit this data to the server in a POST request and then assign the user a session cookie, effectively logging them in. This request is roughly equivalent to the form submission request that might be sent as part of a classic, password-based login. However, in this scenario, the server does not have any secrets or passwords to compare with the submitted data, which means that it is implicitly trusted.

As a result, this behavior can lead to a serious vulnerability if the client application does not properly check that the `access_token` matches the other data in the request. In this case, an attacker can simply change the parameters sent to the server to impersonate any user.

## Improper handling of state parameter

If the `state` parameter is:
- missing,
- a static value that never changes,
- present but not validated,

the OAuth 2.0 flow is likely to be vulnerable to CSRF.

To exploit this, go through the authorization process under your account and pause immediately after authorization. Then send this URL to the logged-in victim, who will add your account to own.

## Disclosure of secrets

Private OAuth 2.0 parameters may leak. For example, if the request in the Authorization Code Grant for `access_token` (step 5) is executed from the client, instead of the server.

## Open redirect

An open redirect can occur if the application redirects the user to `redirect_uri` in case of an error, for example, due to an incorrect scope value. In this case, an attacker crafts the following link:

```http
https://vulnerable-website.com/authorize?
    response_type=code&
    client_id=abcdef123456&
    scope=WRONG_SCOPE&
    redirect_uri=https://attacker-website.com
```

The application, when processing the request, redirects a victim to the attacker's host.

{% embed url="http://blog.intothesymmetry.com/2015/04/open-redirect-in-rfc6749-aka-oauth-20.html" %}

# Security issues in the authorization server

## Abusing API

If the `access_token` allows making requests to the API, it is worth checking whether it is possible to abuse this. Sometimes the granted rights to `access_token` allows you to release new tokens with increased privileges or get access to additional functionality available only with session token.

## Abusing accounts with unconfirmed email

The authorization server can allow to use accounts with unconfirmed email for granting access to protected resources. As a result, a third-party application will trust the received data.

{% embed url="https://gitlab.com/gitlab-org/gitlab/-/issues/37038" %}

## Assignment of accounts based on email address

Applications often allow multiple authentication methods: using a login with a password and using a third-party service such as github or google. There are several ways to attack this:
- If the application does not require email verification on account creation, try creating an account with the victim's email address. Sometimes, when the victim then tries to register or log-in using the third-party service, the application performs a search, sees that the email is already registered and associates the third-party service account with the account created by the attacker. In this case, the attacker will have access to the victim's account.
- If the third-party service does not require email verification on account creation, try creating an account with the victim's email address. The same issue as above could exist, but you would be attacking it from the other direction and getting access to the victim's account for an account takeover.

## Bypass redirect_uri validation via mass assignment

{% embed url="https://nvd.nist.gov/vuln/detail/CVE-2021-27582" %}

{% embed url="https://0xn3va.gitbook.io/cheat-sheets/framework/spring/mass-assignment" %}

## CSRF authorization server

The CSRF vulnerability in doorkeeper allowed attackers to silently install their applications on DigitalOcean to victims and transfer `access_token` to a controlled server. Details of the attack can be viewed [here](https://habr.com/ru/post/246025/) (in Russian) or [here](http://homakov.blogspot.com/2014/12/blatant-csrf-in-doorkeeper-most-popular.html).

## Host header poisoning

Poisoning the `Host` header can lead to account takeover not only during password recovery but also OAuth authentication. Sometimes you can affect `redirect_uri` by poisoning the `Host` header. As a result, when the victim exchanges the authorization code for access token, he / she will send a request with this token to your domain. Example of vulnerable request:

```http
GET /twitter/login HTTP/1.1
Host: attacker-website.com/vulnerable-website.com
```

References:
- [Write up: Account Takeover in Periscope TV](https://hackerone.com/reports/317476) 

## Misconfiguration acr or amr

The authorization server may accept `acr_values` or `amr_values` parameters and use them to process the authentication request. 

`amr_values` (or authentication method reference values) specifies authentication methods used in the authentication. For instance, values might indicate that both password and OTP authentication methods were used.     

`acr_values` (or requested authentication context class reference values) specifies a set of business rules that authentications are being requested to satisfy. These rules can often be satisfied by using a number of different specific authentication methods, either singly or in combination.

In other words, you can set authentication methods via these parameters. If it is configured incorrectly, it may lead to the possibility of bypassing authentication. For example, if a client sends `amr_values=pwd+otp`, you can try to bypass two-factor authentication by passing `amr_values=pwd`.

References:
- [RFC8176: Authentication Method Reference Values](https://datatracker.ietf.org/doc/html/rfc8176)
- [Bypassing 2FA using OpenID Misconfiguration](https://youst.in/posts/bypassing-2fa-using-openid-misconfiguration/)

## redirect_uri session poisoning

If the authorization request (step 3 in the authorization code flow) does not contain any parameters about the client being authorized, the authorization server may take them from the user's session. In this case the authorization request may look like this:

```http
POST /auth HTTP/1.1
Host: auth-server.com
Content-Type: application/json
Cookie: session_id=Ja2upepCYz5IhSdSIUn1lyi2Ylir3afn

{
    "scope": "read:email",
    "authorize": "Authorize"
}
```

An attack based on this behavior would look like this:

1. The victim visits a specially crafted page (just like a typical XSS/CSRF attack scenario).
2. The page redirects to the OAuth authorization page with a "trusted" `client_id`.
3. The page sends a hidden cross-domain request to the OAuth authorization page with an "untrusted" `client_id`, which poisons the session.
4. The victim approves the first page and, since the session contains the updated value, the victim will be redirected to the `redirect_uri` of the "untrusted" client.

In many real systems, third-party users can register their own clients, so this vulnerability may allow them to register an arbitrary `redirect_uri` and leak a token to it.

If the user have approved the client earlier, the server might just redirect the user without asking for confirmation. The OpenID Connect specification provides a `prompt=consent` parameter, which you can append to the URL of the authorization request to potentially bypass this problem. If the server follows OpenID Connection specification, it should ask the users for confirmation of their consent even if they have previously granted it. Without confirmation, the exploitation is harder but still possible, depending on the particular OAuth server implementation.

## Scope upgrade: Authorization Code Grant

With the Authorization Code Grant type, the user's data is requested and sent via secure server-to-server communication, which a third-party attacker is typically not able to manipulate directly. However, it may still be possible to achieve the same result by registering their own client application with the authorization service.

For example, let's say the attacker's malicious client application initially requested access to the user's email address using the `read:email` scope. After the user approves this request, the malicious client application receives an authorization code. As the attacker controls their client application, they can add another scope parameter to the code exchange request containing the additional profile scope:

```http
POST /oauth/access_token HTTP/1.1
Host: auth-server.com
Contetn-Type: application/json
Content-Length: 191

{
    "client_id": "application_client_id",
    "client_secret": "application_client_secret",
    "grant_type": "authorization_code",
    "code": "a68YhewbiYl93TR89hdjYwqP0",
    "scope": "read:email%20read:user"
}
```

If the server does not validate this against the scope from the initial authorization request, it will generate an `access_token` using the new scope and send this to the attacker's client application.

## Scope upgrade: Implicit Grant

For the Implicit Grant type, the `access_token` is sent via the browser, which means an attacker can steal tokens associated with innocent client applications and use them directly. Once they have stolen an `access_token`, they can send a normal browser-based requests to the authorization service's endpoints, manually adding a new scope parameter in the process.

## Scope upgrade: abusing re-release tokens

Applications often support token re-release functionality. There are several ways to attack this:
- Try to explicitly pass the scope in the token re-release request. Sometimes this allows you to change the scope for a new token.
- If the application can act as the OAuth 2.0 provider and you can manage the scope, try to create tokens, change the scope and re-release them. Sometimes the scope of a new token changes.

## Weak redirect_uri configuration

The `redirect_uri` is very important because sensitive data, such as the `code`, is appended to this URL after authorization. If the `redirect_uri` can be redirected to an attacker controlled server, this means the attacker can potentially takeover a victim's account by using the code themselves.

There are several weak configurations, depending on the logic handled by the server:
- Open redirect:

    ```http
    https://auth-server.com/auth?
        redirect_uri=https://attacker-website.com
    ```

- Path traversal or stealing tokens via a proxy page:

    ```http
    https://auth-server.com/auth?
        redirect_url=https://application-website.com/callback/../some/path
    ```

- Weak regex:

    ```http
    https://auth-server.com/auth?
        redirect_url=https://application-website.com.attacker-website.com
    ```

- Discrepancies between the parsing of the URI by the different components:

    ```http
    https://auth-server.com/auth?
        redirect_url=https://application-website.com%20%26@foo.attacker-website.com#@bar.attacker-website.com
    ```

- HTTP parameter pollution:

    ```http
    https://auth-server.com/auth?
        redirect_uri=https://application-website.com/callback&
        redirect_uri=https://attacker-website.com
    ```

- Accidentally permitted any redirect URI beginning with `localhost` in a production environment:

    ```http
    https://auth-server.com/auth?
        redirect_uri=https://localhost.attacker-website.com
    ```

- HTML Injection and stealing tokens via `Referer` header:

    ```html
    <img src="attacker-website.com">
    ```

- Dangerous javascript that handles query parameters and URL fragments.

References:
- [Report: Stealing SSO Login Tokens (snappublisher.snapchat.com)](https://hackerone.com/reports/265943)
- [Write up: OAuth and PostMessage](https://ninetyn1ne.github.io/2022-02-21-oauth-postmessage-misconfig/)

# References

- [OAuth 2.0 Authorization Framework](https://auth0.com/docs/protocols/oauth2)
- [Top X OAuth 2 Hacks (OAuth Implementation vulnerabilities)](https://github.com/0xn3va/cheat-sheets/blob/master/Web%20Application/OAuth/materials/20151215-Top_X_OAuth_2_Hacks-asanso.pdf)
- [Web Security Academy: OAuth 2.0 authentication vulnerabilities](https://portswigger.net/web-security/oauth)
- [PortSwigger Articles: Hidden OAuth attack vectors](https://portswigger.net/research/hidden-oauth-attack-vectors)
- [Write up: Bypassing GitHub's OAuth flow](https://blog.teddykatz.com/2019/11/05/github-oauth-bypass.html)
- [Report: Stealing users Bitbucket app tokens to retrieve sensitive data from connected Bitbucket accounts](https://gitlab.com/gitlab-org/gitlab-foss/-/issues/56500)
- [Report: Chained Bugs to Leak Victim's Uber's FB Oauth Token](https://hackerone.com/reports/202781)
- [Write up: Facebook OAuth Framework Vulnerability](https://www.amolbaikar.com/facebook-oauth-framework-vulnerability/)
- [Write up: How I hacked Github again](http://homakov.blogspot.com/2014/02/how-i-hacked-github-again.html)
