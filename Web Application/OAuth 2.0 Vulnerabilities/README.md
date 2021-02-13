# OAuth 2.0 Authorization Framework

[OAuth 2.0](https://tools.ietf.org/html/rfc6749) is a protocol that allows a user to grant a third-party web site or application access to the user's protected resources, without necessarily revealing their long-term credentials or even their identity.

OAuth 2.0 introduces an authorization layer and separates the role of the client from that of the resource owner. In OAuth 2.0, the client requests access to resources controlled by the resource owner and hosted by the resource server and is issued a different set of credentials than those of the resource owner. Instead of using the resource owner's credentials to access protected resources, the client obtains an `access token`.

> An access token is a credential that can be used by an application to access an API. It informs the API that the bearer of the token has been authorized to access the API and perform specific actions specified by the scope that has been granted.

> An access token can be in any format, but two popular options include opaque strings and JSON Web Tokens (JWT). They should be transmitted to the API as a Bearer credential in an HTTP Authorization header.
 
Access tokens are issued to third-party clients by an authorization server with the approval of the resource owner. The client uses the access token to access the protected resources hosted by the resource server.

The permissions represented by the access token, in OAuth 2.0 terms are known as `scopes`.

> Scope is a mechanism that defines the specific actions applications can be allowed to do or information that they can request on a user’s behalf.

When an application authenticates with Auth0, it specifies the scopes it wants. If those scopes are authorized by the user, then the access token will represent these authorized scopes. For example, a Contacts API may accept three different authorization levels specifying the corresponding scopes:
- Reading contacts (scope `read:contacts`),
- Creating contacts (scope `create:contacts`),
- Deleting contacts (scope `delete:contacts`).

For more information refer to [Scopes](https://auth0.com/docs/scopes/current).

## OAuth 2.0 roles

In any OAuth 2.0 flow we can identify the following roles:
- `Resource Owner` - the entity that can grant access to a protected resource. Typically this is the end-user.
- `Resource Server` - the server hosting the protected resources. This is the API you want to access.
- `Client` - the app requesting access to a protected resource on behalf of the Resource Owner.
- `Authorization Server` - the server that authenticates the Resource Owner, and issues Access Tokens after getting proper authorization.

## Protocol flow

OAuth 2.0 has many "flavors" (called authorization grant types) that you can use. For now we will have a more generic look into the flow.

![oauth2-generic-flow](img/oauth2-generic-flow.png)

1. The Application (Client) asks for authorization from the Resource Owner in order to access the resources.
2. Provided that the Resource Owner authorizes this access, the Application receives an Authorization Grant. This is a credential representing the Resource Owner's authorization.
3. The Application requests an Access Token by authenticating with the Authorization Server and giving the Authorization Grant.
4. Provided that the Application is successfully authenticated and the Authorization Grant is valid, the Authorization Server issues an Access Token and sends it to the Application.
5. The Application requests access to the protected resource by the Resource Server, and authenticates by presenting the Access Token.
6. Provided that the Access Token is valid, the Resource Server serves the Application's request.

## Authorization grant types

The [OAuth 2.0 Authorization Framework specification](https://tools.ietf.org/html/rfc6749) defines four flows to get an Access Token. These flows are called grant types. Deciding which one is suited for your case depends mostly on the type of your application.
- [Authorization Code](https://auth0.com/docs/flows/concepts/auth-code) - used by Web Apps executing on a server. This is also used by mobile apps, using the [Proof Key for Code Exchange (PKCE) technique](https://auth0.com/docs/flows/concepts/auth-code-pkce).
- [Implicit](https://auth0.com/docs/flows/concepts/implicit) - used by javascript-centric apps (Single-Page Applications) executing on the user's browser.
- [Resource Owner Password](https://auth0.com/docs/api-auth/grant/password) - used by trusted apps.
- [Client Credentials](https://auth0.com/docs/flows/concepts/client-credentials) - used for machine-to-machine communication.

The specification also provides an extensibility mechanism for defining additional types.

For details on how each grant type works and when it should be used refer to [API Authorization](https://auth0.com/docs/api-auth).

## OAuth 2.0 endpoints

OAuth 2.0 utilizes two endpoints:
- Authorization endpoint,
- Token endpoint.

### Authorization endpoint

The authorization endpoint is used to interact with the resource owner and get the authorization to access the protected resource. To better understand this, imagine that you want to log in to a service using your Google account. First, the service redirects you to Google in order to authenticate (if you are not already logged in) and then you will get a consent screen, where you will be asked to authorize the service to access some of your data (protected resources); for example, your email address and your list of contacts.

The request parameters of the authorization endpoint are:
- `response_type` - tells the authorization server which grant to execute,
- `client_id` - the id of the application that asks for authorization,
- `redirect_uri` - holds a URL; a successful response from this endpoint results in a redirect to this URL,
- `scope` - a space-delimited list of permissions that the application requires,
- `state` - is a parameter that allows you to restore the previous state of your application. The `state` parameter preserves some state object set by the client in the Authorization request and makes it available to the client in the response.

#### How response type works

This endpoint is used by the Authorization Code and the Implicit grant types. The authorization server needs to know which grant type the application wants to use, since it affects the kind of credential it will issue:
- For `Authorization Code grant` it will issue an authorization code (which can later be exchanged with an access token),
- For `Implicit grant` it will issue an access token.

An access token is an opaque string (or a JWT) that denotes who has authorized which permissions (scopes) to which application. It is meant to be exchanged with an access token at the `token endpoint`.

To inform the authorization server which grant type to use, the `response_type` request parameter is used as follows:
- For `Authorization Code grant`, use `response_type=code` to include an authorization code,
- For `Implicit grant`, use `response_type=token` to include an access token. An alternative is to set `response_type=id_token token` to include both an access token and an [ID token](https://auth0.com/docs/tokens/concepts/id-tokens).

#### How state works

Here is a sequence diagram of the full Authorization Code Flow with a `state` parameter. The Client implements CSRF protection by checking that the `state` exists in the user's session when he comes back to get the `access_token`. The `state` parameter in this design is a key to a session attribute in the authenticated user's session with the Client application.

![how-state-works](img/how-state-works.png)

### Token endpoint

The token endpoint is used by the application in order to get an `access token` or a `refresh token`. It is used by all flows, except for the Implicit Flow (since an access token is issued directly).

> Refresh token is a special kind of token that can be used to obtain a renewed access token. It is useful for renewing expiring access tokens without forcing the user to log in again. Using the refresh token, you can request a new access token at any time until the refresh token is blacklisted.

- In the `Authorization Code Flow`, the application exchanges the authorization code it got from the Authorization endpoint for an access token.
- In the `Client Credentials Flow` and `Resource Owner Password Credentials Flow`, the application authenticates using a set of credentials and then gets an access token.

## Authorization Code flow

Because regular web apps are server-side apps where the source code is not publicly exposed, they can use the [Authorization Code Flow](https://tools.ietf.org/html/rfc6749#section-4.1), which exchanges an authorization code for a token.

Additional parameters:
- `code` - is the authorization code received from the authorization server,
- `grant_type` - is a parameter that explains what the grant type is, and which token is going to be returned,
- `client_secret` - is a secret known only to the application and the authorization server.

### How it works

![auth-code-flow](img/auth-code-flow.png)

1. The user clicks `Login` within the regular web application.
2. The application redirects the user to the login and authorization prompt. The request may look like this:

    ```http
    https://auth-server.com/auth?
        response_type=code&
        client_id=application_client_id&
        redirect_uri=https://application-website.com/callback&
        scope=read:email&
        state=kYlr93jbdhyguIVF73moq
    ```

3. The user authenticates using one of the configured login options and may see a consent page listing the permissions the authorization server will give to the regular web application. The consent page may look like this:

    ![authorize-example](img/authorize-example.png)

4. Once accepted, the authorization server will send a request back to the `redirect_uri` with the `code` and `state` parameters:

    ```http
    https://application-website.com/callback?code=a68YhewbiYl93TR89hdjYwqP0&state=kYlr93jbdhyguIVF73moq
    ```

5. The application using the received `code` and its own `client_id` and `client_secret` will make a request for `access_token`:

    ```http
    POST /oauth/access_token
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

6. Finally, the flow is complete and the application will make an API call to the resource server with `access_token` to access the user's data.

## Implicit flow

The [Implicit flow](https://tools.ietf.org/html/rfc6749#section-4.2) is much simpler. Rather than first obtaining an authorization code and then exchanging it for an access token, the client application receives the access code immediately after the user gives their consent.

### How it works

1. The user clicks `Login` within the regular web application
2. The application redirects the user to the login and authorization prompt. The request may look like this:

    ```http
    https://auth-server.com/auth?
        response_type=token&
        client_id=application_client_id&
        redirect_uri=https://application-website.com/callback&
        scope=read:email&
        state=kYlr93jbdhyguIVF73moq
    ```

3. The user authenticates and decides whether to consent to the requested permissions or not. This process is exactly the same as for the authorization code flow.
4. Once accepted, the authorization server will redirect the user's to the `redirect_uri` specified in the authorization request. However, instead of sending a query parameter containing an authorization code, it will send the access token and other token-specific data as a URL fragment:

    ```http
    https://application-website.com/callback#
        access_token=z0y9x8w7v6u5&
        token_type=Bearer&
        expires_in=5000&
        scope=email:read&
        state=kYlr93jbdhyguIVF73moq
    ```

5. Once the client application has successfully extracted the `access_token` from the URL fragment, it can use it to access the user's data.

# Security issues in the client application

## Improper implementation of the Implicit flow

In the Implicit flow, the `access_token` is sent from the authorization service to the client application via the user's browser as a URL fragment. The client application then accesses the token using javascript. The trouble is, if the application wants to maintain the session after the user closes the page, it needs to store the current user data (normally a `user_id` and the `access_token`) somewhere.

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

Sometimes private OAuth 2.0 parameters may leak. For example, if the request for `access_token` (step 5) is executed from the client, instead of the server.

## Open redirect

An open redirect can occur if the application redirects the user to `redirect_uri` in case of an error, for example, due to an incorrect scope value, see the [link](http://blog.intothesymmetry.com/2015/04/open-redirect-in-rfc6749-aka-oauth-20.html).

In this case, the attacker crafts a following link:

```http
https://vulnerable-website.com/authorize?response_type=code&client_id=abcdef123456&scope=WRONG_SCOPE&redirect_uri=https://attacker-website.com
```

and the application, when processing the request, redirects the victim to the attacker's host.

# Security issues in the authorization service

## Weak redirect_uri configuration

The `redirect_uri` is very important because sensitive data, such as the `code`, is appended to this URL after authorization. If the `redirect_uri` can be redirected to an attacker controlled server, this means the attacker can potentially takeover a victim's account by using the code themselves.

There are several weak configurations, depending on the logic handled by the server:
- Open redirect:

    ```http
    https://auth-server.com/auth?redirect_uri=https://attacker-website.com
    ```

- Path traversal or stealing tokens via a proxy page:

    ```http
    https://auth-server.com/auth?redirect_url=https://application-website.com/callback/../some/path
    ```

- Weak regex:

    ```http
    https://auth-server.com/auth?redirect_url=https://application-website.com.attacker-website.com
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
    https://auth-server.com/auth?redirect_uri=https://localhost.attacker-website.com
    ```

- HTML Injection and stealing tokens via `Referer` header:

    ```html
    <img src="attacker-website.com">
    ```

- Dangerous javascript that handles query parameters and URL fragments

## Scope upgrade: Authorization Code flow

With the Authorization Code grant type, the user's data is requested and sent via secure server-to-server communication, which a third-party attacker is typically not able to manipulate directly. However, it may still be possible to achieve the same result by registering their own client application with the authorization service.

For example, let's say the attacker's malicious client application initially requested access to the user's email address using the `read:email` scope. After the user approves this request, the malicious client application receives an authorization code. As the attacker controls their client application, they can add another scope parameter to the code exchange request containing the additional profile scope:

```http
POST /oauth/access_token
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

If the server does not validate this against the scope from the initial authorization request, it will sometimes generate an `access_token` using the new scope and send this to the attacker's client application.

## Scope upgrade: Implicit flow

For the Implicit grant type, the `access_token` is sent via the browser, which means an attacker can steal tokens associated with innocent client applications and use them directly. Once they have stolen an `access_token`, they can send a normal browser-based requests to the authorization service's endpoints, manually adding a new scope parameter in the process.

## Scope upgrade: abusing re-release tokens

Applications often support token re-release functionality. There are several ways to attack this:
- Try to explicitly pass the scope in the token re-release request. Sometimes this allows you to change the scope for a new token.
- If the application can act as an OAuth 2.0 provider and you can manage the scope, try to create tokens, change the scope and re-release them. Sometimes the scope of a new token changes.

## Assignment of accounts based on email address

Applications often allow multiple authentication methods: using a login with a password and using a third-party service such as github or google. There are several ways to attack this:
- If the application does not require email verification on account creation, try creating an account with the victim's email address. Sometimes, when the victim then tries to register or log-in using the third-party service, the application performs a search, sees that the email is already registered and associates the third-party service account with the account created by the attacker. In this case, the attacker will have access to the victim's account.
- If the third-party service does not require email verification on account creation, try creating an account with the victim's email address. The same issue as above could exist, but you would be attacking it from the other direction and getting access to the victim's account for an account takeover.

## Abusing API

If the `access_token` allows making requests to the API, it is worth checking whether it is possible to abuse this. Sometimes the granted rights to `access_token` allows you to release new tokens with increased privileges or get access to additional functionality available only with session token.

## CSRF authorization server

The CSRF vulnerability in doorkeeper allowed attackers to silently install their applications on DigitalOcean to victims and transfer `access_token` to a controlled server. Details of the attack can be viewed [here](https://habr.com/ru/post/246025/) (in Russian) or [here](http://homakov.blogspot.com/2014/12/blatant-csrf-in-doorkeeper-most-popular.html).

# References

- [OAuth 2.0 Authorization Framework](https://auth0.com/docs/protocols/oauth2)
- [Top X OAuth 2 Hacks (OAuth Implementation vulnerabilities)](https://github.com/0xn3va/cheat-sheets/blob/master/Web%20Application/OAuth/materials/20151215-Top_X_OAuth_2_Hacks-asanso.pdf)
- [Web Security Academy: OAuth 2.0 authentication vulnerabilities](https://portswigger.net/web-security/oauth)
- [Write up: Bypassing GitHub's OAuth flow](https://blog.teddykatz.com/2019/11/05/github-oauth-bypass.html)
- [Report: Stealing users Bitbucket app tokens to retrieve sensitive data from connected Bitbucket accounts](https://gitlab.com/gitlab-org/gitlab-foss/-/issues/56500)
- [Report: Chained Bugs to Leak Victim's Uber's FB Oauth Token](https://hackerone.com/reports/202781)
- [Write up: Facebook OAuth Framework Vulnerability](https://www.amolbaikar.com/facebook-oauth-framework-vulnerability/)
- [Write up: How I hacked Github again](http://homakov.blogspot.com/2014/02/how-i-hacked-github-again.html)
