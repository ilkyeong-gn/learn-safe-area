# iOS WKWebView의 safe area 동작과 최근 변화

- 티켓: [#3](https://github.com/ilkyeong-gn/learn-safe-area/issues/3) (지도 [#1](https://github.com/ilkyeong-gn/learn-safe-area/issues/1))
- 조사일: 2026-10-23
- 대상: iOS 16+ WKWebView, Flutter `webview_flutter`(`webview_flutter_wkwebview`) 네이티브 셸
- 근거: Apple 개발자 문서, WebKit 소스(커밋 `e7c482a` 고정 링크), WebKit 블로그, bugs.webkit.org, flutter/packages 소스(커밋 `108dad0` 고정 링크)
- 표기: 1차 출처로 확인하지 못한 내용은 **미확인**으로 표시한다. 소스 줄 번호는 위 커밋 기준이다.

## 요약

1. WKWebView에서 `env(safe-area-inset-*)` 값은 WebKit 내부 값 `_computedUnobscuredSafeAreaInset`에서 나온다. 페이지가 `viewport-fit=cover`일 때만 이 값이 **WKWebView 자신의 `safeAreaInsets`**가 되고, 그 밖의 경우(기본값 `auto`, `contain`)에는 **항상 0**이다.
2. `viewport-fit`은 WKWebView 내부의 `scrollView.contentInsetAdjustmentBehavior`도 바꾼다. `cover`가 아니면 `.always`, `cover`이면 `.never`가 된다. 단, 앱이 이 속성을 직접 설정했다면 WebKit은 건드리지 않는다.
3. `webview_flutter_wkwebview`는 WebView를 만들 때 `scrollView.contentInsetAdjustmentBehavior = .never`로 고정한다. 그래서 Flutter 셸에서는 `viewport-fit`을 바꿔도 스크롤 뷰가 safe area만큼 자동으로 들여 쓰이지 않는다. 웹이 inset을 처리하려면 `viewport-fit=cover`와 `env()`를 함께 써야 한다.
4. `env()` 값은 **WebView의 frame이 화면의 unsafe 영역과 겹치는 만큼만** 0보다 커진다. 이는 UIKit `safeAreaInsets`의 정의에서 나온다. 네이티브가 WebView를 safe area 안에 배치하면 `cover`여도 `env()`는 0이다.
5. iOS 26에서 공개 API `WKWebView.obscuredContentInsets`가 추가됐다. 이 값을 설정하면 layout viewport가 줄고, `env()`는 safe area에서 이 값을 뺀 값(0 미만은 0)이 된다. `webview_flutter`는 이 API를 노출하지 않는다.

---

## 1. `env(safe-area-inset-*)` 값은 어떻게 정해지는가

### 1.1 값의 출처: WebKit 경로

`env()` 값은 다음 경로로 전달된다.

1. **CSS 쪽:** `EnvironmentVariables::updateSafeAreaInsetVariables()`가 `Page::unobscuredSafeAreaInsets()`의 top/right/bottom/left를 `safe-area-inset-*` 변수에 px 값으로 넣는다. 값이 바뀌면 `didChangeSafeAreaInsets()`가 스타일을 다시 계산한다. ([StyleEnvironmentVariables.cpp L179-193](https://github.com/WebKit/WebKit/blob/e7c482ada164db14120820ad2ab37dd44f1b1160/Source/WebCore/style/StyleEnvironmentVariables.cpp#L179-L193))
2. **Page:** `Page::setUnobscuredSafeAreaInsets()`는 값이 바뀌면 모든 문서의 환경 변수를 갱신한다. ([Page.cpp L4624-4634](https://github.com/WebKit/WebKit/blob/e7c482ada164db14120820ad2ab37dd44f1b1160/Source/WebCore/page/Page.cpp#L4624-L4634))
3. **UI 프로세스(WKWebView):** visible content rect를 갱신할 때 `[self _computedUnobscuredSafeAreaInset]`을 WebContent 프로세스로 보낸다. ([WKWebViewIOS.mm L3339-3347](https://github.com/WebKit/WebKit/blob/e7c482ada164db14120820ad2ab37dd44f1b1160/Source/WebKit/UIProcess/API/ios/WKWebViewIOS.mm#L3339-L3347))

### 1.2 `_computedUnobscuredSafeAreaInset`의 계산 규칙

[WKWebViewIOS.mm L873-893](https://github.com/WebKit/WebKit/blob/e7c482ada164db14120820ad2ab37dd44f1b1160/Source/WebKit/UIProcess/API/ios/WKWebViewIOS.mm#L873-L893)의 로직은 다음과 같다(iOS 기준, watchOS 분기는 뺐다).

```
if (SPI _setUnobscuredSafeAreaInsets: 로 값을 직접 넣었다면) return 그 값
if (!_safeAreaShouldAffectObscuredInsets) {          // == viewport-fit=cover
    insets = self.safeAreaInsets                     // WKWebView(UIView)의 safeAreaInsets
    if (obscured insets가 설정됨)                    // iOS 26 obscuredContentInsets 또는 SPI _obscuredInsets
        각 변 = max(0, insets.변 - obscuredInsets.변)
    return insets
}
return UIEdgeInsetsZero                              // viewport-fit=auto/contain
```

- `_safeAreaShouldAffectObscuredInsets`는 `_avoidsUnsafeArea`를 그대로 돌려준다(표준 content view일 때). ([L4700-4705](https://github.com/WebKit/WebKit/blob/e7c482ada164db14120820ad2ab37dd44f1b1160/Source/WebKit/UIProcess/API/ios/WKWebViewIOS.mm#L4700-L4705))
- `_avoidsUnsafeArea`는 메인 프레임이 commit될 때 `mainFrameCommitData.avoidsUnsafeArea`로 갱신되고([L1365](https://github.com/WebKit/WebKit/blob/e7c482ada164db14120820ad2ab37dd44f1b1160/Source/WebKit/UIProcess/API/ios/WKWebViewIOS.mm#L1365)), 초기값은 `YES`다. ([WKWebView.mm L680](https://github.com/WebKit/WebKit/blob/e7c482ada164db14120820ad2ab37dd44f1b1160/Source/WebKit/UIProcess/API/Cocoa/WKWebView.mm#L680))
- 이 값은 WebCore에서 `avoidsUnsafeArea = viewportFit != ViewportFit::Cover`로 정해진다. ([ViewportConfiguration.cpp L569](https://github.com/WebKit/WebKit/blob/e7c482ada164db14120820ad2ab37dd44f1b1160/Source/WebCore/page/ViewportConfiguration.cpp#L569))

정리하면 다음과 같다.

| 페이지의 `viewport-fit` | `_avoidsUnsafeArea` | `env(safe-area-inset-*)` |
|---|---|---|
| 미지정 / `auto` / `contain` | YES | **0** |
| `cover` | NO | `WKWebView.safeAreaInsets` (obscured insets가 있으면 그만큼 뺀 값) |

참고: 초기값이 `YES`이므로 첫 commit 전에는 `cover` 페이지도 `env()`가 0이다. 이 값이 첫 페인트에 실제로 영향을 주는지는 **미확인**이다.

### 1.3 `safeAreaInsets` 자체의 의미 (UIKit)

Apple 문서([UIView.safeAreaInsets](https://developer.apple.com/documentation/uikit/uiview/safeareainsets))의 정의는 다음과 같다.

- view controller의 root view에서는 상태 바, 다른 바, `additionalSafeAreaInsets`가 inset에 반영된다.
- 그 밖의 view에서는 **해당 view가 실제로 가려지는 부분만** 반영된다. 예를 들어 view가 superview의 safe area 안에 완전히 들어가 있으면 inset은 `0`이다.
- view가 view 계층에 없거나 화면에 보이지 않으면 `0`이다.

따라서 WKWebView의 `env()`는 기기 고유의 고정값이 아니다. **WebView의 frame이 화면의 unsafe 영역과 얼마나 겹치는지**에 따라 정해진다(§4 참고).

---

## 2. `viewport-fit`의 효과

### 2.1 스펙과 Safari에서의 원래 의미

WebKit 블로그 "Designing Websites for iPhone X"(iOS 11)의 설명은 다음과 같다. ([webkit.org/blog/7929](https://webkit.org/blog/7929/designing-websites-for-iphone-x/))

- `viewport-fit`의 기본값은 `auto`이고, 페이지를 자동으로 들여 쓴다. 들여 쓴 영역은 `<body>`/`<html>` 배경색으로 채운다.
- `cover`를 쓰면 자동 inset이 꺼지고 페이지가 화면 전체 크기로 레이아웃된다.
- 함수 이름은 iOS 11 시점의 `constant()`에서 iOS 11.2부터 `env()`로 바뀌었다.

### 2.2 WKWebView 안에서 실제로 바뀌는 것

`viewport-fit`이 바뀌면 `_setAvoidsUnsafeArea:`가 다음 세 가지를 수행한다. ([WKWebViewIOS.mm L3894-3910](https://github.com/WebKit/WebKit/blob/e7c482ada164db14120820ad2ab37dd44f1b1160/Source/WebKit/UIProcess/API/ios/WKWebViewIOS.mm#L3894-L3910))

1. scroll view content inset을 다시 계산하고, 필요하면 view layout size를 다시 보낸다.
2. `_updateScrollViewInsetAdjustmentBehavior`를 호출한다(§3).
3. visible content rect를 갱신해서 `env()` 값을 다시 보낸다.

obscured inset 계산(`_computedObscuredInset`)에서는 `cover`가 아닐 때 scroll view의 system content inset(= safe area)을 `_effectiveObscuredInsetEdgesAffectedBySafeArea` 변에 더한다. `cover`일 때는 0이다. ([L836-851](https://github.com/WebKit/WebKit/blob/e7c482ada164db14120820ad2ab37dd44f1b1160/Source/WebKit/UIProcess/API/ios/WKWebViewIOS.mm#L836-L851)) 이 변 집합의 기본값은 **top | left | right**이고 bottom은 빠져 있다. ([WKWebView.mm L674](https://github.com/WebKit/WebKit/blob/e7c482ada164db14120820ad2ab37dd44f1b1160/Source/WebKit/UIProcess/API/Cocoa/WKWebView.mm#L674)) 즉 `auto`일 때도 WebKit은 하단 safe area를 obscured로 취급하지 않는다. 이것이 웹 화면에서 어떻게 보이는지(하단 콘텐츠가 홈 인디케이터 아래까지 스크롤되는지)는 실기기 검증이 필요하며 **미확인**이다.

---

## 3. `UIScrollView.contentInsetAdjustmentBehavior`

### 3.1 API 정의

Apple 문서([contentInsetAdjustmentBehavior](https://developer.apple.com/documentation/uikit/uiscrollview/contentinsetadjustmentbehavior-swift.property), [ContentInsetAdjustmentBehavior](https://developer.apple.com/documentation/uikit/uiscrollview/contentinsetadjustmentbehavior-swift.enum))의 정의는 다음과 같다.

- 이 속성은 safe area inset을 scroll view의 content 영역에 어떻게 반영할지 정한다. UIScrollView의 기본값은 `.automatic`이다.
- `.always`: safe area inset을 항상 content 조정에 포함한다.
- `.never`: scroll view inset을 조정하지 않는다.
- `.scrollableAxes`: 스크롤 가능한 방향에서만 조정한다.
- 결과는 `adjustedContentInset`(content inset과 safe area에서 파생된 값)으로 나타난다.

### 3.2 WebKit이 이 속성을 다루는 방식

```objc
- (void)_updateScrollViewInsetAdjustmentBehavior {
    if (![_scrollView _contentInsetAdjustmentBehaviorWasExternallyOverridden])
        [_scrollView _setContentInsetAdjustmentBehaviorInternal:
            self._safeAreaShouldAffectObscuredInsets ? UIScrollViewContentInsetAdjustmentAlways
                                                     : UIScrollViewContentInsetAdjustmentNever];
}
```

([WKWebViewIOS.mm L3877-3884](https://github.com/WebKit/WebKit/blob/e7c482ada164db14120820ad2ab37dd44f1b1160/Source/WebKit/UIProcess/API/ios/WKWebViewIOS.mm#L3877-L3884))

- 앱이 이 속성을 **건드리지 않으면**, WebKit이 `viewport-fit`에 맞춰 자동으로 바꾼다. `cover`가 아니면 `.always`(스크롤 뷰가 safe area만큼 들여 씀), `cover`이면 `.never`다.
- 앱이 `webView.scrollView.contentInsetAdjustmentBehavior`를 **직접 설정하면**("externally overridden"), WebKit은 그 값을 유지한다.
- **`env()` 값은 이 속성의 영향을 받지 않는다.** §1.2의 계산은 `viewport-fit`(`_avoidsUnsafeArea`)과 `self.safeAreaInsets`만 사용한다.

### 3.3 Flutter `webview_flutter_wkwebview`의 설정

```swift
super.init(frame: frame, configuration: configuration)
#if os(iOS)
  scrollView.contentInsetAdjustmentBehavior = .never
  scrollView.automaticallyAdjustsScrollIndicatorInsets = false
#endif
```

([WebViewProxyAPIDelegate.swift L11-22](https://github.com/flutter/packages/blob/108dad0d562e9368a45b6b5cdcc26c137bc071c4/packages/webview_flutter/webview_flutter_wkwebview/darwin/webview_flutter_wkwebview/Sources/webview_flutter_wkwebview/WebViewProxyAPIDelegate.swift#L11-L22))

플러그인이 생성 시점에 `.never`를 외부에서 설정하므로, WebKit은 `viewport-fit`에 따라 이 값을 바꾸지 않는다. 결과는 다음과 같다.

| 조합 (Flutter 셸) | 스크롤 뷰 자동 inset | `env()` |
|---|---|---|
| `viewport-fit=auto` | 없음 (`.never` 고정) | 0 |
| `viewport-fit=cover` | 없음 | WKWebView `safeAreaInsets` |

`auto`에서도 WebKit 내부 obscured inset 계산(§2.2)은 여전히 실행된다. 하지만 이것은 `[_scrollView _contentScrollInset]`과 UIKit private 메서드 `_edgesApplyingSafeAreaInsetsToContentInset`에 의존한다([L2597-2604](https://github.com/WebKit/WebKit/blob/e7c482ada164db14120820ad2ab37dd44f1b1160/Source/WebKit/UIProcess/API/ios/WKWebViewIOS.mm#L2597-L2604)). 그래서 behavior가 `.never`일 때 이 값이 0이 되는지는 **미확인**이다. 실무상 Flutter 셸에서 WebView가 화면 끝까지 닿는다면 `auto` 페이지는 노치나 홈 인디케이터에 가려질 수 있다고 봐야 한다(실기기 확인 필요).

플러그인 CHANGELOG 2.9.1에는 "Fixes regression where the behavior for the `UIScrollView` insets were removed."라는 항목이 있다. 이 동작은 플러그인이 의도적으로 유지하는 것이다. ([CHANGELOG.md](https://github.com/flutter/packages/blob/main/packages/webview_flutter/webview_flutter_wkwebview/CHANGELOG.md))

---

## 4. WebView frame과 safe area의 관계

§1.3의 UIKit 정의와 §1.2의 계산을 합치면 다음 결론이 나온다.

| 네이티브 배치 | WKWebView `safeAreaInsets` | `cover` 페이지의 `env()` |
|---|---|---|
| WebView가 화면 전체(상태 바, 홈 인디케이터 영역 포함)를 덮음 | 기기 safe area와 같음 | 기기 safe area 값 |
| WebView를 safe area 안에만 배치 (Flutter에서 `SafeArea` 위젯으로 감싼 경우 등) | 0 | 0 |
| 위쪽만 safe area 안에 배치하고 아래는 화면 끝까지 | top 0, bottom만 값 있음 | bottom만 값 있음 |
| 네이티브 앱 바나 탭 바 아래에 배치 | 해당 변 0 | 해당 변 0 |

- Flutter의 platform view(WKWebView)가 UIKit 계층에서 Flutter 위젯 배치와 같은 위치에 놓이고, 그에 따라 `safeAreaInsets`가 계산된다는 점은 Flutter 쪽 소스로 검증하지 않았다(**미확인**, Flutter 티켓에서 다룰 사항).
- 같은 셸에서 웹이 inset을 처리할지 네이티브가 처리할지를 정하는 일은 결국 **WebView frame을 어디까지 펼치느냐**의 결정이다. 웹이 처리하려면 frame을 끝까지 펼치고 `cover`를 쓴다. 네이티브가 처리하려면 frame을 safe area 안에 두고, 웹의 `env()`는 0이 된다.

---

## 5. 가로 모드, 회전, Dynamic Island

- **가로 모드:** iPhone X 계열은 세로 모드에서 left inset이 0이지만, 가로 모드에서는 센서 하우징 때문에 left/right inset이 생긴다. 블로그는 `max()`로 기본 여백과 합치는 방법을 권장한다. ([webkit.org/blog/7929](https://webkit.org/blog/7929/designing-websites-for-iphone-x/)) WKWebView에서도 값은 `safeAreaInsets`를 그대로 쓰므로 같은 원리가 적용된다(§1.2).
- **회전 중 갱신:** UIKit이 safe area 변경을 알리면 WKWebView는 `safeAreaInsetsDidChange`에서 visible content rect 갱신을 예약한다. ([L3088-3094](https://github.com/WebKit/WebKit/blob/e7c482ada164db14120820ad2ab37dd44f1b1160/Source/WebKit/UIProcess/API/ios/WKWebViewIOS.mm#L3088-L3094)) 애니메이션이 있는 크기 변경(회전) 경로인 `dynamicViewportSizeUpdate`에도 `unobscuredSafeAreaInsets`가 함께 전달된다. ([L5081-5088](https://github.com/WebKit/WebKit/blob/e7c482ada164db14120820ad2ab37dd44f1b1160/Source/WebKit/UIProcess/API/ios/WKWebViewIOS.mm#L5081-L5088)) 그래서 회전 후 `env()` 갱신은 UIKit의 `safeAreaInsets` 갱신 시점을 따른다.
- **iOS 26 회전 관련 보고:** Apple Developer Forums에 "iOS 26에서 가로 회전이나 상태 바 표시가 바뀐 뒤 `safeAreaInsets.top`이 즉시 갱신되지 않는다"는 사용자 보고가 여러 건 있다(FB20281983 등). Apple은 공식으로 확인하지 않았다. ([forums/thread/798014](https://developer.apple.com/forums/thread/798014)) 이 버그가 있다면 WKWebView의 `env()`에도 그대로 전파될 것으로 추정되지만 **미확인**이다.
- **Dynamic Island:** WebKit 소스에는 Dynamic Island를 별도로 다루는 safe area 분기가 없다. 값은 UIKit `safeAreaInsets`의 top에 포함된 형태로만 들어온다(§1.2). 기종별 정확한 pt 값은 Apple 1차 문서에서 확인하지 못했다(**미확인**).

---

## 6. iOS 16 → 26 변화

| 버전 | 변화 | 출처 |
|---|---|---|
| iOS 11 / 11.2 | `viewport-fit`, `constant()` 도입. 11.2부터 `env()`로 교체 | [webkit.org/blog/7929](https://webkit.org/blog/7929/designing-websites-for-iphone-x/) |
| iOS 15.5 | `setMinimumViewportInset(_:maximumViewportInset:)` 추가. 앱 UI 변화에 맞춰 `lv*`/`sv*` 단위 값을 공급한다. `env()`와는 별개 | [WKWebView.h L674-690](https://github.com/WebKit/WebKit/blob/e7c482ada164db14120820ad2ab37dd44f1b1160/Source/WebKit/UIProcess/API/Cocoa/WKWebView.h#L674-L690), [Apple 문서](https://developer.apple.com/documentation/webkit/wkwebview/setminimumviewportinset(_:maximumviewportinset:)) |
| iOS 16 ~ 18 | WKWebView의 safe area나 `env()` 계산 방식 변경을 다룬 공식 문서나 릴리스 노트를 찾지 못함 | **미확인** |
| iOS 26 | 공개 API `obscuredContentInsets` 추가(§6.1) | [WKWebView.h L748-761](https://github.com/WebKit/WebKit/blob/e7c482ada164db14120820ad2ab37dd44f1b1160/Source/WebKit/UIProcess/API/Cocoa/WKWebView.h#L748-L761), [Safari 26.0 블로그](https://webkit.org/blog/17333/webkit-features-in-safari-26-0/), [Apple 문서](https://developer.apple.com/documentation/webkit/wkwebview/obscuredcontentinsets) (iOS 26.0+) |
| iOS 26 | Safari의 떠 있는(Liquid Glass) 하단 바 때문에 fixed 요소의 위치나 하단 색 채움에 회귀가 생김. "WKWebView 전반(Safari와 임베딩 앱 포함)"에 대한 시스템 컴포넌트 수정이 이루어졌다고 WebKit 엔지니어가 언급 | [bug 297779](https://bugs.webkit.org/show_bug.cgi?id=297779) (Comment 10, 13) |
| iOS 26.1 | "Fixed a bottom gap appearing on layouts with viewport-sized fixed containers on iOS." | [Safari 26.1 블로그](https://webkit.org/blog/17541/webkit-features-for-safari-26-1/) |

### 6.1 `obscuredContentInsets` (iOS 26)

헤더 주석의 요지는 다음과 같다.

- web view 좌표계 기준으로 각 변의 inset을 지정한다. 이 값만큼 **layout viewport의 경계가 줄어든다**.
- 가려진 영역은 클라이언트가 관리하는 UI(내비게이션 바, 버튼 등)가 덮어야 한다.
- web view는 inset이 0이 아닌 변 근처에서 fixed/sticky 요소의 렌더링을 자동으로 조정한다.
- 음수는 허용하지 않고, 기본값은 0이다.

([WKWebView.h L748-761](https://github.com/WebKit/WebKit/blob/e7c482ada164db14120820ad2ab37dd44f1b1160/Source/WebKit/UIProcess/API/Cocoa/WKWebView.h#L748-L761))

구현은 다음과 같다.

- setter는 음수 값이면 `NSInvalidArgumentException`을 던진다. 그 외에는 내부 `_setObscuredInsetsInternal:`을 호출해 기존 SPI `_obscuredInsets`와 같은 저장소를 쓴다. ([WKWebView.mm L4061-4087](https://github.com/WebKit/WebKit/blob/e7c482ada164db14120820ad2ab37dd44f1b1160/Source/WebKit/UIProcess/API/Cocoa/WKWebView.mm#L4061-L4087), [WKWebViewIOS.mm L270-277](https://github.com/WebKit/WebKit/blob/e7c482ada164db14120820ad2ab37dd44f1b1160/Source/WebKit/UIProcess/API/ios/WKWebViewIOS.mm#L270-L277))
- 따라서 이 값을 설정하면 §1.2에 따라 `cover` 페이지의 `env()`는 `max(0, safeAreaInsets - obscuredContentInsets)`가 된다. 네이티브 바가 safe area를 이미 덮고 있다면 웹이 그만큼 중복으로 여백을 두지 않게 된다.
- `obscuredContentInsets`와 `minimumViewportInset`은 flutter/packages 저장소에서 검색되지 않는다. 즉 `webview_flutter`는 이 API를 노출하지 않는다(2026-10-23 GitHub code search 기준).

---

## 7. Safari와의 차이

| 항목 | Safari (비교 환경) | 앱 WKWebView (하이브리드웹) |
|---|---|---|
| 엔진 | WKWebView 기반. bug 297779에서 WebKit 엔지니어가 "WKWebView in general (including Safari and embedding apps)"라고 표현 | 같음 |
| `env()` 값 결정 | 같은 `_computedUnobscuredSafeAreaInset` 경로를 거치지만, SPI `_setUnobscuredSafeAreaInsets:`/`_setObscuredInsets:`로 값을 직접 넣을 수 있다([WKWebViewPrivate.h L751-756](https://github.com/WebKit/WebKit/blob/e7c482ada164db14120820ad2ab37dd44f1b1160/Source/WebKit/UIProcess/API/Cocoa/WKWebViewPrivate.h#L751-L756)). Safari가 이 SPI를 실제로 쓰는지는 Safari가 비공개라서 **미확인** | 공개 API만 쓰면 WebView 자신의 `safeAreaInsets`(iOS 26+는 `obscuredContentInsets`를 뺀 값) |
| 브라우저 UI | 주소창과 툴바가 동적으로 줄고 늘며, iOS 26에서는 떠 있는 Liquid Glass 바와 바 아래 색 채움 휴리스틱이 있다(bug 297779 Comment 23, Simon Fraser: "is a heuristic, and could be improved") | 브라우저 UI 없음. 상단과 하단은 전적으로 네이티브 셸이 결정 |
| `contentInsetAdjustmentBehavior` | 앱이 아니므로 해당 없음(Safari 내부 설정은 **미확인**) | 셸이 설정할 수 있다. `webview_flutter`는 `.never`로 고정 |
| `viewport-fit=cover`의 체감 효과 | iOS 26 Safari 세로 모드에서는 `bottom: 0` 동작에 영향이 없다는 보고가 있다(bug 297779 Comment 28, 비 Apple 작성) | WebView가 화면 끝까지 닿으면 `env()`가 0이 아닌 값을 가짐 |

---

## 8. 가이드에 반영할 핵심

1. **`env()`를 쓰려면 `viewport-fit=cover`가 필수다.** WKWebView는 `cover`가 아니면 `env(safe-area-inset-*)`를 항상 0으로 준다(WebKit 소스로 확인). Next.js에서는 `export const viewport = { viewportFit: 'cover' }`로 지정한다.
2. **`env()` 값은 WebView frame과 unsafe 영역이 겹치는 양이다.** 웹과 네이티브의 책임 분담은 "WebView를 화면 끝까지 펼칠 것인가"라는 네이티브 결정에서 시작한다. 셸이 `SafeArea`로 감싸면 웹의 `env()`는 0이 되고, 웹 코드가 같아도 브라우저와 앱에서 결과가 달라진다. 이것은 버그가 아니라 정상 동작이다.
3. **`webview_flutter`는 `contentInsetAdjustmentBehavior = .never`로 고정한다.** 그래서 `viewport-fit=auto`일 때 Safari처럼 자동으로 들여 쓰이는 동작을 기대하면 안 된다. Flutter 셸에서 WebView를 끝까지 펼친다면 웹은 `cover` + `env()` 조합을 써야 한다.
4. **iOS 26 `obscuredContentInsets`를 쓰면 네이티브 바가 덮는 만큼 `env()`가 줄어든다.** 다만 `webview_flutter`는 이 API를 노출하지 않으므로, 쓰려면 플랫폼 코드를 직접 작성하거나 플러그인을 확장해야 한다. 책임 분담표의 선택지로 기록한다.
5. **회전과 iOS 26 회귀:** `env()`는 UIKit `safeAreaInsets` 갱신을 따른다. iOS 26에서 회전이나 상태 바 변경 뒤 safe area가 늦게 갱신된다는 미확인 보고가 있으므로, 가이드의 "흔한 문제" 후보로 기록하고 실기기 검증 대상으로 남긴다.
6. Safari에서 본 결과(떠 있는 바, 색 채움 휴리스틱, `bottom: 0` 위치)를 앱 WebView에 그대로 일반화하지 않는다. 앱에서는 브라우저 UI가 없고 네이티브 셸이 가장자리를 결정한다.

## 미확인 목록

- Flutter + `viewport-fit=auto`에서 WebKit 내부 obscured inset이 실제로 0인지(UIKit private `_edgesApplyingSafeAreaInsetsToContentInset`의 동작)
- `auto`에서 bottom 변이 obscured 계산에서 빠질 때(§2.2) 화면에 보이는 결과
- Flutter platform view(WKWebView)의 `safeAreaInsets`가 Flutter 위젯 배치를 정확히 반영하는지
- iOS 16~18 사이 WKWebView safe area 관련 변경 여부
- Dynamic Island 기종별 safe area top 값
- Safari가 SPI(`_unobscuredSafeAreaInsets`, `_obscuredInsets`)를 쓰는지 여부
- iOS 26 회전/상태 바 safe area 갱신 지연이 WKWebView `env()`에 미치는 영향
