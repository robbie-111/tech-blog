+++
title = "Unity Editor Google 인증을 위한 Go Cloud Functions 개발"
date = 2026-10-13T09:00:00+09:00
tags = ["go", "unity", "oauth", "gcp"]
categories = ["Backend"]
summary = "Unity Editor에서 Google 서비스 접근에 필요한 OAuth 토큰을 발급·갱신하고, 데이터 테이블과 로컬라이제이션 도구에서 사용할 수 있도록 구성한 인증 도구 개발 기록."
draft = false
+++

## 개요

Unity Editor에서 GCP나 Google 스프레드시트처럼 Google 인증 권한이 필요한 서비스를 사용하기 위해, 인증 토큰을 발급받고 유지하는 공통 도구가 필요했다.

각 패키지에 시크릿 키를 배포하는 대신 사용자 인증으로 발급받고, 탈취 시 접근 권한을 철회할 수 있는 OAuth 토큰을 사용하도록 구성했다. Go Cloud Functions로 토큰 교환과 갱신 API를 만들고, 발급받은 토큰은 로컬 보안 저장소에 보관했다. 데이터 테이블 툴이나 로컬라이제이션 도구 등에서도 저장된 토큰과 갱신 기능을 사용할 수 있도록 공통 인터페이스를 제공했다.

토큰의 접근 권한 철회는 Google OAuth의 [revoke 기능](https://developers.google.com/identity/openid-connect/reference#revocation_endpoint)을 통해 처리할 수 있다.

## Editor와 서버의 인증 흐름

Editor에서 브라우저를 열어 Google 로그인과 권한 동의를 진행한다. 인증이 끝나면 로컬 콜백으로 받은 Authorization Code를 서버에 전달하고, 서버는 Google 토큰 엔드포인트에서 이를 토큰으로 교환한다.

```text
브라우저에서 Google 인증
    ↓
Editor의 로컬 콜백으로 인증 코드 수신
    ↓
Go Cloud Functions에서 Google 토큰 교환
    ↓
Editor의 로컬 보안 저장소에 토큰 저장
    ↓
공통 인터페이스를 통해 데이터 테이블·로컬라이제이션 도구에서 사용
```

Editor는 콜백의 `state`를 검사하고 PKCE verifier를 코드 교환 요청에 포함한다. 서버는 Google과의 토큰 교환을 담당하고, Editor는 반환된 토큰의 저장과 만료 시각을 관리한다.

## 토큰 교환과 갱신을 HTTP 함수로 분리

Functions Framework-Go로 코드 교환과 갱신 핸들러를 각각 등록했다. 두 함수는 `POST` 요청을 받아 토큰 정보를 JSON으로 반환한다.

```go
func init() {
    functions.HTTP("ExchangeAuthorizationCode", ExchangeAuthorizationCode)
    functions.HTTP("RefreshAccessToken", RefreshAccessToken)
}
```

| 함수 | 요청 필드 | 역할 |
| --- | --- | --- |
| `ExchangeAuthorizationCode` | `code`, `redirect_uri`, 선택적 `code_verifier` | 인증 코드를 토큰으로 교환 |
| `RefreshAccessToken` | `refresh_token` | Access Token 갱신 |

`ExchangeAuthorizationCode`는 `golang.org/x/oauth2`로 인증 코드를 교환한다. 응답에는 Access Token과 만료 정보가 포함되며, Google에서 발급한 경우 Refresh Token과 ID Token도 함께 전달한다. Editor는 이 결과를 `AuthTokenStorage`를 통해 로컬 보안 저장소에 저장한다.

데이터 테이블 툴이나 로컬라이제이션 도구는 `GoogleApiClient.GetAccessTokenAsync()`를 통해 사용할 토큰을 얻는다. 이 메서드는 만료가 60초 이내로 가까워지면 `RefreshAccessToken`을 호출해 토큰을 갱신하고 저장한다. 갱신 응답에 Refresh Token이 없으면 기존 값을 유지하므로, 각 도구에서 저장과 갱신 로직을 구현할 필요가 없다.

## 배포

두 핸들러를 Google Cloud Functions의 HTTP 함수로 각각 배포하고, OAuth 클라이언트 ID와 Secret은 서버의 `GOOGLE_CLIENT_ID`, `GOOGLE_CLIENT_SECRET` 환경 변수로 제공한다. Editor에서는 각 함수의 URL을 호출하도록 연결한다. 소비 패키지에는 서버의 Client Secret을 전달하지 않고, 공통 인터페이스에서 얻은 사용자 인증 토큰으로 필요한 Google 서비스를 호출하도록 구성했다.
