---
{"dg-publish":true,"permalink":"/posts/type-safe-routes-feat-nuqs-zod/","tags":["client-side-rendering","Browser","Nextjs","routes","type-safe"],"created":"2025-07-06","updated":"2025-07-06T21:41:00"}
---

Nextjs 14, app router를 사용하는 회사 프로젝트에서 url routes를 활용해 UI를 변경해주거나 페이지 전환을 해주는 케이스를 발견했다. pathname과 querystring을 조합해 경로를 만들거나 조건에 따라 다른 경로로 이동시켜주는 로직이었다.

음악 목록을 인기순/최신순, 장르별로 보여주는 예시로 코드를 작성해봤다. (회사 코드와 무관함.. 회사 코드는 더 복잡한 상태였음)

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

#### `window.location.search` 직접 사용 방식의 문제점

`window.location.search` 를 직접 접근해 사용하고, 문자열을 하드코딩해 직접 조합하는 이 방식은

##### SSR 환경에서 사용 불가능하다. 
- `window`는 브라우저 객체이기 때문에 **Next.js SSR(Server-Side Rendering)** 중에는 `ReferenceError` 발생
- `window.location.search`에 접근하면 **서버에서는 존재하지 않음**

#### URLSearchParmas 쿼리 직접 파싱 방식의 문제점

##### 타입 불안정 (모든 값은 string | null)
- `URLSearchParams.get()`의 반환값은 항상 `string | null`
- `number`, `boolean`, `enum` 등의 값이 필요한 경우 수동으로 파싱하거나 예외 처리해야 함

#### 문자열 직접 조합, 수동 인코딩의 문제점

##### 직접 조합 → 인코딩 누락 가능성
- 수동으로 `?filter=popular&category=kids toys` 같은 쿼리 만들 경우,
    - 공백, 한글, &, ?, = 등은 반드시 `encodeURIComponent()` 로 감싸야 안전
- 실수로 생략하면 **잘못된 URL, 파라미터 깨짐, 보안 이슈** 가능

#### 중복 코드와 반복 로직으로, 재사용이 어렵다는 문제점
- 쿼리 파라미터 조합을 수동으로 처리 → 페이지마다 유사한 로직 반복됨
- 유지보수 어려움, 실수 증가
- 특정 페이지의 쿼리 구조가 바뀌면, 관련된 문자열 조합 로직을 **일일이 찾아서 수정**해야 함
- 쿼리 파라미터가 많아질수록 스파게티 코드화

#### 타입검증이 어렵다는 문제점
##### 타입 검증 없음 → 잘못된 값 허용
- `filter=trendiest` 같이 유효하지 않은 값도 막을 방법이 없음
- Zod, enum 등을 적용할 수 없기 때문에 **런타임 에러 가능성 증가**


#### 테스트가 어려움
- URL 조합이 문자열로 흩어져 있어서,
    - 쿼리 문자열이 올바르게 만들어졌는지 테스트가 불편함
    - 쿼리 구조 변경 시 테스트 코드도 복잡하게 수정해야 함


