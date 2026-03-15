---
{"dg-publish":true,"permalink":"/posts/pdf-exporter/","created":"2026-03-15","updated":"2026-03-15"}
---

화면에 있는 컴포넌트를 이미지로 추출하고 PDF로 합쳐주는 익스텐션을 개발하면서 배운 내용을 정리한다. 

실제 크롬 익스텐션을 직접 개발해 본 것은 처음이었는데, 역시 꼭 필요한 순간에 기술을 활용하고, 어떻게든 그것이 가능하도록 하면서 알게되고 배우는 것들이 있다. Wishful thinking으로 원하는 그림을 집요하게 추구하다보면 그것을 구현하는 방법과 구조를 탄탄하게 만들게 되면서 기억에도 오래 남는다.

---

### 목차

1.	처음엔 단순해 보였다
2.	Popup의 한계 — 창을 닫으면 끝
3.	Background — 영원한 중개자
4.	Offscreen — DOM이 필요한 순간
5.	세 레이어가 서로를 필요로 하는 이유
6.	전체 흐름 한눈에

### 1. 처음엔 단순해 보였다

만들고 싶은 건 간단했다. 웹 페이지의 특정 컴포넌트들을 이미지로 추출해서 PDF 한 파일로 합쳐 저장해주는 크롬 익스텐션이었다. 툴바 아이콘 클릭 → 버튼 하나 → PDF 다운로드. 별로 복잡하지 않을 것 같았다.

그런데 막상 만들기 시작하자 문제가 연달아 터졌다.

∙	문제 1. 이미지 추출 API를 호출하면 CORS 오류가 난다.
∙	문제 2. PDF 변환 라이브러리(jsPDF)가 canvas와 Image 객체를 써야 한다.
∙	문제 3. 이미지가 10장이면 30초가 넘는데, 그 사이 팝업을 닫으면 작업이 중단된다.

각 문제를 해결하다 보니, Manifest V3가 제시하는 3중 레이어 구조가 단순한 관례가 아니라 이 문제들에 대한 직접적인 답이라는 걸 알게 됐다.

### 2. Popup의 한계 — 창을 닫으면 작업이 사라진다

처음에는 모든 로직을 popup.js 하나에 넣으려 했다. 버튼 클릭 이벤트를 받고, API를 호출하고, PDF를 만들고, 다운로드까지. 가장 직관적인 방법이다. 하지만 popup은 사용자가 창을 열어두는 동안에만 존재한다. 다른 탭을 클릭하거나 팝업 바깥을 누르면 팝업 창이 닫히고, 실행 중이던 JavaScript도 그 즉시 종료된다. 이미지를 5장 받아오던 중이었다면, 작업은 사라져버린다.
다시 말하면, Popup의 역할은 UI 레이어로 한정된다. 사용자로부터 입력을 받고, 진행 상황을 보여주고, 완료를 알려주는 것. 사용자가 “시작” 버튼을 누르면, popup은 background에 메시지를 던지고 UI 업데이트만 기다린다.

```js
// popup.js — 버튼 클릭 시 background에 위임
exportBtn.addEventListener("click", async () => {
  const res = await chrome.runtime.sendMessage({
    type: "START_EXPORT",
    components, // 추출할 컴포넌트 목록
  });
});

// 진행 상황은 메시지로 수신
chrome.runtime.onMessage.addListener((msg) => {
  if (msg.type === "EXPORT_PROGRESS") showLoading(msg.message);
  if (msg.type === "EXPORT_COMPLETE") showResult(msg.message);
});

```

### 3. Background — 숨은 중개자

Background 스크립트(Manifest V3에서는 서비스 워커)는 팝업과 달리 브라우저가 종료되지 않는 한 메시지가 오면 언제든 깨어난다. 익스텐션 전체에서 유일하게 “항상 연락 가능한” 레이어다.
그래서 background에는 두 가지 책임을 맡겼다. CORS 없이 외부 API 호출 익스텐션의 manifest.json에 host_permissions를 선언하면, background 서비스 워커는 해당 도메인에 CORS 제약 없이 fetch를 날릴 수 있다. Popup이나 Offscreen은 이 권한이 없으므로, 외부 API가 필요한 모든 요청은 background를 경유한다.

