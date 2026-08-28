+++
title = "Unity 딥링크 트러블슈팅: 젠킨스 빌드에서만 Universal Link가 죽는 이유"
date = 2026-08-28T16:00:00+09:00
tags = ["unity", "ios", "deeplink", "universal-link", "fastlane", "appsflyer"]
categories = ["iOS"]
summary = "커스텀 스킴만 지원하던 Unity 딥링크 패키지에 App Link / Universal Link를 추가했다. Xcode GUI 빌드는 잘 되는데 젠킨스 CLI 빌드만 원링크가 동작하지 않았고, 범인은 fastlane resign이었다. 그 추적 과정을 기록한다."
draft = false
+++

## 들어가며

사내 Unity Deeplink 패키지는 지금까지 커스텀 스킴(URI 스킴) 방식만 제공하고 있었다. 그런데 커스텀 스킴은 브라우저나 메신저에서 바로 열리지 않는 경우가 많고, 마케팅 쪽에서도 도메인 기반 링크를 원했기 때문에 Android(App Link), iOS(Universal Link) 방식을 추가해 달라는 요청이 들어왔다.

작업 자체는 두 갈래였다.

- Android — App Link (intent-filter + `assetlinks.json`) 추가
- iOS — Universal Link (Associated Domains + `apple-app-site-association`) 추가

도메인 검증 파일을 올릴 테스트 서버를 직접 구축하는 대신, 실서비스에서 쓰게 될 AppsFlyer OneLink로 테스트하기로 했다. OneLink가 `apple-app-site-association`(AASA)과 `assetlinks.json` 호스팅을 대신해 주기 때문이다.

결론부터 말하면 **구현은 문제가 없었다.** 문제는 빌드 파이프라인 끝자락, fastlane의 resign 단계에서 entitlements가 통째로 날아가고 있었다는 것. 거기까지 가는 과정을 순서대로 남긴다.

## 증상: Xcode 빌드는 되고, 젠킨스 빌드는 안 된다

테스트는 이렇게 진행했다.

1. 운영 중인 게임(운빨존많겜) iOS 빌드로 AppsFlyer OneLink 테스트 → **원링크 접근 시 앱이 열리지 않음**
2. 같은 프로젝트를 로컬에서 Xcode 빌드로 설치해서 테스트 → **정상 동작**
3. 패키지 테스트용 앱(Packages App)으로 원링크 테스트 → **Android, iOS 모두 정상 동작**

같은 코드인데 설치 경로에 따라 결과가 갈렸다. Xcode GUI로 빌드해서 설치한 앱은 Universal Link가 잘 열리고, **젠킨스 CLI로 빌드·배포된 ipa만 동작하지 않았다.** 코드 문제가 아니라 빌드 파이프라인 어딘가의 문제라는 뜻이다.

## 1단계: OneLink 자체가 유효한지 확인

앱을 의심하기 전에 링크부터 확인했다. Universal Link는 iOS가 도메인의 `apple-app-site-association` 파일을 받아서 앱과 도메인을 연결하는 구조라, 이 파일에 우리 앱의 appID(팀ID + 번들ID)가 제대로 들어있는지가 첫 번째 체크포인트다.

