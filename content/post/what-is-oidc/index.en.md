---
title: "OIDC - Authenticating Users with OAuth 2.0"
description: "Explore OpenID Connect flows, ID Tokens, scopes and claims, and Discovery, then try the Authorization Code Flow in a playground."
date: 2026-11-07T16:51:19+09:00
lastmod: 2026-11-08T17:29:07+09:00
slug: what-is-oidc
image: protocol.png
math: false
license:
hidden: false
comments: true
draft: false

tags:
    - OIDC
    - OAuth
    - Security

categories:
    - Security
    - Web

build:
    list: always
---

---

## 🔑 What Is OIDC?

OIDC (OpenID Connect) is an **identity layer that adds authentication to OAuth 2.0**. It lets a Client establish who the user is based on authentication performed by an Authorization Server, and obtain basic profile information about that user.

In the [previous post on OAuth 2.0]({{< relref "post/what-is-oauth" >}}), we looked at using Access Tokens to access HTTP resources with limited permissions. Receiving an Access Token alone, however, does not establish the user's identity for the Client. OAuth 2.0 is a framework for delegating access.

OIDC adds `openid` to the `scope` of an Authorization Request and delivers the authentication result in a JWT called an **ID Token**. The Client validates this token to identify the user who signed in.

In OIDC, the server that authenticates users and acts as the OAuth 2.0 Authorization Server is called the **OpenID Provider (OP)**. The Client that consumes the authentication result is the **Relying Party (RP)**.

The RP needs to know the OP's Authorization Endpoint, Token Endpoint, and other configuration details. OpenID Connect Discovery 1.0 defines how to retrieve this configuration, as we will see later.

---

## 🔄 Protocol Flow

At a high level, OpenID Connect works as follows.

1. The RP (Client) sends an authentication request to the OP.
2. The OP authenticates the user and obtains any required consent for access.
3. The OP issues an ID Token to the RP, usually along with an Access Token. How they are delivered depends on the flow.
4. The RP can use the Access Token to request user information from the UserInfo Endpoint.
5. The UserInfo Endpoint returns information about the user to the RP.

![The overall OpenID Connect protocol flow](protocol.png)

With PKCE applied to the Authorization Code Flow, the steps are more specific.

1. The RP redirects the user's browser to the OP's Authorization Endpoint.
    - The request includes `scope=openid profile email`, `response_type=code`, `client_id`, and `redirect_uri`.
    - It also includes `state` to associate the response with the request, `nonce` to help prevent ID Token replay, and the PKCE parameters `code_challenge` and `code_challenge_method=S256`.
2. The OP authenticates the user and obtains consent for the requested permissions when needed.
3. The OP returns an Authorization Code and `state` to the RP's registered callback URI.
4. The RP receives the callback and compares the returned `state` with the value it stored. The OP validates the Authorization Code itself in the next step.
5. The RP sends a token exchange request to the Token Endpoint.
    - Parameters such as `grant_type=authorization_code`, `code`, `redirect_uri`, and `code_verifier` are sent in an HTTP POST body encoded as `application/x-www-form-urlencoded`.
    - The RP supplies `client_id` and authenticates as required by its registered Client type and authentication method. A Public Client does not keep a `client_secret`.
6. The OP validates the request and issues an Access Token and an ID Token. It may also issue a Refresh Token, depending on the request and its policy.
7. The RP validates the ID Token's signature and claims.
8. Once validation succeeds, the RP creates an application session for the user.

---

## 🪪 ID Token

An ID Token is a JWT that communicates the result of user authentication to the RP. An Access Token is used to access APIs, while an ID Token lets the RP verify a login result. An ID Token must not be used in place of an Access Token for API calls.

