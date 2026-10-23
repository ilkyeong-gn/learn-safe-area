# 하이브리드웹 safe area 실무 문제 사례 모음

- 티켓: [#8](https://github.com/ilkyeong-gn/learn-safe-area/issues/8) (지도 [#1](https://github.com/ilkyeong-gn/learn-safe-area/issues/1))
- 조사일: 2026-10-23
- 가정 시나리오: Flutter `webview_flutter` 네이티브 셸 + Next.js 16(App Router), iOS 16+ / Android 8.0(API 26)+
- 출처 범위: 1차 출처만 사용. WebKit Bugzilla, Chromium·Google Issue Tracker, Chromium 소스 문서, flutter/flutter·flutter/packages·flutter_inappwebview·ionic-team/capacitor·vercel/next.js GitHub 이슈와 PR, 공식 문서.
- Capacitor 사례는 비교용이다. 같은 Android System WebView와 WKWebView 위에서 생기는 문제라서, 네이티브 셸이 Flutter여도 메커니즘은 같다.
- 확인하지 못한 내용은 **미확인**으로 표시했다.

## 먼저 알아둘 배경: Android WebView의 inset 지원 이력

여러 사례의 원인이 Android System WebView의 inset 처리 변경과 이어져 있다. Chromium 공식 문서 [android_webview/docs/insets.md](https://chromium.googlesource.com/chromium/src/+/HEAD/android_webview/docs/insets.md)에 따르면 이력은 다음과 같다.

| WebView 버전 | 변경 |
|---|---|
| M136 | `displayCutout()`·`systemBars()` inset을 `safe-area-inset-*`로 웹 콘텐츠에 전달하기 시작했다. 호환성 문제 때문에 WebView가 **전체 화면일 때만** 전달했다. |
| M139 | `ime()` inset을 지원한다. 키보드가 열리면 Layout Viewport는 그대로 두고 Visual Viewport만 줄인다. |
| M144 | 전체 화면 여부와 관계없이 **모든 WebView**에 inset을 전달한다. 단, WebView 영역이 시스템 바·컷아웃·IME와 실제로 겹칠 때만 0이 아닌 값을 준다. |

이 문서는 opt-out도 안내한다. 앱이 `OnApplyWindowInsetsListener`에서 직접 패딩을 준 inset은 0으로 만들어 넘기라는 것이다(`CONSUMED`도 가능하지만 상태가 바뀔 때 WebView가 갱신 알림을 받지 못하므로 0으로 만드는 쪽을 권장한다).

따라서 같은 웹 코드가 **기기의 WebView 버전에 따라** 0 / 정상 / 이중 패딩으로 다르게 보일 수 있다. 앱 버전이 아니라 Play 스토어로 업데이트되는 WebView 버전이 변수다.

---

## 1. inset이 0으로 들어옴

### 1-1. Android WebView에서 `env(safe-area-inset-*)`가 항상 0px

- **증상**: `viewport-fit=cover`를 선언하고 edge-to-edge로 그려도 `env(safe-area-inset-*)`가 모두 `0px`이다. 상단 헤더가 상태 바 아래로 들어간다.
- **환경**: Android, Android System WebView(2020년 보고 당시 88). 앱 종류 무관.
- **원인**: 오래된 Android WebView는 시스템 inset을 웹 콘텐츠에 전달하지 않았다. 이 지원은 M136(전체 화면 WebView 한정)과 M144(모든 WebView)에 걸쳐 들어왔다.
- **해결책**: WebView 버전이 낮은 기기에서는 CSS만으로 해결할 수 없다. 네이티브 셸이 inset을 직접 처리하거나(Flutter `SafeArea` 등), 셸이 inset 값을 CSS 변수나 JS 브리지로 주입해야 한다. 웹 쪽은 `env(safe-area-inset-top, 0px)`처럼 대체값을 두고, 0일 때도 레이아웃이 깨지지 않게 만든다.
- **출처**: [Chromium 40699457 "safe-area-inset-* values are always 0px in webview"](https://issues.chromium.org/issues/40699457) (Mobile>WebView, `Merged-140` 라벨이 붙어 있다. 정확한 해결 상태와 버전은 트래커 본문을 로그인 없이 읽을 수 없어 **미확인**), [Chromium insets.md](https://chromium.googlesource.com/chromium/src/+/HEAD/android_webview/docs/insets.md)

### 1-2. WebView < 140 기기에서 상단 inset 0, edge-to-edge도 적용 안 됨

- **증상**: `EdgeToEdge.enable()`과 `viewport-fit=cover`를 썼는데도 WebView < 140 기기에서는 콘텐츠가 상태 바와 내비게이션 바 사이에만 그려지고, `env(safe-area-inset-top)`이 `0px`이다. 같은 앱이 최신 WebView 기기에서는 정상이다.
- **환경**: Android 10~17(API 29~37), Android System WebView 91~154, Capacitor 8.5.3.
- **원인**: Capacitor SystemBars 플러그인이 WebView 메이저 버전으로 분기하는데(140 미만은 다른 경로), 그 분기에서 edge-to-edge와 top inset 처리가 빠진다고 보고됐다. 분기가 필요한 근본 이유는 1-1의 WebView 버전별 지원 차이다.
- **해결책**: 미해결(OPEN). 하이브리드웹 관점의 교훈은 "**네이티브 셸이 WebView 버전별 분기를 하면, 웹은 모든 분기 결과를 테스트해야 한다**"이다.
- **출처**: [ionic-team/capacitor#8651](https://github.com/ionic-team/capacitor/issues/8651)

### 1-3. WKWebView에서 첫 렌더 때 inset이 0이었다가 뒤늦게 채워짐

- **증상**: 앱 실행 직후나 백그라운드 복귀 직후 `env(safe-area-inset-*)`가 0이었다가 1초 안에 34px 등으로 바뀐다. 하단 고정 버튼이 툭 튀어 오른다. `load`·`DOMContentLoaded` 시점에도 아직 0이다.
- **환경**: iOS 12~16(보고자 기준), WKWebView. 같은 페이지를 Safari에서 열면 처음부터 값이 있다.
- **원인**: WKWebView가 safe area 값을 페이지 로드 뒤 임의의 시점에 설정한다. 상태 NEW(미해결), 내부 rdar://46193462.
- **해결책**: JS로 inset을 한 번만 읽어 고정하는 패턴을 피하고, CSS `env()`를 직접 써서 값이 바뀌면 자동으로 다시 계산되게 둔다. 레이아웃 점프가 문제라면 하단 고정 요소에 짧은 트랜지션을 거는 것이 한 방법이다(보고된 공식 해결책은 없음).
- **출처**: [WebKit 191872](https://bugs.webkit.org/show_bug.cgi?id=191872), 비슷한 보고: [WebKit 183106](https://bugs.webkit.org/show_bug.cgi?id=183106)

### 1-4. 네이티브 플러그인이 viewport 메타를 덮어써서 `viewport-fit=cover`가 사라짐

- **증상**: `flutter_inappwebview`에서 `supportZoom: false`로 두면 `env(safe-area-inset-bottom)`이 동작하지 않는다.
- **환경**: iOS 17.0, iPhone 15 Pro, Flutter 3.16.3, flutter_inappwebview 6.0.0-beta.31.
- **원인**: 줌을 막는 플러그인 스크립트(`SupportZoomJS`)가 viewport 메타 태그를 통째로 교체하면서, 페이지가 넣은 `viewport-fit=cover`를 지운다. iOS에서는 `viewport-fit=cover`가 없으면 inset이 0이다.
- **해결책**: `supportZoom: true`로 두고, 웹 쪽 viewport 메타에서 `maximum-scale=1, user-scalable=0`으로 줌을 막는다(이슈 코멘트의 우회책). 일반화하면 **셸이 주입하는 user script가 viewport 메타를 건드리는지 확인**해야 한다.
- **출처**: [pichillilorenzo/flutter_inappwebview#1911](https://github.com/pichillilorenzo/flutter_inappwebview/issues/1911)

### 1-5. `viewport-fit` 해석이 iOS와 Android에서 다름

- **증상**: iOS WKWebView는 `viewport-fit=cover`가 아니면 inset을 0으로 주는데, Android WebView는 `viewport-fit=auto`·`contain`에서도 0이 아닌 값을 준다. 같은 CSS가 플랫폼마다 다르게 동작한다.
- **환경**: Android System WebView vs iOS WKWebView. 보고 2025-02.
- **원인**: 두 엔진의 구현 차이. WebKit 쪽에도 반대 방향의 요청(inset을 `viewport-fit`과 무관하게 항상 노출해 달라는 요청, 또는 `cover`가 아니면 좌우 inset을 0으로 하라는 요청)이 열려 있다.
- **해결책**: Next.js에서는 `export const viewport = { viewportFit: 'cover' }`로 명시한다. Next.js `Viewport` 타입에 `viewportFit?: 'auto' | 'cover' | 'contain'` 필드가 있다(공식 문서 페이지에는 이 필드 설명이 없고 소스 타입에만 있음). inset을 쓰는 웹은 항상 `cover`를 선언하고, inset을 쓰지 않을 곳은 셸이 막는 쪽으로 정리한다. Google Issue Tracker 건의 처리 상태는 **미확인**.
- **출처**: [Google Issue Tracker 396827865 "Inconsistent safe-area-inset-* and viewport-fit Behavior on Android WebView vs. iOS WKWebView"](https://issuetracker.google.com/issues/396827865), [WebKit 190771](https://bugs.webkit.org/show_bug.cgi?id=190771), [WebKit 272779](https://bugs.webkit.org/show_bug.cgi?id=272779), [vercel/next.js#46498](https://github.com/vercel/next.js/issues/46498), [Next.js 소스 `extra-types.ts`](https://github.com/vercel/next.js/blob/canary/packages/next/src/lib/metadata/types/extra-types.ts)

---

## 2. 이중 패딩

### 2-1. Flutter가 SafeArea/AppBar로 이미 피했는데 WebView M144+가 inset을 또 보고함

- **증상**: `Scaffold(appBar: ..., body: SafeArea(child: WebViewWidget(...)))` 구조에서 WebView는 상태 바와 겹치지 않는데도, WebView 144 이상 기기에서는 웹이 `env(safe-area-inset-top)`만큼 패딩을 또 준다. 133/134에서는 정상. AppBar를 빼면 최신 WebView는 고쳐지지만 구 WebView에서는 깨진다.
- **환경**: Android 16 Pixel 10, Android System WebView 144.0.7559.132, Flutter 3.41.1, `webview_flutter`.
- **원인**: Flutter는 플랫폼 뷰(WebView)를 화면 전체 크기의 Android View 계층에 올리고 위치를 Flutter 쪽에서 잡는다. 그래서 네이티브 `WindowInsets`가 WebView에 그대로 전달되고, M144부터 WebView가 이를 웹에 넘긴다. Flutter의 `SafeArea`는 Flutter 레이아웃 안의 패딩일 뿐 네이티브 inset을 소비하지 않는다.
- **해결책**: `webview_flutter_android` **4.11.0**에서 `AndroidWebViewController.setInsetsForWebContentToIgnore`가 추가됐다. 내부적으로 `ViewCompat.setOnApplyWindowInsetsListener`로 지정한 inset 타입을 0으로 만든다. 셸이 Flutter 쪽에서 safe area를 처리한다면 이 API로 `systemBars`·`displayCutout`을 무시하게 해야 한다. 반대로 웹이 safe area를 처리하게 하려면 WebView를 화면 끝까지 붙이고 이 API를 쓰지 않는다. PR은 iOS 구현이 이미 비슷한 일을 한다고 설명한다(2-2 참고).
- **출처**: [flutter/flutter#182208](https://github.com/flutter/flutter/issues/182208) (2026-04-09 해결), [flutter/packages#11192](https://github.com/flutter/packages/pull/11192)

### 2-2. (참고) iOS `webview_flutter`는 스크롤 뷰 contentInset 자동 조정을 끈다

- **증상 예방 사례**: WKWebView의 `UIScrollView`는 기본적으로 safe area만큼 `contentInset`을 자동으로 더한다. Flutter가 SafeArea로 피한 뒤에 이것까지 더해지면 이중 여백이 생긴다.
- **환경**: iOS, `webview_flutter_wkwebview`.
- **원인/해결책**: 플러그인 소스가 `scrollView.contentInsetAdjustmentBehavior = .never`로 설정하고, frame이 바뀔 때마다 `adjustedContentInset`을 상쇄하도록 `contentInset`을 음수로 보정한다(주석: "Prevents the contentInsets from being adjusted by iOS and gives control to Flutter."). 즉 `webview_flutter`를 쓰면 iOS에서 **스크롤 뷰 레벨의 자동 여백은 없고**, 웹이 받는 값은 `env()`뿐이다. 직접 WKWebView를 붙이는 셸이라면 같은 설정을 해야 한다.
- **출처**: [flutter/packages `WebViewProxyAPIDelegate.swift`](https://github.com/flutter/packages/blob/main/packages/webview_flutter/webview_flutter_wkwebview/darwin/webview_flutter_wkwebview/Sources/webview_flutter_wkwebview/WebViewProxyAPIDelegate.swift)

### 2-3. edge-to-edge가 아닌 창에서 셸이 inset 변수를 주입

- **증상**: `targetSdk <= 34`이거나 Android 14 이하에서 창이 이미 시스템 바 사이에 있는데도 `--safe-area-inset-top/bottom`이 24px 등으로 주입된다. 이를 읽는 `body` 패딩이 한 번 더 생긴다. 같은 순간 `env(safe-area-inset-*)`는 정확히 `0px`이다.
- **환경**: Android 16 에뮬레이터(WebView 145), Android 13 실기기(WebView 151), Capacitor 8.5.2(8.5.1에서는 정상).
- **원인**: 셸이 시스템 바 높이를 "WebView가 실제로 겹치는지" 확인하지 않고 그대로 주입한 회귀.
- **해결책**: 미해결(OPEN). 교훈: 셸이 inset을 CSS 변수로 주입한다면 **WebView 사각형과 실제로 겹치는 양**만 넘겨야 한다. 웹은 `var(--셸변수, env(...))`처럼 셸 값을 우선하는 패턴을 쓸 때 셸 값이 틀리면 그대로 이중 패딩이 된다.
- **출처**: [ionic-team/capacitor#8623](https://github.com/ionic-team/capacitor/issues/8623)

### 2-4. 하단 탭바 아래 추가 공백(SDK < 35)

- **증상**: 하단 탭바 아래에 제스처 바 높이만큼 빈 공간이 생긴다. SDK 35 이상에서는 정상.
- **환경**: Android API 31·34, Capacitor 8.x SystemBars. 코멘트에 따르면 Android 15+ WebView 144+ 기기에서도 재현.
- **원인**: Android Studio 레이아웃 검사에서 WebView 자체가 화면 끝보다 위에서 끝나 있었다. 셸이 부모 뷰에 패딩을 준 상태에서 웹도 하단 inset을 처리한 것으로 보인다(근본 원인은 이슈에 명시되지 않음: **미확인**).
- **해결책**: 이슈는 COMPLETED로 닫혔으나 해결 PR은 **미확인**. 진단 팁으로 `android:windowBackground`를 빨간색으로 두면 WebView 바깥 공백이 보인다는 코멘트가 유용하다.
- **출처**: [ionic-team/capacitor#8408](https://github.com/ionic-team/capacitor/issues/8408)

---

## 3. 회전·상태 변경 후 값이 갱신되지 않음

### 3-1. 회전 후 `env(safe-area-inset-top)`이 이전 값 유지

- **증상**: 노치 기기에서 세로→가로 회전 후에도 상단 inset 값이 그대로 남아 요소가 엉뚱한 위치에 있다.
- **환경**: iPhone X/XR/XS 계열, WKWebView 및 전체 화면 Safari, `viewport-fit=cover`. 2018년 보고.
- **원인**: WebKit이 회전 시 safe area 갱신을 늦게 하거나 누락. 같은 시기 [WebKit 182304 "WKWebView layout is sometimes wrong after rotation on iPhone X"](https://bugs.webkit.org/show_bug.cgi?id=182304)는 FIXED.
- **해결책**: 2023-03 코멘트에서 "iOS 16.4에서는 기대대로 동작한다"고 보고됐다. 버그 상태는 여전히 NEW이므로 iOS 16 최소 지원 기준에서 완전히 사라졌다고 보기는 어렵다(**미확인**). JS로 값을 캐시하지 말고 `env()`를 직접 쓰는 것이 기본 대응.
- **출처**: [WebKit 191363](https://bugs.webkit.org/show_bug.cgi?id=191363)

### 3-2. 전체 화면 요소에서 inset이 갱신되지 않거나 너무 큼

- **증상**: Element Fullscreen 상태에서 `env(safe-area-inset-*)`가 회전 후에도 갱신되지 않는다. 가로로 회전한 전체 화면에서 상단 safe area가 지나치게 크다.
- **환경**: iOS, WebKit. 2023~2024 보고.
- **원인**: iOS Element Fullscreen이 `viewport-fit`을 존중하지 않았다.
- **해결책**: WebKit에서 FIXED([264012](https://bugs.webkit.org/show_bug.cgi?id=264012), [270923](https://bugs.webkit.org/show_bug.cgi?id=270923)). 수정이 들어간 iOS 버전은 **미확인**.
- **출처**: [WebKit 264209](https://bugs.webkit.org/show_bug.cgi?id=264209)(264012의 중복), [WebKit 264012](https://bugs.webkit.org/show_bug.cgi?id=264012), [WebKit 270923](https://bugs.webkit.org/show_bug.cgi?id=270923)

### 3-3. 전체 화면 종료 후 WebView 뷰포트가 줄어든 채 남음 (iOS 26)

- **증상**: Fullscreen API(예: 360도 뷰어)로 전체 화면에 들어갔다 나오면 WebView 위아래에 흰/검은 띠가 생기고 뷰포트가 줄어든다. 비슷하게 iframe 전체 화면 후 여백이 깨진다는 보고도 있다.
- **환경**: iOS 26, iPhone 17, Capacitor 7.4.4.
- **원인**: **미확인**(OPEN).
- **해결책**: 없음. 웹에서 Fullscreen API를 쓰는 화면은 iOS 26 실기기 회귀 테스트 대상에 넣는다.
- **출처**: [ionic-team/capacitor#8231](https://github.com/ionic-team/capacitor/issues/8231), [ionic-team/capacitor#8054](https://github.com/ionic-team/capacitor/issues/8054)

### 3-4. iOS 26에서 SystemUiMode 변경 후 Flutter SafeArea가 갱신되지 않음

- **증상**: `SystemChrome.setEnabledSystemUIMode(SystemUiMode.immersive)`로 상태 바를 숨겨도 `SafeArea` 상단 패딩이 이전 값으로 남는다. 회전하거나 창 크기를 바꾸면 그제야 갱신된다. iOS 18.6에서는 정상.
- **환경**: iOS 26, Flutter.
- **원인**: **미확인**(OPEN). iOS 26에서 상태 바 숨김에 따른 safe area 변경 알림이 Flutter 엔진에 전달되지 않는 것으로 보인다.
- **해결책**: 없음. 셸이 상태 바를 숨겼다 보였다 하는 화면(동영상 등)이 있으면 iOS 26에서 셸 쪽 inset과 웹 `env()` 값이 서로 어긋나지 않는지 확인한다.
- **출처**: [flutter/flutter#175520](https://github.com/flutter/flutter/issues/175520)

### 3-5. Android 디스플레이 크기 변경 후 edge-to-edge WebView 크기가 맞지 않음

- **증상**: 세로 고정 + `SystemUiMode.edgeToEdge` 앱에서 설정 > 디스플레이 크기를 바꾸고 돌아오면 WebView가 올바르게 리사이즈되지 않는다.
- **환경**: Pixel 5(Android 14), Pixel 3XL(Android 12), Flutter 3.27.2, webview_flutter 4.10.0, webview_flutter_android 4.3.1.
- **원인**: **미확인**(OPEN).
- **해결책**: 없음.
- **출처**: [flutter/flutter#162003](https://github.com/flutter/flutter/issues/162003)

---

## 4. 키보드와 하단 safe area·탭바 겹침

### 4-1. 키보드가 올라와도 `safe-area-inset-bottom`이 남아 키보드 위에 빈 공간

- **증상**: 홈 인디케이터용 하단 패딩이 키보드가 열린 동안에도 남아, 키보드와 콘텐츠 사이에 하얀 공간이 생긴다.
- **환경**: iOS Safari/WKWebView. 2020년 보고(twitter.com 사례), 2025-03까지 핑.
- **원인**: WebKit이 키보드 표시 중에도 safe area inset을 그대로 유지한다. 상태 NEW, rdar://70573100.
- **해결책**: WebKit 수정 없음. 웹에서 키보드 표시 여부를 알아내 하단 inset을 0으로 바꿔야 한다(예: Visual Viewport 높이 변화 감지, 또는 셸이 키보드 상태를 브리지로 전달).
- **출처**: [WebKit 217754](https://bugs.webkit.org/show_bug.cgi?id=217754)

### 4-2. Android에서도 같은 문제: 셸이 키보드 표시 중 하단 inset을 0으로 바꿔 해결

- **증상**: Capacitor SystemBars `insetsHandling: 'css'`에서 키보드가 열리면 `--safe-area-inset-bottom`이 남아 키보드 위에 여분 공간이 생긴다. 보고자는 `keyboardWillShow`에서 클래스를 붙여 변수를 0으로 덮는 우회책을 썼다. 다른 사용자는 `calc(env(safe-area-inset-bottom) + env(keyboard-inset-height))`로 두 값을 더해 버렸다.
- **환경**: Android, Capacitor 8.0.x.
- **원인**: 하단 inset과 키보드 inset을 독립적으로 더한 것. 키보드가 열리면 내비게이션 바 영역은 키보드에 가려지므로 하단 inset은 의미가 없다.
- **해결책**: Capacitor가 [PR #8280](https://github.com/ionic-team/capacitor/pull/8280)에서 키보드 표시 중 하단 inset을 0으로 주입하도록 고쳤다(8.0.1 이후 포인트 릴리스). 하이브리드웹 원칙: **하단 여백 = max(하단 safe area, 키보드 높이)**이지 합이 아니다.
- **출처**: [ionic-team/capacitor#8287](https://github.com/ionic-team/capacitor/issues/8287)

### 4-3. iOS 17.2부터 입력 포커스 이동 시 화면 상단에 정체불명 영역 (Flutter + WKWebView)

- **증상**: `Scaffold(body: SafeArea(child: WebView))`에서 입력창 사이로 포커스를 옮기면 화면 위쪽에 빈 영역이 생긴다. 높이는 키보드 높이와 일치한다. Android와 iOS 17.2 미만에서는 정상.
- **환경**: iOS 17.2~17.3+, Flutter 3.16, `webview_flutter`와 `flutter_inappwebview` 모두.
- **원인**: Flutter 팀 분석 결과 엔진의 키보드 inset 값은 정상이고, Flutter가 키보드만큼 WebView를 줄이는 것과 WKWebView 내부 로직이 겹쳐 생기는 것으로 추정. 확정 원인 **미확인**(OPEN).
- **해결책**: `Scaffold(resizeToAvoidBottomInset: false)` + `SafeArea(maintainBottomViewPadding: true)`로 Flutter가 WebView를 리사이즈하지 않게 하면 사라진다. 대신 WebView가 줄어들지 않으므로 하단 버튼을 키보드 위로 올리는 일은 웹이 Visual Viewport로 직접 해야 한다.
- **출처**: [flutter/flutter#140501](https://github.com/flutter/flutter/issues/140501), [pichillilorenzo/flutter_inappwebview#1947](https://github.com/pichillilorenzo/flutter_inappwebview/issues/1947)

### 4-4. 키보드 높이만큼 WebView를 두 번 줄임 (Android 10 + WebView ≥ 140)

- **증상**: Android 10에서 입력창을 탭하면 WebView가 키보드 높이의 두 배만큼 줄어 화면 전체가 사라지고 창 배경색만 보인다.
- **환경**: Android 10(API 29), WebView ≥ 140, `viewport-fit=cover`, Capacitor SystemBars.
- **원인**: API < 30에서는 `adjustResize`로 창이 이미 키보드만큼 줄어든 상태인데, AndroidX `ime()` 호환 값이 여전히 키보드 높이를 돌려줘서 셸이 부모 뷰에 키보드 높이 패딩을 한 번 더 줬다. WebView가 140 이상으로 업데이트된 Android 10 기기에서만 새로 나타났다(Android 9는 WebView 138에서 멈춤).
- **해결책**: 미해결(OPEN). 교훈: **키보드 리사이즈 주체는 하나만** 둔다(창 `adjustResize`, Flutter `resizeToAvoidBottomInset`, WebView M139+의 Visual Viewport 리사이즈 중 하나).
- **출처**: [ionic-team/capacitor#8601](https://github.com/ionic-team/capacitor/issues/8601)

### 4-5. WebView M139 이후 키보드가 resize 이벤트를 발생시켜 포커스가 풀림

- **증상**: 입력창을 포커스하면 키보드가 열렸다가 바로 닫힌다.
- **환경**: Android System WebView M139 이상.
- **원인**: M139부터 키보드(`ime()`)가 Visual Viewport를 줄이므로 `resize` 이벤트가 더 자주 발생한다. resize 핸들러가 포커스를 해제하는 페이지에서는 "포커스 → 키보드 표시 → resize → 포커스 해제 → 키보드 닫힘" 루프가 생긴다고 Chromium 문서가 직접 경고한다.
- **해결책**: resize 핸들러에서 `blur()`나 리렌더로 포커스를 잃게 하는 코드를 점검한다.
- **출처**: [Chromium insets.md "What Changed?"](https://chromium.googlesource.com/chromium/src/+/HEAD/android_webview/docs/insets.md)

---

## 5. Android 15 edge-to-edge 회귀

### 5-1. targetSdk 35 이상이면 edge-to-edge가 강제되고, Android 16에서는 opt-out도 사라짐

- **증상**: targetSdk를 35로 올리면 앱 콘텐츠가 상태 바와 내비게이션 바 아래로 그려진다.
- **환경**: Android 15(API 35)+, Flutter 3.27부터 기본 `targetSdkVersion`이 Android 15.
- **원인**: Android 15가 SDK 35 이상을 타깃하는 앱에 edge-to-edge를 강제한다. Flutter도 기본 `SystemUiMode`를 edge-to-edge로 바꿨다.
- **해결책**: Android 15에서는 테마에 `android:windowOptOutEdgeToEdgeEnforcement=true`로 임시 opt-out할 수 있지만, Flutter 문서는 Android 16(API 36)에서 이 opt-out이 제거되고 사용 시 크래시할 수 있다고 경고한다. 정식 대응은 edge-to-edge로 이전하고 셸 또는 웹이 inset을 처리하는 것이다.
- **출처**: [Flutter breaking change: Default SystemUiMode edge-to-edge](https://docs.flutter.dev/release/breaking-changes/default-systemuimode-edge-to-edge)

### 5-2. 인앱 WebView로 연 페이지 하단이 제스처 바에 가려짐

- **증상**: `url_launcher`의 `LaunchMode.inAppWebView`로 연 페이지에서 하단 쿠키 배너 버튼 등이 제스처 내비게이션 바 아래에 가려 누를 수 없다.
- **환경**: Android 15, url_launcher 6.3.1.
- **원인**: edge-to-edge 강제 후 플러그인이 띄우는 WebView Activity가 inset을 처리하지 않는다. 외부 웹페이지라 웹 쪽 `env()` 대응도 기대할 수 없다.
- **해결책**: 미해결(OPEN). 교훈: **셸이 띄우는 모든 WebView 화면**(메인 외 약관·결제·외부 링크용)도 inset 처리 대상이다. 제어할 수 없는 외부 페이지는 셸이 safe area 안에 배치해야 한다.
- **출처**: [flutter/flutter#170333](https://github.com/flutter/flutter/issues/170333)

### 5-3. API 레벨마다 inset 주입 결과가 달라 일관된 전략을 세우기 어려움

- **증상**: 같은 빌드(SDK 36)인데 API 35·36 에뮬레이터에서는 inset이 정상 주입되고, API 29 기기에서는 하단 inset이 `0px`이라 푸터가 내비게이션 바 아래로 사라진다. 비슷하게 API < 30에서 `EdgeToEdge.enable()`을 쓰면 하단 inset이 0으로 보고된다는 이슈도 있다.
- **환경**: Android API 29~36, 최신 WebView, Capacitor 8.x SystemBars.
- **원인**: edge-to-edge가 강제되는 API 35+와 선택 사항인 API 34 이하에서 시스템이 주는 inset 구조가 다르고, 셸 구현이 API 35+만 기준으로 짜였다. 코멘트에 따르면 safe area CSS 주입 최소 버전이 API 36(`VANILLA_ICE_CREAM`)으로 걸려 있었다.
- **해결책**: 8394는 닫혔지만, 코멘트에서 제안된 [PR #8366](https://github.com/ionic-team/capacitor/pull/8366)(주입 최소 SDK를 낮추는 PR)은 머지되지 않고 닫혔다. 실제로 고친 변경은 **미확인**. 8603은 OPEN. 교훈: 최소 지원이 Android 8(API 26)이면 **API 26~34(선택적 edge-to-edge)와 35+(강제)를 별도 매트릭스로** 테스트해야 한다.
- **출처**: [ionic-team/capacitor#8394](https://github.com/ionic-team/capacitor/issues/8394), [ionic-team/capacitor#8603](https://github.com/ionic-team/capacitor/issues/8603)

---

## 6. 상태 바·내비게이션 바 색상

### 6-1. API 35+에서 `statusBarColor`가 무시됨

- **증상**: `SystemChrome.setSystemUIOverlayStyle(SystemUiOverlayStyle(statusBarColor: ...))`나 `AnnotatedRegion`으로 상태 바 색을 바꿔도 Android API 35 타깃에서는 반영되지 않는다.
- **환경**: Android 15+, targetSdk 35, Flutter.
- **원인**: Android 15가 edge-to-edge에서 상태 바 색 지정 API를 폐기했다. Flutter는 동작을 바꾸지 않고 문서와 엔진 주석에 이 사실을 명시하는 것으로 이슈를 닫았다.
- **해결책**: Flutter 메인테이너 답변: "상태 바 색을 바꾸려면 원하는 색의 위젯을 화면 상단에 고정해야 한다." 하이브리드웹에서는 **상태 바 뒤 영역을 누가 칠하는지** 정해야 한다. 웹이 칠한다면 WebView를 화면 끝까지 붙이고 상단 `env(safe-area-inset-top)` 영역에 배경을 깔고, 셸이 칠한다면 상단 inset 높이의 Flutter 위젯을 둔다. 아이콘 밝기(`statusBarIconBrightness`)는 여전히 셸이 지정해야 한다.
- **출처**: [flutter/flutter#163545](https://github.com/flutter/flutter/issues/163545), [flutter/flutter#160328](https://github.com/flutter/flutter/issues/160328), [Android `Window.setStatusBarColor` 레퍼런스(API 35에서 deprecated)](https://developer.android.com/reference/android/view/Window#setStatusBarColor(int))

### 6-2. Flutter 3.32 이후 `systemNavigationBarColor: transparent`가 검은색으로 나옴

- **증상**: 3-버튼 내비게이션 바를 투명하게 지정했는데 Flutter 3.32.1부터 항상 검은색. 3.29.3에서는 정상.
- **환경**: Android API 35+, Flutter 3.32.1.
- **원인**: 트리아지 코멘트는 6-1과 같은 API 35 변경과 관련 있다고 본다. 확정 원인 **미확인**(OPEN).
- **해결책**: 없음. Flutter 버전을 올릴 때 3-버튼 내비게이션 기기에서 하단 바 색을 확인한다.
- **출처**: [flutter/flutter#169746](https://github.com/flutter/flutter/issues/169746)

---

## 7. 비교 환경(모바일 브라우저)에서만 나타나는 사례

같은 Next.js 앱이 브라우저에서도 열리므로 참고로 남긴다. WebView 셸에는 직접 해당하지 않을 수 있다.

- [WebKit 281071 "REGRESSION (iOS 18): Safe-area-inset-bottom not working"](https://bugs.webkit.org/show_bug.cgi?id=281071): iOS 18 Safari에서 주소창이 자동으로 펼쳐질 때 하단 고정 내비가 가려진다. NEW.
- [WebKit 300523 "REGRESSION (iOS 26): Safari on iPhones with Dynamic Island shifts the viewport upward when scrolling or closing the keyboard"](https://bugs.webkit.org/show_bug.cgi?id=300523): NEW. 상세 미열람(**미확인**).
- [WebKit 313759](https://bugs.webkit.org/show_bug.cgi?id=313759): iOS Chrome(WKWebView 기반)에서 URL 입력으로 첫 내비게이션을 하면 키보드가 열린 상태의 뷰포트가 캐시돼 `bottom: 0` 요소 위치가 어긋난다. Chrome iOS 148에서 재현되지 않는다는 업데이트가 있다.
- [WebKit 274773](https://bugs.webkit.org/show_bug.cgi?id=274773): `env()`를 CSS 변수로 받아 JS로 읽는 기법이 iOS 17.5에서 0을 돌려준다는 보고. WebKit 엔지니어는 "스크롤 바 등 피할 것이 없는 세로 화면에서는 0이 예상 동작"이라고 답했다. 세로 화면에서 좌우 inset이 0인 것은 정상이다.

---

## 가이드에 반영할 핵심

빈도(여러 출처에서 반복되는가)와 피해(콘텐츠가 가려지거나 조작할 수 없게 되는가)를 기준으로 순위를 매겼다.

1. **Android WebView 버전에 따라 inset 동작이 세 갈래로 나뉜다(0 / 정상 / 이중).** M136 이전에는 0(1-1, 1-2), M144 이후에는 겹치는 만큼 보고(2-1). WebView는 앱과 별개로 업데이트되므로 출시 후 어느 날 갑자기 레이아웃이 바뀐다. 가이드는 셸이 `setInsetsForWebContentToIgnore`(webview_flutter_android 4.11.0+)로 inset 소유권을 명시하는 방식을 다뤄야 한다. 출처가 가장 많고 피해도 가장 크다.
2. **inset을 처리하는 주체는 한 곳이어야 한다.** 셸(Flutter `SafeArea`·부모 뷰 패딩)과 웹(`env()`)이 모두 처리하면 이중 패딩(2-1, 2-3, 2-4)이 생기고, 둘 다 안 하면 가려진다(5-2). 이것이 지도 #1의 "웹/네이티브 책임 분담표"의 근거다.
3. **Android 15/16 edge-to-edge는 피할 수 없다.** opt-out은 Android 16에서 사라진다(5-1). 최소 지원 API 26이면 API 26~34와 35+를 별도로 테스트해야 한다(5-3). 메인 WebView뿐 아니라 셸이 띄우는 모든 WebView 화면이 대상이다(5-2).
4. **키보드: 하단 여백은 합이 아니라 최대값이고, 리사이즈 주체도 하나만.** 키보드가 열려도 `safe-area-inset-bottom`이 남는 문제는 WebKit에서 수년째 미해결이다(4-1). 셸과 WebView가 각각 줄이면 이중 축소가 생긴다(4-3, 4-4). WebView M139+의 Visual Viewport 리사이즈와 Flutter `resizeToAvoidBottomInset` 중 무엇을 쓸지 정해야 한다(지도 #1의 "가상 키보드 처리 권장 패턴"과 직결).
5. **`viewport-fit=cover`를 명시하고, 셸이 viewport 메타를 덮어쓰지 않게 한다.** iOS는 `cover` 없이는 0이고 Android는 `cover`와 무관하게 값을 준다(1-5). 플러그인 스크립트가 메타를 교체하면 조용히 깨진다(1-4). Next.js는 `export const viewport = { viewportFit: 'cover' }`로 선언한다.
6. **값은 늦게 오고 바뀔 수 있다.** 첫 렌더 0(1-3), 회전·전체 화면·시스템 UI 모드 변경 후 미갱신(3-1~3-5)이 반복된다. JS로 한 번 읽어 캐시하지 말고 CSS `env()`를 직접 쓰는 것이 기본 방어다. iOS 26 관련 회귀(3-3, 3-4)는 아직 열려 있어 실기기 확인이 필요하다.
7. **상태 바 색은 Android 15+에서 API로 지정할 수 없다.** 상태 바 뒤 배경을 웹과 셸 중 누가 칠할지 분담표에 넣어야 한다(6-1, 6-2).
