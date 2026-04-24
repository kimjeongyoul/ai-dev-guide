# [FE 최적화 3편] 이미지 최적화의 모든 것 — WebP, AVIF 그리고 가상화 기술의 깊이

&nbsp;

HTTP Archive의 최신 통계에 따르면, 웹 페이지 전체 용량의 약 60~70%는 여전히 이미지가 차지하고 있다. 자바스크립트 번들을 10KB 줄이려고 밤을 새우는 것보다, 이미지 로딩 전략 하나를 제대로 세우는 것이 전체 성능 지표(특히 LCP) 개선에 훨씬 압도적이고 즉각적인 영향을 미친다. 

&nbsp;

현대 웹 애플리케이션에서 이미지 최적화는 단순히 용량을 줄이는 차원을 넘어선다. 사용자 기기의 CPU 연산 비용, 메모리 점유율, 그리고 네트워크 대역폭의 효율적인 분배가 복합적으로 얽힌 엔지니어링의 영역이다. 본 글에서는 차세대 포맷의 기술적 배경부터 `next/image`의 내부 렌더링 전략, 그리고 런타임 성능을 지키는 가상화 기술(Virtualization)까지 심층 분석한다.

&nbsp;

&nbsp;

---

&nbsp;

# 1. 포맷의 진화: WebP와 AVIF의 기술적 임팩트

&nbsp;

과거 JPEG나 PNG는 압축률과 투명도 지원 사이에서 트레이드 오프가 발생했으나, 차세대 포맷은 이를 구조적으로 해결한다. 하지만 무조건 최신 포맷이 좋은 것은 아니며, 디코딩 연산 비용을 함께 고려해야 한다.

&nbsp;

## 1-1. WebP (Web Picture)
구글이 개발한 WebP는 예측 코딩(Predictive Coding)을 사용하여 주변 픽셀의 값을 기반으로 블록 단위 압축을 수행한다.
- **성능**: JPEG 대비 최대 34%, PNG 대비 26% 더 높은 압축률을 제공한다. 손실(Lossy)과 무손실(Lossless) 압축을 모두 지원하며, 투명도(Alpha channel) 처리가 가능해 PNG를 완벽히 대체할 수 있다.
- **호환성**: 현재 대부분의 모던 브라우저에서 네이티브로 지원된다.

&nbsp;

## 1-2. AVIF (AV1 Image File Format)
비디오 코덱인 AV1을 기반으로 한 이미지 포맷으로, 현재 가장 뛰어난 압축 효율을 보여준다.
- **성능**: 동일한 SSIM(구조적 유사도) 기준, WebP보다도 약 20% 이상 용량이 작다. 특히 텍스트가 섞인 이미지나 고대비(High Contrast) 영역의 블록 노이즈(Artifacts) 억제력이 탁월하다. 10비트 이상의 색 농도를 지원하여 HDR 콘텐츠에도 적합하다.
- **트레이드 오프**: 압축률이 높은 만큼 브라우저가 이미지를 디코딩하여 화면에 그리는 CPU 연산 비용이 JPEG나 WebP보다 높다. 저사양 모바일 기기에서는 오히려 렌더링 지연을 유발할 수 있으므로, 파일 크기와 디코딩 시간 사이의 손익 분기점을 테스트해야 한다.

&nbsp;

## 1-3. Content Negotiation 전략
브라우저 파편화를 대비해 `<picture>` 태그를 활용한 폴백(Fallback) 전략이 필수적이다.
```html
<picture>
  <source type="image/avif" srcset="/images/hero.avif">
  <source type="image/webp" srcset="/images/hero.webp">
  <img src="/images/hero.jpg" alt="히어로 배너" loading="eager" fetchpriority="high">
</picture>
```
브라우저는 위에서부터 차례대로 지원 가능한 포맷을 선택하여 다운로드한다.

&nbsp;

&nbsp;

---

&nbsp;

# 2. `next/image`의 내부 최적화 아키텍처

&nbsp;

Next.js 환경에서 `next/image` 컴포넌트는 단순한 HTML `<img>` 래퍼(Wrapper)가 아니다. 이는 CDN, 서버 인프라, 클라이언트 브라우저를 아우르는 정교한 이미지 처리 파이프라인이다.

&nbsp;

## 2-1. On-demand Image Optimization
빌드 타임에 모든 이미지를 변환하는 정적 사이트 생성(SSG) 방식은 이미지가 수만 장인 서비스에서 빌드 시간을 무한대로 늘린다. 
`next/image`는 사용자의 최초 요청 시점(On-demand)에 이미지를 최적화된 포맷과 사이즈로 리사이징하여 응답한다. 변환된 결과물은 Vercel의 Edge Cache나 자체 서버 메모리에 캐싱되어, 두 번째 요청부터는 0ms에 가까운 응답 속도를 보여준다.

&nbsp;

