# Android WebView의 safe area 동작과 edge-to-edge 강제

> 티켓: #4 · 지도: #1 · 조사일: 2026-10-23
> 가정: Flutter `webview_flutter` 네이티브 셸 + Next.js 16, 최소 Android 8.0(API 26)
> 표기: **[소스 분석]** 은 Chromium 소스를 직접 읽고 내린 해석이라 공식 문서에 명시되지 않은 내용이다. **미확인** 은 1차 출처로 확인하지 못한 내용이다.

## 요약

- Android System WebView는 **M136(2025-04 stable)부터** `displayCutout()`·`systemBars()` inset을 `env(safe-area-inset-*)`로 넘기기 시작했다. M136~M143에서는 **화면 전체를 덮는(fullscreen) WebView에만** 적용되고, **M144(2026-01 stable)부터 모든 WebView**에 적용된다. 그 전에는 WebView에서 `env(safe-area-inset-*)`가 사실상 항상 `0px`였다(Chromium 버그 40699457, 2020년 보고).
- 값은 "시스템 UI가 WebView의 화면 영역과 **실제로 겹치는 만큼**"만 들어온다. 네이티브 셸이 이미 inset만큼 padding을 줬거나 inset을 소비(consume)했다면 웹은 `0`을 받는다. 즉 **WebView에서 safe area 값은 셸의 inset 처리에 따라 정해진다.**
- 소스상 WebView는 `viewport-fit` 값을 safe area 계산에 쓰지 않는다 **[소스 분석]**. iOS나 Chrome 브라우저와 달리 `viewport-fit=cover`가 없어도 inset이 들어올 수 있다.
- Android 15에서 targetSdk 35 이상 앱은 edge-to-edge가 기본이 되고, Android 16에서 targetSdk 36 이상 앱은 opt-out(`windowOptOutEdgeToEdgeEnforcement`)이 비활성화된다. Google Play는 2026-08-31부터 신규 앱과 업데이트에 targetSdk 36을 요구한다. 그래서 셸이 inset을 처리하지 않으면 WebView가 상태 바와 내비게이션 바 아래로 그려진다.
- 최소 지원 OS 때문에 생기는 함정이 있다. WebView M139부터는 Android 10(API 29) 이상만 지원하므로 **Android 8.0~9 기기의 WebView는 M138에서 멈춘다.** API 26~27에서는 WebView가 safe area를 아예 계산하지 않고, API 28에서는 fullscreen WebView일 때만 값이 들어온다 **[소스 분석]**.

---

## 1. WebView에서 `env(safe-area-inset-*)` 지원 이력

