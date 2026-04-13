---
{"dg-publish":true,"permalink":"/posts/html-in-canvas-api/","tags":["canvas","HTML"],"created":"2026-04-12","updated":"2026-04-12"}
---


## 목차

1. [문제: HTML과 Canvas의 단절](#1-문제-html과-canvas의-단절)
2. [이전: 개발자들의 3가지 우회로](#2-이전-개발자들의-3가지-우회로)
3. [HTML in Canvas API 해부](#3-html-in-canvas-api-해부)
4. [이후: 무엇이 달라지는가](#4-이후-무엇이-달라지는가)
5. [실제 코드로 보는 Before / After](#5-실제-코드로-보는-before--after)
6. [현재 한계와 주의사항](#6-현재-한계와-주의사항)
7. [전망: 웹의 다음 단계](#7-전망-웹의-다음-단계)

-----

## 1. 문제: HTML과 Canvas의 단절

웹에는 오래된 모순이 있다. 텍스트, 폼, 접근성, 다국어—이 모든 것을 완벽하게 다루는 HTML이 있고, 픽셀 하나하나를 GPU로 조작할 수 있는 Canvas가 있다. 그런데 이 둘은 서로를 볼 수 없다.

WebGL로 3D 게임을 만들거나, 인터랙티브 데이터 시각화를 구현하거나, VR/XR 씬을 구성할 때 반드시 부딪히는 순간이 있다. “이 HTML 버튼을 3D 표면에 붙이고 싶다”, “이 텍스트 박스에 물결 셰이더를 입히고 싶다”, “이 대시보드를 비디오로 export하고 싶다”. 그 순간 개발자는 벽에 부딪힌다.

> “Canvas는 HTML을 모른다. HTML은 GPU를 모른다.”

이것은 단순한 불편함이 아니다. 접근성(a11y), 다국어(i18n), 스크린 리더 지원, 키보드 탐색—이 모든 웹의 장점이 Canvas 경계를 넘는 순간 사라진다. WebGL 씬 안에 들어가는 UI는 처음부터 다시 구현해야 하고, 그 구현체는 당연히 접근성이 없다.

WICG(Web Incubator Community Group)의 **HTML in Canvas** 제안은 이 문제를 근본부터 해결한다. HTML 요소를 WebGL 텍스처로, 살아있는 채로 가져오는 것이다.

-----

## 2. 이전: 개발자들의 3가지 우회로

기존에도 HTML을 Canvas에 그리는 방법이 전혀 없지는 않았다. 다만 모두 심각한 한계를 가지고 있었다.

### 방법 1: CSS 포지셔닝으로 Canvas 위에 HTML 덮기

가장 흔한 접근법이다. Canvas를 배경에 두고, HTML 요소를 `position: absolute`로 그 위에 겹쳐 놓는다. 2D 표면에서는 어느 정도 동작하지만, WebGL의 3D 씬 안에서는 원천적으로 불가능하다. HTML은 z 축을 모르기 때문이다.

```css
/* CSS 포지셔닝 트릭 — 3D에서는 동작 불가 */
.canvas-wrapper {
  position: relative;
}
canvas { position: absolute; inset: 0; }
.html-overlay {
  position: absolute;
  top: 50px; left: 50px;
  /* 이 요소는 WebGL 씬 '안에' 있는 게 아니다 */
  /* 그냥 화면 앞에 떠 있을 뿐 */
}
```

### 방법 2: SVG foreignObject + Canvas drawImage

SVG의 `<foreignObject>`에 HTML을 넣고, 이를 `data:` URI로 변환한 뒤 Canvas에 그리는 방법이다. 실제로 작동하긴 하지만 여러 이유로 실용성이 떨어진다.

```javascript
// SVG foreignObject 방법 — 느리고, 보안 제약 많음
const svg = `
  <svg xmlns="http://www.w3.org/2000/svg">
    <foreignObject width="400" height="200">
      <div xmlns="http://www.w3.org/1999/xhtml">
        ${htmlContent}
      </div>
    </foreignObject>
  </svg>`;

const blob = new Blob([svg], { type: 'image/svg+xml' });
const url  = URL.createObjectURL(blob);
const img  = new Image();
img.onload = () => ctx.drawImage(img, 0, 0);
img.src    = url;

// 문제: 외부 리소스 차단, WebGL 텍스처로 못씀,
//       실시간 업데이트 없음, 접근성 없음
```

### 방법 3: Canvas 안에서 텍스트/UI 직접 구현

가장 고통스러운 방법이다. `ctx.fillText()`, `ctx.strokeRect()`로 HTML이 기본 제공하는 모든 것을 손수 다시 구현한다. 줄바꿈 로직, 커서 깜빡임, 스크롤, 키보드 이벤트, 국제화(CJK 글꼴, RTL 텍스트)… 수천 줄의 코드를 써도 `<input type="text">` 하나의 기능을 따라가지 못한다.

### Before vs After 요약

|          |Before — 우회로의 한계|After — HTML in Canvas|
|----------|----------------|----------------------|
|3D 씬 배치   |❌ 불가            |✅ 가능                  |
|실시간 업데이트  |❌ 없음            |✅ `paint` 이벤트         |
|GPU 셰이더 연동|❌ 불가            |✅ WebGL 텍스처로 직접       |
|접근성 트리    |❌ 소실            |✅ 보존                  |
|CSS 스타일 유지|❌ 안됨            |✅ 그대로 렌더링             |
|라이브 미디어 캡처|❌ 불가            |✅ video, input 지원     |
|보안 모델     |⚠️ 우회 많음         |✅ 브라우저 네이티브           |

-----

## 3. HTML in Canvas API 해부

API는 단순하다. 세 가지 핵심 기본 요소만 이해하면 된다.

### Primitive 01 — `layoutsubtree` 속성

`<canvas layoutsubtree>`로 선언하면, 이 canvas의 자식 HTML 요소들이 실제 레이아웃을 가지게 된다. `getBoundingClientRect()`가 값을 반환하고, 히트 테스팅에도 참여한다. 이 속성 없이는 canvas 자식 요소들은 레이아웃도, 렌더링도 없다.

### Primitive 02 — `drawElement()` / `texElementImage2D()`

2D Canvas에서는 `ctx.drawElement(element, x, y)`, WebGL에서는 `gl.texElementImage2D(target, level, internalformat, format, type, element)`를 사용한다. HTML 요소를 그 순간의 렌더링 상태 그대로 픽셀로 올린다. GPU는 이제 당신의 `<div>`를 텍스처로 볼 수 있다.

### Primitive 03 — `paint` 이벤트

Canvas 자식 요소의 시각적 상태가 변할 때마다 canvas에 `paint` 이벤트가 발생한다. rAF 루프를 돌리지 않아도 된다. 텍스트가 바뀌거나, 비디오 프레임이 넘어가거나, CSS transition이 돌아갈 때만 정확히 한 번 발생한다.

```javascript
// 가장 기본적인 사용 예시
const canvas = document.querySelector('canvas[layoutsubtree]');
const gl     = canvas.getContext('webgl');
const el     = document.getElementById('my-html-content');

// HTML이 변경될 때마다 자동으로 텍스처 업로드
canvas.onpaint = () => {
  gl.bindTexture(gl.TEXTURE_2D, texture);
  gl.texElementImage2D(
    gl.TEXTURE_2D, 0,
    gl.RGBA, gl.RGBA,
    gl.UNSIGNED_BYTE,
    el  // ← 살아있는 HTML 요소
  );
};
```

> **활성화 방법:** Chrome Canary 138.0.7175.0 이상에서 `chrome://flags/#canvas-draw-element`를 Enable로 설정하거나, `--enable-blink-features=CanvasDrawElement` 플래그로 실행한다. 아직 정식 표준이 아니므로 프로덕션 사용은 불가하다.

-----

## 4. 이후: 무엇이 달라지는가

이 API가 가져오는 변화는 “HTML을 예쁘게 꾸밀 수 있다” 수준이 아니다. 웹이 지금까지 할 수 없었던 것들이 가능해진다.

### 1. 픽셀 단위 HTML 왜곡

CSS `transform`은 요소 전체를 이동, 회전, 스케일할 수 있다. 하지만 픽셀 하나하나를 서로 다른 방향으로 이동시키는 것, 즉 텍스트 중간을 물결처럼 왜곡하거나 특정 영역만 배럴 디스토션을 주는 것은 CSS로 불가능하다. GLSL 셰이더는 UV 좌표를 임의의 함수로 리매핑할 수 있기 때문에 HTML 텍스처를 원하는 대로 변형할 수 있다.

### 2. 두 HTML 상태를 동시에 렌더링해 블렌딩

라이트 모드와 다크 모드를 각각 텍스처로 올려서, 셰이더가 픽셀 단위로 두 상태를 블렌딩하는 다크모드 전환 효과를 구현할 수 있다. View Transitions API는 두 스냅샷 사이에 CSS 애니메이션을 제공하지만, HTML in Canvas는 GLSL로 임의의 수학 함수를 적용할 수 있다는 점이 차이점이다. 불꽃, 노이즈, 스캔라인 등등.

### 3. WebGL/WebGPU 3D 씬 안의 실제 HTML UI

게임의 인게임 컴퓨터 화면, XR 환경의 가상 패널, 3D 씬 안의 정보 디스플레이—이 모든 것에 진짜 HTML을 붙일 수 있다. `<video>`도, `<input>`도, 살아서 동작하는 채로.

### 4. HTML 콘텐츠의 이미지/비디오 Export

스크린샷 도구나 헤드리스 브라우저 없이도, 웹 페이지의 특정 섹션을 VideoEncoder로 보내 MP4로 녹화하거나 이미지로 저장할 수 있다. 오늘날 복잡한 서버사이드 렌더링으로 해결하던 것들이 클라이언트에서 가능해진다!

안그래도 요즘 사내에서 쓰고 있는 디자인 툴에서 이미지나 PDF로 추출하는 로직을 구현하느라 애를 먹고 있는데, 폰트깨짐, 서버 트래픽 이슈 등등 해결하지 못한 문제들이 남아있다. 이 실험기능으로 시도해볼 법 하다는 생각이 든다. 좀 더 정리를 해보자.

##### 지금까지 HTML → 이미지/영상 만들기가 왜 힘들었나
웹에서 “이 HTML 화면을 PNG로 저장하거나 MP4로 녹화하고 싶다”는 수요는 엄청 많다. 청구서 export, 차트 다운로드, 소셜 공유 이미지 생성, 화면 녹화 등. 그런데 현재 방법들은 모두 근본적인 결함이 있다.

###### 방법 A: html2canvas / html-to-image (클라이언트 라이브러리)
이 라이브러리들은 DOM을 읽어서 브라우저의 렌더링 엔진을 JavaScript로 직접 재구현해 `<canvas>`에 다시 그린다. 서버 호출도 없고, 실제 브라우저 렌더링도 사용하지 않는다. ￼ 쉽게 말해 렌더링 엔진을 JS로 따라 만든 것이라 정확도에 한계가 있다.


```js
// 현재 방식 — 브라우저 렌더링을 JS로 흉내내는 것
const canvas = await html2canvas(element, { scale: 2, useCORS: true });
const png = canvas.toDataURL('image/png');

```

실제 문제들:
	•	CSS grid, clip-path, 일부 transform은 재현 못 함
	•	CORS가 특히 고통스러움. 페이지가 CDN에서 이미지를 불러오면 canvas가 tainted 상태가 되어 toDataURL()이 보안 오류를 던짐
	•	폰트, SVG, backdrop-filter 등이 종종 깨짐
	•	복잡한 DOM 구조에서는 성능이 급격히 떨어짐

###### 방법 B: Puppeteer / Playwright (헤드리스 브라우저)
서버에서 실제 Chrome을 무헤드로 띄워서 스크린샷을 찍는 방법이 있다. 렌더링 정확도는 완벽하지만 운영 비용이 크다.
동시에 100개 요청이 오면 최소 2030개의 브라우저 인스턴스가 필요하고, Chrome 메모리만 615GB입니다. 요청 큐잉, 타임아웃 처리, 재시도 로직도 따로 구축해야 한다. 트래픽 급증 시 메모리 부족으로 서비스가 죽는 일이 실제로 일어난다.

사용자 요청 → 서버 → Chrome 인스턴스 띄우기 → 페이지 로드 → 스크린샷 → 반환
         ↑ 이 과정이 수 초 걸리고, 인스턴스마다 RAM 수백 MB 소비


HTML in Canvas가 이 문제를 어떻게 해결하나
핵심은 브라우저가 이미 렌더링한 결과물을 픽셀로 직접 가져오는 것이다. 재구현도, 서버도 필요 없다.

이미지 Export (PNG/JPEG):

```js
// HTML in Canvas 방식
const canvas = document.querySelector('canvas[layoutsubtree]');
const el     = document.getElementById('invoice');  // 실제 DOM

canvas.onpaint = () => {
  // 브라우저가 렌더링한 픽셀을 그대로 canvas에 올림
  ctx.drawElement(el, 0, 0);

  // 그냥 toBlob() — 재구현 없음, 서버 없음, 정확도 100%
  canvas.toBlob(blob => {
    const url = URL.createObjectURL(blob);
    downloadLink.href = url;
  }, 'image/png');
};

```


영상 Export (MP4):
제안서 저자들이 명시적으로 언급한 핵심 활용 사례 중 하나가 바로 VideoEncoder로 보내는 readback이다. ￼ WebCodecs API의 VideoEncoder와 연결하면 서버 없이 브라우저에서 직접 MP4를 만들 수 있다.

```js
// HTML → Canvas 픽셀 → VideoEncoder → MP4
const encoder = new VideoEncoder({
  output: (chunk) => muxer.addVideoChunk(chunk),
  error:  (e)     => console.error(e),
});

encoder.configure({ codec: 'avc1', width: 1280, height: 720, framerate: 30 });

// paint 이벤트마다 프레임 추가
canvas.onpaint = () => {
  ctx.drawElement(htmlSection, 0, 0);  // HTML → canvas 픽셀

  const frame = new VideoFrame(canvas, { timestamp: performance.now() * 1000 });
  encoder.encode(frame);               // canvas → MP4 프레임
  frame.close();
};

```

이 방식으로 OffscreenCanvas와 VideoEncoder를 조합하면 실시간보다 10배 빠르게 브라우저에서 MP4를 생성할 수 있다.

정리하면



|          |html2canvas  |Puppeteer  |HTML in Canvas   |
|----------|-------------|-----------|-----------------|
|렌더링 정확도   |❌ JS 재구현 (낮음)|✅ 실제 Chrome|✅ 실제 브라우저 픽셀     |
|서버 필요     |❌ 없음         |✅ 필요       |❌ 없음             |
|운영 비용     |낮음           |**높음**     |낮음               |
|동영상 export|❌ 어려움        |⚠️ 별도 구현    |✅ VideoEncoder 직결|
|CSS 완전 지원 |❌ 일부 깨짐      |✅          |✅                |
|CORS 문제   |❌ 자주 발생      |✅ 없음       |✅ 없음             |

한 마디로, 지금까지 “정확하게 export”하려면 서버가 필수였는데, HTML in Canvas는 브라우저가 이미 그린 픽셀을 그대로 꺼내 쓰기 때문에 클라이언트에서 완벽한 정확도가 가능해지는 것이다. Figma나 Canva 같은 툴이 “다운로드” 버튼을 누를 때 서버를 거치지 않고 처리할 수 있게 되는 셈이다.

### 5. 접근성과 GPU 효과의 공존

기존 Canvas 기반 UI는 접근성 트리에 존재하지 않는다. HTML in Canvas로 그린 요소는 `<canvas>`의 fallback content로 연결되어 스크린 리더가 읽을 수 있다. GPU 효과를 주면서도 접근성을 잃지 않는 첫 번째 방법이다.

-----

## 5. 실제 코드로 보는 Before / After

구체적인 사례로 비교해보자. “WebGL 씬 안의 곡면에 텍스트 박스를 렌더링”하는 시나리오다.

### Before: Canvas 2D API로 텍스트 직접 구현

```javascript
// 기존 방식: 모든 것을 Canvas API로 직접 그림
// 100줄로도 기본 텍스트 박스 하나를 못 만든다

function drawTextBox(ctx, text, x, y, width) {
  // 배경
  ctx.fillStyle = '#fff';
  ctx.fillRect(x, y, width, 120);

  // 텍스트 스타일
  ctx.font = '16px sans-serif';
  ctx.fillStyle = '#000';

  // 줄바꿈 직접 구현 (CJK? RTL? 모두 직접 처리해야 함)
  const words = text.split(' ');
  let line = '', lineY = y + 20;

  for (const word of words) {
    const test = line + word + ' ';
    if (ctx.measureText(test).width > width - 20) {
      ctx.fillText(line, x + 10, lineY);
      line  = word + ' ';
      lineY += 22;
    } else {
      line = test;
    }
  }
  ctx.fillText(line, x + 10, lineY);

  // 스크롤? 커서? 선택? 한글 조합? 불가능에 가까움
}
```

### After: HTML in Canvas로 진짜 HTML 사용

```html
<!-- HTML 구조: canvas 안에 실제 요소를 선언 -->
<canvas id="scene" layoutsubtree>
  <!-- 이 div는 실제 DOM. 레이아웃, 스타일, 접근성 모두 살아있음 -->
  <div id="card" class="info-card">
    <h2>실시간 데이터</h2>
    <p>이 텍스트는 GPU 셰이더가 읽는 텍스처다.</p>
    <input type="text" placeholder="3D 씬 안의 input">
  </div>
</canvas>
```

```javascript
// JavaScript: HTML → WebGL texture, 셰이더 적용
const canvas = document.getElementById('scene');
const card   = document.getElementById('card');
const gl     = canvas.getContext('webgl');

// 텍스처 생성
const tex = gl.createTexture();

// HTML이 바뀔 때마다 자동으로 텍스처 갱신
canvas.onpaint = () => {
  gl.bindTexture(gl.TEXTURE_2D, tex);
  gl.texElementImage2D(        // ← 핵심!
    gl.TEXTURE_2D, 0,
    gl.RGBA, gl.RGBA,
    gl.UNSIGNED_BYTE,
    card                        // ← 살아있는 HTML 요소
  );
  renderScene();                // 셰이더로 3D 씬에 붙이기
};

// JS로 HTML을 수정하면 → paint 이벤트 → 텍스처 자동 갱신
setInterval(() => {
  card.querySelector('h2').textContent = `데이터: ${Math.random().toFixed(3)}`;
}, 500);
```

코드 양의 차이도 크지만, 진짜 차이는 *불가능했던 것이 가능해졌다*는 점이다. `<input>`의 커서, 한글 조합 입력, 스크린 리더 접근성 - 이제 3D 씬 안에서도 공짜로 따라온다.

-----

## 6. 현재 한계와 주의사항

> ⚠️ **실험적 기능 경고:** HTML in Canvas는 아직 WICG 인큐베이션 단계다. Chrome Canary 외 브라우저에서는 동작하지 않으며, API 설계가 언제든 변경될 수 있다. 프로덕션 사용은 불가하다.

### 보안: Canvas Taint 정책

일반적으로 외부 리소스를 Canvas에 그리면 “tainted” 상태가 되어 `toDataURL()`이나 `getImageData()`가 차단된다. HTML in Canvas는 이 제약을 우회하는 대신, 브라우저가 민감한 정보(맞춤법 교정 마커, 자동완성 내용 등)를 렌더링에서 제외하는 미티게이션을 적용한다. 제안서는 PII 노출에 극도의 주의를 당부한다.

### 인터랙티브 요소의 한계

Canvas에 그려진 `<button>`이나 `<a>`는 시각적으로는 보이지만 자동으로 클릭이 동작하지 않는다. `setHitTestRegions()` API로 클릭 영역을 수동 등록해야 한다. 이 부분은 아직 설계 진행 중이다.

### CSS Transform 무시

소스 요소에 적용된 CSS transform은 Canvas에 그릴 때 무시된다. Canvas의 현재 변환 행렬(CTM)만 적용된다.

### Firefox, Safari 미지원

Mozilla는 표준 포지션 검토 중(이슈 #1076)이고, WebKit 역시 별도 포지션 이슈에서 논의 중이다. 멀티브라우저 지원까지는 상당한 시간이 걸릴 것으로 보인다.

### 성능: paint 루프 비용

`paint` 이벤트 내에서 DOM을 수정하면 다음 프레임에 다시 paint가 발생할 수 있다. ResizeObserver와 유사하게 무한 루프를 피하기 위한 설계가 필요하다. 브라우저는 프레임당 Paint 단계를 한 번만 실행하도록 최적화하지만, 핸들러 내부에서 DOM을 건드리는 것은 여전히 주의가 필요하다.

-----

## 7. 전망: 웹의 다음 단계

HTML in Canvas는 하나의 API 제안이 아니다. 웹이 처음 설계될 때부터 존재했던 근본적인 이원성 - 문서 모델과 그래픽 모델의 분리 - 를 처음으로 통합하는 시도다.

WebGPU와의 통합(현재 `copyElementImageToTexture`로 논의 중), WebXR과의 연동, OffscreenCanvas를 통한 Worker 스레드 활용 - 이런 확장 가능성들이 이미 제안서에 언급되어 있다.

실용적인 관점에서 가장 먼저 영향받을 영역은 데이터 시각화다. 차트 라이브러리들은 오랫동안 Canvas 성능과 HTML 접근성 사이에서 타협해왔다. HTML in Canvas가 표준화되면 이 타협이 사라진다. 게임 개발, 크리에이티브 도구, XR 경험 - 모두 같은 방향이다.

지금 당장 프로덕션에서 쓸 수 없는 기술이지만, 방향은 명확하다. DOM과 GPU 사이의 벽은 무너지고 있다. 

> The API is most compelling not when it enables new categories of effect, but when it makes something everyone’s seen a thousand times feel dramatically better. The web has always been flat. This API lets it have depth. but only when that depth serves a purpose.
> 
> “웹은 언제나 평평했다. 이 API는 그것에 깊이를 준다. 단, 그 깊이가 목적을 달성하고자 할 때만.”
> - Matt Rothenberg

로텐버그의 글이 인상깊어서 인용해본다. API가 실제 UX 문제를 더 잘 푸는 데에 도움을 주고 있고, 기술을 위한 기술이 아닌 목적있는 기술이 중요하다는 언급으로 느껴진다. 웹의 무궁무진한 기술, 실험과 브라우저의 신기능들이 어떤 문제를 풀려고 하는지 생각해보는 것의 중요함을 또 느끼며 글 마무리. 


-----

**참고 자료**

- [WICG/html-in-canvas](https://github.com/WICG/html-in-canvas) — 제안서 원문
- [WICG/canvas-place-element](https://github.com/WICG/canvas-place-element) — 관련 제안
- [Matt Rothenberg — HTML in Canvas](https://mattrothenberg.com/notes/html-in-canvas/) — 데모 분석
- Chrome Canary 플래그: `chrome://flags/#canvas-draw-element`
- [Mozilla Standards Position Issue #1076](https://github.com/mozilla/standards-positions/issues/1076)