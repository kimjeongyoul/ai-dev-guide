# [FE 최적화 1편] Core Web Vitals 정복 — LCP와 CLS 수치 개선의 기술적 실체

&nbsp;

웹 성능 최적화의 패러다임이 '단순 로딩 속도'에서 구글이 제시한 '코어 웹 바이탈(Core Web Vitals)'로 이동한 지 수년이 지났다. 하지만 여전히 많은 프로젝트가 Lighthouse 점수 100점을 목표로 하는 '실험실 데이터(Lab Data)'에 매몰되어 있다. 진정한 성능 최적화는 실제 사용자들의 다양한 기기 환경과 네트워크 상태에서 수집되는 '필드 데이터(Field Data, CrUX)'를 기반으로 해야 하며, 이는 브라우저의 렌더링 파이프라인과 리소스 로딩 우선순위에 대한 깊은 이해 없이는 불가능하다. 본 글에서는 LCP, CLS, 그리고 최근 도입된 INP 지표를 프로덕션 레벨에서 개선하기 위한 정교한 엔지니어링 전략을 다룬다.

&nbsp;

&nbsp;

---

&nbsp;

# 1. LCP (Largest Contentful Paint): 메인 스레드의 점유율 전쟁

&nbsp;

LCP는 뷰포트 내에서 가장 큰 이미지나 텍스트 블록이 렌더링되는 시점을 측정한다. 2.5초 이내가 'Good' 등급이지만, 모바일 환경에서는 네트워크 지연과 CPU 성능 한계로 인해 이 수치를 달성하기가 매우 까다롭다. 단순히 파일 용량을 줄이는 것보다, 브라우저가 해당 리소스를 얼마나 '빨리 인지하고' '빨리 실행하는가'가 핵심이다.

&nbsp;

## 1-1. 브라우저 리소스 로딩 파이프라인의 이해
브라우저(Chromium 엔진 기준)는 HTML을 수신하면서 사전 스캐너(Preload Scanner)를 통해 리소스를 미리 파악한다. 하지만 LCP 대상이 되는 히어로 이미지가 CSS의 `background-image`로 선언되어 있거나, JavaScript에 의해 동적으로 삽입된다면 사전 스캐너는 이를 발견하지 못한다. 이는 LCP 지표에 치명적인 '발견 지연(Discovery Delay)'을 유발한다.

&nbsp;

- **Fetch Priority API의 실전 적용**: 
  현대 브라우저는 리소스의 유형(JS, CSS, Image 등)에 따라 기본 우선순위를 부여한다. LCP 후보가 되는 핵심 이미지에는 `fetchpriority="high"`를 부여하여 브라우저의 내부 큐(Queue)에서 최우선 순위로 격상시켜야 한다.
  ```html
  <!-- LCP 이미지 최적화 선언 -->
  <img src="/hero-banner.webp" fetchpriority="high" alt="메인 배너">
  ```
  반대로 페이지 하단에 위치하여 초기 렌더링에 방해가 되는 이미지들에는 `fetchpriority="low"`와 `loading="lazy"`를 함께 사용하여 메인 스레드와 네트워크 대역폭의 경합을 방지해야 한다.

&nbsp;

- **Early Hints (103) 도입 전략**: 
  서버가 최종 응답(200 OK)을 준비하기 위해 DB 조회를 하는 동안, 네트워크는 유휴 상태가 된다. HTTP/2 이상에서 지원하는 Early Hints를 사용하여 브라우저에게 "나중에 이 CSS와 이미지가 필요할 테니 미리 연결하고 다운로드 시작해"라고 알려줌으로써 LCP 시점을 200~500ms 이상 앞당길 수 있다.

&nbsp;

## 1-2. 서버 응답 시간(TTFB)과 렌더링 차단 리소스
LCP의 기반이 되는 TTFB(Time to First Byte)가 늦어지면 이후 모든 공정은 뒤로 밀린다. 특히 `<head>` 태그 내부에 위치한 동기적 자바스크립트와 거대한 CSS 파일은 브라우저의 파싱을 멈추는 '렌더링 차단(Rendering Blocking)' 리소스다.

&nbsp;