## 2-2. 렌더링 전략: Blur-up과 Layout Shift 방지
- **Placeholder**: `placeholder="blur"` 속성을 주면, 10px 내외의 초저해상도 Base64 이미지를 HTML에 미리 인라인 삽입한다. 실제 이미지가 로드될 때까지 가우시안 블러 처리된 플레이스홀더를 보여주어 사용자의 체감 성능(Perceived Performance)을 높인다.
- **Size Reservation**: `width`와 `height`를 필수로 요구함으로써, 브라우저가 이미지 로딩 전에도 정확한 Aspect Ratio를 계산하여 레이아웃 영역을 확보(Reserve)하게 만든다. 이는 CLS 지표를 완벽하게 방어한다.

&nbsp;

&nbsp;

---

&nbsp;

# 3. 런타임 최적화: 리스트 가상화 (Virtualization)

&nbsp;

수천 개의 이미지가 포함된 상품 목록이나 무한 스크롤(Infinite Scroll) 피드에서 모든 이미지를 DOM에 올리는 것은 자살 행위다. 
수천 개의 `<img>` 태그가 메모리에 상주하면 브라우저의 가비지 컬렉션(GC)이 빈번하게 발생하고, 메인 스레드는 레이아웃 계산에 짓눌려 결국 프레임 드랍(Jank)과 앱 크래시로 이어진다.

&nbsp;

## 3-1. Windowing 메커니즘
이때 필요한 것이 **리스트 가상화(Virtualization)** 기술이다.
- **개념**: 사용자의 현재 뷰포트에 노출된 영역(Window)과 스크롤 방향의 아주 작은 여유분(Overscan buffer)만 실제 DOM 노드로 렌더링하고, 나머지 영역은 빈 공간(Spacer)으로 처리한다.
- **동작**: 스크롤이 발생할 때마다 보이지 않게 된 상단의 노드는 제거(Unmount)하고, 새로 나타난 하단의 영역에 새로운 노드를 삽입(Mount)한다.

&nbsp;

## 3-2. 노드 재활용 (Recycling)과 라이브러리
단순한 생성/삭제를 넘어, 기존 DOM 노드를 파괴하지 않고 내부의 텍스트나 `src` 속성 데이터만 교체(Recycle)하여 메모리 재할당 비용을 극단적으로 줄일 수 있다.
`@tanstack/react-virtual`이나 `react-window` 같은 라이브러리가 이러한 수학적 계산과 렌더링 메커니즘을 추상화하여 제공한다. 이 기술을 적용하면 10만 건 이상의 목록에서도 실제 DOM 노드는 수십 개로 유지되며, 메모리 사용량 곡선이 수평을 유지하게 된다.

&nbsp;

&nbsp;

---

&nbsp;

# 4. 이미지 전송 최적화: CDN과 브라우저 캐싱

&nbsp;

이미지 파일 자체를 최적화했다면, 다음은 그것을 사용자 기기까지 가져가는 '배달' 과정의 최적화다.

&nbsp;

- **Image CDN (Content Delivery Network)**: Cloudinary, Imgix 또는 AWS CloudFront + Lambda@Edge 구조를 통해 사용자와 지리적으로 가장 가까운 엣지 로케이션에서 이미지를 서빙해야 한다.
- **Cache-Control 헤더 조작**: 원본 이미지가 절대 변하지 않는 구조라면, 강력한 브라우저 캐싱 정책이 필요하다.
  ```http
  Cache-Control: public, max-age=31536000, immutable
  ```
  `immutable` 지시어를 추가하면, 사용자가 새로고침을 누르더라도 브라우저는 서버에 304 Not Modified 확인 요청조차 보내지 않고 즉시 로컬 디스크 캐시를 사용하여 이미지를 복원한다.

&nbsp;

&nbsp;

---

&nbsp;

# 결론: 이미지는 시스템 리소스의 축소판이다

&nbsp;

이미지 최적화는 단순히 화질을 뭉개서 용량을 줄이는 1차원적인 작업이 아니다. 네트워크 대역폭, 브라우저의 디코딩 CPU 리소스, DOM 노드 수에 따른 메모리 한계를 종합적으로 이해하고 분배하는 고도의 시스템 설계 기법이다. 

&nbsp;

차세대 포맷으로 대역폭을 아끼고, On-demand 리사이징으로 서버 스토리지를 아끼며, 가상화 기술로 클라이언트 메모리를 지키는 것. 이 모든 디테일의 합이 곧 프로덕션 레벨 서비스의 완성도를 결정짓는다.

&nbsp;

&nbsp;

---

&nbsp;

# 다음 편 예고

&nbsp;

> **[FE 최적화 4편] 상태 관리의 선택 — Zustand와 React Query로 복잡성 줄이기**

&nbsp;

Redux의 무거운 보일러플레이트와 단일 스토어 아키텍처가 어떻게 리액트의 리렌더링 파이프라인을 파괴하는지 분석한다. 서버 데이터와 클라이언트 UI 상태를 명확히 분리하고, 각각 React Query와 Zustand를 도입하여 렌더링 범위를 최소화하고 INP 수치를 극적으로 개선하는 실무 아키텍처를 깊이 있게 다룬다.

&nbsp;

&nbsp;

---

&nbsp;

이미지최적화, WebP, AVIF, NextjsImage, 리스트가상화, 가상스크롤, 브라우저캐싱, 프론트엔드성능
