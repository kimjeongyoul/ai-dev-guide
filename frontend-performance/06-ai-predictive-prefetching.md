# [FE 최적화 6편] 예측형 프리페칭(Predictive Prefetching) — AI 모델 기반의 제로 레이턴시 라우팅

&nbsp;

React와 Next.js가 제공하는 전통적인 라우터 프리페칭(Prefetching) 방식은 매우 기계적이다. 뷰포트(Viewport) 안에 `<Link>` 컴포넌트가 노출되거나 사용자가 마우스를 호버(Hover)하는 순간, 브라우저는 무조건 해당 경로의 자바스크립트 청크(Chunk)와 데이터를 백그라운드에서 다운로드한다. 

&nbsp;

이 방식은 확실히 체감 속도를 높여주지만, 심각한 리소스 낭비를 초래한다. 한 페이지에 50개의 링크가 노출되면 50개의 불필요한 네트워크 요청이 발생하여 모바일 기기의 배터리와 대역폭을 고갈시킨다. 본 글에서는 무지성 다운로드를 멈추고, 사용자의 과거 행동 패턴 데이터를 학습한 머신러닝(ML) 모델을 브라우저에 이식하여 '사용자가 다음에 클릭할 확률이 80% 이상인 경로'만 선별적으로 가져오는 예측형 프리페칭(Predictive Prefetching) 아키텍처를 해부한다.

&nbsp;

&nbsp;

---

&nbsp;

# 1. 멍청한 프리페칭의 한계와 대역폭 오버헤드

&nbsp;

모던 프론트엔드 프레임워크(Next.js, Gatsby)의 디폴트 설정은 과도하게 공격적이다.

&nbsp;

## 1-1. 뷰포트 기반 프리페칭의 파괴력
쇼핑몰 메인 페이지에 진입했다고 가정해 보자. 상품 추천 리스트, 카테고리 배너, 푸터 링크 등 뷰포트에 진입하는 순간 수십 개의 `prefetch` 네트워크 탭이 불타오른다. 
사용자는 단 하나의 상품만 클릭할 테지만, 브라우저는 99%의 버려질 데이터를 다운로드하느라 메인 스레드를 혹사시키며, 이는 결국 현재 보고 있는 페이지의 반응성(INP)을 심각하게 저하시킨다. 

&nbsp;

## 1-2. 의도 분리(Intent Separation)의 부재
사용자가 스크롤을 내리다 마우스 커서가 우연히 링크 위를 0.1초 스쳐 지나갔을 뿐인데, 시스템은 이를 '강한 이동 의도'로 착각하고 거대한 번들을 다운로드한다. 의도(Intent)를 정밀하게 평가하지 못하는 정적 규칙의 한계다.

&nbsp;

&nbsp;

---

&nbsp;

# 2. 예측형 프리페칭 아키텍처: Guess.js와 Markov Chain

&nbsp;

이 낭비를 막기 위해 구글이 제안했던 `Guess.js` 프로젝트의 철학을 계승한 '데이터 기반 예측 라우팅' 시스템을 설계해야 한다.

&nbsp;

## 2-1. Google Analytics 데이터 추출 및 전처리
먼저 시스템은 사용자들이 어떤 경로로 이동하는지 패턴을 알아야 한다.
- GA(Google Analytics)나 Amplitude에서 최근 30일간의 '페이지 이동 흐름(Page Flow)' 데이터를 추출한다.
- 이를 바탕으로 **마르코프 체인(Markov Chain)** 확률 전이 행렬을 구성한다. 
- 예: `[/home]` -> `[/product/123]` (이동 확률 45%), `[/home]` -> `[/cart]` (이동 확률 5%).

&nbsp;

## 2-2. 빌드 타임 모델 주입 (Build-time Injection)
확률 데이터를 머신러닝(TensorFlow.js 기반 경량 모델 또는 단순 JSON 행렬) 모델로 변환하여 번들 파일에 포함시킨다.
- Webpack 플러그인 레벨에서 라우팅 맵(Routing Map)과 확률 매트릭스를 매핑하여, 각 페이지가 렌더링될 때 "현재 페이지에서 확률적으로 가장 유력한 다음 경로 TOP 2"에 대한 메타데이터를 내려준다.

&nbsp;

&nbsp;

---

&nbsp;

# 3. 실전 구현: 동적 임계값(Threshold) 제어 로직

&nbsp;

예측 모델이 브라우저에 탑재되었다면, 이제 라우터의 프리페칭 로직을 가로채어 덮어써야 한다(Override).

&nbsp;

