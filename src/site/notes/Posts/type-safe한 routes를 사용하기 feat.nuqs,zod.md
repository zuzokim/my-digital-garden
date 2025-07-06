---
{"dg-publish":true,"permalink":"/posts/type-safe-routes-feat-nuqs-zod/","tags":["client-side-rendering","Browser","Nextjs","routes","type-safe"],"created":"2025-07-06","updated":"2025-07-06T21:41:00"}
---

Nextjs 14, app router를 사용하는 회사 프로젝트에서 url routes를 활용해 UI를 변경해주거나 페이지 전환을 해주는 케이스를 발견했다. pathname과 querystring을 조합해 경로를 만들거나 조건에 따라 다른 경로로 이동시켜주는 로직이었다. 이참에 좀더 라우트를 type-safe하게 관리하는 방법들을 공부해보기로 했다.

음악 목록을 인기순/최신순, 장르별로 보여주는 예시 코드를 작성해보자.

```ts

function MyComponent() {
// 클라이언트에서 window 직접 사용 → SSR 위험, 인코딩/디코딩 수동 처리 필요
  const pathname = window.location.pathname
  const search = window.location.search // "?filter=popular&category=music"

  // 쿼리 직접 파싱 (간단하게)
  const params = new URLSearchParams(search)
  const filter = params.get('filter') // string | null
  const category = params.get('category')

  const handleClick = () => {
    const newFilter = 'latest'
    const newCategory = 'sports'

    // 수동 인코딩, 조합 필요
    const query = `?filter=${encodeURIComponent(newFilter)}&category=${encodeURIComponent(newCategory)}`

    window.location.href = pathname + query
  }

  return (
    <div>
      <p>Filter: {filter}</p>
      <p>Category: {category}</p>
      <button onClick={handleClick}>Change</button>
      ...
    </div>
  )
}
```

이 코드의 문제점은 뭘까?

#### ⚠️`window.location.search` 직접 사용 방식의 문제점

`window.location.search` 를 직접 접근해 사용하고, 문자열을 하드코딩해 직접 조합하는 이 방식은

##### ⚠️SSR 환경에서 사용 불가능하다. 
- `window`는 브라우저 객체이기 때문에 **Next.js SSR(Server-Side Rendering)** 중에는 `ReferenceError` 발생
- `window.location.search`에 접근하면 **서버에서는 존재하지 않음**

#### ⚠️URLSearchParmas 쿼리 직접 파싱 방식의 문제점

##### 타입 불안정 (모든 값은 string | null)
- `URLSearchParams.get()`의 반환값은 항상 `string | null`
- `number`, `boolean`, `enum` 등의 값이 필요한 경우 수동으로 파싱하거나 예외 처리해야 함

#### ⚠️문자열 직접 조합, 수동 인코딩의 문제점

##### 직접 조합 → 인코딩 누락 가능성
- 수동으로 `?filter=popular&category=kids toys` 같은 쿼리 만들 경우,
    - 공백, 한글, &, ?, = 등은 반드시 `encodeURIComponent()` 로 감싸야 안전
- 실수로 생략하면 **잘못된 URL, 파라미터 깨짐, 보안 이슈** 가능

#### ⚠️중복 코드와 반복 로직으로, 재사용이 어렵다는 문제점
- 쿼리 파라미터 조합을 수동으로 처리 → 페이지마다 유사한 로직 반복됨
- 유지보수 어려움, 실수 증가
- 특정 페이지의 쿼리 구조가 바뀌면, 관련된 문자열 조합 로직을 **일일이 찾아서 수정**해야 함
- 쿼리 파라미터가 많아질수록 스파게티 코드화

#### ⚠️타입검증이 어렵다는 문제점
##### 타입 검증 없음 → 잘못된 값 허용
- `filter=trendiest` 같이 유효하지 않은 값도 막을 방법이 없음
- Zod, enum 등을 적용할 수 없기 때문에 **런타임 에러 가능성 증가**


