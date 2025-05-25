---
{"dg-publish":true,"permalink":"/posts/form-hook-feat-typescript/","tags":["React","type-safe"],"created":"2025-05-25","updated":"2025-05-25T20:59:00"}
---

최근에 우연히 form hook을 직접 만들어볼 일이 있었다. 평소에는 주로 react-hook-form 라이브러리를 사용했었는데, 라이브러리의 도움없이 form 데이터와 상태를 다룰 수 있는 추상화된 React 커스텀 훅을 직접 구현해보았다. 사실 react-hook-form 에도 버그가 있고, form 기능 자체도 다루기 까다로운 편이라고 생각해왔어서, 아주 기본적인 기능에 집중하는 대신 타입 안전성을 보장하는 간단한 훅을 만드는 것을 목표로 했다.

- form value 상태 관리 : 중첩된 객체 형태의 상태도 관리할 수 있게 하자.
- error 상태 관리 : 추상화된 인터페이스로 유효성 검사나 에러 상태를 공통적으로 관리할 수 있게 하자.
- Typescript를 적극 활용해 form 상태와 값 접근을 모두 type-safe하게 하자.

이렇게 3가지 목표를 중심으로 구현을 했다.

폼 상태가 중첩된 객체 형태라면, `user.address.street` 같은 경로를 통해 특정 값을 읽거나 업데이트할 수 있어야 한다. 이를 위해 먼저 중첩된 경로를 타입으로 표현할 필요가 있다.

#### 1. 중첩 경로 타입 만들기 — `DotPath<T>`

```ts

// 중첩 깊이 제한을 위한 배열
export type PrevArr = [never, 0, 1, 2, 3, 4, 5, 6, 7, 8, 9, 10];

// 중첩 객체 경로 타입 생성기 // 배열인 경우: 인덱스에 대한 경로도 허용
export type DotPath<T, Depth extends number = 5, Prefix extends string = ""> = [
  Depth
] extends [never]
  ? never
  : Depth extends 0
  ? never
  : T extends readonly (infer U)[]
  ? 
	| Prefix
    | `${Prefix}${number}` 
    | DotPath<U, PrevArr[Depth], `${Prefix}${number}.`>
  : {
      [K in keyof T & string]: T[K] extends object
        ? `${Prefix}${K}` | DotPath<T[K], PrevArr[Depth], `${Prefix}${K}.`>
        : `${Prefix}${K}`;
    }[keyof T & string];
```

- `DotPath<T>`는 타입 `T` 내에서 접근 가능한 모든 경로 문자열(예: `"user"`, `"user.address"`, `"user.address.street"`)를 만들어낸다.
- 배열도 지원하여, 예를 들어 `"users.0.name"` 같은 경로도 포함한다.
- 최대 재귀 깊이를 제한하는 `Depth` 매개변수로 무한 재귀를 방지한다.

