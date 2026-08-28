+++
title = "Keychain access group이 바뀌자 UUID가 초기화됐다"
date = 2026-08-28T16:00:00+09:00
tags = ["ios", "unity", "keychain", "native"]
categories = ["iOS"]
summary = "빌드 파이프라인을 교체하면서 keychain access group 형식이 바뀌었고, 그 결과 저장해둔 UUID를 읽지 못하게 됐다. 원인 추적과 마이그레이션 대응, 그리고 그 과정에서 만난 identifierForVendor 삽질 기록."
draft = false
+++

빌드 파이프라인을 교체한 뒤 UUID가 바뀐다는 얘기가 나왔다. 코드는 그대로였다. 바뀐 건 keychain access group 형식이었다.

기존에는 `(teamid).*` 형태를 쓰고 있었는데 `teamid.bundleid` 로 바뀌었고, 그 순간부터 이전에 저장해둔 UUID를 못 읽어서 새 값이 발급되고 있었다.

### 내 소스에는 그 설정이 없었다

이상한 건 access group을 바꾼 기억이 없다는 것이었다. 그럴 만도 했다. **프로젝트의 Unity 빌드 포스트프로세스에는 `keychain-access-groups` 를 정의하는 코드가 아예 없었다.**

실제로 그 값을 넣고 있던 건 빌드머신이었다. CI 단계에서 entitlements를 임의로 만들어 주입했고, 그 안에서 와일드카드로 서명하고 있었다. 내 쪽 소스만 봐서는 알 수 없는 영역이었다. 나에게는 블랙박스였다.

그러다 iOS Universal Link를 붙이게 됐다. Associated Domains는 **코드사이닝된 entitlements에 들어가 있어야** 동작한다. entitlements를 CI에 맡겨둔 채로는 넘어갈 수가 없었고, 직접 구성해야 했다. 그러면서 앱 전용으로 잡았다.

그게 전부였다. 나는 없던 설정을 새로 정의했다고 생각했는데, 실제로는 **CI가 뒤에서 넣어주던 와일드카드 그룹을 앱 전용 그룹으로 갈아끼운 것**이었다. 저장 위치가 옮겨갔고, 이전 위치의 값은 지워지지도 않은 채 그냥 조회 대상에서 빠졌다.

## 왜 그룹이 바뀌면 값을 잃는가

keychain 아이템은 access group 단위로 격리된다. 그래서 access group이 바뀌면 이전 그룹에 저장한 아이템은 조회되지 않는다. 코드 입장에서는 "저장한 적이 없는 것"과 구분이 안 된다. 그러니 조용히 새 UUID를 만들어 저장하게 된다.

여기서 중요한 건 **access group을 명시하지 않아도 그룹이 없는 게 아니라는 점**이다. entitlements에 keychain access group을 설정하지 않으면 기본 저장소가 `teamid.bundleid` 가 된다. 즉 설정을 지우는 것도 그룹을 바꾸는 행위다.

```xml
<!-- 전: CI가 빌드 단계에서 만들어 주입하던 entitlements.
     프로젝트 소스에는 이 내용이 없었다. -->
<key>keychain-access-groups</key>
<array>
  <string>$(AppIdentifierPrefix)*</string>
</array>

<!-- 후: Universal Link를 붙이려고 직접 구성한 entitlements.
     associated-domains를 넣으면서 keychain 쪽도 앱 전용으로 잡혔다. -->
<key>com.apple.developer.associated-domains</key>
<array>
  <string>applinks:내-도메인</string>
</array>
<key>keychain-access-groups</key>
<array>
  <string>$(AppIdentifierPrefix)$(CFBundleIdentifier)</string>
</array>
```

`.*` 와일드카드 그룹을 쓰고 있었다는 건, 같은 team prefix를 가진 여러 프로젝트가 **같은 UUID를 공유하고 있었다**는 뜻이기도 했다. 그것도 아무도 명시적으로 정한 적 없이 그렇게 되어 있었다. 이게 의도한 동작이었는지부터 확인해야 했다.

## 확인해야 했던 것들

원인 가설은 금방 세웠지만, 대응 로직을 짜기 전에 확인해야 할 게 여러 개였다.

