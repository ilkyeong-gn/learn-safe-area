# Next.js 16에서 safe area를 적용하는 방법

- 티켓: [#6](https://github.com/ilkyeong-gn/learn-safe-area/issues/6) (지도 [#1](https://github.com/ilkyeong-gn/learn-safe-area/issues/1))
- 조사일: 2026-10-23
- 기준 버전: Next.js `v16.4.2`(조사 시점 최신 릴리스), Tailwind CSS `v4.3.3`, `webview_flutter` 4.14.1
- 표기: 1차 출처로 확인하지 못한 항목은 **미확인**, 출처에서 직접 읽히지 않고 추론한 내용은 **(추론)**으로 표시한다.

## 1. 요약

- Next.js 16 App Router에서 `viewport-fit=cover`는 `export const viewport = { viewportFit: 'cover' }`로 설정한다. 이 필드는 공식 문서 페이지에는 나오지 않지만, `Viewport` 타입과 렌더링 코드에 정식으로 들어 있다.
- `viewport`의 각 필드는 key 단위로 덮어써진다. 기본값 `width=device-width, initial-scale=1`은 그대로 유지되므로, 루트 layout에 `viewportFit`만 추가하면 된다.
- `themeColor`는 `<meta name="theme-color">`로 따로 출력되며 `viewport-fit`과 직접 연동되지 않는다. iOS 26 Safari는 상단 바 색을 페이지 배경이나 fixed/sticky 요소에서 가져오는 쪽으로 바뀌었다는 보고가 있지만, 공식 문서로는 확인되지 않았다(미확인).
- 레이아웃은 `env(safe-area-inset-*, 0px)`를 `padding`에 더하는 방식이 기본이다. Tailwind v4에서는 arbitrary value `pt-[env(safe-area-inset-top)]`, CSS 변수 shorthand `pt-(--safe-top)`, 또는 `@utility`로 쓴다.
- SSR에서 WebView를 구분할 신호로는 Android WebView UA의 `; wv` 토큰, Android WebView의 `X-Requested-With` 헤더(값은 앱 패키지명), 네이티브 셸이 설정한 커스텀 UA가 있다. 다만 서버에서 분기하면 렌더링이 dynamic이 되고 hydration 불일치 위험이 생긴다. 따라서 **CSS만으로 양쪽을 처리하는 방식을 기본으로 하고, 분기는 최소화**하는 것이 낫다.

## 2. `viewport-fit=cover` 설정: `export const viewport` / `generateViewport`

### 2.1 공식 API

- `viewport` 객체나 `generateViewport` 함수는 `layout`/`page`에서 export한다. 두 export는 **Server Component에서만** 지원되며, 같은 segment에서 둘을 함께 export할 수 없다. [nextjs.org generateViewport](https://nextjs.org/docs/app/api-reference/functions/generate-viewport)
- 요청 정보에 의존하지 않는 viewport는 정적 `viewport` 객체로 정의하라고 권장한다. [같은 문서](https://nextjs.org/docs/app/api-reference/functions/generate-viewport#generateviewport-function)
- 공식 문서의 Viewport Fields 절은 `themeColor`, `width`, `initialScale`, `maximumScale`, `userScalable`, `interactiveWidget`, `colorScheme`만 다룬다. **`viewportFit`은 문서에 나오지 않는다.** [같은 문서](https://nextjs.org/docs/app/api-reference/functions/generate-viewport#viewport-fields)
- 소스에서는 정식 지원된다.
  - 타입: `ViewportLayout`에 `viewportFit?: 'auto' | 'cover' | 'contain'`과 `interactiveWidget?: 'resizes-visual' | 'resizes-content' | 'overlays-content'`가 정의되어 있다. [extra-types.ts@v16.4.2](https://github.com/vercel/next.js/blob/v16.4.2/packages/next/src/lib/metadata/types/extra-types.ts)
  - 키 매핑: `viewportFit: 'viewport-fit'`. [constants.ts@v16.4.2](https://github.com/vercel/next.js/blob/v16.4.2/packages/next/src/lib/metadata/constants.ts)
  - 렌더링: `createViewportElements()`는 `width, height, initial-scale, minimum-scale, maximum-scale, user-scalable, viewport-fit, interactive-widget` 순서로 `, `를 붙여 하나의 `<meta name="viewport">`를 만든다. [metadata-elements.tsx@v16.4.2](https://github.com/vercel/next.js/blob/v16.4.2/packages/next/src/lib/metadata/metadata-elements.tsx)
  - 이 기능을 요청한 이슈: [vercel/next.js#46498 "Support `viewport-fit` custom metadata viewport property"](https://github.com/vercel/next.js/issues/46498) (closed)
- 기본값: `createDefaultViewport()`는 `width: 'device-width', initialScale: 1`이다. [default-metadata.tsx@v16.4.2](https://github.com/vercel/next.js/blob/v16.4.2/packages/next/src/lib/metadata/default-metadata.tsx)
- 병합 규칙: `mergeViewport()`는 layout에서 page 방향으로 segment를 순회하면서, `width`, `viewportFit`, `interactiveWidget` 등 레이아웃 필드를 **"always override the target with the source"** 방식으로 key 단위로 덮어쓴다. `themeColor`는 `resolveThemeColor()`로 배열로 정규화한 뒤 통째로 교체한다. [metadata-resolution-primitives.ts@v16.4.2](https://github.com/vercel/next.js/blob/v16.4.2/packages/next/src/lib/metadata/metadata-resolution-primitives.ts)
  - 따라서 루트 layout에서 `viewportFit: 'cover'`만 지정해도 기본값 `width=device-width, initial-scale=1`은 남는다. 하위 segment가 `viewportFit`을 다시 지정하지 않는 한 모든 라우트에 적용된다.

### 2.2 권장 설정 예시

```tsx
// app/layout.tsx (Server Component)
import type { Viewport } from 'next'

export const viewport: Viewport = {
  viewportFit: 'cover',
  // 필요할 때만: 가상 키보드가 layout viewport를 줄이도록 할지 (키보드 티켓에서 결정)
  // interactiveWidget: 'resizes-content',
}
```

출력(위 소스 기준으로 예상한 결과):

```html
<meta name="viewport" content="width=device-width, initial-scale=1, viewport-fit=cover" />
```

### 2.3 `generateViewport`를 쓰지 말아야 하는 이유 (UA별 `viewportFit` 분기)

- "Unlike metadata, viewport cannot be streamed because it affects initial page load UI." `generateViewport`가 `cookies()`/`headers()` 같은 runtime data를 읽으면 페이지가 그 값이 정해질 때까지 block된다. Cache Components를 쓰면 `<Suspense>`로 감싸거나 `instant = false`를 지정해야 한다. [generateViewport#with-cache-components](https://nextjs.org/docs/app/api-reference/functions/generate-viewport#with-cache-components)
- `headers()`는 Request-time API다. 이를 쓰는 라우트는 dynamic rendering이 된다. [headers](https://nextjs.org/docs/app/api-reference/functions/headers)
- (추론) `viewport-fit=cover`는 inset이 없는 환경에서 `env()`가 0이 되므로 해가 없다. 그래서 WebView와 브라우저를 나눠서 `viewportFit`을 바꿀 실익이 적다. 정적 `viewport`로 모든 환경에 `cover`를 주는 쪽이 정적 렌더링을 유지하는 데에도 유리하다.

### 2.4 `viewport-fit` 값의 의미 (MDN)

- `auto`: 초기 layout viewport에 영향을 주지 않는다. `contain`: 디스플레이에 내접하는 가장 큰 사각형에 맞춘다. `cover`: 디바이스 디스플레이를 채운다. "It's highly recommended to use the safe area inset variables". [MDN viewport meta](https://developer.mozilla.org/en-US/docs/Web/HTML/Reference/Elements/meta/name/viewport)
- `interactive-widget`의 기본값은 `resizes-visual`이다(가상 키보드가 visual viewport만 줄인다). [같은 문서](https://developer.mozilla.org/en-US/docs/Web/HTML/Reference/Elements/meta/name/viewport)

## 3. `themeColor`와의 상호작용

- Next.js 출력: `themeColor: 'black'`은 `<meta name="theme-color" content="black">`로, `[{ media, color }]` 배열은 `media` 속성이 붙은 여러 meta로 출력된다. viewport meta와는 **별도 태그**다. [generateViewport#themecolor](https://nextjs.org/docs/app/api-reference/functions/generate-viewport#themecolor), [metadata-elements.tsx@v16.4.2](https://github.com/vercel/next.js/blob/v16.4.2/packages/next/src/lib/metadata/metadata-elements.tsx)
- `themeColor`와 `viewportFit`은 같은 `viewport` export 안에 있지만 소스상 서로 참조하지 않는다. 그래서 **Next.js 수준의 상호작용은 없다**. [metadata-resolution-primitives.ts@v16.4.2](https://github.com/vercel/next.js/blob/v16.4.2/packages/next/src/lib/metadata/metadata-resolution-primitives.ts)
- 스펙상 의미: theme-color는 UA가 "the display of the page or of the surrounding user interface"를 꾸밀 때 쓰는 **제안(suggested) 색**이다. MDN 호환성 표기는 "Limited availability"다. [MDN theme-color](https://developer.mozilla.org/en-US/docs/Web/HTML/Reference/Elements/meta/name/theme-color)
- iOS 26 Safari:
  - WebKit 엔지니어(Wenson Hsieh)는 bug 301756 코멘트에서, 상단 바의 단색 tint(color extension)가 "viewport-constrained"(fixed/sticky) 요소가 가장자리에 있을 때 스크롤 중 틈을 막기 위해 쓰인다고 설명했다. [WebKit bug 301756](https://bugs.webkit.org/show_bug.cgi?id=301756)
  - "Safari 26이 theme-color 지원을 중단했다"는 말은 **리포터의 주장**일 뿐이다. WebKit 공식 릴리스 노트([Safari 26.0](https://webkit.org/blog/17333/webkit-features-in-safari-26-0/))에는 해당 언급이 없다. → **미확인**
- 하이브리드웹에서의 의미: (추론, 미확인) WebView 안에서 상태 바/내비게이션 바의 색과 아이콘 밝기는 네이티브 셸(Flutter `SystemChrome` 등)이 정한다. `theme-color`가 WKWebView나 Android WebView의 시스템 바에 반영된다는 1차 출처는 찾지 못했다. `viewport-fit=cover`일 때 WebView가 시스템 바 뒤까지 그려진다면, 그 영역의 색은 결국 **웹 페이지의 배경색**(`html`/`body` 또는 고정 헤더 배경)이 된다. 책임 분담 티켓에서 확인이 필요하다.
- 실무 제안: `themeColor`는 모바일 브라우저(비교 환경)를 위한 장식 정도로 다룬다. safe area 영역의 색은 **CSS 배경**(헤더가 inset 영역까지 덮는 방식)으로 확정하는 것이 두 환경에서 일관적이다.

## 4. `env()` 기반 레이아웃 패턴

### 4.1 기초 규칙

- 변수: `safe-area-inset-top|right|bottom|left`. 사각형 뷰포트에 차지하는 UI가 없으면 `0`, 있으면 `0`보다 큰 `px` 값이다. `safe-area-max-inset-*`는 동적 UI가 접힌 상태의 최댓값(상수)이다. [MDN env()](https://developer.mozilla.org/en-US/docs/Web/CSS/env)
- 스펙: 네 inset이 UA가 고른 사각형을 이룬다. 알 수 없는 변수는 fallback을 쓰고, fallback이 없으면 computed-value time에 invalid가 된다. 스펙 상태는 Editor's Draft(2026-10-08)다. [CSS Environment Variables Module Level 1](https://drafts.csswg.org/css-env-1/)
- fallback은 첫 쉼표 뒤의 값이다. [MDN env()#fallback](https://developer.mozilla.org/en-US/docs/Web/CSS/env)
- `calc()` 안에서 쓸 때는 fallback을 `0`이 아니라 **`0px`**로 둔다. `calc()`의 덧셈은 같은 타입끼리만 유효하기 때문이다. [CSS Values 4 calc type checking](https://www.w3.org/TR/css-values-4/#calc-type-checking)
- 키보드 관련: `keyboard-inset-*` env 변수는 VirtualKeyboard API에 정의되어 있다. [MDN env()](https://developer.mozilla.org/en-US/docs/Web/CSS/env) 자세한 내용은 키보드 티켓 범위다.
- 뷰포트 단위: Tailwind v4는 `h-dvh`(`100dvh`), `h-svh`, `h-lvh`, `h-screen`(`100vh`)을 제공한다. [tailwindcss.com height](https://tailwindcss.com/docs/height)

공통 토큰(권장):

```css
/* app/globals.css */
:root {
  --safe-top: env(safe-area-inset-top, 0px);
  --safe-right: env(safe-area-inset-right, 0px);
  --safe-bottom: env(safe-area-inset-bottom, 0px);
  --safe-left: env(safe-area-inset-left, 0px);
  --header-h: 56px;
  --tabbar-h: 56px;
}
```

### 4.2 고정 헤더

순수 CSS:

```css
.app-header {
  position: fixed;
  inset: 0 0 auto 0;            /* top:0; left:0; right:0 */
  padding-top: var(--safe-top); /* 배경이 노치/상태바 영역까지 칠해짐 */
  padding-left: var(--safe-left);
  padding-right: var(--safe-right);
  background: var(--header-bg);
}
.app-header__bar { height: var(--header-h); }
.app-main {
  padding-top: calc(var(--header-h) + var(--safe-top));
}
```

Tailwind v4:

```html
<header class="fixed inset-x-0 top-0 pt-[env(safe-area-inset-top,0px)] px-[env(safe-area-inset-left,0px)] bg-white">
  <div class="h-14">…</div>
</header>
<main class="pt-[calc(3.5rem+env(safe-area-inset-top,0px))]">…</main>
```

- 핵심: 헤더의 **배경**은 inset 영역까지 덮고 **콘텐츠**는 inset 아래에서 시작한다. iOS 26 Safari에서는 fixed 헤더가 상단 tint의 근거가 된다는 엔지니어 설명이 있다. [WebKit bug 301756](https://bugs.webkit.org/show_bug.cgi?id=301756)
- `px-[...]` 한 줄은 좌우 inset이 같다고 가정한다. 가로 모드처럼 좌우가 다를 수 있으면 `pl-[env(safe-area-inset-left,0px)] pr-[env(safe-area-inset-right,0px)]`로 나눠 쓴다. (추론)

### 4.3 하단 탭바

순수 CSS:

```css
.tabbar {
  position: fixed;
  inset: auto 0 0 0;
  padding-bottom: var(--safe-bottom); /* 홈 인디케이터/제스처 바 영역 */
  background: var(--tabbar-bg);
}
.tabbar__items { height: var(--tabbar-h); }
.app-main {
  padding-bottom: calc(var(--tabbar-h) + var(--safe-bottom));
}
```

Tailwind v4:

```html
<nav class="fixed inset-x-0 bottom-0 pb-[env(safe-area-inset-bottom,0px)] bg-white">
  <div class="h-14 flex">…</div>
</nav>
<main class="pb-[calc(3.5rem+env(safe-area-inset-bottom,0px))]">…</main>
```

- inset이 0인 기기에서도 최소 여백이 필요하면 `max()`를 쓴다. 예: `padding-bottom: max(var(--safe-bottom), 8px)` / `pb-[max(env(safe-area-inset-bottom,0px),0.5rem)]`. (추론: `max()`는 CSS Values 4의 비교 함수다. [css-values-4#comp-func](https://www.w3.org/TR/css-values-4/#comp-func))

### 4.4 풀스크린 모달 / 시트

순수 CSS:

```css
.modal {
  position: fixed;
  inset: 0;
  height: 100dvh;              /* 브라우저 UI 신축을 반영 */
  padding: var(--safe-top) var(--safe-right) var(--safe-bottom) var(--safe-left);
  background: var(--modal-bg); /* 배경은 화면 끝까지 */
  overflow: auto;
}
```

Tailwind v4:

```html
<div class="fixed inset-0 h-dvh overflow-auto bg-white
            pt-[env(safe-area-inset-top,0px)] pr-[env(safe-area-inset-right,0px)]
            pb-[env(safe-area-inset-bottom,0px)] pl-[env(safe-area-inset-left,0px)]">…</div>
```

- 닫기 버튼처럼 상단에 고정된 컨트롤은 `top: calc(var(--safe-top) + 8px)`처럼 inset을 더해 배치한다.

### 4.5 Tailwind v4 표기 정리

- arbitrary value: `top-[117px]`처럼 대괄호를 쓴다. 공백은 `_`로 쓴다. [tailwindcss.com adding-custom-styles](https://tailwindcss.com/docs/adding-custom-styles#using-arbitrary-values)
- CSS 변수 shorthand: `fill-(--my-brand-color)`는 `fill-[var(--my-brand-color)]`와 같다. 따라서 4.1의 토큰을 쓰면 `pt-(--safe-top)`, `pb-(--safe-bottom)`처럼 짧게 쓸 수 있다. [같은 문서](https://tailwindcss.com/docs/adding-custom-styles#using-arbitrary-values)
- `env()` 처리: Tailwind v4의 arbitrary value 디코더는 `env()` 안쪽 식별자를 수학 연산으로 포맷하지 않는다. 또 `calc(env(safe-area-inset-bottom)*2)` → `calc(env(safe-area-inset-bottom) * 2)`처럼 `calc` 연산자 주변에 공백을 자동으로 넣는다. 그래서 `pb-[calc(3.5rem+env(safe-area-inset-bottom,0px))]`에 `_`를 쓸 필요가 없다. [decode-arbitrary-value.test.ts](https://github.com/tailwindlabs/tailwindcss/blob/ddb0beff6626367cc75e4248e47ab2a04017bde6/packages/tailwindcss/src/utils/decode-arbitrary-value.test.ts)
- 반복을 줄이는 `@utility`: [tailwindcss.com adding-custom-styles#adding-custom-utilities](https://tailwindcss.com/docs/adding-custom-styles#adding-custom-utilities)

  ```css
  @utility pt-safe { padding-top: env(safe-area-inset-top, 0px); }
  @utility pb-safe { padding-bottom: env(safe-area-inset-bottom, 0px); }
  @utility px-safe {
    padding-left: env(safe-area-inset-left, 0px);
    padding-right: env(safe-area-inset-right, 0px);
  }
  ```

- Tailwind 코어에 `pb-safe` 같은 safe area 전용 유틸리티가 내장되어 있다는 근거는 찾지 못했다. → **미확인**(없다고 보고 `@utility`로 정의할 것)

## 5. SSR에서 WebView와 브라우저를 구분하는 신호

| 신호 | 위치 | 근거 | 신뢰도/주의 |
|---|---|---|---|
| Android WebView UA의 `; wv` 토큰 | `User-Agent` | Chromium 소스 `AwContentBrowserClient`가 `"; wv"`를 OS 정보에 붙인다. UA reduction 플래그가 켜지면 `"Linux; Android 10; K; wv"`로 통일된다. [aw_content_browser_client.cc](https://source.chromium.org/chromium/chromium/src/+/main:android_webview/browser/aw_content_browser_client.cc) | 기본 UA일 때만 유효하다. 앱이 UA를 덮어쓰면 사라질 수 있다. Chromium은 Android WebView에 UA reduction을 적용할 계획이 없다고 밝혔다. [chromium.org UA reduction](https://www.chromium.org/updates/ua-reduction/) |
| `X-Requested-With: <앱 패키지명>` | 요청 헤더 (Android WebView) | 값은 "the application APK name"이다. [Chrome blog](https://developer.chrome.com/blog/android-webview-header-privacy) 제거 계획은 **취소**되어 다시 모든 요청에 기본으로 전송된다(androidx.webkit 1.15.0, 2025-12-17). [Jetpack WebKit release notes](https://developer.android.com/jetpack/androidx/releases/webkit) | Android 전용이다. iOS에는 해당하는 헤더가 없다(미확인). |
| 셸이 지정한 커스텀 UA 토큰 | `User-Agent` | `webview_flutter`의 `WebViewController.setUserAgent(String?)`: "Sets the value used for the HTTP `User-Agent:` request header." [pub.dev](https://pub.dev/documentation/webview_flutter/latest/webview_flutter/WebViewController/setUserAgent.html) | **양 플랫폼에서 가장 결정적이다.** 단, `setUserAgent`는 값 전체를 교체하는 API다. 기본 UA에 토큰을 덧붙이려면 셸에서 기본 UA를 얻어 이어 붙여야 한다(구체 방법은 미확인). |
| 셸이 첫 요청에 붙이는 커스텀 헤더 | 요청 헤더 | `loadRequest(uri, headers: ...)`: "its key-value pairs will be added as the headers for the request." [pub.dev](https://pub.dev/documentation/webview_flutter/latest/webview_flutter/WebViewController/loadRequest.html) | 문서는 해당 요청에만 붙는다고 서술한다. 이후 내비게이션/RSC 요청에도 유지되는지는 **미확인**이다. 첫 요청에서 쿠키로 옮겨 두는 방식을 고려한다. |
| iOS WKWebView 기본 UA | `User-Agent` | — | iOS WKWebView 기본 UA에 `Safari/` 토큰이 없다는 통념은 1차 출처로 확인하지 못했다. → **미확인**. iOS는 커스텀 UA나 헤더에 의존하는 것을 권장한다. |
| 클라이언트 감지(JS 주입) | 클라이언트 | `runJavaScript(String)`: "Runs the given JavaScript in the context of the current page." [pub.dev](https://pub.dev/documentation/webview_flutter/latest/webview_flutter/WebViewController/runJavaScript.html) | (추론) 페이지 로드 이후에 실행되므로 첫 페인트와 hydration보다 늦을 수 있다. 레이아웃 결정에 쓰면 깜빡임이 생긴다. |

Next.js에서 읽는 방법:

- Server Component: `const ua = (await headers()).get('user-agent')`. `headers()`는 async이고 read-only이며, 쓰면 라우트가 dynamic rendering이 된다. Cache Components에서는 `<Suspense>` 밖에서 호출하면 prerender가 막힌다. [headers](https://nextjs.org/docs/app/api-reference/functions/headers)
- Proxy(Next.js 16에서 `middleware`가 이름을 바꾼 것): `userAgent(request)`가 `browser`, `device.type`, `engine`, `os` 등을 파싱해 준다. [userAgent](https://nextjs.org/docs/app/api-reference/functions/userAgent) Proxy에서 판정한 결과를 `NextResponse.next({ request: { headers } })`로 요청 헤더에 넣거나 `response.cookies.set()`으로 쿠키에 담아 앱으로 넘길 수 있다. v16.0.0에서 "Middleware is deprecated and renamed to Proxy"가 되었다. [proxy.js](https://nextjs.org/docs/app/api-reference/file-conventions/proxy)
  - Proxy는 static export에서는 지원되지 않는다. [proxy.js#platform-support](https://nextjs.org/docs/app/api-reference/file-conventions/proxy#platform-support)
  - (추론) `userAgent()`의 파서가 Android WebView를 별도 `browser.name`으로 구분하는지는 확인하지 못했다(미확인). `; wv`는 `ua` 문자열에서 직접 검사하는 편이 확실하다.

## 6. Hydration 주의점

- Next.js가 꼽는 hydration 불일치 원인에는 렌더 로직의 `typeof window !== 'undefined'` 분기와 `window`/`localStorage` 같은 브라우저 전용 API 사용이 들어 있다. [react-hydration-error](https://nextjs.org/docs/messages/react-hydration-error)
  - 따라서 `navigator.userAgent`나 `window.flutter_inappwebview` 같은 클라이언트 감지 결과로 **첫 렌더의 마크업을 바꾸면 안 된다**. 필요하면 `useEffect` 이후에 바꾼다(Solution 1). 다만 이렇게 하면 첫 페인트 이후에 레이아웃이 바뀐다.
- `suppressHydrationWarning`은 "only works one level deep"인 탈출구일 뿐이다. [같은 문서](https://nextjs.org/docs/messages/react-hydration-error#solution-3-using-suppresshydrationwarning)
  - (추론) `<html data-shell="webview">` 같은 속성을 hydration 전 인라인 스크립트로 넣고 `<html suppressHydrationWarning>`을 거는 패턴도 가능하다. 그러나 이것도 결국 클라이언트 감지이므로 아래 권장안보다 우선순위가 낮다.
- iOS의 전화번호·이메일 자동 링크 변환도 hydration 불일치를 일으킨다. `<meta name="format-detection" content="telephone=no, date=no, email=no, address=no">`로 끌 수 있다. WebView 환경에서도 해당할 수 있다. [같은 문서#common-ios-issues](https://nextjs.org/docs/messages/react-hydration-error#common-ios-issues)
- 서버 판정을 쓸 때는 **서버와 클라이언트가 같은 값을 보게** 해야 한다. 서버에서 판정한 값을 `<html data-shell=...>` 속성으로 렌더하고, 클라이언트는 그 속성만 읽는다. (추론)
- 서버 분기의 비용(추론, 위 headers/generateViewport 문서 근거):
  1. 해당 라우트가 dynamic rendering이 되어 정적 셸 이점을 잃는다.
  2. CDN 캐시를 쓰면 UA나 헤더별로 캐시 키를 나눠야 한다(`Vary`). 그렇지 않으면 WebView용 HTML이 브라우저로 새어 나갈 수 있다.
- **권장 전략**: safe area 자체는 `viewport-fit=cover` + `env()`만으로 처리한다. 이렇게 하면 마크업은 양쪽이 같고 inset 값만 환경별로 달라진다. 0이 될 곳에서는 저절로 0이 된다. WebView 분기는 "셸이 이미 inset을 처리해서 웹은 하지 말아야 하는" 경우처럼 책임 분담이 정말 다를 때로 한정한다. 이 경우에도 클래스나 속성 하나만 토글하고, CSS 변수로 inset을 0으로 덮어쓰는 식으로 범위를 좁힌다. 예: `[data-shell="webview-safe"] { --safe-top: 0px; }`

## 7. 가이드에 반영할 핵심

1. **설정은 한 줄로 끝낸다**: 루트 `app/layout.tsx`에 `export const viewport: Viewport = { viewportFit: 'cover' }`를 둔다. 공식 문서에는 빠져 있지만 타입과 렌더링에서 정식 지원된다(v16.4.2 소스). 기본 `width/initial-scale`은 key 단위 병합으로 유지된다.
2. **`generateViewport`로 UA 분기하지 않는다**: viewport는 스트리밍되지 않는다. runtime data를 읽으면 페이지가 block되고 dynamic이 된다. `cover`는 모든 환경에 줘도 무해하다(inset이 없으면 `env()`는 0).
3. **`themeColor`는 safe area 색의 해법이 아니다**: Next.js에서 viewport-fit과 별개로 출력된다. iOS 26 Safari에서의 역할 변화는 공식 확인이 안 되었고, WebView 시스템 바는 네이티브 셸 담당이다(미확인, 책임 분담 티켓 연계). inset 영역의 색은 헤더, 탭바, 페이지의 **CSS 배경**으로 정한다.
4. **패턴은 "배경은 끝까지, 콘텐츠는 inset 안쪽"**: 고정 헤더는 `padding-top`, 탭바는 `padding-bottom`, 모달은 4방향 `padding`에 `env(safe-area-inset-*, 0px)`를 넣는다. 본문 오프셋은 `calc(바 높이 + inset)`이다. `calc`용 fallback은 반드시 `0px`로 쓴다.
5. **Tailwind v4 표기**: `pt-[env(safe-area-inset-top,0px)]`, `pb-[calc(3.5rem+env(safe-area-inset-bottom,0px))]`(공백 자동 처리), 또는 `:root` 토큰과 `pt-(--safe-top)`을 쓴다. 반복되면 `@utility pt-safe`를 정의한다. 코어 내장 safe 유틸리티는 없다고 본다(미확인).
6. **WebView 감지가 필요하면 셸이 신호를 준다**: 셸이 커스텀 UA 토큰을 넣는 방식(`setUserAgent`)이 가장 확실하다. Android는 `; wv`와 `X-Requested-With`를 보조 신호로 쓴다. iOS 기본 UA로 판별하는 방식은 근거가 없다(미확인). 판정은 Proxy/서버에서 한 번 하고 `<html data-*>`로 내려 hydration 불일치를 막는다. 클라이언트 전용 감지로 첫 렌더 마크업을 바꾸지 않는다.

## 8. 남은 미확인 항목

- iOS 26 Safari의 `theme-color` 지원 중단 여부(공식 릴리스 노트 미기재)
- WKWebView / Android WebView에서 `theme-color`가 무시되는지(1차 출처 없음)
- iOS WKWebView 기본 UA의 형태(`Safari/` 토큰 유무)
- `webview_flutter` `loadRequest` 헤더가 이후 내비게이션에도 전달되는지
- Next.js `userAgent()`가 Android WebView를 별도 browser로 식별하는지
- Tailwind v4 코어의 safe area 전용 유틸리티 존재 여부
