---
{"dg-publish":true,"permalink":"/posts/spa/","created":"2026-02-08","updated":"2026-02-08"}
---

tldr;
fallback이 없는 path를 빌드 타임에 정적 엔트리로 승격시키는 방법

---

SPA에서 라우팅 문제는 대체로 서버 설정에서 시작된다. BrowserRouter를 쓰는 순간, URL은 더 이상 프론트엔드만의 것이 아니게 된다. 브라우저는 새로고침 시 해당 경로를 서버에 그대로 요청하고, 서버가 그 경로를 이해하지 못하면 404를 반환한다. 이건 React Router의 문제가 아니라, 웹이 작동하는 방식 그 자체다.

문제는 서버 설정을 바꿀 수 없는 환경에서 이 일이 벌어질 때다.

내가 마주한 상황은 이랬다.

이미 /subpath는 서버에서 SPA fallback이 설정되어 있었다. 그래서 /subpath/foo 같은 경로도 문제없이 index.html로 떨어지고, 그 안에서 React Router가 다시 라우팅을 이어받았다. 하지만 새로운 진입점으로 /subpath2를 추가하려는 순간 문제가 생겼다. /subpath2는 fallback 대상이 아니었고, 서버는 이 경로를 정적 파일로만 해석했다. 결과는 당연히 404였다.

서버 설정은 손댈 수 없었다. 그렇다면 선택지는 하나뿐이다. 서버가 이해할 수 있는 형태로 프론트엔드가 먼저 맞춰주는 것. 라우트를 추가하지 않고, 엔트리를 추가하기.

처음엔 아래와 같은 해결책들이 떠올랐다.

HashRouter를 쓰거나, URL을 바꾸거나, 인프라팀에 nginx 설정을 요청하거나.

하지만 이 경우 /subpath는 이미 BrowserRouter로 잘 동작하고 있었고, /subpath2만 예외였다. 문제는 “라우팅 방식”이 아니라 “진입점의 존재 여부”였다.

서버의 관점에서 /subpath2는 존재하지 않는 디렉토리였다.

그렇다면 답은 단순했다.

/subpath2를 라우트가 아니라 실제로 존재하는 정적 엔트리로 만들어버리자.

이 순간부터 생각의 방향이 바뀐다.

클라이언트 라우트를 늘리는 게 아니라, 빌드 결과물의 구조를 바꾸는 문제가 된다.

---

Vite에서 정적 엔트리를 만드는 방법

Vite는 SPA 빌드 도구처럼 보이지만, 내부적으로는 Rollup을 사용한다. 그리고 Rollup은 애초에 멀티 엔트리 빌드를 지원한다. 이걸 활용하면 하나의 앱을 여러 개의 index.html 진입점으로 빌드할 수 있다.

```ts
// vite.config.ts
import { defineConfig } from "vite";
import react from "@vitejs/plugin-react";
import { resolve } from "path";

export default defineConfig({
  plugins: [react()],
  build: {
    rollupOptions: {
      input: {
        subpath: resolve(__dirname, "subpath/index.html"),
        subpath2: resolve(__dirname, "subpath2/index.html"),
      },
    },
  },
});
```

이 설정을 이렇게 빌드 결과물을 만든다.

```
dist/
 ├─ subpath/
 │   └─ index.html
 └─ subpath2/
     └─ index.html
```

이제 서버는 /subpath2를 더 이상 “없는 라우트”로 보지 않는다. 그저 존재하는 디렉토리로 인식하고, 그 안의 index.html을 반환해주게 된다.

이렇게 서버 설정은 그대로 두면서, 404 에러를 없앨 수 있다.

 중요한 점은, 이 방식이 BrowserRouter를 포기하지 않는다는 것이다. (위에서 언급했던 HashRouter (못생김)을 쓰지 않을 수 있다.)

각 엔트리 안에서는 여전히 평범한 SPA다.

/subpath2/detail로 새로고침을 해도 서버는 /subpath2/index.html까지만 책임진다. 그 이후의 경로 해석은 전부 클라이언트의 몫이고, 서버는 SPA 라우트를 알 필요가 없다.

이 구조는 순수 SPA도 아니고, 전통적인 MPA도 아니다. 각 path는 독립적인 정적 진입점을 가지지만, 그 안에서는 SPA처럼 동작한다.

뭔가 혼종 같은 형태이지만, MPA 같은 SPA, 정적 엔트리 기반 라우팅이라고 부를 수 있겠다. 

겉으로 보면 이건 우회처럼 보일 수 있다. 하지만 실제로는 서버의 제약을 인정하고, 그 제약 안에서 가능한 현실적인 방식 - fallback이 없는 path를 빌드 타임에 정적 엔트리로 승격시킨 구조- 을 이용한 설계이기도 하다.