- iOS에서 UUID가 실제로 `teamid.*` 키체인에 저장되고 있는가
- 같은 `teamid.*` 키체인을 쓰는 다른 프로젝트가 정말 같은 UUID를 쓰고 있는가
- access group을 바꾸면 기본 keychain 저장소 위치가 바뀌는가
- access group 미설정 시 기본 저장소가 `teamid.bundleid` 인가
- entitlements에 `teamid.bundleid` 와 `teamid.*` 를 **동시에** 등록하면 양쪽 모두 접근 가능한가
- `loadUuid` 가 두 번 호출되는 경로가 있는데 이게 의도된 것인가

마지막 두 개가 대응 설계를 갈랐다. 두 그룹을 동시에 등록해 양쪽에 접근할 수 있다면, 옛 그룹에서 읽어 새 그룹으로 옮기는 마이그레이션이 성립한다.

## 대응은 두 갈래로 나눴다

이미 UUID가 바뀌어버린 프로젝트와, 아직 파이프라인을 교체하지 않아 안 깨진 프로젝트의 상황이 달랐다. 같은 로직으로 덮을 수 없었다.

**아직 이슈가 발생하지 않은 프로젝트**는 마이그레이션 경로를 타게 했다. 새 그룹에 값이 없으면 옛 그룹을 조회해서, 있으면 그 값을 새 그룹에 다시 저장하고 그대로 쓴다. 사용자 입장에서는 아무 일도 일어나지 않는다.

**이미 이슈가 발생한 프로젝트**는 되돌릴 방법이 없다. 옛 그룹의 값이 이미 새 값으로 덮였거나, 애초에 읽지 못한 상태로 새 값이 자리를 잡았기 때문이다. 그래서 여기는 복구 대신 **추적**으로 방향을 잡았다. legacy UUID가 변경된 상황과 저장 실패 상황을 C# 이벤트로 올려서, 이후에 얼마나 발생하는지 트래킹할 수 있게 했다.

복구 못 하는 걸 복구하려고 붙잡는 대신 관측 가능하게 만드는 쪽을 택한 셈이다.

## 같이 정리한 것들

원인을 파다 보니 주변이 같이 지저분했다.

**entitlements가 잘못 생성되던 문제.** `XcodeOption.cs` 를 제거하고 iOS post build 구조를 개편했다. 이번 사고가 정확히 그 형태였다. CI와 프로젝트 양쪽이 같은 파일을 만들고 있었고, 어느 쪽이 최종적으로 반영됐는지는 산출물을 까봐야 알 수 있었다. access group처럼 값 하나가 데이터 유실로 직결되는 설정에서는 그게 그대로 사고가 된다. 생성 주체를 한 곳으로 모으는 게 기능 추가보다 먼저였다.

**Team ID / AppIdentifierPrefix / bundle identifier 처리.** 이 세 값을 조합해 access group 문자열을 만드는데, 처리 로직이 어긋나면 존재하지 않는 그룹을 조회하게 된다. 보정해서 식별 안정성을 올렸다.

**네이티브 인터페이스.** UUID 로드/저장 인터페이스를 단순화하고, 저장 실패 상태를 C# 쪽으로 전달하도록 바꿨다. 이전에는 저장이 실패해도 호출한 쪽에서 알 방법이 없었다.

**문자열 전달과 중복 호출.** iOS 네이티브 문자열 전달을 Unity 마샬링 기반으로 정리해 메모리 관리를 개선했고, `loadUuid` 가 2회 호출되던 구조를 1회로 줄였다. 두 번 호출되는 동안 상태가 달라질 수 있는 구조 자체가 불안했다.

## identifierForVendor는 생각보다 잘 안 바뀐다

패키지 안에서 UUID를 `SystemInfo.deviceUniqueIdentifier` 로 만들고 있었는데, iOS에서는 내부적으로 `identifierForVendor` 를 쓴다.

앱을 재설치하면 새 값이 나올 거라고 생각하고 확인해봤는데, **동일한 값이 나왔다.**

공식 문서에 따르면 이렇다.

**반드시 변경되는 경우**

- 같은 벤더의 앱을 **모두 삭제**한 뒤 해당 벤더의 앱을 재설치할 때
- **Xcode**로 테스트 빌드를 설치할 때
- **Ad-hoc 배포**로 앱을 설치할 때
- App Store를 통해 배포된 앱을 **다른 개발자 계정으로 이전(Transfer)** 할 때 — 벤더 식별 기준이 개발자 identity이기 때문

