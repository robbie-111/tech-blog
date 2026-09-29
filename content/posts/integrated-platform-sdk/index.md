+++
title = "통합 플랫폼 SDK 개발"
date = 2026-09-21T14:00:00+09:00
tags = ["sdk", "unity", "platform", "ios", "android"]
categories = ["SDK"]
summary = "분산된 공용 모듈의 의존성과 초기화 문제를 줄이기 위해, 설정과 기능 진입점을 하나로 모은 Unity 통합 플랫폼 SDK 개발 기록."
draft = false
+++

## 개요

기존에는 인증, 결제, 광고, 커뮤니티 등의 공용 모듈을 각각 별도의 패키지로 제공했다. 각 모듈이 공통 기능을 제공하는 코어 패키지에 의존하면서 패키지 간 의존 관계가 복잡해졌고, 문제가 발생했을 때 원인을 추적하기도 어려웠다.

사용자는 패키지마다 개별적으로 초기화를 수행해야 했으며, 초기화 순서도 신경 써야 했다. 이러한 구조는 SDK를 연동하는 데 진입 장벽이 되었다.

연동 과정에서 비슷한 실수가 반복되었고, 이를 확인하고 안내하는 불필요한 CS에 자원이 소모되는 것도 문제였다.

## 인터페이스 설계 및 디자인

```text
Game Code
    |
    v
PercentSDK.Instance  <---  PercentSDKOptions
    |                          ^
    |                          |
    |                  PercentSdkOptions.asset
    |
    +-- AuthProvider
    +-- ConsentProvider
    +-- AnalyticsProvider
    +-- AdsProvider
    +-- PaymentProvider       (현재 초기화 제외)
    +-- CommunityProvider
    +-- PercentUIProvider
```

```text
Tools > Percent SDK
    |
    v
PercentSdkOptions.asset
    |
    | InitAsync(): Load()
    v
ScriptablePercentSDKOptions
    |
    | ToPercentSDKOptions()
    v
PercentSDKOptions
    |
    v
내부 서비스 구성 및 초기화
```