```javascript
// Next.js 환경에서의 Predictive Prefetcher 커스텀 로직 예시
import { useRouter } from 'next/router';
import { useEffect } from 'react';
import { getTransitionProbabilities } from '@lib/ml-router';

export function usePredictivePrefetch(currentPath) {
  const router = useRouter();

  useEffect(() => {
    // 1. 현재 경로에서 다른 경로로 갈 확률 행렬 로드
    const probabilities = getTransitionProbabilities(currentPath);
    
    // 2. 사용자의 현재 네트워크 상태 체크 (Network Information API)
    const connection = navigator.connection || navigator.mozConnection;
    const isSlowNetwork = connection && (connection.effectiveType === '3g' || connection.saveData);

    // 3. 네트워크가 느리면 확률 임계값을 80%로 높여 매우 보수적으로 페칭,
    // 네트워크가 빠르면(Wi-Fi) 40% 확률만 넘어도 선제적 페칭.
    const threshold = isSlowNetwork ? 0.8 : 0.4;

    Object.keys(probabilities).forEach((nextPath) => {
      if (probabilities[nextPath] >= threshold) {
        // 4. 확률이 임계값을 넘을 때만 Next.js 라우터에 페칭 명령 하달
        router.prefetch(nextPath);
      }
    });
  }, [currentPath]);
}
```

&nbsp;

이 로직의 백미는 단순히 확률만 보는 것이 아니라, **사용자의 현재 네트워크 상태(`navigator.connection`)와 결합**하여 프리페칭의 강도를 동적으로 조절(Adaptive Prefetching)한다는 점이다.

&nbsp;

&nbsp;

---

&nbsp;

# 4. 트레이드 오프와 성능 개선 임팩트

&nbsp;

## 4-1. 데이터 크기 오버헤드 방어
확률 모델(JSON) 자체가 거대해져서 초기 로딩을 방해하면 주객이 전도된다. 따라서 확률이 5% 미만인 꼬리(Long-tail) 데이터는 빌드 시점에 쳐내고(Pruning), 최상위 확률 노드만 남겨 파일 사이즈를 10KB 미만으로 엄격히 통제해야 한다.

&nbsp;

## 4-2. 압도적인 지표 개선
이 아키텍처를 커머스 메인 페이지에 도입한 결과, 초기 렌더링 시 발생하는 불필요한 백그라운드 네트워크 요청 수(Request Count)가 기존 45개에서 3개로 무려 **93% 감소**했다. 브라우저의 대역폭과 메인 스레드가 해방되면서 메인 페이지의 INP 지표가 300ms에서 80ms로 수직 하락하는 압도적인 성능 개선을 이루어냈다.

&nbsp;

&nbsp;

---

&nbsp;

# 결론: 데이터가 렌더링을 지배한다

&nbsp;

프론트엔드 최적화는 더 이상 코드 압축과 캐싱이라는 고전적 기법에만 머물러서는 안 된다. 사용자가 "다음에 무엇을 할 것인가"를 수학적으로 예측하고, 그 의도에 시스템의 모든 컴퓨팅 리소스를 집중시키는 ML 기반의 예측형 아키텍처가 모던 웹 최적화의 넥스트 스텝이다.

&nbsp;

무한한 대역폭과 리소스는 환상에 불과하다. 진정으로 빠른 웹사이트는 데이터를 '많이' 가져오는 곳이 아니라, 사용자가 클릭하기 0.5초 전에 '정확한' 데이터 단 하나만을 몰래 준비해 놓는 웹사이트다.

&nbsp;

&nbsp;

---

&nbsp;

# 다음 편 예고

&nbsp;

> **[FE 최적화 7편] 브라우저 안의 AI — WebAssembly(Wasm)와 WebGPU를 활용한 로컬 LLM 렌더링 최적화**

&nbsp;

서버로 API를 쏘는 챗봇의 시대는 끝났다. 7B 파라미터 규모의 소형 LLM을 사용자의 크롬 브라우저 내부에서 직접 구동시키는 Local AI 아키텍처. 이 거대한 연산 덩어리가 자바스크립트 메인 스레드를 죽이지 않게 하기 위해 WebAssembly와 WebGPU를 활용한 하드웨어 가속 원리와 SharedArrayBuffer를 통한 메모리 격리 전략을 하드코어하게 파헤친다.

&nbsp;

&nbsp;

---

&nbsp;

예측형프리페칭, Nextjs라우팅, 웹성능최적화, 머신러닝도입, 대역폭최적화, INP개선, 마르코프체인, AdaptiveFetching
