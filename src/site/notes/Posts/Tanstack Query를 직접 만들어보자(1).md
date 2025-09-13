---
{"dg-publish":true,"permalink":"/posts/tanstack-query-1/","created":"2025-06-21","updated":"2025-06-21T03:12:00"}
---

미니버전의 Tanstack Query를 직접 만들어보자. 

### 1차 구현

기본적으로 querykey, queryFn을 넘겨 네트워크요청을 할 수 있도록 커스텀 React 훅을 만드는 것부터 해보자.

```ts
import { useEffect, useRef, useState } from 'react';

// eslint-disable-next-line @typescript-eslint/no-explicit-any
type UseQueryProps<T = any> = {
  queryKey: string;
  queryFn: () => T | Promise<T>;
};

// eslint-disable-next-line @typescript-eslint/no-explicit-any
export function useQuery<TData = any, TError = any>({
  queryKey,
  queryFn,
}: UseQueryProps) {
  const [data, setData] = useState<TData | undefined>(undefined);
  const [error, setError] = useState<TError | undefined>(undefined);
  const [isPending, setIsPending] = useState(true);

  useEffect(() => {
    let ignore: boolean = false;
    const fetchData = async () => {
      try {
        const result = await queryFn();
        if (!ignore) {
          setData(result);
          setIsPending(false);
        }
      } catch (err) {
        if (!ignore) {
          setError(err as TError);
          setIsPending(false);
        }
      }
    };
    fetchData();
    return () => {
      ignore = true;
    };

  }, [queryKey, queryFn]);

  return {
    isPending,
    error,
    data,
  };
}
```

로딩, 데이터, 에러 상태를 리턴하도록 했다.

### 문제 상황 1

`useEffect`에서 dependencies array에 `queryFn`을 넣으면 무한루프에 빠지는데 그 이유는 `useEffect`훅을 호출하는 컴포넌트가 리렌더링 될 때마다 함수 `queryFn`의 참조값이 변경되어 `useEffect`가 다시 실행되기 때문이다. 

`useEffect([... , queryFn])`처럼 `queryFn`을 의존성에 넣으면:

1. 컴포넌트 렌더링 → `queryFn` 새로 생성됨
2. `useEffect` 재실행 → `setState` 발생
3. `setState` → 렌더링 다시 일어남 → 또 새로운 `queryFn`
4. 다시 `useEffect` 실행...


### 해결 1

이를 해결하기 위해 `queryFn` 를 dependencies array에서 제거하는 방법을 택했다.

혹은 `useCallback` 으로 `queryFn` 를 랩핑해 참조 안정성을 보장할 수도 있다. 이렇게..

```ts
const memoizedQueryFn = useCallback(queryFn, [queryKey]);

useEffect(() => {
	...
},[queryKey, memoizedQueryFn])
```

### 문제 상황 2

근데 이렇게 했을 때의 문제점은 뭘까?

`queryFn`은 `useQuery` 훅 외부에서 prop으로 주입되고, `queryKey` 와 동기화된다는 보장이 없기 때문이다. 즉, `queryKey`가 같다고 `queryFn` 도 같다고 볼 수 없다. (사용하는 컴포넌트에서❗ 함수가 새로 생성되어 주입될 수 있기 때문)

- `queryKey`가 같아도, `queryFn`이 매 렌더마다 새로 생성된다면, `useCallback(queryFn, [...])`은 아무런 의미가 없다.
- `useCallback`이 효과를 발휘하려면, **첫 번째 인자로 넘기는 함수가 렌더링마다 동일한 참조여야 한다.**  
- `queryKey`는 그저 의존성 중 하나일 뿐이고, 함수 메모이제이션의 본질엔 영향을 주지 않는다.
- `useCallback` 은 "첫 번째 인자"인 함수 자체가 새로 생성되지 않아야 진짜 메모이제이션 효과가 생긴다.
	- `useCallback`의 목적은 "함수 인스턴스를 *기억(memoize)*하는 것"
	- `useCallback(fn, deps)`는 이렇게 작동한다 : "deps가 바뀌지 않았다면, 이전 렌더에서 만든 `fn`을 재사용한다."