#### ⚠️테스트가 어려움
- URL 조합이 문자열로 흩어져 있어서,
    - 쿼리 문자열이 올바르게 만들어졌는지 테스트가 불편함
    - 쿼리 구조 변경 시 테스트 코드도 복잡하게 수정해야 함


### Zod를 활용해 리팩토링해보자.

#### Zod란?

**Zod** (https://zod.dev/)는 타입스크립트를 위한 타입 안전하고 선언적인 **런타임 유효성 검사 및 파싱 라이브러리**다.

- TypeScript 타입처럼 `스키마`를 정의하면,
- 실제 런타임에서 **데이터 구조를 검증하고, 안전하게 파싱**할 수 있게 도와준다.

Zod의 특징은:
- 타입 추론이 (`z.infer<typeof schema>`)
- `safeParse()`로 에러 핸들링을 안전하게 처리
- `.optional()`, `.default()`, `.refine()` 등 유연한 검증 API 제공

```ts
// constants.ts
export const FILTER_OPTIONS = ['popular', 'latest'] as const
export type FilterOption = (typeof FILTER_OPTIONS)[number]

```

```ts
// schemas.ts
import { z } from 'zod'
import { FILTER_OPTIONS } from './constants'

// 쿼리 스키마 정의
export const QuerySchema = z.object({
  filter: z.enum(FILTER_OPTIONS).optional(),
  category: z.string().optional(),
})
```

```ts
import { QuerySchema } from './schemas'

function MyComponent() {
  const params = new URLSearchParams(window.location.search)

  // zod 안전 검증
  const rawQuery = {
    filter: params.get('filter') ?? undefined,
    category: params.get('category') ?? undefined,
  }

  const parseResult = QuerySchema.safeParse(rawQuery)

  const filter = parseResult.success ? parseResult.data.filter : 'popular'
  const category = parseResult.success ? parseResult.data.category : undefined

  const handleClick = () => {
    const newFilter = 'latest'
    const newCategory = 'sports'

    // 여전히 문자열 직접 조합
    const query = `?filter=${encodeURIComponent(newFilter)}&category=${encodeURIComponent(newCategory)}`

    window.location.href = window.location.pathname + query
  }

  return (
    <div>
      <p>Filter: {filter}</p>
      <p>Category: {category}</p>
      <button onClick={handleClick}>Change</button>
      ...
    </div>
  )
}
```

이렇게 하면 위에서 말한 문제점 일부를 개선할 수 있다.


#### ✅ Zod로 개선
##### type-safe : filter 값이 FILTER_OPTIONS enum에 포함되어야 한다.
##### 재사용 가능 : API나 form 등에서도 재사용이 가능하다.
##### 에러 방지 : 유효하지 않은 타입과 값을 검증할 수 있다.
##### 코드 가독성 향상 : enum을 한 곳에서 관리해서 유지보수가 쉽다.

### Nuqs를 활용해 한 번 더 리팩토링해보자.

##### Nuqs란?

Nuqs(https://nuqs.47ng.com/) 는 공식 문서에도 소개되어있듯이, "A type-safe query string state manager for Next.js App Router" 즉, 쿼리스트링을 타입 안전하게 읽고 쓰기 위한 React 훅 라이브러리다.

```ts
import { QuerySchema } from './schemas'
import { useQueryStates } from 'nuqs'
import { parseAsZod } from 'nuqs/zod'

export default function MyComponent() {
  // Nuqs + Zod 통합 parser 사용해 자동 파싱 + 직렬화 + 상태관리
  const [query, setQuery] = useQueryStates(parseAsZod(QuerySchema))

  const { filter, category } = query

  return (
    <div>
      <p>Filter: {filter}</p>
      <p>Category: {category}</p>

      <button onClick={() => setQuery({ filter: 'latest', category: 'sports' })}>
        Change
      </button>
      ...
    </div>
  )
}

```

##### `parseAsZod(QuerySchema)` 가 해결하는 문제
- 문자열로만 들어오는 쿼리 파라미터를 Zod 스키마가 타입 명세와 함께 `.parse()`로 자동 파싱
- `z.enum`, `z.string()`, `z.number()` 등으로 검증 및 기본값 제공
- Zod 스키마의 `.optional()`, `.default()` 등 사용해 유효성 검증 조건 처리
- `safeParse`로 런타임에서 쿼리 값이 유효한지 자동 검증

##### `useQueryStates(...)` 가 해결하는 문제
- `URLSearchParams`를 직접 조작하면 상태관리가 어려운데, React `useState`처럼 쓰게 해줌
- 쿼리 구조가 복잡할 수록 가독성이 저하되는데, `useQueryStates()`가 선언적으로 구조 관리
- 라우터 상태 업데이트시 새로고침이 필요한데, `setQuery()`로 soft navigation (history.push) 가능
- 수동인코딩을 빼먹는 휴먼에러를 방지하고, 내부에서 자동으로 `encodeURIComponent` 처리
- `useSearchParams()` 기반으로 동기화되어 CSR에서 안정적 동작하므로, SSR/CSR 간 상태 불일치를 방지
	- `useSearchParams()` 는 Next.js가 **자동으로 "hydration-safe client-only hook"** 로 처리하는 Client 전용 hook 이기 때문에, SSR 단계에서는 호출되지 않음
	- SSR에서도 쿼리 파라미터가 필요하면 **`searchParams`를 props로 받는 Server Component 구조를 이용** 

### 서버 ↔ 클라이언트 쿼리 구조 일치

서버컴포넌트에서도 쿼리파라미터가 필요하면 searchParams를 props로 받으면 되는데..
이런 문제가 발생할 수 있다.

서버 컴포넌트 (page.tsx)
```ts
const filter = searchParams.filter ?? 'popular' // 문자열 그대로
```

클라이언트 컴포넌트(client.tsx)
```ts
const [query, setQuery] = useQueryStates({
  filter: parseAsEnum(['popular', 'latest']).withDefault('latest'),
})
```

### 문제점
- 서버에선 기본값을 `'popular'`, 클라이언트에선 `'latest'`
- 사용자는 `"popular"`로 접근했는데, 클라이언트에선 `"latest"`로 바뀌어 보임  
    → **같은 URL인데 서로 다른 동작** → 버그 발생

#### 해결 방법 : "쿼리 파서 구조를 일치"시켜서 서버/클라이언트 공통화

```ts
// lib/parsers.ts
// Nuqs 파서 정의
import { parseAsString, parseAsEnum } from 'nuqs'
import { QuerySchema } from './schemas'

export const pageParsers = {
  filter: parseAsEnum(FILTTER_OPTIONS).withDefault('popular'),
  category: parseAsString.optional(),
}
```

```ts
// app/page.tsx
// Server Component (SSR)에서 searchParams 받기
import { pageParsers } from '@/lib/parsers'
import { PageClient } from './client'

export default function Page({
  searchParams,
}: {
  searchParams: { [key: string]: string | string[] | undefined }
}) {
  // SSR-safe 쿼리 파싱
  const filter = pageParsers.filter.parse(searchParams.filter ?? null)
  const category = pageParsers.category?.parse?.(searchParams.category ?? null)

  return <PageClient filter={filter} category={category} />
}
```

```ts
// app/client.tsx
// Client Component에서 동일 파서 재사용 가능
'use client'

import { useQueryStates } from 'nuqs'
import { pageParsers } from '@/lib/parsers'

export function PageClient(props: { filter: string; category?: string }) {
  const [{ filter, category }, setQuery] = useQueryStates(pageParsers)

  return (
    <div>
      <h1>Filter: {filter}</h1>
      <h2>Category: {category}</h2>

      <button onClick={() => setQuery({ filter: 'latest' })}>
        Change
      </button>
      ...
    </div>
  )
}

```

이렇게 `pageParsers` 를 서버/클라이언트에서 재사용하면 쿼리 구조(키, 타입, 기본값, 유효성)가 완전히 동일하게 유지된다. 굳!



