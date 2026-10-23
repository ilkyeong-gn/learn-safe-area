# Safe area의 역사와 CSS 스펙 현황

- 조사일: 2026-10-23
- 티켓: [#2](https://github.com/ilkyeong-gn/learn-safe-area/issues/2) (지도 [#1](https://github.com/ilkyeong-gn/learn-safe-area/issues/1))
- 출처 원칙: W3C/CSSWG 스펙과 회의록, WebKit 블로그, Chrome for Developers, 표준 입장(standards-positions) 이슈, MDN browser-compat-data(BCD) 원본 JSON.

## 요약

Safe area는 2017년 iPhone X와 함께 WebKit이 `viewport-fit=cover`와 `constant(safe-area-inset-*)`로 웹에 처음 들여왔다. 같은 해 CSSWG가 이름을 `env()`로 바꾸기로 결의했고, iOS 11.2부터 `env()`가 표준 형태가 되었다. `env()` 자체는 2020년 1월부터 Baseline "Widely available"이다. 그러나 이를 정의하는 **CSS Environment Variables Module Level 1**은 2025-09-23에야 First Public Working Draft가 되었다. `viewport-fit`은 지금도 별도 스펙(CSS Round Display)에 있다. 최근에는 Chrome이 Android에서 edge-to-edge(Chrome 135)를 도입하면서 `safe-area-max-inset-*`, `keyboard-inset-*`, `interactive-widget` 같은 Blink 주도 기능이 늘었다. WebKit은 이들 신규 기능을 아직 지원하지 않는다.

## 연표

| 날짜 | 사건 | 출처 |
|---|---|---|
| 2017-09-20 | CSSWG 텔레콘(issue #1693)에서 결의: "Rename what was constant variables to environment variables using env()", "The initial set is safe area top, bottom, left and right" | [CSSWG minutes 2017-09-20](https://lists.w3.org/Archives/Public/www-style/2017Sep/0045.html) |
| 2017-09-22 | WebKit 블로그 "Designing Websites for iPhone X" 게시. iOS 11에서 `viewport-fit` 메타 확장(기본값 `auto`, `cover`로 전체 화면 레이아웃)과 `safe-area-inset-top/right/bottom/left`를 소개. iOS 11 출시 당시 함수 이름은 `constant()`였다 | [WebKit blog](https://webkit.org/blog/7929/designing-websites-for-iphone-x/) |
| 2017-10-31 | 같은 글 업데이트: "Beginning with Safari Technology Preview 41 and the iOS 11.2 beta, constant() has been removed and replaced with env()." | [WebKit blog](https://webkit.org/blog/7929/designing-websites-for-iphone-x/) |
| 2018 (Safari 11.1) | BCD 기준으로 Safari 11.1부터 `env()`를 지원하고, `constant` 대체 이름은 11 ~ 11.1 사이에만 있었다 | [BCD css/types/env.json](https://github.com/mdn/browser-compat-data/blob/main/css/types/env.json) |
| 2018-07-04 | CSSWG 시드니 F2F 결의: "UA vendors MUST NOT expose built-in env() features to the web without going through that process" (#2820). safe-area-inset 정식 정의 제안(#2868)은 보류 | [CSSWG minutes Sydney 2018-07-04](https://lists.w3.org/Archives/Public/www-style/2018Jul/0025.html) |
| 2018 (Chrome 69) | Chrome(및 Android Chrome/WebView, BCD mirror)에서 `env()`와 `safe-area-inset-*` 지원 | [BCD env.json](https://github.com/mdn/browser-compat-data/blob/main/css/types/env.json) |
| 2019 (Firefox 65) | Firefox에서 `env()`와 `safe-area-inset-*` 지원. 이로써 2020년 1월에 Baseline "Widely available"이 되었다 | [BCD env.json](https://github.com/mdn/browser-compat-data/blob/main/css/types/env.json), [MDN env()](https://developer.mozilla.org/en-US/docs/Web/CSS/env) |
| 2020 (Firefox Android 79) | Firefox Android에서 `viewport-fit` 지원 | [BCD viewport-fit.json](https://github.com/mdn/browser-compat-data/blob/main/html/elements/meta/name/viewport/viewport-fit.json) |
| 2021 (Chrome 93/94) | `titlebar-area-*`(Window Controls Overlay, Chrome 93)와 VirtualKeyboard API 및 `keyboard-inset-*`(Chrome 94) 지원 | [BCD env.json](https://github.com/mdn/browser-compat-data/blob/main/css/types/env.json), [BCD VirtualKeyboard.json](https://github.com/mdn/browser-compat-data/blob/main/api/VirtualKeyboard.json) |
| 2022-05-05 | VirtualKeyboard API가 W3C Working Draft(Web Editing WG)로 발행됨 | [W3C TR virtual-keyboard](https://www.w3.org/TR/virtual-keyboard/) |
| 2022 (Chrome Android 108) | `<meta name="viewport">`의 `interactive-widget` 지원 | [BCD interactive-widget.json](https://github.com/mdn/browser-compat-data/blob/main/html/elements/meta/name/viewport/interactive-widget.json) |
| 2024-10-10 | `safe-area-max-inset-*` 제안 등장: "[css-env-1] Maximum safe area inset values to allow sliding bottom bar" (csswg-drafts #11019) | [csswg-drafts #11019](https://github.com/w3c/csswg-drafts/issues/11019) |
| 2025-01-22 | CSSWG 결의: "adopt safe-area-max-inset-* values". 2025-01-30에 PR #11617로 스펙에 반영 | [csswg-drafts #11019](https://github.com/w3c/csswg-drafts/issues/11019) |
| 2025-02-12 | W3C TAG 디자인 리뷰 #1046 종결, 결과는 "satisfied" | [TAG review #1046](https://github.com/w3ctag/design-reviews/issues/1046) |
| 2025-02-28 | Chrome 가이드 게시: "From Chrome 135, the viewport is allowed to extend into Android's gesture navigation bar." `safe-area-max-inset-*`도 Chrome 135부터 지원. BCD 기준 Chrome Android의 `viewport-fit` 지원 버전도 135 | [Chrome edge-to-edge guide](https://developer.chrome.com/docs/css-ui/edge-to-edge), [BCD viewport-fit.json](https://github.com/mdn/browser-compat-data/blob/main/html/elements/meta/name/viewport/viewport-fit.json) |
| 2025 (Chrome 138) | `viewport-segment-*`(폴더블)와 `preferred-text-scale` 지원 | [BCD env.json](https://github.com/mdn/browser-compat-data/blob/main/css/types/env.json) |
| 2025-09-23 | **CSS Environment Variables Module Level 1 First Public Working Draft** 발행. 이전 TR 발행은 없다 | [W3C TR css-env-1](https://www.w3.org/TR/css-env-1/), [발행 이력](https://www.w3.org/standards/history/css-env-1/) |
| 2026-09-14 | Mozilla standards-positions #1171(`safe-area-max-inset-*`)에 `position: positive`가 붙고 종결됨 | [mozilla/standards-positions#1171](https://github.com/mozilla/standards-positions/issues/1171) |
| 2026-10-08 | css-env-1 Editor's Draft 최신판 | [ED css-env-1](https://drafts.csswg.org/css-env-1/) |

## 현재 스펙 현황 (2026-10 기준)

### 기능별 정의 위치

Safe area 관련 기능은 스펙 하나에 모여 있지 않다. 정의는 다음과 같이 흩어져 있다.

| 기능 | 정의 스펙 | 성숙도 |
|---|---|---|
| `env()` 함수, `safe-area-inset-*`, `safe-area-max-inset-*`, `viewport-segment-*`, `preferred-text-scale` | [CSS Environment Variables Level 1](https://drafts.csswg.org/css-env-1/) | FPWD 2025-09-23, ED 2026-10-08 |
| `viewport-fit` (`auto` / `contain` / `cover`) | [CSS Round Display Level 1](https://drafts.csswg.org/css-round-display/) §4.1 | ED 2025-12-26. ED 본문에는 `safe-area`나 `env()`에 대한 언급이 없다 |
| `interactive-widget` (`resizes-visual` / `resizes-content` / `overlays-content`) | [CSS Viewport Level 1](https://drafts.csswg.org/css-viewport/) §3.4 | ED 2026-08-29. 이 ED의 meta 속성 목록에는 `viewport-fit`이 없다 |
| `keyboard-inset-*`, `navigator.virtualKeyboard` | [VirtualKeyboard API](https://w3c.github.io/virtual-keyboard/) | W3C WD 2022-05-05 ([TR](https://www.w3.org/TR/virtual-keyboard/)) |
| `titlebar-area-*` | [Window Controls Overlay (WICG)](https://wicg.github.io/window-controls-overlay/#title-bar-area-env-variables) | WICG. 데스크톱 PWA 전용이므로 이 가이드 범위 밖 |

### css-env-1의 핵심 정의 ([ED](https://drafts.csswg.org/css-env-1/))

- 문법: `env( <custom-ident> <integer [0,∞]>*, <declaration-value>? )`. 정수 인덱스는 `viewport-segment-*` 같은 다차원 변수에 쓴다.
- 이름이나 인덱스가 유효하지 않으면 fallback을 쓴다. fallback이 없으면 해당 선언은 "invalid at computed-value time"이 된다.
- 스펙이 정의하는 UA 변수는 `safe-area-inset-{top,right,bottom,left}`, `safe-area-max-inset-{top,right,bottom,left}`, `viewport-segment-*`, `preferred-text-scale`이다. 스펙은 "Additional UA-defined environment variables must not be supported unless/until they are added to this list."라고 규정한다. 다만 `keyboard-inset-*`와 `titlebar-area-*`는 각자 다른 스펙에서 정의된다.
- `safe-area-max-inset-*`는 동적 UA 인터페이스(접히는 툴바, 내비게이션 바 chin 등)가 모두 물러났을 때의 **정적 최댓값**이다. 반면 `safe-area-inset-*`는 동적으로 변한다. 도입 동기는 하단 바를 리레이아웃(jitter) 없이 미리 최대 높이로 잡아 두려는 것이었다 ([#11019](https://github.com/w3c/csswg-drafts/issues/11019)).

### `keyboard-inset-*` 관련 메모

- VirtualKeyboard 스펙은 inset 6종(`top/right/bottom/left/width/height`)을 정의하고, fallback이 없으면 기본값을 `0px`로 둔다. `overlaysContent = true`이면 UA는 viewport를 리사이즈하지 않아야 한다 ([VirtualKeyboard ED](https://w3c.github.io/virtual-keyboard/)).
- 스펙 본문은 inset 갱신을 `overlaysContent` 값에 묶어 두지 않는다. 스펙 예제는 `overlaysContent = true`와 env 변수를 함께 쓴다. 실제 엔진이 어떻게 동작하는지는 키보드 조사 티켓에서 확인해야 한다.

## 엔진별 지원 현황

출처는 [MDN BCD](https://github.com/mdn/browser-compat-data)(2026-10-23 main 브랜치 원본 JSON)다. BCD에서 Android Chrome, Android WebView, Edge, Samsung Internet은 Chrome 데이터의 mirror이고, iOS WebView는 Safari iOS의 mirror다.

| 기능 | WebKit (Safari / iOS) | Blink (Chrome / Android WebView) | Gecko (Firefox) |
|---|---|---|---|
| `env()` | 11.1 (11 ~ 11.1은 `constant()`) | 69 | 65 |
| `safe-area-inset-*` | 11 | 69 | 65 |
| `viewport-fit` meta | iOS 11 | Chrome Android **135** | Firefox Android 79 |
| `safe-area-max-inset-*` | 미지원. [WebKit 입장 #454](https://github.com/WebKit/standards-positions/issues/454)는 열려 있고 입장 미표명 | 135 ([Chrome 가이드](https://developer.chrome.com/docs/css-ui/edge-to-edge)). BCD에는 아직 항목 없음 | 미지원으로 보임(BCD 항목 없음). 2026-09에 입장 positive |
| `keyboard-inset-*` / VirtualKeyboard API | 미지원 ([webkit.org/b/230225](https://webkit.org/b/230225)) | 94 (experimental) | 미지원 ([bugzil.la/1730568](https://bugzil.la/1730568)) |
| `interactive-widget` meta | 미지원 | Chrome Android 108 | Firefox Android 133 |
| `viewport-segment-*`, `preferred-text-scale` | 미지원 | 138 (experimental) | 미지원 |

해석상 주의할 점은 다음과 같다.

- BCD 버전은 "파싱하고 값을 노출함"을 뜻한다. 값이 실제로 0이 아닌지는 플랫폼과 호스트에 달려 있다. 예를 들어 Android WebView에서 inset 값이 어떻게 전달되는지는 네이티브 셸 설정에 따라 다르며, 이는 플랫폼별 동작 티켓에서 다룬다.
- Chrome의 Android edge-to-edge는 Chrome Variation으로 점진 배포되었고 대형 화면 기기는 제외되었다. 페이지가 `viewport-fit=cover`로 opt-in하지 않으면 하단 chin이 기본으로 표시된다. `safe-area-inset-bottom`은 chin이 사라질 때 동적으로 갱신된다. Chrome은 `padding`에 safe-area inset을 직접 쓰는 패턴을 감지하면 layout thrashing을 막으려고 chin을 슬라이드하지 않는다 ([Chrome edge-to-edge guide](https://developer.chrome.com/docs/css-ui/edge-to-edge)).
- iOS 26 / Safari 26에서 safe area 동작이 바뀌었는지는 Safari 26.0 릴리스 노트에서 확인할 수 없었다 ([Safari 26 release notes](https://developer.apple.com/documentation/safari-release-notes/safari-26-release-notes)). Dynamic Island 아래 헤더가 밀린다는 미확정 버그 보고([webkit.org/b/300523](https://bugs.webkit.org/show_bug.cgi?id=300523))가 있으므로 플랫폼별 동작 티켓에서 추가로 조사해야 한다.

## 가이드에 반영할 핵심

1. **`constant()`는 쓰지 않는다.** iOS 11.0 ~ 11.1 전용이었고 iOS 11.2부터 `env()`로 대체되었다. 최소 지원 OS가 iOS 16+이므로 `constant()` 폴백은 필요 없다.
2. **`viewport-fit=cover`는 양 엔진에서 필수 opt-in이다.** iOS는 11부터, Chrome Android는 135부터 이 값이 있어야 콘텐츠가 시스템 UI 아래까지 그려지고 inset이 의미를 가진다. Next.js에서는 `export const viewport = { viewportFit: 'cover' }`로 설정한다(Next.js 적용 세부 사항은 별도 티켓에서 다룬다).
3. **`env()`에는 항상 fallback을 둔다** (예: `env(safe-area-inset-bottom, 0px)`). 스펙상 fallback이 없으면 선언 전체가 invalid at computed-value time이 된다.
4. **신규 변수는 Blink 전용 progressive enhancement로 다룬다.** `safe-area-max-inset-*`(Chrome 135+), `keyboard-inset-*`(Chrome 94+), `interactive-widget`(Chrome Android 108+)는 WebKit에서 동작하지 않는다. iOS에서도 깨지지 않도록 fallback 체인을 설계해야 한다. 예: `env(safe-area-max-inset-bottom, env(safe-area-inset-bottom, 0px))`.
5. **스펙은 아직 Working Draft다.** `env()`와 `safe-area-inset-*`는 사실상 표준(Baseline Widely available)이다. 하지만 css-env-1은 2025-09에 FPWD가 된 단계이고, `viewport-fit`은 별도 스펙(CSS Round Display) 소관이다. 가이드에서 "표준"이라고 말할 때는 이 구분을 명시한다.
6. **Android 하단 inset은 동적으로 변한다.** Chrome 135+에서는 chin이 물러나면서 `safe-area-inset-bottom` 값이 바뀐다. 하단 고정 UI는 리레이아웃 비용을 고려해야 하며, 이것이 `safe-area-max-inset-*`가 도입된 이유다.
