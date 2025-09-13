---
{"dg-publish":true,"permalink":"/posts/nextjs-script/","tags":["Browser","Nextjs"],"created":"2025-05-13","updated":"2025-05-13T22:33:00"}
---

1년 정도 묵혀뒀던 디버깅 로그를 작성해본다. 
너무 정신 없기도 했고, 당시에 어찌저찌 디버깅에 성공해 문제를 해결했었는데, 사실 정확히 이해가 안되고 일단 해결이면 오케입니다 상황이기도 해서 나중에 더 깊게 이해해야지 하다보니 로그 작성이 늦어졌다. 근데 그게 1년이나 걸리다니... 실제로 디버깅 날짜를 짚어보니 한 8개월 정도 됐다. 휴~

**배경**
화이트보드 기능을 제공하는 excalidraw 오픈소스를 셀프호스트해서 앱 내 커스터마이징 컴포넌트로 기능 개발을 한 적이 있다. 셀프호스팅을 하는 경우 excalidraw는 해당 컴포넌트에서 사용되는 정적 에셋(대표적으로 폰트)도 직접 관리를 해주도록 가이드 하고 있다. 
- 폰트 셀프호스팅 가이드 :  https://docs.excalidraw.com/docs/@excalidraw/excalidraw/installation#self-hosting-fonts 

당시 **문제상황**은 이랬다. 
위 개발 가이드를 따라 에셋 폴더를 개발하는 Next 프로젝트 public 디렉토리에 복사 후 경로를 지정해주었다. 로컬에서는 문제없이 동작하던 화이트보드 기능이 최종 Next 앱으로 배포한 뒤 브라우저 런타임에서만 엉뚱한 경로로 라우팅되어 에셋 로드가 되지 않아 제대로 동작하지 않았다. 로컬에서 이것저것 UI와 유틸 기능들을 커스텀하면서 동작확인을 했는데 배포만 하면 깨져버리니 도무지 무엇이 문제인지 알 수가 없었다.

![Screenshot 2025-05-13 at 10.47.22 PM.png](/img/user/Screenshot%202025-05-13%20at%2010.47.22%20PM.png)

Next 는 에러를 맨날 이런식으로 준다. <font color="#d83931">ChunkLoadError 404</font>... ㅠㅠ 어디가 문제인지 참 디버깅하기가 어렵다.

원래는 문제없이 동작하던 기능이라 React 컴포넌트 내 코드 변경점을 찾다가 해당 화이트보드 컴포넌트를 dynamic import하니 문제가 해결된 것처럼 보였다. 그런데 문득 드는 생각이 '이 컴포넌트가 언제라도 dynamic import되지 않으면 또 동작에 문제가 생길텐데, 주석을 남겨놔야 하나?', '이 곳 말고 다른 곳에서 또 컴포넌트가 재활용됐을 때 항상 dynamic 컴포넌트로 작성해야한다고 강제할 수는 없을텐데 어떡하지?' 싶은 생각이 들었다. 그리고 Next 앱으로 배포하는 경우에 생기는 문제라서 좀 더 근본적인 원인을 알아내 고쳐야 했다.

시간이 많이 지나 해결법을 찾아내는 데 어떤 디버깅 과정이 있었는지 세세하게 다 적지는 못하지만, 개발자 도구를 켜서 파악해보니 결론적으로는 Next 앱 실행시 에셋 경로를 찾으려고 하면 경로가 undefined 인게 문제 원인이었다.

기존에는 Next 앱에서 이렇게 next/script의 Script 컴포넌트로 에셋 경로를 설정하고 있었다.

```js
<Script strategy="beforeInteractive" id="excalidraw" >{`window.EXCALIDRAW_ASSET_PATH = '/excalidraw/';`}
</Script>
```

next/script의 Script 컴포넌트 : https://nextjs.org/docs/app/guides/scripts

Next에서 제공해주는 Script는 서트파티 스크립트를 효율적으로 로드하게 해준다. 
- 하나의 layout을 공유하는 여러 라우트 경로에서 서드파티 스크립트를 컴포넌트 형태로 선언적으로 작성해 관리를 쉽게 해주고,
- Strategy 를 prop으로 선택해 로드 전략을 결정할 수 있게 해주고,
- 실험적이지만 Web Worker에서 스크립트를 로드할 수도 있게 해주고,
- 로딩 상태를 처리하기 위해 이벤트 핸들러를 추가 제공해준다.

그 중에서 서드파티 스크립트 excalidraw를 로드하고, `strategy="beforeInteractive"`를 주어서 페이지가 인터랙티브 상태가 되기 전에 에셋을 로드하게 했었다. 그럼 내 생각으로는 페이지 초기 HTML과 함께 서버에서 스크립트가 가장 먼저 실행되기 때문에 에셋 경로가 window 전역 객체에 변수 설정되고, 그 후에 화이트보드 컴포넌트가 해당 에셋 경로를 찾아가므로 문제가 없지 않나? 싶었다. ~~근데 안됨!~~

자 그러면 왜 안되는지를 따져보자.

`strategy="beforeInteractive"`는 Next의 `<Script>` 실행 시점을 최대한 빨리 보장하지만, Excalidraw가 import되기 전에 해당 변수가 설정된다는 걸 완전히 보장하지는 않는다.