The main claims in the [OpenID Connect Core definition of an ID Token](https://openid.net/specs/openid-connect-core-1_0.html#IDToken) are as follows.

- `iss`: Required. The Issuer Identifier of the OP that issued the token. It must exactly match the Issuer the RP is configured to trust.
- `sub`: Required. Identifies the user within that Issuer. It is case-sensitive, must not exceed 255 ASCII characters, and is not reassigned within the same Issuer.
- `aud`: Required. The intended audience of the token. It must include the RP's `client_id` and can be a string or an array of strings.
- `exp`: Required. The token's expiration time, expressed as a Unix timestamp in a JSON number. Expired tokens must not be accepted.
- `iat`: Required. The time the token was issued, expressed as a Unix timestamp in a JSON number.
- `auth_time`: The time the user was actually authenticated. It must be included if `max_age` was requested or if `auth_time` was requested as an Essential Claim. It can differ from `iat`, which records token issuance.
- `nonce`: A string that associates an authentication request with an ID Token. If it was sent in the request, the ID Token must contain the same value, and the RP must compare it with the stored value. It helps prevent replay attacks.
- `acr`: Optional. Authentication Context Class Reference. A string indicating the assurance level or policy satisfied by the authentication. Its meaning depends on the authentication policies used by the OP and RP.
- `amr`: Optional. Authentication Methods References. An array of strings describing the authentication methods used. For example, `pwd` indicates password authentication.
- `azp`: Optional. The Authorized Party: the `client_id` of the Client to which the ID Token was issued. When `aud` contains multiple audiences, the RP should check that this claim is present. If it is present, the RP should verify that it matches its own `client_id`.

The `state` parameter from the OAuth post and the `nonce` introduced here both associate a response with a request, but they check different parts of the response.

- `state`: Checks that the authorization response received at the callback belongs to a request initiated by the RP. The RP compares the returned `state` with its stored value.
- `nonce`: Checks that the ID Token was issued for that authentication request. The RP compares the token's `nonce` claim with the value stored when it made the request.

For each authentication request, the RP generates an unpredictable, single-use `nonce`, stores it in association with the user's session and that request, and sends it to the OP. If the returned ID Token has no `nonce` or its value does not match, the RP rejects the authentication. Once used for validation, the value is discarded to prevent reuse.

Use the combination of `iss` and `sub` to associate a user with an account in the RP. Email addresses and display names can change and are not guaranteed to be unique user identifiers.

Decoding a JWT to read its contents is different from validating an authentication result. In practice, use a well-established OIDC library and follow the [ID Token validation rules](https://openid.net/specs/openid-connect-core-1_0.html#IDTokenValidation), including checks of the signature, allowed algorithm, `iss`, `aud`, `exp`, and any requested `nonce`.

---

## 🧩 Scopes and Claims

### Defining Scopes

A claim is an individual piece of information about a user or an authentication result. A scope provides a way to request a group of related claims. Just as OAuth 2.0 uses scopes to request access, OIDC uses standard scopes to request user information.

An OIDC request must include `openid`. Additional scopes express what other information the Client needs.

- `profile`: Requests basic profile claims: `name`, `family_name`, `given_name`, `middle_name`, `nickname`, `preferred_username`, `profile`, `picture`, `website`, `gender`, `birthdate`, `zoneinfo`, `locale`, and `updated_at`.
- `email`: Requests the `email` and `email_verified` claims.
- `address`: Requests the `address` claim.
- `phone`: Requests the `phone_number` and `phone_number_verified` claims.

For example, a request might use the following scopes. In an actual URL, spaces and other characters must be URL-encoded.

```text
scope=openid profile email phone
```

Requesting a scope does not guarantee that every associated claim will be returned. The available information, user consent, and OP policy determine what is included. In flows that issue an Access Token, the claims requested through these scopes are returned from the UserInfo Endpoint. Depending on the OP's configuration, they may also appear in the ID Token.

### OIDC Standard Claims

OIDC defines the following standard claims, which can appear in an ID Token or a UserInfo response.

| Claim | Type | Description |
| --- | --- | --- |
| `sub` | string | User identifier within the Issuer |
| `name` | string | Full name |
| `given_name` | string | Given name or first name |
| `family_name` | string | Family name or last name |
| `middle_name` | string | Middle name |
| `nickname` | string | Nickname |
| `preferred_username` | string | Preferred username. It must not be assumed to be unique. |
| `profile` | string | Profile page URL |
| `picture` | string | Profile image URL |
| `website` | string | Website URL |
| `email` | string | Email address. It must not be assumed to be unique. |
| `email_verified` | boolean | Whether the OP has verified ownership of the email address |
| `gender` | string | Gender |
| `birthdate` | string | Date of birth, normally in `YYYY-MM-DD` format. A year alone or a date with `0000` as the year is also allowed. |
| `zoneinfo` | string | IANA time zone name, such as `Asia/Seoul` |
| `locale` | string | BCP 47 language tag, such as `en-US` or `ko-KR` |
| `phone_number` | string | Phone number. E.164 format is recommended; if an extension is included, use RFC 3966 format. |
| `phone_number_verified` | boolean | Whether the OP has verified ownership of the phone number |
| `address` | object | JSON object containing postal address information |
| `updated_at` | number | Time the user's information was last updated, as a Unix timestamp |

### Address Claim

The `address` claim is a JSON object that can contain the following fields. All fields are optional, and their values are strings.

| Field | Description |
| --- | --- |
| `formatted` | Full address for display. It may contain line breaks. |
| `street_address` | Street address, including the building number, unit, or apartment |
| `locality` | City or other locality |
| `region` | State, province, or other regional division |
| `postal_code` | Postal code |
| `country` | Country name |

The following fictional Korean address illustrates the shape of these fields.

```json
{
  "address": {
    "formatted": "대한민국 서울특별시 종로구 예시로 123, 101호",
    "street_address": "예시로 123, 101호",
    "locality": "종로구",
    "region": "서울특별시",
    "postal_code": "03000",
    "country": "대한민국"
  }
}
```

---

## 👤 UserInfo Endpoint

The UserInfo Endpoint is an OAuth 2.0 Protected Resource that returns user information when presented with an Access Token. While an ID Token communicates an authentication result, this endpoint lets the RP retrieve additional user information within the permitted scope.

It must use HTTPS and support both HTTP GET and POST. Here is an example GET request with a Bearer Token in the Authorization header.

```http
GET /userinfo HTTP/1.1
Host: server.example.com
Authorization: Bearer ACCESS_TOKEN
```

A [UserInfo response](https://openid.net/specs/openid-connect-core-1_0.html#UserInfoResponse) is a JSON object by default. It can also be a JWT if signing or encryption was requested during Client registration. If the response is signed, its JWS signature must be validated as well.

The response must contain `sub`, and **it must exactly match the `sub` in the validated ID Token.** If the values differ, the user information in that response must not be used.

---

## 🧭 OIDC Discovery

OIDC Discovery is a standard way for an RP to retrieve the OP's configuration. It provides details such as where to send authentication requests, where to exchange tokens, which authentication methods are supported, and where to find public keys for JWT signature verification.

To retrieve the configuration, remove any trailing slash from the known Issuer URL and append `/.well-known/openid-configuration`. For an Issuer of `https://id.example.com`, the request looks like this.

```http
GET /.well-known/openid-configuration HTTP/1.1
Host: id.example.com
```

If the Issuer contains a path, preserve it. For an Issuer of `https://id.example.com/tenant`, the configuration URL is `https://id.example.com/tenant/.well-known/openid-configuration`.

Here is an example Auth0 Discovery document. The fields and supported values can vary by provider and configuration.

```json
{
  "issuer": "https://{yourDomain}.us.auth0.com/",
  "authorization_endpoint": "https://{yourDomain}.us.auth0.com/authorize",
  "token_endpoint": "https://{yourDomain}.us.auth0.com/oauth/token",
  "device_authorization_endpoint": "https://{yourDomain}.us.auth0.com/oauth/device/code",
  "userinfo_endpoint": "https://{yourDomain}.us.auth0.com/userinfo",
  "mfa_challenge_endpoint": "https://{yourDomain}.us.auth0.com/mfa/challenge",
  "jwks_uri": "https://{yourDomain}.us.auth0.com/.well-known/jwks.json",
  "registration_endpoint": "https://{yourDomain}.us.auth0.com/oidc/register",
  "revocation_endpoint": "https://{yourDomain}.us.auth0.com/oauth/revoke",
  "scopes_supported": [
    "openid",
    "profile",
    "offline_access",
    "name",
    "given_name",
    "family_name",
    "nickname",
    "email",
    "email_verified",
    "picture",
    "created_at",
    "identities",
    "phone",
    "address"
  ],
  "response_types_supported": [
    "code",
    "token",
    "id_token",
    "code token",
    "code id_token",
    "token id_token",
    "code token id_token"
  ],
  "code_challenge_methods_supported": [
    "S256",
    "plain"
  ],
  "response_modes_supported": [
    "query",
    "fragment",
    "form_post"
  ],
  "subject_types_supported": [
    "public"
  ],
  "id_token_signing_alg_values_supported": [
    "HS256",
    "RS256",
    "PS256"
  ],
  "token_endpoint_auth_methods_supported": [
    "client_secret_basic",
    "client_secret_post",
    "private_key_jwt"
  ],
  "claims_supported": [
    "aud",
    "auth_time",
    "created_at",
    "email",
    "email_verified",
    "exp",
    "family_name",
    "given_name",
    "iat",
    "identities",
    "iss",
    "name",
    "nickname",
    "phone_number",
    "picture",
    "sub"
  ],
  "entity_profiles_supported": {
    "client": [
      "native_app",
      "web_app",
      "browser_app",
      "service",
      "ai_agent"
    ],
    "subject": [
      "user",
      "service",
      "ai_agent"
    ]
  },
  "request_uri_parameter_supported": false,
  "request_parameter_supported": false,
  "token_endpoint_auth_signing_alg_values_supported": [
    "RS256",
    "RS384",
    "PS256",
    "PS384",
    "ES256",
    "ES384"
  ]
}
```

The following are the main fields in the [Discovery specification](https://openid.net/specs/openid-connect-discovery-1_0.html#ProviderMetadata). The table also includes optional fields that are not present in the example above.

| Field | Description |
| --- | --- |
| `issuer` | OP identifier. It must exactly match the configured Issuer and the ID Token's `iss`. |
| `authorization_endpoint` | URL for authentication and authorization requests through the browser |
| `token_endpoint` | URL for exchanging an Authorization Code or other grant for tokens |
| `userinfo_endpoint` | URL for retrieving user information with an Access Token |
| `jwks_uri` | URL of the OP's public key set (JWKS), used to verify asymmetric signatures |
| `registration_endpoint` | Dynamic Client Registration URL, provided when supported. This is different from a registration page in an administration console. |
| `scopes_supported` | Scopes supported by the OP |
| `response_types_supported` | Supported Response Type combinations, such as `code` and `id_token` |
| `response_modes_supported` | Supported response delivery modes, such as `query`, `fragment`, and `form_post` |
| `grant_types_supported` | Supported Grant Types. If omitted, the specification's defaults are `authorization_code` and `implicit`. |
| `acr_values_supported` | Supported authentication context class values |
| `subject_types_supported` | How `sub` values are assigned. `public` identifies the same user with the same value across Clients; `pairwise` uses a different value for each Sector Identifier. |
| `id_token_signing_alg_values_supported` | Supported JWS `alg` values for ID Token signatures |
| `id_token_encryption_alg_values_supported` | Supported JWE key management algorithms (`alg`) for ID Token encryption |
| `id_token_encryption_enc_values_supported` | Supported JWE content encryption algorithms (`enc`) for ID Tokens |
| `userinfo_signing_alg_values_supported` | Supported JWS `alg` values for UserInfo response signatures |
| `userinfo_encryption_alg_values_supported` | Supported JWE `alg` values for UserInfo response encryption |
| `userinfo_encryption_enc_values_supported` | Supported JWE `enc` values for UserInfo response content encryption |
| `request_object_signing_alg_values_supported` | Supported JWS `alg` values for signing Request Objects, which carry authentication requests as JWTs |
| `request_object_encryption_alg_values_supported` | Supported JWE `alg` values for Request Object encryption |
| `request_object_encryption_enc_values_supported` | Supported JWE `enc` values for Request Object content encryption |
| `token_endpoint_auth_methods_supported` | Supported Client authentication methods at the Token Endpoint, such as `client_secret_basic` and `private_key_jwt` |
| `claims_supported` | Claims the OP can provide. This does not mean every claim will be returned to a particular Client. |
| `code_challenge_methods_supported` | Supported PKCE Code Challenge methods, such as `S256`. This is an OAuth metadata extension field. |

An address returned by Discovery does not make an arbitrary OP trustworthy. Retrieve the configuration over HTTPS from an Issuer the RP trusts, and validate that the returned `issuer` exactly matches the expected value.

The connection flow is shown below.

![Retrieving OP configuration through Discovery and using it for authentication](oidc-discovery-flow.png)

---

## 🧪 Hands-On: OpenID Connect

[OAuth 2.0 Playground](https://www.oauth.com/playground/) provides a simulated Authorization Server where we can inspect each step. This time, let's use the [OpenID Connect Authorization Code Flow](https://www.oauth.com/playground/oidc.html) to obtain both an Access Token and an ID Token. The `client_id`, `client_secret`, and tokens below are demonstration values. Secrets and tokens used in real services must be managed securely.

**First, register a Client and a user.** Click `Register a Client` in the playground, complete registration, and check the issued Client ID, Client Secret, and generated user credentials. Then open the OpenID Connect Authorization Code Flow.

In a real service, you also need to register the Client with the OP in advance and configure its callback Redirect URI and Client authentication method. Discovery retrieves OP configuration; it does not replace Client registration.

### Inspect the Authorization URL

Notice that `scope` contains `openid`. The `profile` and `email` scopes request user information, while `photos` is a separate scope defined by the playground. The request's `state` and `nonce` values will be used to validate the responses later.

![Inspecting the OpenID Connect Authorization URL and scopes](oidc-auth-url.png)

This example uses the Authorization Code Flow without PKCE parameters. In an actual application, apply PKCE with `S256` as described earlier.

### Sign In and Validate State

Click `Authorize` to navigate to the OP, then sign in with the user credentials generated during registration. If a consent screen appears, review the requested permissions and approve access.

![The OP's sign-in and consent screen](auth-dialog.png)

After returning to the callback, compare the returned `state` with the value stored when the request was made. Continue only if they match.

![Comparing State after the callback](state-param.png)

### Exchange the Code for Tokens

Request tokens using the Authorization Code. The playground also sends a Client Secret. In a real application, use the authentication method appropriate to the Client type.

![Request to exchange an Authorization Code for tokens](token-request.png)

We have our tokens! Unlike the previous OAuth exercise, this response includes an `id_token`. The `expires_in` field gives the Access Token's lifetime in seconds; the ID Token's expiration time is recorded in its own `exp` claim.

![Token response containing an Access Token and an ID Token](token-response.png)

```json
{
  "token_type": "Bearer",
  "expires_in": 86400,
  "access_token": "iD3HGFM86rYvNHipok-yXDeeIYRFUT08EyFkswK7mLiYOqeCJmPAz0keLs7GJXn5yAetko43",
  "scope": "openid profile email photo",
  "id_token": "eyJraWQiOiJzMTZ0cVNtODhwREo4VGZCXzdrSEtQUkFQRjg1d1VEVGxteW85SUxUZTdzIiwiYWxnIjoiUlMyNTYifQ.eyJzdWIiOiJibG9vZHktc3RvcmtAZXhhbXBsZS5jb20iLCJuYW1lIjoiQmxvb2R5IFN0b3JrIiwiZW1haWwiOiJibG9vZHktc3RvcmtAZXhhbXBsZS5jb20iLCJpc3MiOiJodHRwczovL3BrLWRlbW8ub2t0YS5jb20vb2F1dGgyL2RlZmF1bHQiLCJhdWQiOiJvZHFIaWRMVV84S1VKT2xqcEJSLWpuTlMiLCJpYXQiOjE3OTQxMjQ4NDQsImV4cCI6MTc5NjcxNjg0NCwiYW1yIjpbInB3ZCJdfQ.ZoPvZPaomdOnnz2GFRGbgaW7PPWIMFDqSBp0gbN4An4a9F-Bc-4_T9EBGV8aGetyjZYAON0gjNV0p0NGFiwettePWKuxBzusuGCEd9iXWWUO9-WTF5e2AGr3_jkg34dbxfiFXy3KgH7m0czm809cMaiZ_ofLYgJHVD8lqMQoWifhoNhpjPqa19Svc3nCHzSYHUgTXQWvA56NmQvyVPh_OM7GMpc6zHopmihJqt3eREof8N-bOd7FL39jeam2-k1TFSDogyJE513aC0OssRADr_TWvtL8xoaPkXM_7bXYs9_7erXmzF9la0hvmOuasieetpLhOvFeoiOJWCU9xhxj4Q"
}
```

### Read the ID Token

Let's decode the Header and Payload of `id_token`.

![Decoded ID Token Header and Payload](parsed-token.png)

The Header looks like this. `kid` is used to select the verification key, and `alg` identifies the signature algorithm.

```json
{
  "kid": "s16tqSm88pDJ8TfB_7kHKPRAPF85wUDTlmyo9ILTe7s",
  "alg": "RS256"
}
```

The Payload contains claims about the authenticated user and the token's issuance.

```json
{
  "sub": "bloody-stork@example.com",
  "name": "Bloody Stork",
  "email": "bloody-stork@example.com",
  "iss": "https://pk-demo.okta.com/oauth2/default",
  "aud": "odqHidLU_8KUJOljpBR-jnNS",
  "iat": 1794124844,
  "exp": 1796716844,
  "amr": [
    "pwd"
  ]
}
```

Here, `pwd` in `amr` indicates password authentication. Even though `sub` looks like an email address, account association should use `iss` and `sub`, rather than `email`.

However, the playground response above is missing the `nonce` sent in the request. When a request includes `nonce`, the ID Token must contain the same value, so **a real RP must not treat this response as a successful authentication.** Use this screen to examine the token's structure. An actual login should complete only after the signature, required claims, and `nonce` have all been validated successfully.

---

## ⚖️ Limitations of OIDC

- Fine-grained authorization: OIDC focuses on authentication and conveying user information. Application roles and resource-specific permissions need to be designed separately.
- Per-service sessions: The OP's login state can enable SSO, but each RP's application session is not automatically shared or terminated at the same time.
- Dependence on the OP: OP outages or changes in account policy can affect new logins and other operations.
- Implementation complexity: Token validation, key rotation, and session management must be implemented correctly. A well-established OIDC library helps with this.
- Token and session theft: An implementation that allows stolen ID Tokens to be replayed, or theft of an RP session, can lead to account compromise. Stealing an ID Token alone does not grant access to every RP; checks such as audience, expiration, and request association still apply.

Some of these limitations follow from OIDC's focus on using authentication performed by an external OP. For example, terminating every service's session together may not always be desirable. Services that need coordinated logout can consider a separate specification such as [OpenID Connect Back-Channel Logout](https://openid.net/specs/openid-connect-backchannel-1_0.html).

---

## 📚 References

- [OpenID Connect Core 1.0](https://openid.net/specs/openid-connect-core-1_0.html)
- [Auth0: Configure Applications with OIDC Discovery](https://auth0.com/docs/get-started/applications/configure-applications-with-oidc-discovery)
- [OAuth 2.0 Playground: OpenID Connect Authorization Code Flow](https://www.oauth.com/playground/oidc.html)
- [OpenID Connect Discovery 1.0](https://openid.net/specs/openid-connect-discovery-1_0.html)
- [OpenID Connect Back-Channel Logout 1.0](https://openid.net/specs/openid-connect-backchannel-1_0.html)
- [RFC 8414: OAuth 2.0 Authorization Server Metadata](https://www.rfc-editor.org/rfc/rfc8414.html)