- **Critical CSS 기법**: 페이지의 첫 화면(Above the Fold)을 그리는 데 필요한 최소한의 CSS만 추출하여 HTML 내부에 인라인(`<style>`)으로 삽입하고, 나머지 거대한 스타일 시트는 `rel="preload"`와 `onload` 이벤트를 활용하여 비동기로 로드한다.
- **Node.js 스트리밍 렌더링**: Next.js App Router 환경에서는 `renderToReadableStream`을 활용하여 HTML 조각을 생성되는 즉시 클라이언트로 스트리밍한다. 사용자는 전체 페이지가 완성되기 전에도 헤더와 스켈레톤 UI를 볼 수 있어 심리적 로딩 체감과 실제 LCP 지표 모두에서 이득을 본다.

&nbsp;

&nbsp;

---

&nbsp;

# 2. CLS (Cumulative Layout Shift): 시각적 안정성의 수학적 설계

&nbsp;

CLS는 사용자 의도와 상관없이 발생하는 레이아웃 이동의 합산 점수다. 구글은 이를 (이동한 요소의 면적 비율) * (이동한 거리 비율)로 계산한다. 0.1 이하를 유지해야 하며, 이는 사용자에게 '시각적 안정성'을 제공하는 핵심 지표다.

&nbsp;

## 2-1. 이미지와 비디오의 고정 영역 확보
이미지 로딩 전에는 높이가 0이었다가 로딩 후 갑자기 커지면서 아래 요소를 밀어내는 것이 CLS의 가장 흔한 주범이다.

&nbsp;

- **Aspect Ratio Boxes**: CSS의 `aspect-ratio` 속성을 사용하여 이미지 로딩 전에도 브라우저가 정확한 공간을 예약(Reserve)하게 한다. 
  ```css
  .image-container {
    width: 100%;
    aspect-ratio: 16 / 9;
    background-color: #f0f0f0; /* 스켈레톤 효과 */
  }
  ```
  현대 브라우저에서는 `<img>` 태그에 `width`와 `height` 속성만 명시해도 내부적으로 비율을 계산하여 공간을 확보해준다. 절대로 "크기 미지정" 상태로 리소스를 방치해서는 안 된다.

&nbsp;

## 2-2. 웹 폰트와 Layout Shift (FOIT vs FOUT)
사용자 정의 폰트가 로드되면서 기본 폰트와 글자 크기, 자간 차이로 인해 텍스트 영역이 미세하게 흔들리는 현상이 발생한다.

&nbsp;

- **font-display: swap의 함정**: 이 설정은 텍스트가 즉시 보이게 해주지만, 폰트 교체 시점의 레이아웃 이동은 막지 못한다.
- **Font-size-adjust**: CSS의 `font-size-adjust` 속성을 사용하여 폴백(Fallback) 폰트의 x-height를 웹 폰트와 맞춤으로써 교체 시의 이질감을 최소화한다.
- **Preconnect**: 폰트 서버(예: fonts.gstatic.com)와의 연결을 HTML 파싱 초기에 미리 맺어 폰트 다운로드 완료 시점을 앞당겨야 한다.

&nbsp;

&nbsp;

---

&nbsp;

# 3. INP (Interaction to Next Paint): 이벤트 루프의 병목 추적

&nbsp;

2024년 3월부터 FID(First Input Delay)를 대체하여 정식 지표가 된 INP는 사용자의 모든 상호작용에 대한 전체적인 응답 지연을 측정한다. 클릭, 키보드 입력 후 다음 프레임이 화면에 그려질 때까지의 시간이 200ms를 넘어가면 사용자는 '답답함'을 느낀다.

&nbsp;

## 3-1. 메인 스레드 점유와 Long Task의 해체
자바스크립트의 실행 시간이 길어지면 브라우저의 이벤트 루프는 사용자의 입력을 처리하지 못하고 블로킹된다. 50ms를 초과하는 작업은 'Long Task'로 간주된다.

&nbsp;

- **Yielding to Main Thread**: 복잡한 연산 로직 중간에 `setTimeout(..., 0)`이나 최신 `scheduler.yield()`를 삽입하여 브라우저에게 제어권을 일시적으로 넘겨주어야 한다. 이를 통해 연산 도중에도 사용자 입력을 우선적으로 처리할 수 있는 틈을 만든다.
- **Web Worker의 적극적 도입**: UI 렌더링과 관련 없는 데이터 가공, 암복호화, 대량의 객체 정렬 등은 별도의 Web Worker 스레드로 분리한다. 메인 스레드는 오직 '화면을 그리는 일'과 '사용자 입력에 반응하는 일'에만 전념하게 아키텍처를 설계해야 한다.

