# Flutter 셸에서 WebView로의 inset 전달 방식

> Research ticket: [#5](https://github.com/ilkyeong-gn/learn-safe-area/issues/5) (map [#1](https://github.com/ilkyeong-gn/learn-safe-area/issues/1))
> 조사일: 2026-10-23. 1차 출처(Flutter API/문서, flutter/flutter·flutter/packages 소스와 이슈, Chromium·WebKit 소스, pub.dev 문서)만 사용했다.
> 소스 링크는 조사 시점의 커밋으로 고정했다: flutter/flutter `7b0a985`, flutter/packages `108dad0`, WebKit `e7c482a`.
> **미확인** 표시는 1차 출처로 확인하지 못했거나 실기기 검증이 필요한 항목이다(실기기 검증은 지도 범위 밖).

## 요약

- WebView 안의 `env(safe-area-inset-*)`는 Flutter의 `MediaQuery`나 `SafeArea`에서 **오지 않는다**. 플랫폼 WebView 엔진(iOS WKWebView, Android System WebView/Chromium)이 **네이티브 뷰 자신의 위치와 OS inset을 보고 직접 계산**한다. Flutter 위젯 트리는 WebView의 **배치(frame)** 를 통해서만 간접적으로 영향을 준다.
- **iOS**: `UiKitView`는 진짜 `UIView`를 뷰 계층에 붙이므로, WKWebView는 자기 `safeAreaInsets`를 가진다. 페이지가 `viewport-fit=cover`일 때만 그 값이 `env()`로 노출된다(WebKit 소스). Flutter가 WebView를 `SafeArea` 안에 두면 겹침이 없으니 값은 0이 된다.
- **Android**: System WebView M136부터 system bar와 display cutout inset이 `env()`로 전달된다. M144부터는 fullscreen이 아닌 WebView에도 전달된다. 그런데 Flutter 안에서는 WebView가 시스템 바와 겹치지 않는데도 inset이 보고되는 버그가 있었다([flutter#182208](https://github.com/flutter/flutter/issues/182208)). `webview_flutter_android` 4.11.0에 **opt-out API** `setInsetsForWebContentToIgnore`가 추가됐지만, **기본값은 그대로**다.
- 결론적으로 Flutter 셸 환경에서 `env()`는 플랫폼·WebView 버전·배치에 따라 값이 달라 신뢰하기 어렵다. 셸이 inset을 **명시적으로 주입**(CSS custom property, JS bridge)하고, 웹이 `env()`를 fallback으로 쓰는 패턴이 가장 예측 가능하다.

---

## 1. Flutter 쪽 inset 모델: `padding` / `viewPadding` / `viewInsets`

`MediaQueryData`는 inset을 세 가지로 나눈다.

| 속성 | 의미 | 키보드가 올라오면 |
|---|---|---|
| `viewPadding` | 시스템 UI(상태 바, 노치, 홈 인디케이터 등)에 **부분적으로** 가려지는 영역 | 변하지 않음 |
| `viewInsets` | 시스템 UI에 **완전히** 가려지는 영역. 주로 키보드 | 키보드 높이만큼 커짐 |
| `padding` | `max(0.0, viewPadding - viewInsets)` | 아래쪽이 0까지 줄어듦 |

- 출처: [MediaQueryData](https://api.flutter.dev/flutter/widgets/MediaQueryData-class.html). 원문은 "padding is calculated by taking `max(0.0, viewPadding - viewInsets)`"이다. inset을 소비한 위젯은 `removePadding` / `removeViewPadding` / `removeViewInsets`로 자식의 `MediaQuery`를 줄여야 한다고 설명한다.
- `SafeArea`는 `MediaQuery`의 `padding`을 읽어 자식을 안쪽으로 밀고, **자식에게 전달되는 `MediaQuery.padding`에서 처리한 변을 0으로 만든다**. `minimum`이 있으면 둘 중 큰 값을 쓴다. `maintainBottomViewPadding: true`이면 아래쪽은 `padding` 대신 `viewPadding`을 쓴다(키보드가 올라와도 하단 여백 유지). 출처: [SafeArea](https://api.flutter.dev/flutter/widgets/SafeArea-class.html).

### Android에서 이 값이 어디서 오는가

Flutter 엔진의 `FlutterView.onApplyWindowInsets`는 API 30+에서 다음 값을 매핑한다([FlutterView.java L735-L888](https://github.com/flutter/flutter/blob/7b0a985e549de7c9401db8def8b77e1d1e618617/engine/src/flutter/shell/platform/android/io/flutter/embedding/android/FlutterView.java#L735-L888)).

- `viewPadding` ← `WindowInsets.Type.systemBars()`. `DisplayCutout`의 safe inset과 waterfall inset 중 큰 값과 다시 max를 취한다.
- `viewInsets` ← `WindowInsets.Type.ime()`
- API 35+에서는 caption bar 영역도 반영된다(`growViewportMetricsToCaptionBar`).
- 메서드는 `super.onApplyWindowInsets(insets)`의 결과를 **그대로 반환하고 inset을 소비하지 않는다**. 따라서 이 소스만 보면 `FlutterView` 아래에 붙은 네이티브 자식 뷰(platform view)도 window inset을 받는다고 추론된다(추론. 하위 뷰 dispatch 경로는 **미확인**).

## 2. `SystemChrome` edge-to-edge

- `SystemUiMode.edgeToEdge`: "Fullscreen display with status and navigation elements rendered over the application." Android는 API 29부터 적용된다. **앱이 Android SDK 15(API 35) 이상을 target하면 기본값이고 Flutter는 기본으로 그렇게 target한다.** iOS에서도 기본 모드다. 출처: [SystemUiMode](https://api.flutter.dev/flutter/services/SystemUiMode.html).
- Flutter 3.27부터 `flutter.targetSdkVersion`이 35가 되어 자동으로 edge-to-edge가 된다. Android 15에서는 `android:windowOptOutEdgeToEdgeEnforcement`로 opt-out할 수 있지만 Android 16부터는 불가능하다(16에서 이 속성을 쓰면 crash 가능성이 있어 `values-35` 분리를 권장). 출처: [Default SystemUiMode edge-to-edge breaking change](https://docs.flutter.dev/release/breaking-changes/default-systemuimode-edge-to-edge).
- 의미: 최소 지원 OS(Android 8.0+)와 관계없이 최신 기기에서는 Flutter 화면이 시스템 바 아래까지 그려진다. 그래서 **WebView를 어디에 배치하느냐**(SafeArea 안인지, 화면 전체인지)가 곧 WebView가 시스템 바와 겹치는지를 결정한다.
- 관련 버그: edge-to-edge에서 시스템 설정으로 화면 크기를 바꾼 뒤 WebView가 리사이즈되지 않는 문제가 열려 있다([flutter#162003](https://github.com/flutter/flutter/issues/162003), open).

## 3. 플랫폼 뷰 방식과 WebView의 네이티브 inset 수신

### 3.1 iOS: `UiKitView`

- iOS platform view는 항상 Hybrid composition이다. 원문: "iOS only uses Hybrid composition, which means that the native `UIView` is appended to the view hierarchy." 출처: [Hosting native iOS views](https://docs.flutter.dev/platform-integration/ios/platform-views).
- 따라서 WKWebView는 UIKit 뷰 계층의 실제 뷰이고, UIKit은 이 뷰의 window 내 위치로 `safeAreaInsets`를 계산한다. 화면 가장자리에 붙어 있으면 값이 있고, 안쪽에 있으면 0이다. ([UIView.safeAreaInsets](https://developer.apple.com/documentation/uikit/uiview/safeareainsets). 페이지 본문이 JS 렌더링이라 원문 문구는 **미확인**)
- **WebKit이 `env(safe-area-inset-*)`를 정하는 방식**(WebKit 소스):
  - `viewport-fit`이 `cover`가 아니면 `avoidsUnsafeArea = true`다([ViewportConfiguration.cpp L569](https://github.com/WebKit/WebKit/blob/e7c482ada164db14120820ad2ab37dd44f1b1160/Source/WebCore/page/ViewportConfiguration.cpp#L569)).
  - `_computedUnobscuredSafeAreaInset`은 `_safeAreaShouldAffectObscuredInsets`(= `avoidsUnsafeArea`)가 false일 때만 `self.safeAreaInsets`를 반환하고, 아니면 `UIEdgeInsetsZero`를 반환한다. 이 값이 웹 콘텐츠로 전달된다([WKWebViewIOS.mm L873-L891](https://github.com/WebKit/WebKit/blob/e7c482ada164db14120820ad2ab37dd44f1b1160/Source/WebKit/UIProcess/API/ios/WKWebViewIOS.mm#L873-L891), [L4700-L4705](https://github.com/WebKit/WebKit/blob/e7c482ada164db14120820ad2ab37dd44f1b1160/Source/WebKit/UIProcess/API/ios/WKWebViewIOS.mm#L4700-L4705), [L3347](https://github.com/WebKit/WebKit/blob/e7c482ada164db14120820ad2ab37dd44f1b1160/Source/WebKit/UIProcess/API/ios/WKWebViewIOS.mm#L3347)).
  - 정리하면 iOS WebView의 `env()`는 **`viewport-fit=cover`이고 WKWebView가 실제로 unsafe 영역과 겹칠 때만** 0이 아니다.
- **`webview_flutter_wkwebview`의 추가 처리**: `scrollView.contentInsetAdjustmentBehavior = .never`로 설정한다. frame이 바뀔 때마다 `adjustedContentInset`을 상쇄하는 음수 `contentInset`을 넣는다. 주석은 "Prevents the contentInsets from being adjusted by iOS and gives control to Flutter."이다([WebViewProxyAPIDelegate.swift L7-L58](https://github.com/flutter/packages/blob/108dad0d562e9368a45b6b5cdcc26c137bc071c4/packages/webview_flutter/webview_flutter_wkwebview/darwin/webview_flutter_wkwebview/Sources/webview_flutter_wkwebview/WebViewProxyAPIDelegate.swift#L7-L58)).
  - 이 처리는 **스크롤 content inset**에 대한 것이다. `viewport-fit=cover`일 때 `env()` 값의 출처인 `self.safeAreaInsets`는 바꾸지 않는다(소스 기반 추론).
  - 추론: `viewport-fit=cover`를 쓰지 않고 WebView를 화면 전체에 배치하면, `env()`는 0이고 iOS 기본 스크롤 inset 보정도 꺼져 있다. 그러면 콘텐츠가 노치·홈 인디케이터 아래에 깔릴 수 있다(실기기 **미확인**).

### 3.2 Android: Texture Layer Hybrid Composition(TLHC) vs Hybrid Composition(HC)

| `webview_flutter_android` 설정 | 호출되는 API | 모드 |
|---|---|---|
| `displayWithHybridComposition: false` (**기본값**) | `initSurfaceAndroidView` | TLHC를 시도하고, 지원되지 않으면 HC로 fallback |
| `displayWithHybridComposition: true` | `initExpensiveAndroidView` | 항상 HC |

- 출처: [android_webview_controller.dart L1312-L1338](https://github.com/flutter/packages/blob/108dad0d562e9368a45b6b5cdcc26c137bc071c4/packages/webview_flutter/webview_flutter_android/lib/src/android_webview_controller.dart#L1312-L1338), [platform_views.dart (initSurfaceAndroidView / initExpensiveAndroidView 문서 주석)](https://github.com/flutter/flutter/blob/7b0a985e549de7c9401db8def8b77e1d1e618617/packages/flutter/lib/src/services/platform_views.dart#L156-L194), [AndroidWebViewWidgetCreationParams](https://pub.dev/documentation/webview_flutter_android/latest/webview_flutter_android/AndroidWebViewWidgetCreationParams-class.html), [webview_flutter_android README](https://pub.dev/packages/webview_flutter_android). README는 기본값이 "subject to change in the future"라고 밝힌다.
- 모드별 특징([Android platform views](https://docs.flutter.dev/platform-integration/android/platform-views)):
  - TLHC: 네이티브 뷰가 texture로 렌더링되고 Flutter가 합성한다. 성능은 좋지만 빠른 스크롤에서 jank가 생기고 SurfaceView 계열에 제약이 있다.
  - HC: 네이티브 뷰가 Android 뷰 계층에 직접 들어간다. 충실도는 높지만 raster/platform thread merge 때문에 Flutter FPS가 떨어진다.
  - HCPP(Flutter 3.44 실험적, API 34+/Vulkan/Impeller 필요)는 전역 플래그로 켠다.
- **inset 관점**: Android System WebView는 Chromium 문서 기준으로 다음과 같다([Android WebView & Insets](https://chromium.googlesource.com/chromium/src.git/+/main/android_webview/docs/insets.md)).
  - **M136**: `displayCutout()`·`systemBars()` inset을 `env(safe-area-inset-*)`로 전달하기 시작했다. 단, WebView가 fullscreen일 때만이다.
  - **M139**: `ime()` inset을 지원한다. 이것은 safe-area가 아니라 **Visual Viewport 리사이즈**로 반영된다.
  - **M144**: fullscreen이 아닌 WebView까지 확대됐다. "The insets should still only report non-zero values when the UI directly overlaps with WebView's bounds."
  - 앱이 `WindowInsets.CONSUMED`를 반환하면 WebView는 safe area inset을 받지 않는다. 권장 opt-out은 해당 타입을 **0으로 바꿔서 전달**하는 것이다(consume하지 말 것).
  - 문서에 `viewport-fit` 언급은 없다. Android WebView가 `viewport-fit`과 무관하게 값을 주는지는 **미확인**이다.
- **Flutter 안에서의 실제 동작(버그)**: [flutter#182208](https://github.com/flutter/flutter/issues/182208) "android webview M144 reporting insets despite not overlapping with system bars".
  - 재현 조건: `Scaffold` + `AppBar(toolbarHeight: 0)` + `body: WebViewWidget`. Flutter 레이아웃상 WebView는 상태 바와 겹치지 않는데도 WebView 144+에서 `env(safe-area-inset-top)`이 0이 아니다. WebView 133/134에서는 정상이었다(Pixel 10, Android 16, WebView 144.0.7559.132 재현 코멘트).
  - 원인(왜 Flutter 배치와 네이티브 bounds 판단이 어긋나는지)은 이슈와 PR에 설명되지 않았다(**미확인**). TLHC/HC 중 어느 모드에서 재현되는지도 기록이 없다(**미확인**. 재현 코드는 기본값이므로 TLHC).
- **해결**: [flutter/packages#11192](https://github.com/flutter/packages/pull/11192) (2026-04-09 merge, `webview_flutter_android` **4.11.0**)가 `AndroidWebViewController.setInsetsForWebContentToIgnore(List<AndroidWebViewInsets>)`를 추가했다.
  - 내부적으로 WebView에 `ViewCompat.setOnApplyWindowInsetsListener`를 걸고, 지정한 타입을 `Insets.NONE`으로 바꾼 `WindowInsetsCompat`을 전달한다. Chromium 문서가 권장하는 "zero out" 방식이다([ViewProxyApi.java 변경](https://github.com/flutter/packages/pull/11192/files)).
  - **opt-in이다.** 기본 동작은 그대로이므로 셸이 직접 호출해야 한다.
  - PR 본문: "It seems Flutter handles safe areas for platform views, so the `AndroidWebViewController` can zero out inset changes to the WebContent." iOS 쪽의 `contentInsetAdjustmentBehavior = .never` 처리와 대응되는 조치라고 설명한다.
  - `AndroidWebViewInsets`는 `systemBars`, `displayCutout`, `captionBar`, `ime`, `mandatorySystemGestures`, `navigationBars`, `statusBars`, `systemGestures`, `tappableElement`를 지원한다([소스 L843-L870](https://github.com/flutter/packages/blob/108dad0d562e9368a45b6b5cdcc26c137bc071c4/packages/webview_flutter/webview_flutter_android/lib/src/android_webview_controller.dart#L843-L870)).
  - 주의: `ime`까지 0으로 만들면 M139+ Visual Viewport 리사이즈 동작도 끄게 된다. Chromium 문서는 opt-out 시 "keyboard may cover inputs near the bottom", `scrollIntoView` 오동작 가능성을 경고한다.

```dart
// webview_flutter_android >= 4.11.0
import 'package:webview_flutter_android/webview_flutter_android.dart';

final controller = WebViewController();
if (controller.platform is AndroidWebViewController) {
  await (controller.platform as AndroidWebViewController)
      .setInsetsForWebContentToIgnore(const [
    AndroidWebViewInsets.systemBars,
    AndroidWebViewInsets.displayCutout,
  ]);
}
```

## 4. 셸이 inset을 직접 주입하는 패턴

`webview_flutter`의 공개 API는 `runJavaScript`, `runJavaScriptReturningResult`, `addJavaScriptChannel`, `setUserAgent` 등이다. **document-start 시점에 user script를 넣는 공개 API는 없다**([WebViewController 4.14.1](https://pub.dev/documentation/webview_flutter/latest/webview_flutter/WebViewController-class.html)). iOS 구현은 JS channel을 위해 내부적으로만 `addUserScript`를 쓴다([webkit_webview_controller.dart L420-L449](https://github.com/flutter/packages/blob/108dad0d562e9368a45b6b5cdcc26c137bc071c4/packages/webview_flutter/webview_flutter_wkwebview/lib/src/webkit_webview_controller.dart#L420-L449)).

### 4.1 CSS custom property push (권장 기본형)

셸이 자기 `MediaQuery`를 읽어 WebView가 덮는 변에 해당하는 값을 문서 루트에 넣는다.

```dart
// WebView를 화면 전체에(SafeArea 밖에) 배치했다고 가정
Future<void> pushInsets(WebViewController c, MediaQueryData mq) {
  final p = mq.viewPadding;   // 키보드와 무관한 시스템 UI 영역
  final k = mq.viewInsets;    // 키보드
  return c.runJavaScript('''
    (function(s){
      s.setProperty('--app-inset-top', '${p.top}px');
      s.setProperty('--app-inset-right', '${p.right}px');
      s.setProperty('--app-inset-bottom', '${p.bottom}px');
      s.setProperty('--app-inset-left', '${p.left}px');
      s.setProperty('--app-keyboard', '${k.bottom}px');
    })(document.documentElement.style);
  ''');
}
// onPageFinished(그리고 필요하면 onPageStarted)와 didChangeMetrics/MediaQuery 변경 시 호출
```

```css
:root {
  --safe-top: var(--app-inset-top, env(safe-area-inset-top, 0px));
  --safe-bottom: var(--app-inset-bottom, env(safe-area-inset-bottom, 0px));
}
header { padding-top: var(--safe-top); }
```

- 셸 변수가 없으면(모바일 브라우저) `env()`로 fallback되므로 같은 Next.js 앱을 양쪽에서 쓸 수 있다.
- 단위: Flutter logical pixel과 WebView CSS px가 1:1이라는 보장은 1차 출처로 확인하지 못했다(**미확인**). Android dp와 iOS pt 기준으로는 일반적으로 일치하지만, 셸에서 WebView 줌이나 `initial-scale`을 바꾸면 달라질 수 있다.
- 한계: `runJavaScript`는 현재 페이지에만 적용된다. 그래서 **첫 페인트 이후**에 적용될 수 있고(레이아웃 점프), 네비게이션할 때마다 다시 넣어야 한다. 정확한 타이밍 보장은 **미확인**.

### 4.2 JS bridge pull

- 웹이 로드 시 `window.<Channel>.postMessage('getInsets')`를 호출하고, 셸이 `onMessageReceived`에서 `runJavaScript`로 콜백을 실행해 값을 넘긴다([addJavaScriptChannel](https://pub.dev/documentation/webview_flutter/latest/webview_flutter/WebViewController/addJavaScriptChannel.html)).
- 웹이 준비된 시점에 요청하므로 타이밍 문제가 줄어든다. 하지만 SSR/첫 페인트 시점에는 여전히 값이 없다.

### 4.3 로드 전 전달(첫 페인트 대응)

- URL query나 커스텀 header(`loadRequest`의 headers), `setUserAgent`에 플래그나 값을 실어 Next.js 서버가 SSR 단계에서 알게 하는 방법이 있다. API 자체는 [WebViewController](https://pub.dev/documentation/webview_flutter/latest/webview_flutter/WebViewController-class.html)에 있다.
- 다만 inset 값은 회전·키보드로 바뀌므로 초기값 용도로만 쓸 수 있다. 실무 적합성은 지도의 "WebView 감지·분기 전략" 결정 사항이다.

## 5. `flutter_inappwebview`와의 차이

| 항목 | `webview_flutter` | `flutter_inappwebview` (6.1.5 문서 기준) |
|---|---|---|
| Android 기본 composition | TLHC (`displayWithHybridComposition=false`) | **HC** (`useHybridComposition` 기본 `true`) |
| iOS content inset | `.never` + `adjustedContentInset` 상쇄(코드 고정) | `contentInsetAdjustmentBehavior` 설정 가능, 기본 `NEVER` |
| iOS viewport inset 범위 | 없음 | `minimumViewportInset` / `maximumViewportInset` |
| document-start 주입 | 공개 API 없음 | `UserScript` + `UserScriptInjectionTime.AT_DOCUMENT_START` |
| Android M144 inset opt-out | `setInsetsForWebContentToIgnore` (4.11.0+) | 해당 API 없음(기본 브랜치 소스에서 `WindowInsets` 사용처는 화면 크기 계산뿐) |

- 출처: [InAppWebViewSettings](https://pub.dev/documentation/flutter_inappwebview/latest/flutter_inappwebview/InAppWebViewSettings-class.html), [UserScriptInjectionTime](https://pub.dev/documentation/flutter_inappwebview/latest/flutter_inappwebview/UserScriptInjectionTime-class.html), [Util.java (WindowInsets 사용)](https://github.com/pichillilorenzo/flutter_inappwebview/blob/master/flutter_inappwebview_android/android/src/main/java/com/pichillilorenzo/flutter_inappwebview_android/Util.java).
- `flutter_inappwebview` 최신 release는 `v6.2.0-beta.3`(2026-02-04)이고 M144 대응 여부는 **미확인**이다.
- AT_DOCUMENT_START를 쓰면 CSS 변수를 첫 페인트 전에 넣을 수 있다는 점이 가장 큰 실무 차이다. 다만 문서상 정의는 iOS 기준이고, Android 쪽 정확한 타이밍은 **미확인**이다.
- 보고된 이슈: iOS에서 `supportZoom: false`이면 `env(safe-area-inset-bottom)`이 무효화된다는 보고가 있다([flutter_inappwebview#1911](https://github.com/pichillilorenzo/flutter_inappwebview/issues/1911), closed. 메인테이너 답변 없이 닫혀 원인은 **미확인**).

## 6. 가이드에 반영할 핵심

1. **`env()`의 값은 Flutter가 아니라 WebView 엔진이 정한다.** Flutter의 `SafeArea`/`MediaQuery`는 WebView의 배치를 통해서만 영향을 준다. "`SafeArea`로 감쌌는데 왜 `env()`가 0이 아니냐"(Android M144+)와 "WebView를 화면 전체에 깔았는데 왜 `env()`가 0이냐"(iOS에서 `viewport-fit=cover` 누락)가 대표적인 혼란이다.
2. **책임 분담을 먼저 정한다.** (a) 셸이 `SafeArea`로 처리하고 웹은 inset을 무시하는 방식, (b) 셸은 WebView를 화면 전체에 깔고 웹이 처리하는 방식 중 하나를 고른다. (a)를 고르면 Android에서 `setInsetsForWebContentToIgnore([systemBars, displayCutout])`를 호출해야 이중 여백을 막을 수 있다(`webview_flutter_android` ≥ 4.11.0).
3. **(b)라면 셸 주입 CSS 변수 + `env()` fallback**을 권장 패턴으로 제시한다. `--app-inset-*`가 있으면 그 값을, 없으면(브라우저) `env(safe-area-inset-*)`를 쓴다. iOS WebView 단독으로 `env()`에 의존하려면 `viewport-fit=cover`가 필수다(Next.js `export const viewport = { viewportFit: 'cover' }`).
4. **Android edge-to-edge는 Flutter 3.27+/targetSdk 35에서 기본이고 Android 16에서는 opt-out이 불가능하다.** "시스템 바 아래까지 그려진다"를 전제로 써야 한다. WebView 버전(M136/M139/M144)에 따라 같은 앱의 동작이 달라지므로, 가이드에 **Android System WebView 버전**을 확인 항목으로 넣는다.
5. **키보드**: Flutter는 `viewInsets`, Android WebView M139+는 Visual Viewport 리사이즈, `ime` opt-out 시 키보드 가림 위험이 있다. 이 내용은 키보드 조사 티켓과 함께 정리해야 한다.
6. **`flutter_inappwebview`는 비교 항목으로만 다룬다.** 기본 HC, document-start user script, inset opt-out API 부재가 차이점이다.
