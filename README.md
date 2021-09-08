A Python OAuth 2.x client, able to obtain, refresh and revoke tokens
from any OAuth2.x/OIDC compliant Authorization Server.

It can act as an OAuth 2.0/2.1 client, to automatically get and renew
access tokens, based on the Client Credentials, Authorization Code,
Refresh token, Device Authorization, or CIBA grants.

It comes with a [requests] add-on to handle OAuth 2.0 Bearer
Token based authorization when accessing APIs.

It also supports OpenID Connect, PKCE, Client Assertions, Token
Revocation, Exchange, and Introspection, Backchannel Authentication
requests, as well as using custom params to any endpoint, and other
important features that are often overlooked in other client libraries.

And it also includes a wrapper around [requests.Session](https://docs.python-requests.org/en/master/api/#request-sessions)
that makes it super easy to use REST-style APIs, with or without OAuth 2.0.

# Installation

As easy as:

    pip install requests_oauth2client

# Usage

Import it like this:

    from requests_oauth2client import *


## Calling APIs with an access token

If you already managed to obtain an access token, you can simply use the
[BearerAuth] Auth Handler for `requests`:

    token = "an_access_token"
    resp = requests.get("https://my.protected.api/endpoint", auth=BearerAuth(token))

This authentication handler will add a properly formatted `Authorization`
header in the request, with your access token according to RFC6750.

## Using an OAuth2Client

[OAuth2Client] offers several methods that implement the
communication to the various endpoints that are standardised by OAuth
2.0 and its extensions. Those endpoints include the Token Endpoint, the Revocation, Introspection, UserInfo, BackChannel Authentication and Device Authorization Endpoints.

To initialize an [OAuth2Client], you only need a Token
Endpoint URI, and the credentials for your application, which are often a `client_id` and a `client_secret`:

    oauth2client = OAuth2Client("https://myas.local/token_endpoint", ("client_id", "client_secret"))

The Token Endpoint is the only Endpoint that is mandatory to obtain
tokens. Credentials are used to authenticate the client everytime it
sends a request to its Authorization Server. Usually, those are a static
Client ID and Secret, which are the direct equivalent of a username and
a password, but meant for an application instead of for a human user. The
default authentication method used by OAuth2Client is *Client Secret
Post*, but other standardised methods such as *Client Secret Basic*,
*Client Secret JWT* or *Private Key JWT* are supported as well. See below.

## Obtaining tokens

[OAuth2Client] has methods to send requests to the Token
Endpoint using the different standardised (and/or custom) grants. Since
the token endpoint and authentication method are already declared for
the client, the only required parameters are those that will be sent in
the request to the Token Endpoint.

Those methods directly return a [BearerToken] if the request
is successful, or raise an exception if it fails.
[BearerToken] will manage the token expiration, will contain
the eventual refresh token that matches the access token, and will keep
track of other associated metadata as well. You can create such a
[BearerToken] yourself if you need:

    bearer_token = BearerToken(access_token="an_access_token", expires_in=60)
    print(bearer_token)
    > {'access_token': 'an_access_token', 'expires_in': 55, 'token_type': 'Bearer'}
    print(bearer_token.expires_at)
    > datetime.datetime(2021, 8, 20, 9, 56, 59, 498793)

Note that the `expires_in` indicator here is not static. It keeps
track of the token lifetime and is calculated as the time flies. You can
check if a token is expired with
[bearer_token.is_expired()](https://guillp.github.io/requests_oauth2client/api/#requests_oauth2client.tokens.BearerToken.is_expired).

You can use a [BearerToken] instance everywhere you can
supply an access_token as string.

### Using OAuth2Client as a requests Auth Handler

While using [OAuth2Client] directly is great for testing or debugging
OAuth2.0 flows, it is not a viable option for actual applications where
tokens must be obtained, used during their lifetime then obtained again
or refreshed once they are expired. `requests_oauth2client` contains
several [requests] compatible Auth Handler (subclasses of
[requests.auth.AuthBase](https://docs.python-requests.org/en/master/user/advanced/#custom-authentication), that will take care of obtaining
tokens when required, then will cache those tokens until they are
expired, and will obtain new ones (or refresh them, when possible), once
the initial token is expired. Those are best used with a
[requests.Session], or an [ApiClient] which is a
Session Subclass with a few enhancements as described below.

### Client Credentials grant

To send a request using the Client Credentials grant, use the aptly
named [.client_credentials()](https://guillp.github.io/requests_oauth2client/api/#requests_oauth2client.client.OAuth2Client.client_credentials) method:

    token = oauth2client.client_credentials(
        scope="myscope",
        resource="https://myapi.local"
        # you may pass additional kw params such as audience, or whatever your AS needs
    )

#### As Auth Handler

You can use the [OAuth2ClientCredentials](https://guillp.github.io/requests_oauth2client/api/#requests_oauth2client.auth.OAuth2ClientCredentialsAuth) auth handler. It
takes an [OAuth2Client] as parameter, and the additional kwargs to pass to
the token endpoint:

    api_client = ApiClient(
        'https://myapi.local/resource',
        auth=OAuth2ClientCredentials(oauth2client, scope='myscope', resource="https://myapi.local")
    )

    resp = api_client.get() # when you send your first request to the API, it will fetch an access token first.

### Authorization Code Grant

Obtaining tokens with the Authorization code grant is made in 3 steps:

1. your application must open specific url called the *Authentication
    Request* in a browser.

2. your application must obtain and validate the *Authorization
Response*, which is a redirection back to your application that contains
an *Authorization Code* as parameter.

3. your application must then exchange this Authorization Code for an
*Access Token*, with a request to the Token Endpoint.

[OAuth2Client] doesn't implement anything that is related
to the Authorization Request or Response. It is only able to exchange
the Authorization Code for a Token in step 3. But
`requests_oauth2client` has other classes to help you with
steps 1 and 2, as described below:

#### Generating Authorization Requests

You can generate valid authorization requests with the
[AuthorizationRequest](https://guillp.github.io/requests_oauth2client/api/#requests_oauth2client.authorization_request.AuthorizationRequest) class:

    auth_request = AuthorizationRequest(
        authorization_endpoint,
        client_id,
        redirect_uri=redirect_uri,
        scope=scope,
        resource=resource, # not mandatory
    ) # add any other param that needs to be sent to your AS
    print(auth_request) # redirect the user to that URL to get a code

This request will look like this (with line breaks for display purposes only):

    https://myas.local/authorize
    ?client_id=my_client_id
    &redirect_uri=http%3A%2F%2Flocalhost%2Fcallback
    &response_type=code
    &state=kHWL4VwcbUbtPR4mtht6yMAGG_S-ZcBh5RxI_IGDmJc
    &nonce=mSGOS1M3LYU9ncTvvutoqUR4n1EtmaC_sQ3db4dyMAc
    &scope=openid+email+profile
    &code_challenge=Dk11ttaDb_Hyq1dObMqQcTIlfYYRVblFMC9lFM3UWW8
    &code_challenge_method=S256
    &resource=https%3A%2F%2Fmy.resource.local%2Fapi

[AuthorizationRequest] supports PKCE and uses it by default. You can avoid
it by passing `code_challenge_method=None` to
[AuthenticationRequest]. You can obtain the generated
code_verifier from `auth_request.code_verifier`.

Redirecting or otherwise sending the user to this url is your
application responsibility, as well as obtaining the Authorization
Response url.

#### Validating the Authorization Response

Once the user is successfully authenticated and authorized, the AS will
respond with a redirection to your redirect_uri. That is the
*Authorization Response*. It contains several parameters that must be
retrieved by your client. The authorization code is one of those
parameters, but you must also validate that the *state* matches your
request. You can do this with:

    params = input("Please enter the full url and/or params obtained on the redirect_uri: ")
    code = auth_request.validate_callback(params)

#### Exchanging code for tokens

To exchange a code for Access and/or ID tokens, use the
[OAuth2Client.authorization_code()](https://guillp.github.io/requests_oauth2client/api/#requests_oauth2client.client.OAuth2Client.authorization_code) method:

    token = oauth2client.authorization_code(
        code=code,
        code_verifier=auth_request.code_verifier,
        redirect_uri=redirect_uri) # redirect_uri is not always mandatory, but some AS still requires it

#### As Auth Handler

The [OAuth2AuthorizationCodeAuth](https://guillp.github.io/requests_oauth2client/api/#requests_oauth2client.auth.OAuth2AuthorizationCodeAuth) handler takes an
[OAuth2Client] and an authorization code as parameter, plus whatever
additional keyword parameters are required by your Authorization Server:

    api_client = ApiClient(
        "https://your.protected.api/endpoint",
        auth=OAuth2AuthorizationCodeAuth(
            client, code,
            code_verifier=auth_request.code_verifier, redirect_uri=redirect_uri)

    resp = api_client.post(data={...}) # first call will exchange the code for an initial access/refresh tokens

[OAuth2AuthorizationCodeAuth](https://guillp.github.io/requests_oauth2client/api/#requests_oauth2client.auth.OAuth2AuthorizationCodeAuth) will take care of refreshing
the token automatically once it is expired, using the refresh token, if
available.

### Device Authorization Grant

Helpers for the Device Authorization Grant are also included. To get
device and user codes:

    client = OAuth2Client(
        token_endpoint="https://myas.local/token",
        device_authorization_endpoint="https://myas.local/device",
        auth=(client_id, client_secret),
    )

    da_resp = client.authorize_device()

`da_resp` contains the Device Code, User Code, Verification
URI and other info returned by the AS:

    da_resp.device_code
    da_resp.user_code
    da_resp.verification_uri
    da_resp.verification_uri_complete
    da_resp.expires_at # just like for BearerToken, expiration is tracked by requests_oauth2client
    da_resp.interval

Send/show the Verification Uri and User Code to the user. He must use a
browser to visit that url, authenticate and input the User Code. You can
then request the Token endpoint to check if the user successfully
authorized you using an \`OAuth2Client\`:

    token = client.device_code(da_resp.device_code)

This will raise an exception, either [AuthorizationPending](https://guillp.github.io/requests_oauth2client/api/#requests_oauth2client.exceptions.AuthorizationPending),
[SlowDown](https://guillp.github.io/requests_oauth2client/api/#requests_oauth2client.exceptions.SlowDown), [ExpiredToken](https://guillp.github.io/requests_oauth2client/api/#requests_oauth2client.exceptions.ExpiredToken), or
[AccessDenied](https://guillp.github.io/requests_oauth2client/api/#requests_oauth2client.exceptions.AccessDenied) if the user did not yet finish authorizing
your device, if you should increase your pooling period, or if the
device code is no longer valid, or the user finally denied your access,
respectively. Other exceptions may be raised depending on the error code
that the AS responds with. If the user did finish authorizing
successfully, `token` will contain your access token.

To make pooling easier, you can use a
[DeviceAuthorizationPoolingJob](https://guillp.github.io/requests_oauth2client/api/#requests_oauth2client.device_authorization.DeviceAuthorizationPoolingJob) like this:

    pool_job = DeviceAuthorizationPoolingJob(
        client,
        device_auth_resp.device_code,
        interval=device_auth_resp.interval
    )

    resp = None
    while resp is None:
        resp = pool_job()

    assert isinstance(resp, BearerToken)

[DeviceAuthorizationPoolingJob](https://guillp.github.io/requests_oauth2client/api/#requests_oauth2client.device_authorization.DeviceAuthorizationPoolingJob) will automatically obey the
pooling period. Everytime you call pool_job(), it will wait the
appropriate number of seconds as indicated by the AS, and will apply
slow_down requests.

#### As Auth Handler

Use [OAuth2DeviceCodeAuth](https://guillp.github.io/requests_oauth2client/api/#requests_oauth2client.auth.OAuth2DeviceCodeAuth) as auth handler to exchange a
device code for an access token:

    api_client = ApiClient(
        "https://your.protected.api/endpoint",
        auth=OAuth2DeviceCodeAuth(
            client, device_auth_resp.device_code,
            interval=device_auth_resp.interval, expires_in=device_auth_resp.expires_in
        )

    resp = api_client.post(data={...}) # first call will hang until the user authorizes your app and the token endpoint returns a token.

## Client-Initiated Backchannel Authentication (CIBA)

To initiate a BackChannel Authentication against the dedicated endpoint:

    client = OAuth2Client(
        token_endpoint="https://myas.local/token",
        backchannel_authentication_endpoint="https://myas.local/backchannel_authorize",
        auth=(client_id, client_secret)
    )

    ba_resp = client.backchannel_authentication_request(
        scope="openid email profile",
        login_hint="user@example.net",
    )

`ba_resp` will contain the response attributes as returned
by the AS, including an `auth_req_id`:

    ba_resp.auth_req_id
    ba_resp.expires_in # decreases as times fly
    ba_resp.expires_at # a datetime to keep track of the expiration date, based on the "expires_in" returned by the AS
    ba_resp.interval # the pooling interval indicated by the AS
    ba_resp.custom # if the AS respond with additional attributes, they are also accessible

To pool the Token Endpoint until the end-user successfully
authenticates:

    pool_job = BackChannelAuthenticationPoolingJob(
        client=client,
        auth_req_id=ba_resp.auth_req_id,
        interval=bca_resp.interval,
    )

    resp = None
    while resp is None:
        resp = pool_job()

    assert isinstance(resp, BearerToken)

## Supported Client Authentication Methods

`requests_oauth2client` supports multiple client
authentication methods, as defined in multiple OAuth2.x standards. You
select the appropriate method to use when initializing your
[OAuth2Client], with the `auth` parameter. Once initialized, a
client will automatically use the configured authentication method every
time it sends a requested to an endpoint that requires client
authentication. You don't have anything else to do afterwards.

- **client_secret_basic**: client_id and client_secret are included in
    clear-text in the Authorization header. To use it, just pass a
    [ClientSecretBasic(client_id, client_secret)](https://guillp.github.io/requests_oauth2client/api/#requests_oauth2client.client_authentication.ClientSecretBasic)} as `auth` parameter:

        client = OAuth2Client(token_endpoint, auth=ClientSecretBasic(client_id, client_secret))

- **client_secret_post**: client_id and client_secret are included as
    part of the body form data. To use it, pass a
    [ClientSecretPost(client_id, client_secret)](https://guillp.github.io/requests_oauth2client/api/#requests_oauth2client.client_authentication.ClientSecretPost) as `auth`
    parameter. This also what is being used as default when you pass a
    tuple `(client_id, client_secret)` as `auth`:

        client = OAuth2Client(token_endpoint, auth=ClientSecretPost(client_id, client_secret))
        # or
        client = OAuth2Client(token_endpoint, auth=(client_id, client_secret))

- **client_secret_jwt**: client generates an ephemeral JWT assertion
    including information about itself (client_id), the AS (url of the
    endpoint), and expiration date. To use it, pass a
    [ClientSecretJWT(client_id, client_secret)](https://guillp.github.io/requests_oauth2client/api/#requests_oauth2client.client_authentication.ClientSecretJWT) as `auth`
    parameter. Assertion generation is entirely automatic, you don't
    have anything to do:

        client = OAuth2Client(token_endpoint, auth=ClientSecretJWT(client_id, client_secret))

- **private_key_jwt**: client uses a JWT assertion like
    _client_secret_jwt_, but it is signed with an _asymmetric_ key. To use
    it, you need a private signing key, in a `dict` that
    matches the JWK format. The matching public key must be registered
    for your client on AS side. Once you have that, using this auth
    method is as simple with the [PrivateKeyJWT(client_id, private_jwk)](https://guillp.github.io/requests_oauth2client/api/#requests_oauth2client.client_authentication.PrivateKeyJWT) auth
    handler:

        private_jwk = {
            "kid": "mykid",
            "kty": "RSA",
            "e": "AQAB", "n": "...", "d": "...", "p": "...",
            "q": "...", "dp": "...", "dq": "...", "qi": "...",
        }

        client = OAuth2Client(
            "https://myas.local/token",
             auth=PrivateKeyJWT(client_id, private_jwk)
        )

- **none**: client only presents its client_id in body form data to
    the AS, without any authentication credentials. Use
    [PublicApp(client_id)](https://guillp.github.io/requests_oauth2client/api/#requests_oauth2client.client_authentication.PublicApp):

        client = OAuth2Client(token_endpoint, auth=PublicApp(client_id, client_secret))

## Token Exchange

To send a token exchange request, use the
[OAuth2Client.token_exchange()](https://guillp.github.io/requests_oauth2client/api/#requests_oauth2client.client.OAuth2Client.token_exchange) method:

    client = OAuth2Client(token_endpoint, auth=...)
    token = client.token_exchange(
        subject_token='your_token_value',
        subject_token_type="urn:ietf:params:oauth:token-type:access_token"
    )

As with the other grant-type specific methods, you may specify
additional keyword parameters, that will be passed to the token
endpoint, including any standardised attribute like
`actor_token` or `actor_token_type`, or any
custom parameter. There are short names for token types, that will be
automatically translated to standardised types:

    token = client.token_exchange(
        subject_token='your_token_value',
        subject_token_type="access_token", # will be automatically replaced by "urn:ietf:params:oauth:token-type:access_token"
        actor_token='your_actor_token',
        actor_token_type='id_token', # will be automatically replaced by "urn:ietf:params:oauth:token-type:id_token"
    )

Or to make it even easier, types can be guessed based on the supplied
subject or actor token:

    token = client.token_exchange(
        subject_token=BearerToken('your_token_value'),  # subject_token_type will be "urn:ietf:params:oauth:token-type:access_token"
        actor_token=IdToken('your_actor_token'), # actor_token_type will be "urn:ietf:params:oauth:token-type:id_token"
    )

## Token Revocation

[OAuth2Client] can send revocation requests to a Revocation
Endpoint. You need to provide a Revocation Endpoint URI when creating
the \`OAuth2Client\`:

    oauth2client = OAuth2Client(
        token_endpoint,
        revocation_endpoint=revocation_endpoint,
        auth=ClientSecretJWT("client_id", "client_secret"))

The [OAuth2Client.revoke_token()](https://guillp.github.io/requests_oauth2client/api/#requests_oauth2client.client.OAuth2Client.revoke_token) method and its specialized aliases
[.revoke_access_token()](https://guillp.github.io/requests_oauth2client/api/#requests_oauth2client.client.OAuth2Client.revoke_access_token) and
[.revoke_refresh_token()](https://guillp.github.io/requests_oauth2client/api/#requests_oauth2client.client.OAuth2Client.revoke_refresh_token) are then available:

    oauth2client.revoke_token("mytoken", token_type_hint="access_token")
    oauth2client.revoke_access_token("mytoken") # will automatically add token_type_hint=access_token
    oauth2client.revoke_refresh_token("mytoken") # will automatically add token_type_hint=refresh_token

Because Revocation Endpoints usually don't return meaningful responses,
those methods return a boolean. This boolean indicates that a request
was successfully sent and no error was returned. If the Authorization
Server actually returns a standardised error, an exception will be
raised instead.

## Token Introspection

[OAuth2Client] can send requests to a Token Introspection
Endpoint. You need to provide an Introspection Endpoint URI when
creating the `OAuth2Client`:

    oauth2client = OAuth2Client(
       token_endpoint,
       introspection_endpoint=introspection_endpoint,
       auth=ClientSecretJWT("client_id", "client_secret"))

The [OAuth2Client.introspect_token()](https://guillp.github.io/requests_oauth2client/api/#requests_oauth2client.client.OAuth2Client.instrospect_token()) method is then available:

    resp = oauth2client.introspect_token("mytoken", token_type_hint="access_token")

It returns whatever data is returned by the introspection endpoint (if
it is a JSON, its content is returned decoded).

## UserInfo Requests

[OAuth2Client] can send requests to an UserInfo Endpoint.
You need to provide an UserInfo Endpoint URI when creating the
`OAuth2Client`:

    oauth2client = OAuth2Client(
       token_endpoint,
       userinfo_endpoint=userinfo_endpoint,
       auth=ClientSecretJWT("client_id", "client_secret"))

The [OAuth2Client.userinfo()](https://guillp.github.io/requests_oauth2client/api/#requests_oauth2client.client.OAuth2Client.userinfo)) method is then available:

    resp = oauth2client.userinfo("mytoken")

It returns whatever data is returned by the userinfo endpoint (if it is
a JSON, its content is returned decoded).

## Initializing an OAuth2Client from a discovery document

You can initialize an [OAuth2Client] with the endpoint URIs mentioned in
a standardised discovery document with the [OAuth2Client.from_discovery_endpoint()](https://guillp.github.io/requests_oauth2client/api/#requests_oauth2client.client.OAuth2Client.from_discovery_document) class method:

    oauth2client = OAuth2Client.from_discovery_endpoint("https://myas.local/.well-known/openid-configuration")

This will fetch the document from the specified URI, then will decode it
and initialize an [OAuth2Client] pointing to the appropriate endpoint
URIs.

## Specialized API Client

Using APIs usually involves multiple endpoints under the same root url,
with a common authentication method. To make it easier,
`requests_oauth2client` includes a specialized
[requests.Session] subclass called ApiClient, which takes a
root url as parameter on initialization. You can then send requests to
different endpoints by passing their relative path instead of the full
url. [ApiClient] also accepts an `auth` parameter with an
AuthHandler. You can pass any of the OAuth2 Auth Handler from this
module, or any [requests]-compatible
[Authentication Handler](https://docs.python-requests.org/en/master/user/advanced/#custom-authentication). Which makes it very easy to call APIs that
are protected with an OAuth2 Client Credentials Grant:

    oauth2client = OAuth2Client("https://myas.local/token", (client_id, client_secret))
    api = ApiClient("https://myapi.local/root", auth=OAuth2ClientCredentialsAuth(oauth2client))
    resp = api.get("/resource/foo") # will actually send a GET to https://myapi.local/root/resource/foo

Note that [ApiClient] will never send requests "outside"
its configured root url, unless you specifically give it full url at
request time. The leading `/` in `/resource` above is
optional. A leading `/` will not "reset" the url path to root, which
means that you can also write the relative path without the `/` and it
will automatically be included:

    api.get("resource/foo") # will actually send a GET to https://myapi.local/root/resource/foo

You may also pass the path as an iterable of strings (or string-able
objects), in which case they will be joined with a `/` and appended to the
url path:

    api.get(["resource", "foo"]) # will actually send a GET to https://myapi.local/root/resource/foo
    api.get(["users", 1234, "details"]) # will actually send a GET to https://myapi.local/root/users/1234/details

[ApiClient] will, by default, raise exceptions whenever a
request returns an error status. You can disable that by passing
`raise_for_status=False` when initializing your
[ApiClient]:

    api = ApiClient(
        "http://httpstat.us",
         raise_for_status=False # this defaults to True
    )
    resp = api.get("500") # without raise_for_status=False, this would raise a requests.exceptions.HTTPError

You may override this at request time:

    resp = api.get("500", raise_for_status=True) # raise_for_status at request-time overrides raise_for_status defined at init-time

## Vendor-Specific clients

`requests_oauth2client` being flexible enough to handle most
use cases, you should be able to use any AS by any vendor as long as it
supports OAuth 2.0.

You can however create a subclass of [OAuth2Client] or [ApiClient] to make it easier to
use with specific Authorization Servers or APIs.
The sub-module `requests_oauth2client.vendor_specific` includes such
classes for Auth0:

    from requests_oauth2client.vendor_specific import Auth0Client

    a0client = Auth0Client("mytenant.eu", (client_id, client_secret))
    # this will automatically initialize the token endpoint to https://mytenant.eu.auth0.com/oauth/token
    # so you can use it directly
    token = a0client.client_credentials(audience="audience")

    # this is a wrapper around Auth0 Management API
    a0mgmt = Auth0ManagementApiClient("mytenant.eu", (client_id, client_secret))
    myusers = a0mgmt.get("users")


[requests]: https://docs.python-requests.org/en/master/
[OAuth2Client]: https://guillp.github.io/requests_oauth2client/api/#requests_oauth2client.client.OAuth2Client
[BearerAuth]: https://guillp.github.io/requests_oauth2client/api/#requests_oauth2client.auth.BearerAuth
[BearerToken]: https://guillp.github.io/requests_oauth2client/api/#requests_oauth2client.tokens.BearerToken
[ApiClient]: https://guillp.github.io/requests_oauth2client/api/#requests_oauth2client.api_client.ApiClient