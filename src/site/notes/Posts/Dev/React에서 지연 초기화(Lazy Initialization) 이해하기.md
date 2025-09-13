---
{"dg-publish":true,"permalink":"/posts/dev/react-lazy-initialization/","tags":["React"],"created":"2025-04-27","updated":"2025-04-27T22:14:00"}
---


React에서 `useState`를 사용할 때, 초기값을 바로 전달하는 대신 **함수 형태로 넘길 수 있다**. 이 방식을 **지연 초기화(Lazy Initialization)** 라고 부른다.

지연 초기화를 사용하면 **컴포넌트가 처음 렌더링될 때만 초기값을 계산**하고, 이후 렌더링에서는 다시 계산하지 않는다.  
특히 **비용이 큰 초기 연산**이 필요한 경우, 불필요한 연산을 줄여 성능을 최적화할 수 있다.

## 지연 초기화 예시

아래는 최근에 회사에서, 개인적으로 개발한 프로젝트에서도 로컬스토리지에 값을 저장하고 꺼내와 초기값으로 사용할 일이 있었는데, 그 로직을 추상화한 코드다. `localStorage`에 저장된 값을 읽어와 상태로 사용하는 커스텀 훅 `useLocalStorage`으로 만들었다.

```ts

import { useState, useEffect } from "react";

export function useLocalStorage<T>(key: string, initialValue: T) {
  const [storedValue, setStoredValue] = useState<T>(() => {
    try {
      const item = window.localStorage.getItem(key);
      return item ? (JSON.parse(item) as T) : initialValue;
    } catch (error) {
      return initialValue;
    }
  });

  useEffect(() => {
    try {
      window.localStorage.setItem(key, JSON.stringify(storedValue));
    } catch (error) {
      // 에러 핸들링
    }
  }, [key, storedValue]);

  return [storedValue, setStoredValue] as const;
}


```

이 코드에서 주목할 부분은 `useState`의 초기값으로 **함수**를 전달하고 있다는 점이다.

```ts

const [storedValue, setStoredValue] = useState<T>(() => {
  const item = window.localStorage.getItem(key);
  ...
});

```

만약 `useState`에 함수가 아닌 결과값을 직접 넣었다면 `window.localStorage.getItem` 호출이 **컴포넌트가 렌더링될 때마다** 실행됐을 것이다.  
그러나 함수를 전달하면 **컴포넌트가 처음 마운트될 때**만 이 함수가 호출된다. 이후 렌더링에서는 초기값 계산이 다시 일어나지 않는다.

## 지연 초기화가 없으면 생길 수 있는 문제

지연 초기화를 사용하지 않는다면 렌더링할 때마다 불필요한 연산이 반복될 수 있다. 예를 들어 다음과 같은 문제가 생긴다.

- `localStorage.getItem`을 렌더링할 때마다 호출하면 브라우저와 통신하는 비용이 누적된다. 작은 비용이라도 여러 컴포넌트에서 반복되면 전체 렌더링 성능에 악영향을 준다.
- 복잡한 데이터 가공, 예를 들어 대용량 배열을 정렬하거나 필터링하는 연산을 매번 렌더링할 때마다 수행하면 프론트엔드 애플리케이션이 눈에 띄게 느려질 수 있다.
- 예상치 못한 부작용이 발생할 수 있다. 예를 들어 초기화 과정에서 서버에 요청을 보내는 경우, 불필요한 API 호출이 매 렌더링마다 일어날 수도 있다.

지연 초기화를 적용하면 이 모든 문제를 깔끔하게 막을 수 있다.

## 언제 지연 초기화를 사용할까

지연 초기화는 다음과 같은 실제 상황에서 자주 사용된다.

- **로컬 스토리지, 세션 스토리지 읽기**  
    사용자가 이전에 저장해둔 설정값을 불러올 때, 초기 렌더링 시 한 번만 읽어오고 이후에는 메모리 상태만 사용한다.
    - `localStorage`나 `sessionStorage`는 I/O 작업이다.
    - 렌더링마다 스토리지에 접근하면 브라우저 성능에 부담을 줄 수 있다.
    - 초기에 한 번만 읽고 이후에는 메모리에 저장된 상태를 사용해야 한다.
- **초기 데이터 변환**  
    서버에서 받아온 데이터나 JSON 파일을 파싱하거나 변환할 때, 렌더링마다 다시 변환하지 않도록 초기에 한 번만 계산한다.
- **복잡한 연산**  
    예를 들어 차트 데이터처럼 수천 건의 데이터 포인트를 계산하거나 가공할 때, 처음 렌더링 시 한 번만 처리하고 이후에는 저장된 값을 사용한다.
	- 수천, 수만 건의 데이터 정렬이나 필터링 같은 복잡한 연산은 시간이 오래 걸린다.
	- 렌더링마다 다시 계산하면 UI가 느려지고 사용자 경험이 나빠진다.
	- 최초 한 번만 계산하고, 그 결과를 상태로 유지하는 게 좋다.
- **초기 설정값 계산**  
    다크 모드 여부, 언어 설정 같은 사용자의 환경 설정을 초기화할 때, 브라우저 API를 통해 조건을 계산하고 이를 상태로 저장한다.
    - 브라우저 API를 이용해 사용자의 기본 설정(다크 모드, 언어 등)을 초기화할 때 사용한다.
	- 매 렌더링마다 브라우저에 질의하는 것은 비효율적이다.
	- 초기 렌더링 시 딱 한 번만 감지해서 상태로 고정하는 것이 성능에 이롭다.
```ts
const [isDarkMode, setIsDarkMode] = useState(() => {
  return window.matchMedia('(prefers-color-scheme: dark)').matches;
});
```


---

### 정리

- `useState`에 **함수 형태로 초기값을 넘기면 지연 초기화가 된다**.
- **초기 렌더링 시에만 함수가 실행되고**, 이후에는 초기 연산이 반복되지 않는다.
- **성능 최적화**와 **불필요한 연산 방지**를 위해 적극적으로 사용하는 것이 좋다.