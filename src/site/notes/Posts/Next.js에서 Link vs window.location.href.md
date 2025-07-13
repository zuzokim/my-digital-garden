---
{"dg-publish":true,"permalink":"/posts/next-js-link-vs-window-location-href/","tags":["Nextjs"],"created":"2025-07-13","updated":"2025-07-13T22:35:00"}
---

Next.js에서 페이지를 이동할 때 <Link /> 컴포넌트를 쓸지, window.location.href를 사용할지 고민한 적이 있다면, 둘의 차이를 단순히 “부드러운 이동 vs 전체 새로고침” 혹은 “내부앱 이동 vs 외부링크이동” 정도로만 이해하고 있을 수 있다. (그게 나였다..) 하지만 실제로는 Next.js의 렌더링 전략과 맞물린 중요한 차이가 있다.

(1) <Link /> 는 클라이언트 사이드 라우팅

Next.js의 < Link href="/about" >는 클라이언트 사이드 라우팅(Client-Side Routing, CSR)을 수행한다. 즉, 브라우저 전체를 새로고침하지 않고 현재 실행 중인 앱 안에서 URL만 바꾸고 필요한 데이터와 컴포넌트만 비동기로 가져와 렌더링한다. 덕분에 빠르고 매끄러운 사용자 경험이 가능하다.

예를 들어, 다음과 같은 구조를 가진 앱에서:

```ts
/app
  /page.tsx
  /about
    /page.tsx
```

홈 페이지에서 아래처럼 <Link />를 사용할 경우:

```ts
// /app/page.tsx
import Link from 'next/link'

export default function HomePage() {
  return (
    <main>
      <h1>Home</h1>
      <Link href="/about">Go to About</Link>
    </main>
  )
}
```

<Link />를 클릭하면 브라우저는 새로고침 없이 URL만 변경하고, Next.js는 내부 라우터를 통해 /about/page.tsx를 비동기로 import() 한다. 만약 해당 페이지가 서버 컴포넌트라면, Next.js는 서버에서 해당 컴포넌트를 JSON 형태로 스트리밍 받아 React 앱 트리 안에 삽입한다.

(2) window.location.href 는 전체 새로고침

반면, window.location.href = "/about"은 브라우저가 전체 페이지를 다시 요청한다. React 상태는 초기화되고, 서버에서 새롭게 HTML과 JavaScript를 모두 받아야 하므로 사용자 경험이 끊기고 느려진다.

(3) 그럼 Server Component와는 어떤 관계가 있을까?


Next.js 13부터 기본적으로 Server Components를 사용해 페이지를 렌더링한다. 이는 각 컴포넌트를 서버에서 렌더링한 후 클라이언트로 스트리밍하는 방식이다. 하지만 CSR과 Server Component는 서로 다른 층위의 개념이다:

- Server Component는 “페이지를 어떻게/어디서 렌더링할 것인가”에 대한 전략이고,
- CSR은 “페이지를 어떻게 전환할 것인가”에 대한 방식이다.

즉, <Link />로 이동해도 결국 렌더링되는 페이지는 Server Component 기반일 수 있다. 중요한 점은 <Link>를 쓰면 Next.js가 서버에서 필요한 데이터만 요청해서 부드럽게 전환하고, window.location.href를 쓰면 전체 문서를 다시 받아야 하므로 Next.js의 최적화 이점을 잃게 된다.