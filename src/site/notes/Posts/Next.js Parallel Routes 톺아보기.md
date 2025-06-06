---
{"dg-publish":true,"permalink":"/posts/next-js-parallel-routes/","tags":["Nextjs"],"created":"2025-06-06","updated":"2025-06-06T18:09:00"}
---

이전 회사에서는 Next.js 14버전을 쓰고 있긴 했지만 여러 기능들과 개념을 적극적으로 활용하지는 않았더랬다. (결과론적인 생각임..) 최근에 개인적으로 Next.js Routes를 활용한 프로젝트도 접해보고, 곧 합류하게 될 회사에서 Next.js 14버전 App Router를 적극 활용하고 있다고 미리 알려주셔서 짬내서 공식문서를 들여다보고 있다.

합류하기 전에 소프트랜딩을 위한 선행학습ㅋㅋ을 원한다고 미리 공부해볼만한 레퍼런스를 알려줄 수 있는지 요청했고, 합류할 프론트엔드 팀에서 흔쾌히 여러가지 키워드를 안내해준 덕에 그동안 정말 원했던 쳐내기 바쁜 일만 하지 않고 공부를 하고 있다. 너무 행복하다ㅠ 혹여 시간적 여유가 되고, 본인이 모범생이라면(?) 이직하는 회사에 이렇게 미리 요청해보는 것도 좋은 것 같다. 모든 회사가 다 온보딩이 스무스하지는 않을테니 대비 차원에서..

---

https://nextjs.org/docs/14/app/building-your-application/routing/parallel-routes

공식 문서를 기준으로 먼저 보자.

### Parallel Routes란?

Next.js 14버전에서 소개된 **병렬 라우팅**은 한 페이지 내에 여러 영역(슬롯)을 **독립적**으로 **동시에** 렌더링하거나, **조건에 따라** 렌더링하도록 설계된 기능이다.

전통적인 라우팅은 하나의 URL에 하나의 페이지를 렌더링하지만,
병렬 라우팅은 폴더구조 컨벤션 아래 `@analytics` `@team` 처럼 여러 슬롯을 구성해 한 화면에 여러 페이지를 배치할 수 있다.

##### 여러 페이지 독립 렌더링
탭, 대시보드, 모달 등의 UI에서 각 영역이 같은 레이아웃 내에서 독립적으로 렌더링, 로딩, 에러, 캐싱을 유지한다.
즉, 로딩이나 에러 상태를 전체 페이지 단위가 아닌 슬롯 개별이 자체적으로 관리하는 구조다.

##### 조건부 렌더링
로그인 상태나 역할에 따라 `@dashboard` 또는 `@login` 슬롯을 조건부로 렌더링할 수 있다.

###### Subnavigation (Active state & Soft navigation)
기본적으로 Next.js는 슬롯마다 활성 상태(혹은 하위 페이지)를 추적한다. 그러나 슬롯 내에서 렌더링되는 콘텐츠는 네비게이션 유형에 따라 달라진다.

네비게이션 유형 : 
- **Soft Navigation** - 클라이언트 사이드 네비게이션에서는 슬롯 내 하위 페이지를 부분적으로 렌더하는데, 이때 현재 URL과 매치가 안되더라도, 기존에 활성화되어있는 하위 페이지를 유지한채로 렌더가 된다.  
- **Hard Navigation** - 브라우저 새로고침을 하거나 URL 직접 접근을 하면 Next는 활성 상태를 기억하지는 못하므로, `default.js` 를 렌더하거나 없다면 fallback으로 404 페이지를 렌더한다.
#### Intercepting Routes
URL을 기반으로 폼이나 로그인 창을 모달처럼 띄우는 UX를 자연스럽게 구현할 수 있다.

##### 코드 스플릿팅 및 재사용
슬롯별 청크로 분리되어 필요한 부분만 HTTP 요청, SSR/클라이언트 스트리밍되어 캐시 및 렌더링 성능을 최적화 할 수 있다.

### App Router 의 layout 과 Parallel Routes

Next.js 13~14버전의 App Router가 이미 각 레이아웃, 중첩된 레이아웃을 부분적으로 렌더링해주는데, Parallel Routes는 왜 필요한가? 하는 궁금증이 생겼다.

기본적으로 App Router도 자체 단일 슬롯으로 보면 된다. 

```ts
// layout.tsx
export default function Layout({ children }: { children: ReactNode }) {
  return <div>{children}</div>;
}
```

이렇게 children 이 단일 자체 슬롯이고, `@children` 으로 폴더를 구성하지 않고도 사용이 가능하다. 다만, 이 경우 하나의 메인 콘텐츠 흐름만 관리할 수 있다. 

게다가 root layout이 URL 경로를 인지하지 못할 때 생기는 에러가 있는데, 예를 들면 아래와 같다.
- `/settings/:id`에서는 좌측 네비게이션을 보여줘야 하는데,
- `/settings/:id/foo`처럼 하위 경로로 이동하면, 루트 레이아웃에서 그걸 감지할 수 없어서 UI가 깨지거나, 예기치 않게 전체가 다시 렌더링되거나, 네비게이션이 사라지는 일이 생긴다.

이유는, 기본적으로 Next.js 컴포넌트는 서버 컴포넌트로 렌더링 되는데, 렌더링시 URL 정보를 자동으로 처리하지 않는다. `usePathname`와 같은 훅을 사용하려면 클라이언트 컴포넌트로 사용해야하고, 혹은 미들웨어에서 헤더를 받는 방식으로처리를 해야한다. 서버 컴포넌트인 layout.js 에서는 현재 어떤 슬롯이 활성화 되어있는지 판단 할 수 없다는게 문제다.

이 문제를 해결하기 위해 슬롯이라는 개념으로 현재 어떤 영역이 활성화되었는지 명시적으로 구성할 수 있게 한 것이 Parallel Routes 다.

```ts
// app/layout.tsx
export default function RootLayout({ children, team, analytics }: any) {
  return (
    <div>
      <aside>{team}</aside>    // 명시적 슬롯
      <main>{children}</main>
      <section>{analytics}</section>  // 명시적 슬롯
    </div>
  );
}
```