`strategy="beforeInteractive"`는 Next에서 다음 순서로 동작:
1. HTML 문서가 서버에서 렌더링됨
2. `<Script strategy="beforeInteractive">`는 Next의 자체 스크립트 로더를 통해`<head>`에 삽입되어 가능한 한 빨리 실행됨
3. React 앱이 hydration되고, 다른 JS 번들이 로딩되기 시작

**잠깐 알아보자**

일반적인 `<script>` 태그는 브라우저가 HTML을 파싱하면서 동기적으로 실행한다.
(브라우저가 HTML을 읽는 도중 `<script>` 태그를 만나면 즉시 실행된다. 뒤에 나오는 js 코드보다 먼저 실행되도록 보장되며 이때 HTML 파싱은 차단된다.)

```html
<head>
  <script>
    window.EXCALIDRAW_ASSET_PATH = '/excalidraw/';
  </script>
</head>
```

그런데 Next는 스크립트를 최적화하기 위해 직접 `<script>` 태그를 쓰지 않고 자체적인 Next 전용 로딩 매커니즘을 통해 로드한다고 한다. 
(Next 내부 로더 (`next/script`)가 hydrate 시점에 실행을 컨트롤)

```html
<script id="__NEXT_DATA__">...</script>
<script>
  // next-script-loader가 동작하면서 script를 삽입
</script>
```


즉:
- `beforeInteractive`는 Next 자체 스크립트 로더 기준으로 “최대한 빨리 실행되도록 노력하는 것”, "Next 내부 번들 기준으로의 상대적인 선 실행"이지,
- 순수 `<script>` 태그처럼 브라우저가 동기적으로 실행하는 것과는 다르다.

그래서 실제로는 race condition 타이밍 이슈가 발생할 수 있다:
- excalidraw 컴포넌트가 동적으로 import되거나 lazy load 되는 경우
- 또는 내부적으로 코드 스플리팅된 번들이 `window.EXCALIDRAW_ASSET_PATH`를 참조할 때 (JS 번들(chunk)이 나눠져 있거나, dynamic import 등으로 로딩 시점이 뒤섞이면 정확히 먼저 실행된다는 보장은 없음)
- 이 시점에 아직 해당 전역 변수가 정확하게 셋업되지 않았다면, excalidraw는 fallback 경로(excalidraw.com)를 쓰거나 잘못된 경로로 chunk를 요청 → **404 ChunkLoadError** 를 뱉는다.


```js
const Excalidraw = dynamic(() => import('@my-app/excalidraw'), { ssr: false });

```
이렇게 dynamic import로 컴포넌트를 렌더하면 해결된 것처럼 보였던 것은 최소한 브라우저에서 `window.EXCALIDRAW_ASSET_PATH` 가 세팅 된 이후에 클라이언트 컴포넌트로 불러와졌기 때문이고, 여전히 완벽하게 보장이 되지는 않는다.

- excalidraw 내부에서 `window.EXCALIDRAW_ASSET_PATH`를 참조했을 때
- Next가 아직 해당 스크립트를 실행하기 전이라면 `undefined`

| -               | 순수 `<script>`    | `<Script strategy="beforeInteractive">` |
| --------------- | ---------------- | --------------------------------------- |
| 실행 시점           | HTML 파싱 도중 즉시 실행 | Next가 hydration 전 "가능한 빨리" 실행           |
| 제어 방식           | 브라우저 기본 동작       | Next의 내부 스크립트 로더                        |
| React보다 먼저?     | ✅ 항상 먼저          | ✅ 보통은 먼저, ❌ 항상 보장은 안 됨                  |
| 에셋 로딩 경로에 안전한가? | ✅                | ⚠️ chunk 분할 + dynamic import 시 위험 가능    |


**해결한 방법**

결론만 먼저,
에셋 경로가 항상 먼저 로드 되도록 확실하게 보장하려면 Next 빌드 타임에 먼저 세팅되도록 하면 된다.

```js
//next.config.js

const webpack = require('webpack');
...
  webpack: (config) => {
    config.plugins.push(
      new webpack.DefinePlugin({
        ['window.EXCALIDRAW_ASSET_PATH']: '/excalidraw/',
      }),
    );
    return config;
  },
...

```

이렇게 `DefinePlugin` 을 활용해서 빌드타임 문자열 상수로 치환하면 타이밍 문제를 해소할 수 있게 된다. excalidraw 컴포넌트를 사용하려면 에셋이 무조건 있어야 하므로 이런 식으로 브라우저 런타임이 아닌 빌드타임에 딱 chunk 경로 고정을 해두면 확실해지는 것이다. (~~대신 이러면 브라우저 콘솔에서 window.EXCALIDRAW_ASSET_PATH 로 변수 접근은 안된다.~~)

이렇게.. 문제 원인을 알아내고 해결했다. 디버깅 당시에 이해가 잘 안되던 Next Script 동작을 지금 다시 공식 문서와 함께 이해해보니 좀 더 잘 이해가 되는 듯 하다. 왜 그때는 이해가 잘 안됐지 싶은데, 그만큼 내가 8개월동안 이해도가 좀 생겼나 싶어서 뿌듯(나 성장했나..?)하고 앞으로는 로그를 더 자주 작성해야겠다는 마음..

끝!