**변경될 수 있는 경우 (비공식/실측)**

- TestFlight 빌드 → App Store 빌드로 업데이트할 때. App Store 배포 전 Apple이 앱을 재서명하기 때문이라고 알려져 있다.
- iOS 베타 버전으로 OS를 업그레이드할 때 변경된 사례가 보고된 적 있다. 정식 동작은 아니다.

**변경되지 않는 경우**

- 같은 벤더의 앱이 **1개라도 기기에 남아 있는 동안**은 앱 업데이트나 OS 정식 업그레이드를 해도 유지된다

> UUID 유지의 기준은 "해당 벤더의 앱이 기기에 하나라도 설치되어 있느냐"다. 모든 앱이 삭제되면 다음 설치 시 새로 발급된다.

핵심은 같은 vendor의 앱이 기기에 하나도 안 남는 상태가 된 뒤 재설치될 때 바뀐다는 것이다. 그런데 내 경우엔 Xcode Development로 재설치했음에도 값이 유지됐다. 문서상으로는 Xcode 설치 시 변경되는 쪽에 해당하는데 그렇지 않았고, 이 부분은 아직 명확히 설명하지 못했다. 삭제 없이 덮어쓰기 설치가 된 것인지, 같은 벤더의 다른 앱이 기기에 남아 있었는지 더 봐야 한다.

어느 쪽이든 결론은 같다. **이 값은 저장해두더라도 언제든 바뀔 수 있다고 가정해야 한다.** 바뀌었을 때 앱이 조용히 다른 사용자로 취급하는 대신, 바뀌었다는 사실을 감지하고 처리할 수 있어야 한다. 이번에 legacy UUID 변경을 이벤트로 올리게 만든 이유가 그것이다.

## 정리

돌아보면 사고의 원인은 UUID 생성 로직이 아니라 **저장 위치를 결정하는 설정**이었다. 코드 한 줄 안 바뀌었는데 데이터가 사라진 이유가 거기 있었다.

그리고 그 설정은 내 소스에 없었다. 없다는 건 안 쓴다는 뜻이 아니라, **누군가 대신 넣어주고 있다**는 뜻일 수 있다. 그 상태에서 그 값을 처음 명시하는 순간이 곧 변경 시점이 된다. 새로 정의하는 것처럼 보이지만 실제로는 덮어쓰는 것이다.

다시 겪는다면 이 순서로 볼 것 같다.

| 확인 | 이유 |
|---|---|
| entitlements의 access group 값이 빌드 산출물에서 실제로 무엇인지 | 소스와 최종 산출물이 다를 수 있다 |
| 내 소스에 없는 설정을 빌드 산출물이 갖고 있는지 | 없다는 게 안 쓴다는 뜻은 아니다 |
| access group을 누가 생성/수정하는지 한 곳으로 모였는지 | 여러 곳에서 건드리면 추적이 안 된다 |
| 옛 그룹과 새 그룹을 동시에 등록해 읽을 수 있는지 | 마이그레이션 가능 여부가 여기서 갈린다 |
| 저장 실패가 호출한 쪽에 전달되는지 | 조용히 실패하면 원인 추적이 불가능하다 |
| 값이 바뀌었을 때 감지할 수단이 있는지 | 복구 못 하는 케이스는 최소한 관측 가능해야 한다 |

기기 식별자를 영속 값처럼 다루는 코드는 대체로 이런 식으로 한 번은 물린다.

## 참조

- [Unity — Plugins for iOS](https://docs.unity3d.com/2020.1/Documentation/Manual/PluginsForIOS.html)
- [Unity — SystemInfo.deviceUniqueIdentifier](https://docs.unity3d.com/6000.3/Documentation/ScriptReference/SystemInfo-deviceUniqueIdentifier.html)
- [Apple — UIDevice.identifierForVendor](https://developer.apple.com/documentation/uikit/uidevice/identifierforvendor)

이 사고의 계기가 된 Universal Link 적용 자체도 따로 앓았다. 그 얘기는 [Unity 딥링크 트러블슈팅]({{< ref "unity-deeplink-universal-link-troubleshooting" >}})에 적었다.
