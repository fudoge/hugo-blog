---
title: "OIDC - OAuth 2.0 기반으로 사용자 인증하기"
description: "OpenID Connect의 인증 흐름과 ID Token, Scope와 Claim, Discovery를 알아보고 Playground에서 Authorization Code Flow를 실습해보자."
date: 2026-11-07T16:51:22+09:00
lastmod: 2026-11-08T17:19:52+09:00
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

## 🔑 OIDC란?

OIDC(OpenID Connect)는 **OAuth 2.0 위에 인증(Authentication)을 더한 identity layer**이다. Client는 Authorization Server가 수행한 인증을 바탕으로 사용자가 누구인지 확인하고, 사용자의 기본 프로필 정보도 얻을 수 있다.

[지난 OAuth 2.0 글]({{< relref "post/what-is-oauth" >}})에서는 Access Token으로 HTTP 리소스에 제한적으로 접근하는 방법을 살펴봤다. 하지만 Access Token을 받았다는 사실만으로 Client가 사용자의 신원을 확인할 수 있는 것은 아니다. OAuth 2.0은 접근 권한 위임을 위한 프레임워크이기 때문이다.

OIDC는 Authorization Request의 `scope`에 `openid`를 포함하고, 인증 결과를 **ID Token**이라는 JWT로 전달한다. Client는 이를 검증해 로그인한 사용자를 식별한다.

OIDC에서는 OAuth 2.0의 Authorization Server 역할을 하며 사용자를 인증하는 서버를 **OpenID Provider(OP)**, 인증 결과를 받아 사용하는 Client를 Relying Party(RP)라고 부른다.

RP는 OP의 Authorization Endpoint, Token Endpoint 등의 정보를 알아야 한다. 이 설정을 조회하는 방법은 뒤에서 살펴볼 OpenID Connect Discovery 1.0에 정의되어 있다.

---

## 🔄 프로토콜 흐름 보기

OpenID Connect의 전체 흐름은 다음과 같다.

1. RP(Client)가 OP에 인증 요청을 보낸다.
2. OP는 사용자를 인증하고, 필요한 접근 동의를 확인한다.
3. OP는 RP에 ID Token을 발급하고, 일반적으로 Access Token도 함께 발급한다. 전달 방식은 사용하는 Flow에 따라 달라진다.
4. RP는 Access Token으로 UserInfo Endpoint에 사용자 정보를 요청할 수 있다.
5. UserInfo Endpoint는 RP에 사용자 정보를 응답한다.

![OpenID Connect의 전체 프로토콜 흐름](protocol.png)

Authorization Code Flow에 PKCE를 적용하면 다음 과정을 거친다.

1. RP는 사용자의 브라우저를 OP의 Authorization Endpoint로 리다이렉트한다.
    - `scope=openid profile email`, `response_type=code`, `client_id`, `redirect_uri`를 포함한다.
    - 요청과 응답을 연결할 `state`, ID Token의 재사용을 방어할 `nonce`, PKCE용 `code_challenge`와 `code_challenge_method=S256`도 보낸다.
2. OP는 사용자를 인증하고, 필요한 경우 권한에 대한 동의를 받는다.
3. OP는 등록된 RP의 콜백 주소로 Authorization Code와 `state`를 전달한다.
4. RP는 콜백을 받아 저장한 값과 응답의 `state`를 비교한다. Authorization Code 자체의 유효성은 다음 단계에서 OP가 검증한다.
5. RP는 Token Endpoint에 토큰 교환 요청을 보낸다.
    - `grant_type=authorization_code`, `code`, `redirect_uri`, `code_verifier` 등을 HTTP POST의 `application/x-www-form-urlencoded` 본문으로 전달한다.
    - 등록된 Client 유형과 인증 방식에 따라 `client_id`를 전달하고 Client 인증도 수행한다. Public Client는 `client_secret`을 보관하지 않는다.