| 시점 | 변화 | 출처 |
|---|---|---|
| 2020-06 | 버그 보고 "safe-area-inset-* values are always 0px in webview"(보고 버전 88, 컴포넌트 Mobile>WebView). | [crbug 40699457](https://issues.chromium.org/issues/40699457) |
| M136 (stable 2025-04-29) | `displayCutout()`와 `systemBars()`를 CSS safe-area-insets로 전달하되 **fullscreen WebView만** 대상. CL "[WebView] Include system bars in safe-area insets"(2025-03-19 머지). M136 브랜치에서 `WebViewSafeAreaIncludesSystemBars` 기능은 기본 활성. | [Understand window insets in WebView](https://developer.android.com/develop/ui/views/layout/webapps/understand-window-insets), [CL 6295663](https://chromium-review.googlesource.com/c/chromium/src/+/6295663), [aw_features.cc @7103](https://chromium.googlesource.com/chromium/src/+/refs/branch-heads/7103/android_webview/common/aw_features.cc), [M136 일정](https://chromiumdash.appspot.com/fetch_milestone_schedule?mstone=136) |
| M137 | Chrome 브라우저의 edge-to-edge 기능(`DrawCutoutEdgeToEdge`)을 WebView에서는 끔. CL 설명: "Safe area insets are not handled correctly on WebView yet". | [CL 6449517](https://chromium-review.googlesource.com/c/chromium/src/+/6449517) |
| M139 (stable 2025-08-05) | `ime()` inset을 visual viewport 크기 조정으로 반영(모든 WebView). 이전에는 layout viewport와 visual viewport가 고정되어 있었다. 같은 마일스톤부터 WebView 최소 OS가 Android 10(API 29)으로 올라감(M138 브랜치 `default_min_sdk_version = 26`, M139 브랜치 `= 29`). | [Understand window insets](https://developer.android.com/develop/ui/views/layout/webapps/understand-window-insets), [config.gni @7204](https://chromium.googlesource.com/chromium/src/+/refs/branch-heads/7204/build/config/android/config.gni), [config.gni @7258](https://chromium.googlesource.com/chromium/src/+/refs/branch-heads/7258/build/config/android/config.gni), [CL 6598935](https://chromium-review.googlesource.com/c/chromium/src/+/6598935) |
| M140 (stable 2025-09-02) | 버그 수정. WindowInsets 리스너가 여러 곳에서 등록되며 서로 덮어써서 "either IME insets wouldn't get calculated or display cutout insets wouldn't" 문제가 있었고, 이를 하나로 통합해 M140에 머지했다. 따라서 **M139에서는 safe area 값이 빠질 수 있었다.** (Capacitor 문서가 "WebView < 140"을 버그 버전으로 보는 근거로 보인다.) | [CL 6875265](https://chromium-review.googlesource.com/c/chromium/src/+/6875265), [CL 6896587 (M140 머지)](https://chromium-review.googlesource.com/c/chromium/src/+/6896587) |
| M144 (stable 2026-01-13) | `displayCutout()`·`systemBars()` 지원을 **fullscreen 여부와 관계없이 모든 WebView**로 확대. 키보드가 docked 상태로 떠 있으면 하단 `systemBars` inset을 0으로 바꿔 전달(CL 2025-11-06 머지, M144 브랜치 전). 관련 기능 플래그(`WebViewSafeAreaIncludesSystemBars`, `WebViewReportImeInsets`, `WebViewUseViewPositionObserverForInsets`)는 모두 기본 활성 상태였고 이후 제거됐다. | [Understand window insets](https://developer.android.com/develop/ui/views/layout/webapps/understand-window-insets), [CL 7122025](https://chromium-review.googlesource.com/c/chromium/src/+/7122025), [CL 7138339](https://chromium-review.googlesource.com/c/chromium/src/+/7138339), [M144 일정](https://chromiumdash.appspot.com/fetch_milestone_schedule?mstone=144) |

### 1.1 동작 원리 (현재 main 기준)

- WebView가 반영하는 inset은 세 가지다. `displayCutout`·`systemBars`는 safe-area-insets API로 웹에 전달되고, `ime`는 visual viewport 크기를 직접 바꾼다. ([insets.md](https://chromium.googlesource.com/chromium/src.git/+/main/android_webview/docs/insets.md))
- 겹침 규칙: "WebView should receive non-zero inset values only when system UI elements … directly overlap with the WebView's screen bounds." 화면 가운데에 있어 시스템 바에 닿지 않는 WebView는 0을 받는다. ([Understand window insets](https://developer.android.com/develop/ui/views/layout/webapps/understand-window-insets))
- 구현에서는 `AwDisplayCutoutController.onApplyWindowInsets`가 `displayCutout() | systemBars()` 합집합을 safe area로 계산하고, IME 하단 inset이 0이 아니면 safe area의 bottom을 0으로 만든다. 그 값을 dip 단위로 환산해 `WebContents.setDisplayCutoutSafeArea`로 보낸다. ([AwDisplayCutoutController.java](https://chromium.googlesource.com/chromium/src/+/main/android_webview/java/src/org/chromium/android_webview/AwDisplayCutoutController.java))
- IME로 인한 visual viewport 축소는 **하단에만** 적용된다. 좌·우·상단이 겹치는 경우는 지원하지 않는다. ([Understand window insets](https://developer.android.com/develop/ui/views/layout/webapps/understand-window-insets))

### 1.2 `viewport-fit`의 역할 [소스 분석]

- Blink의 `ViewportData`는 `viewport-fit` 값이 바뀌면 브라우저 프로세스에 `NotifyViewportFitChanged`로 알리기만 한다. ([viewport_data.cc](https://chromium.googlesource.com/chromium/src/+/main/third_party/blink/renderer/core/frame/viewport_data.cc))
- content 레이어의 `SafeAreaInsetsHostImpl::SetDisplayCutoutSafeArea`는 `viewport-fit` 값을 보지 않고 받은 inset을 그대로 프레임에 보낸다. `viewport-fit`은 `WebContentsObserver`에 알리는 데만 쓰인다. ([safe_area_insets_host_impl.cc](https://chromium.googlesource.com/chromium/src/+/main/content/browser/display_cutout/safe_area_insets_host_impl.cc))
- Chrome 브라우저에서는 이 알림을 받아 `viewport-fit=cover`면 Activity 창을 `LAYOUT_IN_DISPLAY_CUTOUT_MODE_SHORT_EDGES`로, `contain`이면 `NEVER`로 바꾼다. ([display_cutout.md](https://chromium.googlesource.com/chromium/src.git/+/main/docs/ui/android/display_cutout.md)) 반면 `AwContents.java`에는 `viewport-fit`을 처리하는 코드가 없다. ([AwContents.java](https://chromium.googlesource.com/chromium/src/+/main/android_webview/java/src/org/chromium/android_webview/AwContents.java))
- 결론적으로 WebView에서는 **`viewport-fit`이 `auto`/`contain`이어도 inset 값이 들어올 수 있다.** 실제로 Cordova 계열 플러그인 저장소에도 같은 현상을 보고한 사례가 있다(2차 출처라 참고만: [niklasmerz/edge](https://github.com/niklasmerz/edge)). Capacitor 문서는 최신 WebView가 "adheres to the viewport-fit meta tag"라고 쓰지만, 이것이 Capacitor 자체 처리를 말하는 것인지는 **미확인**이다. ([Capacitor SystemBars](https://unpkg.com/@capacitor/core@8.5.2/system-bars.md))
- 실무에서는 iOS·Chrome과 동작을 맞추기 위해 `viewport-fit=cover`를 선언하는 편이 안전하다. 다만 Android WebView에서 cover 여부로 값을 켜고 끌 수 있다고 기대하면 안 된다.

### 1.3 Android 8.0~9(API 26~28) 기기 [소스 분석]

- M138 브랜치의 `AwDisplayCutoutController.maybeCreate`는 `SDK_INT >= P`(API 28)일 때만 컨트롤러를 만든다. 따라서 **API 26~27 WebView는 safe area를 계산하지 않아 `env()`가 항상 0이다.** ([AwDisplayCutoutController.java @7204](https://chromium.googlesource.com/chromium/src/+/refs/branch-heads/7204/android_webview/java/src/org/chromium/android_webview/AwDisplayCutoutController.java))
- API 28(Android 9)은 M138이 마지막 WebView이므로 M136~M138의 동작(fullscreen WebView에만 적용, IME로 visual viewport가 줄지 않음)에 머문다.
- Chrome 브라우저도 138이 Android 8/9를 지원하는 마지막 버전이다. M139부터 Android 10 이상이 필요하다는 공지는 여러 매체가 보도했지만, Google 공식 공지 원문 URL은 **미확인**이다. 위 내용의 근거는 config.gni 브랜치 비교다.

## 2. Display cutout 모드 (`layoutInDisplayCutoutMode`)

출처: [Display cutout 가이드](https://developer.android.com/develop/ui/views/layout/display-cutout)

| 값 | 동작 | targetSdk 35 이상 + Android 15 |
|---|---|---|
| `DEFAULT` | cutout이 시스템 바 안에 있으면 그 영역까지 그리고, 아니면 겹치지 않는다(가로 모드에서 레터박스가 생길 수 있음). | non-floating 창에서는 `ALWAYS`로 해석 |
| `SHORT_EDGES` | 세로·가로 모두 짧은 변의 cutout 영역까지 그린다. 긴 변 쪽으로는 확장하지 않는다. | `ALWAYS`로 해석 |
| `NEVER` | cutout 영역과 절대 겹치지 않는다. | `ALWAYS`로 해석 |
| `ALWAYS` | 항상 cutout 영역까지 확장한다. | 유일하게 허용되는 모드 |

- Android는 API 28부터 display cutout을 지원한다. 제조사가 8.1 이하에서 독자적으로 지원한 경우도 있다(같은 문서).
- WebView 쪽 의미: 창이 cutout까지 그려지고 WebView가 그 영역과 겹치면 `env(safe-area-inset-*)`에 cutout 높이가 반영된다. 가로 모드에서는 cutout이 좌·우 변으로 오므로 `safe-area-inset-left/right`가 0이 아닐 수 있다.
- 테스트 방법: 개발자 옵션 > Drawing > "Simulate a display with a cutout"(같은 문서).

## 3. Android 15 edge-to-edge 강제와 Android 16 opt-out 제거

### Android 15 (API 35)
출처: [Android 15 behavior changes](https://developer.android.com/about/versions/15/behavior-changes-15)

- "Apps are edge-to-edge by default on devices running Android 15 if the app is targeting Android 15 (API level 35)."
- 상태 바: 기본이 투명이다. 상단 offset이 꺼져 inset을 적용하지 않으면 콘텐츠가 상태 바 아래로 그려진다. `setStatusBarColor`는 deprecated이며 효과가 없다.
- 제스처 내비게이션 바: 기본이 투명이다. 하단 offset이 꺼진다. `setNavigationBarColor`가 효과 없다.
- 3버튼 내비게이션: 기본 80% 불투명이다. `navigationBarContrastEnforced`가 기본 true여서 80% 불투명 배경이 깔린다.
- Display cutout: non-floating 창은 `ALWAYS`로 강제된다(2장 참고).
- `Window#setDecorFitsSystemWindows`는 deprecated이고 비활성화됐다.

### Android 16 (API 36)
출처: [Android 16 behavior changes](https://developer.android.com/about/versions/16/behavior-changes-16)

- "For apps targeting Android 16 (API level 36), `R.attr#windowOptOutEdgeToEdgeEnforcement` is deprecated and disabled, and your app can't opt-out of going edge-to-edge."
- targetSdk 36 앱이라도 **Android 15 기기에서는 opt-out이 계속 동작**하고, Android 16 기기에서만 무시된다.
- 함께 바뀐 점으로, sw600dp 이상 화면에서는 `screenOrientation`·`resizableActivity`·aspect ratio 제한이 무시된다(대화면 기기와 폴더블 펼친 상태). 세로 고정 앱도 가로·임의 비율로 실행될 수 있어 가로 cutout과 좌우 inset 대응이 필요하다.

### Google Play 요구
- 2026-08-31부터 신규 앱과 업데이트는 targetSdk 36 이상이어야 한다(연장 시 2026-11-01까지). 기존 앱이 새 사용자에게 계속 노출되려면 targetSdk 35 이상이어야 한다. ([Target API level requirements](https://developer.android.com/google/play/requirements/target-sdk))
- 정리하면, 이 가이드가 다루는 셸은 **edge-to-edge를 끌 수 없다고 가정**해야 한다.

### 하위 버전
- targetSdk 35 이상이면 Android 15 이상 기기에서 자동으로 적용된다. 그 이전 OS에서 같은 동작을 원하면 `enableEdgeToEdge()`를 호출해야 한다. ([Edge-to-edge 가이드](https://developer.android.com/develop/ui/views/layout/edge-to-edge)) 따라서 **같은 앱이라도 Android 14 이하 기기에서는 edge-to-edge가 아닐 수 있다.** 이 경우 WebView가 시스템 바와 겹치지 않으므로 inset이 0이 된다.

## 4. 제스처 내비게이션 vs 3버튼 내비게이션

- WebView가 보는 하단 값은 `systemBars()`(= `statusBars | navigationBars | captionBar`, [WindowInsetsCompat.Type](https://developer.android.com/reference/androidx/core/view/WindowInsetsCompat.Type#systemBars()))에 기반한다. 내비게이션 모드에 따라 내비게이션 바 높이가 다르므로 `safe-area-inset-bottom` 값도 달라진다. 구체적인 dp 값은 기기와 OEM마다 달라 1차 출처로 확정하지 못했다(**미확인**).
- 제스처 모드: 바가 투명하고 콘텐츠가 그 뒤로 그려진다. 홈 제스처용 하단 inset과 뒤로가기용 좌·우 inset은 `systemGestures()`에 따로 있다. ([Edge-to-edge 가이드](https://developer.android.com/develop/ui/views/layout/edge-to-edge)) **WebView는 `systemGestures()`를 웹에 전달하지 않는다.** 전달하는 것은 `displayCutout`과 `systemBars`뿐이다. ([insets.md](https://chromium.googlesource.com/chromium/src.git/+/main/android_webview/docs/insets.md)) 따라서 가장자리 스와이프 제스처 충돌(좌우 캐러셀, 바텀시트 드래그)을 웹에서 CSS만으로 감지할 수는 없다.
- 3버튼 모드: 반투명 scrim(80%)이 깔린다. 투명하게 만들려면 셸에서 `setNavigationBarContrastEnforced(false)`를 호출해야 한다. ([Edge-to-edge 가이드](https://developer.android.com/develop/ui/views/layout/edge-to-edge)) 가로 모드에서는 3버튼 바가 측면에 올 수 있어 `safe-area-inset-left/right`로 들어올 수 있다. 이 동작은 OS 일반 동작에서 유추한 것이며 WebView에서 따로 검증하지는 않았다(**미확인**).
- 키보드가 docked 상태로 떠 있으면 WebView는 하단 safe area를 0으로 보낸다(M144 이상). 하단 고정 UI가 키보드 위에서 내비게이션 바 높이만큼 이중으로 띄워지지 않게 하려는 처리다. ([CL 7122025](https://chromium-review.googlesource.com/c/chromium/src/+/7122025))

## 5. 폴더블·대화면

- Android 16(targetSdk 36)에서 sw600dp 이상은 방향과 비율 제한이 무시된다. 폴더블을 펼치면 화면이 가로로 넓어지고 cutout과 내비게이션 바 위치가 바뀐다. ([Android 16 behavior changes](https://developer.android.com/about/versions/16/behavior-changes-16))
- WebView는 창 크기나 inset이 바뀌면 `onSizeChanged`·`onAttachedToWindow`에서 `requestApplyInsets()`를 다시 호출해 값을 갱신한다. ([AwDisplayCutoutController.java](https://chromium.googlesource.com/chromium/src/+/main/android_webview/java/src/org/chromium/android_webview/AwDisplayCutoutController.java)) 따라서 CSS `env()`는 접기·펼치기·회전 후에도 다시 계산된다.
- 대화면의 taskbar가 어떤 inset 타입으로 WebView에 들어오는지(`navigationBars`에 포함되는지)는 **미확인**이다.
- 힌지·폴드 영역은 safe area가 아니라 Viewport Segments API(Chrome 138에서 출시)로 다룬다. ([Chrome 블로그](https://developer.chrome.com/blog/viewport-segments-api-shipped)) WebView 지원 여부는 **미확인**이다.
- Chrome 브라우저의 edge-to-edge(135)는 대화면 기기를 아직 제외한다. ([Chrome edge-to-edge](https://developer.chrome.com/docs/css-ui/edge-to-edge)) WebView에는 이 제외 규칙이 없다. WebView에서는 셸 창의 inset이 그대로 기준이 된다.

## 6. Chrome 브라우저와의 차이

| 항목 | Chrome (Android) | Android WebView |
|---|---|---|
| `env(safe-area-inset-*)` 시작 | Chrome 69부터 display cutout 지원. `viewport-fit=cover`로 cutout 영역까지 확장 ([New in Chrome 69](https://developer.chrome.com/blog/new-in-chrome-69)) | M136(fullscreen만), M144(전체) |
| `viewport-fit`의 역할 | `cover`→`SHORT_EDGES`, `contain`→`NEVER`로 Activity 창의 cutout 모드를 바꾼다 ([display_cutout.md](https://chromium.googlesource.com/chromium/src.git/+/main/docs/ui/android/display_cutout.md)) | 창을 바꾸지 않는다. 셸이 창을 소유하므로 값은 겹침으로만 정해진다 **[소스 분석]** |
| edge-to-edge | Chrome 135부터 제스처 내비게이션 바 영역으로 viewport 확장. 동적 "chin"이 있고, `viewport-fit=cover`면 처음부터 끝까지 확장 ([Chrome edge-to-edge](https://developer.chrome.com/docs/css-ui/edge-to-edge)) | `DrawCutoutEdgeToEdge`를 WebView에서 끔([CL 6449517](https://chromium-review.googlesource.com/c/chromium/src/+/6449517)). chin 없음. edge-to-edge 여부는 셸이 정한다 |
| `safe-area-inset-bottom` 동적 변화 | chin이 움직이면 값이 갱신됨. `safe-area-max-inset-bottom`은 Chrome 135부터 제공 | chin이 없으므로 시스템 바나 키보드 상태가 바뀔 때만 변한다. `safe-area-max-inset-bottom` 값은 **미확인** |
| 키보드 | 브라우저 자체 정책(`interactive-widget` 등, 키보드 티켓 #7 범위) | M139부터 IME가 visual viewport 하단을 줄인다. 셸이 inset을 소비하면 이 동작이 꺼진다 |
| 업데이트 주체 | Play 스토어의 Chrome | Play 스토어의 Android System WebView. 앱과 독립적으로 갱신되므로 **기기마다 버전이 다르다** |
| 최소 OS | 139부터 Android 10 | M139부터 Android 10(8~9는 M138에서 멈춤) |

## 7. 셸(네이티브) 측 inset 처리 규칙 (WebView 공식 문서)

출처: [Understand window insets in WebView](https://developer.android.com/develop/ui/views/layout/webapps/understand-window-insets)

- **이중 padding**: 셸이 상태 바 40px만큼 padding을 주고 WebView도 40px을 받으면 화면에는 80px 간격이 생긴다.
- **권장(zeroing)**: 셸이 처리한 타입은 `WindowInsetsCompat.Builder(...).setInsets(types, Insets.NONE)`로 0으로 만들어 내려보낸다. `WindowInsetsCompat.CONSUMED`를 반환하면 WebView에 변경 알림이 가지 않는다. 그러면 키보드가 내려간 뒤에도 패딩이 남는 "ghost padding" 문제가 생길 수 있다.
- **겹침 규칙 무시**: WebView에 `setOnApplyWindowInsetsListener`를 달아 원본 insets를 그대로 반환하면, 겹치지 않아도 전체 시스템 inset이 웹에 전달된다(이중 padding 주의).
- **Opt-out**: 처음부터 `CONSUMED`를 반환하면 safe area와 IME 반영이 모두 꺼진다. 이 경우 `scrollIntoView()`가 키보드에 가려질 수 있다.
- **웹 측 주의**: 키보드 때문에 resize 이벤트가 늘어난다. resize 핸들러에서 `blur()`를 호출하면 키보드가 열렸다 닫히기를 반복할 수 있다.
- Flutter 셸(`webview_flutter`)의 platform view 구조에서 이 inset이 실제로 WebView까지 전달되는지는 **미확인**이다. 티켓 #5 범위다.

## 8. 가이드에 반영할 핵심

1. **Android WebView의 safe area 값은 셸이 결정한다.** 같은 웹 코드라도 셸이 (a) 시스템 바만큼 padding을 주고 inset을 0으로 내리면 웹은 `0`을, (b) WebView를 화면 끝까지 그리면 실제 바 높이를 받는다. 웹/네이티브 책임 분담표(#9)에서는 이 두 모델 중 하나를 **명시적으로 선택**해야 한다.
2. **`viewport-fit=cover`는 선언하되, Android WebView에서는 스위치로 쓰지 않는다.** WebView는 `viewport-fit`을 safe area 계산에 쓰지 않는다 [소스 분석]. iOS·Chrome 호환을 위해 선언은 유지한다.
3. **버전 하한을 문서화한다.** `env()`가 의미 있는 값을 주는 조건은 Android 9 이상이면서 WebView M136 이상(fullscreen만)이거나 M144 이상(전체)인 경우다. API 26~27과 오래된 WebView에서는 `env()`가 0이다. 그러므로 **CSS 폴백만으로는 "0 = 안전"인지 "0 = 미지원"인지 구분할 수 없다.** 셸이 padding을 맡거나 셸이 값을 주입하는 방식을 고려해야 한다(감지 전략은 지도의 미결 항목).
4. **edge-to-edge는 끌 수 없다고 가정한다.** targetSdk 36(Play 2026-08-31 요구) + Android 16 기기에서는 opt-out이 무효다. 반대로 Android 14 이하 기기에서는 셸이 `enableEdgeToEdge()`를 호출하지 않으면 edge-to-edge가 아니다. 같은 앱 안에서도 OS별로 값이 달라질 수 있다.
5. **`CONSUMED` 대신 `Insets.NONE` zeroing을 쓴다.** 그래야 키보드 이후 ghost padding이 생기지 않고, M139+ IME visual viewport 축소가 유지된다(#7과 연결).
6. **제스처 영역은 CSS로 알 수 없다.** `systemGestures`는 웹에 전달되지 않는다. 가장자리 스와이프 UI는 셸과 협의해야 한다. 3버튼 모드의 scrim, 가로 모드의 좌·우 inset, 폴더블·대화면(Android 16 sw600dp 방향 강제 해제)도 테스트 매트릭스에 넣는다.
7. **Chrome 브라우저 대응 코드와 분리해서 이해한다.** chin, `safe-area-max-inset-bottom`, 대화면 제외 규칙은 Chrome 전용 동작이다. 같은 Next.js 앱이 두 환경에서 서비스될 때 하단 고정 UI는 `bottom: env(safe-area-inset-bottom)` 형태(Chrome 권장, padding보다 낫다)로 작성하면 양쪽에서 무리 없이 동작할 가능성이 높다. 다만 WebView 쪽은 **미확인**이다.

## 미확인 목록

- `safe-area-max-inset-bottom`의 WebView 값
- WebView에서 Viewport Segments API 지원 여부
- 대화면 taskbar inset이 WebView에 전달되는 타입
- 제스처/3버튼 모드에서의 실제 dp 값(OEM마다 다름)
- Capacitor 문서의 "WebView adheres to viewport-fit" 주장과 소스 분석 결과 사이의 불일치
- Chrome/WebView 139의 Android 10 요구에 대한 Google 공식 공지 원문
- Flutter platform view 경로로 inset이 WebView에 전달되는지(#5)