```js
// background.js — fetch 프록시
if (msg.type === "API_FETCH") {
  fetch(msg.url, {
    method: msg.method,
    headers: msg.headers,
    body: msg.body,
  })
    .then(async (res) => {
      const text = await res.text();
      sendResponse({ ok: res.ok, status: res.status, text });
    });
  return true; // 비동기 응답을 위해 반드시 필요
}
```

return true 한 줄이 중요하다. Chrome 메시지 리스너에서 비동기 응답을 보내려면 반드시 return true를 해야 한다. 없으면 sendResponse가 호출되기 전에 채널이 닫혀버린다.

##### 진행 상태 보존 — 팝업이 닫혔다 열려도
작업이 진행 중일 때 사용자가 팝업을 닫았다가 다시 열 수 있다. 팝업은 다시 생성되는 시점에 이전 상태를 전혀 모른다. Background가 상태를 들고 있다가 팝업이 GET_EXPORT_STATUS를 물어보면 응답해주는 방식으로 이 문제를 해결했다.

```js
// background.js
let exportState = { status: "idle" };
// "idle" | "in_progress" | "complete" | "error"

if (msg.type === "GET_EXPORT_STATUS") {
  sendResponse(exportState); // 팝업 재오픈 시 복원
  return;
}
```

하지만 background도 한 가지를 할 수 없다. DOM이 없다. document도, canvas도, Image 객체도 없다. 그래서 jsPDF 같은 DOM 의존 라이브러리를 background에서 실행할 수 없다. 이게 Offscreen이 필요한 이유다.

### 4. Offscreen — DOM이 필요한 순간
Offscreen 문서는 Manifest V3에서 새로 도입된 개념이다. 화면에는 보이지 않지만 완전한 HTML 환경을 가진 숨겨진 페이지다. Background가 생성하고 종료를 관리하며, DOM이 필요한 모든 작업을 여기서 수행한다.

```js
// background.js — offscreen 생성
await chrome.offscreen.createDocument({
  url: "offscreen.html",
  reasons: ["BLOBS"],
  justification: "PNG to PDF using jsPDF",
});
```

##### 핸드셰이크 — 준비됐을 때 알려줘
Background가 Offscreen을 생성했다고 해서 바로 메시지를 보내면 안 된다. Offscreen 스크립트가 아직 로드 중일 수 있기 때문이다. 그래서 Offscreen은 로드 완료 시점에 OFFSCREEN_READY를 전송하고, Background는 그 신호를 받을 때까지 Promise로 대기한다.

```js
// offscreen.js — 로드 완료 알림
chrome.runtime.sendMessage({ type: "OFFSCREEN_READY" });

// background.js — 준비 대기
const readyPromise = new Promise((resolve, reject) => {
  offscreenReadyResolve = resolve;
  setTimeout(() => reject(new Error("시간 초과")), 5000);
});
await readyPromise; // 이후에 메시지 전달
```

##### PNG → JPEG 변환 (Canvas 사용)
서버에서 받은 PNG를 그대로 PDF에 넣으면 파일이 커진다. JPEG로 변환하면 용량이 크게 줄어드는데, 이 작업에는 `<canvas />`가 필요하다. Background에서는 불가능하고, Offscreen에서만 할 수 있다.

왜 그런지 이해하려면 이미지 포맷의 차이부터 짚고 넘어가야 한다.

##### 이미지 포맷 - 래스터 vs 벡터
이미지 포맷은 크게 두 종류로 나뉜다. 

래스터(Raster) 는 픽셀의 격자로 이미지를 저장한다. PNG, JPEG, WebP가 대표적이다. 픽셀 하나하나의 색상값을 기록하기 때문에 해상도가 고정되어 있고, 크게 확대하면 계단 현상이 생긴다. 사진처럼 색상이 복잡하고 경계가 부드러운 이미지에 적합하다.
벡터(Vector) 는 좌표와 수식으로 이미지를 저장한다. SVG, PDF의 벡터 레이어가 대표적이다. “이 좌표에서 저 좌표까지 이 색으로 곡선을 그려라”는 명령어의 집합이기 때문에 어떤 크기로 출력해도 선명하다. 아이콘, 로고, 도형처럼 명확한 경계를 가진 그래픽에 적합하다.
이 익스텐션에서 추출하는 대상은 웹 컴포넌트다. 디자인 툴이 DOM을 캡처한 결과물이므로, 텍스트·도형·그림자·그라디언트가 모두 섞인 복합 이미지다. 이를 서버에서 래스터 이미지(PNG)로 렌더링해 전달한다.