&nbsp;

## 3-2. 리액트 렌더링 최적화와 INP
리액트 앱에서 대규모 리스트의 상태를 업데이트할 때 발생하는 CPU 부하는 INP 지표의 핵심 방해 요소다.
- **useTransition**: 중요도가 낮은 UI 업데이트(예: 검색 결과 필터링)를 `startTransition`으로 감싸 브라우저가 사용자 입력을 최우선으로 처리하게 한다.
- **컴포넌트 메모이제이션**: 무분별한 리렌더링을 방지하기 위해 `React.memo`와 `useMemo`를 적재적소에 배치하되, 메모이제이션 자체의 오버헤드와 비교하여 전략적으로 선택해야 한다.

&nbsp;

&nbsp;

---

&nbsp;

# 4. 실전 텔레메트리: 필드 데이터 수집과 버켓 분석

&nbsp;

Lighthouse 점수가 잘 나와도 실제 사용자의 경험은 처참할 수 있다. 상용 환경에서의 실시간 모니터링 체계 구축이 필수적이다.

&nbsp;

## 4-1. web-vitals 라이브러리 연동
Google에서 제공하는 공식 라이브러리를 활용하여 실제 사용자의 LCP, CLS, INP 수치를 수집한다.

&nbsp;

```javascript
import { onLCP, onCLS, onINP } from 'web-vitals';

function sendToAnalytics({ name, delta, id, value, entries }) {
  // 수집된 지표를 분석 서버로 전송
  // entries에는 해당 지표를 발생시킨 구체적인 DOM 요소 정보가 포함되어 있어 사후 분석에 용이함
  const body = JSON.stringify({ name, value, id, attribution: entries[0] });
  if (navigator.sendBeacon) {
    navigator.sendBeacon('/api/vitals', body);
  } else {
    fetch('/api/vitals', { body, method: 'POST', keepalive: true });
  }
}

onLCP(sendToAnalytics);
onCLS(sendToAnalytics);
onINP(sendToAnalytics);
```

&nbsp;

## 4-2. 데이터 분석과 의사결정
수집된 데이터를 단순히 평균(Average)으로 보지 마라. 75백분위수(p75) 수치를 기준으로, 기기 사양별(High-end vs Low-end), 네트워크 환경별로 그룹화하여 분석해야 한다. 특정 저사양 기기군에서만 INP가 튄다면 해당 기기군에 대해서만 기능을 간소화하는 '적응형 성능 전략'을 취할 수 있다.

&nbsp;

&nbsp;

---

&nbsp;

# 5. 결론: 성능은 누적된 디테일의 산물이다

&nbsp;

코어 웹 바이탈 개선은 특정 라이브러리 하나로 해결되지 않는다. 리소스의 로딩 우선순위를 설계하고, 브라우저 엔진의 렌더링 방식을 이해하며, 메인 스레드의 부하를 끊임없이 경계하는 엔지니어링의 정수다. 

&nbsp;

당신의 서비스가 사용자에게 '부드럽고 빠른' 경험을 제공하고 있는지, 지금 바로 실험실 데이터가 아닌 실제 사용자들의 데이터로 검증하라. 성능 최적화는 '끝'이 없는 지속적인 관찰과 개선의 프로세스여야 한다.

&nbsp;

&nbsp;

---

&nbsp;

# 다음 편 예고

&nbsp;

> **[FE 최적화 2편] Next.js App Router — 서버 컴포넌트로 성능과 SEO 잡기**

&nbsp;

React 서버 컴포넌트(RSC)가 생성하는 'Flight Data'의 내부 구조는 무엇인가? 클라이언트 번들을 획기적으로 줄이는 서버 컴포넌트의 렌더링 파이프라인과, 스트리밍(Streaming)을 활용해 TTFB 지표를 극적으로 개선하는 실전 아키텍처를 분석한다. 기존 Pages Router 대비 실제 벤치마크 결과와 함께 App Router 도입 시 반드시 고려해야 할 성능적 트레이드 오프를 상세히 다룬다.

&nbsp;

&nbsp;

---

&nbsp;

웹성능최적화, 코어웹바이탈, LCP개선, CLS방지, INP성능, 프론트엔드성능, 렌더링파이프라인, 성능지표모니터링