6. OP는 요청을 검증하고 Access Token과 ID Token을 발급한다. Refresh Token은 요청과 OP 정책에 따라 발급될 수 있다.
7. RP는 ID Token의 서명과 Claim을 검증한다.
8. 검증에 성공하면 RP는 해당 사용자의 애플리케이션 세션을 생성한다.

---

## 🪪 ID Token

ID Token은 사용자의 인증 결과를 RP에 전달하는 JWT이다. Access Token이 API 접근에 쓰인다면, ID Token은 RP가 로그인 결과를 확인하는 데 쓰인다. ID Token을 API 호출용 Access Token 대신 사용해서는 안 된다.

[OpenID Connect Core의 ID Token 정의](https://openid.net/specs/openid-connect-core-1_0.html#IDToken)에 나오는 주요 Claim은 다음과 같다.

- `iss`: 필수. 토큰을 발급한 OP의 Issuer 식별자이다. RP가 신뢰하도록 설정한 Issuer와 정확히 일치해야 한다.
- `sub`: 필수. 해당 Issuer 안에서 사용자를 식별하는 값이다. 대소문자를 구분하며, 255 ASCII 문자를 넘지 않는다. 같은 Issuer 안에서 재할당되지 않는 식별자이다.
- `aud`: 필수. 토큰을 사용할 대상이다. RP의 `client_id`를 포함해야 하며, 문자열 또는 문자열 배열로 표현한다.
- `exp`: 필수. 토큰 만료 시각이다. Unix Timestamp를 JSON number로 표현하며, 만료된 토큰은 허용하지 않는다.
- `iat`: 필수. 토큰 발급 시각이다. Unix Timestamp를 JSON number로 표현한다.
- `auth_time`: 사용자가 실제로 인증된 시각이다. `max_age`를 요청했거나 Essential Claim으로 요청했다면 반드시 포함되어야 한다. 토큰 발급 시각인 `iat`와는 다를 수 있다.
- `nonce`: 인증 요청과 ID Token을 연결하는 문자열이다. 요청에 보냈다면 같은 값이 ID Token에 반드시 포함되어야 하며, RP도 저장한 값과 비교해야 한다. 재사용 공격을 방어하는 데 쓰인다.
- `acr`: 선택. Authentication Context Class Reference이다. 인증이 충족한 보증 수준이나 정책을 문자열로 나타낸다. 값의 의미는 OP와 RP가 사용하는 인증 정책에 따라 해석한다.
- `amr`: 선택. Authentication Methods References이다. 사용한 인증 방법을 문자열 배열로 나타낸다. 예를 들어 `pwd`는 비밀번호 인증을 뜻한다.
- `azp`: 선택. Authorized Party로, ID Token이 발급된 Client의 `client_id`이다. `aud`가 여러 개라면 이 값의 존재 여부를 확인하는 것이 권장되며, 값이 있다면 자신의 `client_id`인지 확인하는 것이 권장된다.

지난 OAuth 글에서 살펴본 `state`와 이번에 등장한 `nonce`는 모두 요청과 응답을 연결하지만, 확인하는 대상이 다르다.

- `state`: 콜백의 인가 응답이 RP가 시작한 요청에 대한 것인지 확인한다. RP는 콜백으로 돌아온 `state`를 저장한 값과 비교한다.
- `nonce`: ID Token이 해당 인증 요청에 대해 발급된 것인지 확인한다. RP는 ID Token 안의 `nonce` Claim을 요청 시 저장한 값과 비교한다.

RP는 인증 요청마다 예측하기 어려운 일회성 `nonce`를 생성해 사용자 세션 및 해당 요청에 연결하여 저장하고, OP에 전달한다. 반환된 ID Token의 `nonce`가 없거나 저장한 값과 다르면 인증을 거부한다. 검증에 사용한 값은 다시 사용하지 않도록 폐기한다.

사용자를 RP의 계정과 연결할 때는 `iss`와 `sub`의 조합을 기준으로 삼는다. 이메일이나 표시 이름은 바뀔 수 있고, 사용자 식별자로서 고유함이 보장되지 않는다.

JWT를 디코딩해 내용을 읽는 것과 인증 결과를 검증하는 것은 다르다. 실제 구현에서는 검증된 OIDC 라이브러리를 사용하고, [ID Token 검증 규칙](https://openid.net/specs/openid-connect-core-1_0.html#IDTokenValidation)에 따라 서명과 허용한 알고리즘, `iss`, `aud`, `exp`, 요청한 `nonce` 등을 확인해야 한다.

---

## 🧩 Scope와 Claim들

### Scope 정의

Claim은 사용자나 인증 결과에 대한 개별 정보이고, Scope는 관련 Claim을 묶어서 요청하는 수단이다. OAuth 2.0에서 Scope로 접근 범위를 요청하듯이, OIDC에서는 표준 Scope로 사용자 정보를 요청한다.

OIDC 요청에는 `openid`가 반드시 포함되어야 한다. 나머지 Scope로 어떤 정보가 필요한지 추가로 표현한다.

- `profile`: 기본 프로필 Claim을 요청한다. `name`, `family_name`, `given_name`, `middle_name`, `nickname`, `preferred_username`, `profile`, `picture`, `website`, `gender`, `birthdate`, `zoneinfo`, `locale`, `updated_at`이 해당한다.
- `email`: `email`, `email_verified` Claim을 요청한다.
- `address`: `address` Claim을 요청한다.
- `phone`: `phone_number`, `phone_number_verified` Claim을 요청한다.

예를 들어, 다음처럼 요청할 수 있다. 실제 URL에서는 공백 등을 URL 인코딩한다.

```text
scope=openid profile email phone
```

Scope로 요청했다고 해서 모든 Claim이 반드시 돌아오는 것은 아니다. OP가 보유한 정보와 사용자 동의, 제공 정책에 따라 일부만 반환될 수 있다. Access Token을 발급하는 Flow에서는 이 Scope로 요청한 Claim을 UserInfo Endpoint에서 받으며, OP의 설정에 따라 ID Token에도 포함될 수 있다.

### OIDC 표준 Claim

OIDC에서 정의한 표준 Claim은 ID Token이나 UserInfo 응답에 포함될 수 있다.

| Claim | Type | Description |
| --- | --- | --- |
| `sub` | string | 해당 Issuer 안에서의 사용자 식별자 |
| `name` | string | 사용자의 전체 이름 |
| `given_name` | string | 이름(first name) |
| `family_name` | string | 성(last name) |
| `middle_name` | string | 미들네임 |
| `nickname` | string | 별명 |
| `preferred_username` | string | 사용자가 선호하는 사용자 이름. 고유한 값이라고 가정해서는 안 된다. |
| `profile` | string | 프로필 페이지 URL |
| `picture` | string | 프로필 이미지 URL |
| `website` | string | 웹사이트 URL |
| `email` | string | 이메일 주소. 고유한 값이라고 가정해서는 안 된다. |
| `email_verified` | boolean | OP가 이메일 주소의 소유를 확인했는지 여부 |
| `gender` | string | 성별 |
| `birthdate` | string | 생년월일. 일반적으로 `YYYY-MM-DD` 형식이며, 연도만 있거나 연도가 `0000`인 표현도 허용된다. |
| `zoneinfo` | string | IANA time zone 문자열. 예: `Asia/Seoul` |
| `locale` | string | BCP 47 언어 태그. 예: `en-US`, `ko-KR` |
| `phone_number` | string | 전화번호. E.164 형식을 권장하며, 내선번호가 있다면 RFC 3966 형식을 사용한다. |
| `phone_number_verified` | boolean | OP가 전화번호의 소유를 확인했는지 여부 |
| `address` | object | 우편 주소 정보를 담은 JSON 객체 |
| `updated_at` | number | 사용자 정보가 마지막으로 갱신된 시각. Unix Timestamp |

### Address Claim

`address`는 다음 필드를 담을 수 있는 JSON 객체이다. 모든 필드는 선택 사항이며, 제공되는 필드의 값은 문자열이다.

| 필드 | 설명 |
| --- | --- |
| `formatted` | 표시용 전체 주소. 줄바꿈을 포함할 수 있다. |
| `street_address` | 도로명, 건물 번호, 동·호수 등을 포함한 상세 주소 |
| `locality` | 도시 등 지역 단위 |
| `region` | 시·도 등 광역 행정 구역 |
| `postal_code` | 우편번호 |
| `country` | 국가명 |

다음은 각 필드의 형태를 보여주기 위한 가상 주소 예시이다.

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

UserInfo Endpoint는 Access Token으로 사용자 정보를 조회하는 OAuth 2.0 Protected Resource이다. ID Token이 인증 결과를 전달한다면, 이 엔드포인트는 허용된 범위의 사용자 정보를 추가로 가져오는 역할을 한다.

HTTPS를 사용하고 HTTP GET과 POST를 모두 지원해야 한다. 다음은 Bearer Token을 Authorization 헤더에 넣는 GET 요청 예시이다.

```http
GET /userinfo HTTP/1.1
Host: server.example.com
Authorization: Bearer ACCESS_TOKEN
```

[UserInfo 응답](https://openid.net/specs/openid-connect-core-1_0.html#UserInfoResponse)은 기본적으로 JSON 객체이다. Client 등록 시 서명이나 암호화를 요청했다면 JWT로 받을 수도 있다. 서명된 응답이라면 JWS 서명도 검증해야 한다.

응답의 `sub`는 필수이며, **검증한 ID Token의 `sub`와 정확히 일치해야 한다.** 다르면 해당 응답의 사용자 정보를 사용해서는 안 된다.

---

## 🧭 OIDC Discovery

OIDC Discovery는 RP가 OP의 설정 정보를 조회하는 표준 기능이다. 로그인 요청을 보낼 주소, 토큰 교환 주소, 지원하는 인증 방식, JWT 서명 검증용 공개 키의 위치 등을 알려준다.

알고 있는 Issuer URL의 끝에서 trailing slash를 제거하고 `/.well-known/openid-configuration`을 붙여 설정을 조회한다. 예를 들어 Issuer가 `https://id.example.com`이라면 다음과 같다.

```http
GET /.well-known/openid-configuration HTTP/1.1
Host: id.example.com
```

Issuer에 경로가 있다면 그 경로도 유지한다. Issuer가 `https://id.example.com/tenant`라면 조회 주소는 `https://id.example.com/tenant/.well-known/openid-configuration`이다.

다음은 Auth0 Discovery 문서의 예시이다. 제공자와 설정에 따라 필드 및 지원 값은 달라질 수 있다.

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


[Discovery 명세](https://openid.net/specs/openid-connect-discovery-1_0.html#ProviderMetadata)의 주요 필드는 다음과 같다. 아래 목록에는 위 예시에 없는 선택 필드도 포함되어 있다.

| 필드 | 설명 |
| --- | --- |
| `issuer` | OP의 식별자. 설정한 Issuer 및 ID Token의 `iss`와 정확히 일치해야 한다. |
| `authorization_endpoint` | 브라우저를 통해 인증·인가 요청을 보내는 주소 |
| `token_endpoint` | Authorization Code 등을 토큰으로 교환하는 주소 |
| `userinfo_endpoint` | Access Token으로 사용자 정보를 조회하는 주소 |
| `jwks_uri` | OP의 공개 키 집합(JWKS)을 조회하는 주소. 비대칭 서명 검증에 사용한다. |
| `registration_endpoint` | Dynamic Client Registration 주소. 지원하는 경우 제공하며, 일반적인 관리 콘솔의 등록 페이지와는 다르다. |
| `scopes_supported` | OP가 지원하는 Scope 목록 |
| `response_types_supported` | `code`, `id_token` 등 지원하는 Response Type 조합 |
| `response_modes_supported` | `query`, `fragment`, `form_post` 등 응답 전달 방식 |
| `grant_types_supported` | 지원하는 Grant Type 목록. 생략되면 명세의 기본값은 `authorization_code`, `implicit`이다. |
| `acr_values_supported` | 지원하는 인증 컨텍스트 클래스 값 목록 |
| `subject_types_supported` | `sub` 발급 방식. `public`은 같은 사용자를 Client들에 동일한 값으로, `pairwise`는 Sector Identifier별로 다른 값으로 식별한다. |
| `id_token_signing_alg_values_supported` | ID Token 서명에 지원하는 JWS `alg` 목록 |
| `id_token_encryption_alg_values_supported` | ID Token 암호화에 지원하는 JWE 키 관리 알고리즘(`alg`) 목록 |
| `id_token_encryption_enc_values_supported` | ID Token의 본문 암호화에 지원하는 JWE `enc` 목록 |
| `userinfo_signing_alg_values_supported` | UserInfo 응답 서명에 지원하는 JWS `alg` 목록 |
| `userinfo_encryption_alg_values_supported` | UserInfo 응답 암호화에 지원하는 JWE `alg` 목록 |
| `userinfo_encryption_enc_values_supported` | UserInfo 응답 본문 암호화에 지원하는 JWE `enc` 목록 |
| `request_object_signing_alg_values_supported` | JWT 형식의 인증 요청인 Request Object의 서명에 지원하는 JWS `alg` 목록 |
| `request_object_encryption_alg_values_supported` | Request Object 암호화에 지원하는 JWE `alg` 목록 |
| `request_object_encryption_enc_values_supported` | Request Object 본문 암호화에 지원하는 JWE `enc` 목록 |
| `token_endpoint_auth_methods_supported` | `client_secret_basic`, `private_key_jwt` 등 Token Endpoint에서 지원하는 Client 인증 방식 |
| `claims_supported` | OP가 제공할 수 있는 Claim 목록. 해당 Client에 모두 반환된다는 의미는 아니다. |
| `code_challenge_methods_supported` | `S256` 등 지원하는 PKCE Code Challenge 방식. OAuth 메타데이터 확장 필드이다. |

Discovery로 가져온 주소라고 해서 임의의 OP를 신뢰해서는 안 된다. RP가 신뢰하는 Issuer를 기준으로 HTTPS로 조회하고, 응답의 `issuer`가 기대한 값과 일치하는지 검증해야 한다.

연결 흐름은 다음과 같다.

![Discovery를 통해 OP의 설정을 조회하고 인증에 사용하는 흐름](oidc-discovery-flow.png)

---

## 🧪 실습: OpenID Connect

[OAuth 2.0 Playground](https://www.oauth.com/playground/)는 모의 Authorization Server로 각 단계를 확인할 수 있는 실습 환경이다. 이번에는 [OpenID Connect Authorization Code Flow](https://www.oauth.com/playground/oidc.html)에서 Access Token과 ID Token을 함께 받아보자. 아래 예시의 `client_id`, `client_secret`, 토큰은 실습용 값이며, 실제 서비스의 Secret과 토큰은 안전하게 관리해야 한다.

**먼저 Client와 사용자를 등록해야 한다.** Playground에서 `Register a Client`를 눌러 등록을 진행하고, 발급된 Client ID와 Client Secret, 생성된 사용자의 로그인 정보를 확인한다. 등록을 마치면 OpenID Connect Authorization Code Flow로 이동한다.

실제 서비스에서도 OP에 Client를 미리 등록하고, 콜백으로 사용할 Redirect URI와 Client 인증 방식을 설정해야 한다. Discovery는 OP 설정을 조회하는 기능이므로, Client 등록을 대신하지 않는다.

### Authorization URL 확인

Authorization URL의 `scope`에 `openid`가 포함되어 있는지 주목하자. `profile`과 `email`은 사용자 정보를 요청하며, `photos`는 Playground가 정의한 별도 Scope이다. 요청에 포함된 `state`와 `nonce`는 이후 응답을 검증할 때 사용한다.

![OpenID Connect Authorization URL과 Scope 확인](oidc-auth-url.png)

이 실습 화면은 PKCE 파라미터가 없는 Authorization Code 예시이다. 실제 애플리케이션에서는 앞서 설명한 것처럼 PKCE의 `S256`을 함께 적용한다.

### 로그인과 State 검증

`Authorize`를 눌러 OP로 이동하고, 등록 과정에서 생성한 사용자 정보로 로그인한다. 동의 화면이 나타나면 요청한 권한을 확인하고 승인한다.

![OP의 로그인 및 권한 동의 화면](auth-dialog.png)

콜백으로 돌아오면 요청 시 저장한 값과 응답의 `state`를 비교한다. 일치할 때만 다음 단계로 진행한다.

![콜백에서 State를 비교하는 화면](state-param.png)

### 토큰 교환

Authorization Code로 토큰을 요청한다. Playground에서는 Client Secret도 함께 전달하며, 실제 애플리케이션에서는 Client 유형에 맞는 인증 방식을 사용한다.

![Authorization Code를 토큰으로 교환하는 요청](token-request.png)

응답으로 토큰을 받았다! 지난 OAuth 실습과 달리 `id_token`이 함께 포함되어 있다. `expires_in`은 Access Token의 유효기간을 초 단위로 나타내며, ID Token의 만료 시각은 내부의 `exp`로 확인한다.

![Access Token과 ID Token을 포함한 토큰 응답](token-response.png)

```json
{
  "token_type": "Bearer",
  "expires_in": 86400,
  "access_token": "iD3HGFM86rYvNHipok-yXDeeIYRFUT08EyFkswK7mLiYOqeCJmPAz0keLs7GJXn5yAetko43",
  "scope": "openid profile email photo",
  "id_token": "eyJraWQiOiJzMTZ0cVNtODhwREo4VGZCXzdrSEtQUkFQRjg1d1VEVGxteW85SUxUZTdzIiwiYWxnIjoiUlMyNTYifQ.eyJzdWIiOiJibG9vZHktc3RvcmtAZXhhbXBsZS5jb20iLCJuYW1lIjoiQmxvb2R5IFN0b3JrIiwiZW1haWwiOiJibG9vZHktc3RvcmtAZXhhbXBsZS5jb20iLCJpc3MiOiJodHRwczovL3BrLWRlbW8ub2t0YS5jb20vb2F1dGgyL2RlZmF1bHQiLCJhdWQiOiJvZHFIaWRMVV84S1VKT2xqcEJSLWpuTlMiLCJpYXQiOjE3OTQxMjQ4NDQsImV4cCI6MTc5NjcxNjg0NCwiYW1yIjpbInB3ZCJdfQ.ZoPvZPaomdOnnz2GFRGbgaW7PPWIMFDqSBp0gbN4An4a9F-Bc-4_T9EBGV8aGetyjZYAON0gjNV0p0NGFiwettePWKuxBzusuGCEd9iXWWUO9-WTF5e2AGr3_jkg34dbxfiFXy3KgH7m0czm809cMaiZ_ofLYgJHVD8lqMQoWifhoNhpjPqa19Svc3nCHzSYHUgTXQWvA56NmQvyVPh_OM7GMpc6zHopmihJqt3eREof8N-bOd7FL39jeam2-k1TFSDogyJE513aC0OssRADr_TWvtL8xoaPkXM_7bXYs9_7erXmzF9la0hvmOuasieetpLhOvFeoiOJWCU9xhxj4Q"
}
```

### ID Token 읽기

`id_token`의 Header와 Payload를 직접 디코딩해보자. 

![ID Token의 Header와 Payload를 디코딩한 화면](parsed-token.png)

Header는 다음과 같다. `kid`는 검증 키를 선택하는 데 쓰이며, `alg`는 서명 알고리즘이다.

```json
{
  "kid": "s16tqSm88pDJ8TfB_7kHKPRAPF85wUDTlmyo9ILTe7s",
  "alg": "RS256"
}
```

Payload에는 인증된 사용자와 토큰 발급에 관한 Claim이 들어 있다.

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

이 예시에서는 `amr`의 `pwd`로 비밀번호 인증을 사용했음을 알 수 있다. `sub`가 이메일 모양이더라도 사용자 계정 연결은 `email`이 아닌 `iss`와 `sub`를 기준으로 해야 한다.

다만, 위 Playground 응답에는 요청에 보냈던 `nonce`가 없다. 요청에 `nonce`를 포함했다면 ID Token에도 동일한 값이 있어야 하므로, **실제 RP에서는 이 응답을 인증 성공으로 처리해서는 안 된다.** 이 화면은 토큰의 구조를 살펴보는 데 사용하고, 실제 로그인 처리는 서명과 필수 Claim, `nonce` 검증까지 성공한 뒤 완료해야 한다.

---

## ⚖️ OIDC의 한계

- 세밀한 권한 관리: OIDC는 사용자의 인증과 정보 전달에 초점을 둔다. 서비스 내부의 역할과 리소스별 접근 권한은 애플리케이션에서 별도로 설계해야 한다.
- 서비스별 세션: OP의 로그인 상태를 이용해 SSO를 구성할 수 있지만, 각 RP의 애플리케이션 세션이 자동으로 공유되거나 동시에 종료되는 것은 아니다.
- OP 의존성: OP의 장애나 계정 정책 변경이 신규 로그인 등에 영향을 줄 수 있다.
- 구현 복잡성: 토큰 검증, 키 교체, 세션 관리 등을 올바르게 구현해야 한다. 검증된 OIDC 라이브러리를 사용하는 것이 좋다.
- 토큰과 세션 탈취: 탈취된 ID Token을 재사용할 수 있는 취약한 구현이나 RP 세션 탈취는 계정 침해로 이어질 수 있다. 다만 ID Token을 탈취했다는 사실만으로 모든 RP에 로그인할 수 있는 것은 아니며, Audience와 만료, 요청 연결 검증 등이 적용된다.

일부 한계는 OIDC가 외부 OP의 인증 결과를 사용하는 데 초점을 두기 때문에 생긴다. 예를 들어 모든 서비스의 세션을 항상 함께 종료하는 것이 적절하지 않을 수도 있다. 로그아웃 연동이 필요한 서비스에는 [OpenID Connect Back-Channel Logout](https://openid.net/specs/openid-connect-backchannel-1_0.html) 같은 별도 규격을 검토할 수 있다.

---

## 📚 References

- [OpenID Connect Core 1.0](https://openid.net/specs/openid-connect-core-1_0.html)
- [Auth0: Configure Applications with OIDC Discovery](https://auth0.com/docs/get-started/applications/configure-applications-with-oidc-discovery)
- [OAuth 2.0 Playground: OpenID Connect Authorization Code Flow](https://www.oauth.com/playground/oidc.html)
- [OpenID Connect Discovery 1.0](https://openid.net/specs/openid-connect-discovery-1_0.html)
- [OpenID Connect Back-Channel Logout 1.0](https://openid.net/specs/openid-connect-backchannel-1_0.html)
- [RFC 8414: OAuth 2.0 Authorization Server Metadata](https://www.rfc-editor.org/rfc/rfc8414.html)