##### PNG와 JPEG의 차이
둘 다 래스터 포맷이지만 압축 방식이 다르다.
PNG 는 무손실(lossless) 압축을 사용한다. 원본 픽셀 데이터를 그대로 복원할 수 있고, 투명도(알파 채널)를 지원한다. 그 대신 파일이 크다. 1920×1080 컴포넌트 스크린샷 한 장이 3–5MB를 넘는 경우가 흔하다.
JPEG 는 손실(lossy) 압축을 사용한다. 사람 눈에 잘 보이지 않는 고주파 색상 변화를 버리는 방식(DCT 변환)으로 용량을 줄인다. 투명도는 지원하지 않지만 품질 계수(0–1)를 조절하면 같은 이미지를 PNG 대비 5–10배 작게 만들 수 있다. 0.92 수준에서는 육안으로 차이를 구별하기 어렵다.

그렇다면 PNG가 더 좋은 것 아닌가? UI 스크린샷처럼 선명한 경계가 많은 이미지는 이론적으로 PNG가 더 적합하다. 하지만 이 익스텐션의 목적은 “화면을 문서화한 PDF”이지 “픽셀 단위의 정밀 재현”이 아니다. 품질 손실이 거의 없는 0.92 수준에서 용량을 크게 줄일 수 있다면 그 tradeoff는 충분히 합리적이다.

##### PDF 안에서 이미지는 어떻게 저장되는가
PDF는 컨테이너 포맷이다. 텍스트, 벡터 그래픽, 래스터 이미지를 모두 담을 수 있는 봉투다. 벡터 그래픽(경로, 텍스트)은 좌표 명령어로 저장되어 아무리 확대해도 선명하다. 반면 래스터 이미지는 PDF 안에서도 래스터 그대로 저장된다. 해상도가 고정되고 크기가 그대로 반영된다.
jsPDF의 addImage()는 래스터 이미지를 PDF 페이지에 삽입하는 방식이다. PNG를 그대로 넣으면 원본 파일 크기가 PDF 용량에 직접 더해진다. 페이지가 10장이면 그 합산이 그대로 파일 크기가 된다. JPEG로 먼저 변환하고 넣으면 각 페이지 이미지가 훨씬 작아지고, 결과 PDF 용량도 비례해서 줄어든다. 

##### Canvas가 변환 다리 역할을 하는 이유
브라우저에서 PNG를 JPEG로 변환할 때 직접적인 API는 없다. `<canvas>`  를 경유하는 것이 표준적인 방법이다.

1.	<img> 태그로 PNG를 로드해 비트맵으로 디코딩한다.
2.	그 비트맵을  `<canvas>` 위에 그린다. 이 시점에 픽셀 데이터가 메모리에 올라온다.
3.	canvas.toDataURL("image/jpeg", 0.92)로 JPEG 인코딩을 수행해 Data URL로 출력한다.

이 과정에서 한 가지 주의할 점이 있다. JPEG는 투명도를 지원하지 않기 때문에, PNG의 알파 채널(투명 영역)이 JPEG로 변환되면 검은색으로 처리된다. 이를 막기 위해 canvas에 먼저 흰 배경을 깔고(ctx.fillStyle = "#fff"), 그 위에 이미지를 올린다. 코드에서 fillRect가 그 역할을 한다.


```js
// offscreen.js — canvas로 PNG → JPEG 변환
function toJpeg(pngDataUrl) {
  return new Promise((resolve) => {
    const img = new Image(); // DOM 필요
    img.onload = () => {
      const canvas = document.createElement("canvas"); // DOM 필요
      const ctx = canvas.getContext("2d");
      ctx.fillStyle = "#fff";
      ctx.fillRect(0, 0, img.width, img.height);
      ctx.drawImage(img, 0, 0);
      resolve(canvas.toDataURL("image/jpeg", 0.92));
    };
    img.src = pngDataUrl;
  });
}
```

파일 다운로드도 Offscreen에서 PDF Blob을 만든 다음 `<a>`  태그 클릭으로 다운로드를 트리거하는 것도 DOM이 필요한 작업이다. Background에서는 할 수 없다. Offscreen이 직접 처리한다.

