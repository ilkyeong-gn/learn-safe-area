# 가상 키보드와 safe area의 상호작용

> 연구 티켓: [#7](https://github.com/ilkyeong-gn/learn-safe-area/issues/7) · 지도: [#1](https://github.com/ilkyeong-gn/learn-safe-area/issues/1)
> 조사일: 2026-10-23 · 전제: Flutter `webview_flutter` 네이티브 셸 + Next.js 16, iOS 16+ / Android 8.0+
> 실기기 검증은 하지 않았다(지도 범위 밖). 1차 출처로 확인하지 못한 항목은 **미확인**으로 표시한다.

## 질문

가상 키보드가 올라올 때 `safe-area-inset-bottom`, `visualViewport`, `interactive-widget`, VirtualKeyboard API(`env(keyboard-inset-*)`)는 iOS WKWebView와 Android WebView에서 각각 어떻게 동작하는가? 하단 고정 UI는 왜 깨지는가? Flutter `resizeToAvoidBottomInset`은 WebView 리사이즈에 어떤 영향을 주는가?

## 요약

- 스펙상 **기본 동작은 `resizes-visual`**이다. 키보드가 열리면 visual viewport만 줄고 layout viewport(ICB)는 그대로라서, `position: fixed; bottom: 0` 요소는 키보드 뒤에 가려진다. iOS의 모든 브라우저와 WKWebView가 이 동작을 하고, Chrome Android도 108부터 이 동작이 기본이다.
- **iOS(WebKit)**: 키보드가 열려도 `safe-area-inset-bottom`이 0이 되지 않는다(WebKit 버그 217754, 2020년부터 NEW 상태). `interactive-widget`(버그 259770)과 VirtualKeyboard API(버그 230225)는 둘 다 구현되지 않았다. iOS에서 쓸 수 있는 신호는 사실상 `visualViewport`뿐이다.
- **Android WebView**: 앱이 WebView 크기를 정한다. 셸이 WebView를 키보드 위로 줄이면 layout viewport 자체가 준다(`resizes-content`와 같은 효과). WebView가 키보드 아래까지 덮여 있으면, M139부터는 IME inset이 **visual viewport만** 줄인다. IME는 `safe-area-inset-*`로 전달되지 않는다.
- **Flutter**: `Scaffold.resizeToAvoidBottomInset`의 기본값은 `true`다. 그래서 기본 상태에서는 body에 있는 WebView 위젯이 키보드 높이만큼 줄어든다. 웹 입장에서는 "WebView가 키보드 위로 리사이즈됨"이 기본 시나리오다.
- 하단 고정 UI가 깨지는 대표 원인은 네 가지다. ① 키보드가 열려도 남아 있는 `safe-area-inset-bottom` 여백(iOS), ② `resizes-visual` 때문에 fixed 요소가 키보드에 가려짐, ③ 셸 리사이즈와 WebView 내부 키보드 처리가 겹침(이중 보정), ④ WebKit의 키보드 + 하단 inset 스크롤 범위 버그(292603).

## 1. 스펙: 세 가지 viewport 동작

CSS Viewport Module(Editor's Draft)은 `interactive-widget` 키를 정의한다. ([css-viewport ED](https://drafts.csswg.org/css-viewport/))

| 값 | 의미 |
|---|---|
| `resizes-visual` (기본값) | visual viewport는 줄이고 initial viewport(ICB)는 줄이지 **않는다** |
| `resizes-content` | initial viewport를 줄이며, 그 결과 visual viewport도 함께 준다 |
| `overlays-content` | 둘 다 줄이지 않는다(키보드가 콘텐츠를 덮는다). `VirtualKeyboard.overlaysContent = true`와 같은 단계를 수행한다 |

- 값이 없거나 잘못되면 `resizes-visual`로 동작한다. ([css-viewport ED](https://drafts.csswg.org/css-viewport/))
- `navigator.virtualKeyboard.overlaysContent`가 `true`이면 UA는 `interactive-widget`을 무시한다. ([css-viewport ED](https://drafts.csswg.org/css-viewport/))
- 이 문서는 Editor's Draft이므로 작업 중인 문서로만 인용해야 한다. ([css-viewport ED](https://drafts.csswg.org/css-viewport/))

### VirtualKeyboard API

- `navigator.virtualKeyboard.overlaysContent = true`로 설정하면 UA는 viewport와 visual viewport를 리사이즈하지 않아야 한다(MUST NOT). 이 setter는 최상위 보안 컨텍스트에서만 동작한다. ([W3C VirtualKeyboard](https://w3c.github.io/virtual-keyboard/))
- `boundingRect`는 키보드와 viewport의 교집합을 client 좌표로 알려 준다. `geometrychange` 이벤트는 이 교집합이 바뀔 때 발생한다. ([W3C VirtualKeyboard](https://w3c.github.io/virtual-keyboard/))
- CSS `env(keyboard-inset-top|right|bottom|left|width|height)`는 키보드 사각형을 viewport 가장자리 기준 inset으로 나타낸다. 기본값은 `0px`이다. ([W3C VirtualKeyboard](https://w3c.github.io/virtual-keyboard/))
- Chromium 94부터 데스크톱과 모바일에서 사용할 수 있다. Firefox와 Safari는 지원하지 않는다. ([developer.chrome.com – VirtualKeyboard API](https://developer.chrome.com/docs/web-platform/virtual-keyboard))
- Interop 2026에는 선정되지 않았다. 논의 과정에서 Chrome Android가 키보드 높이를 잘못 보고하고, 전환 중 화면 높이가 틀리며, 깜빡임이 있다는 문제가 제기되었다. ([web-platform-tests/interop#1016](https://github.com/web-platform-tests/interop/issues/1016))

## 2. 브라우저별 기본 동작(비교 환경)

Chrome 블로그는 브라우저를 두 그룹으로 나눈다. ([Prepare for viewport resize behavior changes](https://developer.chrome.com/blog/viewport-resize-behavior))

- **Group one(visual viewport만 리사이즈)**: iOS/iPadOS Safari, iOS Chrome·Edge, ChromeOS Chrome. 이 그룹에서는 `position: fixed` 요소가 제자리에 남아 **키보드에 가려질 수 있고**, viewport 단위 값은 바뀌지 않는다.
- **Group two(두 viewport 모두 리사이즈)**: Chrome·Firefox·Edge Android(108 이전), iOS Firefox. 이 그룹에서는 fixed 요소의 위치가 바뀌고 viewport 단위 값이 줄어든다.
- Chrome Android는 **108부터 Group one으로 바뀌었다**. 예전 동작이 필요하면 `interactive-widget=resizes-content`로 되돌릴 수 있다. ([같은 글](https://developer.chrome.com/blog/viewport-resize-behavior))
- 같은 글은 "These changes do not affect WebView"라고 적고 있다. Intent to Ship에는 다음 내용이 있다. ([blink-dev Intent to Ship](https://groups.google.com/a/chromium.org/g/blink-dev/c/ge7xTu-VhJ0/m/hq_kcusHAQAJ))
  > "There is no intended behavior change for Android WebView." / "The Android app is responsible for sizing the WebView and can implement either mode via windowSoftInputMode."

## 3. iOS WKWebView

### 3.1 `safe-area-inset-bottom`은 키보드가 열려도 0이 되지 않는다

- [WebKit 버그 217754 "Safe-area-inset-bottom still set when keyboard appears"](https://bugs.webkit.org/show_bug.cgi?id=217754)는 2020-10-15에 등록되었고 지금도 NEW 상태다(InRadar). 키보드가 열려도 홈 인디케이터용 하단 inset이 사라지지 않아 키보드 위에 빈 공간이 생긴다는 내용이다.
- 댓글에 두 가지 우회책이 있다. (c6) input/textarea에 `:focus-visible`이 있을 때 하단 padding을 빼는 CSS 방식인데, Chrome Android에서는 키보드를 닫은 뒤에도 `:focus-visible`이 남을 수 있다는 단점이 있다. (c8) `visualViewport` `resize`로 키보드가 열렸는지 추정해 class를 토글하고, inset용 CSS 변수를 0으로 바꾸는 방식이다. ([같은 버그](https://bugs.webkit.org/show_bug.cgi?id=217754))
- 이 문제가 iOS 26에서 고쳐졌는지는 **미확인**이다. 버그 상태가 NEW이므로 고쳐지지 않았다고 보는 편이 안전하다.

### 3.2 키보드 + 하단 inset의 스크롤 범위 버그

- [WebKit 버그 292603](https://bugs.webkit.org/show_bug.cgi?id=292603): iOS Safari와 WKWebView에서 키보드가 열려 있으면 페이지 하단에 홈 인디케이터 inset 정도 높이의 빈 영역이 생기고, 그 영역까지 스크롤된다. `100dvh`, `overflow`, `overscroll-behavior`로는 없앨 수 없다. iOS 15~18과 iOS 26.0, 26.6.2에서 재현이 보고되었다(2026-09-14 기준). 2026-09-15 댓글은 원인을 이렇게 추정한다. 클라이언트가 지정한 하단 obscured inset과 키보드 inset이 scroll view의 adjusted content inset에 **더해진다**는 것이다. 수정 PR로 WebKit PR #74122가 링크되어 있지만, 반영되었는지는 **미확인**이다.
- WKWebView 안에서 fixed 하단 요소 아래에 스크롤 가능한 빈 공간이 생긴다는 보고(2025-09-29 댓글)도 같은 버그에 달려 있다.

### 3.3 `interactive-widget`, VirtualKeyboard API 미지원

- [WebKit 버그 259770 "Implement the interactive-widget property"](https://bugs.webkit.org/show_bug.cgi?id=259770): 2023-08 등록, NEW 상태다. WebKit 엔지니어의 답변이나 반영된 패치는 없다. 2026년에도 "iOS는 키보드가 입력을 덮는다"는 댓글이 이어지고 있다.
- VirtualKeyboard API: Chromium Intent to Ship 기준으로 WebKit은 "No signal"이다. 구현 요청 버그는 WebKit 230225다. ([blink-dev Intent to Ship: VirtualKeyboard](https://groups.google.com/a/chromium.org/g/blink-dev/c/EF9fJTfqoGI/m/K2-QWCNxAgAJ), [interop#1016](https://github.com/web-platform-tests/interop/issues/1016)) Safari 지원은 없다. ([developer.chrome.com](https://developer.chrome.com/docs/web-platform/virtual-keyboard))
- 결론적으로 **iOS에서는 `env(keyboard-inset-*)`가 항상 fallback 값으로 계산된다.** 키보드 높이를 알 수 있는 웹 신호는 `visualViewport`(Group one 동작)뿐이다.

### 3.4 iOS 26 관련

- iOS 26의 SwiftUI `WebView`와 `WKWebView`에서 입력에 포커스한 뒤 웹뷰를 닫으면, **이후 네이티브 화면의 키보드가 하단 컨트롤을 가린다**(safe area 미반영)는 보고가 있다. DTS는 알려진 이슈로 보이며 수정에 Apple 측 변경이 필요할 수 있다고 답했다(FB20386257). 공식 우회책은 없다. ([Apple Developer Forums 802159](https://developer.apple.com/forums/thread/802159)) 네이티브 셸 쪽 문제이지만 하이브리드 앱에서 같은 증상이 나타날 수 있다.

## 4. Android WebView

Chromium의 [Android WebView & Insets 문서](https://chromium.googlesource.com/chromium/src.git/+/main/android_webview/docs/insets.md)가 핵심 1차 출처다.

| 마일스톤 | 변화 |
|---|---|
| M136 | `displayCutout()`, `systemBars()` inset을 `safe-area-inset-*`로 전달한다. 호환성 문제 때문에 **WebView가 전체 화면일 때만** 전달했다 |
| ~M138 | layout viewport와 visual viewport가 고정이고 크기가 같았다. 키보드에 가려진 영역을 웹에 알릴 방법이 없었다 |
| M139 | `ime()` inset을 지원한다. **IME inset은 visual viewport 크기를 직접 줄이고** layout viewport는 그대로 둔다. WebView 하단이 화면 밖으로 나가면 보이는 부분만큼 visual viewport를 줄인다(상·좌·우는 해당하지 않음) |
| M144 | 전체 화면이 아닌 WebView에도 inset을 지원한다. 단, **실제로 WebView 영역과 겹칠 때만** 0이 아닌 값을 준다 |

- "displayCutout and systemBars are forwarded to the web content via the safe-area-insets API. The ime insets directly affect the size of the Visual Viewport." ([insets.md](https://chromium.googlesource.com/chromium/src.git/+/main/android_webview/docs/insets.md))
  - 따라서 Android WebView에서 키보드는 `safe-area-inset-bottom` 값을 바꾸지 않고 `visualViewport.height`를 바꾼다. 키보드가 열렸을 때 `safe-area-inset-bottom`이 어떤 값이 되는지(내비게이션 바 inset이 남는지, 0이 되는지)는 문서에 없다. **미확인**이다.
- M139 이후에는 키보드 때문에 `resize` 이벤트가 더 많이 발생한다. Chromium 문서는 resize 핸들러가 포커스를 해제하는 사례를 경고한다. 그러면 "입력 포커스 → 키보드 표시 → resize → 포커스 해제 → 키보드 닫힘"의 루프가 생긴다. ([insets.md](https://chromium.googlesource.com/chromium/src.git/+/main/android_webview/docs/insets.md))
- 앱이 inset을 `WindowInsets.CONSUMED`로 반환하면 WebView는 safe area inset을 웹에 보내지 않는다. 앱이 padding을 적용하면서 inset도 그대로 넘기면 **여백이 이중으로** 생길 수 있다. 처리한 inset은 0으로 바꿔서 넘기는 방식이 권장된다. opt-out하면 키보드가 하단 입력을 가리고 `scrollIntoView`가 제대로 동작하지 않을 수 있다. ([insets.md](https://chromium.googlesource.com/chromium/src.git/+/main/android_webview/docs/insets.md))
- **`interactive-widget` meta를 Android WebView가 읽는지는 미확인이다.** Intent to Ship은 WebView는 앱이 `windowSoftInputMode`로 크기를 정한다고만 적었고, insets.md도 이 meta를 언급하지 않는다.
- **VirtualKeyboard API(`overlaysContent`, `env(keyboard-inset-*)`)가 Android WebView에서 실제로 동작하는지도 미확인이다.** Chromium 94부터 "desktop and mobile"이라는 표현만 1차 출처에서 확인했다. ([developer.chrome.com](https://developer.chrome.com/docs/web-platform/virtual-keyboard)) WebView 지원 여부를 따로 확인한 1차 출처는 찾지 못했다.
- Android 15(targetSdk 35+)에서는 edge-to-edge가 강제되어 앱이 inset을 직접 처리해야 한다. ([Android edge-to-edge](https://developer.android.com/develop/ui/views/layout/edge-to-edge)) 키보드는 `WindowInsetsCompat.Type.ime()`로 다룬다. 하위 호환을 위해 `android:windowSoftInputMode="adjustResize"`를 권장한다. ([Control and animate the software keyboard](https://developer.android.com/develop/ui/views/layout/sw-keyboard))

### 비교 환경: Chrome Android edge-to-edge

- Chrome 135부터 소형 화면에서 `viewport-fit=cover`일 때 viewport가 제스처 내비게이션 바 영역까지 확장된다. `safe-area-inset-bottom`은 "chin"이 사라질 때마다 동적으로 갱신된다. ([Chrome on Android edge-to-edge](https://developer.chrome.com/docs/css-ui/edge-to-edge)) 키보드가 열렸을 때 inset이 어떻게 되는지는 이 문서에 없다. **미확인**이다.

## 5. Flutter `resizeToAvoidBottomInset`과 WebView

- `Scaffold.resizeToAvoidBottomInset`(`bool?`)이 `true`이면 body와 floating 위젯이 화면 키보드를 피하도록 크기를 조정한다. 키보드 높이는 `MediaQueryData.viewInsets.bottom`에서 가져오고, **기본값은 `true`**다. ([api.flutter.dev](https://api.flutter.dev/flutter/material/Scaffold/resizeToAvoidBottomInset.html))
- 결과적으로 기본 설정에서는 body 안 WebView 위젯의 높이가 키보드만큼 준다. Android에서 키보드 위로 콘텐츠가 올라가지 않는 문제가 `resizeToAvoidBottomInset: true`로 해결된 사례가 있다. ([flutter/flutter#71364](https://github.com/flutter/flutter/issues/71364))

### 시나리오별 해석(1차 출처를 조합한 추론)

| 셸 설정 | Android WebView(M144+) | iOS WKWebView |
|---|---|---|
| `resizeToAvoidBottomInset: true`(기본) | WebView 영역 자체가 키보드 위로 줄어든다 → 웹에서는 **layout viewport가 줄어든다**(`resizes-content`와 같은 효과). WebView가 IME와 겹치지 않으므로 IME inset은 0이 된다(insets.md M144 설명에 근거). | WebView frame이 줄어든다. 다만 WKWebView 자체의 키보드 처리와 겹치는지(이중 보정)는 **미확인**이다 |
| `false` | WebView가 키보드 아래까지 덮는다 → M139+에서는 IME inset이 **visual viewport만** 줄인다(`resizes-visual`과 같은 효과). 단, Flutter platform view의 Android `WebView`가 window inset을 실제로 받는지는 **미확인**이다 | WebKit 기본 동작(Group one)을 따른다. fixed 하단 UI는 키보드에 가려진다 |

- `webview_flutter_wkwebview`는 iOS에서 `scrollView.contentInsetAdjustmentBehavior = .never`로 설정한다. frame이 바뀔 때마다 `contentInset`을 0으로, 또는 `adjustedContentInset`을 상쇄하는 값으로 맞춰서 "contentInset을 iOS 대신 Flutter가 제어"한다. ([WebViewProxyAPIDelegate.swift @10e39dd](https://github.com/flutter/packages/blob/10e39dd50f871716dc18eb061573bf3428210f4a/packages/webview_flutter/webview_flutter_wkwebview/darwin/webview_flutter_wkwebview/Sources/webview_flutter_wkwebview/WebViewProxyAPIDelegate.swift)) 그래서 iOS에서는 UIKit의 키보드·safe area 자동 inset 조정 대신 셸(Flutter 레이아웃)이 크기를 결정하는 구조다. 이 설정이 WebKit 내부의 visual viewport 축소에 어떤 영향을 주는지는 **미확인**이다.
- 관련 버그는 다음과 같다.
  - [flutter/flutter#140953](https://github.com/flutter/flutter/issues/140953)(OPEN, 2024-01): Android Flutter WebView에서 하단 입력이 키보드에 가려진다. 네이티브 Android WebView와 iOS Flutter에서는 재현되지 않았다. 메인테이너는 platform view가 크기를 잘못 보고하는 문제일 가능성을 언급했다.
  - [flutter/flutter#98090](https://github.com/flutter/flutter/issues/98090)(OPEN, 2022-02): iOS Flutter WebView에서만 키보드가 열릴 때 레이아웃이 reflow되며 배경이 드러난다. Safari와 순수 WKWebView에서는 재현되지 않는다.

## 6. 하단 고정 UI가 깨지는 흔한 원인

1. **iOS에서 남아 있는 `safe-area-inset-bottom`**: `padding-bottom: env(safe-area-inset-bottom)`을 준 입력 바가 키보드 위에서 홈 인디케이터 높이만큼 떠 있다. ([WebKit 217754](https://bugs.webkit.org/show_bug.cgi?id=217754))
2. **`resizes-visual` 기본 동작**: `position: fixed; bottom: 0`은 layout viewport 기준이라 키보드 뒤로 숨는다. iOS와 Chrome Android 108+에서 그렇다. ([Chrome blog](https://developer.chrome.com/blog/viewport-resize-behavior), [css-viewport](https://drafts.csswg.org/css-viewport/))
3. **보정이 이중으로 들어감**: 셸이 WebView를 줄였는데 웹이 `visualViewport`나 키보드 높이로 다시 띄우는 경우, 또는 Android 셸이 inset만큼 padding을 주면서 inset을 소비하지 않고 넘기는 경우다. ([insets.md](https://chromium.googlesource.com/chromium/src.git/+/main/android_webview/docs/insets.md))
4. **키보드가 열린 상태에서 생기는 하단 빈 스크롤 영역**(iOS, WKWebView 포함). ([WebKit 292603](https://bugs.webkit.org/show_bug.cgi?id=292603))
5. **resize 핸들러 부작용**: Android WebView M139+에서 키보드 때문에 resize가 늘어나고, 핸들러가 포커스를 잃게 만들면 키보드가 바로 닫힌다. ([insets.md](https://chromium.googlesource.com/chromium/src.git/+/main/android_webview/docs/insets.md))
6. **WebView 버전 차이**: 같은 Android 기기라도 WebView M139 이전에는 visual viewport가 줄지 않아 키보드 높이를 알 수 없고, M144 이전에는 전체 화면이 아닌 WebView에 inset이 전달되지 않는다. ([insets.md](https://chromium.googlesource.com/chromium/src.git/+/main/android_webview/docs/insets.md))
7. **`env(keyboard-inset-*)`에만 의존**: iOS에서는 지원되지 않아 항상 fallback 값이 쓰인다. ([developer.chrome.com](https://developer.chrome.com/docs/web-platform/virtual-keyboard))

## 7. 가이드에 반영할 핵심

- **키보드 높이를 알려 주는 공통 신호는 `visualViewport`뿐이다.** `env(keyboard-inset-*)`와 `interactive-widget`은 iOS에서 동작하지 않는다. Android WebView 지원도 미확인이므로 점진적 향상(fallback 필수)으로만 소개한다.
- **키보드가 열려 있는 동안에는 하단 `safe-area-inset-bottom` 여백을 웹이 직접 빼야 한다.** iOS는 이 값을 유지하고(217754), Android WebView는 IME를 safe area로 보고하지 않는다. "키보드 열림" 상태를 `visualViewport`로 판단해 하단 inset을 0으로 바꾸는 패턴이 가이드의 후보다(지도 #1의 "가상 키보드 처리 권장 패턴" 결정으로 넘긴다).
- **키보드 리사이즈의 1차 책임은 셸에 있다.** Flutter 기본값(`resizeToAvoidBottomInset: true`)에서는 WebView가 키보드 위로 줄어든다. 웹은 "WebView가 줄어든다"는 전제로 짜되, 셸이 `false`로 바꾸면 동작이 달라진다는 점을 **책임 분담표에 명시**해야 한다(셸 설정이 웹 레이아웃 계약의 일부다).
- **이중 보정 금지 규칙**: 셸이 키보드를 피하도록 WebView를 줄였다면 웹은 키보드 높이를 다시 더하지 않는다. Android 셸이 inset만큼 padding을 줬다면 그 inset은 0으로 바꿔 WebView에 넘긴다. ([insets.md](https://chromium.googlesource.com/chromium/src.git/+/main/android_webview/docs/insets.md))
- **Android WebView 버전 경계(M139, M144)**를 지원 매트릭스에 넣는다. Android 8.0+을 지원하므로 오래된 WebView에서는 키보드 높이를 알 수 없는 경우를 고려해야 한다.
- **resize 핸들러에서 포커스를 건드리지 않는다**(blur·리렌더로 input이 다시 마운트되는 경우 포함).
- 알려진 미해결 플랫폼 버그(WebKit 217754, 292603, 259770, Apple Forums 802159, flutter#140953, #98090)는 "알려진 문제" 절에 링크와 함께 정리한다.

## 미확인 목록

- iOS 26에서 키보드가 열렸을 때 `safe-area-inset-bottom` 값(버그가 NEW 상태라 유지된다고 추정)
- WebKit PR #74122(292603 수정안)가 반영되었는지
- Android WebView가 `interactive-widget` meta를 읽는지
- Android WebView의 VirtualKeyboard API 동작 여부
- 키보드가 열렸을 때 Android WebView·Chrome의 `safe-area-inset-bottom` 값
- Flutter platform view(Hybrid Composition / TLHC)의 Android `WebView`가 window inset(IME 포함)을 실제로 받는지
- iOS에서 Flutter가 WebView frame을 줄일 때 WKWebView 내부 키보드 처리와 이중 보정이 생기는지
