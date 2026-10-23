# Safe Area in Hybrid Web

하이브리드웹 환경에서 safe area를 다루는 실무 지식을 정리하는 연구 컨텍스트. 웹 프론트엔드 개발자가 네이티브 셸과 협업하며 화면 가장자리 처리를 올바르게 하기 위한 공통 어휘.

## Language

**하이브리드웹 (Hybrid Web)**:
네이티브 앱 안의 WebView가 웹 콘텐츠를 렌더링하는 환경. 이 컨텍스트에서는 Flutter 앱이 Next.js 웹을 WebView로 띄우는 구성을 기준으로 한다.
_Avoid_: 웹앱, 하이브리드앱(모호), 모바일웹

**네이티브 셸 (Native Shell)**:
WebView를 품고 있는 네이티브 쪽 앱 코드. 이 컨텍스트에서는 Flutter 앱.
_Avoid_: 앱 껍데기, 래퍼, 컨테이너 앱

**Safe area**:
노치·Dynamic Island·상태 바·홈 인디케이터·제스처 내비게이션 등 시스템 UI나 물리적 형태에 가려지지 않아 콘텐츠를 안전하게 배치할 수 있는 화면 영역.
_Avoid_: 안전 영역(혼용 금지 시 원문 사용), notch area

**Inset**:
화면 가장자리에서 safe area 경계까지의 거리. 웹에서는 `env(safe-area-inset-*)`, Flutter에서는 `MediaQuery.padding`/`viewPadding`, Android에서는 `WindowInsets`로 표현된다.
_Avoid_: 여백, margin

**비교 환경 (Reference Environment)**:
하이브리드웹과의 차이를 설명하기 위해서만 다루는 환경. 모바일 브라우저(Safari/Chrome)와 PWA standalone 모드.
_Avoid_: 일반 웹(모호)