```js
// offscreen.js — Blob URL로 다운로드
const blobUrl = URL.createObjectURL(pdfBlob);
const a = document.createElement("a");
a.href = blobUrl;
a.download = "export.pdf";
a.click();
```

### 5. 세 레이어가 서로를 필요로 하는 이유

세 레이어는 서로의 약점을 채워준다.

∙	Popup은 UI는 잘 만들지만 오래 살지 못한다.
∙	Background는 오래 살고 네트워크도 다루지만 DOM이 없다.
∙	Offscreen은 DOM이 있지만 네트워크 권한이 제한된다.

그래서 이 셋이 협력해야만 “API 호출 → 이미지 처리 → PDF 저장”이라는 전체 흐름이 완성된다.

### 6. 전체 흐름 한눈에 보기

```
[Popup]
  사용자가 버튼 클릭
  → START_EXPORT 메시지 전송
        ↓
[Background]
  Offscreen 문서 생성
  → OFFSCREEN_READY 대기
        ↓
[Offscreen]
  각 컴포넌트에 대해 API_FETCH 요청
        ↓
[Background]
  CORS 없이 이미지 추출 API 호출
  → 응답 반환
        ↓
[Offscreen]
  이미지 다운로드 (서버 준비 완료까지 폴링)
  Canvas로 PNG → JPEG 변환
  jsPDF로 PDF 합치기
  <a> 태그로 파일 다운로드
  → EXPORT_COMPLETE 전송
        ↓
[Background]
  상태 갱신 → Popup에 포워딩
        ↓
[Popup]
  완료 메시지 표시
```

#### 단계별 요약:

1.	사용자가 버튼 클릭 Popup — 추출할 컴포넌트 목록과 함께 START_EXPORT 메시지를 Background로 보낸다. 이후 Popup은 UI 업데이트만 기다린다.
2.	Offscreen 문서 생성 Background — offscreen.html을 생성하고, OFFSCREEN_READY 신호를 기다린다. 준비 완료 신호가 오면 작업을 전달한다.
3.	이미지 추출 API 순차 호출 Offscreen → Background — Offscreen이 각 컴포넌트에 대해 API 요청을 보낸다. 직접 fetch할 수 없으므로 API_FETCH 메시지로 Background에 프록시를 요청한다.
4.	이미지 다운로드 (재시도 포함) Offscreen — 이미지 생성이 서버에서 비동기로 처리되므로 파일이 준비될 때까지 폴링한다. 이 대기 시간이 수십 초에 달할 수 있어, Popup이 아닌 Offscreen에서 처리해야 한다.
5.	PNG → JPEG → PDF 변환 Offscreen — Canvas로 PNG를 JPEG로 변환한 뒤 jsPDF로 PDF를 합친다. DOM이 있는 Offscreen에서만 가능한 작업이다.
6.	완료 알림 전파 Offscreen → Background → Popup — Offscreen이 EXPORT_COMPLETE를 전송 → Background가 상태를 갱신하고 Popup에 포워딩 → Popup이 완료 메시지를 표시한다.

### 정리

처음에는 “왜 파일을 세 개씩이나 만들어야 하지?“ 라는 의문이 있었다. 그런데 CORS 오류를 만나고, 팝업이 닫히면서 작업이 날아가는 걸 경험하고, canvas가 없어서 이미지 변환을 못 하는 상황에 부딪히다 보니 — 이 세 레이어가 서로의 약점을 정확히 메우고 있다는 게 보이기 시작했다.

Manifest V3의 구조가 처음엔 번거롭게 느껴지지만, 각 제약이 생긴 이유를 이해하고 나면 오히려 역할 분리가 명확해서 유지보수하기 훨씬 쉬운 코드가 된다. 익스텐션이 복잡해질수록 이 분리가 빛을 발한다.

그리고 무엇보다, 수동으로 한 장 한 장 캡쳐 후 PDF 병합을 하던 번거로운 수동 작업을 버튼 클릭 한 번으로 자동화해서 업무 생산성을 올릴 수 있게 되었다. "한번에 그냥 되면 좋곘다"는 바람이 실제로 기술로 이루어질 때의 즐거움을 오랜만에 또 만난 작업이었다. 