```ts
const someFn = () => fetchData(userId); // 매 렌더마다 새 함수
const memoizedFn = useCallback(someFn, [queryKey]);
```

여기서 문제가 뭐냐면:
- `someFn`은 컴포넌트 함수가 실행될 때마다 **항상 새로운 함수 리터럴**로 만들어짐
- 즉 `useCallback(새로운 함수, [queryKey])`처럼 호출됨
- 그럼 React 입장에서 비교할 수가 없음 → 이전과 다르다고 판단

제대로 사용하려면 이렇게 해야되지만 잘못쓰기 쉽다..

```ts
const memoizedFn = useCallback(() => fetchUser(userId), [userId]);
```


|항목|설명|
|---|---|
|✅ `useCallback(() => fetch(userId), [userId])`|유효: 의존성만 바뀔 때만 새 함수 생성|
|❌ `useCallback(queryFn, [queryKey])` (queryFn이 렌더마다 새로 만들어짐)|무효: 매번 새 함수라 비교 불가|
|queryKey|단순한 값일 뿐, 함수 identity와 무관|

### 문제 상황 3

Tanstack query가 실제로 어떻게 구현되어있는지까지는 모르지만 기능적 특성상 `queryKey`를 함수의 결과를 식별하는 ID로써 사용하기 때문에 "키가 다르면 쿼리함수도 다른 것으로 간주한다"는 관점에선 괜찮지 않나 하는 생각도 처음엔 들었다.

그러나!

```ts
const [userId, setUserId] = useState(1);

useQuery({
  queryKey: ['user', userId],
  queryFn: () => fetchUser(userId), // 외부 상태에 의존
});
```

예를 들어 이렇게 queryFn이 외부 상태나 값에 의존하는 경우에  `queryKey = ['user', 2]` 인데, 실제로는 `fetchUser(1)` 이 호출될 것이다. 그러면 키는 바뀌었는데 네트워크 요청은 이전 상태값으로 하게되는 치명적인 버그가 발생할 수 있을 듯 했다. ~~(뭔가 회사에서 캐시 업데이트할 때 이런 경우가 많았다.. 왜이렇게 익숙하지.. 이버그..)~~

그래서 이 구현은 좋지 못하다는 결론에 이르렀는데, 


#### 조금 더 깊게 들여다보기 with 클로저

조금 더 들여다보자. 왜 이런걸까?

여기서 등장하는 **클로저**! 이건 **클로저가 상태를 오래 기억해서 발생하는 버그**다.

```ts
function UserDetail() {
	const [userId, setUserId] = useState(1);
	
	const queryFn = () => fetchUser(userId); 
	
	useQuery({
	  queryKey: ['user', userId],
	  queryFn, // 이건 이전 렌더의 userId를 **클로저**로 기억하고 있을 수 있음
	});
}
```

클로저 개념을 짚고 넘어가자.

클로저란 "함수가 선언될 당시의 렉시컬 스코프(변수 환경)를 기억하는 함수" 다.

- 어떤 함수가 **자기 바깥 스코프의 변수**를 참조하고 있을 때,
- 그 함수가 **해당 변수와 함께 묶여 있는 상태**로 존재하는 것.

```ts
function createCounter() {
  let count = 0;

  return function () {
    count += 1;
    return count;
  };
}

const counter = createCounter();
console.log(counter()); // 1
console.log(counter()); // 2
```

여기서 `count`는 `createCounter`가 끝났는데도 살아 있다. 왜냐하면 `return`된 함수가 `count`를 "기억하고 있는 클로저"이기 때문이다.

이 코드에서 `queryFn`은 `userId`를 **현재 렌더 시점의 값으로 캡처** 한다.
즉, **이 렌더링 시점에 선언된 `userId`와 묶여 있는 클로저**가 된다. 
그리고 이후 `userId`가 `2`로 바뀌어도,  이 `queryFn`은 여전히 옛날 `userId = 1` 을 기억하고 있다!

