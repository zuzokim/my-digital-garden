---
{"dg-publish":true,"permalink":"/posts/dev/next-js-parallel-routes/","tags":["Nextjs"],"created":"2025-06-06","updated":"2025-06-06T20:07:00"}
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
- **Soft Navigation** - 클라이언트 사이드 네비게이션에서는 슬롯 내 하위 페이지를 부분적으로 렌더하는데, 이때 현재 URL과 매치가 안되더라도, 기존에 활성화되어있는 하위 페이지를 유지한채로 렌더가 된다. (전체 페이지를 다시 그리지 않고)
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

이 문제를 해결하기 위해 슬롯이라는 개념으로 현재 어떤 영역이 활성화되었는지 명시적으로 구성할 수 있게 한 것이 Parallel Routes 다. 다시 말하면, 단순한 레이아웃 분리 기능을 넘어, "경로 기반 UI 상태 관리"를 가능하게 해준다고 볼 수 있다.

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

### Intercepting Routes : 모달과 탭 UI로 Parallel Routes 이해해보기

위에서 잠깐 언급한 Intercepting Routes는 **기존 경로의 흐름을 중단하지 않고**, 특정 URL 경로를 슬롯에 "가로채서(intercept)" 렌더링할 수 있는 기능이다. 

쉽게 말하면, 유저는 `/users/123` 경로로 이동했지만, 그 페이지를 전체 리디렉션 없이 모달로 렌더링할 수 있게 해주는 방식이다.

#### 예시 ① — 모달을 Intercept 해보자

- `/users`에서 유저 목록을 보고,
- 유저를 클릭하면 `/users/123` 경로로 이동
- 하지만 전체 페이지가 이동하지 않고, 기존 리스트 위에 모달로 유저 상세가 뜨게 한다

```json
// 디렉토리 폴더 구조
app/
  users/
    page.tsx                  // 유저 목록
    @modal/
      [id]/
        page.tsx              // 유저 상세 (모달로 사용)
      default.tsx             // 아무 모달도 없는 기본 상태
    layout.tsx                // 슬롯 정의
```

```ts
// app/users/layout.tsx
export default function UsersLayout({
  children,
  modal,
}: {
  children: React.ReactNode;
  modal: React.ReactNode;
}) {
  return (
    <>
      {children}
      {modal}
    </>
  );
}
```

```ts
// app/users/page.tsx (유저 목록)
'use client';

import Link from 'next/link';

export default function UsersPage() {
  return (
    <div>
      <h1>유저 목록</h1>
      <ul>
        <li><Link href="/users/1">유저 1</Link></li>
        <li><Link href="/users/2">유저 2</Link></li>
      </ul>
    </div>
  );
}
```

```ts
// app/users/@modal/[id]/page.tsx (유저 상세 모달)
'use client';

import { useRouter } from 'next/navigation';

export default function UserModal({ params }: { params: { id: string } }) {
  const router = useRouter();

  return (
    <div className="modal">
      <h2>유저 {params.id} 상세 보기</h2>
      <button onClick={() => router.back()}>닫기</button>
    </div>
  );
}
```

```ts
// app/users/@modal/default.tsx
export default function DefaultModal() {
  return null; // 모달이 없는 상태
}

```


이렇게 하면 아래와 같은 동작 흐름이 나온다.

| **URL**    | **렌더링 결과**                           |
| ---------- | ------------------------------------ |
| `/users`   | 유저 목록만 표시                            |
| `/users/1` | 유저 목록 + 유저 1의 상세 모달 (슬롯 @modal로 렌더링) |
| 새로고침       | 모달 상태 유지 (`/users/1` 경로 그대로)         |

#### 예시 ② — 탭 UI도 병렬로 렌더링해보자

- `/dashboard`에서는 기본 탭(Overview)
- `/dashboard/analytics`로 이동하면 다른 탭만 바뀌고 나머지 레이아웃은 그대로
- 즉, 여러 탭이 독립적으로 렌더링됨

```json
// 디렉토리 구조
app/
  dashboard/
    layout.tsx
    @overview/
      page.tsx
    @analytics/
      page.tsx
    @settings/
      page.tsx

```

```ts
// app/dashboard/layout.tsx
export default function DashboardLayout({
  overview,
  analytics,
  settings,
}: {
  overview: React.ReactNode;
  analytics: React.ReactNode;
  settings: React.ReactNode;
}) {
  return (
    <div>
      <nav>
        <Link href="/dashboard">Overview</Link>
        <Link href="/dashboard/analytics">Analytics</Link>
        <Link href="/dashboard/settings">Settings</Link>
      </nav>
      <section className="tabs">
        {overview}
        {analytics}
        {settings}
      </section>
    </div>
  );
}

```

이렇게 하면 아래와 같은 동작 흐름이 나온다.

- `/dashboard`: `@overview` 탭 렌더링
- `/dashboard/analytics`: `@analytics` 탭 렌더링 (나머지는 그대로 유지)
- 탭 간 전환 시 전체 페이지 리로딩 없음
- 각 탭은 **URL과 강하게 연결되며**, 새로고침 시에도 해당 탭이 유지됨

#### 정리

기존 방식대로 클라이언트 상태관리를 하면 모달은 `useState` 훅 등으로 `isOpen` 과 같은 상태에 따라 띄우고 URL과 무관하게 동작하는 반면, Intercepting Routes를 사용하면 모달을 URL로 제어하고, 새로고침시 모달 상태를 보존할 수 있다.

탭은 탭 전환시 전체 페이지를 이동하는 방식에서 탭만 독립적으로 렌더링하고, 마찬가지로 URL 경로 기반으로 상태 보존이 가능해져 UX를 자연스럽게 만들 수 있다.

클라이언트 상태관리 방식으로 탭을 구현했다면 아래와 같이 했을 것 같은데, 

```ts
'use client';

import { useState } from 'react';

export default function DashboardPage() {
  const [tab, setTab] = useState<'overview' | 'analytics' | 'settings'>('overview');

  return (
    <div>
      <nav>
        <button onClick={() => setTab('overview')}>Overview</button>
        <button onClick={() => setTab('analytics')}>Analytics</button>
        <button onClick={() => setTab('settings')}>Settings</button>
      </nav>
      <section>
        {tab === 'overview' && <Overview />}
        {tab === 'analytics' && <Analytics />}
        {tab === 'settings' && <Settings />}
      </section>
    </div>
  );
}

```

이 방식은 단순하긴 하지만, 탭 별로 다른 콘텐츠를 그리더라도 유저가 어떤 탭을 보고 있는지 URL로 표현할 수 없다. 탭 전환을 브라우저 히스토리에 담을 수 없으니 앞/뒤로 가기가 불가하고, SEO 등에 불리할 수 있다. 또, 새로고침시 활성 탭 상태를 보존할 수 없다는 UX적 한계가 있다.