OneLink 도메인의 AASA를 직접 받아 확인했고, [AppsFlyer OneLink 대시보드](https://hq1.appsflyer.com/onelink)의 템플릿 설정도 점검했다. 여기는 문제가 없었다. Xcode 빌드에서 잘 동작한다는 것 자체가 서버 쪽 설정은 정상이라는 방증이기도 하다.

## 2단계: Unity PostBuildProcess 확인

다음 의심 대상은 Unity 빌드 후처리였다. Universal Link가 동작하려면 Xcode 프로젝트에 Associated Domains capability가 켜져 있고, entitlements 파일에 `applinks:` 도메인이 들어가고, 그 entitlements가 빌드 설정(`CODE_SIGN_ENTITLEMENTS`)에 연결되어 있어야 한다.

처음에는 entitlements 파일은 생성되는데 코드사인에 연결이 안 된 게 아닌지 의심해서, 빌드 설정을 명시적으로 추가했다.

```csharp
// CODE_SIGN_ENTITLEMENTS 빌드 설정 명시적 추가
proj.SetBuildProperty(targetGuid, "CODE_SIGN_ENTITLEMENTS", "Unity-iPhone.entitlements");
```

이 김에 entitlements를 직접 읽고 쓰던 기존 구현을 Unity가 제공하는 `ProjectCapabilityManager`로 갈아엎었다.

```
Before (직접 조작)
- entitlements 파일 수동 읽기/쓰기
- associated-domains 중복 수동 체크
- CODE_SIGN_ENTITLEMENTS 수동 설정
- pbxproj 수동 수정

After (ProjectCapabilityManager)
- entitlements 파일 자동 생성/관리 ✅
- 중복 도메인 자동 방지 ✅
- CODE_SIGN_ENTITLEMENTS 자동 설정 ✅
- pbxproj Capability 자동 등록 ✅
- 코드량 대폭 감소 ✅
```

개선 후 Xcode에서 Export 해보면 entitlements가 잘 적용되어 있었다. 그런데도 젠킨스 빌드는 여전히 동작하지 않았다.

## 3단계: 최종 산출물(ipa)을 직접 까서 검증

여기서부터는 추측 대신 산출물을 직접 확인하기로 했다. ipa는 zip이므로, 다운로드해서 unzip하면 `Payload/` 아래의 `.app`을 얻을 수 있고, `codesign`으로 **실제 서명에 박힌 entitlements**를 볼 수 있다.

```bash
$ codesign -d --entitlements - TSPackages.app

[Dict]
    [Key] application-identifier
    [Value]
        [String] {teamID}.com.percent.ios.percentpackages
    [Key] aps-environment
    [Value]
        [String] production
    [Key] com.apple.developer.applesignin
    [Value]
        [Array]
            [String] Default
    [Key] com.apple.developer.associated-domains
    [Value]
        [String] *   <- 값이 비어있다는 건 적용되지 않았다는 의미
```

`associated-domains`가 와일드카드(`*`)다. 도메인 배열이 있어야 할 자리에 아무것도 없다는 뜻이다. 정상적인 형태가 어떤 건지 확인하려고 앱스토어에서 타사 게임을 받아 같은 명령을 돌려봤다.

```bash
$ codesign -d --entitlements - Kingshot.app
Executable=/Applications/Kingshot.app/Wrapper/kingshot.app/kingshot
[Dict]
    [Key] application-identifier
    [Value]
        [String] 4V98RYJDSR.com.run.tower.defense
    [Key] aps-environment
    [Value]
        [String] production
    [Key] com.apple.developer.applesignin
    [Value]
        [Array]
            [String] Default
    [Key] com.apple.developer.associated-domains
    [Value]
        [Array]
            [String] applinks:appleunilink.centurygame.com   <- 이렇게 나와야 정상
```

이걸로 증상이 명확해졌다. **젠킨스에서 나온 ipa는 서명 단계에서 `associated-domains`를 잃어버리고 있다.** Universal Link는 서명된 entitlements를 기준으로 동작하므로, Xcode 프로젝트에 아무리 잘 설정되어 있어도 최종 서명에 없으면 무용지물이다.

## 4단계: 젠킨스 빌드 파이프라인 추적

젠킨스의 iOS 빌드는 이런 순서로 흘러간다.

```
유니티 빌드 → Xcode 프로젝트/Workspace 추출 → fastlane run build_app → fastlane run resign
```

빌드 로그를 단계별로 훑다가 마지막의 **리사이닝 과정에서 entitlements가 날아가는 게 아닐까** 의심이 들었다. 그래서 리사이닝 이전의 원본 ipa를 백업해서 추출해봤는데 — 이 시점(빌드 후처리 개선 전)에는 원본에서도 동일하게 적용되어 있지 않았다.

다음으로 fastlane 실행 파라미터를 뜯어봤다. 체크리스트를 순서대로 소거해 보면 남는 건 프로비저닝 프로파일뿐이었다. Apple Developer 센터의 프로비저닝 프로파일에는 ad-hoc과 App Store 타입만 존재하는데, 실제 fastlane 실행 파라미터에는 **development로 설정되어 있었다.** 존재하지도 않는 development 프로파일로 서명을 시도하면서 자동으로 와일드카드 프로파일이 잡혔고, 그래서 `associated-domains`가 `*`로 덮어씌워지는 게 아닐까 — 라는 가설을 세웠다.

빌드 커맨드라인은 건드릴 수 없는 상황이라, 대신 development 타입 프로비저닝 프로파일을 새로 생성해 주고 다시 빌드해서 테스트했다. **여전히 적용되지 않았다.** 가설이 틀렸다.

그런데 이번에는, 후처리 개선이 반영된 상태에서 리사이닝 이전의 원본 ipa를 다시 확인해봤더니:

```bash
$ codesign -d --entitlements - TSPackages.app
Executable=/Users/111percent/Documents/Payload/TSPackages.app/TSPackages
[Dict]
    [Key] application-identifier
    [Value]
        [String] {teamID}.com.percent.ios.percentpackages
    [Key] com.apple.developer.applesignin
    [Value]
        [Array]
            [String] Default
    [Key] com.apple.developer.associated-domains
    [Value]
        [Array]
            [String] applinks:tslink.onelink.me   <- 적용되었다!
```

**원본 ipa에는 들어있다.** 즉 유니티 빌드와 `build_app`까지는 정상이고, 범인은 마지막 `resign` 단계다.

## 5단계: resign 로그에서 범인 확정

resign 단계의 로그를 확인했다.

```
[2026-03-13T07:42:43.216Z] + fastlane run resign
[2026-03-13T07:42:44.090Z] [16:42:43]: --- Step: resign ---
[2026-03-13T07:42:44.090Z] /opt/homebrew/Cellar/fastlane/2.232.2/libexec/gems/fastlane-2.232.2/sigh/lib/assets/resign.sh
    build_ios_percentpackages/com.percent.ios.percentpackages_dev_1.3.5-2.ipa
    3498C2EB80897DF74D1927164674552E6B879192
    -p /Users/build/jenkins-remote/workspace/****/ds/ts/core/test/application/build-scripts/includes/provisioningProfiles/adhocpercentpackages.mobileprovision
    build_ios_percentpackages/com.percent.ios.percentpackages_dev_1.3.5-2.ipa
```

fastlane의 resign은 내부적으로 sigh의 `resign.sh`를 호출하는데, 이때 앱의 기존 entitlements를 유지하려면 `use_app_entitlements: true`가 전달되어 `-e` 옵션(entitlements 파일 지정)이 붙어야 한다. 그런데 로그에는 프로비저닝 프로파일을 넘기는 `-p` 옵션만 보이고 `-e`가 없다.

**`use_app_entitlements: "true"`가 실제로는 적용되지 않고 있었던 것이다.** entitlements 지정 없이 리사인되면 프로비저닝 프로파일 기준으로 서명이 다시 이뤄지고, 그 과정에서 `associated-domains`가 와일드카드로 덮어씌워진다. Xcode GUI 설치본과 젠킨스 배포본의 차이도 이걸로 설명된다 — Xcode 설치본은 resign을 거치지 않는다.

## 정리

같은 증상(Universal Link가 특정 빌드에서만 안 됨)을 만난다면 이 순서로 소거하는 걸 권한다.

| 순서 | 확인 항목 | 도구 |
|---|---|---|
| 1 | AASA / OneLink 템플릿이 유효한가 | AASA 직접 조회, OneLink 대시보드 |
| 2 | Xcode 프로젝트에 entitlements가 생성·연결되는가 | Unity PostBuildProcess, `ProjectCapabilityManager` |
| 3 | **최종 ipa의 서명에** `associated-domains`가 있는가 | `codesign -d --entitlements -` |
| 4 | 파이프라인 중간 산출물(리사인 전 ipa)에는 있는가 | 원본 ipa 백업 후 동일 검증 |
| 5 | resign 시 entitlements 유지 옵션이 실제로 전달되는가 | resign 로그에서 `-e` 옵션 유무 |

배운 것 몇 가지.

- **"설정했다"와 "서명에 반영됐다"는 다르다.** Universal Link 문제는 Xcode 프로젝트가 아니라 최종 ipa의 `codesign -d --entitlements -` 출력으로 판단해야 한다. 이 명령 하나면 추측 대신 사실을 볼 수 있다.
- **파이프라인 문제는 중간 산출물을 끊어서 확인하면 빠르다.** 원본 ipa와 리사인 후 ipa를 비교하는 순간 용의자가 한 단계로 좁혀졌다.
- **fastlane resign을 쓴다면 `use_app_entitlements`가 실제 `resign.sh` 호출에 `-e`로 반영되는지 로그로 확인하자.** 옵션을 적어둔 것과 전달된 것은 다를 수 있다.
- Unity 쪽 entitlements 처리는 직접 파일을 조작하는 것보다 `ProjectCapabilityManager`를 쓰는 편이 안전하다. 중복 방지와 `CODE_SIGN_ENTITLEMENTS` 연결까지 알아서 해준다.

## 참조

- [AppsFlyer OneLink를 이용한 DeepLinking](https://velog.io/@gudrmsglgl/AppsFlyer-OneLink%EB%A5%BC-%EC%9D%B4%EC%9A%A9%ED%95%9C-DeepLinking#-%EB%A7%81%ED%81%AC%EC%97%90-%EB%8C%80%ED%95%9C-%EA%B8%B0%EB%B3%B8-%EA%B5%AC%EC%A1%B0)
- [iOS Universal Link 정리](https://cording-cossk3.tistory.com/308)
- [OneLink guide — AppsFlyer](https://support.appsflyer.com/hc/en-us/articles/115005248543-OneLink-guide)
- [iOS Initial Setup — AppsFlyer Dev Hub](https://dev.appsflyer.com/hc/docs/dl_ios_init_setup)
- [Unified Deep Linking — AppsFlyer Dev Hub](https://dev.appsflyer.com/hc/docs/unifieddeeplink)
- [Create a OneLink template — AppsFlyer](https://support.appsflyer.com/hc/en-us/articles/207032246-Create-a-OneLink-template)
- [OneLink troubleshooting and FAQ — AppsFlyer](https://support.appsflyer.com/hc/en-us/articles/360014821438-OneLink-troubleshooting-and-FAQ)
- [appsflyer-unity-plugin DeepLink Integration](https://github.com/AppsFlyerSDK/appsflyer-unity-plugin/blob/master/docs/DeepLinkIntegrate.md)