### 해결 2

그러면 어떻게 해결할까?

클로저는 함수를 선언한 시점의 변수 상태를 기억한다.  
React에서 상태는 계속 바뀌므로, 클로저가 **옛날 상태를 기억한 채** 유지되는 걸 막기 위해서는 `queryFn`이나 콜백 함수가 **최신 상태를 캡처할 수 있도록** 정의하거나 의존성을 명확히 해야한다.

- `useQuery` 를 호출하는 컴포넌트에서 `useCallback` 으로 랩핑후 넘기기
```ts
import { useCallback, useState } from 'react';
import { useQuery } from './useQuery';

function UserDetail({ userId }: { userId: number }) {
  const queryFn = useCallback(() => fetchUser(userId), [userId]);

  const { data, isPending, error } = useQuery({
    queryKey: ['user', userId],
    queryFn, // 호출하는 컴포넌트에서 메모이징된 함수 전달
  });

  if (isPending) return <p>Loading...</p>;
  if (error) return <p>Something went wrong</p>;

  return <div>{data?.name}</div>;
}
```

### 해결 3


근데.. 매번 사용하는 곳에서 랩핑하는 게 불편하다는 생각이 들었다. `useQuery` 내부에서 알아서 방어 처리되면 좋을 . 것 같은데.. 라는 생각을 했고 아래와 같이 구현해봤다.

```ts


import { useEffect, useRef, useState } from 'react';

// eslint-disable-next-line @typescript-eslint/no-explicit-any
type UseQueryProps<T = any> = {
  queryKey: string;
  queryFn: () => T | Promise<T>;
};

// eslint-disable-next-line @typescript-eslint/no-explicit-any
export function useQuery<TData = any, TError = any>({
  queryKey,
  queryFn,
}: UseQueryProps) {
  const [data, setData] = useState<TData | undefined>(undefined);
  const [error, setError] = useState<TError | undefined>(undefined);
  const [isPending, setIsPending] = useState(true);
 
  const queryFnRef = useRef(queryFn);
  if (queryFnRef.current !== queryFn) {
    console.warn(
      '[useQuery] queryFn이 렌더링 되는 동안 변경되었습니다. ' +
        '이는 불필요한 fetches 또는 infinite loops를 일으킬 수 있습니다. queryFn을 useCallback으로 wrapping하는 것을 고려해보세요.'
    );

    queryFnRef.current = queryFn;
  }

  useEffect(() => {
    let ignore: boolean = false;
    const fetchData = async () => {
      try {
        const result = await queryFnRef.current();
        if (!ignore) {
          setData(result);
          setIsPending(false);
        }
      } catch (err) {
        if (!ignore) {
          setError(err as TError);
          setIsPending(false);
        }
      }
    };
    fetchData();
    return () => {
      ignore = true;
    };
    // eslint-disable-next-line react-hooks/exhaustive-deps
  }, [queryKey]);

  return {
    isPending,
    error,
    data,
  };
}
```

이렇게 하면 
- `queryFn`이 **외부 상태를 캡처할 수 있음**을 감안해, `useCallback`을 쓰도록 유도하는 구조로 개선
- 그러나 사용하는 쪽에서 실수 혹은 누락해도 **실행은 안전하게**, **경고는 명확하게** 제공하도록 개선
할 수 있다.


기본 기능만 했는데도 생각할 게 많다..
나중에 아래 소스코드로 실제 Tanstack Query가 어떻게 구현했는지 들여다 봐야겠다.

---

https://github.com/TanStack/query/blob/33d008bbb39f749588ab591d41fefa53f4e18c99/packages/query-core/src/queryClient.ts

https://github.com/TanStack/query/blob/33d008bbb39f749588ab591d41fefa53f4e18c99/packages/query-core/src/queryCache.ts

https://github.com/TanStack/query/blob/33d008bbb39f749588ab591d41fefa53f4e18c99/packages/query-core/src/query.ts#L297
