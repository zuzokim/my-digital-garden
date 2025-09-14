---
{"dg-publish":true,"permalink":"/posts/next-js-prerender-error/","tags":["Nextjs","WebAPI"],"created":"2025-09-14","updated":"2025-09-14"}
---

Next 앱 빌드 시점에 안나던 에러가 발생했다. 

```bash
ReferenceError: document is not defined
```

메세지 상으로는 분명 document나 window 객체 참조 에러였다. 
그런데 전체 검색을 해보아도 직접 document를 참조하는 코드가 없었고, 
혹시 몰라 의심이 가는 컴포넌트들의 최상단에 'use client'로 클라이언트 사이드 렌더를 하도록해보았으나, 실패.

결국엔 해결했는데, 그 이유와 왜 'use client'로는 해결되지 않았는지 기록.

문제가 되는 코드는 코드베이스에 직접 작성한 코드가 아닌, 서드파티 라이브러리를 import하는 부분이었다.
애니메이션 등을 지원하는 클라이언트 사이드에서만 동작하는, 즉 window, document 객체를 직접 참조하는 라이브러리를 import하고 있었기 때문이었다.

그래서 Next.js 앱 빌드 시점에, 그러니까 Node.js 환경에서 번들링 + SSR 준비를 하는 시점에 import된 라이브러리에서 document를 참조하니 발생하는 에러였다.

우선 이렇게 ssr: false 옵션으로 dynamic import를 하게 해서 클라이언트 사이드 렌더링 시점에만 로드하도록 수정해 해결했다.

```ts
import dynamic from "next/dynamic";

const NoSSRComponent = dynamic(() => import("../components/Lottie"), {
  ssr: false,
});
```


### 기록

#### "use client"는 컴포넌트 전체를 무조건 클라이언트에서만 실행시켜주지 않음
- `"use client"`는 **해당 파일을 클라이언트 컴포넌트로 표시**하는 것일 뿐, Next.js가 여전히 **서버에서 초기 빌드/프리렌더**를 시도함.
- 즉, 클라이언트 컴포넌트라도 import 시점(모듈 최상단 코드)은 빌드 서버에서 먼저 실행됨.
- 브라우저 API를 사용하는 외부 라이브러리 import 시점 문제 - `'use client'`로 감싸도 **import는 빌드시 실행되므로 서버에서 터짐**.
```ts
'use client';

// ❌ 모듈 최상단에서 document를 바로 참조 → 빌드 시 터짐
const el = document.querySelector("#root");

export default function Page() {
  return <div>Hello</div>;
}
```

```ts
'use client';
import Lottie from 'lottie-web'; // ❌ lottie-web 내부에서 document를 바로 씀 → 에러
```

#### 바로 알기
- App Router(`app/`)에서는 모든 컴포넌트가 기본적으로 **Server Component**.
- `"use client"`를 붙이면 클라이언트에서 렌더링되지만,
    - **직접적인 렌더링 로직**은 CSR
    - **모듈 import / 초기화 코드**는 여전히 빌드 시 서버에서 실행됨
	    - **빌드 시점 (Node.js 환경에서 번들링 + SSR 준비)**
			- Next.js는 모든 모듈(`.tsx`, `.js`)을 import 해서 번들을 만듦.
			- 이 때 **모듈 최상단 코드(컴포넌트 함수 밖 코드)**는 Node.js 런타임에서 실행됨.
			- 즉, `'use client'`가 있더라도 **import 순간에는 여전히 서버(Node.js)** 환경.
		- **클라이언트 렌더링 시점 (CSR + Hydration)**
			- 브라우저에서 React가 hydration을 하면서 비로소 `useEffect`, `useLayoutEffect`, 이벤트 핸들러 안의 코드가 실행됨.
			- 여기서야 비로소 `document`, `window`, `localStorage` 같은 브라우저 API 사용 가능.